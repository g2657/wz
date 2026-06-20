---
title: "Three.js 精灵文字教程"
description: "详解 Three.js 精灵文字：用 Canvas 2D 绘制内容并实时映射为 Three.js 纹理，涵盖 OrbitControls、Canvas 等关键实现，附完整源码与在线 Demo，适合 Three.js 应用场景 学习与二次开发。"
head:
  - - meta
    - name: keywords
      content: "Three.js,精灵文字,WebGL,源码,教程,在线案例,OrbitControls,相机控制,Canvas纹理,动态贴图"
outline: deep
---
### 精灵文字 · *Sprite Text* · [▶ 在线运行案例](https://z2586300277.github.io/three-cesium-examples/#/codeMirror?navigation=ThreeJS&classify=application&id=spriteText)
- **案例合集：** [三维可视化功能案例（threehub.cn）](https://threehub.cn)

- **开源仓库github地址：** https://github.com/z2586300277/three-cesium-examples

- **400个案例代码: ** [网盘链接](https://pan.quark.cn/s/201da5c82fec)

![精灵文字](https://z2586300277.github.io/three-cesium-examples/threeExamples/application/spriteText.jpg)

## 你将学到什么

- OrbitControls 相机轨道交互
- Canvas 动态纹理贴图
- `requestAnimationFrame` 渲染循环与 `resize` 自适应

## 效果说明

本案例演示 **精灵文字** 效果：用 Canvas 2D 绘制内容并实时映射为 Three.js 纹理；核心用到 OrbitControls、Canvas。建议先打开文首在线案例查看动态画面，再对照下方源码逐步理解。

## 核心概念

- **OrbitControls** 轨道旋转缩放；开 `enableDamping` 时每帧需 `controls.update()`。

## 实现步骤

1. 搭建 Scene / Camera / Renderer 与 OrbitControls
2. rAF 循环中 update 并 render

## 代码要点

```js
import * as THREE from 'three'
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls.js'

const box = document.getElementById('box')

const scene = new THREE.Scene()

const camera = new THREE.PerspectiveCamera(75, box.clientWidth / box.clientHeight, 0.1, 1000)

camera.position.set(0, 10, 10)

const renderer = new THREE.WebGLRenderer()

renderer.setSize(box.clientWidth, box.clientHeight)

box.appendChild(renderer.domElement)

new OrbitControls(camera, renderer.domElement)

scene.add(new THREE.AmbientLight(0xffffff, 3), new THREE.AxesHelper(1000))

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

const citys = await fetch('https://z2586300277.github.io/three-editor/dist/files/other/city.json').then(res => res.json()) // 获取城市数据

const updateCanvasText = createCanvasText({ dpr: 1.4 }) // 创建canvas

const getColor = () => '#' + Math.floor(Math.random() * 0xffffff).toString(16).padStart(6, '0') // 随机颜色

for (const key in citys) {

  const canvas = updateCanvasText({ text: key, color: getColor() })

  const texture = new THREE.TextureLoader().load(canvas.toDataURL())

  const material = new THREE.SpriteMaterial({ map: texture })

  const sprite = new THREE.Sprite(material)

  sprite.scale.set(canvas.width / canvas.height, 1, 1)

  // 设置随机位置
  sprite.position.set(
    Math.random() * 20 - 10,
    Math.random() * 20 - 10,
    Math.random() * 20 - 10
  )

  scene.add(sprite)

}

// 创建canvas文字方法
function createCanvasText(params) {

  const defaultParams = { dpr: 1, maxWidth: 100, fontSize: 20, color: 'white', fontFamily: 'serif', align: 'center', border: false, ...params } // 默认参数

  const { dpr, border, maxWidth, fontSize, align } = defaultParams

  const devicePixelRatio = window.devicePixelRatio * dpr

  // 准备 cnvas
  const canvas = document.createElement('canvas')

  canvas.width = maxWidth * devicePixelRatio

  canvas.height = fontSize * devicePixelRatio

  // 获取 2d 上下文
  const ctx = canvas.getContext('2d')

  ctx.imageSmoothingQuality = 'high'

  ctx.scale(devicePixelRatio, devicePixelRatio)

  // 创建边框
  function createBorder() {

    ctx.strokeStyle = '#fff'

    // 创建宽度为10px的边框
    ctx.lineWidth = 1 * devicePixelRatio;

    ctx.strokeRect(

      ctx.lineWidth / 2,

      ctx.lineWidth / 2,

      canvas.width / devicePixelRatio - ctx.lineWidth,

      canvas.height / devicePixelRatio - ctx.lineWidth

    )

  }

  // 创建文字
  const createText = ({ text, color, fontSize, fontFamily }) => {

    // 参数设定
    ctx.fillStyle = color || defaultParams.color

    ctx.font = fontSize || defaultParams.fontSize + 'px ' + fontFamily || defaultParams.fontFamily

    // 文本长度计算
    let textMaxNum = 0

    let totalWidth = 0

    for (let i = 0; i < text.length; i++) {

      const metrics = ctx.measureText(text[i])

      totalWidth += metrics.width;

      if (totalWidth > maxWidth) break

      textMaxNum++

    }

    text = text.slice(0, textMaxNum)

    // 文字 绘制
    const metrics = ctx.measureText(text) // 文本尺寸

    const actualHeight = metrics.actualBoundingBoxAscent + metrics.actualBoundingBoxDescent // 实际文字高度

    const textFillHeight = (canvas.height / devicePixelRatio - actualHeight) / 2 + metrics.actualBoundingBoxAscent

    let textLeftOffset = 0

    if (align === 'center') textLeftOffset = (canvas.width / devicePixelRatio - metrics.width) / 2

    ctx.fillText(text, textLeftOffset, textFillHeight, canvas.width / devicePixelRatio)

  }

  return (parameters) => {

    ctx.clearRect(0, 0, canvas.width, canvas.height)  // 清空  canvas 文字 

    if (border) createBorder() // 创建边框

    createText(parameters) // 创建文字

    return canvas

  }

}
```

完整源码：[GitHub](https://github.com/z2586300277/three-cesium-examples/blob/dev/threeExamples/application/spriteText.js)

## 小结

- 本文提供 **精灵文字** 完整 Three.js 源码与在线 Demo，建议先运行案例再改 uniform/参数做二次实验
- 更多 Three.js 实战案例见 [three-cesium-examples 合集](https://threehub.cn) 与 [GitHub 开源仓库](https://github.com/z2586300277/three-cesium-examples)

