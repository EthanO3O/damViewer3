# DamViewer Stream — State & Event API

基于 `字段定义.js` 的状态字段与事件接口说明。通过 `engine.event.emit` 修改状态；State 用 `proxy.watch` / `unwatch`，Event 用 `event.on` / `off`（或 `event.watch` / `unwatch`）监听与取消。

## ChangeLog

### 2026-08-17

- 剖切辅助面控件整体加大；鼠标悬停拾取到手柄即放大，点击后不再二次放大（旋转环按轴放大，避免位移跑偏）
- 双 msh：几何改为加载 `{server}damb/{meshName}`（不再走 WS/DB 拓扑）；第二层材料号偏移 **`+10000`**（原 `100000` 废弃）

### 2026-08-10

- 新增应力链：`stressChainMode` 开启后自动 `rayMode=mesh`，**Ctrl+左键**点选 normal / forecast 顶点生成单链（覆盖）
- 走 WS `GET_STRESS_CHAIN`；独立 Actor（壳 + 中心线 + 方向箭头 + 端点标记），不入 cache、不参与主网格拾取
- 需 `targetAttr: "stress"`；`p1/p2/p3`（轴 6/7/8）可单独切；种子策略 `nearHit` / `maxAbs`

### 2026-08-04

- 新增 ACTOR 位移矢量箭头：`arrow` / `arrowMode` / `arrowScale` / `arrowAspect` / `arrowMinMag` / `arrowStride`
- GPU 实例化（杆 + 固定锥角圆锥头）；默认合力向量，可选 XYZ 三轴分显
- 支持 **normal / eph（切片）/ forecast（damf）**；`actor:global:state` 未指定 `type` 时三层一起写
- 需先加载 `targetAttr: "displacements"`；最长箭头 ≈ 模型对角线 × 3% × `arrowScale`

### 2026-07-26

- 新增 ROOT 字段 `switchFx`：统一开关三种切换动效（`meshName` 平面扫光 / `drawRange` faceFly / `attr` 颜色交叉淡入）
- `false` 全关；`true` 或缺省全开；对象 `{ meshName, drawRange, attr }` 可分别控制

### 2026-07-23

- 补充 State / Event 监听与取消监听 API（`watch` / `unwatch`、`on` / `off` / `once`）

### 2026-07-21

- 新增 VIEWER 字段 `colorLegendVisible`（默认 `false`）：每 viewer 右下角色标图例，仅 `true` 时显示

### 2026-07-29

- `viewer:camera:view:done`：等全部 topo 落定 + actor/扫光完成后再最终 fit；done 后关闭自动 fit，外部可安全改相机

### 2026-07-20

- 新增 VIEWER 字段 `dams` / `slopeLineWidth`：加载 `.dams` → **SlopeActor**（粗线 LineSegments2 + JC 白/BD·GD 黑/其它深灰点球；线色走 Actor LUT 深蓝→深红）
- 内部自动相机 fit 与外部 `viewer:camera:view` 分离；内部完成后 emit `viewer:camera:view:done`（payload: `{ id, source }`）
- `viewer:camera:view:done` 表示**内部视角处理已结束**（全部 topo 落定 + actor 构建/扫光完成 + 最终 fit 动画结束），外部此时可安全改相机；之后不会再因增量 topo 自动重置视角

### 2026-07-16

- 新增引擎根 **网络忙状态**：`root.networkBusy` / `networkActive` / `networkByKind`，事件 `net:busy`（兼 `net:start` / `net:end`）
- 覆盖 WS（`sendQuery`）、damf 加载挂载、damb、meshVersions；cache 命中不计 busy
- 新增 VIEWER 字段 `envelopeMode`（见 `字段定义.js` `ENVELOPE_MODE`）：`null`/`""` 关闭；`"max"` / `"min"` 显示最大/最小包络云图
- 开启后旁路 **WS `GET_ENVELOPE`**（二进制格式与 `GET_ATTR` 相同；缺缓存时服务端先 Build），忽略 `targetAttrVersion` / `targetStep`；仅改 `envelopeMode` 也会自动刷云图
- HTTP `POST/GET /damviewer/envelope` 仍可用于调试/强制刷新；前端显示链路不走 HTTP
- `displacements` 包络仍走 `updateDisp`，变形系数 `dispCoef` 照常生效
- 仅一个 attrVersion（甚至单 step）也可算包络；单 step 时 min/max 相同

