# Robot Vision Model Training Learning

## 1. 仓库定位

本仓库用于记录机器人视觉模型训练、导出、部署相关学习过程。

这是 private 学习仓库，主要服务于：

- 视觉模型训练流程学习
- YOLO / ONNX / TensorRT 部署理解
- ROS2 视觉感知接入整理
- 后续整理成公开教程

## 2. 学习主题

当前重点包括：

- 数据采集
- 数据标注
- YOLOv8 / YOLOv8-seg 训练
- 模型评估
- ONNX 导出
- TensorRT 转换
- ROS2 推理节点接入
- 盲道检测
- 斑马线检测
- 红绿灯检测
- 机器人视觉模型部署问题

## 3. 目录结构

    docs/
      00_overview.md
      01_bringup_runbook.md
      02_code_map.md
      03_core_logic.md
      04_experiments.md
      05_bugs_and_fixes.md
      06_public_candidates.md

    logs/
      保存训练、转换、推理日志

    screenshots/
      保存训练结果、终端截图、推理截图

    scripts/
      保存训练、转换、测试脚本

    references/
      保存官方文档、公开教程、开源项目链接

    public_draft/
      保存后续准备公开的教程草稿

## 4. 学习目标

最终需要搞懂：

- 一个视觉模型从数据到部署的完整链路
- 训练数据如何组织
- 标注格式如何转换
- 模型如何训练和验证
- ONNX 和 TensorRT 分别解决什么问题
- 模型如何接入 ROS2 系统
- 部署过程中常见问题如何排查

## 5. 公开原则

本仓库内容默认不公开。

后续 public 版本只基于：

- 官方文档
- 开源项目
- 自己整理的训练流程
- 自己复现的 demo
- 脱敏后的工程经验

不公开：

- 公司模型文件
- 公司训练数据
- 公司文档原文
- 公司真实部署路径
- 公司真实参数
- 公司内部业务流程