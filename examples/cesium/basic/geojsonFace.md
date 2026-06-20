---
title: "Cesium GeoJSON 面教程"
description: "详解 Cesium.js GeoJSON 面：GeoJSON 行政区/路线数据可视化贴地展示，涵盖 GeoJSON 等关键实现，附完整源码与在线 Demo，适合 Cesium 基础 学习与二次开发。"
head:
  - - meta
    - name: keywords
      content: "Cesium.js,GeoJSON 面,WebGL,源码,教程,在线案例,Cesium,GeoJSON,矢量数据"
outline: deep
---

### GeoJSON 面 · *GeoJSON Polygon* · [▶ 在线运行案例](https://z2586300277.github.io/three-cesium-examples/#/codeMirror?navigation=CesiumJS&classify=basic&id=geojsonFace)
- **案例合集：** [三维可视化功能案例（threehub.cn）](https://threehub.cn)

- **开源仓库github地址：** https://github.com/z2586300277/three-cesium-examples

- **400个案例代码: ** [网盘链接](https://pan.quark.cn/s/201da5c82fec)

![GeoJSON 面](https://z2586300277.github.io/three-cesium-examples/cesiumExamples/basic/geojsonFace.jpg)

## 你将学到什么

- **GeoJsonDataSource.load** 加载 GeoJSON 面数据
- 统一设置 **stroke / fill / strokeWidth**
- **scene.pick** 点击 Entity 修改 **polygon.material**
- 封装 `changeMaterial` 批量改色

## 效果说明

加载广东省 GeoJSON，半透明蓝色填充；点击某一区域后该区域变为黄色半透明。

## 核心概念

### GeoJsonDataSource

```js
const dataSource = await Cesium.GeoJsonDataSource.load(url, {
    stroke: Cesium.Color.RED.withAlpha(0.5),
    fill: Cesium.Color.BLUE.withAlpha(0.5),
    strokeWidth: 3,
});
viewer.dataSources.add(dataSource);
viewer.flyTo(dataSource);
```

加载后每个 Feature 对应一个 **Entity**，面要素带 `polygon` 图形。

### 点击改色

```js
viewer.screenSpaceEventHandler.setInputAction((event) => {
    const picked = viewer.scene.pick(event.position);
    if (Cesium.defined(picked) && picked.id) {
        picked.id.polygon.material = Cesium.Color.YELLOW.withAlpha(0.5);
    }
}, Cesium.ScreenSpaceEventType.LEFT_CLICK);
```

`picked.id` 即 Entity；`polygon.material` 接受 `Color` 或 `MaterialProperty`。

### 批量换肤

```js
dataSource.changeMaterial = (params) => {
    dataSource.entities.values.forEach(entity => {
        entity.polygon.material = Cesium.Color.fromCssColorString(params.fillColor)
            .withAlpha(params.fillOpacity);
        entity.polygon.outlineColor = ...;
    });
};
```

## 实现步骤

1. 初始化 Viewer 与底图
2. `setGeoPolygon(viewer, geojsonUrl)` 封装 load + add + changeMaterial
3. `viewer.flyTo(dataSource)` 定位到数据范围
4. 注册 LEFT_CLICK，pick 后改 material

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


// 加载geojson数据
const dataSource = setGeoPolygon(viewer, 'https://z2586300277.github.io/three-editor/dist/files/font/guangdong.json')

// 看向geojson数据
viewer.flyTo(dataSource)

// 点击变色
viewer.screenSpaceEventHandler.setInputAction((event) => {

    const pickedObject = viewer.scene.pick(event.position)

    if (Cesium.defined(pickedObject) && pickedObject.id) {

        pickedObject.id.polygon.material = Cesium.Color.fromCssColorString('yellow').withAlpha(0.5)

    }

}, Cesium.ScreenSpaceEventType.LEFT_CLICK)

// 创建 面
async function setGeoPolygon(viewer, source, params = {}) {

    const dataSource = await Cesium.GeoJsonDataSource.load(source, {

        stroke: Cesium.Color.fromCssColorString(params.strokeColor || 'red').withAlpha(params.strokeOpacity || 0.5), // 边界

        fill: Cesium.Color.fromCssColorString(params.fillColor || 'blue').withAlpha(params.fillOpacity || 0.5), // 填充

        strokeWidth: params.strokeWidth || 3,

        markerSymbol: '?',

        ...params

    })

    dataSource.changeMaterial = (params) => dataSource.entities.values.forEach(entity => {

        entity.polygon.material = Cesium.Color.fromCssColorString(params.fillColor || 'blue').withAlpha(params.fillOpacity || 0.5)

        entity.polygon.outlineColor = Cesium.Color.fromCssColorString(params.strokeColor || 'red').withAlpha(params.strokeOpacity || 0.5)

        entity.polygon.outlineWidth = params.strokeWidth || 3

    })

    viewer.dataSources.add(dataSource)

    return dataSource

}
```

完整源码：[GitHub](https://github.com/z2586300277/three-cesium-examples/blob/dev/cesiumExamples/basic/geojsonFace.js)

## 小结

- 本文提供 **GeoJSON 面** 完整 Cesium.js 源码与在线 Demo，建议先运行案例再改 uniform/参数做二次实验
- 更多 Cesium.js 实战案例见 [three-cesium-examples 合集](https://threehub.cn) 与 [GitHub 开源仓库](https://github.com/z2586300277/three-cesium-examples)

