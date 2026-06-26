<!-- @format -->

# AI SW Basic - Agent Leak App 장애 분석 보고서

## 개발 환경

| 항목         | 내용                     |
| ------------ | ------------------------ |
| OS           | macOS (Apple Silicon M3) |
| 실행 환경    | Docker Ubuntu 22.04      |
| Architecture | linux/arm64              |

---

# 실행 환경 구성

## Docker 실행

```bash
docker run --rm -it \
  --platform linux/arm64 \
  -v "$(pwd)":/mission \
  -w /mission \
  ubuntu:22.04 bash
```

## 일반 사용자 생성

```bash
useradd -m agent-admin
su - agent-admin
cd /mission
```

## 환경변수 설정

```bash
export AGENT_HOME=/mission
export AGENT_PORT=15034
export MEMORY_LIMIT=256
export CPU_MAX_OCCUPY=70
export MULTI_THREAD_ENABLE=true

export AGENT_UPLOAD_DIR=$AGENT_HOME/upload_files
export AGENT_KEY_PATH=$AGENT_HOME/api_keys
export AGENT_LOG_DIR=$AGENT_HOME/logs
```

## 필수 디렉터리 생성

```bash
mkdir -p upload_files
mkdir -p api_keys
mkdir -p logs
```

## secret.key 생성

```bash
echo "agent_api_key_test" > api_keys/secret.key
```

## 권한 설정

```bash
chmod 777 upload_files api_keys logs
chmod 644 api_keys/secret.key
```

---

# Boot Sequence 결과

모든 Boot Check를 통과하였다.

```text
Checking User Account        [OK]
Checking Environment Variables [OK]
Checking Required Files      [OK]
Checking Port Availability   [OK]
Checking Log Permission      [OK]
Checking Mission Environment [OK]

All Boot Checks Passed!
Agent READY
```

---

# monitor.sh

monitor.sh를 작성하여 다음 항목을 확인하였다.

- Agent Process 실행 여부
- Port 15034 사용 여부
- CPU 사용률
- Memory 사용률
- Disk 사용률
- Agent Process Resource 사용률
- monitor.log 생성

실행

```bash
./monitor.sh
```

실행 결과

```text
Checking process ... OK
Checking port 15034 ... OK

CPU Usage
MEM Usage
DISK Used
Agent CPU Usage
Agent MEM Usage

Log appended:
/var/log/agent-app/monitor.log
```

---

# 장애 분석

## 1. OOM (Out Of Memory)

### 장애 발생

프로그램 실행 후 Heap Memory가 지속적으로 증가하였다.

```text
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

```text
[CRITICAL] Memory limit exceeded

[SYSTEM] SELF-TERMINATED
```

메시지와 함께 프로그램이 종료되었다.

### 원인

```text
MEMORY_LIMIT=256MB
```

Heap Memory가 제한값을 초과하면서 MemoryGuard가 프로세스를 종료하였다.

### 해결 방법

```bash
export MEMORY_LIMIT=512
```

또는

```bash
export MEMORY_LIMIT=1024
```

으로 변경하여 더 많은 메모리를 사용할 수 있도록 설정한다.

---

## 2. CPU Warning

### 장애 발생

초기 실행 환경

```text
CPU_MAX_OCCUPY=70%
```

실행 결과

```text
[ CPU ] Limit : 70%

WARNING
Recommend Under 50%
```

CPU 사용률이 권장 기준을 초과하였다.

### 원인

CPU 사용 제한이 권장값(50%)보다 높게 설정되어 있었다.

### 해결 방법

환경변수를 수정하였다.

```bash
export CPU_MAX_OCCUPY=50
```

변경 후

```text
CPU Limit : 50%

OK
```

으로 변경되어 CPU 경고가 제거되었다.

---

## 3. Potential Deadlock

### 장애 발생

초기 실행 환경

```text
MULTI_THREAD_ENABLE=True
```

실행 결과

```text
SYSTEM WARNING

POTENTIAL DEADLOCK IN CONCURRENT MODE
```

멀티스레드 환경에서 Deadlock 가능성이 경고되었다.

### 원인

동시성(Concurrent Mode)이 활성화되어 있어 여러 스레드가 동시에 자원에 접근하면서 Deadlock 가능성이 존재하였다.

### 해결 방법

환경변수를 변경하였다.

```bash
export MULTI_THREAD_ENABLE=false
```

변경 후

```text
THREAD Concurrency : False

OK

SYSTEM STATUS

STABLE
```

로 변경되어 Deadlock Warning이 제거되고 Stable 상태가 확인되었다.

---

# 수행 결과

| 항목                  | 결과 |
| --------------------- | ---- |
| Boot Sequence         | 성공 |
| monitor.sh 실행       | 성공 |
| OOM 재현              | 성공 |
| CPU Warning 재현      | 성공 |
| CPU Warning 해결      | 성공 |
| Deadlock Warning 재현 | 성공 |
| Deadlock Warning 해결 | 성공 |

---

# 사용한 환경변수

초기 설정

```text
AGENT_HOME=/mission
AGENT_PORT=15034
MEMORY_LIMIT=256
CPU_MAX_OCCUPY=70
MULTI_THREAD_ENABLE=true
```

CPU 수정

```text
CPU_MAX_OCCUPY=50
```

Deadlock 수정

```text
MULTI_THREAD_ENABLE=false
```

Memory 수정 예시

```text
MEMORY_LIMIT=512
또는
MEMORY_LIMIT=1024
```

---

# 실행 결과 캡처

아래 순서대로 캡처를 첨부하였다.

1. Root 실행 오류
2. Boot Sequence 성공
3. OOM 발생
4. CPU Warning 해결
5. Deadlock Warning 해결
