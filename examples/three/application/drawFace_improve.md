---
title: "Three.js 绘制面_内置点教程"
description: "详解 Three.js 绘制面_内置点：支持鼠标拾取、绘制或拖拽交互，涵盖 OrbitControls、Raycaster 等关键实现，附完整源码与在线 Demo，适合 Three.js 应用场景 学习与二次开发。"
head:
  - - meta
    - name: keywords
      content: "Three.js,绘制面_内置点,WebGL,源码,教程,在线案例,OrbitControls,相机控制,Raycaster,射线检测"
outline: deep
---

### 绘制面_内置点 · *Draw Face* · [▶ 在线运行案例](https://z2586300277.github.io/three-cesium-examples/#/codeMirror?navigation=ThreeJS&classify=application&id=drawFace_improve)
- **案例合集：** [三维可视化功能案例（threehub.cn）](https://threehub.cn)

- **开源仓库github地址：** https://github.com/z2586300277/three-cesium-examples

- **400个案例代码: ** [网盘链接](https://pan.quark.cn/s/201da5c82fec)

![绘制面_内置点](https://z2586300277.github.io/three-cesium-examples/threeExamples/application/draw_face_improve.jpg)

## 你将学到什么

- OrbitControls 相机轨道交互
- Raycaster 鼠标拾取与交互
- `requestAnimationFrame` 渲染循环与 `resize` 自适应

## 效果说明

本案例演示 **绘制面_内置点** 效果：支持鼠标拾取、绘制或拖拽交互；核心用到 OrbitControls、Raycaster。建议先打开文首在线案例查看动态画面，再对照下方源码逐步理解。

## 核心概念

- **Scene / Camera / WebGLRenderer** 构成最小渲染闭环；大场景可开 `logarithmicDepthBuffer` 缓解 Z-fighting。
- **OrbitControls** 提供轨道旋转/缩放；开启 `enableDamping` 后需在 animate 中 `controls.update()`。
- **Raycaster** 将屏幕坐标转为射线，与场景求交得到世界坐标，常用于绘制/拾取。

## 实现步骤

1. 搭建灯光与环境（如有）
2. requestAnimationFrame 循环 update + render

## 代码要点

```js
import * as THREE from 'three'
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls.js'

const box = document.getElementById('box')

const scene = new THREE.Scene()

const camera = new THREE.PerspectiveCamera(75, box.clientWidth / box.clientHeight, 0.1, 1000)

camera.position.set(0, 3, 3)

const renderer = new THREE.WebGLRenderer({ antialias: true, alpha: true, logarithmicDepthBuffer: true })

renderer.setSize(box.clientWidth, box.clientHeight)

box.appendChild(renderer.domElement)

const controls = new OrbitControls(camera, renderer.domElement)

controls.enableDamping = true

const directionalLight = new THREE.DirectionalLight(0xffffff, 1)

directionalLight.position.set(0, 20, 0)

scene.add(directionalLight, new THREE.AmbientLight(0xffffff, 1))

/* 增加一个面 */
const plane = new THREE.PlaneGeometry(5, 5)

const material = new THREE.MeshStandardMaterial({ color: 0xffffff })

const planeMesh = new THREE.Mesh(plane, material)

planeMesh.rotation.x -= Math.PI / 2

scene.add(planeMesh)

animate()

function animate() {

  requestAnimationFrame(animate)

  controls.update()

  renderer.render(scene, camera)

}

window.onresize = () => {

  renderer.setSize(box.clientWidth, box.clientHeight)

  camera.aspect = box.clientWidth / box.clientHeight

  camera.updateProjectionMatrix()

}

// 事件
const raycaster = new THREE.Raycaster()

const getPoint = event => {

  const mouse = new THREE.Vector2(

    (event.offsetX / event.target.clientWidth) * 2 - 1,

    -(event.offsetY / event.target.clientHeight) * 2 + 1

  )

  raycaster.setFromCamera(mouse, camera)

  const intersects = raycaster.intersectObjects(scene.children)

  if (intersects.length > 0) return intersects[0].point

}

const setPointBox = point => {

  const box = new THREE.BoxGeometry(0.04, 0.04, 0.04)

  const material = new THREE.MeshStandardMaterial({ color: 0xff0000 })

  const boxMesh = new THREE.Mesh(box, material)

  boxMesh.position.copy(point)

  scene.add(boxMesh)

}

/* 开始绘制 */
const pointList = []; let drawMesh = null; let stop = false

box.addEventListener('contextmenu', () => {

  stop = true

  // const { indexGroup, faceGroup, uvGroup } = multShapeGroup(pointList)

  // if (drawMesh) updateMultShapePlaneGeometry(drawMesh.geometry, faceGroup, indexGroup, uvGroup)

})

// 移动
box.addEventListener('mousemove', (event) => {

  if (stop) return

  const point = getPoint(event)
    
  if (!point || !drawMesh || pointList.length < 2) return


    // update_shape(pointList)

})

box.addEventListener('click', (event) => {

  const point = getPoint(event)

  if (!point || stop) return

  setPointBox(point)

  point.y += 0.001

  pointList.push(point)
  if (pointList.length < 4) return

  if (!drawMesh) {
    draw_shape_v2(pointList)
  }else{
    update_shape(pointList)
  }
})

// 此方法也可使用
const draw_shape = (pointList)=>{
    let v2_p = []
    pointList.map(item=>{
        v2_p.push(new THREE.Vector2(item.x,item.z))
    })
    const shape = new THREE.Shape(v2_p)
    const geometry = new THREE.ShapeGeometry(shape);
    const positions = geometry.getAttribute('position')
    for (let i = 0; i < positions.count; i++) {
        let y = positions.array[i*3+1]
        positions.array[i*3+1] = positions.array[i*3+2] + 0.01
        positions.array[i*3+2] = y
    }
    geometry.attributes.position.needsUpdate = true
    const material = new THREE.MeshBasicMaterial({ color: 0xff0000, side: THREE.DoubleSide });
     drawMesh = new THREE.Mesh(geometry, material);
    scene.add(drawMesh);
}


const draw_shape_v2 = (pointList)=>{
    let v2_p = []
    const shape = new THREE.Shape()
    shape.autoClose = true
    pointList.map(item=>{
        v2_p.push(new THREE.Vector2(item.x,item.z))
        
    })
    shape.moveTo(v2_p[0].x,v2_p[0].y)
    
    v2_p.map((item,index)=>{
        if (index>0) {
            shape.lineTo(item.x,item.y)
        }
    })
    
    const geometry = new THREE.ShapeGeometry(shape);
    const positions = geometry.getAttribute('position')
    console.log(positions);
    
    for (let i = 0; i < positions.count; i++) {
        let y = positions.array[i*3+1]
        positions.array[i*3+1] = positions.array[i*3+2] + 0.01
        positions.array[i*3+2] = y
    }
    geometry.attributes.position.needsUpdate = true
    const material = new THREE.MeshBasicMaterial({ color: 0xff0000, side: THREE.DoubleSide });
     drawMesh = new THREE.Mesh(geometry, material);
    scene.add(drawMesh);
}

const update_shape = (pointList)=>{
    drawMesh.geometry.dispose()
    drawMesh.material = null
    scene.remove(drawMesh)
    draw_shape_v2(pointList)
}
```

完整源码：[GitHub](https://github.com/z2586300277/three-cesium-examples/blob/dev/threeExamples/application/draw_face_improve.js)

## 小结

- 本文提供 **绘制面_内置点** 完整 Three.js 源码与在线 Demo，建议先运行案例再改 uniform/参数做二次实验
- 更多 Three.js 实战案例见 [three-cesium-examples 合集](https://threehub.cn) 与 [GitHub 开源仓库](https://github.com/z2586300277/three-cesium-examples)

