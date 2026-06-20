---
title: "Cesium 视角切换教程"
description: "详解 Cesium.js 视角切换：加载倾斜摄影或人工 3D Tiles 白膜并自动定位相机，涵盖 Cesium3DTileset、Cesium、3D 等关键实现，附完整源码与在线 Demo，适合 Cesium 基础 学习与二次开发。"
head:
  - - meta
    - name: keywords
      content: "Cesium.js,视角切换,WebGL,源码,教程,在线案例,Cesium,3D Tiles,倾斜摄影,Cesium Entity,实体"
outline: deep
---

### 视角切换 · *Switch View* · [▶ 在线运行案例](https://z2586300277.github.io/three-cesium-examples/#/codeMirror?navigation=CesiumJS&classify=basic&id=switchView)
- **案例合集：** [三维可视化功能案例（threehub.cn）](https://threehub.cn)

- **开源仓库github地址：** https://github.com/z2586300277/three-cesium-examples

- **400个案例代码: ** [网盘链接](https://pan.quark.cn/s/201da5c82fec)

![视角切换](https://z2586300277.github.io/three-cesium-examples/cesiumExamples/basic/switchView.jpg)

## 你将学到什么

- Cesium **相机定位** API 全家桶：fly / zoom / setView / lookAt
- 对 **Entity、3D Tiles、经纬度** 三类目标分别如何定位
- **trackedEntity** 第三人称跟随与取消
- **HeadingPitchRange** 控制观察角度

## 效果说明

场景加载成都附近 **3D Tiles 白膜** 与 **glb 园区模型**，dat.GUI 提供十余个按钮，逐一演示不同相机 API 的视觉效果差异。

## 核心概念

Cesium 相机操作分三层：

| 层级 | 对象 | 典型方法 |
|------|------|---------|
| 便捷 | `viewer` | `flyTo` / `zoomTo` / `trackedEntity` |
| 相机 | `viewer.camera` | `flyTo` / `setView` / `lookAt` / `viewBoundingSphere` |
| 动画 | `flyHome` | 回到初始视角 |

### flyTo vs setView vs zoomTo

- **flyTo**：带过渡动画，适合产品交互
- **setView**：瞬间跳转，适合初始化或调试
- **zoomTo**：框选目标，动画较短，常用于数据加载后定位

### HeadingPitchRange

```js
new Cesium.HeadingPitchRange(heading, pitch, range)
```

- `heading`：绕 Z 轴旋转（正北为 0）
- `pitch`：俯仰（负值俯视地面）
- `range`：相机到目标距离

### trackedEntity

```js
viewer.trackedEntity = entity;   // 开启跟随
viewer.trackedEntity = undefined; // 取消
```

相机会锁定实体，地球在下方「自转」——适合车辆、飞机跟踪。

## 实现步骤

1. 初始化 Viewer，ArcGIS 影像 + 天地图注记
2. `Cesium3DTileset.fromUrl` 加载白膜，`entities.add` 加载 glb
3. 用 dat.GUI 把各 API 封装成可点击函数
4. 对比 Entity / tileset / 经纬度 三种 `flyTo` 写法

## 代码要点

```js
import * as Cesium from 'cesium'
import { GUI } from 'dat.gui'

const box = document.getElementById('box')

const viewer = new Cesium.Viewer(box, {

    animation: false,//是否创建动画小器件，左下角仪表    

    baseLayerPicker: false,//是否显示图层选择器，右上角图层选择按钮

    baseLayer: Cesium.ImageryLayer.fromProviderAsync(Cesium.ArcGisMapServerImageryProvider.fromUrl('https://server.arcgisonline.com/arcgis/rest/services/World_Imagery/MapServer')),

    fullscreenButton: false,//是否显示全屏按钮，右下角全屏选择按钮

    timeline: false,//是否显示时间轴    

    infoBox: false,//是否显示信息框   

})

// 天地图注记图层
viewer.imageryLayers.addImageryProvider(

    new Cesium.WebMapTileServiceImageryProvider({

        url: "https://t0.tianditu.gov.cn/cva_w/wmts?tk=c4e3a9d54b4a79e885fff9da0fca712a&service=wmts&request=GetTile&version=1.0.0&LAYER=cva&tileMatrixSet=w&TileMatrix={TileMatrix}&TileRow={TileRow}&TileCol={TileCol}&style=default&format=tiles",

        layer: "tdtAnnoLayer",

        style: "default",

        format: "image/jpeg",

        tileMatrixSetID: "GoogleMapsCompatible"

    })

)

const tileset = await Cesium.Cesium3DTileset.fromUrl(FILE_HOST + '3dtiles/house/tileset.json')

viewer.scene.primitives.add(tileset)

const entity = viewer.entities.add({

    name: 'gltf',

    position: Cesium.Cartesian3.fromDegrees(104.0668, 30.5728, 0), // 设置位置

    model: {

        uri: FILE_HOST + '/models/glb/map_park.glb',

    }

})

const gui = new GUI()

// viewer.flyTo viewer.zoomTo viewer.trackedEntity view.camera.setView  viewer.camera.lookAt 

const obj = {

    '重置最初:setView': () => viewer.camera.flyHome(1),

    '经纬度定位:flyTo': () => viewer.camera.flyTo({ destination: Cesium.Cartesian3.fromDegrees(116.4074, 39.9042, 1000) }),

    '实体:flyTo': () => viewer.flyTo(entity),

    '实体:zoomTo': () => viewer.zoomTo(entity),

    '实体跟随:trackedEntity': () => viewer.trackedEntity = entity,

    '取消实体跟随:trackedEntity': () => viewer.trackedEntity = undefined,

    '实体:lookAt': () => viewer.camera.lookAt(entity.position.getValue(Cesium.JulianDate.now()), new Cesium.Cartesian3(0, 0, 1000)),

    '瓦片:viewBoundingSphere': () => viewer.camera.viewBoundingSphere(tileset.boundingSphere, new Cesium.HeadingPitchRange(0, -0.5, 0)),

    '瓦片:flyToBoundingSphere': () => viewer.camera.flyToBoundingSphere(tileset.boundingSphere, new Cesium.HeadingPitchRange(0, -0.5, 0)),

    '瓦片:zoomTo': () => viewer.zoomTo(tileset),

    '瓦片:flyTo': () => viewer.flyTo(tileset),

    '瓦片:setView': () => viewer.camera.setView({
        destination: tileset.boundingSphere.center,
        orientation: {
            heading: 0,
            pitch: -Math.PI / 4,
            roll: 0
        }
    }),

    '瓦片:lookAt': () => viewer.camera.lookAt(tileset.boundingSphere.center),

    '相机:zoomIn 1000': () => viewer.camera.zoomIn(1000),

    '相机:zoomOut 1000': () => viewer.camera.zoomOut(1000),

}

for (const key in obj) gui.add(obj, key)

    console.log(viewer.camera)
```

完整源码：[GitHub](https://github.com/z2586300277/three-cesium-examples/blob/dev/cesiumExamples/basic/switchView.js)

## 小结

- 本文提供 **视角切换** 完整 Cesium.js 源码与在线 Demo，建议先运行案例再改 uniform/参数做二次实验
- 更多 Cesium.js 实战案例见 [three-cesium-examples 合集](https://threehub.cn) 与 [GitHub 开源仓库](https://github.com/z2586300277/three-cesium-examples)

