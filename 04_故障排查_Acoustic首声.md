# 故障排查（Acoustic 首声）

返回导航：

- [README.md](./README.md)

目标：

- 在“仅 acoustic 出声”路线下尽快拿到第一条可听 wav。

先做三条自检：

1. `checkpoints/my_acoustic/` 是否有 `model_ckpt_steps_*.ckpt`。
2. `configs/my_acoustic.yaml` 的 `vocoder_ckpt` 文件是否存在。
3. 推理 DS 是否有 `ph_seq ph_dur f0_seq f0_timestep`。

常见报错 -> 动作：

1. `Vocoder ckpt '...' not found`
- 校正 `vocoder_ckpt`。
- 或先用 `--mel` 跑 acoustic，再单独 `vocode.py`。

2. `There are no matching exp starting with ...`
- 检查 `checkpoints/` 下实验目录。
- `--exp` 尽量写完整名。

3. `Validation set is empty!`
- `datasets[*].test_prefixes` 至少填 1 条命中样本。

4. `The following phonemes are not covered in transcriptions`
- 补数据覆盖缺失音素，或精简字典。

5. `Mel base must be set to 'e'`
- 保持 `mel_base: 'e'`。

6. `Skipped 'xxx': empty gt f0`
- 该样本会被丢弃。
- 检查音频是否接近静音，必要时调 `f0_min/f0_max` 或换 `pe`。

7. `Please specify a language by --lang option`
- 多语言模型推理时补 `--lang` 或在 DS 里加 `lang`。

8. `Please specify a speaker or speaker mix by --spk option`
- 多说话人模型推理时补 `--spk` 或 DS 里给 `spk_mix`。

9. `hnsep: vr` 报错
- 先改 `hnsep: world` 跑通，再回到 `vr`。

10. 改了 yaml 训练没变化
- 重新训练时加 `--reset` 强制生效。

没报错但声音怪：

1. 先确保“稳定出声”，再调质量。
2. 核查采样率体系（默认 44.1k / hop 512）。
3. 核查 `ph_dur` 对齐质量。
4. 核查 `vocoder_ckpt` 是否匹配。
5. 训练步数太少时音色不稳是常见现象。

相关路线：

- [02_仅Acoustic出声_不训Variance.md](./02_仅Acoustic出声_不训Variance.md)
- [05_自动音高说明.md](./05_自动音高说明.md)
