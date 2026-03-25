# DiffSinger + MakeDiffSinger 一体化保姆级教程（从 0 到可训练可推理）

> 分流导航（按目标效果跳转）：`learn_docs/README.md`
>
> - 快速数据制作：`learn_docs/01_快速数据制作_30分钟.md`
> - 仅 acoustic 出声：`learn_docs/02_仅Acoustic出声_不训Variance.md`
> - 故障排查：`learn_docs/04_故障排查_Acoustic首声.md`

> 适用对象：
>
> - 你有原始人声录音，想做自己的 DiffSinger 声库；
> - 你觉得官方文档分散、脚本多、容易踩坑；
> - 你希望一份从数据制作到训练、推理、导出的完整执行手册。

---

## 0. 先说结论：两个仓库分别做什么？

- `openvpi/MakeDiffSinger`：做数据（切片、标注、强制对齐、补 variance 字段）。
- `openvpi/DiffSinger`：训练、推理、导出 ONNX。

你要的最终目标是：

1. 用 MakeDiffSinger 做出符合 DiffSinger 要求的数据集目录。
2. 用 DiffSinger 训练 `acoustic` 和（可选）`variance` 模型。
3. 先 `variance` 再 `acoustic` 推理出 wav。

---

## 1. 本教程校验基线（最新主线）

校验时间：**2026-03-26**

- DiffSinger: `openvpi/DiffSinger` `main`
  - 最新提交：`ebc3805f941a14490a8e8817d0a4553fb94c7945`
  - 提交时间：`2026-01-14 21:54:01 +0800`
- MakeDiffSinger: `openvpi/MakeDiffSinger` `main`
  - 最新提交：`1edcd6661d7ee4f642a702c32b65e45fd73beb41`
  - 提交时间：`2026-03-11 16:26:36 +0800`

这份教程按以上代码行为编写，不按旧博客/旧 issue 猜。

---

## 2. 你会做出的目录（先看成品）

建议先统一工作目录（示例）：

```text
/work/svs/
  DiffSinger/
  MakeDiffSinger/
  dataset-tools/               # 可选，GUI 工具
  datasets/
    my_singer/
      raw/
        wavs/
        transcriptions.csv
      binary_acoustic/
      binary_variance/
```

DiffSinger 真正吃的是：

```text
datasets/my_singer/raw/
  wavs/*.wav
  transcriptions.csv
```

---

## 3. 环境准备（建议 3 套环境，最稳）

### 3.1 环境 A：MFA 对齐环境（MakeDiffSinger acoustic_forced_alignment）

```bash
git clone https://github.com/openvpi/MakeDiffSinger.git
cd MakeDiffSinger/acoustic_forced_alignment

conda create -n mds_mfa python=3.8 --yes
conda activate mds_mfa
conda install -c conda-forge montreal-forced-aligner==2.0.6 --yes
pip install -r requirements.txt
```

注意：

- README 有些地方写 `acoustic-forced-alignment`，但仓库真实目录是 **`acoustic_forced_alignment`**（下划线）。

### 3.2 环境 B：DiffSinger 训练/推理环境

```bash
git clone https://github.com/openvpi/DiffSinger.git
cd DiffSinger

# 先按你的 CUDA/CPU 安装 PyTorch（官方命令）
pip install -r requirements.txt
```

### 3.3 环境 C：ONNX 导出环境（必须 PyTorch 1.13.x）

```bash
cd DiffSinger
# 新环境后：
pip install -r requirements-onnx.txt
```

`export.py` 代码里对版本是硬检查：必须 `1.13.x`。

---

## 4. 数据制作总流程（不迷路版）

### 4.1 一句话流程图

1. 切片 + `.lab` 标注（dataset-tools: AudioSlicer + MinLabel）
2. MakeDiffSinger `acoustic_forced_alignment` 强制对齐，生成 `ph_seq/ph_dur`
3. 产出可训练 acoustic 的 `transcriptions.csv`
4. （可选）MakeDiffSinger `variance-temp-solution` 扩展 `ph_num/note_seq/note_dur`
5. DiffSinger 训练 acoustic / variance
6. 推理时先 variance 再 acoustic

### 4.2 先准备输入（支持跳过自制字典）

你至少要有：

1. 切好的 wav（建议 5~15 秒一条）
2. 每条 wav 对应同名 `.lab`（空格分隔词/音节）
3. 一个可用字典文件（`syllable<TAB>phoneme phoneme ...`）
4. 与该字典匹配的 MFA 声学模型（zip）

