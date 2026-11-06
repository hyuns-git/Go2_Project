Go2 확장독 안의 Jetson Orin Nano에 USB 무선랜을 꽂아 쓰는데 SSH가 수시로 끊겼다. 재연결 설정을 넣고 복구용 watchdog 스크립트까지 만들어 뒀는데, 로그를 보니 연결을 끊고 있던 것이 그 watchdog이었다. 설정 내용과, 로그로 범인을 찾은 과정을 적는다.

## 환경

- 대상: Jetson Orin Nano (Ubuntu 20.04, JetPack 5), Go2 확장독에 장착
- 무선랜 어댑터: TP-Link Archer T4U Plus (USB ID `2357:0115`), 외부 DKMS 드라이버 `88x2bu`
- 네트워크: NetworkManager. 층마다 AP가 여러 대고 SSID는 모두 같다.
- 연결 구조: 로봇 내부망은 유선(`eth0`), 인터넷과 SSH 접속은 무선(`wlan0`)

## 재부팅하면 Wi-Fi가 안 붙는다

USB 어댑터라서 부팅 초반 NetworkManager가 스캔을 끝낸 뒤에야 드라이버가 올라올 수 있다. 다른 Wi-Fi 프로필이 여러 개 저장돼 있으면 엉뚱한 곳에 붙기도 한다. 프로필의 자동 연결 우선순위를 올리고 재시도 횟수 제한을 풀었다.

```bash
nmcli connection modify <프로필> \
  connection.autoconnect yes \
  connection.autoconnect-priority 100 \
  connection.autoconnect-retries 0
```

그래도 어댑터가 늦게 잡히는 경우를 위해 부팅 20초 뒤에 강제로 연결하는 서비스를 하나 만들었다. `/etc/systemd/system/wifi-reconnect.service`는 아래 정도면 된다.

```ini
[Unit]
Description=Wi-Fi reconnect after boot
After=NetworkManager.service

[Service]
Type=oneshot
ExecStart=/bin/bash -c 'sleep 20 && nmcli connection up <프로필>'

[Install]
WantedBy=multi-user.target
```

`systemctl enable wifi-reconnect.service`만 하고 재부팅하지 않았다면 로그가 없는 것이 정상이다. 상태가 `disabled`로 나오면 `enable`이 안 된 것이다.

## 명령이 늦게 가거나 패킷이 샌다

로봇에 실시간으로 명령을 보내는 경로라서 무선 칩이 주기적으로 잠드는 절전 모드는 꺼야 한다. 프로필 단위와 전역 설정, 두 군데에 넣었다.

```bash
nmcli connection modify <프로필> 802-11-wireless.powersave 2
```

```ini
# /etc/NetworkManager/conf.d/wifi-powersave-off.conf
[connection]
wifi.powersave = 2
```

AP가 여러 대인 환경에서는 스캔 때 MAC 주소가 바뀌면 AP가 매번 새 기기로 보고 재인증이 늦어질 수 있다. 스캔용 MAC 랜덤화도 껐다.

```ini
# /etc/NetworkManager/conf.d/wifi-scan.conf
[device]
wifi.scan-rand-mac-address=no
```

## 끊기면 알아서 복구하게 만든 watchdog

사람이 없을 때도 복구되게 하려고 게이트웨이로 ping을 보내 보고, 실패하면 연결을 내렸다 올리는 스크립트를 만들었다. cron으로 1분마다 돌렸다.

```bash
#!/bin/bash
GATEWAY=$(ip route | grep default | awk '{print $3}')
if ! ping -c 2 -W 3 "$GATEWAY" > /dev/null 2>&1; then
    logger "wifi-watchdog: connection lost, reconnecting"
    nmcli connection down <프로필>
    sleep 2
    nmcli connection up <프로필>
fi
```

```
* * * * * /usr/local/bin/wifi-watchdog.sh
```

이 스크립트는 8월 5일에 넣었고, 이후에 드라이버를 통째로 내렸다 올리는 두 번째 watchdog을 systemd 타이머(30초 간격)로 추가했다. cron 쪽이 그때까지 계속 돌고 있다는 사실은 8월 20일에야 알았다.

## 증상. SSH가 끊기고 dmesg에 드라이버 등록과 해제가 반복된다

8월 20일, 계단 테스트 중에 Orin Nano로 들어가는 SSH가 계속 끊겼다. `dmesg`에는 드라이버가 내려갔다 올라오는 줄이 반복됐다.

```
usbcore: deregistering interface driver rtl88x2bu
usbcore: registered new interface driver rtl88x2bu
```

