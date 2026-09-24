# ha-pitfalls

Home Assistant 踩坑记录与解决方案合集。每篇一个独立问题，统一按「**现象 → 原因 → 解决 → 验证**」写，尽量给出可以自己复现的验证命令，而不是「我改好了就行」。

实测环境：**Home Assistant OS 2026.9** / HA Core `2026.9.3` / Supervisor `2026.09.2`（amd64）。

## 索引

| 文档 | 一句话 |
|---|---|
| [JMGLink HA 3D Home 安装避坑](docs/01-JMGLink-HA-3D-Home-安装避坑.md) | 加载项商店加不上仓库（Gitee 禁止匿名 git clone）、装完侧边栏没有入口（`ingress_panel` 开关默认关） |

## 说明

- 这里记录的都是**实测复现 + 定位到根因**的问题，包含报错原文与判定命令，方便搜索引擎命中和对照。
- 与「Home Assistant 官方文档的用法」不同，本仓库只写**官方文档没写、但实际会卡住你的地方**。
