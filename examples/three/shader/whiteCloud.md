---
title: "Three.js 白云教程"
description: "详解 Three.js 白云：基于 WebGL 实现「白云」可视化效果，附完整可运行源码，涵盖 ShaderMaterial、OrbitControls、glTF/Draco 等关键实现，附完整源码与在线 Demo，适合 Three.js 着色器 学习与二次开发。"
head:
  - - meta
    - name: keywords
      content: "Three.js,白云,WebGL,源码,教程,在线案例,ShaderMaterial,自定义着色器,GLSL,OrbitControls,相机控制,glTF"
outline: deep
---

### 白云 · *White Cloud* · [▶ 在线运行案例](https://z2586300277.github.io/three-cesium-examples/#/codeMirror?navigation=ThreeJS&classify=shader&id=whiteCloud)
- **案例合集：** [三维可视化功能案例（threehub.cn）](https://threehub.cn)

- **开源仓库github地址：** https://github.com/z2586300277/three-cesium-examples

- **400个案例代码: ** [网盘链接](https://pan.quark.cn/s/201da5c82fec)

![白云](https://z2586300277.github.io/three-cesium-examples/threeExamples/shader/whiteCloud.jpg)

## 你将学到什么

- ShaderMaterial 自定义着色器实现核心视觉效果
- OrbitControls 相机轨道交互
- glTF/Draco 模型加载与优化
- `requestAnimationFrame` 渲染循环与 `resize` 自适应

## 效果说明

本案例演示 **白云** 效果：基于 WebGL 实现「白云」可视化效果，附完整可运行源码；核心用到 ShaderMaterial、OrbitControls、glTF/Draco。建议先打开文首在线案例查看动态画面，再对照下方源码逐步理解。

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
import * as THREE from "three";
import { OrbitControls } from "three/addons/controls/OrbitControls.js";
import { GLTFLoader } from "three/addons/loaders/GLTFLoader.js";

const { innerWidth, innerHeight } = window;
const aspect = innerWidth / innerHeight;

class Base {
    constructor () {
        this.init();
        this.main();
    }
    main() {
        this.deltaTime = {
            value: 0
        }

        var fragmentSrc = [
            "precision mediump float;",

            "uniform vec2 iResolution;",
            "uniform float iGlobalTime;",
            "uniform vec2 iMouse;",

            "float hash( float n ) {",
            "return fract(sin(n)*43758.5453);",
            "}",

            "float noise( in vec3 x ) {",
            "vec3 p = floor(x);",
            "vec3 f = fract(x);",
            "f = f*f*(3.0-2.0*f);",
            "float n = p.x + p.y*57.0 + 113.0*p.z;",
            "return mix(mix(mix( hash(n+  0.0), hash(n+  1.0),f.x),",
            "mix( hash(n+ 57.0), hash(n+ 58.0),f.x),f.y),",
            "mix(mix( hash(n+113.0), hash(n+114.0),f.x),",
            "mix( hash(n+170.0), hash(n+171.0),f.x),f.y),f.z);",
            "}",

            "vec4 map( in vec3 p ) {",
            "float d = 0.2 - p.y;",
            "vec3 q = p - vec3(1.0,0.1,0.0)*iGlobalTime;",
            "float f;",
            "f  = 0.5000*noise( q ); q = q*2.02;",
            "f += 0.2500*noise( q ); q = q*2.03;",
            "f += 0.1250*noise( q ); q = q*2.01;",
            "f += 0.0625*noise( q );",
            "d += 3.0 * f;",
            "d = clamp( d, 0.0, 1.0 );",
            "vec4 res = vec4( d );",
            "res.xyz = mix( 1.15*vec3(1.0,0.95,0.8), vec3(0.7,0.7,0.7), res.x );",
            "return res;",
            "}",

            "vec3 sundir = vec3(-1.0,0.0,0.0);",

            "vec4 raymarch( in vec3 ro, in vec3 rd ) {",
            "vec4 sum = vec4(0, 0, 0, 0);",
            "float t = 0.0;",
            "for(int i=0; i<64; i++) {",
            "if( sum.a > 0.99 ) continue;",

            "vec3 pos = ro + t*rd;",
            "vec4 col = map( pos );",

            "#if 1",
            "float dif =  clamp((col.w - map(pos+0.3*sundir).w)/0.6, 0.0, 1.0 );",
            "vec3 lin = vec3(0.65,0.68,0.7)*1.35 + 0.45*vec3(0.7, 0.5, 0.3)*dif;",
            "col.xyz *= lin;",
            "#endif",

            "col.a *= 0.35;",
            "col.rgb *= col.a;",
            "sum = sum + col*(1.0 - sum.a);	",

            "#if 0",
            "t += 0.1;",
            "#else",
            "t += max(0.1,0.025*t);",
            "#endif",
            "}",

            "sum.xyz /= (0.001+sum.w);",
            "return clamp( sum, 0.0, 1.0 );",
            "}",

            "void main(void) {",
            "vec2 q = gl_FragCoord.xy / iResolution.xy;",
            "vec2 p = -1.0 + 2.0*q;",
            "p.x *= iResolution.x/ iResolution.y;",
            "vec2 mo = -1.0 + 2.0*iMouse.xy / iResolution.xy;",

            // camera
            "vec3 ro = 4.0*normalize(vec3(cos(2.75-3.0*mo.x), 0.7+(mo.y+1.0), sin(2.75-3.0*mo.x)));",
            "vec3 ta = vec3(0.0, 1.0, 0.0);",
            "vec3 ww = normalize( ta - ro);",
            "vec3 uu = normalize(cross( vec3(0.0,1.0,0.0), ww ));",
            "vec3 vv = normalize(cross(ww,uu));",
            "vec3 rd = normalize( p.x*uu + p.y*vv + 1.5*ww );",


            "vec4 res = raymarch( ro, rd );",

            "float sun = clamp( dot(sundir,rd), 0.0, 1.0 );",
            "vec3 col = vec3(0.6,0.71,0.75) - rd.y*0.2*vec3(1.0,0.5,1.0) + 0.15*0.5;",
            "col += 0.2*vec3(1.0,.6,0.1)*pow( sun, 8.0 );",
            "col *= 0.95;",
            "col = mix( col, res.xyz, res.w );",
            "col += 0.1*vec3(1.0,0.4,0.2)*pow( sun, 3.0 );",

            "gl_FragColor = vec4( col, 1.0 );",
            "}"
        ];

        const geometry = new THREE.SphereGeometry(5000, 50);
        const material = new THREE.ShaderMaterial({
            transparent: true,
            side: THREE.BackSide,
            uniforms: {
                iGlobalTime: this.deltaTime,
                iResolution: {
                    value: {
                        x: window.innerWidth,
                        y: window.innerHeight
                    },
                },
                iMouse: {
                    value: {
                        x: 0,
                        y: 0
                    }
                }
            },
            vertexShader: `
                varying vec2 vUv;
                void main() {
                    vUv = uv;
                    gl_Position = projectionMatrix * modelViewMatrix * vec4(position,1.0);
                }`,
            fragmentShader: fragmentSrc.join("\n"),
        });
        const mesh = new THREE.Mesh(geometry, material);
        this.scene.add(mesh);
    
        const scope = this
        function animate() {

            scope.controls.update();
            scope.renderer.render(scope.scene, scope.camera);
            scope.deltaTime.value = scope.clock.getElapsedTime()

            requestAnimationFrame(animate);
        }

        animate()

    }
    init() {
        this.clock = new THREE.Clock();

        this.loader = new GLTFLoader();

        this.renderer = new THREE.WebGLRenderer({
            antialias: true,
            logarithmicDepthBuffer: true,
        });
        this.renderer.setPixelRatio(window.devicePixelRatio);
        this.renderer.setSize(innerWidth, innerHeight);
        document.body.appendChild(this.renderer.domElement);

        this.camera = new THREE.PerspectiveCamera(60, aspect, 0.01, 10000);
        this.camera.position.set(5, 5, 5);

        this.scene = new THREE.Scene();

        this.controls = new OrbitControls(this.camera, this.renderer.domElement);

        const light = new THREE.AmbientLight(0xffffff, 0.5);
        this.scene.add(light);
    }

}
new Base();
```

完整源码：[GitHub](https://github.com/z2586300277/three-cesium-examples/blob/dev/threeExamples/shader/whiteCloud.js)

## 小结

- 本文提供 **白云** 完整 Three.js 源码与在线 Demo，建议先运行案例再改 uniform/参数做二次实验
- 更多 Three.js 实战案例见 [three-cesium-examples 合集](https://threehub.cn) 与 [GitHub 开源仓库](https://github.com/z2586300277/three-cesium-examples)

