---
title: "Cesium CSS 元素教程"
description: "详解 Cesium.js CSS 元素：基于 WebGL 实现「CSS 元素」可视化效果，附完整可运行源码，涵盖 Cesium 等关键实现，附完整源码与在线 Demo，适合 Cesium 基础 学习与二次开发。"
head:
  - - meta
    - name: keywords
      content: "Cesium.js,CSS 元素,WebGL,源码,教程,在线案例,Cesium,Cesium Entity,实体"
outline: deep
---

### CSS 元素 · *CSS DOM Overlay* · [▶ 在线运行案例](https://z2586300277.github.io/three-cesium-examples/#/codeMirror?navigation=CesiumJS&classify=basic&id=cssElement)
- **案例合集：** [三维可视化功能案例（threehub.cn）](https://threehub.cn)

- **开源仓库github地址：** https://github.com/z2586300277/three-cesium-examples

- **400个案例代码: ** [网盘链接](https://pan.quark.cn/s/201da5c82fec)

![CSS 元素](https://z2586300277.github.io/three-cesium-examples/cesiumExamples/basic/cssElement.jpg)

## 你将学到什么

- 不用 Entity，用 **原生 HTML** 做信息面板
- **postRender** 每帧把世界坐标投影到屏幕像素
- **SceneTransforms.worldToWindowCoordinates** 坐标转换
- 背面/过远时隐藏 DOM

## 效果说明

北京坐标处有一个可点击的蓝色边框 div「2dDOM」，随相机移动始终钉在地标上方；过远或转到地球背面时自动隐藏。

## 核心概念

Cesium 内置 **HtmlOverlay / InfoBox** 等方案；本案例演示 **最小自实现**：

```
地理坐标 (Cartesian3)
    ↓ worldToWindowCoordinates
屏幕像素 (x, y)
    ↓ CSS transform: translate
DOM 定位
```

### 容器层级

```js
const css2dContainer = document.createElement('div');
Object.assign(css2dContainer.style, {
    position: 'absolute', top: '0', left: '0',
    pointerEvents: 'none',  // 容器穿透，子元素可单独开启
    zIndex: '1',
});
box.appendChild(css2dContainer);
```

### postRender 同步

```js
viewer.scene.postRender.addEventListener(() => {
    const windowCoord = Cesium.SceneTransforms.worldToWindowCoordinates(
        viewer.scene, position
    );
    if (windowCoord) {
        DOM.style.transform = `translate(${windowCoord.x}px, ${windowCoord.y - offsetHeight}px)`;
    }
    // 距离过远则隐藏
    const maxDistance = ...;
    DOM.style.display = distance > maxDistance ? 'none' : 'block';
});
```

::: tip 与 Three.js CSS2DRenderer 对比
思路相同：每帧投影 + translate。Cesium 需自己算 `maxDistance` 处理背面。
:::

## 实现步骤

1. 在 Viewer 容器外再套一层 `css2dContainer`
2. 创建样式化 div，设置 `pointerEvents: 'all'` 以响应点击
3. `setCss2dDom(viewer, DOM, [lon, lat, height])` 注册 postRender
4. 返回 destroy 函数：移除 DOM 并取消监听

## 代码要点

```js
import * as Cesium from 'cesium'

const box = document.getElementById('box')

// 创建一个专门用于放置2dDOM的容器
const css2dContainer = document.createElement('div')

Object.assign(css2dContainer.style, {

    position: 'absolute',

    top: '0',

    left: '0',

    pointerEvents: 'none',

    zIndex: '1',

})

box.appendChild(css2dContainer)

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

// 创建2dDOM
const DOM = document.createElement('div')

// 样式 
Object.assign(DOM.style, {

    width: '100px',

    height: '30px',

    border: '1px solid blue',

    color: 'white',

    fontSize: '20px',

    cursor: 'pointer'

})

DOM.innerHTML = '2dDOM'

setCss2dDom(viewer, DOM, [116.46, 39.92, 0]) // 设置2dDOM 移动

viewer.entities.add({ position: Cesium.Cartesian3.fromDegrees(116.46, 39.92), point: { pixelSize: 10 } }) // 创建测试点

viewer.camera.flyTo({ destination: Cesium.Cartesian3.fromDegrees(116.46, 39.92, 10000000) }) // 定位

/* 设置2dDOM 移动 */
function setCss2dDom(viewer, DOM, position) {

    if (!position) return

    if (!(position instanceof Cesium.Cartesian3)) position = Cesium.Cartesian3.fromDegrees(...position)

    Object.assign(DOM.style, {

        pointerEvents: 'all',

        zIndex: 'auto',

    })

    const { offsetHeight } = DOM

    const { camera, scene } = viewer

    css2dContainer.appendChild(DOM)

    const destroy = viewer.scene.postRender.addEventListener(() => {

        const windowCoord = Cesium.SceneTransforms.worldToWindowCoordinates(viewer.scene, position)

        if (windowCoord) {

            DOM.style.transform = `translate(${windowCoord.x}px, ${windowCoord.y - offsetHeight}px)`

        }

        const maxDistance = scene.globe.ellipsoid.cartesianToCartographic(camera.position).height + scene.globe.ellipsoid.maximumRadius

        Cesium.Cartesian3.distance(camera.position, position) > maxDistance ? DOM.style.display = 'none' : DOM.style.display = 'block'

    })

    return () => {

        viewer.css2dContainer.removeChild(DOM)

        destroy()

    }

}
```

完整源码：[GitHub](https://github.com/z2586300277/three-cesium-examples/blob/dev/cesiumExamples/basic/cssElement.js)

## 小结

- 本文提供 **CSS 元素** 完整 Cesium.js 源码与在线 Demo，建议先运行案例再改 uniform/参数做二次实验
- 更多 Cesium.js 实战案例见 [three-cesium-examples 合集](https://threehub.cn) 与 [GitHub 开源仓库](https://github.com/z2586300277/three-cesium-examples)

