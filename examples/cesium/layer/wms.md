---
title: "Cesium OGC- wms服务教程"
description: "详解 Cesium.js OGC- wms服务：基于 WebGL 实现「OGC- wms服务」可视化效果，附完整可运行源码，涵盖 Viewer、Scene、Camera 等关键实现，附完整源码与在线 Demo，适合 Cesium 图层 学习与二次开发。"
head:
  - - meta
    - name: keywords
      content: "Cesium.js,OGC- wms服务,WebGL,源码,教程,在线案例,Cesium"
outline: deep
---

### OGC- wms服务 · *OGC- wms服务 * · [▶ 在线运行案例](https://z2586300277.github.io/three-cesium-examples/#/codeMirror?navigation=CesiumJS&classify=layer&id=wms)
- **案例合集：** [三维可视化功能案例（threehub.cn）](https://threehub.cn)

- **开源仓库github地址：** https://github.com/z2586300277/three-cesium-examples

- **400个案例代码: ** [网盘链接](https://pan.quark.cn/s/201da5c82fec)

![OGC- wms服务](https://z2586300277.github.io/three-cesium-examples/cesiumExamples/layer/wms.jpg)

## 你将学到什么

- Scene / Camera / Renderer 标准渲染管线搭建
- 案例完整源码结构与可复用初始化模板

## 效果说明

本案例演示 **OGC- wms服务** 效果：基于 WebGL 实现「OGC- wms服务」可视化效果，附完整可运行源码。建议先打开文首在线案例查看动态画面，再对照下方源码逐步理解。

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
    imageryProvider: false,
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
    skyBox: new Cesium.SkyBox({ show: false })
})


// 加载wms
let wms = new Cesium.WebMapServiceImageryProvider({
    url: "https://mesonet.agron.iastate.edu/cgi-bin/wms/nexrad/n0r.cgi?",
    layers: "nexrad-n0r",
    credit: "demo",
    parameters: {
        transparent: "true",
        format: "image/png",
    },
})
viewer.imageryLayers.addImageryProvider(wms)
```

完整源码：[GitHub](https://github.com/z2586300277/three-cesium-examples/blob/dev/cesiumExamples/layer/wms.js)

## 小结

- 本文提供 **OGC- wms服务** 完整 Cesium.js 源码与在线 Demo，建议先运行案例再改 uniform/参数做二次实验
- 更多 Cesium.js 实战案例见 [three-cesium-examples 合集](https://threehub.cn) 与 [GitHub 开源仓库](https://github.com/z2586300277/three-cesium-examples)

