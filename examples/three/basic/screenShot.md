---
title: "Three.js 截图教程"
description: "详解 Three.js 截图：原场景渲染后经 EffectComposer 叠加 Bloom/模糊等全屏后期，涵盖 EffectComposer、UnrealBloomPass、OrbitControls 等关键实现，附完整源码与在线 Demo，适合 Three.js 基础 学习与二次开发。"
head:
  - - meta
    - name: keywords
      content: "Three.js,截图,WebGL,源码,教程,在线案例,EffectComposer,后期处理,Bloom,辉光,OrbitControls,相机控制"
outline: deep
---

### 截图 · *Screenshot* · [▶ 在线运行案例](https://z2586300277.github.io/three-cesium-examples/#/codeMirror?navigation=ThreeJS&classify=basic&id=screenShot)
- **案例合集：** [三维可视化功能案例（threehub.cn）](https://threehub.cn)

- **开源仓库github地址：** https://github.com/z2586300277/three-cesium-examples

- **400个案例代码: ** [网盘链接](https://pan.quark.cn/s/201da5c82fec)

## 你将学到什么

- **`renderer.domElement.toDataURL()`** 导出 canvas 为 PNG
- 有 **Bloom 后期** 时需 `composer.render()` 再截图
- 编程触发 `<a download>` 下载

## 效果说明

黄色立方体 + UnrealBloomPass 辉光。点击「截图」下载 `myImage.png`。

## 核心概念

```js
function screenShot() {
    renderer.render(scene, camera);  // 可选
    composer.render();               // 含后期时必须
    const base64 = renderer.domElement.toDataURL('image/png', 0.8);
    const link = document.createElement('a');
    link.href = base64;
    link.download = 'myImage.png';
    link.click();
}
```

::: tip
若截图为黑，初始化 Renderer 时设 `preserveDrawingBuffer: true`。
:::

## 代码要点

```js
import * as THREE from 'three'
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls.js'
import { UnrealBloomPass } from 'three/examples/jsm/postprocessing/UnrealBloomPass.js';
import { EffectComposer } from 'three/examples/jsm/postprocessing/EffectComposer.js';
import { RenderPass } from 'three/examples/jsm/postprocessing/RenderPass.js';
import * as dat from 'dat.gui'

const box = document.getElementById('box')

const scene = new THREE.Scene()

const camera = new THREE.PerspectiveCamera(75, box.clientWidth / box.clientHeight, 0.1, 1000)

camera.position.set(0, 10, 10)

const renderer = new THREE.WebGLRenderer()

renderer.setSize(box.clientWidth, box.clientHeight)

box.appendChild(renderer.domElement)

new OrbitControls(camera, renderer.domElement)

const composer = new EffectComposer(renderer);

const renderPass = new RenderPass(scene, camera);

composer.addPass(renderPass);

const bloomPass = new UnrealBloomPass(new THREE.Vector2(window.innerWidth, window.innerHeight), 1.5, 0, 0)

composer.addPass(bloomPass);

animate()

function animate() {

    requestAnimationFrame(animate)

    composer.render()
}

window.onresize = () => {

    renderer.setSize(box.clientWidth, box.clientHeight)

    camera.aspect = box.clientWidth / box.clientHeight

    camera.updateProjectionMatrix()

}

// 物体
const geometry = new THREE.BoxGeometry()

const material = new THREE.MeshBasicMaterial({ color: 'yellow' })

const cube = new THREE.Mesh(geometry, material)

scene.add(cube)

// 截图
new dat.GUI().add({ '截图': screenShot }, '截图')

function screenShot() {

    renderer.render(scene, camera)

    composer.render()

    const base64 = renderer.domElement.toDataURL(['image/png', '0.8'])

    const link = document.createElement('a');

    link.href = base64;

    link.download = 'myImage.png';

    link.click();

}
```

完整源码：[GitHub](https://github.com/z2586300277/three-cesium-examples/blob/dev/threeExamples/basic/screenShot.js)

## 小结
