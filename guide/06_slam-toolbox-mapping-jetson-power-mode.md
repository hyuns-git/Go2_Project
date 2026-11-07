Go2 EDU에서 `/scan`을 slam_toolbox에 넣고 사무실 한 층을 매핑했는데, 지도가 육안으로는 멀쩡해 보이는데도 품질 점검 스크립트가 세 번 연속 FAIL을 냈다. 로그에서 Ceres가 쓸 수 있는 스레드가 4개뿐이라는 메시지를 찾았고, 원인은 Jetson의 전력 모드였다. 설정, 주행 방법, 점검 결과와 아직 풀지 못한 부분까지 적는다.

## 환경

- Jetson Orin NX (Ubuntu 22.04, ROS2 Humble), 전력 모드 15W로 운용
- 입력: 앞 글의 변환 노드가 만든 `/scan` (180빔, 10 Hz), TF `odom → base_footprint`
- slam_toolbox `sync_slam_toolbox_node`. 스캔을 버리지 않고 모두 처리하는 대신 CPU를 많이 쓴다.
- 지도 저장: `nav2_map_server`의 `map_saver_cli`

## 설정과 실행

L1 LiDAR 기준으로 바꾼 값은 아래와 같다.

```yaml
# slam_precise.yaml (일부)
max_laser_range: 10.0
minimum_travel_distance: 0.25
scan_buffer_maximum_scan_distance: 12.0
```

실행은 tmux 창에 띄운다. SSH가 끊겨도 노드가 같이 죽지 않게 하려는 것이다.

```bash
ros2 run slam_toolbox sync_slam_toolbox_node \
  --ros-args --params-file ~/ros2_ws/src/go2_nav_bridge/config/slam_precise.yaml
```

시작할 때 이 경고가 한 줄 뜬다. LiDAR의 최소 거리는 0.3 m인데 설정의 최소 거리가 0.0 m라는 내용이고, 그대로 두고 매핑을 돌렸다.

```text
[WARN] [slam_toolbox]: minimum laser range setting (0.0 m) exceeds the capabilities of the used Lidar (0.3 m)
```

## 주행 방법

벽 한 바퀴를 돌 때 지켜 온 규칙이다.

- 출발점은 벽에서 1.5~2 m 떨어진 곳에 테이프로 표시해 두고, 출발 후 10초 정지한다.
- 벽을 따라 사각형으로, 속도는 0.3 m/s 이하로 간다. 방 한가운데는 지나가지 않는다.
- 코너에서는 3초 정지, 30도씩 세 번 나눠 회전, 3초 정지 순서로 돈다.
- 출발점으로 돌아와 10초 정지하고, 반대 방향으로 한 바퀴 더 돈 뒤 다시 10초 정지한다.

끝나면 지도를 저장하고 점검 스크립트로 평가한다.

```bash
ros2 run nav2_map_server map_saver_cli -f ~/maps/l1_test03 --occ 0.65 --free 0.25
python3 ~/go2-nav/tools/eval_map.py ~/maps/l1_test03.pgm
```

`eval_map.py`는 지도의 벽 픽셀마다 이웃한 벽 픽셀 수를 세서 평균을 낸다. 벽이 한 줄로 얇게 그려지면 이웃이 2개 안팎이고, 두 겹으로 번지면 5개 이상이 된다. 통과 기준은 평균 3.0 미만이다.

같은 사무실에서 같은 규칙으로 Orin Nano(Foxy)에서 돌렸을 때는 이 값이 2.51이었다. 이웃 수 2.0~2.6이 얇은 벽, 3.0 이상이 이중벽 의심이다. 방 한가운데를 20 m 원으로 돌았을 때는 벽이 점으로 흩어졌고, 벽에서 1.5~2 m 떨어져 사각형으로 돌고 코너에서 멈췄을 때 벽이 한 픽셀 두께의 연속선으로 나왔다. 이 규칙은 그때 얻은 것이다.

## 지도는 멀쩡한데 점검이 FAIL이다

세 번 돌린 결과다.

```text
l1_test01   벽 평균 이웃수 5.441   두꺼운 벽(이웃 5+) 70.4%   FAIL
l1_test02   벽 평균 이웃수 5.103   두꺼운 벽(이웃 5+) 66.7%   FAIL
l1_test03   벽 평균 이웃수 4.767   두꺼운 벽(이웃 5+) 62.1%   FAIL
```

