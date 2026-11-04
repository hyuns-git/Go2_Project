Orin NX를 JetPack 6.2.2(Ubuntu 22.04)로 다시 설치하고 USB 무선랜 드라이버를 올리는 데 하루가 통째로 들어갔다. 막힌 곳이 여러 번이었고, 그중 하나는 부팅 후 검은 화면까지 갔다. 어떤 명령이 어떤 이유로 필요한지, 어디서 틀렸는지를 순서대로 적는다.

환경은 Jetson Orin NX 16GB, AVerMedia NX215B 캐리어 보드, NVMe SSD 부팅, 플래시용 호스트로 Ubuntu 20.04 x86 PC다.

## 먼저 보드가 무엇인지 확인한다

처음에는 이 장비를 Orin Nano에 D115W 캐리어 보드로 알고 있었다. 실제로는 이렇게 확인됐다.

```bash
free -h                              # RAM 15Gi, Orin NX 16GB 쪽
cat /proc/device-tree/model          # NVIDIA Orin NX 16G, 캐리어 AVerMedia NX215
```

D115W는 확장독 안의 Orin Nano 쪽 보드였다. 사용자 매뉴얼도 NX215, NX215B, D115OXB 같은 계열을 한 문서로 묶어서 다룬다. 보드를 잘못 알면 매뉴얼의 복구 모드 위치부터 달라진다. NX215B에서 복구용 포트는 USB Type-C가 아니라 Micro-B USB 2.0이었다. D115W 매뉴얼 기준으로 Type-C일 수 있다는 안내를 받았다가 정정됐고, Micro USB 케이블이 없어서 한 번 멈췄다.

## 플래시 준비

플래시 호스트는 네이티브 x86 Ubuntu여야 한다. WSL2는 플래시 중 문제가 생길 위험이 있어서 쓰지 않았다. JetPack 6용 SDK Manager가 호스트 Ubuntu 20.04도 지원해서 여분 PC를 그대로 썼다.

AVerMedia가 배포하는 JetPack 6.2.2 올인원 패키지를 받아서 풀고 `Linux_for_Tegra`로 들어간다. 압축은 `sudo`로 풀어야 한다. 일반 권한으로 풀면 rootfs의 소유권이 어긋난다. 그다음 설정 스크립트를 돌린다.

```bash
sudo ./setup.sh
```

선택지는 이렇게 골랐다. 캐리어 보드는 `D115`, 모드는 `Normal Mode`, 카메라 프로파일은 `no camera`. `D115`가 NX215B에 해당한다는 것은 `nv-d115*.dtb` 안에 `nx215` 문자열이 들어 있는 것으로 확인했다. Super Mode는 Orin Nano 8G/4G용이라 Orin NX 16G에는 해당이 없다.

계정은 `sudo ./create_user.sh`로 만든다. `sudo`를 빼면 `Please use root to decompress this BSP.`가 나오고, 호스트가 x86이라 arm64 rootfs 안에서 명령을 돌리려면 `qemu-user-static`이 필요해서 `ERROR: please install qemu-user-static`도 한 번 만났다.

## 복구 모드로 들어가기

보드를 복구 모드로 넣고 Micro USB로 호스트에 연결한 다음, 호스트에서 인식 여부를 본다.

```bash
lsusb | grep -i nvidia     # NVIDIA 장치(ID 0955:7323)가 보여야 한다
```

이 줄이 안 나오면 `install.sh`를 돌리지 않는다. 수동 절차는 전원을 끄고 RECOVERY 버튼(SW4)을 누른 채 전원을 켠 뒤 2초쯤 후에 놓는 것이다. 버튼이 먹히지 않을 때는 부팅된 보드에서 소프트웨어로 들어갈 수 있다.

```bash
sudo reboot --force forced-recovery
```

