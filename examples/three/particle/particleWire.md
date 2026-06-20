---
title: "Three.js 粒子线教程"
description: "详解 Three.js 粒子线：基于 WebGL 实现「粒子线」可视化效果，附完整可运行源码，涵盖 ShaderMaterial、THREE.Points、BufferGeometry 等关键实现，附完整源码与在线 Demo，适合 Three.js 粒子 学习与二次开发。"
head:
  - - meta
    - name: keywords
      content: "Three.js,粒子线,WebGL,源码,教程,在线案例,ShaderMaterial,自定义着色器,GLSL,粒子特效,Points,BufferGeometry"
outline: deep
---

### 粒子线 · *Wire* · [▶ 在线运行案例](https://z2586300277.github.io/three-cesium-examples/#/codeMirror?navigation=ThreeJS&classify=particle&id=particleWire)
- **案例合集：** [三维可视化功能案例（threehub.cn）](https://threehub.cn)

- **开源仓库github地址：** https://github.com/z2586300277/three-cesium-examples

- **400个案例代码: ** [网盘链接](https://pan.quark.cn/s/201da5c82fec)

![粒子线](https://z2586300277.github.io/three-cesium-examples/threeExamples/particle/particleWire.jpg)

## 你将学到什么

- ShaderMaterial 自定义着色器实现核心视觉效果
- THREE.Points 粒子点渲染
- BufferGeometry 自定义顶点/索引数据
- `requestAnimationFrame` 渲染循环与 `resize` 自适应

## 效果说明

本案例演示 **粒子线** 效果：基于 WebGL 实现「粒子线」可视化效果，附完整可运行源码；核心用到 ShaderMaterial、THREE.Points、BufferGeometry。建议先打开文首在线案例查看动态画面，再对照下方源码逐步理解。

## 核心概念
- **Scene / Camera / WebGLRenderer** 构成最小渲染闭环；大场景可开 `logarithmicDepthBuffer` 缓解 Z-fighting。
- **ShaderMaterial** 通过 `uniforms` + 自定义 GLSL 控制逐像素/逐点效果；透明粒子常配合 `depthTest: false`。
- **THREE.Points** 将每个顶点渲染为可控大小的粒子；可用自定义 attribute（如 `u_index`）驱动片元/顶点动画。

## 实现步骤

1. 搭建 Scene、PerspectiveCamera、WebGLRenderer，挂载 canvas 并处理 `resize`
2. 定义 uniforms / onBeforeCompile 或 ShaderMaterial，编写 GLSL 与材质参数
3. 在 `requestAnimationFrame` 循环中更新状态并 render（Cesium 为 `viewer.render` 或自动渲染）

## 代码要点

```js
import * as THREE from 'three'

const container = document.getElementById('box');

const scene = new THREE.Scene();

const camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 1000);

const renderer = new THREE.WebGLRenderer({ antialias: true });

renderer.setSize(window.innerWidth, window.innerHeight);

container.appendChild(renderer.domElement);

const particlesGeometry = new THREE.BufferGeometry();

const particleCount = 300;

const posArray = new Float32Array(particleCount * 3);

const velocityArray = new Float32Array(particleCount * 3);

for (let i = 0; i < particleCount * 3; i++) { 
    
    posArray[i] = (Math.random() - 0.5) * 8; velocityArray[i] = (Math.random() - 0.5) * 0.02;

}

particlesGeometry.setAttribute('position', new THREE.BufferAttribute(posArray, 3)); 

particlesGeometry.setAttribute('velocity', new THREE.BufferAttribute(velocityArray, 3));

const particleMaterial = new THREE.ShaderMaterial({
    vertexShader: `
        uniform float uTime;
        uniform vec2 uMouse;
        attribute vec3 velocity;
        varying vec3 vPosition;
        varying float vDistance;
        void main() {
            vPosition = position;
            vec3 pos = position;
            float dist = length(pos.xy - uMouse);
            float influence = 1.0 - clamp(dist / 2.0, 0.0, 1.0);
            pos.xy += uMouse * influence * 0.1;
            pos += velocity * sin(uTime + position.x * 2.0);
            pos.y += sin(uTime * 0.5 + position.x) * 0.2;
            vec4 mvPosition = modelViewMatrix * vec4(pos, 1.0);
            vDistance = influence;
            gl_PointSize = mix(3.0, 8.0, influence) * (1.0 / -mvPosition.z);
            gl_Position = projectionMatrix * mvPosition;
        }
    `,
    fragmentShader: `
        uniform float uTime;
        varying vec3 vPosition;
        varying float vDistance;
        void main() {
            float distanceToCenter = length(gl_PointCoord - vec2(0.5));
            if (distanceToCenter > 0.5) discard;
            vec3 baseColor = vec3(0.3, 0.6, 1.0);
            vec3 pulseColor = vec3(1.0, 0.4, 0.8);
            vec3 color = mix(baseColor, pulseColor, vDistance);
            color = color + 0.2 * sin(uTime + vPosition.xyx + vec3(0,2,4));
            float alpha = 1.0 - distanceToCenter * 2.0;
            alpha *= 0.8 + 0.2 * sin(uTime * 2.0);
            gl_FragColor = vec4(color, alpha);
        }
    `,
    uniforms: {
        uTime: { value: 0 },
        uMouse: { value: new THREE.Vector2(0, 0) }
    },
    transparent: true,
    blending: THREE.AdditiveBlending
});

const particleSystem = new THREE.Points(particlesGeometry, particleMaterial); scene.add(particleSystem);

const linesMaterial = new THREE.ShaderMaterial({
    vertexShader: `
        uniform float uTime;
        uniform vec2 uMouse;
        varying vec3 vPosition;
        varying float vDistance;
        void main() {
            vPosition = position;
            vec3 pos = position;
            float dist = length(pos.xy - uMouse);
            float influence = 1.0 - clamp(dist / 2.0, 0.0, 1.0);
            pos.xy += uMouse * influence * 0.1;
            pos.y += sin(uTime * 0.5 + position.x) * 0.2;
            vDistance = influence;
            gl_Position = projectionMatrix * modelViewMatrix * vec4(pos, 1.0);
        }
    `,
    fragmentShader: `
        uniform float uTime;
        varying vec3 vPosition;
        varying float vDistance;
        void main() {
            vec3 baseColor = vec3(0.3, 0.6, 1.0);
            vec3 pulseColor = vec3(1.0, 0.4, 0.8);
            vec3 color = mix(baseColor, pulseColor, vDistance);
            color = color + 0.2 * sin(uTime + vPosition.xyx + vec3(0,2,4));
            float alpha = 0.15 + 0.1 * vDistance;
            alpha *= 0.8 + 0.2 * sin(uTime * 2.0);
            gl_FragColor = vec4(color, alpha);
        }
    `,
    uniforms: {
        uTime: { value: 0 },
        uMouse: { value: new THREE.Vector2(0, 0) }
    },
    transparent: true,
    blending: THREE.AdditiveBlending
});

const linesGeometry = new THREE.BufferGeometry();
const linePositions = [];
for (let i = 0; i < particleCount; i++) {
    for (let j = i + 1; j < particleCount; j++) {
        const dist = Math.sqrt(
            Math.pow(posArray[i * 3] - posArray[j * 3], 2) +
            Math.pow(posArray[i * 3 + 1] - posArray[j * 3 + 1], 2)
        );
        if (dist < 2) {
            linePositions.push(
                posArray[i * 3], posArray[i * 3 + 1], posArray[i * 3 + 2],
                posArray[j * 3], posArray[j * 3 + 1], posArray[j * 3 + 2]
            );
        }
    }
}
linesGeometry.setAttribute('position', new THREE.Float32BufferAttribute(linePositions, 3));

const lines = new THREE.LineSegments(linesGeometry, linesMaterial);

scene.add(lines);

camera.position.z = 5;

const mouse = new THREE.Vector2(); let isMouseDown = false;

function onMouseMove(event) { mouse.x = (event.clientX / window.innerWidth) * 2 - 1; mouse.y = -(event.clientY / window.innerHeight) * 2 + 1; particleMaterial.uniforms.uMouse.value = mouse; linesMaterial.uniforms.uMouse.value = mouse; }

function onMouseDown() { isMouseDown = true; }

function onMouseUp() { isMouseDown = false; }

window.addEventListener('mousemove', onMouseMove);

window.addEventListener('mousedown', onMouseDown);

window.addEventListener('mouseup', onMouseUp);

window.addEventListener('touchstart', () => isMouseDown = true);

window.addEventListener('touchend', () => isMouseDown = false);

function animate(timestamp) {

    requestAnimationFrame(animate); const time = timestamp * 0.001;

    particleMaterial.uniforms.uTime.value = time;
    
    linesMaterial.uniforms.uTime.value = time;

    const positions = particlesGeometry.attributes.position.array; const velocities = particlesGeometry.attributes.velocity.array;

    for (let i = 0; i < positions.length; i += 3) {

        if (isMouseDown) { velocities[i] += (Math.random() - 0.5) * 0.001; velocities[i + 1] += (Math.random() - 0.5) * 0.001; }

        positions[i] += velocities[i]; positions[i + 1] += velocities[i + 1];

        if (Math.abs(positions[i]) > 4) velocities[i] *= -0.9; if (Math.abs(positions[i + 1]) > 4) velocities[i + 1] *= -0.9;

    }

    particlesGeometry.attributes.position.needsUpdate = true;

    const linePositions = linesGeometry.attributes.position.array; let lineIndex = 0; for (let i = 0; i < particleCount; i++) { for (let j = i + 1; j < particleCount; j++) { const dist = Math.sqrt(Math.pow(positions[i * 3] - positions[j * 3], 2) + Math.pow(positions[i * 3 + 1] - positions[j * 3 + 1], 2)); if (dist < 2) { linePositions[lineIndex++] = positions[i * 3]; linePositions[lineIndex++] = positions[i * 3 + 1]; linePositions[lineIndex++] = positions[i * 3 + 2]; linePositions[lineIndex++] = positions[j * 3]; linePositions[lineIndex++] = positions[j * 3 + 1]; linePositions[lineIndex++] = positions[j * 3 + 2]; } } } linesGeometry.attributes.position.needsUpdate = true;

    renderer.render(scene, camera);
}


function onWindowResize() { 
    
    camera.aspect = window.innerWidth / window.innerHeight;
    
    camera.updateProjectionMatrix(); 
    
    renderer.setSize(window.innerWidth, window.innerHeight);

}

window.addEventListener('resize', onWindowResize); animate(0);
```

完整源码：[GitHub](https://github.com/z2586300277/three-cesium-examples/blob/dev/threeExamples/particle/particleWire.js)

## 小结

- 本文提供 **粒子线** 完整 Three.js 源码与在线 Demo，建议先运行案例再改 uniform/参数做二次实验
- 更多 Three.js 实战案例见 [three-cesium-examples 合集](https://threehub.cn) 与 [GitHub 开源仓库](https://github.com/z2586300277/three-cesium-examples)

