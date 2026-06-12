# Storagy Simulation System (Docker)

Storagy 로봇 ROS 2 시뮬레이션(Gazebo + Nav2 + YOLO 사람감지 + LLM 에이전트 + 웹 대시보드)을
**Docker 하나로** 실행할 수 있게 패키징한 레포입니다.

- **GPU 불필요** — 소프트웨어 렌더링으로 동작
- **Windows / macOS(Apple Silicon 포함) / Ubuntu** 어디서나 동일하게 실행
- Gazebo/RViz 화면은 **웹 브라우저(noVNC)** 로 표시 — 호스트에 X 서버 설정 불필요
- `src/` 폴더가 **볼륨 마운트**되어 있어, 내려받은 사람이 코드를 직접 수정·실험 가능

## 구성

| 구성요소 | 내용 |
|---|---|
| 베이스 이미지 | `tiryoh/ros2-desktop-vnc:humble` (ROS 2 Humble + MATE 데스크탑 + noVNC) |
| 시뮬레이터 | Gazebo Harmonic (gz-sim8) + ros_gz |
| 내비게이션 | Nav2, slam_toolbox (apt 바이너리) |
| 비전 | YOLOv8 (ultralytics, CPU 전용 PyTorch) |
| LLM 에이전트 | LangChain / LangGraph + OpenAI API |
| 웹 대시보드 | Flask (포트 8090) |

## 빠른 시작

사전 준비: [Docker Desktop](https://www.docker.com/products/docker-desktop/)(Windows/Mac) 또는 Docker Engine + compose 플러그인(Ubuntu).

```bash
git clone https://github.com/bluephysi01/storagy-simulation-system-docker.git
cd storagy-simulation-system-docker

# (선택) LLM 에이전트를 쓰려면 OpenAI API 키 설정
cp .env.example .env       # .env 를 열어 본인 키 입력. 없어도 시뮬레이션 자체는 동작

# 실행 — Docker Hub에 올라간 이미지를 받아서 실행 (없으면 로컬 빌드: docker compose build)
docker compose up -d
```

브라우저에서 접속:

| 주소 | 화면 |
|---|---|
| http://localhost:6080 | 리눅스 데스크탑 (Gazebo / RViz 창이 여기에 뜸) |
| http://localhost:8090 | 웹 대시보드 |

컨테이너가 시작되면 데스크탑 세션 안에서 시뮬레이션(`ros2 launch storagy full_bringup.launch.py`)이
터미널과 함께 **자동 실행**됩니다. 첫 실행은 Gazebo 모델 로딩 때문에 1~2분 걸릴 수 있습니다.

종료:

```bash
docker compose down
```

## 코드 수정하기 (볼륨 마운트)

호스트의 `./src` 폴더가 컨테이너의 `/opt/storagy_sim_origin_ws/src` 에 마운트되어 있어,
**호스트에서 파일을 수정하면 컨테이너에 즉시 반영**됩니다.

- `src/storagy/scripts/*.py` (YOLO, 배회 노드 등) — 소스에서 직접 실행되므로 **시뮬레이션 재시작만** 하면 반영
- 런치파일 / 월드 / 맵 / URDF / 메시지 정의 등 — 수정 후 워크스페이스 **재빌드** 필요:

```bash
docker compose exec storagy-sim rebuild_ws.sh
docker compose restart
```

noVNC 데스크탑 안의 터미널에서 직접 `rebuild_ws.sh` / `run_sim.sh` 를 실행해도 됩니다.

## 직접 빌드 / 이미지 배포 (메인테이너용)

```bash
# 소스에서 이미지 빌드
docker compose build

# Docker Hub 로그인 후 푸시
docker login
docker compose push   # bluephysi01/storagy-simulation-system-docker:latest
```

Docker Hub 저장소가 **Public** 이어야 다른 사람이 로그인 없이
`docker compose up` 만으로 이미지를 받을 수 있습니다 (신규 저장소 기본값은 Public).

## 폴더 구조

```
├── Dockerfile              # 이미지 정의 (Gazebo/Nav2/YOLO/LLM 의존성 + colcon build)
├── docker-compose.yml      # 포트, 볼륨 마운트, .env 주입
├── docker/
│   ├── run_sim.sh          # 데스크탑 세션에서 시뮬레이션 실행 (자동 시작됨)
│   └── rebuild_ws.sh       # src 수정 후 컨테이너 안에서 colcon 재빌드
├── requirements.txt        # Python 의존성 (CPU 전용)
├── yolov8n.pt              # YOLOv8 nano 가중치
├── .env.example            # OPENAI_API_KEY 템플릿 (.env 로 복사해 사용)
└── src/
    ├── storagy/            # 로봇 모델, 월드, 맵, 런치, YOLO/배회 스크립트
    ├── storagy_interfaces/ # 커스텀 메시지/서비스
    └── storagy_llm/        # LLM 에이전트 + 웹 대시보드
```

## 트러블슈팅

- **Gazebo 화면이 안 뜸 / 검은 화면**: 첫 로딩이 느립니다. noVNC 데스크탑의 "Storagy Simulation"
  터미널 로그를 확인하세요. 메모리 부족이면 Docker Desktop 리소스(메모리 6GB 이상 권장)를 늘려주세요.
- **LLM 에이전트가 응답하지 않음**: `.env` 의 `OPENAI_API_KEY` 가 설정됐는지 확인 후
  `docker compose up -d` 로 재생성하세요.
- **포트 충돌**: `docker-compose.yml` 의 `6080`, `8090` 을 다른 포트로 바꾸세요.
