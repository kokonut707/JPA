# Chapter 7. @Embeddable 객체를 다른 테이블에 매핑 (@SecondaryTable)

## 1. 모델링 배경

- 하나의 엔티티(`Writer`) 데이터가 **여러 테이블**(`writer`, `writer_intro`, `writer_address`)에 나뉘어 저장되어 있는 경우
- 각 테이블을 별도 엔티티로 만들지 않고, 소개(`Intro`)와 주소(`Address`)를 **값 객체(`@Embeddable`)** 로 모델링
- Chapter 6에서는 값 타입이 **소유 엔티티와 같은 테이블**에 매핑됐지만, 이번에는 값 타입을 **다른 테이블**에 매핑

```
Writer (엔티티)
 ├─ id, name                → writer 테이블
 ├─ Address (@Embeddable)   → writer_address 테이블
 └─ Intro   (@Embeddable)   → writer_intro 테이블
```

### 테이블 (`ddl.sql`)
```sql
create table jpabegin.writer (
  id integer not null auto_increment primary key,
  name varchar(100)
) engine innodb character set utf8mb4;

create table jpabegin.writer_intro (
  writer_id integer not null primary key,   -- writer.id 와 같은 값 (PK이자 FK 역할)
  content_type varchar(50),
  content varchar(4000)
) engine innodb character set utf8mb4;

create table jpabegin.writer_address (
  writer_id integer not null primary key,
  addr1 varchar(100),
  addr2 varchar(100),
  zipcode varchar(5)
) engine innodb character set utf8mb4;
```
- 보조 테이블의 PK(`writer_id`)가 메인 테이블 `writer.id`와 **1:1로 대응**

## 2. @SecondaryTable

엔티티에 **추가 테이블**을 지정하는 어노테이션. 여러 개면 `@SecondaryTables`로 묶는다.

```java
@Entity
@Table(name = "writer")
@SecondaryTables({
        @SecondaryTable(name = "writer_address",
                pkJoinColumns = @PrimaryKeyJoinColumn(name = "writer_id", referencedColumnName = "id")
        ),
        @SecondaryTable(name = "writer_intro",
                pkJoinColumns = @PrimaryKeyJoinColumn(name = "writer_id", referencedColumnName = "id")
        )}
)
public class Writer { ... }
```

| 속성 | 의미 |
|---|---|
| `@SecondaryTable(name)` | 추가로 매핑할 테이블명 |
| `pkJoinColumns` | 보조 테이블의 PK 컬럼이 메인 테이블의 어떤 컬럼과 조인되는지 지정 |
| `@PrimaryKeyJoinColumn(name)` | **보조 테이블 쪽** PK 컬럼 (`writer_id`) |
| `@PrimaryKeyJoinColumn(referencedColumnName)` | **메인 테이블 쪽** PK 컬럼 (`id`) |

## 3. 매핑 방식 2가지

### 방식 1. @Embeddable 클래스의 @Column에 table 지정 (직접 명시) — `Intro`
```java
@Embeddable
public class Intro {
    @Column(table = "writer_intro", name = "content_type")
    private String contentType;

    @Column(table = "writer_intro")      // name 생략 → 필드명 content
    private String content;
    ...
}
```
```java
@Entity
@SecondaryTables(...)
public class Writer {
    @Embedded
    private Intro intro;                 // 따로 설정 없이 사용
}
```
- 값 타입 **자체에** 어느 테이블인지 박아 둠
- `Intro`는 `Writer`에서만 쓰이므로 이 방식이 간단

### 방식 2. @AttributeOverride로 엔티티 쪽에서 table 지정 — `Address`
```java
@Embeddable
public class Address {                   // table 지정 없음 → 여러 곳에서 재사용 가능
    @Column(name = "addr1")
    private String address1;
    @Column(name = "addr2")
    private String address2;
    @Column(name = "zipcode")
    private String zipcode;
    ...
}
```
```java
@Embedded
@AttributeOverrides({
        @AttributeOverride(name = "address1", column = @Column(table = "writer_address", name = "addr1")),
        @AttributeOverride(name = "address2", column = @Column(table = "writer_address", name = "addr2")),
        @AttributeOverride(name = "zipcode",  column = @Column(table = "writer_address"))
})
private Address address;
```
- 값 타입은 그대로 두고, **사용하는 엔티티가** 어느 테이블에 매핑할지 `@AttributeOverride`의 `column(table=...)`로 지정
- `Address`처럼 **여러 엔티티에서 재사용**되는 값 객체에 적합 (Chapter 6의 `@AttributeOverride`와 같은 문법에 `table`만 추가)

