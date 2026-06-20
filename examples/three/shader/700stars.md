---
title: "Three.js 1000stars留念教程"
description: "详解 Three.js 1000stars留念：基于 WebGL 实现「1000stars留念」可视化效果，附完整可运行源码，涵盖 onBeforeCompile、OrbitControls、GSAP 等关键实现，附完整源码与在线 Demo，适合 Three.js 着色器 学习与二次开发。"
head:
  - - meta
    - name: keywords
      content: "Three.js,1000stars留念,WebGL,源码,教程,在线案例,onBeforeCompile,shader注入,OrbitControls,相机控制,GSAP,动画"
outline: deep
---

### 1000stars留念 · *1000stars* · [▶ 在线运行案例](https://z2586300277.github.io/three-cesium-examples/#/codeMirror?navigation=ThreeJS&classify=shader&id=700stars)
- **案例合集：** [三维可视化功能案例（threehub.cn）](https://threehub.cn)

- **开源仓库github地址：** https://github.com/z2586300277/three-cesium-examples

- **400个案例代码: ** [网盘链接](https://pan.quark.cn/s/201da5c82fec)

![1000stars留念](https://z2586300277.github.io/three-cesium-examples/threeExamples/application/700stars.jpg)

## 你将学到什么

- onBeforeCompile 注入 GLSL 改造内置材质
- OrbitControls 相机轨道交互
- GSAP 时间轴与补间动画
- 场景雾效增强纵深
- `requestAnimationFrame` 渲染循环与 `resize` 自适应

## 效果说明

本案例演示 **1000stars留念** 效果：基于 WebGL 实现「1000stars留念」可视化效果，附完整可运行源码；核心用到 onBeforeCompile、OrbitControls、GSAP。建议先打开文首在线案例查看动态画面，再对照下方源码逐步理解。

## 核心概念

- **Scene / Camera / WebGLRenderer** 构成最小渲染闭环；大场景可开 `logarithmicDepthBuffer` 缓解 Z-fighting。
- **onBeforeCompile** 在 Three 拼好内置 shader 后替换 `#include <xxx>` 片段，适合在 PBR 材质上叠加大屏特效。
- **OrbitControls** 提供轨道旋转/缩放；开启 `enableDamping` 后需在 animate 中 `controls.update()`。

## 实现步骤

1. 搭建灯光与环境（如有）
2. requestAnimationFrame 循环 update + render

## 代码要点

```js
import * as THREE from "three";
import { ImprovedNoise } from 'three/examples/jsm/Addons.js';
// 700stars留念 共筑 共享
import {  DirectionalLight, AmbientLight, Mesh, PlaneGeometry, MeshLambertMaterial, Vector2,Color } from 'three';
let fbm = `
    // https://github.com/yiwenl/glsl-fbm/blob/master/3d.glsl
    #define NUM_OCTAVES 6

    float mod289(float x){return x - floor(x * (1.0 / 289.0)) * 289.0;}
    vec4 mod289(vec4 x){return x - floor(x * (1.0 / 289.0)) * 289.0;}
    vec4 perm(vec4 x){return mod289(((x * 34.0) + 1.0) * x);}

    float noise(vec3 p){
        vec3 a = floor(p);
        vec3 d = p - a;
        d = d * d * (3.0 - 2.0 * d);

        vec4 b = a.xxyy + vec4(0.0, 1.0, 0.0, 1.0);
        vec4 k1 = perm(b.xyxy);
        vec4 k2 = perm(k1.xyxy + b.zzww);

        vec4 c = k2 + a.zzzz;
        vec4 k3 = perm(c);
        vec4 k4 = perm(c + 1.0);

        vec4 o1 = fract(k3 * (1.0 / 41.0));
        vec4 o2 = fract(k4 * (1.0 / 41.0));

        vec4 o3 = o2 * d.z + o1 * (1.0 - d.z);
        vec2 o4 = o3.yw * d.x + o3.xz * (1.0 - d.x);

        return o4.y * d.y + o4.x * (1.0 - d.y);
    }


    float fbm(vec3 x) {
      float v = 0.0;
      float a = 0.5;
      vec3 shift = vec3(100);
      for (int i = 0; i < NUM_OCTAVES; ++i) {
        v += a * noise(x);
        x = x * 2.0 + shift;
        a *= 0.5;
      }
      return v;
    }
  `;
let renderer,scene,camera
let init_scene = async () => {
    const box = document.getElementById("box");
    console.log(box);
    

 scene = new THREE.Scene();

 camera = new THREE.PerspectiveCamera(
    60,
  box.clientWidth / box.clientHeight,
  0.1,
  1000
);
let vHeight = 3;
camera.position.set(30, vHeight + 2, 20).setLength(15);
scene.background = new Color(0.5, 1, 0.875);
// scene.fog = new Fog(scene.background, 20, 45);
// camera.position.set(20, 50, 0);
 renderer = new THREE.WebGLRenderer({
  antialias: true,
  alpha: true,
  logarithmicDepthBuffer: true,
});

renderer.setSize(box.clientWidth, box.clientHeight);
box.appendChild(renderer.domElement);
// new OrbitControls(camera, renderer.domElement);

window.onresize = () => {
  renderer.setSize(box.clientWidth, box.clientHeight);
  camera.aspect = box.clientWidth / box.clientHeight;
  camera.updateProjectionMatrix();
};
    let light = new DirectionalLight(0xffffff, 0.25);
    light.position.setScalar(1);
    scene.add(light, new AmbientLight(0xffffff, 0.75));
};

let terrain;
let plane, material;
const load_terrain = () => {
    let perlin = new ImprovedNoise();
    plane = new PlaneGeometry(50, 50, 500, 500);
    plane.rotateX(-Math.PI / 2);
    let { position } = plane.attributes;
    let uv = plane.attributes.uv;
    let v2 = new Vector2();
    for (let i = 0; i < position.count; i++) {
        v2.fromBufferAttribute(uv, i).multiplyScalar(15);
        let n = perlin.noise(v2.x, v2.y, 0.314);
        n = Math.abs(n);
        n = Math.pow(n, 3);
        position.setY(i, Math.min(n * 35, 10));
    }
    plane.computeVertexNormals();
    material = new MeshLambertMaterial({
        color: 0xface8d,
        // wireframe:true
    });
    material.onBeforeCompile = (shader) => {
        // shader.uniforms.time = globalUniforms.time;
        shader.vertexShader = `
        varying vec3 vPos;
        ${shader.vertexShader}
        `.replace(`#include <begin_vertex>`, `#include <begin_vertex>
            vPos = position;`);
        shader.fragmentShader = `
            #define ss(a,b,c) smoothstep(a,b,c)
            uniform float time;
            varying vec3 vPos;
            ${fbm}
            ${shader.fragmentShader}
        `
            .replace(`vec4 diffuseColor = vec4( diffuse, opacity );`, `
                vec3 col = diffuse;

                float d = noise(vPos * vec3(0.05, 1, 0.05));
                col = mix(col + 0.2, vec3(1, 0.2, 0.01), d);

                vec3 strokePos = vPos * vec3(0.1, 3., 0.1);
                d = fbm(strokePos);
                float e = fwidth(strokePos.y);
                col = mix(col * (0.5 + 0.5 * ss(2., 8., vPos.y)), col, ss(0.4 - e, 0.4, abs(d)));

                col = mix(diffuse + 0.1, col, ss(0.5, 1.5, vPos.y));

                // wind
                float dw = noise(vec3(vPos.x, vPos.y, vPos.z + time) * vec3(0.1, 10, 0.1));
                d = ss(0.1, 0., abs(dw));
                d = max(d, ss(1., 0., abs(dw)));
                d = max(d, pow(abs(noise(vPos - vec3(0, 0, time))), 1.));
                d *= smoothstep(2., -0.5, abs(vPos.y));
                col = mix(col, diffuse + 0.25, d);

                vec4 diffuseColor = vec4( col, opacity );
                `)
            .replace(`#include <dithering_fragment>`, `gl_FragColor.rgb = mix(gl_FragColor.rgb, vec3(0.5, 1, 0.875), pow(ss(7., 10., vPos.y), 0.5));`);
    };
    terrain = new Mesh(plane, material);
    scene.add(terrain);
};
// 定时生成新地形并平滑过渡
const generate_new_terrain_1 = () => {
    let perlin = new ImprovedNoise();
    let new_position = plane.attributes.position.clone();
    let uv = plane.attributes.uv;
    let v2 = new Vector2();
    let random = Math.random();
    for (let i = 0; i < new_position.count; i++) {
        v2.fromBufferAttribute(uv, i).multiplyScalar(15);
        let n = perlin.noise(v2.x, v2.y, random);
        n = Math.abs(n);
        n = Math.pow(n, 3);
        new_position.setY(i, Math.min(n * 35, 10));
    }
    new_position.needsUpdate = true;
    // const old_position_array = plane.attributes.position.array
    const new_position_array = new_position.array;
    return new_position_array;
};
// 地形插值
import { gsap } from 'gsap';
const new_fun = () => {
    const position = plane.attributes.position;
    const position_array = position.array;
    const new_position_array = generate_new_terrain_1;
    gsap.to(position_array, {
        duration: 1.5, // 动画时长
        ease: "power2.out", // 缓动效果
        endArray: new_position_array, // 动画目标值
        onUpdate: () => {
            position.needsUpdate = true; // 通知 Three.js 属性已更新
        },
    });
};
const render = () => {
    renderer.render(scene, camera);
    requestAnimationFrame(render);
};
init_scene();
load_terrain();
setInterval(() => {
    new_fun();
}, 2000);
render();

// 文字
const ele = () => {
    const box = document.getElementById("box");
    box.style.position = 'relative';
    const div = document.createElement('div');
    div.style.position = 'absolute';
    div.style.top = '0';
    div.style.left = '0';
    div.style.width = '100%';
    div.style.height = '100%';
    div.style.display = 'flex';
    div.style.justifyContent = 'center';
    div.style.alignItems = 'center';
    div.innerHTML = `
        <div style="text-align: center; 
                    width: 90%; 
                    max-width: 800px;
                    background: rgba(255, 255, 255, 0.1);
                    backdrop-filter: blur(10px);
                    padding: 2rem;
                    border-radius: 20px;
                    box-shadow: 0 8px 32px rgba(0, 0, 0, 0.1);">
            <div style="font-size: min(5vw, 45px); 
                        margin-bottom: 1.5rem;
                        white-space: nowrap;
                        color: #2d3436;
                        font-weight: bold;
                        text-shadow: 2px 2px 4px rgba(0,0,0,0.2);
                        transition: transform 0.3s;
                        cursor: pointer;
                        font-family: 'Microsoft YaHei', sans-serif;"
                        onmouseover="this.style.transform='scale(1.05)'"
                        onmouseout="this.style.transform='scale(1)'">
                共筑3D世界,共享3D世界
            </div>
            <div style="font-size: min(5vw, 45px);
                        margin-bottom: 1.5rem;
                        white-space: nowrap;
                        color: #2d3436;
                        font-weight: bold;
                        text-shadow: 2px 2px 4px rgba(0,0,0,0.2);
                        transition: transform 0.3s;
                        cursor: pointer;
                        font-family: 'Arial', sans-serif;"
                        onmouseover="this.style.transform='scale(1.05)'"
                        onmouseout="this.style.transform='scale(1)'">
                Build & Share 3D World Together
            </div>
            <div style="font-size: min(2vw, 16px);
                        text-align: right;
                        margin-top: 2rem;
                        color:rgb(173, 233, 255);
                        font-style: italic;
                        letter-spacing: 1px;">
                for three-cesium-examples 1000 stars ⭐
            </div>
        </div>
    `;
    box.appendChild(div);
};

ele()
```

完整源码：[GitHub](https://github.com/z2586300277/three-cesium-examples/blob/dev/threeExamples/application/700stars.js)

## 小结

- 本文提供 **1000stars留念** 完整 Three.js 源码与在线 Demo，建议先运行案例再改 uniform/参数做二次实验
- 更多 Three.js 实战案例见 [three-cesium-examples 合集](https://threehub.cn) 与 [GitHub 开源仓库](https://github.com/z2586300277/three-cesium-examples)

