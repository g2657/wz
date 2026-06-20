---
title: "Three.js 发散飞线教程"
description: "详解 Three.js 发散飞线：[addFly description]，涵盖 ShaderMaterial、OrbitControls、THREE.Points 等关键实现，附完整源码与在线 Demo，适合 Three.js 应用场景 学习与二次开发。"
head:
  - - meta
    - name: keywords
      content: "Three.js,发散飞线,WebGL,源码,教程,在线案例,ShaderMaterial,自定义着色器,GLSL,OrbitControls,相机控制,粒子特效"
outline: deep
---

### 发散飞线 · *Diffuse Line* · [▶ 在线运行案例](https://z2586300277.github.io/three-cesium-examples/#/codeMirror?navigation=ThreeJS&classify=application&id=diffuseLine)
- **案例合集：** [三维可视化功能案例（threehub.cn）](https://threehub.cn)

- **开源仓库github地址：** https://github.com/z2586300277/three-cesium-examples

- **400个案例代码: ** [网盘链接](https://pan.quark.cn/s/201da5c82fec)

![发散飞线](https://z2586300277.github.io/three-cesium-examples/threeExamples/application/diffuseLine.jpg)

## 你将学到什么

- ShaderMaterial 自定义着色器实现核心视觉效果
- OrbitControls 相机轨道交互
- THREE.Points 粒子点渲染
- QuadraticBezierCurve3 二次贝塞尔曲线路径
- Raycaster 鼠标拾取与交互
- BufferGeometry 自定义顶点/索引数据
- `requestAnimationFrame` 渲染循环与 `resize` 自适应

## 效果说明

本案例演示 **发散飞线** 效果：从中心或锚点定时生成随机曲线，沿路径播放粒子流光，支持鼠标拾取、绘制或拖拽交互；核心用到 ShaderMaterial、OrbitControls、THREE.Points。建议先打开文首在线案例查看动态画面，再对照下方源码逐步理解。

## 核心概念
- **Scene / Camera / WebGLRenderer** 构成最小渲染闭环；大场景可开 `logarithmicDepthBuffer` 缓解 Z-fighting。
- **ShaderMaterial** 通过 `uniforms` + 自定义 GLSL 控制逐像素/逐点效果；透明粒子常配合 `depthTest: false`。
- **OrbitControls** 提供轨道旋转/缩放；开启 `enableDamping` 后需在 animate 中 `controls.update()`。
- **THREE.Points** 将每个顶点渲染为可控大小的粒子；可用自定义 attribute（如 `u_index`）驱动片元/顶点动画。
- 曲线类 `getPoints(n)` 将贝塞尔/样条离散为路径点，再写入 BufferGeometry 驱动飞线或路径动画。

## 实现步骤

1. 搭建 Scene、PerspectiveCamera、WebGLRenderer，挂载 canvas 并处理 `resize`
2. 定义 uniforms / onBeforeCompile 或 ShaderMaterial，编写 GLSL 与材质参数
3. 用曲线离散点构建 BufferGeometry，写入自定义 attribute 驱动动画
4. 创建 OrbitControls（及 Raycaster 等交互控件，若源码包含）
5. 在定时器或 GSAP 时间轴中更新 uniform / 变换，驱动特效播放
6. 在 `requestAnimationFrame` 循环中更新状态并 render（Cesium 为 `viewer.render` 或自动渲染）

## 代码要点

```js
import * as THREE from "three";
import { OrbitControls } from "three/examples/jsm/controls/OrbitControls.js";
import Stats from 'three/examples/jsm/libs/stats.module.js';

class InitFly {
    constructor({
        texture
    } = opt) {
        this.flyId = 0; //id
        this.flyArr = []; //存储所有飞线
        this.baicSpeed = 1; //基础速度
        this.texture = 0.0;
        if (texture && !texture.isTexture) {
            this.texture = new THREE.TextureLoader().load(texture)
        } else {
            this.texture = texture;
        }
        this.flyShader = {
            vertexshader: ` 
                uniform float size; 
                uniform float time; 
                uniform float u_len; 
                attribute float u_index;
                varying float u_opacitys;
                void main() { 
                    if( u_index < time + u_len && u_index > time){
                        float u_scale = 1.0 - (time + u_len - u_index) /u_len;
                        u_opacitys = u_scale;
                        vec4 mvPosition = modelViewMatrix * vec4(position, 1.0);
                        gl_Position = projectionMatrix * mvPosition;
                        gl_PointSize = size * u_scale * 300.0 / (-mvPosition.z);
                    } 
                }
                `,
            fragmentshader: ` 
                uniform sampler2D u_map;
                uniform float u_opacity;
                uniform vec3 color;
                uniform float isTexture;
                varying float u_opacitys;
                void main() {
                    vec4 u_color = vec4(color,u_opacity * u_opacitys);
                    if( isTexture != 0.0 ){
                        gl_FragColor = u_color * texture2D(u_map, vec2(gl_PointCoord.x, 1.0 - gl_PointCoord.y));
                    }else{
                        gl_FragColor = u_color;
                    }
                }`
        }
    }
    /**
     * [addFly description]
     *
     * @param   {String}  opt.color  [颜色_透明度]
     * @param   {Array}   opt.curve  [线的节点]
     * @param   {Number}  opt.width  [宽度]
     * @param   {Number}  opt.length [长度]
     * @param   {Number}  opt.speed  [速度]
     * @param   {Number}  opt.repeat [重复次数]
     * @return  {Mesh}               [return 图层]
     */
    addFly({
        color = "rgba(255,255,255,1)",
        curve = [],
        width = 1,
        length = 10,
        speed = 1,
        repeat = 1,
        texture = null,
        callback
    } = opt) {
        let colorArr = this.getColorArr(color);
        let geometry = new THREE.BufferGeometry();
        let material = new THREE.ShaderMaterial({
            uniforms: {
                color: {
                    value: colorArr[0],
                    type: "v3"
                },
                size: {
                    value: width,
                    type: "f"
                },
                u_map: {
                    value: texture ? texture : this.texture,
                    type: "t2"
                },
                u_len: {
                    value: length,
                    type: "f"
                },
                u_opacity: {
                    value: colorArr[1],
                    type: "f"
                },
                time: {
                    value: -length,
                    type: "f"
                },
                isTexture: {
                    value: 1.0,
                    type: "f"
                }
            },
            transparent: true,
            depthTest: false,
            vertexShader: this.flyShader.vertexshader,
            fragmentShader: this.flyShader.fragmentshader
        });
        const [position, u_index] = [
            [],
            []
        ];
        curve.forEach(function (elem, index) {
            position.push(elem.x, elem.y, elem.z);
            u_index.push(index);
        })
        geometry.setAttribute("position", new THREE.Float32BufferAttribute(position, 3));
        geometry.setAttribute("u_index", new THREE.Float32BufferAttribute(u_index, 1));
        let mesh = new THREE.Points(geometry, material);
        mesh.name = "fly";
        mesh._flyId = this.flyId;
        mesh._speed = speed;
        mesh._repeat = repeat;
        mesh._been = 0;
        mesh._total = curve.length;
        mesh._callback = callback;
        this.flyId++;
        this.flyArr.push(mesh);
        return mesh
    }
    /**
     * 根据线条组生成路径
     * @param {*} arr 需要生成的线条组
     * @param {*} dpi 密度
     */
    tranformPath(arr, dpi = 1) {
        const vecs = [];
        for (let i = 1; i < arr.length; i++) {
            let src = arr[i - 1];
            let dst = arr[i];
            let s = new THREE.Vector3(src.x, src.y, src.z);
            let d = new THREE.Vector3(dst.x, dst.y, dst.z);
            let length = s.distanceTo(d) * dpi;
            let len = parseInt(length);
            for (let i = 0; i <= len; i++) {
                vecs.push(s.clone().lerp(d, i / len))
            }
        }
        return vecs;
    }
    /**
     * [remove 删除]
     * @param   {Object}  mesh  [当前飞线]
     */
    remove(mesh) {
        mesh.material.dispose();
        mesh.geometry.dispose();
        this.flyArr = this.flyArr.filter(elem => elem._flyId != mesh._flyId);
        mesh.parent.remove(mesh);
        mesh = null;
    }
    /**
     * [animation 动画] 
     * @param   {Number}  delta  [执行动画间隔时间] 
     */
    animation(delta = 0.015) {
        if (delta > 0.2) return;
        this.flyArr.forEach(elem => {
            if (!elem.parent) return;
            if (elem._been > elem._repeat) {
                elem.visible = false;
                if (typeof elem._callback === 'function') {
                    elem._callback(elem);
                }
                this.remove(elem)
            } else {
                let uniforms = elem.material.uniforms;
                //完结一次
                if (uniforms.time.value < elem._total) {
                    uniforms.time.value += delta * (this.baicSpeed / delta) * elem._speed;
                } else {
                    elem._been += 1;
                    uniforms.time.value = -uniforms.u_len.value;
                }
            }
        })
    }
    color(c) {
        return new THREE.Color(c);
    }
    getColorArr(str) {
        if (Array.isArray(str)) return str; //error
        var _arr = [];
        str = str + '';
        str = str.toLowerCase().replace(/\s/g, "");
        if (/^((?:rgba)?)\(\s*([^\)]*)/.test(str)) {
            var arr = str.replace(/rgba\(|\)/gi, '').split(',');
            var hex = [
                pad2(Math.round(arr[0] * 1 || 0).toString(16)),
                pad2(Math.round(arr[1] * 1 || 0).toString(16)),
                pad2(Math.round(arr[2] * 1 || 0).toString(16))
            ];
            _arr[0] = this.color('#' + hex.join(""));
            _arr[1] = Math.max(0, Math.min(1, (arr[3] * 1 || 0)));
        } else if ('transparent' === str) {
            _arr[0] = this.color();
            _arr[1] = 0;
        } else {
            _arr[0] = this.color(str);
            _arr[1] = 1;
        }

        function pad2(c) {
            return c.length == 1 ? '0' + c : '' + c;
        }
        return _arr;
    }
}

function Initialize(opt) {
    var camera, controls, scene, renderer;
    var clock = new THREE.Clock();
    var thm = this;
    var df_Mouse, df_Raycaster;
    var df_Width, df_Height; //当前盒子的高宽
    var df_canvas;

    var stats

    function init() {
        camera = new THREE.PerspectiveCamera(60, window.innerWidth / window.innerHeight, 1, 10000);
        camera.position.set(0, 400, 400);
        camera.lookAt(new THREE.Vector3(0, 0, 0))
        scene = new THREE.Scene()

        renderer = new THREE.WebGLRenderer({ antialias: true });
        renderer.setPixelRatio(window.devicePixelRatio * 1.3);
        renderer.setSize(window.innerWidth, window.innerHeight);

        document.querySelector(opt.id).appendChild(renderer.domElement);
        df_canvas = renderer.domElement
        controls = new OrbitControls(camera, renderer.domElement);
        window.addEventListener('resize', onWindowResize, false);
        renderer.domElement.addEventListener('mouseup', onDocumentMouseUp, false);
        df_Width = window.innerWidth;
        df_Height = window.innerHeight;
        df_Mouse = new THREE.Vector2();
        df_Raycaster = new THREE.Raycaster();
        // onload 
        if (opt.load) {
            opt.load({
                camera, controls, scene, renderer
            })
        }
        if (Stats) {
            stats = new Stats();
            document.querySelector(opt.id).appendChild(stats.dom);
        }
    }

    function onDocumentMouse(event) {
        event.preventDefault();
        df_Mouse.x = ((event.clientX - df_canvas.getBoundingClientRect().left) / df_canvas.offsetWidth) * 2 - 1;
        df_Mouse.y = -((event.clientY - df_canvas.getBoundingClientRect().top) / df_canvas.offsetHeight) * 2 + 1;
        df_Raycaster.setFromCamera(df_Mouse, camera);
        return {
            mouse: df_Mouse,
            event: event,
            raycaster: df_Raycaster
        }
    }
    function onWindowResize() {
        camera.aspect = window.innerWidth / window.innerHeight;
        camera.updateProjectionMatrix();
        renderer.setSize(window.innerWidth, window.innerHeight);
    }
    function onDocumentMouseUp(event) {
        if (typeof opt.mouseUp === 'function') {
            opt.mouseUp(onDocumentMouse(event))
        }
    }
    function animate() {
        requestAnimationFrame(animate);
        var delta = clock.getDelta();
        renderer.render(scene, camera);
        if (opt.animation) opt.animation(delta);
        if (stats) stats.update();
    }
    init();
    animate();
}

var _Fly;
var GL = new Initialize({
    id: "#box",
    animation: function (dalte) {
        if (_Fly) {
            // 更新线 必须
            _Fly.animation(dalte);
        }
    },
    load({ scene, camera }) {
        _Fly = new InitFly({
            texture: FILE_HOST + "threeExamples/application/flyLine/point.png"
        });
        let index = 0;
        var time = setInterval(() => {
            if (index >= 4000) {
                clearInterval(time)
            }
            var x = 0;
            var z = 0;
            var x1 = THREE.MathUtils.randFloat(-200, 200);
            var z1 = THREE.MathUtils.randFloat(-200, 200);
            var curve = new THREE.QuadraticBezierCurve3(
                new THREE.Vector3(x, 0, z),
                new THREE.Vector3((x + x1) / 2, THREE.MathUtils.randInt(200, 420), (z1 + z) / 2),
                new THREE.Vector3(x1, 0, z1)
            );
            var points = curve.getPoints(500);
            var flyMesh = _Fly.addFly({
                color: `rgba(${THREE.MathUtils.randInt(0, 255)},${THREE.MathUtils.randInt(0, 255)},${THREE.MathUtils.randInt(0, 255)},1)`,
                curve: points,
                width: 9,
                length: 150,
                speed: 1,
                repeat: Infinity
            })
            scene.add(flyMesh);
            index++;
        })
    }
})
```

完整源码：[GitHub](https://github.com/z2586300277/three-cesium-examples/blob/dev/threeExamples/application/diffuseLine.js)

## 小结

- 本文提供 **发散飞线** 完整 Three.js 源码与在线 Demo，建议先运行案例再改 uniform/参数做二次实验
- 更多 Three.js 实战案例见 [three-cesium-examples 合集](https://threehub.cn) 与 [GitHub 开源仓库](https://github.com/z2586300277/three-cesium-examples)

