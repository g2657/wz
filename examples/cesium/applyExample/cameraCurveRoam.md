---
title: "Cesium 曲线漫游教程"
description: "详解 Cesium.js 曲线漫游：加载倾斜摄影或人工 3D Tiles 白膜并自动定位相机，涵盖 Cesium3DTileset、Cesium、3D 等关键实现，附完整源码与在线 Demo，适合 Cesium 应用 学习与二次开发。"
head:
  - - meta
    - name: keywords
      content: "Cesium.js,曲线漫游,WebGL,源码,教程,在线案例,Cesium,3D Tiles,倾斜摄影,Cesium Entity,实体"
outline: deep
---

### 曲线漫游 · *Curve Roam* · [▶ 在线运行案例](https://z2586300277.github.io/three-cesium-examples/#/codeMirror?navigation=CesiumJS&classify=applyExample&id=cameraCurveRoam)
- **案例合集：** [三维可视化功能案例（threehub.cn）](https://threehub.cn)

- **开源仓库github地址：** https://github.com/z2586300277/three-cesium-examples

- **400个案例代码: ** [网盘链接](https://pan.quark.cn/s/201da5c82fec)

![曲线漫游](https://z2586300277.github.io/three-cesium-examples/cesiumExamples/basic/cameraCurveRoam.jpg)

## 你将学到什么
- Cesium3DTileset 加载 3D Tiles 倾斜摄影
- Cesium Entity 高层实体 API
- 3D Tiles 流式 LOD 场景
- `requestAnimationFrame` 渲染循环与 `resize` 自适应

## 效果说明

本案例演示 **曲线漫游** 效果：加载倾斜摄影或人工 3D Tiles 白膜并自动定位相机；核心用到 Cesium3DTileset、Cesium、3D。建议先打开文首在线案例查看动态画面，再对照下方源码逐步理解。

## 核心概念
- **Viewer** 聚合 Scene、Camera、Clock 与渲染循环，是 Cesium 应用入口。
- **Cesium3DTileset** 流式加载 LOD 瓦片，适合城市倾斜摄影；常用 `viewer.zoomTo(tileset)` 或 `viewBoundingSphere` 定位。
- **Entity** 面向点线面/模型/标签的高层 API；与 Primitive 相比更适合交互与属性驱动。

## 实现步骤
1. 创建 Viewer，配置地形/影像（若案例需要）并设置初始相机
2. 异步加载模型 / 3D Tiles / GeoJSON 等资源并加入 scene 或 entities
3. 在 `requestAnimationFrame` 循环中更新状态并 render（Cesium 为 `viewer.render` 或自动渲染）

## 代码要点

```js
import * as Cesium from 'cesium'
import * as dat from 'dat.gui'

const box = document.getElementById('box')

const viewer = new Cesium.Viewer(box, {

    animation: false,//是否创建动画小器件，左下角仪表    

    baseLayerPicker: false,//是否显示图层选择器，右上角图层选择按钮

    baseLayer: Cesium.ImageryLayer.fromProviderAsync(

        Cesium.ArcGisMapServerImageryProvider.fromUrl(GLOBAL_CONFIG.getLayerUrl())
        
    ),

    fullscreenButton: false,//是否显示全屏按钮，右下角全屏选择按钮

    timeline: false,//是否显示时间轴    

    infoBox: false,//是否显示信息框   

})

const tileset = await Cesium.Cesium3DTileset.fromUrl(FILE_HOST + '3dtiles/house/tileset.json')

viewer.scene.primitives.add(tileset)

viewer.flyTo(tileset);

// 经纬度 高度
const list = [
    [121.47857119758791, 29.79125471709178, 16.455626729366145],
    [121.47888991686754, 29.79121144438129, 16.43945469952735],
    [121.4793563626501, 29.79115700403782, 16.500224202937577],
    [121.47959615722343, 29.791255451852457, 19.638183861734586],
    [121.4799150177678, 29.791206202923174, 19.709391069654206],
    [121.48017710101357, 29.791136574675704, 19.707021008968702],
    [121.48024839194412, 29.791355774130647, 16.498928502606283],
    [121.47938339181717, 29.791564467242317, 17.821189061225503],
    [121.4788135287918, 29.79168578787095, 19.667240655082814],
    [121.47901177358922, 29.791479635806983, 19.655729311572056]
]

const cartesianPoints = list.map(item => Cesium.Cartesian3.fromDegrees(item[0], item[1], item[2] + 3))

// CatmullRomSpline 插值
const catmullRomSpline = new Cesium.CatmullRomSpline({
    points: cartesianPoints,
    times: cartesianPoints.map((_, index) => index / (cartesianPoints.length - 1))
})

const numPoints = 1000 // 插值点数量
const interpolatedPoints = []
for (let i = 0; i < numPoints; i++) {
    const t = i / (numPoints - 1)
    const point = catmullRomSpline.evaluate(t)
    interpolatedPoints.push(point)
}

// 示例线路
viewer.entities.add({
    name: '路线',
    polyline: {
        positions: interpolatedPoints,
        width: 1,
        material: Cesium.Color.YELLOW
    }
})

// 相机第一人称漫游
const flyControls = {
    active: false,
    speed: 2.5,
    position: 0,
    loop: true,
    start() { this.active = true; animate(); },
    stop() { this.active = false; },
    reset() { this.position = 0; }
}

// 简化的GUI控制面板
const gui = new dat.GUI()
gui.add(flyControls, 'speed').name('速度')
gui.add(flyControls, 'loop').name('循环')
gui.add(flyControls, 'start').name('开始')
gui.add(flyControls, 'stop').name('停止')
gui.add(flyControls, 'reset').name('重置')

// 动画函数
function animate() {
    if (!flyControls.active) return;

    flyControls.position += flyControls.speed / 10000;

    // 处理循环或结束
    if (flyControls.position >= 1) {
        if (flyControls.loop) flyControls.position = 0;
        else return flyControls.active = false
    }

    // 计算当前位置和下一位置
    const currentPos = catmullRomSpline.evaluate(flyControls.position);
    const nextT = flyControls.position + 0.01 >= 1 ? 0.01 : flyControls.position + 0.01;
    const nextPos = catmullRomSpline.evaluate(nextT);

    // 设置相机朝向
    const direction = Cesium.Cartesian3.subtract(nextPos, currentPos, new Cesium.Cartesian3())
    if (Cesium.Cartesian3.magnitude(direction) < 0.01) return requestAnimationFrame(animate);
    Cesium.Cartesian3.normalize(direction, direction)

    // 计算上方向(地球表面法向量)
    const up = Cesium.Cartesian3.normalize(Cesium.Ellipsoid.WGS84.geodeticSurfaceNormal(currentPos, new Cesium.Cartesian3()), new Cesium.Cartesian3())

    viewer.camera.setView({ destination: currentPos, orientation: { direction, up } })

    requestAnimationFrame(animate)

}
```

完整源码：[GitHub](https://github.com/z2586300277/three-cesium-examples/blob/dev/cesiumExamples/basic/cameraCurveRoam.js)

## 小结
- 本文提供 **曲线漫游** 完整 Cesium.js 源码与在线 Demo，建议先运行案例再改 uniform/参数做二次实验
- 更多 Cesium.js 实战案例见 [three-cesium-examples 合集](https://threehub.cn) 与 [GitHub 开源仓库](https://github.com/z2586300277/three-cesium-examples)

