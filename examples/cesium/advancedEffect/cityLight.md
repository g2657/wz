---
title: "Cesium 城市光影教程"
description: "详解 Cesium.js 城市光影：加载倾斜摄影或人工 3D Tiles 白膜并自动定位相机，涵盖 Cesium3DTileset、3D 等关键实现，附完整源码与在线 Demo，适合 Cesium 高级特效 学习与二次开发。"
head:
  - - meta
    - name: keywords
      content: "Cesium.js,城市光影,WebGL,源码,教程,在线案例,Cesium,3D Tiles,倾斜摄影"
outline: deep
---

### 城市光影 · *City Light* · [▶ 在线运行案例](https://z2586300277.github.io/three-cesium-examples/#/codeMirror?navigation=CesiumJS&classify=advancedEffect&id=cityLight)
- **案例合集：** [三维可视化功能案例（threehub.cn）](https://threehub.cn)

- **开源仓库github地址：** https://github.com/z2586300277/three-cesium-examples

- **400个案例代码: ** [网盘链接](https://pan.quark.cn/s/201da5c82fec)

![城市光影](https://z2586300277.github.io/three-cesium-examples/cesiumExamples/expand/cityLight.jpg)

## 你将学到什么

- Cesium3DTileset 加载 3D Tiles 倾斜摄影
- 3D Tiles 流式 LOD 场景

## 效果说明

本案例演示 **城市光影** 效果：加载倾斜摄影或人工 3D Tiles 白膜并自动定位相机；核心用到 Cesium3DTileset、3D。建议先打开文首在线案例查看动态画面，再对照下方源码逐步理解。

## 核心概念
- **Viewer** 聚合 Scene、Camera、Clock 与渲染循环，是 Cesium 应用入口。
- **Cesium3DTileset** 流式加载 LOD 瓦片，适合城市倾斜摄影；常用 `viewer.zoomTo(tileset)` 或 `viewBoundingSphere` 定位。
- 阅读下方完整源码时，建议从 `init` / `load` / `animate` 三条主线入手，再深入 shader 与工具函数。

## 实现步骤
1. 创建 Viewer，配置地形/影像（若案例需要）并设置初始相机
2. 异步加载模型 / 3D Tiles / GeoJSON 等资源并加入 scene 或 entities
3. 在 `requestAnimationFrame` 循环中更新状态并 render（Cesium 为 `viewer.render` 或自动渲染）

## 代码要点

```js
import * as Cesium from 'cesium'
import * as dat from 'dat.gui'

const box = document.getElementById('box')

const viewer = new Cesium.Viewer(box, {

    animation: false,//是否创建动画小器件，左下角仪表    

    baseLayerPicker: false,//是否显示图层选择器，右上角图层选择按钮

    baseLayer: Cesium.ImageryLayer.fromProviderAsync(Cesium.ArcGisMapServerImageryProvider.fromUrl('https://server.arcgisonline.com/arcgis/rest/services/World_Imagery/MapServer')),

    fullscreenButton: false,//是否显示全屏按钮，右下角全屏选择按钮

    timeline: false,//是否显示时间轴    

    infoBox: false,//是否显示信息框   

})

// https://guangfus:663/3dtiles/whiteModel/tileset.json
const tileset = await Cesium.Cesium3DTileset.fromUrl('https://g2657.github.io/gz-city/tileset.json')

viewer.scene.primitives.add(tileset)

tileset.maximumScreenSpaceError = 1

viewer.flyTo(tileset)

const uniforms = {
    u_sweep_color: { value: Cesium.Color.fromBytes(43, 167, 255, 255), type: Cesium.UniformType.VEC3 },
    u_mix_color1: { value: Cesium.Color.fromBytes(9, 9, 14, 255), type: Cesium.UniformType.VEC3 },
    u_mix_color2: { value: Cesium.Color.fromBytes(0, 128, 255, 255), type: Cesium.UniformType.VEC3 },
    u_sweep_width: { value: 0.03, type: Cesium.UniformType.FLOAT },
    u_time: { value: 0, type: Cesium.UniformType.FLOAT },
    u_model_height: { value: 100, type: Cesium.UniformType.FLOAT },
    u_height_offset: { value: 0.0, type: Cesium.UniformType.FLOAT }
}

const gui = new dat.GUI()
gui.addColor({ sweepColor: '#2ba7ff' }, 'sweepColor').onChange(v => {
    const hex = v.replace('#', '')
    const r = parseInt(hex.substring(0, 2), 16)
    const g = parseInt(hex.substring(2, 4), 16)
    const b = parseInt(hex.substring(4, 6), 16)
    uniforms.u_sweep_color.value = Cesium.Color.fromBytes(r, g, b, 255)
})

gui.addColor({ mixColor1: '#09090e' }, 'mixColor1').onChange(v => {
    const hex = v.replace('#', '')
    const r = parseInt(hex.substring(0, 2), 16)
    const g = parseInt(hex.substring(2, 4), 16)
    const b = parseInt(hex.substring(4, 6), 16)
    uniforms.u_mix_color1.value = Cesium.Color.fromBytes(r, g, b, 255)
})
gui.addColor({ mixColor2: '#0080ff' }, 'mixColor2').onChange(v => {
    const hex = v.replace('#', '')
    const r = parseInt(hex.substring(0, 2), 16)
    const g = parseInt(hex.substring(2, 4), 16)
    const b = parseInt(hex.substring(4, 6), 16)
    uniforms.u_mix_color2.value = Cesium.Color.fromBytes(r, g, b, 255)
})

gui.add({ sweepWidth: 0.2 }, 'sweepWidth', 0.05, 1.0).onChange(v => {
    uniforms.u_sweep_width.value = v
})
gui.add({ modelHeight: 100 }, 'modelHeight', 10, 1000).name('模型高度').onChange(v =>  uniforms.u_model_height.value = v)
gui.add({ heightOffset: 0.0 }, 'heightOffset', -50, 50).name('高度偏移').onChange(v => uniforms.u_height_offset.value = v)

const shader = new Cesium.CustomShader({
    vertexShaderText: `void vertexMain(VertexInput vsInput, inout czm_modelVertexOutput vsOutput) {
            float adjustedZ = vsInput.attributes.positionMC.z + u_height_offset;
            float normalizedHeight = clamp(adjustedZ / u_model_height, 0.0, 1.0);
            float enhancedHeight = sqrt(normalizedHeight);
            v_uv = vec2(enhancedHeight, enhancedHeight);
        }`,
    fragmentShaderText: `float random(vec2 st) {
            return fract(sin(dot(st.xy, vec2(12.9898, 78.233))) * 43758.5453123);
        }
        
        void fragmentMain(FragmentInput fsInput, inout czm_modelMaterial material) {
            float gradientFactor = smoothstep(0.0, 1.0, v_uv.y);
            vec3 originColor = mix(u_mix_color1, u_mix_color2, gradientFactor);
            float t = fract(u_time * 2.) * 2.;
            vec2 absUv = abs(v_uv - t);
            
            vec2 st = v_uv * 15.;
            vec2 ipos = floor(st + u_time * 5.);
            float r = random(ipos) + .2;
            
            float d = clamp(distance(0., absUv.y) / u_sweep_width, 0., 1.);
            float diffuse = clamp(-dot(czm_sunDirectionEC, fsInput.attributes.normalEC), 0., .45);
            
            vec3 color = mix(u_sweep_color * r + u_sweep_color * .8, originColor, d);
            material.diffuse = color;
            material.emissive = vec3(diffuse) * (1. - d);
        }`,
    uniforms,
    varyings: { v_uv: Cesium.VaryingType.VEC2 }
})

viewer.scene.preRender.addEventListener(function(scene, time) {
    shader.setUniform("u_time", performance.now() * 0.0001);
});

tileset.customShader = shader
```

完整源码：[GitHub](https://github.com/z2586300277/three-cesium-examples/blob/dev/cesiumExamples/expand/cityLight.js)

## 小结
- 本文提供 **城市光影** 完整 Cesium.js 源码与在线 Demo，建议先运行案例再改 uniform/参数做二次实验
- 更多 Cesium.js 实战案例见 [three-cesium-examples 合集](https://threehub.cn) 与 [GitHub 开源仓库](https://github.com/z2586300277/three-cesium-examples)

