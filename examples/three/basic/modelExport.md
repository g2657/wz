---
title: "Three.js 模型导出教程"
description: "详解 Three.js 模型导出：基于 WebGL 实现「模型导出」可视化效果，附完整可运行源码，涵盖 OrbitControls 等关键实现，附完整源码与在线 Demo，适合 Three.js 基础 学习与二次开发。"
head:
  - - meta
    - name: keywords
      content: "Three.js,模型导出,WebGL,源码,教程,在线案例,OrbitControls,相机控制"
outline: deep
---

### 模型导出 · *GLTF Export* · [▶ 在线运行案例](https://z2586300277.github.io/three-cesium-examples/#/codeMirror?navigation=ThreeJS&classify=basic&id=modelExport)
- **案例合集：** [三维可视化功能案例（threehub.cn）](https://threehub.cn)

- **开源仓库github地址：** https://github.com/z2586300277/three-cesium-examples

- **400个案例代码: ** [网盘链接](https://pan.quark.cn/s/201da5c82fec)

## 你将学到什么

- **GLTFExporter.parse** 导出场景
- `{ binary: true }` → **glb**；否则 gltf JSON
- `embedImages: true` 内嵌贴图

## 效果说明

加载 house.obj，点击「导出模型」下载 **scene.glb**。

```js
exporter.parse(scene, (result) => {
    const blob = result instanceof ArrayBuffer
        ? new Blob([result], { type: 'model/gltf-binary' })
        : new Blob([JSON.stringify(result)], { type: 'model/gltf+json' });
    // download blob...
}, { binary: true, onlyVisible: true, embedImages: true });
```

## 代码要点

```js
import * as THREE from 'three'
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls.js'
import { OBJLoader } from 'three/examples/jsm/loaders/OBJLoader.js'
import { MTLLoader } from 'three/examples/jsm/loaders/MTLLoader.js'
import { GLTFExporter } from 'three/addons/exporters/GLTFExporter.js';

const box = document.getElementById('box')

const scene = new THREE.Scene()

const camera = new THREE.PerspectiveCamera(75, box.clientWidth / box.clientHeight, 0.1, 100000)

camera.position.set(5, 5, 5)

const renderer = new THREE.WebGLRenderer({ antialias: true, alpha: true, logarithmicDepthBuffer: true })

renderer.setSize(box.clientWidth, box.clientHeight)

box.appendChild(renderer.domElement)

const controls = new OrbitControls(camera, renderer.domElement)

controls.enableDamping = true

scene.add(new THREE.AmbientLight(0xffffff, 3))

window.onresize = () => {

    renderer.setSize(box.clientWidth, box.clientHeight)

    camera.aspect = box.clientWidth / box.clientHeight

    camera.updateProjectionMatrix()

}

animate()

function animate() {

    requestAnimationFrame(animate)

    controls.update()

    renderer.render(scene, camera)

}

const urls = [0, 1, 2, 3, 4, 5].map(k => (FILE_HOST + 'files/sky/skyBox0/' + (k + 1) + '.png'));

const textureCube = new THREE.CubeTextureLoader().load(urls);

scene.background = textureCube

const objLoader = new OBJLoader()

const mtlLoader = new MTLLoader()

mtlLoader.load(FILE_HOST + 'files/model/house/house.mtl', (mtl) => {

    mtl.preload()

    objLoader.setMaterials(mtl)

    objLoader.load(FILE_HOST + 'files/model/house/house.obj', (obj) => scene.add(obj))

})

const exporter = new GLTFExporter();

const button = document.createElement('button');
button.textContent = '导出模型';
button.style.position = 'absolute';
button.style.top = '10px';
button.style.left = '100px';
box.appendChild(button);

button.onclick = async () => {

    exporter.parse(scene, (result) => {

        const outBlob = result instanceof ArrayBuffer
            ? new Blob([result], { type: 'model/gltf-binary' })
            : new Blob([JSON.stringify(result, null, 2)], { type: 'model/gltf+json' })

        const link = document.createElement('a')
        link.href = URL.createObjectURL(outBlob)
        link.download = result instanceof ArrayBuffer ? 'scene.glb' : 'scene.gltf'
        link.click()
        URL.revokeObjectURL(link.href)

    }, { binary: true, onlyVisible: true, embedImages: true })

}
```

完整源码：[GitHub](https://github.com/z2586300277/three-cesium-examples/blob/dev/threeExamples/basic/modelExport.js)

## 小结

- 本文提供 **模型导出** 完整 Three.js 源码与在线 Demo，建议先运行案例再改 uniform/参数做二次实验
- 更多 Three.js 实战案例见 [three-cesium-examples 合集](https://threehub.cn) 与 [GitHub 开源仓库](https://github.com/z2586300277/three-cesium-examples)

