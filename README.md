# dvServer

大坝网格 / 属性可视化后端。提供网格上传（attrstore / damb）、forecast（damf）导出、测点（dams）导出、DXF 剖面叠图。

## 配置文件

启动时通过 `-config` 指定 JSON 配置（默认 `config.json`）：

```bash
go run . -config config.json
# 达梦示例：
go run . -config "config(dm).json"
```

仓库内两份示例：

| 文件 | 用途 |
|------|------|
| [`config.json`](./config.json) | MySQL |
| [`config(dm).json`](./config(dm).json) | 达梦（DM） |

数据目录（与 JSON 配置无关）。默认都在 **dvServer 可执行文件所在目录** 下创建 / 读取；`go run` 时回退到当前工作目录。可用环境变量覆盖某一项。

[`start.sh`](./start.sh) 与二进制放在同一目录：脚本以自身所在目录为工作目录启动，不依赖 `/data`。

| 用途 | 默认路径 | 环境变量覆盖 |
|------|----------|--------------|
| 临时文件 | `{程序目录}/tmp` | `TEMP_DIR` |
| damb | `{程序目录}/damb` | `DAMB_DIR` |
| 物理量 / 网格 delta / catalog | `{程序目录}/attrstore` | `ATTR_DIR` |
| forecast（damf） | `{程序目录}/forecast` | `FORECAST_DIR` |
| 测点（dams） | `{程序目录}/dams` | `DAMS_DIR` |
| 地形 | `{程序目录}/terrain` | `TERRAIN_DIR` |
| 前端页面 | `{程序目录}/panel`，或与程序目录并列的 `panel` | `UI_DIR` |

前端：把含 `index.html` 的 `panel` 目录放在程序目录下，或与 `dvServer` 目录并列。启动后浏览器打开 `http://<host>:<httpPort>/`。没有 `index.html` 时只提供 `/damviewer` API，不报错退出。

读写代码在包 [`attrfile/`](./attrfile/)、[`meshfile/`](./meshfile/)。

旧月表一次性迁到文件：`go run . -config config.json -migrate-attrs 工程名`（`*` 表示全部 mesh），完成后退出。

### 字段说明

#### 通用

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `httpPort` | int | 否 | HTTP 监听端口，默认 `3000`，实际监听 `:{httpPort}` |
| `host` | string | 否 | 本机连库主机；拼进 `sqlLinks` 里的 `%s` |
| `dockerHost` | string | 否 | Docker 内连库主机；`InitEngine(..., inDocker=true)` 时用它替换 `%s` |
| `dbType` | string | 建议填 | `mysql` 或 `dm`；不填则根据 `sqlLinks` 内容推断 |
| `sqlLinks` | object | **是** | 四条连接串，见下表 |
| `envelope` | string[] | 否 | 启动后（及 upload 刷新后）自动算包络的 mesh 名；`[]` / 默认 = 不自动算；含 `"*"` = 全部 |
| `fullCacheMeshes` | string[] | 否 | 启动时全量常驻缓存的 mesh；`[]` / 默认 = 全部走按需零散缓存；含 `"*"` = registry 内全部全量预载 |
| `maxScatteredMemoryMB` | int | 否 | 非全量 mesh 的零散缓存内存上限（MB）；`<=0` = 不限制 |

`POST /damviewer/envelope` 始终可手动刷新包络，不依赖 `envelope` 配置。

#### `sqlLinks`（必填四键）

| 键 | 用途 |
|----|------|
| `Root` | 连实例（建库 / 管理） |
| `Main` | 主库 `dam_main`（或 DM schema `DAM_MAIN`） |
| `SubOpen` | 打开已有子库；第二个 `%s` 为库名 / schema |
| `SubCreate` | 创建子库用的连接 |

键名大小写不敏感（会规范成 `Root` / `Main` / `SubOpen` / `SubCreate`）。MySQL 连接串通常带一个 `%s`（host）；`SubOpen` 再带第二个 `%s`（库名）。

#### 达梦专用（`dbType: "dm"`）

