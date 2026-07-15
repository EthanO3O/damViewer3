# DamViewer Stream — State & Event API

基于 `字段定义.js` 的状态字段与事件接口说明。通过 `engine.event.emit` 修改状态，通过 `proxy.watch` / `event.watch` 监听变化。

## ChangeLog

### 2026-07-16

- 新增 VIEWER 字段 `envelopeMode`（见 `字段定义.js` `ENVELOPE_MODE`）：`null`/`""` 关闭；`"max"` / `"min"` 显示最大/最小包络云图
- 开启后旁路 `GET /damviewer/envelope`（缺缓存会先 `POST` 构建），忽略 `targetAttrVersion` / `targetStep`；仅改 `envelopeMode` 也会自动刷云图
- `displacements` 包络仍走 `updateDisp`，变形系数 `dispCoef` 照常生效
- 仅一个 attrVersion（甚至单 step）也可算包络；单 step 时 min/max 相同

### 2026-07-14

- 新增 VIEWER 字段 `dualMeshName`：启用/关闭双 msh 加载（非空进入 dual 并同步 `meshName`；`null`/`""` 关闭后按 normal 重载）
- `getAllActorState(viewerId, 'dualMesh')` / `actor:state` 的 `type:'dualMesh'` 已接入
- 补充双 msh 开关与 Actor 读写示例

---

## 目录

