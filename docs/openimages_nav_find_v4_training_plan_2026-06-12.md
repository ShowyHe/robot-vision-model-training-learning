# OpenImages Nav-Find v4 训练规划记录

日期：2026-06-12  
仓库：`robot-vision-model-training-learning`  
任务：基于 OpenImages 生成盲人导盲/寻物场景可用的 YOLO 检测数据子集，并进行 40 epoch 训练测试。

---

## 1. 本次目标

本次训练不是做通用 COCO/OpenImages 大模型，而是做一个面向机器人导盲与寻物功能的视觉检测模型。

核心目标：

1. 保留寻物和导航场景需要的细分类别。
2. 不再把功能不同的物体合并成泛类。
3. 使用 OpenImages 作为预训练/初筛数据来源。
4. 控制数据规模，避免全量 OpenImages 训练时间过长和 CUDA watchdog timeout。
5. 先跑一个 40 epoch 的可用版本，用于后续实机/图片测试。

---

## 2. 背景问题

之前直接训练全量 OpenImages 时，出现过 CUDA kernel timeout：

```text
CUDA error: the launch timed out and was terminated
NVRM: GPU is probably locked! Notify Timeout Seconds: 7
```

判断：

- 不是显存 OOM。
- GPU 负责桌面显示，长 kernel 被 watchdog 杀掉。
- 全量 OpenImages 规模过大，单轮耗时过长。
- 需要缩小训练子集，并降低训练负载。

因此本次改为：

```text
OpenImages 全量数据源
→ 抽取导航/寻物相关类别
→ 生成可控规模子集
→ 使用已有 OpenImages 训练权重继续 fine-tune
```

---

## 3. 类别设计原则

### 3.1 不合并功能不同的类别

错误示例：

```text
bowl / plate / spoon / fork / knife → 餐具
```

这种做法不适合寻物，因为模型只会输出 `餐具`，无法告诉机器人用户要找的是碗、盘子、勺子还是叉子。

因此 v2 之后采用英文细分类别，例如：

```text
bowl
plate
spoon
fork
knife
```

同理，以下类别也不合并：

```text
car / bus / truck / van / taxi
laptop / keyboard / mouse / monitor / tablet_computer
mobile_phone / corded_phone / telephone
pressure_cooker / slow_cooker
ceiling_fan / fan
```

### 3.2 全部使用英文类别名

原因：

1. 训练、部署、日志、ROS 解析更稳定。
2. 避免中文编码、字体、导出兼容问题。
3. 后续可在应用层把英文类别映射成中文语音播报。

---

## 4. 目标类别文件

目标类别文件：

```text
target_v2_classes.yaml
```

注意事项：

- OpenImages 中没有 `Fan`。
- 实际类别为：

```text
Ceiling fan
Mechanical fan
```

因此配置中应使用：

```yaml
ceiling_fan: Ceiling fan
fan: Mechanical fan
```

---

## 5. 各版本数据子集尝试

### 5.1 v1：废弃

v1 问题：

- 把多个细类合并成中文大类。
- 不适合寻物任务。
- 例如 `bowl/plate/spoon/fork/knife` 合成 `餐具` 后，模型无法输出具体物体。

结论：

```text
v1 废弃，不用于正式训练。
```

---

### 5.2 v2：类别正确，但规模偏大

v2 改为 61 个英文细类。

生成结果：

```text
valid classes: 61
missing: []
train images: 315818
val images: 5229
```

问题：

- 类别设计正确。
- 但训练图片数 31.5 万，超出预期。
- 原因是脚本按“只要图片里有一个目标类未满 quota，就保留整张图”。
- 一张图可能同时带入 person/car/chair/table 等大类，导致大类数量明显超出 quota。

结论：

```text
v2 不作为 40 epoch 正式训练集。
```

---

### 5.3 v3：脚本逻辑错误，废弃

v3 目标是压缩到 8~12 万张，但 oversample 逻辑写错。

错误逻辑：

```python
def oversample_target(n):
    ...
    return n
```

这导致大类也被 oversample：

```text
person raw_box=491295 → 目标也变成 491295
car raw_box=118574 → 目标也变成 118574
chair raw_box=65083 → 目标也变成 65083
```

实际结果：

```text
train images: 668049
```

结论：

```text
v3 废弃，不训练。
```

---

### 5.4 v4：当前可用版本

v4 重新设计采样逻辑：

```text
小类优先保留
小类才 oversample
大类不 oversample
中大类只补到 quota
总量受控
```

v4 生成结果：

```text
valid classes: 61
missing: []
train images: 67913
val images: 14187
```

当前判断：

```text
v4 可以用于 40 epoch 正式测试。
```

---

## 6. v4 数据集路径

数据源：

```text
/mnt/data/datasets/openimages/open-images-v7
```

v4 子集输出：

```text
/mnt/data/datasets/openimages/oiv7_nav_find_subset_v4
```

YOLO 配置：

```text
/mnt/data/datasets/openimages/oiv7_nav_find_subset_v4/data.yaml
```

训练集：

```text
oiv7_nav_find_subset_v4/images/train
oiv7_nav_find_subset_v4/labels/train
```

验证集：

```text
oiv7_nav_find_subset_v4/images/val
oiv7_nav_find_subset_v4/labels/val
```

---

## 7. v4 训练集统计摘要

训练集总量：

```text
train images: 67913
negative images: 1978
classes: 61
```

部分大类：

