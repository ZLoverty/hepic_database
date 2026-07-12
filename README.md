# hepic_database

材料属性数据库，独立于 [HEPiC](https://github.com/ZLoverty/HEPiC)（桌面应用）和 hepic_device（树莓派设备后端）两个代码仓库维护。

## 目的

材料参数（PLA/PETG 各牌号的力值范围、温度、速度等）会比 app 代码更新得频繁。把数据独立成仓库后，
**更新材料库不需要重新发布/重装 app**：两个消费端（HEPiC 桌面应用、hepic_device 设备后端）各自在启动时
读取这里的数据快照，只要这个仓库发布了新版本，它们下次启动/同步时就能拿到最新数据。

## 目录结构

```
manifest.json          数据版本号 + 各 family 文件的 sha256 校验和
families/
  pla_family.yaml
  petg_family.yaml
```

`families/*.yaml` 的格式与 HEPiC 里 `HEPiC/database/material_database.py` 期望的完全一致
（YAML anchor 模板 + 每个牌号一个 mapping，需含 `PI_Code` 字段），可以直接被两个消费端的
`MaterialDatabase` 解析。

## 发布新版本的流程

1. 编辑/新增 `families/*.yaml`。
2. 更新 `manifest.json`：
   - 重新计算改动文件的 sha256（`shasum -a 256 families/xxx.yaml`）。
   - 把 `version` 换成新的版本号（建议用日期，如 `2026.08.01`，同一天多次发布可加后缀 `.1`/`.2`）。
3. 提交，打 tag（**必须和 manifest.json 里的 `version` 一致，加 `v` 前缀**，如 `v2026.08.01`）并推送。
4. `.github/workflows/release.yml` 会自动：
   - 校验 tag 版本号与 `manifest.json` 的 `version` 是否一致；
   - 校验 `families/*.yaml` 的 sha256 是否和 `manifest.json` 里登记的一致（防止改了文件忘记更新 checksum）；
   - 把 `manifest.json` + `families/` 打包成 `materials.zip`，附加到对应的 GitHub Release 上。
5. 两个消费端仓库（HEPiC、hepic_device）的代码都不需要改动。

也可以在 Actions 页面用 `workflow_dispatch` 手动触发（输入同样格式的 tag）。

## 消费端（待接入）

目前 HEPiC / hepic_device 里的 `MaterialDatabase` 还是直接读取 HEPiC 包内置的
`HEPiC/database/materials/` 快照（作为离线兜底数据）。后续会在 HEPiC 里加一个
`materials_sync.py`，在应用/服务启动时请求本仓库最新 Release 的 `materials.zip`，对比其中
`manifest.json` 的 `version` 与本地缓存的 `version`，不一致就下载覆盖本地缓存目录 —— 到那时
这个仓库就会成为两端的唯一数据源头。