字典这一步建议分两种路径：

- 路径 A（推荐）：**跳过自制字典**，直接用现成标准字典（很多人实际都这么做，包括中/日/英/粤/韩/德/西等语种的社区字典）。
- 路径 B（可选）：自己制作或改造字典（只在你有特殊音素体系需求时再做）。

实践上，你只要保证两件事就能跳过“造字典”这件事：

1. `.lab` 里的词条都在你选的标准字典里；
2. 你手上的 MFA 模型与该字典是同一套方案（同一来源/同一版本）。

补充：当前这两个仓库内置示例字典主要是 `opencpop-extension.txt`（中文），其他语言通常使用外部标准字典。

### 4.3 30 分钟快跑版（先出可训练数据）

如果你现在目标只是“尽快做出可训练数据”，可以先走这个最小路径：

1. 准备好 `segments/*.wav + 同名 .lab + 标准字典 + 匹配 MFA 模型`。
2. 跑 `validate_labels.py`（先保证词条可识别）。
3. 跑 `reformat_wavs.py`（给 MFA 用）。
4. 跑 `mfa align`（得到初始 TextGrid）。
5. 跑 `enhance_tg.py`（补 AP/SP，修对齐）。
6. 跑 `build_dataset.py`（产出 `raw/wavs + transcriptions.csv`）。

这条快跑路径里，你可以先后置这些环节：

- `validate_lengths.py`
- `check_tg.py`
- 整套人工精修链：`combine_tg.py` / `align_tg_words.py` / `slice_tg.py`
- `summary_pitch.py`

后面模型稳定了，再回头补精修流程。

---

## 5. MakeDiffSinger 第 1 段：`acoustic_forced_alignment`

下面是可直接执行的标准顺序（包含质量增强项）。如果你要快速起步，优先按第 4.3 节跑最小路径。

### 5.1 质检时长

```bash
cd MakeDiffSinger/acoustic_forced_alignment
python validate_lengths.py --dir /path/to/segments
```

规则（代码写死）：

- `< 2s` 报 Too short
- `> 20s` 报 Too long

### 5.2 质检标注

```bash
python validate_labels.py \
  --dir /path/to/segments \
  --dictionary /path/to/dictionary.txt
```

这里的 `--dictionary` 可以直接指向你下载的标准字典文件，不要求你先自制字典。

它会检查：

- wav 是否有同名 lab
- lab 是否有 OOV 词条
- 字典音素是否被数据覆盖
- 输出 `phoneme_distribution.jpg`

### 5.3 重采样给 MFA

```bash
python reformat_wavs.py \
  --src /path/to/segments \
  --dst /path/to/mfa_input
```

- 输出 16kHz / mono / PCM_16 wav
- 同时复制 lab
- 可加 `--normalize` 做峰值归一化

### 5.4 跑 MFA 强制对齐

```bash
mfa align \
  /path/to/mfa_input \
  /path/to/dictionary.txt \
  /path/to/mfa_model.zip \
  /path/to/mfa_textgrids \
  --beam 100 --clean --overwrite
```

注意 `dictionary.txt` 和 `mfa_model.zip` 必须是同一套方案（匹配版本）。

### 5.5 检查 TextGrid 完整性

```bash
python check_tg.py \
  --wavs /path/to/mfa_input \
  --tg /path/to/mfa_textgrids
```

如果有缺失，优先排查：

- lab 中非法词条
- wav 质量/时长异常
- `--beam` 太小

### 5.6 增强 TextGrid（补 AP/SP、修长句）

```bash
python enhance_tg.py \
  --wavs /path/to/mfa_input \
  --dictionary /path/to/dictionary.txt \
  --src /path/to/mfa_textgrids \
  --dst /path/to/final_textgrids
```

初次建议默认参数，后续再调这些阈值：

- `--br_len --br_db --br_centroid`
- `--voicing_thresh_vowel --voicing_thresh_breath`

### 5.7（可选）人工精修 TextGrid

#### 5.7.1 合并短句为长句（便于人工修）

```bash
pip install natsort
python combine_tg.py \
  --wavs /path/to/mfa_input \
  --tg /path/to/final_textgrids \
  --out /path/to/combined
```

#### 5.7.2 用 Praat / vLabeler 编辑

重点修：

