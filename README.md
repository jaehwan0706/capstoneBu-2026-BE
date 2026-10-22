# capstone-backend

음원 기반 악기 분리·타브 생성 및 카피 보조 웹서비스의 백엔드입니다.

## 기술 스택

| 항목 | 선택 |
|---|---|
| 언어 | Python 3.12 |
| 패키지 관리 | uv |
| API | FastAPI + Pydantic 2 |
| DB | PostgreSQL 17 + SQLAlchemy 2.0(동기) + Alembic |
| 작업 큐 | Celery 5 + Redis 7 |
| 파일 저장소 | SeaweedFS (S3 호환, 배포 시 AWS S3로 교체 가능) |
| AI 모델 | 기타 분리 BS-Roformer-SW (audio-separator) · 음 인식 basic-pitch (ONNX) · 박 추정 Beat This! |
| AI 실행 | Celery worker 전용 이미지 · PyTorch CPU · 가중치는 `model-cache` 볼륨 (라이선스: [`docs/MODELS.md`](docs/MODELS.md)) |
| 품질 | ruff · mypy · pytest |

## 처음 세팅 (Windows 기준)

### 1. 필수 설치
- **Docker Desktop**: https://www.docker.com/products/docker-desktop
- **uv** (PowerShell):
  ```powershell
  powershell -ExecutionPolicy ByPass -c "irm https://astral.sh/uv/install.ps1 | iex"
  ```
- **Git**

### 2. 의존성 설치
```bash
uv python install 3.12   # Python 3.12 자동 설치 (이미 있으면 생략됨)
uv sync                  # .venv 생성 + 패키지 설치 (uv.lock 버전 그대로)
uv sync --group ai       # AI 코드를 로컬에서 작성·실행할 때만 (PyTorch 등, 수 GB)
```

API·CI는 `uv sync`(AI 패키지 없음)만으로 동작합니다. AI 패키지는 worker 이미지에만 들어갑니다.

### 3. 환경 변수
노션 **DB·IP** 페이지에서 `.env` 파일을 받아 **레포 최상위 폴더**(`pyproject.toml`과 같은 위치)에 둡니다.
`.env`는 git에 올리지 않습니다. 설정값을 추가·변경하면 노션의 `.env`도 함께 갱신해 주세요.

AI 설정은 `AI_*` 환경 변수로 덮어쓸 수 있고, 없으면 `app/core/config.py`의 기본값을 씁니다.
```dotenv
AI_MODEL_CACHE_DIR=/models
AI_SEPARATION_MODEL=BS-Roformer-SW.ckpt
AI_CONFIG_VERSION=2026-10-01
```

### 4. 전체 실행
```bash
docker compose up --build
```

| 주소 | 내용 |
|---|---|
| http://localhost:8000/docs | Swagger (API 문서·테스트) |
| http://localhost:8000/api/v1/health | 서버 상태 |
| http://localhost:8000/api/v1/health/ready | DB·Redis·저장소 연결 상태 |
| http://localhost:9333 | SeaweedFS 관리 화면 |

Celery 모니터링(Flower)까지 띄우려면: `docker compose --profile monitor up` → http://localhost:5555

이미지는 빌드 타깃 두 개로 나뉩니다: `api`(AI 패키지 없음) / `worker`(AI 패키지 포함). 코드는 같고 실행 명령만 다릅니다.

### 5. AI 모델 다운로드 (처음 한 번)
```bash
docker compose run --rm worker python scripts/download_models.py
```
가중치는 이미지에 넣지 않고 `model-cache` 볼륨(`/models`)에 받습니다. 이미 있으면 건너뛰고, 파일별 SHA-256을 출력합니다.

> **worker는 코드 자동 반영(--reload)이 없습니다.** `app/` 코드를 고친 뒤에는 `docker compose restart worker`로 다시 띄워 주세요.

## 자주 쓰는 명령어

```bash
uv add <패키지>                 # 의존성 추가
uv add --dev <패키지>           # 개발용 의존성 추가

uv run ruff check .             # 린트
uv run ruff format .            # 자동 포맷
uv run mypy app                 # 타입 체크
uv run pytest                   # 테스트

# DB 마이그레이션 (컨테이너 안에서 실행)
docker compose exec api alembic revision --autogenerate -m "add songs"
docker compose exec api alembic upgrade head
```

## 폴더 구조

**도메인(기능)별**로 나눕니다. 공통 코드는 `core/`에만, 기능 코드는 `domains/<기능>/` 안에만 둡니다.

