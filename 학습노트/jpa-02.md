# Chapter 2. 코드 구조와 영속 컨텍스트

## 1. 기본 코드 구조

```java
EntityManagerFactory emf = Persistence.createEntityManagerFactory("jpabegin"); // 앱 시작 시 1번

EntityManager entityManager = emf.createEntityManager();   // 작업(요청) 단위
EntityTransaction transaction = entityManager.getTransaction();
try {
    transaction.begin();
    // ... JPA 작업
    transaction.commit();
} catch (Exception ex) {
    logger.error("에러 발생: " + ex.getMessage(), ex);
    transaction.rollback();
} finally {
    entityManager.close();                                 // 반드시 닫기
}

emf.close();                                               // 앱 종료 시
```

### 규칙
1. **EMF**: 앱 전체에서 하나만 만들어 공유 (스레드 안전, 생성 비용 큼)
2. **EM**: 요청/작업마다 새로 만들고 **스레드 간 공유 금지**
3. 변경이 일어나는 작업은 **트랜잭션 필수**
4. 예외 발생 시 `rollback()`, 마지막엔 `em.close()` (`finally`)
5. 예외가 난 EM은 더 쓰지 않는 것이 안전

### EMF와 EM이란?

**EntityManagerFactory (EMF)** — EntityManager를 만들어내는 공장
```java
EntityManagerFactory emf = Persistence.createEntityManagerFactory("jpabegin");
```
- `persistence.xml`의 설정(DB 연결 정보, 엔티티 목록, 다이얼렉트, 커넥션 풀 등)을 읽어서 생성
- 설정 파싱, 매핑 정보 분석, 커넥션 풀 준비까지 하므로 **생성 비용이 크다**
- **앱 전체에서 하나만** 만들어 공유, **스레드 안전**
- 앱 종료 시 `emf.close()`

**EntityManager (EM)** — 엔티티 저장·조회·수정·삭제를 실제로 수행하는 객체
```java
EntityManager em = emf.createEntityManager();
```
- `persist()`, `find()`, `remove()` 등 JPA 기능의 진입점
- 내부에 **영속 컨텍스트**(1차 캐시, 쓰기 지연 저장소)를 하나씩 가짐
- 생성 비용이 작아서 **작업(요청)마다 새로 만들고 `close()`**
- **스레드 안전하지 않음** → 여러 스레드 공유 금지
- `close()` 하면 영속 컨텍스트가 사라지고, 관리하던 엔티티는 모두 준영속이 됨

| 구분 | EMF | EM |
|---|---|---|
| 개수 | 앱당 1개 | 작업마다 1개 |
| 생성 비용 | 큼 | 작음 |
| 스레드 안전 | O | **X** |
| 수명 | 앱 시작 ~ 종료 | 작업 시작 ~ 종료 |
| 하는 일 | EM 생성 | 엔티티 CRUD, 영속 컨텍스트 관리 |

> EMF = 식당(한 번 차려두고 계속 운영), EM = 손님 한 팀을 담당하는 종업원, 영속 컨텍스트 = 그 종업원의 주문 메모

- EMF를 요청마다 만들면 설정 로딩·커넥션 풀 생성이 반복되어 매우 느려지고, EM을 공유하면 영속 컨텍스트가 뒤섞여 데이터가 꼬인다.

## 2. 영속 컨텍스트 (Persistence Context)

- **엔티티를 영구 저장하는 환경**. `EntityManager`가 내부적으로 하나씩 가짐
- `em.persist()`, `em.find()` 등으로 다루는 엔티티는 영속 컨텍스트에 보관·관리됨
- EM 생성 시 만들어지고, `close()` 하면 사라짐
- 내부 구성: **1차 캐시(식별자 → 엔티티)**, 쓰기 지연 SQL 저장소, 최초 상태 스냅샷

### 2-1. 1차 캐시와 동일성 보장
```java
User u1 = em.find(User.class, "user@user.com");   // DB 조회 후 1차 캐시에 저장
User u2 = em.find(User.class, "user@user.com");   // 1차 캐시에서 반환 (SELECT 없음)
System.out.println(u1 == u2);                     // true
```
- 같은 EM 안에서 같은 식별자는 **같은 인스턴스** 보장
- `persist()` 한 엔티티도 1차 캐시에 저장 → 바로 `find()` 해도 SELECT가 나가지 않음
- 영속 컨텍스트 범위 내 캐시라서 EM이 달라지면 공유되지 않음

### 2-2. 쓰기 지연 (Write-Behind) — `UserSaveMain`
```java
transaction.begin();
User user = new User("user@user.com", "user", LocalDateTime.now());
entityManager.persist(user);                       // INSERT를 바로 보내지 않고 보관
logger.info("EntityManager.persist 호출함");
transaction.commit();                              // flush → INSERT 실행 → DB 커밋
logger.info("EntityTransaction.commit 호출함");
```
- 예상 로그 순서: `persist 호출함` → (커밋 시점) `insert into user ...` → `commit 호출함`
- SQL을 모아뒀다가 **flush 시점에 한꺼번에 전송** (배치 최적화 여지)

