# Configuration
## LeRobot installation
```sh
git clone https://github.com/huggingface/lerobot.git --depth 1
cd lerobot
pip install -U pip
pip install -e .
```
## 查看端口
```sh
ls /dev/tty.*
```
- **leader**
```sh
/dev/ttyACM1
```
- **follower**
```sh
/dev/ttyACM0
```
## Calibrate the leader
```
lerobot-calibrate \
  --teleop.type=so101_leader \
  --teleop.port=/dev/ttyACM1 \
  --teleop.id=minerobots_leader
```
## Calibrate the follower
```
lerobot-calibrate \
  --robot.type=so101_follower \
  --robot.port=/dev/ttyACM0 \
  --robot.id=minerobots_follower
```
# Teleoperate
## No display
```
lerobot-teleoperate \
  --robot.type=so101_follower \
  --robot.port=/dev/ttyACM0 \
  --robot.id=minerobots_follower \
  --teleop.type=so101_leader \
  --teleop.port=/dev/ttyACM1 \
  --teleop.id=minerobots_leader \
  --fps=30 \
  --display_data=false
```
## With UI but no cameras
```
lerobot-teleoperate \
  --robot.type=so101_follower \
  --robot.port=/dev/ttyACM0 \
  --robot.id=minerobots_follower \
  --teleop.type=so101_leader \
  --teleop.port=/dev/ttyACM1 \
  --teleop.id=minerobots_leader \
  --fps=60 \
  --display_data=true
```
## 确认摄像头的ID
```
python - << 'PY'
import cv2

for idx in range(6):
    cap = cv2.VideoCapture(idx)
    if not cap.isOpened():
        continue
    print(f"正在显示 index {idx} 的摄像头画面，按 q 退出这个摄像头，切换下一个。")
    while True:
        ret, frame = cap.read()
        if not ret:
            break
        cv2.imshow(f"Camera {idx}", frame)
        if cv2.waitKey(1) & 0xFF == ord('q'):
            break
    cap.release()
    cv2.destroyAllWindows()
PY
```

- index_0: side view，Fps = 30
- index_1: top view，Fps = 25
- index_ 2: arm view，Fps = 30
### 查看camera参数
```
lerobot-find-cameras opencv
```
### teleoperate with 3 cameras
```
lerobot-teleoperate \
    --robot.type=so101_follower \
    --robot.port=/dev/tty.usbmodem5AE60570611 \
    --robot.id=so101_follower_mac \
    --robot.cameras="{side: {type: opencv, index_or_path: 0, width: 1920, height: 1080, fps: 30}, arm: {type: opencv, index_or_path: 2, width: 1920, height: 1080, fps: 30}, top: {type: opencv, index_or_path: 1, width: 1920, height: 1080, fps: 24}}" \
    --teleop.type=so101_leader \
	--teleop.port=/dev/tty.usbmodem5AE60848661 \
    --teleop.id=so101_leader_mac \
    --display_data=true
```
# Record the dataset
To record a dataset (e.g. for `lerobot/pusht` or your own task), use `lerobot-record`.

```bash
TOP_CAM_INDEX=6
SIDE_CAM_INDEX=2
WRIST_CAM_INDEX=4

CAMERAS="{top: {type: opencv, index_or_path: $TOP_CAM_INDEX, width: 640, height: 480, fps: 30}, side: {type: opencv, index_or_path: $SIDE_CAM_INDEX, width: 640, height: 480, fps: 30}, wrist: {type: opencv, index_or_path: $WRIST_CAM_INDEX, width: 640, height: 480, fps: 30}}"
echo "Using camera config: Top=$TOP_CAM_INDEX, Side=$SIDE_CAM_INDEX, Wrist=$WRIST_CAM_INDEX"

lerobot-record \
    --robot.type=so101_follower \
    --robot.port="/dev/ttyACM0" \
    --robot.id=minerobots_follower \
    --robot.cameras="$CAMERAS" \
    --teleop.type=so101_leader \
    --teleop.port="/dev/ttyACM1" \
    --teleop.id=minerobots_leader \
    --display_data=true \
    --dataset.repo_id="cfu/minerobots-hughug" \
    --dataset.num_episodes=15 \
    --dataset.episode_time_s=15 \
    --dataset.reset_time_s=2 \
    --dataset.single_task="pick up the screw driver and release it" \
    --dataset.root="data/so101_dataset_$(date +%Y%m%d_%H%M%S)"
```

