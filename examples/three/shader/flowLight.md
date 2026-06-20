---
title: "Three.js 流光教程"
description: "详解 Three.js 流光：原场景渲染后经 EffectComposer 叠加 Bloom/模糊等全屏后期，涵盖 EffectComposer、UnrealBloomPass、OrbitControls 等关键实现，附完整源码与在线 Demo，适合 Three.js 着色器 学习与二次开发。"
head:
  - - meta
    - name: keywords
      content: "Three.js,流光,WebGL,源码,教程,在线案例,EffectComposer,后期处理,Bloom,辉光,OrbitControls,相机控制"
outline: deep
---

### 流光 · *Flow Light* · [▶ 在线运行案例](https://z2586300277.github.io/three-cesium-examples/#/codeMirror?navigation=ThreeJS&classify=shader&id=flowLight)
- **案例合集：** [三维可视化功能案例（threehub.cn）](https://threehub.cn)

- **开源仓库github地址：** https://github.com/z2586300277/three-cesium-examples

- **400个案例代码: ** [网盘链接](https://pan.quark.cn/s/201da5c82fec)

![流光](https://z2586300277.github.io/three-cesium-examples/threeExamples/shader/flowLight.jpg)

## 你将学到什么

- EffectComposer 多 Pass 后期处理管线
- UnrealBloomPass 辉光 Bloom 效果
- OrbitControls 相机轨道交互
- GSAP 时间轴与补间动画
- `requestAnimationFrame` 渲染循环与 `resize` 自适应

## 效果说明

本案例演示 **流光** 效果：原场景渲染后经 EffectComposer 叠加 Bloom/模糊等全屏后期；核心用到 EffectComposer、UnrealBloomPass、OrbitControls。建议先打开文首在线案例查看动态画面，再对照下方源码逐步理解。

## 核心概念
- **Scene / Camera / WebGLRenderer** 构成最小渲染闭环；大场景可开 `logarithmicDepthBuffer` 缓解 Z-fighting。
- **EffectComposer** 以多 Pass 链式渲染：RenderPass → 特效 Pass → 输出屏幕，替代直接 `renderer.render`。
- **OrbitControls** 提供轨道旋转/缩放；开启 `enableDamping` 后需在 animate 中 `controls.update()`。

## 实现步骤

1. 创建 OrbitControls 并处理 resize
2. composer.addPass 串联后处理
3. mixer.update(delta) 或 gsap.to 驱动属性

## 代码要点

```js
import {
	AmbientLight,
	Color,
	DirectionalLight,
	DoubleSide,
	Mesh,
	MeshBasicMaterial,
	PerspectiveCamera,
	Scene,
	TextureLoader,
	TorusKnotGeometry,
	Vector2,
	WebGLRenderer
} from 'three'
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls.js'
import gsap from 'gsap'
import { EffectComposer } from 'three/examples/jsm/postprocessing/EffectComposer.js'
import { RenderPass } from 'three/examples/jsm/postprocessing/RenderPass.js'
import { UnrealBloomPass } from 'three/examples/jsm/postprocessing/UnrealBloomPass.js'
import { OutputPass } from 'three/examples/jsm/postprocessing/OutputPass.js'

const size = { width: window.innerWidth, height: window.innerHeight }
const scene = new Scene()
scene.background = new Color('black')

const camera = new PerspectiveCamera(50, size.width / size.height, 1, 10000)
camera.position.set(0, 0, 50)

const renderer = new WebGLRenderer({ antialias: true, alpha: true , logarithmicDepthBuffer: true})
renderer.setSize(size.width, size.height)
renderer.setPixelRatio(window.devicePixelRatio)
document.body.appendChild(renderer.domElement)

const controls = new OrbitControls(camera, renderer.domElement)
controls.enableDamping = true

const textureLoader = new TextureLoader()
const lineTexture = textureLoader.load(FILE_HOST + 'images/channels/flowLight.png')
lineTexture.offset.x = -0.6

const geometry = new TorusKnotGeometry( 10, 0.2, 800, 16 )

const material = new MeshBasicMaterial({ color: 0xffffff, map: lineTexture, side: DoubleSide })
const torus = new Mesh(geometry, material)
scene.add(torus)

gsap.to(lineTexture.offset, {
	x: 0.6,
	duration: 5,
	repeat: -1
})

const renderPass = new RenderPass(scene, camera)
const composer = new EffectComposer(renderer)
const bloomPass = new UnrealBloomPass(new Vector2(size.width, size.height), 3, 0.8, 0.85)
const outputPass = new OutputPass()
composer.addPass(renderPass)
composer.addPass(bloomPass)
composer.addPass(outputPass)

const animate = () => {
	requestAnimationFrame(animate)
	controls.update()
	composer.render()
}

animate()
```

完整源码：[GitHub](https://github.com/z2586300277/three-cesium-examples/blob/dev/threeExamples/shader/flowLight.js)

## 小结

- 本文提供 **流光** 完整 Three.js 源码与在线 Demo，建议先运行案例再改 uniform/参数做二次实验
- 更多 Three.js 实战案例见 [three-cesium-examples 合集](https://threehub.cn) 与 [GitHub 开源仓库](https://github.com/z2586300277/three-cesium-examples)

