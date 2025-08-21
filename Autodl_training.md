
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

 

````markdown
# 数据采集模块

成功完成的数据采集模块的 bash code （采集 50 个 episodes；2 camera）

```bash
python lerobot/scripts/control_robot.py \
--robot.type=alicia_duo \
--control.type=record \
--control.fps=30 \
--control.single_task="pickBox1" \
--control.root=/home/zexuan/Robot/data/pickBox1 \
--control.repo_id=Enstar07/pickBox1 \
--control.num_episodes=50 \
--control.warmup_time_s=15 \
--control.episode_time_s=28 \
--control.reset_time_s=10 \
--control.push_to_hub=true
````

---

### 对应的数据训练模块的 bash code 本地训练跑通测试代码

```bash
python lerobot/scripts/train.py \
  --dataset.repo_id /home/zexuan/Robot/data/pickBox1 \
  --policy.type act \
  --output_dir outputs/train/pickBoxTest  \
  --job_name pickBoxTest \
  --policy.device cuda \
  --batch_size 32 \
  --steps 100  \
  --save_freq 10  \
  --eval_freq 10  \
  --log_freq 10
```

---

### 对应的数据训练模块的 bash code AUTODL 训练代码（采集 50 个 episodes；2 camera）

```bash
python lerobot/scripts/train.py \
  --dataset.repo_id /root/lerobot/data/pickBox1  \
  --policy.type act \
  --output_dir outputs/train/pickBox2  \
  --job_name pickBox2  \
  --policy.device cuda \
  --batch_size 32 \
  --steps 60000 \
  --save_freq 4000 \
  --eval_freq 4000 \
  --log_freq 1000 
```

---

### smolvla 尝试

```bash
python lerobot/scripts/train.py \
  --policy.path=lerobot/smolvla_base \
  --dataset.repo_id=/root/lerobot/data/pickBox1 \
  --output_dir=outputs/train/pickBox1_vla \
  --job_name=pickBox1_vla \
  --policy.device=cuda \
  --batch_size=32 \
  --steps=30000 \
  --save_freq=2000 \
  --eval_freq=1000 \
  --log_freq=1000
```

---

### 成功进行 autoDL 80000 次训练，后台程序训练

```bash
nohup python lerobot/scripts/train.py \
  --dataset.repo_id /root/lerobot/data/pickBox1 \
  --policy.type act \
  --output_dir outputs/train/pickBox3 \
  --job_name pickBox3 \
  --policy.device cuda \
  --batch_size 32 \
  --steps 80000 \
  --save_freq 5000 \
  --eval_freq 5000 \
  --log_freq 500 \
  > train.log 2>&1 &
```

---

### 远程后台训练的用法

你看到的这一行：

```
[3] 2175
```

是正常的提示信息，不是报错。

✅ 它的含义如下：

* `[3]`：这是当前 shell 会话中启动的第 3 个后台任务。
* `2175`：这是这个后台进程的 **进程号（PID）**。

🔍 举例说明：
假如你连续运行了几个训练任务：

```bash
nohup python train1.py > train1.log 2>&1 &   # [1] 1234
nohup python train2.py > train2.log 2>&1 &   # [2] 1567
nohup python train3.py > train3.log 2>&1 &   # [3] 2175
```

系统就会这样编号。

✅ 接下来你可以：

* 查看实时日志：

  ```bash
  tail -f train.log
  ```
* 查看进程是否在运行：

  ```bash
  ps -p 2175
  ```
* 终止训练（如果需要）：

  ```bash
  kill -9 2175
  ```

---

 

````markdown
# 数据训练模块 - AutoDL

## 增大训练参数 (成功训练的)

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
````

---

## AutoDL 50000 次训练 (后台运行)

进行 autoDL 50000 次训练，后台程序训练 **diffusion** 正在运行：2025.07.31

```bash
nohup python lerobot/scripts/train.py \
  --dataset.repo_id /root/lerobot/data/pickBox1 \
  --policy.type tdmpc \
  --output_dir outputs/train/pickBox5 \
  --job_name pickBox5 \
  --policy.device cuda \
  --batch_size 1 \
  --steps 50000 \
  --save_freq 1000 \
  --eval_freq 1000 \
  --log_freq 500 \
  > train.log 2>&1 &
```

 

 


