

````markdown
# WSL to Ubuntu 设置方案

## 方案：解决摄像头和机械臂的 WSL to Ubuntu 问题

### 步骤 1：先全部 bind  
在多个 PowerShell 窗口中分别执行：

```powershell
usbipd bind --busid 1-4
usbipd bind --busid 1-9
usbipd bind --busid 3-1
usbipd bind --busid 3-2
````

### 步骤 2：attach 到 WSL

同样在 PowerShell 中执行：

```powershell
usbipd attach --busid 1-4 --wsl --auto-attach
usbipd attach --busid 1-9 --wsl --auto-attach
usbipd attach --busid 3-2 --wsl --auto-attach
usbipd attach --busid 3-3 --wsl --auto-attach
```

---

### 步骤 3：在 Ubuntu 里执行

#### 一键完成机械臂的 USB

```bash
sudo modprobe cdc_acm && sudo modprobe ch341 && sudo chown root:dialout /dev/ttyACM* && sudo chmod 666 /dev/ttyACM*
```

#### 一键完成的摄像头

```bash
sudo modprobe videodev && sudo modprobe uvcvideo && sudo chgrp video /dev/video* && sudo chmod 666 /dev/video*
```

#### 一键完成机械臂的摄像头和机械臂的USB：

```bash
sudo modprobe cdc_acm && sudo modprobe ch341 && sudo chown root:dialout /dev/ttyACM* && sudo chmod 666 /dev/ttyACM* && sudo modprobe videodev && sudo modprobe uvcvideo && sudo chgrp video /dev/video* && sudo chmod 666 /dev/video*
```

---

### 步骤 4：最后检测

```bash
ls -l /dev/ttyACM*
ls -l /dev/video*
```


#### 验证摄像头编号，检测camera脚本：
``` 

ffplay -f v4l2 -input_format mjpeg -video_size 640x480 -framerate 30 -i /dev/video0
```
