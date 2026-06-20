---
title: "Cesium 点聚合 Supercluster教程"
description: "详解 Cesium.js 点聚合 Supercluster：基于 WebGL 实现「点聚合 Supercluster」可视化效果，附完整可运行源码，涵盖 Viewer、Scene、Camera 等关键实现，附完整源码与在线 Demo，适合 Cesium 基础 学习与二次开发。"
head:
  - - meta
    - name: keywords
      content: "Cesium.js,点聚合 Supercluster,WebGL,源码,教程,在线案例,Cesium"
outline: deep
---

### 点聚合 Supercluster · *Point Cluster (Supercluster)* · [▶ 在线运行案例](https://z2586300277.github.io/three-cesium-examples/#/codeMirror?navigation=CesiumJS&classify=basic&id=multPointCluster)
- **案例合集：** [三维可视化功能案例（threehub.cn）](https://threehub.cn)

- **开源仓库github地址：** https://github.com/z2586300277/three-cesium-examples

- **400个案例代码: ** [网盘链接](https://pan.quark.cn/s/201da5c82fec)

![点聚合 Supercluster](https://z2586300277.github.io/three-cesium-examples/cesiumExamples/basic/multPointCluster.jpg)

## 你将学到什么

- 引入 **Supercluster** 做屏幕空间聚合
- `getClusters(bbox, zoom)` 按视口与层级取簇
- **camera.changed** 监听视角变化重算
- 聚合点展开时取 **最近子点** 代表位置

## 效果说明

加载全国城市 JSON，缩放地图时相近城市合并为单图标；放大后拆散为独立 billboard。

## 核心概念

### Supercluster 流程

```
GeoJSON Feature[] → supercluster.load()
                         ↓
              getClusters([west,south,east,north], zoom)
                         ↓
              BillboardCollection.add / removeAll
```

```js
const supercluster = new Supercluster({
    radius: 40,    // 聚合半径（像素级算法参数）
    extent: 512,
    minZoom: 0,
    maxZoom: 16,
});
supercluster.load(points);
```

### 视口与层级

```js
const bbox = viewer.camera.computeViewRectangle();
const bounds = [west, south, east, north].map(i => Cesium.Math.toDegrees(i));

// 用当前渲染瓦片 level 近似 zoom
const level = viewer.scene.globe._surface._tilesToRender[0]._level;
const clusters = supercluster.getClusters(bounds, level);
```

### 聚合点代表位置

若 `cluster.properties.cluster` 为真，从 `getLeaves` 里找 **离簇中心最近** 的原始点，避免图标漂在空处。

## 实现步骤

1. fetch 城市 JSON，转为 `{ type:'Feature', geometry:{coordinates} }`
2. `setClusterCollection(viewer, points, callback)` 封装聚合
3. 初次 `setBillboards(clusters)`
4. `camera.changed` → `billboards.removeAll()` → 重新 getClusters → setBillboards

## 代码要点

```js
/* 注 Supercluster 依赖请自行引入  此处我为 src 引入 */
import * as Cesium from 'cesium'

const DOM = document.getElementById('box')

const viewer = new Cesium.Viewer(DOM, {

    animation: false,//是否创建动画小器件，左下角仪表    

    baseLayerPicker: false,//是否显示图层选择器，右上角图层选择按钮

    baseLayer: Cesium.ImageryLayer.fromProviderAsync(Cesium.ArcGisMapServerImageryProvider.fromUrl('https://server.arcgisonline.com/arcgis/rest/services/World_Imagery/MapServer')),

    fullscreenButton: false,//是否显示全屏按钮，右下角全屏选择按钮

    timeline: false,//是否显示时间轴    

    infoBox: false,//是否显示信息框   

})

viewer._cesiumWidget._creditContainer.style.display = "none"

viewer.camera.setView({ destination: Cesium.Cartesian3.fromDegrees(116.3974, 39.9093, 18000000) }) // 设置视角

const citys = await fetch('https://z2586300277.github.io/three-editor/dist/files/other/city.json').then(res => res.json()) // 获取城市数据

const points = Object.values(citys).map((val, k) => ({ type: 'Feature', pid: k + '-' + val[0] + '-' + val[1], geometry: { coordinates: val } }))

// 建立聚合
setClusterCollection(viewer, points, (billboards, data) => {

    const [longitude, latitude] = data.geometry.coordinates

    const { pid } = data

    billboards.add({

        position: Cesium.Cartesian3.fromDegrees(longitude, latitude),

        image: 'https://z2586300277.github.io/three-editor/dist/site.png', // 你的图片路径

        scale: 0.05,

        eyeOffset: new Cesium.Cartesian3(0, 0, 150), // 偏移高度

        id: pid

    })

})

/* 聚合方法 */
function setClusterCollection(viewer, points, callback = () => { }, options = {}) {

    // 创建聚合
    const supercluster = new Supercluster({ radius: 40, extent: 512, minZoom: 0, maxZoom: 16, ...options }) // 密集程度 radius, 切片大小 extent, 最小层级 minZoom, 最大层级 maxZoom

    supercluster.load(points)

    // 初始化聚合
    const clusters = supercluster.getClusters([-180, -85, 180, 85], 2)

    // 获取当前视角的边界
    const getBounds = () => {

        const bbox = viewer.camera.computeViewRectangle()

        return [bbox.west, bbox.south, bbox.east, bbox.north].map(i => Cesium.Math.toDegrees(i))  // minx, miny, maxx, maxy westLng, southLat, eastLng, northLat

    }

    // 获取当前视角的层级
    const getLevel = () => {

        var tileRender = viewer.scene.globe._surface._tilesToRender;

        if (tileRender && tileRender.length > 0) {

            return tileRender[0]._level

        }

    }

    // 创建billboard集合对象
    const billboards = viewer.scene.primitives.add(new Cesium.BillboardCollection())

    const setBillboards = arr => arr.forEach((cluster) => {

        let returnCluster = cluster

        // 判断聚合点
        if (cluster?.properties?.cluster) {

            const clusterId = cluster.properties.cluster_id

            const clusterCoordinates = cluster.geometry.coordinates

            const leaves = supercluster.getLeaves(clusterId, Infinity, 0)

            // 初始化最小距离和最接近的点
            let minDistance = Infinity

            let closestPoint = null

            // 遍历所有原始点
            for (const leaf of leaves) {

                // 计算原始点与聚合点的距离
                const leafCoordinates = leaf.geometry.coordinates

                const distance = Cesium.Cartesian3.distance(

                    Cesium.Cartesian3.fromDegrees(...clusterCoordinates),

                    Cesium.Cartesian3.fromDegrees(...leafCoordinates)

                )

                // 如果这个距离小于当前的最小距离，更新最小距离和最接近的点
                if (distance < minDistance) {

                    minDistance = distance

                    closestPoint = leaf

                }

            }

            returnCluster = closestPoint

        }

        callback(billboards, returnCluster)

    })

    setBillboards(clusters)

    // 动态更新聚合
    viewer.camera.changed.addEventListener(function () {

        const level = getLevel()

        if (!level) return

        const clusters = supercluster.getClusters(getBounds(), level);

        billboards.removeAll()

        setBillboards(clusters)

    })

    return supercluster

}
```

完整源码：[GitHub](https://github.com/z2586300277/three-cesium-examples/blob/dev/cesiumExamples/basic/multPointCluster.js)

## 小结

- 本文提供 **点聚合 Supercluster** 完整 Cesium.js 源码与在线 Demo，建议先运行案例再改 uniform/参数做二次实验
- 更多 Cesium.js 实战案例见 [three-cesium-examples 合集](https://threehub.cn) 与 [GitHub 开源仓库](https://github.com/z2586300277/three-cesium-examples)

