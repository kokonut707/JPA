# Chapter 4. 엔티티 매핑 설정

## 1. 예제로 보는 매핑

### 테이블 (`ddl.sql`)
```sql
create table jpabegin.hotel_info (
  hotel_id varchar(50) not null primary key,
  nm varchar(50),
  year int,
  grade varchar(2),
  created datetime,
  modified datetime
) engine innodb character set utf8mb4;
```

### enum – `Grade`
```java
public enum Grade {
    S1, S2, S3, S4, S5, S6, S7
}
```

### 엔티티 – `Hotel`
```java
@Entity
@Table(name = "hotel_info")
public class Hotel {
    @Id
    @Column(name = "hotel_id")
    private String id;

    @Column(name = "nm")
    private String name;

    private int year;

    @Enumerated(EnumType.STRING)
    private Grade grade;

    private LocalDateTime created;

    @Column(name = "modified")
    private LocalDateTime lastModified;

    protected Hotel() {
    }

    public Hotel(String id, String name, int year, Grade grade) {
        this.id = id;
        this.name = name;
        this.year = year;
        this.grade = grade;
        this.created = LocalDateTime.now();
        this.lastModified = LocalDateTime.now();
    }
    // getter ..., toString
}
```

### 필드 ↔ 컬럼 매핑표
| 필드 | 컬럼 | 매핑 방법 |
|---|---|---|
| `id` | `hotel_id` | `@Id` + `@Column(name)` |
| `name` | `nm` | `@Column(name = "nm")` |
| `year` | `year` | 어노테이션 없음 (이름 동일) |
| `grade` | `grade` | `@Enumerated(EnumType.STRING)` |
| `created` | `created` | 이름 동일 |
| `lastModified` | `modified` | `@Column(name = "modified")` |

### 사용
```java
EMF.init();
Hotel entity = new Hotel("H-01", "말", 2022, Grade.S5);
save(entity);                       // persist
Hotel h2 = read("H-01");            // find
logger.info("있음: {}", h2);
EMF.close();
```
- `created`, `lastModified`는 **생성자에서 값을 채운다** (JPA가 자동으로 넣어주는 것이 아님)
- 저장 후 `grade` 컬럼에는 `"S5"` 문자열이 들어감 (`varchar(2)`)

## 2. 주요 어노테이션

### @Entity
```java
@Entity                    // 엔티티 이름 = 클래스 단순 이름
@Entity(name = "Member")   // JPQL에서 쓸 이름 지정
```
- persistence.xml의 `<class>`에 등록 (예제: `<class>jpabasic.reserve.domain.Hotel</class>`)

### @Table
```java
@Table(name = "hotel_info")        // 테이블명 지정
@Table(schema = "crm", name = "activity_log")   // 스키마 지정
```
- 생략하면 **엔티티 이름 = 테이블 이름**
- 그 외 속성: `catalog`, `uniqueConstraints`, `indexes` (DDL 생성용)

### @Id – 식별자 지정

**`@Entity`로 선언한 모든 엔티티에는 `@Id`가 필수.** 없으면 에러가 난다.
```java
@Entity
public class User {
    // @Id가 없으면 → "No identifier specified for entity: ...User"
    @Id
    private String email;
}
```

#### 왜 필요한가
식별자는 JPA가 엔티티를 다루는 기준이다.
- `em.find(User.class, "a@a.com")` : 어떤 필드가 PK인지 알아야 조회 가능
- **1차 캐시** : 영속 컨텍스트가 `식별자 → 엔티티` Map이라서 키가 필요
- **동일성 보장** : "같은 ID면 같은 인스턴스"의 기준
- **변경 감지 / 삭제** : `UPDATE/DELETE ... WHERE email=?` 의 WHERE 조건

#### 필요한 경우와 아닌 경우
| 대상 | `@Id` |
|---|---|
| `@Entity` 클래스 | **필수** |
| `@Embeddable` 값 타입 | 불필요 (식별자가 없는 것이 특징) |
| 상속 관계의 자식 엔티티 | 부모 쪽에 있으면 상속받으므로 불필요 |
| `@MappedSuperclass` 부모 | 공통 PK를 여기에 두는 경우가 많음 |
| 일반 클래스, DTO | 불필요 |

#### DB의 PK와의 관계
**테이블의 PK 컬럼에 대응하는 필드에 `@Id`를 붙인다.**
```java
// DDL: hotel_id varchar(50) not null primary key
@Id
@Column(name = "hotel_id")
private String id;
```
- `@Id`는 **JPA 쪽 선언**이고, DB의 PK 제약과는 별개다. JPA가 DB에 PK가 실제로 걸려 있는지 검사하지는 않는다.
    - DDL에 `primary key`가 없어도 엔티티에서 `id`에 `@Id`를 붙여 동작이 가능하지만, 중복 ID가 들어갈 수 있어 권장되지 않는다.
    - `hbm2ddl.auto`로 테이블을 자동 생성하면 `@Id` 기준으로 PK가 만들어진다.
