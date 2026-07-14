//状态
let STATES = {
    ROOT: {
        // 🔹 初始化配置（网络请求后设置）
        CONTAINER: 'container',               // HTMLElement 容器
        LEGEND_VISIBLE: 'legendVisible', // 是否显示图例
        SELECT_VIEWER_STATE: 'selectViewerState', // 当前选中的 viewer state
        VCOUNT: 'vCount',         // 渲染器数量
        LOCK_CAM: 'lockCam',      // 是否锁定相机联动
        SPRITE_IMG: 'spriteImg',  // 标签贴图 base64/url
        LEGEND_IMAGE: 'legendImage', // 图例图片 base64
    },
    VIEWER: {
        ID: 'id',
        CAMERA_MODE: "cameraMode",//{"perspective" 透视相机,"orthographic" 正交相机,
        CAMERA_POS:"cameraPos",
        CAMERA_LOOKAT:"cameraLookat",
        CAMERA_LOOK_AT_FIT_RATIO: "cameraLookAtFitRatio",//相机视角与模型视角的距离倍率 默认0.6
        AUTO_ROTATE:"autoRotate",

        RAY_MODE: "rayMode", // null|"" 关闭；"mesh" 体拾取；"cut" 切面拾取；"measuring" 测点拾取
        AXIS_STYLE: "axisStyle", //gid / blender
        AXIS_POSITION: "axisPosition",// left-top , right-top ,left-bottom  默认右下角
        AXIS_SIZE: "axisSize", //默认150

        MESH_NAME :"meshName",
        TARGET_ATTR:"targetAttr",//设置物理量名
        TARGET_ATTR_VERSION:"targetAttrVersion", //设置物理量版本（damfScenario 开启时不参与云图，走 damf 旁路）
        TARGET_MESH_VERSION:"targetMeshVersion", //设置浇筑版本（damf 打开时自动对齐 base_mesh_version）
        TARGET_STEP:"targetStep", //设置物理量小步
        TARGET_AXIS:"targetAxis", //设置选中的轴(x,y,z 等用下标 0/1/2)
        UPDATE_ATTR:"updateAttr", //更新 target* 后触发云图刷新（必填触发项）

        ALL_MESH:"allMesh",  //文件名或空 全坝段线框
        ALL_MESH_SHOW:"allMeshShow",  //显示隐藏全坝段线框(默认true)；viewer 级独立，各窗口可分别显隐
        /** 全量 damb 低透明叠层（overlayMesh），不是 damf 增量预演 */
        FORECAST:"forecast",
        /** 增量预演 damf 场景 id（上传产物 {id}.damf / {id}.damfc）；空字符串 = 关闭 */
        DAMF_SCENARIO:"damfScenario",

        COLOR_PALETTE:"colorPalette", //色板图  base64
        NUM_ARR:"numArr", //最大最小值显式数组  字符串数组 
        MESH_VERSIONS:"meshVersions", //有多少个浇筑版本
        MIN:"min", //当前显式内容的最大值
        MAX:"max", //当前显式内容的最小值
        LOCK_RANGE: "lockRange",//锁定最大最小值
        PRELOAD:"preload",//预加载列表 []

        HIGHLIGHT:"highLight", //高亮范围
        ENABLEHIGHLIGHT:"enableHighLight",
        /** 单元高亮：{ cellIds: number[] } */
        CELL_HIGHLIGHT:"cellHighlight",
        CUT:"cut", //random() = 按回车键 
        SHOWCLIPPLANE:"showClipPlane",//bool 材质裁剪开关
        SHOW_CLIP_PLANE_HELPER:"showClipPlaneHelper",//bool 仅裁剪辅助面显示
        CLIP_PLANE_NORMAL:"clipPlaneNormal",//切片方向
        CLIP_PLANE_CONSTANT:"clipPlaneConstant",//切片位置(-1000,1000)
        GRID:"grid",//全局网格线开关(默认false)
        OPACITY:"opacity",//viewer 级透明度 0–1，统一下发到各 actor
        USE_DEFAULT_COLOR:"useDefaultColor",//viewer 级默认色，统一下发到各 actor

        /** 测点数据 Point[] 或 { config?, measuring:[] }；显隐/透明度走 actorManager(type=measuring) */
        MEASURING:"measuring",

    },
    ACTOR: {
        EDGE: 'edge',         // 边框线 bool
        GRID: 'grid',             // 网格线 bool
        LIGHT: 'light',           // 发光 bool
        OPACITY: 'opacity',       // 透明度 0.0 - 1.0
        NAME: 'name',             // 名称 
        DISPLAY_MODE: 'displayMode', // 显示模式（point|line|surface)
        SHOW: 'show',             // 显示总开关 bool
        DROP: 'drop',// 删除 actor
        /** forecast 增量 Actor 专有：所属 damf scenario id（只读语义，由挂载写入） */
        SCENARIO_ID: 'scenarioId',

        USE_DEFAULT_COLOR: 'useDefaultColor', // 是否使用默认色 bool

        DISP_COEF:"dispCoef",//变形系数
        MAT_STYLE:"matStyle",//显示材料  值:normal|shader

        ISO_LINE_NUM:"isoLineNum", //等值线 分段数 0为关闭
        ISO_LINE_WIDTH:"isoLineWidth", //等值线 线宽 默认1.5

        COLOR_STEP_NUM:"colorStepNum", //色阶显式数 默认 5
        USE_COLOR_STEP:"usecolorStep", //是否启用色阶 bool
        REMAINING:"remaining", //是否显示全坝未浇筑部分
    },
}

