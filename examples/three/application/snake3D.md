---
title: "Three.js 3D贪吃蛇教程"
description: "详解 Three.js 3D贪吃蛇：基于 WebGL 实现「3D贪吃蛇」可视化效果，附完整可运行源码，涵盖 OrbitControls、THREE.Points、glTF/Draco 等关键实现，附完整源码与在线 Demo，适合 Three.js 应用场景 学习与二次开发。"
head:
  - - meta
    - name: keywords
      content: "Three.js,3D贪吃蛇,WebGL,源码,教程,在线案例,OrbitControls,相机控制,粒子特效,Points,glTF,模型加载"
outline: deep
---

### 3D贪吃蛇 · *Snake 3D* · [▶ 在线运行案例](https://z2586300277.github.io/three-cesium-examples/#/codeMirror?navigation=ThreeJS&classify=application&id=snake3D)
- **案例合集：** [三维可视化功能案例（threehub.cn）](https://threehub.cn)

- **开源仓库github地址：** https://github.com/z2586300277/three-cesium-examples

- **400个案例代码: ** [网盘链接](https://pan.quark.cn/s/201da5c82fec)

![3D贪吃蛇](https://z2586300277.github.io/three-cesium-examples/threeExamples/application/snake3D.jpg)

## 你将学到什么

- OrbitControls 相机轨道交互
- THREE.Points 粒子点渲染
- glTF/Draco 模型加载与优化
- BufferGeometry 自定义顶点/索引数据
- 场景雾效增强纵深
- `requestAnimationFrame` 渲染循环与 `resize` 自适应

## 效果说明

本案例演示 **3D贪吃蛇** 效果：基于 WebGL 实现「3D贪吃蛇」可视化效果，附完整可运行源码；核心用到 OrbitControls、THREE.Points、glTF/Draco。建议先打开文首在线案例查看动态画面，再对照下方源码逐步理解。

## 核心概念

- **Scene / Camera / WebGLRenderer** 构成最小渲染闭环；大场景可开 `logarithmicDepthBuffer` 缓解 Z-fighting。
- **OrbitControls** 提供轨道旋转/缩放；开启 `enableDamping` 后需在 animate 中 `controls.update()`。
- **THREE.Points** 将每个顶点渲染为可控大小的粒子；可用自定义 attribute（如 `u_index`）驱动片元/顶点动画。

## 实现步骤

1. 搭建灯光与环境（如有）
2. requestAnimationFrame 循环 update + render

## 代码要点

```html
<!DOCTYPE html>
<html lang="zh-CN">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
    <script src="https://cdnjs.cloudflare.com/ajax/libs/three.js/r128/three.min.js"></script>
    <script src="https://cdn.jsdelivr.net/npm/three@0.128.0/examples/js/controls/OrbitControls.min.js"></script>
    <script src="https://cdn.jsdelivr.net/npm/three@0.128.0/examples/js/loaders/GLTFLoader.js"></script>
    <title>3D贪吃蛇：浮游世界大冒险 - 第一人称模式</title>
    <style>
        * {
            margin: 0;
            padding: 0;
            box-sizing: border-box;
            touch-action: none;
            user-select: none;
            font-family: 'Segoe UI', 'Microsoft YaHei', sans-serif;
        }
        
        body {
            background: linear-gradient(135deg, #0c1e3a, #0a1429);
            color: #e6f7ff;
            min-height: 100vh;
            overflow: hidden;
            display: flex;
            flex-direction: column;
        }
        
        .game-container {
            display: flex;
            flex: 1;
            position: relative;
            overflow: hidden;
        }
        
        #gameCanvas {
            position: absolute;
            top: 0;
            left: 0;
            width: 100%;
            height: 100%;
        }
        
        /* 分数显示 */
        #scoreDisplay {
            position: absolute;
            top: 1.5%;
            left: 1.5%;
            background: rgba(10, 25, 50, 0.8);
            padding: 1.2vh 1vw;
            border-radius: 2.5vh;
            font-size: calc(12px + 0.8vw);
            font-weight: bold;
            color: #00ffcc;
            z-index: 5;
            box-shadow: 0 0.5vh 1.5vh rgba(0, 0, 0, 0.6);
            border: 0.2vh solid rgba(0, 200, 255, 0.4);
            backdrop-filter: blur(8px);
            min-width: 80px;
            min-height: 4vh;
            display: flex;
            align-items: center;
            justify-content: center;
        }
        
        /* 小地图 - 正方形 */
        .mini-map-container {
            position: absolute;
            bottom: 11vh; 
            right: 10%; 
            width: 18vw;
            min-width: 100px;
            max-width: 300px;
            min-height: 100px;
            max-height: 300px;
            padding-top: 18vw; /* 高度等于宽度 */
            background: rgba(10, 25, 50, 0.85);
            border-radius: 1.2vh;
            z-index: 5;
        
            box-shadow: 0 0 2vh rgba(0, 0, 0, 0.8);
        
        }
        
        #miniMap {
            position: absolute;
            top: 0;
            left: 0;
            width: 100%;
            height: 100%;
        }
        
        
        
        /* 高度指示器 */
        .height-indicator {
            position: absolute;
            top: 67%;
            left: 72%;
            color: #00ffaa;
            font-size: calc(10px + 0.8vw);
            z-index: 20;
            transform: translate(-50%, 0);
        }
        
        /* 虚拟摇杆 */
        .joystick-container {
            position: absolute;
            z-index: 10;
            left: 5%;
            bottom: 8%;
            transition: all 0.3s ease;
            touch-action: none;
            cursor: move;
            width: 14vw;
            min-width: 90px;
            max-width: 120px;
            height: 14vw;
            min-height: 90px;
            max-height: 120px;
        }
        
        .joystick-base {
            width: 100%;
            height: 100%;
            border-radius: 50%;
            background: rgba(10, 25, 50, 0.75);
            border: 0.2vh solid rgba(0, 200, 255, 0.4);
            box-shadow: 0 0 2.5vh rgba(0, 0, 0, 0.7);
            backdrop-filter: blur(8px);
            display: flex;
            align-items: center;
            justify-content: center;
            position: relative;
        }
        
        .joystick-head {
            width: 50%;
            height: 50%;
            border-radius: 50%;
            background: linear-gradient(135deg, #00aaff, #00ffaa);
            box-shadow: 0 0 2vh rgba(0, 200, 255, 0.7);
            position: absolute;
            cursor: pointer;
            transition: transform 0.1s ease;
            display: flex;
            flex-direction: column;
            align-items: center;
            justify-content: center;
            color: #fff;
            font-weight: bold;
            font-size: calc(8px + 0.8vw);
            text-align: center;
            text-shadow: 0 0 0.5vh rgba(0, 0, 0, 0.7);
        }
        
        .joystick-head:active {
            transform: scale(1.1);
        }
        
        .joystick-stats {
            font-size: calc(6px + 0.7vw);
            text-align: center;
            line-height: 1.2;
        }
        
        .direction-indicator {
            position: absolute;
            top: -3.5vh;
            left: 0;
            width: 100%;
            text-align: center;
            color: #00ffcc;
            font-size: calc(8px + 0.8vw);
            font-weight: bold;
            opacity: 0;
            transition: opacity 0.3s;
            text-shadow: 0 0 0.5vh rgba(0, 200, 255, 0.7);
        }
        
        .joystick-container.active .direction-indicator {
            opacity: 1;
        }
        
        .joystick-hint {
            position: absolute;
            bottom: -4.5vh;
            left: 0;
            width: 100%;
            text-align: center;
            color: #00aaff;
            font-size: calc(8px + 0.7vw);
            opacity: 0.9;
            animation: pulse 2s infinite;
            font-weight: bold;
        }
        
        /* 游戏结束 */
        .game-over {
            position: absolute;
            top: 0;
            left: 0;
            width: 100%;
            height: 100%;
            background: rgba(5, 10, 20, 0.95);
            display: flex;
            flex-direction: column;
            align-items: center;
            justify-content: center;
            z-index: 20;
            display: none;
        }
        
        .game-over-content {
            background: rgba(10, 25, 50, 0.97);
            border-radius: 2.5vh;
            padding: 3vh;
            text-align: center;
            box-shadow: 0 0 3vh rgba(0, 200, 255, 0.6);
            border: 0.3vh solid rgba(0, 200, 255, 0.7);
            max-width: 90%;
            width: 35vw;
            min-width: 280px;
            backdrop-filter: blur(15px);
        }
        
        .game-over h2 {
            font-size: calc(20px + 2vw);
            color: #ff3366;
            margin-bottom: 2vh;
            text-shadow: 0 0 1.5vh rgba(255, 50, 100, 0.7);
        }
        
        .final-score {
            font-size: calc(18px + 1.8vw);
            color: #00ffcc;
            margin: 2vh 0;
            text-shadow: 0 0 1vh rgba(0, 255, 200, 0.7);
        }
        
        .restart-btn {
            background: linear-gradient(45deg, #00ccaa, #00a0ff);
            border: none;
            padding: 1.5vh 4vw;
            font-size: calc(14px + 1.2vw);
            border-radius: 50px;
            color: white;
            cursor: pointer;
            margin-top: 2.5vh;
            font-weight: bold;
            box-shadow: 0 0.6vh 2vh rgba(0, 200, 255, 0.7);
            transition: transform 0.2s, box-shadow 0.2s;
        }
        
        .restart-btn:hover {
            transform: scale(1.05);
            box-shadow: 0 0.8vh 2.5vh rgba(0, 200, 255, 0.8);
        }
        
        .restart-btn:active {
            transform: scale(0.95);
            box-shadow: 0 0.3vh 1.2vh rgba(0, 200, 255, 0.4);
        }
        
        /* 暂停界面 */
        .pause-overlay {
            position: absolute;
            top: 0;
            left: 0;
            width: 100%;
            height: 100%;
            background: rgba(0, 0, 0, 0);
            display: flex;
            align-items: center;
            justify-content: center;
            z-index: 0;
            display: none;
        }
        
        .pause-text {
            font-size: calc(20px + 3vw);
            color: #00ffcc;
            text-shadow: 0 0 2vh rgba(0, 255, 200, 0.8);
            animation: pulse 2s infinite;
        }
        
        .loading-text {
            color: #00ffcc;
            font-size: calc(18px + 2.5vw);
            text-shadow: 0 0 1.5vh rgba(0, 255, 200, 0.8);
            animation: pulse 2s infinite;
        }
        
        /* 垂直功率控制 - 右下角新位置 */
        .power-control-vertical {
            position: absolute;
            right: 2vw;
            bottom: 2vw;
            z-index: 10;
            border-radius: 1.2vw;
            display: flex;
            flex-direction: column;
            align-items: center;
            width: 5vw;
            min-width: 50px;
            max-width: 80px;
            height: 42vh;
            min-height: 200px;
            max-height: 300px;
            padding: 1vw 0.5vw 1vw 0.5vw;
        }
        .power-slider-vertical {
            height: 68%;
            width: 100%;
            position: relative;
            display: flex;
            flex-direction: column;
            align-items: center;
            justify-content: flex-end;
        }
        /* 功率滑块 */
        .power-slider-vertical input[type="range"] {
            -webkit-appearance: slider-vertical;
            writing-mode: bt-lr;
            height: 100%;
            width: 5vw;
            min-width: 40px;
            max-width: 70px;
            border-radius: 1vw;
            border: none;
            margin: 0;
            outline: none;
        }
        .power-slider-vertical input[type="range"]::-webkit-slider-thumb {
            -webkit-appearance: none;
            width: 3.2vw;
            min-width: 35px;
            max-width: 50px;
            height: 3.2vw;
            min-height: 35px;
            max-height: 50px;
            border-radius: 50%;
            cursor: pointer;
            transition: box-shadow 0.2s;
        }
       
        .power-slider-vertical input[type="range"]:focus {
            outline: none;
        }
        /* 标记点 */
        .power-markers {
            position: absolute;
            left: 50%;
            top: 0;
            transform: translateX(-50%);
            height: 100%;
            width: 8px;
            z-index: 1;
            pointer-events: none;
        }
        /* 仅保留中间一个点，美观化 */
        .power-marker {
            display: none;
        }
        .power-marker.medium {
            display: block;
            position: absolute;
            left: 50%;
            top: 50%;
            transform: translate(-50%, -50%);
            width: 16px;
            height: 16px;
            border-radius: 50%;
            background: linear-gradient(135deg, #ffff55 60%, #00ffaa 100%);
            opacity: 0.95;
            box-shadow: 0 0 12px #00ffaa88, 0 0 4px #ffff55;
            border: 2px solid #00ffaa;
        }
        /* 档把 */
        .power-gear {
            position: absolute;
            top: -2.8vw;
            left: 50%;
            transform: translateX(-50%);
            width: 2.8vw;
            height: 2.8vw;
            min-width: 30px;
            max-width: 45px;
            min-height: 30px;
            max-height: 45px;
            border-radius: 50%;
            display: flex;
            align-items: center;
            justify-content: center;
            font-size: calc(10px + 0.6vw);
            font-weight: bold;
            cursor: pointer;
            z-index: 2;
        }
        
        /* 功率锁定按钮 */
        .power-lock-btn {
            position: absolute;
            top: -2.8vw;
            right: -1.2vw;
            width: 2vw;
            min-width: 20px;
            max-width: 30px;
            height: 2vw;
            min-height: 20px;
            max-height: 30px;
            border-radius: 50%;
            background: rgba(10, 25, 50, 0.85);
            display: flex;
            align-items: center;
            justify-content: center;
            cursor: pointer;
            z-index: 2;
            font-size: calc(8px + 0.6vw);
        }
        .power-lock-btn.locked {
            background: #ff5555cc;
            color: #fff;
            box-shadow: 0 0 16px #ff5555bb;
        }
        .power-lock-btn:hover {
            filter: brightness(1.2);
        }
        /* 功率吸附指示 */
        .power-snap-indicator {
            position: absolute;
            top: 50%;
            left: 50%;
            transform: translate(-50%, -50%);
            width: 2.2vw;
            min-width: 20px;
            max-width: 30px;
            height: 0.4vw;
            min-height: 3px;
            max-height: 5px;
            background: #00ffaa99;
            border-radius: 0.2vw;
            z-index: 0;
        }
        .power-indicator-vertical {
            font-size: calc(10px + 0.7vw);
            color: #00ffaa;
            font-weight: bold;
            margin-top: 1vw;
            text-align: center;
            width: 100%;
            padding: 0.7vw 0;
            background: rgba(0,0,0,0.38);
            border-radius: 0.9vw;
            border: 2px solid #00ffaa55;
            box-shadow: 0 0 10px #00ffaa33;
            letter-spacing: 0.05em;
        }
        /* 摇杆拖动手柄 */
        .drag-handle {
            position: absolute;
            top: 1vh;
            right: 1vw;
            width: 10vw;
            min-width: 15px;
            max-width: 25px;
            height: 10vw;
            min-height: 15px;
            max-height: 25px;
            color: #00aaff;
            opacity: 0.7;
            cursor: move;
            font-size: calc(12px + 0.8vw);
        }
    
        
        
        /* 视角切换按钮 */
        .view-toggle-btn {
           position: absolute;
            top: 1.5%;
            right: 1.5%;
            background: rgba(10, 25, 50, 0.8);
            padding: 1.2vh 1vw;
            border-radius: 2.5vh;
            font-size: calc(12px + 0.8vw);
            font-weight: bold;
            color: #00ffcc;
            z-index: 5;
            box-shadow: 0 0.5vh 1.5vh rgba(0, 0, 0, 0.6);
            border: 0.2vh solid rgba(0, 200, 255, 0.4);
            backdrop-filter: blur(8px);
            min-width: 80px;
            min-height: 4vh;
            display: flex;
            align-items: center;
            justify-content: center;
            cursor: pointer;
            transition: all 0.3s ease;
        }
        
        .view-toggle-btn:hover {
            background: rgba(20, 45, 80, 0.9);
            transform: scale(1.05);
        }
        
        .view-toggle-btn:active {
            transform: scale(0.95);
        }
        
        
        
        /* 光柱方向指示器 */
        .beam-direction {
            position: absolute;
            top: 10%;
            left: 50%;
            transform: translateX(-50%);
            background: rgba(0, 200, 255, 0.3);
            padding: 0.5vh 1.5vw;
            border-radius: 1.5vh;
            font-size: calc(8px + 0.7vw);
            font-weight: bold;
            color: #00ffaa;
            z-index: 9;
            display: none;
            max-width: 90%;
            white-space: nowrap;
            overflow: hidden;
            text-overflow: ellipsis;
        }
        
        /* 第一人称准星 */
        .crosshair {
            position: absolute;
            top: 50%;
            left: 50%;
            transform: translate(-50%, -50%);
            width: 30px;
            height: 30px;
            pointer-events: none;
            z-index: 15;
            display: none;
        }
        
        .crosshair::before, .crosshair::after {
            content: '';
            position: absolute;
            background: #00ffcc;
            box-shadow: 0 0 5px rgba(0, 255, 200, 0.8);
        }
        
        .crosshair::before {
            width: 4px;
            height: 20px;
            left: 50%;
            top: 50%;
            transform: translate(-50%, -50%);
        }
        
        .crosshair::after {
            width: 20px;
            height: 4px;
            left: 50%;
            top: 50%;
            transform: translate(-50%, -50%);
        }
        
        /* 触摸控制面板 */
        .touch-controls {
            position: absolute;
            bottom: 0;
            left: 0;
            width: 100%;
            height: 100%;
            z-index: 5;
            display: none;
        }
        
        .touch-left {
            position: absolute;
            left: 0;
            bottom: 0;
            width: 50%;
            height: 100%;
            background: rgba(0, 0, 0, 0.1);
        }
        
        .touch-right {
            position: absolute;
            right: 0;
            bottom: 0;
            width: 50%;
            height: 100%;
            background: rgba(0, 0, 0, 0.1);
        }
        
        .touch-sensitivity {
            position: absolute;
            top: 10px;
            right: 10px;
            background: rgba(10, 25, 50, 0.8);
            padding: 8px 15px;
            border-radius: 20px;
            z-index: 20;
            display: flex;
            align-items: center;
            gap: 10px;
        }
        
        .touch-sensitivity label {
            color: #00ffaa;
            font-size: 14px;
        }
        
        .touch-sensitivity input {
            width: 100px;
        }
        
        @keyframes pulse {
            0% { opacity: 0.7; }
            50% { opacity: 1; }
            100% { opacity: 0.7; }
        }
        
        /* 变形虫指示器 */
        .amoeba-indicator {
            position: absolute;
            top: 8%;
            left: 1.5vw;
            background: rgba(10, 25, 50, 0.8);
            padding: 0.8vh 1.5vw;
            border-radius: 2vh;
            font-size: calc(10px + 0.8vw);
            font-weight: bold;
            color: #ff88ff;
            z-index: 5;
            box-shadow: 0 0.5vh 1.5vh rgba(0, 0, 0, 0.6);
            border: 0.2vh solid rgba(200, 0, 255, 0.4);
            backdrop-filter: blur(8px);
            display: none;
        }
        
        /* 设备检测类 */
        .mobile-controls {
            display: none;
        }
        
        @media (max-width: 768px) {
            .mini-map-container {
                right: 5%;
                bottom: 15vh;
            }
            
            .joystick-container {
                left: 3%;
                bottom: 12%;
            }
            
            .power-control-vertical {
                right: 3vw;
                bottom: 3vw;
            }
            
            .beam-direction {
                top: 8%;
                font-size: calc(8px + 0.6vw);
            }
        }
        
        @media (max-width: 480px) {
            #scoreDisplay {
                font-size: 14px;
                padding: 0.8vh 2vw;
            }
            
            .view-toggle-btn {
                font-size: 14px;
                padding: 0.8vh 2vw;
            }
            
            .amoeba-indicator {
                font-size: 12px;
                padding: 0.6vh 2vw;
            }
            
            .mini-map-container {
                width: 120px;
                height: 120px;
                padding-top: 120px;
                bottom: 140px;
                right: 10px;
            }
            
            .map-title {
                font-size: 12px;
                top: -25px;
            }
            
            .height-indicator {
                font-size: 12px;
                top: 65%;
                left: 70%;
            }
            
            .joystick-container {
                width: 100px;
                height: 100px;
                left: 10px;
                bottom: 100px;
            }
            
            .joystick-head {
                font-size: 14px;
            }
            
            .joystick-stats {
                font-size: 12px;
            }
            
            .power-control-vertical {
                width: 50px;
                height: 200px;
                right: 10px;
                bottom: 10px;
            }
            
            .power-slider-vertical input[type="range"] {
                min-width: 40px;
            }
            
            .power-gear {
                min-width: 30px;
                min-height: 30px;
                font-size: 12px;
            }
            
            .power-lock-btn {
                min-width: 20px;
                min-height: 20px;
                font-size: 10px;
            }
            
            .power-indicator-vertical {
                font-size: 12px;
            }
            
            .drag-handle {
                font-size: 16px;
            }
        }
    </style>
</head>
<body>
    <div class="game-container">
        <div id="gameCanvas"></div>
        
        <!-- 分数显示 -->
        <div id="scoreDisplay">🧬: 0</div>
        
        <!-- 视角切换按钮 -->
        <div class="view-toggle-btn" id="viewToggleBtn">👁️<u>V</u></div>
        
        <!-- 第一人称提示 -->
        <div class="fp-indicator" id="fpIndicator"> 👁️/🎥 <u>V</u></div>
        
        <!-- 光柱方向指示器 -->
        <div class="beam-direction" id="beamDirection">光柱方向: 0°</div>
        
        <!-- 加速度指示器 -->
        <div class="acceleration-indicator" id="accelerationIndicator">📈: 0.0x</div>
        
        <!-- 变形虫指示器 -->
        <div class="amoeba-indicator" id="amoebaIndicator">变形虫: 5</div>
        
        <!-- 第一人称准星 -->
        <div class="crosshair" id="crosshair"></div>
        
        <!-- 小地图 -->
        <div class="map-title">蛇头所在XZ平面视图</div>
        <div class="mini-map-container">
            <canvas id="miniMap"></canvas>
        </div>
        <div class="height-indicator">y: 0</div>
        
        <!-- 虚拟摇杆 -->
        <div class="joystick-container" id="joystickContainer">
            <div class="drag-handle">↕</div>
            <div class="joystick-base">
                <div class="joystick-head" id="joystickHead" style="transform: translate(-1.94643px, -16.3393px); top: 3px; left: 21px;">
                    <div class="joystick-stats">0°<br>50%</div>
                </div>
                <div class="direction-indicator" id="directionIndicator">↑ 0°</div>
            </div>
        </div>
        
        <!-- 垂直功率控制（档把模式） -->
        <div class="power-control-vertical" id="powerControlVertical">
            <div class="power-slider-vertical">
                <div class="power-snap-indicator"></div>
                <div class="power-markers">
                    <div class="power-marker low"></div>
                    <div class="power-marker medium"></div>
                    <div class="power-marker high"></div>
                </div>
                <input type="range" min="0" max="100" value="50" id="powerSliderVertical" orient="vertical">
                <div class="power-gear" id="powerGear">P</div>
                <div class="power-lock-btn" id="powerLockBtn">🔓</div>
            </div>
            <div class="power-indicator-vertical" id="powerIndicatorVertical">0.50</div>
        </div>
        
        <!-- 游戏结束 -->
        <div class="game-over" id="gameOver">
            <div class="game-over-content">
                <h2>游戏结束!</h2>
                <div class="final-score">分数: <span id="finalScore">0</span></div>
                <button class="restart-btn" id="restartBtn">重新开始</button>
            </div>
        </div>
        
        <!-- 暂停界面 -->
        <div class="pause-overlay" id="pauseOverlay">
            <div class="pause-text">游戏暂停</div>
        </div>
        
        <!-- 触摸控制区域 -->
        <div class="touch-controls" id="touchControls">
            <div class="touch-left" id="touchLeft"></div>
            <div class="touch-right" id="touchRight"></div>
            <input type="range" min="1" max="10" value="5" id="sensitivitySlider">
        </div>
    </div>

    <script>
        // 游戏变量
        let scene, camera, renderer, controls;
        let snake = [], foods = [], obstacles = [], aiSnakes = [], algae = [], kelps = [], amoebas = [], diatoms = [], segmentedWorms = [];
        let score = 0 ; // 分数
        let snakeLength = 5 ; // 初始蛇身长度
        let gameRunning = true ; // 游戏是否运行
        let gridHelper; 
        let snakeBodyMaterials = [];
        let frameCount = 0;
        let lastFpsUpdate = 0;
        let firstPersonMode = false; // 第一人称模式标志
        let gameStartTime = 0;
        let WORLD_SIZE = 2000;
        let OBSTACLE_COUNT = 15;
        let FOOD_COUNT = 900;
        let AI_SNAKE_COUNT = 40;
        let ALGAE_COUNT = 200;
        let KELP_COUNT = 30;
        let AMOEBA_COUNT = 5; // 变形虫数量
        let DIATOM_COUNT = 15; // 硅藻数量
        let WORM_COUNT = 30;   // 多段虫数量
        
        // 平滑移动变量
        let moveProgress = 0;
        const MOVE_DISTANCE = 1;
        const MOVE_DURATION = 10;
        let moveStartTime = 0;
        let isMoving = false;
        let targetPosition = new THREE.Vector3(0, 0, 0);
        
        // 方向控制变量
        let horizontalAngle = Math.PI / 3;
        let verticalAngle = 0;
        let targetVerticalAngle = 0;
        const ROTATION_SPEED = 0.05;
        let direction = new THREE.Vector3(1, 0, 0);
        
        // 小地图变量
        let miniMapCtx;
        let MINI_MAP_SIZE = 300;
        const Y_THRESHOLD = 40;
        
        // 摇杆变量
        let joystickActive = false;
        let joystickAngle = 0;
        let joystickPower = 0.5;
        const JOYSTICK_RADIUS = 70;
        
        // 键盘控制变量
        let keys = {};
        const BASE_KEY_SPEED = 0.005;
        const BASE_POWER_SPEED = 1;
        
        // 光柱指示线
        let pathCylinder;
        let pathCylinderSolidMaterial, pathCylinderWireMaterial;
        
        // 蛇身位置历史记录
        let positionHistory = [];
        const HISTORY_MAX_LENGTH = 10000;
        const SEGMENT_DISTANCE = 12;
        
        // 食物吸附变量
        const ATTRACTION_DISTANCE = 0;
        const ATTRACTION_SPEED = 100;
        
        // 功率锁定状态
        let powerLocked = false;
        
        // 鼠标控制变量
        let isMouseDown = false;
        let lastMouseX = 0;
        let lastMouseY = 0;
        const MOUSE_SENSITIVITY = 0.005;

        // 触摸控制变量
        let isTouching = false;
        let touchStartX = 0;
        let touchStartY = 0;
        let touchLastX = 0;
        let touchLastY = 0;
        let touchSensitivity = 10; // 默认灵敏度

        // GLTF加载器
        let gltfLoader = new THREE.GLTFLoader();
        let amoebaModel = null; // 存储加载的变形虫模型

        document.addEventListener('keydown', function(event) {
            switch (event.key) {
                case 'ArrowUp':
                    keys['w'] = true;
                    if (!keyAcceleration['w'].pressed) {
                        keyAcceleration['w'].pressed = true;
                        keyAcceleration['w'].startTime = performance.now();
                    }
                    break;
                case 'ArrowDown':
                    keys['s'] = true;
                    if (!keyAcceleration['s'].pressed) {
                        keyAcceleration['s'].pressed = true;
                        keyAcceleration['s'].startTime = performance.now();
                    }
                    break;
                case 'ArrowLeft':
                    keys['a'] = true;
                    if (!keyAcceleration['a'].pressed) {
                        keyAcceleration['a'].pressed = true;
                        keyAcceleration['a'].startTime = performance.now();
                    }
                    break;
                case 'ArrowRight':
                    keys['d'] = true;
                    if (!keyAcceleration['d'].pressed) {
                        keyAcceleration['d'].pressed = true;
                        keyAcceleration['d'].startTime = performance.now();
                    }
                    break;
            }
        });

        document.addEventListener('keyup', function(event) {
            switch (event.key) {
                case 'ArrowUp':
                    keys['w'] = false;
                    keyAcceleration['w'].pressed = false;
                    keyAcceleration['w'].acceleration = 0;
                    document.getElementById('accelerationIndicator').style.display = 'none';
                    break;
                case 'ArrowDown':
                    keys['s'] = false;
                    keyAcceleration['s'].pressed = false;
                    keyAcceleration['s'].acceleration = 0;
                    document.getElementById('accelerationIndicator').style.display = 'none';
                    break;
                case 'ArrowLeft':
                    keys['a'] = false;
                    keyAcceleration['a'].pressed = false;
                    keyAcceleration['a'].acceleration = 0;
                    document.getElementById('accelerationIndicator').style.display = 'none';
                    break;
                case 'ArrowRight':
                    keys['d'] = false;
                    keyAcceleration['d'].pressed = false;
                    keyAcceleration['d'].acceleration = 0;
                    document.getElementById('accelerationIndicator').style.display = 'none';
                    break;
            }
        });

        // 键盘加速度系统
        let keyAcceleration = {
            a: { pressed: false, startTime: 0, acceleration: 0 },
            d: { pressed: false, startTime: 0, acceleration: 0 },
            w: { pressed: false, startTime: 0, acceleration: 0 },
            s: { pressed: false, startTime: 0, acceleration: 0 }
        };
        const ACCELERATION_RATE = 0.5; // 加速度增加速率
        const MAX_ACCELERATION = 12.0;    // 最大加速度值
        const ACCELERATION_DECAY = 0.5; // 加速度衰减速率
               
        // 初始化场景
        function init() {
            const loadingOverlay = document.getElementById('loadingOverlay');
            
            try {
                // 创建场景
                scene = new THREE.Scene();
                scene.background = new THREE.Color(0x0c1e3a);
                scene.fog = new THREE.FogExp2(0x0c1e3a, 0.0004);
                
                // 创建相机
                camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 1, 3000);
                camera.position.set(500, 400, 500);
                camera.lookAt(0, 0, 0);
                
                // 创建渲染器 - 添加像素比处理
                renderer = new THREE.WebGLRenderer({ 
                    antialias: true,
                    powerPreference: "high-performance"
                });
                
                // 设置设备像素比
                const pixelRatio = Math.min(window.devicePixelRatio, 2); // 限制最大像素比为2
                renderer.setPixelRatio(pixelRatio);
                renderer.setSize(window.innerWidth, window.innerHeight);
                renderer.shadowMap.enabled = true;
                renderer.shadowMap.type = THREE.PCFSoftShadowMap;
                document.getElementById('gameCanvas').appendChild(renderer.domElement);
                
                // 添加轨道控制
                controls = new THREE.OrbitControls(camera, renderer.domElement);
                controls.enableDamping = true;
                controls.dampingFactor = 0.05;
                controls.screenSpacePanning = false;
                controls.minDistance = 100;
                controls.maxDistance = 2500;
                controls.enablePan = true;
                
                // 添加光源
                const ambientLight = new THREE.AmbientLight(0x404040, 1.0);
                scene.add(ambientLight);
                
                const directionalLight = new THREE.DirectionalLight(0xffffff, 0.8);
                directionalLight.position.set(600, 700, 600);
                directionalLight.castShadow = true;
                directionalLight.shadow.mapSize.width = 1024;
                directionalLight.shadow.mapSize.height = 1024;
                scene.add(directionalLight);
                
                const backLight = new THREE.DirectionalLight(0x2255ff, 0.5);
                backLight.position.set(-400, 350, -400);
                scene.add(backLight);
                
                // 创建网格
                gridHelper = new THREE.GridHelper(WORLD_SIZE, 100, 0x305080, 0x203050);
                gridHelper.position.y = 0;
                scene.add(gridHelper);
                
                // 创建边界指示
                createBoundaryIndicators();
                
                // 初始化蛇身材质
                initSnakeMaterials();
                
                // 初始化蛇
                initSnake();
                
                // 创建食物
                createFoods();
                
                // 创建障碍物
                createObstacles();
                
                // 创建AI蛇
                createAISnakes();
                
                // 创建小球藻
                createAlgae();
                
                // 创建海带
                createKelps();
                
                // 创建变形虫
                createAmoebas();
                
                // 创建硅藻
                createDiatoms();
                
                // 创建多段虫
                createSegmentedWorms();
                
                // 创建光柱指示线
                createPathCylinder();
                
                // 初始化小地图
                initMiniMap();
                
                // 添加事件监听器
                window.addEventListener('resize', onWindowResize);
                window.addEventListener('keydown', onKeyDown);
                window.addEventListener('keyup', onKeyUp);
                document.getElementById('restartBtn').addEventListener('click', resetGame);
                document.getElementById('viewToggleBtn').addEventListener('click', toggleFirstPerson);
                
                // 摇杆事件
                setupJoystick();
                
                // 摇杆容器拖动事件
                setupJoystickDrag();
                
                // 功率控制事件
                setupPowerControlVertical();
                
                // 功率锁定按钮事件
                document.getElementById('powerLockBtn').addEventListener('click', togglePowerLock);
                
                // 鼠标事件监听器
                document.addEventListener('mousemove', onMouseMove);
                document.addEventListener('mousedown', function() {
                    if (firstPersonMode) {
                        document.body.requestPointerLock = document.body.requestPointerLock || 
                                                         document.body.mozRequestPointerLock ||
                                                         document.body.webkitRequestPointerLock;
                        document.body.requestPointerLock();
                    }
                });
                
                // 触摸事件监听器
                setupTouchControls();
                
                // 灵敏度控制
                document.getElementById('sensitivitySlider').addEventListener('input', function() {
                    touchSensitivity = parseInt(this.value);
                });
                
                // 记录游戏开始时间
                gameStartTime = Date.now();
                
                // 隐藏加载提示
                setTimeout(() => {
                    if (loadingOverlay) loadingOverlay.style.display = 'none';
                }, 0);
                
                // 开始动画循环
                animate();
                
                // 开始第一次移动
                startMove();
            } catch (error) {
                console.error("初始化错误:", error);
                if (loadingOverlay) {
                    loadingOverlay.innerHTML = `
                        <div style="text-align: center; padding: 20px; background: rgba(10,25,50,0.9); border-radius: 10px; max-width: 500px;">
                            <h2 style="color: #ff5555; margin-bottom: 15px;">初始化失败</h2>
                            <p style="color: #a0d5ff; margin-bottom: 20px;">${error.message || "无法初始化3D渲染器"}</p>
                            <p style="color: #a0d5ff; margin-bottom: 20px;">请确保您的设备支持WebGL</p>
                            <button style="background: linear-gradient(45deg, #00ccaa, #00a0ff); 
                                border: none; padding: 12px 30px; border-radius: 50px; 
                                color: white; cursor: pointer; font-size: 1rem;"
                                onclick="location.reload()">重新加载</button>
                        </div>
                    `;
                }
            }
        }
        
        // 创建硅藻
        function createDiatoms() {
            for (let i = 0; i < DIATOM_COUNT; i++) {
                const diatomGroup = new THREE.Group();
                
                // 创建硅藻外壳（使用BufferGeometry）
                const shellGeometry = new THREE.IcosahedronGeometry(30, 2);
                const shellMaterial = new THREE.MeshPhongMaterial({ 
                    color: 0x77ddff,
                    transparent: true,
                    opacity: 0.8,
                    shininess: 80,
                    emissive: 0x4488cc,
                    emissiveIntensity: 0.2,
                    wireframe: false
                });
                
                const shell = new THREE.Mesh(shellGeometry, shellMaterial);
                shell.castShadow = true;
                shell.receiveShadow = true;
                diatomGroup.add(shell);
                
                // 创建硅藻内部（可食用部分）
                const innerGeometry = new THREE.SphereGeometry(20, 16, 16);
                const innerMaterial = new THREE.MeshPhongMaterial({ 
                    color: 0xffaa77,
                    emissive: 0xcc8855,
                    emissiveIntensity: 0.4
                });
                
                const inner = new THREE.Mesh(innerGeometry, innerMaterial);
                inner.visible = false; // 初始不可见
                inner.castShadow = true;
                inner.receiveShadow = true;
                diatomGroup.add(inner);
                
                // 随机位置
                diatomGroup.position.set(
                    Math.floor(Math.random() * WORLD_SIZE) - WORLD_SIZE/2,
                    Math.floor(Math.random() * 100) - 50,
                    -(Math.floor(Math.random() * WORLD_SIZE) - WORLD_SIZE/2)
                );
                
                // 随机旋转
                diatomGroup.rotation.x = Math.random() * Math.PI;
                diatomGroup.rotation.y = Math.random() * Math.PI * 2;
                diatomGroup.rotation.z = Math.random() * Math.PI;
                
                // 设置硅藻属性
                diatomGroup.userData = {
                    type: 'diatom',
                    durability: 3, // 耐久度
                    shell: shell,
                    inner: inner,
                    hitCount: 0 // 被撞击次数
                };
                
                scene.add(diatomGroup);
                diatoms.push(diatomGroup);
            }
        }
        
        // 创建变形虫
        function createAmoebas() {
            gltfLoader.load('amoeba.glb', function(gltf) {
                const model = gltf.scene;
                
                for (let i = 0; i < AMOEBA_COUNT; i++) {
                    const amoeba = model.clone();
                    
                    // 设置位置（y=-200）
                    amoeba.position.set(
                        Math.floor(Math.random() * WORLD_SIZE) - WORLD_SIZE/2,
                        -200,
                        -(Math.floor(Math.random() * WORLD_SIZE) - WORLD_SIZE/2)
                    );
                    
                    // 随机缩放
                    const scale = 0.8 + Math.random() * 0.4;
                    amoeba.scale.set(scale, scale, scale);
                    
                    // 随机旋转
                    amoeba.rotation.x = Math.random() * Math.PI;
                    amoeba.rotation.y = Math.random() * Math.PI * 2;
                    amoeba.rotation.z = Math.random() * Math.PI;
                    
                    // 设置动画参数
                    amoeba.userData = {
                        pulseSpeed: 0.5 + Math.random() * 0.5,
                        rotationSpeed: new THREE.Vector3(
                            (Math.random() - 0.5) * 0.01,
                            (Math.random() - 0.5) * 0.01,
                            (Math.random() - 0.5) * 0.01
                        )
                    };
                    
                    scene.add(amoeba);
                    amoebas.push(amoeba);
                }
                
                // 更新指示器
                document.getElementById('amoebaIndicator').textContent = `变形虫: ${amoebas.length}`;
                document.getElementById('amoebaIndicator').style.display = 'block';
                
            }, undefined, function(error) {
                console.error('Error loading amoeba model', error);
            });
        }
        
        // 更新变形虫
        function updateAmoebas(timestamp) {
            for (let i = 0; i < amoebas.length; i++) {
                const amoeba = amoebas[i];
                
                // 脉动效果
                const pulse = Math.sin(timestamp * 0.001 * amoeba.userData.pulseSpeed) * 0.1 + 1;
                amoeba.scale.set(pulse, pulse, pulse);
                
                // 旋转
                amoeba.rotation.x += amoeba.userData.rotationSpeed.x;
                amoeba.rotation.y += amoeba.userData.rotationSpeed.y;
                amoeba.rotation.z += amoeba.userData.rotationSpeed.z;
            }
        }
        
        // 检查与变形虫的碰撞
        function checkAmoebaCollision() {
            const headPos = snake[0].position;
            
            for (let i = amoebas.length - 1; i >= 0; i--) {
                const amoeba = amoebas[i];
                const distance = headPos.distanceTo(amoeba.position);
                
                if (distance < 50) {
                    // 增加分数
                    score += 200;
                    
                    // 添加蛇身段
                    addSnakeSegment();
                    addSnakeSegment();
                    addSnakeSegment();
                    
                    // 移除变形虫
                    scene.remove(amoeba);
                    amoebas.splice(i, 1);
                    
                    // 更新指示器
                    document.getElementById('amoebaIndicator').textContent = `变形虫: ${amoebas.length}`;
                    
                    // 更新UI
                    updateUI();
                    
                    break;
                }
            }
        }
        
        // 创建多段虫
        function createSegmentedWorms() {
            for (let i = 0; i < WORM_COUNT; i++) {
                const wormGroup = new THREE.Group();
                const segments = [];
                const segmentCount = 5 + Math.floor(Math.random() * 4); // 5-8节
                
                // 创建虫头
                const headGeometry = new THREE.SphereGeometry(12, 16, 16);
                const headMaterial = new THREE.MeshPhongMaterial({ 
                    color: 0xffaa77,
                    emissive: 0xcc8855,
                    emissiveIntensity: 0.3
                });
                
                const head = new THREE.Mesh(headGeometry, headMaterial);
                head.castShadow = true;
                head.receiveShadow = true;
                wormGroup.add(head);
                segments.push(head);
                
                // 创建虫身节段
                for (let j = 1; j < segmentCount; j++) {
                    const segmentGeometry = new THREE.TorusGeometry(8, 4, 8, 16);
                    const segmentMaterial = new THREE.MeshPhongMaterial({ 
                        color: 0xffaa77,
                        emissive: 0xcc8855,
                        emissiveIntensity: 0.2
                    });
                    
                    const segment = new THREE.Mesh(segmentGeometry, segmentMaterial);
                    segment.castShadow = true;
                    segment.receiveShadow = true;
                    
                    // 设置位置（连接在前一节段后面）
                    segment.position.y = -j * 22;
                    wormGroup.add(segment);
                    segments.push(segment);
                }
                
                // 随机位置
                wormGroup.position.set(
                    Math.floor(Math.random() * WORLD_SIZE) - WORLD_SIZE/2,
                    Math.floor(Math.random() * 200) - 100,
                    -(Math.floor(Math.random() * WORLD_SIZE) - WORLD_SIZE/2)
                );
                
                // 随机方向
                const direction = new THREE.Vector3(
                    Math.random() - 0.5,
                    Math.random() - 0.5,
                    Math.random() - 0.5
                ).normalize();
                
                // 设置多段虫属性
                wormGroup.userData = {
                    type: 'segmentedWorm',
                    segments: segments,
                    direction: direction,
                    speed: 0.5 + Math.random() * 0.5,
                    waveOffset: Math.random() * Math.PI * 2,
                    turnCounter: 0,
                    turnInterval: 100 + Math.floor(Math.random() * 100)
                };
                
                scene.add(wormGroup);
                segmentedWorms.push(wormGroup);
            }
        }
        
        // 更新多段虫
        function updateSegmentedWorms(timestamp) {
            for (let i = 0; i < segmentedWorms.length; i++) {
                const worm = segmentedWorms[i];
                const segments = worm.userData.segments;
                
                // 随机改变方向
                worm.userData.turnCounter++;
                if (worm.userData.turnCounter > worm.userData.turnInterval) {
                    worm.userData.direction = new THREE.Vector3(
                        Math.random() - 0.5,
                        Math.random() - 0.5,
                        Math.random() - 0.5
                    ).normalize();
                    worm.userData.turnCounter = 0;
                    worm.userData.turnInterval = 100 + Math.floor(Math.random() * 100);
                }
                
                // 移动整个虫
                worm.position.add(worm.userData.direction.clone().multiplyScalar(worm.userData.speed));
                
                // 边界检查
                if (Math.abs(worm.position.x) > WORLD_SIZE/2 - 50) {
                    worm.userData.direction.x *= -1;
                    worm.position.x = Math.sign(worm.position.x) * (WORLD_SIZE/2 - 50);
                }
                if (Math.abs(worm.position.y) > WORLD_SIZE/2 - 50) {
                    worm.userData.direction.y *= -1;
                    worm.position.y = Math.sign(worm.position.y) * (WORLD_SIZE/2 - 50);
                }
                if (Math.abs(worm.position.z) > WORLD_SIZE/2 - 50) {
                    worm.userData.direction.z *= -1;
                    worm.position.z = Math.sign(worm.position.z) * (WORLD_SIZE/2 - 50);
                }
                
                // 波浪形运动
                const waveIntensity = 5;
                const waveSpeed = 0.02;
                
                for (let j = 0; j < segments.length; j++) {
                    if (j > 0) {
                        // 波浪运动偏移
                        const waveOffset = j * 0.5 + worm.userData.waveOffset;
                        const waveX = Math.sin(timestamp * 0.001 * waveSpeed + waveOffset) * waveIntensity;
                        const waveY = Math.cos(timestamp * 0.001 * waveSpeed + waveOffset) * waveIntensity;
                        
                        // 应用波浪效果
                        segments[j].position.x = waveX;
                        segments[j].position.y = -j * 22 + waveY;
                    }
                    
                    // 指向前一节段
                    if (j > 0) {
                        segments[j].lookAt(segments[j-1].position);
                    }
                }
            }
        }
        
        // 检查硅藻碰撞
        function checkDiatomCollision() {
            const headPos = snake[0].position;
            
            for (let i = diatoms.length - 1; i >= 0; i--) {
                const diatom = diatoms[i];
                const distance = headPos.distanceTo(diatom.position);
                
                if (distance < 40) {
                    // 撞击硅藻
                    diatom.userData.hitCount++;
                    
                    // 更新外壳材质显示裂纹效果
                    const hitRatio = diatom.userData.hitCount / diatom.userData.durability;
                    diatom.userData.shell.material.color = new THREE.Color(
                        0.7 - hitRatio * 0.3,
                        0.7 - hitRatio * 0.5,
                        1.0 - hitRatio * 0.2
                    );
                    
                    // 显示裂纹粒子效果
                    createCrackParticles(diatom.position);
                    
                    // 如果耐久度耗尽
                    if (diatom.userData.hitCount >= diatom.userData.durability) {
                        // 显示内部可食用部分
                        diatom.userData.inner.visible = true;
                        diatom.userData.shell.visible = false;
                        
                        // 增加分数
                        score += 100;
                        
                        // 更新UI
                        updateUI();
                    }
                    
                    break;
                }
            }
        }
        
        // 创建裂纹粒子效果
        function createCrackParticles(position) {
            const particleCount = 20;
            const particleGeometry = new THREE.BufferGeometry();
            const positions = new Float32Array(particleCount * 3);
            const colors = new Float32Array(particleCount * 3);
            const sizes = new Float32Array(particleCount);
            
            for (let i = 0; i < particleCount; i++) {
                const i3 = i * 3;
                
                // 随机位置
                positions[i3] = position.x + (Math.random() - 0.5) * 20;
                positions[i3 + 1] = position.y + (Math.random() - 0.5) * 20;
                positions[i3 + 2] = position.z + (Math.random() - 0.5) * 20;
                
                // 随机颜色（浅蓝色）
                colors[i3] = 0.6 + Math.random() * 0.4;
                colors[i3 + 1] = 0.8 + Math.random() * 0.2;
                colors[i3 + 2] = 1.0;
                
                // 随机大小
                sizes[i] = 1 + Math.random() * 3;
            }
            
            particleGeometry.setAttribute('position', new THREE.BufferAttribute(positions, 3));
            particleGeometry.setAttribute('color', new THREE.BufferAttribute(colors, 3));
            particleGeometry.setAttribute('size', new THREE.BufferAttribute(sizes, 1));
            
            const particleMaterial = new THREE.PointsMaterial({
                size: 3,
                vertexColors: true,
                transparent: true,
                opacity: 0.8,
                sizeAttenuation: true
            });
            
            const particles = new THREE.Points(particleGeometry, particleMaterial);
            scene.add(particles);
            
            // 设置粒子消失
            setTimeout(() => {
                scene.remove(particles);
            }, 1000);
        }
        
        // 检查多段虫碰撞
        function checkWormCollision() {
            const headPos = snake[0].position;
            
            for (let i = segmentedWorms.length - 1; i >= 0; i--) {
                const worm = segmentedWorms[i];
                const segments = worm.userData.segments;
                
                for (let j = segments.length - 1; j >= 0; j--) {
                    const segment = segments[j];
                    const distance = headPos.distanceTo(segment.getWorldPosition(new THREE.Vector3()));
                    
                    if (distance < 25) {
                        // 如果虫只有一节，可以食用
                        if (segments.length === 1) {
                            // 增加分数
                            score += 80;
                            
                            // 移除虫
                            scene.remove(worm);
                            segmentedWorms.splice(i, 1);
                            
                            // 更新UI
                            updateUI();
                            
                            break;
                        } 
                        // 否则分裂虫
                        else if (j > 0 && j < segments.length - 1) {
                            splitWorm(worm, j);
                            break;
                        }
                    }
                }
            }
        }
        
        // 分裂多段虫
        function splitWorm(worm, splitIndex) {
            const segments = worm.userData.segments;
            
            // 创建新虫（从分裂点到尾部）
            const newWormGroup = new THREE.Group();
            const newSegments = [];
            
            // 复制位置和方向
            newWormGroup.position.copy(worm.position);
            newWormGroup.userData = {
                type: 'segmentedWorm',
                segments: newSegments,
                direction: worm.userData.direction.clone(),
                speed: worm.userData.speed,
                waveOffset: Math.random() * Math.PI * 2,
                turnCounter: 0,
                turnInterval: 100 + Math.floor(Math.random() * 100)
            };
            
            // 移动节段到新虫
            for (let i = splitIndex + 1; i < segments.length; i++) {
                worm.remove(segments[i]);
                newWormGroup.add(segments[i]);
                newSegments.push(segments[i]);
            }
            
            // 更新原虫的节段
            worm.userData.segments = segments.slice(0, splitIndex + 1);
            
            // 添加到场景和数组
            scene.add(newWormGroup);
            segmentedWorms.push(newWormGroup);
        }
        
        // 检查硅藻内部碰撞（食用）
        function checkDiatomInnerCollision() {
            const headPos = snake[0].position;
            
            for (let i = diatoms.length - 1; i >= 0; i--) {
                const diatom = diatoms[i];
                
                // 只有当内部可见时才可食用
                if (diatom.userData.inner.visible) {
                    const distance = headPos.distanceTo(diatom.position);
                    
                    if (distance < 30) {
                        // 增加分数
                        score += 150;
                        
                        // 添加蛇身段
                        addSnakeSegment();
                        addSnakeSegment();
                        addSnakeSegment();
                        
                        // 移除硅藻
                        scene.remove(diatom);
                        diatoms.splice(i, 1);
                        
                        // 更新UI
                        updateUI();
                        
                        break;
                    }
                }
            }
        }
        
        // 设置触摸控制
        function setupTouchControls() {
            const touchLeft = document.getElementById('touchLeft');
            const touchRight = document.getElementById('touchRight');
            
            touchLeft.addEventListener('touchstart', onTouchStart);
            touchLeft.addEventListener('touchmove', onTouchMove);
            touchLeft.addEventListener('touchend', onTouchEnd);
            
            touchRight.addEventListener('touchstart', onTouchStart);
            touchRight.addEventListener('touchmove', onTouchMove);
            touchRight.addEventListener('touchend', onTouchEnd);
        }
        
        // 触摸开始
        function onTouchStart(e) {
            if (!firstPersonMode || !gameRunning) return;
            
            isTouching = true;
            const touch = e.touches[0];
            touchStartX = touch.clientX;
            touchStartY = touch.clientY;
            touchLastX = touch.clientX;
            touchLastY = touch.clientY;
        }
        
        // 触摸移动
        function onTouchMove(e) {
            if (!isTouching || !firstPersonMode || !gameRunning) return;
            
            const touch = e.touches[0];
            const deltaX = touch.clientX - touchLastX;
            const deltaY = touch.clientY - touchLastY;
            
            touchLastX = touch.clientX;
            touchLastY = touch.clientY;
            
            // 根据灵敏度调整
            const sensitivity = touchSensitivity * 0.001;
            
            // 更新水平角度
            horizontalAngle += deltaX * sensitivity;
            
            // 更新垂直角度并限制范围
            joystickPower -= deltaY * sensitivity;
            joystickPower = Math.max(0, Math.min(1, joystickPower));
           
            
            // 更新方向向量
            updateDirectionVector();
            
            // 更新功率控制显示
            updatePowerControl();
            
            // 更新摇杆显示
            updateJoystickDisplay();
        }
        
        // 触摸结束
        function onTouchEnd() {
            isTouching = false;
        }
        
        // 鼠标移动事件处理
        function onMouseMove(event) {
            if (!firstPersonMode || !gameRunning) return;
            
            // 计算鼠标移动量
            const movementX = event.movementX || event.mozMovementX || event.webkitMovementX || 0;
            const movementY = event.movementY || event.mozMovementY || event.webkitMovementY || 0;
            
            // 更新水平角度
            horizontalAngle += movementX * MOUSE_SENSITIVITY;
            
            // 更新垂直角度并限制范围
            joystickPower = Math.max(0, Math.min(1, joystickPower - movementY * MOUSE_SENSITIVITY));
            
            // 更新方向向量
            updateDirectionVector();
            
            // 更新光柱方向指示器
            updateBeamIndicator();
        }
        
        // 更新光柱方向指示器
        function updateBeamIndicator() {
            const beamIndicator = document.getElementById('beamDirection');
            const angleDeg = Math.round((horizontalAngle * 180 / Math.PI + 360) % 360);
            const powerPercent = Math.round(joystickPower * 100);
            const elevationAngle = Math.round(targetVerticalAngle * 180 / Math.PI);
            
            beamIndicator.textContent = `光柱方向: ${angleDeg}° | 功率: ${powerPercent}% | 仰角: ${elevationAngle}°`;
        }
        
        // 切换第一人称模式
        function toggleFirstPerson() {
            firstPersonMode = !firstPersonMode;
            const btn = document.getElementById('viewToggleBtn');
            const indicator = document.getElementById('fpIndicator');
            const beamIndicator = document.getElementById('beamDirection');
            const crosshair = document.getElementById('crosshair');
            const touchControls = document.getElementById('touchControls');
            
            if (firstPersonMode) {
                btn.textContent = "🎥V";
                btn.style.backgroundColor = "rgba(20, 60, 120, 0.9)";
                indicator.style.display = 'block';
                beamIndicator.style.display = 'block';
                crosshair.style.display = 'block';
                touchControls.style.display = 'block';
                
                
                // 在第一人称模式下隐藏蛇头
                if (snake.length > 0) {
                    snake[0].visible = false;
                }
                
                // 禁用轨道控制
                controls.enabled = false;
                
                // 切换光柱材质为线框
                if (pathCylinder) {
                    pathCylinder.material = pathCylinderWireMaterial;
                }
                
                // 请求指针锁定
                if ('requestPointerLock' in document.body) {
                    document.body.requestPointerLock();
                }
            } else {
                btn.textContent = "👁️V";
                btn.style.backgroundColor = "rgba(10, 25, 50, 0.8)";
                indicator.style.display = 'none';
                beamIndicator.style.display = 'none';
                crosshair.style.display = 'none';
                touchControls.style.display = 'none';
                
                // 在第三人称模式下显示蛇头
                if (snake.length > 0) {
                    snake[0].visible = true;
                }
                
                // 启用轨道控制
                controls.enabled = true;
                
                // 切换光柱材质为实心
                if (pathCylinder) {
                    pathCylinder.material = pathCylinderSolidMaterial;
                }
                
                // 退出指针锁定
                if ('exitPointerLock' in document) {
                    document.exitPointerLock();
            }
            }
            
            // 更新光柱指示器
            updateBeamIndicator();
        }
        
        // 更新第一人称相机
        function updateFirstPersonCamera() {
            if (!snake.length) return;
            
            const head = snake[0];
            
            // 设置相机位置在蛇头前方
            const offset = direction.clone().multiplyScalar(30); // 30单位前方
            camera.position.copy(head.position).add(offset);
            
            // 设置相机朝向与蛇前进方向相同
            camera.lookAt(head.position.clone().add(direction.clone().multiplyScalar(100)));
            
            // 更新光柱方向指示器
            const beamIndicator = document.getElementById('beamDirection');
            const angleDeg = Math.round((horizontalAngle * 180 / Math.PI + 360) % 360);
            const powerPercent = Math.round(joystickPower * 100);
            const elevationAngle = Math.round(verticalAngle * 180 / Math.PI);
            
            beamIndicator.textContent = `光柱方向: ${angleDeg}° | 功率: ${powerPercent}% | 仰角: ${elevationAngle}°`;
        }
        
        // 设置垂直功率控制
        function setupPowerControlVertical() {
            const powerSlider = document.getElementById('powerSliderVertical');
            const powerIndicator = document.getElementById('powerIndicatorVertical');
            const powerGear = document.getElementById('powerGear');
            
            // 功率滑块事件
            powerSlider.addEventListener('input', function() {
                let rawValue = parseInt(this.value);
                
                // 在0.5功率附近添加吸附效果 (48-52之间吸附到50)
                if (rawValue >= 48 && rawValue <= 52) {
                    rawValue = 50;
                    this.value = 50;
                }
                
                joystickPower = rawValue / 100;
                
                // 更新功率指示器
                powerIndicator.textContent = joystickPower.toFixed(2);
                
                
                
                // 更新档把颜色
                updateGearColor();
                
                // 更新摇杆显示
                updateJoystickDisplay();
            });
            
            // 初始化档把颜色
            updateGearColor();
        }
        
        // 切换功率锁定状态
        function togglePowerLock() {
            const lockBtn = document.getElementById('powerLockBtn');
            powerLocked = !powerLocked;
            
            if (powerLocked) {
                lockBtn.classList.add('locked');
                lockBtn.textContent = '🔒';
            } else {
                lockBtn.classList.remove('locked');
                lockBtn.textContent = '🔓';
            }
        }
        
        // 更新档把颜色
        function updateGearColor() {
            const powerGear = document.getElementById('powerGear');
            if (!powerGear) return;
            
            // 根据功率值设置不同颜色
            if (joystickPower < 0.3) {
                powerGear.style.background = 'linear-gradient(45deg, #ff5555, #ff9966)';
            } else if (joystickPower < 0.7) {
                powerGear.style.background = 'linear-gradient(45deg, #44aa66, #66cc88)';
            } else {
                powerGear.style.background = 'linear-gradient(45deg, #0066cc, #0099ff)';
            }
        }
        
        // 更新摇杆显示
        function updateJoystickDisplay() {
            const joystickStats = document.querySelector('.joystick-stats');
            if (!joystickStats) return;
            
            const angleDeg = Math.round((horizontalAngle * 180 / Math.PI + 360) % 360);
            const powerPercent = Math.round(joystickPower * 100);
            
            joystickStats.innerHTML = `${angleDeg}°<br>${powerPercent}%`;
        }
        
        // 创建AI蛇
        function createAISnakes() {
            for (let i = 0; i < AI_SNAKE_COUNT; i++) {
                const aiSnake = {
                    body: [],
                    direction: new THREE.Vector3(
                        Math.random() * 2 - 1,
                        Math.random() * 2 - 1,
                        Math.random() * 2 - 1
                    ).normalize(),
                    length: 3,
                    positionHistory: [],
                    color: new THREE.Color(0xffaa00),
                    speed: Math.random() * 0.5 + 0.8,
                    turnCounter: 0,
                    turnInterval: Math.floor(Math.random() * 100) + 50
                };
                
                // 创建AI蛇头
                const headGeometry = new THREE.BoxGeometry(10, 10, 10);
                const headMaterial = new THREE.MeshPhongMaterial({ 
                    color: aiSnake.color,
                    emissive: 0xaa5500,
                    emissiveIntensity: 0.3
                });
                const head = new THREE.Mesh(headGeometry, headMaterial);
                
                // 随机位置
                head.position.set(
                    Math.floor(Math.random() * WORLD_SIZE) - WORLD_SIZE/2,
                    Math.floor(Math.random() * WORLD_SIZE) - WORLD_SIZE/2,
                    -(Math.floor(Math.random() * WORLD_SIZE) - WORLD_SIZE/2)
                );
                
                head.castShadow = true;
                head.receiveShadow = true;
                scene.add(head);
                aiSnake.body.push(head);
                
                // 创建初始蛇身
                for (let j = 1; j < aiSnake.length; j++) {
                    const segmentGeometry = new THREE.BoxGeometry(9, 9, 9);
                    const segmentMaterial = new THREE.MeshPhongMaterial({ 
                        color: aiSnake.color,
                        emissive: 0xaa5500,
                        emissiveIntensity: 0.2
                    });
                    const segment = new THREE.Mesh(segmentGeometry, segmentMaterial);
                    
                    segment.position.copy(head.position);
                    segment.position.x -= aiSnake.direction.x * 10 * j;
                    segment.position.y -= aiSnake.direction.y * 10 * j;
                    segment.position.z -= aiSnake.direction.z * 10 * j;
                    
                    segment.castShadow = true;
                    segment.receiveShadow = true;
                    scene.add(segment);
                    aiSnake.body.push(segment);
                }
                
                aiSnakes.push(aiSnake);
            }
        }
        
        // 创建小球藻
        function createAlgae() {
            for (let i = 0; i < ALGAE_COUNT; i++) {
                const algaeGroup = new THREE.Group();
                
                // 创建小球藻主体
                const mainGeometry = new THREE.SphereGeometry(12, 8, 8);
                mainGeometry.scale(
                    0.8 + Math.random() * 0.4,
                    0.8 + Math.random() * 0.4,
                    0.8 + Math.random() * 0.4
                );
                
                // 粗糙材质
                const algaeMaterial = new THREE.MeshPhongMaterial({ 
                    color: 0x55ff55,
                    transparent: true,
                    opacity: 0.7,
                    shininess: 20,
                    roughness: 0.9,
                    emissive: 0x22aa22,
                    emissiveIntensity: 0.3
                });
                
                const algaeMesh = new THREE.Mesh(mainGeometry, algaeMaterial);
                algaeMesh.castShadow = true;
                algaeMesh.receiveShadow = true;
                algaeGroup.add(algaeMesh);
                
                // 添加絮状效果
                const fluffCount = 2 + Math.floor(Math.random() * 8);
                for (let j = 0; j < fluffCount; j++) {
                    const fluffGeometry = new THREE.SphereGeometry(
                        2 + Math.random() * 4, 
                        6, 
                        6
                    );
                    
                    // 随机位置偏移
                    const offset = new THREE.Vector3(
                        (Math.random() - 0.5) * 25,
                        (Math.random() - 0.5) * 25,
                        (Math.random() - 0.5) * 25
                    );
                    
                    const fluffMesh = new THREE.Mesh(fluffGeometry, algaeMaterial);
                    fluffMesh.position.copy(offset);
                    fluffMesh.castShadow = true;
                    fluffMesh.receiveShadow = true;
                    algaeGroup.add(fluffMesh);
                }
                
                // 随机位置
                algaeGroup.position.set(
                    Math.floor(Math.random() * WORLD_SIZE) - WORLD_SIZE/2,
                    Math.floor(Math.random() * WORLD_SIZE) - WORLD_SIZE/2,
                    -(Math.floor(Math.random() * WORLD_SIZE) - WORLD_SIZE/2)
                );
                
                // 随机漂浮速度
                algaeGroup.userData = {
                    speed: Math.random() * 0.5 + 0.2,
                    direction: new THREE.Vector3(
                        Math.random() * 0.1 - 0.05,
                        Math.random() * 0.1 - 0.05,
                        Math.random() * 0.1 - 0.05
                    ),
                    rotationSpeed: new THREE.Vector3(
                        (Math.random() - 0.5) * 0.01,
                        (Math.random() - 0.5) * 0.01,
                        (Math.random() - 0.5) * 0.01
                    )
                };
                
                scene.add(algaeGroup);
                algae.push(algaeGroup);
            }
        }
        
        // 创建海带
        function createKelps() {
            for (let i = 0; i < KELP_COUNT; i++) {
                const kelpGroup = new THREE.Group();
                
                // 创建海带茎
                const stemGeometry = new THREE.CylinderGeometry(1.8, 2.0, 100, 12, 32);
                const stemMaterial = new THREE.MeshPhongMaterial({ 
                    color: 0x5533cc,
                    shininess: 60,
                    emissive: 0x331188,
                    emissiveIntensity: 0.3
                });
                
                // 扭曲茎干
                const positionAttribute = stemGeometry.attributes.position;
                for (let j = 0; j < positionAttribute.count; j++) {
                    const y = positionAttribute.getY(j);
                    const twistAmount = Math.sin(y * 0.1) * 7;
                    positionAttribute.setX(j, positionAttribute.getX(j) + twistAmount);
                }
                
                const stem = new THREE.Mesh(stemGeometry, stemMaterial);
                stem.position.y = 50;
                kelpGroup.add(stem);
                
                // 创建叶片材质
                const leafMaterial = new THREE.MeshPhongMaterial({ 
                    color: 0x8855ff,
                    shininess: 40,
                    emissive: 0x5533cc,
                    emissiveIntensity: 0.2,
                    side: THREE.DoubleSide,
                    transparent: true,
                    opacity: 0.85
                });
                
                // 创建多层叶片
                for (let j = 0; j < 6; j++) {
                    const layerHeight = j * 15;
                    
                    // 创建主叶片
                    const leafGeometry = new THREE.PlaneGeometry(35, 16, 16, 4);
                    
                    // 弯曲叶片
                    const positions = leafGeometry.attributes.position.array;
                    for (let k = 0; k < positions.length; k += 3) {
                        const y = positions[k + 1];
                        const bend = Math.sin((y + 8) * 0.3) * 8;
                        positions[k] += bend;
                    }
                    leafGeometry.attributes.position.needsUpdate = true;
                    
                    const leaf = new THREE.Mesh(leafGeometry, leafMaterial);
                    leaf.position.y = layerHeight;
                    leaf.rotation.x = Math.PI / 2;
                    
                    // 随机旋转角度
                    const rotationAngle = (j % 2 === 0 ? 1 : -1) * (Math.PI / 4 + Math.random() * 0.2);
                    leaf.rotation.z = rotationAngle;
                    
                    // 存储原始值用于动画
                    leaf.userData = {
                        originalX: leaf.position.x,
                        originalY: leaf.position.y,
                        originalRotationZ: leaf.rotation.z
                    };
                    
                    kelpGroup.add(leaf);
                    
                    // 添加对称叶片
                    const leaf2 = leaf.clone();
                    leaf2.rotation.z = -rotationAngle;
                    
                    // 存储原始值
                    leaf2.userData = {
                        originalX: leaf2.position.x,
                        originalY: leaf2.position.y,
                        originalRotationZ: leaf2.rotation.z
                    };
                    
                    kelpGroup.add(leaf2);
                    
                    // 添加小型侧叶
                    for (let s = 0; s < 2; s++) {
                        const sideLeafGeometry = new THREE.PlaneGeometry(15, 8, 8, 2);
                        
                        // 弯曲侧叶
                        const sidePositions = sideLeafGeometry.attributes.position.array;
                        for (let k = 0; k < sidePositions.length; k += 3) {
                            const y = sidePositions[k + 1];
                            const bend = Math.sin((y + 4) * 0.4) * 4;
                            sidePositions[k] += bend;
                        }
                        sideLeafGeometry.attributes.position.needsUpdate = true;
                        
                        const sideLeaf = new THREE.Mesh(sideLeafGeometry, leafMaterial);
                        sideLeaf.position.y = layerHeight + 5;
                        sideLeaf.position.x = (s === 0 ? -1 : 1) * 15;
                        sideLeaf.rotation.x = Math.PI / 2;
                        sideLeaf.rotation.z = (s === 0 ? 1 : -1) * (Math.PI / 3 + Math.random() * 0.2);
                        
                        // 存储原始值
                        sideLeaf.userData = {
                            originalX: sideLeaf.position.x,
                            originalY: sideLeaf.position.y,
                            originalRotationZ: sideLeaf.rotation.z
                        };
                        
                        kelpGroup.add(sideLeaf);
                    }
                }
                
                // 添加顶部装饰
                const topGeometry = new THREE.ConeGeometry(4, 12, 8);
                const topMaterial = new THREE.MeshPhongMaterial({
                    color: 0xaa77ff,
                    emissive: 0x7744dd,
                    emissiveIntensity: 0.3,
                    shininess: 50
                });
                const top = new THREE.Mesh(topGeometry, topMaterial);
                top.position.y = 100;
                kelpGroup.add(top);
                
                // 随机位置
                kelpGroup.position.set(
                    Math.floor(Math.random() * WORLD_SIZE) - WORLD_SIZE/2,
                    Math.floor(Math.random() * 100) - 50,
                    -(Math.floor(Math.random() * WORLD_SIZE) - WORLD_SIZE/2)
                );
                
                // 随机缩放
                const scale = 0.7 + Math.random() * 0.6;
                kelpGroup.scale.set(scale, scale, scale);
                
                kelpGroup.castShadow = true;
                kelpGroup.receiveShadow = true;
                
                // 摆动参数
                kelpGroup.userData = {
                    baseRotation: Math.random() * Math.PI * 2,
                    swingSpeed: Math.random() * 0.01 + 0.02,
                    swingAmplitude: Math.random() * 0.3 + 0.2,
                    leafWaveOffset: Math.random() * Math.PI * 2,
                    timeOffset: Math.random() * 1000
                };
                
                scene.add(kelpGroup);
                kelps.push(kelpGroup);
                
                // 添加气泡效果
                createBubblesAroundKelp(kelpGroup);
            }
        }
        
        // 在海带周围创建气泡
        function createBubblesAroundKelp(kelp) {
            const bubbleGroup = new THREE.Group();
            const bubbleMaterial = new THREE.MeshPhongMaterial({
                color: 0x88ccff,
                transparent: true,
                opacity: 0.6,
                shininess: 100,
                emissive: 0x4488ff,
                emissiveIntensity: 0.3
            });
            
            // 创建多个气泡
            for (let i = 0; i < 8; i++) {
                const bubbleGeometry = new THREE.SphereGeometry(1.5 + Math.random() * 1, 8, 8);
                const bubble = new THREE.Mesh(bubbleGeometry, bubbleMaterial);
                
                // 随机位置
                const angle = Math.random() * Math.PI * 2;
                const radius = 15 + Math.random() * 10;
                const height = Math.random() * 80;
                
                bubble.position.set(
                    Math.cos(angle) * radius,
                    height,
                    Math.sin(angle) * radius
                );
                
                bubble.castShadow = true;
                bubble.receiveShadow = true;
                bubble.userData = {
                    speed: 0.5 + Math.random() * 0.1,
                    startHeight: height,
                    amplitude: 0.5 + Math.random(),
                    offset: Math.random() * Math.PI * 2
                };
                
                bubbleGroup.add(bubble);
            }
            
            kelp.add(bubbleGroup);
        }
        
        // 创建光柱指示线
        function createPathCylinder() {
            const cylinderGeometry = new THREE.CylinderGeometry(8, 8, 1, 16);
            
            // 实心材质
            pathCylinderSolidMaterial = new THREE.MeshPhongMaterial({
                color: 0x00ffaa,
                transparent: true,
                opacity: 0.4,
                emissive: 0x00ffaa,
                emissiveIntensity: 0.3,
                side: THREE.DoubleSide
            });
            
            // 线框材质（用于第一人称模式）
            pathCylinderWireMaterial = new THREE.MeshBasicMaterial({
                color: 0x00ffaa,
                wireframe: true,
                transparent: true,
                opacity: 0.6,
                side: THREE.DoubleSide
            });
            
            pathCylinder = new THREE.Mesh(cylinderGeometry, pathCylinderSolidMaterial);
            pathCylinder.rotation.x = Math.PI / 2;
            scene.add(pathCylinder);
        }
        
        // 更新光柱指示线
        function updatePathCylinder() {
            if (!snake.length) return;
            
            const head = snake[0];
            const headPos = head.position;
            
            // 计算光柱方向
            const directionVector = direction.clone().normalize();
            
            // 计算光柱长度
            const distanceToBoundary = Math.min(
                (WORLD_SIZE/2 - Math.abs(headPos.x)) / Math.abs(directionVector.x),
                (WORLD_SIZE/2 - Math.abs(headPos.y)) / Math.abs(directionVector.y),
                (WORLD_SIZE/2 - Math.abs(headPos.z)) / Math.abs(directionVector.z)
            );
            
            const cylinderLength = Math.min(distanceToBoundary, WORLD_SIZE);
            
            // 更新光柱位置和尺寸
            pathCylinder.scale.set(1, cylinderLength, 1);
            pathCylinder.position.copy(headPos);
            pathCylinder.position.add(directionVector.clone().multiplyScalar(cylinderLength/2));
            
            // 旋转光柱以匹配方向
            pathCylinder.lookAt(headPos.clone().add(directionVector.clone().multiplyScalar(100)));
            pathCylinder.rotateX(Math.PI / 2);
        }
        
        // 设置摇杆功能
        function setupJoystick() {
            const joystickHead = document.getElementById('joystickHead');
            const joystickContainer = document.getElementById('joystickContainer');
            const directionIndicator = document.getElementById('directionIndicator');
            
            let startX, startY;
            let baseRect;
            
            // 触摸开始
            joystickHead.addEventListener('touchstart', (e) => {
                e.preventDefault();
                const touch = e.touches[0];
                const rect = joystickHead.getBoundingClientRect();
                startX = touch.clientX - rect.left - rect.width/2;
                startY = touch.clientY - rect.top - rect.height/2;
                baseRect = joystickContainer.getBoundingClientRect();
                joystickActive = true;
                joystickContainer.classList.add('active');
            });
            
            // 触摸移动
            document.addEventListener('touchmove', (e) => {
                if (!joystickActive) return;
                e.preventDefault();
                
                const touch = e.touches[0];
                const centerX = baseRect.left + baseRect.width/2;
                const centerY = baseRect.top + baseRect.height/2;
                
                const deltaX = touch.clientX - centerX;
                const deltaY = touch.clientY - centerY;
                
                // 计算距离和角度
                const distance = Math.min(Math.sqrt(deltaX * deltaX + deltaY * deltaY), baseRect.width/2);
                joystickAngle = Math.atan2(deltaY, deltaX);
              
                // 在功率未锁定时更新功率
                if (!powerLocked) {
                    joystickPower = distance / (baseRect.width/2);
                }
                
                // 更新摇杆头位置
                const offsetX = distance * Math.cos(joystickAngle);
                const offsetY = distance * Math.sin(joystickAngle);
                
                joystickHead.style.transform = `translate(${offsetX}px, ${offsetY}px)`;
                
                // 更新方向指示
                const angleDeg = Math.round((joystickAngle * 180 / Math.PI + 360) % 360);
                directionIndicator.textContent = `↑ ${angleDeg}°`;
                
                // 更新游戏方向
                horizontalAngle = joystickAngle + Math.PI/2;
                verticalAngle = Math.max(-Math.PI/3, Math.min(Math.PI/3, (joystickPower - 0.5) * Math.PI/3));
                updateDirectionVector();
                
                // 更新功率控制
                if (!powerLocked) {
                    updatePowerControl();
                }
                
                // 更新摇杆显示
                updateJoystickDisplay();
            });
            
            // 触摸结束
            document.addEventListener('touchend', () => {
                if (!joystickActive) return;
                joystickActive = false;
                joystickContainer.classList.remove('active');
                // 保持摇杆在当前位置，不重置
                directionIndicator.textContent = '';
            });
            
            // 鼠标事件
            joystickHead.addEventListener('mousedown', (e) => {
                e.preventDefault();
                const rect = joystickHead.getBoundingClientRect();
                startX = e.clientX - rect.left - rect.width/2;
                startY = e.clientY - rect.top - rect.height/2;
                baseRect = joystickContainer.getBoundingClientRect();
                joystickActive = true;
                joystickContainer.classList.add('active');
            });
            
            document.addEventListener('mousemove', (e) => {
                if (!joystickActive) return;
                e.preventDefault();
                
                const centerX = baseRect.left + baseRect.width/2;
                const centerY = baseRect.top + baseRect.height/2;
                
                const deltaX = e.clientX - centerX;
                const deltaY = e.clientY - centerY;
                
                // 计算距离和角度
                const distance = Math.min(Math.sqrt(deltaX * deltaX + deltaY * deltaY), baseRect.width/2);
                joystickAngle = Math.atan2(deltaY, deltaX);
                
                // 在功率未锁定时更新功率
                if (!powerLocked) {
                    joystickPower = distance / (baseRect.width/2);
                }
                
                // 更新摇杆头位置
                const offsetX = distance * Math.cos(joystickAngle);
                const offsetY = distance * Math.sin(joystickAngle);
                
                joystickHead.style.transform = `translate(${offsetX}px, ${offsetY}px)`;
                
                // 更新方向指示
                const angleDeg = Math.round((joystickAngle * 180 / Math.PI + 360) % 360);
                directionIndicator.textContent = `↑ ${angleDeg}°`;
                
                // 更新游戏方向
                horizontalAngle = joystickAngle + Math.PI/2;
                verticalAngle = Math.max(-Math.PI/2, Math.min(Math.PI/2, (joystickPower - 0.5) * Math.PI/2));
                updateDirectionVector();
                
                // 更新功率控制
                if (!powerLocked) {
                    updatePowerControl();
                }
                
                // 更新摇杆显示
                updateJoystickDisplay();
            });
            
            document.addEventListener('mouseup', () => {
                if (!joystickActive) return;
                joystickActive = false;
                joystickContainer.classList.remove('active');
                directionIndicator.textContent = '';
            });
        }
        
        // 更新功率控制
        function updatePowerControl() {
            const powerSlider = document.getElementById('powerSliderVertical');
            const powerIndicator = document.getElementById('powerIndicatorVertical');
            
            if (powerSlider && powerIndicator) {
                // 更新滑块位置
                powerSlider.value = Math.round(joystickPower * 100);
                
                // 更新功率指示器
                powerIndicator.textContent = joystickPower.toFixed(2);
                
                // 更新档把颜色
                updateGearColor();
            }
        }
        
        // 初始化小地图
        function initMiniMap() {
            const miniMap = document.getElementById('miniMap');
            // 使用容器尺寸作为参考
            const container = document.querySelector('.mini-map-container');
            const size = Math.min(container.clientWidth, container.clientHeight);
            
            // 设置像素比
            const pixelRatio = Math.min(window.devicePixelRatio, 2);
            miniMap.width = size * pixelRatio;
            miniMap.height = size * pixelRatio;
            miniMap.style.width = size + 'px';
            miniMap.style.height = size + 'px';
            
            miniMapCtx = miniMap.getContext('2d');
        }
        
        // 更新小地图
        function updateMiniMap() {
            if (!miniMapCtx || snake.length === 0) return;
            
            const head = snake[0];
            const headY = head.position.y;
            
            // 清空画布
            miniMapCtx.clearRect(0, 0, miniMapCtx.canvas.width, miniMapCtx.canvas.height);
            
            // 动态计算缩放比例
            const scale = miniMapCtx.canvas.width / WORLD_SIZE;
            
            // 绘制网格
            miniMapCtx.strokeStyle = 'rgba(100, 180, 255, 0.3)';
            miniMapCtx.lineWidth = 1;
            
            const gridSize = 10;
            const mapSize = miniMapCtx.canvas.width;
            for (let i = 0; i <= gridSize; i++) {
                const pos = i * (mapSize / gridSize);
                miniMapCtx.beginPath();
                miniMapCtx.moveTo(pos, 0);
                miniMapCtx.lineTo(pos, mapSize);
                miniMapCtx.stroke();
                
                miniMapCtx.beginPath();
                miniMapCtx.moveTo(0, pos);
                miniMapCtx.lineTo(mapSize, pos);
                miniMapCtx.stroke();
            }
            
            // 绘制边界
            miniMapCtx.strokeStyle = 'rgba(0, 200, 255, 0.5)';
            miniMapCtx.lineWidth = 2;
            miniMapCtx.strokeRect(0, 0, mapSize, mapSize);
            
            // 绘制食物
            for (let food of foods) {
                if (Math.abs(food.position.y - headY) > Y_THRESHOLD) continue;
                
                const x = (food.position.x + WORLD_SIZE/2) * scale;
                const z = mapSize - ((-food.position.z) + WORLD_SIZE/2) * scale;
                
                if (x >= 0 && x <= mapSize && z >= 0 && z <= mapSize) {
                    miniMapCtx.fillStyle = '#ff5555';
                    miniMapCtx.beginPath();
                    miniMapCtx.arc(x, z, 2, 0, Math.PI * 2);
                    miniMapCtx.fill();
                }
            }
            
            // 绘制障碍物
            for (let obstacle of obstacles) {
                if (Math.abs(obstacle.position.y - headY) > Y_THRESHOLD) continue;
                
                const x = (obstacle.position.x + WORLD_SIZE/2) * scale;
                const z = mapSize - ((-obstacle.position.z) + WORLD_SIZE/2) * scale;
                
                if (x >= 0 && x <= mapSize && z >= 0 && z <= mapSize) {
                    miniMapCtx.fillStyle = '#000000';
                    miniMapCtx.beginPath();
                    miniMapCtx.arc(x, z, 3, 0, Math.PI * 2);
                    miniMapCtx.fill();
                }
            }
            
            // 绘制AI蛇
            for (let aiSnake of aiSnakes) {
                if (aiSnake.body.length === 0) continue;
                
                const aiHead = aiSnake.body[0];
                if (Math.abs(aiHead.position.y - headY) > Y_THRESHOLD) continue;
                
                const aiX = (aiHead.position.x + WORLD_SIZE/2) * scale;
                const aiZ = mapSize - ((-aiHead.position.z) + WORLD_SIZE/2) * scale;
                
                if (aiX >= 0 && aiX <= mapSize && aiZ >= 0 && aiZ <= mapSize) {
                    miniMapCtx.fillStyle = '#ffaa00';
                    miniMapCtx.beginPath();
                    miniMapCtx.arc(aiX, aiZ, 3, 0, Math.PI * 2);
                    miniMapCtx.fill();
                }
            }
            
            // 绘制小球藻
            for (let alga of algae) {
                if (Math.abs(alga.position.y - headY) > Y_THRESHOLD) continue;
                
                const x = (alga.position.x + WORLD_SIZE/2) * scale;
                const z = mapSize - ((-alga.position.z) + WORLD_SIZE/2) * scale;
                
                if (x >= 0 && x <= mapSize && z >= 0 && z <= mapSize) {
                    miniMapCtx.fillStyle = '#55ff55';
                    miniMapCtx.beginPath();
                    miniMapCtx.arc(x, z, 3, 0, Math.PI * 2);
                    miniMapCtx.fill();
                }
            }
            
            // 绘制海带
            for (let kelp of kelps) {
                if (Math.abs(kelp.position.y - headY) > Y_THRESHOLD) continue;
                
                const x = (kelp.position.x + WORLD_SIZE/2) * scale;
                const z = mapSize - ((-kelp.position.z) + WORLD_SIZE/2) * scale;
                
                if (x >= 0 && x <= mapSize && z >= 0 && z <= mapSize) {
                    miniMapCtx.fillStyle = '#aa88ff';
                    miniMapCtx.beginPath();
                    miniMapCtx.arc(x, z, 4, 0, Math.PI * 2);
                    miniMapCtx.fill();
                }
            }
            
            // 绘制变形虫
            for (let amoeba of amoebas) {
                if (Math.abs(amoeba.position.y - headY) > Y_THRESHOLD) continue;
                
                const x = (amoeba.position.x + WORLD_SIZE/2) * scale;
                const z = mapSize - ((-amoeba.position.z) + WORLD_SIZE/2) * scale;
                
                if (x >= 0 && x <= mapSize && z >= 0 && z <= mapSize) {
                    miniMapCtx.fillStyle = '#ff88cc';
                    miniMapCtx.beginPath();
                    miniMapCtx.arc(x, z, 5, 0, Math.PI * 2);
                    miniMapCtx.fill();
                }
            }
            
            // 玩家蛇头位置转换
            const headX = (head.position.x + WORLD_SIZE/2) * scale;
            const headZ = mapSize - ((-head.position.z) + WORLD_SIZE/2) * scale;
            
            // 确保玩家蛇头在可见范围内
            if (headX >= 0 && headX <= mapSize && headZ >= 0 && headZ <= mapSize) {
                // 绘制玩家蛇头
                miniMapCtx.fillStyle = '#00ffaa';
                miniMapCtx.beginPath();
                miniMapCtx.arc(headX, headZ, 5, 0, Math.PI * 2);
                miniMapCtx.fill();
                
                // 绘制玩家蛇身
                for (let i = 1; i < snake.length; i++) {
                    const segment = snake[i];
                    const segX = (segment.position.x + WORLD_SIZE/2) * scale;
                    const segZ = mapSize - ((-segment.position.z) + WORLD_SIZE/2) * scale;
                    
                    if (segX >= 0 && segX <= mapSize && segZ >= 0 && segZ <= mapSize) {
                        miniMapCtx.fillStyle = '#00aaff';
                        miniMapCtx.beginPath();
                        miniMapCtx.arc(segX, segZ, 3, 0, Math.PI * 2);
                        miniMapCtx.fill();
                    }
                }
                
                // 方向指示
                const dirX = headX + direction.x * 20;
                const dirZ = headZ + direction.z * 20;
                
                miniMapCtx.strokeStyle = '#00ffaa';
                miniMapCtx.lineWidth = 2;
                miniMapCtx.beginPath();
                miniMapCtx.moveTo(headX, headZ);
                miniMapCtx.lineTo(dirX, dirZ);
                miniMapCtx.stroke();
            }
            
            // 更新高度指示器
            document.querySelector('.height-indicator').textContent = `y: ${Math.round(head.position.y)}`;
        }
        
        // 初始化蛇身材质
        function initSnakeMaterials() {
            snakeBodyMaterials = [];
            
            // 创建蛇身材质的渐变
            const headColor = "#00ffaa";
            const bodyStartColor = "#00ffaa";
            const bodyEndColor = "#00aaff";
            
            for (let i = 0; i < 10; i++) {
                const ratio = i / 9;
                const color = interpolateColor(bodyStartColor, bodyEndColor, ratio);
                
                snakeBodyMaterials.push(new THREE.MeshPhongMaterial({ 
                    color: new THREE.Color(color),
                    shininess: 80,
                    emissive: new THREE.Color(color).multiplyScalar(0.1),
                    emissiveIntensity: 0.5
                }));
            }
        }
        
        // 颜色插值函数
        function interpolateColor(color1, color2, ratio) {
            const r1 = parseInt(color1.substring(1, 3), 16);
            const g1 = parseInt(color1.substring(3, 5), 16);
            const b1 = parseInt(color1.substring(5, 7), 16);
            
            const r2 = parseInt(color2.substring(1, 3), 16);
            const g2 = parseInt(color2.substring(3, 5), 16);
            const b2 = parseInt(color2.substring(5, 7), 16);
            
            const r = Math.round(r1 + (r2 - r1) * ratio);
            const g = Math.round(g1 + (g2 - g1) * ratio);
            const b = Math.round(b1 + (b2 - b1) * ratio);
            
            return `#${r.toString(16).padStart(2, '0')}${g.toString(16).padStart(2, '0')}${b.toString(16).padStart(2, '0')}`;
        }
        
        // 初始化蛇
        function initSnake() {
            // 移除现有蛇身
            snake.forEach(segment => scene.remove(segment));
            snake = [];
            positionHistory = [];
            
            // 创建蛇头
            const headGeometry = new THREE.BoxGeometry(12, 12, 12);
            const headMaterial = new THREE.MeshPhongMaterial({ 
                color: new THREE.Color("#00ffaa"),
                shininess: 100,
                emissive: new THREE.Color("#00ffaa").multiplyScalar(0.3),
                emissiveIntensity: 0.3
            });
            const head = new THREE.Mesh(headGeometry, headMaterial);
            head.position.set(0, 0, 0);
            head.castShadow = true;
            head.receiveShadow = true;
            scene.add(head);
            snake.push(head);
            targetPosition.copy(head.position);
            
            // 创建初始蛇身
            for (let i = 1; i < snakeLength; i++) {
                addSnakeSegment();
            }
        }
        
        // 添加蛇身段
        function addSnakeSegment() {
            const segmentGeometry = new THREE.BoxGeometry(11, 11, 11);
            const materialIndex = (snake.length - 1) % snakeBodyMaterials.length;
            const segment = new THREE.Mesh(segmentGeometry, snakeBodyMaterials[materialIndex]);
            
            // 位置在蛇尾之后
            const lastSegment = snake[snake.length - 1];
            segment.position.copy(lastSegment.position);
            segment.position.x -= direction.x * 12;
            segment.position.y -= direction.y * 12;
            segment.position.z -= direction.z * 12;
            
            segment.castShadow = true;
            segment.receiveShadow = true;
            scene.add(segment);
            snake.push(segment);
        }
        
        // 创建食物
        function createFoods() {
            // 移除旧食物
            foods.forEach(food => scene.remove(food));
            foods = [];
            
            const foodGeometry = new THREE.SphereGeometry(8, 16, 16);
            const foodMaterial = new THREE.MeshPhongMaterial({ 
                color: 0xff5555,
                shininess: 100,
                emissive: 0xaa0000,
                emissiveIntensity: 0.5
            });
            
            for (let i = 0; i < FOOD_COUNT; i++) {
                const food = new THREE.Mesh(foodGeometry, foodMaterial);
                food.castShadow = true;
                food.receiveShadow = true;
                
                // 随机位置
                food.position.set(
                    Math.floor(Math.random() * WORLD_SIZE) - WORLD_SIZE/2,
                    Math.floor(Math.random() * WORLD_SIZE) - WORLD_SIZE/2,
                    -(Math.floor(Math.random() * WORLD_SIZE) - WORLD_SIZE/2)
                );
                
                scene.add(food);
                foods.push(food);
            }
        }
        
        // 创建障碍物
        function createObstacles() {
            // 移除旧障碍物
            obstacles.forEach(obstacle => scene.remove(obstacle));
            obstacles = [];
            
            const obstacleMaterial = new THREE.MeshPhongMaterial({ 
                color: 0xaa55ff,
                shininess: 60,
                emissive: 0x5511aa,
                emissiveIntensity: 0.3
            });
            
            for (let i = 0; i < OBSTACLE_COUNT; i++) {
                const size = Math.random() * 30 + 20;
                const obstacleGeometry = new THREE.BoxGeometry(size, size, size);
                
                const obstacle = new THREE.Mesh(obstacleGeometry, obstacleMaterial);
                
                // 随机位置
                obstacle.position.set(
                    Math.floor(Math.random() * WORLD_SIZE) - WORLD_SIZE/2,
                    Math.floor(Math.random() * WORLD_SIZE) - WORLD_SIZE/2,
                    -(Math.floor(Math.random() * WORLD_SIZE) - WORLD_SIZE/2)
                );
                
                obstacle.castShadow = true;
                obstacle.receiveShadow = true;
                scene.add(obstacle);
                obstacles.push(obstacle);
            }
        }
        
        // 创建边界指示
        function createBoundaryIndicators() {
            const boundaryMaterial = new THREE.MeshBasicMaterial({ 
                color: 0x203050,
                wireframe: true,
                opacity: 0.1,
                transparent: true
            });
            
            const boundaryGeometry = new THREE.BoxGeometry(WORLD_SIZE, WORLD_SIZE, WORLD_SIZE);
            const boundary = new THREE.Mesh(boundaryGeometry, boundaryMaterial);
            scene.add(boundary);
        }
        
        // 更新方向向量
        function updateDirectionVector() {
            // 平滑过渡垂直角度
            verticalAngle += (targetVerticalAngle - verticalAngle) * 0.2;
            
            // 根据水平角和垂直角计算方向向量
            direction.x = Math.cos(verticalAngle) * Math.sin(horizontalAngle);
            direction.y = Math.sin(verticalAngle);
            direction.z = Math.cos(verticalAngle) * Math.cos(horizontalAngle);
            
            // 归一化
            direction.normalize();
            
            // 翻转z轴
            direction.z = -direction.z;
            
            // 更新光柱指示线
            updatePathCylinder();
            
            // 更新光柱指示器
            updateBeamIndicator();
        }
        
        // 开始移动
        function startMove() {
            if (isMoving) return;
            
            isMoving = true;
            moveStartTime = performance.now();
            moveProgress = 0;
            
            // 设置目标位置
            targetPosition.copy(snake[0].position);
            targetPosition.add(direction.clone().multiplyScalar(MOVE_DISTANCE));
        }
        
        // 更新移动
        function updateMove(timestamp) {
            if (!isMoving) return;
            
            const elapsed = timestamp - moveStartTime;
            moveProgress = Math.min(elapsed / MOVE_DURATION, 1);
            
            // 计算蛇头当前位置
            const newHeadPosition = new THREE.Vector3().lerpVectors(
                snake[0].position, 
                targetPosition, 
                moveProgress
            );
            
            // 保存蛇头原位置
            const prevHeadPosition = snake[0].position.clone();
            
            // 更新蛇头位置
            snake[0].position.copy(newHeadPosition);
            
            // 记录位置历史
            positionHistory.unshift(snake[0].position.clone());
            if (positionHistory.length > HISTORY_MAX_LENGTH) {
                positionHistory.pop();
            }
            
            // 更新蛇身位置
            for (let i = 1; i < snake.length; i++) {
                const targetIndex = Math.min(positionHistory.length - 1, i * SEGMENT_DISTANCE);
                if (targetIndex < positionHistory.length) {
                    const targetPosition = positionHistory[targetIndex];
                    snake[i].position.lerp(targetPosition, 0.3);
                    
                }
            }
            
            // 在移动过程中检查食物碰撞
            checkFoodCollision();
            
            // 检查与AI蛇的碰撞
            checkAISnakeCollision();
            
            // 检查与小球藻的碰撞
            checkAlgaeCollision();
            
            // 检查与海带的碰撞
            checkKelpCollision();
            
            // 检查与变形虫的碰撞
            checkAmoebaCollision();
            
            // 检查移动是否完成
            if (moveProgress >= 1) {
                isMoving = false;
                
                // 检查碰撞
                if (checkCollision()) {
                    gameOver();
                } else {
                    // 立即开始下一次移动
                    startMove();
                }
            }
        }
        
        // 检查食物碰撞
        function checkFoodCollision() {
            const headPos = snake[0].position;
            
            for (let j = foods.length - 1; j >= 0; j--) {
                const food = foods[j];
                const distance = headPos.distanceTo(food.position);
                
                // 使用更精确的碰撞检测
                if (distance < 50) {
                    score += 10;
                    
                    // 每吃一个食物增加2段身体
                    addSnakeSegment();
                    addSnakeSegment();
                    
                    // 将被吃的食物移动到新位置
                    relocateFood(food);
                    
                    // 更新UI
                    updateUI();
                    
                    // 跳出循环，避免同一帧检测多个食物
                    break;
                }
            }
        }
        
        // 检查与小球藻的碰撞
        function checkAlgaeCollision() {
            const headPos = snake[0].position;
            
            for (let i = algae.length - 1; i >= 0; i--) {
                const alga = algae[i];
                const distance = headPos.distanceTo(alga.position);
                
                if (distance < 30) {
                    // 增加分数
                    score += 50
                    
                    // 添加蛇身段
                    addSnakeSegment();
                    
                    // 将被吃的小球藻变成食物
                    convertToFood(alga);
                    
                    // 从场景中移除小球藻
                    scene.remove(alga);
                    algae.splice(i, 1);
                    
                    // 更新UI
                    updateUI();
                    
                    break;
                }
            }
        }
        
        // 检查与海带的碰撞
        function checkKelpCollision() {
            const headPos = snake[0].position;
            
            for (let i = kelps.length - 1; i >= 0; i--) {
                const kelp = kelps[i];
                const distance = headPos.distanceTo(kelp.position);
                
                if (distance < 50) {
                    // 增加分数
                    score += 30
                    
                    // 添加蛇身段
                    addSnakeSegment();
                    addSnakeSegment();
                    
                    // 将被吃的海带变成食物
                    convertToFood(kelp);
                    
                    // 从场景中移除海带
                    scene.remove(kelp);
                    kelps.splice(i, 1);
                    
                    // 更新UI
                    updateUI();
                    
                    break;
                }
            }
        }
        
        // 将物体转换为食物
        function convertToFood(obj) {
            // 创建食物
            const foodGeometry = new THREE.SphereGeometry(8, 16, 16);
            const foodMaterial = new THREE.MeshPhongMaterial({ 
                color: 0xff5555,
                emissive: 0xaa0000,
                emissiveIntensity: 0.5
            });
            const food = new THREE.Mesh(foodGeometry, foodMaterial);
            
            food.position.copy(obj.position);
            food.castShadow = true;
            food.receiveShadow = true;
            
            scene.add(food);
            foods.push(food);
        }
        
        // 检查与AI蛇的碰撞
        function checkAISnakeCollision() {
            const headPos = snake[0].position;
            
            for (let i = aiSnakes.length - 1; i >= 0; i--) {
                const aiSnake = aiSnakes[i];
                if (aiSnake.body.length === 0) continue;
                
                const aiHead = aiSnake.body[0];
                const distance = headPos.distanceTo(aiHead.position);
                
                if (distance < 20) {
                    // 玩家蛇撞到AI蛇头，AI蛇变成食物
                    convertAISnakeToFood(aiSnake);
                    
                    // 从场景中移除AI蛇
                    for (let segment of aiSnake.body) {
                        scene.remove(segment);
                    }
                    
                    // 从AI蛇数组中移除
                    aiSnakes.splice(i, 1);
                    
                    // 增加分数
                    score += 500;
                    updateUI();
                    
                    break;
                }
            }
        }
        
        // 将AI蛇转换为食物
        function convertAISnakeToFood(aiSnake) {
            for (let i = 0; i < aiSnake.body.length; i++) {
                const segment = aiSnake.body[i];
                
                // 创建食物
                const foodGeometry = new THREE.SphereGeometry(8, 16, 16);
                const foodMaterial = new THREE.MeshPhongMaterial({ 
                    color: 0xff5555,
                    emissive: 0xaa0000,
                    emissiveIntensity: 0.5
                });
                const food = new THREE.Mesh(foodGeometry, foodMaterial);
                
                food.position.copy(segment.position);
                food.castShadow = true;
                food.receiveShadow = true;
                
                scene.add(food);
                foods.push(food);
            }
        }
        
        // 重新定位食物到新位置
        function relocateFood(food) {
            food.position.set(
                Math.floor(Math.random() * WORLD_SIZE) - WORLD_SIZE/2,
                Math.floor(Math.random() * WORLD_SIZE) - WORLD_SIZE/2,
                -(Math.floor(Math.random() * WORLD_SIZE) - WORLD_SIZE/2)
            );
        }
        
        // 检查碰撞
        function checkCollision() {
            const head = snake[0].position;
            
            // 边界碰撞
            if (Math.abs(head.x) > WORLD_SIZE/2 - 10 || 
                Math.abs(head.y) > WORLD_SIZE/2 - 10 || 
                Math.abs(head.z) > WORLD_SIZE/2 - 10) {
                return true;
            }
            
            // 障碍物碰撞
            for (let obstacle of obstacles) {
                // 使用更大的检测阈值
                if (head.distanceTo(obstacle.position) < 1) {
                    return true;
                }
            }
            
            return false;
        }
        
        // 游戏结束
        function gameOver() {
            gameRunning = false;
            document.getElementById('finalScore').textContent = score;
            document.getElementById('gameOver').style.display = 'flex';
        }
        
        // 重置游戏
        function resetGame() {
            // 重置游戏状态
            score = 0;
            snakeLength = 3;
            gameRunning = true;
            isMoving = true;
            horizontalAngle = Math.PI / 3;
            verticalAngle = 0;
            targetVerticalAngle = 0;
            joystickPower = 0.5;
            positionHistory = [];
            powerLocked = false;
            firstPersonMode = false;
            
            // 重置加速度
            for (let key in keyAcceleration) {
                keyAcceleration[key].acceleration = 0;
                keyAcceleration[key].pressed = false;
            }
            
            // 重置UI状态
            document.getElementById('powerLockBtn').classList.remove('locked');
            document.getElementById('powerLockBtn').textContent = '🔓';
            document.getElementById('viewToggleBtn').textContent = "👁️V";
            document.getElementById('viewToggleBtn').style.backgroundColor = "rgba(10, 25, 50, 0.8)";
            document.getElementById('fpIndicator').style.display = 'none';
            document.getElementById('beamDirection').style.display = 'none';
            document.getElementById('accelerationIndicator').style.display = 'none';
            document.getElementById('amoebaIndicator').style.display = 'none';
            document.getElementById('crosshair').style.display = 'none';
            document.getElementById('touchControls').style.display = 'none';
            
            // 重置相机
            camera.position.set(500, 400, 500);
            camera.lookAt(0, 0, 0);
            controls.reset();
            controls.enabled = true;
            
            // 更新方向向量
            updateDirectionVector();
            
            // 重新初始化蛇和食物
            initSnake();
            createFoods();
            createObstacles();
            
            // 移除所有AI蛇
            for (let aiSnake of aiSnakes) {
                for (let segment of aiSnake.body) {
                    scene.remove(segment);
                }
            }
            aiSnakes = [];
            
            // 移除所有小球藻
            for (let alga of algae) {
                scene.remove(alga);
            }
            algae = [];
            
            // 移除所有海带
            for (let kelp of kelps) {
                scene.remove(kelp);
            }
            kelps = [];
            
            // 移除所有变形虫
            for (let amoeba of amoebas) {
                scene.remove(amoeba);
            }
            amoebas = [];
            
            // 重新创建AI蛇
            createAISnakes();
            
            // 重新创建小球藻和海带
            createAlgae();
            createKelps();
            
            // 重新创建变形虫
            createAmoebas();
            
            // 更新UI
            updateUI();
            
            // 更新功率控制
            updatePowerControl();
            
            // 更新摇杆显示
            updateJoystickDisplay();
            
            // 隐藏游戏结束画面
            document.getElementById('gameOver').style.display = 'none';
            
            // 重置游戏时间
            gameStartTime = Date.now();
            
            // 开始第一次移动
            startMove();
        }
        
        // 更新UI
        function updateUI() {
            document.getElementById('scoreDisplay').textContent = `🧬: ${score}`;
        }
        
        // 键盘事件处理
        function onKeyDown(event) {
            const key = event.key.toLowerCase();
            keys[key] = true;
            
            // 更新加速度状态
            if (['a', 'd', 'w', 's'].includes(key) && !keyAcceleration[key].pressed) {
                keyAcceleration[key].pressed = true;
                keyAcceleration[key].startTime = performance.now();
            }
            
            switch (event.key) {
                case ' ':
                    gameRunning = !gameRunning;
                    document.getElementById('pauseOverlay').style.display = gameRunning ? 'none' : 'flex';
                    break;
                case 'r':
                case 'R':
                    resetGame();
                    break;
                case 'v':
                case 'V':
                    toggleFirstPerson();
                    break;
            }
        }
        
        function onKeyUp(event) {
            const key = event.key.toLowerCase();
            keys[key] = false;
            
            // 重置加速度状态
            if (['a', 'd', 'w', 's'].includes(key)) {
                keyAcceleration[key].pressed = false;
                keyAcceleration[key].acceleration = 0;
                document.getElementById('accelerationIndicator').style.display = 'none';
            }
        }
        
        // 处理键盘输入（带加速度）
        function handleKeyboardInput() {
            let anyKeyPressed = false;
            let maxAcceleration = 0;
            
            // 更新加速度
            for (let key in keyAcceleration) {
                if (keyAcceleration[key].pressed) {
                    // 增加加速度（但不超过最大值）
                    keyAcceleration[key].acceleration = Math.min(
                        MAX_ACCELERATION, 
                        keyAcceleration[key].acceleration + ACCELERATION_RATE
                    );
                    
                    anyKeyPressed = true;
                    maxAcceleration = Math.max(maxAcceleration, keyAcceleration[key].acceleration);
                } else if (keyAcceleration[key].acceleration > 0) {
                    // 衰减加速度
                    keyAcceleration[key].acceleration = Math.max(
                        0, 
                        keyAcceleration[key].acceleration - ACCELERATION_DECAY
                    );
                }
            }
            
            // 显示加速度指示器
            if (anyKeyPressed) {
                const indicator = document.getElementById('accelerationIndicator');
                indicator.style.display = 'block';
                indicator.textContent = `📈: ${maxAcceleration.toFixed(1)}x`;
            }
            
            // 计算实际速度：基础速度 * (1 + 加速度)
            const currentSpeedA = BASE_KEY_SPEED * (1 + keyAcceleration.a.acceleration);
            const currentSpeedD = BASE_KEY_SPEED * (1 + keyAcceleration.d.acceleration);
            const currentSpeedW = BASE_POWER_SPEED * (1 + keyAcceleration.w.acceleration);
            const currentSpeedS = BASE_POWER_SPEED * (1 + keyAcceleration.s.acceleration);

            if (keys['a']) {
                horizontalAngle -= currentSpeedA;
            }
            if (keys['d']) {
                horizontalAngle += currentSpeedD;
            }
            if (keys['w'] && !powerLocked) {
                joystickPower = Math.min(1, joystickPower + 0.0012*currentSpeedW);
            }
            if (keys['s'] && !powerLocked) {
                joystickPower = Math.max(0, joystickPower - 0.0012*currentSpeedS);
            }
            
            // 更新摇杆角度
            joystickAngle = horizontalAngle - Math.PI/2;
            
            // 更新垂直角度
            verticalAngle = Math.max(-Math.PI/2, Math.min(Math.PI/2, (joystickPower - 0.5) * Math.PI/2));
            
            // 更新方向向量
            updateDirectionVector();
            
            // 更新功率控制
            updatePowerControl();
            
            // 更新摇杆显示
            updateJoystickDisplay();
        }
        
        // 窗口大小调整
        function onWindowResize() {
            camera.aspect = window.innerWidth / window.innerHeight;
            camera.updateProjectionMatrix();
            
            // 设置设备像素比
            const pixelRatio = Math.min(window.devicePixelRatio, 2);
            renderer.setPixelRatio(pixelRatio);
            renderer.setSize(window.innerWidth, window.innerHeight);
            
            // 重新初始化小地图以适应新尺寸
            initMiniMap();
        }
        
        // 更新AI蛇
        function updateAISnakes() {
            for (let aiSnake of aiSnakes) {
                if (aiSnake.body.length === 0) continue;
                
                // 随机改变方向
                aiSnake.turnCounter++;
                if (aiSnake.turnCounter > aiSnake.turnInterval) {
                    aiSnake.direction = new THREE.Vector3(
                        Math.random() * 2 - 1,
                        Math.random() * 2 - 1,
                        Math.random() * 2 - 1
                    ).normalize();
                    aiSnake.turnCounter = 0;
                    aiSnake.turnInterval = Math.floor(Math.random() * 100) + 50;
                }
                
                // 移动AI蛇头
                const head = aiSnake.body[0];
                head.position.add(aiSnake.direction.clone().multiplyScalar(aiSnake.speed));
                
                // 边界检查
                if (Math.abs(head.position.x) > WORLD_SIZE/2 - 10) {
                    aiSnake.direction.x *= -1;
                    head.position.x = Math.sign(head.position.x) * (WORLD_SIZE/2 - 10);
                }
                if (Math.abs(head.position.y) > WORLD_SIZE/2 - 10) {
                    aiSnake.direction.y *= -1;
                    head.position.y = Math.sign(head.position.y) * (WORLD_SIZE/2 - 10);
                }
                if (Math.abs(head.position.z) > WORLD_SIZE/2 - 10) {
                    aiSnake.direction.z *= -1;
                    head.position.z = Math.sign(head.position.z) * (WORLD_SIZE/2 - 10);
                }
                
                // 记录位置历史
                aiSnake.positionHistory.unshift(head.position.clone());
                if (aiSnake.positionHistory.length > HISTORY_MAX_LENGTH) {
                    aiSnake.positionHistory.pop();
                }
                
                // 更新AI蛇身
                for (let i = 1; i < aiSnake.body.length; i++) {
                    const targetIndex = Math.min(aiSnake.positionHistory.length - 1, i * SEGMENT_DISTANCE);
                    if (targetIndex < aiSnake.positionHistory.length) {
                        const targetPosition = aiSnake.positionHistory[targetIndex];
                        aiSnake.body[i].position.lerp(targetPosition, 0.3);
                    }
                }
            }
        }
        
        // 更新小球藻
        function updateAlgae() {
            for (let i = 0; i < algae.length; i++) {
                const alga = algae[i];
                
                // 更新位置
                alga.position.add(alga.userData.direction);
                
                // 旋转
                alga.rotation.x += alga.userData.rotationSpeed.x;
                alga.rotation.y += alga.userData.rotationSpeed.y;
                alga.rotation.z += alga.userData.rotationSpeed.z;
                
                // 边界检查 - 反弹
                if (Math.abs(alga.position.x) > WORLD_SIZE/2 - 20) {
                    alga.userData.direction.x *= -1;
                    alga.position.x = Math.sign(alga.position.x) * (WORLD_SIZE/2 - 20);
                }
                if (Math.abs(alga.position.y) > WORLD_SIZE/2 - 20) {
                    alga.userData.direction.y *= -1;
                    alga.position.y = Math.sign(alga.position.y) * (WORLD_SIZE/2 - 20);
                }
                if (Math.abs(alga.position.z) > WORLD_SIZE/2 - 20) {
                    alga.userData.direction.z *= -1;
                    alga.position.z = Math.sign(alga.position.z) * (WORLD_SIZE/2 - 20);
                }
            }
        }
        
        // 更新海带
        function updateKelps() {
            const time = Date.now() * 0.001;
            
            for (let i = 0; i < kelps.length; i++) {
                const kelp = kelps[i];
                const t = time + kelp.userData.timeOffset * 0.001;
                
                // 整体摆动效果
                kelp.rotation.y = kelp.userData.baseRotation + 
                                 Math.sin(t * kelp.userData.swingSpeed) * kelp.userData.swingAmplitude;
                
                // 叶片波浪效果
                const leaves = [];
                kelp.traverse(child => {
                    if (child.isMesh && child.geometry.type === "PlaneGeometry") {
                        leaves.push(child);
                    }
                });
                
                for (let j = 0; j < leaves.length; j++) {
                    const leaf = leaves[j];
                    const waveOffset = kelp.userData.leafWaveOffset + j * 0.3;
                    
                    // 波浪效果 - 上下移动
                    const waveY = Math.sin(t * 1.5 + waveOffset) * 3;
                    
                    // 波浪效果 - 左右摆动
                    const waveX = Math.sin(t * 1.8 + waveOffset) * 2;
                    
                    // 应用波浪效果
                    leaf.position.y = leaf.userData.originalY + waveY;
                    leaf.position.x = leaf.userData.originalX + waveX;
                    
                    // 轻微旋转变化
                    leaf.rotation.z = leaf.userData.originalRotationZ + 
                                     Math.sin(t * 2 + waveOffset) * 0.1;
                }
                
                // 更新气泡效果
                const bubbleGroup = kelp.children.find(child => child.isGroup);
                if (bubbleGroup) {
                    bubbleGroup.children.forEach(bubble => {
                        // 气泡上下浮动
                        bubble.position.y = bubble.userData.startHeight + 
                                          Math.sin(t * bubble.userData.speed + bubble.userData.offset) * 
                                          bubble.userData.amplitude * 10;
                        
                        // 气泡旋转
                        bubble.rotation.x += 0.01;
                        bubble.rotation.y += 0.015;
                    });
                }
            }
        }
        
        // 吸引附近食物
        function attractNearbyFoods() {
            if (snake.length === 0) return;
            
            const headPos = snake[0].position;
            
            for (let i = 0; i < foods.length; i++) {
                const food = foods[i];
                const distance = headPos.distanceTo(food.position);
                
                // 如果食物在吸引范围内，则向蛇头移动
                if (distance < ATTRACTION_DISTANCE) {
                    // 计算从食物指向蛇头的方向向量
                    const direction = new THREE.Vector3().subVectors(headPos, food.position).normalize();
                    
                    // 根据距离调整移动速度
                    const speed = ATTRACTION_SPEED * (distance / ATTRACTION_DISTANCE);
                    
                    // 移动食物
                    food.position.add(direction.multiplyScalar(speed));
                }
            }
        }
        
        // 动画循环
        function animate(timestamp) {
            requestAnimationFrame(animate);
            
            frameCount++;
            if (timestamp - lastFpsUpdate >= 1000) {
                fps = frameCount;
                frameCount = 0;
                lastFpsUpdate = timestamp;
            }
            
            // 处理键盘输入
            handleKeyboardInput();
            
            // 更新游戏状态
            if (gameRunning) {
                // 更新方向向量
                updateDirectionVector();
                
                // 更新移动
                if (isMoving) {
                    updateMove(timestamp);
                }
                
                // 更新AI蛇
                if (frameCount % 2 === 0) {
                    updateAISnakes();
                }
                
                // 更新小球藻
                updateAlgae();
                
                // 更新海带
                updateKelps();
                
                // 更新变形虫
                updateAmoebas(timestamp);
                
                // 使食物旋转
                for (let food of foods) {
                    food.rotation.x += 0.008;
                    food.rotation.y += 0.012;
                }
                
                // 吸引附近食物
                attractNearbyFoods();
                
                // 蛇头呼吸效果
                if (snake.length > 0) {
                    const head = snake[0];
                    const scale = 1 + Math.sin(timestamp * 0.003) * 0.1;
                    head.scale.set(scale, scale, scale);
                }
                
                // 更新光柱
                updatePathCylinder();
                
                // 更新小地图
                updateMiniMap();
                
                // 更新第一人称视角
                if (firstPersonMode) {
                    updateFirstPersonCamera();
                }
            }
            
            // 更新控件
            if (controls.enabled) {
                controls.update();
            }
            
            // 渲染场景
            renderer.render(scene, camera);
        }
        
        // 设置摇杆容器拖动功能
        function setupJoystickDrag() {
            const joystickContainer = document.getElementById('joystickContainer');
            let isDragging = false;
            let startX, startY;
            let startLeft, startBottom;

            // 鼠标事件
            joystickContainer.addEventListener('mousedown', function(e) {
                // 仅当点击在容器上但不是摇杆头时才拖动
                if (e.target !== joystickContainer && !e.target.classList.contains('drag-handle')) {
                    return;
                }
                
                isDragging = true;
                startX = e.clientX;
                startY = e.clientY;
                
                // 获取当前位置
                startLeft = parseFloat(joystickContainer.style.left) || 60;
                startBottom = parseFloat(joystickContainer.style.bottom) || 60;
                
                e.preventDefault();
            });

            document.addEventListener('mousemove', function(e) {
                if (!isDragging) return;
                
                const deltaX = e.clientX - startX;
                const deltaY = startY - e.clientY;
                
                // 计算新位置
                let newLeft = startLeft + deltaX;
                let newBottom = startBottom - deltaY;
                
                // 边界检查
                newLeft = Math.max(10, Math.min(window.innerWidth - joystickContainer.offsetWidth - 10, newLeft));
                newBottom = Math.max(10, Math.min(window.innerHeight - joystickContainer.offsetHeight - 10, newBottom));
                
                // 应用新位置
                joystickContainer.style.left = newLeft + 'px';
                joystickContainer.style.bottom = newBottom + 'px';
            });

            document.addEventListener('mouseup', function() {
                isDragging = false;
            });

            // 触摸事件
            joystickContainer.addEventListener('touchstart', function(e) {
                // 仅当点击在容器上但不是摇杆头时才拖动
                if (e.target !== joystickContainer && !e.target.classList.contains('drag-handle')) {
                    return;
                }
                
                if (e.touches.length === 1) {
                    isDragging = true;
                    const touch = e.touches[0];
                    startX = touch.clientX;
                    startY = touch.clientY;
                    
                    startLeft = parseFloat(joystickContainer.style.left) || 60;
                    startBottom = parseFloat(joystickContainer.style.bottom) || 60;
                    
                    e.preventDefault();
                }
            }, { passive: false });

            document.addEventListener('touchmove', function(e) {
                if (!isDragging || e.touches.length !== 1) return;
                
                const touch = e.touches[0];
                const deltaX = touch.clientX - startX;
                const deltaY = touch.clientY - startY;
                
                let newLeft = startLeft + deltaX;
                let newBottom = startBottom - deltaY;
                
                // 边界检查
                newLeft = Math.max(10, Math.min(window.innerWidth - joystickContainer.offsetWidth - 10, newLeft));
                newBottom = Math.max(10, Math.min(window.innerHeight - joystickContainer.offsetHeight - 10, newBottom));
                
                joystickContainer.style.left = newLeft + 'px';
                joystickContainer.style.bottom = newBottom + 'px';
                
                e.preventDefault();
            }, { passive: false });

            document.addEventListener('touchend', function() {
                isDragging = false;
            });
        }
        
        // 初始化游戏
        window.onload = function() {
            init();
            updateDirectionVector();
            updateGearColor();
            updateJoystickDisplay();
        };
    </script>
</body>
</html>
```

完整源码：[GitHub](https://github.com/z2586300277/three-cesium-examples/blob/dev/threeExamples/application/snake3D.html)

## 小结

- 本文提供 **3D贪吃蛇** 完整 Three.js 源码与在线 Demo，建议先运行案例再改 uniform/参数做二次实验
- 更多 Three.js 实战案例见 [three-cesium-examples 合集](https://threehub.cn) 与 [GitHub 开源仓库](https://github.com/z2586300277/three-cesium-examples)

