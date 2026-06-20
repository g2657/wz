---
title: "Cesium 百度图层教程"
description: "详解 Cesium.js 百度图层：基于 WebGL 实现「百度图层」可视化效果，附完整可运行源码，涵盖 Viewer、Scene、Camera 等关键实现，附完整源码与在线 Demo，适合 Cesium 图层 学习与二次开发。"
head:
  - - meta
    - name: keywords
      content: "Cesium.js,百度图层,WebGL,源码,教程,在线案例,Cesium"
outline: deep
---

### 百度图层 · *Baidu Layer* · [▶ 在线运行案例](https://z2586300277.github.io/three-cesium-examples/#/codeMirror?navigation=CesiumJS&classify=layer&id=baiduLayer)
- **案例合集：** [三维可视化功能案例（threehub.cn）](https://threehub.cn)

- **开源仓库github地址：** https://github.com/z2586300277/three-cesium-examples

- **400个案例代码: ** [网盘链接](https://pan.quark.cn/s/201da5c82fec)

![百度图层](https://z2586300277.github.io/three-cesium-examples/cesiumExamples/layer/baiduLayer.jpg)

## 你将学到什么

- Scene / Camera / Renderer 标准渲染管线搭建
- 案例完整源码结构与可复用初始化模板

## 效果说明

本案例演示 **百度图层** 效果：基于 WebGL 实现「百度图层」可视化效果，附完整可运行源码。建议先打开文首在线案例查看动态画面，再对照下方源码逐步理解。

## 核心概念
- **Viewer** 聚合 Scene、Camera、Clock 与渲染循环，是 Cesium 应用入口。
- 阅读下方完整源码时，建议从 `init` / `load` / `animate` 三条主线入手，再深入 shader 与工具函数。

## 实现步骤
1. 创建 Viewer，配置地形/影像（若案例需要）并设置初始相机
2. 在 `requestAnimationFrame` 循环中更新状态并 render（Cesium 为 `viewer.render` 或自动渲染）

## 代码要点

```js
import * as Cesium from 'cesium'

const box = document.getElementById('box')

const viewer = new Cesium.Viewer(box, {

    animation: false,//是否创建动画小器件，左下角仪表    

    baseLayerPicker: false,//是否显示图层选择器，右上角图层选择按钮

    baseLayer: false,

    fullscreenButton: false,//是否显示全屏按钮，右下角全屏选择按钮

    geocoder: false,//是否显示geocoder小器件，右上角查询按钮    

    homeButton: false,//是否显示Home按钮，右上角home按钮 

    sceneMode: Cesium.SceneMode.SCENE3D,//初始场景模式

    sceneModePicker: false,//是否显示3D/2D选择器，右上角按钮 

    navigationHelpButton: false,//是否显示右上角的帮助按钮  

    selectionIndicator: false,//是否显示选取指示器组件   

    timeline: false,//是否显示时间轴    

    infoBox: false,//是否显示信息框   

    scene3DOnly: true,//如果设置为true，则所有几何图形以3D模式绘制以节约GPU资源  

    orderIndependentTranslucency: false, //是否启用无序透明

    contextOptions: { webgl: { alpha: true } },

    skyBox: new Cesium.SkyBox({ show: false })

})

viewer.scene.sun.show = false

viewer.scene.moon.show = false

viewer.scene.skyBox.show = false

viewer.scene.backgroundColor = new Cesium.Color(0.0, 0.0, 0.0, 0.0)

viewer._cesiumWidget._creditContainer.style.display = "none"

/* 百度 影像服务 */
class BaiduImageryProvider {

    constructor(options) {

        // 创建错误事件对象
        this._errorEvent = new Cesium.Event()

        // 定义瓦片宽度和高度
        this._tileWidth = 256

        this._tileHeight = 256

        // 定义最大和最小级别
        this._maximumLevel = 18

        this._minimumLevel = 1

        // 定义瓦片范围的南西角和东北角坐标
        let southwestInMeters = new Cesium.Cartesian2(-33554054, -33746824)

        let northeastInMeters = new Cesium.Cartesian2(33554054, 33746824)

        // 创建 WebMercatorTilingScheme 对象
        this._tilingScheme = new Cesium.WebMercatorTilingScheme({

            rectangleSouthwestInMeters: southwestInMeters,

            rectangleNortheastInMeters: northeastInMeters

        })

        // 获取瓦片范围
        this._rectangle = this._tilingScheme.rectangle

        // 创建资源对象
        this._resource = Cesium.Resource.createIfNeeded(options.url)

        // 设置其他属性的初始值
        this._tileDiscardPolicy = undefined

        this._credit = undefined

        this._readyPromise = undefined

    }

    // 定义属性访问器
    get url() {

        return this._resource.url

    }

    get proxy() {

        return this._resource.proxy

    }

    get tileWidth() {

        if (!this.ready) throw new Cesium.DeveloperError('tileWidth must not be called before the imagery provider is ready.')

        return this._tileWidth
    }

    get tileHeight() {

        if (!this.ready) throw new Cesium.DeveloperError('tileHeight must not be called before the imagery provider is ready.')

        return this._tileHeight

    }

    get maximumLevel() {

        if (!this.ready) throw new Cesium.DeveloperError('maximumLevel must not be called before the imagery provider is ready.')

        return this._maximumLevel

    }

    get minimumLevel() {

        if (!this.ready) throw new Cesium.DeveloperError('minimumLevel must not be called before the imagery provider is ready.')

        return this._minimumLevel

    }

    get tilingScheme() {

        if (!this.ready) throw new Cesium.DeveloperError('tilingScheme must not be called before the imagery provider is ready.')

        return this._tilingScheme

    }

    get tileDiscardPolicy() {

        if (!this.ready) throw new Cesium.DeveloperError('tileDiscardPolicy must not be called before the imagery provider is ready.')

        return this._tileDiscardPolicy

    }

    get rectangle() {

        if (!this.ready) throw new Cesium.DeveloperError('rectangle must not be called before the imagery provider is ready.')

        return this._rectangle

    }

    get errorEvent() {

        return this._errorEvent

    }

    get ready() {

        return this._resource

    }

    get readyPromise() {

        return this._readyPromise

    }

    get credit() {

        if (!this.ready) {

            throw new Cesium.DeveloperError('credit must not be called before the imagery provider is ready.')

        }

        return this._credit

    }

    // 请求影像数据
    requestImage(x, y, level, request) {

        let xTileCount = this._tilingScheme.getNumberOfXTilesAtLevel(level)

        let yTileCount = this._tilingScheme.getNumberOfYTilesAtLevel(level)

        // 构建请求 URL
        let url = this.url

            .replace("{x}", x - xTileCount / 2)

            .replace("{y}", yTileCount / 2 - y - 1)

            .replace("{z}", level)

            .replace("{s}", Math.floor(10 * Math.random()))

        // 加载影像数据
        return Cesium.ImageryProvider.loadImage(this, url)

    }

}

viewer.imageryLayers.addImageryProvider(

    new BaiduImageryProvider({

        url: 'https://maponline0.bdimg.com/tile/?qt=vtile&x={x}&y={y}&z={z}&styles=pl&scaler=1&udt=20210709'

    })

)
```

完整源码：[GitHub](https://github.com/z2586300277/three-cesium-examples/blob/dev/cesiumExamples/layer/baiduLayer.js)

## 小结
- 本文提供 **百度图层** 完整 Cesium.js 源码与在线 Demo，建议先运行案例再改 uniform/参数做二次实验
- 更多 Cesium.js 实战案例见 [three-cesium-examples 合集](https://threehub.cn) 与 [GitHub 开源仓库](https://github.com/z2586300277/three-cesium-examples)

