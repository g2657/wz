---
title: "Cesium 地图滤镜教程"
description: "详解 Cesium.js 地图滤镜：基于 WebGL 实现「地图滤镜」可视化效果，附完整可运行源码，涵盖 Viewer、Scene、Camera 等关键实现，附完整源码与在线 Demo，适合 Cesium 图层 学习与二次开发。"
head:
  - - meta
    - name: keywords
      content: "Cesium.js,地图滤镜,WebGL,源码,教程,在线案例,Cesium"
outline: deep
---
### 地图滤镜 · *Map Filter* · [▶ 在线运行案例](https://z2586300277.github.io/three-cesium-examples/#/codeMirror?navigation=CesiumJS&classify=layer&id=mapfilterLayer)
- **案例合集：** [三维可视化功能案例（threehub.cn）](https://threehub.cn)

- **开源仓库github地址：** https://github.com/z2586300277/three-cesium-examples

- **400个案例代码: ** [网盘链接](https://pan.quark.cn/s/201da5c82fec)

![地图滤镜](https://z2586300277.github.io/three-cesium-examples/cesiumExamples/layer/mapfilterLayer.jpg)

## 你将学到什么

- Scene / Camera / Renderer 标准渲染管线搭建
- 案例完整源码结构与可复用初始化模板

## 效果说明

本案例演示 **地图滤镜** 效果：基于 WebGL 实现「地图滤镜」可视化效果，附完整可运行源码。建议先打开文首在线案例查看动态画面，再对照下方源码逐步理解。

## 核心概念

- **Viewer** 封装地球、相机、图层；可关闭 animation/timeline 等 UI 精简界面。

- **ImageryLayer** 叠加 XYZ/WMTS/ArcGIS 等底图，`imageryLayers.add/remove` 管理。

## 实现步骤

1. 初始化 `Cesium.Viewer` 与底图图层
2. 添加 Entity / Primitive / DataSource 等业务对象
3. 按需 `camera.flyTo` 定位视角

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

    baseLayer: false, // 不显示默认图层

})

viewer.imageryLayers.addImageryProvider(

    new Cesium.UrlTemplateImageryProvider({

        url: 'https://webrd02.is.autonavi.com/appmaptile?lang=zh_cn&size=1&scale=2&style=8&x={x}&y={y}&z={z}',

        maximumLevel: 18

    })

)

setViewerTheme(viewer) // 设置主题

function setViewerTheme(viewer, options = {}) {

    const baseLayer = viewer.imageryLayers.get(0)

    if (!baseLayer) return

    baseLayer.brightness = options.brightness ?? 0.6

    baseLayer.contrast = options.contrast ?? 1.8

    baseLayer.gamma = options.gamma ?? 0.3

    baseLayer.hue = options.hue ?? 1

    baseLayer.saturation = options.saturation || 0

    const baseFragShader = (viewer.scene.globe)._surfaceShaderSet.baseFragmentShaderSource.sources

    for (let i = 0; i < baseFragShader.length; i++) {

        const strS = 'color = czm_saturation(color, textureSaturation);\n#endif\n'

        let strT = 'color = czm_saturation(color, textureSaturation);\n#endif\n'

        if (!options.invertColor) {

            strT += `
                color.r = 1.0 - color.r;
                color.g = 1.0 - color.g;
                color.b = 1.0 - color.b;
            `
            
        }

        strT += `
            color.r = color.r * ${options.filterRGB_R ?? 100}.0/255.0;
            color.g = color.g * ${options.filterRGB_G ?? 138}.0/255.0;
            color.b = color.b * ${options.filterRGB_B ?? 230}.0/255.0;
        `

        baseFragShader[i] = baseFragShader[i].replace(strS, strT)

    }

    viewer.scene.requestRender();

}
```

完整源码：[GitHub](https://github.com/z2586300277/three-cesium-examples/blob/dev/cesiumExamples/layer/mapfilterLayer.js)

## 小结

- 本文提供 **地图滤镜** 完整 Cesium.js 源码与在线 Demo，建议先运行案例再改 uniform/参数做二次实验
- 更多 Cesium.js 实战案例见 [three-cesium-examples 合集](https://threehub.cn) 与 [GitHub 开源仓库](https://github.com/z2586300277/three-cesium-examples)

