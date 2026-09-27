# 프로젝트 컨텍스트

이 저장소는 LWIR(Long-Wave Infrared) 객체 탐지·추적 시스템에 사용할 AI 모델의
개발 workspace다. YOLO 계열을 포함한 모델을 LWIR 데이터에 맞게 fine-tuning하고,
경량화·양자화·변환하여 DEEPX NPU에서 실행 가능한 결과물을 만드는 것이 목적이다.

## 환경 구분

- `training`은 데이터 전처리, 학습, 평가, checkpoint 생성 및 ONNX export를 담당한다.
- `dxcompile`은 모델 경량화, 양자화, DEEPX DX Compiler 변환 및 NPU 배포 검증을 담당한다.
- 두 환경의 Dockerfile과 Python 의존성은 분리해서 관리한다.
- 학습 의존성은 `requirements/training.txt`에만, DX Compile용 추가 의존성은
  `requirements/dxcompile.txt`에만 추가한다.
- DX-COM은 DEEPX 베이스 이미지에 포함되어 있으므로 Python requirements에 추가하지
  않는다.
- DX 환경은 DX-AllSuite v2.4.3 commit
  `5749ab70cc75cede356c57b9cc7cf91229e47742`, DX-Compiler v2.4.2 및 DX-COM
  v2.4.1로 고정한다. 버전 변경은 이 값들을 함께 갱신하고 검증하는 별도 작업이다.
- 현재 학습 이미지의 기본값 `python:3.10-slim-bookworm`은 임시값이다. 확정된 CUDA,
  PyTorch 및 GPU 요구사항을 확인하기 전에는 이를 최종 학습 환경으로 간주하지 않는다.

## 주요 경로와 명령

- Compose 설정: `compose.yaml`
- 학습 Dockerfile: `docker/Dockerfile.training`
- DX Compile Dockerfile: `docker/Dockerfile.dxcompile`
- 학습 코드와 설정: `training/`
- 경량화·변환 코드와 설정: `optimization/`
- 학습 이미지 빌드: `docker compose build training`
- 학습 컨테이너 실행: `docker compose run --rm training`
- DX Compile 이미지 빌드: `docker compose build dxcompile`
- DX Compile 컨테이너 실행: `docker compose run --rm dxcompile`
- 컴파일러 확인: `docker compose run --rm dxcompile dxcom --version`

환경 설치 및 Docker 사용법의 현재 기준은 `README.md`를 따른다.

## 기본 모델 파이프라인

LWIR 데이터 준비 -> 학습 및 검증 -> 모델 export -> 경량화·양자화 -> DEEPX 변환
-> 정확도 회귀 검증 -> DEEPX NPU 성능 검증 순서로 작업한다.

모델을 환경 사이에 전달할 때 다음 계약을 명시하고 일관되게 유지한다.

- 입력 tensor의 shape, dtype, 채널 순서 및 동적 batch 여부
- resize, letterbox, 정규화 등 전처리 방식
- 출력 tensor의 이름, shape 및 의미
- confidence threshold, NMS 등 후처리 방식
- class 목록과 class index 매핑
- opset, checkpoint 및 export 설정

## 작업 원칙

- 학습과 DX Compile 환경의 의존성을 서로 섞지 않는다.
- 버전은 가능한 한 명시적으로 고정하고, 변경 이유와 호환성 영향을 함께 기록한다.
- 전처리와 후처리 로직을 학습 평가, export 검증 및 NPU 추론에서 동일하게 유지한다.
- 최적화 전후의 정확도를 동일한 데이터셋과 지표로 비교한다.
- 성능 수치는 입력 크기, batch size, precision, warm-up, 반복 횟수 및 대상 하드웨어와
  함께 기록한다.
- 데이터셋, checkpoint, 변환 모델 및 대용량 산출물을 Git에 추가하기 전에 저장 정책과
  `.gitignore`를 확인한다.
- 사용자 데이터, checkpoint 또는 실험 결과를 임의로 삭제하거나 덮어쓰지 않는다.
- DEEPX 도구의 명령과 옵션은 고정된 버전에서 확인한다. 현재 기준은 DX-COM 2.4.1,
  대상 하드웨어 M1이며, 다른 버전을 임의로 사용하지 않는다.
- 기존 사용자의 변경 사항과 무관한 파일은 수정하지 않는다.

## 변경 및 검증

- 변경 전 관련 코드, 설정 및 문서를 먼저 확인한다.
- 변경 후에는 범위에 맞는 최소 검증을 실행하고 사용한 명령과 결과를 보고한다.
- Docker 설정 변경 시 가능하면 `docker compose config`로 구성을 검증한다.
- 학습 코드 변경 시 작은 샘플 또는 smoke test를 우선 사용하고, 장시간 학습은 명시적
  요청 없이 시작하지 않는다.
- export 또는 최적화 코드 변경 시 원본 프레임워크 출력과 export/변환 모델 출력을
  수치적으로 비교한다.
- 새로운 실행 절차, 환경 변수 또는 결과물 경로를 추가하면 관련 README도 갱신한다.

## 커뮤니케이션

- 사용자 응답과 프로젝트 문서는 특별한 요청이 없으면 한국어로 작성한다.
- 파일명, 코드 식별자, CLI 명령, 라이브러리 및 모델의 공식 명칭은 원문을 유지한다.
- 확정되지 않은 모델 구조, 데이터셋 경로, GPU/CUDA 버전 또는 DEEPX 옵션을 임의로
  가정하지 말고, 확인 가능한 저장소 정보가 없으면 그 가정을 명시한다.
