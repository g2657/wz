---
title: "Three.js 官方选择辉光简化版教程"
description: "详解 Three.js 官方选择辉光简化版：原场景渲染后经 EffectComposer 叠加 Bloom/模糊等全屏后期，涵盖 ShaderMaterial、EffectComposer、UnrealBloomPass 等关键实现，附完整源码与在线 Demo，适合 Three.js 后期处理 学习与二次开发。"
head:
  - - meta
    - name: keywords
      content: "Three.js,官方选择辉光简化版,WebGL,源码,教程,在线案例,ShaderMaterial,自定义着色器,GLSL,EffectComposer,后期处理,Bloom"
outline: deep
---

### 官方选择辉光简化版 · *Three Bloom* · [▶ 在线运行案例](https://z2586300277.github.io/three-cesium-examples/#/codeMirror?navigation=ThreeJS&classify=effectComposer&id=threeSelectBloom)
- **案例合集：** [三维可视化功能案例（threehub.cn）](https://threehub.cn)

- **开源仓库github地址：** https://github.com/z2586300277/three-cesium-examples

- **400个案例代码: ** [网盘链接](https://pan.quark.cn/s/201da5c82fec)

![官方选择辉光简化版](https://z2586300277.github.io/three-cesium-examples/threeExamples/effectComposer/threeSelectBloom.jpg)

## 你将学到什么

- ShaderMaterial 自定义着色器实现核心视觉效果
- EffectComposer 多 Pass 后期处理管线
- UnrealBloomPass 辉光 Bloom 效果
- OrbitControls 相机轨道交互
- glTF/Draco 模型加载与优化
- Raycaster 鼠标拾取与交互
- `requestAnimationFrame` 渲染循环与 `resize` 自适应

## 效果说明

本案例演示 **官方选择辉光简化版** 效果：原场景渲染后经 EffectComposer 叠加 Bloom/模糊等全屏后期，支持鼠标拾取、绘制或拖拽交互；核心用到 ShaderMaterial、EffectComposer、UnrealBloomPass。建议先打开文首在线案例查看动态画面，再对照下方源码逐步理解。

## 核心概念
- **Scene / Camera / WebGLRenderer** 构成最小渲染闭环；大场景可开 `logarithmicDepthBuffer` 缓解 Z-fighting。
- **ShaderMaterial** 通过 `uniforms` + 自定义 GLSL 控制逐像素/逐点效果；透明粒子常配合 `depthTest: false`。
- **EffectComposer** 以多 Pass 链式渲染：RenderPass → 特效 Pass → 输出屏幕，替代直接 `renderer.render`。
- **OrbitControls** 提供轨道旋转/缩放；开启 `enableDamping` 后需在 animate 中 `controls.update()`。

## 实现步骤

1. 搭建 Scene、PerspectiveCamera、WebGLRenderer，挂载 canvas 并处理 `resize`
2. 异步加载模型 / 3D Tiles / GeoJSON 等资源并加入 scene 或 entities
3. 定义 uniforms / onBeforeCompile 或 ShaderMaterial，编写 GLSL 与材质参数
4. 组装 EffectComposer Pass 链，在 animate 中调用 `composer.render()`
5. 创建 OrbitControls（及 Raycaster 等交互控件，若源码包含）
6. 在 `requestAnimationFrame` 循环中更新状态并 render（Cesium 为 `viewer.render` 或自动渲染）

## 代码要点

```js
import * as THREE from "three"
import { GLTFLoader } from "three/addons/loaders/GLTFLoader.js"
import { OrbitControls } from "three/addons/controls/OrbitControls.js"
import { EffectComposer } from "three/addons/postprocessing/EffectComposer.js"
import { RenderPass } from "three/addons/postprocessing/RenderPass.js"
import { ShaderPass } from "three/addons/postprocessing/ShaderPass.js"
import { UnrealBloomPass } from "three/addons/postprocessing/UnrealBloomPass.js"

const renderer = new THREE.WebGLRenderer({ antialias: true, logarithmicDepthBuffer: true, })
renderer.setSize(window.innerWidth, window.innerHeight)
document.body.appendChild(renderer.domElement)

const scene = new THREE.Scene()
const camera = new THREE.PerspectiveCamera(40, window.innerWidth / window.innerHeight, 1, 100000)
camera.position.set(200, 200, 200)
new OrbitControls(camera, renderer.domElement)

const light = new THREE.DirectionalLight(0xffffff, 3)
light.position.set(100, 100, 100)
scene.add(light)

// 物体
new GLTFLoader().load(FILE_HOST + "files/model/Fox.glb", (gltf) => scene.add(gltf.scene))
const mesh = new THREE.Mesh(new THREE.BoxGeometry(20, 20, 20), new THREE.MeshStandardMaterial({ color: 0x00ff00 }))
mesh.position.set(50, 0, 0)
scene.add(mesh)

// 后期处理
const renderScene = new RenderPass(scene, camera)
const bloomPass = new UnrealBloomPass(new THREE.Vector2(window.innerWidth, window.innerHeight), 0.8, 0.4, 0.0)
const bloomComposer = new EffectComposer(renderer)
bloomComposer.renderToScreen = false
bloomComposer.addPass(renderScene)
bloomComposer.addPass(bloomPass)

const finalPass = new ShaderPass(
    new THREE.ShaderMaterial({
        uniforms: {
            baseTexture: { value: null },
            bloomTexture: { value: bloomComposer.renderTarget2.texture },
        },
        vertexShader: `
        varying vec2 vUv;
        void main() {
            vUv = uv;
            gl_Position = projectionMatrix * modelViewMatrix * vec4( position, 1.0 );
        }
        `,
        fragmentShader: `
        uniform sampler2D baseTexture;
        uniform sampler2D bloomTexture;
        varying vec2 vUv;
        void main() {
            gl_FragColor = ( texture2D( baseTexture, vUv ) + vec4( 1.0 ) * texture2D( bloomTexture, vUv ) );
        }
        `,
        defines: {},
    }),
    "baseTexture"
)
finalPass.needsSwap = true
const finalComposer = new EffectComposer(renderer)
finalComposer.addPass(renderScene)
finalComposer.addPass(finalPass)

// 点击切换辉光
window.addEventListener("click", onClick)
function onClick(event) {
    const raycaster = new THREE.Raycaster()
    const mouse = new THREE.Vector2(
        (event.offsetX / event.target.clientWidth) * 2 - 1,
        -(event.offsetY / event.target.clientHeight) * 2 + 1
    )
    raycaster.setFromCamera(mouse, camera)
    const intersects = raycaster.intersectObjects(scene.children)
    if (intersects.length > 0)  intersects[0].object.layers.toggle(1) // 切换图层
}

// 窗口大小变化
window.onresize = function () {
    camera.aspect = window.innerWidth / window.innerHeight
    camera.updateProjectionMatrix()
    renderer.setSize(window.innerWidth, window.innerHeight)
    bloomComposer.setSize(window.innerWidth, window.innerHeight)
    finalComposer.setSize(window.innerWidth, window.innerHeight)
}

// 辉光图层
const bloomLayer = new THREE.Layers()
bloomLayer.set(1)

const darkMaterial = new THREE.MeshBasicMaterial({ color: "black" })
const materials = {}

render()
function render() {
    requestAnimationFrame(render)
    scene.traverse(obj => {
        if (obj.isMesh && bloomLayer.test(obj.layers) === false) {
            materials[obj.uuid] = obj.material // 保存原材质
            obj.material = darkMaterial	// 替换材质
        }
    })
    bloomComposer.render() // 渲染到bloomComposer
    scene.traverse(obj => {
        if (materials[obj.uuid]) {
            obj.material = materials[obj.uuid] // 恢复原材质
            delete materials[obj.uuid] // 删除原材质
        }
    })
    finalComposer.render()
}
```

完整源码：[GitHub](https://github.com/z2586300277/three-cesium-examples/blob/dev/threeExamples/effectComposer/threeSelectBloom.js)

## 小结

- 本文提供 **官方选择辉光简化版** 完整 Three.js 源码与在线 Demo，建议先运行案例再改 uniform/参数做二次实验
- 更多 Three.js 实战案例见 [three-cesium-examples 合集](https://threehub.cn) 与 [GitHub 开源仓库](https://github.com/z2586300277/three-cesium-examples)

