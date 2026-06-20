---
title: "Cesium 内网百度教程"
description: "详解 Cesium.js 内网百度：基于 WebGL 实现「内网百度」可视化效果，附完整可运行源码，涵盖 Viewer、Scene、Camera 等关键实现，附完整源码与在线 Demo，适合 Cesium 离线地图 学习与二次开发。"
head:
  - - meta
    - name: keywords
      content: "Cesium.js,内网百度,WebGL,源码,教程,在线案例,Cesium"
outline: deep
---

### 内网百度 · *Intranet Baidu* · [▶ 在线运行案例](https://z2586300277.github.io/three-cesium-examples/#/codeMirror?navigation=CesiumJS&classify=offline&id=baiDu)
- **案例合集：** [三维可视化功能案例（threehub.cn）](https://threehub.cn)

- **开源仓库github地址：** https://github.com/z2586300277/three-cesium-examples

- **400个案例代码: ** [网盘链接](https://pan.quark.cn/s/201da5c82fec)

![内网百度](https://z2586300277.github.io/three-cesium-examples/cesiumExamples/offline/baidu.jpg)

## 你将学到什么

- Scene / Camera / Renderer 标准渲染管线搭建
- 案例完整源码结构与可复用初始化模板

## 效果说明

本案例演示 **内网百度** 效果：基于 WebGL 实现「内网百度」可视化效果，附完整可运行源码。建议先打开文首在线案例查看动态画面，再对照下方源码逐步理解。

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

    baseLayer: false, // 不显示默认图层

    fullscreenButton: false,//是否显示全屏按钮，右下角全屏选择按钮

    timeline: false,//是否显示时间轴    

    infoBox: false,//是否显示信息框   

})

// 这里 https://github.com/z2586300277/3d-file-server 是我存放离线地图瓦片资源的仓库 

// 瓦片下载 - 可通过多种方式 例如 望远网 地图资源下载 

// 这里我只下载了 3 - 5 级的瓦片    

/* 百度 影像服务 */
class BaiduImageryProvider {
    constructor(options) {
        this._errorEvent = new Cesium.Event();
        this._tileWidth = 256;
        this._tileHeight = 256;
        this._maximumLevel = 18;
        this._minimumLevel = 1;
        this._tilingScheme = new Cesium.WebMercatorTilingScheme({
            rectangleSouthwestInMeters: new Cesium.Cartesian2(-33554054, -33746824),
            rectangleNortheastInMeters: new Cesium.Cartesian2(33554054, 33746824)
        });
        this._rectangle = this._tilingScheme.rectangle;
        this._resource = Cesium.Resource.createIfNeeded(options.url);
    }

    get url() { return this._resource.url; }
    get proxy() { return this._resource.proxy; }
    get tileWidth() { return this._tileWidth; }
    get tileHeight() { return this._tileHeight; }
    get maximumLevel() { return this._maximumLevel; }
    get minimumLevel() { return this._minimumLevel; }
    get tilingScheme() { return this._tilingScheme; }
    get tileDiscardPolicy() { return this._tileDiscardPolicy; }
    get rectangle() { return this._rectangle; }
    get errorEvent() { return this._errorEvent; }
    get ready() { return this._resource; }
    get readyPromise() { return this._readyPromise; }
    get credit() { return this._credit; }

    requestImage(x, y, level) {
        let url = this.url
            .replace("{x}", x - this._tilingScheme.getNumberOfXTilesAtLevel(level) / 2)
            .replace("{y}", this._tilingScheme.getNumberOfYTilesAtLevel(level) / 2 - y - 1)
            .replace("{z}", level)
            .replace("{s}", Math.floor(10 * Math.random()));
        return Cesium.ImageryProvider.loadImage(this, url);
    }
}

viewer.imageryLayers.addImageryProvider(

    new BaiduImageryProvider({

        url: FILE_HOST + 'map/Baidu/tiles/{z}/{x}/{y}.jpg'

    })

)
```

完整源码：[GitHub](https://github.com/z2586300277/three-cesium-examples/blob/dev/cesiumExamples/offline/baidu.js)

## 小结
- 本文提供 **内网百度** 完整 Cesium.js 源码与在线 Demo，建议先运行案例再改 uniform/参数做二次实验
- 更多 Cesium.js 实战案例见 [three-cesium-examples 合集](https://threehub.cn) 与 [GitHub 开源仓库](https://github.com/z2586300277/three-cesium-examples)

