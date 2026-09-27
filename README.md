# LWIR 모델 개발 환경

학습 환경과 DX Compile 환경을 서로 다른 Docker 이미지로 관리합니다. 이 문서는
환경 구성 방법만 설명하며 학습 및 경량화 방법은 다루지 않습니다.

## 먼저 읽기

Docker 환경은 두 개이며 필요한 환경만 선택해서 사용합니다.

| 목적 | Compose 서비스 | 처음 한 번 | 이후 실행 |
| --- | --- | --- | --- |
| 모델 학습 | `training` | 이미지 빌드 필요 | 빌드된 이미지 바로 실행 |
| DEEPX 모델 변환 | `dxcompile` | DEEPX 베이스와 프로젝트 이미지 빌드 필요 | 빌드된 이미지 바로 실행 |

아래 Quick Start는 명령을 위에서 아래로 실행할 수 있도록 **최초 빌드**와
**빌드 후 실행**을 분리한 것입니다. 각 명령의 자세한 설명은 뒤의 상세 설명에
있습니다.

## 공통 사전 확인

처음 시작할 때 서버 또는 PC에서 다음 명령을 실행합니다.

```bash
git --version
docker --version
docker compose version
```

학습 Docker를 사용할 서버에서는 GPU도 확인합니다.

```bash
nvidia-smi
```

DX Compile Docker를 사용할 PC에서는 CPU 아키텍처를 확인합니다.

```bash
uname -m
```

DX Compiler를 사용하려면 `x86_64`가 출력되어야 합니다. 명령을 찾을 수 없거나
Docker 권한 오류가 발생하면 다음 단계로 진행하지 말고 서버 관리자에게 Git,
Docker, Docker Compose 또는 NVIDIA Container Toolkit 설치를 요청합니다.

## 학습 Docker Quick Start

### 처음 한 번: 빌드부터 실행까지

```bash
git clone https://github.com/lwir-deepx-tracking-system/model-development.git
cd model-development
cp .env.example .env
```

`.env`를 열고 `TRAIN_BASE_IMAGE`의 `3.10` 부분을 팀원이 사용하는 Python
major/minor 버전으로 변경합니다. Python 3.10을 사용하면 수정하지 않아도 됩니다.

```env
TRAIN_BASE_IMAGE=python:3.10-slim-bookworm
```

이미지를 빌드하고 라이브러리와 GPU를 확인합니다.

```bash
docker compose build training

docker compose run --rm training python -c \
  'import torch, torchvision, ultralytics; print(torch.__version__); print(torchvision.__version__); print(ultralytics.__version__); print(torch.version.cuda)'

docker compose run --rm training python -c \
  'import torch; print(torch.cuda.is_available()); print(torch.cuda.get_device_name(0) if torch.cuda.is_available() else "GPU unavailable")'
```

검증이 끝나면 학습 컨테이너에 들어갑니다.

```bash
docker compose run --rm training
```

컨테이너에서 나오려면 `exit`를 실행합니다.

### 빌드 후: 평소에 실행만 할 때

```bash
cd model-development
docker compose run --rm training
```

코드만 변경한 경우에는 이미지를 다시 빌드하지 않아도 됩니다.

## DX Compile Docker Quick Start

### 처음 한 번: 빌드부터 실행까지

프로젝트와 DX-AllSuite를 같은 상위 디렉터리에 받습니다. 위에서 프로젝트를 이미
받았다면 첫 두 명령은 건너뜁니다.

```bash
git clone https://github.com/lwir-deepx-tracking-system/model-development.git
cd model-development
cp .env.example .env

cd ..
git clone https://github.com/DEEPX-AI/dx-all-suite.git
cd dx-all-suite
git checkout 5749ab70cc75cede356c57b9cc7cf91229e47742
git submodule update --init --recursive
cat release.ver
```

`cat release.ver` 결과가 `v2.4.3`인지 확인한 후 DEEPX 베이스 이미지를 만듭니다.

```bash
./docker_build.sh --target=dx-compiler --ubuntu_version=22.04
docker tag dx-compiler:ubuntu-22.04 \
  dx-compiler:2.4.2-dxcom-2.4.1-ubuntu-22.04

docker run --rm --entrypoint bash \
  dx-compiler:2.4.2-dxcom-2.4.1-ubuntu-22.04 \
  -lc 'source /venv-dxnn/bin/activate && dxcom --version'
```

`DX-COM (DEEPX Compiler) 2.4.1`이 출력되면 프로젝트용 이미지를 만듭니다.

```bash
cd ../model-development
docker compose build dxcompile
docker compose run --rm dxcompile dxcom --version
```

검증이 끝나면 DX Compile 컨테이너에 들어갑니다.

```bash
docker compose run --rm dxcompile
```

컨테이너에서 나오려면 `exit`를 실행합니다.

### 빌드 후: 평소에 실행만 할 때

```bash
cd model-development
docker compose run --rm dxcompile
```

