---
title: "Three.js 绘制围栏教程"
description: "详解 Three.js 绘制围栏：支持鼠标拾取、绘制或拖拽交互，涵盖 OrbitControls、Raycaster、BufferGeometry 等关键实现，附完整源码与在线 Demo，适合 Three.js 应用场景 学习与二次开发。"
head:
  - - meta
    - name: keywords
      content: "Three.js,绘制围栏,WebGL,源码,教程,在线案例,OrbitControls,相机控制,Raycaster,射线检测,BufferGeometry"
outline: deep
---
### 绘制围栏 · *Draw Fence* · [▶ 在线运行案例](https://z2586300277.github.io/three-cesium-examples/#/codeMirror?navigation=ThreeJS&classify=application&id=drawFence)
- **案例合集：** [三维可视化功能案例（threehub.cn）](https://threehub.cn)

- **开源仓库github地址：** https://github.com/z2586300277/three-cesium-examples

- **400个案例代码: ** [网盘链接](https://pan.quark.cn/s/201da5c82fec)

![绘制围栏](https://z2586300277.github.io/three-cesium-examples/threeExamples/application/drawFence.jpg)

## 你将学到什么

- OrbitControls 相机轨道交互
- Raycaster 鼠标拾取与交互
- BufferGeometry 自定义顶点/索引数据
- `requestAnimationFrame` 渲染循环与 `resize` 自适应

## 效果说明

本案例演示 **绘制围栏** 效果：支持鼠标拾取、绘制或拖拽交互；核心用到 OrbitControls、Raycaster、BufferGeometry。建议先打开文首在线案例查看动态画面，再对照下方源码逐步理解。

## 核心概念

- **OrbitControls** 轨道旋转缩放；开 `enableDamping` 时每帧需 `controls.update()`。

## 实现步骤

1. 搭建 Scene / Camera / Renderer 与 OrbitControls
2. rAF 循环中 update 并 render

## 代码要点

```js
import * as THREE from 'three'
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls.js'

const box = document.getElementById('box')

const scene = new THREE.Scene()

const camera = new THREE.PerspectiveCamera(75, box.clientWidth / box.clientHeight, 0.1, 1000)

camera.position.set(0, 3, 3)

const renderer = new THREE.WebGLRenderer({ antialias: true, alpha: true, logarithmicDepthBuffer: true })

renderer.setSize(box.clientWidth, box.clientHeight)

box.appendChild(renderer.domElement)

const controls = new OrbitControls(camera, renderer.domElement)

controls.enableDamping = true

const plane = new THREE.PlaneGeometry(5, 5)

const material = new THREE.MeshBasicMaterial({ color: 'gray' })

const planeMesh = new THREE.Mesh(plane, material)

planeMesh.rotation.x -= Math.PI / 2

scene.add(planeMesh)

const texture = new THREE.TextureLoader().load(FILE_HOST + 'images/channels/wall_g.png')

texture.wrapT = THREE.RepeatWrapping

texture.repeat.y = 15

animate()

function animate() {

    texture.offset.y -= 0.005;

    requestAnimationFrame(animate)

    controls.update()

    renderer.render(scene, camera)

}

// 事件
const raycaster = new THREE.Raycaster()

const getPoint = event => {

    const mouse = new THREE.Vector2((event.offsetX / event.target.clientWidth) * 2 - 1, -(event.offsetY / event.target.clientHeight) * 2 + 1)

    raycaster.setFromCamera(mouse, camera)

    const intersects = raycaster.intersectObjects(scene.children)

    if (intersects.length > 0) return intersects[0].point

}

/* 开始绘制 */
const pointList = []; let drawMesh = null; let fenceHeight = 1

box.addEventListener('click', (event) => {

    const point = getPoint(event)

    if (!point) return

    point.y += 0.001

    pointList.push(point)

    if (pointList.length < 2) return

    const formatPoints = pointList.reduce((i, j) => {

        const k = new THREE.Vector3().copy(j)

        k.y += fenceHeight

        return [...i, k, j]

    }, [])

    const { indexGroup, faceGroup, uvGroup } = multShapeGroup(formatPoints)

    if (!drawMesh) {

        const geometry = multShapePlaneGeometry(faceGroup, indexGroup, uvGroup)

        const material = new THREE.MeshBasicMaterial({ side: THREE.DoubleSide, map: texture, transparent: true, color: Math.random() * 0xffffff })

        drawMesh = new THREE.Mesh(geometry, material)

        scene.add(drawMesh)

    }

    else updateMultShapePlaneGeometry(drawMesh.geometry, faceGroup, indexGroup, uvGroup)

})

/* 处理顶点算法 */
function multShapeGroup(formatPoints) {

    const { length } = formatPoints

    const indexGroup = formatPoints.map((_, k) => (k - 1 > -1 && k + 1 < length) && (k % 2 === 0 ? [k, k + 1, k - 1] : [k, k - 1, k + 1])).filter((i) => i).reduce((i, j) => [...i, ...j], [])

    const faceGroup = formatPoints.reduce((j, i) => [...j, i.x, i.y, i.z], [])

    const uvMaxMin = formatPoints.reduce((p, i) => ({ x: [...p['x'], i['x']], y: [...p['y'], i['y']], z: [...p['z'], i['z']] }), { x: [], y: [], z: [] })

    const Maxp = new THREE.Vector3(Math.max(...uvMaxMin.x), Math.max(...uvMaxMin.y), Math.max(...uvMaxMin.z))  // 最大点

    const Minp = new THREE.Vector3(Math.min(...uvMaxMin.x), Math.min(...uvMaxMin.y), Math.min(...uvMaxMin.z))  // 最小点

    const W = Maxp.x - Minp.x

    const H = Maxp.y - Minp.y

    const L = W > H ? W : H

    const uvGroup = formatPoints.map(i => new THREE.Vector2((i.x - Minp.x) / L, (i.y - Minp.y) / L)).reduce((i, j) => [...i, ...j], [])

    return { indexGroup, faceGroup, uvGroup }

}

/* 根据顶点组生成物体 */
function multShapePlaneGeometry(faceGroup, indexGroup, uvGroup) {

    const geometry = new THREE.BufferGeometry()

    const vertices = new Float32Array(faceGroup)

    geometry.setAttribute('position', new THREE.BufferAttribute(vertices, 3))

    if (indexGroup) geometry.index = new THREE.BufferAttribute(new Uint16Array(indexGroup), 1)

    if (uvGroup) geometry.attributes.uv = new THREE.Float32BufferAttribute(uvGroup, 2)

    geometry.computeVertexNormals()

    return geometry

}

/* 更新顶点 */
function updateMultShapePlaneGeometry(geometry, faceGroup, indexGroup, uvGroup) {

    geometry.setIndex(indexGroup)

    geometry.setAttribute('position', new THREE.Float32BufferAttribute(faceGroup, 3))

    geometry.setAttribute('uv', new THREE.Float32BufferAttribute(uvGroup, 2))

    delete geometry.attributes.normal

    geometry.computeVertexNormals()

}
```

完整源码：[GitHub](https://github.com/z2586300277/three-cesium-examples/blob/dev/threeExamples/application/drawFence.js)

## 小结

- 本文提供 **绘制围栏** 完整 Three.js 源码与在线 Demo，建议先运行案例再改 uniform/参数做二次实验
- 更多 Three.js 实战案例见 [three-cesium-examples 合集](https://threehub.cn) 与 [GitHub 开源仓库](https://github.com/z2586300277/three-cesium-examples)

