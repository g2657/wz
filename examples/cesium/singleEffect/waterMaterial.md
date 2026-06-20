---
title: "Cesium 水波材质教程"
description: "详解 Cesium.js 水波材质：基于 WebGL 实现「水波材质」可视化效果，附完整可运行源码，涵盖 水面反射/镜像材质 等关键实现，附完整源码与在线 Demo，适合 Cesium 特效 学习与二次开发。"
head:
  - - meta
    - name: keywords
      content: "Cesium.js,水波材质,WebGL,源码,教程,在线案例,Cesium,水面,反射"
outline: deep
---

### 水波材质 · *Water Material* · [▶ 在线运行案例](https://z2586300277.github.io/three-cesium-examples/#/codeMirror?navigation=CesiumJS&classify=singleEffect&id=waterMaterial)
- **案例合集：** [三维可视化功能案例（threehub.cn）](https://threehub.cn)

- **开源仓库github地址：** https://github.com/z2586300277/three-cesium-examples

- **400个案例代码：** [网盘链接](https://pan.quark.cn/s/201da5c82fec)

![水波材质](https://z2586300277.github.io/three-cesium-examples/cesiumExamples/expand/waterMaterial.jpg)

## 你将学到什么

- 水面反射/镜像材质

## 效果说明

本案例演示 **水波材质** 效果：基于 WebGL 实现「水波材质」可视化效果，附完整可运行源码；核心用到 水面反射/镜像材质。建议先打开文首在线案例查看动态画面，再对照下方源码逐步理解。

## 核心概念
- **Viewer** 聚合 Scene、Camera、Clock 与渲染循环，是 Cesium 应用入口。
- 阅读下方完整源码时，建议从 `init` / `load` / `animate` 三条主线入手，再深入 shader 与工具函数。

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

    baseLayer: Cesium.ImageryLayer.fromProviderAsync(Cesium.ArcGisMapServerImageryProvider.fromUrl('https://server.arcgisonline.com/arcgis/rest/services/World_Imagery/MapServer')),

    fullscreenButton: false,//是否显示全屏按钮，右下角全屏选择按钮

    timeline: false,//是否显示时间轴    

    infoBox: false,//是否显示信息框   

})

// 添加水波纹效果
const positions = [-109.080842, 45.002073, -105.91517, 45.002073 , -104.058488, 46.996596]; // 示例坐标数组
const index = 1; // 示例索引

const primitives = new Cesium.Primitive({
    geometryInstances: new Cesium.GeometryInstance({
        id: 'waterRipple' + index,
        geometry: new Cesium.PolygonGeometry({
            polygonHierarchy: new Cesium.PolygonHierarchy(Cesium.Cartesian3.fromDegreesArray(positions)),
            height: 0,
        }),
    }),
    appearance: new Cesium.EllipsoidSurfaceAppearance({
        material: new Cesium.Material({
            fabric: {
                type: "Water",
                uniforms: {
                    baseWaterColor: Cesium.Color.fromCssColorString('rgba(64,157,253,0.5)'),
                    blendColor: Cesium.Color.fromCssColorString('rgba(64,157,253,0.3)'),
                    normalMap: FILE_HOST + "images/drei/normal.jpg",
                    frequency: 500.0,
                    animationSpeed: 0.1,
                    amplitude: 20,
                    specularIntensity: 5
                }
            }
        }),
    }),
})

viewer.scene.primitives.add(primitives);

// 定位到水波纹位置
viewer.camera.flyTo({
    destination: Cesium.Cartesian3.fromDegrees(-106.0, 44.5, 120000), // 提升高度可以看清水面效果
    orientation: {
        heading: Cesium.Math.toRadians(0),
        pitch: Cesium.Math.toRadians(-45), // 斜视角度更容易看出波纹
        roll: 0
    },
    duration: 1
});
```

完整源码：[GitHub](https://github.com/z2586300277/three-cesium-examples/blob/dev/cesiumExamples/expand/waterMaterial.js)

## 小结
- 本文提供 **水波材质** 完整 Cesium.js 源码与在线 Demo，建议先运行案例再改 uniform/参数做二次实验
- 更多 Cesium.js 实战案例见 [three-cesium-examples 合集](https://threehub.cn) 与 [GitHub 开源仓库](https://github.com/z2586300277/three-cesium-examples)

