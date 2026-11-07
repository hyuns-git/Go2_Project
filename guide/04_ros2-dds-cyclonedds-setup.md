Go2 EDU의 Jetson에서 `ros2 topic list`를 쳤는데 로봇 토픽이 안 보이거나, 보이는데 `echo`가 "message type is invalid"로 실패하거나, 실행 중에 `ddsi_udp_conn_write ... failed` 로그가 계속 올라오는 경우가 있다. Orin NX에 Humble을 올리고 `unitree_ros2`를 붙이면서 이 세 가지를 차례로 만났다. 각각 어디를 봤고 무엇을 고쳤는지 적는다. 마지막에는 WSL2 노트북에서 같은 토픽을 받으려고 만든 설정도 붙였다.

## 환경

- 로봇: Unitree Go2 EDU. 내부망 `192.168.123.0/24`, Go2 본체가 `.161`
- Jetson Orin NX (Ubuntu 22.04, JetPack 6.2.2, ROS2 Humble). 내부망 인터페이스 `enP8p1s0`, 주소 `192.168.123.20`
- 같은 Orin NX에 USB 이더넷 어댑터가 하나 더 꽂혀 있다. LiDAR 센서용이고 IP는 고정하지 않았다(이름은 `enx<MAC>` 형태).
- Wi-Fi용 USB 어댑터(`wlx...`)도 있다.
- RMW는 `rmw_cyclonedds_cpp`. `unitree_ros2` 저장소의 `setup.sh`가 이걸 설정한다.

인터페이스가 세 개 이상 있는 보드라서 DDS가 어느 인터페이스를 쓰는지가 문제의 중심이 된다.

## setup.sh가 이전 보드 설정 그대로다

저장소를 받아서 `setup.sh`를 열어 봤다.

```bash
cat ~/unitree_ros2/setup.sh
```

```text
#!/bin/bash
echo "Setup unitree ros2 environment"
source /opt/ros/foxy/setup.bash
source $HOME/unitree_ros2/cyclonedds_ws/install/setup.bash
export RMW_IMPLEMENTATION=rmw_cyclonedds_cpp
export CYCLONEDDS_URI='<CycloneDDS><Domain><General><Interfaces>
    <NetworkInterface name="enp3s0" priority="default" multicast="default" />
    </Interfaces></General></Domain></CycloneDDS>'
```

고칠 곳이 두 군데다. 이 보드는 Humble인데 `foxy`를 source하고 있고, 인터페이스 이름 `enp3s0`는 이 보드에 없다. `CYCLONEDDS_URI`에는 파일 경로 대신 XML 문자열이 그대로 들어 있고, CycloneDDS는 여기에 적힌 인터페이스 하나만 쓴다. 이름이 틀리면 토픽이 아예 안 보인다.

```bash
ip -br addr    # 내부망(192.168.123.x)이 붙은 인터페이스 이름을 확인
sed -i 's/foxy/humble/' ~/unitree_ros2/setup.sh
sed -i 's/enp3s0/enP8p1s0/' ~/unitree_ros2/setup.sh
```

JetPack을 다시 올리면 인터페이스 이름이 바뀔 수 있다. 재플래시한 뒤에는 `grep NetworkInterface ~/unitree_ros2/setup.sh`로 이름이 맞는지부터 본다.

## colcon build가 rosidl_generator_dds_idl을 못 찾는다

`unitree_ros2`를 빌드했더니 `cyclonedds_ws`까지 한꺼번에 병렬로 빌드되면서 실패했다.

```text
--- stderr: unitree_api
CMake Error at CMakeLists.txt:26 (find_package):
  By not providing "Findrosidl_generator_dds_idl.cmake" in CMAKE_MODULE_PATH
  this project has asked CMake to find a package configuration file provided by
  "rosidl_generator_dds_idl", but CMake did not find one.
...
Summary: 0 packages finished [5.52s]
  1 package failed: unitree_api
  3 packages aborted: cyclonedds unitree_go unitree_hg
```

`unitree_api`가 `rosidl_generator_dds_idl`을 요구하는데 Humble 기본 설치에는 없다. apt로 세 패키지를 깔고 다시 빌드했다.

```bash
sudo apt update
sudo apt install ros-humble-rmw-cyclonedds-cpp ros-humble-rosidl-generator-dds-idl libyaml-cpp-dev
source /opt/ros/humble/setup.bash
cd ~/unitree_ros2
colcon build
```