- 반대로 `@Id` 필드는 **DB에서도 행을 유일하게 식별하는 값**이어야 한다. 중복이 있으면 같은 식별자의 엔티티가 여러 개가 되어 1차 캐시, 변경 감지, 삭제가 꼬인다.
- PK가 컬럼 여러 개(복합 키)이면 `@IdClass` 또는 `@EmbeddedId`를 사용 (이후에 다룸)
- PK(유일 식별 컬럼)가 전혀 없는 테이블은 엔티티로 매핑하기 어렵다.

### @Id / @Column
```java
@Id
@Column(name = "hotel_id")
private String id;
```
| 속성 | 기본값 | 설명 |
|---|---|---|
| `name` | 필드명 | 컬럼명 |
| `nullable` | true | NOT NULL 여부 (DDL용) |
| `length` | 255 | 문자열 길이 (DDL용) |
| `unique` | false | 유니크 제약 (DDL용) |
| `insertable` / `updatable` | true | INSERT/UPDATE 포함 여부 |
| `columnDefinition` | | 컬럼 DDL 직접 지정 |
| `precision`, `scale` | | BigDecimal 자릿수 |

- `@Column`을 **생략해도** 필드는 같은 이름의 컬럼에 매핑됨 (`year`, `created`)
- 매핑에서 제외하려면 `@Transient`
- 카멜 케이스 필드(`lastModified`)는 자동으로 스네이크 케이스로 바뀌지 않음 → 이름이 다르면 `@Column`으로 지정

### @Enumerated
```java
@Enumerated(EnumType.STRING)
private Grade grade;
```
| 타입 | 저장 값 | 비고 |
|---|---|---|
| `EnumType.ORDINAL` (기본) | 순서 (0, 1, 2 …) | enum 순서가 바뀌면 데이터 오염 ✘ |
| `EnumType.STRING` | 이름 ("S5") | 안전, 가독성 좋음 ✔ |

→ **항상 `EnumType.STRING`** (예제 DDL도 `grade varchar(2)`)

### 자바 타입 매핑 (예제 기준)
| 자바 | DB |
|---|---|
| `String` | varchar |
| `int` | int |
| `LocalDateTime` | datetime |
| enum (`STRING`) | varchar |

> `year` 같은 기본형(`int`)은 NULL 컬럼과 매핑 시 문제가 될 수 있어, null 가능성이 있으면 `Integer` 사용

## 3. 기본 생성자 제약

**엔티티 요구사항**
1. **기본 생성자(파라미터 없는 생성자)** 필수 — `public` 또는 `protected`
2. 클래스는 **final 불가**
3. 메서드와 영속 필드는 final 불가

```java
protected Hotel() {          // JPA용. 외부에서 사용 금지
}

public Hotel(String id, String name, int year, Grade grade) { ... }   // 개발자용
```

### 이유
- JPA 구현체가 DB 결과로 엔티티를 만들 때 **리플렉션으로 기본 생성자 호출**
- 프록시(지연 로딩, `getReference`)가 엔티티를 **상속**해 만들어지므로 final이면 불가
- 다른 생성자를 만들면 컴파일러가 기본 생성자를 자동 생성하지 않으므로 **직접 추가**해야 한다

## 4. 필드 접근 vs 프로퍼티 접근

`@Id`를 **어디에 붙였는지**로 결정된다.

### 필드 접근
```java
@Id
@Column(name = "hotel_id")
private String id;           // 필드에 매핑 → 필드 접근
```
- getter/setter와 무관하게 필드 값을 직접 읽고 씀
- getter에 로직이 있어도 영향 없음

### 프로퍼티 접근
```java
@Id
@Column(name = "hotel_id")
public String getId() { return id; }    // getter에 매핑 → 프로퍼티 접근
```
- getter/setter를 통해 값 접근 → getter/setter에 로직이 있으면 JPA 동작에 영향
- 프로퍼티 이름 = `getName` → `name`

### @Access로 혼합
```java
@Entity
@Access(AccessType.FIELD)
public class Hotel {
    @Access(AccessType.PROPERTY)
    @Column(name = "nm")
    public String getName() { ... }
}
```

| 구분 | 필드 접근 | 프로퍼티 접근 |
|---|---|---|
| @Id 위치 | 필드 | getter |
| 접근 | 필드 직접 | getter/setter |
| 일반성 | **가장 일반적** | 특수한 경우 |

## 5. 정리

- 이름이 다르면 `@Table`/`@Column`으로 매핑 (`hotel_info`, `hotel_id`, `nm`, `modified`)
- enum은 `@Enumerated(EnumType.STRING)`
- 기본 생성자는 `protected` 이상, 클래스/필드/메서드 final 금지
- `@Id`를 필드에 붙여 **필드 접근**으로 통일