### 2026-07-14

- 新增 VIEWER 字段 `dualMeshName`：启用/关闭双 msh 加载（非空进入 dual 并同步 `meshName`；`null`/`""` 关闭后按 normal 重载）
- `getAllActorState(viewerId, 'dualMesh')` / `actor:state` 的 `type:'dualMesh'` 已接入
- 补充双 msh 开关与 Actor 读写示例

---

## 目录

- [DamViewer Stream — State \& Event API](#damviewer-stream--state--event-api)
  - [ChangeLog](#changelog)
    - [2026-08-17](#2026-08-17)
    - [2026-08-10](#2026-08-10)
    - [2026-08-04](#2026-08-04)
    - [2026-07-26](#2026-07-26)
    - [2026-07-23](#2026-07-23)
    - [2026-07-21](#2026-07-21)
    - [2026-07-29](#2026-07-29)
    - [2026-07-20](#2026-07-20)
    - [2026-07-16](#2026-07-16)
    - [2026-07-14](#2026-07-14)
  - [目录](#目录)
  - [获取 Proxy](#获取-proxy)
  - [修改 State](#修改-state)
  - [State 字段](#state-字段)
    - [ROOT](#root)
    - [VIEWER](#viewer)
    - [ACTOR](#actor)
  - [预演相关说明（forecast / damfScenario）](#预演相关说明forecast--damfscenario)
  - [包络图（envelopeMode）](#包络图envelopemode)
  - [位移矢量箭头（arrow）](#位移矢量箭头arrow)
  - [应力链（stressChainMode）](#应力链stresschainmode)
  - [切换动效（switchFx）](#切换动效switchfx)
  - [网络忙状态（networkBusy）](#网络忙状态networkbusy)
  - [双 msh（dualMeshName）](#双-mshdualmeshname)
  - [Event 事件](#event-事件)
    - [ROOT](#root-1)
    - [NET](#net)
    - [VIEWER](#viewer-1)
    - [ACTOR](#actor-1)
    - [ACTOR\_GLOBAL / VIEWER\_GLOBAL](#actor_global--viewer_global)
    - [WS / DATA](#ws--data)
  - [监听绑定](#监听绑定)
    - [监听 State](#监听-state)
    - [取消 State 监听](#取消-state-监听)
    - [监听 Event](#监听-event)
    - [取消 Event 监听](#取消-event-监听)
  - [常用示例](#常用示例)
    - [切换浇筑步 / 物理场](#切换浇筑步--物理场)
    - [边坡监测 dams / SlopeActor](#边坡监测-dams--slopeactor)
    - [切换动效 switchFx](#切换动效-switchfx)
    - [网络忙状态](#网络忙状态)
    - [包络图 envelopeMode](#包络图-envelopemode)
    - [位移矢量箭头 arrow](#位移矢量箭头-arrow)
    - [应力链 stressChainMode](#应力链-stresschainmode)
    - [双 msh dualMeshName](#双-msh-dualmeshname)
    - [增量预演 damfScenario](#增量预演-damfscenario)
    - [Actor 控制](#actor-控制)
    - [全坝段线框显隐](#全坝段线框显隐)
  - [默认配置](#默认配置)

---

## 获取 Proxy

```js
engine.getRootState()
engine.getAllViewerState()                           // [proxy, proxy] Viewer 代理对象数组
engine.getAllActorState(viewerId)                    // { [matId]: stateProxy }（仅 normal 材料）
engine.getAllActorState(viewerId, 'measuring')       // { [name]: stateProxy }
engine.getAllActorState(viewerId, 'eph')             // 切片 Eph 材料
engine.getAllActorState(viewerId, 'forecast')        // damf 增量 forecast Actor（damfScenario 激活时）
engine.getAllActorState(viewerId, 'dualMesh')        // 双 msh Actor（dualMeshName 已启用时）

// 示例
engine.getAllActorState(viewerId)[matId].edge
engine.getAllActorState(viewerId, 'measuring')['H01XH02'].show
```

> **`actor:global:state`**：未传 `type` 时同步 **normal + eph + forecast**；也可 `type: 'eph' | 'forecast' | 'normal'` 只打一层。

---

## 修改 State

| 事件 | 用途 | 要点 |
|------|------|------|
| `root:state` | 修改根状态 | payload 直接为字段 |
| `viewer:state` | 修改单个 viewer | 需带 `viewerId` / `id` |
| `viewer:global:state` | 多窗口同步修改 viewer | `id` 可选；不带则全部 |
| `actor:state` | 修改单个 actor | 需 `viewerId` + `matId` / `actorid`；`type` 可选 `normal` / `eph` / `forecast` / `measuring` / `dualMesh` |
| `actor:global:state` | 多窗口/单窗口全部 actor | `viewers`/`viewerId` 可选；未传 `type` 时写 normal+eph+forecast |

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
| `NETWORK_BUSY` | `networkBusy` | 全局是否有在途网络任务（WS / damf / damb 等） |
| `NETWORK_ACTIVE` | `networkActive` | 在途任务数 |
| `NETWORK_BY_KIND` | `networkByKind` | 按 kind 计数，如 `{ ws: 2, damf: 1 }` |
| `SWITCH_FX` | `switchFx` | 三种切换动效总开关（见下方专节） |

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
| `COLOR_LEGEND_VISIBLE` | `colorLegendVisible` | 右下角色标图例显隐，默认 `false` |
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
| `SHOW_CLIP_PLANE_HELPER` | `showClipPlaneHelper` | 裁剪辅助面 + 平移/旋转控件；悬停拾取到手柄即放大 |
| `CLIP_PLANE_NORMAL` | `clipPlaneNormal` | 切片方向 |
| `CLIP_PLANE_CONSTANT` | `clipPlaneConstant` | 切片位置 `(-1000, 1000)` |
| `GRID` | `grid` | 全局网格线开关（默认 `false`） |
| `OPACITY` | `opacity` | viewer 级透明度 `0–1`，统一下发到各 actor |
| `USE_DEFAULT_COLOR` | `useDefaultColor` | viewer 级默认色，统一下发到各 actor |
| `MEASURING` | `measuring` | 测点数据 `Point[]` 或 `{ config?, measuring:[] }`；显隐/透明度走 `actorManager(type=measuring)` |
| `DAMS` | `dams` | 边坡监测 dams id；空=关闭；加载 `{server}dams/{id}` → SlopeActor |
| `SLOPE_LINE_WIDTH` | `slopeLineWidth` | SlopeActor 线宽（像素），默认 `3.5` |
| `SLOPE_POINT_RADIUS` | `slopePointRadius` | SlopeActor 点球世界半径，默认 `0.8` |
| `STRESS_CHAIN_MODE` | `stressChainMode` | 应力链模式；`true` 时自动 `rayMode=mesh`，Ctrl+左键点选生成单链（见下方专节） |
| `STRESS_CHAIN_SEED` | `stressChainSeed` | 当前链种子 `{ vertexIndex?, vertexId?, axis?, hitPoint? }`；`null` = 无链 |
| `STRESS_CHAIN_AXIS` | `stressChainAxis` | 主应力分量：`6`/`"p1"`、`7`/`"p2"`、`8`/`"p3"`（默认 `6`） |
| `STRESS_CHAIN_SEED_POLICY` | `stressChainSeedPolicy` | 共用顶点时选种子单元：`nearHit`（默认）/ `maxAbs` |
| `STRESS_CHAIN_SHOW_SHELL` | `stressChainShowShell` | 链壳显隐，默认 `true` |
| `STRESS_CHAIN_SHOW_LINE` | `stressChainShowLine` | 中心线显隐，默认 `true` |
| `STRESS_CHAIN_SHOW_ARROWS` | `stressChainShowArrows` | 方向箭头显隐，默认 `true` |
| `STRESS_CHAIN_SHELL_OPACITY` | `stressChainShellOpacity` | 链壳透明度，默认 `0.85` |
| `STRESS_CHAIN_LINE_WIDTH` | `stressChainLineWidth` | 中心线宽，默认 `2.5` |
| `STRESS_CHAIN_ARROW_SCALE` | `stressChainArrowScale` | 箭头长度倍率，默认 `0.6` |
| `STRESS_CHAIN_ARROW_OPACITY` | `stressChainArrowOpacity` | 箭头透明度，默认 `0.92` |
| `STRESS_CHAIN_ARROW_COLOR` | `stressChainArrowColor` | 箭头颜色，默认 `"#000000"` |
| `STRESS_CHAIN_MAX_CELLS` | `stressChainMaxCells` | 链最长单元数，默认 `200` |
| `STRESS_CHAIN_ALIGN_COS` | `stressChainAlignCos` | 相邻单元主方向对齐阈值（余弦），默认 `0.5` |

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
| `ARROW` | `arrow` | 位移矢量箭头开关 `bool`（默认 `false`） |
| `ARROW_MODE` | `arrowMode` | `resultant` 合力（默认）/ `xyz` 三轴 RGB 分显 |
| `ARROW_SCALE` | `arrowScale` | 长度倍率（相对自适应参考长度，默认 `1`） |
| `ARROW_ASPECT` | `arrowAspect` | 锥头底半径/头高（固定锥角，宜 `0.35~0.55`，默认 `0.45`） |
| `ARROW_MIN_MAG` | `arrowMinMag` | 模长阈值，低于不画 |
| `ARROW_STRIDE` | `arrowStride` | 顶点抽样步长 `>=1`，越大越稀 |
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

- 开启后旁路 **WS `GET_ENVELOPE`**（`mesh` + `name` + `mode:min|max`），二进制流与 `GET_ATTR` 相同（LE float32，按 stride）
- 缺缓存时服务端在同一次请求内 `BuildEnvelope` 再下发；HTTP `POST /damviewer/envelope` 仍可手动全量刷新
- **忽略** `targetAttrVersion` / `targetStep`；仍使用 `targetAttr` + `targetAxis`
- 数据源优先级：`envelopeMode` → `damfScenario` → 普通 attr
- 仅改 `envelopeMode` 会自动刷新云图；切换物理量时仍建议 bump `updateAttr`
- `displacements`：写入 `updateDisp`，变形由 actor `dispCoef` 控制
- 仅一个 attrVersion 也可算；若只有一个 step，则 min/max 相同

---

## 位移矢量箭头（arrow）

对应 `字段定义.js` 的 `ACTOR.ARROW*`。在已加载 **`displacements`** 后，于顶点绘制杆 + 圆锥头矢量（不写深度，叠在模型上）。

| 字段 | 默认 | 说明 |
|------|------|------|
| `arrow` | `false` | 开关 |
| `arrowMode` | `"resultant"` | 合力一根 / `"xyz"` 三轴分显（X 红 Y 绿 Z 蓝） |
| `arrowScale` | `1` | 最长箭头 ≈ 模型对角线 × 3% × scale |
| `arrowAspect` | `0.45` | 锥头底半径 / 头高（头形固定，只拉长圆柱杆） |
| `arrowMinMag` | `1e-8` | 模长阈值 |
| `arrowStride` | `1` | 抽样步长，大模型建议 `4` 起 |

适用范围：**normal** 主网格、**eph** 切片、**forecast** damf 增量。合力模式颜色跟顶点 `scalar`/LUT；三轴模式用固定 RGB。

---

## 应力链（stressChainMode）

对应 `字段定义.js` / `state_viewer.js` 的 `VIEWER.STRESS_CHAIN_*`。在已加载 **`targetAttr: "stress"`**（9 分量）后，从种子顶点沿主应力方向走链，单独渲染一条链（覆盖，不同时显示多条）。

| 字段 | 默认 | 说明 |
|------|------|------|
| `stressChainMode` | `false` | 开：自动切 `rayMode=mesh`；关：恢复之前的 rayMode 并清链 |
| `stressChainSeed` | `null` | 由 Ctrl+左键写入；也可手动设 `{ vertexIndex }` 或 `{ vertexId }` |
| `stressChainAxis` | `6` | `p1`/`p2`/`p3`（6/7/8）；改轴会用同一种子重算 |
| `stressChainSeedPolicy` | `"nearHit"` | 顶点被多单元共用时：距拾取点最近 / `\|σ\|` 最大 |
| `stressChainMaxCells` | `200` | 链最长单元数 |
| `stressChainAlignCos` | `0.5` | 相邻单元主方向夹角余弦下限，越小越容易拐弯 |

交互：

- **Ctrl+左键**点选 **normal** 或 **forecast** 表面顶点（切片 / 应力链自身不作为种子）
- 开启后普通 actor 透明度会压到约 `0.1`，便于看链
- 箭头不写深度；端点为屏幕固定大小的圆点（白=种子，另一端着色）

数据：旁路 WS **`GET_STRESS_CHAIN`**，二进制不入 cache。改 `targetAttr` / 版本 / step 时若模式仍开，会按当前种子刷新。

---

## 切换动效（switchFx）

挂在 **引擎根**（`root.state.switchFx`），默认全开：

```js
{
  meshName: true,   // meshName 切换：平面扫光（出/入）
  drawRange: true,  // meshVersion / drawRange：faceFly 增减面动效
  attr: true,       // 物理量切换：shader 颜色交叉淡入
}
```

| 写法 | 行为 |
|------|------|
| 缺省 / `true` | 三种动效全开 |
| `false` | 三种动效全关（瞬时切换） |
| `{ meshName, drawRange, attr }` | 按字段分别开关；某键 `!== false` 即开 |

通过 `root:state` 修改（赋值整对象更稳妥，避免只改嵌套字段时漏触发）：

```js
// 全关
engine.event.emit("root:state", { switchFx: false })

// 只保留 mesh 扫光，关掉 drawRange / attr
engine.event.emit("root:state", {
  switchFx: { meshName: true, drawRange: false, attr: false },
})

// 恢复默认全开
engine.event.emit("root:state", {
  switchFx: { meshName: true, drawRange: true, attr: true },
})
```

也可直接写根 state：

```js
const root = engine.getRootState()
root.switchFx = { meshName: true, drawRange: false, attr: false }
```

---

## 网络忙状态（networkBusy）

挂在 **引擎根**（`DVEngineContext` / `NetStatusManager`），不放在 finite 业务层。WS 池与 forecast HTTP 统一引用计数。

| kind | 来源 |
|------|------|
| `ws` | `Net.sendQuery`（attr / envelope / topo / cut…） |
| `damf` | damf 加载 + 挂载整段 |
| `http.damb` | `allMesh` / `forecast` / 双 msh 的 damb |
| `http.meshVersions` | mesh 切换拉版本列表 |
| `http.forecast` / `http.forecast.warm` | 单独拉 damf / 预热（未被外层 `damf` 包住时） |

cache 命中不计 busy。

```js
// 绑 UI（State）
const netUi = {
  stateHandlers: {
    networkBusy: (busy) => { /* loading overlay */ },
  },
}
engine.getRootState().watch(netUi)

// 或事件（busy 仅在 true↔false 翻转时发出）
const onBusy = ({ busy, active, byKind }) => { /* ... */ }
engine.event.on('net:busy', onBusy)

// 快照
engine.getNetStatus?.() // { busy, active, byKind }

// 取消
engine.getRootState().unwatchAll(netUi)
engine.event.off('net:busy', onBusy)
```

相关事件：`net:start` / `net:end` / `net:busy`（常量 `EVENTS.NET.*`）。

---

## 双 msh（dualMeshName）

| 操作 | 行为 |
|------|------|
| `dualMeshName: "<mesh>"` | 进入双 msh；若与当前 `meshName` 不同会同步写入 `meshName` 并按 dual 加载 |
| `dualMeshName: null` / `""` | 关闭 dual；**保持**当前 `meshName`，按 normal 重载 |
| Actor 读写 | `getAllActorState(viewerId, 'dualMesh')`；`actor:state` 可带 `type:'dualMesh'` |

第二层材质号相对原始材料号有偏移（`DUAL_MESH_MAT_OFFSET = 10000`），与后端 `DualMeshMaterialOffset` 一致。几何来自 **`GET {server}damb/{meshName}`**（单 dim，`matId >= 10000` 为第二层），不再走 WS 拉双 msh 拓扑。

---

## Event 事件

### ROOT

| 常量 | 事件名 | 说明 |
|------|--------|------|
| `STATE` | `root:state` | 修改根状态 |

### NET

| 常量 | 事件名 | 说明 |
|------|--------|------|
| `START` | `net:start` | 单个网络任务开始 `{ id, kind, meta, busy, active, byKind }` |
| `END` | `net:end` | 单个网络任务结束 `{ id, kind, ok, error?, … }` |
| `BUSY` | `net:busy` | 汇总忙状态变化 `{ busy, active, byKind }` |

### VIEWER

| 常量 | 事件名 | 说明 |
|------|--------|------|
| `RESIZE` | `viewer:resize` | 重新计算 canvas 尺寸 |
| `ID` | `viewer:id` | Viewer ID 相关 |
| `CAMERAVIEWE` | `viewer:camera:view` | 外部发送：切换 camera 位置（与内部自动 fit 分离） |
| `CAMERA_VIEW_DONE` | `viewer:camera:view:done` | 内部视角处理结束，外部可改相机；payload: `{ id, source }` |
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

State 与 Event 是两套 API：改字段走 state proxy；发/收消息走 `engine.event`。

### 监听 State

拿到 proxy（如 `engine.getRootState()`、viewer/actor state）后订阅字段变化。推荐在对象上声明 `stateHandlers`，再 `watch`：

```js
const ui = {
  stateHandlers: {
    // key = 字段名；回调 (newVal, oldVal)
    [STATES.ROOT.VCOUNT]: (vCount, prev) => {
      // 渲染窗口数量变化
    },
    networkBusy: (busy) => {
      // loading overlay
    },
  },
}

const root = engine.getRootState()
root.watch(ui) // 一次性绑上 ui.stateHandlers 里所有 key
```

也可按字段订阅（第 2 参为 subscriber 身份，用于之后 unwatch；第 3 参可选自定义回调）：

```js
const token = {}
root.watch('networkBusy', token, (_sub, _key, oldVal, newVal) => {
  // newVal === busy
})

// 若省略第 3 参，则调用 token.stateHandlers.networkBusy(newVal, oldVal)
root.watch('networkBusy', ui)
```

> 注意：`watch('networkBusy', (busy) => {})` **无效**——第 2 参是 subscriber，不是回调。请用 `stateHandlers` 或三参数形式。

变更在微任务中批处理：同一次 tick 内多次赋值，按 key 合并后再回调。

### 取消 State 监听

```js
const root = engine.getRootState()

root.unwatch('networkBusy', ui) // 取消某一字段
root.unwatchAll(ui)             // 取消该 subscriber 上所有字段（dispose 常用）
```

### 监听 Event

两种写法等价可用：

**1. `on` / `once`（业务侧常用）**

```js
const onBusy = ({ busy, active, byKind }) => { /* ... */ }
const onRay = (payload) => { /* ... */ }

engine.event.on('net:busy', onBusy)
engine.event.on(EVENTS.VIEWER.RAY_CB, onRay)

// 只触发一次，自动卸掉
engine.event.once('finite:ws:connect', () => { /* ... */ })
```

**2. `watch` + `eventHandlers`（模块内部常用）**

```js
const mod = {
  eventHandlers: {
    [EVENTS.WS.CONNECT]: () => { /* WebSocket 连接成功 */ },
    'viewer:ray:cb': (payload) => { /* ... */ },
  },
}

engine.event.watch(mod) // 把 mod.eventHandlers 全部挂上
```

发送：`engine.event.emit(eventName, detail?)`。

### 取消 Event 监听

```js
// on 时必须保留同一函数引用，才能精确 off
engine.event.off('net:busy', onBusy)

// 去掉该事件上全部监听
engine.event.off('net:busy')

// 通配符
engine.event.off('net:*')

// watch 注册的，用 unwatch 整对象卸掉
engine.event.unwatch(mod)
```

> 匿名函数无法取消：`on('net:busy', () => {})` 之后无法精确 `off`。请先赋给变量再 `on` / `off`。

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

### 边坡监测 dams / SlopeActor

```js
engine.event.emit("viewer:state", {
  viewerId: "<viewerId>",
  state: {
    dams: "xiluodu-meas",
    slopeLineWidth: 3.5, //线宽
    slopePointRadius: 0.8, //点大小
    targetAttr: "目标位移测值(mm)",
    targetStep: "2026-07-01 11:00:00",
    updateAttr: Math.random(),
  },
})
// 关闭
engine.event.emit("viewer:state", {
  viewerId: "<viewerId>",
  state: { dams: "" },
})
```

### 切换动效 switchFx

```js
// 性能优先：关掉 drawRange / attr，保留 meshName 扫光
engine.event.emit("root:state", {
  switchFx: { meshName: true, drawRange: false, attr: false },
})

// 全部瞬时切换（无动效）
engine.event.emit("root:state", { switchFx: false })
```

### 网络忙状态

```js
const netUi = {
  stateHandlers: {
    networkBusy: (busy) => { /* loading overlay */ },
  },
}
engine.getRootState().watch(netUi)

const onBusy = ({ busy, active, byKind }) => {
  // byKind.ws / byKind.damf / …
}
engine.event.on("net:busy", onBusy)

engine.getNetStatus() // { busy, active, byKind }

// 取消
engine.getRootState().unwatchAll(netUi)
engine.event.off("net:busy", onBusy)
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

### 位移矢量箭头 arrow

先加载位移，再开箭头（未传 `type` 时 normal / eph / forecast 一起开）：

```js
const viewerId = engine.getAllViewerState()[0].id

engine.event.emit("viewer:state", {
  viewerId,
  state: {
    targetAttr: "displacements",
    targetAxis: 3,
    updateAttr: Date.now(),
  },
})

engine.event.emit("actor:global:state", {
  viewerId,
  state: {
    arrow: true,
    arrowMode: "resultant", // 或 "xyz"
    arrowScale: 1,
    arrowAspect: 0.45,
    arrowStride: 4,
  },
})

// 只开某一层：type: "normal" | "eph" | "forecast"
// engine.event.emit("actor:global:state", {
//   viewerId, type: "eph", state: { arrow: true, arrowStride: 2 },
// })
```

### 应力链 stressChainMode

先切到应力场，再开模式；**Ctrl+左键**点选生成链：

```js
const viewerId = engine.getAllViewerState()[0].id

engine.event.emit("viewer:state", {
  viewerId,
  state: {
    targetAttr: "stress",
    targetAxis: 6, // 云图轴；链用 stressChainAxis
    updateAttr: Date.now(),
    stressChainMode: true,
    stressChainAxis: 6, // p1；7=p2，8=p3
    stressChainSeedPolicy: "nearHit", // 或 "maxAbs"
    stressChainShowArrows: true,
    stressChainMaxCells: 200,
  },
})

// 关闭（清链，恢复原先 rayMode）
engine.event.emit("viewer:state", {
  viewerId,
  state: { stressChainMode: false, stressChainSeed: null },
})

// 不拾取、直接指定种子
engine.event.emit("viewer:state", {
  viewerId,
  state: { stressChainSeed: { vertexIndex: 1234, axis: 6 } },
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
  viewerId: engine.getAllViewerState()[0].id,
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