스크립트는 "파라미터 말고 주행을 고쳐라"며 속도를 더 줄이고, 코너 정지를 5초로 늘리고, 벽과의 거리를 일정하게 유지하라고 권했다. 그런데 RViz에서 보면 바깥 벽은 얇고 깨끗했고, 방 안쪽에 벽과 이어지지 않은 덩어리와 부채꼴로 퍼진 점 무리가 떠 있었다. 덩어리 안의 픽셀은 이웃이 많아서 평균을 끌어올리는 것으로 보였다. 이 점검이 FAIL을 낸 이유는 벽 두께보다 이 덩어리 쪽에 있을 가능성이 크다.

## Ceres가 4스레드밖에 못 쓴다

slam_toolbox 창에는 이 로그가 같이 올라와 있었다.

```text
Specified options.num_threads: 50 exceeds maximum available from the threading model Ceres was compiled with: 4.
...
Message Filter dropping message: frame 'base_footprint' at time ... reason 'discarding message because the queue is full'
```

`slam_toolbox`는 루프 클로저를 Ceres로 최적화한다. 이 메시지는 Ceres가 쓸 수 있는 스레드가 4개라는 뜻이었다. 같은 시각 `tegrastats`를 보니 코어 여덟 개 중 네 개가 `off`였다.

```bash
sudo tegrastats --interval 1000
```

```text
CPU [25%@1420,14%@1420,17%@1420,16%@1420,off,off,off,off]
```

15W 모드에서는 코어 네 개가 꺼진다. 코어가 절반이면 루프 클로저 최적화가 주행을 따라가지 못하고, 그 사이에 스캔이 큐에 쌓이다가 버려진다. 속도를 줄이라는 점검 스크립트의 처방은 이 상태에서는 원인을 비껴간다.

현재 모드를 보고 MAXN으로 바꿨다.

```bash
sudo nvpmodel -q
sudo nvpmodel -m 0
sudo jetson_clocks
```

바꾼 뒤 `tegrastats`에서 여덟 개 코어가 모두 클럭 값(`@729`)을 보였다. 세 번째 매핑이 이 상태에서 돌린 것이고 결과는 위의 4.767이다. 앞의 5.1~5.4에서 조금 나아졌지만 통과 기준에는 한참 못 미친다.

이 보드를 15W로 낮춰 둔 이유는 배터리 소모다. MAXN은 배터리를 많이 쓴다. 그래서 15W를 기본으로 두고 매핑할 때만 MAXN으로 올리는 운용을 생각해 볼 만하다.

## 같은 증상일 때 확인할 것

```bash
sudo nvpmodel -q                      # 현재 전력 모드
sudo tegrastats --interval 1000       # off로 표시된 코어가 있는가
# slam_toolbox 로그에서 아래 두 줄이 반복되는가
#   num_threads ... exceeds maximum available ... Ceres
#   Message Filter dropping message ... queue is full
python3 eval_map.py <지도>.pgm        # 육안과 숫자를 둘 다 본다
```

`Message Filter dropping message`가 한 번 뜬 것만으로는 판단하지 않는다. 반복해서 뜨는지, 같은 시각에 CPU 코어가 꺼져 있는지를 같이 본다.

## 확인하지 못한 것

- MAXN으로 올린 뒤에도 방 안쪽 덩어리와 부채꼴 점 무리는 남았다. 이 덩어리의 원인은 확정하지 못했다. 방 한가운데를 지나가서 L1의 산란 패턴이 찍힌 것인지, 사무실 집기인지, 코너 회전 때 스캔이 잘못 붙은 것인지는 가르지 못했다.
- Orin Nano(Foxy)의 2.51과 이 보드(Humble)의 4.7~5.4 사이에 어떤 차이가 있는지는 하나씩 갈라 보지 못했다. 보드, ROS 배포판, 변환 노드 구현, 전력 모드가 모두 달랐다.
- 회전할 때 스캔과 TF의 타임스탬프를 맞추는 수정을 넣어 다시 돌렸는데 같은 모양이 나왔다. 그 테스트는 slam_toolbox를 막 재시작해서 pose graph가 거의 빈 상태였다. 그래서 수정이 무효였는지 판단하지 못했고, 코너만 짧게 돌리는 테스트는 진단 방법으로 맞지 않았다. 전체 루프를 돌려 봐야 한다.
- 이 기록은 시험 매핑 3회의 결과까지다. 이후 기존에 만들어 둔 지도(`office_ext01`)를 재사용해서 AMCL로 위치를 잡은 내용은 다음 글에서 다룬다.
