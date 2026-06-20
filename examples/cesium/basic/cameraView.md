---
title: "Cesium 记录视角教程"
description: "详解 Cesium.js 记录视角：基于 WebGL 实现「记录视角」可视化效果，附完整可运行源码，涵盖 Viewer、Scene、Camera 等关键实现，附完整源码与在线 Demo，适合 Cesium 基础 学习与二次开发。"
head:
  - - meta
    - name: keywords
      content: "Cesium.js,记录视角,WebGL,源码,教程,在线案例,Cesium"
outline: deep
---

### 记录视角 · *Save Camera View* · [▶ 在线运行案例](https://z2586300277.github.io/three-cesium-examples/#/codeMirror?navigation=CesiumJS&classify=basic&id=cameraView)
- **案例合集：** [三维可视化功能案例（threehub.cn）](https://threehub.cn)

- **开源仓库github地址：** https://github.com/z2586300277/three-cesium-examples

- **400个案例代码: ** [网盘链接](https://pan.quark.cn/s/201da5c82fec)

![记录视角](https://z2586300277.github.io/three-cesium-examples/cesiumExamples/layer/defaultLayer.jpg)

## 你将学到什么

- 序列化 **camera** 的位置、方向、视锥参数
- 用 **sessionStorage** 在浏览器会话内持久化书签
- 恢复视角时 **flyTo** 与 **setView** 的差异
- **Model.fromGltfAsync** 低层加载 glTF

## 效果说明

调整相机到满意角度后点击「保存视角」，刷新页面或再次打开仍可恢复；可配置 glb URL，加载模型到东京坐标并飞到该处。

## 核心概念

### 需要保存哪些字段？

| 字段 | 含义 |
|------|------|
| `positionWC` / 经纬高 | 相机世界坐标 |
| `directionWC` / `upWC` | 视线与上方向（直接赋值可瞬间恢复） |
| `heading` / `pitch` / `roll` | 欧拉角（只读，但可传给 flyTo） |
| `frustum.fov/near/far` | 透视视锥 |

```js
const saveView = () => ({
    positionDegrees: cartesian3ToDegrees(camera.positionWC),
    position: camera.positionWC,
    direction: camera.directionWC,
    up: camera.upWC,
    frustum: {
        fov: camera.frustum.fov,
        near: camera.frustum.near,
        far: camera.frustum.far,
    },
    heading: camera.heading,
    pitch: camera.pitch,
    roll: camera.roll,
});
```

### 两种恢复方式

1. **直接写 WC 分量**（页面初始化）：无动画，适合首屏
2. **flyTo + orientation**（用户点击恢复）：有过渡

源码里 `loadView` 随机二选一，仅为演示两种 API。

### sessionStorage 结构

```js
{
    url: '.../coffeeMug.glb',
    view: { positionDegrees, heading, pitch, roll, ... }
}
```

::: warning
`Cartesian3` 序列化后是普通 `{x,y,z}` 对象，JSON 存取后需重新构造或直接用分量赋值。
:::

## 实现步骤

1. 初始化 Viewer 与 `camera` 引用
2. 页面加载时读 `sessionStorage`，若有 `view` 则直接赋 `positionWC` 等
3. GUI「保存视角」调用 `saveView()` 写入 storage
4. 「恢复保存视角」读 storage 后 `loadView`
5. 可选：按 `storage.url` 用 `Model.fromGltfAsync` 加载模型

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

- 本文提供 **记录视角** 完整 Cesium.js 源码与在线 Demo，建议先运行案例再改 uniform/参数做二次实验
- 更多 Cesium.js 实战案例见 [three-cesium-examples 合集](https://threehub.cn) 与 [GitHub 开源仓库](https://github.com/z2586300277/three-cesium-examples)

