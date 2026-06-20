---
title: "Cesium 自动旋转教程"
description: "详解 Cesium.js 自动旋转：基于 WebGL 实现「自动旋转」可视化效果，附完整可运行源码，涵盖 Viewer、Scene、Camera 等关键实现，附完整源码与在线 Demo，适合 Cesium 基础 学习与二次开发。"
head:
  - - meta
    - name: keywords
      content: "Cesium.js,自动旋转,WebGL,源码,教程,在线案例,Cesium"
outline: deep
---

### 自动旋转 · *Auto Rotate* · [▶ 在线运行案例](https://z2586300277.github.io/three-cesium-examples/#/codeMirror?navigation=CesiumJS&classify=basic&id=autoRotate)
- **案例合集：** [三维可视化功能案例（threehub.cn）](https://threehub.cn)

- **开源仓库github地址：** https://github.com/z2586300277/three-cesium-examples

- **400个案例代码: ** [网盘链接](https://pan.quark.cn/s/201da5c82fec)

![自动旋转](https://z2586300277.github.io/three-cesium-examples/cesiumExamples/layer/defaultLayer.jpg)

## 你将学到什么

- **viewer.clock.onTick** 每帧回调
- **camera.rotate(axis, angle)** 绕轴旋转
- 展览大屏「地球自动转」实现

## 效果说明

加载影像地球后，相机 **持续绕 Z 轴（UNIT_Z）** 旋转，类似展厅自动播放。

## 核心概念

```js
viewer.clock.onTick.addEventListener(() => {
    viewer.scene.camera.rotate(Cesium.Cartesian3.UNIT_Z, 0.01);
});
```

| 参数 | 含义 |
|------|------|
| `UNIT_Z` | 绕地心垂直轴转（经度方向环绕） |
| `0.01` | 弧度/ tick，越大越快 |

可在回调内加条件（鼠标交互时暂停等）。

与 OrbitControls 的 `autoRotate` 不同，这里是 **直接改 Cesium 相机**。

## 实现步骤

1. 初始化 Viewer，隐藏 credit 容器
2. `viewer.clock.onTick.addEventListener` 注册每帧回调
3. `camera.rotate(UNIT_Z, angle)` 持续绕地轴旋转
4. 可选：鼠标按下时跳过 rotate，实现「交互暂停」

## 代码要点

```js
import * as Cesium from 'cesium'

// Cesium官网的token
Cesium.Ion.defaultAccessToken = "your-cesium-ion-access-token"

const box = document.getElementById('box')

const viewer = new Cesium.Viewer(box, {

    imageryProvider: false, //关闭默认底图

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

viewer.scene.sun.show = false

viewer.scene.moon.show = false

viewer.scene.skyBox.show = false

viewer.scene.backgroundColor = new Cesium.Color(0.0, 0.0, 0.0, 0.0)

viewer._cesiumWidget._creditContainer.style.display = "none"

console.log(Cesium.VERSION)
```

完整源码：[GitHub](https://github.com/z2586300277/three-cesium-examples/blob/dev/cesiumExamples/layer/defaultLayer.js)

## 小结
