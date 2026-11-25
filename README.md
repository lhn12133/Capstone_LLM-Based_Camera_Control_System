# Capstone-II | 다중 카메라 지능형 영상 관제 시스템

> IP 카메라 영상의 실시간 관제와 Vision AI 객체 탐지, 탐지 데이터 저장, 파인튜닝된 LLM의 자연어 관제 분석을 하나의 웹 대시보드로 통합하는 프로젝트입니다.
>
> **문서 기준일: 2026-11-25** · 현재 구현 및 테스트 결과와 최종 통합 설계를 구분하여 작성했습니다.

## 1. 프로젝트 개요

기존 CCTV 관제 방식은 여러 영상 채널을 관제자가 직접 확인해야 하므로 탐지 결과를 체계적으로 관리하고 상황을 신속히 파악하는 데 어려움이 있습니다. 본 프로젝트는 다중 카메라 영상 스트리밍, 객체 탐지, 데이터 저장 및 자연어 관제 분석을 연계하여 관제 업무를 지원하는 것을 목표로 합니다.

- 다중 IP 카메라의 영상을 2×2 웹 대시보드에서 확인
- YOLO를 활용한 사람 탐지 및 탐지 이벤트 생성
- 탐지 이미지와 메타데이터를 분리 저장하고 이력을 조회
- NATS 기반 비동기 메시지 전달 및 LLM 분석 모듈 통합
- 카메라별 탐지 정보와 LLM 분석 결과를 연계하여 상황 판단을 지원

## 2. 전체 시스템 구성도

![전체 시스템 구성도](images/01_architecture.png)

**영상 경로:** IP Camera → RTSP → MediaMTX → WebRTC → Dashboard

**검증된 탐지 경로:** MP4 → OpenCV/YOLO → MinIO + PostgreSQL → Dashboard 탐지 결과 탭

**최종 통합 목표:** MediaMTX RTSP → Vision AI → 저장 완료 후 NATS 이벤트 발행 → Fine-tuned LLM → 분석 결과 저장 및 Dashboard 표시

영상 전송과 탐지 이벤트 처리를 분리하여 카메라 입력을 교체하더라도 동일한 MediaMTX 경로와 데이터 인터페이스를 유지할 수 있도록 설계합니다. NATS 발행과 LLM 저장·표시 경로는 통합 검증이 필요한 설계입니다.

## 3. 개발 기술 스택

![개발 기술 스택](images/04_tech_stack.png)

| 구분 | 기술 | 적용 목적 |
|---|---|---|
| 영상 입력 | MERCUSYS MC510, IP Camera | 네트워크 카메라 영상 제공 (4채널 목표) |
| 영상 스트리밍 | RTSP, MediaMTX | 카메라별 영상 경로 관리, RTSP 입력 및 재배포 |
| 브라우저 영상 | WebRTC, HLS | 실시간 웹 영상 표시, HLS는 대체 경로 |
| Vision AI | Python, OpenCV, Ultralytics YOLOv8n | 영상 프레임 처리, 사람 탐지, confidence/bbox 추출 |
| 관계형 DB | PostgreSQL, psycopg | 탐지 이벤트, 객체 메타데이터 관리 |
| 객체 스토리지 | MinIO, minio Python SDK | 탐지 시점 JPEG 이미지 저장 및 조회 |
| 메시지 브로커 | NATS, nats-py | 탐지 이벤트 및 LLM 분석 메시지 전달 |
| 자연어 AI | Fine-tuned LLM | 관제 상황의 자연어 설명 및 분석 (팀원 개발 완료 보고, 통합 미검증) |
| 백엔드 | Python, Flask | Dashboard 제공, 탐지 조회 API, MinIO 이미지 프록시 |
| 프론트엔드 | HTML, CSS, JavaScript | 4채널 영상 화면, 이벤트 및 탐지 결과 표시 |
| 실행 환경 | Windows, Anaconda, Docker Desktop | Python 개발 환경 및 DB/메시징 인프라 실행 |
| 환경 설정 | python-dotenv, .env | 서버 주소 및 접속 정보 분리 관리 |
| 협업 | GitHub | 팀 코드 공유 및 버전 관리 |

LLM의 기반 모델, LoRA/QLoRA 여부, 학습 데이터 규모, 모델 가중치, 추론 API 및 평가 지표는 저장소 접근이 되지 않아 확인하지 못했습니다. 발표에서는 확인 후 추가하고 추정 기술을 확정적으로 기재하지 않습니다.