| 字段 | 说明 |
|------|------|
| `dmUser` | 达梦用户名 |
| `dmPassword` | 达梦密码（勿做 URL encode，驱动按最后一个 `@` 切分） |
| `dmPort` | 达梦端口，默认 `5236` |
| `dmBulkPath` | 批量导入路径；空字符串可用 |

### MySQL 示例（`config.json`）

```json
{
  "httpPort": 3000,
  "host": "127.0.0.1",
  "dockerHost": "iwhr.leve.vip",
  "dbType": "mysql",
  "envelope": [],
  "fullCacheMeshes": ["normal", "pre_jingling"],
  "maxScatteredMemoryMB": 10240,
  "sqlLinks": {
    "Root": "root:123456@tcp(%s:3306)/?charset=utf8mb4&parseTime=True&loc=Local",
    "Main": "root:123456@tcp(%s:3306)/dam_main?charset=utf8mb4&parseTime=True&loc=Local&allowAllFiles=true",
    "SubOpen": "root:123456@tcp(%s:3306)/%s?charset=utf8mb4&parseTime=True&loc=Local&allowAllFiles=true",
    "SubCreate": "root:123456@tcp(%s:3306)/?charset=utf8mb4&parseTime=True&loc=Local"
  }
}
```

### 达梦示例（`config(dm).json`）

```json
{
  "httpPort": 3000,
  "host": "iwhr.leve.vip",
  "dockerHost": "iwhr.leve.vip",
  "dbType": "dm",
  "dmUser": "SYSDBA",
  "dmPassword": "Dam123456897",
  "dmPort": 5237,
  "dmBulkPath": "",
  "envelope": [],
  "fullCacheMeshes": [],
  "maxScatteredMemoryMB": 10240,
  "sqlLinks": {
    "Root": "dm://%s:5237",
    "Main": "dm://%s:5237?schema=DAM_MAIN",
    "SubOpen": "dm://%s:5237?schema=%s",
    "SubCreate": "dm://%s:5237"
  }
}
```

预加载相关字段（`envelope` / `fullCacheMeshes` / `maxScatteredMemoryMB`）与 MySQL 语义相同；达梦示例里默认不自动包络、不全量预载，按需改成具体 mesh 名即可。

### 缓存相关配置怎么选

| 场景 | 建议 |
|------|------|
| 常用 mesh、内存够 | `fullCacheMeshes` 写上 mesh 名，启动全量常驻 |
| 全部 mesh 都常驻 | `fullCacheMeshes: ["*"]` |
| 省内存、按需加载 | `fullCacheMeshes: []`，用 `maxScatteredMemoryMB` 卡住零散缓存 |
| 上传后自动刷包络 | 把 mesh 写入 `envelope`（或 `"*"`） |

---

## HTTP 接口一览

| 方法 | 路径 | 说明 |
|------|------|------|
| `GET` | `/` 或 `/damviewer/` | 前端页面（`panel/index.html`；后者兼容 Nginx 把 `/tDamviewer/` 反代到 `/damviewer/`） |
| `GET` | `/damviewer/ws` | WebSocket（网格拓扑 / 属性 / 切片 / 应力链等） |
| `POST` | `/damviewer/upload` | 上传网格 / 测点（异步处理） |
| `POST` | `/damviewer/remove` | 删除指定 version 及之后的入库数据 |
| `GET` | `/damviewer/meshes` | 工程列表（主库 `mesh_registry`） |
| `GET` | `/damviewer/meshVersions?mesh=` | 查询某 mesh 的全部 version 记录 |
| `GET` | `/damviewer/attrVersions?mesh=` | 物理量导入目录（`attr_import_log`） |
| `GET` | `/damviewer/attrOnePoint` | 单点属性时间序列 |
| `POST` | `/damviewer/envelope?mesh=` | 全量重算包络并覆盖缓存 |
| `GET` | `/damviewer/envelope?mesh=` | 包络摘要；可带 `attrName` / `refresh` |
| `POST` | `/damviewer/dxf-overlay` | DXF 剖面叠图，直接返回 SVG |
| `GET` | `/damviewer/damb` | 列出可用 damb 名（不含 `.damb`，排除 `integral`） |
| `GET` | `/damviewer/damb/:name` | 下载 `./damb/{name}.damb` |
| `GET` | `/damviewer/attrFile` | 下载一场物理量 `.f32`（`mesh` / `meshVersion` / `version` / `name` / `step`） |
| `GET` | `/damviewer/verticesFile?mesh=` | 普通网格顶点 float32 LE（与 WS `GET_VERTICES` 同内容；无则 404，前端回退 WS） |
| `GET` | `/damviewer/topoFile?mesh=&tag=` | 普通网格拓扑：JSON layout + `0x1F0x1E` + uint32 LE（`tag` = surfaces/faces/grids/edges/lines/points） |
| `GET` | `/damviewer/forecast/:name` | 下载 `{name}.damf`（gzip） |
| `GET` | `/damviewer/forecast/:name/attr/:attr` | 下载属性文件 `.damfa` |
| `GET` | `/damviewer/dams/:name` | 下载 `{name}.dams`（gzip） |
| `GET` | `/damviewer/terrain/:name` | 下载 `{name}.damt`（gzip，地形 TIN） |