- `sentences` tier 切句边界
- `phones` tier 对齐精度

#### 5.7.3 回对齐 words tier

```bash
python align_tg_words.py \
  --tg /path/to/combined \
  --dictionary /path/to/dictionary.txt \
  --overwrite
```

#### 5.7.4 切回短句

```bash
python slice_tg.py \
  --wavs /path/to/combined \
  --out /path/to/refined_segments
```

如果你走了人工精修这条线，后续 `build_dataset.py` 建议加 `--skip_silence_insertion`。

### 5.8 生成 DiffSinger acoustic 训练集

```bash
python build_dataset.py \
  --wavs /path/to/mfa_input \
  --tg /path/to/final_textgrids \
  --dataset /path/to/datasets/my_singer/raw
```

可选：

- `--skip_silence_insertion`：不随机前后插静音
- `--wav_subtype PCM_16|PCM_24|PCM_32|FLOAT|DOUBLE`

产物：

```text
/path/to/datasets/my_singer/raw/
  wavs/*.wav
  transcriptions.csv   # name, ph_seq, ph_dur
```

---

## 6. MakeDiffSinger 第 2 段：`variance-temp-solution`

这段是“把 acoustic 数据扩展成 variance 训练可用”。

### 6.1 安装

```bash
cd MakeDiffSinger/variance-temp-solution
pip install -r requirements.txt
```

### 6.2 如果你是旧 `transcriptions.txt`

```bash
python convert_txt.py /path/to/transcriptions.txt
```

会生成同目录 `transcriptions.csv`。

### 6.3 补 `ph_num`（训练 dur predictor 常用）

#### 方案 A：双拼型字典（中文常见）

```bash
python add_ph_num.py \
  /path/to/transcriptions.csv \
  --dictionary /path/to/dictionary.txt
```

#### 方案 B：自己给元音/辅音表

```bash
python add_ph_num.py \
  /path/to/transcriptions.csv \
  --vowels /path/to/vowels.txt \
  --consonants /path/to/consonants.txt
```

#### 方案 C：高级（结合 TextGrid）

```bash
python add_ph_num_advanced.py \
  /path/to/transcriptions.csv \
  --tg /path/to/textgrids \
  --vowels /path/to/vowels.txt \
  --consonants /path/to/consonants.txt
```

### 6.4 粗估音符序列

```bash
python estimate_midi.py \
  /path/to/transcriptions.csv \
  /path/to/wavs
```

可选 RMVPE：

```bash
python estimate_midi.py ... --pe rmvpe
```

RMVPE 模型路径（代码写死）：

```text
MakeDiffSinger/variance-temp-solution/assets/rmvpe/model.pt
```

### 6.5 转 DS，手工修 MIDI/连音，再回写 CSV

```bash
python convert_ds.py csv2ds /path/to/transcriptions.csv /path/to/wavs
# 用 SlurCutter 手工修 .ds
python convert_ds.py ds2csv /path/to/wavs /path/to/transcriptions.csv -f
```

说明：

- `csv2ds` 若 CSV 没 `note_slur`，会自动按词对齐生成。
- `ds2csv` 支持“只有 ds 没有 wav”的条目，名字会带 `#index`。

### 6.6 可选工具

```bash
python correct_cents.py csv /path/to/transcriptions.csv /path/to/wavs
python correct_cents.py ds /path/to/ds_dir
python eliminate_short.py /path/to/ds_dir 0.03
```

---

## 7. 接入 DiffSinger 配置（重点）

## 7.1 复制模板

```bash
cd DiffSinger
cp configs/templates/config_acoustic.yaml configs/my_acoustic.yaml
cp configs/templates/config_variance.yaml configs/my_variance.yaml
```

## 7.2 先改 acoustic 配置（最小可跑通）

下面给你一个“单说话人、单语言、先跑通”的示例核心段：

```yaml
# configs/my_acoustic.yaml
base_config:
  - configs/acoustic.yaml

dictionaries:
  zh: dictionaries/opencpop-extension.txt

datasets:
  - raw_data_dir: /abs/path/datasets/my_singer/raw
    speaker: my_singer
    spk_id: 0
    language: zh
    test_prefixes:
      - sample_0001
      - sample_0002

binary_data_dir: /abs/path/datasets/my_singer/binary_acoustic

# 建议首轮简化
use_lang_id: false
num_lang: 1
use_spk_id: false
num_spk: 1

use_energy_embed: false
use_breathiness_embed: false
use_voicing_embed: false
use_tension_embed: false

# 先避免 VR 额外文件坑
hnsep: world

# 按你实际下载位置改
vocoder: NsfHifiGAN
vocoder_ckpt: checkpoints/pc_nsf_hifigan_44.1k_hop512_128bin_2025.02/model.ckpt
```

