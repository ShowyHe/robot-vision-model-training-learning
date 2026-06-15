# OpenImages Nav-Find v4 训练结果记录

日期：2026-06-13  
对应规划文档：`docs/openimages_nav_find_v4_training_plan_2026-06-12.md`  
任务：验证 `oiv7_nav_find_subset_v4` 数据集与 40 epoch 训练方案是否可作为机器人导盲/寻物视觉模型 baseline。

---

## 1. 本文件目的

上一个文档记录了：

```text
为什么要做 OpenImages 子集
为什么废弃 v1/v2/v3
为什么选择 v4
如何训练 40 epoch
```

本文件记录实际训练结果，形成闭环：

```text
规划 → 执行 → 结果 → 判断 → 下一步
```

---

## 2. 训练配置

### 2.1 数据集

```text
/mnt/data/datasets/openimages/oiv7_nav_find_subset_v4/data.yaml
```

v4 数据集规模：

```text
train images: 67913
val images: 14187
classes: 61
```

### 2.2 模型初始权重

```text
/mnt/data/datasets/openimages/runs/oiv7_half_b8_w4_e20_new/weights/best.pt
```

这是已有 OpenImages 训练权重，不是 v1/v2/v3 的错误类别体系权重。

### 2.3 正式训练命令

```bash
cd /mnt/data/datasets/openimages
mkdir -p runs/logs

yolo detect train \
  model=/mnt/data/datasets/openimages/runs/oiv7_half_b8_w4_e20_new/weights/best.pt \
  data=oiv7_nav_find_subset_v4/data.yaml \
  imgsz=640 \
  batch=6 \
  workers=2 \
  epochs=40 \
  amp=True \
  cache=False \
  mosaic=0.3 \
  mixup=0 \
  copy_paste=0 \
  close_mosaic=10 \
  compile=False \
  plots=False \
  save=True \
  save_period=5 \
  project=runs/nav_find_v4 \
  name=e40_b6 \
  2>&1 | tee runs/logs/nav_find_v4_e40_b6.log
```

### 2.4 实际训练输出目录

注意：Ultralytics 自动套了一层 `runs/detect/runs/...`。

```text
/mnt/data/datasets/openimages/runs/detect/runs/nav_find_v4/e40_b6
```

权重路径：

```text
/mnt/data/datasets/openimages/runs/detect/runs/nav_find_v4/e40_b6/weights/best.pt
/mnt/data/datasets/openimages/runs/detect/runs/nav_find_v4/e40_b6/weights/last.pt
```

---

## 3. 训练环境

```text
Ultralytics: 8.4.61
Python: 3.10.12
PyTorch: 2.11.0+cu128
CUDA: 0
GPU: NVIDIA GeForce RTX 5070 Ti, 15833 MiB
Model: YOLOv8m
Parameters: 25,891,639
GFLOPs: 79.3
```

训练中：

```text
AMP checks passed
train cache created successfully
val cache created successfully
0 corrupt images
```

---

## 4. 训练稳定性

### 4.1 CUDA timeout 问题

本次 40 epoch 没有出现 CUDA watchdog timeout。

显存使用稳定：

```text
GPU_mem ≈ 4.06G
```

训练完成时间：

```text
40 epochs completed in 7.867 hours
```

判断：

```text
v4 数据规模和训练参数是稳定可跑的。
```

这说明从全量 OpenImages 改为 v4 子集是有效的。

---

## 5. 总体训练结果

最终验证结果：

```text
all:
P        = 0.501
R        = 0.521
mAP50    = 0.510
mAP50-95 = 0.430
```

结论：

```text
v4 已经可以作为第一版机器人寻物/导航视觉 baseline。
但它不是最终模型。
```

---

## 6. 训练趋势

训练过程指标：

```text
epoch 1:  mAP50=0.414, mAP50-95=0.338
epoch 10: mAP50=0.468, mAP50-95=0.387
epoch 20: mAP50=0.479, mAP50-95=0.404
epoch 30: mAP50=0.497, mAP50-95=0.421
epoch 40: mAP50=0.510, mAP50-95=0.429
```