地形默认目录见上文 `{程序目录}/terrain`（`TERRAIN_DIR` 可覆盖）。

从 DWG 生成：`node tools/terrain-from-dwg/index.mjs <file.dwg> dvServer/terrain/<id>.damt`

已开启 CORS（开发期允许任意 Origin）。

### 常用查询参数

**meshes**

- 无查询参数；返回 `{ "data": [ MeshRegistry, … ] }`

**remove**

- `mesh`（必填；JSON 也可用 `meshName`）
- `version`（必填，`> 0`；JSON / 查询也可用 `ts`）

**meshVersions**

- `mesh`（必填）

**verticesFile**

- `mesh`（必填）；body 为 float32 LE xyz，与 WS `GET_VERTICES` 相同。`Cache-Control: no-store`（浇筑增量会改同一 URL）

**topoFile**

- `mesh`、`tag`（必填；`surfaces` / `faces` / `grids` / `edges` / `lines` / `points`）
- body：`{ mesh, tag, layout }` JSON + `0x1F 0x1E` + 按 layout 顺序的 uint32 LE 索引（与对应 `GET_*` 相同）

**attrVersions**

- `mesh`（必填）；返回 `{ mesh, data }`（`version_ts` / `mesh_version` / `attr_detail` / `step`）

**attrOnePoint**

- `mesh`、`attrName`、`vertexIndex`（必填）
- `axis`（可选；标量可空，矢量指定分量）

**envelope**

- `mesh`（必填）
- `attrName`（可选；有则返回该属性 min/max 数据，无则返回摘要）
- `refresh=1|true`（可选；GET 时先重算再返回）

---

## 删除版本 `POST /damviewer/remove`

删除指定 `version` **及之后**的入库数据（拓扑 / 半动态 / 属性 / `mesh_version` / `attr_import_log`），清理逻辑与原上传重算前相同，但不重新写入。完成后刷新该 mesh 缓存。

单向约束（业务：先删再传）：正在写入 attrstore 时删除返回 `409`；删除进行中**允许**上传。

查询参数：

```bash
curl -X POST "http://localhost:3000/damviewer/remove?mesh=normal&version=1710000000000"
```

或 POST JSON（`mesh` / `meshName`，`version` / `ts` 均可）：

```bash
curl -X POST http://localhost:3000/damviewer/remove \
  -H "Content-Type: application/json" \
  -d '{"mesh":"normal","version":1710000000000}'
```

成功：`{ "mesh": "normal", "version": 1710000000000, "ok": true }`

---

## DXF 剖面叠图 `POST /damviewer/dxf-overlay`

独立模块，与网格 / WS 无关。把结果 DXF 叠到底图 DXF 上，成功直接返回 **`image/svg+xml`**（不是 JSON）。

`multipart/form-data`：

| 字段 | 是否必填 | 说明 |
|------|----------|------|
| `result` | **是** | 结果 DXF（每次都传） |
| `base` | 否 | 底图 DXF。首次或换底图时带上，进程内缓存；之后只传 `result` |

