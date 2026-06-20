---
title: "Three.js 帧率控制教程"
description: "详解 Three.js 帧率控制：基于 WebGL 实现「帧率控制」可视化效果，附完整可运行源码，涵盖 OrbitControls、glTF/Draco 等关键实现，附完整源码与在线 Demo，适合 Three.js 基础 学习与二次开发。"
head:
  - - meta
    - name: keywords
      content: "Three.js,帧率控制,WebGL,源码,教程,在线案例,OrbitControls,相机控制,glTF,模型加载"
outline: deep
---

### 帧率控制 · *Fixed FPS* · [▶ 在线运行案例](https://z2586300277.github.io/three-cesium-examples/#/codeMirror?navigation=ThreeJS&classify=basic&id=renderFrame)
- **案例合集：** [三维可视化功能案例（threehub.cn）](https://threehub.cn)

- **开源仓库github地址：** https://github.com/z2586300277/three-cesium-examples

- **400个案例代码: ** [网盘链接](https://pan.quark.cn/s/201da5c82fec)

## 你将学到什么

- **限制最高 FPS** 降低 GPU 占用（笔记本省电、后台标签页）
- rAF 仍每帧调用，但 **按需 render**
- Stats 监视实际渲染次数

## 效果说明

大场景 build3.glb + GUI 调节目标 fps（1~300）。`renderT = 1/fps`，仅当 `timeS > renderT` 时才 `renderer.render`。

## 核心概念

```js
let timeS = 0, fps = 60, renderT = 1 / fps;

function animate() {
    timeS += clock.getDelta();
    if (timeS > renderT) {
        controls.update();
        stats.update();
        renderer.render(scene, camera);
        timeS = 0;
    }
    requestAnimationFrame(animate);
}
```

与 [入门·帧率](/examples/three/introduction/帧率)（Stats 监视）互补：本篇是 **主动限帧**。

## 代码要点

```js
import * as THREE from 'three'
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls.js'
import { GLTFLoader } from 'three/examples/jsm/loaders/GLTFLoader.js'
import { DRACOLoader } from 'three/examples/jsm/loaders/DRACOLoader.js'
import Stats from 'three/examples/jsm/libs/stats.module.js';
import { GUI } from 'dat.gui'

const box = document.getElementById('box')

const scene = new THREE.Scene()

const camera = new THREE.PerspectiveCamera(75, box.clientWidth / box.clientHeight, 0.1, 100000)

camera.position.set(10, 10, 10)

const renderer = new THREE.WebGLRenderer({ antialias: true, alpha: true, logarithmicDepthBuffer: true })

renderer.setSize(box.clientWidth, box.clientHeight)

box.appendChild(renderer.domElement)

const controls = new OrbitControls(camera, renderer.domElement)

controls.enableDamping = true

window.onresize = () => {

    renderer.setSize(box.clientWidth, box.clientHeight)

    camera.aspect = box.clientWidth / box.clientHeight

    camera.updateProjectionMatrix()

}

const stats = new Stats()

stats.dom.style.top = '50px'

document.body.appendChild(stats.dom)

const clock = new THREE.Clock()

let timeS = 0, fps = 60, renderT = 1 / fps

let gui = new GUI()
   
gui.add({ fps }, 'fps', 1, 300).onFinishChange(v => {

    fps = v

    renderT = 1 / fps

})


animate()

function animate() {

    timeS += clock.getDelta()

    if (timeS > renderT) {

        controls.update()

        stats.update()

        renderer.render(scene, camera)

        timeS = 0

    }

    requestAnimationFrame(animate)

}


const loader = new GLTFLoader()

loader.setDRACOLoader(new DRACOLoader().setDecoderPath(FILE_HOST + 'js/three/draco/'))

const urls = [0, 1, 2, 3, 4, 5].map(k => (FILE_HOST + 'files/sky/skyBox0/' + (k + 1) + '.png'));

const textureCube = new THREE.CubeTextureLoader().load(urls);

loader.load(

    FILE_HOST + '/models/glb/build3.glb',

    gltf => {

        gltf.scene.traverse(child => {

            if (child.isMesh) child.material.envMap = textureCube

        })

        scene.add(gltf.scene)

    }

)
```

完整源码：[GitHub](https://github.com/z2586300277/three-cesium-examples/blob/dev/threeExamples/basic/renderFrame.js)

## 小结