## 4. 주요 기능 및 데이터 흐름

![탐지 처리 흐름](images/02_data_flow.png)

### 4.1 영상 스트리밍

- 카메라별 MediaMTX 경로: `cam01`, `cam02`, `cam03`, `cam04`.
- Dashboard는 카메라별 WebRTC URL을 iframe에 표시합니다.
- 개별 영상 불러오기/끄기, 전체 영상 불러오기, 실행 중 URL 변경 기능을 제공합니다.
- 실제 카메라와 MediaMTX 간 RTSP 인증 성공 후 영상 수신을 확인해야 합니다.

### 4.2 Vision AI

- 기본 모델: `yolov8n.pt`.
- OpenCV `VideoCapture`로 영상 프레임을 획득하여 YOLO로 추론합니다.
- 현재 MVP의 저장 대상은 `class_id=0`(person), 기본 confidence 0.50 이상인 객체입니다.
- 기본 이벤트 저장 간격은 3초이며, UUID로 이벤트를 식별합니다. 지속적으로 사람이 탐지되면 여러 이벤트가 생성될 수 있습니다. 이 수치는 정확도 지표나 추적된 고유 인원 수가 아닙니다.
- 탐지 이벤트가 생성되면 원본 프레임을 JPEG로 업로드한 후 이벤트 및 객체 정보를 DB에 저장합니다. bbox는 별도 메타데이터입니다.
- 실제 다중 카메라 동시 추론 및 행동 인식은 추가 검증 대상입니다.

### 4.3 저장 및 Dashboard 조회

- PostgreSQL DB: `capstone` / MinIO bucket: `capstone2` (현재 통합 테스트 환경).
- `GET /api/detections`: `detection_events`와 `detected_objects`를 event_id 기준으로 JOIN하여 최신 탐지 결과를 반환합니다.
- `GET /image/<image_key>`: MinIO 저장 이미지를 Dashboard가 제공하는 경로로 반환합니다.
- 탐지 결과 탭은 3초 주기로 데이터를 갱신합니다.
- MP4를 OpenCV에서 직접 읽으면 MediaMTX에 자동으로 영상이 게시되지 않습니다. 따라서 MP4 탐지 결과 조회 성공과 4채널 실시간 영상 검증은 구분합니다.

### 4.4 NATS와 LLM 연동

기존 Dashboard 코드의 NATS 구독 Subject는 다음과 같습니다.

| Subject | 역할 |
|---|---|
| `cs.vision.control.detected` | Vision 탐지 이벤트 전달 |
| `cs.llm.control.update` | LLM 분석 결과 갱신 |

현재 Dashboard 구독 코드와 NATS 연결은 존재하지만, Vision MVP에는 NATS 발행 코드가 없습니다. LLM 팀의 실제 입력/출력 계약 확인 후 저장 완료 이벤트를 발행하고, LLM 분석 결과를 동일 `event_id`로 연결하는 방식을 권장합니다. DB 저장과 메시지 발행 간 불일치, 중복 이벤트 및 재처리 정책은 통합 단계에서 확인해야 합니다.

**연동용 데이터 예시 (제안, 실제 측정 결과 아님):**

```json
{
  "event_id": "00000000-0000-4000-8000-000000000001",
  "camera_id": "cam01",
  "event_type": "person_detected",
  "occurred_at": "2026-11-25T19:00:00+09:00",
  "object_count": 1,
  "max_confidence": 0.94,
  "image_key": "detections/example.jpg",
  "objects": [{"class_id": 0, "class_name": "person", "confidence": 0.94, "bbox": [120, 80, 340, 460], "track_id": null}]
}
```

LLM 출력 예시: "CAM01에서 사람 1명이 탐지되었습니다. 현재 정보만으로는 이상 행동 여부를 판단할 수 없어 지속적인 모니터링이 필요합니다." 사람 탐지만으로 침입이나 위험 상황이라고 단정하지 않습니다.

## 5. 데이터 저장 구조

![데이터 저장 구조](images/03_storage.png)