- 底图画 `HATCH` / `LWPOLYLINE` / `LINE`
- 叠加层目前只画 `result` 里的 **`ARC`**，并用底图 `HATCH` 做遮罩：只显示落在底图填充区内的曲线
- 缓存随进程，重启后需再传 `base`
- 失败返回 JSON：`{ "error": "…" }`

```bash
# 首次（带底图）
curl -X POST http://localhost:3000/damviewer/dxf-overlay \
  -F "base=@base.dxf" \
  -F "result=@RESULT_Outline.dxf" \
  -o overlay.svg

# 之后只更新结果
curl -X POST http://localhost:3000/damviewer/dxf-overlay \
  -F "result=@RESULT_Outline.dxf" \
  -o overlay.svg
```

---

## 上传接口 `POST /damviewer/upload`

`multipart/form-data`。超过约 100MB 的部分会落到临时文件（非体积上限），实测 GB 级也可传。

上传成功后**立即返回**，实际导入 / 转换在后台执行；完成后会向 `config.url` 发 POST 回调。

### 全局约束

1. **同一时间只允许一个导入任务**。忙时返回 `409`：
   ```json
   { "status": "busy", "message": "已有导入任务正在进行，请稍后再试" }
   ```
   删除进行中**不**拦上传；上传写入 attrstore 时不允许删除。
2. **必须**带表单字段 `config`（JSON 字符串），且至少包含 `meshName`。
3. `dams` 与 `forecast` **不能同时开启**。
4. 文件字段名可用专用字段（`msh` / `res` / `zip` 等），也可用 `files`，服务端会按扩展名 / 文件名兜底识别。
5. **同一 ts 重传不再自动删除该版本及之后数据**。需要删除时先调用 `POST /damviewer/remove`。

### 上传 config 字段

```json
{
  "meshName": "normal",
  "ts": 1710000000000,
  "id": "task-uuid",
  "url": "https://your-callback/hook",
  "forecast": false,
  "dams": false,
  "meshVersion": 0,
  "requiredResult": "",
  "unitCoefficient": {},
  "zip": null
}
```

| 字段 | 说明 |
|------|------|
| `meshName` | **必填**，目标 mesh 名 |
| `ts` | 时间戳，会原样回传到回调 |
| `id` | 任务 id；`dams` 模式必填 |
| `url` | 后台完成后的回调地址（POST JSON） |
| `forecast` | `true` → 导出 damf，不写 attrstore |
| `dams` | `true` → 测点 yml → `.dams` |
| `meshVersion` | **forecast 必填**：baseline mesh 版本，须在 attrstore catalog 中已存在 |
| `zip` | zip 模式必填：包内 txt 文件名映射 |

---

### 上传模式

服务端根据文件自动判定，三种网格模式 + 一种测点模式：

#### 1. msh + res（attrstore / forecast）

| 表单字段 | 说明 |
|----------|------|
| `msh` | 网格文件 |
| `res` | 结果文件 |
| `config` | JSON |

也可用 `files` 上传，文件名以 `.msh` / `.res` 结尾即可。

- `forecast=false`：解析后写入 `ATTR_DIR`（默认 `./attrstore`）：网格 delta + 场文件
- `forecast=true`：导出 `.damf`，**不写 attrstore**；必须提供 catalog 中已有的 `meshVersion`

#### 2. 双 msh（双网格合并 → damb）

| 表单字段 | 说明 |
|----------|------|
| `msh` | 第一个 msh |
| `msh2` | 第二个 msh |
| `config` | JSON |

也可两个 `.msh` 都放在 `files` 里。

**行为：**

- 不要带 `res`
- 写出 `{DAMB_DIR}/{meshName}.damb`（不写 attrstore）
- 第二层材料号 = 原 matId **`+10000`**（前后端统一；原 `100000` 已废弃）
- 回调带 `"mode": "damb"`，不读 catalog 最新 version
- **暂不支持** `forecast=true`（forecast 仍走 damf，需要 msh+res）

前端 `dualMeshName` 打开后从 `GET /damviewer/damb/:name` 加载该文件。

#### 3. zip（txt 打包）

两条路，按 `config.zip` 是否带场数据（`res`）分流。`forecast=true` 时第三条：只出 damf。

