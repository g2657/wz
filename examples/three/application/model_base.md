---
title: "Three.js 生成模型底座教程"
description: "详解 Three.js 生成模型底座：基于 WebGL 实现「生成模型底座」可视化效果，附完整可运行源码，涵盖 OrbitControls、glTF/Draco 等关键实现，附完整源码与在线 Demo，适合 Three.js 应用场景 学习与二次开发。"
head:
  - - meta
    - name: keywords
      content: "Three.js,生成模型底座,WebGL,源码,教程,在线案例,OrbitControls,相机控制,glTF,模型加载"
outline: deep
---
### 生成模型底座 · *Model Base* · [▶ 在线运行案例](https://z2586300277.github.io/three-cesium-examples/#/codeMirror?navigation=ThreeJS&classify=application&id=model_base)
- **案例合集：** [三维可视化功能案例（threehub.cn）](https://threehub.cn)

- **开源仓库github地址：** https://github.com/z2586300277/three-cesium-examples

- **400个案例代码: ** [网盘链接](https://pan.quark.cn/s/201da5c82fec)

![生成模型底座](https://z2586300277.github.io/three-cesium-examples/threeExamples/application/model_base.jpg)

## 你将学到什么

- OrbitControls 相机轨道交互
- glTF/Draco 模型加载与优化
- 监听窗口 `resize` 同步更新 camera 与 renderer

## 效果说明

本案例演示 **生成模型底座** 效果：基于 WebGL 实现「生成模型底座」可视化效果，附完整可运行源码；核心用到 OrbitControls、glTF/Draco。建议先打开文首在线案例查看动态画面，再对照下方源码逐步理解。

## 核心概念

- **Loader** 异步加载模型；glTF 返回 `gltf.scene`，加载后注意 `scale` 与坐标系。Draco 需配置 `DRACOLoader`。

- **OrbitControls** 轨道旋转缩放；开 `enableDamping` 时每帧需 `controls.update()`。

- **CubeTexture** 六面贴图作 `scene.background`；`scene.environment` 供 PBR 材质反射。

## 实现步骤

1. 搭建 Scene / Camera / Renderer 与 OrbitControls
2. Loader 异步加载模型/纹理资源
3. rAF 循环中 update 并 render

## 代码要点

```js
import * as THREE from 'three';
import { OrbitControls } from "three/examples/jsm/Addons.js";
import { GLTFLoader } from 'three/examples/jsm/loaders/GLTFLoader.js'
import { DRACOLoader } from 'three/examples/jsm/loaders/DRACOLoader.js'
const DOM = document.getElementById('box')

var scene = new THREE.Scene();
scene.background = new THREE.Color('gainsboro');

var camera = new THREE.PerspectiveCamera(30, innerWidth / innerHeight);
camera.position.set(0, 4, 4);
camera.lookAt(scene.position);

var renderer = new THREE.WebGLRenderer({ antialias: true });
renderer.setSize(innerWidth, innerHeight);
renderer.setAnimationLoop(animationLoop);

function animationLoop() {
    renderer.render(scene, camera);
}
DOM.appendChild(renderer.domElement);

window.addEventListener("resize", () => {
    camera.aspect = innerWidth / innerHeight;
    camera.updateProjectionMatrix();
    renderer.setSize(innerWidth, innerHeight);
});

var controls = new OrbitControls(camera, renderer.domElement);
controls.enableDamping = true;
controls.autoRotate = true;

var light = new THREE.DirectionalLight('white', 3);
light.position.set(1, 1, 1);
scene.add(light);

let group

const loader = new GLTFLoader()

loader.setDRACOLoader(new DRACOLoader().setDecoderPath(FILE_HOST + 'js/three/draco/'))

loader.load(

    HOST + '/files/model/car.glb',

    function (gltf) {

        group = gltf.scene

        scene.add(group)
        add_model_base()
    }

)

// 模型底座
function add_model_base(){
    const box = new THREE.Box3()
    box.setFromObject(group)
    // const helper = new THREE.Box3Helper(box)
    // scene.add(helper)
    const center = box.getCenter(new THREE.Vector3())
    const size = box.getSize(new THREE.Vector3())
    const shape = new  THREE.Shape()
    shape.moveTo(center.x,center.z)
    // 底座大小在这控制 这里取半径的根号2倍
    let radius = Math.max(size.x,size.z) / 2 * Math.sqrt(2)
    shape.arc(0,0,radius,0,Math.PI * 2)
    let m = new THREE.MeshBasicMaterial({color:'red',side:2})
    const geo = new THREE.ShapeGeometry(shape,32)
    const mesh = new THREE.Mesh(geo,m)
    geo.center()
    mesh.rotateX(-Math.PI / 2)
    scene.add(mesh)
}
```

完整源码：[GitHub](https://github.com/z2586300277/three-cesium-examples/blob/dev/threeExamples/application/model_base.js)

## 小结

- 本文提供 **生成模型底座** 完整 Three.js 源码与在线 Demo，建议先运行案例再改 uniform/参数做二次实验
- 更多 Three.js 实战案例见 [three-cesium-examples 合集](https://threehub.cn) 与 [GitHub 开源仓库](https://github.com/z2586300277/three-cesium-examples)