### 7.3 再改 variance 配置（常用起步）

```yaml
# configs/my_variance.yaml
base_config:
  - configs/variance.yaml

dictionaries:
  zh: dictionaries/opencpop-extension.txt

datasets:
  - raw_data_dir: /abs/path/datasets/my_singer/raw
    speaker: my_singer
    spk_id: 0
    language: zh
    test_prefixes:
      - sample_0001
      - sample_0002

binary_data_dir: /abs/path/datasets/my_singer/binary_variance

use_lang_id: false
num_lang: 1
use_spk_id: false
num_spk: 1

predict_dur: true
predict_pitch: true
predict_energy: false
predict_breathiness: false
predict_voicing: false
predict_tension: false

hnsep: world
```

### 7.4 `test_prefixes` 必须命中（否则会报错）

DiffSinger 代码会断言：验证集不能为空。

- 报错典型：`Validation set is empty!`

### 7.5 重要坑：`select_test_set.py` 与新配置不兼容

`MakeDiffSinger/acoustic_forced_alignment/select_test_set.py` 当前读的是旧字段：

- `spk_ids`
- `speakers`
- `raw_data_dir`

但 DiffSinger 当前模板是 `datasets:` 列表结构。直接跑会 `KeyError: 'spk_ids'`。

#### 替代方案（推荐）

手工填 `datasets[*].test_prefixes`，或者用下面脚本自动抽样：

```bash
python - <<'PY'
import csv, random, pathlib, yaml
cfg = pathlib.Path('configs/my_acoustic.yaml')
hp = yaml.safe_load(cfg.read_text(encoding='utf-8'))
for ds in hp['datasets']:
    raw = pathlib.Path(ds['raw_data_dir'])
    with open(raw / 'transcriptions.csv', 'r', encoding='utf-8') as f:
        names = [r['name'] for r in csv.DictReader(f)]
    k = min(5, len(names))
    ds['test_prefixes'] = sorted(random.sample(names, k)) if k > 0 else []
cfg.write_text(yaml.safe_dump(hp, sort_keys=False, allow_unicode=True), encoding='utf-8')
print('updated:', cfg)
PY
```

---

## 8. DiffSinger 训练：标准命令顺序

### 8.1 acoustic 预处理

```bash
python scripts/binarize.py --config configs/my_acoustic.yaml
```

### 8.2 acoustic 训练

首次训练建议：

```bash
python scripts/train.py --config configs/my_acoustic.yaml --exp_name my_acoustic --reset
```

中断续训：

```bash
python scripts/train.py --config configs/my_acoustic.yaml --exp_name my_acoustic
```

原因：`set_hparams()` 默认会优先读取 `checkpoints/<exp>/config.yaml`。

- 你改了配置但不 `--reset`，改动可能不生效。

### 8.3 variance 预处理 + 训练

```bash
python scripts/binarize.py --config configs/my_variance.yaml
python scripts/train.py --config configs/my_variance.yaml --exp_name my_variance --reset
```

### 8.4 看训练曲线

```bash
tensorboard --logdir checkpoints/
```

---

## 9. 推理（先 variance 后 acoustic）

假设你有 `song.ds`。

### 9.1 variance 自动补参

```bash
python scripts/infer.py variance song.ds --exp my_variance
```

默认会输出：

```text
song_variance.ds
```

（当 `--out` 不改、`--title` 不给时，代码会自动加 `_variance` 后缀）

你也可以只预测指定参数：

```bash
python scripts/infer.py variance song.ds --exp my_variance --predict dur --predict pitch
```

可选标签：`dur`, `pitch`, `energy`, `breathiness`, `voicing`, `tension`。

### 9.2 acoustic 合成 wav

```bash
python scripts/infer.py acoustic song_variance.ds --exp my_acoustic
```

常用参数：

- `--ckpt 120000` 指定步数
- `--key 2` 移调
- `--gender -1~1`（模型启用了 `use_key_shift_embed` 时）
- `--steps N` 采样步数
- `--depth 0~1` 浅扩散深度
- `--spk ...` 多说话人混合

