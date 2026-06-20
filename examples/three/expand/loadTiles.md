---
title: "Three.js 加载3dtiles教程"
description: "详解 Three.js 加载3dtiles：基于 WebGL 实现「加载3dtiles」可视化效果，附完整可运行源码，涵盖 OrbitControls、glTF/Draco、3D 等关键实现，附完整源码与在线 Demo，适合 Three.js 扩展 学习与二次开发。"
head:
  - - meta
    - name: keywords
      content: "Three.js,加载3dtiles,WebGL,源码,教程,在线案例,OrbitControls,相机控制,glTF,模型加载,3D Tiles"
outline: deep
---
### 加载3dtiles · *Load Tiles* · [▶ 在线运行案例](https://z2586300277.github.io/three-cesium-examples/#/codeMirror?navigation=ThreeJS&classify=expand&id=loadTiles)
- **案例合集：** [三维可视化功能案例（threehub.cn）](https://threehub.cn)

- **开源仓库github地址：** https://github.com/z2586300277/three-cesium-examples

- **400个案例代码: ** [网盘链接](https://pan.quark.cn/s/201da5c82fec)

![加载3dtiles](https://z2586300277.github.io/three-cesium-examples/threeExamples/expand/loadTiles.jpg)

## 你将学到什么

- OrbitControls 相机轨道交互
- glTF/Draco 模型加载与优化
- 3D Tiles 流式 LOD 场景
- `requestAnimationFrame` 渲染循环与 `resize` 自适应

## 效果说明

本案例演示 **加载3dtiles** 效果：基于 WebGL 实现「加载3dtiles」可视化效果，附完整可运行源码；核心用到 OrbitControls、glTF/Draco、3D。建议先打开文首在线案例查看动态画面，再对照下方源码逐步理解。

## 核心概念

- **Loader** 异步加载模型；glTF 返回 `gltf.scene`，加载后注意 `scale` 与坐标系。Draco 需配置 `DRACOLoader`。

- **OrbitControls** 轨道旋转缩放；开 `enableDamping` 时每帧需 `controls.update()`。

## 实现步骤

1. 搭建 Scene / Camera / Renderer 与 OrbitControls
2. Loader 异步加载模型/纹理资源
3. rAF 循环中 update 并 render

## 代码要点

```js
import * as THREE from 'three'
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls.js'
import { TilesRenderer } from '3d-tiles-renderer'

const box = document.getElementById('box')

const scene = new THREE.Scene()

const camera = new THREE.PerspectiveCamera(75, box.clientWidth / box.clientHeight, 0.1, 1000)

camera.position.set(0, 30, 30)

const renderer = new THREE.WebGLRenderer({ antialias: true, alpha: true, logarithmicDepthBuffer: true })

renderer.setSize(box.clientWidth, box.clientHeight)

box.appendChild(renderer.domElement)

new OrbitControls(camera, renderer.domElement)

scene.add(new THREE.AxesHelper(1000))

// 加载3d tiles
const tilesRenderer = new TilesRenderer(FILE_HOST + '3dtiles/test/tileset.json')

tilesRenderer.setCamera(camera)

tilesRenderer.setResolutionFromRenderer(camera, renderer)

const model = new THREE.Group().add(tilesRenderer.group)

scene.add(model)

const box3 = new THREE.Box3()

tilesRenderer.addEventListener('load-tile-set', () => {

    if (tilesRenderer.getBoundingBox(box3)) {

        box3.getCenter(tilesRenderer.group.position)

        tilesRenderer.group.position.multiplyScalar(-1)

    }

})

animate()

function animate() {

    requestAnimationFrame(animate)

    tilesRenderer.update()

    renderer.render(scene, camera)

}

// tilesRenderer.errorTarget = 1 // 设置错误阈值，默认值为0.5，范围0~1，值越小越严格

// https://blog.csdn.net/m0_73348873/article/details/151783069

/* function initTiles() {
    tilesRenderer = new TilesRenderer("3dtiles路径/tileset.json");
    const gltfLoader = new GLTFLoader();
 
    // Draco
    const dracoLoader = new DRACOLoader();
    dracoLoader.setDecoderPath("https://unpkg.com/three@0.180.0/examples/jsm/libs/draco/");
    gltfLoader.setDRACOLoader(dracoLoader);
 
    // KTX2
    const ktx2Loader = new KTX2Loader();
    ktx2Loader.setTranscoderPath("https://unpkg.com/three@0.180.0/examples/jsm/libs/basis/");
    ktx2Loader.detectSupport(renderer);
    gltfLoader.setKTX2Loader(ktx2Loader);
 
    tilesRenderer.manager.addHandler(/\.(gltf|glb)$/g, gltfLoader);
    tilesRenderer.setCamera(camera);
    tilesRenderer.setResolutionFromRenderer(camera, renderer);
 
    // 更新矩阵并设置相机位置
    let loadedTileSetHandled = false;
    tilesRenderer.addEventListener("load-tile-set", () => {
        if (loadedTileSetHandled) return;
        loadedTileSetHandled = true;
 
        const sphere = new THREE.Sphere();
        tilesRenderer.getBoundingSphere(sphere);
        const center = sphere.center.clone(); // 获取包围球中心
        const radius = sphere.radius; // 获取包围球半径
        controls.target.copy(center); // 把控制器目标设为包围球中心
        const offset = new THREE.Vector3(radius * 2, radius, 0); // 给相机一个偏移
        camera.position.copy(center).add(offset); // 设置相机位置
 
        const m = (tilesRenderer as any).root.transform; // 获取原始矩阵
        const rotationMat3 = new THREE.Matrix3().set(m[0], m[1], m[2], m[4], m[5], m[6], m[8], m[9], m[10]); // 取出旋转部分
        rotationMat3.transpose(); // 逆旋转
        const rotationMat4 = new THREE.Matrix4().setFromMatrix3(rotationMat3); // 转回Matrix4以便应用
        const rotX90 = new THREE.Matrix4().makeRotationX((90 * Math.PI) / 180); // x轴旋转90度矩阵
        rotationMat4.multiply(rotX90); // 合并矩阵（由z轴向上坐标系 转为 y轴向上坐标系）
        const translationMatrix1 = new THREE.Matrix4().makeTranslation(center.x, center.y, center.z); // T(center)
        const translationMatrix2 = new THREE.Matrix4().makeTranslation(-center.x, -center.y, -center.z); // T(-center)
        const finalMatrix = new THREE.Matrix4().multiplyMatrices(translationMatrix1, rotationMat4).multiply(translationMatrix2); // 最终矩阵 = T(center) * R⁻¹ * T(-center)
 
        tilesRenderer.group.matrix.copy(finalMatrix); // 设置矩阵
        tilesRenderer.group.matrixAutoUpdate = false; // 禁止自动更新矩阵
        tilesRenderer.group.updateMatrixWorld(true); // 更新矩阵
    });
 
    scene.add(tilesRenderer.group); // 添加到场景
}
 */
```

完整源码：[GitHub](https://github.com/z2586300277/three-cesium-examples/blob/dev/threeExamples/expand/loadTiles.js)

## 小结

- 本文提供 **加载3dtiles** 完整 Three.js 源码与在线 Demo，建议先运行案例再改 uniform/参数做二次实验
- 更多 Three.js 实战案例见 [three-cesium-examples 合集](https://threehub.cn) 与 [GitHub 开源仓库](https://github.com/z2586300277/three-cesium-examples)