# Finetune XVLA
To finetune `lerobot/xvla-base` (which requires `lerobot >= 0.4.3` and specific configuration handling), use the following command.
Note: Ensure you are using the `lerobot_latest` installation which includes the necessary patch for XVLA config loading.

```bash
export PYTHONPATH=/workspace/lerobot_latest/src
# export PYTHONPATH=/workspace/lerobot/src
export HF_USER=cfu
# export DATASET_NAME=minerobots-pick
export DATASET_NAME=minerobots-pick-screwdriver
# export DATASET_NAME=minerobots-pick-redball
export REPO_NAME=minerobots_screwdriver_xvla_50k
lerobot-train \
  --dataset.repo_id=cfu/${DATASET_NAME} \
  --policy.optimizer_lr=0.0001 \
  --steps=50000 \
  --policy.action_mode=auto \
  --policy.type=xvla \
  --policy.pretrained_path=lerobot/xvla-base \
  --output_dir=outputs/train/${REPO_NAME} \
  --job_name=${REPO_NAME} \
  --policy.device=cuda \
  --wandb.enable=true \
  --policy.push_to_hub=true \
  --policy.repo_id=${HF_USER}/${REPO_NAME} \
  --policy.resize_imgs_with_padding=[224,224] 
```

# Finetune ACT from pre-trained
minerobots-hughug

minerobots-screwdriver
```bash
export HF_USER=cfu
export DATASET_NAME=minerobots-screwdriver
export REPO_NAME=minerobots_screwdriver_act_50k_0
lerobot-train \
  --dataset.repo_id=cfu/${DATASET_NAME} \
  --steps=30000 \
  --policy.type=act \
  --policy.pretrained_path=cfu/minerobots-srewdriver-act-20k \
  --output_dir=outputs/train/${REPO_NAME} \
  --job_name=${REPO_NAME} \
  --policy.device=cuda \
  --wandb.enable=true \
  --policy.push_to_hub=true \
  --policy.repo_id=${HF_USER}/${REPO_NAME}
#   --policy.pretrained_path=cfu/minerobots-ball-act-100k \
#   --policy.optimizer_lr=0.0001 \
```

# Inference

#### hug_n_play
minerobots-srewdriver-act-20k
#### hughug
minerobots_hughug_act_10k_0
```bash
lerobot-record \
  --robot.type=so101_follower \
  --robot.port=/dev/ttyACM0 \
  --robot.cameras="{side: {type: opencv, index_or_path: 2, width: 640, height: 480, fps: 30}, top: {type: opencv, index_or_path: 6, width: 640, height: 480, fps: 30}, wrist: {type: opencv, index_or_path: 4, width: 640, height: 480, fps: 30}}" \
  --policy.path=cfu/minerobots-srewdriver-act-20k \
  --dataset.repo_id=lerobot/eval_inference_v57 \
  --dataset.single_task="inference_act" \
  --dataset.num_episodes=10 \
  --dataset.reset_time_s=0 \
  --display_data=false
```

