Jetson Orin Nano에 USB 무선랜 어댑터를 꽂아 쓰는데, 재부팅할 때마다 Wi-Fi가 붙지 않아서 HDMI 모니터를 연결해 손으로 다시 잡아야 했다. NetworkManager 설정, systemd 서비스, cron 스크립트로 이 문제를 줄인 과정을 적는다. 끊김을 기록하려고 만든 ping 로거에서 막힌 곳과, 만들어 놓고 한계가 드러난 watchdog도 같이 적는다.

## 환경

- 보드: Jetson Orin Nano (Go2 위에 올린 확장독 안)
- 무선랜: TP-Link T4U Plus (USB 어댑터)
- 네트워크 관리: NetworkManager, `nmcli`
- 접속 환경: 층마다 AP가 여러 대이고 SSID는 모두 같은 건물
- 같은 설정을 나중에 Orin NX(Ubuntu 22.04)에도 그대로 옮겼다.

아래 명령에서 `<프로필>`은 `nmcli connection show`로 보이는 Wi-Fi 연결 프로필 이름이다.

## 증상 1. 재부팅하면 Wi-Fi가 자동으로 붙지 않는다

원인으로 본 것은 두 가지다.

첫째, 이 보드에는 Wi-Fi 프로필이 여러 개 등록돼 있었다. 자동 연결 우선순위가 같으면 엉뚱한 곳에 붙을 수 있고, 재시도 횟수에 제한이 있으면 몇 번 실패한 뒤 포기한다. 둘째, USB 어댑터는 드라이버가 부팅 초반 NetworkManager의 첫 스캔이 끝난 뒤에 올라올 수 있다. 그러면 스캔 시점에는 무선 장치가 없어서 아무 데도 붙지 않는다.

프로필에 자동 연결, 최우선순위, 무제한 재시도를 지정한다.

```bash
nmcli connection modify <프로필> \
  connection.autoconnect yes \
  connection.autoconnect-priority 100 \
  connection.autoconnect-retries 0
```

적용됐는지 확인한다.

```bash
nmcli connection show <프로필> | grep -E "autoconnect|priority"
```

어댑터가 늦게 올라오는 경우를 위해 부팅 20초 뒤에 연결을 한 번 더 시도하는 서비스를 추가했다.

```bash
sudo tee /etc/systemd/system/wifi-reconnect.service > /dev/null << 'EOF'
[Unit]
Description=Reconnect Wi-Fi after boot
After=NetworkManager.service

[Service]
Type=oneshot
ExecStart=/bin/bash -c 'sleep 20 && nmcli connection up "<프로필>"'

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable wifi-reconnect.service
```

`enable`만 하고 재부팅하지 않았다면 로그가 없는 게 정상이다. 상태가 `disabled`로 나오면 `enable`이 안 된 것이다.

```bash
systemctl is-enabled wifi-reconnect.service
```

## 증상 2. 연결은 됐는데 절전 때문에 지연이 튈 수 있다

Wi-Fi 절전이 켜져 있으면 무선 칩이 주기적으로 잠든다. 그 사이에 명령 지연이나 패킷 손실이 생긴다. 로봇에 실시간으로 명령을 보내는 용도라서 절전을 끄기로 했다.

프로필 단위로 끄는 명령이다. 값 2가 끔이다.

```bash
nmcli connection modify <프로필> 802-11-wireless.powersave 2
```

프로필이 바뀌어도 유지되도록 NetworkManager 전역 설정에도 넣는다.

```bash
sudo tee /etc/NetworkManager/conf.d/wifi-powersave-off.conf > /dev/null << 'EOF'
[connection]
wifi.powersave = 2
EOF
```

적용 후에는 `iw dev <무선 인터페이스> get power_save`로 `off`가 나오는지 본다.

## 증상 3. AP가 여러 대인 건물에서는 로밍이 소극적이다

건물은 층마다 AP가 여러 대이고 SSID가 같다. 리눅스와 일반 USB 어댑터 조합은 로밍에 소극적이라서, 신호가 거의 끊길 때까지 기존 AP를 붙잡는 경우가 많다.

로밍 자체를 고치지는 못했다. 대신 MAC 주소 랜덤화를 껐다. 스캔할 때마다 MAC이 바뀌면 AP가 매번 새 기기로 보고 재인증이 늦어질 수 있기 때문이다.

```bash
sudo tee /etc/NetworkManager/conf.d/wifi-scan.conf > /dev/null << 'EOF'
[device]
wifi.scan-rand-mac-address=no
EOF

sudo systemctl restart NetworkManager
```

이 설정이 로밍을 실제로 좋아지게 했는지는 측정하지 못했다. 글 끝에 적는다.

## 증상 4. 끊기면 사람이 와서 다시 붙여야 한다

게이트웨이로 ping을 보내 두 번 모두 실패하면 연결을 내렸다 올리는 스크립트를 만들어 cron으로 1분마다 돌렸다.

```bash
sudo tee /usr/local/bin/wifi-watchdog.sh > /dev/null << 'EOF'
#!/bin/bash
GW=$(ip route | awk '/^default/ {print $3; exit}')
if ! ping -c 2 -W 3 "$GW" > /dev/null 2>&1; then
  nmcli connection down "<프로필>"
  sleep 2
  nmcli connection up "<프로필>"
fi
EOF

sudo chmod +x /usr/local/bin/wifi-watchdog.sh
echo '* * * * * root /usr/local/bin/wifi-watchdog.sh' | sudo tee /etc/cron.d/wifi-watchdog
```