/**
 * ┌─────────────────────────────────────────────────────────────────┐
 * │ 预演相关 state 区分                                              │
 * ├──────────────┬──────────────────────────────────────────────────┤
 * │ forecast     │ 全量 .damb 低透明叠层（overlayMesh），与主模型    │
 * │              │ 物理量版本无关，仅参考几何                         │
 * ├──────────────┼──────────────────────────────────────────────────┤
 * │ damfScenario │ 增量预演 id；加载 {id}.damf + 挂 forecast Actor │
 * │              │ normal=DB 几何 + damf res 前段                    │
 * │              │ forecast=damf 增量 mesh + damf res 后段         │
 * │              │ 云图不看 targetAttrVersion；测点仍走 WS           │
 * └──────────────┴──────────────────────────────────────────────────┘
 */

/**
 * 获取proxy方法:
 * engine.getRootState()
 * engine.getAllViewerState()//[proxy,proxy]Viewer代理对象数组
 * engine.getAllActorState('Gdphc4s52BjQT1mRObv77')// { [matId]: stateProxy }（仅 normal 材料）
 * engine.getAllActorState(viewerId, 'measuring')// { [name]: stateProxy }
 * engine.getAllActorState(viewerId, 'eph')// 切片 Eph 材料
 * 获取边框线状态 engine.getAllActorState(viewerId)[matId].edge
 * 获取测点显隐 engine.getAllActorState(viewerId, 'measuring')['H01XH02'].show
 *
 * damfScenario 增量 forecast Actor：与 normal 同结构 state（show/grid/edge/opacity…），
 * 目前未接入 actor:state 的 type 路由；需通过 actorManager._forecast 内部 Map 读写（后续可扩展 type:'forecast'）。
 */

/**
 * 监听state绑定方法
 * 
 * const eventHandlers={
 *  [STATES.ROOT.VCOUNT]: (payload) => { //监听这个,意味着修改渲染窗口数量以后
         //do something u need
    },
     ...
     ...
     ...
 * }
 * 
 * 
 * proxy.watch(this) //启动监听
 * 
 * 
 */




