---
title: "Three.js 着色器行星教程"
description: "详解 Three.js 着色器行星：基于 WebGL 实现「着色器行星」可视化效果，附完整可运行源码，涵盖 ShaderMaterial、OrbitControls 等关键实现，附完整源码与在线 Demo，适合 Three.js 着色器 学习与二次开发。"
head:
  - - meta
    - name: keywords
      content: "Three.js,着色器行星,WebGL,源码,教程,在线案例,ShaderMaterial,自定义着色器,GLSL,OrbitControls,相机控制"
outline: deep
---

### 着色器行星 · *Shader Planet* · [▶ 在线运行案例](https://z2586300277.github.io/three-cesium-examples/#/codeMirror?navigation=ThreeJS&classify=shader&id=shader_planet)
- **案例合集：** [三维可视化功能案例（threehub.cn）](https://threehub.cn)

- **开源仓库github地址：** https://github.com/z2586300277/three-cesium-examples

- **400个案例代码: ** [网盘链接](https://pan.quark.cn/s/201da5c82fec)

![着色器行星](https://z2586300277.github.io/three-cesium-examples/threeExamples/shader/shader_planet.jpg)

## 你将学到什么

- ShaderMaterial 自定义着色器实现核心视觉效果
- OrbitControls 相机轨道交互
- `requestAnimationFrame` 渲染循环与 `resize` 自适应

## 效果说明

本案例演示 **着色器行星** 效果：基于 WebGL 实现「着色器行星」可视化效果，附完整可运行源码；核心用到 ShaderMaterial、OrbitControls。建议先打开文首在线案例查看动态画面，再对照下方源码逐步理解。

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
import * as THREE from "three";
import { OrbitControls } from "three/examples/jsm/controls/OrbitControls.js";

const box = document.getElementById("box");
const scene = new THREE.Scene();
const texture = await new THREE.TextureLoader().load(FILE_HOST + 'images/channels/8k_stars_milky_way.jpg')
scene.background = texture;
const camera = new THREE.PerspectiveCamera(
    75,
    box.clientWidth / box.clientHeight,
    0.1,
    1000,
);
camera.position.set(0, 0, 20);
const renderer = new THREE.WebGLRenderer();
renderer.setSize(box.clientWidth, box.clientHeight);
box.appendChild(renderer.domElement);
new OrbitControls(camera, renderer.domElement);
window.onresize = () => {
    renderer.setSize(box.clientWidth, box.clientHeight);
    camera.aspect = box.clientWidth / box.clientHeight;
    camera.updateProjectionMatrix();
};


function animate() {
    // uniforms.iTime.value += 0.01
    mesh.rotation.y += 0.01;
    requestAnimationFrame(animate);
    renderer.render(scene, camera);
}
import init, { fbm } from "three_noise";
const generate_texture = async () => {
    await init();

    // let noise = new ImprovedNoise();

    let texture_height = 1024,
        texture_width = 1024;
    let texture_data = new Uint8Array(texture_height * texture_width * 4);
    for (let x = 0; x < texture_width; x++) {
        for (let y = 0; y < texture_height; y++) {
            // const noisevalue = noise.noise(x, y, 0.325);
            let fbm_value = fbm(
                x / texture_width,
                y / texture_height,
                6,
                2.0,
                1.5,
            );
            let color = fbm_value * 128 + 128;
            let i = (x + y * texture_width) * 4;
            texture_data[i] = color;
            texture_data[i + 1] = color;
            texture_data[i + 2] = fbm_value * 255;
            texture_data[i + 3] = 255;
        }
    }
    const texture = new THREE.DataTexture(
        texture_data,
        texture_width,
        texture_height,
        THREE.RGBAFormat,
    );
    texture.wrapS = THREE.RepeatWrapping;
    texture.wrapT = THREE.RepeatWrapping;
    texture.repeat.set(1, -1);
    texture.needsUpdate = true;
    return texture;
};
let material,mesh;
const add_sky_sphere = async () => {
    const texture = await generate_texture();
    const uniforms = {
        u_texture: {
            value: texture,
        },
    };
    const sphere_geo = new THREE.SphereGeometry(5, 100, 100);
    const vertexShader = `
              varying vec2 vUv;
              void main() {
                  vUv = uv;
                  gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0);
              }
          `;
    const fragmentShader = `
            varying vec2 vUv;
            uniform sampler2D u_texture;
            void main(){
                  vec2 uv = vUv;
                  vec4 color = texture2D(u_texture,uv);
                  gl_FragColor = color;
            }
          `;
    material = new THREE.ShaderMaterial({
        uniforms,
        vertexShader,
        fragmentShader,
        side: THREE.DoubleSide,
        // wireframe:true
    });
    mesh = new THREE.Mesh(sphere_geo, material);
    // mesh.translateX(-10);
    material.needsUpdate = true
    scene.add(mesh);
};

const change_material = ()=>{
    if (params.showTerrain) {
        material.vertexShader = `
        uniform sampler2D u_texture;
        varying vec2 vUv;
        void main() {
            vUv = uv;
            vec4 color = texture2D(u_texture, uv);
            // float height = length(color);
            float height = color.r;
            vec3 newPosition = position + normal * height * 1.5;
            gl_Position = projectionMatrix * modelViewMatrix * vec4(newPosition, 1.0);
        }
        `
    }else{
        material.vertexShader = `
        varying vec2 vUv;
        void main() {
            vUv = uv;
            gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0);
        }
        `
    }
    material.needsUpdate = true
}
import { GUI } from "dat.gui";
const params = {
    showTerrain:false
}
const add_gui = ()=>{
    const gui = new GUI()
    gui.add(params,'showTerrain').onChange(change_material)
}
await add_sky_sphere();
add_gui()
animate();
```

完整源码：[GitHub](https://github.com/z2586300277/three-cesium-examples/blob/dev/threeExamples/shader/shader_planet.js)

## 小结

- 本文提供 **着色器行星** 完整 Three.js 源码与在线 Demo，建议先运行案例再改 uniform/参数做二次实验
- 更多 Three.js 实战案例见 [three-cesium-examples 合集](https://threehub.cn) 与 [GitHub 开源仓库](https://github.com/z2586300277/three-cesium-examples)

