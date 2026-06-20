---
title: "Cesium 水波纹教程"
description: "详解 Cesium.js 水波纹：水波纹扩散材质，涵盖 Cesium 等关键实现，附完整源码与在线 Demo，适合 Cesium 特效 学习与二次开发。"
head:
  - - meta
    - name: keywords
      content: "Cesium.js,水波纹,WebGL,源码,教程,在线案例,Cesium,Cesium Entity,实体"
outline: deep
---

### 水波纹 · *Ripple* · [▶ 在线运行案例](https://z2586300277.github.io/three-cesium-examples/#/codeMirror?navigation=CesiumJS&classify=singleEffect&id=ripple)
- **案例合集：** [三维可视化功能案例（threehub.cn）](https://threehub.cn)

- **开源仓库github地址：** https://github.com/z2586300277/three-cesium-examples

- **400个案例代码: ** [网盘链接](https://pan.quark.cn/s/201da5c82fec)

![水波纹](https://z2586300277.github.io/three-cesium-examples/cesiumExamples/expand/ripple.jpg)

## 你将学到什么

- Cesium Entity 高层实体 API

## 效果说明

本案例演示 **水波纹** 效果：水波纹扩散材质；核心用到 Cesium。建议先打开文首在线案例查看动态画面，再对照下方源码逐步理解。

## 核心概念
- **Viewer** 聚合 Scene、Camera、Clock 与渲染循环，是 Cesium 应用入口。
- **Entity** 面向点线面/模型/标签的高层 API；与 Primitive 相比更适合交互与属性驱动。
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
    baseLayer: Cesium.ImageryLayer.fromProviderAsync(Cesium.ArcGisMapServerImageryProvider.fromUrl(GLOBAL_CONFIG.getLayerUrl())),
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

viewer.camera.setView({
    destination: Cesium.Cartesian3.fromDegrees(116.36485552299206, 39.99754814959118, 5000.0)
});

/**
 * 水波纹扩散材质
 * @param {*} options
 * @param {String} options.color 颜色
 * @param {Number} options.duration 持续时间 毫秒
 * @param {Number} options.count 波浪数量
 * @param {Number} options.gradient 渐变曲率
 */
function CircleWaveMaterialProperty(options) {
    this._definitionChanged = new Cesium.Event();
    this.color = Cesium.defaultValue(options.color && new Cesium.Color.fromCssColorString(options.color), Cesium.Color.RED);
    this.duration = Cesium.defaultValue(options.duration, 1000);
    this.count = Cesium.defaultValue(options.count, 2);
    if (this.count <= 0) {
        this.count = 1;
    }
    this.gradient = Cesium.defaultValue(options.gradient, 0.1);
    if (this.gradient > 1) {
        this.gradient = 1;
    }
    this.time = new Date().getTime();
}
Object.defineProperties(CircleWaveMaterialProperty.prototype, {
    isConstant: {
        get: function () {
            return false;
        },
    },
    definitionChanged: {
        get: function () {
            return this._definitionChanged;
        },
    },
    color: Cesium.createPropertyDescriptor('color'),
    gradient: Cesium.createPropertyDescriptor('gradient'),
    duration: Cesium.createPropertyDescriptor('duration'),
    count: Cesium.createPropertyDescriptor('count'),
});
CircleWaveMaterialProperty.prototype.getType = function () {
    return Cesium.Material.CircleWaveMaterialType;
};
CircleWaveMaterialProperty.prototype.getValue = function (time, result) {
    if (!Cesium.defined(result)) {
        result = {};
    }
    result.color = Cesium.Property.getValueOrClonedDefault(this.color, time, Cesium.Color.WHITE, result.color);
    result.time = ((new Date().getTime() - this.time) % this.duration) / this.duration;
    result.count = this.count;
    result.gradient = 1 + 10 * (1 - this.gradient);
    return result;
};
CircleWaveMaterialProperty.prototype.equals = function (other) {
    const reData =
        this === other ||
        (other instanceof CircleWaveMaterialProperty
            && Cesium.Property.equals(this.color, other.color)
            && Cesium.Property.equals(this.duration, other.duration)
            && Cesium.Property.equals(this.count, other.count)
            && Cesium.Property.equals(this.gradient, other.gradient));
    return reData;
};
Cesium.Material.CircleWaveMaterialType = 'CircleWaveMaterial';
Cesium.Material.CircleWaveSource = `
              czm_material czm_getMaterial(czm_materialInput materialInput) {
                czm_material material = czm_getDefaultMaterial(materialInput);
                material.diffuse = 1.5 * color.rgb;
                vec2 st = materialInput.st;
                vec3 str = materialInput.str;
                float dis = distance(st, vec2(0.5, 0.5));
                float per = fract(time);
                if (abs(str.z) > 0.001) {
                  discard;
                }
                if (dis > 0.5) {
                  discard;
                } else {
                  float perDis = 0.5 / count;
                  float disNum;
                  float bl = .0;
                  for (int i = 0; i <= 9; i++) {
                    if (float(i) <= count) {
                      disNum = perDis *float(i) - dis + per / count;
                      if (disNum > 0.0) {
                        if (disNum < perDis) {
                          bl = 1.0 - disNum / perDis;
                        } else if(disNum - perDis < perDis) {
                          bl = 1.0 - abs(1.0 - disNum / perDis);
                        }
                        material.alpha = pow(bl, gradient);
                      }
                    }
                  }
                }
                return material;
              }
              `;
Cesium.Material._materialCache.addMaterial(Cesium.Material.CircleWaveMaterialType, {
    fabric: {
        type: Cesium.Material.CircleWaveMaterialType,
        uniforms: {
            color: new Cesium.Color(181, 241, 254, 1),
            time: 1,
            count: 1,
            gradient: 0.1,
        },
        source: Cesium.Material.CircleWaveSource,
    },
    translucent: function () {
        return true;
    },
});
// Cesium.CircleWaveMaterialProperty = CircleWaveMaterialProperty;

    viewer.entities.add({
        position: Cesium.Cartesian3.fromDegrees(116.36485552299206, 39.99754814959118, 100),
        ellipse: {
            semiMinorAxis: 1000,
            semiMajorAxis: 1000,
            height: 10,
            material: new CircleWaveMaterialProperty({
                color: '#FFCB33',
                duration: 3000,
                gradient: 0,
                count: 4,
            }),
        },
    })
```

完整源码：[GitHub](https://github.com/z2586300277/three-cesium-examples/blob/dev/cesiumExamples/expand/ripple.js)

## 小结
- 本文提供 **水波纹** 完整 Cesium.js 源码与在线 Demo，建议先运行案例再改 uniform/参数做二次实验
- 更多 Cesium.js 实战案例见 [three-cesium-examples 合集](https://threehub.cn) 与 [GitHub 开源仓库](https://github.com/z2586300277/three-cesium-examples)