| 저장소 / 테이블 | 주요 역할 및 필드 |
|---|---|
| `cameras` | 카메라 관리: camera_id, camera_name, location, mediamtx_path, status (기존 설계) |
| `detection_events` | 이벤트: event_id, camera_id, event_type, occurred_at, object_count, max_confidence, image_key, status |
| `detected_objects` | 객체: object_id, event_id, class_id, class_name, confidence, bbox_x1/y1/x2/y2, track_id |
| `llm_analysis` | 분석: analysis_id, event_id, summary, behavior, risk_level, recommended_action (기존 설계, 실제 DDL 및 연동 확인 필요) |
| MinIO | JPEG 파일, PostgreSQL의 image_key와 연결 |

`camera_id`를 기준으로 카메라별 데이터를 구분하고 `event_id`를 기준으로 이벤트·객체·분석 결과를 연결하는 설계입니다. 실제 FK 제약 및 LLM 필드는 최신 SQL 스키마 확인이 필요합니다.

## 6. 로컬 개발 환경 및 실행 포트

| 서비스 | 주소 / 포트 | 목적 |
|---|---|---|
| IP Camera RTSP | 카메라 IP:554 | 인증 기반 카메라 영상 수신 |
| MediaMTX RTSP | 127.0.0.1:8554 | Vision 영상 입력 |
| MediaMTX WebRTC | 127.0.0.1:8889 | Dashboard 영상 재생 |
| MediaMTX HLS | 127.0.0.1:8888 | 대체 브라우저 영상 재생 |
| PostgreSQL | 127.0.0.1:5432 | 메타데이터 DB |
| MinIO | 127.0.0.1:9000 / 9001 | 객체 API / 관리 콘솔 |
| NATS | 127.0.0.1:4222 | 메시지 통신 |
| Dashboard | http://127.0.0.1:5001 | 웹 관제 UI |
| Main Server (기존 설정) | http://127.0.0.1:8000 | 기존 이벤트 API 연결 대상 (실행 상태 별도 확인) |

위 주소는 같은 PC에서 실행하는 예시입니다. 다른 PC에서 접속하는 경우 MediaMTX 및 서비스의 실제 접근 가능한 주소와 인증·방화벽 정책을 사용해야 합니다. 비밀번호 및 API 키는 README에 포함하지 않습니다.

## 7. 현재 구현 및 테스트 현황

| 기능 | 상태 | 근거 / 남은 작업 |
|---|---|---|
| MediaMTX 서버 | 기동 확인 | Windows v1.18.2 포트 리스너 정상 기동 |
| 실제 IP 카메라 | 부분 확인 | DHCP, ping, TCP 554 통신 확인. RTSP 401 인증 오류 해결 필요 |
| 4채널 Dashboard | UI 구현 | CAM01~04 레이아웃 및 영상 제어, 실제 동시 출력 미검증 |
| YOLO 사람 탐지 | MP4 테스트 성공 | 실시간 실제 카메라 추론 미검증 |
| MinIO / PostgreSQL | 저장·조회 테스트 성공 | MP4에서 생성한 탐지 이미지 및 메타데이터 조회 |
| NATS | 서버 및 Dashboard 연결 확인 | Vision MVP 발행·LLM 구독 통합 필요 |
| Fine-tuned LLM | 팀원 개발 완료 보고 | 실제 모델, 실행 방법, 입출력 확인 필요 |
| 전체 통합 | 진행 예정 | 실제 카메라 → 분석 → Dashboard 자동화 검증 필요 |

## 8. 실행 및 통합 순서

1. Docker Desktop 실행 후 `docker ps`로 PostgreSQL, MinIO, NATS 확인.
2. `mediamtx.yml`에서 카메라 IP·RTSP 계정 설정 후 `mediamtx.exe mediamtx.yml` 실행. `ready` 로그와 WebRTC 영상 확인.
3. `dashboard_app_4cam.py` 실행 후 `http://127.0.0.1:5001` 접속.
4. `vision_to_storage.py`를 MP4 또는 MediaMTX RTSP 입력으로 실행. 최신 로컬 수정본은 `VIDEO_SOURCE`, 기존 배포 ZIP은 `RTSP_URL` 환경변수를 사용하므로 실행 파일에 맞춰 설정.
5. 탐지 결과 탭에서 시간, 카메라, 클래스, 신뢰도 및 이미지 조회 확인.
6. LLM 담당자에게 모델 추론 실행 방법, 입력/출력 스키마, 사용 Subject를 확인. 기존 DB 이벤트 1건으로 추론 및 저장 테스트.
7. NATS 발행/구독 및 Dashboard 분석 결과 표시를 연결하고 통합 테스트.
8. 실제 4대 카메라 동시 재생, 탐지 처리량, 연결 복구, 이벤트 중복/유실, DB 및 LLM 오류 대응을 검증.

