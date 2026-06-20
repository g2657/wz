---
title: "Three.js 地理边界教程"
description: "详解 Three.js 地理边界：基于 WebGL 实现「地理边界」可视化效果，附完整可运行源码，涵盖 OrbitControls、CatmullRomCurve3 等关键实现，附完整源码与在线 Demo，适合 Three.js 扩展 学习与二次开发。"
head:
  - - meta
    - name: keywords
      content: "Three.js,地理边界,WebGL,源码,教程,在线案例,OrbitControls,相机控制,样条曲线,路径动画"
outline: deep
---
### 地理边界 · *Geo Border* · [▶ 在线运行案例](https://z2586300277.github.io/three-cesium-examples/#/codeMirror?navigation=ThreeJS&classify=expand&id=geoBorder)
- **案例合集：** [三维可视化功能案例（threehub.cn）](https://threehub.cn)

- **开源仓库github地址：** https://github.com/z2586300277/three-cesium-examples

- **400个案例代码: ** [网盘链接](https://pan.quark.cn/s/201da5c82fec)

![地理边界](https://z2586300277.github.io/three-cesium-examples/threeExamples/expand/geoBorder.jpg)

## 你将学到什么

- OrbitControls 相机轨道交互
- CatmullRomCurve3 样条曲线路径
- `requestAnimationFrame` 渲染循环与 `resize` 自适应

## 效果说明

本案例演示 **地理边界** 效果：基于 WebGL 实现「地理边界」可视化效果，附完整可运行源码；核心用到 OrbitControls、CatmullRomCurve3。建议先打开文首在线案例查看动态画面，再对照下方源码逐步理解。

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

camera.position.set(0, 0, 500)

const renderer = new THREE.WebGLRenderer({ antialias: true, alpha: true, logarithmicDepthBuffer: true })

renderer.setSize(box.clientWidth, box.clientHeight)

box.appendChild(renderer.domElement)

const controls = new OrbitControls(camera, renderer.domElement)

controls.enableDamping = true

scene.add(new THREE.AmbientLight(0xffffff, 1))

const directionalLight = new THREE.DirectionalLight(0xffffff, 2)

directionalLight.position.set(300, 300, 300)

scene.add(directionalLight)

const map = new THREE.TextureLoader().load(FILE_HOST + 'images/channels/lmap.png')

map.wrapS = THREE.RepeatWrapping

map.wrapT = THREE.RepeatWrapping

map.needsUpdate = true

animate()

function animate() {

  requestAnimationFrame(animate)

  map.offset.x += 0.001

  controls.update()

  renderer.render(scene, camera)

}

window.onresize = () => {

  renderer.setSize(box.clientWidth, box.clientHeight)

  camera.aspect = box.clientWidth / box.clientHeight

  camera.updateProjectionMatrix()

}

/* 边界 */
const group = new THREE.Group()

fetch(FILE_HOST + 'files/json/chinaBound.json').then(r => r.json()).then(res => {

  const { features } = res

  features.forEach((i) => {

    if (i.geometry.type === 'MultiPolygon') i.geometry.coordinates.forEach((j) => j.forEach((z) => createShapeWithCoord(group, z)))

    else if (i.geometry.type === 'Polygon') i.geometry.coordinates.forEach((j) => createShapeWithCoord(group, j))

    else if (i.geometry.type === 'LineString') i.geometry.coordinates.length > 1 && createShapeWithCoord(group, i.geometry.coordinates)

  })

  translationOriginForGroup(group)

  scene.add(group)

})

function createShapeWithCoord(group, coordinates) {
  
  if (coordinates.length < 1000) return // 设置点数限制 如果点太少则不绘制

  const curvePoints = coordinates.map((k) => coordToVector3(k))

  const curve = new THREE.CatmullRomCurve3(curvePoints)

  const geometry = new THREE.TubeGeometry(curve, curvePoints.length - 1, 1, 40, false)

  const material = new THREE.MeshPhongMaterial({ color: 0xffffff , map , transparent: true })

  const mesh = new THREE.Mesh(geometry, material)

  translationOriginForMesh(mesh)

  group.attach(mesh)

}

function coordToVector3(coord, slace = 10000) {

  const [lng, lat] = coord

  const x = lng * 20037508.34 / 180

  const y = Math.log(Math.tan((90 + lat) * Math.PI / 360)) / (Math.PI / 180) * 20037508.34 / 180

  return new THREE.Vector3(x / slace, y / slace, 0)

}

function translationOriginForMesh(mesh) {
    
  const boundingBox = new THREE.Box3().setFromObject(mesh)

  boundingBox.getCenter(mesh.position)

  mesh.geometry.center()

}

// 设置组中心点
function translationOriginForGroup(group) {

  const boundingBox = new THREE.Box3().setFromObject(group)

  boundingBox.getCenter(group.position)

  group.traverse((c) => {

      c.isMesh && c.position.sub(group.position)

  })

  group.position.set(0, 0, 0)

}
```

完整源码：[GitHub](https://github.com/z2586300277/three-cesium-examples/blob/dev/threeExamples/expand/geoBorder.js)

## 小结

- 本文提供 **地理边界** 完整 Three.js 源码与在线 Demo，建议先运行案例再改 uniform/参数做二次实验
- 更多 Three.js 实战案例见 [three-cesium-examples 合集](https://threehub.cn) 与 [GitHub 开源仓库](https://github.com/z2586300277/three-cesium-examples)

