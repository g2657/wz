---
title: "Cesium 渐变行政区教程"
description: "详解 Cesium.js 渐变行政区：自定义材质类型名称，涵盖 GeoJSON 等关键实现，附完整源码与在线 Demo，适合 Cesium 应用 学习与二次开发。"
head:
  - - meta
    - name: keywords
      content: "Cesium.js,渐变行政区,WebGL,源码,教程,在线案例,Cesium,GeoJSON,矢量数据"
outline: deep
---

### 渐变行政区 · *Gradient Area* · [▶ 在线运行案例](https://z2586300277.github.io/three-cesium-examples/#/codeMirror?navigation=CesiumJS&classify=applyExample&id=gradienGeojsonFace)
- **案例合集：** [三维可视化功能案例（threehub.cn）](https://threehub.cn)

- **开源仓库github地址：** https://github.com/z2586300277/three-cesium-examples

- **400个案例代码: ** [网盘链接](https://pan.quark.cn/s/201da5c82fec)

![渐变行政区](https://z2586300277.github.io/three-cesium-examples/cesiumExamples/expand/gradienGeojsonFace.jpg)

## 你将学到什么

- GeoJSON 矢量面/线/点加载

## 效果说明

本案例演示 **渐变行政区** 效果：自定义材质类型名称，GeoJSON 行政区/路线数据可视化贴地展示；核心用到 GeoJSON。建议先打开文首在线案例查看动态画面，再对照下方源码逐步理解。

## 核心概念
- **Viewer** 聚合 Scene、Camera、Clock 与渲染循环，是 Cesium 应用入口。
- 阅读下方完整源码时，建议从 `init` / `load` / `animate` 三条主线入手，再深入 shader 与工具函数。

## 实现步骤
1. 创建 Viewer，配置地形/影像（若案例需要）并设置初始相机
2. 在 `requestAnimationFrame` 循环中更新状态并 render（Cesium 为 `viewer.render` 或自动渲染）

## 代码要点

```js
import * as Cesium from "cesium";

/**
 * 自定义材质类型名称
 * @const {string}
 */
const MATERIAL_TYPE = "Custom";

/**
 * 自定义材质属性类
 * @class
 */
class CustomMaterialProperty {
  /**
   * @param {Object=} options 配置项
   */
  constructor(options = {}) {
    this._definitionChanged = new Cesium.Event();
    this._color = undefined;
    this._colorSubscription = undefined;

    this.color = options.color || Cesium.Color.RED;
    this.duration = options.duration || 2000;
    this._time = performance.now();
  }

  /**
   * @return {boolean}
   */
  get isConstant() {
    return false;
  }

  /**
   * @return {Cesium.Event}
   */
  get definitionChanged() {
    return this._definitionChanged;
  }

  /**
   * @return {string}
   */
  getType() {
    return MATERIAL_TYPE;
  }

  /**
   * @param {Cesium.JulianDate} time
   * @param {Object=} result
   * @return {Object}
   */
  getValue(time, result = {}) {
    result.color = Cesium.Property.getValueOrUndefined(this.color, time);
    result.time =
      ((performance.now() - this._time) % this.duration) / this.duration;
    return result;
  }

  /**
   * @param {CustomMaterialProperty} other
   * @return {boolean}
   */
  equals(other) {
    return (
      this === other ||
      (other instanceof CustomMaterialProperty && this._color === other._color)
    );
  }
}

// 定义颜色属性
Object.defineProperty(
  CustomMaterialProperty.prototype,
  "color",
  Cesium.createPropertyDescriptor("color")
);

// 注册自定义材质
Cesium.Material._materialCache.addMaterial(MATERIAL_TYPE, {
  fabric: {
    type: MATERIAL_TYPE,
    uniforms: {
      color: new Cesium.Color(1, 1, 0, 1),
      time: 1,
      spacing: 40,
      width: 1,
    },
    source: `
      uniform vec4 color;
      czm_material czm_getMaterial(czm_materialInput materialInput) {
        czm_material material = czm_getDefaultMaterial(materialInput);
        vec2 st = materialInput.st;
        float alpha = distance(st, vec2(.5));
        material.alpha = color.a * alpha * 1.5;
        material.diffuse = color.rgb * 1.3;
        return material;
      }`,
  },
  translucent: () => true,
});

/**
 * 初始化 viewer
 * @type {Cesium.Viewer}
 */
const viewer = new Cesium.Viewer("box", {
  animation: false,
  baseLayerPicker: false,
  baseLayer: Cesium.ImageryLayer.fromProviderAsync(
    Cesium.ArcGisMapServerImageryProvider.fromUrl(
      "https://server.arcgisonline.com/arcgis/rest/services/World_Imagery/MapServer"
    )
  ),
  fullscreenButton: false,
  timeline: false,
  infoBox: false,
});

/**
 * 预定义的颜色配置
 * @type {Array<Array<number>>}
 */
const COLOR_CONFIGS = [
  [15, 176, 255],
  [18, 76, 154],
  [64, 196, 228],
  [66, 178, 190],
  [51, 176, 204],
  [140, 183, 229],
  [0, 244, 188],
  [19, 159, 240],
];

/**
 * 添加材质到地图
 * @async
 */
async function addMaterial() {
  const dataSource = await Cesium.GeoJsonDataSource.load(
    "https://z2586300277.github.io/three-editor/dist/files/font/guangdong.json"
  );

  const entities = dataSource.entities.values;
  const colors = COLOR_CONFIGS.map(
    ([r, g, b]) => new Cesium.Color(r / 255, g / 255, b / 255, 1)
  );

  entities.forEach((entity, index) => {
    entity.polygon.extrudedHeight = 10000;
    entity.polygon.outline = false;
    entity.polygon.material = new CustomMaterialProperty({
      color: colors[index % colors.length],
    });
  });

  viewer.dataSources.add(dataSource);

  viewer.camera.flyTo({
    destination: Cesium.Cartesian3.fromDegrees(113.280637, 23.125178, 20000),
    orientation: {},
    duration: 3,
  });
}

addMaterial();
```

完整源码：[GitHub](https://github.com/z2586300277/three-cesium-examples/blob/dev/cesiumExamples/expand/gradienGeojsonFace.js)

## 小结
- 本文提供 **渐变行政区** 完整 Cesium.js 源码与在线 Demo，建议先运行案例再改 uniform/参数做二次实验
- 更多 Cesium.js 实战案例见 [three-cesium-examples 合集](https://threehub.cn) 与 [GitHub 开源仓库](https://github.com/z2586300277/three-cesium-examples)

