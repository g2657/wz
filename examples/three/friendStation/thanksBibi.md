---
title: "Three.js 感谢来自BiBi的支持教程"
description: "详解 Three.js 感谢来自BiBi的支持：原场景渲染后经 EffectComposer 叠加 Bloom/模糊等全屏后期，涵盖 EffectComposer、UnrealBloomPass、OrbitControls 等关键实现，附完整源码与在线 Demo，适合 Three.js 资源 学习与二次开发。"
head:
  - - meta
    - name: keywords
      content: "Three.js,感谢来自BiBi的支持,WebGL,源码,教程,在线案例,EffectComposer,后期处理,Bloom,辉光,OrbitControls,相机控制"
outline: deep
---

### 感谢来自BiBi的支持 · *Thanks BiBi* · [▶ 在线运行案例](https://z2586300277.github.io/three-cesium-examples/#/codeMirror?navigation=ThreeJS&classify=friendStation&id=thanksBibi)
- **案例合集：** [三维可视化功能案例（threehub.cn）](https://threehub.cn)

- **开源仓库github地址：** https://github.com/z2586300277/three-cesium-examples

- **400个案例代码: ** [网盘链接](https://pan.quark.cn/s/201da5c82fec)

![感谢来自BiBi的支持](https://z2586300277.github.io/three-cesium-examples/threeExamples/application/thanksBibi.jpg)

## 你将学到什么

- EffectComposer 多 Pass 后期处理管线
- UnrealBloomPass 辉光 Bloom 效果
- OrbitControls 相机轨道交互
- CSS2D/3D 标签 DOM 叠加
- `requestAnimationFrame` 渲染循环与 `resize` 自适应

## 效果说明

本案例演示 **感谢来自BiBi的支持** 效果：原场景渲染后经 EffectComposer 叠加 Bloom/模糊等全屏后期；核心用到 EffectComposer、UnrealBloomPass、OrbitControls。建议先打开文首在线案例查看动态画面，再对照下方源码逐步理解。

## 核心概念

- **Scene / Camera / WebGLRenderer** 构成最小渲染闭环；大场景可开 `logarithmicDepthBuffer` 缓解 Z-fighting。
- **EffectComposer** 以多 Pass 链式渲染：RenderPass → 特效 Pass → 输出屏幕，替代直接 `renderer.render`。
- **OrbitControls** 提供轨道旋转/缩放；开启 `enableDamping` 后需在 animate 中 `controls.update()`。

## 实现步骤

1. 搭建灯光与环境（如有）
2. requestAnimationFrame 循环 update + render

## 代码要点

```js
import * as THREE from 'three'
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls.js'
import { FontLoader } from 'three/addons/loaders/FontLoader.js';
import { TextGeometry } from 'three/addons/geometries/TextGeometry.js';
import { CSS3DRenderer, CSS3DObject } from 'three/examples/jsm/renderers/CSS3DRenderer.js'
import { UnrealBloomPass } from 'three/examples/jsm/postprocessing/UnrealBloomPass.js';
import { EffectComposer } from 'three/examples/jsm/postprocessing/EffectComposer.js';
import { RenderPass } from 'three/examples/jsm/postprocessing/RenderPass.js';

const DOM = document.getElementById('box')

const scene = new THREE.Scene()

const camera = new THREE.PerspectiveCamera(75, DOM.clientWidth / DOM.clientHeight, 0.1, 10000)

camera.position.set(0, 0, 1200)

const renderer = new THREE.WebGLRenderer({ antialias: true, alpha: true , logarithmicDepthBuffer: true})

renderer.setSize(DOM.clientWidth, DOM.clientHeight)

renderer.setPixelRatio( window.devicePixelRatio * 2)

DOM.appendChild(renderer.domElement)

scene.add(new THREE.AmbientLight(0xffffff, 2))

const controls = new OrbitControls(camera, renderer.domElement)

controls.enableDamping = true

const renderPass = new RenderPass(scene, camera);

const bloomPass = new UnrealBloomPass(new THREE.Vector2(DOM.clientWidth, DOM.clientHeight), 0.5, 0.4, 0.1);

const composer = new EffectComposer(renderer);

composer.addPass(renderPass);

composer.addPass(bloomPass);

// Css3DOM
const css3DRender = setCss3DRenderer(DOM)

// 添加内嵌 iframe
const iframeDOM = document.createElement('iframe')

iframeDOM.src = '//player.bilibili.com/player.html?isOutside=true&aid=116435523210402&bvid=BV1h3oTB7EER&cid=37658037265&p=1'

iframeDOM.style.width = '100%'

iframeDOM.style.height = '80%'

iframeDOM.style.border = 'none'

const mesh = new CSS3DObject(iframeDOM)

mesh.scale.multiplyScalar(1.2)

mesh.position.y -= 120

scene.add(mesh)

// 添加文字
const loader = new FontLoader()

loader.load(`https://z2586300277.github.io/3d-file-server/` + 'files/json/font.json', font => createText(font))

animate()

function animate() {

    requestAnimationFrame(animate)

    controls.update()

    renderer.render(scene, camera)

    composer.render()

    css3DRender.render(scene, camera) // Css3D渲染

}

window.onresize = () => {

    renderer.setSize(box.clientWidth, box.clientHeight)

    camera.aspect = box.clientWidth / box.clientHeight

    camera.updateProjectionMatrix()

    css3DRender.resize()

}

function createText(font) {

    const text =

    `           Three-Cesium-Examples 

                Thanks From BiBi







    Stars Collect Forward Comment

`

    const geometry = new TextGeometry(text, {

        font: font,

        size: 80,

        depth: 5,

        curveSegments: 12,

        bevelEnabled: true,

        bevelThickness: 8,

        bevelSize: 3,

        bevelOffset: 0,

        bevelSegments: 5

    })

    geometry.center()

    const mesh = new THREE.Mesh(geometry, new THREE.MeshStandardMaterial({color:'pink'}))

    scene.add(mesh)

}

/* css3d 渲染 */
function setCss3DRenderer(DOM) {

    const css3DRender = new CSS3DRenderer()

    css3DRender.resize = () => {

        css3DRender.setSize(DOM.clientWidth, DOM.clientHeight)

        css3DRender.domElement.style.zIndex = 0

        css3DRender.domElement.style.position = 'relative'

        css3DRender.domElement.style.top = -DOM.clientHeight + 'px'

        css3DRender.domElement.style.height = DOM.clientHeight + 'px'

        css3DRender.domElement.style.width = DOM.clientWidth + 'px'

        css3DRender.domElement.style.pointerEvents = 'none'

    }

    css3DRender.resize()

    DOM.appendChild(css3DRender.domElement)

    return css3DRender

}
```

完整源码：[GitHub](https://github.com/z2586300277/three-cesium-examples/blob/dev/threeExamples/application/thanksBibi.js)

## 小结

- 本文提供 **感谢来自BiBi的支持** 完整 Three.js 源码与在线 Demo，建议先运行案例再改 uniform/参数做二次实验
- 更多 Three.js 实战案例见 [three-cesium-examples 合集](https://threehub.cn) 与 [GitHub 开源仓库](https://github.com/z2586300277/three-cesium-examples)