이 명령은 재부팅이 아니다. 다음 부팅을 복구 모드로 하라는 표시를 남기고 전원을 끈다. 이 보드는 자동 전원 켜짐 스위치(SW2)가 기본 꺼짐이라 저절로 다시 켜지지 않고 Power 버튼(SW3)을 직접 눌러야 한다. 꺼진 뒤 안 켜져서 고장난 줄 알았지만 정상 동작이었다.

복구 모드가 잡히면 플래시한다.

```bash
sudo ./install.sh
```

대상 선택에서 `0) nvme0n1p1`을 골랐다. 보드 자체의 M.2 NVMe SSD다. 10~20분 걸리고, 그동안 케이블과 전원은 건드리지 않는다.

플래시 중에 한 번 사고가 있었다. 안내문에 `<원본_tar.gz_파일명>` 같은 자리표시자가 있었는데 그대로 실행하는 바람에 풀어 둔 BSP 폴더가 통째로 지워져서 압축을 다시 풀었다. 자리표시자는 실제 값으로 바꿔서 쳐야 한다. 호스트 PC의 호스트명이 플래시할 보드와 같은 이름으로 바뀐 것도 뒤늦게 발견했다. 같은 이름이 네트워크에 둘이면 헷갈리니 한쪽은 되돌린다.

## USB 무선랜 드라이버

설치가 끝나고 TP-Link T4U Plus(USB ID `2357:0115`, 칩셋 RTL88x2bu)를 올렸다. `rtl88x2bu` DKMS는 커널 `5.15.185-tegra`용으로 `installed`였는데 모듈이 로드되지 않았다. `modprobe`는 `cfg80211_*` 심볼이 `disagrees about version`, `Unknown symbol`이라고 했다. `vermagic`은 양쪽이 같아서 커널 버전 문제는 아니었다.

단서는 `lsmod`에 있었다. `cfg80211`이 `compat`이라는 모듈에 의존해 올라와 있었다. 커널에 내장된 cfg80211이 아니라 Intel Wi-Fi 지원용 백포트 버전이 `/lib/modules/5.15.185-tegra/updates/compat/`에 깔려 있었던 것이다. `updates/` 디렉토리는 기본 모듈 경로보다 우선한다. `88x2bu`는 내장 cfg80211 기준으로 빌드됐으니 심볼 체크섬이 맞지 않았다. 이 보드에는 M.2 Wi-Fi 카드가 없어서(`lspci`에 아무것도 안 나왔다) 백포트가 필요 없었다.

여기서 `updates/` 폴더 전체를 `.bak`으로 옮기고 재부팅했더니 부팅 로고 뒤로 검은 화면이 나왔다. SSH는 됐고 `journalctl`에 이 한 줄이 있었다.

```
Failed to insert module 'nvgpu': Invalid argument
```

`nvgpu`는 Tegra GPU 드라이버이고, 같은 `updates/` 안에 들어 있었다. 폴더를 원위치하고 `depmod -a` 후 재부팅해서 복구했다. 이후에는 백포트에 해당하는 세 곳만 옮겼다.

```bash
sudo mkdir -p ~/wifi_backport_disabled
sudo mv /lib/modules/5.15.185-tegra/updates/compat ~/wifi_backport_disabled/
sudo mv /lib/modules/5.15.185-tegra/updates/net/wireless ~/wifi_backport_disabled/
sudo mv /lib/modules/5.15.185-tegra/updates/net/mac80211 ~/wifi_backport_disabled/
sudo depmod -a
find /lib/modules/5.15.185-tegra/updates -maxdepth 2   # 재부팅 전에 확인
```

재부팅 전에 `find`로 `nvgpu.ko`, `opensrc-disp/`, `dkms/88x2bu.ko`가 남아 있는지 보고 갔다. 그 뒤로 드라이버가 정상적으로 올라왔다. 모듈 디렉토리는 통째로 옮기지 않는다. 옮기기 전에 안에 무엇이 있는지 보고, 재부팅 전에 확인한다.
