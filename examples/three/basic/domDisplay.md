---
title: "Three.js DOM 遮挡教程"
description: "详解 Three.js DOM 遮挡：基于 WebGL 实现「DOM 遮挡」可视化效果，附完整可运行源码，涵盖 OrbitControls、Raycaster、CSS2D/3D 等关键实现，附完整源码与在线 Demo，适合 Three.js 基础 学习与二次开发。"
head:
  - - meta
    - name: keywords
      content: "Three.js,DOM 遮挡,WebGL,源码,教程,在线案例,OrbitControls,相机控制,Raycaster,射线检测,CSS2D,HTML标注"
outline: deep
---

### DOM 遮挡 · *DOM Occlusion* · [▶ 在线运行案例](https://z2586300277.github.io/three-cesium-examples/#/codeMirror?navigation=ThreeJS&classify=basic&id=domDisplay)
- **案例合集：** [三维可视化功能案例（threehub.cn）](https://threehub.cn)

- **开源仓库github地址：** https://github.com/z2586300277/three-cesium-examples

- **400个案例代码: ** [网盘链接](https://pan.quark.cn/s/201da5c82fec)

## 你将学到什么

- CSS3D 标签被 **3D 物体挡住** 时隐藏（opacity=0）
- 相机 → 标签方向 **Raycaster** 检测中间 Mesh
- 100 个 CSS3D 标签 + 随机 TorusKnot 障碍

## 效果说明

大量「顽皮宝」DOM 标签散布在场景；当 **结形体** 挡在相机与标签之间时，标签淡出，模拟真实遮挡。

## 核心概念

```js
function createRender(mesh) {
    const direction = new THREE.Vector3()
        .subVectors(mesh.position, camera.position).normalize();
    const raycaster = new THREE.Raycaster(
        camera.position, direction, 0,
        mesh.position.distanceTo(camera.position)
    );
    const hits = raycaster.intersectObjects(meshs);
    mesh.div.style.opacity = hits.length > 0 ? 0 : 1;
}
```

射线长度 = 相机到标签距离，只检测 **之间** 的物体，不检测标签后方。

## 代码要点

```js
import * as THREE from 'three'
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls.js'
import { CSS3DRenderer, CSS3DObject } from 'three/examples/jsm/renderers/CSS3DRenderer.js'

const DOM = document.getElementById('box')

const scene = new THREE.Scene()

const camera = new THREE.PerspectiveCamera(50, DOM.clientWidth / DOM.clientHeight, 0.1, 1000)

camera.position.set(20, 20, 20)

const renderer = new THREE.WebGLRenderer({ antialias: true, alpha: true, logarithmicDepthBuffer: true })

renderer.setSize(DOM.clientWidth, DOM.clientHeight)

DOM.appendChild(renderer.domElement)

const controls = new OrbitControls(camera, renderer.domElement)

controls.enableDamping = true

scene.add( new THREE.GridHelper(50, 10))

const css3DRender = setCss3DRenderer(DOM)

const setCss3dDOM = (div, position) => {

    const mesh = new CSS3DObject(div)

    div.style.pointerEvents = 'none'

    mesh.div = div

    mesh.position.copy(position)

    scene.add(mesh)

    return mesh

}

const R = () => Math.random() * 20 - 10

const list = []

const meshs = []

function createRender(mesh) {

    const direction = new THREE.Vector3().subVectors(mesh.position, camera.position).normalize()

    const raycaster = new THREE.Raycaster(camera.position, direction, 0, mesh.position.distanceTo(camera.position))

    const intersects = raycaster.intersectObjects(meshs)

    mesh.div.style.opacity = intersects.length > 0 ? 0 : 1

}

for (let i = 0; i < 100; i++) {

    const mesh = setCss3dDOM(createDom('顽皮宝' + i), { x: R(), y: R(), z: R() })

    mesh.scale.multiplyScalar(0.02)

    mesh.lookAt(R(), R(), R())

    mesh.update = () => createRender(mesh)

    list.push(mesh)

    if (i % 40 === 0) {

        const boxGeometry = new THREE.TorusKnotGeometry( 4, 0.6, 100, 16 )

        const boxMesh = new THREE.Mesh(boxGeometry, new THREE.MeshBasicMaterial({ color: Math.random() * 0xffffff , transparent: true, opacity: 0.45 }))

        boxMesh.position.copy({ x: R(), y: R(), z: R() })

        meshs.push(boxMesh)

        scene.add(boxMesh)

    }

}

// 创建dom
function createDom(text) {

    const div = document.createElement('div')

    div.style.position = 'absolute'

    div.style.transition = 'all 0.2s'

    const img = document.createElement('img')

    img.src = HOST + '/files/author/flowers-10.jpg'

    img.style.width = '50px'

    img.style.height = '50px'

    div.appendChild(img)

    div.innerHTML += text

    div.style.color = 'white'

    document.body.appendChild(div)

    return div

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

animate()

function animate() {

    meshs.forEach(mesh => { mesh.rotation.y += 0.01; mesh.rotation.x += 0.01 })

    list.forEach(mesh => mesh.update())

    requestAnimationFrame(animate)

    controls.update()

    renderer.render(scene, camera)

    css3DRender.render(scene, camera) 

}

window.onresize = () => {

    renderer.setSize(DOM.clientWidth, DOM.clientHeight)

    camera.aspect = DOM.clientWidth / DOM.clientHeight

    camera.updateProjectionMatrix()

    css3DRender.resize()

}
```

完整源码：[GitHub](https://github.com/z2586300277/three-cesium-examples/blob/dev/threeExamples/basic/domDisplay.js)

## 小结

- 本文提供 **DOM 遮挡** 完整 Three.js 源码与在线 Demo，建议先运行案例再改 uniform/参数做二次实验
- 更多 Three.js 实战案例见 [three-cesium-examples 合集](https://threehub.cn) 与 [GitHub 开源仓库](https://github.com/z2586300277/three-cesium-examples)

