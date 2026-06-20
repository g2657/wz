---
title: "Three.js 加载动画教程"
description: "详解 Three.js 加载动画：基于 WebGL 实现「加载动画」可视化效果，附完整可运行源码，涵盖 OrbitControls、glTF/Draco 等关键实现，附完整源码与在线 Demo，适合 Three.js 基础 学习与二次开发。"
head:
  - - meta
    - name: keywords
      content: "Three.js,加载动画,WebGL,源码,教程,在线案例,OrbitControls,相机控制,glTF,模型加载"
outline: deep
---

### 加载动画 · *Loading Progress* · [▶ 在线运行案例](https://z2586300277.github.io/three-cesium-examples/#/codeMirror?navigation=ThreeJS&classify=basic&id=loadingAnimate)
- **案例合集：** [三维可视化功能案例（threehub.cn）](https://threehub.cn)

- **开源仓库github地址：** https://github.com/z2586300277/three-cesium-examples

- **400个案例代码: ** [网盘链接](https://pan.quark.cn/s/201da5c82fec)

![加载动画](https://z2586300277.github.io/three-cesium-examples/threeExamples/basic/loadingAnimate.jpg)

## 你将学到什么

- **LoadingManager** 管理多资源总进度
- **GLTFLoader.load** 第三个参数 xhr 下载进度
- 两层进度：**下载 %** 与 **解析/导入 %**

## 效果说明

加载 `LittlestTokyo.glb` 大场景时，屏幕中央浮层显示 **「下载 xx%」** 与 **「导入 xx%」**，完成后显示「加载完成」。

## 核心概念

### LoadingManager 回调

```js
const manager = new THREE.LoadingManager();
manager.onStart = (url, loaded, total) => { /* 开始 */ };
manager.onProgress = (url, loaded, total) => { /* 总进度 */ };
manager.onLoad = () => { /* 全部完成 */ };
manager.onError = (url) => { /* 失败 */ };

const loader = new GLTFLoader(manager);
```

传入 Manager 后，Loader 内部加载的 **纹理、bin 等子资源** 都会计入 `onProgress`。

### xhr 下载进度

```js
loader.load(url, onLoad, (xhr) => {
    const pct = (xhr.loaded / xhr.total * 100).toFixed(2);
    // 仅反映网络下载，不含 GPU 解析
});
```

| 阶段 | 回调 | 含义 |
|------|------|------|
| 下载 | load 的第 3 参数 xhr | 字节流进度 |
| 导入 | manager.onProgress | 含纹理解析等 |

本案例 **两者同时显示**，用户体验更完整。

## 代码要点

```js
import * as THREE from 'three'
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls.js'
import { GLTFLoader } from 'three/examples/jsm/loaders/GLTFLoader.js'
import { DRACOLoader } from 'three/examples/jsm/loaders/DRACOLoader.js'

const loadingDiv = document.createElement('div')
loadingDiv.innerText = '加载中...'
Object.assign(loadingDiv.style, {
    pointerEvents: 'none',
    position: 'fixed',
    top: '50%',
    left: '50%',
    transform: 'translate(-50%,-50%)',
    color: 'white',
    fontSize: '20px',
    backgroundColor: 'rgba(0,0,0,0.5)',
    padding: '10px 20px',
    borderRadius: '5px'
})

document.body.appendChild(loadingDiv)

const box = document.getElementById('box')

const scene = new THREE.Scene()

const camera = new THREE.PerspectiveCamera(75, box.clientWidth / box.clientHeight, 0.1, 1000000)

camera.position.set(0, 400, 400)

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

scene.add(new THREE.AmbientLight(0xffffff, 3))


const manager = new THREE.LoadingManager();

manager.onStart = function (url, itemsLoaded, itemsTotal) {
    loadingDiv.innerText = '开始加载'
};

manager.onLoad = function () {
    loadingDiv.innerHTML = '加载完成'
};

manager.onProgress = function (url, itemsLoaded, itemsTotal) {
    loadingDiv.innerText = '导入' + (itemsLoaded / itemsTotal * 100).toFixed(2) + '%' 
}

manager.onError = function (url) {

}

const loader = new GLTFLoader(manager)

loader.setDRACOLoader(new DRACOLoader().setDecoderPath(FILE_HOST + 'js/three/draco/'))

loader.load(

    FILE_HOST + '/files/model/LittlestTokyo.glb?time=' + new Date().getTime(),

    gltf => {

        scene.add(gltf.scene)

    },

    xhr => {

        loadingDiv.innerText = '下载' + (xhr.loaded / xhr.total * 100).toFixed(2) + '%'

    }

)

animate()

function animate() {

    requestAnimationFrame(animate)

    controls.update()

    renderer.render(scene, camera)

}
```

完整源码：[GitHub](https://github.com/z2586300277/three-cesium-examples/blob/dev/threeExamples/basic/loadingAnimate.js)

## 小结

- 本文提供 **加载动画** 完整 Three.js 源码与在线 Demo，建议先运行案例再改 uniform/参数做二次实验
- 更多 Three.js 实战案例见 [three-cesium-examples 合集](https://threehub.cn) 与 [GitHub 开源仓库](https://github.com/z2586300277/three-cesium-examples)

