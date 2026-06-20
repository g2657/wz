---
title: "Three.js 文件分片（打包 zip）教程"
description: "详解 Three.js 文件分片（打包 zip）：基于 WebGL 实现「文件分片（打包 zip）」可视化效果，附完整可运行源码，涵盖 OrbitControls、glTF/Draco 等关键实现，附完整源码与在线 Demo，适合 Three.js 基础 学习与二次开发。"
head:
  - - meta
    - name: keywords
      content: "Three.js,文件分片（打包 zip）,WebGL,源码,教程,在线案例,OrbitControls,相机控制,glTF,模型加载"
outline: deep
---

### 文件分片（打包 zip） · *File Chunks* · [▶ 在线运行案例](https://z2586300277.github.io/three-cesium-examples/#/codeMirror?navigation=ThreeJS&classify=basic&id=fileChunks)
- **案例合集：** [三维可视化功能案例（threehub.cn）](https://threehub.cn)

- **开源仓库github地址：** https://github.com/z2586300277/three-cesium-examples

- **400个案例代码: ** [网盘链接](https://pan.quark.cn/s/201da5c82fec)

## 你将学到什么

- 大文件 **slice 分片** 打包 zip 上传/下载思路
- JSZip 合并 **ArrayBuffer** 还原完整 glb
- `URL.createObjectURL` 临时加载

## 效果说明

选择本地 `.glb` → 均分 5 片写入 zip → 自动下载 `_chunks.zip` → 再解压合并 → **GLTFLoader** 加载进场景。

## 核心概念

```js
for (let i = 0; i * chunkSize < file.size; i++) {
    zip.file(`${i}.chunk`, file.slice(i * chunkSize, (i + 1) * chunkSize));
}
// 还原：按序 async arraybuffer → Uint8Array 拼接 → Blob → loader.load(url)
```

适用于 **超大模型分片上传** 后端再合并；本案例为前端演示全流程。

## 代码要点

```js
import * as THREE from 'three'
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls.js'
import { GLTFLoader } from 'three/examples/jsm/loaders/GLTFLoader.js'
import { DRACOLoader } from 'three/examples/jsm/loaders/DRACOLoader.js'
import JSZip from 'jszip'

const box = document.getElementById('box')

const scene = new THREE.Scene()

const camera = new THREE.PerspectiveCamera(50, box.clientWidth / box.clientHeight, 0.1, 10000000)

camera.position.set(10, 10, 10)

const renderer = new THREE.WebGLRenderer({ antialias: true, alpha: true, logarithmicDepthBuffer: true })

renderer.setSize(box.clientWidth, box.clientHeight)

box.appendChild(renderer.domElement)

const controls = new OrbitControls(camera, renderer.domElement)

controls.enableDamping = true

scene.add(new THREE.AxesHelper(500), new THREE.AmbientLight(0xffffff, 2))

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

// 文件地址
const urls = [0, 1, 2, 3, 4, 5].map(k => (FILE_HOST + 'files/sky/skyBox0/' + (k + 1) + '.png'));

const textureCube = new THREE.CubeTextureLoader().load(urls);

scene.background = textureCube;

// 创建一个文件上传的输入框
const input = document.createElement('input')
input.type = 'file'
input.accept = '.glb'
Object.assign(input.style, { position: 'absolute', top: '30px', left: '100px', zIndex: 9999 })
document.body.appendChild(input)

const zip = new JSZip();

input.onchange = async (e) => {

  const file = e.target.files[0]

  const chunkSize = Math.ceil(file.size / 5) // 每个分片的大小，这里分成5份

  for (let i = 0; i * chunkSize < file.size; i++) {

    zip.file(`${i}.chunk`, file.slice(i * chunkSize, (i + 1) * chunkSize))

  }

  const zipBlob = await zip.generateAsync({ type: 'blob' })

  downloadBlob(zipBlob, file.name.split('.').slice(0, -1).join('.') + '_chunks.zip')

  // 演示如何加载分片压缩包
  loadZipChunksModel(zipBlob)

}

function downloadBlob(blob, filename) {
  const link = document.createElement('a');
  link.href = URL.createObjectURL(blob);
  link.download = filename;
  document.body.appendChild(link);
  link.click();
  document.body.removeChild(link);
  URL.revokeObjectURL(link.href);
}

function loadZipChunksModel(zipBlob) {

  const loader = new GLTFLoader().setDRACOLoader(new DRACOLoader().setDecoderPath(FILE_HOST + 'js/three/draco/'))

  JSZip.loadAsync(zipBlob).then(async (unzipped) => {

    const chunkFiles = Object.keys(unzipped.files).filter(name => name.endsWith('.chunk')).sort((a, b) => parseInt(a) - parseInt(b))

    const chunks = []
    for (const chunkFile of chunkFiles) {
      const chunkData = await unzipped.files[chunkFile].async('arraybuffer')
      chunks.push(chunkData)
    }

    const completeArray = new Uint8Array(chunks.reduce((acc, chunk) => acc + chunk.byteLength, 0))
    let offset = 0
    for (const chunk of chunks) {
      completeArray.set(new Uint8Array(chunk), offset)
      offset += chunk.byteLength
    }

    const blob = new Blob([completeArray], { type: 'application/octet-stream' })
    const url = URL.createObjectURL(blob)

    loader.load(url, (gltf) => {
      gltf.scene.traverse((child) => {
        if (child?.material) child.material.envMap = textureCube
      })
      scene.add(gltf.scene)
      URL.revokeObjectURL(url)
    })

  })


}
```

完整源码：[GitHub](https://github.com/z2586300277/three-cesium-examples/blob/dev/threeExamples/basic/fileChunks.js)

## 小结

- 本文提供 **文件分片（打包 zip）** 完整 Three.js 源码与在线 Demo，建议先运行案例再改 uniform/参数做二次实验
- 更多 Three.js 实战案例见 [three-cesium-examples 合集](https://threehub.cn) 与 [GitHub 开源仓库](https://github.com/z2586300277/three-cesium-examples)

