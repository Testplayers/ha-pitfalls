# JMGLink HA 3D Home 安装避坑

[JMGLink HA 3D Home](https://github.com/freedomTT/JMGLink-ha-3d-home) 是一个跑在 Home Assistant 里的 **3D 户型图可视化控制面板**：以 3D 形式展示房间/设备，在 3D 场景里直接控制灯、窗帘、风扇、门窗（可绑传感器）。

它**不是 HACS 集成，而是 HA 加载项（Add-on）**，所以只能按加载项的方式装。

官方给的仓库地址是 Gitee：`https://gitee.com/tuotuo4ever/jmglink-ha-3d-home.git`。
照着官方文档装，会连续踩两个坑 —— **装不上**、**装上了但侧边栏没入口**。下面是实测复现与解决办法。

> 实测环境：Home Assistant OS 2026.9（HA Core `2026.9.3` / Supervisor `2026.09.2`，amd64），加载项版本 `0.1.6`。

---

## 坑 1：加载项商店添加仓库失败（Gitee 禁止匿名 clone）

### 现象

设置 → 加载项 → 加载项商店 → 右上角「仓库」→ 填官方那个 Gitee 地址，保存后报错：

```
Cmd('git') failed due to: exit code(128)
  cmdline: git clone -v --recursive --depth=1 --shallow-submodules --
           https://gitee.com/tuotuo4ever/jmglink-ha-3d-home.git /data/apps/git/18f904e9
  stderr: 'Cloning into '/data/apps/git/18f904e9'...'
fatal: could not read Username for 'https://gitee.com': No such device or address
```

关键是最后一行：**`could not read Username`** —— 服务端要认证，而 Supervisor 的 `git clone` 是非交互的，没法输入账号密码，直接失败。

### 为什么容易误判

这个仓库其实是**公开**的：

- 浏览器打开 `https://gitee.com/tuotuo4ever/jmglink-ha-3d-home` → **200**，能看
- Gitee API `https://gitee.com/api/v5/repos/tuotuo4ever/jmglink-ha-3d-home` → 正常返回，且 `private: false`、`public: true`

但**匿名 git 协议**拿不到：

```bash
curl -s -o /dev/null -w '%{http_code}\n' \
  'https://gitee.com/tuotuo4ever/jmglink-ha-3d-home/info/refs?service=git-upload-pack'
# 401   ← 页面能看、API 能读，但 git clone 匿名被拒
```

所以「网页打得开」不能证明「Supervisor 能 clone」。

### 解决办法

**把仓库地址换成 GitHub 上的同源仓库**，再添加一次：

```
https://github.com/freedomTT/JMGLink-ha-3d-home
```

HA 侧添加前建议先确认 git 协议通（GitHub 网页偶尔慢，但 git 协议一般正常）：

```bash
curl -s -o /dev/null -w '%{http_code} %{time_total}s\n' \
  'https://github.com/freedomTT/JMGLink-ha-3d-home/info/refs?service=git-upload-pack'
# 期望 200，几百毫秒
```

添加成功后，商店里会出现加载项，slug 形如：

```
66093878_jmglink_ha_3d_home
```

> **注意**：前面的 `66093878` 是**仓库哈希前缀**，每位用户、每个仓库都不一样，不要照抄。

**命令行方式（可选）**：在 HA 宿主机上用 Supervisor token 添加仓库，再安装：

```bash
TOKEN=<宿主机环境变量里的 SUPERVISOR_TOKEN>

# 1) 添加仓库
curl -s -X POST http://supervisor/store/repositories \
  -H "Authorization: Bearer $TOKEN" -H 'Content-Type: application/json' \
  -d '{"repository":"https://github.com/freedomTT/JMGLink-ha-3d-home"}'

# 2) 安装（首次会拉镜像，约 1 分钟）——建议后台跑，别让连接超时
curl -s -X POST http://supervisor/addons/66093878_jmglink_ha_3d_home/install \
  -H "Authorization: Bearer $TOKEN" -H 'Content-Type: application/json' -d '{}'

# 3) 启动
curl -s -X POST http://supervisor/addons/66093878_jmglink_ha_3d_home/start \
  -H "Authorization: Bearer $TOKEN" -H 'Content-Type: application/json' -d '{}'
```

---

## 坑 2：装完侧边栏没有「3D Home」入口

### 现象

加载项装好、也**启动了**，一切看起来正常：

- 容器状态 `Up ... (healthy)`，`/api/health` 返回 200
- 加载项日志显示已正确运行在加载项模式：

```json
{"runtime":{"mode":"addon","port":8099,"hasHaToken":true,"hasSupervisorToken":true},
 "msg":"Runtime configuration loaded"}
{"ha":{"connected":true,"mode":"addon","mock":false,"configured":true},
 "msg":"Home Assistant connection status"}
```

但**左侧边栏就是没有「3D Home」**，刷新、重登都没用。

查加载项信息，会发现：

```
GET /addons/66093878_jmglink_ha_3d_home/info
→ ingress: true, ingress_url: /api/hassio_ingress/xxxx/ , ingress_panel: false
```

### 原因：`ingress_panel` 是一个「持久化开关」，默认关闭

这是最反直觉的一点：

新版本 Supervisor 里，**`ingress_panel` 并不是根据 `config.yaml` 的 `panel_icon` / `panel_title` 推导出来的**，而是一个独立持久化的布尔值：

```python
# supervisor/apps/app.py
@property
def ingress_panel(self) -> bool | None:
    if not self.with_ingress:
        return None
    return self.persist[ATTR_INGRESS_PANEL]      # ← 持久化开关，安装后默认 False
```

也就是说，加载项 `config.yaml` 里写的：

```yaml
ingress: true
ingress_port: 8099
panel_icon: mdi:floor-plan
panel_title: 3D Home
panel_admin: false
```

**只提供了面板的标题和图标，并不会自动把面板挂到侧边栏**。「要不要显示在侧边栏」是一个需要手动打开的开关 —— 在 UI 上它就是加载项页面里的「**在侧边栏显示**」。

### 解决办法

#### 方法 A：UI（推荐）

设置 → 加载项 → **JMGLink HA 3D Home** → 打开「**在侧边栏显示**」→ 浏览器硬刷新（`Ctrl+Shift+R`）。

#### 方法 B：API / 命令行

```bash
TOKEN=<SUPERVISOR_TOKEN>
SLUG=66093878_jmglink_ha_3d_home

curl -s -X POST "http://supervisor/addons/$SLUG/options" \
  -H "Authorization: Bearer $TOKEN" -H 'Content-Type: application/json' \
  -d '{"ingress_panel": true}'
```

这个接口就是 UI 那个开关背后的实现，schema 里是 `vol.Optional(ATTR_INGRESS_PANEL): vol.Boolean()`（**无默认值**），所以**只传这一个键不会顺手改掉 `log_level` / `external_access` 等选项**：

```python
# supervisor/api/apps.py
if ATTR_INGRESS_PANEL in body:
    app.ingress_panel = body[ATTR_INGRESS_PANEL]
    await self.sys_ingress.update_hass_panel(app)   # 立刻推给 HA Core 注册面板
```

### 背后的完整链路（有助于举一反三）

1. 开关打开后，Supervisor 立即 `POST` 到 HA Core 的 `/api/hassio_push/panel/{addon}`
2. Core 的 `components/hassio/addon_panel.py` 调 `frontend.async_register_built_in_panel(...)` 注册内置面板
3. 关键：**面板的 URL 路径就是加载项的 slug**

```python
frontend.async_register_built_in_panel(
    hass, "app",
    frontend_url_path=addon,            # ← 就是 slug
    sidebar_title=data.title,           # ← config.yaml 的 panel_title
    sidebar_icon=data.icon,             # ← config.yaml 的 panel_icon
    require_admin=data.admin,           # ← config.yaml 的 panel_admin
    config={"addon": addon}, update=True,
)
```

所以本次安装后，面板地址是：

```
http://<你的HA地址>/66093878_jmglink_ha_3d_home
```

（不是 `/3d-home` 之类的友好名字，因为 slug 带仓库哈希前缀 —— 建议加个书签。）

### 验证三连

不想只靠"看起来好了"的话，用这三步确认：

```bash
# 1) Supervisor 日志：只有 Core 返回 200/201 才会打印这一行
docker logs hassio_supervisor --since 10m 2>&1 | grep -i 'ingress'
# INFO (MainThread) [supervisor.ingress] Update Ingress as panel for 66093878_jmglink_ha_3d_home

# 2) 开关状态已置为 true，且其他 options 没被改动
curl -s http://supervisor/addons/66093878_jmglink_ha_3d_home/info \
  -H "Authorization: Bearer $TOKEN"
# ingress_panel: true ; options: {"log_level":"info","external_access":false}

# 3) 面板路由真的注册了（对照一个不存在的路径）
curl -s -o /dev/null -w 'panel=%{http_code}\n' http://127.0.0.1/66093878_jmglink_ha_3d_home
curl -s -o /dev/null -w 'bogus=%{http_code}\n' http://127.0.0.1/definitely-not-a-panel-xyz
# panel=200  bogus=404   ← 404 是有效对照，说明这个 200 不是"页面外壳恒 200"
```

最后浏览器硬刷新一次，侧边栏就会出现「3D Home」。

---

## 补充说明

- **镜像来源是作者的私有 registry**：`docker.jmglink.cn/jmglink-ha-3d:0.1.6`（约 273 MB，OCI 索引含 `amd64` / `arm64`）。安装时由 Supervisor 匿名拉取，实测可直连。
- **项目很早期**（`0.1.x`、无 LICENSE、个人项目），适合折腾党；生产环境请自行评估。
- **数据与素材目录**：项目数据在加载项自己的 `/data`，模型素材目录是 `/media/ha-3d`（加载项配置里映射了 `media`）。**大幅改户型前先导出项目 JSON 备份。**
- **加载项配置项**只有两个：`log_level`（默认 `info`，排错时改 `debug`）、`external_access`（默认 `false`；正常走 Ingress 面板不用开）。
- 后续升级：仓库已加进加载项商店，作者发新版后，在「设置 → 加载项」里直接点更新即可（无需重新添加仓库）。

## 参考

- 上游仓库：<https://github.com/freedomTT/JMGLink-ha-3d-home>（官方文档给的 Gitee 镜像：<https://gitee.com/tuotuo4ever/jmglink-ha-3d-home>）
- HA 加载项配置项说明：<https://developers.home-assistant.io/docs/add-ons/configuration>
