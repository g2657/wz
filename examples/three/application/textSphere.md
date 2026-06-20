---
title: "Three.js 球体文字教程"
description: "详解 Three.js 球体文字：基于 WebGL 实现「球体文字」可视化效果，附完整可运行源码，涵盖 OrbitControls 等关键实现，附完整源码与在线 Demo，适合 Three.js 应用场景 学习与二次开发。"
head:
  - - meta
    - name: keywords
      content: "Three.js,球体文字,WebGL,源码,教程,在线案例,OrbitControls,相机控制"
outline: deep
---

### 球体文字 · *Text Sphere* · [▶ 在线运行案例](https://z2586300277.github.io/three-cesium-examples/#/codeMirror?navigation=ThreeJS&classify=application&id=textSphere)
- **案例合集：** [三维可视化功能案例（threehub.cn）](https://threehub.cn)

- **开源仓库github地址：** https://github.com/z2586300277/three-cesium-examples

- **400个案例代码: ** [网盘链接](https://pan.quark.cn/s/201da5c82fec)

![球体文字](https://z2586300277.github.io/three-cesium-examples/threeExamples/application/textSphere.jpg)

## 你将学到什么

- OrbitControls 相机轨道交互
- `requestAnimationFrame` 渲染循环与 `resize` 自适应

## 效果说明

本案例演示 **球体文字** 效果：基于 WebGL 实现「球体文字」可视化效果，附完整可运行源码；核心用到 OrbitControls。建议先打开文首在线案例查看动态画面，再对照下方源码逐步理解。

## 核心概念
- **Scene / Camera / WebGLRenderer** 构成最小渲染闭环；大场景可开 `logarithmicDepthBuffer` 缓解 Z-fighting。
- **OrbitControls** 提供轨道旋转/缩放；开启 `enableDamping` 后需在 animate 中 `controls.update()`。
- 阅读下方完整源码时，建议从 `init` / `load` / `animate` 三条主线入手，再深入 shader 与工具函数。

## 实现步骤

1. 搭建 Scene、PerspectiveCamera、WebGLRenderer，挂载 canvas 并处理 `resize`
2. 创建 OrbitControls（及 Raycaster 等交互控件，若源码包含）
3. 在 `requestAnimationFrame` 循环中更新状态并 render（Cesium 为 `viewer.render` 或自动渲染）

## 代码要点

```js
import * as THREE from 'three'
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls.js'
import { FontLoader } from 'three/examples/jsm/loaders/FontLoader.js'
import { TextGeometry } from 'three/examples/jsm/geometries/TextGeometry.js'

const DOM = document.getElementById('box')
const scene = new THREE.Scene()
const camera = new THREE.PerspectiveCamera(75, DOM.clientWidth / DOM.clientHeight, 0.1, 1000)
camera.position.set(1, 2, 3)
const renderer = new THREE.WebGLRenderer({ antialias: true, alpha: true, logarithmicDepthBuffer: true })
renderer.setSize(DOM.clientWidth, DOM.clientHeight)
DOM.appendChild(renderer.domElement)
const controls = new OrbitControls(camera, renderer.domElement)
controls.enableDamping = true
controls.dampingFactor = 0.01

animate()
function animate() {
    controls.update()
    renderer.render(scene, camera)
    requestAnimationFrame(animate)
}

window.onresize = () => {
    renderer.setSize(DOM.clientWidth, DOM.clientHeight)
    camera.aspect = DOM.clientWidth / DOM.clientHeight
    camera.updateProjectionMatrix()
}

scene.add(new THREE.AxesHelper(1000))

scene.add(new THREE.AmbientLight(0x404040, 5));

let directionalLight = new THREE.DirectionalLight(0xffffff, 0.5);
directionalLight.castShadow = true;
directionalLight.position.set(0, 50, 0);

scene.add(directionalLight);

const TEXT = [
    "three.js",
    "tres.js",
    "react-three-fiber",
    "cesium",
    "javascript",
    "html",
    "babylon",
    "webgl",
    "glsl",
    "shader",
    "css",
    "vue",
    "vite",
];


// 创建球体几何体
const radius = 2;
const segments = 64;
const rings = 64;
const geometry = new THREE.SphereGeometry(radius, segments, rings);

const widthSegments = 32; // 横向细分段数
const heightSegments = 32; // 纵向细分段数

// 生成顶点数据
const positions = [];
for (let phiIndex = 0; phiIndex <= heightSegments; phiIndex++) {
    const phi = (phiIndex * Math.PI) / heightSegments;
    for (let thetaIndex = 0; thetaIndex <= widthSegments; thetaIndex++) {
        const theta = (thetaIndex * 2 * Math.PI) / widthSegments;

        // 计算点的坐标
        const x = radius * Math.sin(phi) * Math.cos(theta);
        const y = radius * Math.sin(phi) * Math.sin(theta);
        const z = radius * Math.cos(phi);

        positions.push(x, y, z);
    }
}

// 创建球体材质
const material = new THREE.MeshBasicMaterial({
    color: 0x00ff00, // 球体的颜色
    transparent: true, // 开启透明
    opacity: 0, // 设置透明度
});

// 创建球体网格
const sphere = new THREE.Mesh(geometry, material);
scene.add(sphere);

const font = await new Promise((resolve) => new FontLoader().load('https://z2586300277.github.io/three-editor/dist/files/font/cn1.json', (f) => resolve(f)))

const totalCount = positions.length;

const savePos = [];
const textMeshes = []; // 存储文本网格的数组

// 创建并布局文本网格
for (let i = 0; i < totalCount; i += 3) {
    const x = positions[i];
    const y = positions[i + 1];
    const z = positions[i + 2];

    const pos = new THREE.Vector3(x, y, z);

    // 遍历文本数组
    const index = THREE.MathUtils.randInt(0, TEXT.length - 1);
    const textGeometry = new TextGeometry(TEXT[index], {
        font,
        size: 0.1 /* 字体大小 */,
        depth: 0.001 /* 文本厚度 */,
        curveSegments: 1 /* 曲线点数 (5降低优化性能) */,
        bevelEnabled: true /* 是否开启斜角 */,
        bevelThickness: 0.001 /* 斜角深度 */,
        bevelSize: 0.001 /* 斜角与原始文本轮廓之间的延伸距离 */,
        bevelSegments: 1 /* 斜角的分段数 (3降低优化性能) */,
        bevelOffset: 0 /* 斜角偏移 */,
    });

    // 计算文本几何体的边界框
    textGeometry.computeBoundingBox();

    if (textGeometry) {
        // 根据文本几何体计算布局位置
        const textWidth = textGeometry.boundingBox.max.x - textGeometry.boundingBox.min.x;
        const textHeight = textGeometry.boundingBox.max.y - textGeometry.boundingBox.min.y;
        const textCenterX = -textWidth / 2;
        const textCenterY = -textHeight / 2;
        // 平移到球体表面
        textGeometry.translate(textCenterX, textCenterY, 0);

        // 创建文本网格
        const textMaterial = new THREE.MeshBasicMaterial({ color: 0xffffff }); // 使用基础材质
        const textMesh = new THREE.Mesh(textGeometry, textMaterial);

        // 获取球体表面的法线向量
        const normal = new THREE.Vector3(pos.x, pos.y, pos.z).normalize();
        // 将文本网格平移到球体表面
        textMesh.position.copy(normal.multiplyScalar(radius));

        // 调整文字朝向球心
        textMesh.lookAt(new THREE.Vector3(pos.x * 2, pos.y * 2, pos.z * 2));

        const isClose = savePos.some(
            (p) => p.distanceTo(textMesh.position) <= 0.8
        );
        // 根据距离调整密度因子
        if (!isClose) {
            // 将文本网格添加到球体上
            sphere.add(textMesh);
            // 存储文本网格到数组中
            textMeshes.push(textMesh);
            savePos.push(textMesh.position);
        }
    }
}
```

完整源码：[GitHub](https://github.com/z2586300277/three-cesium-examples/blob/dev/threeExamples/application/textSphere.js)

## 小结

- 本文提供 **球体文字** 完整 Three.js 源码与在线 Demo，建议先运行案例再改 uniform/参数做二次实验
- 更多 Three.js 实战案例见 [three-cesium-examples 合集](https://threehub.cn) 与 [GitHub 开源仓库](https://github.com/z2586300277/three-cesium-examples)

