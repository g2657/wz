---
title: "Cesium Canvas 文字点教程"
description: "详解 Cesium.js Canvas 文字点：用 Canvas 2D 绘制内容并实时映射为 Three.js 纹理，涵盖 Canvas 等关键实现，附完整源码与在线 Demo，适合 Cesium 基础 学习与二次开发。"
head:
  - - meta
    - name: keywords
      content: "Cesium.js,Canvas 文字点,WebGL,源码,教程,在线案例,Cesium,Canvas纹理,动态贴图"
outline: deep
---

### Canvas 文字点 · *Canvas Text Billboards* · [▶ 在线运行案例](https://z2586300277.github.io/three-cesium-examples/#/codeMirror?navigation=CesiumJS&classify=basic&id=multText)
- **案例合集：** [三维可视化功能案例（threehub.cn）](https://threehub.cn)

- **开源仓库github地址：** https://github.com/z2586300277/three-cesium-examples

- **400个案例代码: ** [网盘链接](https://pan.quark.cn/s/201da5c82fec)

![Canvas 文字点](https://z2586300277.github.io/three-cesium-examples/cesiumExamples/basic/multText.jpg)

## 你将学到什么

- 用 **Canvas 2D** 动态生成文字纹理
- **devicePixelRatio × dpr** 高清屏清晰文字
- 纹理作为 **Billboard.image** 批量贴到城市坐标

## 效果说明

加载 city.json，每个城市名渲染成彩色 canvas，再以 billboard 形式钉在对应经纬度。

## 核心概念

### createCanvasText 工厂

```js
function createCanvasText(params) {
    const { dpr, fontSize, maxWidth } = { dpr: 1, maxWidth: 100, fontSize: 20, ...params };
    const ratio = window.devicePixelRatio * dpr;
    canvas.width = maxWidth * ratio;
    canvas.height = fontSize * ratio;
    ctx.scale(ratio, ratio);
    // 返回 (opts) => canvas.toDataURL() 或 canvas 本身
}
```

每次调用 `updateCanvasText({ text: key, color })` 重绘并返回 image 源。

### 为何不用 Label？

| Label | Canvas Billboard |
|-------|------------------|
| 字体样式有限 | 任意 CSS 字体、描边、背景 |
| 每 Entity 开销 | Collection 合批 |
| 适合少量 | 适合上百城市名 |

## 实现步骤

1. fetch city.json → `{ 城市名: [lon, lat] }`
2. `createCanvasText({ dpr: 1.4 })` 得 update 函数
3. `BillboardCollection` 循环 add，`image: updateCanvasText({ text, color })`
4. `flyTo` 中国上空总览

## 代码要点

```js
import * as Cesium from 'cesium'

const box = document.getElementById('box')

const viewer = new Cesium.Viewer(box, {

    animation: false,//是否创建动画小器件，左下角仪表    

    baseLayerPicker: false,//是否显示图层选择器，右上角图层选择按钮

    baseLayer: Cesium.ImageryLayer.fromProviderAsync(Cesium.ArcGisMapServerImageryProvider.fromUrl('https://server.arcgisonline.com/arcgis/rest/services/World_Imagery/MapServer')),

    fullscreenButton: false,//是否显示全屏按钮，右下角全屏选择按钮

    timeline: false,//是否显示时间轴    

    infoBox: false,//是否显示信息框   

})

viewer.camera.flyTo({ destination: Cesium.Cartesian3.fromDegrees(116.46, 39.92, 8000000) }) // 设置相机位置

const citys = await fetch('https://z2586300277.github.io/three-editor/dist/files/other/city.json').then(res => res.json()) // 获取城市数据

const updateCanvasText = createCanvasText({ dpr: 1.4 }) // 创建canvas

const billboards = new Cesium.BillboardCollection() // 创建合集

viewer.scene.primitives.add(billboards) // 添加图层

const getColor = () => '#' + Math.floor(Math.random() * 0xffffff).toString(16).padStart(6, '0') // 随机颜色

for (const key in citys) {

    const [longitude, latitude] = citys[key]

    billboards.add({

        position: Cesium.Cartesian3.fromDegrees(longitude, latitude),

        image: updateCanvasText({ text: key, color: getColor() }),

        scale: 0.5
        
    })
    
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

        return canvas.toDataURL()

    }

}
```

完整源码：[GitHub](https://github.com/z2586300277/three-cesium-examples/blob/dev/cesiumExamples/basic/multText.js)

## 小结

- 本文提供 **Canvas 文字点** 完整 Cesium.js 源码与在线 Demo，建议先运行案例再改 uniform/参数做二次实验
- 更多 Cesium.js 实战案例见 [three-cesium-examples 合集](https://threehub.cn) 与 [GitHub 开源仓库](https://github.com/z2586300277/three-cesium-examples)

