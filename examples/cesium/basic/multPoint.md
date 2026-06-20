---
title: "Cesium 海量点教程"
description: "详解 Cesium.js 海量点：基于 WebGL 实现「海量点」可视化效果，附完整可运行源码，涵盖 Viewer、Scene、Camera 等关键实现，附完整源码与在线 Demo，适合 Cesium 基础 学习与二次开发。"
head:
  - - meta
    - name: keywords
      content: "Cesium.js,海量点,WebGL,源码,教程,在线案例,Cesium"
outline: deep
---

### 海量点 · *Mass Points* · [▶ 在线运行案例](https://z2586300277.github.io/three-cesium-examples/#/codeMirror?navigation=CesiumJS&classify=basic&id=multPoint)
- **案例合集：** [三维可视化功能案例（threehub.cn）](https://threehub.cn)

- **开源仓库github地址：** https://github.com/z2586300277/three-cesium-examples

- **400个案例代码: ** [网盘链接](https://pan.quark.cn/s/201da5c82fec)

![海量点](https://z2586300277.github.io/three-cesium-examples/cesiumExamples/basic/multPoint.jpg)

## 你将学到什么

- **BillboardCollection** 相对 Entity 的性能优势
- 双重 for 循环生成 **64800** 个格网点
- 每个 billboard 设 **id** 供 pick 识别

## 效果说明

全球每 1° 经纬交点处放置一张小图标，随机着色；点击可在控制台输出 `object.id`。

## 核心概念

| 方式 | 适用规模 | API |
|------|---------|-----|
| Entity.point | 数百以内 | `entities.add` |
| **BillboardCollection** | 数万 | `scene.primitives.add` |
| PointPrimitiveCollection | 数十万点（无图标） | 同 Primitive 层 |

```js
const billboards = new Cesium.BillboardCollection();
viewer.scene.primitives.add(billboards);

billboards.add({
    position: Cesium.Cartesian3.fromDegrees(longitude, latitude),
    image: HOST + '/files/author/z2586300277.png',
    scale: 0.1,
    color: new Cesium.Color(Math.random(), Math.random(), Math.random(), 1),
    id: 'billboard-' + longitude + '-' + latitude,
});
```

Entity 每个点一个 draw call；Collection **合批** 同纹理 billboard。

## 实现步骤

1. `camera.setView` 定初始全球视角
2. 创建 `BillboardCollection` 并 add 到 scene
3. 经度 -180~180、纬度 -90~90 双重循环 add
4. `LEFT_CLICK` + `scene.pick` 读 id

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

// 设置一个视角
viewer.camera.setView({

    destination: Cesium.Cartesian3.fromRadians(2.100117282185777, 0.6195146302793972, 104244.23864046125),

    orientation: {

        direction: new Cesium.Cartesian3(0.5153454276260272, -0.7794098602398831, 0.3562855034741005),

        up: new Cesium.Cartesian3(-0.1511548595883593, 0.326557215595639, 0.9330126437327882)

    }

})

// 添加点击事件监听器
viewer.screenSpaceEventHandler.setInputAction(function (event) {

    const object = viewer.scene.pick(event.position)

    console.log(object.id)

}, Cesium.ScreenSpaceEventType.LEFT_CLICK)

const billboards = new Cesium.BillboardCollection(); //  创建billboard集合对象

viewer.scene.primitives.add(billboards); //  添加billboard集合对象到场景中

const color = () => new Cesium.Color(Math.random(), Math.random(), Math.random(), 1); // 随机颜色

//  生成64800个点，每个经度、纬度值各生成一个点，高度为0（贴地表）
for (var longitude = -180; longitude < 180; longitude++) {

    for (var latitude = -90; latitude < 90; latitude++) {

        billboards.add({

            position: Cesium.Cartesian3.fromDegrees(longitude, latitude),

            image: HOST + '/files/author/z2586300277.png', // 图标

            scale: 0.1, // 调整图标的大小

            color: color(), // 随机颜色

            id: 'billboard' + '-' + longitude + '-' + latitude

        })

    }

}
```

完整源码：[GitHub](https://github.com/z2586300277/three-cesium-examples/blob/dev/cesiumExamples/basic/multPoint.js)

## 小结

- 本文提供 **海量点** 完整 Cesium.js 源码与在线 Demo，建议先运行案例再改 uniform/参数做二次实验
- 更多 Cesium.js 实战案例见 [three-cesium-examples 合集](https://threehub.cn) 与 [GitHub 开源仓库](https://github.com/z2586300277/three-cesium-examples)

