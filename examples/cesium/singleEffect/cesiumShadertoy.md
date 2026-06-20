---
title: "Cesium 使用Shadertoy教程"
description: "详解 Cesium.js 使用Shadertoy：基于 WebGL 实现「使用Shadertoy」可视化效果，附完整可运行源码，涵盖 场景雾效增强纵深 等关键实现，附完整源码与在线 Demo，适合 Cesium 特效 学习与二次开发。"
head:
  - - meta
    - name: keywords
      content: "Cesium.js,使用Shadertoy,WebGL,源码,教程,在线案例,Cesium,雾效"
outline: deep
---

### 使用Shadertoy · *Use Shadertoy* · [▶ 在线运行案例](https://z2586300277.github.io/three-cesium-examples/#/codeMirror?navigation=CesiumJS&classify=singleEffect&id=cesiumShadertoy)
- **案例合集：** [三维可视化功能案例（threehub.cn）](https://threehub.cn)

- **开源仓库github地址：** https://github.com/z2586300277/three-cesium-examples

- **400个案例代码: ** [网盘链接](https://pan.quark.cn/s/201da5c82fec)

![使用Shadertoy](https://z2586300277.github.io/three-cesium-examples/cesiumExamples/expand/cesiumShadertoy.jpg)

## 你将学到什么

- 场景雾效增强纵深

## 效果说明

本案例演示 **使用Shadertoy** 效果：基于 WebGL 实现「使用Shadertoy」可视化效果，附完整可运行源码；核心用到 场景雾效增强纵深。建议先打开文首在线案例查看动态画面，再对照下方源码逐步理解。

## 核心概念
- **Viewer** 聚合 Scene、Camera、Clock 与渲染循环，是 Cesium 应用入口。
- 阅读下方完整源码时，建议从 `init` / `load` / `animate` 三条主线入手，再深入 shader 与工具函数。

## 实现步骤
1. 创建 Viewer，配置地形/影像（若案例需要）并设置初始相机
2. 在 `requestAnimationFrame` 循环中更新状态并 render（Cesium 为 `viewer.render` 或自动渲染）

## 代码要点

```js
import * as Cesium from 'cesium'

const viewer = new Cesium.Viewer(document.getElementById('box'), {
    animation: false,
    baseLayerPicker: false,
    baseLayer: Cesium.ImageryLayer.fromProviderAsync(
        Cesium.ArcGisMapServerImageryProvider.fromUrl('https://server.arcgisonline.com/arcgis/rest/services/World_Imagery/MapServer')
    ),
    fullscreenButton: false,
    timeline: false,
    infoBox: false,
});

const customMaterial = new Cesium.Material({
    translucent: false,
    fabric: {
        type: "CustomBoxShader",
        uniforms: {
            iTime: 0.0,
            iResolution: new Cesium.Cartesian2(1024, 1024),
        },
        source: `
            uniform float iTime;
            uniform vec2 iResolution;
            void mainImage( out vec4 o, vec2 u )
            {
                vec2 v = iResolution.xy;
                u = .2*(u+u-v)/v.y;    
                vec4 z = o = vec4(1,2,3,0);
                for (float a = .5, t = iTime, i; ++i < 19.; 
                    o += (1. + cos(z+t))  / length((1.+i*dot(v,v)) * sin(1.5*u/(.5-dot(u,u)) - 9.*u.yx + t))
                    )  
                    v = cos(++t - 7.*u*pow(a += .03, i)) - 5.*u, 
                    u += tanh(40. * dot(u *= mat2(cos(i + .02*t - vec4(0,11,33,0))), u)
                    * cos(1e2*u.yx + t)) / 2e2 + .2 * a * u + cos(4./exp(dot(o,o)/1e2) + t) / 3e2;
                o = 25.6 / (min(o, 13.) + 164. / o) - dot(u, u) / 250.;
            }
            czm_material czm_getMaterial(czm_materialInput materialInput) {
                czm_material material = czm_getDefaultMaterial(materialInput);
                vec4 color = vec4(0.0, 0.0, 0.0, 1.0);
                mainImage(color, materialInput.st * iResolution);
                material.diffuse = color.rgb;
                material.alpha = 1.0;
                return material;
            }
        `,
    },
});

const appearance = new Cesium.MaterialAppearance({
    material: customMaterial,
    flat: false,
    faceForward: true,
    translucent: true,
    closed: true,
    materialCacheKey: "shadertoy-material-appearance",
});

const scene = viewer.scene;
viewer.clock.currentTime.secondsOfDay = 65398;
scene.globe.enableLighting = true;
scene.fog.enabled = true;

const destination = {
    x: -2280236.925141378,
    y: 5006991.049189922,
    z: 3215839.258024074,
};
const boxSize = 25;

const boxGeometry = Cesium.BoxGeometry.fromDimensions({
    dimensions: new Cesium.Cartesian3(boxSize, boxSize, boxSize),
});
const modelMatrix = Cesium.Transforms.eastNorthUpToFixedFrame(destination);
const boxInstance = new Cesium.GeometryInstance({ geometry: boxGeometry });

const primitive = new Cesium.Primitive({
    geometryInstances: boxInstance,
    appearance,
    asynchronous: false,
    modelMatrix,
});
scene.primitives.add(primitive);

let lastTime = Date.now();
scene.preRender.addEventListener(() => {
    const now = Date.now();
    appearance.material.uniforms.iTime += (now - lastTime) / 1000;
    lastTime = now;
});

viewer.camera.lookAt(
    destination,
    new Cesium.HeadingPitchRange(6.283185307179577, -0.4706003213405664, 100),
);
```

完整源码：[GitHub](https://github.com/z2586300277/three-cesium-examples/blob/dev/cesiumExamples/expand/cesiumShadertoy.js)

## 小结
- 本文提供 **使用Shadertoy** 完整 Cesium.js 源码与在线 Demo，建议先运行案例再改 uniform/参数做二次实验
- 更多 Cesium.js 实战案例见 [three-cesium-examples 合集](https://threehub.cn) 与 [GitHub 开源仓库](https://github.com/z2586300277/three-cesium-examples)

