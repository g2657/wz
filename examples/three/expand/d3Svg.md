---
title: "Three.js D3 svg与Three教程"
description: "详解 Three.js D3 svg与Three：基于 WebGL 实现「D3 svg与Three」可视化效果，附完整可运行源码，涵盖 OrbitControls 等关键实现，附完整源码与在线 Demo，适合 Three.js 扩展 学习与二次开发。"
head:
  - - meta
    - name: keywords
      content: "Three.js,D3 svg与Three,WebGL,源码,教程,在线案例,OrbitControls,相机控制"
outline: deep
---

### D3 svg与Three · *D3 SVG Three* · [▶ 在线运行案例](https://z2586300277.github.io/three-cesium-examples/#/codeMirror?navigation=ThreeJS&classify=expand&id=d3Svg)
- **案例合集：** [三维可视化功能案例（threehub.cn）](https://threehub.cn)

- **开源仓库github地址：** https://github.com/z2586300277/three-cesium-examples

- **400个案例代码: ** [网盘链接](https://pan.quark.cn/s/201da5c82fec)

![D3 svg与Three](https://z2586300277.github.io/three-cesium-examples/threeExamples/expand/d3Svg.jpg)

## 你将学到什么

- OrbitControls 相机轨道交互
- `requestAnimationFrame` 渲染循环与 `resize` 自适应

## 效果说明

本案例演示 **D3 svg与Three** 效果：基于 WebGL 实现「D3 svg与Three」可视化效果，附完整可运行源码；核心用到 OrbitControls。建议先打开文首在线案例查看动态画面，再对照下方源码逐步理解。

## 核心概念
- **Scene / Camera / WebGLRenderer** 构成最小渲染闭环；大场景可开 `logarithmicDepthBuffer` 缓解 Z-fighting。
- **OrbitControls** 提供轨道旋转/缩放；开启 `enableDamping` 后需在 animate 中 `controls.update()`。
- 阅读下方完整源码时，建议从 `init` / `load` / `animate` 三条主线入手，再深入 shader 与工具函数。

## 实现步骤

1. 搭建 Scene、PerspectiveCamera、WebGLRenderer，挂载 canvas 并处理 `resize`
2. 创建 OrbitControls（及 Raycaster 等交互控件，若源码包含）
3. 在 `requestAnimationFrame` 循环中更新状态并 render（Cesium 为 `viewer.render` 或自动渲染）

## 代码要点

```js
import * as THREE from 'three'
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls.js'
import { SVGLoader } from 'three/addons/loaders/SVGLoader.js';
import * as d3 from "d3";

const box = document.getElementById('box')

const scene = new THREE.Scene()

const camera = new THREE.PerspectiveCamera(75, box.clientWidth / box.clientHeight, 0.1, 100000)

camera.position.set(50, 50, 50)

const renderer = new THREE.WebGLRenderer({ antialias: true, alpha: true, logarithmicDepthBuffer: true })

renderer.setSize(box.clientWidth, box.clientHeight)

box.appendChild(renderer.domElement)

new OrbitControls(camera, renderer.domElement)

scene.add(new THREE.AxesHelper(100), new THREE.GridHelper(100, 10))

animate()

function animate() {

  requestAnimationFrame(animate)

  renderer.render(scene, camera)

}

window.onresize = () => {

  renderer.setSize(box.clientWidth, box.clientHeight)

  camera.aspect = box.clientWidth / box.clientHeight

  camera.updateProjectionMatrix()

}

const data = await d3.json(FILE_HOST + "other/volcano.json");
const n = data.width;
const m = data.height;
const width = 928;
const height = Math.round(m / n * width);
const path = d3.geoPath().projection(d3.geoIdentity().scale(width / n));
const contours = d3.contours().size([n, m]);
const color = d3.scaleSequential(d3.interpolateTurbo).domain(d3.extent(data.values)).nice();

const svg = d3.create("svg")
  .attr("width", width)
  .attr("height", height)
  .attr("viewBox", [0, 0, width, height])
  .attr("style", "max-width: 100%; height: auto;");

svg.append("g")
  .attr("stroke", "black")
  .selectAll()
  .data(color.ticks(20))
  .join("path")
  .attr("d", d => path(contours.contour(data.values, d)))
  .attr("fill", color);

const div = document.createElement('div');
div.style.position = 'absolute';
div.style.width = '300px';
div.appendChild(svg.node());
document.body.appendChild(div);

const svgString = new XMLSerializer().serializeToString(svg.node());

const svgData = new Blob([svgString], { type: 'image/svg+xml' });
const url = URL.createObjectURL(svgData);

const loader = new SVGLoader();

loader.load(
  url,
  function (data) {
    const paths = data.paths;
    const group = new THREE.Group();

    // 过滤和清理路径数据
    const filteredPaths = paths.filter(path =>
      path.subPaths && path.subPaths.length > 0
    );

    filteredPaths.forEach((path, index) => {

      let pathColor = 0x00ff00; // 默认绿色

      if (path.userData && path.userData.style) {
        const fillMatch = path.userData.style.fill;
        if (fillMatch && fillMatch !== 'none') pathColor = fillMatch;
      }

      const material = new THREE.MeshBasicMaterial({
        color: pathColor,
        side: THREE.DoubleSide,
        transparent: true,
        opacity: 0.8
      });

      const shapes = SVGLoader.createShapes(path);

      shapes.forEach(shape => {
        const geometry = new THREE.ShapeGeometry(shape);
        geometry.computeBoundingBox();
        const center = geometry.boundingBox.getCenter(new THREE.Vector3());
        geometry.translate(-center.x, -center.y, 0);
        const mesh = new THREE.Mesh(geometry, material);
        mesh.position.set(0, 0, index * 0.1); // 轻微分层避免z-fighting
        mesh.scale.setScalar(0.1); // 适当缩放
        group.add(mesh);
      });
    });

    const box = new THREE.Box3().setFromObject(group);
    const center = box.getCenter(new THREE.Vector3());
    group.position.sub(center);
    scene.add(group);
  }
)
```

完整源码：[GitHub](https://github.com/z2586300277/three-cesium-examples/blob/dev/threeExamples/expand/d3Svg.js)

## 小结

- 本文提供 **D3 svg与Three** 完整 Three.js 源码与在线 Demo，建议先运行案例再改 uniform/参数做二次实验
- 更多 Three.js 实战案例见 [three-cesium-examples 合集](https://threehub.cn) 与 [GitHub 开源仓库](https://github.com/z2586300277/three-cesium-examples)

