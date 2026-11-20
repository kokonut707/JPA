# Chapter 11. 컬렉션 매핑의 N+1 문제, 페이징 한계, 그리고 CQRS

## 1. 배경

값 콜렉션(`Set`, `List`, `Map`)을 엔티티에 매핑하면 **단건 처리**(저장/수정/삭제)는 편하지만, **목록 조회**에서는 성능 문제가 생긴다.

- **N+1 쿼리 문제**: 목록 1번 조회 + 각 엔티티의 콜렉션 조회 N번
- **페이징 한계**: 콜렉션을 `join fetch`로 한 번에 가져오면 DB가 아니라 **메모리에서 페이징**

## 2. 예제 구성 – `Role`, `Main11`

```java
@Entity
@Table(name = "role")
public class Role {
    @Id
    private String id;
    private String name;

    @ElementCollection(fetch = FetchType.EAGER)      // 즉시 로딩
    @CollectionTable(
            name = "role_perm",
            joinColumns = @JoinColumn(name = "role_id")
    )
    @Column(name = "perm")
    private Set<String> permissions = new HashSet<>();
    ...
}
```
- 테이블은 jpa-08의 `role`, `role_perm`을 그대로 사용 (Chapter 8과 달리 `fetch = EAGER`)

### 테스트 데이터와 조회 조건 (`Main11`)
```java
for (int i = 11; i <= 20; i++) {
    saveRole("R" + i, "name" + i);       // R11 ~ R20, 각 롤마다 권한 F1, F2
}
selectNoFetchMode();
selectFetchMode();
```
- 롤 10건 × 권한 2건 = `role_perm` 20행
- 두 조회 모두 `setFirstResult(6)`, `setMaxResults(3)` → `id` 순으로 **7번째부터 3건**(R17, R18, R19)을 가져오는 **페이징**

## 3. N+1 문제 – `selectNoFetchMode`

```java
TypedQuery<Role> query = em.createQuery(
        "select r from Role r where r.name like :name order by r.id", Role.class);
query.setParameter("name", "name%");
query.setFirstResult(6);
query.setMaxResults(3);
List<Role> roles = query.getResultList();
roles.forEach(r -> logger.info("... perms={}", r.getPermissions().size()));
```

### 발생하는 쿼리
1. `select ... from role where name like ? order by id limit ?, ?` → **롤 3건 조회 (1번)**
2. 롤마다 `select ... from role_perm where role_id=?` → **3번 추가**

→ 총 **1 + N (= 1 + 3) 쿼리**. 조회한 롤이 N건이면 쿼리가 N+1번 나간다.

### 원인
- JPQL은 **엔티티만 먼저** 조회하고, 값 콜렉션은 별도 쿼리로 가져온다. (`EAGER`여도 JPQL에서는 콜렉션 조회가 별도 쿼리로 나간다.)
- `LAZY`였다면 `getPermissions()`에 접근하는 시점마다 쿼리가 나가므로 마찬가지로 N+1이 된다.
- 목록 화면에서 100건을 보여주면 **101번의 쿼리**가 발생해 DB 부하와 응답 시간이 커진다.

## 4. fetch join과 페이징 한계 – `selectFetchMode`

N+1을 피하려고 콜렉션을 **fetch join**으로 한 번에 가져오는 방법.

```java
TypedQuery<Role> query = em.createQuery(
        "select r from Role r left join fetch r.permissions where r.name like :name order by r.id",
        Role.class);
query.setParameter("name", "name%");
query.setFirstResult(6);
query.setMaxResults(3);
```

### 결과
- 쿼리는 1번으로 줄어든다. (role + role_perm 조인)
- 하지만 Hibernate가 다음과 같은 **경고 로그**를 남기고, SQL에 `limit`를 붙이지 못한다.
  ```
  firstResult/maxResults specified with collection fetch; applying in memory
  ```
- 즉 **조건에 맞는 모든 행을 DB에서 읽어 와서 메모리에서 잘라낸다.**

### 왜 DB에서 페이징할 수 없나
- `role`과 `role_perm`을 조인하면 **롤 1건이 권한 수만큼 여러 행**으로 늘어난다. (롤 10건 → 20행)
- SQL의 `limit 3`은 **행 기준**이라, "롤 3건"이 아니라 "행 3건"이 되어 롤 단위 페이징이 깨진다.
- 그래서 Hibernate가 전체를 읽고 메모리에서 롤 단위로 자르는 방식을 택한다.

### 문제점

