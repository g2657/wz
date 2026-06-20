---
title: "Three.js 多视图教程"
description: "详解 Three.js 多视图：原场景渲染后经 EffectComposer 叠加 Bloom/模糊等全屏后期，涵盖 EffectComposer、UnrealBloomPass、OrbitControls 等关键实现，附完整源码与在线 Demo，适合 Three.js 扩展 学习与二次开发。"
head:
  - - meta
    - name: keywords
      content: "Three.js,多视图,WebGL,源码,教程,在线案例,EffectComposer,后期处理,Bloom,辉光,OrbitControls,相机控制"
outline: deep
---

### 多视图 · *Mult Views* · [▶ 在线运行案例](https://z2586300277.github.io/three-cesium-examples/#/codeMirror?navigation=ThreeJS&classify=expand&id=multViews)
- **案例合集：** [三维可视化功能案例（threehub.cn）](https://threehub.cn)

- **开源仓库github地址：** https://github.com/z2586300277/three-cesium-examples

- **400个案例代码: ** [网盘链接](https://pan.quark.cn/s/201da5c82fec)

![多视图](https://z2586300277.github.io/three-cesium-examples/threeExamples/expand/multViews.jpg)

## 你将学到什么

- EffectComposer 多 Pass 后期处理管线
- UnrealBloomPass 辉光 Bloom 效果
- OrbitControls 相机轨道交互
- glTF/Draco 模型加载与优化
- 骨骼动画与 AnimationMixer
- `requestAnimationFrame` 渲染循环与 `resize` 自适应

## 效果说明

本案例演示 **多视图** 效果：原场景渲染后经 EffectComposer 叠加 Bloom/模糊等全屏后期；核心用到 EffectComposer、UnrealBloomPass、OrbitControls。建议先打开文首在线案例查看动态画面，再对照下方源码逐步理解。

## 核心概念
- **Scene / Camera / WebGLRenderer** 构成最小渲染闭环；大场景可开 `logarithmicDepthBuffer` 缓解 Z-fighting。
- **EffectComposer** 以多 Pass 链式渲染：RenderPass → 特效 Pass → 输出屏幕，替代直接 `renderer.render`。
- **OrbitControls** 提供轨道旋转/缩放；开启 `enableDamping` 后需在 animate 中 `controls.update()`。

## 实现步骤

1. 搭建 Scene、PerspectiveCamera、WebGLRenderer，挂载 canvas 并处理 `resize`
2. 异步加载模型 / 3D Tiles / GeoJSON 等资源并加入 scene 或 entities
3. 组装 EffectComposer Pass 链，在 animate 中调用 `composer.render()`
4. 创建 OrbitControls（及 Raycaster 等交互控件，若源码包含）
5. 在 `requestAnimationFrame` 循环中更新状态并 render（Cesium 为 `viewer.render` 或自动渲染）

## 代码要点

```js
import * as THREE from 'three'
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls.js'
import { EffectComposer } from 'three/examples/jsm/postprocessing/EffectComposer.js'
import { RenderPass } from 'three/examples/jsm/postprocessing/RenderPass.js'
import { UnrealBloomPass } from 'three/examples/jsm/postprocessing/UnrealBloomPass.js'
import { GLTFLoader } from 'three/examples/jsm/loaders/GLTFLoader.js'
import { DRACOLoader } from 'three/examples/jsm/loaders/DRACOLoader.js'

const box = document.getElementById('box')
const scene = new THREE.Scene()

const renderer = new THREE.WebGLRenderer()
renderer.setSize(box.clientWidth, box.clientHeight)
box.appendChild(renderer.domElement)

// 视口尺寸计算（主视口占 65%，下方三个子视口各占 35%/3）
const layout = () => {
    const W = box.clientWidth, H = box.clientHeight
    const subH = Math.floor(H * 0.35)
    return { W, H, subH, mainH: H - subH, subW: Math.floor(W / 3) }
}

// composer 工厂
const makeComposer = (cam, w, h) => {
    const c = new EffectComposer(renderer)
    c.addPass(new RenderPass(scene, cam))
    c.addPass(new UnrealBloomPass(new THREE.Vector2(w, h), 0.8, 0, 0))
    c.setSize(w, h)
    return c
}

// 主相机（透视）
const { W: W0, mainH: mainH0, subH: subH0, subW: subW0 } = layout()
const camera = new THREE.PerspectiveCamera(75, W0 / mainH0, 0.1, 1000)
camera.position.set(400, 400, 400)
const mainComposer = makeComposer(camera, W0, mainH0)

// 正交相机工厂（前视图 / 左视图 / 俯视图）
const ORTHO = 500
const makeOrtho = (pos, up) => {
    const aspect = subW0 / subH0
    const cam = new THREE.OrthographicCamera(-ORTHO * aspect, ORTHO * aspect, ORTHO, -ORTHO, 0.1, 5000)
    cam.position.copy(pos); cam.up.copy(up); cam.lookAt(0, 0, 0)
    return cam
}
const subCams = [
    makeOrtho(new THREE.Vector3(0, 0, 2000), new THREE.Vector3(0, 1, 0)),   // 前视图
    makeOrtho(new THREE.Vector3(2000, 0, 0), new THREE.Vector3(0, 1, 0)),   // 左视图
    makeOrtho(new THREE.Vector3(0, 2000, 0), new THREE.Vector3(0, 0, -1)),  // 俯视图
]
const subComposers = subCams.map(cam => makeComposer(cam, subW0, subH0))

// 控制器：点击哪个视口就激活哪个
const mainControls = new OrbitControls(camera, renderer.domElement)
const subControls = subCams.map(cam => {
    const ctrl = new OrbitControls(cam, renderer.domElement)
    ctrl.enabled = false
    return ctrl
})
renderer.domElement.addEventListener('pointerdown', e => {
    const rect = renderer.domElement.getBoundingClientRect()
    const { mainH, subW } = layout()
    const py = e.clientY - rect.top
    const px = e.clientX - rect.left
    mainControls.enabled = false
    subControls.forEach(c => c.enabled = false)
    if (py < mainH) mainControls.enabled = true
    else subControls[Math.min(Math.floor(px / subW), 2)].enabled = true
}, { capture: true })

// 动画循环
const clock = new THREE.Clock()
let mixer = null

function animate() {
    requestAnimationFrame(animate)
    const { W, subH, mainH, subW } = layout()

    renderer.setScissorTest(true)

    renderer.setViewport(0, subH, W, mainH)
    renderer.setScissor(0, subH, W, mainH)
    mainComposer.render()

    subComposers.forEach((c, i) => {
        renderer.setViewport(i * subW, 0, subW, subH)
        renderer.setScissor(i * subW, 0, subW, subH)
        c.render()
    })

    mainControls.update()
    subControls.forEach(c => c.update())
    if (mixer) mixer.update(clock.getDelta())
}
animate()

window.onresize = () => {
    const { W, H, subH, mainH, subW } = layout()
    renderer.setSize(W, H)
    camera.aspect = W / mainH
    camera.updateProjectionMatrix()
    mainComposer.setSize(W, mainH)
    subCams.forEach(cam => {
        cam.left = -ORTHO * subW / subH; cam.right = ORTHO * subW / subH
        cam.updateProjectionMatrix()
    })
    subComposers.forEach(c => c.setSize(subW, subH))
}

// 加载模型
const loader = new GLTFLoader()
loader.setDRACOLoader(new DRACOLoader().setDecoderPath(FILE_HOST + 'js/three/draco/'))

const textureCube = new THREE.CubeTextureLoader().load(
    [1, 2, 3, 4, 5, 6].map(k => FILE_HOST + 'files/sky/skyBox0/' + k + '.png')
)

loader.load(FILE_HOST + '/files/model/LittlestTokyo.glb', gltf => {
    gltf.scene.traverse(child => {
        if (child.isMesh) child.material.envMap = textureCube
    })
    scene.add(gltf.scene)
    mixer = new THREE.AnimationMixer(gltf.scene)
    gltf.animations.forEach(clip => mixer.clipAction(clip).play())
})
```

完整源码：[GitHub](https://github.com/z2586300277/three-cesium-examples/blob/dev/threeExamples/expand/multViews.js)

## 小结

- 本文提供 **多视图** 完整 Three.js 源码与在线 Demo，建议先运行案例再改 uniform/参数做二次实验
- 更多 Three.js 实战案例见 [three-cesium-examples 合集](https://threehub.cn) 与 [GitHub 开源仓库](https://github.com/z2586300277/three-cesium-examples)