이번에는 `Summary: 4 packages finished [7min 11s]`로 끝났다. 예제 패키지의 `backslash-newline at end of file` 경고가 길게 나오지만 빌드 결과와는 상관없다.

## topic list는 되는데 echo가 "message type is invalid"

빌드가 끝난 뒤 환경을 읽고 토픽을 확인했다.

```bash
source ~/unitree_ros2/setup.sh
ros2 topic list
timeout 3 ros2 topic echo /sportmodestate --no-arr | head -30
```

`ros2 topic list`에는 `/utlidar/robot_odom`, `/uwbstate`, `/wirelesscontroller` 같은 토픽이 줄줄이 나왔다. 그런데 echo는 이렇게 실패했다.

```text
The message type 'unitree_go/msg/SportModeState' is invalid
```

`topic list`는 DDS discovery에서 이름과 타입 문자열만 받아 오면 되지만, `echo`는 그 타입의 메시지 정의를 로컬에서 찾아야 한다. 토픽은 보이는데 echo만 실패하면 DDS 통신은 이미 되고 있고 메시지 패키지를 못 찾는 것이다.

원인은 `setup.sh`가 `cyclonedds_ws/install/setup.bash`를 source하는데, 실제로 빌드한 곳은 `~/unitree_ros2` 루트라서 `unitree_go`가 `~/unitree_ros2/install`에 설치된 것이었다. 경로를 고쳤다.

```bash
sed -i 's#cyclonedds_ws/install/setup.bash#install/setup.bash#' ~/unitree_ros2/setup.sh
source ~/unitree_ros2/setup.sh
timeout 3 ros2 topic echo /sportmodestate --no-arr | head -30
```

```text
stamp:
  sec: 1786492346
  nanosec: 899877491
error_code: 1001
...
mode: 0
progress: 0.0
gait_type: 0
```

`gait_type: 0`이면 정지 상태다. 파이프로 `head`를 닫으면 `BrokenPipeError`가 한 줄 뜨는데 무시해도 된다.

터미널마다 환경을 읽도록 `~/.bashrc`에 넣을 때는 순서가 중요하다. `CYCLONEDDS_URI`와 `RMW_IMPLEMENTATION`은 마지막에 source하는 `unitree_ros2/setup.sh`가 정하게 해야 다른 워크스페이스가 덮어쓰지 않는다. 한 가지 더, `ROS_DOMAIN_ID`가 비어 있는 보드가 있었다. 다른 장비와 맞춰야 하니 값을 명시해 둔다.

```bash
echo 'export ROS_DOMAIN_ID=0' >> ~/.bashrc
```

## 실행 중에 ddsi_udp_conn_write failed가 계속 뜬다

LiDAR 변환 노드를 띄웠더니 터미널에 이런 로그가 반복됐다.

```text
[INFO] [scan_maker_os1]: scan_maker_os1 started
1786594069.476434 [0] tev: ddsi_udp_conn_write to udp/239.255.0.1:7400 failed with retcode -1
1786594072.975772 [0] tev: ddsi_udp_conn_write to udp/192.168.123.20:34595 failed with retcode -1
```

`239.255.0.1:7400`은 DDS discovery가 쓰는 멀티캐스트 주소다. `192.168.123.20`은 이 보드 자신의 `enP8p1s0` 주소였고 로봇 쪽 장치가 아니었다. 먼저 이 경고가 실제 데이터 단절인지 갈랐다.

```bash
ros2 topic echo /utlidar/robot_odom --once
```

odom은 정상으로 들어왔다(`frame_id: odom`, `child_frame_id: base_link`). 경고가 떠도 로봇 토픽은 받고 있었다.

경로를 확인했다.

```bash
ip route get 192.168.123.20
ip route get 239.255.0.1
```

```text
local 192.168.123.20 dev lo src 192.168.123.20 uid 1000
multicast 239.255.0.1 dev enx<MAC> src 169.254.44.195 rt_offload_failed uid 1000
```

멀티캐스트 경로가 DDS가 지정한 `enP8p1s0`이 아닌 LiDAR용 USB 이더넷으로 잡혀 있다. 이 어댑터는 고정 IP가 없어서 `169.254.x.x` 링크로컬 주소를 쓴다. 같은 시각 `dmesg`에는 이 어댑터의 링크가 내려갔다 올라오는 기록이 있었다.

