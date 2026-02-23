# 🕒 MySQL Delayed Replication 

> 운영자의 실수(DROP, DELETE 등)로부터 데이터를 보호하기 위한
> **지연 복제(Delayed Replication)** 실습 프로젝트

---

## 📌 Overview

본 실습은 MySQL의 **Delayed Replication(지연 복제)** 기능을 활용하여
운영 중 발생할 수 있는 인적 오류(Human Error)를 방어하는 방법을 검증합니다.

Replica 서버가 Source 서버를 일정 시간(delay) 뒤따라가도록 설정하여
사고 발생 시 복구 가능한 “골든타임”을 확보하는 것이 핵심 목표입니다.



## 🏗 Architecture

```
[ Source ]  ──── binlog ────▶  [ Replica ]
                               (Delay 60s)
```

* Source에서 발생한 변경사항은 즉시 전송
* Replica는 설정한 시간만큼 대기 후 적용
* 사고 발생 시 지연 시간 내 복구 가능


## 🚀 Setup

### 1️⃣ Replica에서 복제 중지

```sql
STOP REPLICA;
```


### 2️⃣ 지연 시간 설정 (60초)

```sql
CHANGE REPLICATION SOURCE TO SOURCE_DELAY = 60;
```

> 실무에서는 1시간(3600초) 이상 설정하는 경우도 있음



### 3️⃣ 복제 재시작

```sql
START REPLICA;
```

### 4️⃣ 설정 확인

```sql
SHOW REPLICA STATUS\G;
```

확인 항목:

```
SQL_Delay: 60
```



## 🧪 Validation Scenario

### Phase 1 — Delay 동작 확인

#### Source

```sql
INSERT INTO products VALUES ('Golden Item');
```

#### Replica

```sql
SHOW REPLICA STATUS\G;
```

확인:

```
SQL_Remaining_Delay: 60 → 59 → 58 ...
```

✔ 60초가 지나기 전까지 데이터는 Replica에 반영되지 않음

---

### Phase 2 — 사고 발생 & 대응

#### 1️⃣ Source에서 실수 발생

```sql
DELETE FROM products;
```


#### 2️⃣ Replica에서 즉시 실행

```sql
STOP REPLICA;
```

⚠ 핵심:
`SQL_Remaining_Delay`가 0이 되기 전에 멈춰야 한다.


#### 3️⃣ 데이터 확인

```sql
SELECT * FROM products;
```

✔ 데이터가 살아있다면 복구 성공



## 📊 Key Status Fields

| Field                   | Description | Meaning                        |
| ----------------------- | ----------- | ------------------------------ |
| `SQL_Delay`             | 목표 지연 시간    | 설정한 초 단위 시간                    |
| `SQL_Remaining_Delay`   | 실행 대기 시간    | 숫자: 카운트다운 중<br>NULL: 대기 이벤트 없음 |
| `Seconds_Behind_Source` | 전체 시차       | Delay + 네트워크 지연                |

---

## 🛠 Troubleshooting

### ❗ Error 1046 — No database selected

```sql
USE testdb;
```

반드시 데이터베이스 선택 후 실행

---

### ❗ SQL_Remaining_Delay = NULL

원인:

* Source에서 새로운 변경 이벤트가 없음

해결:

```sql
INSERT INTO products VALUES ('Test');
```

---

## 💡 Lessons Learned

* 지연 복제는 하드웨어 장애 대응 기술이 아니다.
* 이는 **운영자의 실수로부터 데이터를 보호하는 전략**이다.
* “삭제 후 복구”가 아니라 “삭제 반영 전 차단”이 핵심이다.
* 백업 전략과 함께 사용해야 진정한 보호 체계가 된다.



## 🔐 Real-World Usage

* 금융권 및 대형 서비스 운영 환경
* 1시간~24시간 지연 설정
* Backup + Delayed Replica + Multi Replica 전략



# 📌 Conclusion

> Delayed Replication은 단순한 복제 옵션이 아니라
> 운영 리스크를 제어하는 시간 기반 방어 전략이다.

