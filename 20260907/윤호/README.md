# 윤호 — Docker에서 Linux 커널의 namespace와 cgroup까지

## 탐구 목표

**Docker 컨테이너는 Linux의 어떤 기능으로 구성되며, 격리와 자원 제한은 어떻게 구현되는가?**

Docker의 실행 결과를 관찰하고, namespace와 cgroup의 커널 인터페이스에서 그 설정을 확인합니다. 최종적으로 Docker 설정 → Linux 커널 기능 → 관찰한 동작을 자신의 말로 연결해 설명합니다.

## 첫 회차 범위

- 이미지·컨테이너·프로세스의 관계와 실행 수명주기를 개괄합니다.
- namespace는 종류별 역할을 훑고 **PID namespace 하나**를 중심으로 직접 확인합니다.
- cgroup은 역할과 계층을 이해하고 **CPU 사용량 제한 하나**를 중심으로 직접 확인합니다. 메모리 제한은 추가 탐구로 둡니다.
- namespace 하나와 자원 제한 하나의 증거를 확보하는 데 집중합니다. Docker 없는 재현과 커널 소스 추적은 선택입니다.

## 핵심 질문

1. 이미지와 실행 중인 컨테이너, Linux 프로세스는 어떤 관계인가?
2. 같은 프로세스가 컨테이너 안과 Linux 호스트에서 어떻게 보이는가?
3. namespace는 무엇을 분리하며, 왜 격리와 자원 제한은 별개의 문제인가?
4. cgroup은 프로세스 집합의 자원을 어떻게 관리하며, Docker의 CPU 제한은 어디에 나타나는가?
5. 제한에 도달했을 때 실제 동작과 관측값은 어떻게 달라지는가?
6. 컨테이너의 중지·삭제 시 실행 중인 프로세스와 파일은 각각 어떻게 되는가?

