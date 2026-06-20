---
title: "Three.js 雷达着色器教程"
description: "详解 Three.js 雷达着色器：基于 WebGL 实现「雷达着色器」可视化效果，附完整可运行源码，涵盖 ShaderMaterial、OrbitControls 等关键实现，附完整源码与在线 Demo，适合 Three.js 着色器 学习与二次开发。"
head:
  - - meta
    - name: keywords
      content: "Three.js,雷达着色器,WebGL,源码,教程,在线案例,ShaderMaterial,自定义着色器,GLSL,OrbitControls,相机控制"
outline: deep
---

### 雷达着色器 · *Radar Shader* · [▶ 在线运行案例](https://z2586300277.github.io/three-cesium-examples/#/codeMirror?navigation=ThreeJS&classify=shader&id=radarShader)
- **案例合集：** [三维可视化功能案例（threehub.cn）](https://threehub.cn)

- **开源仓库github地址：** https://github.com/z2586300277/three-cesium-examples

- **400个案例代码: ** [网盘链接](https://pan.quark.cn/s/201da5c82fec)

![雷达着色器](https://z2586300277.github.io/three-cesium-examples/threeExamples/shader/radarShader.jpg)

## 你将学到什么

- ShaderMaterial 自定义着色器实现核心视觉效果
- OrbitControls 相机轨道交互
- `requestAnimationFrame` 渲染循环与 `resize` 自适应

## 效果说明

本案例演示 **雷达着色器** 效果：基于 WebGL 实现「雷达着色器」可视化效果，附完整可运行源码；核心用到 ShaderMaterial、OrbitControls。建议先打开文首在线案例查看动态画面，再对照下方源码逐步理解。

## 核心概念
- **Scene / Camera / WebGLRenderer** 构成最小渲染闭环；大场景可开 `logarithmicDepthBuffer` 缓解 Z-fighting。
- **ShaderMaterial** 通过 `uniforms` + 自定义 GLSL 控制逐像素/逐点效果；透明粒子常配合 `depthTest: false`。
- **OrbitControls** 提供轨道旋转/缩放；开启 `enableDamping` 后需在 animate 中 `controls.update()`。

## 实现步骤

1. 搭建 Scene、PerspectiveCamera、WebGLRenderer，挂载 canvas 并处理 `resize`
2. 定义 uniforms / onBeforeCompile 或 ShaderMaterial，编写 GLSL 与材质参数
3. 创建 OrbitControls（及 Raycaster 等交互控件，若源码包含）
4. 在 `requestAnimationFrame` 循环中更新状态并 render（Cesium 为 `viewer.render` 或自动渲染）

## 代码要点

```js
import * as THREE from 'three'
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls.js'

const box = document.getElementById('box')
const scene = new THREE.Scene()
const camera = new THREE.PerspectiveCamera(75, box.clientWidth / box.clientHeight, 0.1, 1000)
camera.position.set(0, 10, 10)
const renderer = new THREE.WebGLRenderer()
renderer.setSize(box.clientWidth, box.clientHeight)
box.appendChild(renderer.domElement)
new OrbitControls(camera, renderer.domElement)
scene.add(new THREE.AxesHelper(50000))
window.onresize = () => {
    renderer.setSize(box.clientWidth, box.clientHeight)
    camera.aspect = box.clientWidth / box.clientHeight
    camera.updateProjectionMatrix()
}

const { mesh, uniforms } = getShaderMesh()
scene.add(mesh)

animate()
function animate() {
    uniforms.iTime.value += 0.01
    requestAnimationFrame(animate)
    renderer.render(scene, camera)
}

function getShaderMesh() {
    const uniforms = {
        iTime: {
            value: 0
        },
        iResolution: {
            value: new THREE.Vector2(1900, 1900)
        },
        iChannel0: {
            value: window.iChannel0
        }
    }
    const geometry = new THREE.PlaneGeometry(20, 20);
    const material = new THREE.ShaderMaterial({
        uniforms,
        side: 2,
        depthWrite: false,
        transparent: true,
        vertexShader: `
            varying vec3 vPosition;
            varying vec2 vUv;
            void main() { 
                vUv = uv; 
                vec4 mvPosition = modelViewMatrix * vec4(position, 1.0);
                gl_Position = projectionMatrix * mvPosition;
            }
        `,
        fragmentShader: `
        #define SMOOTH(r,R) (1.0-smoothstep(R-1.0,R+1.0, r))
        #define RANGE(a,b,x) ( step(a,x)*(1.0-step(b,x)) )
        #define RS(a,b,x) ( smoothstep(a-1.0,a+1.0,x)*(1.0-smoothstep(b-1.0,b+1.0,x)) )
        #define M_PI 3.1415926535897932384626433832795

        #define blue1 vec3(0.74,0.95,1.00)
        #define blue2 vec3(0.87,0.98,1.00)
        #define blue3 vec3(0.35,0.76,0.83)
        #define blue4 vec3(0.953,0.969,0.89)
        #define red   vec3(1.00,0.38,0.227)

        #define MOV(a,b,c,d,t) (vec2(a*cos(t)+b*cos(0.1*(t)), c*sin(t)+d*cos(0.1*(t))))

        uniform float ratio;

        float PI = 3.1415926;
		uniform float iTime;
		uniform vec2 iResolution; 
		varying vec2 vUv;
        
        
        float movingLine(vec2 uv, vec2 center, float radius)
        {
            //angle of the line
            float theta0 = 90.0 * iTime;
            vec2 d = uv - center;
            float r = sqrt( dot( d, d ) );
            if(r<radius)
            {
                //compute the distance to the line theta=theta0
                vec2 p = radius*vec2(cos(theta0*M_PI/180.0),
                                    -sin(theta0*M_PI/180.0));
                float l = length( d - p*clamp( dot(d,p)/dot(p,p), 0.0, 1.0) );
                d = normalize(d);
                //compute gradient based on angle difference to theta0
                float theta = mod(180.0*atan(d.y,d.x)/M_PI+theta0,360.0);
                float gradient = clamp(1.0-theta/90.0,0.0,1.0);
                return SMOOTH(l,1.0)+0.5*gradient;
            }
            else return 0.0;
        }

        float circle(vec2 uv, vec2 center, float radius, float width)
        {
            float r = length(uv - center);
            return SMOOTH(r-width/2.0,radius)-SMOOTH(r+width/2.0,radius);
        }

        float circle2(vec2 uv, vec2 center, float radius, float width, float opening)
        {
            vec2 d = uv - center;
            float r = sqrt( dot( d, d ) );
            d = normalize(d);
            if( abs(d.y) > opening )
                return SMOOTH(r-width/2.0,radius)-SMOOTH(r+width/2.0,radius);
            else
                return 0.0;
        }
        float circle3(vec2 uv, vec2 center, float radius, float width)
        {
            vec2 d = uv - center;
            float r = sqrt( dot( d, d ) );
            d = normalize(d);
            float theta = 180.0*(atan(d.y,d.x)/M_PI);
            return smoothstep(2.0, 2.1, abs(mod(theta+2.0,45.0)-2.0)) *
                mix( 0.5, 1.0, step(45.0, abs(mod(theta, 180.0)-90.0)) ) *
                (SMOOTH(r-width/2.0,radius)-SMOOTH(r+width/2.0,radius));
        }

        float triangles(vec2 uv, vec2 center, float radius)
        {
            vec2 d = uv - center;
            return RS(-8.0, 0.0, d.x-radius) * (1.0-smoothstep( 7.0+d.x-radius,9.0+d.x-radius, abs(d.y)))
                + RS( 0.0, 8.0, d.x+radius) * (1.0-smoothstep( 7.0-d.x-radius,9.0-d.x-radius, abs(d.y)))
                + RS(-8.0, 0.0, d.y-radius) * (1.0-smoothstep( 7.0+d.y-radius,9.0+d.y-radius, abs(d.x)))
                + RS( 0.0, 8.0, d.y+radius) * (1.0-smoothstep( 7.0-d.y-radius,9.0-d.y-radius, abs(d.x)));
        }

        float _cross(vec2 uv, vec2 center, float radius)
        {
            vec2 d = uv - center;
            int x = int(d.x);
            int y = int(d.y);
            float r = sqrt( dot( d, d ) );
            if( (r<radius) && ( (x==y) || (x==-y) ) )
                return 1.0;
            else return 0.0;
        }
        float dots(vec2 uv, vec2 center, float radius)
        {
            vec2 d = uv - center;
            float r = sqrt( dot( d, d ) );
            if( r <= 2.5 )
                return 1.0;
            if( ( r<= radius) && ( (abs(d.y+0.5)<=1.0) && ( mod(d.x+1.0, 50.0) < 2.0 ) ) )
                return 1.0;
            else if ( (abs(d.y+0.5)<=1.0) && ( r >= 50.0 ) && ( r < 115.0 ) )
                return 0.5;
            else
                return 0.0;
        }
        float bip1(vec2 uv, vec2 center)
        {
            return SMOOTH(length(uv - center),3.0);
        }
        float bip2(vec2 uv, vec2 center)
        {
            float r = length(uv - center);
            float R = 8.0+mod(87.0*iTime, 80.0);
            return (0.5-0.5*cos(30.0*iTime)) * SMOOTH(r,5.0)
                + SMOOTH(6.0,r)-SMOOTH(8.0,r)
                + smoothstep(max(8.0,R-20.0),R,r)-SMOOTH(R,r);
        }
		void main() { 
            vec2 _uv = vec2(vUv.x * iResolution.x, vUv.y * iResolution.y);
            vec3 finalColor;
            vec2 uv = _uv;
            //center of the image
            vec2 c = vec2(iResolution.x / 2.0, iResolution.y / 2.0);
            finalColor = vec3( 0.3*_cross(uv, c, 240.0) );
            finalColor += ( circle(uv, c, 100.0, 1.0)
                        + circle(uv, c, 165.0, 1.0) ) * blue1;
            finalColor += (circle(uv, c, 240.0, 2.0) );//+ dots(uv,c,240.0)) * blue4;
            finalColor += circle3(uv, c, 313.0, 4.0) * blue1;
            finalColor += triangles(uv, c, 315.0 + 30.0*sin(iTime)) * blue2;
            finalColor += movingLine(uv, c, 240.0) * blue3;
            finalColor += circle(uv, c, 10.0, 1.0) * blue3;
            finalColor += 0.7 * circle2(uv, c, 262.0, 1.0, 0.5+0.2*cos(iTime)) * blue3;
            if( length(uv-c) < 240.0 )
            {
                //animate some bips with random movements
                vec2 p = 130.0*MOV(1.3,1.0,1.0,1.4,3.0+0.1*iTime);
                finalColor += bip1(uv, c+p) * vec3(1,1,1);
                p = 130.0*MOV(0.9,-1.1,1.7,0.8,-2.0+sin(0.1*iTime)+0.15*iTime);
                finalColor += bip1(uv, c+p) * vec3(1,1,1);
                p = 50.0*MOV(1.54,1.7,1.37,1.8,sin(0.1*iTime+7.0)+0.2*iTime);
                finalColor += bip2(uv,c+p) * red;
            }

			gl_FragColor = vec4( finalColor, 1.0 );
			
		}
        `
    })
    const mesh = new THREE.Mesh(geometry, material);
    return {
        mesh,
        uniforms
    }
}
```

完整源码：[GitHub](https://github.com/z2586300277/three-cesium-examples/blob/dev/threeExamples/shader/radarShader.js)

## 小结

- 本文提供 **雷达着色器** 完整 Three.js 源码与在线 Demo，建议先运行案例再改 uniform/参数做二次实验
- 更多 Three.js 实战案例见 [three-cesium-examples 合集](https://threehub.cn) 与 [GitHub 开源仓库](https://github.com/z2586300277/three-cesium-examples)

