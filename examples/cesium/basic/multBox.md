---
title: "Cesium 海量 Box教程"
description: "详解 Cesium.js 海量 Box：基于 WebGL 实现「海量 Box」可视化效果，附完整可运行源码，涵盖 Viewer、Scene、Camera 等关键实现，附完整源码与在线 Demo，适合 Cesium 基础 学习与二次开发。"
head:
  - - meta
    - name: keywords
      content: "Cesium.js,海量 Box,WebGL,源码,教程,在线案例,Cesium"
outline: deep
---

### 海量 Box · *Mass Boxes* · [▶ 在线运行案例](https://z2586300277.github.io/three-cesium-examples/#/codeMirror?navigation=CesiumJS&classify=basic&id=multBox)
- **案例合集：** [三维可视化功能案例（threehub.cn）](https://threehub.cn)

- **开源仓库github地址：** https://github.com/z2586300277/three-cesium-examples

- **400个案例代码: ** [网盘链接](https://pan.quark.cn/s/201da5c82fec)

![海量 Box](https://z2586300277.github.io/three-cesium-examples/cesiumExamples/basic/multBox.jpg)

## 你将学到什么

- **BoxGeometry.fromDimensions** 创建立方体
- **Transforms.eastNorthUpToFixedFrame** 贴地定向
- 10000 个 instance 单次 Primitive 渲染

## 效果说明

全球随机 10000 个半透明红色长方体，尺寸约 40km×30km×500km（夸张高度便于远距可见）。

## 核心概念

```js
const position = Cesium.Cartesian3.fromDegrees(longitude, latitude, 0);
const dimensions = new Cesium.Cartesian3(40000.0, 30000.0, 500000.0);

instances.push(new Cesium.GeometryInstance({
    geometry: Cesium.BoxGeometry.fromDimensions({
        vertexFormat: Cesium.PerInstanceColorAppearance.VERTEX_FORMAT,
        dimensions,
    }),
    modelMatrix: Cesium.Transforms.eastNorthUpToFixedFrame(position),
    attributes: {
        color: Cesium.ColorGeometryInstanceAttribute.fromColor(
            Cesium.Color.RED.withAlpha(0.2)
        ),
    },
}));

viewer.scene.primitives.add(new Cesium.Primitive({
    geometryInstances: instances,
    appearance: new Cesium.PerInstanceColorAppearance(),
}));
```

**eastNorthUpToFixedFrame**：以该经纬点为原点，东-北-天为局部轴，box 竖直「长」向天顶。

## 实现步骤

1. 循环 10000 次随机经纬
2. push GeometryInstance 到数组
3. 一个 Primitive 提交全部 instance

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

const instances = [];

for (let i = 0; i < 10000; i++) {

    const longitude = Math.random() * 360 - 180;

    const latitude = Math.random() * 180 - 90;

    const position = Cesium.Cartesian3.fromDegrees(longitude, latitude, 0);

    const dimensions = new Cesium.Cartesian3(40000.0, 30000.0, 500000.0);

    const color = Cesium.Color.RED.withAlpha(0.2);

    instances.push(new Cesium.GeometryInstance({

        geometry: new Cesium.BoxGeometry.fromDimensions({

            vertexFormat: Cesium.PerInstanceColorAppearance.VERTEX_FORMAT,

            dimensions: dimensions

        }),

        modelMatrix: Cesium.Transforms.eastNorthUpToFixedFrame(position),

        attributes: {

            color: Cesium.ColorGeometryInstanceAttribute.fromColor(color)

        }

    }));

}

viewer.scene.primitives.add(new Cesium.Primitive({

    geometryInstances: instances,

    appearance: new Cesium.PerInstanceColorAppearance()

}));
```

完整源码：[GitHub](https://github.com/z2586300277/three-cesium-examples/blob/dev/cesiumExamples/basic/multBox.js)

## 小结

- 本文提供 **海量 Box** 完整 Cesium.js 源码与在线 Demo，建议先运行案例再改 uniform/参数做二次实验
- 更多 Cesium.js 实战案例见 [three-cesium-examples 合集](https://threehub.cn) 与 [GitHub 开源仓库](https://github.com/z2586300277/three-cesium-examples)

