---
title: "Three.js CSS 元素教程"
description: "详解 Three.js CSS 元素：基于 WebGL 实现「CSS 元素」可视化效果，附完整可运行源码，涵盖 OrbitControls、CSS2D/3D 等关键实现，附完整源码与在线 Demo，适合 Three.js 基础 学习与二次开发。"
head:
  - - meta
    - name: keywords
      content: "Three.js,CSS 元素,WebGL,源码,教程,在线案例,OrbitControls,相机控制,CSS2D,HTML标注"
outline: deep
---

### CSS 元素 · *CSS2D / CSS3D* · [▶ 在线运行案例](https://z2586300277.github.io/three-cesium-examples/#/codeMirror?navigation=ThreeJS&classify=basic&id=cssElement)
- **案例合集：** [三维可视化功能案例（threehub.cn）](https://threehub.cn)

- **开源仓库github地址：** https://github.com/z2586300277/three-cesium-examples

- **400个案例代码: ** [网盘链接](https://pan.quark.cn/s/201da5c82fec)

## 你将学到什么

- **CSS2DObject** — DOM 始终面向相机（billboard 标签）
- **CSS3DObject** — DOM 参与 3D 变换（可旋转进场景）
- 三渲染器同帧：`WebGLRenderer` + `css2DRender` + `css3DRender`

## 效果说明

5 组标签沿 Z / Y 排列：**2D 标签** 平贴屏幕方向，**3D 标签** 缩小后立在场景中，可看出透视差异。

## 核心概念

```js
// 每帧必须三个都 render
renderer.render(scene, camera);
css3DRender.render(scene, camera);
css2DRender.render(scene, camera);
```

CSS 层叠在 WebGL canvas 之上，通过 `position:relative; top:-height` 与 canvas 对齐。`pointerEvents: 'none'` 避免挡住 WebGL 操作（2D 标签可单独 `auto` 接收点击）。

对比 [screenCoord](/examples/three/basic/screenCoord) 手算投影，本方案由引擎处理矩阵。

## 代码要点

```js
import * as THREE from 'three'
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls.js'
import { CSS2DRenderer, CSS2DObject } from 'three/examples/jsm/renderers/CSS2DRenderer.js'
import { CSS3DRenderer, CSS3DObject } from 'three/examples/jsm/renderers/CSS3DRenderer.js'

const DOM = document.getElementById('box')

const scene = new THREE.Scene()

const camera = new THREE.PerspectiveCamera(75, DOM.clientWidth / DOM.clientHeight, 0.1, 1000)

camera.position.set(10, 10, 10)

const renderer = new THREE.WebGLRenderer()

renderer.setSize(DOM.clientWidth, DOM.clientHeight)

DOM.appendChild(renderer.domElement)

new OrbitControls(camera, renderer.domElement)

scene.add(new THREE.AxesHelper(500))

// Css3DOM
const css3DRender = setCss3DRenderer(DOM)

// Css2DOM
const css2DRender = setCss2DRenderer(DOM)

const setCss2dDOM = (DOM, position) => {

    DOM.style.pointerEvents = 'auto'

    const mesh = new CSS2DObject(DOM)

    mesh.position.copy(position)

    scene.add(mesh)

    return mesh

}


const setCss3dDOM = (DOM, position) => {

    const mesh = new CSS3DObject(DOM)

    mesh.position.copy(position)

    scene.add(mesh)

    return mesh

}

for (let i = 0; i < 5; i++) {

    setCss2dDOM(createDom('2D' + i), { x: 0, y: 0, z: i * 2 }) // 2d dom

    setCss3dDOM(createDom('3D' + i), { x: 0, y: i * 2, z: 0 }).scale.multiplyScalar(0.02) // 3d dom

}

animate()

function animate() {

    requestAnimationFrame(animate)

    renderer.render(scene, camera)

    css3DRender.render(scene, camera) // Css3D渲染

    css2DRender.render(scene, camera) // Css2D渲染

}

window.onresize = () => {

    renderer.setSize(box.clientWidth, box.clientHeight)

    camera.aspect = box.clientWidth / box.clientHeight

    camera.updateProjectionMatrix()

    css3DRender.resize()

    css2DRender.resize()

}



// 创建dom
function createDom(text) {

    const div = document.createElement('div')

    const img = document.createElement('img')

    img.src = HOST + '/files/author/z2586300277.png'

    img.style.width = '50px'

    img.style.height = '50px'

    div.appendChild(img)

    div.innerHTML += text

    div.style.color = 'white'

    return div

}

/* css2d渲染 */
function setCss2DRenderer(DOM) {

    const css2DRender = new CSS2DRenderer()

    css2DRender.resize = () => {

        css2DRender.setSize(DOM.clientWidth, DOM.clientHeight)

        css2DRender.domElement.style.zIndex = 0

        css2DRender.domElement.style.position = 'relative'

        css2DRender.domElement.style.top = -DOM.clientHeight * 2 + 'px'

        css2DRender.domElement.style.height = DOM.clientHeight + 'px'

        css2DRender.domElement.style.width = DOM.clientWidth + 'px'

        css2DRender.domElement.style.pointerEvents = 'none'

    }

    css2DRender.resize()

    DOM.appendChild(css2DRender.domElement)

    return css2DRender

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

完整源码：[GitHub](https://github.com/z2586300277/three-cesium-examples/blob/dev/threeExamples/basic/cssElement.js)

## 小结

- 本文提供 **CSS 元素** 完整 Three.js 源码与在线 Demo，建议先运行案例再改 uniform/参数做二次实验
- 更多 Three.js 实战案例见 [three-cesium-examples 合集](https://threehub.cn) 与 [GitHub 开源仓库](https://github.com/z2586300277/three-cesium-examples)