loss 变化：

```text
box_loss: 1.067  → 0.7765
cls_loss: 2.349  → 0.7052
dfl_loss: 1.326  → 1.120
```

判断：

```text
训练趋势正常。
模型没有崩溃。
没有一开始就停滞。
最后 10 轮关闭 mosaic 后，指标继续小幅上涨。
```

---

## 7. 效果较好的类别

以下类别目前可初步认为效果较好，适合进入实图测试：

| Class | Images | Instances | mAP50 | mAP50-95 | 判断 |
|---|---:|---:|---:|---:|---|
| car | 5095 | 9924 | 0.736 | 0.583 | 好 |
| bus | 82 | 103 | 0.707 | 0.649 | 好 |
| bicycle | 252 | 418 | 0.680 | 0.520 | 好 |
| bicycle_wheel | 245 | 780 | 0.784 | 0.615 | 好 |
| stop_sign | 13 | 13 | 0.995 | 0.988 | 很好，但验证样本少 |
| fire_hydrant | 15 | 15 | 0.909 | 0.876 | 很好，但验证样本少 |
| wheelchair | 48 | 117 | 0.725 | 0.512 | 好 |
| bookcase | 78 | 118 | 0.664 | 0.525 | 可用 |
| coffee_cup | 155 | 189 | 0.855 | 0.811 | 很好 |
| mobile_phone | 105 | 147 | 0.811 | 0.770 | 很好 |
| laptop | 52 | 70 | 0.631 | 0.564 | 可用 |
| keyboard | 63 | 67 | 0.737 | 0.663 | 好 |
| mouse | 19 | 24 | 0.874 | 0.760 | 很好 |
| fork | 37 | 50 | 0.638 | 0.560 | 可用 |
| knife | 59 | 80 | 0.779 | 0.636 | 好 |
| light_switch | 10 | 12 | 0.763 | 0.730 | 看起来好，但验证样本少 |
| ceiling_fan | 11 | 11 | 0.938 | 0.859 | 看起来很好，但验证样本少 |

对机器人寻物功能比较关键的正向结果：

```text
mobile_phone
coffee_cup
laptop
keyboard
mouse
fork
knife
```

这些类别值得优先做实图测试。

---

## 8. 明显偏弱的类别

以下类别需要重点关注：

| Class | Images | Instances | P | R | mAP50 | 问题 |
|---|---:|---:|---:|---:|---:|---|
| person | 7144 | 16753 | 0.578 | 0.0487 | 0.210 | 严重漏检 |
| chair | 311 | 827 | 0.385 | 0.207 | 0.209 | 偏弱 |
| table | 593 | 949 | 0.569 | 0.0674 | 0.274 | 严重漏检 |
| dining_table | 32 | 76 | 0.359 | 0.158 | 0.161 | 弱 |
| cabinetry | 145 | 220 | 0.339 | 0.133 | 0.162 | 弱 |
| book | 95 | 720 | 0.228 | 0.0444 | 0.199 | 严重漏检 |
| suitcase | 23 | 33 | 0.191 | 0.394 | 0.218 | 弱 |
| toaster | 1 | 1 | 0 | 0 | 0 | 基本没学到，验证样本极少 |

重点问题：

```text
person recall 只有 0.0487。
```

这不是验证样本太少导致的，因为 person 验证集中有：

```text
Images: 7144
Instances: 16753
```

因此 person 是当前 v4 最大问题之一。

---

## 9. 小样本类别指标解释

部分类别验证样本极少：

```text
pressure_cooker: 1 instance
slow_cooker: 2 instances
dishwasher: 3 instances
toaster: 1 instance
light_switch: 12 instances
power_socket: 21 instances
ceiling_fan: 11 instances
```

因此这些类别的 mAP 不能作为稳定判断依据。

例如：

