---
title: "Three.js 模型粒子化教程"
description: "详解 Three.js 模型粒子化：基于 WebGL 实现「模型粒子化」可视化效果，附完整可运行源码，涵盖 OrbitControls、THREE.Points、glTF/Draco 等关键实现，附完整源码与在线 Demo，适合 Three.js 应用场景 学习与二次开发。"
head:
  - - meta
    - name: keywords
      content: "Three.js,模型粒子化,WebGL,源码,教程,在线案例,OrbitControls,相机控制,粒子特效,Points,glTF,模型加载"
outline: deep
---
### 模型粒子化 · *Model Particle* · [▶ 在线运行案例](https://z2586300277.github.io/three-cesium-examples/#/codeMirror?navigation=ThreeJS&classify=application&id=modelParticle)
- **案例合集：** [三维可视化功能案例（threehub.cn）](https://threehub.cn)

- **开源仓库github地址：** https://github.com/z2586300277/three-cesium-examples

- **400个案例代码: ** [网盘链接](https://pan.quark.cn/s/201da5c82fec)

![模型粒子化](https://z2586300277.github.io/three-cesium-examples/threeExamples/application/modelParticle.jpg)

## 你将学到什么

- glTF/FBX/OBJ 外部模型加载
- 相机交互控制器
- 点云 / 粒子 / 实例化渲染
- GSAP / anime.js 属性动画
- requestAnimationFrame 渲染循环

## 效果说明

本案例演示 **模型粒子化** 效果：基于 WebGL 实现「模型粒子化」可视化效果，附完整可运行源码；核心用到 OrbitControls、THREE.Points、glTF/Draco。建议先打开文首在线案例查看动态画面，再对照下方源码逐步理解。

## 核心概念

- **Loader** 异步加载模型；glTF 返回 `gltf.scene`，加载后注意 `scale` 与坐标系。Draco 需配置 `DRACOLoader`。

- **OrbitControls** 轨道旋转缩放；开 `enableDamping` 时每帧需 `controls.update()`。

- **Points** 大量顶点用点精灵渲染；**InstancedMesh** 相同几何体批量绘制，降低 draw call。

- 时间线库驱动 position/rotation/uniform，与 rAF 渲染循环配合。

## 实现步骤

1. 搭建 Scene / Camera / Renderer 与 OrbitControls
2. Loader 异步加载模型/纹理资源
3. rAF 循环中 update 并 render

## 代码要点

```js
import * as THREE from 'three'
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls.js'
import { GLTFLoader } from 'three/examples/jsm/loaders/GLTFLoader.js'
import { DRACOLoader } from 'three/examples/jsm/loaders/DRACOLoader.js'
import gsap from 'gsap'

const box = document.getElementById('box')

const scene = new THREE.Scene()

const camera = new THREE.PerspectiveCamera(75, box.clientWidth / box.clientHeight, 0.1, 100000)

camera.position.set(300, 300, 300)

const renderer = new THREE.WebGLRenderer({ antialias: true, alpha: true, logarithmicDepthBuffer: true })

renderer.setSize(box.clientWidth, box.clientHeight)

box.appendChild(renderer.domElement)

new OrbitControls(camera, renderer.domElement)

const loader = new GLTFLoader()

loader.setDRACOLoader(new DRACOLoader().setDecoderPath(FILE_HOST + 'js/three/draco/'))

loader.load(

    FILE_HOST + '/files/model/LittlestTokyo.glb',

    gltf => {

        let count = 0

        gltf.scene.traverse(child => {

            if (child.isMesh) {

                count++

                setTimeout(() => {

                    const particles = createPoints(child)

                    scene.add(particles)

                    gsap.to(particles.material, { opacity: 0.9, duration: 1 })

                }, count * 100)

            }

        })

    }

)

animate()

function animate() {

    requestAnimationFrame(animate)

    renderer.render(scene, camera)

}

function createPoints(mesh) {

    mesh.updateMatrixWorld()

    const positions = []

    const vertex = new THREE.Vector3();

    for (let i = 0; i < mesh.geometry.attributes.position.count; i++) {

        vertex.fromBufferAttribute(mesh.geometry.attributes.position, i)

        vertex.applyMatrix4(mesh.matrixWorld)

        positions.push(vertex.x, vertex.y, vertex.z)

    }

    const geometry = new THREE.BufferGeometry()

    geometry.setAttribute('position', new THREE.Float32BufferAttribute(positions, 3))

    const colors = new Float32Array(positions.map(() => Math.random()))

    geometry.setAttribute('color', new THREE.Float32BufferAttribute(colors, 3))

    const material = new THREE.PointsMaterial({

        size: 10,

        vertexColors: true,

        opacity: 0,

        map: new THREE.TextureLoader().load(HOST + 'files/images/snow.png'),

        transparent: true

    })

    const particles = new THREE.Points(geometry, material)

    return particles

}
```

完整源码：[GitHub](https://github.com/z2586300277/three-cesium-examples/blob/dev/threeExamples/application/modelParticle.js)

## 小结

- 本文提供 **模型粒子化** 完整 Three.js 源码与在线 Demo，建议先运行案例再改 uniform/参数做二次实验
- 更多 Three.js 实战案例见 [three-cesium-examples 合集](https://threehub.cn) 与 [GitHub 开源仓库](https://github.com/z2586300277/three-cesium-examples)

