---
title: "Three.js 道路流光教程"
description: "详解 Three.js 道路流光：原场景渲染后经 EffectComposer 叠加 Bloom/模糊等全屏后期，涵盖 ShaderMaterial、EffectComposer、OrbitControls 等关键实现，附完整源码与在线 Demo，适合 Three.js 应用场景 学习与二次开发。"
head:
  - - meta
    - name: keywords
      content: "Three.js,道路流光,WebGL,源码,教程,在线案例,ShaderMaterial,自定义着色器,GLSL,EffectComposer,后期处理,OrbitControls"
outline: deep
---

### 道路流光 · *Road Shader* · [▶ 在线运行案例](https://z2586300277.github.io/three-cesium-examples/#/codeMirror?navigation=ThreeJS&classify=application&id=roadShader)
- **案例合集：** [三维可视化功能案例（threehub.cn）](https://threehub.cn)

- **开源仓库github地址：** https://github.com/z2586300277/three-cesium-examples

- **400个案例代码: ** [网盘链接](https://pan.quark.cn/s/201da5c82fec)

![道路流光](https://z2586300277.github.io/three-cesium-examples/threeExamples/application/roadShader.jpg)

## 你将学到什么

- ShaderMaterial 自定义着色器实现核心视觉效果
- EffectComposer 多 Pass 后期处理管线
- OrbitControls 相机轨道交互
- `requestAnimationFrame` 渲染循环与 `resize` 自适应

## 效果说明

本案例演示 **道路流光** 效果：原场景渲染后经 EffectComposer 叠加 Bloom/模糊等全屏后期；核心用到 ShaderMaterial、EffectComposer、OrbitControls。建议先打开文首在线案例查看动态画面，再对照下方源码逐步理解。

## 核心概念
- **Scene / Camera / WebGLRenderer** 构成最小渲染闭环；大场景可开 `logarithmicDepthBuffer` 缓解 Z-fighting。
- **ShaderMaterial** 通过 `uniforms` + 自定义 GLSL 控制逐像素/逐点效果；透明粒子常配合 `depthTest: false`。
- **EffectComposer** 以多 Pass 链式渲染：RenderPass → 特效 Pass → 输出屏幕，替代直接 `renderer.render`。
- **OrbitControls** 提供轨道旋转/缩放；开启 `enableDamping` 后需在 animate 中 `controls.update()`。

## 实现步骤

1. 搭建 Scene、PerspectiveCamera、WebGLRenderer，挂载 canvas 并处理 `resize`
2. 定义 uniforms / onBeforeCompile 或 ShaderMaterial，编写 GLSL 与材质参数
3. 组装 EffectComposer Pass 链，在 animate 中调用 `composer.render()`
4. 创建 OrbitControls（及 Raycaster 等交互控件，若源码包含）
5. 在 `requestAnimationFrame` 循环中更新状态并 render（Cesium 为 `viewer.render` 或自动渲染）

## 代码要点

```js
import * as THREE from 'three';
import { EffectComposer } from 'three/addons/postprocessing/EffectComposer.js';
import { RenderPass } from 'three/addons/postprocessing/RenderPass.js';
import { BloomPass } from 'three/addons/postprocessing/BloomPass.js';
import { OutputPass } from 'three/addons/postprocessing/OutputPass.js';
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls.js';

class Base {
    initThree(el) {
        this.container = el;
        this.renderer = new THREE.WebGLRenderer({ antialias: true, alpha: true });
        this.renderer.setSize(this.container.offsetWidth, this.container.offsetHeight);
        this.container.appendChild(this.renderer.domElement);
        this.scene = new THREE.Scene();
        this.camera = new THREE.PerspectiveCamera(
            45,
            this.container.offsetWidth / this.container.offsetHeight,
            1,
            2000
        );
        this.camera.position.set(0, 10, 50);
        new OrbitControls(this.camera, this.renderer.domElement);
        this.animate();
        window.addEventListener('resize', this.onResize.bind(this));
    }
    animate() {
        this.renderer.render(this.scene, this.camera);
        requestAnimationFrame(this.animate.bind(this));
    }
    onResize() {
        if (this.container) {
            this.camera.aspect = this.container.offsetWidth / this.container.offsetHeight;
            this.camera.updateProjectionMatrix();
            this.renderer.setSize(this.container.offsetWidth, this.container.offsetHeight);
        }
    }
}

class Road extends Base {
    constructor() {
        super();
        this.speed = 0.005;
    }
    animate() {
        if (this.materials) {
            this.materials.forEach((m) => {
                m.uniforms.uTime.value += this.speed;
                if (m.uniforms.uTime.value > 1) {
                    m.uniforms.uTime.value = 0;
                }
            })
        }
        if (this.composer) {
            this.renderer.autoClear = false;
            this.renderer.clear();
            this.normalObj.visible = false;
            this.composer.render();
            this.renderer.clearDepth();
            this.normalObj.visible = true;
        }
        this.renderer.render(this.scene, this.camera);
        this.threeAnim = requestAnimationFrame(this.animate.bind(this));
    }
    initBloom() {
        const params = {
            threshold: 0,
            strength: 0.5,
            radius: 0.5,
            exposure: 0.5
        };
        const renderScene = new RenderPass(this.scene, this.camera);
        const bloomPass = new BloomPass(5, 20, 100);
        bloomPass.threshold = params.threshold;
        bloomPass.strength = params.strength;
        bloomPass.radius = params.radius;
        const composer = new EffectComposer(this.renderer);
        composer.addPass(renderScene);
        composer.addPass(bloomPass);
        composer.addPass(new OutputPass());
        this.composer = composer;
    }
    createChart(that) {
        this.initBloom();
        const commonUniforms = {
            uFade: { value: new THREE.Vector2(0, 0.6) },
            uOffset: { value: new THREE.Vector2(40, 20) }
        };
        const vertexMoveHeight = `
          float getMove(float u, float offset) {
            float a = u * PI * 2.0;
            return sin(a + PI * 0.25) * u * offset;
          }
          float getHeight(float u, float offset) {
            float a = u * PI * 3.0;
            return cos(a) * u * offset;
          }
        `;
        const spline = new THREE.LineCurve3(
            new THREE.Vector3(0, 0, that.height * 0.25),
            new THREE.Vector3(0, 0, -that.height * 0.75)
        );
        const geometry = new THREE.TubeGeometry(spline, that.height, that.lineWidth, 8, false);

        const vertexShader = `
      float PI = acos(-1.0);
      uniform vec2 uOffset;
      varying vec2 vUv;
      ${vertexMoveHeight}
      void main(void) {
        vUv = uv;
        float m = getMove(uv.x, uOffset.x);
        float h = getHeight(uv.x, uOffset.y);
        vec3 newPosition = position;
        newPosition.x += m;
        newPosition.y += h;
        gl_Position = projectionMatrix * modelViewMatrix * vec4(newPosition, 1.0);
      }
    `;

        const fragmentShader = `
      varying vec2 vUv;
      uniform float uSpeed;
      uniform float uTime;
      uniform vec2 uFade;
      uniform vec3 uColor;
      uniform float uDirection;
      void main() {
        vec3 color = uColor;
        float s = -uTime * uSpeed;
        float v = (uDirection == 1.0) ? vUv.x : -vUv.x;
        float d = mod(v + s, 1.0);
        if (d > uFade.y) discard;
        else {
          float alpha = smoothstep(uFade.x, uFade.y, d);
          if (alpha < 0.0001) discard;
          gl_FragColor = vec4(color, alpha);
        }
      }
    `;

        const materials = [];
        const amount = that.amount;
        const step = (that.width - that.gap) / amount;

        for (let i = 0; i < amount; i++) {
            const color = new THREE.Color();
            const v = i / amount;
            color.setHSL(
                THREE.MathUtils.lerp(that.hueStart, that.hueEnd, v),
                1,
                THREE.MathUtils.lerp(that.lightStart, that.lightEnd, v)
            );

            const material = new THREE.ShaderMaterial({
                side: THREE.DoubleSide,
                transparent: true,
                uniforms: {
                    uColor: { value: color },
                    uTime: { value: THREE.MathUtils.lerp(-1, 1, Math.random()) },
                    uDirection: { value: i < amount * 0.5 ? 1 : 0 },
                    uSpeed: { value: THREE.MathUtils.lerp(1, 1.5, Math.random()) },
                    ...commonUniforms
                },
                vertexShader,
                fragmentShader
            });

            materials.push(material);

            const mesh = new THREE.Mesh(geometry, material);
            mesh.position.x = i * step + (i > amount * 0.5 - 1 ? that.gap : 0);
            mesh.position.y = Math.random() * 5;
            this.scene.add(mesh);
        }

        this.materials = materials;

        const planeGeometry = new THREE.PlaneGeometry(
            that.width,
            that.height,
            that.width * 0.25,
            that.height * 0.25
        );

        const planeMaterial = new THREE.ShaderMaterial({
            side: THREE.DoubleSide,
            transparent: true,
            uniforms: {
                uColor: { value: new THREE.Color('blue') },
                ...commonUniforms
            },
            vertexShader: `
        float PI = acos(-1.0);
        uniform vec2 uOffset;
        ${vertexMoveHeight}
        void main(void) {
          float m = getMove(uv.y, uOffset.x);
          float h = getHeight(uv.y, uOffset.y);
          vec3 newPosition = position;
          newPosition.x += m;
          newPosition.z += h;
          gl_Position = projectionMatrix * modelViewMatrix * vec4(newPosition, 1.0);
        }
      `,
            fragmentShader: `
        uniform vec3 uColor;
        void main() {
          gl_FragColor = vec4(uColor, 0.6);
        }
      `
        });

        this.planeMat = planeMaterial;

        const plane = new THREE.Mesh(planeGeometry, planeMaterial);
        plane.rotateX(-Math.PI * 0.5);
        plane.position.set(that.width * 0.5, -1, -that.height * 0.25);
        this.normalObj = plane;
        this.scene.add(plane);
    }
}

var road = new Road();
road.initThree(document.getElementById('box'));
road.createChart({
    lineWidth: 0.5,
    width: 48,
    height: 400,
    gap: 8,
    amount: 20,
    hueStart: 0.9,
    hueEnd: 0.1,
    lightStart: 0.5,
    lightEnd: 1.0
});
```

完整源码：[GitHub](https://github.com/z2586300277/three-cesium-examples/blob/dev/threeExamples/application/roadShader.js)

## 小结

- 本文提供 **道路流光** 完整 Three.js 源码与在线 Demo，建议先运行案例再改 uniform/参数做二次实验
- 更多 Three.js 实战案例见 [three-cesium-examples 合集](https://threehub.cn) 与 [GitHub 开源仓库](https://github.com/z2586300277/three-cesium-examples)

