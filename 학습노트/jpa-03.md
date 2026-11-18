# Chapter 3. 엔티티 단위 CRUD 처리

## 1. 한눈에 보기

| 작업 | 메서드 | 트랜잭션 | 비고 |
|---|---|---|---|
| 생성 | `em.persist(entity)` | 필요 | 비영속 → 영속 |
| 조회 | `em.find(Class, id)` | **불필요** | 없으면 `null` |
| 수정 | 영속 엔티티 값 변경 | 필요 | 변경 감지 (`changeName`) |
| 삭제 | `em.remove(entity)` | 필요 | 영속 상태여야 함 |

## 2. 코드 구조

```
jpabasic.reserve
├─ app
│  ├─ NewUserService       ← 저장
│  ├─ GetUserService       ← 조회
│  ├─ ChangeNameService    ← 이름 변경
│  ├─ RemoveUserService    ← 삭제
│  └─ NoUserException      ← 대상 없음 (RuntimeException)
├─ domain/User             ← 엔티티 (toString 추가)
├─ jpa/EMF                 ← EntityManagerFactory 보관
└─ main/Main               ← 콘솔 명령어 처리
```

### EMF 유틸 – EMF를 한 곳에서 공유
```java
public class EMF {
    private static EntityManagerFactory emf;

    public static void init() {
        emf = Persistence.createEntityManagerFactory("jpabegin");
    }
    public static EntityManager createEntityManager() {
        return emf.createEntityManager();
    }
    public static void close() {
        emf.close();
    }
}
```
- 앱 시작 시 `EMF.init()` 1번, 종료 시 `EMF.close()`
- 서비스는 `EMF.createEntityManager()`로 **작업마다 새 EM** 사용

### 콘솔 Main (명령어 → 서비스 호출)
| 명령어 | 동작 |
|---|---|
| `new 이메일 이름` | 저장 |
| `get 이메일` | 조회 |
| `change name 이메일 새이름` | 이름 변경 |
| `remove 이메일` | 삭제 |
| `exit` | 종료 |

## 3. 생성 – `NewUserService`

```java
public void saveNewUser(User user) {
    EntityManager em = EMF.createEntityManager();
    EntityTransaction tx = em.getTransaction();
    try {
        tx.begin();
        em.persist(user);
        tx.commit();
    } catch(Exception ex) {
        tx.rollback();
        throw ex;            // 예외를 다시 던져 호출 쪽에서 처리
    } finally {
        em.close();
    }
}
```
- `Main`에서는 이미 존재하는 사용자 저장 시 `EntityExistsException`을 잡아 "사용자가 이미 존재함"으로 처리하려는 구조 (실제로 던져지는 예외 타입은 실행해서 확인해 볼 것)
- PK(식별자)가 이미 있으면 INSERT 실패 → 롤백

## 4. 조회 – `GetUserService`

```java
public User getUser(String email) {
    EntityManager em = EMF.createEntityManager();
    try {
        User user = em.find(User.class, email);
        if (user == null) {
            throw new NoUserException();
        }
        return user;
    } finally {
        em.close();
    }
}
```
- **트랜잭션 없이** 조회 (조회만 하므로 불필요)
- 없으면 `null` → 서비스에서 `NoUserException`으로 변환
- `em.close()` 후 반환되는 `User`는 **준영속 상태**지만, 단순 필드 읽기(`toString`)는 문제 없음 (지연 로딩이 있으면 문제가 됨)

### find() 동작
1. 1차 캐시 확인 → 있으면 반환
2. 없으면 SELECT → 1차 캐시에 저장 → 반환
3. DB에도 없으면 `null`

## 5. 수정 – `ChangeNameService`

```java
tx.begin();
User user = em.find(User.class, email);
if (user == null) {
    throw new NoUserException();
}
user.changeName(newName);    // 영속 상태 → 커밋 시 UPDATE
tx.commit();
```
- **조회(영속 상태) → 값 변경 → 커밋** 이 수정의 전부
- `User`에 setter 대신 `changeName()`을 두어 변경 의도를 드러냄
- 예외 시 `rollback()` 후 다시 던짐

## 6. 삭제 – `RemoveUserService`

```java
tx.begin();
User user = em.find(User.class, email);
if (user == null) {
    throw new NoUserException();
}
em.remove(user);             // 삭제 상태 → 커밋 시 DELETE
tx.commit();
```
- **영속 상태 엔티티만** 삭제 가능 (그래서 먼저 `find`)
- 준영속 엔티티를 `remove()` 하면 `IllegalArgumentException`

## 7. 서비스 메서드 공통 패턴

```java
EntityManager em = EMF.createEntityManager();
EntityTransaction tx = em.getTransaction();
try {
    tx.begin();
    // 작업
    tx.commit();
} catch (Exception ex) {
    tx.rollback();
    throw ex;
} finally {
    em.close();
}
```
- 변경 작업(저장/수정/삭제): 위 전체 패턴
- 조회 전용: 트랜잭션 없이 `try/finally`로 `em.close()`만
- 이 반복 코드를 줄이는 것이 스프링의 `@Transactional` 역할 (이번 실습은 스프링 없이 직접 구현)

## 8. 준영속 엔티티와 merge (보충 개념)

- 실습 코드는 **항상 같은 EM 안에서 조회 → 변경**하므로 `merge()`가 필요 없다
- 이전 EM에서 가져온 준영속 엔티티를 수정하려면 `em.merge(detached)` (반환된 영속 인스턴스를 사용), 단 필드 전체를 덮어쓰는 점에 주의
- 권장: **ID로 다시 `find()` → 영속 엔티티 변경**

## 9. 정리

- 엔티티 단위 CRUD는 **영속 상태 관리**가 핵심
- 수정에 `update` 메서드는 없다 → 영속 엔티티를 변경(변경 감지)
- 변경·삭제는 **트랜잭션 + 영속 상태**, 조회는 트랜잭션 불필요
- 서비스 계층에서 EM 생성/종료와 트랜잭션 경계를 책임진다
