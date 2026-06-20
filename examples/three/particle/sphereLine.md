---
title: "Three.js 球体线条教程"
description: "详解 Three.js 球体线条：基于 WebGL 实现「球体线条」可视化效果，附完整可运行源码，涵盖 OrbitControls、THREE.Points、BufferGeometry 等关键实现，附完整源码与在线 Demo，适合 Three.js 粒子 学习与二次开发。"
head:
  - - meta
    - name: keywords
      content: "Three.js,球体线条,WebGL,源码,教程,在线案例,OrbitControls,相机控制,粒子特效,Points,BufferGeometry"
outline: deep
---
### 球体线条 · *Sphere Line* · [▶ 在线运行案例](https://z2586300277.github.io/three-cesium-examples/#/codeMirror?navigation=ThreeJS&classify=particle&id=sphereLine)
- **案例合集：** [三维可视化功能案例（threehub.cn）](https://threehub.cn)

- **开源仓库github地址：** https://github.com/z2586300277/three-cesium-examples

- **400个案例代码: ** [网盘链接](https://pan.quark.cn/s/201da5c82fec)

![球体线条](https://z2586300277.github.io/three-cesium-examples/threeExamples/particle/sphereLine.jpg)

## 你将学到什么

- OrbitControls 相机轨道交互
- THREE.Points 粒子点渲染
- BufferGeometry 自定义顶点/索引数据
- `requestAnimationFrame` 渲染循环与 `resize` 自适应

## 效果说明

本案例演示 **球体线条** 效果：基于 WebGL 实现「球体线条」可视化效果，附完整可运行源码；核心用到 OrbitControls、THREE.Points、BufferGeometry。建议先打开文首在线案例查看动态画面，再对照下方源码逐步理解。

## 核心概念

- **OrbitControls** 轨道旋转缩放；开 `enableDamping` 时每帧需 `controls.update()`。

- **Points** 大量顶点用点精灵渲染；**InstancedMesh** 相同几何体批量绘制，降低 draw call。

## 实现步骤

1. 搭建 Scene / Camera / Renderer 与 OrbitControls
2. rAF 循环中 update 并 render

## 代码要点

```js
import * as THREE from 'three'
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls.js'
import * as dat from 'dat.gui'

const box = document.getElementById('box')

const scene = new THREE.Scene()

const camera = new THREE.PerspectiveCamera(50, box.clientWidth / box.clientHeight, 0.1, 10000)

camera.position.set(1000, 1000, 1000)

const renderer = new THREE.WebGLRenderer({ antialias: true, alpha: true, logarithmicDepthBuffer: true })

renderer.setSize(box.clientWidth, box.clientHeight)

box.appendChild(renderer.domElement)

const controls = new OrbitControls(camera, renderer.domElement)

controls.enableDamping = true

let group;
let particlesData = [];
let positions, colors;
let particles;
let pointCloud;
let particlePositions;
let linesMesh;

let maxParticleCount = 1000;
let particleCount = 500;
let r = 800;

let effectController = {
    showDots: true,
    showLines: true,
    minDistance: 150,
    limitConnections: false,
    maxConnections: 20,
    particleCount: 500
}

group = new THREE.Group();
scene.add(group);

let segments = maxParticleCount * maxParticleCount;

positions = new Float32Array(segments * 3);
colors = new Float32Array(segments * 3);

let pMaterial = new THREE.PointsMaterial({
    color: 0xFFFFFF,
    size: 3,
    blending: THREE.AdditiveBlending,
    transparent: true,
    sizeAttenuation: false
});

particles = new THREE.BufferGeometry();
particlePositions = new Float32Array(maxParticleCount * 3);

function getPos(radius, a, b) {
    const x = radius * Math.sin(a) * Math.cos(b);
    const y = radius * Math.sin(a) * Math.sin(b);
    const z = radius * Math.cos(a);
    return { x, y, z };
}

for (let i = 0; i < maxParticleCount; i++) {

    const p = getPos(r, Math.PI * 2 * Math.random(), Math.PI * 2 * Math.random())

    let x = p.x;
    let y = p.y;
    let z = p.z;

    particlePositions[i * 3] = x;
    particlePositions[i * 3 + 1] = y;
    particlePositions[i * 3 + 2] = z;

    particlesData.push({
        velocity: new THREE.Vector3(- 1 + Math.random() * 2, - 1 + Math.random() * 2, - 1 + Math.random() * 2),
        numConnections: 0
    });

}

particles.setDrawRange(0, particleCount);
particles.setAttribute('position', new THREE.BufferAttribute(particlePositions, 3).setUsage(THREE.DynamicDrawUsage));

pointCloud = new THREE.Points(particles, pMaterial);
group.add(pointCloud);

let geometry = new THREE.BufferGeometry();

geometry.setAttribute('position', new THREE.BufferAttribute(positions, 3).setUsage(THREE.DynamicDrawUsage));
geometry.setAttribute('color', new THREE.BufferAttribute(colors, 3).setUsage(THREE.DynamicDrawUsage));

geometry.computeBoundingSphere();

geometry.setDrawRange(0, 0);

let material = new THREE.LineBasicMaterial({
    vertexColors: true,
    blending: THREE.AdditiveBlending,
    transparent: true
});

linesMesh = new THREE.LineSegments(geometry, material);
group.add(linesMesh);
animate()

function animate() {


    let vertexpos = 0;
    let colorpos = 0;
    let numConnected = 0;
    let O = new THREE.Vector3(0, 0, 0)

    for (let i = 0; i < particleCount; i++)
        particlesData[i].numConnections = 0;

    for (let i = 0; i < particleCount; i++) {

        // get the particle
        let particleData = particlesData[i];

        particlePositions[i * 3] += particleData.velocity.x;
        particlePositions[i * 3 + 1] += particleData.velocity.y;
        particlePositions[i * 3 + 2] += particleData.velocity.z;

        const v = new THREE.Vector3(particlePositions[i * 3], particlePositions[i * 3 + 1], particlePositions[i * 3 + 2])
        if (v.distanceTo(O) > r + 20) {
            particleData.velocity.x = - particleData.velocity.x;
            particleData.velocity.y = - particleData.velocity.y;
            particleData.velocity.z = - particleData.velocity.z;
        }
        if (v.distanceTo(O) < r) {
            particleData.velocity.x = + particleData.velocity.x;
            particleData.velocity.y = + particleData.velocity.y;
            particleData.velocity.z = + particleData.velocity.z;
        }

        if (effectController.limitConnections && particleData.numConnections >= effectController.maxConnections)
            continue;

        for (let j = i + 1; j < particleCount; j++) {

            let particleDataB = particlesData[j];
            if (effectController.limitConnections && particleDataB.numConnections >= effectController.maxConnections)
                continue;

            let dx = particlePositions[i * 3] - particlePositions[j * 3];
            let dy = particlePositions[i * 3 + 1] - particlePositions[j * 3 + 1];
            let dz = particlePositions[i * 3 + 2] - particlePositions[j * 3 + 2];
            let dist = Math.sqrt(dx * dx + dy * dy + dz * dz);

            if (dist < effectController.minDistance) {

                particleData.numConnections++;
                particleDataB.numConnections++;

                let alpha = 1.0 - dist / effectController.minDistance;

                positions[vertexpos++] = particlePositions[i * 3];
                positions[vertexpos++] = particlePositions[i * 3 + 1];
                positions[vertexpos++] = particlePositions[i * 3 + 2];

                positions[vertexpos++] = particlePositions[j * 3];
                positions[vertexpos++] = particlePositions[j * 3 + 1];
                positions[vertexpos++] = particlePositions[j * 3 + 2];

                colors[colorpos++] = alpha;
                colors[colorpos++] = alpha;
                colors[colorpos++] = alpha;

                colors[colorpos++] = alpha;
                colors[colorpos++] = alpha;
                colors[colorpos++] = alpha;

                numConnected++;

            }

        }

    }

    linesMesh.geometry.setDrawRange(0, numConnected * 2);
    linesMesh.geometry.attributes.position.needsUpdate = true;
    linesMesh.geometry.attributes.color.needsUpdate = true;

    pointCloud.geometry.attributes.position.needsUpdate = true;

    requestAnimationFrame(animate);

    group.rotation.y += 0.001
    renderer.render(scene, camera);
    controls.update()
}

let gui = new dat.GUI()

gui.add(effectController, "showDots").onChange(function (value) {

    pointCloud.visible = value;

});
gui.add(effectController, "showLines").onChange(function (value) {

    linesMesh.visible = value;

});
gui.add(effectController, "minDistance", 10, 300);
gui.add(effectController, "limitConnections");
gui.add(effectController, "maxConnections", 0, 30, 1);
gui.add(effectController, "particleCount", 0, maxParticleCount, 1).onChange(function (value) {

    particleCount = parseInt(value);
    particles.setDrawRange(0, particleCount);

});
```

完整源码：[GitHub](https://github.com/z2586300277/three-cesium-examples/blob/dev/threeExamples/particle/sphereLine.js)

## 小结

- 本文提供 **球体线条** 完整 Three.js 源码与在线 Demo，建议先运行案例再改 uniform/参数做二次实验
- 更多 Three.js 实战案例见 [three-cesium-examples 合集](https://threehub.cn) 与 [GitHub 开源仓库](https://github.com/z2586300277/three-cesium-examples)

