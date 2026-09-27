# LWIR 모델 개발 환경

학습 환경과 DX Compile 환경을 서로 다른 Docker 이미지로 관리합니다. 이 문서는
환경 구성 방법만 설명하며 학습 및 경량화 방법은 다루지 않습니다.

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
프로젝트의 상위 디렉터리로 이동한 뒤 서브모듈을 포함해 내려받습니다.

```bash
cd ..
git clone --recurse-submodules https://github.com/DEEPX-AI/dx-all-suite.git
cd dx-all-suite
```

이미 DX-AllSuite를 받은 경우에는 다음 명령으로 서브모듈 상태를 맞춥니다.

```bash
git submodule update --init --recursive
```

## 4. DX Compiler 베이스 이미지 만들기

DX-AllSuite 디렉터리에서 Ubuntu 22.04용 DX Compiler 이미지를 빌드합니다.

```bash
./docker_build.sh --target=dx-compiler --ubuntu_version=22.04
```

빌드 과정에서 DX Compiler 패키지를 내려받으므로 네트워크 연결이 필요합니다.
완료되면 다음 명령으로 이미지가 생성됐는지 확인합니다.

```bash
docker image ls dx-compiler:ubuntu-22.04
```

목록에 `dx-compiler`와 `ubuntu-22.04`가 표시되면 정상입니다.

## 5. DX Compiler 버전 확인하기

생성한 베이스 이미지에서 DX-COM을 직접 실행합니다.

```bash
docker run --rm --entrypoint bash dx-compiler:ubuntu-22.04 \
  -lc 'source /venv-dxnn/bin/activate && dxcom --version'
```

현재 프로젝트에서 확인한 버전은 다음과 같습니다.

```text
DX-COM (DEEPX Compiler) 2.4.1
Target Hardware: M1
```

다른 버전이 표시되면 DEEPX 공식 저장소가 업데이트됐을 가능성이 있으므로 팀에
확인한 뒤 진행합니다.

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
dx-compiler:ubuntu-22.04
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
DEEPX 베이스 이미지: dx-compiler:ubuntu-22.04
프로젝트 최종 이미지: lwir-dxcompile:local
```

이미지 이름을 바꿔야 할 때만 프로젝트 루트에 `.env`를 만들고 값을 지정합니다.

```env
DX_COMPILER_BASE_IMAGE=dx-compiler:ubuntu-22.04
DX_COMPILE_IMAGE=lwir-dxcompile:local
```

## 11. 학습 Docker 환경

학습 Docker 환경은 DX Compiler와 독립되어 있습니다. 현재
`python:3.10-slim-bookworm`은 학습 환경이 확정되기 전의 임시 기본값입니다.
학습용 베이스 이미지와 라이브러리 버전이 정해지면 다음 파일을 수정합니다.

```text
.env
requirements/training.txt
```

`.env` 작성 예시는 다음과 같습니다.

```env
TRAIN_BASE_IMAGE=<학습용 베이스 이미지>:<버전>
```

학습 이미지를 빌드하고 실행하는 명령은 다음과 같습니다.

```bash
docker compose build training
docker compose run --rm training
```

학습 서비스에만 `gpus: all`이 설정되어 있습니다. 학습 서버에서 GPU를 연결하려면
NVIDIA Container Toolkit이 설치되어 있어야 합니다.