```
capstone-backend/
├── app/
│   ├── main.py              # 앱 생성, 라우터 등록 (Spring의 Application 클래스)
│   ├── core/                # 공통 (Spring의 global 패키지)
│   │   ├── config.py        #   설정 (.env 읽기)
│   │   ├── database.py      #   DB 연결·세션
│   │   ├── base_entity.py   #   엔티티 부모 클래스 (생성일·수정일·소프트 삭제)
│   │   ├── exceptions.py    #   ErrorCode, BusinessException, 예외 핸들러
│   │   ├── response.py      #   공통 응답 ApiResponse
│   │   ├── middleware.py    #   trace_id
│   │   ├── storage.py       #   파일 저장소 (S3 / presigned URL)
│   │   └── celery_app.py    #   비동기 작업 큐 설정
│   └── domains/             # 기능별 폴더 ← 여기에 기능 추가
│       └── health/          #   예) router.py · schemas.py · models.py · service.py · tasks.py
├── tests/                   # app/ 과 같은 구조 (tests/core, tests/domains/<기능>)
├── alembic/                 # DB 마이그레이션
├── scripts/                 # 운영 스크립트 (모델 다운로드 등)
├── docs/                    # 문서 (모델 라이선스 등)
├── docker/                  # 컨테이너 설정 파일
├── docker-compose.yml
├── Dockerfile
└── pyproject.toml           # 의존성·도구 설정 (Spring의 build.gradle)
```

새 기능을 추가하는 방법은 [`app/domains/README.md`](app/domains/README.md)를 보세요.

## AI 코드 작성 규칙

- **AI 라이브러리는 모듈 최상단에서 import하지 않습니다.** (torch, basic_pitch, beat_this, audio_separator, librosa, music21 등)
  함수 안에서 지연 import 하세요. Celery가 모든 도메인의 `tasks.py`를 자동 import하고, api 이미지와 CI에는 AI 패키지가 없습니다.
  ```python
  def separate(path: str) -> str:
      from audio_separator.separator import Separator  # 함수 안에서
      ...
  ```
- **모델 이름·임계값은 하드코딩하지 않고** `get_settings().ai_*`에서 읽습니다. 값을 바꾸면 `AI_CONFIG_VERSION`도 함께 올립니다.
- 모델 사전 로드는 `app/core/celery_app.py`의 `preload_ai_models` 훅에 연결합니다.
- 공통 코드는 `core/`, 기능 코드는 `domains/<기능>/`에 둡니다. 파일 저장 key는 `{workspace_id}/{song_id}/...` 규칙을 따릅니다.
- 실패는 `ErrorCode`의 AI 파이프라인 코드(`SEPARATION_NO_GUITAR`, `TAB_LOW_CONFIDENCE` 등)로 표현합니다.
- 새 모델·패키지를 추가하면 [`docs/MODELS.md`](docs/MODELS.md)에 라이선스를 적습니다.

## 공통 응답 형식

```json
// 성공
{ "success": true, "data": { ... } }

// 실패
{ "success": false, "error": { "code": "RESOURCE_NOT_FOUND", "message": "요청한 리소스를 찾을 수 없습니다." }, "meta": { "trace_id": "..." } }
```

- Service에서 `raise BusinessException(ErrorCode.XXX)` → 자동으로 위 형식으로 변환
- 에러 코드는 `app/core/exceptions.py`의 `ErrorCode`에서 중앙 관리

## 브랜치 · CI/CD

```
main      ── 프로덕션. 직접 push 금지, develop에서 승격
develop   ── 개발 서버. 기능 브랜치의 PR 대상
  ├─ feat/#12-workspace-session   (fix / refactor / chore / docs / test / ci 동일)
hotfix/#34-xxx  ── main에서 분기 → main 머지 후 develop에도 반영
```

| 워크플로 | 언제 | 하는 일 |
|---|---|---|
| `ci.yml` | develop·main으로 PR, push | ruff · mypy · 마이그레이션(upgrade → check → downgrade → upgrade) · pytest · 도커 빌드 |
| `cd.yml` | develop·main push | GHCR에 이미지 push (`sha-<커밋>` + `develop`/`latest`). 서버 배포 단계는 배포 대상 확정 후 활성화 |

- **CI가 실패한 PR은 머지하지 않습니다.** PR 올리기 전에 로컬에서 `ruff` · `mypy` · `pytest`를 먼저 돌려 주세요.
- 모델을 바꾸고 마이그레이션 파일을 빼먹으면 CI의 `alembic check`에서 실패합니다.
- 롤백: 서버에서 이전 `sha-<커밋>` 이미지로 `IMAGE_TAG`를 바꿔 다시 띄웁니다.

## IDE 설정

**VS Code** (추천)
- 확장: `Python`, `Pylance`, `Ruff`, `Docker`
- `Ctrl+Shift+P` → *Python: Select Interpreter* → `.venv` 선택

**PyCharm / IntelliJ IDEA**
- IntelliJ는 *Python* 플러그인 설치 필요 (Ultimate 권장)
- *Settings → Project → Python Interpreter* → *Add Interpreter → Existing* → `.venv\Scripts\python.exe`
- *Ruff* 플러그인 설치 권장
