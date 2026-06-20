---
title: "Cesium 流动飞线运动教程"
description: "详解 Cesium.js 流动飞线运动：基于 WebGL 实现「流动飞线运动」可视化效果，附完整可运行源码，涵盖 Cesium 等关键实现，附完整源码与在线 Demo，适合 Cesium 应用 学习与二次开发。"
head:
  - - meta
    - name: keywords
      content: "Cesium.js,流动飞线运动,WebGL,源码,教程,在线案例,Cesium,Cesium Entity,实体"
outline: deep
---

### 流动飞线运动 · *Flowing Line* · [▶ 在线运行案例](https://z2586300277.github.io/three-cesium-examples/#/codeMirror?navigation=CesiumJS&classify=applyExample&id=flyLine)
- **案例合集：** [三维可视化功能案例（threehub.cn）](https://threehub.cn)

- **开源仓库github地址：** https://github.com/z2586300277/three-cesium-examples

- **400个案例代码: ** [网盘链接](https://pan.quark.cn/s/201da5c82fec)

![流动飞线运动](https://z2586300277.github.io/three-cesium-examples/cesiumExamples/basic/flyLine.jpg)

## 你将学到什么

- Cesium Entity 高层实体 API

## 效果说明

本案例演示 **流动飞线运动** 效果：基于 WebGL 实现「流动飞线运动」可视化效果，附完整可运行源码；核心用到 Cesium。建议先打开文首在线案例查看动态画面，再对照下方源码逐步理解。

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
import { Color, defined, Event, Material, Property } from 'cesium'

const box = document.getElementById('box')

const viewer = new Cesium.Viewer(box, {

    animation: false,//是否创建动画小器件，左下角仪表    

    baseLayerPicker: false,//是否显示图层选择器，右上角图层选择按钮

    baseLayer: Cesium.ImageryLayer.fromProviderAsync(Cesium.ArcGisMapServerImageryProvider.fromUrl(GLOBAL_CONFIG.getLayerUrl())),

    fullscreenButton: false,//是否显示全屏按钮，右下角全屏选择按钮

    timeline: false,//是否显示时间轴    

    infoBox: false,//是否显示信息框   

})

viewer._cesiumWidget._creditContainer.style.display = "none"

viewer.clock.shouldAnimate = true

//定位北京
viewer.camera.flyTo({

    destination: Cesium.Cartesian3.fromDegrees(116.41, 36.91, 10000000),

    orientation: {

        heading: Cesium.Math.toRadians(0),

        pitch: Cesium.Math.toRadians(-90),

        roll: 0

    }

})

/* 飞线材质类 */
class PolylineTrailLinkMaterialProperty {

    constructor(image, color = Color.WHITE, duration = 1000) {

        this._definitionChanged = new Event()

        this._color = undefined

        this.color = color

        this.duration = duration

        this._time = new Date().getTime()

        this.image = image

        Material._materialCache.addMaterial('PolylineTrailLink', {

            fabric: {

                type: 'PolylineTrailLink',

                uniforms: {

                    color: color.withAlpha(1.0),

                    image: image,

                    time: 0

                },

                source: `
                    czm_material czm_getMaterial(czm_materialInput materialInput) 
                    {
                    czm_material material = czm_getDefaultMaterial(materialInput);
                    vec2 st = materialInput.st;
                    vec4 sampledColor = texture(image, vec2(fract(3.0*st.s - time), st.t));
                    material.alpha = sampledColor.a * color.a;
                    material.diffuse = (sampledColor.rgb + color.rgb) / 2.0;
                    return material;
                }`

            },

            translucent: () => true

        })

    }

    get isConstant() { return false }

    get definitionChanged() { return this._definitionChanged }

    getType(_) { return 'PolylineTrailLink' }

    getValue(time, result) {

        if (!defined(result)) result = {}

        result.color = Property.getValueOrClonedDefault(this._color, time, Color.WHITE, result.color)

        result.image = this.image

        result.time = (new Date().getTime() - this._time) % this.duration / this.duration

        return result

    }

    equals(other) { return this === other || Property.equals(this._color, other._color) }

}

// 生成一组飞线动画
[
    [[116.41, 36.91], [130.40, 45.39]],
    [[116.41, 36.91], [114.11, 39.44]],
    [[116.41, 36.91], [109.62, 25.72]],
    [[116.41, 36.91], [121.48, 31.22]],
    [[116.41, 36.91], [13.78, 12.31]],
    [[116.41, 36.91], [74.12, 33.50]],
].forEach(([p1, p2]) => createPlaneCurve(p1, p2))

// 组合
function createPlaneCurve(p1, p2) {

    const { curvePoints } = getGenerateCurve(p1, p2, { maxHeight: 100000 })

    setEntityAnimate(viewer, viewer.entities.add({

        model: {

            uri: FILE_HOST + 'models/glb/plane.glb',

            minimumPixelSize: 40,

            maximumScale: 100

        }

    }), curvePoints)

    viewer.entities.add({

        polyline: {

            positions: curvePoints,

            width: 8,

            material: new PolylineTrailLinkMaterialProperty(FILE_HOST + 'images/channels/line.webp', Cesium.Color.RED, 2000)

        }

    })

}

/* 生成曲线 */
function getGenerateCurve(start, end, params = {}) {

    const [startLongitude, startLatitude] = start

    const [endLongitude, endLatitude] = end

    const startCartographic = Cesium.Cartographic.fromDegrees(startLongitude, startLatitude)

    const endCartographic = Cesium.Cartographic.fromDegrees(endLongitude, endLatitude)

    const geodesic = new Cesium.EllipsoidGeodesic(startCartographic, endCartographic)

    const curvePoints = []

    for (let t = 0; t <= 1; t += (params.step || 0.01)) {

        const pointCartographic = geodesic.interpolateUsingFraction(t)

        pointCartographic.height = (params.maxHeight || 400000) * Math.sin(Math.PI * t)

        const pointCartesian = Cesium.Cartographic.toCartesian(pointCartographic)

        curvePoints.push(pointCartesian)

    }

    endCartographic.height = 0

    const endPointCartesian = Cesium.Cartographic.toCartesian(endCartographic)

    curvePoints.push(endPointCartesian)

    function getCurvePointAtTime(t) {

        const pointCartographic = geodesic.interpolateUsingFraction(t)

        pointCartographic.height = (params.maxHeight || 400000) * Math.sin(Math.PI * t)

        return Cesium.Cartographic.toCartesian(pointCartographic)

    }

    return { curvePoints, getCurvePointAtTime }

}

/* 飞行动画 */
function setEntityAnimate(viewer, entity, curvePoints, params = {}) {

    const start = Cesium.JulianDate.fromDate(new Date())   // 设置起始时间

    const speedFactor = params.speed || 20; // 增大这个值会让飞机飞得更快，减小这个值会让飞机飞得更慢

    let stop = Cesium.JulianDate.addSeconds(start, curvePoints.length / speedFactor, new Cesium.JulianDate())

    function setProperty(t1, t2) {

        const property = new Cesium.SampledPositionProperty()

        for (let i = 0; i < curvePoints.length; i++)  property.addSample(Cesium.JulianDate.addSeconds(t1, i / speedFactor, new Cesium.JulianDate()), curvePoints[i])

        entity.position = property

        entity.orientation = new Cesium.VelocityOrientationProperty(property)

        entity.availability = new Cesium.TimeIntervalCollection([new Cesium.TimeInterval({ start: t1, stop: t2 })])

    }

    setProperty(start, stop)

    // 监听飞机的位置属性，当飞机到达终点时重新设置位置属性
    viewer.clock.onTick.addEventListener(function (clock) {

        if (Cesium.JulianDate.compare(clock.currentTime, stop) >= 0) {

            const newStart = Cesium.JulianDate.clone(stop);

            stop = Cesium.JulianDate.addSeconds(newStart, curvePoints.length / speedFactor, new Cesium.JulianDate());

            setProperty(newStart, stop)

        }

    })

}
```

完整源码：[GitHub](https://github.com/z2586300277/three-cesium-examples/blob/dev/cesiumExamples/basic/flyLine.js)

## 小结
- 本文提供 **流动飞线运动** 完整 Cesium.js 源码与在线 Demo，建议先运行案例再改 uniform/参数做二次实验
- 更多 Cesium.js 实战案例见 [three-cesium-examples 合集](https://threehub.cn) 与 [GitHub 开源仓库](https://github.com/z2586300277/three-cesium-examples)