| 방식 | 쿼리 수 | 페이징 | 문제 |
|---|---|---|---|
| 조회 + 콜렉션 접근 (`NoFetch`) | 1 + N | DB에서 처리 | **N+1** |
| `left join fetch` (`Fetch`) | 1 | **메모리에서 처리** | 데이터가 많으면 **메모리 부족/성능 저하** |

- 두 방법 모두 한쪽을 고치면 다른 쪽 문제가 생기는 **딜레마**다.

### 참고: 완화책 (노트 보충)
- `hibernate.default_batch_fetch_size` 또는 `@BatchSize`: 콜렉션을 **IN 절로 묶어서** 가져와 `1 + N`을 `1 + (N / batch size)`로 줄이는 Hibernate 설정 (페이징은 DB에서 유지)
- 그래도 **조회 전용 화면이 엔티티 구조에 끌려가는** 근본 문제는 남는다.

## 5. 해결 방향 – CQRS (조회와 변경 모델 분리)

### 문제의 본질
- 하나의 **엔티티 모델**을 변경(저장/수정/삭제)과 조회(목록/검색) **둘 다**에 쓰려니 충돌이 생긴다.
  - 변경: 도메인 규칙과 일관성이 중요 → 엔티티 + 값 콜렉션이 적합
  - 조회: 화면에 필요한 데이터를 **효율적으로** 읽는 것이 중요 → 엔티티 구조가 오히려 방해

### CQRS (Command Query Responsibility Segregation)
**명령(Command, 변경)과 조회(Query)의 책임을 분리**하는 패턴. 모델 단위에서 분리한다.

```
          ┌─ 변경(Command) ─→ 도메인 모델(JPA 엔티티, 값 콜렉션) ──┐
요청 ─────┤                                                           ├→ DB
          └─ 조회(Query)   ─→ 조회 전용 모델(DTO, 전용 쿼리)    ────┘
```

| 구분 | 변경 쪽 | 조회 쪽 |
|---|---|---|
| 모델 | 도메인 엔티티 (`Role`) | 화면용 조회 모델 (DTO) |
| 목적 | 일관성, 도메인 규칙 | 조회 성능, 화면 요구 반영 |
| 기술 | JPA (영속 컨텍스트, 변경 감지) | JPQL DTO 프로젝션, 네이티브 SQL, JdbcTemplate, MyBatis, jOOQ 등 |
| 페이징 | 해당 없음 | **DB 레벨 페이징** |

### 적용 예 (개념 예시, 레포 코드 아님)
```java
// 조회 전용 모델 – 엔티티와 무관한 DTO
public record RoleSummary(String id, String name, long permCount) {}
```
```sql
-- 조회 전용 쿼리: 조인 + 집계 + DB 페이징을 직접 제어
select r.id, r.name, count(p.perm) as perm_count
from role r left join role_perm p on p.role_id = r.id
where r.name like 'name%'
group by r.id, r.name
order by r.id
limit 3 offset 6;
```
- 목록 화면은 `Role` 엔티티를 거치지 않고 **필요한 컬럼만 한 번의 쿼리**로 읽는다.
- 변경 기능(롤 생성, 권한 부여/회수)은 기존처럼 `Role` 엔티티와 값 콜렉션으로 처리한다.

### 효과
- N+1과 메모리 페이징 문제를 **구조적으로 피한다.**
- 조회 요구가 바뀌어도 **도메인 모델을 건드리지 않는다.** (조회 모델만 수정)
- 변경 모델은 도메인 로직에 집중하고, 조회 모델은 성능에 집중하는 **역할 분리**

### 아키텍처 구성 포인트
- 패키지/모듈을 **변경(command)** 과 **조회(query)** 로 나눈다.
- 같은 DB를 보되, 조회 쪽은 엔티티 대신 **조회 전용 쿼리/DTO**를 쓴다.
- 규모가 커지면 조회 전용 저장소(읽기 DB, 캐시, 검색엔진)로 확장할 수도 있다. (단, 데이터 동기화 비용이 생김)
- 모든 화면에 적용할 필요는 없고, **목록·검색처럼 조회 부담이 큰 곳**부터 적용

## 6. 정리

| 항목 | 내용 |
|---|---|
| N+1 | JPQL 목록 조회 후 각 엔티티의 콜렉션을 개별 쿼리로 조회 → `1 + N` |
| fetch join의 한계 | 콜렉션 조인 + `setFirstResult/MaxResults` → **메모리 페이징** 경고 |
| 완화 | `default_batch_fetch_size`/`@BatchSize` (근본 해결은 아님) |
| 근본 해결 | **CQRS**: 변경 모델(엔티티)과 조회 모델(DTO/전용 쿼리) 분리 |
| 원칙 | 변경은 도메인 모델로, 조회는 화면에 맞는 전용 모델로 |
