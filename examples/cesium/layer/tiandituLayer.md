---
title: "Cesium 天地图教程"
description: "详解 Cesium.js 天地图：基于 WebGL 实现「天地图」可视化效果，附完整可运行源码，涵盖 Viewer、Scene、Camera 等关键实现，附完整源码与在线 Demo，适合 Cesium 图层 学习与二次开发。"
head:
  - - meta
    - name: keywords
      content: "Cesium.js,天地图,WebGL,源码,教程,在线案例,Cesium"
outline: deep
---

### 天地图 · *天地图 * · [▶ 在线运行案例](https://z2586300277.github.io/three-cesium-examples/#/codeMirror?navigation=CesiumJS&classify=layer&id=tiandituLayer)
- **案例合集：** [三维可视化功能案例（threehub.cn）](https://threehub.cn)

- **开源仓库github地址：** https://github.com/z2586300277/three-cesium-examples

- **400个案例代码: ** [网盘链接](https://pan.quark.cn/s/201da5c82fec)

![天地图](https://z2586300277.github.io/three-cesium-examples/cesiumExamples/layer/tiandituLayer.jpg)

## 你将学到什么

- Scene / Camera / Renderer 标准渲染管线搭建
- 案例完整源码结构与可复用初始化模板

## 效果说明

本案例演示 **天地图** 效果：基于 WebGL 实现「天地图」可视化效果，附完整可运行源码。建议先打开文首在线案例查看动态画面，再对照下方源码逐步理解。

## 核心概念

- **Viewer** 封装地球、相机、图层与 clock；可关闭 animation/timeline 精简 UI。
- SkyBox 六面图换天空；Water 用法线贴图 + time；地形需 depthTestAgainstTerrain。

## 实现步骤
1. 创建 Viewer，配置地形/影像（若案例需要）并设置初始相机
2. 在 `requestAnimationFrame` 循环中更新状态并 render（Cesium 为 `viewer.render` 或自动渲染）

## 代码要点

```js
import * as Cesium from 'cesium'

const box = document.getElementById('box')

const viewer = new Cesium.Viewer(box, {

    animation: false,//是否创建动画小器件，左下角仪表    

    baseLayerPicker: false,//是否显示图层选择器，右上角图层选择按钮

    fullscreenButton: false,//是否显示全屏按钮，右下角全屏选择按钮

    geocoder: false,//是否显示geocoder小器件，右上角查询按钮    

    homeButton: false,//是否显示Home按钮，右上角home按钮 

    sceneMode: Cesium.SceneMode.SCENE3D,//初始场景模式

    sceneModePicker: false,//是否显示3D/2D选择器，右上角按钮 

    navigationHelpButton: false,//是否显示右上角的帮助按钮  

    selectionIndicator: false,//是否显示选取指示器组件   

    timeline: false,//是否显示时间轴    

    infoBox: false,//是否显示信息框   

    scene3DOnly: true,//如果设置为true，则所有几何图形以3D模式绘制以节约GPU资源  

    orderIndependentTranslucency: false, //是否启用无序透明

    contextOptions: { webgl: { alpha: true } },

    skyBox: new Cesium.SkyBox({ show: false }),

    requestRenderMode: true, // 是否开启请求渲染模式

})

viewer.scene.sun.show = false

viewer.scene.moon.show = false

viewer.scene.skyBox.show = false

viewer.scene.backgroundColor = new Cesium.Color(0.0, 0.0, 0.0, 0.0)

viewer._cesiumWidget._creditContainer.style.display = "none"

// 天地图影像图层
viewer.imageryLayers.addImageryProvider(

    new Cesium.WebMapTileServiceImageryProvider({

        url: "https://t0.tianditu.gov.cn/img_w/wmts?tk=c4e3a9d54b4a79e885fff9da0fca712a&service=wmts&request=GetTile&version=1.0.0&LAYER=img&tileMatrixSet=w&TileMatrix={TileMatrix}&TileRow={TileRow}&TileCol={TileCol}&style=default&format=tiles",

        layer: "tdtImgBasicLayer",

        style: "default",

        format: "image/jpeg",

        tileMatrixSetID: "GoogleMapsCompatible"

    })

)

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

// 天地图境界线
viewer.imageryLayers.addImageryProvider(

    new Cesium.WebMapTileServiceImageryProvider({

        url: "https://t0.tianditu.gov.cn/ibo_w/wmts?tk=c4e3a9d54b4a79e885fff9da0fca712a&service=wmts&request=GetTile&version=1.0.0&LAYER=ibo&tileMatrixSet=w&TileMatrix={TileMatrix}&TileRow={TileRow}&TileCol={TileCol}&style=default&format=tiles",

        layer: "tdtBoundaryLayer",

        style: "default",

        format: "image/jpeg",

        tileMatrixSetID: "GoogleMapsCompatible"

    })

)
```

完整源码：[GitHub](https://github.com/z2586300277/three-cesium-examples/blob/dev/cesiumExamples/layer/tiandituLayer.js)

## 小结

- 本文提供 **天地图** 完整 Cesium.js 源码与在线 Demo，建议先运行案例再改 uniform/参数做二次实验
- 更多 Cesium.js 实战案例见 [three-cesium-examples 合集](https://threehub.cn) 与 [GitHub 开源仓库](https://github.com/z2586300277/three-cesium-examples)

