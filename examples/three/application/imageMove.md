---
title: "Three.js 图片移动教程"
description: "详解 Three.js 图片移动：基于 WebGL 实现「图片移动」可视化效果，附完整可运行源码，涵盖 onBeforeCompile 等关键实现，附完整源码与在线 Demo，适合 Three.js 应用场景 学习与二次开发。"
head:
  - - meta
    - name: keywords
      content: "Three.js,图片移动,WebGL,源码,教程,在线案例,onBeforeCompile,shader注入"
outline: deep
---

### 图片移动 · *Image Move* · [▶ 在线运行案例](https://z2586300277.github.io/three-cesium-examples/#/codeMirror?navigation=ThreeJS&classify=application&id=imageMove)
- **案例合集：** [三维可视化功能案例（threehub.cn）](https://threehub.cn)

- **开源仓库github地址：** https://github.com/z2586300277/three-cesium-examples

- **400个案例代码: ** [网盘链接](https://pan.quark.cn/s/201da5c82fec)

![图片移动](https://z2586300277.github.io/three-cesium-examples/threeExamples/application/imageMove.jpg)

## 你将学到什么

- onBeforeCompile 注入 GLSL 改造内置材质

## 效果说明

本案例演示 **图片移动** 效果：基于 WebGL 实现「图片移动」可视化效果，附完整可运行源码；核心用到 onBeforeCompile。建议先打开文首在线案例查看动态画面，再对照下方源码逐步理解。

## 核心概念

- **Scene** 容纳对象，**Camera** 定义视点，**WebGLRenderer** 输出 canvas。
- 替换 `#include <begin_vertex>` 等 chunk 注入特效，适合 PBR 材质叠加大屏效果。

## 实现步骤

1. 搭建 Scene、PerspectiveCamera、WebGLRenderer，挂载 canvas 并处理 `resize`
2. 定义 uniforms / onBeforeCompile 或 ShaderMaterial，编写 GLSL 与材质参数
3. 在 `requestAnimationFrame` 循环中更新状态并 render（Cesium 为 `viewer.render` 或自动渲染）

## 代码要点

```js
import * as THREE from 'three';

const [w, h] = [innerWidth, innerHeight]
const scene = new THREE.Scene()
const camera = new THREE.PerspectiveCamera(75, w / h, 0.1, 1000);
camera.position.z = 3;

const renderer = new THREE.WebGLRenderer({ antialias: true });
renderer.setSize(w, h);
document.body.appendChild(renderer.domElement);

const loader = new THREE.TextureLoader()
const urls = [
    FILE_HOST + 'images/wx_star.png',
    FILE_HOST + 'images/QQ.png',
    FILE_HOST + 'images/nico.jpg',
]
const tex = urls.map(u => loader.load(u));
tex[2].wrapS = tex[2].wrapT = THREE.RepeatWrapping; tex[2].repeat.set(.1, .1);

function R(g, t) {
    const m = new THREE.MeshMatcapMaterial({ matcap: t, transparent: true });
    m.onBeforeCompile = sh => {
        sh.vertexShader = sh.vertexShader
            .replace('#include <common>', `\n#include <common>\nvarying vec2 vUv;`)
            .replace('#include <fog_vertex>', `\n#include <fog_vertex>\nvUv=uv;`);
        sh.fragmentShader = sh.fragmentShader
            .replace('#include <common>', `\n#include <common>\nvarying vec2 vUv;\nfloat sdf(vec2 c,vec2 s,float r){return length(max(abs(c)-s+r,0.0))-r;}`)
            .replace('#include <dithering_fragment>', `\n#include <dithering_fragment>\nfloat d=sdf(vUv-vec2(.5),vec2(.5),.08);\nfloat a=1.-smoothstep(0.,.002,d);\ngl_FragColor=vec4(outgoingLight,a);`);
    };
    return new THREE.Mesh(g, m);
}

const scale = 1.25;
const g = new THREE.PlaneGeometry(1 * scale, 1.4 * scale);
const group = new THREE.Group();
[[-1, 0, 1, .1], [0, 0, .5, 0], [1, 0, 1, -.1]].forEach((p, i) => {
    const m = R(g, tex[i]);
    m.position.set(p[0] * scale, p[1] * scale, p[2] * scale);
    if (p[3]) m.rotation.y = Math.PI * p[3];
    group.add(m);
});
scene.add(group);

const mouse = new THREE.Vector2(), clock = new THREE.Clock();
addEventListener('mousemove', e => { mouse.x = (e.clientX / w) * 2 - 1; mouse.y = -(e.clientY / h) * 2 + 1 });

renderer.setAnimationLoop(() => {
    const d = clock.getDelta(), x = mouse.x * -0.3, y = mouse.y * .3;
    group.rotation.y += (x - group.rotation.y) * 3 * d;
    group.rotation.x += (y - group.rotation.x) * 3 * d;
    renderer.render(scene, camera);
});
```

完整源码：[GitHub](https://github.com/z2586300277/three-cesium-examples/blob/dev/threeExamples/application/imageMove.js)

## 小结

- 本文提供 **图片移动** 完整 Three.js 源码与在线 Demo，建议先运行案例再改 uniform/参数做二次实验
- 更多 Three.js 实战案例见 [three-cesium-examples 合集](https://threehub.cn) 与 [GitHub 开源仓库](https://github.com/z2586300277/three-cesium-examples)

