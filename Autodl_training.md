
# AutoDL 线上 GPU 训练日志 - 2025-08-20

## 一、环境配置

### 1. Conda 初始化与环境创建
```bash
conda env list
conda activate base
conda init
````

重启终端后，进入 `base` 环境：

```bash
(base) root@autodl-container:~# 
```

创建新环境并激活：

```bash
conda create -y -n lerobot python=3.10
conda activate lerobot
```

成功后提示：

```bash
(lerobot) root@autodl-container:~#
```

### 2. 网络加速设置

```bash
source /etc/network_turbo
```

输出：

```
设置成功
注意：仅限于学术用途，不承诺稳定性保证
```

### 3. 拉取代码

```bash
git clone https://github.com/Xuanya-Robotics/lerobot.git
cd lerobot
git clone https://github.com/Xuanya-Robotics/Alicia_duo_sdk.git Alicia_duo_sdk
```

### 4. 依赖安装

```bash
conda install ffmpeg -c conda-forge
pip install -e .
```

---

## 二、Hugging Face 设置

### 1. 配置 Git & 登录 HuggingFace

```bash
git config --global credential.helper store
hf auth login --token  {your huggingface token}
```

输出：

```
The token has been saved to /root/.cache/huggingface/token
Login successful.
The current active token is: `Aloha_test`
```

### 2. 设置 HF\_USER

```bash
HF_USER=$(hf auth whoami | head -n 1) && echo $HF_USER
```

输出：

```
Enstar07
```

---

## 三、训练任务

### 1. 使用 HuggingFace 数据集训练

（可能受限于网络延迟）

```bash
python lerobot/scripts/train.py \
  --dataset.repo_id Enstar07/demo_dataset3 \
  --policy.type act \
  --output_dir outputs/train/act_alicia_duo_model_final \
  --job_name alicia_duo_act_training_final \
  --policy.device cuda \
  --batch_size 32 \
  --steps 2000 \
  --save_freq 500 \
  --eval_freq 500 \
  --log_freq 100
```

### 2. 使用 AutoDL 本地数据训练

（推荐，数据通过 FileZilla 上传）

```bash
python lerobot/scripts/train.py \
  --dataset.repo_id /root/lerobot/data/zhuiPick3 \
  --policy.type act \
  --output_dir outputs/train/act_alicia_duo_model_final \
  --job_name alicia_duo_act_training_final \
  --policy.device cuda \
  --batch_size 32 \
  --steps 2000 \
  --save_freq 500 \
  --eval_freq 500 \
  --log_freq 100
```

### 3. 增大训练参数（成功的配置）

```bash
python lerobot/scripts/train.py \
  --dataset.repo_id /root/lerobot/data/zhuiPick3 \
  --policy.type act \
  --output_dir outputs/train/act_alicia_duo_model_final \
  --job_name alicia_duo_act_training_final \
  --policy.device cuda \
  --batch_size 64 \
  --steps 20000 \
  --save_freq 1000 \
  --eval_freq 1000 \
  --log_freq 500
```

---

## 四、训练结果与部署

* 训练好的模型文件夹用 **FileZilla** 转存到本地。
* 修改 `dp_inference.py`：

  ```python
  ckpt_path = "/home/zexuan/Robot/xuanArm/system2/lerobot/outputs/train/filezilla/020000/pretrained_model"
  ```
* 成功运行，部署到机械臂：

  ```bash
  (lerobot) zexuan@LAPTOP-B0G888QT:~/Robot/xuanArm/system2/lerobot/examples$ python ...
  ```

---
 
 


