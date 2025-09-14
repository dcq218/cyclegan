# 3D PET 图像合成与分类项目 (基于 CycleGAN 和 3D ResNet)

本项目旨在使用 CycleGAN 模型实现两种不同 3D PET 医学影像（FDG-PET 与 CFT-PET）之间的无监督图像到图像转换。此外，项目还利用生成的伪图像来训练一个 3D ResNet 分类器，用于下游的临床诊断任务（例如，帕金森病 vs. 正常对照）。

整个流程使用 PyTorch 框架实现，并集成了 TensorBoard 进行详细的可视化监控。

## ✨ 主要功能

- **3D CycleGAN 训练**:
  - **模型**: 生成器采用 3D U-Net 架构，判别器采用 3D PatchGAN 架构。
  - **数据集**: 支持 `.nii` 格式的 3D 医学影像，并能智能处理复杂的混合数据集（包含成对与非成对样本）。
  - **监控**: 全程使用 TensorBoard 记录损失变化和图像生成效果（包括真实图像、伪图像及误差图）。
  - **存档**: 每次训练都会生成带时间戳的独立文件夹，整洁地保存所有模型权重、日志和 TensorBoard 记录。

- **3D 分类器训练**:
  - **模型**: 采用 3D ResNet-18 架构进行分类。
  - **数据**: 使用 CycleGAN 生成的伪图像作为训练数据。
  - **损失函数**: 使用 Focal Loss 来处理可能存在的类别不平衡问题。
  - **监控**: 使用 TensorBoard 记录训练过程中的损失下降和验证集准确率变化。

- **模块化代码**:
  - **配置**: 所有超参数均通过 `options.py` 统一管理，易于调整。
  - **脚本**: 功能分离，`trainer.py` 负责训练，`tester.py` 负责测试，`generate_fakes.py` 负责生成数据。

## 🔧 环境与安装

本项目基于 Python 和 PyTorch。建议您使用 `conda` 或 `venv` 创建一个独立的虚拟环境。

**1. 克隆项目 (如果您已上传到 GitHub):**
```bash
git clone [您的 GitHub 项目链接]
cd [项目目录]
```

**2. 安装依赖库:**
建议创建一个 `requirements.txt` 文件，并将以下内容复制进去，然后通过 `pip` 安装。

*requirements.txt:*
```
torch
torchvision
numpy
nibabel
matplotlib
tensorboard
```

通过以下命令安装：
```bash
pip install -r requirements.txt
```

## 📂 数据集准备

本项目需要特定的目录结构来正确解析成对和非成对数据。请在项目根目录下创建一个 `datasets` 文件夹，并按如下结构组织您的 `.nii` 文件：

```
datasets/
├── NC_paired_FDG+CFT/
│   ├── 42632/
│   │   ├── ..._FDG.nii
│   │   └── ..._CFT.nii
│   └── .../
│
├── PD_paired_FDG+CFT/
│   ├── 18726/
│   │   ├── ..._FDG.nii
│   │   └── ..._CFT.nii
│   └── .../
│
└── NC_unpaired_FDG+CFT/
    ├── FDG/
    │   ├── ..._FDG.nii
    │   └── ...
    └── CFT/
        ├── ..._CFT.nii
        └── ...
```

## 🚀 使用指南

整个工作流程分为四个主要阶段：训练CycleGAN -> 生成伪图像 -> 训练分类器 -> 测试。

### 阶段一：训练 CycleGAN

此命令将启动 CycleGAN 的训练过程。所有日志、模型权重和 TensorBoard 文件将保存在 `./checkpoints/` 目录下的一个带时间戳的新文件夹中。

```bash
python main.py --mode train --dataroot "./datasets" --epochs 200 --batch_size 1 --device "cuda:0"
```

**监控训练过程**:
训练开始后，打开一个**新的终端**，运行以下命令启动 TensorBoard：
```bash
tensorboard --logdir=./checkpoints
```
然后在浏览器中打开 `http://localhost:6006/`。

### 阶段二：生成用于分类的伪图像

训练完 CycleGAN 后，使用此命令加载训练好的生成器，将所有源 FDG 图像转换为伪 CFT 图像，并保存在 `./generated_images` 目录中。文件名将自动包含 `NC` 或 `PD` 标签。

```bash
python generate_fakes.py --dataroot "./datasets" --generated_dir "./generated_images" --load_epoch 200 --device "cuda:0"
```
- `--load_epoch`: 指定加载哪个轮次保存的生成器权重。

### 阶段三：训练分类器

使用上一步生成的图像来训练 3D ResNet 分类器。所有相关文件将保存在 `./checkpoints_classifier/` 目录下的新文件夹中。

```bash
python train_classifier.py --classifier_dataroot "./generated_images" --classifier_epochs 100 --batch_size 2 --device "cuda:0"
```

**监控分类器训练**:
打开一个**新的终端**，运行：
```bash
tensorboard --logdir=./checkpoints_classifier
```

### 阶段四：运行推理 (测试)

使用训练好的生成器转换单个或多个测试图像。

1.  将待测试的 FDG `.nii` 文件放入一个文件夹（例如 `./test_A`）。
2.  运行以下命令：

```bash
python main.py --mode test --dataroot "./test_A" --result_dir "./results" --load_epoch 200 --device "cuda:0"
```
- `--dataroot`: 指向测试图像所在的文件夹。
- `--result_dir`: 指定转换后图像的保存位置。

## 📄 项目结构

```
.
├── main.py                   # 主入口: 选择训练或测试CycleGAN
├── options.py                # 管理所有命令行参数
├── trainer.py                # CycleGAN 训练器核心逻辑
├── tester.py                 # CycleGAN 测试/推理逻辑
├── dataset.py                # CycleGAN 数据集加载器 (处理混合数据)
|
├── generate_fakes.py         # 批量生成伪图像的脚本
├── train_classifier.py       # 分类器训练核心逻辑
├── classifier_dataset.py     # 分类器数据集加载器
|
├── models/
│   ├── __init__.py
│   ├── generator.py          # 3D U-Net 生成器
│   ├── discriminator.py      # 3D PatchGAN 判别器
│   └── classification.py     # 3D ResNet 分类器
|
├── datasets/                 # (需自建) 存放原始 .nii 数据
├── checkpoints/              # (自动生成) 保存CycleGAN训练结果
├── checkpoints_classifier/   # (自动生成) 保存分类器训练结果
└── generated_images/         # (自动生成) 保存生成的伪图像
```

## 🤝 致谢

本项目的 CycleGAN 实现参考了 [Jun-Yan Zhu](https://junyanz.github.io/) 等人发表的原始论文：
> [*Unpaired Image-to-Image Translation using Cycle-Consistent Adversarial Networks*](https://arxiv.org/abs/1703.10593)

## 📜 许可证

(您可以在此处添加您选择的许可证
