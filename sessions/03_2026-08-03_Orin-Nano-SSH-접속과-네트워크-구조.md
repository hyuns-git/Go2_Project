---
no: 03
period: 2026-08-03 ~ 2026-08-05
category: [네트워크, 환경 세팅]
source: 대화 기록 80개 메시지 (원문 확인)
---

# 03. Orin Nano SSH 접속 문제와 네트워크 구조 정리

## 한 줄 요약
확장독의 Orin Nano에 노트북에서 SSH로 접속하면서 인터넷도 같이 쓰려는 과정이다. 시행착오가 길었고, 결국 로봇용 AP를 거치지 않고 Orin Nano의 Wi-Fi(wlan0)가 일반 공유기에서 받은 주소로 붙어서 성공했다. 접속 후 `.bashrc` 프롬프트 제거와 `unitree_ros2` 예제 빌드까지 했다.

## 네트워크 구조 (이 세션에서 확정된 것)
| 장치 | 인터페이스 | 주소 | 역할 |
|---|---|---|---|
| Go2 본체 | 내부 | 192.168.123.161 | 로봇 내부 통신 |
| Orin Nano (확장독) | eth0 (유선) | 192.168.123.18 | 로봇 내부망, ROS2/CycloneDDS 통신 |
| Orin Nano | wlan0 | 일반 LAN 대역(192.168.1.x), DHCP | SSH 접속과 인터넷 |
| 개발 PC가 로봇에 유선 직결할 때 | 이더넷 | 192.168.123.0/24 안의 빈 주소 | Unitree 문서는 `.99`를 안내, 관례적으로 `.99` 또는 `.222` |

## 시행착오 순서와 원인
1. MobaXterm에서 접속 안 됨: 포트가 24로 잘못 입력돼 있었음 (SSH 기본은 22). 
2. Claude가 `192.168.123.222`를 로봇 쪽 주소로 오해했다가, 사용자 PC 자신의 주소임을 정정. 이 과정에서 Claude가 "eth0의 .18을 바꾸라"고 잘못 안내했고, 사용자가 `.18`은 의도한 값이며 ROS2 통신이 이미 잘 되는 상태라고 바로잡음. 이후 `.18`은 건드리지 않는 것으로 확정.
3. 노트북 Wi-Fi 어댑터 하나에 서로 다른 두 대역(DHCP로 받은 일반 LAN + 수동 192.168.123.x)을 얹으면 그때마다 인터넷이 끊김. 노트북 내장 Wi-Fi 7 어댑터(Intel BE201)의 드라이버에서 반복됨. 이 방식은 포기.
4. 어댑터를 둘로 나누는 방식으로 전환. 내장 Wi-Fi는 인터넷용, USB Wi-Fi 어댑터(Realtek 8912AU 칩셋, 윈도우에서는 별도 "Wi-Fi 9"로 보임)는 로봇용으로 쓰려고 함.
5. 로봇용 AP에 연결해도 `ping`이 안 감. 원인은 해당 AP(별도 공유기)의 AP 격리(client isolation)로 무선 클라이언트끼리 통신이 막혀 있었던 것. 노트북 쪽 IP 설정으로는 해결할 수 없는 문제였다.
6. 최종 성공: 같은 일반 공유기 대역에서 Orin Nano의 wlan0 DHCP 주소(192.168.1.x)로 SSH 접속. 로봇 내부 통신은 계속 eth0(`.18`)로.

정리하면 교훈이 둘이다. 하나는 한 Wi-Fi 어댑터에 두 대역을 얹지 않는다는 것, 다른 하나는 ping이 안 갈 때는 IP 설정보다 먼저 AP 격리를 의심한다는 것이다.

## 접속 후 한 일
- `.bashrc`의 `ros:foxy(1) noetic(2) ?` 프롬프트 제거: 원인은 FishROS 설치 도구가 `# >>> fishros initialize >>>` 블록을 자동 생성한 것이었다. `echo/read/case` 부분을 지우고 foxy 쪽 줄만 조건 없이 실행되게 고쳤다. 고친 내용은 `source /opt/ros/foxy/setup.bash`, `source ~/cyclonedds_ws/install/setup.bash`, `RMW_IMPLEMENTATION=rmw_cyclonedds_cpp`, `CYCLONEDDS_URI`, `LD_LIBRARY_PATH` export. 수정 전에 `cp ~/.bashrc ~/.bashrc.bak`으로 백업하고, 건드리는 범위가 이 블록뿐임을 먼저 설명했다.
- `unitree_ros2` 예제 빌드: `colcon build`는 `~/unitree_ros2/example/`에서 해야 한다. `example/src/src/go2` 안에서 실행하면 `0 packages finished`. 성공하면 `Finished <<< unitree_ros2_example [1min 22s]`.
- 빌드된 실행 파일은 표준 위치(`install/<pkg>/lib/<pkg>/`)가 아니라 `install/unitree_ros2_example/bin/`에 설치되어서, `ros2 run`, `ros2 pkg executables`로 안 보인다. 직접 경로로 실행한다.
- `go2_stand_example`은 일어서기 하나만, `go2_sport_client`는 숫자 인자로 동작을 고른다. Unitree GitHub README는 `go2_sport_client 3`(앉기), `4`(일어서기)로 안내한다.
- SLAM이 만든 `.pgm`(흑백 이미지) 지도 파일을 보는 방법을 문의했고, `.pgm + .yaml + .posegraph + .data` 조합은 slam_toolbox 결과물이다(대화에서는 Cartographer 계열이라고 했으나 7월 기록상 slam_toolbox가 맞다).

## 확인이 필요한 항목
- `go2_sport_client`의 숫자와 동작의 대응(3=앉기, 4=일어서기)은 README 안내 기준이다. 대화에서 sport mode 번호표(0=idle 등)는 Claude의 기억 기반 설명이었고 검증되지 않았다. 소스의 `go2_sport_client.cpp`나 공식 문서로 확인해야 한다.
- 사용자 PC 유선 주소: Unitree 문서 안내는 `.99`이고, 사용자 보고서(02번 세션)에는 `eth0`와 `192.168.123.222`로 적혀 있다. 이 세션 첫 메시지에서 사용자는 "SSH 접속용으로 222를 와이파이 주소로 설정했다"고도 했다. 사용자도 어느 쪽인지 기억하지 못해서 실제 값은 미확정이다. 글에는 "문서 안내는 `.99`, 같은 대역의 빈 주소면 된다"까지만 쓴다.