```text
person      img=6661  box=24282
car         img=4225  box=12486
chair       img=2289  box=12121
table       img=2500  box=4013
book        img=2502  box=16333
bottle      img=2502  box=8367
```

部分小类：

```text
pressure_cooker  img=318  box=367
slow_cooker      img=500  box=627
light_switch     img=532  box=685
power_socket     img=800  box=1719
dishwasher       img=533  box=563
toaster          img=300  box=358
fan              img=1000 box=1156
```

说明：

- 小类 box 数量大于原始 raw_box，是因为 oversample 重复采样导致训练中重复出现。
- 这不是凭空增加新信息，只是提高小类在训练中的出现频率。
- 后续真正提高小类泛化能力，仍然需要实拍补数据。

---

## 8. v4 验证集注意事项

验证集总量：

```text
val images: 14187
negative images: 413
```

部分类别验证样本很少：

```text
pressure_cooker  img=1   box=1
slow_cooker      img=2   box=2
dishwasher       img=3   box=3
toaster          img=1   box=1
light_switch     img=10  box=12
power_socket     img=16  box=21
```

结论：

- 这些小类的 per-class mAP 波动会很大。
- 不应只看这些小类的验证 mAP 来判断模型是否可用。
- 后续应使用实际场景图片/视频单独测试。

---

## 9. 训练权重选择

使用已有 OpenImages 训练权重作为起点：

```text
/mnt/data/datasets/openimages/runs/oiv7_half_b8_w4_e20_new/weights/best.pt
```

不要使用：

```text
nav_subset_check_e1/weights/best.pt
```

原因：

- `nav_subset_check_e1` 是早期 v1 类别体系的检查模型。
- v1 类别合并错误，不适合作为本次 61 类英文细类训练起点。

---

## 10. 40 epoch 正式训练命令

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

参数说明：

```text
batch=6          降低 watchdog timeout 风险
workers=2        减少 CPU/IO 干扰
amp=True         混合精度训练
cache=False      不把大数据集塞进内存
mosaic=0.3       保留轻量增强
mixup=0          关闭强混合增强
copy_paste=0     关闭 copy-paste 增强
close_mosaic=10  最后 10 轮关闭 mosaic，更贴近真实图
compile=False    避免 torch compile 额外不稳定
plots=False      不生成训练图片和图表
save_period=5    每 5 轮保存一次，方便中断恢复
```

---

## 11. 训练监控命令

另开终端：

```bash
nvidia-smi
```

查看日志：

```bash
tail -f /mnt/data/datasets/openimages/runs/logs/nav_find_v4_e40_b6.log
```

如果需要看训练进程：

```bash
ps -ef | grep yolo | grep -v grep
```

---

## 12. 中断恢复

如果训练中断，使用 `last.pt` 恢复：

```bash
yolo detect train resume=True model=/mnt/data/datasets/openimages/runs/nav_find_v4/e40_b6/weights/last.pt
```

注意：

- `resume=True` 会沿用原训练配置。
- 如果要改 batch、epochs、增强参数，不建议 resume。
- 改参数时应从某个 `best.pt` 或 `last.pt` 作为 `model=` 开新 run。

---

## 13. 结果判断标准

训练完成后先看：

```text
mAP50
mAP50-95
precision
recall
per-class AP
```

但本任务不能只看总 mAP，因为：

1. 大类样本多，容易拉高总体指标。
2. 小类验证样本太少，per-class AP 波动大。
3. 最终任务是机器人寻物，不是排行榜评测。

更关键的测试：

```text
手机能不能识别
门/门把手能不能识别
桌椅柜子能不能识别
水杯/瓶子/碗盘勺叉能不能识别
开关/插座能不能识别
风扇/空调类是否容易混淆
室内家具是否误检严重
```

---

## 14. 后续数据补强方向

OpenImages 只是第一阶段。

后续必须补实拍数据，优先级如下：

### P0：导盲/安全相关

```text
person
car
bus
truck
bicycle
motorcycle
stairs
door
traffic_light
traffic_sign
fire_hydrant
waste_container
```

### P1：高频寻物

```text
chair
table
bookcase
cupboard
cabinetry
book
mobile_phone
laptop
keyboard
mouse
monitor
backpack
handbag
suitcase
bottle
bowl
plate
spoon
fork
knife
```

### P2：小样本弱类

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

## 15. 当前结论

当前可执行方案：

```text
使用 oiv7_nav_find_subset_v4/data.yaml
使用 61 个英文细类
使用已有 OpenImages best.pt 作为起点
跑 40 epoch
plots=False
batch=6
workers=2
```

当前不再改数据集，先完成一次完整训练，再根据结果决定下一步。

---

## 16. 关键路径汇总

```text
原始数据：
/mnt/data/datasets/openimages/open-images-v7

目标类别：
/mnt/data/datasets/openimages/target_v2_classes.yaml

v4 脚本：
/mnt/data/datasets/openimages/make_v4_subset.py

v4 数据集：
/mnt/data/datasets/openimages/oiv7_nav_find_subset_v4

v4 YOLO yaml：
/mnt/data/datasets/openimages/oiv7_nav_find_subset_v4/data.yaml

预训练权重：
/mnt/data/datasets/openimages/runs/oiv7_half_b8_w4_e20_new/weights/best.pt

训练输出：
/mnt/data/datasets/openimages/runs/nav_find_v4/e40_b6

训练日志：
/mnt/data/datasets/openimages/runs/logs/nav_find_v4_e40_b6.log
```
