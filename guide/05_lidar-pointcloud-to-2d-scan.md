Go2 EDU 머리의 LiDAR 점군(`/utlidar/cloud_deskewed`)으로 slam_toolbox를 돌리려면 2D `LaserScan`이 필요하다. `pointcloud_to_laserscan`을 쓰는 대신 변환 노드를 직접 만들었다. 이유는 로봇 본체와 Jetson의 시계가 49초 어긋나 있었고, 점군의 90% 가까이가 좌표 (0,0,0)인 더미 점이었기 때문이다. 이 두 가지를 어떻게 확인했고 노드를 어떻게 짰는지 적는다.

## 환경

- 센서: Go2 EDU 머리의 L1 LiDAR. 토픽 `/utlidar/cloud_deskewed` (PointCloud2)
- 측정과 첫 구현: 확장독의 Jetson Orin Nano (Ubuntu 20.04, ROS2 Foxy)
- 최종 노드: Jetson Orin NX (Ubuntu 22.04, ROS2 Humble)
- 출력: `/scan`, `/odom`, TF (`odom` → `base_footprint` → `base_link`)

점군은 이미 `odom` 좌표계로 변환돼서 나온다. `frame_id`가 `odom`이어서 LiDAR에서 `base_link`로 가는 정적 TF는 따로 만들 필요가 없었다.

## 점군 타임스탬프가 49초 뒤에 있다

Jetson에서 점군이 도착할 때마다 헤더의 시각과 Jetson의 현재 시각 차이를 열 번 재 봤다.

```python
s.append(m.header.stamp.sec + m.header.stamp.nanosec * 1e-9 - time.time())
```

```text
offset = -49.1263 s (min -49.1641 / max -49.1196)
```

편차가 0.045초라서 상수 오프셋이다. LiDAR를 발행하는 로봇 본체 보드의 시계와 Jetson의 시계가 맞지 않는다는 뜻이다.

이 상태에서 `pointcloud_to_laserscan`은 점군의 원래 스탬프로 TF를 조회하는데, tf2 버퍼는 기본 10초치만 들고 있다. 49초 전의 변환은 버퍼에 없으니 조회가 실패하거나 엉뚱한 자세에 붙는다. 그래서 점군의 스탬프를 읽지 않고, 받은 시점의 Jetson 시계로 스캔과 TF와 odom을 함께 찍는 노드를 만들었다.

같은 증상은 다른 센서에서도 나왔다. 이전에 붙였던 Ouster 센서용 변환 노드를 RViz에 올렸을 때는 아무것도 안 뜨고 slam_toolbox 쪽에 이 로그만 계속 쌓였다.

```text
Message Filter dropping message: frame 'base_footprint' ... reason 'discarding message because the queue is full'
```

이때 오프셋은 108초였다. TF와 odom을 로봇 원본 스탬프 대신 `self.get_clock().now()`로 찍게 고치자 맵이 떴다. 오프셋 값은 로봇을 재부팅할 때마다 달라질 수 있어서 세션을 시작할 때마다 다시 재는 것이 좋다. Orin Nano 쪽에서는 이후 같은 방법으로 -92.01초가 나온 적도 있다.

## 점군의 86.5%가 (0,0,0)이다

높이 필터를 정하려고 점군의 Z 분포를 뽑았더니 p5부터 p95까지 전부 `+0.000 m`로 나왔다. 포인트의 대부분이 한 점에 몰려 있다는 뜻이라서 필드 구조와 분류를 따로 확인했다.

```text
=== PointCloud2 구조 ===
  frame_id : odom
  w x h : 11554 x 1 = 11554
  point_step: 32  is_dense: True
  field x offset= 0 type=7
  field y offset= 4 type=7
  field z offset= 8 type=7
  field intensity offset=16 type=7
=== 포인트 분류 (12 프레임, 총 138668점) ===
  NaN/Inf     :       0 (0.0%)
  정확히 0,0,0: 120000 (86.5%)  <- 더미
  유효        :   18668 (13.5%) = 프레임당 1556점
```

`is_dense: True`이고 NaN도 없다. `isfinite`로 거르는 방식으로는 이 점들이 하나도 걸러지지 않는다. 반사가 없을 때 0으로 채워 보내는 값으로 보인다. 로봇을 엎드린 자세와 선 자세에서 각각 재 봐도 같았다.

이 점들은 `odom` 원점에 박혀 있으니 그대로 쓰면 로봇이 멀어질수록 출발점에 가짜 장애물 덩어리가 생긴다. 변환 노드의 첫 단계는 좌표가 정확히 (0,0,0)인 점을 버리는 것이다.

```python
valid = np.isfinite(p).all(1) & (np.abs(p).max(1) > 1e-6)
p = p[valid]
```

## 높이 범위를 어디서 자를까

더미를 걸러낸 유효점으로 다시 재 봤다.

