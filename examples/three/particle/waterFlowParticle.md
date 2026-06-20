---
title: "Three.js 喷泉水流教程"
description: "详解 Three.js 喷泉水流：用 Canvas 2D 绘制内容并实时映射为 Three.js 纹理，涵盖 ShaderMaterial、OrbitControls、THREE.Points 等关键实现，附完整源码与在线 Demo，适合 Three.js 粒子 学习与二次开发。"
head:
  - - meta
    - name: keywords
      content: "Three.js,喷泉水流,WebGL,源码,教程,在线案例,ShaderMaterial,自定义着色器,GLSL,OrbitControls,相机控制,粒子特效"
outline: deep
---

### 喷泉水流 · *Water Flow* · [▶ 在线运行案例](https://z2586300277.github.io/three-cesium-examples/#/codeMirror?navigation=ThreeJS&classify=particle&id=waterFlowParticle)
- **案例合集：** [三维可视化功能案例（threehub.cn）](https://threehub.cn)

- **开源仓库github地址：** https://github.com/z2586300277/three-cesium-examples

- **400个案例代码: ** [网盘链接](https://pan.quark.cn/s/201da5c82fec)

![喷泉水流](https://z2586300277.github.io/three-cesium-examples/threeExamples/particle/waterFlowParticle.jpg)

## 你将学到什么

- ShaderMaterial 自定义着色器实现核心视觉效果
- OrbitControls 相机轨道交互
- THREE.Points 粒子点渲染
- Canvas 动态纹理贴图
- BufferGeometry 自定义顶点/索引数据
- `requestAnimationFrame` 渲染循环与 `resize` 自适应

## 效果说明

本案例演示 **喷泉水流** 效果：用 Canvas 2D 绘制内容并实时映射为 Three.js 纹理；核心用到 ShaderMaterial、OrbitControls、THREE.Points。建议先打开文首在线案例查看动态画面，再对照下方源码逐步理解。

## 核心概念
- **Scene / Camera / WebGLRenderer** 构成最小渲染闭环；大场景可开 `logarithmicDepthBuffer` 缓解 Z-fighting。
- **ShaderMaterial** 通过 `uniforms` + 自定义 GLSL 控制逐像素/逐点效果；透明粒子常配合 `depthTest: false`。
- **OrbitControls** 提供轨道旋转/缩放；开启 `enableDamping` 后需在 animate 中 `controls.update()`。
- **THREE.Points** 将每个顶点渲染为可控大小的粒子；可用自定义 attribute（如 `u_index`）驱动片元/顶点动画。
- **CanvasTexture** 每帧或按需把 2D Canvas 内容上传 GPU，适合动态文字、图表、视频帧贴图。

## 实现步骤

1. 搭建 Scene、PerspectiveCamera、WebGLRenderer，挂载 canvas 并处理 `resize`
2. 定义 uniforms / onBeforeCompile 或 ShaderMaterial，编写 GLSL 与材质参数
3. 创建 OrbitControls（及 Raycaster 等交互控件，若源码包含）
4. 在 `requestAnimationFrame` 循环中更新状态并 render（Cesium 为 `viewer.render` 或自动渲染）

## 代码要点

```js
import * as THREE from 'three'
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls.js'
import { GUI } from 'dat.gui'

const box = document.getElementById('box')
const scene = new THREE.Scene()
scene.background = new THREE.Color(0x1a1a2e)

const camera = new THREE.PerspectiveCamera(60, box.clientWidth / box.clientHeight, 0.1, 1000)
camera.position.set(0, 5, 12)

const renderer = new THREE.WebGLRenderer({ antialias: true })
renderer.setSize(box.clientWidth, box.clientHeight)
box.appendChild(renderer.domElement)

new OrbitControls(camera, renderer.domElement)
scene.add(new THREE.AmbientLight(0xffffff, 0.5))

// 可配置参数
const config = {
    count: 3000,
    nozzleX: 0,
    nozzleY: 6,
    velocityX: 1.5,
    velocityY: 2.0,
    spread: 1.5,
    gravity: 10,
    particleSize: 0.15,
    lifeTime: 2.0,
}

const uniforms = {
    time: { value: 0 },
    nozzlePos: { value: new THREE.Vector3(config.nozzleX, config.nozzleY, 0) },
    velocity: { value: new THREE.Vector2(config.velocityX, config.velocityY) },
    spread: { value: config.spread },
    gravity: { value: config.gravity },
    lifeTime: { value: config.lifeTime },
}

const material = new THREE.ShaderMaterial({
    uniforms,
    vertexShader: `
        attribute float size;
        attribute float phase;
        attribute vec3 randomVel;
        uniform float time;
        uniform vec3 nozzlePos;
        uniform vec2 velocity;
        uniform float spread;
        uniform float gravity;
        uniform float lifeTime;
        varying float vAlpha;
        
        void main() {
            // 粒子生命周期（循环）
            float age = mod(time + phase, lifeTime);
            float ageRatio = age / lifeTime;
            
            // 初始位置 + 随机偏移
            vec3 pos = nozzlePos;
            pos.x += randomVel.x * 0.2;
            pos.y += randomVel.y * 0.2;
            pos.z += randomVel.z * 0.3;
            
            // 初始速度 + 随机扩散
            vec3 vel = vec3(velocity.x, velocity.y, 0.0);
            vel += randomVel * spread;
            
            // 受重力的抛物运动
            pos += vel * age;
            pos.y -= 0.5 * gravity * age * age;
            
            // 透明度：开始淡入，结束淡出
            vAlpha = smoothstep(0.0, 0.05, ageRatio) * (1.0 - smoothstep(0.85, 1.0, ageRatio));
            
            vec4 mvPosition = modelViewMatrix * vec4(pos, 1.0);
            gl_Position = projectionMatrix * mvPosition;
            gl_PointSize = size * (250.0 / -mvPosition.z);
        }
    `,
    fragmentShader: `
        uniform sampler2D map;
        varying float vAlpha;
        
        void main() {
            vec4 texColor = texture2D(map, gl_PointCoord);
            gl_FragColor = vec4(texColor.rgb, texColor.a * vAlpha);
        }
    `,
    transparent: true,
    depthWrite: false,
    blending: THREE.AdditiveBlending,
})

// 生成纹理
const c = document.createElement('canvas')
c.width = c.height = 32
const ctx = c.getContext('2d')
const g = ctx.createRadialGradient(16, 16, 0, 16, 16, 16)
g.addColorStop(0, 'rgba(200,235,255,1)')
g.addColorStop(1, 'rgba(120,190,255,0)')
ctx.fillStyle = g
ctx.fillRect(0, 0, 32, 32)
uniforms.map = { value: new THREE.CanvasTexture(c) }

function buildGeometry() {
    const geo = new THREE.BufferGeometry()
    const positions = [], sizes = [], phases = [], randomVels = []
    
    for (let i = 0; i < config.count; i++) {
        positions.push(0, 0, 0)
        sizes.push(config.particleSize * (0.8 + Math.random() * 0.4))
        phases.push(Math.random() * config.lifeTime) // 错开生成时间
        randomVels.push(
            (Math.random() - 0.5),
            (Math.random() - 0.5) * 0.6,
            (Math.random() - 0.5) * 0.8
        )
    }
    
    geo.setAttribute('position', new THREE.Float32BufferAttribute(positions, 3))
    geo.setAttribute('size', new THREE.Float32BufferAttribute(sizes, 1))
    geo.setAttribute('phase', new THREE.Float32BufferAttribute(phases, 1))
    geo.setAttribute('randomVel', new THREE.Float32BufferAttribute(randomVels, 3))
    return geo
}

const particles = new THREE.Points(buildGeometry(), material)
scene.add(particles)

// GUI
const gui = new GUI()
gui.add(config, 'nozzleX', -5, 5).name('喷嘴X').onChange(v => uniforms.nozzlePos.value.x = v)
gui.add(config, 'nozzleY', 0, 10).name('喷嘴高度').onChange(v => uniforms.nozzlePos.value.y = v)
gui.add(config, 'velocityX', -5, 5).name('水平速度').onChange(v => uniforms.velocity.value.x = v)
gui.add(config, 'velocityY', 0, 5).name('上升速度').onChange(v => uniforms.velocity.value.y = v)
gui.add(config, 'spread', 0, 3).name('扩散').onChange(v => uniforms.spread.value = v)
gui.add(config, 'gravity', 0, 20).name('重力').onChange(v => uniforms.gravity.value = v)
gui.add(config, 'lifeTime', 0.5, 5).name('生命周期').onChange(v => uniforms.lifeTime.value = v)
gui.add(config, 'count', 500, 8000, 500).name('粒子数').onChange(() => {
    particles.geometry.dispose()
    particles.geometry = buildGeometry()
})
gui.add(config, 'particleSize', 0.05, 0.5).name('粒子大小').onChange(() => {
    particles.geometry.dispose()
    particles.geometry = buildGeometry()
})

const clock = new THREE.Clock()

function animate() {
    requestAnimationFrame(animate)
    uniforms.time.value = clock.getElapsedTime()
    renderer.render(scene, camera)
}
animate()

window.onresize = () => {
    renderer.setSize(box.clientWidth, box.clientHeight)
    camera.aspect = box.clientWidth / box.clientHeight
    camera.updateProjectionMatrix()
}
```

完整源码：[GitHub](https://github.com/z2586300277/three-cesium-examples/blob/dev/threeExamples/particle/waterFlowParticle.js)

## 小结

- 本文提供 **喷泉水流** 完整 Three.js 源码与在线 Demo，建议先运行案例再改 uniform/参数做二次实验
- 更多 Three.js 实战案例见 [three-cesium-examples 合集](https://threehub.cn) 与 [GitHub 开源仓库](https://github.com/z2586300277/three-cesium-examples)

