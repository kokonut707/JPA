# Chapter 5. 엔티티 식별자 생성 방식

## 1. 개요

식별자(@Id)를 만드는 4가지 방식

| 방식 | 사용 | 예제 엔티티 | DB |
|---|---|---|---|
| 직접 할당 | `@Id` 만 사용 | `User`(email), `Hotel`(id) | 전부 |
| 식별 컬럼 | `GenerationType.IDENTITY` | `Review` | MySQL (auto_increment) |
| 시퀀스 | `@SequenceGenerator` | `ActivityLog` | **Oracle** |
| 테이블 | `@TableGenerator` | `AccessLog` | MySQL (`id_seq` 테이블) |

- 식별자 타입은 **래퍼 타입(`Long`)** → 저장 전 `null`로 "식별자 없음" 표현

## 2. 직접 할당

```java
@Entity
public class Hotel {
    @Id
    @Column(name = "hotel_id")
    private String id;          // new Hotel("H-01", ...) 처럼 직접 지정
}
```
- `persist()` 전에 **반드시** 값을 지정 (없으면 예외)
- 이메일, 호텔ID, 주문번호, UUID 등

## 3. 식별 컬럼 (IDENTITY)

### DDL
```sql
create table jpabegin.review (
  review_id integer not null auto_increment,
  hotel_id varchar(50) not null,
  mark integer,
  writer_name varchar(20),
  `comment` text,
  created datetime
) engine innodb character set utf8mb4;
```

