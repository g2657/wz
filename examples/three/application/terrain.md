---
title: "Three.js 程序化地形生成教程"
description: "详解 Three.js 程序化地形生成：基于 WebGL 实现「程序化地形生成」可视化效果，附完整可运行源码，涵盖 ShaderMaterial、OrbitControls、场景雾效增强纵深 等关键实现，附完整源码与在线 Demo，适合 Three.js 应用场景 学习与二次开发。"
head:
  - - meta
    - name: keywords
      content: "Three.js,程序化地形生成,WebGL,源码,教程,在线案例,ShaderMaterial,自定义着色器,GLSL,OrbitControls,相机控制,雾效"
outline: deep
---

### 程序化地形生成 · *Terrain* · [▶ 在线运行案例](https://z2586300277.github.io/three-cesium-examples/#/codeMirror?navigation=ThreeJS&classify=application&id=terrain)
- **案例合集：** [三维可视化功能案例（threehub.cn）](https://threehub.cn)

- **开源仓库github地址：** https://github.com/z2586300277/three-cesium-examples

- **400个案例代码: ** [网盘链接](https://pan.quark.cn/s/201da5c82fec)

![程序化地形生成](https://z2586300277.github.io/three-cesium-examples/threeExamples/application/generate_terrain.jpg)

## 你将学到什么

- ShaderMaterial 自定义着色器实现核心视觉效果
- OrbitControls 相机轨道交互
- 场景雾效增强纵深
- `requestAnimationFrame` 渲染循环与 `resize` 自适应

## 效果说明

本案例演示 **程序化地形生成** 效果：基于 WebGL 实现「程序化地形生成」可视化效果，附完整可运行源码；核心用到 ShaderMaterial、OrbitControls、场景雾效增强纵深。建议先打开文首在线案例查看动态画面，再对照下方源码逐步理解。

## 核心概念

- **Scene / Camera / WebGLRenderer** 构成最小渲染闭环；大场景可开 `logarithmicDepthBuffer` 缓解 Z-fighting。
- **ShaderMaterial** 通过 `uniforms` + 自定义 GLSL 控制逐像素/逐点效果；透明粒子常配合 `depthTest: false`。
- **OrbitControls** 提供轨道旋转/缩放；开启 `enableDamping` 后需在 animate 中 `controls.update()`。

## 实现步骤

1. 搭建灯光与环境（如有）
2. requestAnimationFrame 循环 update + render

## 代码要点

```js
import {
    Scene,
    Color,
    Fog,
    PerspectiveCamera,
    WebGLRenderer,
    DirectionalLight,
    AmbientLight,
    GridHelper,
    PlaneGeometry,
    Mesh,
    ShaderMaterial,
    DoubleSide,
  } from "three";
  import { OrbitControls } from "three/examples/jsm/controls/OrbitControls.js";


  
  let scene,
    camera,
    renderer,
    controls;
  let init_scene = () => {
    scene = new Scene();
    scene.background = new Color(0.5, 1, 0.875);
    scene.fog = new Fog(scene.background, 20, 45);
    camera = new PerspectiveCamera(60, innerWidth / innerHeight, 1, 1000);
    let vHeight = 3;
    camera.position.set(30, vHeight + 2, 20).setLength(15);
    renderer = new WebGLRenderer({ antialias: true });
    renderer.setSize(innerWidth, innerHeight);
    document.body.appendChild(renderer.domElement);
    window.addEventListener("resize", () => {
      camera.aspect = innerWidth / innerHeight;
      camera.updateProjectionMatrix();
      renderer.setSize(innerWidth, innerHeight);
    });
    controls = new OrbitControls(camera, renderer.domElement);
    controls.target.set(0, vHeight, 0);
    controls.update();
    controls.minPolarAngle = Math.PI * 0.4;
    controls.maxPolarAngle = Math.PI * 0.5;
    controls.minDistance = 10;
    controls.maxDistance = 20;
    controls.enableDamping = true;
    controls.enablePan = false;
    let light = new DirectionalLight(0xffffff, 0.25);
    light.position.setScalar(1);
    scene.add(light, new AmbientLight(0xffffff, 0.75));
    add_helper();
  };
  const add_helper = () => {
    const grid = new GridHelper(50, 50);
    scene.add(grid);
  };
  let mesh;
  const generate_terrain = async () => {
    // let perlin = new ImprovedNoise();
    const planeGeo = new PlaneGeometry(50, 50, 500, 500);
    planeGeo.rotateX(-Math.PI / 2);
    let vertexShader = `
        vec3 hash(vec3 p) {
            p = vec3( dot(p, vec3(127.1, 311.7, 74.7)),
                      dot(p, vec3(269.5, 183.3, 246.1)),
                      dot(p, vec3(113.5, 271.9, 124.6)) );
            return fract(sin(p) * 43758.5453123);
        }
  
          // returns 3D value noise
        float noise( in vec3 x )
        {
            // grid
            vec3 p = floor(x);
            vec3 w = fract(x);
            
            // quintic interpolant
            vec3 u = w*w*w*(w*(w*6.0-15.0)+10.0);
  
            
            // gradients
            vec3 ga = hash( p+vec3(0.0,0.0,0.0) );
            vec3 gb = hash( p+vec3(1.0,0.0,0.0) );
            vec3 gc = hash( p+vec3(0.0,1.0,0.0) );
            vec3 gd = hash( p+vec3(1.0,1.0,0.0) );
            vec3 ge = hash( p+vec3(0.0,0.0,1.0) );
            vec3 gf = hash( p+vec3(1.0,0.0,1.0) );
            vec3 gg = hash( p+vec3(0.0,1.0,1.0) );
            vec3 gh = hash( p+vec3(1.0,1.0,1.0) );
            
            // projections
            float va = dot( ga, w-vec3(0.0,0.0,0.0) );
            float vb = dot( gb, w-vec3(1.0,0.0,0.0) );
            float vc = dot( gc, w-vec3(0.0,1.0,0.0) );
            float vd = dot( gd, w-vec3(1.0,1.0,0.0) );
            float ve = dot( ge, w-vec3(0.0,0.0,1.0) );
            float vf = dot( gf, w-vec3(1.0,0.0,1.0) );
            float vg = dot( gg, w-vec3(0.0,1.0,1.0) );
            float vh = dot( gh, w-vec3(1.0,1.0,1.0) );
          
            // interpolation
            return va +
                  u.x*(vb-va) +
                  u.y*(vc-va) +
                  u.z*(ve-va) +
                  u.x*u.y*(va-vb-vc+vd) +
                  u.y*u.z*(va-vc-ve+vg) +
                  u.z*u.x*(va-vb-ve+vf) +
                  u.x*u.y*u.z*(-va+vb+vc-vd+ve-vf-vg+vh);
        }
                  varying vec2 v_uv;
          void main(){
    v_uv = uv;
          vec3 in_position = position;
           float noise_value = noise(position);
           float y = abs(noise_value);
           y = pow(y,3.);
           in_position.y = min(y*35.,15.)*2.;
            gl_Position = projectionMatrix * modelViewMatrix * vec4( in_position, 1.0 );
          }
    `;
    let fragmentShader = `
                  varying vec2 v_uv;
  
        void main(){
          gl_FragColor = vec4(v_uv,0.5,1.);
        }
    `;
    let m = new ShaderMaterial({
      vertexShader,
      fragmentShader,
      side: DoubleSide,
    });
    mesh = new Mesh(planeGeo, m);
    scene.add(mesh);
  };
  
  const render = () => {
    renderer.render(scene, camera);
    requestAnimationFrame(render);
    mesh.rotation.y += 0.001
  };
    init_scene();
    generate_terrain();
    render();
```

完整源码：[GitHub](https://github.com/z2586300277/three-cesium-examples/blob/dev/threeExamples/application/generate_terrain.js)

## 小结

- 本文提供 **程序化地形生成** 完整 Three.js 源码与在线 Demo，建议先运行案例再改 uniform/参数做二次实验
- 更多 Three.js 实战案例见 [three-cesium-examples 合集](https://threehub.cn) 与 [GitHub 开源仓库](https://github.com/z2586300277/three-cesium-examples)

