---
title: "Cesium 路线导航教程"
description: "详解 Cesium.js 路线导航：基于 WebGL 实现「路线导航」可视化效果，附完整可运行源码，涵盖 Cesium 等关键实现，附完整源码与在线 Demo，适合 Cesium 应用 学习与二次开发。"
head:
  - - meta
    - name: keywords
      content: "Cesium.js,路线导航,WebGL,源码,教程,在线案例,Cesium,Cesium Entity,实体"
outline: deep
---

### 路线导航 · *Route Nav* · [▶ 在线运行案例](https://z2586300277.github.io/three-cesium-examples/#/codeMirror?navigation=CesiumJS&classify=applyExample&id=routeNavigation)
- **案例合集：** [三维可视化功能案例（threehub.cn）](https://threehub.cn)

- **开源仓库github地址：** https://github.com/z2586300277/three-cesium-examples

- **400个案例代码: ** [网盘链接](https://pan.quark.cn/s/201da5c82fec)

![路线导航](https://z2586300277.github.io/three-cesium-examples/cesiumExamples/basic/routeNavigation.jpg)

## 你将学到什么

- Cesium Entity 高层实体 API

## 效果说明

本案例演示 **路线导航** 效果：基于 WebGL 实现「路线导航」可视化效果，附完整可运行源码；核心用到 Cesium。建议先打开文首在线案例查看动态画面，再对照下方源码逐步理解。

## 核心概念
- **Viewer** 聚合 Scene、Camera、Clock 与渲染循环，是 Cesium 应用入口。
- **Entity** 面向点线面/模型/标签的高层 API；与 Primitive 相比更适合交互与属性驱动。
- 阅读下方完整源码时，建议从 `init` / `load` / `animate` 三条主线入手，再深入 shader 与工具函数。

## 实现步骤
1. 创建 Viewer，配置地形/影像（若案例需要）并设置初始相机
2. 在 `requestAnimationFrame` 循环中更新状态并 render（Cesium 为 `viewer.render` 或自动渲染）

## 代码要点

```js
import * as Cesium from 'cesium'

const box = document.getElementById('box')

const viewer = new Cesium.Viewer(box, {

    animation: false,//是否创建动画小器件，左下角仪表    

    baseLayerPicker: false,//是否显示图层选择器，右上角图层选择按钮

    baseLayer: false, // 不显示默认图层

    fullscreenButton: false,//是否显示全屏按钮，右下角全屏选择按钮

    timeline: false,//是否显示时间轴    

    infoBox: false,//是否显示信息框   

})

const layer = Cesium.ImageryLayer.fromProviderAsync(Cesium.ArcGisMapServerImageryProvider.fromUrl(GLOBAL_CONFIG.getLayerUrl()))

viewer.imageryLayers.add(layer)

const list = [
    {
        "longitude": 116.3877535895933,
        "latitude": 39.917986883763334,
        "height": 5
    },
    {
        "longitude": 116.3879258383737,
        "latitude": 39.91794008705796,
        "height": 5
    },
    {
        "longitude": 116.38861928968578,
        "latitude": 39.91781284391525,
        "height": 5
    },
    {
        "longitude": 116.38869191428421,
        "latitude": 39.91818495388228,
        "height": 5
    }
]

const cartesianPoints = list.map(item => {
    const { longitude, latitude, height } = item
    return Cesium.Cartesian3.fromDegrees(longitude, latitude, height)
})

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

viewer.entities.add({
    name: '路线',
    polyline: {
        positions: interpolatedPoints,
        width: 1,
        material: Cesium.Color.RED
    }
})

// 添加无人机
const entity = viewer.entities.add({
    position: Cesium.Cartesian3.fromDegrees(list[0].longitude, list[0].latitude, 7),
    model: { uri: FILE_HOST + '/models/uav.glb' },
    viewFrom: new Cesium.Cartesian3(0, -20, 10) // 设置第三人称视角偏移（后方20米，上方10米）
})
viewer.trackedEntity = entity // 设置相机跟随飞机

// 动画
const start = Cesium.JulianDate.fromDate(new Date())   // 设置起始时间
const speedFactor = 50 // 设置速度因子，值越大速度越快
let stop = Cesium.JulianDate.addSeconds(start, interpolatedPoints.length / speedFactor, new Cesium.JulianDate())

function setProperty(t1, t2) {
    const property = new Cesium.SampledPositionProperty()
    for (let i = 0; i < interpolatedPoints.length; i++)  property.addSample(Cesium.JulianDate.addSeconds(t1, i / speedFactor, new Cesium.JulianDate()), interpolatedPoints[i])
    entity.position = property
    entity.orientation = new Cesium.VelocityOrientationProperty(property)
    entity.availability = new Cesium.TimeIntervalCollection([new Cesium.TimeInterval({ start: t1, stop: t2 })])
}

setProperty(start, stop)

// 监听飞机的位置属性，当飞机到达终点时重新设置位置属性
viewer.clock.onTick.addEventListener(function (clock) {
    if (Cesium.JulianDate.compare(clock.currentTime, stop) >= 0) {
        const newStart = Cesium.JulianDate.clone(stop);
        stop = Cesium.JulianDate.addSeconds(newStart, interpolatedPoints.length / speedFactor, new Cesium.JulianDate());
        setProperty(newStart, stop)
    }
})

viewer.clock.shouldAnimate = true // 开始动画
```

完整源码：[GitHub](https://github.com/z2586300277/three-cesium-examples/blob/dev/cesiumExamples/basic/routeNavigation.js)

## 小结
- 本文提供 **路线导航** 完整 Cesium.js 源码与在线 Demo，建议先运行案例再改 uniform/参数做二次实验
- 更多 Cesium.js 实战案例见 [three-cesium-examples 合集](https://threehub.cn) 与 [GitHub 开源仓库](https://github.com/z2586300277/three-cesium-examples)