### 엔티티
```java
@Entity
public class Review {
    @Id
    @Column(name = "review_id")
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "hotel_id")
    private String hotelId;
    private int mark;
    @Column(name = "writer_name")
    private String writerName;
    private String comment;
    private LocalDateTime created;

    public Review(int mark, String hotelId, String writerName, String comment) {
        ...        // id는 넣지 않는다
    }
}
```
- `@Table`을 생략 → 테이블명 `review` (엔티티 이름)
- 예약어인 `comment`는 DDL에서 백틱(`` ` ``)으로 감쌈

### 동작 – `MainNative`
```java
tx.begin();
Review review = new Review(5, "H-01", "작성자", "댓글");
logger.info("persist 실행 전");
em.persist(review);
logger.info("persist 실행 함");
logger.info("생성한 식별자: {}", review.getId());   // 이미 값이 채워져 있음
logger.info("커밋하기 전");
tx.commit();
logger.info("커밋함");
```
- **`persist()` 호출 즉시 INSERT 실행** → DB가 만든 값을 받아 `id`에 채움
  (로그: `persist 실행 전` → `insert into review ...` → `persist 실행 함` → `생성한 식별자: N`)
- 쓰기 지연이 동작하지 않음, JDBC batch insert 불가

## 4. 시퀀스 (SEQUENCE) – `ActivityLog` (Oracle)

### DDL (Oracle, `ddl-oracle.sql`)
```sql
create table crm.activity_log (
    id integer not null,
    user_id varchar2(50),
    activity_type varchar2(50),
    created DATE
);
ALTER TABLE crm.activity_log ADD CONSTRAINT al_idx01 PRIMARY KEY (id);

create sequence crm.activity_seq
start with 1 increment by 1 nomaxvalue;
```

### 엔티티
```java
@Entity
@Table(schema = "crm", name = "activity_log")
public class ActivityLog {
    @Id
    @SequenceGenerator(
            name = "log_seq_gen",
            sequenceName = "activity_seq",
            schema = "crm",
            allocationSize = 1
    )
    @GeneratedValue(generator = "log_seq_gen")
    private Long id;
    @Column(name = "user_id")
    private String userId;
    @Column(name = "activity_type")
    private String activityType;
    private LocalDateTime created;
}
```

### 설정 포인트
- **MySQL은 시퀀스 미지원** → 별도 persistence-unit `oracle` 사용
```xml
<persistence-unit name="oracle" transaction-type="RESOURCE_LOCAL">
    <class>jpabasic.reserve.domain.ActivityLog</class>
    <properties>
        <property name="jakarta.persistence.jdbc.driver" value="oracle.jdbc.OracleDriver"/>
        <property name="jakarta.persistence.jdbc.url" value="jdbc:oracle:thin:@localhost:1521:xe"/>
        <property name="hibernate.dialect" value="org.hibernate.dialect.OracleDialect"/>
        ...
    </properties>
</persistence-unit>
```
- Oracle JDBC 드라이버 의존성 추가 (`com.oracle.database.jdbc:ojdbc11:21.1.0.0`)
- `EMFOracle` 클래스: `Persistence.createEntityManagerFactory("oracle")` 로 별도 EMF 생성
- `schema = "crm"` 으로 스키마 지정 (`@Table`, `@SequenceGenerator` 모두)

### 동작 – `MainSequence`
- `persist()` 시점에 **시퀀스 다음 값 조회(`select crm.activity_seq.nextval ...`)** 후 id 할당 → 이 때는 INSERT 안 나감
- INSERT는 **커밋(flush) 시점**까지 지연 (IDENTITY와 차이)
- `persist` 직후 `log.getId()` 로 식별자 확인 가능

### ⚠ 주의: allocationSize
- **`allocationSize`는 DB 시퀀스의 `INCREMENT BY` 값과 같아야 한다.**
- 예제는 시퀀스 `increment by 1` → `allocationSize = 1`로 일치
- `allocationSize` 기본값은 **50**이므로, 생략하면서 시퀀스는 1씩 증가하면 ID 오류/중복 발생
- 값이 크면 DB 호출 횟수가 줄어 성능에 유리 (이 경우 시퀀스도 같은 값으로 생성)

## 5. 테이블 (TABLE) – `AccessLog`

### DDL
```sql
create table jpabegin.access_log (
  id integer not null,
  path varchar(200),
  accessed datetime
) engine innodb character set utf8mb4;

create table id_seq (            -- 키 생성 전용 테이블
    entity varchar(100) not null primary key,
    nextval bigint
) engine innodb character set utf8mb4;
```

### 엔티티
```java
@Entity
@Table(name = "access_log")
public class AccessLog {
    @Id
    @TableGenerator(
            name = "accessIdGen",
            table = "id_seq",
            pkColumnName = "entity",
            pkColumnValue = "accesslog",
            valueColumnName = "nextval",
            initialValue = 0,
            allocationSize = 1
    )
    @GeneratedValue(generator = "accessIdGen")
    private Long id;
    private String path;
    private LocalDateTime accessed;
}
```

| 속성 | 의미 |
|---|---|
| `table` | 키 생성용 테이블명 (`id_seq`) |
| `pkColumnName` | 엔티티(시퀀스) 이름을 담는 컬럼 (`entity`) |
| `pkColumnValue` | 이 엔티티에 해당하는 행의 키 값 (`accesslog`) |
| `valueColumnName` | 다음 값을 저장하는 컬럼 (`nextval`) |
| `initialValue` | 시작 값 (0) |
| `allocationSize` | 한 번에 확보할 개수 (1) |

### 동작 – `MainTableGenId`
- `persist()` 시 `id_seq` 테이블을 조회/갱신해서 식별자 확보 → INSERT는 **커밋 시점**
- 모든 DB에서 사용 가능, 대신 테이블 조회/갱신(락) 때문에 **성능은 시퀀스보다 불리**
- `allocationSize`를 키우면 DB 접근 횟수 감소
- 참고: 예제 `MainTableGenId`에는 `tx.commit()`이 두 번 있는데, 두 번째 호출은 불필요(이미 커밋된 트랜잭션)

## 6. AUTO

```java
@GeneratedValue   // = GenerationType.AUTO
```
- DB 방언에 맞춰 구현체가 선택. 환경에 따라 결과가 달라질 수 있어 **전략을 명시**하는 편이 안전
- 예제에서는 `generator`만 지정하는 방식(`@GeneratedValue(generator = "...")`)으로 `@SequenceGenerator`/`@TableGenerator`와 연결

## 7. 전략 비교

| 항목 | 직접 할당 | IDENTITY | SEQUENCE | TABLE |
|---|---|---|---|---|
| 키 생성 주체 | 앱 | DB 컬럼 | DB 시퀀스 | 키 테이블 |
| persist 시 id 확보 | 직접 | **INSERT 직후** | nextval 조회 | 테이블 조회 |
| INSERT 시점 | flush | **persist 즉시** | flush | flush |
| 쓰기 지연 / JDBC 배치 | O | **X** | O | O |
| 지원 DB | 전부 | MySQL 등 | Oracle, PostgreSQL 등 | 전부 |
| 성능 | 좋음 | 보통 | 좋음 | 낮음 |

## 8. 선택 가이드

- **MySQL**: `IDENTITY` 가 가장 단순. 대량 INSERT 성능이 중요하면 `TABLE`(allocationSize 확대) 또는 직접 할당
- **Oracle / PostgreSQL**: `SEQUENCE` (allocationSize와 INCREMENT BY 일치 필수)
- **비즈니스 값/분산 환경**: 직접 할당(UUID 등)

## 9. 정리

- `IDENTITY`: persist 즉시 INSERT → 식별자도 즉시 확보
- `SEQUENCE`: nextval 조회 후 flush 때 INSERT, `allocationSize` 일치 필수
- `TABLE`: 전용 테이블(`id_seq`)로 시퀀스 흉내, 범용이지만 느림
- DB가 다르면 persistence-unit/EMF를 따로 구성 (`jpabegin` / `oracle`)
