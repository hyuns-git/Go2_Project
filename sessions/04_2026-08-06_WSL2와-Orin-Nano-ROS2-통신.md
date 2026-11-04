---
no: 04
period: 2026-08-06 ~ 2026-08-07
category: [네트워크(DDS), 환경 세팅]
source: 대화 기록 12개 메시지 (원문 확인)
---

# 04. WSL2와 Orin Nano 사이 ROS2 통신 (미완료)

## 한 줄 요약
SSH는 되는데 ROS2 토픽이 안 보이는 문제의 원인을 둘로 나눠 진단하고 설정 파일까지 만들었다. 세션은 마지막 talker/listener 검증 전에 끝났다.

## 증상과 원인
- 증상: Windows의 WSL2에서 Orin Nano로 SSH는 되지만 `ros2 topic list`에 Orin의 토픽이 나오지 않음
- 원인 1: WSL2의 기본 NAT 네트워크 모드가 DDS discovery에 쓰는 멀티캐스트를 막음
- 원인 2: Orin의 `CYCLONEDDS_URI`가 파일이 아니라 인라인 XML 문자열로 설정되어 있고 eth0(192.168.123.18, 로봇 내부망)에만 묶여 있음. SSH로 들어오는 wlan0 경로는 DDS가 쓰지 않음

## 한 일
1. Orin에서 `ROS_DOMAIN_ID`가 비어 있어서 `export ROS_DOMAIN_ID=0`을 `~/.bashrc`에 추가하고 확인
2. Orin에 `~/cyclonedds_wsl.xml` 작성: `eth0`와 `wlan0`를 모두 등록하고, wlan0는 `multicast="false"`로 두고, 상대(WSL)의 주소를 `<Peer>`에 unicast로 지정. 벤더 파일을 건드리지 않으려고 `~/.bashrc`의 Unitree setup 줄 뒤에 `export CYCLONEDDS_URI=file:///home/unitree/cyclonedds_wsl.xml`을 덧붙여 덮어씀
3. WSL을 mirrored 네트워킹 모드로 전환: Windows `%USERPROFILE%\.wslconfig`에 `[wsl2]`, `networkingMode=mirrored`, `dnsTunneling=true`, `firewall=true`, `autoProxy=true` 입력 후 `wsl --shutdown`
4. mirrored 적용 확인: WSL에서 `ip addr`에 Windows 호스트의 실제 인터페이스가 보임. 일반 LAN 대역 주소(`eth3`)와 로봇 내부망 대역 주소(`eth5`, 192.168.123.100)가 둘 다 잡힘
5. Orin의 XML에서 Peer를 WSL 주소로 채우고(`sed -i`), WSL에도 대응하는 `~/cyclonedds_wsl.xml`(`eth3` 지정, Orin의 wlan0 주소를 Peer로)을 만들어 `CYCLONEDDS_URI`와 `ROS_DOMAIN_ID`를 `~/.bashrc`에 추가

## 상태와 남은 일
- 미완료: 최종 talker/listener 테스트 전에 세션이 끝남
- 백업 경로: WSL의 `eth5`가 로봇 내부망 대역이라, wlan0 경로가 안 되면 이쪽으로 직접 통신 가능성이 있음 (검증 안 됨)
- 사용자 요청 사항: 설명은 대충 하지 말고 명령어 하나하나 자세히. RViz와 Nav2 도구를 WSL에서 같이 쓸 수 있는지 문의했고 mirrored 모드를 기반으로 가능하다고 답함

## 메모
- 이 구성은 이후 세션에서 노트북, Jetson 간 DDS 통신 문제를 볼 때 출발점이 된다.