### 9.3 多说话人 `--spk` 写法

- 单人：`--spk spkA`
- 平均混合：`--spk spkA|spkB`
- 指定权重：`--spk spkA:0.7|spkB:0.3`

### 9.4 先导 mel 再单独 vocoder

```bash
python scripts/infer.py acoustic song_variance.ds --exp my_acoustic --mel
python scripts/vocode.py song_variance.mel.pt --exp my_acoustic
```

注意脚本名是 `vocode.py`，不是 `vocoder.py`。

### 9.5 术语澄清：“自动音高”到底属于哪一块？

社区里说的“自动音高”，在 DiffSinger 里通常指：

- `variance` 模型的 **pitch predictor** 根据音符/时值信息自动生成 `f0_seq`（更接近训练集风格的自然音高曲线）。

它不属于 acoustic 本体。acoustic 只负责“吃参数并合成声音”，其中就包括 `f0_seq`。

所以结论是：

1. 只训练 acoustic：不能自动生 `f0_seq`，你需要手工或外部工具提供音高曲线。
2. 训练了 variance 且 `predict_pitch: true`：可以在 `infer variance` 时自动补 `f0_seq`，再交给 acoustic 合成。

是否需要“额外训练步骤”：

- 不需要第三种新模型；**需要的只是把 variance 训练起来并开启 `predict_pitch`**。
- 如果你原本只训 acoustic，那就要新增一次 variance 训练（这是必要步骤，不是可选优化）。

对应数据要求（训练 variance 的 pitch 模块）：

- 至少要有 `name`, `ph_seq`, `ph_dur`, `note_seq`, `note_dur`。
- 若你还想自动时长（dur）也一起预测，则还需要 `ph_num` 并启用 `predict_dur: true`。

---

## 10. DS 字段怎么准备才不报错

### 10.1 acoustic 推理最小常见字段

- `ph_seq`
- `ph_dur`
- `f0_seq`
- `f0_timestep`

如果 acoustic 模型启用了某个 variance embed（如 `use_energy_embed: true`），还需要该参数和 timestep：

- `energy` + `energy_timestep`（其他同理）

### 10.2 variance 推理常见字段

推荐至少有：

- `ph_seq`
- `ph_num`
- `note_seq`
- `note_dur`
- `note_slur`

可选：

- `note_glide`（当配置启用 `use_glide_embed`）
- 手工 `ph_dur` / `f0_seq`（可与自动预测混用）

---

## 11. ONNX 导出（生产部署）

必须在 PyTorch 1.13.x 环境执行。

```bash
python scripts/export.py acoustic --exp my_acoustic
python scripts/export.py variance --exp my_variance
python scripts/export.py nsf-hifigan --config configs/my_acoustic.yaml --ckpt /path/to/vocoder.ckpt
```

---

## 12. 常见报错与定位

### 12.1 `Vocoder ckpt ... not found`

- `vocoder_ckpt` 路径写错或文件没放到位。

### 12.2 `Validation set is empty!`

- `test_prefixes` 没命中任何样本。

### 12.3 `The following phonemes are not covered in transcriptions`

- 字典音素覆盖不完整，补录或精简字典。

### 12.4 `Mel base must be set to 'e'`

- acoustic binarizer 对 `mel_base` 有断言，不要改成旧值。

### 12.5 `hnsep: vr` 相关错误

- 缺 `hnsep_ckpt` 指向模型，或同目录缺 `config.yaml`。
- 新手先用 `hnsep: world` 更省事。

### 12.6 `ph_num does not sum to number of phones`

- `ph_num` 与 `ph_seq` 不一致，回 CSV 修正。

### 12.7 `select_test_set.py` KeyError

- 原因是脚本仍按旧配置 schema；用第 7.5 节替代方案。

---

## 13. 一套最小可复现命令清单（复制即用）

### 13.0 数据制作快跑版（6 条命令）

```bash
cd MakeDiffSinger/acoustic_forced_alignment
python validate_labels.py --dir /data/segments --dictionary /data/dictionary.txt
python reformat_wavs.py --src /data/segments --dst /data/mfa_input
mfa align /data/mfa_input /data/dictionary.txt /data/mfa_model.zip /data/mfa_textgrids --beam 100 --clean --overwrite
python enhance_tg.py --wavs /data/mfa_input --dictionary /data/dictionary.txt --src /data/mfa_textgrids --dst /data/final_textgrids
python build_dataset.py --wavs /data/mfa_input --tg /data/final_textgrids --dataset /data/datasets/my_singer/raw
```