> **Note**: If `policy.device=cuda` is not available (e.g. on pure CPU or misconfigured ROCm), switch to `cpu`, but training will be very slow.
# Change motor
## 查看电机的ID
```py
python - << 'PY'
from scservo_sdk import PortHandler, PacketHandler, COMM_SUCCESS

# TODO: 改成你刚才 ls 出来的那个串口名
PORT = "/dev/tty.usbmodem5AE60570611"

# 常见波特率（STS 系列默认 1000000）
baud_candidates = [1000000, 500000, 250000, 115200]

for baud in baud_candidates:
    port = PortHandler(PORT)
    if not port.openPort():
        print("❌ 无法打开串口", PORT)
        break

    if not port.setBaudRate(baud):
        print("❌ 设置波特率失败", baud)
        port.closePort()
        continue

    packet = PacketHandler(0)  # 协议版本 0：STS/SMS/SCS 系列用的
    print(f"🔍 在波特率 {baud} 下扫描 ID 0~30...")

    found = False
    for sid in range(0, 31):
        model, comm_result, error = packet.ping(port, sid)
        if comm_result == COMM_SUCCESS:
            print(f"✅ 找到舵机: ID={sid}, 波特率={baud}, 型号={model}")
            found = True

    port.closePort()

    if found:
        print("🎉 扫描结束，上面就是这个电机当前的 ID 和波特率")
        break
else:
    print("⚠️ 在常见波特率(1000000/500000/250000/115200) 和 ID 0~30 范围内没扫到这只舵机")
PY
```
## 修改电机的ID
```py
python - << 'PY'
from scservo_sdk import PortHandler, PacketHandler, COMM_SUCCESS

# ===== 根据实际情况修改这里 =====
PORT     = "/dev/tty.usbmodem5AE60570611"
BAUDRATE = 1000000          # STS3215 默认 1M
OLD_ID   = 1                # 现在上电后的 ID
NEW_ID   = 4                # 想要改成的 ID

# ===== ST3215 寄存器地址 =====
ADDR_ID   = 0x05            # ID 在 EPROM 的地址
ADDR_LOCK = 0x37            # Lock flag 在 SRAM 的地址

port = PortHandler(PORT)
if not port.openPort():
    print("❌ 无法打开串口", PORT)
    raise SystemExit

if not port.setBaudRate(BAUDRATE):
    print("❌ 无法设置波特率", BAUDRATE)
    port.closePort()
    raise SystemExit

packet = PacketHandler(0)   # Feetech STS 协议版本 0

# 1) 先确认 OLD_ID 能 ping 通
print(f"🔍 先 ping 当前 ID={OLD_ID} ...")
model, comm_result, error = packet.ping(port, OLD_ID)
if comm_result != COMM_SUCCESS:
    print("⚠️ ping 失败，确认 OLD_ID/BAUDRATE 是否正确")
    port.closePort()
    raise SystemExit
print(f"✅ 找到舵机，model={model}")

# 2) 解锁 EEPROM：Lock flag 写 0
print("🔓 正在解锁 EEPROM (Lock flag=0)...")
dxl_comm_result, dxl_error = packet.write1ByteTxRx(port, OLD_ID, ADDR_LOCK, 0)
if dxl_comm_result != COMM_SUCCESS:
    print("❌ 解锁通信错误:", packet.getTxRxResult(dxl_comm_result))
    port.closePort()
    raise SystemExit
if dxl_error != 0:
    print("⚠️ 舵机返回错误:", packet.getRxPacketError(dxl_error))
print("✅ 已写 Lock=0（解锁）")

# 3) 写入新的 ID=4 到 EPROM
print(f"✏️ 正在把 ID 从 {OLD_ID} 改成 {NEW_ID} ...")
dxl_comm_result, dxl_error = packet.write1ByteTxRx(port, OLD_ID, ADDR_ID, NEW_ID)
if dxl_comm_result != COMM_SUCCESS:
    print("❌ 改 ID 通信错误:", packet.getTxRxResult(dxl_comm_result))
    port.closePort()
    raise SystemExit
if dxl_error != 0:
    print("⚠️ 舵机返回错误:", packet.getRxPacketError(dxl_error))
print("✅ 写 ID=4 指令已发出")

# 4) 用新 ID 再 ping 一次确认
print(f"🔍 用新 ID={NEW_ID} 再 ping 一次...")
model, comm_result, error = packet.ping(port, NEW_ID)
if comm_result == COMM_SUCCESS:
    print(f"🎉 改 ID 成功！现在舵机响应 ID={NEW_ID}")
else:
    print("⚠️ 用新 ID ping 失败，请检查")

# 5) （可选）重新上锁：Lock flag 写 1，防止以后误写 EEPROM
print("🔐 （可选）重新锁定 EEPROM (Lock flag=1)...")
dxl_comm_result, dxl_error = packet.write1ByteTxRx(port, NEW_ID, ADDR_LOCK, 1)
if dxl_comm_result != COMM_SUCCESS:
    print("⚠️ 上锁通信错误:", packet.getTxRxResult(dxl_comm_result))
elif dxl_error != 0:
    print("⚠️ 舵机返回错误:", packet.getRxPacketError(dxl_error))
else:
    print("✅ 已写 Lock=1（上锁）")

port.closePort()
PY
```