간격은 30~60초였고, 어댑터를 물리적으로 뽑았다 꽂은 `USB disconnect`는 캡처 구간 전체에서 한 번뿐이었다. 누군가 드라이버를 일부러 내렸다 올리고 있다는 뜻이다.

NetworkManager 로그를 시각 순으로 봤다. 시각과 메시지만 남기고 줄였다.

```
09:33:03 device (wlan0): Activation: starting connection '<SSID>'
09:33:03 device (wlan0): state change: prepare -> config
09:33:07 device (wlan0): state change: config -> unmanaged (reason 'removed', sys-iface-state: 'removed')
09:33:07 radio killswitch ... disappeared
09:33:13 device (wlan0): driver supports Access Point (AP) mode
```

연결을 시작하고 4초 만에 `wlan0` 장치 자체가 사라졌다가 6초 뒤에 새 장치로 나타났다. 인증 실패나 DHCP 실패 로그는 없다. 연결하는 도중에 누군가 장치를 지운 것이다.

## 원인 1. 연결 중에 드라이버를 재로드하는 watchdog

두 번째 watchdog은 `wlan0`가 응답하지 않으면 `modprobe -r 88x2bu`로 드라이버를 재로드한다. 문제는 연결 시도가 진행되는 중에도 응답이 없다고 판단한다는 점이다. 재로드가 연결 시도를 끊고, 끊긴 뒤 다시 연결을 시도하면 또 재로드가 들어온다.

`nmcli`의 연결 상태 코드를 먼저 확인했다. 숫자는 NetworkManager의 장치 상태 값이다.

| 코드 | 상태 |
|---|---|
| 30 | disconnected |
| 40 | prepare |
| 50 | config |
| 60 | need-auth |
| 70 | ip-config |
| 80 | ip-check |
| 90 | secondaries |
| 100 | connected |

40에서 90까지는 연결 진행 중이다. 이 범위면 이번 주기를 건너뛰게 고쳤다. 출력 형식을 짐작으로 처리하지 않으려고 파싱 결과를 먼저 찍어 봤다.

```bash
nmcli -t -f GENERAL.STATE dev show wlan0 | cut -d: -f2 | grep -oE '[0-9]+' | head -1
# 100
```

```bash
STATE=$(nmcli -t -f GENERAL.STATE dev show wlan0 2>/dev/null | cut -d: -f2 | grep -oE '[0-9]+' | head -1)
if [ -n "$STATE" ] && [ "$STATE" -ge 40 ] && [ "$STATE" -le 90 ]; then
    logger -t wifi_watchdog "wlan0 연결 진행 중(상태 $STATE), 이번 사이클 건너뜀"
    exit 0
fi
GATEWAY=$(ip route | grep default | awk '{print $3}' | head -1)
if [ -z "$GATEWAY" ] || ! ping -c 2 -W 2 -I wlan0 "$GATEWAY" > /dev/null 2>&1; then
    logger -t wifi_watchdog "wlan0 응답 없음(상태 $STATE), 88x2bu 재로드 시도"
    modprobe -r 88x2bu
    # 이하 재로드 동작은 기존과 같다
fi
```

배포하고 로그를 보니 건너뛰기는 의도대로 동작했다.

```
10:33:15 wifi_watchdog: wlan0 연결 진행 중(상태 50), 이번 사이클 건너뜀
10:33:56 wifi_watchdog: wlan0 연결 진행 중(상태 70), 이번 사이클 건너뜀
```

그런데 끊김은 계속됐다. NetworkManager 로그에는 이번에 `reason 'user-requested'`가 찍혔다. 누군가 `nmcli`로 연결을 내렸다는 뜻이다.

## 원인 2. 같은 일을 하는 cron이 따로 돌고 있었다

고친 건 `wifi_watchdog.sh`(언더스코어)였다. 타이머와 cron을 모두 찾아봤다.

```bash
systemctl list-timers --all | grep -i wifi
crontab -l
```

```
wifi-watchdog.timer  wifi-watchdog.service
* * * * * /usr/local/bin/wifi-watchdog.sh
```

cron이 부르는 건 `wifi-watchdog.sh`(하이픈)였다. 처음에 만든 스크립트가 그대로 매분 돌고 있었다. 연결 상태를 확인하지 않고, ping이 한 번 실패하면 `nmcli connection down`부터 하는 구조라서 연결 중이든 아니든 끊어 버린다. 파일 날짜는 8월 5일이었다.

cron 항목을 지우고 스크립트 이름을 바꿔서 실행되지 않게 했다.

```bash
crontab -l | grep -v 'wifi-watchdog.sh' | crontab -
sudo mv /usr/local/bin/wifi-watchdog.sh /usr/local/bin/wifi-watchdog.sh.disabled
```