```text
=== 유효점 Z 분포 (odom frame) ===
  p5  -0.112 m   p50 -0.051 m   p75 +0.001 m   p90 +0.364 m   p99 +1.292 m
=== 수평거리 분포 ===
  p0 0.30 m   p50 0.95 m   p95 5.02 m   p100 7.83 m
robot_odom.pose.z: +0.322 m   (서 있을 때)
```

엎드렸을 때 `robot_odom.pose.z`는 0.053 m였고, 서면 0.322 m였다. Go2가 서면 몸통이 바닥에서 0.3 m쯤 올라오니 `odom`의 z=0은 바닥면이다. 그래서 `base_footprint`를 z=0에 놓았다.

수평거리 최소값이 0.30 m라서 `range_min`은 0.30으로 잡았다. 높이 범위는 `0.05 ~ 0.60 m`로 잡았다. 바닥과 다리 높이의 노이즈를 거르는 값이다. 측정 스크립트가 추천한 값은 0.08 ~ 0.83 m였는데, 이 값을 그대로 쓰지는 않았다.

## 변환 노드의 처리 순서

한 번 오는 점군에는 유효점이 프레임당 1500개쯤밖에 없다. 한 프레임으로는 벽이 듬성듬성해서 `odom` 좌표계로 약 0.75초(8프레임) 쌓은 뒤 변환한다. 쌓는 프레임 수는 각도 커버리지(유효한 빔의 비율)를 재서 정했다. 1.0도 빔 기준으로 2프레임이 30%, 4프레임이 41%, 8프레임이 49%, 12프레임이 51%여서 8프레임 이후로는 늘어나는 폭이 작았다. 빔은 2.0도 간격 180개로 정했다. 이 구성의 각도 커버리지는 예측이 53%, 실측이 93/180빔으로 51.7%였다.

1. 점군을 받으면 더미 점을 버리고 높이 범위 안의 점만 `odom` 좌표로 큐에 넣는다.
2. 10 Hz 타이머가 큐의 점을 합치고, 최신 `robot_odom`의 위치와 yaw로 `base_footprint` 기준 좌표로 바꾼다.
3. 각도를 2도 간격 180칸으로 나누고 칸마다 가장 가까운 거리를 `ranges`에 넣는다.
4. `/scan`은 현재 시각으로 스탬프를 찍어서 낸다.
5. 50 Hz 타이머가 yaw만 담은 `odom → base_footprint`, `base_footprint → base_link` TF와 `/odom`을 같은 방식으로 낸다.

```python
dx, dy = rotate(p[:, :2] - pos, -yaw)          # base_footprint 기준
idx = ((np.arctan2(dy, dx) + np.pi) / np.deg2rad(2.0)).astype(int) % 180
ranges = np.full(180, np.inf)
np.minimum.at(ranges, idx, np.hypot(dx, dy))
```

롤과 피치는 TF에 넣지 않았다. 4족 보행이라 걸을 때 몸이 계속 흔들리기 때문에 `base_footprint`는 위치와 yaw만 따라가게 두었다.

실행하면 시작 로그에 설정이 찍힌다.

```text
[scan_maker_l1]: scan_maker_l1: 180 beams (2.00 deg), accum 0.75s, z=[0.05,0.6]
```

검증 때는 `/scan`이 10 Hz로 나왔고, TF의 z는 0.000으로 고정됐고, 유효 빔은 61.9%였다.

## 같은 증상일 때 확인할 것

```bash
# 점군 스탬프와 로컬 시계의 차이 (수 초 이상이면 tf 조회가 깨진다)
ros2 topic echo /utlidar/cloud_deskewed --field header.stamp --once; date +%s

# 필드 구조와 더미 점 비율은 PointCloud2 필드를 numpy로 읽어서 확인한다
ros2 topic hz /scan
ros2 run tf2_ros tf2_echo odom base_footprint
```

`pointcloud_to_laserscan`으로 `/scan`이 비어 있거나 slam_toolbox에 `queue is full`이 쌓이면 먼저 스탬프 차이를 본다. 지도 한가운데 이유 없는 덩어리가 생기면 점군에 (0,0,0)이 섞여 있는지 본다.

## 확인하지 못한 것

- 49초 오프셋 때문에 `pointcloud_to_laserscan`이 실제로 실패하는 장면은 내가 직접 보지 못했다. tf2 버퍼가 10초치라는 점에서 생기는 결과로 설명했을 뿐이다. 108초 오프셋에서 TF 스탬프를 로컬 시계로 바꾸자 맵이 뜬 것은 확인했다.
- 벽 앞에서 회전할 때 스캔이 산탄처럼 번지는 문제는 끝내 원인을 확정하지 못했다. 스캔과 TF가 서로 다른 타이머에서 스탬프를 받는 것을 의심해 한 스탬프로 맞추는 수정을 넣었지만, 같은 모양의 번짐이 그대로 나왔다. 그 테스트는 slam_toolbox를 막 재시작해서 지도가 비어 있는 상태였고, 변수가 하나 더 끼어 있어서 이 수정의 효과 여부는 가르지 못했다.