| 表单字段 | 说明 |
|----------|------|
| `zip` | zip 包 |
| `config` | JSON，且必须含非空 `zip` 配置 |

`config.zip` 示例（**带场数据**）：

```json
{
  "meshName": "normal",
  "ts": 1710000000000,
  "id": "xxx",
  "url": "https://callback",
  "forecast": false,
  "zip": {
    "mesh": {
      "coordinates": "节点.txt",
      "elements": { "file": "单元.txt", "skipHeaderLines": 1 }
    },
    "res": {
      "位移": { "file": "位移.txt", "skipHeaderLines": 2, "stripCoordCols": 3 }
    }
  }
}
```

`coordinates` / `elements` / 各 res 条目支持两种写法：

- 字符串：`"节点.txt"`
- 对象：`{ "file": "...", "skipHeaderLines": 2, "stripCoordCols": 3 }`

`stripCoordCols` 只用于 `id x y z val...`（如 TDISPL）。`RESULT_DISPLACEMENT.OUT` 已是 `id ux uy uz`，不要剥坐标；列数不够时解析器会跳过 strip。

**带场数据**（`config.zip` 有 `mesh` + 非空 `res`）：

- 与 msh+res 同一套写出：`InsertMesh` + `InsertAttr` → `ATTR_DIR`（默认 `./attrstore`）
- 网格 delta / catalog 在 `meshfile`，场文件与侧文件在 `attrfile`
- 回调带 `msh` / `res`（来自 catalog）

**仅网格**（有 `mesh.coordinates` + `mesh.elements`，且 **没有** `res`）：

- 走与双 msh 相同的 **damb 写出**（`Convert(true)` → `{meshName}.damb`）
- **没有** `+10000` 材料偏移（单层，无 Secondary）
- 回调 `"mode": "damb"`，不写 attrstore

**forecast=true**（须带 `res`，且 `meshVersion` 已在 catalog 中）：

- 走 `HandleUploadToDamfZip`，导出 `.damf` / `.damfa` / `.damfc`，不写 attrstore

#### 4. dams（测点 → `.dams`）

`config.dams = true`，且必须有 `id`。与 forecast 互斥。写出 `{DAMS_DIR}/{id}.dams`（gzip），前端用 `GET /damviewer/dams/:id` 加载。

**必须同时传 3 个文件**（缺一不可）：

| 表单字段 | 说明 |
|----------|------|
| `points` | 测点坐标。字段名 `points`，或 `files` 里文件名含 `points` |
| `lines` | 测线。字段名 `lines`，或文件名含 `lines` |
| `data` | 监测时序。字段名 `data`，或文件名含 `data` / 为 `data.yml` / `data.yaml` / `data.csv` |
| `config` | JSON，须含 `meshName`、`id`、`"dams": true` |

扩展名常写成 `.yml`，但内容是**逗号分隔文本**（非严格 YAML）。仓库样例：[`溪洛渡/points.yml`](../溪洛渡/points.yml)、[`lines.yml`](../溪洛渡/lines.yml)、[`data.yml`](../溪洛渡/data.yml)。

三者关系：`points` 定义点 → `lines` 用点 ID 连成测线并给出设备 ID → `data` 按设备 ID 填各时点物理量。

**`points`（无表头）** — 每行 `id,x,y,z`，`id` 不可重复：

```
GD01,36.504,21.448,799.380
JC08,36.224,29.122,803.798
```

点名前缀供前端分类着色：`JC` 白、`BD` / `GD` 黑、其它深灰。

**`lines`（无表头）** — 每行 `起点ID,终点ID,设备ID`：

```
JC08,GD02,324001001
JC08,GD07,324001002
```

起点、终点必须出现在 `points` 中；设备 ID 必须非空，用来对齐 `data`。

**`data`（第一行是表头）** — 至少 3 列：`设备ID,时点,attr...`。第 3 列起的表头即物理量名（前端 `slopeAttrs`）：

```
设备ID号,目标时点,目标位移测值(mm),目标温度(°C),原始位移增量(mm)
324001001,2026-07-01 11:00:00,169.26,23.8,-4.01
324001002,2026-07-01 11:00:00,288.3,24.3,-2.33
```

