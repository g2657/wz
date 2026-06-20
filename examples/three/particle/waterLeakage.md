---
title: "Three.js 水流粒子教程"
description: "详解 Three.js 水流粒子：基于 WebGL 实现「水流粒子」可视化效果，附完整可运行源码，涵盖 ShaderMaterial、OrbitControls、THREE.Points 等关键实现，附完整源码与在线 Demo，适合 Three.js 粒子 学习与二次开发。"
head:
  - - meta
    - name: keywords
      content: "Three.js,水流粒子,WebGL,源码,教程,在线案例,ShaderMaterial,自定义着色器,GLSL,OrbitControls,相机控制,粒子特效"
outline: deep
---

### 水流粒子 · *Water Leakage* · [▶ 在线运行案例](https://z2586300277.github.io/three-cesium-examples/#/codeMirror?navigation=ThreeJS&classify=particle&id=waterLeakage)
- **案例合集：** [三维可视化功能案例（threehub.cn）](https://threehub.cn)

- **开源仓库github地址：** https://github.com/z2586300277/three-cesium-examples

- **400个案例代码: ** [网盘链接](https://pan.quark.cn/s/201da5c82fec)

![水流粒子](https://z2586300277.github.io/three-cesium-examples/threeExamples/particle/waterLeakage.jpg)

## 你将学到什么

- ShaderMaterial 自定义着色器实现核心视觉效果
- OrbitControls 相机轨道交互
- THREE.Points 粒子点渲染
- BufferGeometry 自定义顶点/索引数据
- `requestAnimationFrame` 渲染循环与 `resize` 自适应

## 效果说明

本案例演示 **水流粒子** 效果：基于 WebGL 实现「水流粒子」可视化效果，附完整可运行源码；核心用到 ShaderMaterial、OrbitControls、THREE.Points。建议先打开文首在线案例查看动态画面，再对照下方源码逐步理解。

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
import { GUI } from 'three/examples/jsm/libs/lil-gui.module.min.js'

const box = document.getElementById('box')
const scene = new THREE.Scene()
const camera = new THREE.PerspectiveCamera(75, box.clientWidth / box.clientHeight, 0.0000001, 100000) // 调整相机的近裁剪面为更小的值，让近距离的粒子可见
camera.position.set(10, 10, 5)
const renderer = new THREE.WebGLRenderer({ antialias: true, alpha: true, logarithmicDepthBuffer: true })
renderer.setSize(box.clientWidth, box.clientHeight)
box.appendChild(renderer.domElement)
new OrbitControls(camera, renderer.domElement)
scene.add(new THREE.AxesHelper(10))

const mesh = createMesh()
scene.add(mesh)

function animate() {
    mesh.render()
    requestAnimationFrame(animate);
    renderer.render(scene, camera);
}
animate()

window.onresize = () => {
    renderer.setSize(box.clientWidth, box.clientHeight)
    camera.aspect = box.clientWidth / box.clientHeight
    camera.updateProjectionMatrix()
}


/* 方法 */
function createMesh() {

    const params = {
        maxParticles: 3000,
        spawnRate: 12,
        gravity: 10,
        minSize: 0.5,
        maxSize: 1,
        minStrength: 1,
        maxStrength: 4,
        spread: 0.6,
        burstProbability: 0.7,
        burstMultiplier: 3,
        color: "#9da7af",
        blendingMode: "Additive",
        rainLength: 0.6,
        opacity: 0.6,
        collisionY: 0,      // 改名为collisionY，表示碰撞高度，而不是地面
        enableSplash: true,
        splashParticles: 6,
        splashSize: 0.3,
        splashSpeed: 3,
        splashLifeTime: 0.6
    }

    // 设置UI
    const gui = new GUI();
    const particleFolder = gui.addFolder('粒子系统');
    particleFolder.add(params, 'maxParticles').name('最大粒子数').onChange(value => {
        scene.remove(emitter.points);
        emitter = new ParticleSystem(value);
        scene.add(emitter.points);
    });
    particleFolder.add(params, 'spawnRate').name('生成速率');

    const particlePhysicsFolder = gui.addFolder('物理属性');
    particlePhysicsFolder.add(params, 'gravity').name('重力');
    particlePhysicsFolder.add(params, 'minStrength').name('最小喷射强度');
    particlePhysicsFolder.add(params, 'maxStrength').name('最大喷射强度');
    particlePhysicsFolder.add(params, 'spread').name('发散程度').step(0.01).max(1).min(0)

    const particleVisualFolder = gui.addFolder('视觉属性');
    particleVisualFolder.add(params, 'minSize').name('最小粒子尺寸');
    particleVisualFolder.add(params, 'maxSize').name('最大粒子尺寸');
    particleVisualFolder.addColor(params, 'color').name('粒子颜色').onChange(value => {
        emitter.material.uniforms.color.value.set(value);
        splashSystem.material.uniforms.color.value.set(value); // 同步更新水花颜色
    });
    particleVisualFolder.add(params, 'burstProbability').name('突发概率');
    particleVisualFolder.add(params, 'burstMultiplier').name('突发倍数');
    particleVisualFolder.add(params, 'blendingMode', ['Additive', 'Normal']).name('混合模式').onChange(value => {
        emitter.material.blending = value === 'Additive' ? THREE.AdditiveBlending : THREE.NormalBlending;
        emitter.material.needsUpdate = true;
    });
    particleVisualFolder.add(params, 'rainLength', 0.1, 1.0).name('雨滴长度').onChange(value => {
        emitter.material.uniforms.rainLength.value = value;
    });
    particleVisualFolder.add(params, 'opacity', 0.1, 1.0).step(0.1).name('整体透明度').onChange(value => {
        emitter.material.uniforms.globalOpacity.value = value;
        splashSystem.material.uniforms.globalOpacity.value = value; // 同步更新水花透明度
    });

    const splashFolder = gui.addFolder('水花效果');
    splashFolder.add(params, 'enableSplash').name('启用水花效果');
    splashFolder.add(params, 'splashParticles', 1, 15).step(1).name('水花粒子数');
    splashFolder.add(params, 'splashSize', 0.1, 1.0).name('水花大小');
    splashFolder.add(params, 'splashSpeed', 1, 10).name('水花飞溅速度');
    splashFolder.add(params, 'splashLifeTime', 0.1, 2.0).name('水花生命时间');
    splashFolder.add(params, 'collisionY').name('碰撞高度'); // 修改名称
    splashFolder.open();

    particleFolder.open();
    particlePhysicsFolder.open();
    particleVisualFolder.open();

    class SplashParticle {
        constructor(position) {
            this.position = position.clone();
            const angle = Math.random() * Math.PI * 2;
            const speed = Math.random() * params.splashSpeed;
            this.velocity = new THREE.Vector3(
                Math.cos(angle) * speed,
                Math.random() * speed * 0.8 + speed * 0.5,
                Math.sin(angle) * speed
            );
            this.life = 0;
            this.maxLife = params.splashLifeTime * (0.7 + Math.random() * 0.6);
            this.size = params.splashSize * (0.5 + Math.random() * 0.5);
        }

        update(delta) {
            this.velocity.y -= params.gravity * delta * 0.8;
            this.position.addScaledVector(this.velocity, delta);
            this.life += delta;
            if (this.position.y < params.collisionY) { // 更新属性名
                this.position.y = params.collisionY;
                this.velocity.y = 0;
            }
            return this.life <= this.maxLife;
        }
    }

    class SplashSystem {
        constructor(maxCount = 1000) {
            this.maxCount = maxCount;
            this.particles = [];
            this.geometry = new THREE.BufferGeometry();
            this.positions = new Float32Array(maxCount * 3);
            this.sizes = new Float32Array(maxCount);
            this.opacities = new Float32Array(maxCount);
            this.geometry.setAttribute('position', new THREE.BufferAttribute(this.positions, 3));
            this.geometry.setAttribute('size', new THREE.BufferAttribute(this.sizes, 1));
            this.geometry.setAttribute('opacity', new THREE.BufferAttribute(this.opacities, 1));
            this.geometry.setDrawRange(0, 0);
            this.material = new THREE.ShaderMaterial({
                uniforms: {
                    color: { value: new THREE.Color(params.color) },
                    globalOpacity: { value: params.opacity }
                },
                vertexShader: `
                attribute float size;
                attribute float opacity;
                varying float vOpacity;
                void main() {
                    vOpacity = opacity;
                    vec4 mvPosition = modelViewMatrix * vec4(position, 1.0);
                
                    // 改进粒子大小计算，防止在近距离时粒子消失
                    float distance = max(length(mvPosition.xyz), 0.1); // 防止除以接近0的值
                    float scale = 300.0 / distance;
                    
                    // 限制最大缩放比例，防止过近时粒子过大
                    scale = min(scale, 50.0);
                    
                    gl_PointSize = size * scale;
                    gl_Position = projectionMatrix * mvPosition;
                }
            `,
                fragmentShader: `
                uniform vec3 color;
                uniform float globalOpacity;
                varying float vOpacity;
                void main() {
                    vec2 center = vec2(0.5, 0.5);
                    float dist = distance(gl_PointCoord, center);
                    float alpha = smoothstep(0.5, 0.2, dist) * vOpacity * globalOpacity;
                    if (alpha < 0.01) discard;
                    gl_FragColor = vec4(color, alpha);
                }
            `,
                blending: THREE.AdditiveBlending,
                transparent: true
            });
            this.points = new THREE.Points(this.geometry, this.material);
            scene.add(this.points);
        }

        addSplash(position) {
            if (!params.enableSplash) return;
            for (let i = 0; i < params.splashParticles; i++) {
                if (this.particles.length < this.maxCount) {
                    this.particles.push(new SplashParticle(position));
                }
            }
        }

        update(delta) {
            let alive = 0;
            let aliveParticles = [];
            for (let i = 0; i < this.particles.length; i++) {
                const p = this.particles[i];
                if (p.update(delta)) {
                    this.positions[alive * 3] = p.position.x;
                    this.positions[alive * 3 + 1] = p.position.y;
                    this.positions[alive * 3 + 2] = p.position.z;
                    this.sizes[alive] = p.size;
                    this.opacities[alive] = 1.0 - (p.life / p.maxLife);
                    alive++;
                    aliveParticles.push(p);
                }
            }
            this.particles = aliveParticles;
            this.geometry.setDrawRange(0, alive);
            this.geometry.attributes.position.needsUpdate = true;
            this.geometry.attributes.size.needsUpdate = true;
            this.geometry.attributes.opacity.needsUpdate = true;
        }
    }

    const splashSystem = new SplashSystem();
    splashSystem.points.frustumCulled = false // 禁用视锥体剔除，以确保水花在相机外也能渲染

    class Particle {
        constructor() {
            this.position = new THREE.Vector3(0, 5, 0);
            const angle = Math.random() * Math.PI * 2;
            const strength = Math.random() * (params.maxStrength - params.minStrength) + params.minStrength;
            const spread = Math.random() * params.spread;
            this.velocity = new THREE.Vector3(
                Math.cos(angle) * spread * strength,
                -Math.random() * strength - 2,
                Math.sin(angle) * spread * strength
            );
            this.life = 0;
            this.maxLife = 1 + Math.random() * 0.5;
            this.size = this.initialSize = Math.random() * (params.maxSize - params.minSize) + params.minSize;
        }
        update(delta) {
            this.velocity.y -= params.gravity * delta * 0.5;
            this.position.addScaledVector(this.velocity, delta);
            this.life += delta;
            const lifeRatio = this.life / this.maxLife;
            this.size = this.initialSize * (1 - lifeRatio * 0.5);
            if (this.position.y <= params.collisionY && this.velocity.y < 0) { // 更新属性名
                splashSystem.addSplash(new THREE.Vector3(this.position.x, params.collisionY, this.position.z)); // 更新属性名
                this.life = this.maxLife + 1;
            }
            return this.life <= this.maxLife;
        }
    }

    class ParticleSystem {
        constructor(maxCount = params.maxParticles) {
            this.maxCount = maxCount;
            this.particles = [];
            this.geometry = new THREE.BufferGeometry();
            this.positions = new Float32Array(this.maxCount * 3);
            this.sizes = new Float32Array(this.maxCount);
            this.opacities = new Float32Array(this.maxCount);
            this.geometry.setAttribute('position', new THREE.BufferAttribute(this.positions, 3));
            this.geometry.setAttribute('size', new THREE.BufferAttribute(this.sizes, 1));
            this.geometry.setAttribute('opacity', new THREE.BufferAttribute(this.opacities, 1));
            this.geometry.setDrawRange(0, 0);
            this.material = new THREE.ShaderMaterial({
                uniforms: {
                    color: { value: new THREE.Color(params.color) },
                    rainLength: { value: params.rainLength },
                    globalOpacity: { value: params.opacity }
                },
                vertexShader: `
                attribute float size;
                attribute float opacity;
                varying float vOpacity;
                void main() {
                    vOpacity = opacity;
                    vec4 mvPosition = modelViewMatrix * vec4(position, 1.0);
                    
                    // 改进粒子大小计算，防止在近距离时粒子消失
                    float distance = max(length(mvPosition.xyz), 0.1); // 防止除以接近0的值
                    float scale = 300.0 / distance;
                    
                    // 限制最大缩放比例，防止过近时粒子过大
                    scale = min(scale, 50.0);
                    
                    gl_PointSize = size * scale;
                    gl_Position = projectionMatrix * mvPosition;
                }
            `,
                fragmentShader: `
                uniform vec3 color;
                uniform float rainLength;
                uniform float globalOpacity;
                varying float vOpacity;
                void main() {
                    vec2 center = vec2(0.5, 0.5);
                    vec2 uv = gl_PointCoord - center;
                    float width = 0.05;
                    float y_offset = (gl_PointCoord.y - 0.5) * 2.0;
                    float shape = step(abs(uv.x), width * (1.0 - pow(y_offset, 2.0)));
                    shape *= step(gl_PointCoord.y, 0.5 + rainLength / 2.0) * step(0.5 - rainLength / 2.0, gl_PointCoord.y);
                    shape *= 1.0 - (0.5 - gl_PointCoord.y) * 0.5;
                    float alpha = shape * vOpacity * globalOpacity;
                    if (alpha < 0.01) discard;
                    gl_FragColor = vec4(color, alpha);
                }
            `,
                blending: params.blendingMode === 'Additive' ? THREE.AdditiveBlending : THREE.NormalBlending,
                depthTest: false,
                side: THREE.DoubleSide,
                transparent: true
            });
            this.points = new THREE.Points(this.geometry, this.material);
        }
        spawn(count = params.spawnRate) {
            const burst = Math.random() > params.burstProbability ? params.burstMultiplier : 1;
            for (let i = 0, n = count * burst; i < n && this.particles.length < this.maxCount; i++) {
                this.particles.push(new Particle());
            }
        }
        update(delta) {
            let alive = 0;
            let aliveParticles = [];
            for (let i = 0; i < this.particles.length; i++) {
                const p = this.particles[i];
                if (p.update(delta)) {
                    this.positions[alive * 3] = p.position.x;
                    this.positions[alive * 3 + 1] = p.position.y;
                    this.positions[alive * 3 + 2] = p.position.z;
                    this.sizes[alive] = p.size;
                    this.opacities[alive] = 1.0 - (p.life / p.maxLife);
                    alive++;
                    aliveParticles.push(p);
                }
            }
            this.particles = aliveParticles;
            this.geometry.setDrawRange(0, alive);
            this.geometry.attributes.position.needsUpdate = true;
            this.geometry.attributes.size.needsUpdate = true;
            this.geometry.attributes.opacity.needsUpdate = true;
            this.spawn(params.spawnRate);
        }
    }

    let emitter = new ParticleSystem(params.maxParticles)
    emitter.points.frustumCulled = false // 禁用视锥体剔除，以确保粒子在相机外也能渲染
    // scene.add(emitter.points)
    const mesh = emitter.points
    const clock = new THREE.Clock()
    mesh.render = () => {
        const delta = clock.getDelta();

        // 确保水花系统始终使用最新的颜色和透明度值
        splashSystem.material.uniforms.color.value.copy(emitter.material.uniforms.color.value);
        splashSystem.material.uniforms.globalOpacity.value = emitter.material.uniforms.globalOpacity.value;
    
        emitter.update(delta);
        splashSystem.update(delta);
    }

    return mesh

}
```

完整源码：[GitHub](https://github.com/z2586300277/three-cesium-examples/blob/dev/threeExamples/particle/waterLeakage.js)

## 小结

- 本文提供 **水流粒子** 完整 Three.js 源码与在线 Demo，建议先运行案例再改 uniform/参数做二次实验
- 更多 Three.js 实战案例见 [three-cesium-examples 合集](https://threehub.cn) 与 [GitHub 开源仓库](https://github.com/z2586300277/three-cesium-examples)

