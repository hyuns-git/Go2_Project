Go2 EDU에서 이미 만들어 둔 지도를 불러와 AMCL로 위치를 잡고, Nav2의 `navigate_to_pose`로 지정한 지점까지 보내는 데까지 갔다. Foxy에서 쓰던 `nav2_go2.yaml`을 Humble로 옮기면서 AMCL이 안 뜨고, 복구 서버 설정이 안 맞고, 행동 트리 XML이 없는 액션을 부르는 문제를 차례로 만났다. 각각 어떤 메시지를 보고 무엇을 고쳤는지 적는다.

## 환경

- Jetson Orin NX (Ubuntu 22.04, ROS2 Humble), 전력 모드 MAXN
- 입력: 앞 글들의 `scan_maker_l1`이 내는 `/scan`, `/odom`, TF (`odom → base_footprint → base_link`)
- 지도: 기존에 만들어 둔 2D 지도(`office_ext01`, 315×261 픽셀)를 재사용
- 로봇을 움직이는 경로: Nav2의 `/cmd_vel` → 직접 만든 `cmd_vel_bridge` → Go2 Sport API
- tmux 안에서 노드를 띄운다. 로봇을 껐다 켜면 Orin도 같이 재부팅되어 tmux와 노드가 전부 죽는다.

## 위치추정부터 따로 띄운다

Nav2 전체를 한 번에 올리지 않고, 지도와 AMCL만 먼저 올려서 위치가 잡히는지 보았다.

```bash
ros2 launch nav2_bringup localization_launch.py \
  map:=$HOME/maps/office_ext01.yaml \
  params_file:=$HOME/ros2_ws/src/go2_nav_bridge/config/nav2_go2.yaml
```

`params_file`을 빼면 기본 AMCL 파라미터로 뜨기 때문에 항상 명시했다.

## AMCL이 robot_model_type에서 실패한다

지도는 정상으로 올라왔는데 AMCL이 활성화되지 못했다. 설정에는 이렇게 적혀 있었다.

```bash
grep -n "robot_model_type" ~/ros2_ws/src/go2_nav_bridge/config/nav2_go2.yaml
```

```text
29:  robot_model_type: "omni"
```

로그의 `Declared types are ...` 목록에 `nav2_amcl::OmniMotionModel`이 있었다. Humble의 AMCL은 짧은 별칭 `omni`를 받지 않고 플러그인의 전체 클래스 이름을 요구한다. Go2는 옆걸음도 걸을 수 있어서 omni 모델 자체는 맞는 선택이었고, 표기만 바꿨다.

```bash
sed -i 's/robot_model_type: "omni"/robot_model_type: "nav2_amcl::OmniMotionModel"/' \
  ~/ros2_ws/src/go2_nav_bridge/config/nav2_go2.yaml
```

## 다시 띄웠더니 노드가 두 벌이다

고친 뒤 같은 launch를 바로 다시 보냈는데, 실패한 launch의 `amcl`과 `map_server`가 아직 살아 있었다. lifecycle manager가 bringup을 중단해도 프로세스는 죽지 않는다.

```bash
ros2 node list
```

```text
/amcl
/lifecycle_manager_localization
/map_server
/scan_maker_l1
```

같은 이름의 노드가 두 벌 뜨려고 경합하게 되니 정리하고 한 번만 올렸다.

```bash
pkill -f nav2_bringup; pkill -f amcl; pkill -f map_server
ros2 node list     # /scan_maker_l1만 남아야 한다
```

## RViz에 "Frame [map] does not exist"가 뜬다

AMCL과 `map_server`가 active가 된 뒤에도 RViz의 Fixed Frame `map`에 오류가 떴다. AMCL은 `/initialpose`를 받기 전까지 `map → odom` TF를 내지 않기 때문에 정상이다. RViz에서 `2D Pose Estimate`로 로봇이 실제로 서 있는 위치와 방향을 찍어서 초기 위치를 준다. 클릭한 지점이 위치이고, 클릭하고 끌어서 정한 방향이 yaw다.

찍은 직후에는 스캔 점(빨간 점)이 벽선에서 벗어나 방 안쪽에 흩어졌다. 로봇을 걸어 보면서 `/amcl_pose`의 공분산을 봤다.

```bash
ros2 topic echo /amcl_pose --once
```

공분산 36개 중 대각선(x, y, yaw) 값은 다음처럼 줄었다.

```text
                 x        y        yaw
초기 직후     0.171    0.225    0.058
조금 걸은 뒤  0.049    0.191    0.038
방향을 꺾은 뒤 0.118    0.035    0.051
```

복도 같은 구조에서는 이동 방향의 오차는 빨리 줄고 옆 방향 오차는 늦게 줄어든다. 방향을 꺾으면 두 축이 서로 뒤바뀌며 줄어서, 코너에서 정지한 뒤 30도씩 나눠 도는 식으로 여러 방향으로 움직여 주면 두 값이 같이 내려간다. 마지막 판정은 RViz에서 스캔 점이 벽선과 5 cm 안쪽으로 겹치는지 눈으로 보는 것이었고, 걸어 본 뒤의 화면은 벽선과 잘 겹쳤다.

