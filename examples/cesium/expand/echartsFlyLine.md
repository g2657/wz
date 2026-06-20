---
title: "Cesium echarts飞线教程"
description: "详解 Cesium.js echarts飞线：ECharts 图表与 Three.js 场景同屏联动展示，涵盖 ECharts 等关键实现，附完整源码与在线 Demo，适合 Cesium 扩展 学习与二次开发。"
head:
  - - meta
    - name: keywords
      content: "Cesium.js,echarts飞线,WebGL,源码,教程,在线案例,Cesium,ECharts,数据可视化"
outline: deep
---

### echarts飞线 · *EchartsFlyLine* · [▶ 在线运行案例](https://z2586300277.github.io/three-cesium-examples/#/codeMirror?navigation=CesiumJS&classify=expand&id=echartsFlyLine)
- **案例合集：** [三维可视化功能案例（threehub.cn）](https://threehub.cn)

- **开源仓库github地址：** https://github.com/z2586300277/three-cesium-examples

- **400个案例代码: ** [网盘链接](https://pan.quark.cn/s/201da5c82fec)

![echarts飞线](https://z2586300277.github.io/three-cesium-examples/cesiumExamples/expand/echartsFlyLine.jpg)

## 你将学到什么

- ECharts 与 WebGL 场景联动
- 监听窗口 `resize` 同步更新 camera 与 renderer

## 效果说明

本案例演示 **echarts飞线** 效果：ECharts 图表与 Three.js 场景同屏联动展示；核心用到 ECharts。建议先打开文首在线案例查看动态画面，再对照下方源码逐步理解。

## 核心概念

- **Viewer** 封装地球、相机、图层与 clock；可关闭 animation/timeline 精简 UI。
- **ScreenSpaceEventHandler** 监听点击；`scene.pick` 取 Entity，`pickPosition` 取地表坐标。
- **flyTo** 带动画定位；**trackedEntity** 第三人称跟随实体。
- 每帧 **worldToWindowCoordinates** 投影，translate 定位 DOM。

## 实现步骤
1. 创建 Viewer，配置地形/影像（若案例需要）并设置初始相机
2. 在 `requestAnimationFrame` 循环中更新状态并 render（Cesium 为 `viewer.render` 或自动渲染）

## 代码要点

```js
/* 注 echarts 版本使用 4.9.0  请自行引入  此处我为 src 引入 */
import * as Cesium from 'cesium'

const DOM = document.getElementById('box')

const viewer = new Cesium.Viewer(DOM, {

    animation: false,//是否创建动画小器件，左下角仪表    

    baseLayerPicker: false,//是否显示图层选择器，右上角图层选择按钮

    baseLayer: Cesium.ImageryLayer.fromProviderAsync(Cesium.ArcGisMapServerImageryProvider.fromUrl(GLOBAL_CONFIG.getLayerUrl())),

    fullscreenButton: false,//是否显示全屏按钮，右下角全屏选择按钮

    timeline: false,//是否显示时间轴    

    infoBox: false,//是否显示信息框   

})

viewer._cesiumWidget._creditContainer.style.display = "none"

// 视角设置北京
viewer.camera.flyTo({ destination: Cesium.Cartesian3.fromDegrees(116.4551, 40.2539, 10000000) })

class RegisterCoordinateSystem {
    static dimensions = ['lng', 'lat']
    constructor(glMap) {
        this._GLMap = glMap
        this._mapOffset = [0, 0]
        this.dimensions = ['lng', 'lat']
    }
    setMapOffset(mapOffset) {
        this._mapOffset = mapOffset
    }
    getBMap() {
        return this._GLMap
    }
    fixLat(lat) {
        return lat >= 90 ? 89.99999999999999 : lat <= -90 ? -89.99999999999999 : lat
    }
    dataToPoint(coords) {
        let lonlat = [99999, 99999]
        coords[1] = this.fixLat(coords[1])
        let position = Cesium.Cartesian3.fromDegrees(coords[0], coords[1])
        if (!position) return lonlat
        let coordinates = this._GLMap.cartesianToCanvasCoordinates(position)
        if (!coordinates) return lonlat
        if (this._GLMap.mode === Cesium.SceneMode.SCENE3D) {
            if (Cesium.Cartesian3.angleBetween(this._GLMap.camera.position, position) > Cesium.Math.toRadians(75)) return !1
        }
        return [coordinates.x - this._mapOffset[0], coordinates.y - this._mapOffset[1]]
    }
    pointToData(pt) {
        var mapOffset = this._mapOffset
        pt = this._bmap.project([pt[0] + mapOffset[0], pt[1] + mapOffset[1]])
        return [pt.lng, pt.lat]
    }
    getViewRect() {
        let api = this._api
        return new echarts.graphic.BoundingRect(0, 0, api.getWidth(), api.getHeight())
    }
    getRoamTransform() {
        return echarts.matrix.create()
    }
    static create(echartModel, api) {
        this._api = api
        let registerCoordinateSystem
        echartModel.eachComponent('GLMap', function (seriesModel) {
            let painter = api.getZr().painter
            if (painter) {
                let glMap = echarts.glMap
                registerCoordinateSystem = new RegisterCoordinateSystem(glMap, api)
                registerCoordinateSystem.setMapOffset(seriesModel.__mapOffset || [0, 0])
                seriesModel.coordinateSystem = registerCoordinateSystem
            }
        })
        echartModel.eachSeries(function (series) {
            'GLMap' === series.get('coordinateSystem') && (series.coordinateSystem = registerCoordinateSystem)
        })
    }
}

const mockClickChart = (event, chart) => {
    const evmousedown = new MouseEvent('mousedown', { bubbles: true, cancelable: true });
    const evmouseup = new MouseEvent('mouseup', { bubbles: true, cancelable: true });
    const evmouseclick = new MouseEvent('click', { bubbles: true, cancelable: true });
    for (const key in event) {
        try {
            Object.defineProperty(evmousedown, key, { value: event[key] });
            Object.defineProperty(evmouseup, key, { value: event[key] });
            Object.defineProperty(evmouseclick, key, { value: event[key] });
        } catch (err) { /* event 对象中部分属性是只读，忽略即可 */ }
    }

    // 事件触发的容器，即不是 #app 也不是 canvas，而是中间这个 div
    const container = chart._dom.firstElementChild;
    container.dispatchEvent(evmousedown);
    container.dispatchEvent(evmouseup);
    container.dispatchEvent(evmouseclick);
}

class EchartsLayer {

    constructor(map, options) {
        this._map = map;
        this._overlay = this._createChartOverlay();
        if (options) this._registerMap();
        this._overlay.setOption(options || {});
    }
    _registerMap() {
        if (!this._isRegistered) {
            echarts.registerCoordinateSystem('GLMap', RegisterCoordinateSystem);
            echarts.registerAction({ type: 'GLMapRoam', event: 'GLMapRoam', update: 'updateLayout' }, function (t, e) { });
            echarts.extendComponentModel({
                type: 'GLMap',
                getBMap() {
                    return this.__GLMap;
                },
                defaultOption: { roam: false },
            });
            echarts.extendComponentView({
                type: 'GLMap',
                init(t, e) {
                    this.api = e;
                    echarts.glMap.postRender.addEventListener(this.moveHandler, this);
                },
                moveHandler(t, e) {
                    this.api.dispatchAction({ type: 'GLMapRoam' });
                },
                render(t, e, i) { },
                dispose(t) {
                    echarts.glMap.postRender.removeEventListener(this.moveHandler, this);
                },
            });
            this._isRegistered = true;
        }
    }
    _createChartOverlay() {
        var scene = this._map.scene;
        scene.canvas.setAttribute('tabIndex', 0);
        const ele = document.createElement('div');
        ele.style.position = 'absolute';
        ele.style.top = '0px';
        ele.style.left = '0px';
        ele.style.width = scene.canvas.width + 'px';
        ele.style.height = scene.canvas.height + 'px';
        ele.style.pointerEvents = 'none';
        ele.setAttribute('id', 'echarts');
        ele.setAttribute('class', 'echartMap');
        this._map.container.appendChild(ele);
        this._echartsContainer = ele;
        echarts.glMap = scene;
        this._chart = echarts.init(ele);
        const handler = new Cesium.ScreenSpaceEventHandler(scene.canvas);
        handler.setInputAction(click => mockClickChart(event, this._chart), Cesium.ScreenSpaceEventType.LEFT_CLICK);
        return this._chart;
    }
    dispose() {
        if (this._echartsContainer) {
            this._map.container.removeChild(this._echartsContainer);
            this._echartsContainer = null;
        }
        if (this._overlay) {
            this._overlay.dispose();
            this._overlay = null;
        }
    }
    updateOverlay(t) {
        if (this._overlay) {
            this._overlay.setOption(t);
        }
    }
    getMap() {
        return this._map;
    }
    getOverlay() {
        return this._overlay;
    }
    show() {
        if (this._echartsContainer) {
            this._echartsContainer.style.visibility = 'visible';
        }
    }
    hide() {
        if (this._echartsContainer) {
            this._echartsContainer.style.visibility = 'hidden';
        }
    }
    remove() {
        this._chart.clear();
        if (this._echartsContainer.parentNode) {
            this._echartsContainer.parentNode.removeChild(this._echartsContainer);
        }
        this._map = undefined;
    }
    resize() {
        const me = this;
        const container = me._map.container;
        me._echartsContainer.style.width = container.clientWidth + 'px';
        me._echartsContainer.style.height = container.clientHeight + 'px';
        me._chart.resize();
    }
}

const echartsLayer = new EchartsLayer(viewer, {

    animation: false,

    GLMap: {},

    series: [
        {
            type: 'lines',
            coordinateSystem: 'GLMap',
            zlevel: 2,
            effect: { show: true, period: 6, trailLength: 0.1, symbol: 'arrow', symbolSize: 5 },
            lineStyle: { normal: { color: '#60ff44', width: 1, opacity: 0.4, curveness: 0.2 } },
            data: [{ fromName: '北京', toName: '无锡', coords: [[116.4551, 40.2539], [120.3442, 31.5527],], }, { fromName: '上海', toName: '无锡', coords: [[121.4648, 31.2891], [120.3442, 31.5527],], }, { fromName: '广州', toName: '无锡', coords: [[113.5107, 23.2196], [120.3442, 31.5527],], }, { fromName: '大连', toName: '无锡', coords: [[122.2229, 39.4409], [120.3442, 31.5527],], }, { fromName: '青岛', toName: '无锡', coords: [[120.4651, 36.3373], [120.3442, 31.5527],], }, { fromName: '石家庄', toName: '无锡', coords: [[114.4995, 38.1006], [120.3442, 31.5527],], }, { fromName: '南昌', toName: '无锡', coords: [[116.0046, 28.6633], [120.3442, 31.5527],], }, { fromName: '合肥', toName: '无锡', coords: [[117.29, 32.0581], [120.3442, 31.5527],], }, { fromName: '呼和浩特', toName: '无锡', coords: [[111.4124, 40.4901], [120.3442, 31.5527],], }, { fromName: '宿州', toName: '无锡', coords: [[117.5535, 33.7775], [120.3442, 31.5527],], }, { fromName: '曲阜', toName: '无锡', coords: [[117.323, 35.8926], [120.3442, 31.5527],], }, { fromName: '杭州', toName: '无锡', coords: [[119.5313, 29.8773], [120.3442, 31.5527],], }, { fromName: '武汉', toName: '无锡', coords: [[114.3896, 30.6628], [120.3442, 31.5527],], }, { fromName: '深圳', toName: '无锡', coords: [[114.5435, 22.5439], [120.3442, 31.5527],], }, { fromName: '珠海', toName: '无锡', coords: [[113.7305, 22.1155], [120.3442, 31.5527],], }, { fromName: '福州', toName: '无锡', coords: [[119.4543, 25.9222], [120.3442, 31.5527],], }, { fromName: '西安', toName: '无锡', coords: [[109.1162, 34.2004], [120.3442, 31.5527],], }, { fromName: '赣州', toName: '无锡', coords: [[116.0046, 25.6633], [120.3442, 31.5527],], },],
        },
        {
            type: 'effectScatter',
            coordinateSystem: 'GLMap',
            zlevel: 2,
            rippleEffect: { brushType: 'stroke' },
            label: { normal: { show: true, position: 'right', formatter: '{b}' } },
            itemStyle: { normal: { color: '#60ff44' } },
            data: [{ name: '北京', value: [116.4551, 40.2539, 100] }, { name: '上海', value: [121.4648, 31.2891, 30] }, { name: '广州', value: [113.5107, 23.2196, 20] }, { name: '大连', value: [122.2229, 39.4409, 10] }, { name: '青岛', value: [120.4651, 36.3373, 20] }, { name: '石家庄', value: [114.4995, 38.1006, 20] }, { name: '南昌', value: [116.0046, 28.6633, 10] }, { name: '合肥', value: [117.29, 32.0581, 30] }, { name: '呼和浩特', value: [111.4124, 40.4901, 10] }, { name: '宿州', value: [117.5535, 33.7775, 10] }, { name: '曲阜', value: [117.323, 35.8926, 10] }, { name: '杭州', value: [119.5313, 29.8773, 10] }, { name: '武汉', value: [114.3896, 30.6628, 10] }, { name: '深圳', value: [114.5435, 22.5439, 10] }, { name: '珠海', value: [113.7305, 22.1155, 10] }, { name: '福州', value: [119.4543, 25.9222, 20] }, { name: '西安', value: [109.1162, 34.2004, 60] }, { name: '赣州', value: [116.0046, 25.6633, 10] },],
        },
    ],

})

echartsLayer._chart.on('click', params => console.log(params))

window.addEventListener('resize', () => echartsLayer.resize())
```

完整源码：[GitHub](https://github.com/z2586300277/three-cesium-examples/blob/dev/cesiumExamples/expand/echartsFlyLine.js)

## 小结

- 本文提供 **echarts飞线** 完整 Cesium.js 源码与在线 Demo，建议先运行案例再改 uniform/参数做二次实验
- 更多 Cesium.js 实战案例见 [three-cesium-examples 合集](https://threehub.cn) 与 [GitHub 开源仓库](https://github.com/z2586300277/three-cesium-examples)

