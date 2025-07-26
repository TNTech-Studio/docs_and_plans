# Creeper-GO

An Autonomous Trophy of Compliance by TNTech

    “Aww man!” – The anti-hero we deserve, now roaming your venue.

把“DISQUALIFY”写在履带上——一只搭载 NVIDIA Orin 的苦力怕小车，会追逐没戴胸牌的人并当场“自爆”告示：**YOU ARE DISQUALIFIED**。

**详细代码开源地址**

- [摄像机图传前端](https://github.com/TNTech-Studio/car_web_client)

- [小车控制端](https://github.com/TNTech-Studio/creeper_go)

- [端侧模型推理以及目标跟踪](https://github.com/TNTech-Studio/car_yolo)

- [自动拍摄数据集](https://github.com/TNTech-Studio/auto-shot)

---

## 详细介绍  

Creeper-GO 是 TNTech 为本届黑客松量身打造的“移动规则警察”。我们借用了松灵机器人的履带底盘，在上面安放一块 **Jetson Orin 开发板**，训练了轻量级 **YOLOv8s** 模型，仅识别两种目标：  
1. `person`  
2. `badge`  

### 狩猎逻辑

1. **单目标锁定**  
   YOLO 为检测到的每个人分配唯一 ID。小车只跟随第一帧里出现的那个人，避免“多人混战”。  
2. **距离推断**  
   不用深度相机——我们直接看这位受害者在画面里“多大”：  
   ```
   sizeRatio = max(w_img / W_frame, h_img / H_frame)
   ```  
   当 `sizeRatio > 0.90` 说明你几乎占满屏幕 ≈ 小车顶到腿，立即进入爆炸流程。  
3. **DISQUALIFY 爆炸**  
   伺服电机掀开 3D 打印头盖，LED Matrix 滚动红字“YOU ARE DISQUALIFIED”，扬声器播放经典嘶嘶声 + 低频冲击波采样（别担心，只破你的防不破骨骼）。  

### 为什么酷  

* **艺术层面**：把大会“带胸牌”条例物化为可怕的实体，让规则以追猎的方式介入空间——一场赛博行为艺术。  
* **技术层面**：  
  - Jetson Orin 的 40 TOPS 让 YOLOv8s 以 90 FPS 跑在边缘端；  
  - 仅 2 类别目标，模型权重 6.8 MB，加载到 L2 缓存无需 DMA；  
  - 纯 2D 几何估距，无需 Lidar、VIO、SLAM，也能实现“贴脸爆炸”。  
* **交互层面**：被追的人惊慌逃窜、围观者狂拍视频、主办方满场高喊“请佩戴胸牌”——声光电社交裂变一条龙。  

---

## 技术栈  

| 模块 | 组件 / 库 | 说明 |
|------|-----------|------|
| 底盘 | 松灵机器人 Mini-Track | 0.7 m/s 橡胶履带，USB-CAN 控制 |
| 计算 | NVIDIA Jetson Orin Dev Kit | 8-core CPU + 2048 CUDA + 32 TOPS NPU |
| 推理 | Python 3.10 + PyTorch 2.2 + TensorRT 8.6 | YOLOv8s 人/胸牌二分类，浮点→INT8 量化 |
| 机电 | Arduino Pro Micro | 驱动双路电机、2×伺服、128×32 LED Matrix |
| 视觉 | OpenCV 4.9 | 目标框滤波、ID 关联 |
| 框架 | ROS 2 Humble | Topic：/detect、/track、/motor_cmd |
| 外壳 | PT材质喷漆 | 绿皮像素纹理 |

---

> Creeper-GO：让规则自己长腿，跑过来告诉你——**你被淘汰了。**


# 简单的文档

## 总体架构

小车端侧运行三个模块：

1. 端侧 Yolo 推理，上报每一帧检测到的目标，并传回图片、进行目标跟踪。使用 Quart 实现，通过 SSE 把事件上报给多端

2. 车机控制端，接收目标跟踪结果，控制小车运动。使用 ROS 2 实现，通过 Topic 发布运动指令

3. 外设模块，控制扬声器、灯带、舵机

在外面可以（可选）运行一个图传客户端，可以看到小车的摄像头图像和检测结果。

## 车机状态机

![creeper_states](./img/creeper_states.svg)
