# Feetech Motor Test Script

A simple test script for Feetech motors with **scan, read, move, sweep, and torque control**.  
Supports three input units: `degrees`, `m100`, and `raw`.

好的 👍 我帮你把上面的内容整理成一份 **口语化、简单英文表达的 README.md**，保持所有核心内容不变，只优化格式和表述：

````markdown
# Feetech Motor Test Script

This script supports:  
- Scan motors on the bus (`scan`)  
- Read current position (`read`)  
- Move to a specific position (`move`)  
- Sweep back and forth in a range (`sweep`)  
- Turn torque on/off (`torque_on` / `torque_off`)  

It also supports three types of input units:  
- **degrees** (recommended for arm joints)  
- **m100** (normalized, from -100 to 100)  
- **raw** (original counts, no normalization)  

> Note: The script uses the `FeetechMotorsBus` class, and works with `Goal_Position` / `Present_Position`.  
You can switch normalization on or off when needed.

---

### Based on
This script is based on [Feetech Motor Code](https://github.com/SpaceEnstar01/so101/tree/main/lerobot/src/lerobot/motors/feetech).

---

## Usage Examples

1. **Scan motors** (see which IDs are available) ✅
```bash
python test_feetech_motor.py --port /dev/ttyACM0 scan
````

2. **Read current position (in degrees)** ✅

```bash
python test_feetech_motor.py --port /dev/ttyACM0 --id 1 --unit degrees read
```

3. **Move motor to 10°**

```bash
python test_feetech_motor.py --port /dev/ttyACM0 --id 1 --unit degrees move --value 10
```

4. **Sweep between -30° and 30°**

```bash
python test_feetech_motor.py --port /dev/ttyACM0 --id 1 --unit degrees sweep --min -30 --max 30 --step 5 --delay 0.4
```

5. **Use normalized units (-100 to 100)**

```bash
python test_feetech_motor.py --port /dev/ttyACM0 --id 2 --unit m100 move --value 50
```

6. **Use raw counts (⚠ careful!)**

```bash
python test_feetech_motor.py --port /dev/ttyACM0 --id 3 --unit raw move --value 2048
```

7. **Turn torque on/off**

```bash
python test_feetech_motor.py --port /dev/ttyACM0 --id 1 torque_on
python test_feetech_motor.py --port /dev/ttyACM0 --id 1 torque_off
```





```python 
# test_feetech_motor.py
#!/usr/bin/env python3
# -*- coding: utf-8 -*-

import argparse
import time
import sys

from lerobot.motors import Motor, MotorNormMode
from lerobot.motors.feetech import FeetechMotorsBus, OperatingMode


def build_bus(port: str, motor_id: int, model: str, unit: str, protocol: int):
    """
    创建仅包含一个电机的总线对象，便于单独调试。
    """
    if unit == "degrees":
        norm = MotorNormMode.DEGREES
    elif unit == "m100":
        norm = MotorNormMode.RANGE_M100_100
    elif unit == "raw":
        # raw 模式我们通过 normalize=False 读写，norm 取什么都不会影响 raw 结果
        norm = MotorNormMode.RANGE_M100_100
    else:
        raise ValueError(f"Unknown unit: {unit}")

    motors = {"joint": Motor(motor_id, model, norm)}
    bus = FeetechMotorsBus(
        port=port,
        motors=motors,
        calibration=None,
        protocol_version=protocol,
    )
    return bus


def set_position_mode(bus: FeetechMotorsBus, name="joint"):
    # 配置电机基础参数 & 切到位置模式
    bus.configure_motors()
    bus.write("Operating_Mode", name, OperatingMode.POSITION.value)


def torque(bus: FeetechMotorsBus, enable: bool, name="joint"):
    if enable:
        bus.enable_torque(name)
    else:
        bus.disable_torque(name)

def read_pos(bus: "FeetechMotorsBus", unit: str, name: str = "joint"):
    """
    读取电机当前位置并根据 unit 返回不同单位：
      - "raw"     : 原始计数（int）
      - "degrees" : 按 formula angle = raw / 4095 * 360 返回角度（float）
      - "m100"    : 归一化到 [-100, 100]（float）

    注意：
      - 这里强制使用 normalize=False 读取原始计数，避免依赖 calibration 文件。
      - 假设原始计数范围为 [0, 4095] 映射到 [0°, 360°]。如需不同映射（例如中心为 2048 对应 0° 等），请告知。
    """
    # 先读入原始计数（不依赖 calibration）
    raw_val = bus.read("Present_Position", name, normalize=False)

    # 确保 raw_val 是数值
    try:
        raw = float(raw_val)
    except Exception:
        # 如果 bus.read 返回非数值（异常或 None），直接返回原值
        return raw_val

    if unit == "raw":
        # 返回原始计数（保留 int 语义）
        return int(raw)

    elif unit == "degrees":
        # raw -> degrees 映射（假设 0..4095 -> 0..360）
        # 使用 4095 而非 4096，遵循你给出的公式
        angle_deg = (raw / 4095.0) * 360.0
        return float(angle_deg)

    elif unit == "m100":
        # raw -> [-100, 100]
        m100 = (raw / 4095.0) * 200.0 - 100.0
        return float(m100)

    else:
        raise ValueError(f"Unknown unit '{unit}'. Expected one of: raw, degrees, m100.")


def move_to(bus: FeetechMotorsBus, target, unit: str, name="joint", wait=True, tol=None, timeout=5.0):
    """
    位置控制。target 单位取决于 unit：
    - degrees: 角度（°）
    - m100: -100~100
    - raw: 原始计数
    """
    if unit == "raw":
        bus.write("Goal_Position", name, int(target), normalize=False)
    else:
        bus.write("Goal_Position", name, float(target), normalize=True)

    if not wait:
        return

    # 默认容差
    if tol is None:
        tol = 1 if unit == "raw" else (0.5 if unit == "degrees" else 1.0)  # 0.5° / 1 m100 / 1 tick

    t0 = time.time()
    while True:
        cur = read_pos(bus, unit, name)
        err = abs(cur - target)
        if err <= tol:
            break
        if time.time() - t0 > timeout:
            print(f"[WARN] wait timeout. target={target} cur={cur} err={err}")
            break
        time.sleep(0.02)


def op_scan(port: str, protocol: int):
    """
    扫描总线电机；因 Feetech 总线实现写在类里，这里构造一个临时 bus 来调用 broadcast_ping。
    """
    # 构造一个占位电机（模型随便用常见 sts3215），只为实例化 bus；不会对该 ID 进行读写。
    bus = FeetechMotorsBus(
        port=port,
        motors={"dummy": Motor(1, "sts3215", MotorNormMode.RANGE_M100_100)},
        calibration=None,
        protocol_version=protocol,
    )
    bus.connect(handshake=False)
    try:
        ids_models = bus.broadcast_ping(raise_on_error=False)
        if not ids_models:
            print("No motors found.")
            return
        print("Found motors (id -> model_number):")
        for mid, mnum in sorted(ids_models.items()):
            print(f"  {mid} -> {mnum}")
    finally:
        bus.disconnect()


def main():
    ap = argparse.ArgumentParser(description="Single Feetech motor tester (SO101 leader friendly)")
    ap.add_argument("--port", required=True, help="Serial port, e.g. /dev/ttyACM0")
    ap.add_argument("--id", type=int, default=1, help="Motor ID (default 1)")
    ap.add_argument("--model", default="sts3215", help="Motor model (default sts3215)")
    ap.add_argument("--protocol", type=int, default=0, choices=[0, 1], help="Feetech protocol (default 0)")
    ap.add_argument("--unit", choices=["degrees", "m100", "raw"], default="degrees",
                    help="Command/feedback unit (default degrees)")
    ap.add_argument("cmd", choices=["scan", "read", "move", "sweep", "torque_on", "torque_off", "home"],
                    help="Operation")

    # move/sweep args
    ap.add_argument("--value", type=float, help="Target for move/home (deg / m100 / raw)")
    ap.add_argument("--min", dest="vmin", type=float, help="Sweep min (deg / m100 / raw)")
    ap.add_argument("--max", dest="vmax", type=float, help="Sweep max (deg / m100 / raw)")
    ap.add_argument("--step", type=float, default=None, help="Sweep step")
    ap.add_argument("--delay", type=float, default=0.5, help="Delay between sweep steps (s)")
    ap.add_argument("--timeout", type=float, default=5.0, help="Wait timeout (s) for move/home")
    ap.add_argument("--tol", type=float, default=None, help="Position tolerance for move/home")

    args = ap.parse_args()

    if args.cmd == "scan":
        op_scan(args.port, args.protocol)
        return

    # 其他命令需要一个具体电机
    bus = build_bus(args.port, args.id, args.model, args.unit, args.protocol)
    bus.connect()
    try:
        set_position_mode(bus)
        if args.cmd == "torque_on":
            torque(bus, True)
            print("Torque enabled.")
            return
        if args.cmd == "torque_off":
            torque(bus, False)
            print("Torque disabled.")
            return

        if args.cmd == "read":
            val_raw = bus.read("Present_Position", "joint", normalize=False)
            val_unit = read_pos(bus, args.unit, "joint")
            print(f"Present_Position raw={val_raw}, {args.unit}={val_unit}")
            return

        # 运动类操作先上扭矩
        torque(bus, True)

        if args.cmd == "home":
            # 默认 home 到 0（degrees/m100）或到当前 raw 值（如果 value 未给则不动）
            target = args.value if args.value is not None else (0.0 if args.unit != "raw" else read_pos(bus, "raw"))
            print(f"Go home -> {target} ({args.unit})")
            move_to(bus, target, args.unit, wait=True, tol=args.tol, timeout=args.timeout)
            cur = read_pos(bus, args.unit)
            print(f"Reached: {cur} ({args.unit})")
            return

        if args.cmd == "move":
            if args.value is None:
                print("ERROR: --value is required for move", file=sys.stderr)
                sys.exit(2)
            print(f"Move -> {args.value} ({args.unit})")
            move_to(bus, args.value, args.unit, wait=True, tol=args.tol, timeout=args.timeout)
            cur = read_pos(bus, args.unit)
            print(f"Reached: {cur} ({args.unit})")
            return

        if args.cmd == "sweep":
            if args.vmin is None or args.vmax is None:
                print("ERROR: --min/--max are required for sweep", file=sys.stderr)
                sys.exit(2)
            vmin, vmax = args.vmin, args.vmax
            if vmin > vmax:
                vmin, vmax = vmax, vmin

            # 默认步长
            if args.step is None:
                if args.unit == "raw":
                    step = max(1, int((vmax - vmin) / 20))
                elif args.unit == "degrees":
                    step = max(1.0, (vmax - vmin) / 20.0)
                else:
                    step = max(1.0, (vmax - vmin) / 20.0)
            else:
                step = args.step

            seq = []
            x = vmin
            while x <= vmax + 1e-9:
                seq.append(x)
                x += step
            seq += list(reversed(seq[:-1]))  # 回程

            print(f"Sweep {vmin} -> {vmax} ({args.unit}), step={step}")
            for tgt in seq:
                move_to(bus, tgt, args.unit, wait=True, tol=args.tol, timeout=args.timeout)
                cur = read_pos(bus, args.unit)
                print(f" at {cur} / target {tgt}")
                time.sleep(args.delay)
            return

    finally:
        # 出于安全，测试结束默认下扭矩再断开
        try:
            torque(bus, False)
        except Exception:
            pass
        bus.disconnect()


if __name__ == "__main__":
    main()