//事件
let EVENTS = {
    ROOT: {
        STATE: 'root:state' //******* 
    },
    VIEWER: {
        RESIZE: 'viewer:resize', //发送后重新计算canvas尺寸
        ID: 'viewer:id',
        CAMERAVIEWE:'viewer:camera:view', //发送, 切换camera位置
        CAMERASET:'viewer:camera:set',
        STATE:'viewer:state', //修改单个viewer state时使用
        RAY_CB:'viewer:ray:cb', // 订阅, ray模式返回数据
        RAY_ProLine:'viewer:ray:processline',
        SHOWMAX:'viewer:show:max', //发送,显示max标签
        SHOWMIN:'viewer:show:min', //发送,显示min标签
        CLEARMINMAX:'viewer:clear:minmax', //法相,清空标签
        GETMAXTAS:'viewer:max:tas', //获取温度影里最大值
        SENDMAXTAS:"viewer:max:res",//温度影里最大值回调
        SHOWINTEGRAL:"viewer:show:integral",
        HIDEINTEGRAL:"viewer:hide:integral",
        INITDONE:"viewer:init:done", //初始化完成 可以setMeshName了
        ALL_MESH_DONE:"viewer:allmesh:done", //全坝段线框 allMesh 加载完成
        CLIP_POS_CB:'viewer:clipPos:cb', // 订阅, 剖切平面数据坐标位置回调
    },
    ACTOR: {
        STATE: 'actor:state', //******* type 可选 normal|eph|measuring（forecast 待扩展）
        DONE:'actor:done'    //可订阅；payload.kind 可为 normal|forecast|…，forecast 时带 scenarioId
    },
    ACTOR_GLOBAL: {
        STATE: 'actor:global:state', //******* 
    },
    VIEWER_GLOBAL: {
        RESIZE: 'viewer:global:resize',
        ACTIVATE: 'viewer:global:activate',//切换viewer时会发送,带对象id
        INDEX: 'viewer:global:index',
        STATE: 'viewer:global:state', //******* 

    },
    SOME:{
        STATE:'some:state',
    },
    WS: {
        CONNECT: 'finite:ws:connect', //ws连接成功时发送
        CLOSE: 'finite:ws:close',
        ERROR: 'finite:ws:error',
    },
    DATA: { //你不一定用得到
        MESH_READY: 'finite:data:mesh_ready',
        VERTICES_UPDATED: 'finite:data:vertices_updated',
        SURFACE_UPDATED: 'finite:data:surface_updated',
        LINE_UPDATED: 'finite:data:line_updated',
        EDGE_UPDATED: 'finite:data:edge_updated',
        GRID_UPDATED: 'finite:data:grid_updated',
        POINT_UPDATED: 'finite:data:point_updated',
        FACE_UPDATED: 'finite:data:face_updated',
        ATTR_UPDATED: 'finite:data:attr_updated',
        CUT_READY: 'finite:data:cut_ready', //切片 bin 就绪（含 damfScenario 的 GET_CUT_DAMF 单包）
    },
}

/**
 * 事件监听绑定方法:
 *  在对象上声明 eventHandlers={} 属性
 * const eventHandlers = {
     [EVENTS.WS.CONNECT]: () => {  //使用事件上需要监听的字段当作key,value是一个箭头函数
        //do something u need
        //参数可能有可能没有
     },
     ...
     ...
     ...
   }
    event.watch(this) //this是需要监听的对象 

   发送事件方法:event.emit
   engine.event.emit(EVENTS.ROOT.STATE)
 */

/*************************
 * 修改state方法:
 * engine.event.emit(EVENTS.ROOT.STATE,{vCount:1}) 
 * engine.event.emit("root:state",{spriteImg：})
 * engine.event.emit(EVENTS.VIEWER.STATE,{...}) //单窗口，需带id
 * engine.event.emit("viewer:state",{id:'',state:{grid:false,edge:true,})
 * engine.event.emit(EVENTS.ACTOR.STATE,{...}) //单窗口但材料修改，需带id
 * engine.event.emit("actor:state",{viewer:'',actorid:'',state:{grid:false,edge:true,})
 * engine.event.emit(EVENTS.VIEWER_GLOBAL.STATE,{...}) //多窗口下
 * engine.event.emit("viewer:global:state",{id:['',''],state:{})
 * engine.event.emit(EVENTS.ACTOR_GLOBAL.STATE,{...}) //viewers传多个为多窗口多窗口的现因，需带viewer的id，不带表示全部//单窗口的所有actor
 * engine.event.emit("actor:global:state",{viewers:['',''],state:{grid:false,edge:true,})
 * 
 */

