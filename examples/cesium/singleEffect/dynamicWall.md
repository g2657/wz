---
title: "Cesium 动态围墙教程"
description: "详解 Cesium.js 动态围墙：基于 WebGL 实现「动态围墙」可视化效果，附完整可运行源码，涵盖 Cesium 等关键实现，附完整源码与在线 Demo，适合 Cesium 特效 学习与二次开发。"
head:
  - - meta
    - name: keywords
      content: "Cesium.js,动态围墙,WebGL,源码,教程,在线案例,Cesium,Cesium Entity,实体"
outline: deep
---

### 动态围墙 · *Dynamic Wall* · [▶ 在线运行案例](https://z2586300277.github.io/three-cesium-examples/#/codeMirror?navigation=CesiumJS&classify=singleEffect&id=dynamicWall)
- **案例合集：** [三维可视化功能案例（threehub.cn）](https://threehub.cn)

- **开源仓库github地址：** https://github.com/z2586300277/three-cesium-examples

- **400个案例代码: ** [网盘链接](https://pan.quark.cn/s/201da5c82fec)

![动态围墙](https://z2586300277.github.io/three-cesium-examples/cesiumExamples/effect/dynamicWall.jpg)

## 你将学到什么

- Cesium Entity 高层实体 API

## 效果说明

本案例演示 **动态围墙** 效果：基于 WebGL 实现「动态围墙」可视化效果，附完整可运行源码；核心用到 Cesium。建议先打开文首在线案例查看动态画面，再对照下方源码逐步理解。

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

    baseLayer: Cesium.ImageryLayer.fromProviderAsync(Cesium.ArcGisMapServerImageryProvider.fromUrl('https://server.arcgisonline.com/arcgis/rest/services/World_Imagery/MapServer')),

    fullscreenButton: false,//是否显示全屏按钮，右下角全屏选择按钮

    timeline: false,//是否显示时间轴    

    infoBox: false,//是否显示信息框   

})

// 🐘优雅永不过时 改编自 https://juejin.cn/post/7431590701496533030
class DynamicWallMaterialProperty {
    constructor(options) {
        // 默认参数设置
        this._definitionChanged = new Cesium.Event() // 材质定义变更事件
        this._color = undefined // 颜色属性
        this._colorSubscription = undefined // 颜色变化订阅
        this.color = options.color // 从选项中获取颜色
        this.duration = options.duration // 持续时间
        this.trailImage = options.trailImage // 路径图像
        this._time = new Date().getTime() // 当前时间戳
        this._viewer = options.viewer // Cesium 视图对象
    }
    // 返回材质类型
    getType(time) {
        return MaterialType // 返回材质类型名称
    }
    getValue(time, result) {
        if (!Cesium.defined(result)) {
            result = {} // 如果结果未定义，则初始化为空对象
        }
        result.color = Cesium.Property.getValueOrClonedDefault(
            this._color, // 获取颜色值
            time, // 当前时间
            Cesium.Color.WHITE, // 默认颜色为白色
            result.color // 结果对象中的颜色属性
        )
        // 使用自定义的路径图像
        result.image = this.trailImage
        // 计算时间进度
        if (this.duration) {
            result.time =
                ((new Date().getTime() - this._time) % this.duration) / this.duration
        }
        this._viewer.scene.requestRender() // 请求重新渲染场景
        return result
    }
    // 比较两个 DynamicWallMaterialProperty 对象是否相等
    equals(other) {
        return (
            this === other || // 判断是否为同一对象
            (other instanceof DynamicWallMaterialProperty && // 判断是否为 DynamicWallMaterialProperty 的实例
                Cesium.Property.equals(this._color, other._color)) // 比较颜色属性
        )
    }
}

function _getDirectionWallShader(options) {
    if (options && options.get) {
        // 定义了一个函数 czm_getMaterial，输入参数为 materialInput，返回值为 czm_material 类型的材质。
        var materail = `czm_material czm_getMaterial(czm_materialInput materialInput)
      {
      // 调用 czm_getDefaultMaterial 函数获取一个默认的材质实例，并将其存储在 material 变量中
          czm_material material = czm_getDefaultMaterial(materialInput);
          // 获取纹理坐标（st）的二维向量
          vec2 st = materialInput.st;`
        // 垂直方向动态效果
        if (options.freely == 'vertical') {
            //（由下到上）
            // texture(image, vec2(u,v))，st.s 是水平方向上的纹理坐标，st.t 是垂直方向上的纹理坐标。
            // 如果要实现上下垂直滚动的效果，就要设置st.t随时间的动态，而st.s值不变。
            // 如果要实现左右水平滚动的效果，就要设置st.s随时间的动态，而st.t值不变。
            materail +=
                // 纹理采样，依据时间动态变化,fract 函数用于计算余数，使纹理坐标在[0, 1)范围内循环。
                'vec4 colorImage = texture(image, vec2(fract(st.s), fract(float(' +
                options.count +
                ')*st.t' +
                options.direction +
                ' time)));\n '
        } else {
            // 水平方向的动态效果
            //（逆时针）
            materail +=
                'vec4 colorImage = texture(image, vec2(fract(float(' +
                options.count +
                ')*st.s ' +
                options.direction +
                ' time), fract(st.t)));\n '
            console.log('materail2: ', materail)
        }
        //泛光
        materail += `vec4 fragColor;
          fragColor.rgb = (colorImage.rgb+color.rgb) / 1.0;
          fragColor = czm_gammaCorrect(fragColor);
          material.diffuse = colorImage.rgb;
          material.alpha = colorImage.a;
          material.emission = fragColor.rgb;
          return material;
      }`
        return materail
    }
}

Object.defineProperties(DynamicWallMaterialProperty.prototype, {
    isConstant: {
        get: function () {
            return false // 返回材质是否是常量（动态材质返回 false）
        },
    },
    definitionChanged: {
        get: function () {
            return this._definitionChanged // 返回定义变更事件
        },
    },
    color: Cesium.createPropertyDescriptor('color'), // 创建颜色属性描述符
})

var MaterialType = 'wallType' + parseInt(Math.random() * 1000)
// 将材质添加到缓存中
Cesium.Material._materialCache.addMaterial(MaterialType, {
    fabric: {
        type: MaterialType, // 设置材质类型
        uniforms: {
            color: new Cesium.Color(1.0, 0.0, 0.0, 0.5), // 设置颜色属性
            image: '', // 设置图像路径
            time: -20, // 设置时间属性
        },
        source: _getDirectionWallShader({
            get: true,
            count: 3.0,
            freely: 'vertical', //或者standard
            direction: '+',
        }),
    },
    translucent: function (material) {
        return true // 确定材质是否是半透明的
    },
})

let positions = Cesium.Cartesian3.fromDegreesArray([
    113.8236839, 22.528061, 113.9236839, 22.628061, 114.0236839, 22.528061,
    113.9236839, 22.428061, 113.8236839, 22.528061,
])
// 绘制墙体
const wall = viewer.entities.add({
    name: '立体墙效果',
    wall: {
        positions: positions,
        // 设置高度
        maximumHeights: new Array(positions.length).fill(3000),
        minimumHeights: new Array(positions.length).fill(0),
        material: new DynamicWallMaterialProperty({
            viewer,
            trailImage: FILE_HOST + '/images/texture/flyLine1.png',
            color: Cesium.Color.RED,
            duration: 1500,
        }),
    },
})
viewer.zoomTo(wall)
```

完整源码：[GitHub](https://github.com/z2586300277/three-cesium-examples/blob/dev/cesiumExamples/effect/dynamicWall.js)

## 小结
- 本文提供 **动态围墙** 完整 Cesium.js 源码与在线 Demo，建议先运行案例再改 uniform/参数做二次实验
- 更多 Cesium.js 实战案例见 [three-cesium-examples 合集](https://threehub.cn) 与 [GitHub 开源仓库](https://github.com/z2586300277/three-cesium-examples)

