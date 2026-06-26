<!-- @format -->

# AI SW Basic - SRE Failure Analysis Report

# 1. 개발 환경

| 항목         | 내용                     |
| ------------ | ------------------------ |
| OS           | macOS (Apple Silicon M3) |
| Runtime      | Docker Ubuntu 22.04      |
| Architecture | linux/arm64              |

---

# 2. 미션 목표

본 미션은 Agent 서비스를 실행한 뒤 발생하는 장애(OOM, CPU Spike, Deadlock)를 관찰하고 원인을 분석한 후 환경변수 변경을 통해 장애를 재현 및 완화하는 것을 목표로 한다.

---

# 3. 실행 환경

## Docker 실행

```bash
docker run --rm -it \
  --platform linux/arm64 \
  -v "$(pwd)":/mission \
  -w /mission \
  ubuntu:22.04 bash
```

## 실행 환경 구성

```bash
useradd -m agent-admin
su - agent-admin

export AGENT_HOME=/mission
export AGENT_PORT=15034
export MEMORY_LIMIT=256
export CPU_MAX_OCCUPY=70
export MULTI_THREAD_ENABLE=true
```

필수 디렉터리 생성

```bash
mkdir -p upload_files api_keys logs
echo "agent_api_key_test" > api_keys/secret.key
```

---

# 4. Boot Sequence

## 현상

초기 실행 시 Root 계정에서는 프로그램이 실행되지 않았다.

```
Running as 'root' is forbidden.
```

## 원인

운영 환경에서는 Root 계정으로 서비스 실행을 제한하기 때문이다.

## 조치

agent-admin 계정을 생성하여 실행하였다.

## 결과

```
All Boot Checks Passed!

Agent READY
```

---

# 5. Failure Report #1 - OOM

## 현상

프로세스 실행 후 Heap Memory가 지속적으로 증가하였다.

```
25MB
50MB
75MB
100MB
125MB
150MB
175MB
200MB
225MB
250MB
275MB
```

이후

```
[CRITICAL] Memory limit exceeded

SELF-TERMINATED
```

메시지와 함께 프로세스가 종료되었다.

## 증거

- Boot Log
- Memory 증가 로그
- OOM 종료 메시지
- Screenshot : 03_OOM_Crash.png

## 원인

MEMORY_LIMIT=256MB로 설정되어 있었으며 Heap Memory가 제한값을 초과하였다.

MemoryGuard가 프로세스를 보호하기 위해 프로세스를 강제 종료하였다.

## 조치

```
export MEMORY_LIMIT=512
```

또는

```
export MEMORY_LIMIT=1024
```

로 변경하여 더 많은 메모리를 사용할 수 있도록 설정하였다.

※ 이번 실습에서는 환경변수 변경은 수행하였으나 증가된 생존 시간을 정량적으로 측정하지는 않았다.

---

# 6. Failure Report #2 - CPU Spike

## 현상

초기 설정

```
CPU_MAX_OCCUPY=70%
```

실행 결과

```
WARNING

Recommend Under 50%
```

CPU 사용률이 권장치를 초과하였다.

## 증거

Boot 화면의 CPU Warning

Screenshot : 04_CPU_Fixed.png

## 원인

CPU 제한값이 권장 기준보다 높게 설정되어 있었다.

## 조치

```
export CPU_MAX_OCCUPY=50
```

변경 후

```
CPU Limit : 50%

OK
```

로 변경되었으며 CPU Warning이 제거되는 것을 확인하였다.

---

# 7. Failure Report #3 - Potential Deadlock

## 현상

초기 실행

```
MULTI_THREAD_ENABLE=True
```

실행 결과

```
SYSTEM WARNING

POTENTIAL DEADLOCK IN CONCURRENT MODE
```

Deadlock 가능성이 출력되었다.

## 증거

Boot Log

Screenshot : 05_Deadlock_Fixed.png

## 원인

멀티스레드 환경에서 공유 자원 접근으로 인해 Deadlock 가능성이 존재하였다.

실제 Deadlock(PID는 살아 있으나 로그가 멈추는 상태)은 이번 실습에서 재현하지 못하였다.

## 조치

```
export MULTI_THREAD_ENABLE=false
```

변경 후

```
THREAD Concurrency : False

SYSTEM STATUS : STABLE
```

상태가 되어 Warning이 제거되는 것을 확인하였다.

---

# 8. monitor.sh 분석

monitor.sh는 다음 정보를 수집하도록 작성하였다.

- 프로세스 실행 여부
- 포트 상태
- CPU 사용률
- Memory 사용률
- Disk 사용률

사용 명령어

| 목적          | 명령어 |
| ------------- | ------ |
| Process 확인  | ps     |
| CPU 사용률    | top    |
| Memory 사용률 | free   |
| Disk 사용률   | df     |
| Port 확인     | ss     |

수집한 결과를 monitor.log에 기록하였다.

---

# 9. 운영 환경 개선 방안

## OOM

- Memory Threshold 기반 사전 경고
- Heap 사용률 80% 이상 시 Alert 발생
- 메모리 누수 코드 수정

## CPU

- CPU 80% 이상 지속 시 Alert
- Worker Thread 수 제한
- Rate Limiting 적용

## Deadlock

- Lock Timeout 적용
- Lock 획득 순서 통일
- Thread Dump 자동 저장
- Watchdog을 이용한 교착 상태 감지

---

# 10. 회고

이번 미션을 통해 단순히 프로그램을 실행하는 것이 아니라 운영 환경에서 장애를 재현하고 분석하는 과정이 중요하다는 것을 확인하였다.

특히 OOM, CPU Spike, Deadlock은 단순히 환경변수만 변경하는 것이 아니라 로그, PID, 시스템 자원 사용률 등을 함께 확인해야 원인을 정확하게 분석할 수 있다는 점을 배울 수 있었다.