const defaultConfig = {
    container: [],
    axes: { size: '8rem', type: 'blender' },
    renderer: {},         // ThreeRender 配置
    wsUrl: "ws://sky.leve.vip:8888/jl_ws/",
    server:"http://sky.leve.vip:3000/",
    meshName:'test3',
    camPos:{x: 315.6295145760044, y: 284.775713880556, z: 247.455733574656},
    camTarget: {x: 40.152403153231, y: -3.3184214088838164, z: 124.09636532313567},
}

//示例

//同时切换浇筑步/物理场,允许只修改其中一部分,但是updateAttr总是需要修改
engine.event.emit("viewer:state",{
    viewerId:"meEnXC0yvSfZ10KxrLm-g",
    state:{
        targetAttr: "stress",
        targetAttrVersion:1767161923853,
        targetMeshVersion:1767161923853,
        targetStep:1,
        targetAxis:3,
        updateAttr:Math.random()
    }
})

// ---------- 增量预演 damfScenario（与 state.forecast 全量 damb 叠层无关）----------
// 前置：meshName 已加载且与 damf 内 mesh_name 一致；通常只需设 damfScenario
engine.event.emit("viewer:global:state",{
    state:{
        damfScenario:"your-scenario-id"  // 关闭: damfScenario:""
    }
})
// damf 模式下切换物理场：不需 targetAttrVersion；normal+forecast 共用 targetAttr/targetStep/targetAxis
engine.event.emit("viewer:state",{
    viewerId:"<viewerId>",
    state:{
        targetAttr:"stress",
        targetStep:"1",
        targetAxis:0,
        updateAttr:Date.now()
    }
})
// damf 体拾取 / 切面拾取
engine.event.emit("viewer:global:state",{ state:{ rayMode:"mesh" } })  // normal+forecast 外表面
engine.event.emit("viewer:global:state",{ state:{ rayMode:"cut" } })   // Eph 切面（需先切片）
// 切片（damfScenario 开启时内部 GET_CUT_DAMF，服务端双域切、单包下发）
engine.event.emit("viewer:global:state",{ state:{ cut:Math.random() } }) // 或 showClipPlane + clipPlaneConstant

//某个viewer下全部actor 改为点模式
engine.event.emit("actor:global:state",{
    viewerId:"axajqrpCGTVvRmILdCtrn",
    state:{
      displayMode:"point"
    }
})

//某个viewer下全部actor的网格显示关闭
engine.event.emit("actor:global:state",{
    viewerId:"r9MaKsZX2QrLq2rpMj7R0",
    state:{
        grid:false
    }
})

//某个viewer下全部actor的材料使用法线材料
engine.event.emit("actor:global:state",{
    viewerId:"kek02-YfADpTQQZHC5LzI",
    state:{
        matStyle:"normal"
    }
})

//某个viewer下 材料号为1的单个actor,关闭显示
engine.event.emit("actor:state",{
    viewerId:"r9MaKsZX2QrLq2rpMj7R0",
    matId:"1",
    state:{
        show:false
    }
})

//某个viewer下全部actor,开启色阶,分两段
engine.event.emit("actor:global:state",{
    viewerId:"WFn2yEpDthoY8U67znzTu",
    state:{
        usecolorStep: true,
        colorStepNum: 2,
    }
})

//单个 viewer 隐藏全坝段线框（不影响其他窗口）
engine.event.emit("viewer:state",{
    viewerId:"<viewerId>",
    state:{ allMeshShow: false }
})
//全部 viewer 同步显隐
engine.event.emit("viewer:global:state",{
    state:{ allMeshShow: true }
})
//指定若干 viewer 同步显隐
engine.event.emit("viewer:global:state",{
    id:["<viewerId1>","<viewerId2>"],
    state:{ allMeshShow: false }
})