하이픈과 언더스코어만 다른 파일이 두 개 있었고, 나는 언더스코어 쪽만 고치고 있었다. 같은 일을 하는 자동 복구가 있으면 `systemctl list-timers`와 cron을 둘 다 봐야 한다.

## 그 뒤에도 남은 끊김

cron을 지운 뒤 2분 동안 NetworkManager에 연결 이벤트가 없어서 조용해졌다고 봤지만, 끊김은 한 번 더 났다. 이번에는 상태 100(연결됨)에서 watchdog이 재로드를 실행한 기록이 있었다. 게이트웨이로 보낸 ping 두 번이 모두 실패하면 이미 연결된 상태여도 드라이버를 내리기 때문이다.

dmesg에는 이 드라이버가 연결을 맺는 함수에서 커널 WARN을 내는 기록도 있었다.

```
pc : rtw_join_done_chk_ch+0xf8/0x3f0 [88x2bu]
Call trace:
 rtw_join_done_chk_ch+0xf8/0x3f0 [88x2bu]
 mlmeext_joinbss_event_callback+0x164/0x2f0 [88x2bu]
```

watchdog이 만든 끊김과는 별개로 드라이버 자체에도 문제가 있었다. 이 드라이버 문제는 고치지 못했고, 그날은 SSH 대신 `mosh`를 설치해서 세션이 끊겨도 이어지게 하고 테스트로 돌아갔다.

```bash
sudo apt install mosh
mosh <사용자>@<Orin Nano 주소>
```

## 같은 패턴이 Orin NX에서도 나왔다

9월 14일에 Orin NX에서도 같은 증상을 만났다. 이쪽은 Intel AX210 카드에 같은 watchdog을 복사해 둔 상태였다. Wi-Fi 메뉴가 생겼다 사라지길 반복했고, `dmesg`에는 `iwlwifi` 모듈이 30초 간격으로 다시 로드되는 줄이 이어졌다.

```
Loading modules backported from iwlwifi
```

watchdog 로그에 답이 있었다.

```
wlP1p1s0 응답 없음(상태 30), 드라이버 재로드 시도
```

상태 30은 AP에 붙어 있지 않은 정상 상태다. 스크립트가 이것을 고장으로 읽고 30초마다 드라이버를 재로드했다. 유닛 이름은 `wifi-watchdog.timer`(하이픈)인데 `wifi_watchdog.timer`로 멈추려다 `Unit ... not loaded`를 받은 것도 이때다.

```bash
sudo systemctl stop wifi-watchdog.timer
```

## 비슷한 증상일 때 볼 명령

```bash
# 누가 장치를 지우고 있나
journalctl -u NetworkManager --since "10 min ago" --no-pager | grep -iE "deactivat|activat|reason"
sudo dmesg -T | grep -iE "88x2bu|deregister|registered new interface"

# 복구 스크립트가 어디서 도는지 (타이머와 cron 모두)
systemctl list-timers --all | grep -i wifi
crontab -l; sudo crontab -l; ls /etc/cron.d
journalctl -t wifi_watchdog --since "10 min ago" --no-pager

# 지금 장치 상태 코드
nmcli -t -f GENERAL.STATE dev show wlan0
```

`reason 'removed'`는 장치 자체가 사라졌다는 뜻이고, `reason 'user-requested'`는 누군가 연결을 내렸다는 뜻이다. 둘 다 시각을 복구 스크립트 로그와 맞춰 보면 범인이 나온다. 복구 스크립트는 "연결 중"(40~90)과 "끊김"(30)을 구분해야 하고, ping이 한 번 실패했다고 바로 재로드하면 안 된다.

## 확인하지 못한 것

- 8월 20일 이후 Orin Nano의 끊김이 완전히 사라졌는지는 기록이 없다. `rtw_join_done_chk_ch` 커널 WARN은 고치지 못했다.
- 처음 만든 스크립트가 `ping -I wlan0` 없이 첫 번째 기본 경로의 게이트웨이로 ping을 보냈다. Orin Nano에는 유선(`eth0`)도 연결돼 있어서 이 게이트웨이가 유선 쪽이었을 가능성이 있지만 확인하지 않았다.
- 절전 끄기와 MAC 랜덤화 끄기가 끊김을 줄였는지 측정한 기록은 없다.
- Orin NX에서 타이머를 멈춘 뒤 Wi-Fi 메뉴 깜빡임이 멈췄는지 따로 기록하지 않았다.
- `wifi-reconnect.service` 파일 전문은 보관본이 없어서 위 내용은 동작 설명에 맞춰 최소 형태로 다시 적은 것이다.
