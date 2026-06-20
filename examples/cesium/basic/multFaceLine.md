---
title: "Cesium 海量面线教程"
description: "详解 Cesium.js 海量面线：基于 WebGL 实现「海量面线」可视化效果，附完整可运行源码，涵盖 Viewer、Scene、Camera 等关键实现，附完整源码与在线 Demo，适合 Cesium 基础 学习与二次开发。"
head:
  - - meta
    - name: keywords
      content: "Cesium.js,海量面线,WebGL,源码,教程,在线案例,Cesium"
outline: deep
---

### 海量面线 · *Mass Polygons & Polylines* · [▶ 在线运行案例](https://z2586300277.github.io/three-cesium-examples/#/codeMirror?navigation=CesiumJS&classify=basic&id=multFaceLine)
- **案例合集：** [三维可视化功能案例（threehub.cn）](https://threehub.cn)

- **开源仓库github地址：** https://github.com/z2586300277/three-cesium-examples

- **400个案例代码: ** [网盘链接](https://pan.quark.cn/s/201da5c82fec)

![海量面线](https://z2586300277.github.io/three-cesium-examples/cesiumExamples/basic/multFaceLine.jpg)

## 你将学到什么

- **GeometryInstance** 收集大量面/线几何
- 单个 **Primitive** 一次 draw 多种 instance
- **PerInstanceColorAppearance** / **PolylineColorAppearance**

## 效果说明

随机生成 **10000** 个三角形面 + 对应边框线，红绿交替填充，白线描边。

## 核心概念

### 合批模式

```js
// 收集阶段
faceCollection.instances.push(new Cesium.GeometryInstance({
    geometry: new Cesium.PolygonGeometry({
        polygonHierarchy: new Cesium.PolygonHierarchy(
            Cesium.Cartesian3.fromDegreesArray(positions)
        ),
        height: 0,
        vertexFormat: Cesium.PerInstanceColorAppearance.VERTEX_FORMAT,
    }),
    attributes: {
        color: Cesium.ColorGeometryInstanceAttribute.fromColor(color),
    },
    id: 'face' + i,
}));

// 提交阶段 — 一个 Primitive 渲染全部面
viewer.scene.primitives.add(new Cesium.Primitive({
    geometryInstances: faceCollection.instances,
    appearance: new Cesium.PerInstanceColorAppearance({ closed: true }),
}));
```

线集合同理，用 `PolylineGeometry` + `PolylineColorAppearance`。

### positions 格式

本案例 `positions` 为 `[lon1, lat1, lon2, lat2, lon3, lat3]` 三角面三顶点（度）。

## 实现步骤

1. 封装 `faceCollection` / `lineCollection` 带 `add()` 方法 push instance
2. 循环 10000 次随机经纬生成三角
3. callback 结束后各 add 一个 Primitive

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

setFaceCollection(viewer, (faceCollection, lineCollection) => {

    for (var i = 0; i < 10000; i++) {

        var longitude = Math.random() * 360 - 180;

        var latitude = Math.random() * 180 - 90;

        var positions = [longitude, latitude, longitude + Math.random(), latitude, longitude, latitude + Math.random()];

        faceCollection.add({ positions, color: i % 2 == 0 ? 'red' : 'green', id: 'face' + i, opacity: 1 })

        lineCollection.add({ positions, color: '#fff', id: 'line' + i, width: 1.0, opacity: 0.5 })

    }

})

// 创建大量面和线段
function setFaceCollection(viewer, callback) {

    const lineCollection = {

        instances: [],

        add({ positions, color = '#fff', id = '', width = 1.0, opacity = 1 }) {

            if (!positions) return

            this.instances.push(new Cesium.GeometryInstance({

                geometry: new Cesium.PolylineGeometry({

                    positions: Cesium.Cartesian3.fromDegreesArray(positions),

                    width: width * 3,

                    vertexFormat: Cesium.PolylineColorAppearance.VERTEX_FORMAT

                }),

                attributes: {

                    color: Cesium.ColorGeometryInstanceAttribute.fromColor(Cesium.Color.fromCssColorString(color).withAlpha(opacity))

                },

                id

            }))

        }

    }

    const faceCollection = {

        instances: [],

        add({ positions, color = '#fff', id = '', opacity = 1 }) {

            if (!positions) return

            this.instances.push(new Cesium.GeometryInstance({

                geometry: new Cesium.PolygonGeometry({

                    polygonHierarchy: new Cesium.PolygonHierarchy(Cesium.Cartesian3.fromDegreesArray(positions)),

                    height: 0,

                    vertexFormat: Cesium.PerInstanceColorAppearance.VERTEX_FORMAT

                }),

                attributes: {

                    color: Cesium.ColorGeometryInstanceAttribute.fromColor(Cesium.Color.fromCssColorString(color).withAlpha(opacity))

                },

                id

            }))

        }

    }

    if (callback) callback(faceCollection, lineCollection)

    // 增加面集合到场景中
    viewer.scene.primitives.add(

        new Cesium.Primitive({

            geometryInstances: faceCollection.instances,

            appearance: new Cesium.PerInstanceColorAppearance({

                closed: true

            })

        })

    )

    // 增加线集合到场景中
    viewer.scene.primitives.add(new Cesium.Primitive({

        geometryInstances: lineCollection.instances,

        appearance: new Cesium.PolylineColorAppearance()

    }))

}
```

完整源码：[GitHub](https://github.com/z2586300277/three-cesium-examples/blob/dev/cesiumExamples/basic/multFaceLine.js)

## 小结

- 本文提供 **海量面线** 完整 Cesium.js 源码与在线 Demo，建议先运行案例再改 uniform/参数做二次实验
- 更多 Cesium.js 实战案例见 [three-cesium-examples 合集](https://threehub.cn) 与 [GitHub 开源仓库](https://github.com/z2586300277/three-cesium-examples)