### 2-3. 변경 감지 (Dirty Checking) — `UserUpdateMain`
```java
transaction.begin();
User user = entityManager.find(User.class, "user@user.com");   // 스냅샷 저장
if (user == null) {
    System.out.println("User 없음");
} else {
    String newName = "이름" + (System.currentTimeMillis() % 100);
    user.changeName(newName);                                  // 객체 값만 변경
    logger.info("User.changeName 호출함");
}
transaction.commit();                                          // flush → UPDATE
logger.info("EntityTransaction.commit 호출함");
```
1. 영속 컨텍스트에 들어올 때 **스냅샷(최초 상태)** 저장
2. flush 시 현재 값과 스냅샷 비교
3. 다르면 UPDATE 쿼리 생성 → DB 전송
- `changeName 호출함` 로그 이후, 커밋 시점에 `update user set ...` 이 찍힌다
- 기본 UPDATE는 **모든 컬럼**을 SET에 포함 (변경 컬럼만: Hibernate `@DynamicUpdate`)

## 3. flush

영속 컨텍스트의 변경 내용을 DB에 **동기화**하는 작업 (트랜잭션 커밋과는 다름: flush 후에도 롤백 가능).

### flush 발생 시점
1. `transaction.commit()` 시 자동
2. **JPQL 등 쿼리 실행 직전** 자동 (쿼리 결과가 최신 변경을 반영해야 하므로)
3. `em.flush()` 직접 호출

```
persist/변경 → (쓰기 지연 저장소) → flush → SQL 전송 → commit
```
- flush 해도 1차 캐시는 비워지지 않는다.

## 4. 엔티티의 생명주기 (상태)

```
        persist()           detach()/clear()/close()
비영속 ─────────→ 영속 ──────────────────────→ 준영속
(new)             (managed)   ←── merge() ───    (detached)
                    │
                    │ remove()
                    ↓
                  삭제 (removed)
```

| 상태 | 설명 |
|---|---|
| **비영속 (new)** | `new User(...)` 만 한 상태. JPA와 무관 |
| **영속 (managed)** | 영속 컨텍스트가 관리. 변경 감지/1차 캐시/쓰기 지연 적용 |
| **준영속 (detached)** | 한때 영속이었지만 분리됨. 변경해도 DB 반영 안 됨 |
| **삭제 (removed)** | 삭제 예약. flush/commit 시 DELETE |

- 준영속이 되는 경우: `detach()`, `clear()`, **`close()`** (EM이 닫히면 그 EM이 관리하던 엔티티는 모두 준영속)

### 영속 상태란?

**엔티티가 현재 EntityManager의 영속 컨텍스트에 등록되어, JPA가 그 객체를 추적·관리하는 상태.**
"DB의 한 행과 연결되어 JPA가 감시하는 객체"라고 이해하면 된다.

영속 상태가 되는 경로
```java
// 1) persist()로 직접 등록
User user = new User("a@a.com", "홍길동", LocalDateTime.now());  // 비영속: 그냥 자바 객체
em.persist(user);                                                // 영속

// 2) 조회하면 자동으로 영속
User found = em.find(User.class, "a@a.com");                     // 영속
```

영속 상태이기 때문에 얻는 것

| 기능 | 내용 |
|---|---|
| 변경 감지 | 값을 바꾸면 커밋 시점에 UPDATE 자동 실행 |
| 1차 캐시 | 같은 ID를 다시 `find()` 해도 SELECT 없이 같은 객체 반환 |
| 쓰기 지연 | INSERT/UPDATE를 모아뒀다가 flush 시점에 전송 |
| 동일성 보장 | 같은 EM 안에서 같은 ID는 같은 인스턴스 (`==` true) |

영속 vs 준영속 비교
```java
User user = em.find(User.class, "a@a.com");    // 영속
user.changeName("김철수");                      // UPDATE 예약됨
tx.commit();                                   // DB 반영

em.detach(user);                               // 준영속으로 분리
user.changeName("박영희");                      // 아무 일도 안 일어남 (DB 반영 안 됨)
```
- 두 경우 모두 같은 `User`, 같은 `changeName()` 호출이다. 차이는 **영속 컨텍스트가 관리 중인가**뿐이다.
- 영속 상태는 **EntityManager가 살아 있는 동안만** 유지된다.
- 그래서 수정/삭제는 먼저 `find()`로 영속 상태로 만든 뒤 수정/삭제한다.

## 5. 정리

- 코드 틀: `EMF(1개) → EM(작업당) → Transaction(begin/commit/rollback) → close`
- 로그(`org.hibernate.SQL`)로 **flush 시점에 SQL이 실행**됨을 확인
- 영속 컨텍스트 = **1차 캐시 + 쓰기 지연 + 변경 감지**
- 영속 상태일 때만 JPA 자동 기능이 적용된다