## 언제 다시 빌드하나?

| 변경 내용 | 실행할 명령 |
| --- | --- |
| 학습 또는 변환 코드만 변경 | 다시 빌드하지 않음 |
| `requirements/training.txt` 또는 학습 Dockerfile 변경 | `docker compose build training` |
| `requirements/dxcompile.txt` 또는 DX Compile Dockerfile 변경 | `docker compose build dxcompile` |
| 고정된 DXAllSuite/DX-COM 버전 변경 | DEEPX 베이스 이미지부터 다시 빌드 |

이하 내용은 위 Quick Start에 대한 상세 설명입니다.

## 1. 사전 준비

DX Compiler는 Linux `x86_64` 환경에서 사용합니다. 먼저 Git, Docker와 Docker
Compose가 설치되어 있는지 확인합니다.

```bash
git --version
docker --version
docker compose version
uname -m
```

`uname -m` 결과는 `x86_64`여야 합니다. Docker 명령에서 권한 오류가 발생하면
서버 관리자에게 Docker 사용 권한을 요청합니다.

## 2. 이 저장소 받기

작업할 디렉터리에서 이 저장소를 내려받습니다.

```bash
git clone https://github.com/lwir-deepx-tracking-system/model-development.git
cd model-development
```

이미 저장소를 받은 경우에는 이 단계를 건너뜁니다.

## 3. DEEPX 공식 저장소 받기

DX Compiler 베이스 이미지는 DEEPX 공식 DX-AllSuite 저장소에서 만듭니다. 현재
프로젝트는 아래 버전으로 고정되어 있습니다.

```text
DX-AllSuite: v2.4.3
DX-AllSuite commit: 5749ab70cc75cede356c57b9cc7cf91229e47742
DX-Compiler: v2.4.2
DX-COM: v2.4.1
```

프로젝트의 상위 디렉터리로 이동한 뒤 저장소를 내려받고, 지정된 commit으로
이동한 다음 서브모듈을 받습니다.

```bash
cd ..
git clone https://github.com/DEEPX-AI/dx-all-suite.git
cd dx-all-suite
git checkout 5749ab70cc75cede356c57b9cc7cf91229e47742
git submodule update --init --recursive
```

이미 DX-AllSuite를 받은 경우에는 다음 명령으로 고정 commit과 서브모듈 상태를
맞춥니다. `git checkout` 후 detached HEAD 안내가 나오는 것은 정상입니다.

```bash
git fetch origin
git checkout 5749ab70cc75cede356c57b9cc7cf91229e47742
git submodule update --init --recursive
```

버전 파일을 확인합니다.

```bash
cat release.ver
```

다음 값이 출력되어야 합니다.

```text
v2.4.3
```

## 4. DX Compiler 베이스 이미지 만들기

DX-AllSuite 디렉터리에서 Ubuntu 22.04용 DX Compiler 이미지를 빌드합니다.

```bash
./docker_build.sh --target=dx-compiler --ubuntu_version=22.04
docker tag dx-compiler:ubuntu-22.04 \
  dx-compiler:2.4.2-dxcom-2.4.1-ubuntu-22.04
```

빌드 과정에서 DX Compiler 패키지를 내려받으므로 네트워크 연결이 필요합니다.
완료되면 다음 명령으로 이미지가 생성됐는지 확인합니다.

```bash
docker image ls dx-compiler:2.4.2-dxcom-2.4.1-ubuntu-22.04
```

목록에 `dx-compiler`와 `2.4.2-dxcom-2.4.1-ubuntu-22.04`가 표시되면 정상입니다.

## 5. DX Compiler 버전 확인하기

생성한 베이스 이미지에서 DX-COM을 직접 실행합니다.

```bash
docker run --rm --entrypoint bash \
  dx-compiler:2.4.2-dxcom-2.4.1-ubuntu-22.04 \
  -lc 'source /venv-dxnn/bin/activate && dxcom --version'
```

현재 프로젝트에서 확인한 버전은 다음과 같습니다.

```text
DX-COM (DEEPX Compiler) 2.4.1
Target Hardware: M1
```

다른 버전이 표시되면 고정 commit과 Docker 이미지 태그를 다시 확인합니다.

## 6. 프로젝트용 DX Compile 이미지 만들기

이제 프로젝트 디렉터리로 돌아갑니다. 아래 경로는 2단계와 3단계에서 두 저장소를
같은 상위 디렉터리에 받은 경우를 기준으로 합니다.

```bash
cd ../model-development
```

프로젝트에서 추가로 필요한 Python 라이브러리는 다음 파일에 버전과 함께
작성합니다.

```text
requirements/dxcompile.txt
```

DX-COM 자체는 베이스 이미지에 이미 설치되어 있으므로 이 파일에 추가하지
않습니다. 추가 라이브러리가 아직 없다면 파일을 수정하지 않아도 됩니다.

프로젝트용 이미지를 빌드합니다.

