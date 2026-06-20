---
title: "Cesium 影像教程"
description: "详解 Cesium.js 影像：基于 WebGL 实现「影像」可视化效果，附完整可运行源码，涵盖 Viewer、Scene、Camera 等关键实现，附完整源码与在线 Demo，适合 Cesium 离线地图 学习与二次开发。"
head:
  - - meta
    - name: keywords
      content: "Cesium.js,影像,WebGL,源码,教程,在线案例,Cesium"
outline: deep
---

### 影像 · *影像 * · [▶ 在线运行案例](https://z2586300277.github.io/three-cesium-examples/#/codeMirror?navigation=CesiumJS&classify=offline&id=img)
- **案例合集：** [三维可视化功能案例（threehub.cn）](https://threehub.cn)

- **开源仓库github地址：** https://github.com/z2586300277/three-cesium-examples

- **400个案例代码: ** [网盘链接](https://pan.quark.cn/s/201da5c82fec)

![影像](https://z2586300277.github.io/three-cesium-examples/cesiumExamples/offline/img.jpg)

## 你将学到什么

- Scene / Camera / Renderer 标准渲染管线搭建
- 案例完整源码结构与可复用初始化模板

## 效果说明

本案例演示 **影像** 效果：基于 WebGL 实现「影像」可视化效果，附完整可运行源码。建议先打开文首在线案例查看动态画面，再对照下方源码逐步理解。

## 核心概念

- **Viewer** 封装地球、相机、图层与 clock；可关闭 animation/timeline 精简 UI。

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

    baseLayer: false, // 不显示默认图层

    fullscreenButton: false,//是否显示全屏按钮，右下角全屏选择按钮

    timeline: false,//是否显示时间轴    

    infoBox: false,//是否显示信息框   

})

let imagelayer = new Cesium.SingleTileImageryProvider({
    url: FILE_HOST + "images/offlineLayer/world_img.jpg",
    tileWidth: 256,
    tileHeight: 256,
});
viewer.imageryLayers.addImageryProvider(imagelayer);
```

完整源码：[GitHub](https://github.com/z2586300277/three-cesium-examples/blob/dev/cesiumExamples/offline/img.js)

## 小结

- 本文提供 **影像** 完整 Cesium.js 源码与在线 Demo，建议先运行案例再改 uniform/参数做二次实验
- 更多 Cesium.js 实战案例见 [three-cesium-examples 合集](https://threehub.cn) 与 [GitHub 开源仓库](https://github.com/z2586300277/three-cesium-examples)