说明：这版故意省掉了长度校验、TextGrid 完整性检查和人工精修，目的是先跑通。后续再按第 5 节补齐质量流程。

### 13.1 数据制作（acoustic）

```bash
cd MakeDiffSinger/acoustic_forced_alignment
python validate_lengths.py --dir /data/segments
python validate_labels.py --dir /data/segments --dictionary /data/dictionary.txt
python reformat_wavs.py --src /data/segments --dst /data/mfa_input
mfa align /data/mfa_input /data/dictionary.txt /data/mfa_model.zip /data/mfa_textgrids --beam 100 --clean --overwrite
python check_tg.py --wavs /data/mfa_input --tg /data/mfa_textgrids
python enhance_tg.py --wavs /data/mfa_input --dictionary /data/dictionary.txt --src /data/mfa_textgrids --dst /data/final_textgrids
python build_dataset.py --wavs /data/mfa_input --tg /data/final_textgrids --dataset /data/datasets/my_singer/raw
```

### 13.2 扩展 variance 字段

```bash
cd MakeDiffSinger/variance-temp-solution
python add_ph_num.py /data/datasets/my_singer/raw/transcriptions.csv --dictionary /data/dictionary.txt
python estimate_midi.py /data/datasets/my_singer/raw/transcriptions.csv /data/datasets/my_singer/raw/wavs
python convert_ds.py csv2ds /data/datasets/my_singer/raw/transcriptions.csv /data/datasets/my_singer/raw/wavs
# 手工修 .ds
python convert_ds.py ds2csv /data/datasets/my_singer/raw/wavs /data/datasets/my_singer/raw/transcriptions.csv -f
```

### 13.3 训练与推理

```bash
cd DiffSinger
python scripts/binarize.py --config configs/my_variance.yaml
python scripts/binarize.py --config configs/my_acoustic.yaml
python scripts/train.py --config configs/my_variance.yaml --exp_name my_variance --reset
python scripts/train.py --config configs/my_acoustic.yaml --exp_name my_acoustic --reset
python scripts/infer.py variance /path/to/song.ds --exp my_variance
python scripts/infer.py acoustic /path/to/song_variance.ds --exp my_acoustic
```

### 13.4 仅 acoustic 出声（不训 variance）超短路线图

适用场景：你只想先让模型“出声”，暂时不做自动参数（自动音高/自动时长）。

#### 步骤 1：先按 13.0 做出 acoustic 数据集

最终至少要有：

```text
/data/datasets/my_singer/raw/wavs/*.wav
/data/datasets/my_singer/raw/transcriptions.csv   # 含 name, ph_seq, ph_dur
```

#### 步骤 2：配置并训练 acoustic

```bash
cd DiffSinger
cp configs/templates/config_acoustic.yaml configs/my_acoustic.yaml
```

把 `configs/my_acoustic.yaml` 关键项改成你的真实路径（最小建议）：

- `datasets[*].raw_data_dir`
- `datasets[*].test_prefixes`（至少命中 1 条）
- `binary_data_dir`
- `vocoder_ckpt`
- `hnsep: world`（先简化）

然后训练：

```bash
python scripts/binarize.py --config configs/my_acoustic.yaml
python scripts/train.py --config configs/my_acoustic.yaml --exp_name my_acoustic --reset
```

#### 步骤 3：准备一个“带手工音高”的 DS 文件

因为你没训练 variance，这个 DS 必须自己带：

- `ph_seq`
- `ph_dur`
- `f0_seq`
- `f0_timestep`

可选来源：

1. 直接用你已有的编辑器工程导出的 `.ds`（最推荐）。
2. 临时用 `MakeDiffSinger/variance-temp-solution/convert_ds.py csv2ds` 生成一个带 `f0_seq` 的 DS 做测试。

#### 步骤 4：直接 acoustic 推理

```bash
python scripts/infer.py acoustic /path/to/song_manual_f0.ds --exp my_acoustic
```

这条路线的限制：

- 能出声，但没有“自动音高”（不会自动生成 `f0_seq`）。
- 你后续如果要自动化参数，再补训 variance 即可（不需要重训第三种模型）。

### 13.5 acoustic 首次可听样本故障排查（按报错走）

