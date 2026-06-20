---
title: "Three.js 波浪粒子教程"
description: "详解 Three.js 波浪粒子：基于 WebGL 实现「波浪粒子」可视化效果，附完整可运行源码，涵盖 ShaderMaterial、THREE.Points 等关键实现，附完整源码与在线 Demo，适合 Three.js 粒子 学习与二次开发。"
head:
  - - meta
    - name: keywords
      content: "Three.js,波浪粒子,WebGL,源码,教程,在线案例,ShaderMaterial,自定义着色器,GLSL,粒子特效,Points"
outline: deep
---

### 波浪粒子 · *Wave* · [▶ 在线运行案例](https://z2586300277.github.io/three-cesium-examples/#/codeMirror?navigation=ThreeJS&classify=particle&id=waveParticleShader)
- **案例合集：** [三维可视化功能案例（threehub.cn）](https://threehub.cn)

- **开源仓库github地址：** https://github.com/z2586300277/three-cesium-examples

- **400个案例代码: ** [网盘链接](https://pan.quark.cn/s/201da5c82fec)

![波浪粒子](https://z2586300277.github.io/three-cesium-examples/threeExamples/particle/waveParticleShader.jpg)

## 你将学到什么

- ShaderMaterial 自定义着色器实现核心视觉效果
- THREE.Points 粒子点渲染
- `requestAnimationFrame` 渲染循环与 `resize` 自适应

## 效果说明

本案例演示 **波浪粒子** 效果：基于 WebGL 实现「波浪粒子」可视化效果，附完整可运行源码；核心用到 ShaderMaterial、THREE.Points。建议先打开文首在线案例查看动态画面，再对照下方源码逐步理解。

## 核心概念
- **Scene / Camera / WebGLRenderer** 构成最小渲染闭环；大场景可开 `logarithmicDepthBuffer` 缓解 Z-fighting。
- **ShaderMaterial** 通过 `uniforms` + 自定义 GLSL 控制逐像素/逐点效果；透明粒子常配合 `depthTest: false`。
- **THREE.Points** 将每个顶点渲染为可控大小的粒子；可用自定义 attribute（如 `u_index`）驱动片元/顶点动画。

## 实现步骤

1. 定义 uniforms，在 rAF 中更新并 render
2. 构建几何 attribute 或 instanceMatrix 并 add 到 scene
3. gui.add 绑定可调参数

## 代码要点

```js
import { BufferAttribute, Clock, Color, PerspectiveCamera, PlaneGeometry, Points, Scene, ShaderMaterial, WebGLRenderer } from 'three';
import { GUI } from 'three/examples/jsm/libs/lil-gui.module.min.js';

const sizes = {
    width: window.innerWidth,
    height: window.innerHeight
}

const scene = new Scene()

const camera = new PerspectiveCamera(75, sizes.width / sizes.height, 0.1, 100)
camera.position.z = 10
camera.position.y = 1.1
camera.position.x = 0
scene.add(camera)


const planeGeometry = new PlaneGeometry(20, 20, 150, 150)
const planeMaterial = new ShaderMaterial({
    uniforms: {
        uTime: { value: 0 },
        uElevation: { value: 0.482 },
        ucolor: { value: new Color(0x9ab0f4) },
        bsize: { value: 3 }
    },
    vertexShader: `
        uniform float uTime;
        uniform float uElevation;

        attribute float aSize;
        uniform float bsize;

        varying float vPositionY;
        varying float vPositionZ;

        void main() {
            vec4 modelPosition = modelMatrix * vec4(position, 1.0);
            modelPosition.y = sin(modelPosition.x - uTime) * sin(modelPosition.z * 0.6 + uTime) * uElevation;

            vec4 viewPosition = viewMatrix * modelPosition;
            gl_Position = projectionMatrix * viewPosition;

            gl_PointSize = 2.0 * aSize;
            gl_PointSize *= ( 1.0 / - viewPosition.z ) * bsize;

            vPositionY = modelPosition.y;
            vPositionZ = modelPosition.z;
        }
    `,
    fragmentShader: `
        varying float vPositionY;
        varying float vPositionZ;
        uniform vec3 ucolor;

        void main() {
            float strength = (vPositionY + 0.25) * 0.3;
            gl_FragColor = vec4(ucolor, strength);
        }
    `,
    transparent: true,
})
const planeSizesArray = new Float32Array(planeGeometry.attributes.position.count)
for (let i = 0; i < planeSizesArray.length; i++) {
    planeSizesArray[i] = Math.random() * 4.0
}
planeGeometry.setAttribute('aSize', new BufferAttribute(planeSizesArray, 1))
const plane = new Points(planeGeometry, planeMaterial)
plane.rotation.x = - Math.PI * 0.4
scene.add(plane)


window.addEventListener('resize', () => {
    sizes.width = window.innerWidth
    sizes.height = window.innerHeight
    camera.aspect = sizes.width / sizes.height
    camera.updateProjectionMatrix()
    renderer.setSize(sizes.width, sizes.height)
    renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2))
})

const renderer = new WebGLRenderer()
document.body.appendChild(renderer.domElement)
renderer.setSize(sizes.width, sizes.height)
renderer.setPixelRatio(Math.min(window.devicePixelRatio, 1))

const gui = new GUI()
const planeFolder = gui.addFolder('Plane')
planeFolder.addColor(planeMaterial.uniforms.ucolor, 'value').name('Color')
planeFolder.add(planeMaterial.uniforms.bsize, 'value').min(0).max(10).step(0.001).name('Size')

const clock = new Clock()
const animate = () => {
    const elapsedTime = clock.getElapsedTime()
    planeMaterial.uniforms.uTime.value = elapsedTime
    renderer.render(scene, camera)
    window.requestAnimationFrame(animate)
}

animate()
```

完整源码：[GitHub](https://github.com/z2586300277/three-cesium-examples/blob/dev/threeExamples/particle/waveParticleShader.js)

## 小结

- 本文提供 **波浪粒子** 完整 Three.js 源码与在线 Demo，建议先运行案例再改 uniform/参数做二次实验
- 更多 Three.js 实战案例见 [three-cesium-examples 合集](https://threehub.cn) 与 [GitHub 开源仓库](https://github.com/z2586300277/three-cesium-examples)

