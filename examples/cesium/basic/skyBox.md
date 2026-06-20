---
title: "Cesium 天空盒教程"
description: "详解 Cesium.js 天空盒：基于 WebGL 实现「天空盒」可视化效果，附完整可运行源码，涵盖 Viewer、Scene、Camera 等关键实现，附完整源码与在线 Demo，适合 Cesium 基础 学习与二次开发。"
head:
  - - meta
    - name: keywords
      content: "Cesium.js,天空盒,WebGL,源码,教程,在线案例,Cesium"
outline: deep
---

### 天空盒 · *Sky Box* · [▶ 在线运行案例](https://z2586300277.github.io/three-cesium-examples/#/codeMirror?navigation=CesiumJS&classify=basic&id=skyBox)
- **案例合集：** [三维可视化功能案例（threehub.cn）](https://threehub.cn)

- **开源仓库github地址：** https://github.com/z2586300277/three-cesium-examples

- **400个案例代码: ** [网盘链接](https://pan.quark.cn/s/201da5c82fec)

![天空盒](https://z2586300277.github.io/three-cesium-examples/cesiumExamples/basic/skyBox.jpg)

## 你将学到什么

- **scene.skyBox** 替换默认渐变天空
- 立方体贴图 **六面命名**（positiveX / negativeX …）
- **UrlTemplateImageryProvider** 加载 XYZ 瓦片底图

## 效果说明

使用高德卫星影像作底图，天空换成自定义六面 PNG，形成「地面实景 + 定制天空」的视觉效果。

## 核心概念

### SkyBox

```js
viewer.scene.skyBox = new Cesium.SkyBox({
    sources: {
        positiveX: 'px.png',  // 右 (+X)
        negativeX: 'nx.png',  // 左 (-X)
        positiveY: 'py.png',  // 前 (+Y) — 注意与 Three.js 轴向可能不同
        negativeY: 'ny.png',  // 后 (-Y)
        positiveZ: 'pz.png',  // 上 (+Z)
        negativeZ: 'nz.png',  // 下 (-Z)
    }
});
```

本案例注释说明了 **贴图轴与 Cesium 期望面的映射关系**（px/nx/py 等需按实际摄影机朝向调整）。

### 关闭默认底图

```js
const viewer = new Cesium.Viewer(box, { baseLayer: false });
viewer.imageryLayers.addImageryProvider(
    new Cesium.UrlTemplateImageryProvider({
        url: 'https://.../{z}/{x}/{y}',
        maximumLevel: 18,
    })
);
```

## 实现步骤

1. Viewer 设 `baseLayer: false`，手动 add 影像
2. 准备 6 张无缝立方体贴图
3. `new Cesium.SkyBox({ sources })` 赋给 `scene.skyBox`
4. 若天空方向不对，交换 positiveY/negativeY 等面

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
 
viewer.imageryLayers.addImageryProvider(

    new Cesium.UrlTemplateImageryProvider({

        //高德卫星影像
        url: 'https://webst03.is.autonavi.com/appmaptile?style=6&x={x}&y={y}&z={z}',

        maximumLevel: 18

    })

)

// px => -90, nx => 90, py => 0, ny => 180, pz => 0, nz => 180
viewer.scene.skyBox = new Cesium.SkyBox({
    sources: {
        positiveX: FILE_HOST + 'files/cesiumSky/px.png', // 右面
        negativeX: FILE_HOST + 'files/cesiumSky/nx.png', // 左面
        positiveY: FILE_HOST + 'files/cesiumSky/pz.png', // 将前面用作上面
        negativeY: FILE_HOST + 'files/cesiumSky/nz.png', // 将后面用作下面
        positiveZ: FILE_HOST + 'files/cesiumSky/py.png', // 将上面用作前面
        negativeZ: FILE_HOST + 'files/cesiumSky/ny.png'  // 将下面用作后面
    }
});
```

完整源码：[GitHub](https://github.com/z2586300277/three-cesium-examples/blob/dev/cesiumExamples/basic/skyBox.js)

## 小结

- 本文提供 **天空盒** 完整 Cesium.js 源码与在线 Demo，建议先运行案例再改 uniform/参数做二次实验
- 更多 Cesium.js 实战案例见 [three-cesium-examples 合集](https://threehub.cn) 与 [GitHub 开源仓库](https://github.com/z2586300277/three-cesium-examples)

