# Beagle Line Follow (Windows + WSL + ROS2 + UDP) – README

이 프로젝트는 **Windows(인지/GUI) + WSL(ROS2 제어) + UDP(느슨한 결합) + Windows 하드웨어 브리지(모터 구동)** 구조로 비글 로봇 라인 추종을 수행합니다.  
카메라 영상 처리(OpenCV)와 디버깅은 **Windows**에서, 제어 로직(PD 제어, /cmd_vel)은 **WSL의 ROS2 노드**에서 수행하도록 분리했습니다.

---

## 1. 전체 제어 구조 (Architecture)

### 역할 분담

- **Windows (Perception / Vision / GUI)**
  - 카메라 스트림 수신 및 라인 검출(ROI + HSV threshold + moments centroid)
  - 인지 결과를 **UDP로 WSL에 전송**
  - Windows에서 **Beagle API(roboid)** 로 모터를 직접 구동하는 **하드웨어 브리지** 실행

- **WSL (ROS2 Control)**
  - UDP로 들어온 인지 결과를 기반으로 **PD 제어 수행**
  - 표준 토픽 **/cmd_vel** 퍼블리시
  - /cmd_vel을 다시 UDP로 Windows에 전달 (모터 구동용)

- **UDP (Windows ↔ WSL 연결)**
  - Windows → WSL: 인지 결과 전달 (line follow error, found, m00 등)
  - WSL → Windows: 최종 모터 명령 (/cmd_vel 기반 linear_x, angular_z) 전달
  - JSON payload 사용 → **디버깅/확장 용이**, 환경 분리로 개발 편의성 ↑

---

## 2. 왜 이렇게 분리했나? (Design Rationale)

- **Windows**: 카메라/GUI/OpenCV 작업이 편하고 드라이버/디스플레이 환경이 안정적
- **WSL**: ROS2 기반 제어 노드 운영이 편하고 패키지/의존성 관리가 쉬움
- **UDP**: 두 환경을 느슨하게 연결해 **독립 개발/디버깅/확장**이 쉬움  
  (라인 팔로우 외에도 YOLO, LiDAR 이벤트 등 같은 패턴으로 확장 가능)

---

## 3. ROS2의 역할

- 제어 노드(`mvp_*_controller_node`)가 ROS2 Node로 동작
- 제어 출력은 표준 토픽 **/cmd_vel** 사용
- 다른 기능(예: YOLO, LiDAR, line follow)도 동일한 방식으로 노드 추가/확장 가능

---

## 4. UDP 통신 요약

### 방향 1) Windows → WSL (인지 결과)
- 예: line follow error, found, m00
- JSON payload 사용

### 방향 2) WSL → Windows (최종 모터 명령)
- /cmd_vel(linear_x, angular_z)을 UDP(JSON)로 전달
- Windows에서 roboid(Beagle) wheel 명령으로 변환 후 모터 구동

---

## 5. 핵심 포트 / IP 정리

### 포트(Port)
- **10002** : `line_follow_sender_ver2.py` (Windows) → `mvp_line_follow_controller_node_ver2.py` (WSL)
- **9999**  : `wsl2_cmdvel_udp_sender.py` (WSL) → `beagle_udp_bridge.py` (Windows)

### IP 방향(IP)
- **Windows → WSL 전송 시**
  - 목적지 = **WSL IP** (예: `172.17.20.88`)
  - WSL에서 `hostname -I`로 확인

- **WSL → Windows 전송 시**
  - 목적지 = **Windows vEthernet(WSL) IP** (예: `172.17.16.1`)
  - Windows에서 WSL 가상 네트워크 IP로 고정/확인 필요

---

## 6. Pipeline (ver2) – 프로그램 파이프라인

### (1) Windows: `line_follow_sender_ver2.py`
- 카메라 스트림(CAP_FFMPEG) 수신
- 하단 ROI (`h*0.6`)에서 흰선 마스크 생성 (HSV threshold)
- `cv2.moments(mask)`로 중심점(cx) 계산
- `error_norm`, `found`, `m00`를 UDP(기본 10002)로 WSL 전송

### (2) WSL: `mvp_line_follow_controller_node_ver2.py`
- UDP 수신(10002)
- `found=True`이고 `m00`가 유효하면 PD 제어로 **/cmd_vel 생성**
- 유효하지 않으면 정지 (`/cmd_vel = 0`)

### (3) WSL: `wsl2_cmdvel_udp_sender.py`
- ROS2 `/cmd_vel` 구독
- Windows `9999`로 UDP 전송 (JSON)

### (4) Windows: `beagle_udp_bridge.py`
- UDP `9999` 수신
- `linear_x / angular_z`를 Beagle wheel 명령으로 변환
- 실제 비글 로봇 구동

---

## 7. 실행 순서 (Run Order)

권장 실행 순서: **Windows Bridge(모터)** → **WSL Controller(ROS2 제어)** → **WSL CmdVel Sender** → **Windows Vision Sender**  
이유: 최종 구동 경로(9999 수신 브리지)가 먼저 떠 있어야 /cmd_vel이 발생했을 때 유실 없이 모터로 전달됨.

### 7.0 사전 확인 (필수)

#### WSL IP 확인 (Windows → WSL 전송 목적지)
PowerShell에서:
```PowerShell
ipconfig

#### Windows IP 확인 (WSL → Windows 전송 목적지)
WSL에서:
```bash
hostname -I

### 7.1 Windows 터미널 1: 하드웨어 브리지 실행 (UDP 9999 수신 → Beagle 구동)
```PowerShell
python {프로그램_경로}\beagle_udp_bridge.py

**정상 체크**
- Listening UDP ... :9999 같은 수신 대기 로그가 떠야 함
- 이후 명령 수신 시 [RECV] linear_x=..., angular_z=... 로그가 찍혀야 함

### 7.2 WSL 터미널 1: ROS2 제어 노드 실행 (UDP 10002 수신 → /cmd_vel 퍼블리시)
```bash
source /opt/ros/humble/setup.bash
python3 mvp_line_follow_controller_node_ver2.py

**정상 체크**
- UDP 10002 수신 대기 로그가 떠야 함
- (Vision Sender 실행 후) /cmd_vel이 0이 아닌 값으로 변해야 함

### 7.3 WSL 터미널 2: /cmd_vel → Windows UDP 송신 노드 실행 (UDP 9999 전송)
```bash
source /opt/ros/humble/setup.bash
python3 wsl2_cmdvel_udp_sender.py --ros-args -p win_ip:=172.17.16.1 -p win_port:=9999
⚠️ 중요: win_ip는 반드시 Windows vEthernet(WSL) IP를 넣어야 함 (예: 172.17.16.1)

**정상 체크**
- /cmd_vel 구독 시작 로그
- /cmd_vel 발생 시 Windows Bridge에서 수신 로그가 찍혀야 함

### 7.4 Windows 터미널 2: 비전/인지 Sender 실행 (카메라 → 라인 추출 → UDP 10002 전송)
```PowerShell
python {프로그램_경로}\line_follow_sender_ver2.py --show \
  --source "http://192.168.66.1:9527/videostream.cgi?loginuse=admin&loginpas=admin" \
  --wsl-ip 172.17.20.88 --wsl-port 10002

**정상 체크**
- 표시 창에서 found=True가 자주 뜨고 m00가 충분히 크게 유지
- 라인이 잡히면 WSL 제어 노드가 /cmd_vel 생성 → Windows Bridge가 수신 → 모터 구동
