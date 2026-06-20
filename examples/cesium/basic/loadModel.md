---
title: "Cesium 加载模型教程"
description: "详解 Cesium.js 加载模型：加载倾斜摄影或人工 3D Tiles 白膜并自动定位相机，涵盖 Cesium3DTileset、Cesium、3D 等关键实现，附完整源码与在线 Demo，适合 Cesium 基础 学习与二次开发。"
head:
  - - meta
    - name: keywords
      content: "Cesium.js,加载模型,WebGL,源码,教程,在线案例,Cesium,3D Tiles,倾斜摄影,Cesium Entity,实体"
outline: deep
---

### 加载模型 · *Load Model* · [▶ 在线运行案例](https://z2586300277.github.io/three-cesium-examples/#/codeMirror?navigation=CesiumJS&classify=basic&id=loadModel)
- **案例合集：** [三维可视化功能案例（threehub.cn）](https://threehub.cn)

- **开源仓库github地址：** https://github.com/z2586300277/three-cesium-examples

- **400个案例代码: ** [网盘链接](https://pan.quark.cn/s/201da5c82fec)

![加载模型](https://z2586300277.github.io/three-cesium-examples/cesiumExamples/basic/loadModel.jpg)

## 你将学到什么

- **3D Tiles** 与 **glTF Entity** 两种加载方式
- `viewer.scene.primitives.add(tileset)` vs `viewer.entities.add`
- 经纬高坐标转换与 **贴地偏移** 原理

## 效果说明

加载倾斜摄影 **3D Tiles** 建筑白膜，在其包围球中心放置一辆 **glTF 汽车模型**，并将 tileset 整体贴地。

## 核心概念

Cesium 有两套主要渲染 API：

| API | 适用 | 加载方式 |
|-----|------|---------|
| **Primitive** | 3D Tiles、地形、大批量静态几何 | `scene.primitives.add()` |
| **Entity** | 点线面、模型、标签等高层对象 | `viewer.entities.add()` |

### 3D Tiles

```js
const tileset = await Cesium.Cesium3DTileset.fromUrl(url);
viewer.scene.primitives.add(tileset);
viewer.camera.viewBoundingSphere(tileset.boundingSphere, new Cesium.HeadingPitchRange(0, -0.5, 0));
```

3D Tiles 是 **流式 LOD 格式**，适合城市级倾斜摄影、点云、人工模型库。

### Entity glTF

```js
viewer.entities.add({
    position: tileset.boundingSphere.center,
    model: {
        uri: 'car.glb',
        minimumPixelSize: 128,  // 远距离最小像素，保证可见
        maximumScale: 200,
    }
});
```

### 贴地偏移 adjust3dtilesPosition

倾斜摄影数据常 **悬浮或陷入地下**，需计算高度偏移：

```
1. boundingSphere.center → Cartographic（经纬高）
2. 取椭球面高度 0 的表面点 surface
3. offset = surface - center
4. tileset.modelMatrix 应用平移
```

涉及 Cesium 空间数学：`Cartographic.fromCartesian` → `Cartesian3.fromRadians` → `Matrix4.fromTranslation`。

## 实现步骤

1. `new Cesium.Viewer()` 精简 UI，配置 ArcGIS 影像底图
2. `await Cesium3DTileset.fromUrl()` 加载 tileset
3. `adjust3dtilesPosition(tileset)` 贴地
4. `viewBoundingSphere` 飞到模型
5. Entity 加载 glTF 到 tileset 中心

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

// 3dtiles 模型
const tileset = await Cesium.Cesium3DTileset.fromUrl(FILE_HOST + '3dtiles/test/tileset.json')

viewer.scene.primitives.add(tileset)

adjust3dtilesPosition(tileset)

// 设置视角
viewer.camera.viewBoundingSphere(tileset.boundingSphere, new Cesium.HeadingPitchRange(0, -0.5, 0))

// gltf 模型 放到 3dtiles 模型中心
viewer.entities.add({

    name: 'gltf',

    position: tileset.boundingSphere.center,

    model: {

        uri: HOST + '/files/model/car.glb',

        minimumPixelSize: 128,

        maximumScale: 200,

    }

})

// 贴地
function adjust3dtilesPosition(tileset) {

    const boundingSphere = tileset.boundingSphere

    const cartographic = Cesium.Cartographic.fromCartesian(boundingSphere.center) // 获取中心点

    const surface = Cesium.Cartesian3.fromRadians(cartographic.longitude, cartographic.latitude, 0.0) // 获取表面点

    const offset = Cesium.Cartesian3.subtract(surface, boundingSphere.center, new Cesium.Cartesian3()) // 计算偏移

    tileset.modelMatrix = Cesium.Matrix4.fromTranslation(offset) // 设置偏移

}
```

完整源码：[GitHub](https://github.com/z2586300277/three-cesium-examples/blob/dev/cesiumExamples/basic/loadModel.js)

## 小结

- 大场景建筑 → **3D Tiles**；单个可交互模型 → **Entity.model**
- 相关：[Cesium 相机](/examples/cesium/basic/cameraView) · [3D Tiles 着色](/examples/cesium/advancedEffect/tilesShader)

> 基础功能 · Cesium.js