- [获取 Proxy](#获取-proxy)
- [修改 State](#修改-state)
- [State 字段](#state-字段)
  - [ROOT](#root)
  - [VIEWER](#viewer)
  - [ACTOR](#actor)
- [预演相关说明](#预演相关说明-forecast--damfscenario)
- [包络图（envelopeMode）](#包络图envelopemode)
- [双 msh（dualMeshName）](#双-mshdualmeshname)
- [Event 事件](#event-事件)
- [监听绑定](#监听绑定)
- [常用示例](#常用示例)
- [默认配置](#默认配置)

---

## 获取 Proxy

```js
engine.getRootState()
engine.getAllViewerState()                           // [proxy, proxy] Viewer 代理对象数组
engine.getAllActorState(viewerId)                    // { [matId]: stateProxy }（仅 normal 材料）
engine.getAllActorState(viewerId, 'measuring')       // { [name]: stateProxy }
engine.getAllActorState(viewerId, 'eph')             // 切片 Eph 材料
engine.getAllActorState(viewerId, 'dualMesh')        // 双 msh Actor（dualMeshName 已启用时）

// 示例
engine.getAllActorState(viewerId)[matId].edge
engine.getAllActorState(viewerId, 'measuring')['H01XH02'].show
```

> **damfScenario 增量 forecast Actor**：与 normal 同结构 state（`show` / `grid` / `edge` / `opacity` …），目前未接入 `actor:state` 的 type 路由；需通过 `actorManager._forecast` 内部 Map 读写（后续可扩展 `type:'forecast'`）。

---

## 修改 State

| 事件 | 用途 | 要点 |
|------|------|------|
| `root:state` | 修改根状态 | payload 直接为字段 |
| `viewer:state` | 修改单个 viewer | 需带 `viewerId` / `id` |
| `viewer:global:state` | 多窗口同步修改 viewer | `id` 可选；不带则全部 |
| `actor:state` | 修改单个 actor | 需 `viewerId` + `matId` / `actorid`；`type` 可选 `normal` / `eph` / `measuring` / `dualMesh` |
| `actor:global:state` | 多窗口/单窗口全部 actor | `viewers`/`viewerId` 可选 |

```js
engine.event.emit(EVENTS.ROOT.STATE, { vCount: 1 })
engine.event.emit("viewer:state", { id: '', state: { grid: false } })
engine.event.emit("actor:state", { viewer: '', actorid: '', state: { edge: true } })
engine.event.emit("viewer:global:state", { id: ['', ''], state: {} })
engine.event.emit("actor:global:state", { viewers: ['', ''], state: { grid: false } })
```

---

## State 字段

### ROOT

| 常量 | 字段名 | 说明 |
|------|--------|------|
| `CONTAINER` | `container` | HTMLElement 容器 |
| `LEGEND_VISIBLE` | `legendVisible` | 是否显示图例 |
| `SELECT_VIEWER_STATE` | `selectViewerState` | 当前选中的 viewer state |
| `VCOUNT` | `vCount` | 渲染器数量 |
| `LOCK_CAM` | `lockCam` | 是否锁定相机联动 |
| `SPRITE_IMG` | `spriteImg` | 标签贴图 base64 / url |
| `LEGEND_IMAGE` | `legendImage` | 图例图片 base64 |

### VIEWER

| 常量 | 字段名 | 说明 |
|------|--------|------|
| `ID` | `id` | Viewer ID |
| `CAMERA_MODE` | `cameraMode` | `"perspective"` 透视 / `"orthographic"` 正交 |
| `CAMERA_POS` | `cameraPos` | 相机位置 |
| `CAMERA_LOOKAT` | `cameraLookat` | 相机注视点 |
| `CAMERA_LOOK_AT_FIT_RATIO` | `cameraLookAtFitRatio` | 相机视角与模型距离倍率，默认 `0.6` |
| `AUTO_ROTATE` | `autoRotate` | 自动旋转 |
| `RAY_MODE` | `rayMode` | `null\|""` 关闭；`"mesh"` 体拾取；`"cut"` 切面拾取；`"measuring"` 测点拾取 |
| `AXIS_STYLE` | `axisStyle` | 坐标轴样式：`gid` / `blender` |
| `AXIS_POSITION` | `axisPosition` | `left-top` / `right-top` / `left-bottom`（默认右下角） |
| `AXIS_SIZE` | `axisSize` | 坐标轴尺寸，默认 `150` |
| `MESH_NAME` | `meshName` | 网格名 |
| `DUAL_MESH_NAME` | `dualMeshName` | 双 msh 网格名（单个 string）；非空进入 dual 并同步 `meshName`；`null`/`""` 关闭后按 normal 重载 |
| `TARGET_ATTR` | `targetAttr` | 物理量名 |
| `TARGET_ATTR_VERSION` | `targetAttrVersion` | 物理量版本（`damfScenario` / `envelopeMode` 开启时不参与云图旁路） |
| `TARGET_MESH_VERSION` | `targetMeshVersion` | 浇筑版本（damf 打开时自动对齐 `base_mesh_version`） |
| `TARGET_STEP` | `targetStep` | 物理量小步 |
| `TARGET_AXIS` | `targetAxis` | 选中轴下标 `0/1/2`（x/y/z）；位移常用 `3` 合位移 |
| `UPDATE_ATTR` | `updateAttr` | 更新 `target*` 后触发云图刷新（**必填触发项**） |
| `ALL_MESH` | `allMesh` | 全坝段线框文件名或空 |
| `ALL_MESH_SHOW` | `allMeshShow` | 全坝段线框显隐（默认 `true`）；各窗口可分别控制 |
| `FORECAST` | `forecast` | 全量 `.damb` 低透明叠层（overlayMesh），非 damf 增量预演 |
| `DAMF_SCENARIO` | `damfScenario` | 增量预演 damf 场景 id；空字符串 = 关闭 |
| `ENVELOPE_MODE` | `envelopeMode` | 包络图：`null`/`""` 关；`"max"` 最大包络；`"min"` 最小包络（见下方专节） |
| `COLOR_PALETTE` | `colorPalette` | 色板图 base64 |
| `NUM_ARR` | `numArr` | 最大最小值显式数组（字符串数组） |
| `MESH_VERSIONS` | `meshVersions` | 浇筑版本列表 |
| `MIN` | `min` | 当前显式内容最小值 |
| `MAX` | `max` | 当前显式内容最大值 |
| `LOCK_RANGE` | `lockRange` | 锁定最大最小值 |
| `PRELOAD` | `preload` | 预加载列表 `[]` |
| `HIGHLIGHT` | `highLight` | 高亮范围 |
| `ENABLEHIGHLIGHT` | `enableHighLight` | 是否启用高亮 |
| `CELL_HIGHLIGHT` | `cellHighlight` | 单元高亮：`{ cellIds: number[] }` |
| `CUT` | `cut` | 切片触发（常用 `Math.random()` 或回车） |
| `SHOWCLIPPLANE` | `showClipPlane` | 材质裁剪开关 `bool` |
| `SHOW_CLIP_PLANE_HELPER` | `showClipPlaneHelper` | 仅显示裁剪辅助面 `bool` |
| `CLIP_PLANE_NORMAL` | `clipPlaneNormal` | 切片方向 |
| `CLIP_PLANE_CONSTANT` | `clipPlaneConstant` | 切片位置 `(-1000, 1000)` |
| `GRID` | `grid` | 全局网格线开关（默认 `false`） |
| `OPACITY` | `opacity` | viewer 级透明度 `0–1`，统一下发到各 actor |
| `USE_DEFAULT_COLOR` | `useDefaultColor` | viewer 级默认色，统一下发到各 actor |
| `MEASURING` | `measuring` | 测点数据 `Point[]` 或 `{ config?, measuring:[] }`；显隐/透明度走 `actorManager(type=measuring)` |

### ACTOR

| 常量 | 字段名 | 说明 |
|------|--------|------|
| `EDGE` | `edge` | 边框线 `bool` |
| `GRID` | `grid` | 网格线 `bool` |
| `LIGHT` | `light` | 发光 `bool` |
| `OPACITY` | `opacity` | 透明度 `0.0 – 1.0` |
| `NAME` | `name` | 名称 |
| `DISPLAY_MODE` | `displayMode` | 显示模式：`point` / `line` / `surface` |
| `SHOW` | `show` | 显示总开关 `bool` |
| `DROP` | `drop` | 删除 actor |
| `SCENARIO_ID` | `scenarioId` | forecast 增量 Actor 专有：所属 damf scenario id（只读，由挂载写入） |
| `USE_DEFAULT_COLOR` | `useDefaultColor` | 是否使用默认色 `bool` |
| `DISP_COEF` | `dispCoef` | 变形系数 |
| `MAT_STYLE` | `matStyle` | 显示材料：`normal` / `shader` |
| `ISO_LINE_NUM` | `isoLineNum` | 等值线分段数，`0` 为关闭 |
| `ISO_LINE_WIDTH` | `isoLineWidth` | 等值线线宽，默认 `1.5` |
| `COLOR_STEP_NUM` | `colorStepNum` | 色阶显式数，默认 `5` |
| `USE_COLOR_STEP` | `usecolorStep` | 是否启用色阶 `bool` |
| `REMAINING` | `remaining` | 是否显示全坝未浇筑部分 |

---

## 预演相关说明（forecast / damfScenario）

| 字段 | 含义 |
|------|------|
| `forecast` | 全量 `.damb` 低透明叠层（overlayMesh），与主模型物理量版本无关，仅参考几何 |
| `damfScenario` | 增量预演 id；加载 `{id}.damf` + 挂 forecast Actor。`normal` = DB 几何 + damf res 前段；`forecast` = damf 增量 mesh + damf res 后段。云图不看 `targetAttrVersion`；测点仍走 WS |

**前置条件**：`meshName` 已加载且与 damf 内 `mesh_name` 一致；通常只需设置 `damfScenario`。

---

## 包络图（envelopeMode）

对应 `字段定义.js` 的 `VIEWER.ENVELOPE_MODE` / `envelopeMode`。

| 值 | 行为 |
|----|------|
| `null` / `""` | 关闭，走普通 `requestAttr` 或 damf 旁路 |
| `"max"` | 显示最大包络场 |
| `"min"` | 显示最小包络场 |

要点：

- 开启后旁路 `GET /damviewer/envelope?mesh=&attrName=`（缺缓存会先 `POST` 构建）
- **忽略** `targetAttrVersion` / `targetStep`；仍使用 `targetAttr` + `targetAxis`
- 数据源优先级：`envelopeMode` → `damfScenario` → 普通 attr
- 仅改 `envelopeMode` 会自动刷新云图；切换物理量时仍建议 bump `updateAttr`
- `displacements`：写入 `updateDisp`，变形由 actor `dispCoef` 控制
- 仅一个 attrVersion 也可算；若只有一个 step，则 min/max 相同

---

## 双 msh（dualMeshName）

| 操作 | 行为 |
|------|------|
| `dualMeshName: "<mesh>"` | 进入双 msh；若与当前 `meshName` 不同会同步写入 `meshName` 并按 dual 加载 |
| `dualMeshName: null` / `""` | 关闭 dual；**保持**当前 `meshName`，按 normal 重载 |
| Actor 读写 | `getAllActorState(viewerId, 'dualMesh')`；`actor:state` 可带 `type:'dualMesh'` |

第二层材质号相对原始材料号有偏移（`DUAL_MESH_MAT_OFFSET = 100000`），与后端 `DualMeshMaterialOffset` 一致。

---

## Event 事件

### ROOT

| 常量 | 事件名 | 说明 |
|------|--------|------|
| `STATE` | `root:state` | 修改根状态 |

### VIEWER

| 常量 | 事件名 | 说明 |
|------|--------|------|
| `RESIZE` | `viewer:resize` | 重新计算 canvas 尺寸 |
| `ID` | `viewer:id` | Viewer ID 相关 |
| `CAMERAVIEWE` | `viewer:camera:view` | 切换 camera 位置 |
| `CAMERASET` | `viewer:camera:set` | 设置相机 |
| `STATE` | `viewer:state` | 修改单个 viewer state |
| `RAY_CB` | `viewer:ray:cb` | 订阅：ray 模式返回数据 |
| `RAY_ProLine` | `viewer:ray:processline` | 射线处理线 |
| `SHOWMAX` | `viewer:show:max` | 显示 max 标签 |
| `SHOWMIN` | `viewer:show:min` | 显示 min 标签 |
| `CLEARMINMAX` | `viewer:clear:minmax` | 清空最值标签 |
| `GETMAXTAS` | `viewer:max:tas` | 获取温度应力最大值 |
| `SENDMAXTAS` | `viewer:max:res` | 温度应力最大值回调 |
| `SHOWINTEGRAL` | `viewer:show:integral` | 显示积分 |
| `HIDEINTEGRAL` | `viewer:hide:integral` | 隐藏积分 |
| `INITDONE` | `viewer:init:done` | 初始化完成，可 `setMeshName` |
| `ALL_MESH_DONE` | `viewer:allmesh:done` | 全坝段线框 `allMesh` 加载完成 |
| `CLIP_POS_CB` | `viewer:clipPos:cb` | 订阅：剖切平面坐标位置回调 |

### ACTOR

| 常量 | 事件名 | 说明 |
|------|--------|------|
| `STATE` | `actor:state` | 修改 actor；`type` 可选 `normal` / `eph` / `measuring` / `dualMesh`（`forecast` 待扩展） |
| `DONE` | `actor:done` | 可订阅；`payload.kind` 可为 `normal` / `forecast` / …，forecast 时带 `scenarioId` |

### ACTOR_GLOBAL / VIEWER_GLOBAL

| 常量 | 事件名 | 说明 |
|------|--------|------|
| `ACTOR_GLOBAL.STATE` | `actor:global:state` | 全局 actor 状态 |
| `VIEWER_GLOBAL.RESIZE` | `viewer:global:resize` | 全局 resize |
| `VIEWER_GLOBAL.ACTIVATE` | `viewer:global:activate` | 切换 viewer 时发送，带对象 id |
| `VIEWER_GLOBAL.INDEX` | `viewer:global:index` | viewer 索引 |
| `VIEWER_GLOBAL.STATE` | `viewer:global:state` | 全局 viewer 状态 |

### WS / DATA

| 常量 | 事件名 | 说明 |
|------|--------|------|
| `WS.CONNECT` | `finite:ws:connect` | WebSocket 连接成功 |
| `WS.CLOSE` | `finite:ws:close` | WebSocket 关闭 |
| `WS.ERROR` | `finite:ws:error` | WebSocket 错误 |
| `DATA.MESH_READY` | `finite:data:mesh_ready` | 网格就绪 |
| `DATA.VERTICES_UPDATED` | `finite:data:vertices_updated` | 顶点更新 |
| `DATA.SURFACE_UPDATED` | `finite:data:surface_updated` | 表面更新 |
| `DATA.LINE_UPDATED` | `finite:data:line_updated` | 线更新 |
| `DATA.EDGE_UPDATED` | `finite:data:edge_updated` | 边更新 |
| `DATA.GRID_UPDATED` | `finite:data:grid_updated` | 网格线更新 |
| `DATA.POINT_UPDATED` | `finite:data:point_updated` | 点更新 |
| `DATA.FACE_UPDATED` | `finite:data:face_updated` | 面更新 |
| `DATA.ATTR_UPDATED` | `finite:data:attr_updated` | 属性更新 |
| `DATA.CUT_READY` | `finite:data:cut_ready` | 切片 bin 就绪（含 damfScenario 的 `GET_CUT_DAMF` 单包） |

---

## 监听绑定

### 监听 State

```js
const eventHandlers = {
  [STATES.ROOT.VCOUNT]: (payload) => {
    // 渲染窗口数量变化后执行
  },
}

proxy.watch(this) // 启动监听
```

### 监听 Event

```js
const eventHandlers = {
  [EVENTS.WS.CONNECT]: () => {
    // WebSocket 连接成功
  },
}

event.watch(this) // this 为需要监听的对象
engine.event.emit(EVENTS.ROOT.STATE)
```

---

## 常用示例

### 切换浇筑步 / 物理场

允许只改其中一部分，但 **`updateAttr` 总是需要修改**。

```js
engine.event.emit("viewer:state", {
  viewerId: "meEnXC0yvSfZ10KxrLm-g",
  state: {
    targetAttr: "stress",
    targetAttrVersion: 1767161923853,
    targetMeshVersion: 1767161923853,
    targetStep: 1,
    targetAxis: 3,
    updateAttr: Math.random(),
  },
})
```

### 包络图 envelopeMode

与 `字段定义.js` 示例一致。

```js
// 开最大包络（可与 targetAttr / targetAxis / updateAttr 同一次写入）
engine.event.emit("viewer:state", {
  viewerId: "<viewerId>",
  state: {
    envelopeMode: "max", // 或 "min"；关闭: null / ""
    targetAttr: "stress",
    targetAxis: 0,
    updateAttr: Date.now(),
  },
})

// 位移包络 + 变形（dispCoef 仍写 actor）
engine.event.emit("viewer:state", {
  viewerId: "<viewerId>",
  state: {
    envelopeMode: "min",
    targetAttr: "displacements",
    targetAxis: 3,
    updateAttr: Date.now(),
  },
})
engine.event.emit("actor:global:state", {
  viewerId: "<viewerId>",
  state: { dispCoef: 50 },
})
```

### 双 msh dualMeshName

```js
// 进入 dual（若与当前 meshName 不同会同步切换）
engine.event.emit("viewer:state", {
  viewerId: "<viewerId>",
  state: { dualMeshName: "pre_jingling" },
})

// 关闭 dual：保持 meshName，按 normal 重载
engine.event.emit("viewer:state", {
  viewerId: "<viewerId>",
  state: { dualMeshName: null },
})

// 读写 dual Actor
engine.getAllActorState(viewerId, "dualMesh")
engine.event.emit("actor:state", {
  viewerId,
  matId: "1",
  type: "dualMesh",
  state: { show: false },
})
```

### 增量预演 damfScenario

```js
// 开启（关闭：damfScenario: ""）
engine.event.emit("viewer:global:state", {
  state: { damfScenario: "your-scenario-id" },
})

// damf 模式下切换物理场：不需 targetAttrVersion
engine.event.emit("viewer:state", {
  viewerId: "<viewerId>",
  state: {
    targetAttr: "stress",
    targetStep: "1",
    targetAxis: 0,
    updateAttr: Date.now(),
  },
})

// 体拾取 / 切面拾取
engine.event.emit("viewer:global:state", { state: { rayMode: "mesh" } })
engine.event.emit("viewer:global:state", { state: { rayMode: "cut" } })

// 切片（damfScenario 开启时内部 GET_CUT_DAMF）
engine.event.emit("viewer:global:state", { state: { cut: Math.random() } })
```

### Actor 控制

```js
// 全部 actor → 点模式
engine.event.emit("actor:global:state", {
  viewerId: "axajqrpCGTVvRmILdCtrn",
  state: { displayMode: "point" },
})

// 关闭网格
engine.event.emit("actor:global:state", {
  viewerId: "r9MaKsZX2QrLq2rpMj7R0",
  state: { grid: false },
})

// 法线材料
engine.event.emit("actor:global:state", {
  viewerId: "kek02-YfADpTQQZHC5LzI",
  state: { matStyle: "normal" },
})

// 单个 actor 隐藏
engine.event.emit("actor:state", {
  viewerId: "r9MaKsZX2QrLq2rpMj7R0",
  matId: "1",
  state: { show: false },
})

// 开启色阶，分两段
engine.event.emit("actor:global:state", {
  viewerId: "WFn2yEpDthoY8U67znzTu",
  state: { usecolorStep: true, colorStepNum: 2 },
})
```

### 全坝段线框显隐

```js
// 单个 viewer
engine.event.emit("viewer:state", {
  viewerId: "<viewerId>",
  state: { allMeshShow: false },
})

// 全部 viewer
engine.event.emit("viewer:global:state", {
  state: { allMeshShow: true },
})

// 指定若干 viewer
engine.event.emit("viewer:global:state", {
  id: ["<viewerId1>", "<viewerId2>"],
  state: { allMeshShow: false },
})
```

---

## 默认配置

```js
const defaultConfig = {
  container: [],
  axes: { size: "8rem", type: "blender" },
  renderer: {},
  wsUrl: "ws://sky.leve.vip:8888/jl_ws/",
  server: "http://sky.leve.vip:3000/",
  meshName: "test3",
  camPos: { x: 315.63, y: 284.78, z: 247.46 },
  camTarget: { x: 40.15, y: -3.32, z: 124.10 },
}
```

---

*字段与事件定义见 [`字段定义.js`](./字段定义.js)。*