## Foxy에서 쓰던 Nav2 설정이 Humble에서 안 맞는다

위치가 잡힌 다음 컨트롤러, 플래너, 복구 서버, `bt_navigator`까지 전체 Nav2를 올리면서 `nav2_go2.yaml`을 Humble 규칙으로 고쳤다.

- `recoveries_server`는 `behavior_server`로 이름이 바뀌었고 플러그인 경로도 달라졌다.
- 복구 서버의 `transform_timeout`은 `transform_tolerance`가 되었다.
- 복구 동작 중 `back_up`은 뺐다. Go2는 뒤쪽 시야가 없어서 후진은 쓰지 않기로 했다.
- `bt_navigator`의 `default_bt_xml_filename`은 더 이상 쓰지 않고, `default_nav_to_pose_bt_xml`과 `default_nav_through_poses_bt_xml`로 나뉘었다.
- `plugin_lib_names`는 Humble 기본 목록 전체(40여 개)를 넣어야 했다.

마지막이 까다로웠다. 설정에서 `back_up`을 빼도 Humble에 들어 있는 기본 행동 트리 XML의 복구 부분이 `BackUp` 액션을 부르고 있었다. 서버에 없는 액션을 부르면 복구가 발동하는 순간 트리가 멈출 수 있어서, 재시작하기 전에 먼저 XML을 확인했다.

```bash
grep -n "BackUp\|<Recovery\|RoundRobin" \
  /opt/ros/humble/share/nav2_bt_navigator/behavior_trees/navigate_to_pose_w_replanning_and_recovery.xml
```

기본 XML을 고치지 않고 복사해서 `BackUp` 줄을 지운 사본을 만들고, yaml이 그 사본을 가리키게 했다. 이렇게 한 뒤 전체 Nav2가 올라왔다.

## 목표 지점으로 보내는 도구

목표를 `/goal_pose` 토픽으로 쏘면 "보냈다"는 것만 알 수 있고 도착했는지는 알 수 없다. `bt_navigator`가 여는 `navigate_to_pose`는 액션 서버라서 목표를 보내고, 피드백을 받고, 성공이나 실패 결과를 받을 수 있다. 그래서 `waypoint_tool.py`는 액션 클라이언트로 짰다.

```python
from nav2_msgs.action import NavigateToPose
from rclpy.action import ActionClient

client = ActionClient(node, NavigateToPose, 'navigate_to_pose')
goal = NavigateToPose.Goal()
goal.pose.header.frame_id = 'map'
# x, y, yaw(쿼터니언으로 변환)는 waypoints.yaml에서 읽는다
future = client.send_goal_async(goal)
```

성공하면 종료 코드 0, 실패하면 1을 돌려준다. 쉘에서 여러 번 반복 실행하면서 성공 횟수를 셀 때 쓰려는 것이다. 저장된 웨이포인트 파일에는 예전 형식(`yaw`)과 새 형식(`yaw_rad`, `yaw_deg`)이 섞여 있어서 두 가지를 다 읽게 했다.

처음 실행했을 때는 목표는 수락되는데 로봇이 움직이지 않았다. Nav2는 `/cmd_vel`을 정상으로 내고 있었고, 이것을 Sport API로 전달하는 `cmd_vel_bridge`가 떠 있지 않았다. 브리지를 띄운 뒤 시험 지점 두 곳에 모두 도착했다. 브리지에는 0.4초 워치독이 있어서 명령이 끊기면 로봇이 멈춘다.

## 같은 증상일 때 확인할 것

```bash
ros2 node list                        # 같은 이름의 노드가 중복되어 있지 않은가
ros2 lifecycle get /amcl              # (끝나지 않을 때가 있으니 timeout 3을 붙인다)
ros2 topic echo /amcl_pose --once     # 공분산 대각선이 줄어드는가
ros2 run tf2_ros tf2_echo map odom    # initialpose 이후에 나오는가
ros2 topic echo /cmd_vel              # Nav2가 명령을 내고 있는가
ros2 node list | grep cmd_vel_bridge  # 그 명령을 로봇에 전달하는 노드가 떠 있는가
```

목표는 수락되는데 로봇이 안 움직이면 `/cmd_vel`이 나오는지, 그리고 그것을 받는 노드가 있는지부터 본다.

## 확인하지 못한 것

- 시험 지점 두 곳에 도착한 것까지만 확인했다. 계획했던 반복 시험(5회 중 4회 이상 성공)의 성공률은 재지 못했다.
- 이 지도를 어떤 경로로 만들었는지와, 새 매핑 대신 이 지도를 재사용해도 좌표계가 맞는지는 이 글에서 따로 검증하지 않았다. 시험 매핑 세 번이 품질 기준을 통과하지 못한 뒤에 새로 만드는 대신 기존 지도를 재사용하는 쪽으로 갔다.
- AMCL 공분산이 최종적으로 얼마까지 내려갔는지는 재지 않았다. 판정은 RViz에서 눈으로 본 겹침이었다.
- 위 설정은 Humble 기준이다. 다른 배포판에서도 같은 클래스 이름과 파라미터를 쓰는지는 확인하지 못했다.
