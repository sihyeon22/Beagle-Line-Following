# Beagle Fusion Run Guide (Windows + WSL2 + ROS2)

이 문서는 아래 파일만 가지고 새 PC에서 전체 파이프라인을 실행하는 방법입니다.

## 1. 준비 파일

아래 6개 파일을 준비합니다.

- `beagle/train/traffic_light_best.pt`
- `beagle/train/speed_best.pt`
- `beagle/fusion_sender.py`
- `beagle/beagle_udp_bridge.py`
- `wsl_self_drive/mvp_fusion_controller_node.py`
- `wsl_self_drive/wsl2_cmdvel_udp_sender.py`

권장 폴더 구조:

```text
C:\Users\USER\Desktop\beagle\
  fusion_sender.py
  beagle_udp_bridge.py
  train\traffic_light_best.pt
  train\speed_best.pt

C:\Users\USER\Desktop\wsl_self_drive\
  mvp_fusion_controller_node.py
  wsl2_cmdvel_udp_sender.py
```

---

## 2. Windows 설치

### 2.1 Python 설치

- Python 3.10 또는 3.11 권장

확인:

```powershell
python --version
pip --version
```

### 2.2 Windows Python 모듈 설치

```powershell
pip install ultralytics opencv-python numpy roboid
```

---

## 3. WSL2 + Ubuntu 설치

PowerShell(관리자)에서:

```powershell
wsl --install
```

재부팅 후 Ubuntu 초기 계정을 생성합니다.

확인:

```powershell
wsl -l -v
```

---

## 4. WSL(Ubuntu)에서 ROS2 설치 (Humble 기준)

Ubuntu 터미널에서:

```bash
sudo apt update && sudo apt install -y software-properties-common curl gnupg lsb-release
sudo add-apt-repository universe -y
sudo curl -sSL https://raw.githubusercontent.com/ros/rosdistro/master/ros.key -o /usr/share/keyrings/ros-archive-keyring.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/ros-archive-keyring.gpg] http://packages.ros.org/ros2/ubuntu $(. /etc/os-release && echo $UBUNTU_CODENAME) main" | sudo tee /etc/apt/sources.list.d/ros2.list > /dev/null
sudo apt update
sudo apt install -y ros-humble-ros-base python3-colcon-common-extensions python3-rosdep ros-humble-geometry-msgs ros-humble-rclpy python3-pip
sudo rosdep init
rosdep update
echo "source /opt/ros/humble/setup.bash" >> ~/.bashrc
source ~/.bashrc
```

확인:

```bash
ros2 --version
```

---

## 5. IP 확인 (매우 중요)

### 5.1 WSL IP (Windows -> WSL UDP용)

WSL에서:

```bash
hostname -I
```

예: `172.17.20.88`

### 5.2 Windows Host IP (WSL -> Windows UDP용)

Windows에서:

```powershell
ipconfig
```

`vEthernet (WSL)` 또는 `vEthernet (WSL (Hyper-V firewall))`의 IPv4를 확인합니다.

예: `172.17.16.1`

---

## 6. 실행 순서 (권장)

순서를 지키는 것이 안정적입니다.

### 6.1 Windows 터미널 #1: Beagle Bridge

```powershell
python C:\Users\USER\Desktop\beagle\beagle_udp_bridge.py
```

### 6.2 WSL 터미널 #1: Fusion Controller

```bash
cd /mnt/c/Users/USER/Desktop/wsl_self_drive
python3 mvp_fusion_controller_node.py
```

### 6.3 WSL 터미널 #2: /cmd_vel -> Windows UDP

아래 `win_ip`는 5.2에서 확인한 Windows IP로 변경합니다.

```bash
cd /mnt/c/Users/USER/Desktop/wsl_self_drive
python3 wsl2_cmdvel_udp_sender.py --ros-args -p win_ip:=172.17.16.1 -p win_port:=9999
```

### 6.4 Windows 터미널 #2: Fusion Sender (카메라 1개로 line + yolo)

아래 `--wsl-ip`는 5.1에서 확인한 WSL IP로 변경합니다.

```powershell
python C:\Users\USER\Desktop\beagle\fusion_sender.py --source 0 --traffic-model C:\Users\USER\Desktop\beagle\train\traffic_light_best.pt --speed-model C:\Users\USER\Desktop\beagle\train\speed_best.pt --wsl-ip 172.17.20.88 --line-port 10002 --yolo-port 10000 --show --show-mask
```

---

## 7. 동작 로직 요약

Fusion 우선순위:

1. `traffic_red` 감지 -> 정지 (기본 5초, `green`이면 조기 해제)
2. 라인 유효성 체크 (`found`, `m00`, timeout)
3. 속도 표지판 오버라이드
   - `speed_30` -> 8, 5초
   - `speed_80` -> 25, 3초
4. 라인 에러 기반 조향(PD)

---

## 8. 체크포인트

### 8.1 WSL에서 /cmd_vel 확인

```bash
ros2 topic echo /cmd_vel
```

### 8.2 로그 확인 포인트

- `mvp_fusion_controller_node.py`: `RED/GREEN`, `SPEED 30/80` 상태 로그
- `wsl2_cmdvel_udp_sender.py`: `Sent cmd_vel` 로그
- `beagle_udp_bridge.py`: `[RECV]`, `[MOTOR]` 로그

---

## 9. 자주 발생하는 문제

### 9.1 모델 파일 경로 오류

오류 예: `FileNotFoundError ... traffic_best.pt`

해결: 실제 파일명 확인 후 정확히 지정

- `traffic_light_best.pt`
- `speed_best.pt`

### 9.2 카메라 프레임 grab 실패 (`MSMF` 경고)

원인: 카메라 중복 점유 또는 백엔드 충돌

해결:
- 카메라 앱/Zoom/브라우저 종료
- `fusion_sender.py`만 카메라를 열도록 유지
- 다른 sender 동시 실행 금지

### 9.3 통신이 안 될 때

- WSL IP / Windows IP를 서로 반대로 넣지 않았는지 확인
- 방화벽에서 Python UDP 통신 허용

---

## 10. 종료 순서

권장 종료 순서:

1. `fusion_sender.py`
2. `wsl2_cmdvel_udp_sender.py`
3. `mvp_fusion_controller_node.py`
4. `beagle_udp_bridge.py`

