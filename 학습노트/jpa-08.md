# Chapter 8. 값 콜렉션 매핑 – Set (@ElementCollection)

## 1. 개념

- **값 콜렉션**: 엔티티가 가진 `Set`, `List` 등의 컬렉션 요소가 **엔티티가 아니라 값(String, 값 객체)** 인 경우
- 컬렉션의 값들은 **별도 테이블**에 저장되고, 소유 엔티티의 PK를 외래 키로 가진다
- 값 타입이므로 **독립적인 식별자(@Id)가 없고**, 소유 엔티티와 생명주기를 함께한다

```
Role (엔티티)  ──  role 테이블
 └─ Set<String> permissions  ──  role_perm 테이블 (role_id, perm)
```

### 테이블 (`ddl.sql`)
```sql
create table jpabegin.role (
  id varchar(20) not null primary key,
  name varchar(100)
) engine innodb character set utf8mb4;

create table jpabegin.role_perm (
  role_id varchar(20),
  perm varchar(50),
  grantor varchar(50),
  primary key (role_id, perm)
) engine innodb character set utf8mb4;
```
- `role_perm`의 PK가 `(role_id, perm)` → **같은 롤에 같은 권한이 중복될 수 없는** Set의 성격과 일치
- `grantor`는 2절(`Role`)에서는 사용하지 않고, 4절(`Role2`)에서 사용

## 2. 단순 값(String) Set 매핑 – `Role`

```java
@Entity
@Table(name = "role")
public class Role {
    @Id
    private String id;
    private String name;

    @ElementCollection
    @CollectionTable(
            name = "role_perm",
            joinColumns = @JoinColumn(name = "role_id")
    )
    @Column(name = "perm")
    private Set<String> permissions = new HashSet<>();

    protected Role() {}
    ...
}
```

| 어노테이션 | 역할 |
|---|---|
| `@ElementCollection` | 값 콜렉션임을 선언 |
| `@CollectionTable(name)` | 콜렉션을 저장할 테이블 (`role_perm`) |
| `@CollectionTable(joinColumns)` | 소유 엔티티를 가리키는 FK 컬럼 (`role_id`) |
| `@Column(name = "perm")` | 요소(String) 값이 저장되는 컬럼 |

- 필드는 `new HashSet<>()`으로 **초기화**해 두는 것이 안전하다.
- 별도 엔티티·`@Id` 없이 어노테이션 3개만으로 매핑이 끝난다.

## 3. 동작 – `MainRole`

### 저장
```java
Role role = new Role("R07", "관리자", Set.of("F1", "F2"));
em.persist(role);
tx.commit();
```
- `role`에 INSERT, 이어서 콜렉션 요소마다 **`role_perm`에 INSERT** (F1, F2 → 2건)

### 조회 – 지연 로딩(Lazy)과 즉시 로딩(Eager)
```java
Role role = em.find(Role.class, roleId);           // role만 조회
for (String perm : role.getPermissions()) { ... }  // 이 시점에 role_perm 조회
```

| 방식 | 설정 | 동작 |
|---|---|---|
| **Lazy (기본값)** | 설정 없음 | 콜렉션에 **실제 접근할 때** `role_perm` 조회 |
| **Eager** | `@ElementCollection(fetch = FetchType.EAGER)` | 엔티티 조회 시 `role_perm`도 **함께** 조회 |

```java
@ElementCollection(fetch = FetchType.EAGER)
@CollectionTable(name = "role_perm", joinColumns = @JoinColumn(name = "role_id"))
@Column(name = "perm")
private Set<String> permissions = new HashSet<>();
```
- Lazy: 필요할 때만 조회해 **불필요한 쿼리 방지**. 단, **EM이 닫힌 뒤(준영속)** 접근하면 `LazyInitializationException`이 발생하므로 EM이 열려 있는 동안 접근해야 한다. 예제도 같은 트랜잭션 안에서 읽는다.
- Eager: 항상 함께 쓰는 데이터일 때 유리하지만, 안 쓰는데도 조회되는 비용이 있다.

### 수정 방법 1. 컬렉션 요소 변경 – `updateRoleByModifyingCollection`
```java
Role role = em.find(Role.class, roleId);
role.getPermissions().add("F3");
role.getPermissions().remove("F1");
tx.commit();
```
- 컬렉션에 접근하는 시점에 `role_perm`을 조회하고, 커밋 시 변경 내용에 맞춰 **`INSERT`(F3)** 와 **`DELETE`(F1)** 가 실행된다.

### 수정 방법 2. 새 Set 할당 – `updateRoleByAssigningNewSet`
```java
Role role = em.find(Role.class, roleId);
role.setPermissions(Set.of("F4", "F5"));
tx.commit();
```
- 컬렉션을 **통째로 교체**하는 방식. 기존 `role_perm` 행이 삭제되고 새 요소가 INSERT된다.
- 위 두 방식 모두 **엔티티에 `update` 메서드를 호출하지 않아도** 변경 감지로 반영된다.

### 전체 삭제 – `revokeAll()`
```java
public void revokeAll() {
    this.permissions.clear();                  // SELECT -> DELETE
    // this.permissions = new HashSet<>();     // DELETE
}
```

| 방법 | 동작 |
|---|---|
| `permissions.clear()` | 콜렉션 접근 → **SELECT 후 DELETE** (`clear()` 호출 시 접근 시점에 조회가 일어남) |
| `permissions = new HashSet<>()` | 조회 없이 **DELETE만** 실행 (예제 주석 기준) |

