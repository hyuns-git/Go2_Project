Jetson Orin NX를 JetPack 6.2.2(Ubuntu 22.04)로 다시 설치하고 USB 무선랜 어댑터를 붙이는 데 하루가 걸렸다. 보드 종류를 잘못 알고 시작했고, 복구 모드가 안 들어가서 한 번 멈췄고, 드라이버를 고치다가 부팅 후 검은 화면까지 갔다. 각 증상에서 무엇을 봤고 어떻게 풀었는지 적는다.

## 환경

- 대상: Jetson Orin NX 16GB, AVerMedia NX215B 캐리어 보드, NVMe SSD 부팅
- 플래시용 호스트: Ubuntu 20.04 x86 PC (WSL2는 플래시 위험 때문에 쓰지 않았다)
- 소프트웨어: AVerMedia JetPack 6.2.2 올인원 패키지, SDK Manager 2.4.1
- 무선랜 어댑터: TP-Link T4U Plus (USB ID `2357:0115`, RTL88x2bu)

## 증상 1. 매뉴얼의 복구 방법이 내 보드와 안 맞는다

이 장비를 Orin Nano에 D115W 보드로 알고 있었다. 확인해 보니 달랐다.

```bash
free -h                         # RAM 15Gi: Orin NX 16GB 쪽
cat /proc/device-tree/model     # NVIDIA Orin NX 16G, AVerMedia NX215
```

D115W는 확장독 안쪽 Orin Nano의 캐리어 보드였다. 사용자 매뉴얼은 NX215, NX215B, D115OXB 같은 계열을 한 문서로 묶어서 다룬다. 보드를 잘못 알면 매뉴얼의 복구 포트 안내부터 달라진다. NX215B의 복구용 포트는 USB Type-C가 아니라 Micro-B USB 2.0이었다.

## 증상 2. 플래시 준비 스크립트가 오류를 낸다

AVerMedia 패키지를 받아 `sudo`로 풀고 `Linux_for_Tegra`에서 설정 스크립트를 돌렸다. 일반 권한으로 풀면 rootfs 소유권이 어긋난다.

```bash
sudo ./setup.sh
```

선택지는 캐리어 보드 `D115`, 모드 `Normal Mode`, 카메라 `no camera`였다. `D115`가 NX215B에 해당한다는 건 `nv-d115*.dtb` 안에 `nx215` 문자열이 들어 있는 것으로 확인했다. Super Mode는 Orin Nano 8G/4G 전용이라 Orin NX 16G는 대상이 아니다.

계정 생성 단계에서 오류를 두 번 만났다.

```
Please use root to decompress this BSP.
ERROR: please install qemu-user-static
```

첫 번째는 `sudo ./create_user.sh`로 `sudo`를 붙이면 된다. 두 번째는 호스트가 x86인데 arm64 rootfs 안에서 명령을 실행해야 해서 필요한 `qemu-user-static` 패키지를 설치하면 사라졌다.

## 증상 3. 복구 모드에 안 들어간다

플래시 전에 호스트에서 보드가 복구 모드로 보이는지 확인한다.

```bash
lsusb | grep -i nvidia     # ID 0955:7323 이 보여야 한다
```

이 줄이 안 나오면 `install.sh`를 돌리지 않는다. 수동 절차는 전원을 끄고 RECOVERY 버튼(SW4)을 누른 채 전원을 켜서 2초쯤 뒤에 놓는 것이다. 나는 버튼을 눌러도 그냥 부팅됐다.

부팅된 보드에서 소프트웨어로 복구 모드에 들어갈 수 있다.

```bash
sudo reboot --force forced-recovery
```

이 명령은 재부팅이 아니다. 다음 부팅을 복구 모드로 하겠다는 표시를 남기고 전원을 끈다. 이 보드는 자동 전원 켜짐 스위치(SW2)가 기본으로 꺼져 있어서 저절로 다시 켜지지 않는다. 꺼진 보드가 안 켜져서 고장난 줄 알았는데, Power 버튼(SW3)을 한 번 누르니 `lsusb`에 NVIDIA 장치가 나타났다.

복구 모드가 잡히면 플래시한다.

```bash
sudo ./install.sh
```

대상 선택에서 `0) nvme0n1p1`을 골랐다. 보드에 달린 M.2 NVMe SSD이고, 10~20분 걸린다. 진행 중에는 케이블과 전원을 건드리지 않는다.