```bash
docker compose build dxcompile
```

이 명령은 다음 두 환경을 합쳐 `lwir-dxcompile:local` 이미지를 만듭니다.

```text
dx-compiler:2.4.2-dxcom-2.4.1-ubuntu-22.04
        +
requirements/dxcompile.txt
        =
lwir-dxcompile:local
```

## 7. 프로젝트용 이미지 검증하기

Compose를 통해 DX-COM 버전을 확인합니다.

```bash
docker compose run --rm dxcompile dxcom --version
```

다음 내용이 표시되면 컴파일러 환경 구성이 완료된 것입니다.

```text
DX-COM (DEEPX Compiler) 2.4.1
Target Hardware: M1
```

## 8. DX Compiler 컨테이너 들어가기

컴파일러 컨테이너의 터미널을 열려면 다음 명령을 사용합니다.

```bash
docker compose run --rm dxcompile
```

현재 저장소는 컨테이너의 `/deepx/workspace`에 연결됩니다. 컨테이너에서 나가려면
다음을 실행합니다.

```bash
exit
```

`--rm` 옵션을 사용하므로 종료된 임시 컨테이너는 자동으로 삭제됩니다. 빌드한
이미지와 프로젝트 파일은 삭제되지 않습니다.

## 9. 추가 라이브러리 변경하기

`requirements/dxcompile.txt`를 수정한 후 프로젝트용 이미지만 다시 빌드합니다.
DEEPX 베이스 이미지는 다시 만들 필요가 없습니다.

```bash
docker compose build dxcompile
docker compose run --rm dxcompile dxcom --version
```

캐시를 사용하지 않고 처음부터 다시 설치하려면 다음 명령을 사용합니다.

```bash
docker compose build --no-cache dxcompile
```

## 10. 이미지 설정 변경하기

기본 설정은 다음과 같습니다.

```text
DEEPX 베이스 이미지: dx-compiler:2.4.2-dxcom-2.4.1-ubuntu-22.04
프로젝트 최종 이미지: lwir-dxcompile:local
```

프로젝트 루트의 `.env`에서 이미지 이름을 바꿀 수 있습니다. 처음 설정할 때
`.env.example`을 복사했다면 아래 값이 이미 들어 있습니다.

```env
DX_COMPILER_BASE_IMAGE=dx-compiler:2.4.2-dxcom-2.4.1-ubuntu-22.04
DX_COMPILE_IMAGE=lwir-dxcompile:local
```

프로젝트용 이미지 빌드 중 DX-COM 버전이 2.4.1이 아니면 빌드가 실패합니다. 버전을
업데이트할 때는 DX-AllSuite commit, 베이스 이미지 태그, `compose.yaml` 및
`docker/Dockerfile.dxcompile`을 함께 변경하고 다시 검증합니다.

## 11. 학습 Docker 환경

학습 Docker 환경은 DX Compiler와 독립되어 있습니다. 학습용 Python 라이브러리는
`requirements/training.txt`에 버전이 고정되어 있으며, PyTorch 패키지는 CUDA 11.8
wheel을 사용합니다.

팀원은 학습 서버에서 사용하는 Python의 major/minor 버전을 확인합니다.

```bash
python --version
```

프로젝트 루트에서 `.env.example`을 `.env`로 복사하고 아래 `<PYTHON_VERSION>`
부분만 확인한 버전으로 바꿉니다.

```bash
cp .env.example .env
```

예를 들어 Python 3.10을 사용하면 다음과 같이 작성합니다.

```env
TRAIN_BASE_IMAGE=python:3.10-slim-bookworm
```

일반 형식은 다음과 같습니다.

```env
TRAIN_BASE_IMAGE=python:<PYTHON_VERSION>-slim-bookworm
```

`.env`는 로컬 및 서버별 설정이므로 Git에 포함하지 않습니다. 값을 설정하지 않으면
`compose.yaml`의 임시 기본값인 `python:3.10-slim-bookworm`을 사용합니다.

학습 이미지를 빌드하고 실행하는 명령은 다음과 같습니다.

```bash
docker compose build training
docker compose run --rm training
```

설치된 핵심 라이브러리 버전은 다음 명령으로 확인합니다.

```bash
docker compose run --rm training python -c \
  'import torch, torchvision, ultralytics; print(torch.__version__); print(torchvision.__version__); print(ultralytics.__version__); print(torch.version.cuda)'
```

학습 서비스에만 `gpus: all`이 설정되어 있습니다. 학습 서버에서 GPU를 연결하려면
NVIDIA Container Toolkit이 설치되어 있어야 합니다. 이미지 빌드가 완료되면 학습
서버에서 다음 명령으로 컨테이너가 GPU를 인식하는지 확인합니다.

```bash
docker compose run --rm training python -c \
  'import torch; print(torch.cuda.is_available()); print(torch.cuda.get_device_name(0) if torch.cuda.is_available() else "GPU unavailable")'
```