目标：你在“13.4 路线”里，尽快拿到第一条可听 wav。  
建议按下面顺序排查，避免来回试错。

#### 13.5.1 先做三条快速自检

1. `checkpoints/my_acoustic/` 下是否已有 `model_ckpt_steps_*.ckpt`。
2. `configs/my_acoustic.yaml` 里 `vocoder_ckpt` 文件是否真实存在。
3. 推理 DS 是否至少包含：`ph_seq`, `ph_dur`, `f0_seq`, `f0_timestep`。

#### 13.5.2 常见报错 -> 直接处理动作

1. 报错：`Vocoder ckpt '...' not found`
处理：
- 改 `vocoder_ckpt` 为真实路径。
- 或先用 `--mel` 验证 acoustic 主体可跑，再单独用 `vocode.py`。

2. 报错：`There are no matching exp starting with ...`
处理：
- 检查 `checkpoints/` 下实验目录名。
- `--exp` 写完整目录名，不要只写模糊前缀。

3. 报错：`Validation set is empty!`
处理：
- 给每个 `datasets[*].test_prefixes` 填至少 1 条可命中的样本名。

4. 报错：`The following phonemes are not covered in transcriptions`
处理：
- 补数据覆盖缺失音素，或精简字典中无用音素。

5. 报错：`Mel base must be set to 'e'`
处理：
- 不要改 `mel_base`，保持 `'e'`。

6. 预处理日志：`Skipped 'xxx': empty gt f0`
处理：
- 该条样本会被丢弃；先检查音频是否接近纯清音/静音。
- 试调 `f0_min/f0_max` 或更换 `pe`（例如 `parselmouth`/`rmvpe`）。

7. 报错（多语言）：`Please specify a language by --lang option`
处理：
- 推理命令加 `--lang`，或在 DS 里给每段加 `lang`。

8. 报错（多说话人）：`Please specify a speaker or speaker mix by --spk option`
处理：
- 推理命令加 `--spk spk_name`，或在 DS 中提供 `spk_mix`。

9. `hnsep: vr` 相关报错（预处理阶段）
处理：
- 新手先设 `hnsep: world`，跑通后再回到 `vr`。

10. 现象：改了 yaml 但训练表现/日志没变化
处理：
- 首次或强制覆盖时用 `--reset`：
```bash
python scripts/train.py --config configs/my_acoustic.yaml --exp_name my_acoustic --reset
```

#### 13.5.3 没报错但声音很怪（可出声，质量差）

1. 先确认“能稳定出声”再谈质量，避免一次调太多参数。
2. 保证数据和模型采样率体系一致（本仓库默认 44.1k / hop 512）。
3. 核查 `ph_dur` 与实际发音是否离谱（对齐坏会直接影响节奏和咬字）。
4. 声码器不匹配会导致“糊/炸/金属感”，优先核对 `vocoder_ckpt`。
5. 训练步数过少时音色不稳属正常，先观察更多 steps 再判断。

---

## 14. 你现在应该怎么做（实战建议）

按这个顺序最稳：

1. 先只做单说话人 + 中文 + acoustic（不启用各类 variance embed），确保能稳定出声。
2. 再做 variance（先 `predict_dur + predict_pitch`）。
3. 最后再尝试 energy / breathiness / voicing / tension 等控制维度。

这样排错成本最低，成功率最高。

---

## 15. 官方入口（建议收藏）

- DiffSinger：<https://github.com/openvpi/DiffSinger>
- DiffSinger Getting Started：<https://github.com/openvpi/DiffSinger/blob/main/docs/GettingStarted.md>
- DiffSinger Best Practices：<https://github.com/openvpi/DiffSinger/blob/main/docs/BestPractices.md>
- DiffSinger Configuration Schemas：<https://github.com/openvpi/DiffSinger/blob/main/docs/ConfigurationSchemas.md>
- MakeDiffSinger：<https://github.com/openvpi/MakeDiffSinger>
- MakeDiffSinger acoustic_forced_alignment：<https://github.com/openvpi/MakeDiffSinger/tree/main/acoustic_forced_alignment>
- MakeDiffSinger variance-temp-solution：<https://github.com/openvpi/MakeDiffSinger/tree/main/variance-temp-solution>
- dataset-tools（AudioSlicer / MinLabel / SlurCutter）：<https://github.com/openvpi/dataset-tools>