| 구분 | 방식 1 (`@Column`에 직접) | 방식 2 (`@AttributeOverride`) |
|---|---|---|
| table 지정 위치 | 값 타입 클래스 안 | 엔티티의 `@Embedded` 필드 |
| 적합한 경우 | 한 엔티티 전용 값 타입 | 여러 곳에서 재사용되는 값 타입 |
| 엔티티 코드 | 간단 (`@Embedded`만) | 필드마다 override 필요 |

## 4. 전체 엔티티 – `Writer`

```java
@Entity
@Table(name = "writer")
@SecondaryTables({ ... })
public class Writer {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private String name;

    @Embedded
    @AttributeOverrides({ ... })         // 방식 2
    private Address address;

    @Embedded
    private Intro intro;                 // 방식 1

    protected Writer() {}

    public Writer(String name, Address address, Intro intro) { ... }
    ...
}
```
- persistence.xml에는 엔티티 `Writer`만 `<class>`로 등록 (값 타입은 등록 불필요)

## 5. 데이터 조작 시 동작 – `MainEmb2`

### 저장
```java
Writer w = new Writer("name",
        new Address("주소1", "주소2", "12345"),
        new Intro("text/plain", "소개글"));
em.persist(w);
tx.commit();
return w.getId();
```
- `INSERT`가 **연관된 모든 테이블**(`writer`, `writer_address`, `writer_intro`)에 실행됨
- `id`는 IDENTITY 전략이므로 `persist()` 시점에 `writer`가 먼저 INSERT되고, 발급된 id가 보조 테이블의 `writer_id`로 사용됨

### 조회
```java
Writer writer = em.find(Writer.class, id);
```
- 보조 테이블에 행이 없을 수도 있으므로, 데이터 누락을 막기 위해 **`LEFT JOIN`** 으로 한 번에 조회

### 수정
```java
Writer writer = em.find(Writer.class, id);
writer.getIntro().setContent("변경");                              // 값 객체 내부 변경
writer.setAddress(new Address("새주소1", "새주소2", "11111"));     // 값 객체 교체
tx.commit();
```
- 변경 감지로 해당 보조 테이블에 **UPDATE** 실행
- 기존 값이 있으면 `UPDATE`, 보조 테이블에 행이 없던 경우에는 `INSERT`를 수행
- `Address`는 불변 객체이므로 **새 객체로 교체**, `Intro`는 setter로 내부 값을 변경하는 두 방식이 예제에 모두 나온다.

### 삭제
```java
Writer writer = em.find(Writer.class, id);
em.remove(writer);
```
- 엔티티를 삭제하면 **`@SecondaryTable`로 매핑된 테이블의 행도 함께 삭제**됨

## 6. 정리

| 항목 | 내용 |
|---|---|
| 목적 | 한 엔티티의 데이터가 여러 테이블에 나뉘어 있을 때, 값 객체로 모델링 |
| 핵심 어노테이션 | `@SecondaryTable(s)`, `@PrimaryKeyJoinColumn`, `@Column(table=...)` |
| 직접 지정 | 값 타입의 `@Column`에 `table` 속성 |
| 재사용 값 타입 | 엔티티의 `@AttributeOverride`에서 `table` 속성 |
| 저장 | 모든 테이블에 INSERT |
| 조회 | `LEFT JOIN` |
| 수정 | 있으면 UPDATE / 없으면 INSERT |
| 삭제 | 보조 테이블 행도 함께 삭제 |

- 보조 테이블의 PK는 메인 테이블의 PK와 **같은 값**을 가진다 (`writer_id` = `writer.id`).
- 보조 테이블 데이터는 **엔티티가 아니라 값 객체**이므로 독립적인 식별자·생명주기가 없고, `Writer`에 종속된다.
