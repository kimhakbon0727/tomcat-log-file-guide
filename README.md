## 📂 상황별 로그 확인 가이드

> 오류 발생 시 **"지금 어떤 상황인가"** 를 먼저 파악하고, 아래 시나리오에 맞는 로그를 확인하세요.
> 로그 파일 경로: `$CATALINA_HOME/logs/`

---

### 🖥️ 시나리오 1 — 홈페이지 자체가 아예 안 열릴 때

**증상:** 브라우저에서 구청 홈페이지 접속 시 연결 자체가 안 됨 (브라우저 오류 화면)

**→ 확인할 로그: `catalina.out`**

```bash
tail -f $CATALINA_HOME/logs/catalina.out
grep "SEVERE" $CATALINA_HOME/logs/catalina.out | tail -30
```

**이유:** 톰캣 서버 자체가 죽었거나 기동에 실패한 상황이므로
서버 레벨 로그인 catalina.out 에서 원인을 찾아야 함

---

### 🔍 시나리오 2 — 특정 메뉴/페이지만 404 뜰 때

**증상:** 홈은 정상인데 "민원신청", "공지사항" 등 특정 페이지만 Not Found

**→ 확인할 로그: `localhost_access_log`**

```bash
grep " 404 " $CATALINA_HOME/logs/localhost_access_log.$(date +%Y-%m-%d).txt | tail -20
```

**이유:** 어떤 URL에서 404가 발생했는지 요청 기록이 남아 있어
URL 경로 오류인지, 배포 누락인지 판단 가능

---

### 💥 시나리오 3 — 특정 기능 클릭 시 500 에러 뜰 때

**증상:** "민원 접수하기" 버튼 클릭 시 500 Internal Server Error 페이지 출력

**→ 확인할 로그: `localhost_access_log` → `localhost.log` 순서로 확인**

```bash
# 1단계: 500 발생 시각과 URL 특정
grep " 500 " $CATALINA_HOME/logs/localhost_access_log.$(date +%Y-%m-%d).txt | tail -20

# 2단계: 해당 시각 기준으로 앱 로그에서 원인 추적
grep -A 20 "Exception" $CATALINA_HOME/logs/localhost.$(date +%Y-%m-%d).log | tail -60
```

**이유:** access_log 로 "언제, 어떤 요청에서" 500이 났는지 특정하고
localhost.log 에서 실제 코드 오류 원인(Exception)을 찾는 순서

---

### 📦 시나리오 4 — 배포(업데이트) 후 사이트가 안 뜰 때

**증상:** 새 버전 WAR 배포 후 홈페이지 접속 안 됨, 톰캣은 프로세스 살아있음

**→ 확인할 로그: `localhost.log`**

```bash
tail -100 $CATALINA_HOME/logs/localhost.$(date +%Y-%m-%d).log
grep "SEVERE\|Exception" $CATALINA_HOME/logs/localhost.$(date +%Y-%m-%d).log | tail -30
```

**이유:** 톰캣은 정상이지만 앱(WAR) 초기화에 실패한 상황
앱 배포/초기화 오류는 localhost.log 에 기록됨
(DB 연결 실패, 설정 오류, 라이브러리 충돌 등)

---

### 🐌 시나리오 5 — 홈페이지가 갑자기 느려졌을 때

**증상:** 특정 시간대부터 페이지 로딩이 매우 느려졌다는 민원 접수

**→ 확인할 로그: `localhost_access_log`**

```bash
# 해당 시간대 요청량 확인 (예: 14시대)
grep "\[28/Apr/2026:14:" $CATALINA_HOME/logs/localhost_access_log.$(date +%Y-%m-%d).txt | wc -l

# 느린 응답 요청 확인 (응답시간 포맷 설정된 경우)
grep "\[28/Apr/2026:14:" $CATALINA_HOME/logs/localhost_access_log.$(date +%Y-%m-%d).txt | tail -30
```

**이유:** 어느 시간대에 트래픽이 몰렸는지, 특정 URL에 요청이 집중됐는지
access_log 에서 패턴을 먼저 파악해야 원인 추적 가능

---

### 💾 시나리오 6 — 서버가 갑자기 다운됐다 살아날 때 (간헐적 장애)

**증상:** 하루에 몇 번씩 홈페이지가 잠깐 안 되다가 다시 됨

**→ 확인할 로그: `catalina.out`**

```bash
# OutOfMemoryError 여부 확인
grep "OutOfMemoryError" $CATALINA_HOME/logs/catalina.out

# 톰캣 재기동 시각 확인
grep "Server startup\|Server shutdown" $CATALINA_HOME/logs/catalina.out
```

**이유:** 간헐적 다운은 메모리 부족(OOM) 또는 JVM 크래시가 원인인 경우가 많음
catalina.out 에서 재기동 시각과 직전 오류를 함께 확인

---

### 👤 시나리오 7 — 로그인은 되는데 특정 사용자만 오류 난다고 할 때

**증상:** 일반 사용자는 정상인데 특정 직원/부서만 기능 오류 발생

**→ 확인할 로그: `localhost_access_log` → `localhost.log` 순서로 확인**

```bash
# 해당 사용자 세션의 요청 패턴 확인 (IP 기준)
grep "해당IP" $CATALINA_HOME/logs/localhost_access_log.$(date +%Y-%m-%d).txt

# 해당 시각 앱 오류 확인
grep -A 15 "Exception" $CATALINA_HOME/logs/localhost.$(date +%Y-%m-%d).log
```

**이유:** 특정 사용자 요청만 오류나는 경우 권한, 세션, 데이터 문제가 원인
access_log 로 요청 패턴 확인 후 localhost.log 에서 Exception 원인 추적

---

### 📋 한눈에 보는 시나리오별 로그 요약

| 상황 | 1순위 로그 | 2순위 로그 |
|---|---|---|
| 홈페이지 자체가 안 열림 | `catalina.out` | - |
| 특정 페이지만 404 | `access_log` | - |
| 특정 기능 클릭 시 500 | `access_log` | `localhost.log` |
| 배포 후 사이트 안 뜸 | `localhost.log` | - |
| 갑자기 느려짐 | `access_log` | - |
| 간헐적 다운/재기동 | `catalina.out` | - |
| 특정 사용자만 오류 | `access_log` | `localhost.log` |