- 设备 ID 必须能在 `lines` 里找到；对不上的行会跳过
- 时点字符串原样作为 step（前端 `slopeSteps`）
- 空单元格或非数字写成 NaN
- 同一设备同一时点后写覆盖先写

`config` 示例：

```json
{
  "meshName": "xiluodu",
  "id": "xiluodu-meas",
  "ts": 1710000000000,
  "url": "https://callback",
  "dams": true
}
```

### 模式互斥规则（常见 400）

| 情况 | 错误含义 |
|------|----------|
| zip 与 msh 同时有 | 不能混传 |
| zip 与 res 同时有 | 不能混传 |
| 两个 msh + res | 双 msh 不要带 res |
| 只有一个 msh、无 res | 单 msh 必须配 res |
| 超过两个 msh | 最多两个 |
| zip 模式但 `config.zip` 为空 | 需要 zip 字段映射 |
| forecast + 双 msh | 暂不支持（双 msh 只出 damb） |
| forecast 且 `meshVersion` 缺失 / 不存在 | 校验失败 |
| dams 无 `id` | 必填 |
| dams 缺 `points` / `lines` / `data` | 三个文件都必须传 |
| dams + forecast | 不能同时开 |

### 成功响应（同步返回）

网格上传：

```json
{
  "message": "文件上传成功，后台正在执行自动导入",
  "timestamp": 1710000000000
}
```

dams 上传：

```json
{
  "message": "测点文件上传成功，后台正在生成 dams",
  "timestamp": 1710000000000,
  "mode": "dams"
}
```

### 后台回调 `POST config.url`

Content-Type: `application/json`，超时 5s。

**attrstore / forecast 通用结构：**

```json
{
  "id": "task-uuid",
  "ts": 1710000000000,
  "ok": true,
  "msh": "{...mesh_version JSON string...}",
  "res": "{...attr_import_log JSON string...}"
}
```

失败时 `ok=false`，并带 `error` 字符串。`msh` / `res` 为 JSON **字符串**（不是嵌套对象）。

**damb 写出（双 msh，或 zip 仅网格）：**

```json
{
  "id": "task-uuid",
  "ts": 1710000000000,
  "ok": true,
  "mode": "damb"
}
```

失败时同样 `ok=false` + `error`，仍带 `"mode": "damb"`。

**dams 回调：**

```json
{
  "id": "xiluodu-meas",
  "ts": 1710000000000,
  "ok": true,
  "mode": "dams",
  "stat": {
    "mesh_name": "xiluodu",
    "point_count": 173,
    "line_count": 301,
    "attr_count": 3,
    "attr_names": ["目标位移测值(mm)", "目标温度(°C)", "原始位移增量(mm)"],
    "step": ["2026-07-01 11:00:00"],
    "kind": "measuring",
    "version": "1.0p"
  }
}
```

若 `url` 为空，回调会静默失败（服务端打日志），不影响已完成的导入 / 导出。

### 上传注意点（速查）

1. **串行**：全局一把锁，并发第二次会 `409`，请排队或等回调后再发。
2. **异步**：HTTP 200 只表示文件已收下并开始处理，最终结果看回调 `ok`。
3. **forecast**：只导出 damf，不写 attrstore；`meshVersion` 必须是 catalog 中已有的 baseline。
4. **双 msh**：写出 `{meshName}.damb`，不要 res、不要 forecast；第二层 mat **+10000**。
5. **zip**：文件名以 `config.zip` 为准，解压后按映射读取。有 `res` → attrstore；仅网格（无 res）→ damb（无 +10000）。
6. **字段名容错**：推荐用 `msh`/`res`/`zip`/`msh2`；用 `files` 时务必保证扩展名正确。dams 推荐用 `points`/`lines`/`data`；走 `files` 时文件名须分别含 `points` / `lines` / `data`。
7. **dams**：必须同时传 points + lines + data 三个文件；`config.id` 即下载名 `{id}.dams`。
8. **临时目录**：同名覆盖（如 `current.msh`），不要并行上传。

---

## WebSocket `GET /damviewer/ws`

