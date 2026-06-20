---
title: "Three.js 渲染贴图物体教程"
description: "详解 Three.js 渲染贴图物体：基于 WebGL 实现「渲染贴图物体」可视化效果，附完整可运行源码，涵盖 OrbitControls、glTF/Draco 等关键实现，附完整源码与在线 Demo，适合 Three.js 基础 学习与二次开发。"
head:
  - - meta
    - name: keywords
      content: "Three.js,渲染贴图物体,WebGL,源码,教程,在线案例,OrbitControls,相机控制,glTF,模型加载"
outline: deep
---

### 渲染贴图物体 · *Render Target* · [▶ 在线运行案例](https://z2586300277.github.io/three-cesium-examples/#/codeMirror?navigation=ThreeJS&classify=basic&id=renderTarget)
- **案例合集：** [三维可视化功能案例（threehub.cn）](https://threehub.cn)

- **开源仓库github地址：** https://github.com/z2586300277/three-cesium-examples

- **400个案例代码: ** [网盘链接](https://pan.quark.cn/s/201da5c82fec)

## 你将学到什么

- **WebGLRenderTarget** 渲染到纹理
- **camera.layers** 切换：先拍模型，再拍 UI 层
- 纹理贴到 **Plane / Sphere** 当「监视器」

## 效果说明

电脑模型只在 layer 1。第一遍相机 layer 1 → renderTarget；第二遍 layer 0 显示带「屏幕内容」的平面和球体。

## 核心概念

```js
const rt = new THREE.WebGLRenderTarget(w, h);
model.traverse(c => c.layers.set(1));

function animate() {
    camera.layers.set(1);
    renderer.setRenderTarget(rt);
    renderer.render(scene, camera);

    camera.layers.set(0);
    renderer.setRenderTarget(null);
    renderer.render(scene, camera);  // plane.map = rt.texture
}
```

镜子、小地图、Portal 都基于此模式。

## 代码要点

```js
import * as THREE from 'three'
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls.js'
import { GLTFLoader } from 'three/examples/jsm/loaders/GLTFLoader.js'

const box = document.getElementById('box')

const scene = new THREE.Scene()

const camera = new THREE.PerspectiveCamera(50, box.clientWidth / box.clientHeight, 0.1, 100000)

camera.position.set(2, 0, 9)

const renderer = new THREE.WebGLRenderer({ antialias: true, alpha: true, logarithmicDepthBuffer: true })

renderer.setSize(box.clientWidth, box.clientHeight)

box.appendChild(renderer.domElement)

new OrbitControls(camera, renderer.domElement)

const urls = [0, 1, 2, 3, 4, 5].map(k => (FILE_HOST + 'files/sky/skyBox0/' + (k + 1) + '.png'));

const textureCube = new THREE.CubeTextureLoader().load(urls);

scene.background = textureCube;

const renderTarget = new THREE.WebGLRenderTarget(box.clientWidth, box.clientHeight)

new GLTFLoader().load(

    FILE_HOST + 'models/glb/computer.glb',

    gltf => {

        const model = gltf.scene

        model.traverse(child =>  child.layers.set(1))

        scene.add(model)

    }

)

const plane = new THREE.Mesh(new THREE.PlaneGeometry(4, 4), new THREE.MeshBasicMaterial({ map: renderTarget.texture }))

scene.add(plane)

const sphere = new THREE.Mesh(new THREE.SphereGeometry(1, 32, 32), new THREE.MeshBasicMaterial({ map: renderTarget.texture }))

sphere.position.set(3, 0, 0)

scene.add(sphere)

animate()

function animate() {

    camera.layers.set(1)

    renderer.setRenderTarget(renderTarget)

    renderer.render(scene, camera)

    camera.layers.set(0)

    renderer.setRenderTarget(null)

    renderer.render(scene, camera)

    requestAnimationFrame(animate)

}
```

完整源码：[GitHub](https://github.com/z2586300277/three-cesium-examples/blob/dev/threeExamples/basic/renderTarget.js)

## 小结
