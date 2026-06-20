---
title: "Three.js 蒸汽粒子教程"
description: "详解 Three.js 蒸汽粒子：基于 WebGL 实现「蒸汽粒子」可视化效果，附完整可运行源码，涵盖 ShaderMaterial、OrbitControls、THREE.Points 等关键实现，附完整源码与在线 Demo，适合 Three.js 粒子 学习与二次开发。"
head:
  - - meta
    - name: keywords
      content: "Three.js,蒸汽粒子,WebGL,源码,教程,在线案例,ShaderMaterial,自定义着色器,GLSL,OrbitControls,相机控制,粒子特效"
outline: deep
---

### 蒸汽粒子 · *Steam Particle* · [▶ 在线运行案例](https://z2586300277.github.io/three-cesium-examples/#/codeMirror?navigation=ThreeJS&classify=particle&id=steamParticle)
- **案例合集：** [三维可视化功能案例（threehub.cn）](https://threehub.cn)

- **开源仓库github地址：** https://github.com/z2586300277/three-cesium-examples

- **400个案例代码: ** [网盘链接](https://pan.quark.cn/s/201da5c82fec)

![蒸汽粒子](https://z2586300277.github.io/three-cesium-examples/threeExamples/particle/steamParticle.jpg)

## 你将学到什么

- ShaderMaterial 自定义着色器实现核心视觉效果
- OrbitControls 相机轨道交互
- THREE.Points 粒子点渲染
- BufferGeometry 自定义顶点/索引数据
- `requestAnimationFrame` 渲染循环与 `resize` 自适应

## 效果说明

本案例演示 **蒸汽粒子** 效果：基于 WebGL 实现「蒸汽粒子」可视化效果，附完整可运行源码；核心用到 ShaderMaterial、OrbitControls、THREE.Points。建议先打开文首在线案例查看动态画面，再对照下方源码逐步理解。

## 核心概念
- **Scene / Camera / WebGLRenderer** 构成最小渲染闭环；大场景可开 `logarithmicDepthBuffer` 缓解 Z-fighting。
- **ShaderMaterial** 通过 `uniforms` + 自定义 GLSL 控制逐像素/逐点效果；透明粒子常配合 `depthTest: false`。
- **OrbitControls** 提供轨道旋转/缩放；开启 `enableDamping` 后需在 animate 中 `controls.update()`。
- **THREE.Points** 将每个顶点渲染为可控大小的粒子；可用自定义 attribute（如 `u_index`）驱动片元/顶点动画。

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

// 初始化场景
const box = document.getElementById('box')
const scene = new THREE.Scene()
scene.background = new THREE.Color(0x445566)

const camera = new THREE.PerspectiveCamera(50, box.clientWidth / box.clientHeight, 0.1, 1000)
camera.position.set(0, 8, 20)

// 设置渲染器
const renderer = new THREE.WebGLRenderer({ 
  antialias: true, 
  alpha: true 
})
renderer.setSize(box.clientWidth, box.clientHeight)
box.appendChild(renderer.domElement)

// 轨道控制器
const controls = new OrbitControls(camera, renderer.domElement)
controls.enableDamping = true

scene.add(new THREE.AmbientLight(0xffffff, 0.5))

// 可配置参数
const config = {
    particleCount: 3000,
    particleSize: 1.2,
    width: 12,
    depth: 2,
    height: 15,
    riseSpeed: 0.4,
    spread: 0.3,
    turbulence: 0.3,
    density: 0.4,
}

const uniforms = {
    time: { value: 0 },
    baseColor: { value: new THREE.Color(0xffffff) },
    height: { value: config.height },
    turbulence: { value: config.turbulence },
    density: { value: config.density },
}

const material = new THREE.ShaderMaterial({
    uniforms,
    vertexShader: `
        attribute float size;
        attribute float phase;
        attribute vec3 velocity;
        uniform float time;
        uniform float height;
        uniform float turbulence;
        varying float vAlpha;
        varying float vAge;
        void main() {
            float age = mod(time * 0.3 + phase, 1.0);
            vAge = age;
            vec3 pos = position + velocity * age * height;
            pos.x += sin(age * 8.0 + phase * 20.0) * turbulence * (0.5 + age);
            pos.z += cos(age * 6.0 + phase * 15.0) * turbulence * (0.3 + age * 0.5);
            pos.x *= (1.0 + age * 1.5);
            pos.z *= (1.0 + age * 0.8);
            vec4 mvPosition = modelViewMatrix * vec4(pos, 1.0);
            gl_Position = projectionMatrix * mvPosition;
            gl_PointSize = size * (1.0 + age * 3.0) * (250.0 / -mvPosition.z);
            float fadeIn = smoothstep(0.0, 0.1, age);
            float fadeOut = 1.0 - smoothstep(0.6, 1.0, age);
            vAlpha = fadeIn * fadeOut;
        }
    `,
    fragmentShader: `
        uniform vec3 baseColor;
        uniform float density;
        varying float vAlpha;
        varying float vAge;
        void main() {
            float dist = length(gl_PointCoord - 0.5) * 2.0;
            if (dist > 1.0) discard;
            float edge = 1.0 - smoothstep(0.3, 1.0, dist);
            vec3 color = mix(baseColor, vec3(0.85, 0.88, 0.92), vAge * 0.3);
            gl_FragColor = vec4(color, vAlpha * edge * density);
        }
    `,
    blending: THREE.NormalBlending,
    depthWrite: false,
    transparent: true,
})

function buildGeometry() {
    const geo = new THREE.BufferGeometry()
    const positions = [], sizes = [], phases = [], velocities = []
    for (let i = 0; i < config.particleCount; i++) {
        positions.push(
            (Math.random() - 0.5) * config.width,
            Math.random() * 0.3,
            (Math.random() - 0.5) * config.depth
        )
        sizes.push(config.particleSize * (0.6 + Math.random() * 0.8))
        phases.push(Math.random())
        velocities.push(
            (Math.random() - 0.5) * config.spread,
            config.riseSpeed * (0.8 + Math.random() * 0.4),
            (Math.random() - 0.5) * config.spread * 0.5
        )
    }
    geo.setAttribute('position', new THREE.Float32BufferAttribute(positions, 3))
    geo.setAttribute('size', new THREE.Float32BufferAttribute(sizes, 1))
    geo.setAttribute('phase', new THREE.Float32BufferAttribute(phases, 1))
    geo.setAttribute('velocity', new THREE.Float32BufferAttribute(velocities, 3))
    return geo
}

const steam = new THREE.Points(buildGeometry(), material)
scene.add(steam)

// GUI
const gui = new GUI()
gui.add(config, 'height', 5, 30).name('上升高度').onChange(v => uniforms.height.value = v)
gui.add(config, 'turbulence', 0, 1).name('湍流强度').onChange(v => uniforms.turbulence.value = v)
gui.add(config, 'density', 0.1, 1).name('浓度').onChange(v => uniforms.density.value = v)
gui.add(config, 'width', 1, 20).name('喷口宽度').onChange(() => { steam.geometry.dispose(); steam.geometry = buildGeometry() })
gui.add(config, 'depth', 0.5, 10).name('喷口深度').onChange(() => { steam.geometry.dispose(); steam.geometry = buildGeometry() })
gui.add(config, 'riseSpeed', 0.1, 1).name('上升速度').onChange(() => { steam.geometry.dispose(); steam.geometry = buildGeometry() })
gui.add(config, 'particleCount', 500, 8000, 500).name('粒子数量').onChange(() => { steam.geometry.dispose(); steam.geometry = buildGeometry() })
gui.add(config, 'particleSize', 0.3, 3).name('粒子大小').onChange(() => { steam.geometry.dispose(); steam.geometry = buildGeometry() })
gui.addColor({ color: '#ffffff' }, 'color').name('蒸汽颜色').onChange(v => uniforms.baseColor.value.set(v))

// 动画
const clock = new THREE.Clock()

function animate() {
    requestAnimationFrame(animate)
    uniforms.time.value = clock.getElapsedTime()
    controls.update()
    renderer.render(scene, camera)
}

animate()

window.onresize = () => {
    renderer.setSize(box.clientWidth, box.clientHeight)
    camera.aspect = box.clientWidth / box.clientHeight
    camera.updateProjectionMatrix()
}
```

完整源码：[GitHub](https://github.com/z2586300277/three-cesium-examples/blob/dev/threeExamples/particle/steamParticle.js)

## 小结

- 本文提供 **蒸汽粒子** 完整 Three.js 源码与在线 Demo，建议先运行案例再改 uniform/参数做二次实验
- 更多 Three.js 实战案例见 [three-cesium-examples 合集](https://threehub.cn) 与 [GitHub 开源仓库](https://github.com/z2586300277/three-cesium-examples)

