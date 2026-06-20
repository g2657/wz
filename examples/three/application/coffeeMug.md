---
title: "Three.js 咖啡教程"
description: "详解 Three.js 咖啡：基于 WebGL 实现「咖啡」可视化效果，附完整可运行源码，涵盖 ShaderMaterial、OrbitControls、glTF/Draco 等关键实现，附完整源码与在线 Demo，适合 Three.js 应用场景 学习与二次开发。"
head:
  - - meta
    - name: keywords
      content: "Three.js,咖啡,WebGL,源码,教程,在线案例,ShaderMaterial,自定义着色器,GLSL,OrbitControls,相机控制,glTF"
outline: deep
---

### 咖啡 · *Coffee Mug* · [▶ 在线运行案例](https://z2586300277.github.io/three-cesium-examples/#/codeMirror?navigation=ThreeJS&classify=application&id=coffeeMug)
- **案例合集：** [三维可视化功能案例（threehub.cn）](https://threehub.cn)

- **开源仓库github地址：** https://github.com/z2586300277/three-cesium-examples

- **400个案例代码: ** [网盘链接](https://pan.quark.cn/s/201da5c82fec)

![咖啡](https://z2586300277.github.io/three-cesium-examples/threeExamples/application/coffeeMug.jpg)

## 你将学到什么

- ShaderMaterial 自定义着色器实现核心视觉效果
- OrbitControls 相机轨道交互
- glTF/Draco 模型加载与优化
- `requestAnimationFrame` 渲染循环与 `resize` 自适应

## 效果说明

本案例演示 **咖啡** 效果：基于 WebGL 实现「咖啡」可视化效果，附完整可运行源码；核心用到 ShaderMaterial、OrbitControls、glTF/Draco。建议先打开文首在线案例查看动态画面，再对照下方源码逐步理解。

## 核心概念
- **Scene / Camera / WebGLRenderer** 构成最小渲染闭环；大场景可开 `logarithmicDepthBuffer` 缓解 Z-fighting。
- **ShaderMaterial** 通过 `uniforms` + 自定义 GLSL 控制逐像素/逐点效果；透明粒子常配合 `depthTest: false`。
- **OrbitControls** 提供轨道旋转/缩放；开启 `enableDamping` 后需在 animate 中 `controls.update()`。

## 实现步骤

1. 搭建 Scene、PerspectiveCamera、WebGLRenderer，挂载 canvas 并处理 `resize`
2. 异步加载模型 / 3D Tiles / GeoJSON 等资源并加入 scene 或 entities
3. 定义 uniforms / onBeforeCompile 或 ShaderMaterial，编写 GLSL 与材质参数
4. 创建 OrbitControls（及 Raycaster 等交互控件，若源码包含）
5. 在 `requestAnimationFrame` 循环中更新状态并 render（Cesium 为 `viewer.render` 或自动渲染）

## 代码要点

```js
import * as THREE from 'three'
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls.js'
import Stats from 'three/examples/jsm/libs/stats.module.js';
import { GLTFLoader } from 'three/examples/jsm/loaders/GLTFLoader.js';
import {GUI} from 'dat.gui'

const initializeScene = ({ root, antialias = true } = {}) => {
    // Create scene
    const scene = new THREE.Scene();

    // Create camera
    const camera = new THREE.PerspectiveCamera(
        35,
        window.innerWidth / window.innerHeight,
        0.1,
        1000,
    );
    camera.position.z = 110;

    // Create renderer
    const renderer = new THREE.WebGLRenderer({ antialias });
    renderer.setSize(window.innerWidth, window.innerHeight);
    // renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));
    const controls = new OrbitControls(camera, renderer.domElement);
    controls.enableDamping = true;
    root.appendChild(renderer.domElement);

    const onWindowResize = () => {
        // Adjust camera and renderer on window resize
        camera.aspect = window.innerWidth / window.innerHeight;
        camera.updateProjectionMatrix();
        controls.update();
        renderer.setSize(window.innerWidth, window.innerHeight);
        renderer.render(scene, camera);
    };
    onWindowResize();
    window.addEventListener('resize', onWindowResize, false);

    // Create GUI
    const gui = new GUI({ container: root });

    const stats = new Stats();
    stats.showPanel(0);
    root.appendChild(stats.domElement);

    return {
        scene,
        renderer,
        camera,
        controls,
        gui,
        stats,
    };
};

const init = (root) => {
    const { scene, renderer, camera, gui, stats, controls } = initializeScene({
        root,
    });

    camera.position.set(12, 6, 12);

    const gltfLoader = new GLTFLoader();
    gltfLoader.load(FILE_HOST + 'examples/coffeeMug/coffeeMug.glb', (gltf) => {
        gltf.scene.getObjectByName('baked').material.map.anisotropy = 8;
        controls.target.y += 3;
        scene.add(gltf.scene);
    });

    const textureLoader = new THREE.TextureLoader();
    const perlinTexture = textureLoader.load(FILE_HOST + 'examples/coffeeMug/perlin.png');
    perlinTexture.wrapS = THREE.RepeatWrapping;
    perlinTexture.wrapT = THREE.RepeatWrapping;

    const smokeGeometry = new THREE.PlaneGeometry(1, 1, 16, 64);
    smokeGeometry.translate(0, 0.5, 0);
    smokeGeometry.scale(1.5, 6, 1.5);

    const smokeMaterial = new THREE.ShaderMaterial({
        // wireframe: true,
        vertexShader:`#define M_PI 3.1415926535897932384626433832795

        varying vec2 vUv;
        
        uniform float uTime;
        uniform sampler2D uPerlinTexture;
        
        vec2 rotate2D(vec2 value, float angle)
        {
        float s = sin(angle);
        float c = cos(angle);
        mat2 m = mat2(c, s, -s, c);
        return m * value;
        }

        
        void main()
        {
          vUv = uv;
        
          vec3 newPosition = position;
          float angle = texture(
            uPerlinTexture,
            vec2(0.5, uv.y * 0.3 + uTime * 0.02
          )).x * 7.;
          newPosition.xz = rotate2D(position.xz, angle);
        
          vec2 windOffset = vec2(
            texture(uPerlinTexture, vec2(0.2, uTime * 0.02)).x - 0.5,
            texture(uPerlinTexture, vec2(0.7, uTime * 0.02)).x - 0.5
          );
        
          newPosition.xz += windOffset * pow(uv.y, 2.) * 8.;
        
          gl_Position = projectionMatrix * viewMatrix * modelMatrix * vec4(newPosition, 1.0);
        }
        `,
        fragmentShader:`varying vec2 vUv;

        uniform float uTime;
        uniform sampler2D uPerlinTexture;
        
        void main()
        {
        
          vec2 uv = vec2(vUv.x * 0.5, vUv.y * 0.3 - uTime / 15.);
        
          float intensity = texture2D(uPerlinTexture, uv).x;
          intensity = smoothstep(0.4, 1.0, intensity);
          
          intensity *= smoothstep(0.0, 0.1, vUv.x);
          intensity *= smoothstep(1.0, 0.9, vUv.x);
        
          intensity *= smoothstep(0.0, 0.1, vUv.y);
          intensity *= smoothstep(1.0, 0.4, vUv.y);
        
        
          gl_FragColor = vec4(1.0, 0.8, 0.6, intensity);
          
          #include <tonemapping_fragment>
          #include <colorspace_fragment>
        }
        `,
        uniforms: {
            uTime: { value: 0 },
            uPerlinTexture: { value: perlinTexture },
        },
        transparent: true,
        depthWrite: false,
        side: THREE.DoubleSide,
    });

    const smoke = new THREE.Mesh(smokeGeometry, smokeMaterial);
    smoke.position.y = 1.83;
    scene.add(smoke);

    const clock = new THREE.Clock();

    const tick = () => {
        requestAnimationFrame(tick);
        stats.begin();

        controls.update();

        smokeMaterial.uniforms.uTime.value = clock.getElapsedTime();

        stats.end();
        renderer.render(scene, camera);
    };

    tick();

    return renderer;
};

init(document.getElementById('box'));
```

完整源码：[GitHub](https://github.com/z2586300277/three-cesium-examples/blob/dev/threeExamples/application/coffeeMug.js)

## 小结

- 本文提供 **咖啡** 完整 Three.js 源码与在线 Demo，建议先运行案例再改 uniform/参数做二次实验
- 更多 Three.js 实战案例见 [three-cesium-examples 合集](https://threehub.cn) 与 [GitHub 开源仓库](https://github.com/z2586300277/three-cesium-examples)

