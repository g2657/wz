---
title: "Three.js 透明渐变教程"
description: "详解 Three.js 透明渐变：基于 WebGL 实现「透明渐变」可视化效果，附完整可运行源码，涵盖 ShaderMaterial、OrbitControls 等关键实现，附完整源码与在线 Demo，适合 Three.js 着色器 学习与二次开发。"
head:
  - - meta
    - name: keywords
      content: "Three.js,透明渐变,WebGL,源码,教程,在线案例,ShaderMaterial,自定义着色器,GLSL,OrbitControls,相机控制"
outline: deep
---

### 透明渐变 · *Trans Grad* · [▶ 在线运行案例](https://z2586300277.github.io/three-cesium-examples/#/codeMirror?navigation=ThreeJS&classify=shader&id=transparentGradient)
- **案例合集：** [三维可视化功能案例（threehub.cn）](https://threehub.cn)

- **开源仓库github地址：** https://github.com/z2586300277/three-cesium-examples

- **400个案例代码: ** [网盘链接](https://pan.quark.cn/s/201da5c82fec)

![透明渐变](https://z2586300277.github.io/three-cesium-examples/threeExamples/shader/transparentGradient.jpg)

## 你将学到什么

- ShaderMaterial 自定义着色器实现核心视觉效果
- OrbitControls 相机轨道交互
- `requestAnimationFrame` 渲染循环与 `resize` 自适应

## 效果说明

本案例演示 **透明渐变** 效果：基于 WebGL 实现「透明渐变」可视化效果，附完整可运行源码；核心用到 ShaderMaterial、OrbitControls。建议先打开文首在线案例查看动态画面，再对照下方源码逐步理解。

## 核心概念
- **Scene / Camera / WebGLRenderer** 构成最小渲染闭环；大场景可开 `logarithmicDepthBuffer` 缓解 Z-fighting。
- **ShaderMaterial** 通过 `uniforms` + 自定义 GLSL 控制逐像素/逐点效果；透明粒子常配合 `depthTest: false`。
- **OrbitControls** 提供轨道旋转/缩放；开启 `enableDamping` 后需在 animate 中 `controls.update()`。

## 实现步骤

1. 搭建 Scene、PerspectiveCamera、WebGLRenderer，挂载 canvas 并处理 `resize`
2. 定义 uniforms / onBeforeCompile 或 ShaderMaterial，编写 GLSL 与材质参数
3. 创建 OrbitControls（及 Raycaster 等交互控件，若源码包含）
4. 在 `requestAnimationFrame` 循环中更新状态并 render（Cesium 为 `viewer.render` 或自动渲染）

## 代码要点

```js
import * as THREE from 'three'
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls.js'
import { GUI } from "three/examples/jsm/libs/lil-gui.module.min.js";

const box = document.getElementById('box')

const scene = new THREE.Scene()

const camera = new THREE.PerspectiveCamera(50, box.clientWidth / box.clientHeight, 0.1, 1000)

camera.position.set(0, 0, 50)

const renderer = new THREE.WebGLRenderer({ antialias: true, alpha: true, logarithmicDepthBuffer: true })

renderer.setSize(box.clientWidth, box.clientHeight)

box.appendChild(renderer.domElement)

const controls = new OrbitControls(camera, renderer.domElement)

controls.enableDamping = true

const uniforms = {
    color: { value: new THREE.Color(0xffffff * Math.random()) },
    uvScale: { value: 0.1 },
    intensity: { value: 3 }
}

const material = new THREE.ShaderMaterial({
    vertexShader: `
    varying vec2 vUv;
    void main() {
      vUv = uv; 
      gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0);
    }
  `,
    fragmentShader: `
    varying vec2 vUv;
    uniform vec3 color;
    uniform float uvScale;
    uniform float intensity;
    void main() {
      vec2 uv = vUv * uvScale;
      float distance = length(uv);
      float alpha = smoothstep(0.0, 1., distance);
      gl_FragColor = vec4(color * intensity, alpha);
    }
  `,
    transparent: true,
    side: THREE.DoubleSide,
    uniforms: uniforms

});

const gui = new GUI()
gui.addColor(material.uniforms.color, 'value').name('color')
gui.add(material.uniforms.intensity, 'value').min(0).max(10).name('intensity')
gui.add(material.uniforms.uvScale, 'value').min(0).max(1).name('uvScale')

// 通过点绘制成一个五角星
function createStarShape(radiusOuter, radiusInner, points) {
    const shape = new THREE.Shape();
    const angleStep = (Math.PI * 2) / points;

    for (let i = 0; i < points; i++) {
        const angleOuter = i * angleStep; // 外点的角度
        const angleInner = angleOuter + angleStep / 2; // 内点的角度

        const xOuter = Math.cos(angleOuter) * radiusOuter;
        const yOuter = Math.sin(angleOuter) * radiusOuter;
        const xInner = Math.cos(angleInner) * radiusInner;
        const yInner = Math.sin(angleInner) * radiusInner;

        if (i === 0) {
            shape.moveTo(xOuter, yOuter); // 第一个点
        } else {
            shape.lineTo(xOuter, yOuter); // 连接到外点
        }
        shape.lineTo(xInner, yInner); // 连接到内点
    }

    shape.closePath(); // 闭合形状

    return shape;
}

const starShape = createStarShape(5, 2, 5);

const starGeometry = new THREE.ShapeGeometry(starShape);

const star = new THREE.Mesh(starGeometry, material);

star.position.y += 10;

scene.add(star)

// 随机绘制成 4，5，6，7，8 边形
function createPolygonShape(radius, points) {

    const shape = new THREE.Shape();

    const angleStep = (Math.PI * 2) / points;

    for (let i = 0; i < points; i++) {

        const angle = i * angleStep;

        const x = Math.cos(angle) * radius;

        const y = Math.sin(angle) * radius;

        if (i === 0) {

            shape.moveTo(x, y);

        } else {

            shape.lineTo(x, y);

        }

    }

    shape.closePath();

    return shape;

}

const polygonGeometries = [4, 5, 6, 7, 8].map(points => {

    const shape = createPolygonShape(5, points);

    return new THREE.ShapeGeometry(shape);

});

const polygons = polygonGeometries.map(geometry => {

    const m = new THREE.ShaderMaterial({
        vertexShader: material.vertexShader,
        fragmentShader: material.fragmentShader,
        transparent: true,
        side: THREE.DoubleSide,
        uniforms: {
            ...uniforms,
            color: {
                value: new THREE.Color(0xffffff * Math.random())
            }
        }
    })

    return new THREE.Mesh(geometry, m)

})

polygons.forEach((polygon, index) => {

    polygon.position.x = (index - 2) * 10;

    scene.add(polygon);

});

animate()

function animate() {

    requestAnimationFrame(animate)

    controls.update()

    renderer.render(scene, camera)

}

window.onresize = () => {

    renderer.setSize(box.clientWidth, box.clientHeight)

    camera.aspect = box.clientWidth / box.clientHeight

    camera.updateProjectionMatrix()

}

scene.background = new THREE.CubeTextureLoader().load([0, 1, 2, 3, 4, 5].map(k => ('https://z2586300277.github.io/three-editor/dist/files/scene/skyBox0/' + (k + 1) + '.png')));
```

完整源码：[GitHub](https://github.com/z2586300277/three-cesium-examples/blob/dev/threeExamples/shader/transparentGradient.js)

## 小结

- 本文提供 **透明渐变** 完整 Three.js 源码与在线 Demo，建议先运行案例再改 uniform/参数做二次实验
- 更多 Three.js 实战案例见 [three-cesium-examples 合集](https://threehub.cn) 与 [GitHub 开源仓库](https://github.com/z2586300277/three-cesium-examples)

