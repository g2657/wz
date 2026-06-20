---
title: "Cesium 绘制文字教程"
description: "详解 Cesium.js 绘制文字：基于 WebGL 实现「绘制文字」可视化效果，附完整可运行源码，涵盖 Viewer、Scene、Camera 等关键实现，附完整源码与在线 Demo，适合 Cesium 基础 学习与二次开发。"
head:
  - - meta
    - name: keywords
      content: "Cesium.js,绘制文字,WebGL,源码,教程,在线案例,Cesium"
outline: deep
---

### 绘制文字 · *Draw Text* · [▶ 在线运行案例](https://z2586300277.github.io/three-cesium-examples/#/codeMirror?navigation=CesiumJS&classify=basic&id=cesiumText)
- **案例合集：** [三维可视化功能案例（threehub.cn）](https://threehub.cn)

- **开源仓库github地址：** https://github.com/z2586300277/three-cesium-examples

- **400个案例代码: ** [网盘链接](https://pan.quark.cn/s/201da5c82fec)

![绘制文字](https://z2586300277.github.io/three-cesium-examples/cesiumExamples/layer/defaultLayer.jpg)

## 你将学到什么

- **Entity.label** 绘制场景内文字
- **HeightReference.CLAMP_TO_GROUND** 自动贴地
- **globe.getHeight** 查询地形高度后手动设高
- **disableDepthTestDistance** 避免被地形遮挡

## 效果说明

地球上有三组标注：**贴地**（随地形起伏）、**不贴地**（固定 2000m 高度）、**自动计算贴地**（等地形加载后用 `getHeight` 取高程再放置）。

## 核心概念

### Entity Label

```js
label: {
    text: '贴地',
    font: '14pt monospace',
    outlineWidth: 2,
    showBackground: true,
    backgroundColor: Cesium.Color.WHITE,
    verticalOrigin: Cesium.VerticalOrigin.TOP,
    heightReference: Cesium.HeightReference.CLAMP_TO_GROUND,
    disableDepthTestDistance: Number.POSITIVE_INFINITY,
}
```

| 属性 | 作用 |
|------|------|
| `heightReference` | `CLAMP_TO_GROUND` 贴椭球/地形；默认 `NONE` 用 position 高度 |
| `disableDepthTestDistance` | 设为 ∞ 时 label 始终可见，不被地形 depth test 裁掉 |
| `verticalOrigin` | 文字相对锚点的垂直对齐 |

### 手动查询地形高度

```js
viewer.scene.globe.depthTestAgainstTerrain = true;

const carto = Cesium.Cartographic.fromDegrees(lon, lat, 10);
const height = viewer.scene.globe.getHeight(
    new Cesium.Cartographic(carto.longitude, carto.latitude)
);
// 需在 terrain 加载完成后调用，否则 height 可能为 undefined
```

### 动态修改

Entity 属性可运行时赋值：

```js
text.label.text = '贴地文字';
text.label.fillColor = Cesium.Color.RED;
```

## 实现步骤

1. 开启 `depthTestAgainstTerrain` 使贴地 label 与地形正确遮挡
2. 添加贴地 Entity（point + label，`CLAMP_TO_GROUND`）
3. 添加高空 Entity 作对比（无 heightReference）
4. `flyTo` 到第三点后，在 `complete` 回调里延迟 `getHeight` 再 add Entity

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

- 本文提供 **绘制文字** 完整 Cesium.js 源码与在线 Demo，建议先运行案例再改 uniform/参数做二次实验
- 更多 Cesium.js 实战案例见 [three-cesium-examples 合集](https://threehub.cn) 与 [GitHub 开源仓库](https://github.com/z2586300277/three-cesium-examples)