이 스크립트에는 문제가 있다. ping이 실패한 이유가 "끊김"인지 "아직 연결하는 중"인지 구분하지 않는다. 그래서 연결을 맺고 있는 도중에도 ping이 실패하면 그 연결을 내려 버린다. 나중에 이 때문에 진행 중인 연결이 강제로 끊기는 일이 생겼다.

그대로 가져다 쓰지 말고, 쓰려면 연결 상태를 먼저 확인하는 조건을 직접 넣어야 한다. 지금 이 스크립트는 처음 적용했던 모습 그대로 적어 둔 것이다.

## 증상 5. 지연을 기록하려는 ping 로거가 안 뜬다

끊김이 언제 생기는지 보려고 게이트웨이에 ping을 계속 보내 결과를 파일에 쌓는 셸 스크립트를 만들고 `ping-logger.service`로 부팅 시 자동 시작되게 했다. 여기서 네 군데에서 막혔다.

`systemctl status`에 `status=203/EXEC`가 나왔다. 실행 파일이 없거나 실행 권한이 없을 때 나오는 오류다. 파일을 옮기고 다시 만드는 사이에 생긴 것으로 보였고, 파일이 제자리에 있고 `chmod +x`가 된 뒤에는 `active (running)`이 됐다.

부팅 직후에는 네트워크가 올라오기 전이라 ping이 실패하고 스크립트가 바로 끝났다. 유닛 파일에 `StartLimitIntervalSec`를 `[Service]` 섹션에 넣었더니 파싱 오류가 났다. 이 항목은 `[Unit]` 섹션에 둬야 한다. `Requires=wifi-reconnect.service`는 빼고 `After=`만 남겼다.

로그의 모든 줄이 `INFO`로 분류됐다. 한글 로케일에서는 ping 출력이 `시간=`으로 나와서 `time=`을 찾는 패턴에 걸리지 않았다. ping 앞에 `LC_ALL=C`를 붙여 로케일을 고정했다.

```bash
LC_ALL=C ping -c 1 -W 2 "$GW"
```

## 증상 6. 다른 보드로 옮기니 스크립트가 실패한다

같은 설정을 Orin NX에 옮길 때 세 가지가 걸렸다.

로그 경로가 이전 계정의 홈 디렉터리로 하드코딩돼 있어서 계정 이름이 다른 보드에서 `mkdir: 허가 거부`가 났다. 경로를 `$HOME` 기준으로 고쳤다.

`sed -i`로 한 줄을 고치다가 `LC_ALL=ping`처럼 줄이 깨졌다. 그 뒤로는 여러 줄 설정을 `sed`로 고치지 않고 위의 `tee ... << 'EOF'` 방식으로 파일을 통째로 쓴다.

인터페이스 이름도 달랐다. Orin Nano의 무선은 `wlan0`였는데, Orin NX에서는 MAC 기반 이름(`wlx<MAC>`)이 붙었다. 스크립트에서 인터페이스 이름을 직접 쓰고 있었다면 고쳐야 한다.

옮긴 뒤에는 항목별로 같은 값이 들어갔는지 대조했다.

```bash
nmcli connection show <프로필> | grep -E "autoconnect|priority|powersave"
cat /etc/NetworkManager/conf.d/wifi-*.conf
systemctl is-enabled wifi-reconnect.service
```

## 같은 증상이면 먼저 볼 것

- 재부팅 후 안 붙으면 `nmcli connection show <프로필> | grep autoconnect`로 자동 연결과 우선순위를 본다.
- 서비스가 안 뜨면 `systemctl status <유닛>`의 `status=` 값을 본다. `203/EXEC`는 파일 경로와 실행 권한이다.
- 로그가 이상하면 `LC_ALL=C`로 명령을 실행해서 출력 형식이 로케일 때문에 바뀐 건 아닌지 본다.
- 스크립트에서 계정 이름, 홈 경로, 인터페이스 이름이 하드코딩돼 있는지 `grep`으로 찾는다.
- 자동 복구 스크립트를 넣기 전에는 그 스크립트가 "연결 중" 상태를 어떻게 다루는지 확인한다.

## 확인하지 못한 것

- 층을 옮겨 다니며 로밍이 실제로 좋아졌는지 측정하지 않았다. MAC 랜덤화를 끈 효과도 측정한 기록이 없다.
- 절전을 끈 뒤 지연이 줄었는지 비교한 수치가 없다.
- 5GHz나 2.4GHz 밴드 고정, 공유기의 MAC 기반 고정 IP, AP의 802.11k/v/r(빠른 로밍) 지원 여부는 적용하지도 확인하지도 못했다. 로밍에는 이 중 마지막 항목이 가장 크게 작용할 것으로 본다.
- `ping-logger` 서비스를 재시작하면 원격 터미널 접속이 불안정해지는 것 같다는 관찰이 있었다. 원인은 모른다.
- 위의 서비스 파일과 스크립트는 실제로 쓴 파일 전문이 남아 있지 않아서, 적용했던 내용에 맞춰 다시 쓴 것이다.
