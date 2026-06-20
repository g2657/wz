---
title: "Cesium 点击事件教程"
description: "详解 Cesium.js 点击事件：基于 WebGL 实现「点击事件」可视化效果，附完整可运行源码，涵盖 Cesium 等关键实现，附完整源码与在线 Demo，适合 Cesium 基础 学习与二次开发。"
head:
  - - meta
    - name: keywords
      content: "Cesium.js,点击事件,WebGL,源码,教程,在线案例,Cesium,Cesium Entity,实体"
outline: deep
---
### 点击事件 · *Click Event* · [▶ 在线运行案例](https://z2586300277.github.io/three-cesium-examples/#/codeMirror?navigation=CesiumJS&classify=basic&id=clickEvent)
- **案例合集：** [三维可视化功能案例（threehub.cn）](https://threehub.cn)

- **开源仓库github地址：** https://github.com/z2586300277/three-cesium-examples

- **400个案例代码: ** [网盘链接](https://pan.quark.cn/s/201da5c82fec)

![点击事件](https://z2586300277.github.io/three-cesium-examples/cesiumExamples/basic/clickEvent.jpg)

## 你将学到什么

- Cesium Viewer 初始化
- Cesium Entity 高层 API
- Cesium 鼠标拾取交互
- Cesium 影像图层
- Cesium 动态材质属性

## 效果说明

点击地球表面或 Entity 时，控制台输出 **经纬高**，或对 Entity 弹出提示。演示 Cesium 最基础的 **屏幕空间拾取** 流程。

> 基础功能 · Cesium.js

## 核心概念

- **Viewer** 封装地球、相机、图层；可关闭 animation/timeline 等 UI 精简界面。

- **Entity** 加点线面、模型、标签；适合业务对象与交互。

- **ScreenSpaceEventHandler** 监听点击；`scene.pick` 取 Entity，`pickPosition` 取地表坐标。

- **ImageryLayer** 叠加 XYZ/WMTS/ArcGIS 等底图，`imageryLayers.add/remove` 管理。

## 实现步骤

1. 初始化 `Cesium.Viewer` 与底图图层
2. 添加 Entity / Primitive / DataSource 等业务对象
3. 配置 ScreenSpaceEventHandler 交互
4. 按需 `camera.flyTo` 定位视角

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

const url = GLOBAL_CONFIG.getLayerUrl()
 
const layer = Cesium.ImageryLayer.fromProviderAsync(

    Cesium.ArcGisMapServerImageryProvider.fromUrl(url)

)

viewer.imageryLayers.add(layer)

// 添加点击事件监听器
viewer.screenSpaceEventHandler.setInputAction(function (event) {

    const object = viewer.scene.pick(event.position)

    const cartesian = viewer.scene.pickPosition(event.position)

    if (Cesium.defined(cartesian)) {

        const cartographic = Cesium.Cartographic.fromCartesian(cartesian)

        const longitude = Cesium.Math.toDegrees(cartographic.longitude)

        const latitude = Cesium.Math.toDegrees(cartographic.latitude)

        const height = cartographic.height

        console.log('经度：', longitude, '纬度：', latitude, '高度：', height)

    }

    if (Cesium.defined(object)) {

        const { id } = object

        alert('点击到的对象：' + id.name + '-----id：'+ id.id)
        
    }

}, Cesium.ScreenSpaceEventType.LEFT_CLICK)

// 视角定位到中国
viewer.camera.flyTo({

    destination: Cesium.Cartesian3.fromDegrees(116.39, 39.9, 10000000)

})

// 测试点
const point = viewer.entities.add({

    name: '测试点',

    id: '点-id',

    position: Cesium.Cartesian3.fromDegrees(116.39, 39.9),

    point: {

        pixelSize: 10,

        color: Cesium.Color.RED,

        outlineColor: Cesium.Color.WHITE,

        outlineWidth: 2

    }

})

// 测试面
const polygon = viewer.entities.add({

    name: '测试多边形',

    id: '多边形-id',

    polygon: {

        hierarchy: Cesium.Cartesian3.fromDegreesArray([

            90.38, 30.91,

            80.38, 30.89,

            100.4, 39.89,

            105.4, 39.91

        ]),

        material: Cesium.Color.RED.withAlpha(0.5)

    }

})

// 测试线
const polyline = viewer.entities.add({

    name: '测试线段',

    id: '线段-id',

    polyline: {

        positions: Cesium.Cartesian3.fromDegreesArray([

            116.41, 36.91,

            100.41, 30.89

        ]),

        width: 20,

        material: new Cesium.PolylineGlowMaterialProperty({

            glowPower: 0.1,

            color: Cesium.Color.YELLOW

        })

    }

})
```

完整源码：[GitHub](https://github.com/z2586300277/three-cesium-examples/blob/dev/cesiumExamples/basic/clickEvent.js)

## 小结

- 本文提供 **点击事件** 完整 Cesium.js 源码与在线 Demo，建议先运行案例再改 uniform/参数做二次实验
- 更多 Cesium.js 实战案例见 [three-cesium-examples 合集](https://threehub.cn) 与 [GitHub 开源仓库](https://github.com/z2586300277/three-cesium-examples)

