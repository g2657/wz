---
title: "Three.js Echarts结合教程"
description: "详解 Three.js Echarts结合：ECharts 图表与 Three.js 场景同屏联动展示，涵盖 OrbitControls、glTF/Draco、ECharts 等关键实现，附完整源码与在线 Demo，适合 Three.js 扩展 学习与二次开发。"
head:
  - - meta
    - name: keywords
      content: "Three.js,Echarts结合,WebGL,源码,教程,在线案例,OrbitControls,相机控制,glTF,模型加载,ECharts,数据可视化"
outline: deep
---
### Echarts结合 · *Combine Echarts* · [▶ 在线运行案例](https://z2586300277.github.io/three-cesium-examples/#/codeMirror?navigation=ThreeJS&classify=expand&id=combineEcharts)
- **案例合集：** [三维可视化功能案例（threehub.cn）](https://threehub.cn)

- **开源仓库github地址：** https://github.com/z2586300277/three-cesium-examples

- **400个案例代码: ** [网盘链接](https://pan.quark.cn/s/201da5c82fec)

![Echarts结合](https://z2586300277.github.io/three-cesium-examples/threeExamples/expand/combineEcharts.jpg)

## 你将学到什么

- glTF/FBX/OBJ 外部模型加载
- 相机交互控制器
- CSS2D/3D 标签渲染
- ECharts 与三维融合
- requestAnimationFrame 渲染循环

## 效果说明

本案例演示 **Echarts结合** 效果：ECharts 图表与 Three.js 场景同屏联动展示；核心用到 OrbitControls、glTF/Draco、ECharts。建议先打开文首在线案例查看动态画面，再对照下方源码逐步理解。

## 核心概念

- **Loader** 异步加载模型；glTF 返回 `gltf.scene`，加载后注意 `scale` 与坐标系。Draco 需配置 `DRACOLoader`。

- **OrbitControls** 轨道旋转缩放；开 `enableDamping` 时每帧需 `controls.update()`。

- DOM 元素叠加在 3D 坐标上，适合信息面板（注意与 WebGL 深度关系）。

- 二维图表/飞线与 Cesium/Three 场景叠加或纹理映射。

## 实现步骤

1. 搭建 Scene / Camera / Renderer 与 OrbitControls
2. Loader 异步加载模型/纹理资源
3. rAF 循环中 update 并 render

## 代码要点

```js
import * as THREE from 'three'
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls.js'
import { CSS3DRenderer, CSS3DObject } from 'three/examples/jsm/renderers/CSS3DRenderer.js'
import { GLTFLoader } from 'three/examples/jsm/loaders/GLTFLoader.js'
import * as echarts from 'echarts'

const DOM = document.getElementById('box')

const scene = new THREE.Scene()

const camera = new THREE.PerspectiveCamera(75, DOM.clientWidth / DOM.clientHeight, 0.1, 10000)

camera.position.set(50, 90, 300)

const renderer = new THREE.WebGLRenderer({ antialias: true, alpha: true, logarithmicDepthBuffer: true })

renderer.setSize(DOM.clientWidth, DOM.clientHeight)

DOM.appendChild(renderer.domElement)

scene.add(new THREE.AmbientLight(0xffffff, 3))

new OrbitControls(camera, renderer.domElement)

const css3DRender = setCss3DRenderer(DOM)

new GLTFLoader().load(FILE_HOST + "files/model/Fox.glb", gltf => scene.add(gltf.scene))

animate()

function animate() {

    requestAnimationFrame(animate)

    renderer.render(scene, camera)

    css3DRender.render(scene, camera)
}

window.onresize = () => {

    renderer.setSize(box.clientWidth, box.clientHeight)

    camera.aspect = box.clientWidth / box.clientHeight

    camera.updateProjectionMatrix()

    css3DRender.resize()

}

/* css3d 渲染 */
function setCss3DRenderer(DOM) {

    const css3DRender = new CSS3DRenderer()

    css3DRender.resize = () => {

        css3DRender.setSize(DOM.clientWidth, DOM.clientHeight)

        css3DRender.domElement.style.zIndex = 0

        css3DRender.domElement.style.position = 'relative'

        css3DRender.domElement.style.top = -DOM.clientHeight + 'px'

        css3DRender.domElement.style.height = DOM.clientHeight + 'px'

        css3DRender.domElement.style.width = DOM.clientWidth + 'px'

        css3DRender.domElement.style.pointerEvents = 'none'

    }

    css3DRender.resize()

    DOM.appendChild(css3DRender.domElement)

    return css3DRender

}

/* 图表 ---------------------------------------------------------------------- */

const container = document.createElement("div")
container.style.width = "300px"
container.style.height = "200px"
const myChart = echarts.init(container)

myChart.setOption({
    graphic: {
      elements: [
        {
          type: 'text',
          left: 'center',
          top: 'center',
          style: {
            text: 'Echarts',
            fontSize: 80,
            fontWeight: 'bold',
            lineDash: [0, 200],
            lineDashOffset: 0,
            fill: 'transparent',
            stroke: '#fff',
            lineWidth: 1
          },
          keyframeAnimation: {
            duration: 3000,
            loop: true,
            keyframes: [
              {
                percent: 0.7,
                style: {
                  fill: 'transparent',
                  lineDashOffset: 200,
                  lineDash: [200, 0]
                }
              },
              {
                // Stop for a while.
                percent: 0.8,
                style: {
                  fill: 'transparent'
                }
              },
              {
                percent: 1,
                style: {
                  fill: 'black'
                }
              }
            ]
          }
        }
      ]
    }
  })

const css3DObject = new CSS3DObject(container)
css3DObject.position.set(0, 130, 0)
scene.add(css3DObject)

const container2 = document.createElement("div")
container2.style.width = "300px"
container2.style.height = "300px"
const myChart2 = echarts.init(container2)

myChart2.setOption({
    xAxis: {
        type: 'category',
        boundaryGap: false,
        data: ['Mon', 'Tue', 'Wed', 'Thu', 'Fri', 'Sat', 'Sun']
    },
    yAxis: {
        type: 'value'
    },
    series: [
        {
            data: [820, 932, 901, 934, 1290, 1330, 1320],
            type: 'line',
            areaStyle: {}
        }
    ]
})

const css3DObject2 = new CSS3DObject(container2)
css3DObject2.position.set(0, -80, 0)
scene.add(css3DObject2)
```

完整源码：[GitHub](https://github.com/z2586300277/three-cesium-examples/blob/dev/threeExamples/expand/combineEcharts.js)

## 小结

- 本文提供 **Echarts结合** 完整 Three.js 源码与在线 Demo，建议先运行案例再改 uniform/参数做二次实验
- 更多 Three.js 实战案例见 [three-cesium-examples 合集](https://threehub.cn) 与 [GitHub 开源仓库](https://github.com/z2586300277/three-cesium-examples)

