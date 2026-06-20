---
title: "Three.js 粒子效果的行星教程"
description: "详解 Three.js 粒子效果的行星：基于 WebGL 实现「粒子效果的行星」可视化效果，附完整可运行源码，涵盖 onBeforeCompile、OrbitControls、THREE.Points 等关键实现，附完整源码与在线 Demo，适合 Three.js 粒子 学习与二次开发。"
head:
  - - meta
    - name: keywords
      content: "Three.js,粒子效果的行星,WebGL,源码,教程,在线案例,onBeforeCompile,shader注入,OrbitControls,相机控制,粒子特效,Points"
outline: deep
---

### 粒子效果的行星 · *Planet* · [▶ 在线运行案例](https://z2586300277.github.io/three-cesium-examples/#/codeMirror?navigation=ThreeJS&classify=particle&id=PlanetParticle)
- **案例合集：** [三维可视化功能案例（threehub.cn）](https://threehub.cn)

- **开源仓库github地址：** https://github.com/z2586300277/three-cesium-examples

- **400个案例代码: ** [网盘链接](https://pan.quark.cn/s/201da5c82fec)

![粒子效果的行星](https://z2586300277.github.io/three-cesium-examples/threeExamples/particle/PlanetParticle.jpg)

## 你将学到什么

- onBeforeCompile 注入 GLSL 改造内置材质
- OrbitControls 相机轨道交互
- THREE.Points 粒子点渲染
- BufferGeometry 自定义顶点/索引数据
- 监听窗口 `resize` 同步更新 camera 与 renderer

## 效果说明

本案例演示 **粒子效果的行星** 效果：基于 WebGL 实现「粒子效果的行星」可视化效果，附完整可运行源码；核心用到 onBeforeCompile、OrbitControls、THREE.Points。建议先打开文首在线案例查看动态画面，再对照下方源码逐步理解。

## 核心概念

- **Scene / Camera / WebGLRenderer** 构成最小渲染闭环；大场景可开 `logarithmicDepthBuffer` 缓解 Z-fighting。
- **onBeforeCompile** 在 Three 拼好内置 shader 后替换 `#include <xxx>` 片段，适合在 PBR 材质上叠加大屏特效。
- **OrbitControls** 提供轨道旋转/缩放；开启 `enableDamping` 后需在 animate 中 `controls.update()`。
- **THREE.Points** 将每个顶点渲染为可控大小的粒子；可用自定义 attribute（如 `u_index`）驱动片元/顶点动画。

## 实现步骤

1. 搭建灯光与环境（如有）
2. requestAnimationFrame 循环 update + render

## 代码要点

```html
<!DOCTYPE html><html lang="en"><head>
    <title>three.js</title>
    <meta charset="utf-8">
    <meta name="viewport" content="width=device-width, user-scalable=no, minimum-scale=1.0, maximum-scale=1.0">
 
  </head>
  <body>

    <script type="importmap">
        {
          "imports": {
            "three": "https://threejs.org/build/three.module.js",
            "three/addons/": "https://threejs.org/examples/jsm/"
          }
        }
      </script>

    <script type="module">
      import * as THREE from "three";
      import { OrbitControls } from "three/addons/controls/OrbitControls.js";
      let scene = new THREE.Scene();
      scene.background = new THREE.Color(0x160016);
      let camera = new THREE.PerspectiveCamera(
        60,
        innerWidth / innerHeight,
        1,
        1000
      );
      camera.position.set(0, 4, 100);
      let renderer = new THREE.WebGLRenderer();
      renderer.setSize(innerWidth, innerHeight);
      document.body.appendChild(renderer.domElement);
      window.addEventListener("resize", (event) => {
        camera.aspect = innerWidth / innerHeight;
        camera.updateProjectionMatrix();
        renderer.setSize(innerWidth, innerHeight);
      });

      let controls = new OrbitControls(camera, renderer.domElement);
      controls.enableDamping = true;
      controls.enablePan = false;

      let gu = {
        time: { value: 0 },
      };

      let sizes = [];
      let shift = [];
      let pushShift = () => {
        shift.push(
          Math.random() * Math.PI,
          Math.random() * Math.PI * 2,
          (Math.random() * 0.9 + 0.1) * Math.PI * 0.1,
          Math.random() * 0.9 + 0.1
        );
      };
      let pts = new Array(50000).fill().map((p) => {
        sizes.push(Math.random() * 1.5 + 0.5);
        pushShift();
        return new THREE.Vector3()
          .randomDirection()
          .multiplyScalar(Math.random() * 0.5 + 9.5);
      });
      for (let i = 0; i < 100000; i++) {
        let r = 10,
          R = 40;
        let rand = Math.pow(Math.random(), 1.5);
        let radius = Math.sqrt(R * R * rand + (1 - rand) * r * r);
        pts.push(
          new THREE.Vector3().setFromCylindricalCoords(
            radius,
            Math.random() * 2 * Math.PI,
            (Math.random() - 0.5) * 2
          )
        );
        sizes.push(Math.random() * 1.5 + 0.5);
        pushShift();
      }

      let g = new THREE.BufferGeometry().setFromPoints(pts);
      g.setAttribute("sizes", new THREE.Float32BufferAttribute(sizes, 1));
      g.setAttribute("shift", new THREE.Float32BufferAttribute(shift, 4));
      let m = new THREE.PointsMaterial({
        size: 0.125,
        transparent: true,
        depthTest: false,
        blending: THREE.AdditiveBlending,
        onBeforeCompile: (shader) => {
          shader.uniforms.time = gu.time;
          shader.vertexShader = `
      uniform float time;
      attribute float sizes;
      attribute vec4 shift;
      varying vec3 vColor;
      ${shader.vertexShader}
    `
            .replace(`gl_PointSize = size;`, `gl_PointSize = size * sizes;`)
            .replace(
              `#include <color_vertex>`,
              `#include <color_vertex>
        float d = length(abs(position) / vec3(40., 10., 40));
        d = clamp(d, 0., 1.);
        vColor = mix(vec3(227., 155., 0.), vec3(100., 50., 255.), d) / 255.;
      `
            )
            .replace(
              `#include <begin_vertex>`,
              `#include <begin_vertex>
        float t = time;
        float moveT = mod(shift.x + shift.z * t, PI2);
        float moveS = mod(shift.y + shift.z * t, PI2);
        transformed += vec3(cos(moveS) * sin(moveT), cos(moveT), sin(moveS) * sin(moveT)) * shift.w;
      `
            );
          shader.fragmentShader = `
      varying vec3 vColor;
      ${shader.fragmentShader}
    `
            .replace(
              `#include <clipping_planes_fragment>`,
              `#include <clipping_planes_fragment>
        float d = length(gl_PointCoord.xy - 0.5);
        //if (d > 0.5) discard;
      `
            )
            .replace(
              `
                float d = length(gl_PointCoord.xy - 0.5);
              vec4 diffuseColor = vec4( diffuse, opacity );`,
              `vec4 diffuseColor = vec4( vColor, smoothstep(0.5, 0.1, d)/* * 0.5 + 0.5*/ );`
            );
        },
      });
      let p = new THREE.Points(g, m);
      p.rotation.order = "ZYX";
      p.rotation.z = 0.2;
      scene.add(p);

      let clock = new THREE.Clock();

      renderer.setAnimationLoop(() => {
        controls.update();
        let t = clock.getElapsedTime() * 0.5;
        gu.time.value = t * Math.PI;
        p.rotation.y = t * 0.05;
        renderer.render(scene, camera);
      });
    </script> 

  </body>

</html>
```

完整源码：[GitHub](https://github.com/z2586300277/three-cesium-examples/blob/dev/threeExamples/particle/PlanetParticle.html)

## 小结

- 本文提供 **粒子效果的行星** 完整 Three.js 源码与在线 Demo，建议先运行案例再改 uniform/参数做二次实验
- 更多 Three.js 实战案例见 [three-cesium-examples 合集](https://threehub.cn) 与 [GitHub 开源仓库](https://github.com/z2586300277/three-cesium-examples)