## 9. 프로젝트 특징 및 기대 효과

- **확장성:** 카메라별 독립 스트림 경로와 공통 이벤트 스키마를 사용하여 채널 확장을 고려합니다.
- **분리 저장:** 이미지와 구조화된 정보를 분리하여 조회 및 관리에 활용합니다.
- **모듈화:** 영상 수신, 객체 탐지, 데이터 관리, LLM 분석, 웹 UI를 분리하여 팀원 간 병렬 개발을 지원합니다.
- **관제 지원:** 객체 탐지 결과를 자연어로 설명하여 관제자가 상황을 파악하도록 지원하는 것이 목표입니다. 자동 위험 판단 정확도는 아직 검증되지 않았습니다.
- **보안·개인정보:** RTSP 계정 보호, 비밀번호의 GitHub 노출 방지, 인증된 내부망 사용, 영상 접근 권한 및 보관 정책을 고려합니다.

## 10. 보고서·PPT 활용 문구

### 보고서용 시스템 설명

본 프로젝트는 다중 IP 카메라 기반 지능형 영상 관제 시스템의 구현을 목표로 한다. MediaMTX를 이용해 영상 스트림을 관리하고, OpenCV와 YOLO를 활용하여 사람 객체를 탐지한다. 탐지 시 생성된 이미지 데이터는 MinIO에 저장하며, 탐지 시간, 카메라 식별자, 객체 클래스, 신뢰도 및 경계 상자 좌표는 PostgreSQL에 저장한다. Flask 기반 웹 대시보드는 4채널 실시간 관제 UI와 탐지 이력 조회 기능을 제공한다. 또한 NATS 기반 이벤트 전달 구조와 파인튜닝된 LLM을 연계하여 자연어 관제 분석을 제공하는 것을 최종 통합 목표로 한다. 현재 MP4 기반 객체 탐지와 이미지 및 메타데이터 저장·조회가 검증되었으며, 실제 다중 카메라 및 LLM 통합 검증을 진행할 예정이다.

### PPT 구성 추천

| 슬라이드 | 핵심 내용 | 사용 이미지 |
|---|---|---|
| 1 | 프로젝트 배경과 개발 목표 | 01_architecture.png (일부 활용 가능) |
| 2 | 전체 시스템 구성 및 모듈 역할 | 01_architecture.png |
| 3 | 개발 기술 스택 | 04_tech_stack.png |
| 4 | 영상 탐지 및 이벤트 처리 과정 | 02_data_flow.png |
| 5 | DB와 객체 스토리지 설계 | 03_storage.png |
| 6 | 구현 결과 및 테스트 | 실제 Dashboard 탐지 결과 캡처 (개인정보 제거 후 삽입) |
| 7 | LLM 통합 및 향후 검증 | 02_data_flow.png 하단 통합 경로 |

## 11. 확인한 자료 및 문서 범위

- `capstone_detection_mvp.zip` 내부 `vision_to_storage.py`, `db.py`, `storage.py`, README 및 requirements.
- `capstone_dashboard_4cam/dashboard_app_4cam.py` 및 README: 카메라 URL, NATS Subject, 탐지 API 및 이미지 조회.
- 기존 `지능형_영상_관제_시스템_구성도.png`, `다중_카메라_지능형_탐지_시스템_db_구조.png`: 초기 설계 참고. 기존 6채널 표기는 현재 4채널 UI 기준으로 재정리했고 MinIO 저장 경로를 반영함.
- 현재 대화의 MP4 테스트 결과, MediaMTX 로그 및 LLM 개발 완료 보고.
- [LLM fine-tuning 저장소](https://github.com/26-2capstone/fine-tuning): 접근 권한 문제로 실제 소스 검토 불가. 구체적 학습 방식·추론 구현은 확인 전까지 미확인.

이 README와 이미지는 현재 공유된 자료를 정리한 문서이며, 최신 팀 전체 저장소의 모든 소스나 실제 운영 환경을 검증한 결과는 아닙니다.
