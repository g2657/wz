---
title: "Three.js 人物虚化教程"
description: "详解 Three.js 人物虚化：用 Canvas 2D 绘制内容并实时映射为 Three.js 纹理，涵盖 onBeforeCompile、OrbitControls、Canvas 等关键实现，附完整源码与在线 Demo，适合 Three.js 游戏复刻 学习与二次开发。"
head:
  - - meta
    - name: keywords
      content: "Three.js,人物虚化,WebGL,源码,教程,在线案例,onBeforeCompile,shader注入,OrbitControls,相机控制,Canvas纹理,动态贴图"
outline: deep
---

### 人物虚化 · *人物虚化* · [▶ 在线运行案例](https://z2586300277.github.io/three-cesium-examples/#/codeMirror?navigation=ThreeJS&classify=game&id=characterBlur)
- **案例合集：** [三维可视化功能案例（threehub.cn）](https://threehub.cn)

- **开源仓库github地址：** https://github.com/z2586300277/three-cesium-examples

- **400个案例代码: ** [网盘链接](https://pan.quark.cn/s/201da5c82fec)

![人物虚化](https://z2586300277.github.io/three-cesium-examples/threeExamples/game/characterBlur.jpg)

## 你将学到什么

- onBeforeCompile 注入 GLSL 改造内置材质
- OrbitControls 相机轨道交互
- Canvas 动态纹理贴图
- FBXLoader 加载 FBX 城市/角色模型
- 骨骼动画与 AnimationMixer
- `requestAnimationFrame` 渲染循环与 `resize` 自适应

## 效果说明

本案例演示 **人物虚化** 效果：用 Canvas 2D 绘制内容并实时映射为 Three.js 纹理；核心用到 onBeforeCompile、OrbitControls、Canvas。建议先打开文首在线案例查看动态画面，再对照下方源码逐步理解。

## 核心概念
- **Scene / Camera / WebGLRenderer** 构成最小渲染闭环；大场景可开 `logarithmicDepthBuffer` 缓解 Z-fighting。
- **onBeforeCompile** 在 Three 拼好内置 shader 后替换 `#include <xxx>` 片段，适合在 PBR 材质上叠加大屏特效。
- **OrbitControls** 提供轨道旋转/缩放；开启 `enableDamping` 后需在 animate 中 `controls.update()`。
- **CanvasTexture** 每帧或按需把 2D Canvas 内容上传 GPU，适合动态文字、图表、视频帧贴图。

## 实现步骤

1. 搭建 Scene、PerspectiveCamera、WebGLRenderer，挂载 canvas 并处理 `resize`
2. 异步加载模型 / 3D Tiles / GeoJSON 等资源并加入 scene 或 entities
3. 定义 uniforms / onBeforeCompile 或 ShaderMaterial，编写 GLSL 与材质参数
4. 创建 OrbitControls（及 Raycaster 等交互控件，若源码包含）
5. 在 `requestAnimationFrame` 循环中更新状态并 render（Cesium 为 `viewer.render` 或自动渲染）

## 代码要点

```js
import * as THREE from "three";
import { FBXLoader } from "three/examples/jsm/loaders/FBXLoader.js";
import { OrbitControls } from "three/examples/jsm/controls/OrbitControls.js";
import { GUI } from "dat.gui";

const modelUrl = "https://ylfq.github.io/model/walk.fbx";

const canvas = document.createElement("canvas");
canvas.style.width = "100vw !important";
canvas.style.height = "100vh !important";
document.body.appendChild(canvas);

const renderer = new THREE.WebGLRenderer({ canvas: canvas, antialias: true, alpha: true });
renderer.setClearColor(0x333333, 0);
renderer.shadowMap.enabled = true;
renderer.shadowMap.type = THREE.PCFShadowMap;

const scene = new THREE.Scene();

const loader_fbx = new FBXLoader();

const modelMaterialUniforms = {
  blur: 0.85,
  blockSize: 12.0,
};

const model = await loader_fbx.loadAsync(modelUrl);
model.traverse((obj) => {
  if (obj instanceof THREE.Mesh) {
    obj.castShadow = true;
    obj.receiveShadow = true;

    /** @type {THREE.MeshPhongMaterial} */
    const m = obj.material;
    m.onBeforeCompile = (shader) => {
      shader.uniforms.blur = {
        get value() {
          return modelMaterialUniforms.blur;
        },
      };
      shader.uniforms.blockSize = {
        get value() {
          return modelMaterialUniforms.blockSize;
        },
      };

      shader.fragmentShader = shader.fragmentShader.replace(
        /* glsl */ `void main() {`,
        /* glsl */ `
          // 虚化阈值
          uniform float blur;
          // 虚化区块大小
          uniform float blockSize;

          // 虚化矩阵，自定义区块内的虚化顺序，当像素对应的虚化矩阵值小于blur时，该像素不进行渲染
          const mat4 blurMatrix = mat4(
            0.1, 0.5, 0.7, 0.3,
            0.6, 0.6, 0.9, 0.8,
            0.8, 1.0, 0.2, 0.6,
            0.3, 0.7, 0.5, 0.4
          );
            
          void main() {
            ivec2 xyInBlur = ivec2(fract(gl_FragCoord.xy / blockSize) * 4.0);
            if (blurMatrix[xyInBlur.y][xyInBlur.x] <= blur) discard;
        `
      );
    };
  }
});
const mixer = new THREE.AnimationMixer(model);
const action = mixer.clipAction(model.animations[0]);
action.setLoop(THREE.LoopRepeat);
action.play();

const light_ambient = new THREE.AmbientLight(0xffffff, 0.7);

const light_directional = new THREE.DirectionalLight(0xffffff, 0.8);
light_directional.position.set(200, 200, 200);
light_directional.castShadow = true;
light_directional.shadow.camera.left = -100;
light_directional.shadow.camera.right = 100;
light_directional.shadow.camera.top = 100;
light_directional.shadow.camera.bottom = -100;
light_directional.shadow.camera.near = 1;
light_directional.shadow.camera.far = 1000;
light_directional.shadow.mapSize.width = 1024;
light_directional.shadow.mapSize.height = 1024;

scene.add(model, light_ambient, light_directional);

const camera = new THREE.PerspectiveCamera(90, 1, 0.1, 1000);
camera.position.set(0, 0, 140);

const controls = new OrbitControls(camera, canvas);
controls.enableDamping = true;

const timer = new THREE.Timer();

const tick = (delta, elapsed) => {
  controls.update(delta);

  // 更新动画并限制模型不进行移动
  mixer.update(delta * 0.9);
  model.children[2].children[0].position.set(0, 0, 0);
};

const render = () => {
  renderer.render(scene, camera);
};

const ani = () => {
  const elapsed = timer.getElapsed();
  const delta = timer.getDelta();

  timer.update();

  tick(delta, elapsed);
  render();

  requestAnimationFrame(ani);
};

const data = {
  get blur() {
    return modelMaterialUniforms.blur;
  },
  set blur(v) {
    modelMaterialUniforms.blur = v;
  },

  get blockSize() {
    return modelMaterialUniforms.blockSize;
  },
  set blockSize(v) {
    modelMaterialUniforms.blockSize = v;
  },
};
const gui = new GUI();
gui.add(data, "blur", 0, 1, 0.001).name("虚化强度");
gui.add(data, "blockSize", 2.0, 20.0, 4.0).name("虚化区块大小");

new ResizeObserver(() => {
  const rect = document.body.getBoundingClientRect();
  const w = rect.width;
  const h = rect.height;
  const a = w / h;
  const dpr = window.devicePixelRatio * 1.25;

  renderer.setSize(w, h, false);
  renderer.setPixelRatio(dpr);

  camera.aspect = a;
  camera.updateProjectionMatrix();
}).observe(document.body);

ani();
```

完整源码：[GitHub](https://github.com/z2586300277/three-cesium-examples/blob/dev/threeExamples/game/characterBlur.js)

## 小结

- 本文提供 **人物虚化** 完整 Three.js 源码与在线 Demo，建议先运行案例再改 uniform/参数做二次实验
- 更多 Three.js 实战案例见 [three-cesium-examples 合集](https://threehub.cn) 与 [GitHub 开源仓库](https://github.com/z2586300277/three-cesium-examples)

