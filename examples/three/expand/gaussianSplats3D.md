---
title: "Three.js 高斯溅射教程"
description: "详解 Three.js 高斯溅射：参考引用自  https://github.com/mkkellogg/GaussianSplats3D，涵盖 Viewer、Scene、Camera 等关键实现，附完整源码与在线 Demo，适合 Three.js 扩展 学习与二次开发。"
head:
  - - meta
    - name: keywords
      content: "Three.js,高斯溅射,WebGL,源码,教程,在线案例"
outline: deep
---

### 高斯溅射 · *gaussianSplats3D* · [▶ 在线运行案例](https://z2586300277.github.io/three-cesium-examples/#/codeMirror?navigation=ThreeJS&classify=expand&id=gaussianSplats3D)
- **案例合集：** [三维可视化功能案例（threehub.cn）](https://threehub.cn)

- **开源仓库github地址：** https://github.com/z2586300277/three-cesium-examples

- **400个案例代码: ** [网盘链接](https://pan.quark.cn/s/201da5c82fec)

![高斯溅射](https://z2586300277.github.io/three-cesium-examples/threeExamples/expand/gaussianSplats3D.webp)

## 你将学到什么

- Scene / Camera / Renderer 标准渲染管线搭建
- 案例完整源码结构与可复用初始化模板

## 效果说明

本案例演示 **高斯溅射** 效果：基于 WebGL 实现「高斯溅射」可视化效果，附完整可运行源码。建议先打开文首在线案例查看动态画面，再对照下方源码逐步理解。

## 核心概念

- 阅读下方完整源码时，建议从 `init` / `load` / `animate` 三条主线入手，再深入 shader 与工具函数。

## 实现步骤

1. 搭建灯光与环境（如有）
2. requestAnimationFrame 循环 update + render

## 代码要点

```js
import * as GaussianSplats3D from '@mkkellogg/gaussian-splats-3d'


/**
 * 参考引用自  https://github.com/mkkellogg/GaussianSplats3D
 * 可结合Three.js 融合 更多玩法参考源文档
 * @type {GaussianSplats3D.Viewer}
 */

// 修改初始化配置
const viewer = new GaussianSplats3D.Viewer({
    'useSharedArrayBuffer': false,
    'useBuiltInControls': true,
    'sharedMemoryForWorkers':false,
    'cameraUp': [0, -1, -0.6],
    'initialCameraPosition': [-1, -4, 6],
    'initialCameraLookAt': [0, 4, 0]
});
//使用私有对象存储带宽较低耐心等待一下 http://app.foxicle.xyz:9000/public-bucket/model/3dgs/garden.ksplat
viewer.addSplatScene(FILE_HOST + 'other/deskFlower.ksplat', {
    'splatAlphaRemovalThreshold': 5,
    'showLoadingUI': true,
    'position': [0, 1, 0],
    'rotation': [0, 0, 0, 1],
    'scale': [1.5, 1.5, 1.5]
}).then(() => {
    viewer.start()
});
```

完整源码：[GitHub](https://github.com/z2586300277/three-cesium-examples/blob/dev/threeExamples/expand/gaussianSplats3D.js)

## 小结

- 本文提供 **高斯溅射** 完整 Three.js 源码与在线 Demo，建议先运行案例再改 uniform/参数做二次实验
- 更多 Three.js 实战案例见 [three-cesium-examples 合集](https://threehub.cn) 与 [GitHub 开源仓库](https://github.com/z2586300277/three-cesium-examples)

