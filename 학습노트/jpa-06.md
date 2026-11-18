# Chapter 6. @Embeddable

## 1. 왜 필요한가?

테이블 컬럼은 평평하게 나열되지만, 객체에서는 **의미 있는 단위로 묶고 싶다.**

```
[hotel_info 테이블]  hotel_id, nm, year, grade, addr1, addr2, zipcode, created, modified
[객체]               Hotel ─ Address(address1, address2, zipcode)
```
- 필드가 많아지면 응집도가 떨어짐 → **값 타입(Value Type)** 으로 분리
- 값 타입에는 `@Id`가 없고, 소유 엔티티와 **같은 테이블**에 매핑됨 (테이블 추가 없음)

## 2. 기본 사용법

### @Embeddable – `jpabasic.common.Address`
```java
@Embeddable
public class Address {
    @Column(name = "addr1")
    private String address1;
    @Column(name = "addr2")
    private String address2;
    @Column(name = "zipcode")
    private String zipcode;

    protected Address() {          // 기본 생성자 필수
    }

    public Address(String address1, String address2, String zipcode) {
        this.address1 = address1;
        this.address2 = address2;
        this.zipcode = zipcode;
    }
    // getter만 제공 (setter 없음 → 불변), toString
}
```
- 값 타입 안에서 `@Column`으로 컬럼명 지정 가능
- **setter 없이 getter만** → 불변 객체

### @Embedded – `Hotel`
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

    @Embedded
    private Address address;      // addr1, addr2, zipcode 컬럼으로 매핑

    private LocalDateTime created;
    @Column(name = "modified")
    private LocalDateTime lastModified;
    ...
}
```

### DDL (`hotel_info`에 주소 컬럼 추가)
```sql
create table jpabegin.hotel_info (
  hotel_id varchar(50) not null primary key,
  nm varchar(50),
  year int,
  grade varchar(2),
  addr1 varchar(100),
  addr2 varchar(100),
  zipcode varchar(5),
  created datetime,
  modified datetime
) engine innodb character set utf8mb4;
```

### 사용 – `MainEmbeddable`
```java
Address address = new Address("주소1", "주소2", "12345");
Hotel hotel = new Hotel("H00", "HN", 2022, Grade.S7, address);
em.persist(hotel);
...
Hotel hotel = em.find(Hotel.class, "H00");
logger.info("주소: {}", hotel.getAddress());
```
- `Address` 객체 하나가 `addr1`, `addr2`, `zipcode` **세 컬럼**으로 저장/조회됨

### 규칙
1. **기본 생성자 필수** (`protected` 가능)
2. 엔티티가 아니므로 **`@Id` 없음**, 단독으로 `persist()` 불가
3. 소유 엔티티의 생명주기를 따른다
4. `@Embedded`는 생략 가능하지만 명시하면 가독성이 좋다
5. persistence.xml의 `<class>` 등록은 엔티티만 한다 (예제: `Employee`, `Hotel`)

## 3. 같은 타입을 두 번 쓸 때 – @AttributeOverride

### 문제
`Employee`가 집 주소와 직장 주소, **`Address` 타입 필드 2개**를 가지면 둘 다 `addr1/addr2/zipcode` 컬럼에 매핑되어 **컬럼명 중복 오류**.

### 해결 – `Employee`
```java
@Entity
public class Employee {
    @Id
    private String id;

    @Embedded
    private Address homeAddress;          // addr1, addr2, zipcode (기본 매핑)

    @AttributeOverrides({
            @AttributeOverride(name = "address1", column = @Column(name = "waddr1")),
            @AttributeOverride(name = "address2", column = @Column(name = "waddr2")),
            @AttributeOverride(name = "zipcode",  column = @Column(name = "wzipcode"))
    })
    @Embedded
    private Address workAddress;          // waddr1, waddr2, wzipcode 로 재정의
}
```
- `name`: **값 타입 안의 필드명** (`address1`, `address2`, `zipcode`)
- `column`: 실제 컬럼으로 재정의
- `homeAddress`는 `Address`의 `@Column` 설정(addr1…)을 그대로 사용

### DDL
```sql
create table jpabegin.employee (
  id varchar(10) not null primary key,
  addr1 varchar(100),
  addr2 varchar(100),
  zipcode varchar(5),
  waddr1 varchar(100),
  waddr2 varchar(100),
  wzipcode varchar(5)
) engine innodb character set utf8mb4;
```

### 사용 – `MainAttrOver`
```java
Address home = new Address("집주소1", "집주소2", "12345");
Address work = new Address("직장주소1", "직장주소2", "68890");
Employee employee = new Employee("EMP01", home, work);
em.persist(employee);
```

## 4. null 처리 – `MainNullEmbedded`

```java
em.persist(new Hotel("H009", "HN", 2022, Grade.S7, null));   // 주소 없이 저장
...
Hotel hotel = em.find(Hotel.class, "H009");
logger.info("주소: {}", hotel.getAddress());
```
- 임베디드 객체가 `null` → `addr1`, `addr2`, `zipcode` **모두 NULL**로 저장
- 조회 시 **모든 컬럼이 NULL이면 임베디드 객체도 `null`** 로 로딩 (빈 `Address` 객체가 아님)
- → `hotel.getAddress()` 사용 전 null 체크 필요

## 5. 설계 시 주의점

### 불변 객체
- 여러 엔티티가 **같은 `Address` 인스턴스를 공유**하면 한쪽 변경이 다른 쪽에 영향 → 값 타입은 **불변**으로 (예제: setter 없음)
- 변경이 필요하면 **새 객체로 교체**

### equals / hashCode
- 값 타입은 값이 같으면 같은 것 → 필요 시 모든 필드 기준으로 `equals()`/`hashCode()` 재정의 (예제의 `Address`에는 아직 없음)

### 그 외
- 값 타입 안에 `@Enumerated` 등 다른 매핑 어노테이션 사용 가능
- 값 타입을 다른 테이블에 매핑하는 방법(`@CollectionTable` 등)은 이후 챕터에서 다룬다

## 6. 정리

| 어노테이션 | 위치 | 역할 |
|---|---|---|
| `@Embeddable` | 값 타입 클래스 | 엔티티에 포함될 수 있는 타입 선언 |
| `@Embedded` | 엔티티 필드 | 값 타입 사용 |
| `@AttributeOverrides` / `@AttributeOverride` | 엔티티 필드 | 값 타입의 컬럼 매핑 재정의 |

- 값 타입 = 식별자 없음, 소유 엔티티와 **한 테이블**, 생명주기 종속
- 같은 타입을 중복 사용하면 **컬럼명 충돌** → `@AttributeOverride`
- 임베디드가 `null`이면 컬럼이 전부 NULL, 모두 NULL이면 조회 시 `null`
