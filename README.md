# CloudPulse

기획서에 따른 클라우드 인프라 모니터링 및 장애 관리 웹 서비스입니다.
FastAPI와 psutil이 실제 실행 호스트의 CPU/메모리/디스크/가동 시간을 수집하고,
독립 프로세스인 테스트 서비스의 HTTP 응답을 검사합니다. 실제 장애와 복구를 SQLite에 저장합니다.

## 로컬 실행

Python 3.12 이상에서 가상환경을 만들고 다음 명령을 각각 실행합니다.

```bash
python -m venv .venv
# Linux: source .venv/bin/activate
# Windows: .venv\Scripts\Activate.ps1
pip install -r requirements.txt
uvicorn demo:app --host 127.0.0.1 --port 9001
uvicorn app:app --host 127.0.0.1 --port 8088 --workers 1
```

브라우저에서 http://127.0.0.1:8088 에 접속합니다.
기본 검사 주기는 10초이며 화면은 5초마다 최신 저장값을 읽습니다.
로컬 지표를 AWS 지표로 표시하지 않습니다. AWS 구성은 배포 때 확인한 값으로 채웁니다.

## AWS 배포

서울 리전의 인증된 AWS CloudShell에서 프로젝트를 압축 해제합니다.
관리 접속을 허용할 단일 공인 IPv4를 `ADMIN_CIDR`에 지정하고 실행합니다.

```bash
export ADMIN_CIDR='승인한_관리자_공인IPv4/32'
python3 deploy/provision.py
```

VPC 10.10.0.0/16, 퍼블릭 서브넷 10.10.1.0/24, 인터넷 게이트웨이,
라우트 테이블, 보안 그룹과 Ubuntu 24.04 t3.micro 서버 1대를 생성합니다.
EBS는 암호화한 gp3 8 GiB이며 CPU 크레딧은 standard입니다.
Nginx :80이 정적 웹과 API를 제공하고, 내부 API :8000 및 테스트 서비스 :9001은
127.0.0.1에만 바인딩합니다. SSH는 지정한 /32에서만 네트워크 접근을 허용합니다.
이 스크립트는 SSH 키나 AWS IAM 권한을 생성하지 않습니다.
원격 관리는 기존 권한을 통한 EC2 Instance Connect 등을 사용할 수 있습니다.
HTTPS는 도메인/인증서가 필요한 선택 확장 항목입니다.

진행 상태는 `deployment.json`에 즉시 저장합니다. 동일한 폴더에서 재실행하면
기록된 자원을 재사용하며 서버가 이미 생성됐다면 추가 생성하지 않습니다.
배포 후 EC2/EBS/공인 IPv4 등 사용 요금이 발생할 수 있으므로 실행 기간을 관리합니다.

## 실제 장애/복구 시연

관리 접속한 서버에서 다음 명령으로 테스트 서비스만 중지합니다.

```bash
sudo systemctl stop cloudpulse-demo
# 10초 검사 주기와 화면 갱신을 기다린 후 장애 화면 캡처
sudo systemctl start cloudpulse-demo
# 다음 검사 이후 복구 화면과 Incidents 기록 캡처
```

대시보드 프로세스와 수집기는 계속 동작합니다. 연속 실패는 장애 한 건으로 기록됩니다.
백엔드를 재시작해도 SQLite 데이터는 유지됩니다. 수집기와 대상 서비스가 같은 EC2에 있어
EC2 전체 장애는 자체 탐지하지 못하며, 검사 주기로 인해 실제 장애 시각과 감지 시각이 다를 수 있습니다.

## 보안 검증 기록

`data/security.json`의 `tests` 배열에는 실제 실행한 검증만 기록합니다.
각 항목은 `name`, `expected`, `observed`, `passed`, `ts`를 포함합니다.
빈 배열은 검증 전 상태이며, 설정 화면을 근거로 외부 접근 검증 성공을 추정하지 않습니다.
공개 웹 API는 읽기 전용이며 임의 URL 등록이나 서비스 중지 기능을 제공하지 않습니다.

## 구성

- `app.py`: 수집기, API, SQLite 장애 상태 전이
- `demo.py`: 독립 테스트 서비스
- `static/`: 한국어 다크 대시보드와 6개 화면
- `deploy/install.sh`: Ubuntu / Nginx / systemd 구성
- `deploy/provision.py`: AWS 리소스 및 인스턴스 부팅 배포

운영 데이터는 기본 `data/` 폴더에 저장하며 `CLOUDPULSE_DATA`로 변경할 수 있습니다.
단일 수집기를 유지하기 위해 Uvicorn은 반드시 `--workers 1`로 실행합니다.
지표는 7일을 보관하며 화면은 최근 120개 측정값과 최신 100개 장애를 표시합니다.
