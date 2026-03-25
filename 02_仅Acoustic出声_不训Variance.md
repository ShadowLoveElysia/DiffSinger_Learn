# 仅 Acoustic 出声（不训 Variance）

返回导航：

- [README.md](./README.md)

适用场景：

- 你只想先听到模型声音。
- 暂时不做自动音高/自动时长。

## 步骤 1：先有 acoustic 数据

先完成：

- [01_快速数据制作_30分钟.md](./01_快速数据制作_30分钟.md)

至少需要：

```text
/data/datasets/my_singer/raw/wavs/*.wav
/data/datasets/my_singer/raw/transcriptions.csv   # name, ph_seq, ph_dur
```

## 步骤 2：配置并训练 acoustic

```bash
cd DiffSinger
cp configs/templates/config_acoustic.yaml configs/my_acoustic.yaml
```

`configs/my_acoustic.yaml` 最少改这些：

- `datasets[*].raw_data_dir`
- `datasets[*].test_prefixes`（至少命中1条）
- `binary_data_dir`
- `vocoder_ckpt`
- `hnsep: world`（先降低复杂度）

训练：

```bash
python scripts/binarize.py --config configs/my_acoustic.yaml
python scripts/train.py --config configs/my_acoustic.yaml --exp_name my_acoustic --reset
```

## 步骤 3：准备 DS（必须带手工音高）

因为你没训练 variance，推理 DS 必须包含：

- `ph_seq`
- `ph_dur`
- `f0_seq`
- `f0_timestep`

来源建议：

1. 直接用编辑器导出的 `.ds`。
2. 临时用 `MakeDiffSinger/variance-temp-solution/convert_ds.py csv2ds` 生成测试 DS。

## 步骤 4：推理出声

```bash
python scripts/infer.py acoustic /path/to/song_manual_f0.ds --exp my_acoustic
```

限制：

- 可以出声，但没有自动音高（不会自动生成 `f0_seq`）。

如果你要自动音高：

- 看 [05_自动音高说明.md](./05_自动音高说明.md)
- 再走 [03_标准全流程入口.md](./03_标准全流程入口.md)
