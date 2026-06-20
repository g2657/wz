---
title: "Three.js 自定义遮罩通道教程"
description: "详解 Three.js 自定义遮罩通道：原场景渲染后经 EffectComposer 叠加 Bloom/模糊等全屏后期，涵盖 EffectComposer、OrbitControls、自定义 等关键实现，附完整源码与在线 Demo，适合 Three.js 后期处理 学习与二次开发。"
head:
  - - meta
    - name: keywords
      content: "Three.js,自定义遮罩通道,WebGL,源码,教程,在线案例,EffectComposer,后期处理,OrbitControls,相机控制,ShaderPass"
outline: deep
---

### 自定义遮罩通道 · *Custom Mask* · [▶ 在线运行案例](https://z2586300277.github.io/three-cesium-examples/#/codeMirror?navigation=ThreeJS&classify=effectComposer&id=customMaskPass)
- **案例合集：** [三维可视化功能案例（threehub.cn）](https://threehub.cn)

- **开源仓库github地址：** https://github.com/z2586300277/three-cesium-examples

- **400个案例代码: ** [网盘链接](https://pan.quark.cn/s/201da5c82fec)

![自定义遮罩通道](https://z2586300277.github.io/three-cesium-examples/threeExamples/effectComposer/customMaskPass.jpg)

## 你将学到什么

- EffectComposer 多 Pass 后期处理管线
- OrbitControls 相机轨道交互
- 自定义 ShaderPass 调色/特效
- `requestAnimationFrame` 渲染循环与 `resize` 自适应

## 效果说明

本案例演示 **自定义遮罩通道** 效果：原场景渲染后经 EffectComposer 叠加 Bloom/模糊等全屏后期；核心用到 EffectComposer、OrbitControls、自定义。建议先打开文首在线案例查看动态画面，再对照下方源码逐步理解。

## 核心概念
- **Scene / Camera / WebGLRenderer** 构成最小渲染闭环；大场景可开 `logarithmicDepthBuffer` 缓解 Z-fighting。
- **EffectComposer** 以多 Pass 链式渲染：RenderPass → 特效 Pass → 输出屏幕，替代直接 `renderer.render`。
- **OrbitControls** 提供轨道旋转/缩放；开启 `enableDamping` 后需在 animate 中 `controls.update()`。

## 实现步骤

1. 搭建 Scene、PerspectiveCamera、WebGLRenderer，挂载 canvas 并处理 `resize`
2. 定义 uniforms / onBeforeCompile 或 ShaderMaterial，编写 GLSL 与材质参数
3. 组装 EffectComposer Pass 链，在 animate 中调用 `composer.render()`
4. 创建 OrbitControls（及 Raycaster 等交互控件，若源码包含）
5. 在 `requestAnimationFrame` 循环中更新状态并 render（Cesium 为 `viewer.render` 或自动渲染）

## 代码要点

```js
import * as THREE from 'three'
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls.js'
import { ShaderPass } from 'three/examples/jsm/postprocessing/ShaderPass.js'
import { EffectComposer } from 'three/examples/jsm/postprocessing/EffectComposer.js'
import { RenderPass } from 'three/examples/jsm/postprocessing/RenderPass.js'

class ScreenMaskPass extends ShaderPass {

    constructor() {

        super({

            name: 'ScreenMaskShader',

            uniforms: {
                tDiffuse: { value: null },
                opacity: { value: 1.0 },
                intensity: { value: 2.0 },
                maskColor: { value: new THREE.Color(1, 1, 1) },
                R: { value: 0.1 },
                sr: { value: 1.2 }
            },

            vertexShader: `
                varying vec2 vUv;
                void main() {
                vUv = uv;
                gl_Position = projectionMatrix * modelViewMatrix * vec4( position, 1.0 );
                }
            `,

            fragmentShader: `
                uniform float opacity;
                uniform float intensity;
                uniform sampler2D tDiffuse;
                uniform vec3 maskColor;
                uniform float R;
                uniform float sr;
                varying vec2 vUv;
                void main() {
                // 阴影颜色
                vec4 texel = texture2D( tDiffuse, vUv );
                // 距离中心的距离
                float dist = sqrt((vUv.x-0.5)*(vUv.x-0.5)+(vUv.y-0.5)*(vUv.y-0.5));
                // 渐变, sr 是开始黑色参数
                float rr = (sr - smoothstep(R, R + 0.5, dist));
                // 叠加黑色
                texel *= vec4(maskColor * rr * vec3(intensity,intensity,intensity), 1.0);
                gl_FragColor = opacity * texel;
                }
            `

        })

    }

}

const box = document.getElementById('box')

const scene = new THREE.Scene()

const camera = new THREE.PerspectiveCamera(75, box.clientWidth / box.clientHeight, 0.1, 1000)

camera.position.set(0, 2, 8)

const renderer = new THREE.WebGLRenderer()

renderer.setSize(box.clientWidth, box.clientHeight)

box.appendChild(renderer.domElement)

new OrbitControls(camera, renderer.domElement)

const composer = new EffectComposer(renderer)

const renderPass = new RenderPass(scene, camera)

composer.addPass(renderPass)

const screenMaskPass = new ScreenMaskPass()

composer.addPass(screenMaskPass)

scene.add(new THREE.AxesHelper(500), new THREE.GridHelper(500, 50))

animate()

function animate() {

    requestAnimationFrame(animate)

    composer.render()

}

window.onresize = () => {

    renderer.setSize(box.clientWidth, box.clientHeight)

    camera.aspect = box.clientWidth / box.clientHeight

    camera.updateProjectionMatrix()

}

// 文件地址
const urls = [0, 1, 2, 3, 4, 5].map(k => ('https://z2586300277.github.io/three-editor/dist/files/scene/skyBox0/' + (k + 1) + '.png'));

const textureCube = new THREE.CubeTextureLoader().load(urls);

scene.background = textureCube;
```

完整源码：[GitHub](https://github.com/z2586300277/three-cesium-examples/blob/dev/threeExamples/effectComposer/customMaskPass.js)

## 小结

- 本文提供 **自定义遮罩通道** 完整 Three.js 源码与在线 Demo，建议先运行案例再改 uniform/参数做二次实验
- 更多 Three.js 实战案例见 [three-cesium-examples 合集](https://threehub.cn) 与 [GitHub 开源仓库](https://github.com/z2586300277/three-cesium-examples)