前端 DataCenter 走此连接拉拓扑 / 属性。常用 `type`（大小写不敏感）：

| type | 说明 |
|------|------|
| `GET_VERTICES` / `GET_SURFACES` / `GET_FACES` / `GET_GRIDS` / `GET_EDGES` / `GET_LINES` / `GET_POINTS` | 网格拓扑（前端优先走 HTTP `verticesFile` / `topoFile`，失败才用这些） |
| `GET_ATTR` | 物理量场 |
| `GET_ENVELOPE` | 包络场（二进制同 `GET_ATTR`） |
| `GET_CUT` / `GET_CUT_DAMF` | 平面切片 |
| `GET_CELL_HIGHLIGHT` | 单元高亮几何 |
| `GET_LAST_STEP_CELLS` | 末段：最后一挡新增体单元的局部网格 + 当前节点标量 |
| `START_ATTR_WARM` / `CANCEL_ATTR_WARM` | 时间轴预热 |
| `GET_STRESS_CHAIN` | 应力链（不入 cache） |
| `HEARTBEAT` | 保活，服务端丢弃不回包 |

### `GET_STRESS_CHAIN`

从种子顶点沿主应力（p1/p2/p3）走链，一次下发壳面 / 线 / 顶点 / 标量 / 单元中心 / 方向箭头。

payload：

| 字段 | 必填 | 说明 |
|------|------|------|
| `mesh` | 是 | mesh 名 |
| `meshVersion` | 是 | 浇筑版本 |
| `version` / `attrVersion` | 是 | 物理量版本 |
| `vertexIndex` 或 `vertexId` | 是 | 数组下标，或 GiD 节点号 |
| `step` | 否 | 默认 `"1"` |
| `attrName` / `name` | 否 | 默认 `"stress"`，stride 须 ≥ 9 |
| `axis` | 否 | `p1`/`p2`/`p3` 或 `6`/`7`/`8`，默认 `p1` |
| `maxCells` | 否 | 默认 `200` |
| `alignCos` | 否 | 默认 `0.5` |
| `seedPolicy` | 否 | `maxAbs`（默认）/ `nearHit` |
| `hitPoint` | 否 | `[x,y,z]`，`nearHit` 时用 |

Start 元数据带 `layout`（二进制块顺序）以及 `cellIds` / `seedCell` / `axis` 等；随后 chunked binary：`faces` → `lines` → `vertices` → `vertexGlobalIndices` → `scalars` → `cellCenters` → `arrows`（每箭头 7 个 float：`cx,cy,cz,dx,dy,dz,mag`）。

### `GET_LAST_STEP_CELLS`

返回最后一挡浇筑**新增**的体单元：该挡表面有新增面的材料，且只保留这一挡新加的 cell（已打 version 的单元要求 `Version ==` 该挡；历史顺序编号用该挡 `CellStart`..`CellEnd`，1 起、含尾）。更早的 cell 不返回。

payload：

| 字段 | 必填 | 说明 |
|------|------|------|
| `mesh` | 是 | mesh 名 |
| `meshVersion` | 是 | 浇筑版本，`> 0` |
| `version` | 是 | 物理量版本，`> 0` |
| `name` | 是 | 物理量名 |
| `step` | 是 | 时间步字符串 |
| `axis` | 否 | 分量下标，默认 `0` |
| `scalarsOnly` | 否 | `true` 时只重发标量；顶点顺序与全量响应一致（按全局顶点下标升序压缩） |

高阶单元只保留角点（16 节点且前后重复时压成 8，六面体高阶取前 8）。物理量读不到时仍下发几何，标量填 0。

Start 元数据：`mode`（`full` / `scalars`）、`layout`、`cellCount`、`vertexCount`、`matIds`、`lastMeshVersion`、`scalarName`、`axis`。

随后 chunked binary，小端：

- `full`：`vertices` float32 → `cellOffsets` uint32 → `cellConn` uint32 → `scalars` float32
- `scalars`：仅 `scalars` float32

`cellOffsets` 长度为单元数 + 1，`cellConn` 是压缩后的局部顶点下标。切片和补面由前端完成，本接口不切平面。