```text
ax88179 - Link status is: 0
ax88179 - Link status is: 1
r8168: enP8p1s0: link up
```

USB 어댑터 쪽 링크가 부팅 후 17분 사이에 몇 차례 내려갔다 올라왔고, 내부망 인터페이스는 `/sys/class/net/enP8p1s0/carrier`가 `1`로 안정적이었다.

## WSL2 노트북에서 Orin의 토픽이 안 보일 때

노트북의 WSL2에서 Orin으로 SSH는 되는데 `ros2 topic list`에 Orin 토픽이 나오지 않았다. 의심한 것은 두 가지다.

- WSL2의 기본 NAT 모드가 DDS discovery용 멀티캐스트를 넘겨 주지 않는다.
- Orin의 `CYCLONEDDS_URI`가 내부망 인터페이스 하나에만 묶여 있어서, SSH로 들어오는 Wi-Fi 경로는 DDS가 쓰지 않는다.

Windows 쪽 `%USERPROFILE%\.wslconfig`에서 네트워크 모드를 mirrored로 바꿨다.

```ini
[wsl2]
networkingMode=mirrored
dnsTunneling=true
firewall=true
autoProxy=true
```

`wsl --shutdown`으로 재시작하니 WSL의 `ip addr`에 Windows 호스트의 실제 인터페이스가 그대로 나왔다. 일반 LAN 대역 주소와, 로봇 내부망 대역(`192.168.123.x`) 주소가 둘 다 잡혔다.

Orin에는 벤더 파일을 건드리지 않으려고 별도 XML을 만들고, `~/.bashrc`에서 `setup.sh`를 source하는 줄 뒤에 환경 변수를 덮어썼다. 인터페이스 두 개를 등록하되 Wi-Fi 쪽은 멀티캐스트를 끄고, 상대 주소를 `Peer`로 직접 지정하는 구성이다.

```xml
<CycloneDDS>
  <Domain>
    <General>
      <Interfaces>
        <NetworkInterface name="eth0" />
        <NetworkInterface name="wlan0" multicast="false" />
      </Interfaces>
    </General>
    <Discovery>
      <Peers>
        <Peer address="<WSL 주소>" />
      </Peers>
    </Discovery>
  </Domain>
</CycloneDDS>
```

```bash
export CYCLONEDDS_URI=file:///home/<user>/cyclonedds_wsl.xml
```

WSL 쪽에도 같은 형태의 XML을 만들고, Peer에는 Orin의 Wi-Fi 주소를 넣었다. 여기 적은 인터페이스 이름은 이 구성을 만들 때 쓴 Orin Nano(Ubuntu 20.04) 기준이다.

## 같은 증상일 때 확인할 것

```bash
grep NetworkInterface ~/unitree_ros2/setup.sh   # 인터페이스 이름이 이 보드에 있는가
ip -br addr                                      # 내부망 주소가 붙은 인터페이스
echo $ROS_DOMAIN_ID                              # 비어 있지 않은가
ros2 topic list | grep -E "sportmode|utlidar"    # 토픽이 보이는가
ros2 topic echo /sportmodestate --once           # echo까지 되는가
ip route get 239.255.0.1                         # 멀티캐스트가 어느 NIC로 나가는가
```

토픽이 안 보이면 인터페이스 이름과 `ROS_DOMAIN_ID`, 보이는데 echo가 안 되면 메시지 패키지 source 경로, 둘 다 되는데 write 경고만 뜨면 데이터가 들어오는지부터 확인하면 된다.

## 확인하지 못한 것

- WSL2 구성은 설정과 mirrored 모드 적용까지만 확인했다. talker/listener로 실제 통신이 되는지 확인하기 전에 작업이 끝났다.
- `ddsi_udp_conn_write` 경고가 멀티캐스트 경로 때문인지는 확정하지 못했다. 경로가 LiDAR용 어댑터로 잡혀 있다는 것과, 경고가 떠도 데이터는 들어왔다는 것까지만 확인했다. 어댑터에 고정 IP를 주거나 멀티캐스트 경로를 고정해서 경고가 사라지는지는 해 보지 않았다.
- `apt`로 설치한 세 패키지 중 어느 것이 오류를 해결했는지는 따로 가르지 않았다.
