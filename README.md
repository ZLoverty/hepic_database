# hepic_database

材料属性数据库，独立于 [HEPiC](https://github.com/ZLoverty/HEPiC)（桌面应用）和 hepic_device（树莓派设备后端）两个代码仓库维护。

## 目的

材料参数（PLA/PETG 各牌号的力值范围、温度、速度等）会比 app 代码更新得频繁。把数据独立成仓库后，
**更新材料库不需要重新发布/重装 app**：两个消费端（HEPiC 桌面应用、hepic_device 设备后端）各自在启动时
读取这里的数据快照，只要这个仓库发布了新版本，它们下次启动/同步时就能拿到最新数据。

## 目录结构

```
manifest.json          数据版本号
families/
  pla_family.yaml
  petg_family.yaml
  asa_family.yaml
```

`families/*.yaml` 的格式与 HEPiC 里 `HEPiC/database/material_database.py` 期望的完全一致
（YAML anchor 模板 + 每个牌号一个 mapping，需含 `PI_Code` 字段），可以直接被两个消费端的
`MaterialDatabase` 解析。

## 发布新版本的流程

1. 编辑/新增 `families/*.yaml`。
2. 更新 `manifest.json` 的 `version`（建议用日期，如 `2026.08.01`，同一天多次发布可加后缀 `.1`/`.2`）。
3. 提交，打 tag（**必须和 manifest.json 里的 `version` 一致，加 `v` 前缀**，如 `v2026.08.01`）并推送。
4. `.github/workflows/release.yml` 会自动：
   - 校验 tag 版本号与 `manifest.json` 的 `version` 是否一致；
   - 把 `manifest.json` + `families/` 打包成 `materials.zip`，附加到对应的 GitHub Release 上。
5. 两个消费端仓库（HEPiC、hepic_device）的代码都不需要改动。

也可以在 Actions 页面用 `workflow_dispatch` 手动触发（输入同样格式的 tag）。

下载完整性由 GitHub 自己保证：Releases API 会给每个 asset 返回 `digest`（sha256），消费端
下载 `materials.zip` 后直接跟这个值比对，不需要我们在 `manifest.json` 里再手动维护一份逐文件
checksum。

## 消费端

HEPiC（`HEPiC/database/materials_sync.py`）与 hepic_device 都已接入：应用/服务启动时请求本仓库
最新 Release，对比 `manifest.json` 的 `version` 与本地缓存版本，不一致就下载 `materials.zip`
（校验 GitHub 返回的 digest 后）覆盖本地缓存目录。网络异常时静默跳过，用本地缓存或内置快照兜底。
