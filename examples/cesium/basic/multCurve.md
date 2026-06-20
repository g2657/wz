---
title: "Cesium 海量曲线教程"
description: "详解 Cesium.js 海量曲线：基于 WebGL 实现「海量曲线」可视化效果，附完整可运行源码，涵盖 Viewer、Scene、Camera 等关键实现，附完整源码与在线 Demo，适合 Cesium 基础 学习与二次开发。"
head:
  - - meta
    - name: keywords
      content: "Cesium.js,海量曲线,WebGL,源码,教程,在线案例,Cesium"
outline: deep
---

### 海量曲线 · *Mass Curves* · [▶ 在线运行案例](https://z2586300277.github.io/three-cesium-examples/#/codeMirror?navigation=CesiumJS&classify=basic&id=multCurve)
- **案例合集：** [三维可视化功能案例（threehub.cn）](https://threehub.cn)

- **开源仓库github地址：** https://github.com/z2586300277/three-cesium-examples

- **400个案例代码: ** [网盘链接](https://pan.quark.cn/s/201da5c82fec)

![海量曲线](https://z2586300277.github.io/three-cesium-examples/cesiumExamples/basic/multCurve.jpg)

## 你将学到什么

- **CatmullRomSpline** 过控制点生成平滑曲线
- `multiplier` 控制插值细分密度
- 300+ 条随机曲线合批为单个 Primitive

## 效果说明

先用 5 个中国城市坐标画一条示范曲线，再随机 300 条跨洋曲线，颜色随机、半透明。

## 核心概念

### 样条插值

```js
const spline = new Cesium.CatmullRomSpline({
    times: [0, 0.25, 0.5, 0.75, 1],  // 归一化参数
    points: cartesianPoints,          // Cartesian3 控制点
});

for (let i = 0; i < numOfPoints; i++) {
    const time = i / (numOfPoints - 1);
    curvePoints.push(spline.evaluate(time));
}
```

`numOfPoints = 控制点数 × multiplier`，multiplier 越大曲线越 smooth、顶点越多。

### 与直线飞线区别

| 方式 | 路径 |
|------|------|
| 直线 | 控制点依次相连 |
| **Catmull-Rom** | 过所有控制点的平滑弧线，适合迁徙/物流可视化 |

## 实现步骤

1. `setCurveCollection` 封装 `generateCurvePoints` + instance 收集
2. 添加京沪广深杭 5 点示范曲线
3. 循环 300 次随机 5 控制点（10 个度数）生成曲线
4. 一个 `Primitive` + `PolylineColorAppearance` 提交

## 代码要点

```js
import * as Cesium from 'cesium'

const box = document.getElementById('box')

const viewer = new Cesium.Viewer(box, {

    animation: false,//是否创建动画小器件，左下角仪表    

    baseLayerPicker: false,//是否显示图层选择器，右上角图层选择按钮

    baseLayer: Cesium.ImageryLayer.fromProviderAsync(Cesium.ArcGisMapServerImageryProvider.fromUrl('https://server.arcgisonline.com/arcgis/rest/services/World_Imagery/MapServer')),

    fullscreenButton: false,//是否显示全屏按钮，右下角全屏选择按钮

    timeline: false,//是否显示时间轴    

    infoBox: false,//是否显示信息框   

})


// 经纬度坐标5个点
const points = [116.405285, 39.904989, 121.472644, 31.231706, 113.280637, 23.125178, 114.057868, 22.543099, 120.153576, 30.287459]

const getColor = () => '#' + Math.floor(Math.random() * 0xffffff).toString(16).padEnd(6, '0') // 随机16进制颜色

setCurveCollection(viewer, curveCollection => {

    curveCollection.add({ positions: points, color: getColor(), width: 2, opacity: 0.5, id: 'curve1', multiplier: 1 }) // 添加一条曲线

    // 随机生成 300 个曲线
    for (let i = 0; i < 300; i++) {

        const positions = Array.from({ length: 10 }, () => Math.random() * 360 - 180).reduce((acc, cur) => acc.concat(cur), [])

        curveCollection.add({ positions, color: getColor(), width: 2, opacity: 0.5, id: i, multiplier: 20 })

    }

})

/* 创建曲线合集 */
function setCurveCollection(viewer, callback) {

    /* 曲线算法 */
    function generateCurvePoints(flattenedPoints, multiplier = 30) {

        const numOfPoints = flattenedPoints.length / 2 * multiplier

        // 将一维数组转换为二维数组
        const points = [];

        for (let i = 0; i < flattenedPoints.length; i += 2) {

            points.push([flattenedPoints[i], flattenedPoints[i + 1]])

        }

        const times = points.map((_, index) => index / (points.length - 1))

        const cartesianPoints = points.map(point => Cesium.Cartesian3.fromDegrees(point[0], point[1]))

        const spline = new Cesium.CatmullRomSpline({

            times: times,

            points: cartesianPoints

        });

        const curvePoints = [];

        for (let i = 0; i < numOfPoints; i++) {

            const time = i / (numOfPoints - 1)

            curvePoints.push(spline.evaluate(time))

        }

        return curvePoints;

    }

    const curveCollection = {

        instances: [],

        add({ positions, color = '#fff', id = '', width = 1.0, opacity = 1, multiplier = 10 }) {

            if (!positions) return

            this.instances.push(new Cesium.GeometryInstance({

                geometry: new Cesium.PolylineGeometry({

                    positions: generateCurvePoints(positions, multiplier),

                    width,

                    vertexFormat: Cesium.PolylineColorAppearance.VERTEX_FORMAT

                }),

                attributes: {

                    color: Cesium.ColorGeometryInstanceAttribute.fromColor(Cesium.Color.fromCssColorString(color).withAlpha(opacity))

                },

                id

            }))

        }

    }

    if (callback) callback(curveCollection)

    // 增加线集合到场景中
    viewer.scene.primitives.add(new Cesium.Primitive({

        geometryInstances: curveCollection.instances,

        appearance: new Cesium.PolylineColorAppearance()

    }))

}
```

完整源码：[GitHub](https://github.com/z2586300277/three-cesium-examples/blob/dev/cesiumExamples/basic/multCurve.js)

## 小结

- 本文提供 **海量曲线** 完整 Cesium.js 源码与在线 Demo，建议先运行案例再改 uniform/参数做二次实验
- 更多 Cesium.js 实战案例见 [three-cesium-examples 合集](https://threehub.cn) 与 [GitHub 开源仓库](https://github.com/z2586300277/three-cesium-examples)

