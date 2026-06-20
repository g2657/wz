---
title: "Three.js 朋克风教程"
description: "详解 Three.js 朋克风：用 Canvas 2D 绘制内容并实时映射为 Three.js 纹理，涵盖 OrbitControls、Canvas 等关键实现，附完整源码与在线 Demo，适合 Three.js 应用场景 学习与二次开发。"
head:
  - - meta
    - name: keywords
      content: "Three.js,朋克风,WebGL,源码,教程,在线案例,OrbitControls,相机控制,Canvas纹理,动态贴图"
outline: deep
---
### 朋克风 · *Style Punk* · [▶ 在线运行案例](https://z2586300277.github.io/three-cesium-examples/#/codeMirror?navigation=ThreeJS&classify=application&id=punk)
- **案例合集：** [三维可视化功能案例（threehub.cn）](https://threehub.cn)

- **开源仓库github地址：** https://github.com/z2586300277/three-cesium-examples

- **400个案例代码: ** [网盘链接](https://pan.quark.cn/s/201da5c82fec)

![朋克风](https://z2586300277.github.io/three-cesium-examples/threeExamples/application/punk.jpg)

## 你将学到什么

- OrbitControls 相机轨道交互
- Canvas 动态纹理贴图
- `requestAnimationFrame` 渲染循环与 `resize` 自适应

## 效果说明

本案例演示 **朋克风** 效果：用 Canvas 2D 绘制内容并实时映射为 Three.js 纹理；核心用到 OrbitControls、Canvas。建议先打开文首在线案例查看动态画面，再对照下方源码逐步理解。

## 核心概念

- **OrbitControls** 轨道旋转缩放；开 `enableDamping` 时每帧需 `controls.update()`。

## 实现步骤

1. 搭建 Scene / Camera / Renderer 与 OrbitControls
2. rAF 循环中 update 并 render

## 代码要点

```js
import * as THREE from 'three';
import { OrbitControls } from "three/examples/jsm/Addons.js";
const DOM = document.getElementById('box')

var scene = new THREE.Scene();

var camera = new THREE.PerspectiveCamera(60, innerWidth / innerHeight);
camera.position.set(-1, 0.5, 1).setLength(75);

camera.lookAt(scene.position);

var renderer = new THREE.WebGLRenderer({ antialias: true });
renderer.setSize(innerWidth, innerHeight);
renderer.setAnimationLoop(animationLoop);
function animationLoop() {
    renderer.render(scene, camera);
}
DOM.appendChild(renderer.domElement);

window.addEventListener("resize", () => {
    camera.aspect = innerWidth / innerHeight;
    camera.updateProjectionMatrix();
    renderer.setSize(innerWidth, innerHeight);
});

var controls = new OrbitControls(camera, renderer.domElement);
controls.target.set(0, 12, 0);

controls.update();
controls.enableDamping = true;
controls.autoRotate = true;

var light = new THREE.DirectionalLight('white', 3);
light.position.set(1, 1, 1);
scene.add(light);
import{BoxGeometry, MeshLambertMaterial,Mesh,CanvasTexture,MeshBasicMaterial,Clock,CylinderGeometry,MathUtils,RepeatWrapping
} from 'three';

function CircularMotionOfLetters() {
    let g = new BoxGeometry();
    g.translate(0, 0.5, 0);
    let m = new MeshLambertMaterial({ color: 0x7f7f7f });
    for (let z = -5; z < 5; z++) {
        for (let x = -5; x < 5; x++) {
            let o = new Mesh(g, m);
            o.position.set(x, 0, z).multiplyScalar(4);
            o.rotation.y = Math.random() * Math.PI;
            let posLen = o.position.length();
            let posScale = 1 - MathUtils.clamp(posLen / 50, 0, 1);
            posScale = Math.pow(posScale, 4);
            o.scale.set(MathUtils.randInt(1, 4), 1 + Math.floor(Math.random() * posScale * 75), MathUtils.randInt(1, 4));
            if (posLen <= 25) {
                scene.add(o);
            }
        }
    }
    let cnvTexture;
    function create_ring_letter() {
        let cnv = document.createElement('canvas');
        cnv.width = 1024;
        cnv.height = 128;
        let ctx = cnv.getContext('2d');
        ctx.fillStyle = 'magenta';
        ctx.fillRect(0, 0, cnv.width, cnv.height);
        ctx.clearRect(0, 10, cnv.width, cnv.height - 20);
        ctx.textBaseline = 'middle';
        ctx.textAlign = "center";
        ctx.fillStyle = 'aqua';
        ctx.font = "bold 80px Arial";
        ctx.fillText("Welcome to Nico Space", cnv.width * 0.5, cnv.height * 0.5);
        cnvTexture = new CanvasTexture(cnv);
        cnvTexture.wrapS = RepeatWrapping;
        cnvTexture.wrapT = RepeatWrapping;
        cnvTexture.repeat.set(3, 1);
        let gc = new CylinderGeometry(25, 25, 5, 72, 1, true);
        const material = new MeshBasicMaterial({ alphaTest: 0.5, side: 2, map: cnvTexture });
        const mesh = new Mesh(gc, material);
        mesh.position.y = 25;
        scene.add(mesh);
    }
    create_ring_letter();
    let clock = new Clock();
    function animate() {
        requestAnimationFrame(animate);
        let t = clock.getDelta();
        cnvTexture.offset.x += t * 0.125;
    }
    animate();
}

CircularMotionOfLetters()
```

完整源码：[GitHub](https://github.com/z2586300277/three-cesium-examples/blob/dev/threeExamples/application/punk.js)

## 小结

- 本文提供 **朋克风** 完整 Three.js 源码与在线 Demo，建议先运行案例再改 uniform/参数做二次实验
- 更多 Three.js 实战案例见 [three-cesium-examples 合集](https://threehub.cn) 与 [GitHub 开源仓库](https://github.com/z2586300277/three-cesium-examples)