- 요소를 전부 지우는 것이 목적이라면 **새 빈 Set으로 교체**하는 쪽이 쿼리가 하나 적다.
- 값 콜렉션 항목에는 별도 식별자가 없으므로 개별 `UPDATE`는 하지 않고, **삭제 후 삽입** 방식으로 처리된다.

## 4. @Embeddable 타입의 Set 매핑 – `Role2`

값이 단순 문자열이 아니라 **여러 필드를 가진 값 객체**일 때 (`perm` + `grantor`).

### 값 타입 – `GrantedPermission`
```java
@Embeddable
public class GrantedPermission {
    @Column(name = "perm")
    private String permission;
    private String grantor;

    protected GrantedPermission() {}

    public GrantedPermission(String permission, String grantor) { ... }

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (o == null || getClass() != o.getClass()) return false;
        GrantedPermission that = (GrantedPermission) o;
        return Objects.equals(permission, that.permission) && Objects.equals(grantor, that.grantor);
    }

    @Override
    public int hashCode() {
        return Objects.hash(permission, grantor);
    }
}
```

### 엔티티 – `Role2`
```java
@Entity
@Table(name = "role")
public class Role2 {
    @Id
    private String id;
    private String name;

    @ElementCollection
    @CollectionTable(
            name = "role_perm",
            joinColumns = @JoinColumn(name = "role_id")
    )
    private Set<GrantedPermission> permissions = new HashSet<>();
    ...
}
```
- 요소가 `@Embeddable`이므로 **`@Column`을 컬렉션 필드에 붙이지 않는다.** 컬럼 매핑은 `GrantedPermission` 내부의 설정(`perm`, `grantor`)을 사용한다.
- 같은 `role` 테이블에 매핑되는 `Role`(String Set)과 `Role2`(값 객체 Set)를 **비교용으로 따로** 만들어 둔 예제이며, 둘 다 persistence.xml에 등록되어 있다.

### ⚠ equals / hashCode 구현 필수
- `Set`은 요소의 `equals()/hashCode()`로 **같은 요소인지 판단**한다.
- 값 객체는 식별자가 없어서, JPA도 이 비교로 **어떤 요소가 추가/삭제되었는지** 알아낸다.
- 구현하지 않으면 같은 값을 가진 객체가 서로 다른 요소로 취급되어 **삭제·갱신이 의도대로 동작하지 않는다.**
- 모든 필드를 기준으로 구현 (예제: `permission`, `grantor`)

### 동작 – `MainRole2`
```java
// 저장
new Role2(roleId, "관리자", Set.of(
        new GrantedPermission("F1", "admin1"),
        new GrantedPermission("F2", "admin1")));

// 요소 변경: 추가 + 삭제
role.getPermissions().add(new GrantedPermission("F3", "admin1"));
Optional<GrantedPermission> permOpt = role.getPermissions().stream()
        .filter(p -> "F1".equals(p.getPermission())).findFirst();
permOpt.ifPresent(p -> role.getPermissions().remove(p));

// 새 Set 할당
role.setPermissions(Set.of(new GrantedPermission("F3", "admin1"),
                           new GrantedPermission("F4", "admin1")));

// 전체 삭제
this.permissions = new HashSet<>();
```
- 삭제할 요소를 `permission` 값(F1)으로 **먼저 찾은 뒤 그 객체를 `remove()`** 한다. 값 객체는 `equals()` 기준으로 같은 요소가 지워지므로, 찾은 객체를 그대로 넘기는 방식이 안전하다.
- 저장·조회·수정 흐름은 `MainRole`(String Set)과 동일하고, `role_perm`에 `perm`과 `grantor`가 함께 저장된다.

## 5. 비교 정리

| 구분 | `Set<String>` (`Role`) | `Set<@Embeddable>` (`Role2`) |
|---|---|---|
| 요소 | 단순 값 | 여러 필드를 가진 값 객체 |
| 컬럼 지정 | 컬렉션 필드에 `@Column(name="perm")` | 값 객체 내부의 `@Column` |
| equals/hashCode | String이 이미 구현 | **직접 구현 필수** |
| 테이블 | `role_perm` | `role_perm` (컬럼이 더 많음) |

## 6. 알아둘 점

- 값 콜렉션은 **소유 엔티티에 완전히 종속**: 엔티티가 삭제되면 콜렉션 테이블의 행도 함께 삭제되고, 콜렉션 요소는 독립적으로 `persist()/find()` 할 수 없다.
- 요소가 많거나 요소를 **독립적으로 조회·수정**해야 한다면 값 콜렉션보다 **별도 엔티티 + 연관 매핑**이 적합하다.
- **Set의 equals 기준과 테이블 PK 기준을 맞춰야** 한다. 예제의 `Role2`는 `equals`가 `permission + grantor`를 쓰는 반면 DDL의 PK는 `(role_id, perm)`이므로, 같은 `perm`에 다른 `grantor`를 넣으면 PK 충돌이 날 수 있다 (이 부분은 코드를 보고 정리한 주의점이며 직접 실행해 확인한 것은 아님).

## 7. 핵심 정리

| 항목 | 내용 |
|---|---|
| 값 콜렉션 매핑 | `@ElementCollection` + `@CollectionTable` |
| 단순 값 | `@Column`으로 요소 컬럼 지정 |
| 값 객체 | `@Embeddable` 타입, **equals/hashCode 필수** |
| 저장 | 엔티티 + 콜렉션 요소 INSERT |
| 조회 | 기본 **Lazy** (접근 시 조회), `fetch = EAGER`로 즉시 로딩 |
| 수정 | 요소 추가/삭제 → INSERT/DELETE, 새 Set 할당 → 삭제 후 삽입 |
| 전체 삭제 | `clear()`는 SELECT 후 DELETE, 새 Set 할당은 DELETE만 |
