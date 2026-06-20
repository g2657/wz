---
title: "Three.js 流动围栏教程"
description: "详解 Three.js 流动围栏：基于 WebGL 实现「流动围栏」可视化效果，附完整可运行源码，涵盖 OrbitControls、BufferGeometry 等关键实现，附完整源码与在线 Demo，适合 Three.js 应用场景 学习与二次开发。"
head:
  - - meta
    - name: keywords
      content: "Three.js,流动围栏,WebGL,源码,教程,在线案例,OrbitControls,相机控制,BufferGeometry"
outline: deep
---
### 流动围栏 · *Sport Fence* · [▶ 在线运行案例](https://z2586300277.github.io/three-cesium-examples/#/codeMirror?navigation=ThreeJS&classify=application&id=sportFence)
- **案例合集：** [三维可视化功能案例（threehub.cn）](https://threehub.cn)

- **开源仓库github地址：** https://github.com/z2586300277/three-cesium-examples

- **400个案例代码: ** [网盘链接](https://pan.quark.cn/s/201da5c82fec)

![流动围栏](https://z2586300277.github.io/three-cesium-examples/threeExamples/application/sportFence.jpg)

## 你将学到什么

- OrbitControls 相机轨道交互
- BufferGeometry 自定义顶点/索引数据
- `requestAnimationFrame` 渲染循环与 `resize` 自适应

## 效果说明

本案例演示 **流动围栏** 效果：基于 WebGL 实现「流动围栏」可视化效果，附完整可运行源码；核心用到 OrbitControls、BufferGeometry。建议先打开文首在线案例查看动态画面，再对照下方源码逐步理解。

## 核心概念

- **OrbitControls** 轨道旋转缩放；开 `enableDamping` 时每帧需 `controls.update()`。

## 实现步骤

1. 搭建 Scene / Camera / Renderer 与 OrbitControls
2. rAF 循环中 update 并 render

## 代码要点

```js
import * as THREE from 'three'
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls.js'
import { GUI } from 'three/examples/jsm/libs/lil-gui.module.min.js'

const box = document.getElementById('box')

const scene = new THREE.Scene()

const camera = new THREE.PerspectiveCamera(75, box.clientWidth / box.clientHeight, 0.1, 100000)

camera.position.set(0, 50, 50)

const renderer = new THREE.WebGLRenderer({ antialias: true, alpha: true, logarithmicDepthBuffer: true })

renderer.setSize(box.clientWidth, box.clientHeight)

box.appendChild(renderer.domElement)

new OrbitControls(camera, renderer.domElement)

scene.add(new THREE.AxesHelper(100), new THREE.GridHelper(100, 10))

const points = [
    new THREE.Vector3(10, 0, 20),
    new THREE.Vector3(25, 0, 0),
    new THREE.Vector3(-30, 0, -20),
    new THREE.Vector3(-20, 0, 30),
];

const height = 20;
const fenceGeometry = createFenceGeometry(points, height);

const color = new THREE.Color(0xb9f9c3);

const material = new THREE.MeshBasicMaterial({
    color,
    transparent: true,
    side: THREE.DoubleSide,
    map: new THREE.TextureLoader().load(FILE_HOST + 'images/channels/wall_g.png')
});
const fence = new THREE.Mesh(fenceGeometry, material);
scene.add(fence);

const texture = new THREE.TextureLoader().load(FILE_HOST + 'images/channels/wall_line.png')
texture.wrapS = THREE.RepeatWrapping;
texture.wrapT = THREE.RepeatWrapping;
texture.repeat.x = 2

const fence2 = new THREE.Mesh(fenceGeometry.clone(), new THREE.MeshBasicMaterial({
    color,
    map: texture,
    transparent: true,
    side: THREE.DoubleSide,
}));
scene.add(fence2);

function createFenceGeometry(points, height) {
    const positions = [];
    const uvs = [];
    const indices = [];

    let totalLength = 0;
    for (let i = 0; i < points.length; i++) {
        const current = points[i];
        const next = points[(i + 1) % points.length];
        totalLength += current.distanceTo(next);
    }

    let currentLength = 0;

    for (let i = 0; i < points.length; i++) {
        const current = points[i];
        const next = points[(i + 1) % points.length];

        const segmentLength = current.distanceTo(next);

        positions.push(
            current.x, current.y, current.z,
            next.x, next.y, next.z
        );

        positions.push(
            next.x, current.y + height, next.z,
            current.x, current.y + height, current.z
        );

        const segmentUStart = currentLength / totalLength;
        const segmentUEnd = (currentLength + segmentLength) / totalLength;
        uvs.push(
            segmentUStart * 2, 0,
            segmentUEnd * 2, 0,
            segmentUEnd * 2, 1,
            segmentUStart * 2, 1
        );

        const vertexOffset = i * 4;
        indices.push(
            vertexOffset, vertexOffset + 1, vertexOffset + 2,
            vertexOffset, vertexOffset + 2, vertexOffset + 3
        );
        currentLength += segmentLength;
    }

    const geometry = new THREE.BufferGeometry();
    geometry.setAttribute('position', new THREE.Float32BufferAttribute(positions, 3));
    geometry.setAttribute('uv', new THREE.Float32BufferAttribute(uvs, 2));
    geometry.setIndex(indices);
    geometry.computeVertexNormals();
    return geometry;
}

animate()

function animate() {
    texture.offset.y -= 0.005;
    requestAnimationFrame(animate)
    renderer.render(scene, camera)
}

window.onresize = () => {
    renderer.setSize(box.clientWidth, box.clientHeight)
    camera.aspect = box.clientWidth / box.clientHeight
    camera.updateProjectionMatrix()
}

new GUI().addColor({ color: color.getHex() }, 'color').onChange(value => {
    material.color.set(value);
    fence2.material.color.set(value);
});
```

完整源码：[GitHub](https://github.com/z2586300277/three-cesium-examples/blob/dev/threeExamples/application/sportFence.js)

## 小结

- 本文提供 **流动围栏** 完整 Three.js 源码与在线 Demo，建议先运行案例再改 uniform/参数做二次实验
- 更多 Three.js 实战案例见 [three-cesium-examples 合集](https://threehub.cn) 与 [GitHub 开源仓库](https://github.com/z2586300277/three-cesium-examples)

