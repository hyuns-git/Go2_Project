# velog 발행 계획과 상태

글은 `STYLE.md`의 기준을 지켜서 쓴다. 글 원고는 이 폴더의 `NN_*.md`이고, 본문 첫 줄부터 그대로 velog 본문이 됩니다. 제목과 태그는 아래 표에 있습니다. 하루에 한 편씩, 원고가 있고 아직 발행되지 않은 가장 앞 번호를 올립니다.

| 번호 | 원고 | 제목 | 태그 | 상태 |
|---|---|---|---|---|
| 01 | 01_go2-edu-internal-network.md | Go2 EDU에 Jetson 두 대를 붙일 때: 내부 네트워크 구조와 접속 방법 | Go2, ROS2, Jetson, 네트워크, 로보틱스 | 발행 2026-11-04 https://velog.io/@hhyun0418/go2-edu-internal-network (v2 재작성본 반영 완료, 제목: Go2 EDU에 Jetson 두 대 붙이기: 내부망 구조, SSH 접속, 인터넷이 끊길 때) |
| 02 | 02_jetson-flash-and-driver.md | Orin NX에 JetPack 6.2.2 올리고 USB 무선랜 붙이기: 복구 모드, cfg80211 충돌, 검은 화면 | Jetson, JetPack, Ubuntu, 리눅스, 로보틱스 | 발행 2026-11-05 https://velog.io/@hhyun0418/jetson-flash-and-driver |
| 03 | 03_wifi-stabilization-watchdog.md (원고 A), 03_wifi-stability.md (원고 B) | Jetson USB Wi-Fi가 끊길 때: NetworkManager 설정과 watchdog 오작동 잡기 (A) / Jetson Orin에서 USB 무선랜이 재부팅 후 안 붙을 때: 자동 연결, 절전 끄기, 재연결 watchdog (B) | Jetson, 무선랜, NetworkManager, 리눅스, 로보틱스 | 발행 2026-11-06 https://velog.io/@hhyun0418/jetson-wifi-stability (원고 B가 올라감. 같은 날 원고 A가 따로 추가돼 있어서 A는 미발행. 어느 쪽을 03번으로 둘지 사용자 결정 필요) |
| 04 | (작성 예정) | ROS2 DDS 통신 설정: CycloneDDS, ROS_DOMAIN_ID, WSL2 | | 원고 없음 |
| 05 | (작성 예정) | LiDAR 점군에서 2D scan으로 | | 원고 없음 |
| 06 | (작성 예정) | slam_toolbox로 지도 만들기 | | 원고 없음 |
| 07 | (작성 예정) | AMCL과 Nav2로 Point-to-Point 이동 | | 원고 없음 |
| 08 | (작성 예정) | 계단 주행 | | 원고 없음 |

## 발행 방법 (Claude in Chrome, velog 로그인된 상태)

1. 새 탭에서 `https://velog.io/write`를 연다.
2. `javascript_tool`로 제목 textarea(`textarea[placeholder*="제목"]`)에 React 방식으로 값을 넣는다. `HTMLTextAreaElement.prototype`의 `value` setter로 값을 쓰고 `input` 이벤트를 보낸다.
3. 본문은 `document.querySelector('.CodeMirror').CodeMirror.setValue(원고 전체)`로 넣는다. 원고는 JSON 문자열로 바꿔서 넣고, 넣은 뒤 글자 수가 원고와 같은지 확인한다.
4. 태그 입력칸을 클릭하고 태그마다 입력 후 Enter를 친다.
5. 하단의 `출간하기` 버튼을 JS로 `click()`해서 설정 창을 연다(좌표 클릭은 창이 안 열릴 때가 있었음).
6. URL 입력칸을 영문 slug로 바꾼다(`input` 요소의 `value`는 `HTMLInputElement.prototype` setter로). 공개 설정이 `전체 공개`인지 확인하고, 설정 창 안의 마지막 `출간하기` 버튼을 클릭한다.
7. 발행 후 주소가 `https://velog.io/@hhyun0418/<slug>`로 바뀌는지, 제목과 소제목, 표, 코드 블록이 그대로 렌더링되는지 확인한다.
8. 이 표의 상태 칸에 발행 날짜와 주소를 적고 커밋, 푸시한다.

## 규칙

- 원고를 발행 직전에 고치지 않는다. 틀린 내용이 보이면 발행하지 말고 상태 칸에 이유를 적는다.
- Chrome 연결이 안 되거나 로그인이 풀려 있으면 발행하지 않고 멈춘다. 비밀번호를 입력하거나 계정을 만들지 않는다.
- 하루에 한 편만 올린다. 원고가 없으면 그날은 올리지 않는다.
- 회사명, 사람 이름, 접속 주소, 계정, 시리얼 번호는 글에 넣지 않는다.
