---
title: "Three.js 模型导航教程"
description: "详解 Three.js 模型导航：基于 WebGL 实现「模型导航」可视化效果，附完整可运行源码，涵盖 OrbitControls、glTF/Draco、GSAP 等关键实现，附完整源码与在线 Demo，适合 Three.js 应用场景 学习与二次开发。"
head:
  - - meta
    - name: keywords
      content: "Three.js,模型导航,WebGL,源码,教程,在线案例,OrbitControls,相机控制,glTF,模型加载,GSAP,动画"
outline: deep
---

### 模型导航 · *Model Nav* · [▶ 在线运行案例](https://z2586300277.github.io/three-cesium-examples/#/codeMirror?navigation=ThreeJS&classify=application&id=model_navigation)
- **案例合集：** [三维可视化功能案例（threehub.cn）](https://threehub.cn)

- **开源仓库github地址：** https://github.com/z2586300277/three-cesium-examples

- **400个案例代码: ** [网盘链接](https://pan.quark.cn/s/201da5c82fec)

![模型导航](https://z2586300277.github.io/three-cesium-examples/threeExamples/application/nav.jpg)

## 你将学到什么

- OrbitControls 相机轨道交互
- glTF/Draco 模型加载与优化
- GSAP 时间轴与补间动画
- `requestAnimationFrame` 渲染循环与 `resize` 自适应

## 效果说明

本案例演示 **模型导航** 效果：基于 WebGL 实现「模型导航」可视化效果，附完整可运行源码；核心用到 OrbitControls、glTF/Draco、GSAP。建议先打开文首在线案例查看动态画面，再对照下方源码逐步理解。

## 核心概念

- **Scene / Camera / WebGLRenderer** 构成最小渲染闭环；大场景可开 `logarithmicDepthBuffer` 缓解 Z-fighting。
- **OrbitControls** 提供轨道旋转/缩放；开启 `enableDamping` 后需在 animate 中 `controls.update()`。
- 阅读下方完整源码时，建议从 `init` / `load` / `animate` 三条主线入手，再深入 shader 与工具函数。

## 实现步骤

1. 搭建灯光与环境（如有）
2. requestAnimationFrame 循环 update + render

## 代码要点

```js
// 模型导航 从0，0，0自动寻路至56，0，0 改变坐标即可重新计算（version：0.1:导航错误等未处理）
import * as THREE from "three";
import { OrbitControls } from "three/examples/jsm/controls/OrbitControls.js";
import { GLTFLoader } from "three/examples/jsm/loaders/GLTFLoader.js";
import { DRACOLoader } from "three/examples/jsm/loaders/DRACOLoader.js";
const box = document.getElementById("box");

const scene = new THREE.Scene();

const camera = new THREE.PerspectiveCamera(
  75,
  box.clientWidth / box.clientHeight,
  0.1,
  1000
);

camera.position.set(20, 50, 0);

const renderer = new THREE.WebGLRenderer({
  antialias: true,
  alpha: true,
  logarithmicDepthBuffer: true,
});

renderer.setSize(box.clientWidth, box.clientHeight);

scene.add(new THREE.AxesHelper(10))
scene.add(new THREE.GridHelper(100, 100))
box.appendChild(renderer.domElement);

new OrbitControls(camera, renderer.domElement);

window.onresize = () => {
  renderer.setSize(box.clientWidth, box.clientHeight);

  camera.aspect = box.clientWidth / box.clientHeight;
  camera.updateProjectionMatrix();
};

animate();

function animate() {
  requestAnimationFrame(animate);
  TWEEN.update()
  renderer.render(scene, camera);
}

scene.add(new THREE.AmbientLight(0xfff, 4));
// 加载模型 gltf/ glb  draco解码器
const loader = new GLTFLoader();

loader.setDRACOLoader(
  new DRACOLoader().setDecoderPath(
    FILE_HOST + "js/three/draco/"
  )
);

import { init } from "@recast-navigation/core";
import * as TRR from "@recast-navigation/three";
let path;
const add_nav_mesh = async () => {
  console.log(init);
  await init();
  loader.load(
    FILE_HOST + "files/model/navmesh02.glb",
    (gltf) => {
      scene.add(gltf.scene);
      let navMesh = TRR.threeToSoloNavMesh(gltf.scene.children);
      console.log(navMesh);
      // navmesh helper
      // const helper = new TRR.NavMeshHelper(navMesh);
      // scene.add(helper);
      gltf.scene.traverse((mesh) => {
        if (mesh.isMesh) {
          mesh.material.side = THREE.DoubleSide;
        }
      });
      let pathResult = compute_route(navMesh.navMesh);
      // 实例化路径
      path = pathResult.path
      draw_path(pathResult.path);
    }
  );
};
import { NavMeshQuery } from "@recast-navigation/core";
import { Vector3 } from "three";
const compute_route = (navMesh) => {
  const navMeshQuery = new NavMeshQuery(navMesh);
  const startPosition = new Vector3(0, 0, 0);
  const endPosition = new Vector3(56, 0, 0);
  let route = navMeshQuery.computePath(startPosition, endPosition);
  return route;
};
import * as Path from "three.path";
import { StaticDrawUsage, Mesh, MeshPhongMaterial, DoubleSide } from "three";
const draw_path = (path) => {
  const points = new Path.PathPointList();
  points.set(path, 0.1, 10, new Vector3(0, 1, 0));
  const geo = new Path.PathGeometry(
    {
      pathPointList: points,
      options: {
        width: 0.1, // default is 0.1
        arrow: true, // default is true
        progress: 1, // default is 1
        side: "both", // "left"/"right"/"both", default is "both"
      },
      usage: StaticDrawUsage, // geometry usage
    },
    false
  );
  var material = new MeshPhongMaterial({
    color: 0x58dede,
    depthWrite: true,
    transparent: true,
    opacity: 0.9,
    side: DoubleSide,
  });
  var mesh = new Mesh(geo, material);
  scene.add(mesh);
  through_path(path)
};
const computeDistance = (vec1, vec2) => {
  return Math.sqrt(
    Math.pow(vec1.x - vec2.x, 2) +
    Math.pow(vec1.y - vec2.y, 2) +
    Math.pow(vec1.z - vec2.z, 2)
  );
};
// 运动路径示意
import * as TWEEN from "@tweenjs/tween.js";
const add_box = () => {
  const geometry = new THREE.BoxGeometry(1, 1, 1); //几何体
  const material = new THREE.MeshBasicMaterial({ color: 0xff0000 }); //材质
  const mesh = new THREE.Mesh(geometry, material); //网格模型
  // mesh.position.set(0, 10, 0); //网格模型位置
  scene.add(mesh); //场景添加网格模型
  return mesh;
};
let i = 0;
const through_path = () => {
  const distance = computeDistance(path[i], path[i + 1]);
  const tween = new TWEEN.Tween(path[i]);
  let box = add_box();
  const time = Math.round(distance / 2);
  tween
    .to(path[i + 1], time * 1000)
    .onUpdate((item) => {
      box.position.copy(item);
    })
    .onComplete(() => {
      if (i + 2 < path.length) {
        i = i + 1;
        through_path();
      }
    });
  tween.start();
};
await add_nav_mesh();
```

完整源码：[GitHub](https://github.com/z2586300277/three-cesium-examples/blob/dev/threeExamples/application/nav.js)

## 小结

- 本文提供 **模型导航** 完整 Three.js 源码与在线 Demo，建议先运行案例再改 uniform/参数做二次实验
- 更多 Three.js 实战案例见 [three-cesium-examples 合集](https://threehub.cn) 与 [GitHub 开源仓库](https://github.com/z2586300277/three-cesium-examples)

