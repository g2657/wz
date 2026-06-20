---
title: "Three.js 着色器光效教程"
description: "详解 Three.js 着色器光效：用 Canvas 2D 绘制内容并实时映射为 Three.js 纹理，涵盖 ShaderMaterial、OrbitControls、Canvas 等关键实现，附完整源码与在线 Demo，适合 Three.js 着色器 学习与二次开发。"
head:
  - - meta
    - name: keywords
      content: "Three.js,着色器光效,WebGL,源码,教程,在线案例,ShaderMaterial,自定义着色器,GLSL,OrbitControls,相机控制,Canvas纹理"
outline: deep
---

### 着色器光效 · *Shader Light* · [▶ 在线运行案例](https://z2586300277.github.io/three-cesium-examples/#/codeMirror?navigation=ThreeJS&classify=shader&id=shaderLight)
- **案例合集：** [三维可视化功能案例（threehub.cn）](https://threehub.cn)

- **开源仓库github地址：** https://github.com/z2586300277/three-cesium-examples

- **400个案例代码: ** [网盘链接](https://pan.quark.cn/s/201da5c82fec)

![着色器光效](https://z2586300277.github.io/three-cesium-examples/threeExamples/shader/shaderLight.jpg)

## 你将学到什么

- ShaderMaterial 自定义着色器实现核心视觉效果
- OrbitControls 相机轨道交互
- Canvas 动态纹理贴图
- `requestAnimationFrame` 渲染循环与 `resize` 自适应

## 效果说明

本案例演示 **着色器光效** 效果：用 Canvas 2D 绘制内容并实时映射为 Three.js 纹理；核心用到 ShaderMaterial、OrbitControls、Canvas。建议先打开文首在线案例查看动态画面，再对照下方源码逐步理解。

## 核心概念
- **Scene / Camera / WebGLRenderer** 构成最小渲染闭环；大场景可开 `logarithmicDepthBuffer` 缓解 Z-fighting。
- **ShaderMaterial** 通过 `uniforms` + 自定义 GLSL 控制逐像素/逐点效果；透明粒子常配合 `depthTest: false`。
- **OrbitControls** 提供轨道旋转/缩放；开启 `enableDamping` 后需在 animate 中 `controls.update()`。
- **CanvasTexture** 每帧或按需把 2D Canvas 内容上传 GPU，适合动态文字、图表、视频帧贴图。

## 实现步骤

1. 搭建 Scene、PerspectiveCamera、WebGLRenderer，挂载 canvas 并处理 `resize`
2. 定义 uniforms / onBeforeCompile 或 ShaderMaterial，编写 GLSL 与材质参数
3. 创建 OrbitControls（及 Raycaster 等交互控件，若源码包含）
4. 在 `requestAnimationFrame` 循环中更新状态并 render（Cesium 为 `viewer.render` 或自动渲染）

## 代码要点

```js
import * as THREE from 'three'
import { OrbitControls } from 'three/addons/controls/OrbitControls.js'

let scene, camera, renderer, controls
let fakeLights = []
const clock = new THREE.Clock()

init()
animate()

function init() {
    scene = new THREE.Scene()
    scene.background = new THREE.Color('#0a0a1a')

    camera = new THREE.PerspectiveCamera(
        60,
        window.innerWidth / window.innerHeight,
        0.1,
        1000
    )
    camera.position.set(0, 15, 30)

    renderer = new THREE.WebGLRenderer({ antialias: true, preserveDrawingBuffer: true })
    renderer.setSize(window.innerWidth, window.innerHeight)
    renderer.setPixelRatio(window.devicePixelRatio)
    document.body.appendChild(renderer.domElement)

    controls = new OrbitControls(camera, renderer.domElement)
    controls.enableDamping = true
    createFakeLights()
}

function createGlowTexture(color) {
    const canvas = document.createElement('canvas')
    canvas.width = 256
    canvas.height = 256
    const ctx = canvas.getContext('2d')

    const gradient = ctx.createRadialGradient(128, 128, 0, 128, 128, 128)
    gradient.addColorStop(0, color)
    gradient.addColorStop(0.3, color.replace('1)', '0.6)'))
    gradient.addColorStop(0.6, color.replace('1)', '0.2)'))
    gradient.addColorStop(1, color.replace('1)', '0)'))

    ctx.fillStyle = gradient
    ctx.fillRect(0, 0, 256, 256)

    const texture = new THREE.CanvasTexture(canvas)
    texture.needsUpdate = true
    return texture
}

function createFakeLight(position, color, size) {
    const group = new THREE.Group()

    const glowTexture = createGlowTexture(color)

    const coreMaterial = new THREE.ShaderMaterial({
        uniforms: {
            uTexture: { value: glowTexture },
            uOpacity: { value: 1.0 },
            uTime: { value: 0 }
        },
        vertexShader: `
                    varying vec2 vUv;
                    void main() {
                        vUv = uv;
                        gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0);
                    }
                `,
        fragmentShader: `
                    uniform sampler2D uTexture;
                    uniform float uOpacity;
                    uniform float uTime;
                    varying vec2 vUv;

                    void main() {
                        vec4 texColor = texture2D(uTexture, vUv);
                        float pulse = 0.8 + 0.2 * sin(uTime * 2.0);
                        gl_FragColor = vec4(texColor.rgb, texColor.a * uOpacity * pulse);
                    }
                `,
        transparent: true,
        blending: THREE.AdditiveBlending,
        depthWrite: false,
        side: THREE.DoubleSide
    })

    const coreGeometry = new THREE.PlaneGeometry(size, size)
    const core = new THREE.Mesh(coreGeometry, coreMaterial)
    core.position.copy(position)
    core.lookAt(camera.position)
    group.add(core)

    for (let i = 1; i <= 3; i++) {
        const layerMaterial = coreMaterial.clone()
        layerMaterial.uniforms.uOpacity.value = 0.3 / i
        const layerGeometry = new THREE.PlaneGeometry(size * (1 + i * 0.5), size * (1 + i * 0.5))
        const layer = new THREE.Mesh(layerGeometry, layerMaterial)
        layer.position.copy(position)
        layer.position.y -= i * 0.1
        layer.lookAt(camera.position)
        group.add(layer)
    }

    return { group, core, material: coreMaterial }
}

function createFakeLights() {
    const lightConfigs = [
        { pos: new THREE.Vector3(-10, 5, -10), color: 'rgba(255, 100, 100, 1)', size: 8 },
        { pos: new THREE.Vector3(10, 5, -10), color: 'rgba(100, 255, 100, 1)', size: 8 },
        { pos: new THREE.Vector3(0, 8, 10), color: 'rgba(100, 100, 255, 1)', size: 10 },
        { pos: new THREE.Vector3(-15, 3, 5), color: 'rgba(255, 200, 100, 1)', size: 6 },
        { pos: new THREE.Vector3(15, 3, 5), color: 'rgba(255, 100, 200, 1)', size: 6 }
    ]

    lightConfigs.forEach(config => {
        const fakeLight = createFakeLight(config.pos, config.color, config.size)
        scene.add(fakeLight.group)
        fakeLights.push(fakeLight)
    })
}

function animate() {
    requestAnimationFrame(animate)

    const elapsed = clock.getElapsedTime()

    fakeLights.forEach(fakeLight => {
        fakeLight.material.uniforms.uTime.value = elapsed
        fakeLight.core.lookAt(camera.position)
    })

    controls.update()
    renderer.render(scene, camera)
}
```

完整源码：[GitHub](https://github.com/z2586300277/three-cesium-examples/blob/dev/threeExamples/shader/shaderLight.js)

## 小结

- 本文提供 **着色器光效** 完整 Three.js 源码与在线 Demo，建议先运行案例再改 uniform/参数做二次实验
- 更多 Three.js 实战案例见 [three-cesium-examples 合集](https://threehub.cn) 与 [GitHub 开源仓库](https://github.com/z2586300277/three-cesium-examples)

