---
title: "Cesium 雷达扫描教程"
description: "详解 Cesium.js 雷达扫描：基于 WebGL 实现「雷达扫描」可视化效果，附完整可运行源码，涵盖 Cesium 等关键实现，附完整源码与在线 Demo，适合 Cesium 特效 学习与二次开发。"
head:
  - - meta
    - name: keywords
      content: "Cesium.js,雷达扫描,WebGL,源码,教程,在线案例,Cesium,Cesium Entity,实体"
outline: deep
---

### 雷达扫描 · *Radar Scan* · [▶ 在线运行案例](https://z2586300277.github.io/three-cesium-examples/#/codeMirror?navigation=CesiumJS&classify=singleEffect&id=radar)
- **案例合集：** [三维可视化功能案例（threehub.cn）](https://threehub.cn)

- **开源仓库github地址：** https://github.com/z2586300277/three-cesium-examples

- **400个案例代码: ** [网盘链接](https://pan.quark.cn/s/201da5c82fec)

![雷达扫描](https://z2586300277.github.io/three-cesium-examples/cesiumExamples/effect/radar.jpg)

## 你将学到什么

- Cesium Entity 高层实体 API

## 效果说明

本案例演示 **雷达扫描** 效果：基于 WebGL 实现「雷达扫描」可视化效果，附完整可运行源码；核心用到 Cesium。建议先打开文首在线案例查看动态画面，再对照下方源码逐步理解。

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
    animation: false,
    baseLayerPicker: false,
    baseLayer: Cesium.ImageryLayer.fromProviderAsync(
        Cesium.ArcGisMapServerImageryProvider.fromUrl(
            'https://server.arcgisonline.com/arcgis/rest/services/World_Imagery/MapServer'
        )
    ),
    fullscreenButton: false,
    timeline: false,
    infoBox: false
})

// ================= RadarSolidScan 类实现 =================
class RadarSolidScan {
    constructor(options) {
        this.viewer = options.viewer
        this.id = options.id || 'radar'
        this.position = Cesium.Cartesian3.fromDegrees(...options.position, 0)
        this.longitude = options.position[0]
        this.latitude = options.position[1]
        this.shortwaveRange = options.shortwaveRange || 50000
        this.positionArr = []
        this.heading = 0
        this.tickListener = null
        this.addEntities()
        this.addPostRender()
    }

    addEntities() {
        this.entity = this.viewer.entities.add({
            id: this.id,
            position: this.position,
            wall: {
                positions: new Cesium.CallbackProperty(() => {
                    return Cesium.Cartesian3.fromDegreesArrayHeights(this.positionArr)
                }, false),
                material: Cesium.Color.fromCssColorString("#00dcff82"),
                distanceDisplayCondition: new Cesium.DistanceDisplayCondition(0.0, 10.5e6)
            },
            ellipsoid: {
                radii: new Cesium.Cartesian3(
                    this.shortwaveRange,
                    this.shortwaveRange,
                    this.shortwaveRange
                ),
                maximumCone: Cesium.Math.toRadians(90),
                material: Cesium.Color.fromCssColorString("#00dcff82"),
                outline: true,
                outlineColor: Cesium.Color.fromCssColorString("#00dcff82"),
                outlineWidth: 1,
                distanceDisplayCondition: new Cesium.DistanceDisplayCondition(0.0, 10.5e6)
            }
        })
    }

    addPostRender() {
        this.tickListener = this.viewer.clock.onTick.addEventListener(() => {
            this.heading += 1.0
            if (this.heading >= 360) this.heading = 0
            this.positionArr = this.calcPoints(
                this.longitude,
                this.latitude,
                this.shortwaveRange,
                this.heading
            )
        })
    }

    calcPoints(x1, y1, radius, heading) {
        const m = Cesium.Transforms.eastNorthUpToFixedFrame(
            Cesium.Cartesian3.fromDegrees(x1, y1)
        )
        const rad = Cesium.Math.toRadians(heading)
        const rx = radius * Math.cos(rad)
        const ry = radius * Math.sin(rad)
        const translation = Cesium.Cartesian3.fromElements(rx, ry, 0)
        const d = Cesium.Matrix4.multiplyByPoint(
            m,
            translation,
            new Cesium.Cartesian3()
        )
        const c = Cesium.Cartographic.fromCartesian(d)
        const x2 = Cesium.Math.toDegrees(c.longitude)
        const y2 = Cesium.Math.toDegrees(c.latitude)
        return this.computeCirclularFlight(x1, y1, x2, y2, 0, 90)
    }

    computeCirclularFlight(x1, y1, x2, y2, fx, angle) {
        let positionArr = []
        positionArr.push(x1, y1, 0)
        const radius = Cesium.Cartesian3.distance(
            Cesium.Cartesian3.fromDegrees(x1, y1),
            Cesium.Cartesian3.fromDegrees(x2, y2)
        )
        for (let i = fx; i <= fx + angle; i++) {
            const rad = Cesium.Math.toRadians(i)
            const h = radius * Math.sin(rad)
            const r = Math.cos(rad)
            const x = (x2 - x1) * r + x1
            const y = (y2 - y1) * r + y1
            positionArr.push(x, y, h)
        }
        return positionArr
    }

    clear() {
        if (this.tickListener) {
            this.tickListener()
            this.tickListener = null
        }
        if (this.viewer && this.id) {
            const entity = this.viewer.entities.getById(this.id)
            if (entity) {
                this.viewer.entities.remove(entity)
            }
        }
        this.positionArr = []
        this.heading = 0
    }

    destroy() {
        this.clear()
        this.viewer = null
    }

    stop() {
        if (this.tickListener) {
            this.tickListener()
            this.tickListener = null
        }
    }

    start() {
        if (!this.tickListener) {
            this.addPostRender()
        }
    }

    hide() {
        if (this.entity) this.entity.show = false
    }

    show() {
        if (this.entity) this.entity.show = true
    }
}

// ================= 使用示例 =================
const radar = new RadarSolidScan({
    viewer: viewer,
    id: 'radar1',
    position: [120, 36],
    shortwaveRange: 50000 // 50公里
})

viewer.flyTo(radar.entity)
```

完整源码：[GitHub](https://github.com/z2586300277/three-cesium-examples/blob/dev/cesiumExamples/effect/radar.js)

## 小结
- 本文提供 **雷达扫描** 完整 Cesium.js 源码与在线 Demo，建议先运行案例再改 uniform/参数做二次实验
- 更多 Cesium.js 实战案例见 [three-cesium-examples 合集](https://threehub.cn) 与 [GitHub 开源仓库](https://github.com/z2586300277/three-cesium-examples)