```text
ceiling_fan mAP50=0.938，看起来很好，但 val 只有 11 个实例。
toaster mAP=0，看起来很差，但 val 只有 1 个实例。
```

实际结论应通过真实图片测试确认。

---

## 10. v4 当前判断

### 10.1 成功点

```text
1. 40 epoch 完整跑完。
2. 没有 CUDA timeout。
3. 显存稳定在约 4.06G。
4. 训练耗时约 7.9 小时，可接受。
5. 61 个英文细类训练流程跑通。
6. 手机、电脑设备、杯子、部分餐具、部分交通类效果不错。
7. 可以作为第一版 baseline。
```

### 10.2 不足点

```text
1. person 严重漏检。
2. chair/table/book 等高频室内目标偏弱。
3. toaster/pressure_cooker/dishwasher 等小样本类别无法稳定判断。
4. 验证集不均衡，小类 per-class mAP 波动大。
5. 还没有经过真实场景图片/视频测试。
```

---

## 11. 下一步：实际图片测试

训练指标只能说明模型在验证集上的表现。

机器人寻物任务还必须测试真实场景图片或视频。

### 11.1 建测试目录

```bash
cd /mnt/data/datasets/openimages
mkdir -p test_images
```

把真实测试图片放到：

```text
/mnt/data/datasets/openimages/test_images
```

### 11.2 推理命令

```bash
yolo detect predict \
  model=/mnt/data/datasets/openimages/runs/detect/runs/nav_find_v4/e40_b6/weights/best.pt \
  source=/mnt/data/datasets/openimages/test_images \
  imgsz=640 \
  conf=0.25 \
  save=True \
  save_txt=True \
  project=/mnt/data/datasets/openimages/runs/nav_find_v4_predict \
  name=real_test \
  exist_ok=True
```

输出位置：

```text
/mnt/data/datasets/openimages/runs/nav_find_v4_predict/real_test
```

### 11.3 重点测试对象

优先测试：

```text
person
chair
table
mobile_phone
coffee_cup
bottle
laptop
keyboard
mouse
book
backpack
handbag
bowl
plate
spoon
fork
knife
light_switch
power_socket
ceiling_fan
fan
```

---

## 12. v5 改进方向

v5 不建议推倒重来，应在 v4 基础上针对性改。

### 12.1 数据采样方向

```text
person: 提高有效采样，解决 recall 过低
chair/table/book: 增加训练占比
家具类: 单独补一批室内场景数据
小家电类: 依赖实拍补数据
```

### 12.2 验证集方向

构建一个更均衡的人工验证集：

```text
每类尽量 30~100 张真实测试图
重点类更多
小样本类靠实拍补齐
```

### 12.3 实拍补数据优先级

P0：机器人导航安全相关

```text
person
stairs
door
door_handle
chair
table
```

P1：高频寻物

```text
mobile_phone
coffee_cup
bottle
book
laptop
keyboard
mouse
backpack
handbag
bowl
plate
spoon
fork
knife
```

P2：OpenImages 小样本弱类

```text
pressure_cooker
slow_cooker
light_switch
power_socket
dishwasher
toaster
ceiling_fan
fan
```

---

## 13. 当前结论

本次 v4 训练闭环结论：

```text
v4 方案成立。
它解决了全量 OpenImages 太大和 CUDA timeout 风险。
它跑通了 61 个英文细类的训练。
它可以作为第一版机器人寻物/导航视觉 baseline。
但它不是最终模型。
下一步必须做真实图片/视频测试，并针对 person、chair、table、book、小家电类继续补数据和重采样。
```

---

## 14. 与规划文档的关系

对应规划文档：

```text
docs/openimages_nav_find_v4_training_plan_2026-06-12.md
```

本文件是结果文档：

```text
docs/openimages_nav_find_v4_training_result_2026-06-13.md
```

两份文档形成闭环：

```text
训练规划：为什么这样做、怎么做
训练结果：实际跑完后效果如何、下一步怎么改
```
