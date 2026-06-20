---
title: "Three.js 天空云教程"
description: "详解 Three.js 天空云：用 Canvas 2D 绘制内容并实时映射为 Three.js 纹理，涵盖 ShaderMaterial、InstancedMesh、Canvas 等关键实现，附完整源码与在线 Demo，适合 Three.js 着色器 学习与二次开发。"
head:
  - - meta
    - name: keywords
      content: "Three.js,天空云,WebGL,源码,教程,在线案例,ShaderMaterial,自定义着色器,GLSL,InstancedMesh,实例化,Canvas纹理"
outline: deep
---

### 天空云 · *Cloud Shader* · [▶ 在线运行案例](https://z2586300277.github.io/three-cesium-examples/#/codeMirror?navigation=ThreeJS&classify=shader&id=cloudShader)
- **案例合集：** [三维可视化功能案例（threehub.cn）](https://threehub.cn)

- **开源仓库github地址：** https://github.com/z2586300277/three-cesium-examples

- **400个案例代码: ** [网盘链接](https://pan.quark.cn/s/201da5c82fec)

![天空云](https://z2586300277.github.io/three-cesium-examples/threeExamples/shader/cloudShader.jpg)

## 你将学到什么

- ShaderMaterial 自定义着色器实现核心视觉效果
- InstancedMesh 实例化合批渲染
- Canvas 动态纹理贴图
- `requestAnimationFrame` 渲染循环与 `resize` 自适应

## 效果说明

本案例演示 **天空云** 效果：用 Canvas 2D 绘制内容并实时映射为 Three.js 纹理，大批量相同几何体实例化渲染，降低 draw call；核心用到 ShaderMaterial、InstancedMesh、Canvas。建议先打开文首在线案例查看动态画面，再对照下方源码逐步理解。

## 核心概念
- **Scene / Camera / WebGLRenderer** 构成最小渲染闭环；大场景可开 `logarithmicDepthBuffer` 缓解 Z-fighting。
- **ShaderMaterial** 通过 `uniforms` + 自定义 GLSL 控制逐像素/逐点效果；透明粒子常配合 `depthTest: false`。
- **CanvasTexture** 每帧或按需把 2D Canvas 内容上传 GPU，适合动态文字、图表、视频帧贴图。

## 实现步骤

1. 定义 uniforms，在 rAF 中更新并 render
2. 构建几何 attribute 或 instanceMatrix 并 add 到 scene
3. gui.add 绑定可调参数

## 代码要点

```js
import {
    CanvasTexture,
    TextureLoader,
    PlaneGeometry,
    ShaderMaterial,
    InstancedMesh,
    Object3D,
    Vector2,
    InstancedBufferAttribute,
    PerspectiveCamera,
    WebGLRenderer,
    PCFSoftShadowMap,
    Scene,
    Color
} from "three";
import Stats from "three/examples/jsm/libs/stats.module.js";
import { GUI } from "dat.gui";

function createRandom(min = 0, max = 1) {
    return Math.random() * (max - min) + min;
}
function resize(render, cameras, callback) {
    cameras = Array.isArray(cameras) ? cameras : [cameras];
    window.addEventListener("resize", () => {
        const [w, h] = [window.innerWidth, window.innerHeight];
        render.setSize(window.innerWidth, window.innerHeight);
        cameras.forEach((camera) => {
            if (camera.type === "OrthographicCamera") {
                camera.top = 15 * (h / w);
                camera.bottom = -15 * (h / w);
            } else if (camera.type === "PerspectiveCamera") {
                camera.aspect = window.innerWidth / window.innerHeight;
            }
            camera.updateProjectionMatrix();
        });
        callback && callback(w, h);
    });
}
function initStats(showPanel = 0) {
    const stats = new Stats();
    stats.showPanel(showPanel);
    const dom = document.querySelector("#box");
    dom.appendChild(stats.dom);
    return stats;
}
function initRenderer(props = {}) {
    const dom = document.getElementById("box");
    dom.style.width = "100vw";
    dom.style.height = "100vh";

    const renderer = new WebGLRenderer({ antialias: true, ...props });
    renderer.shadowMap.enabled = true;
    renderer.shadowMapSoft = true;
    renderer.shadowMap.type = PCFSoftShadowMap;
    renderer.setPixelRatio(devicePixelRatio);

    renderer.setClearColor(new Color(0xffffff));
    renderer.setSize(dom.offsetWidth, dom.offsetHeight);

    dom.appendChild(renderer.domElement);
    window.renderer = renderer;

    return renderer;
}
function initGUI(params) {
    return new GUI(params);
}
function initScene() {
    const scene = new Scene();
    window.scene = scene;
    return scene;
}
window.onload = () => {
    init();
};

const vs = /* glsl */ `
  varying vec2 vUv;
  void main(){
    vUv = uv;
    gl_Position = projectionMatrix * modelViewMatrix * instanceMatrix * vec4(position, 1.0);
  }
  `;

const fs = /* glsl */ `
  varying vec2 vUv;
  uniform sampler2D map;
  uniform float fogNear;
  uniform float fogFar;
  uniform vec3 fogColor;
  uniform int enableFog; // 0: false, 1: true
  
  void main(){
    if(enableFog == 1){
      // 计算片源深度 
      float depth = gl_FragCoord.z / gl_FragCoord.w;
      // 计算归一化的深度
      float fogFactor = smoothstep(fogNear, fogFar, depth);
      // 计算雾透明度
      gl_FragColor.w *= pow(gl_FragCoord.z, 20.0);
      // 最终结果
      gl_FragColor = mix(texture2D(map, vUv), vec4(fogColor, gl_FragColor.w), fogFactor);
    }else{
      gl_FragColor = texture2D(map, vUv);
    }
  }
  `

async function init() {
    const dummy = new Object3D();
    const mouse = new Vector2();
    const halfSize = new Vector2(window.innerWidth / 2, window.innerHeight / 2);
    const startTime = Date.now();
    const params = {
        count: 800,
        enableFog: true,
        fogColor: '#4584b4',
        fogNear: -100,
        fogFar: 3000,
    };


    const renderer = initRenderer();
    const camera = new PerspectiveCamera(30, halfSize.x / halfSize.y, 1, params.count * 1.5);
    window.camera = camera;

    const status = initStats();
    const scene = initScene();

    // background
    const backgroundCanvas = document.createElement("canvas");
    const ctx = backgroundCanvas.getContext("2d");
    const gradient = ctx.createLinearGradient(0, 0, 0, backgroundCanvas.height);
    gradient.addColorStop(0, "#1e4877");
    gradient.addColorStop(0.5, "#4584b4");
    ctx.fillStyle = gradient;
    ctx.fillRect(0, 0, backgroundCanvas.width, backgroundCanvas.height);

    const bgTexture = new CanvasTexture(backgroundCanvas);
    scene.background = bgTexture;

    // cloud
    const loader = new TextureLoader();
    const cloudTexture = await loader.loadAsync(
        FILE_HOST + 'images/channels/cloud.png'
    );

    const geometry = new PlaneGeometry(64, 64);
    const material = new ShaderMaterial({
        uniforms: {
            map: {
                value: cloudTexture,
            },
            fogColor: {
                value: new Color(params.fogColor),
            },
            fogNear: {
                value: params.fogNear,
            },
            fogFar: {
                value: params.fogFar,
            },
            enableFog: {
                value: Number(params.enableFog),
            }
        },
        vertexShader: vs,
        fragmentShader: fs,
        depthWrite: false,
        depthTest: false,
        transparent: true,
    });

    const mesh = new InstancedMesh(geometry, material, params.count);
    scene.add(mesh);


    function updateMeshCount() {
        const count = params.count;
        mesh.count = count;
        mesh.dispose();
        mesh.instanceMatrix = new InstancedBufferAttribute(
            new Float32Array(count * 16),
            16
        );

        for (let j = 0, k = count; j < k; j++) {
            dummy.position.x = createRandom(-500, 500);
            dummy.position.y = -Math.random() * Math.random() * 200 - 15;
            dummy.position.z = j;
            dummy.rotation.z = Math.random() * Math.PI;
            dummy.scale.x = dummy.scale.y = Math.random() * Math.random() * 1.5 + 0.5;
            dummy.updateMatrix();
            mesh.setMatrixAt(j, dummy.matrix);
        }

        mesh.instanceMatrix.needsUpdate = true;
        camera.position.z = count;
        camera.far = count * 1.5;
        camera.updateProjectionMatrix();
    }

    updateMeshCount();

    function render() {
        camera.position.x += (mouse.x - camera.position.x) * 0.01;
        camera.position.y += (-mouse.y - camera.position.y) * 0.01;
        camera.position.z = -((Date.now() - startTime) * 0.03) % params.count + params.count;

        renderer.render(scene, camera);
        status.update();
        requestAnimationFrame(render);
    }
    render();


    renderer.domElement.addEventListener("mousemove", ({ clientX, clientY }) => {
        mouse.set(
            (clientX - halfSize.x) * 0.25,
            (clientY - halfSize.y) * 0.15
        );
    })

    resize(renderer, [camera], () => {
        halfSize.set(window.innerWidth / 2, window.innerHeight / 2);
    })

    // gui
    const gui = initGUI();

    gui.add(params, "count", 0, 10000).onChange(updateMeshCount).name("Count");
    const enableFogOption = gui.add(params, "enableFog").name('Enable Fog');

    const fogFolder = gui.addFolder('Fog');
    fogFolder.show(params.enableFog);
    fogFolder.addColor(params, "fogColor").onChange((e) => {
        material.uniforms.fogColor.value.set(e);
        material.needsUpdate = true;
    }).name("Fog Color");
    fogFolder.add(params, "fogNear", -1000, 1000).onChange((e) => {
        material.uniforms.fogNear.value = e;
        material.needsUpdate = true;
    }).name("Fog Near");
    fogFolder.add(params, "fogFar", 0, 5000).onChange((e) => {
        material.uniforms.fogFar.value = e;
        material.needsUpdate = true;
    }).name("Fog Far");

    enableFogOption.onChange((e) => {
        material.uniforms.enableFog.value = e ? 1 : 0;
        material.needsUpdate = true;
        fogFolder.show(e);
    });
}
```

完整源码：[GitHub](https://github.com/z2586300277/three-cesium-examples/blob/dev/threeExamples/shader/cloudShader.js)

## 小结

- 本文提供 **天空云** 完整 Three.js 源码与在线 Demo，建议先运行案例再改 uniform/参数做二次实验
- 更多 Three.js 实战案例见 [three-cesium-examples 合集](https://threehub.cn) 与 [GitHub 开源仓库](https://github.com/z2586300277/three-cesium-examples)