플래시 준비 중에 두 가지 실수도 있었다. 안내문의 `<원본_tar.gz_파일명>`은 자리표시자인데 그대로 실행하는 바람에 풀어 둔 BSP 폴더가 통째로 지워졌고 압축을 다시 풀었다. 호스트 PC의 호스트명이 보드와 같은 `nvidia-desktop`으로 바뀐 것도 뒤늦게 알았다. 같은 이름이 네트워크에 둘이 되니 한쪽은 되돌려야 한다.

## 증상 4. 무선랜 드라이버가 올라오지 않는다

설치가 끝나고 `rtl88x2bu` DKMS를 커널 `5.15.185-tegra`용으로 빌드했다. `dkms status`는 `installed`인데 모듈이 로드되지 않았다. `modprobe`는 `cfg80211_*` 심볼에 대해 `Unknown symbol`, `disagrees about version`이라고 했다. `vermagic`은 양쪽이 같아서 커널 버전 문제는 아니었다.

단서는 `lsmod`에 있었다.

```bash
lsmod | grep -E "cfg80211|compat"
# cfg80211   1044480  0
# compat       16384  1 cfg80211
```

`cfg80211`이 `compat`이라는 모듈에 의존해서 올라와 있었다. 커널에 내장된 cfg80211이 아니라 Intel Wi-Fi 지원용 백포트 버전이 `/lib/modules/5.15.185-tegra/updates/compat/`에 깔려 있었던 것이다. `updates/` 디렉토리는 기본 모듈 경로보다 우선한다. `88x2bu`는 내장 cfg80211 기준으로 빌드됐으니 심볼 체크섬이 맞지 않았다.

이 보드에는 M.2 Wi-Fi 카드가 없다(`lspci`에 아무것도 안 나왔다). 백포트는 필요 없는 모듈이었다.

## 증상 5. 폴더를 옮겼더니 부팅 후 검은 화면이 된다

백포트를 치우려고 `updates/` 폴더 전체를 `.bak`으로 옮기고 재부팅했다. 부팅 로고가 뜬 뒤 화면이 까맣게 됐다. SSH는 됐고, 로그에 이 줄이 있었다.

```bash
sudo journalctl -u systemd-modules-load.service -b --no-pager
# Failed to insert module 'nvgpu': Invalid argument
```

`nvgpu`는 Tegra GPU 드라이버이고, 같은 `updates/` 안에 들어 있었다. 화면 출력에 필요한 모듈까지 같이 사라진 것이다. SSH로 들어가 폴더를 원위치하고 모듈 의존성을 다시 만든 뒤 재부팅해서 복구했다.

```bash
sudo mv /lib/modules/5.15.185-tegra/updates.bak /lib/modules/5.15.185-tegra/updates
sudo depmod -a
sudo reboot
```

그다음에는 백포트에 해당하는 세 곳만 옮겼다.

```bash
sudo mkdir -p ~/wifi_backport_disabled
sudo mv /lib/modules/5.15.185-tegra/updates/compat ~/wifi_backport_disabled/
sudo mv /lib/modules/5.15.185-tegra/updates/net/wireless ~/wifi_backport_disabled/
sudo mv /lib/modules/5.15.185-tegra/updates/net/mac80211 ~/wifi_backport_disabled/
sudo depmod -a
find /lib/modules/5.15.185-tegra/updates -maxdepth 2    # 재부팅 전에 확인
```

재부팅 전에 `find`로 `nvgpu.ko`, `opensrc-disp/`, `dkms/88x2bu.ko`가 남아 있는지 봤다. 이후 드라이버가 정상적으로 올라왔다.

## 같은 증상이면 먼저 볼 것

- 보드 이름은 `/proc/device-tree/model`로 확인하고, 매뉴얼도 그 이름 기준으로 본다.
- 복구 모드는 호스트의 `lsusb`에 NVIDIA 장치가 보이는 것을 확인한 뒤에 플래시한다.
- 모듈 문제는 `lsmod`의 의존 관계와 `/lib/modules/<커널>/updates/` 안의 내용부터 본다. 폴더를 옮기기 전에 `find`로 안에 뭐가 있는지 보고, 재부팅 전에 다시 확인한다.
- 안내문의 `<...>` 자리표시자는 실제 값으로 바꿔서 실행한다.