Docker의 Linux 컨테이너는 namespace와 cgroup을 활용합니다. 다만 파일시스템 구성과 보안에는 다른 기능도 관여하므로 두 기능만으로 컨테이너 전체를 설명했다고 보지는 않습니다. [Docker Engine 보안 문서](https://docs.docker.com/engine/security/)

## 먼저 정할 실험 환경

Linux 컨테이너를 대상으로 합니다. Linux 호스트에서 `/proc`와 cgroup 상태를 직접 관찰할 수 있는 환경을 사용합니다. 별도의 Linux VM에 Docker Engine을 설치한 환경도 가능합니다.

Docker Desktop의 Linux 컨테이너는 Linux VM 안에서 실행되므로, 이 문서의 'Linux 호스트'는 컨테이너를 실행하는 그 Linux 환경을 뜻합니다. macOS 터미널의 프로세스 번호와 Linux VM 안의 PID를 그대로 비교하지 않습니다. Desktop을 사용한다면 어느 관찰 지점에 접근할 수 있는지 먼저 확인합니다. [Docker Desktop 실행 환경](https://docs.docker.com/desktop/features/networking/)

기록할 항목:

- 호스트 OS와 Linux 커널 버전, Docker Engine / Desktop 여부
- Docker 버전과 context, rootless 여부
- cgroup v1/v2 및 cgroup driver, 실행 이미지와 명령어
- CPU 개수, 컨테이너에 설정한 제한, 관측 위치

`docker info`로 환경을 확인합니다. 아래 cgroup 파일명은 v2 기준이며, v1 환경에서는 해당 버전의 문서로 조정합니다. Docker 설정에 따라 실제 cgroup 경로도 달라지므로 고정 경로를 추측하지 않습니다. [Docker Runtime metrics](https://docs.docker.com/engine/containers/runmetrics/)

## 탐구 흐름

| 단계 | 할 일 | 남길 증거 |
|---|---|---|
| 예상 | 컨테이너 안팎의 PID와 CPU 제한 결과를 예상 | 초기 설명 |
| 개념 | 프로세스·namespace·cgroup을 자료와 AI로 학습 | 자신의 말로 쓴 역할 구분 |
| 관찰 | Docker로 실행하고 커널 인터페이스를 확인 | PID·namespace 식별자·cgroup 경로와 값 |
| 검증 | CPU 부하를 제한 전후로 비교 | 실행 조건·측정 시간·관측값 |
| 설명 | 설정과 커널 기능, 결과를 연결 | 수정한 설명과 남은 질문 |

## 실험 1 — PID namespace와 프로세스

1. 단순하게 계속 실행되는 프로세스를 가진 실험용 컨테이너를 실행합니다.
2. 컨테이너 내부에서 프로세스 목록을 확인하고, `docker inspect`의 `State.Pid`로 Linux 호스트 관점의 컨테이너 초기 프로세스 PID를 찾습니다.
3. Linux 호스트의 `/proc/<PID>/status`에서 `NSpid`를 살펴보고, 내부에서 관찰한 PID와 관계를 설명합니다. 관측 위치에 따라 보이는 계층이 달라질 수 있습니다.
4. `/proc/<PID>/ns/pid`와 호스트 셸의 `/proc/self/ns/pid` 링크를 비교합니다. 같은 호스트 커널 안에서 namespace 식별자가 같은지 다른지 확인합니다.
5. 컨테이너 초기 프로세스를 종료하거나 컨테이너를 중지하고 실행 상태 변화를 관찰합니다.

PID namespace가 프로세스 ID의 관점을 분리한다는 설명을 실제 출력으로 뒷받침합니다. `/proc/<PID>/ns/`의 다른 항목은 목록을 살펴보되 모두 실험할 필요는 없습니다. [Linux namespaces(7)](https://man7.org/linux/man-pages/man7/namespaces.7.html)

## 실험 2 — cgroup과 CPU 제한

1. CPU 부하를 만드는 작은 작업을 실험용 컨테이너에서 정해진 시간 동안 실행합니다. 예를 들어 단일 작업을 10~20초 관찰하며, 같은 작업을 제한 전후에 비교합니다.
2. Docker의 CPU 제한 옵션(예: `--cpus=0.5`)을 설정하고 어떤 결과가 나올지 예상합니다.
3. Linux 호스트에서 컨테이너 초기 프로세스의 `/proc/<PID>/cgroup`을 읽고, 해당 호스트의 cgroup 마운트와 연결해 실제 디렉터리를 찾습니다.
4. v2 환경에서는 `cpu.max`의 quota와 period, `cpu.stat`의 사용량과 throttling 관련 값의 변화를 확인합니다. `docker stats`는 보조 관측으로 사용합니다.
5. 옵션 값만 확인하는 데서 끝내지 않고, 제한 전후의 관측 차이가 설정과 어떻게 연결되는지 설명합니다.

CPU 사용 비율은 측정 구간과 부하, 상위 cgroup의 제한에도 영향을 받을 수 있습니다. 순간 수치가 특정 값과 정확히 같아야 한다고 가정하지 않습니다. CPU 대역폭 제한과 특정 CPU에 배치하는 것은 별개의 질문으로 구분합니다. [Linux cgroup v2 문서](https://docs.kernel.org/admin-guide/cgroup-v2.html)

실험은 종료 시간이 정해진 작은 부하로 수행하고, 관찰한 호스트의 기존 cgroup 설정을 직접 바꾸지 않습니다.

## 추가 탐구 — 필요한 것만 선택

### 기존 Docker 수명주기와 파일 실험

- 같은 이미지로 만든 두 컨테이너에서 파일 변경이 공유되는지 확인합니다.
- 중지 후 재시작과 삭제 후 새로 생성할 때 파일 상태가 어떻게 다른지 확인합니다.
- 처음에는 별도 볼륨·바인드 마운트 없이 확인하고, 이후 볼륨 사용 시와 비교합니다. 이미지의 저장 관련 설정도 기록합니다.

### namespace 확장과 Docker 없는 재현

Mount·UTS·Network·User namespace 중 하나를 선택해 다루는 대상을 확인합니다. Linux 실험 환경에서 `unshare`나 `nsenter`로 작은 재현을 시도할 수 있습니다. `unshare` 실험 하나가 Docker 전체를 구현한 것과 같지는 않습니다. 권한이나 rootless 설정 때문에 확인하지 못했다면 그 조건을 기록합니다.

### 메모리 제한

cgroup v2의 `memory.max`, `memory.current`, `memory.events`를 살펴봅니다. 메모리 압력·회수·OOM과 프로세스 종료를 구분해 질문을 세웁니다. 재현한다면 작은 메모리 제한이 걸린 실험용 컨테이너 안에서만 수행하고 관련 이벤트와 종료 상태를 함께 확인합니다.

### 커널 소스 추적

실험한 Linux 커널 버전에 맞는 소스에서 PID namespace 또는 cgroup CPU 제어의 관심 경로 하나를 따라갑니다. 사용자 인터페이스와 내부 자료구조·함수의 관계를 적으며, 소스 전체를 읽는 것을 목표로 하지 않습니다. 실행 커널과 읽은 소스 버전이 다르면 차이를 표시합니다.

## 완료 기준

- namespace의 환경 분리와 cgroup의 자원 관리를 구분해 설명합니다.
- PID namespace 하나와 CPU 제한 하나에 대해 Docker 설정·커널 인터페이스·관찰 결과를 연결한 기록을 남깁니다.
- 환경·명령어·예상·실제 결과를 남겨 재확인할 수 있게 합니다.
- 아직 검증하지 못한 설명과 다음 질문을 구분합니다.

이 문서는 탐구 계획입니다. 실제 실험 결과는 아래에 학습 후 작성합니다. AI는 개념 설명과 실험 준비에 활용할 수 있으며, 핵심 내용은 공식 자료와 관찰로 확인합니다.

## 개인 기록 — 학습 후 작성

### 내가 처음 생각했던 Docker

### 핵심 개념을 내 말로 설명

### 실험 환경과 명령어

### 실험별 예상 → 결과 → 이유

### 처음과 달라진 생각

### 아직 설명하지 못하는 질문

### 참고 자료와 AI 도움을 받은 부분
