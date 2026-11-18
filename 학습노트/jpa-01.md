# Chapter 1. JPA 시작하기 (JPA 3.0, 스프링 없이)

## 1. 핵심 개념

- **JPA (Jakarta Persistence API)**: 자바 객체와 관계형 DB 테이블을 매핑하는 **ORM 표준 스펙** (인터페이스 모음)
- **구현체**: Hibernate (이번 실습), EclipseLink 등. JPA는 스펙, Hibernate는 실제 동작하는 구현
- **JPA 3.0의 특징**: 패키지가 `javax.persistence` → **`jakarta.persistence`** 로 변경됨 (기능은 2.2와 거의 동일, 네임스페이스만 변경)
- **Hibernate 버전 대응**

  | JPA | Hibernate |
  |---|---|
  | 2.2 (javax) | 5.x |
  | 3.0 (jakarta) | **6.0** ← 실습 버전 (6.0.0.Final) |
  | 3.1 (jakarta) | 6.1 이상 |

> 주의: Hibernate 5.x는 `javax.persistence`, 6.x는 `jakarta.persistence`. 섞어 쓰면 에러 발생.

## 2. 프로젝트 구조

```
JPA
├─ pom.xml
└─ src/main
   ├─ java
   │  ├─ jpabasic/reserve/domain/User.java      ← 엔티티
   │  └─ main
   │     ├─ UserSaveMain.java                   ← 저장
   │     ├─ UserGetMain.java                    ← 조회
   │     └─ UserUpdateMain.java                 ← 수정
   └─ resources/META-INF/persistence.xml        ← JPA 설정 (위치/이름 고정)
```

## 3. 프로젝트 설정 (pom.xml)

```xml
<groupId>jpa-basic</groupId>
<artifactId>jpa-01</artifactId>
<version>1.0-SNAPSHOT</version>

<properties>
    <maven.compiler.source>17</maven.compiler.source>
    <maven.compiler.target>17</maven.compiler.target>
    <hibernate.version>6.0.0.Final</hibernate.version>
</properties>

<dependencies>
    <!-- JPA 3.0 구현체 -->
    <dependency>
        <groupId>org.hibernate</groupId>
        <artifactId>hibernate-core</artifactId>
        <version>${hibernate.version}</version>
    </dependency>
    <!-- 커넥션 풀 (HikariCP 연동) -->
    <dependency>
        <groupId>org.hibernate</groupId>
        <artifactId>hibernate-hikaricp</artifactId>
        <version>${hibernate.version}</version>
    </dependency>
    <!-- JDBC 드라이버 (MySQL) -->
    <dependency>
        <groupId>mysql</groupId>
        <artifactId>mysql-connector-java</artifactId>
        <version>8.0.28</version>
    </dependency>
    <!-- 로깅 -->
    <dependency>
        <groupId>ch.qos.logback</groupId>
        <artifactId>logback-classic</artifactId>
        <version>1.2.11</version>
    </dependency>
</dependencies>
```

| 의존성 | 역할 |
|---|---|
| `hibernate-core` | JPA 3.0 구현체 (`jakarta.persistence` API도 함께 포함) |
| `hibernate-hikaricp` | Hibernate가 **HikariCP 커넥션 풀**을 쓰도록 연결 |
| `mysql-connector-java` | MySQL JDBC 드라이버 |
| `logback-classic` | 로깅 구현체 (Hibernate 로그 출력) |

- Java **17** 사용 (Hibernate 6은 Java 11 이상 필요)

## 4. persistence.xml

- 위치: `src/main/resources/META-INF/persistence.xml`
- JPA 3.0은 **jakarta 네임스페이스**와 `jakarta.persistence.*` 프로퍼티를 사용

```xml
<?xml version="1.0" encoding="utf-8" ?>

<persistence xmlns="https://jakarta.ee/xml/ns/persistence"
             xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
             xsi:schemaLocation="https://jakarta.ee/xml/ns/persistence
                https://jakarta.ee/xml/ns/persistence/persistence_3_0.xsd"
             version="3.0">

    <persistence-unit name="jpabegin" transaction-type="RESOURCE_LOCAL">
        <class>jpabasic.reserve.domain.User</class>
        <exclude-unlisted-classes>true</exclude-unlisted-classes>
        <properties>
            <!-- DB 연결 정보 -->
            <property name="jakarta.persistence.jdbc.driver" value="com.mysql.cj.jdbc.Driver"/>
            <property name="jakarta.persistence.jdbc.url"
                      value="jdbc:mysql://localhost/jpabegin?characterEncoding=utf8"/>
            <property name="jakarta.persistence.jdbc.user" value="jpauser"/>
            <property name="jakarta.persistence.jdbc.password" value="jpapass"/>

            <!-- Hibernate 방언 -->
            <property name="hibernate.dialect" value="org.hibernate.dialect.MySQLDialect"/>

            <!-- HikariCP 커넥션 풀 -->
            <property name="hibernate.hikari.poolName" value="pool"/>
            <property name="hibernate.hikari.maximumPoolSize" value="10"/>
            <property name="hibernate.hikari.minimumIdle" value="10"/>
            <property name="hibernate.hikari.connectionTimeout" value="1000"/> <!-- 1s -->
        </properties>
    </persistence-unit>
</persistence>
```

### 설정 정리
| 설정 | 설명 |
|---|---|
| `persistence-unit name="jpabegin"` | `Persistence.createEntityManagerFactory("jpabegin")` 에서 사용하는 이름 |
| `transaction-type="RESOURCE_LOCAL"` | 애플리케이션이 직접 트랜잭션 관리 (스프링/JTA 없이 쓰는 방식) |
| `<class>` + `exclude-unlisted-classes=true` | 명시한 클래스만 엔티티로 관리 |
| `jakarta.persistence.jdbc.*` | driver / url / user / password (JPA 3.0 표준 프로퍼티) |
| `hibernate.dialect` | DB 방언 (MySQL SQL 문법에 맞춰 SQL 생성) |
| `hibernate.hikari.maximumPoolSize` | 커넥션 풀 최대 크기 (10) |
| `hibernate.hikari.minimumIdle` | 유지할 최소 유휴 커넥션 수 (10 → 풀 크기 고정) |
| `hibernate.hikari.connectionTimeout` | 커넥션 획득 대기 시간(ms) (1000 = 1초) |

> 참고: `hibernate.show_sql=true`를 추가하면 실행 SQL을 콘솔에서 바로 볼 수 있다 (Chapter 2에서는 logback 설정으로 SQL 로그를 출력).

## 5. 테이블과 엔티티 매핑

### 사전 준비
```sql
create database jpabegin CHARACTER SET utf8mb4;

CREATE USER 'jpauser'@'localhost' IDENTIFIED BY 'jpapass';
CREATE USER 'jpauser'@'%' IDENTIFIED BY 'jpapass';
GRANT ALL PRIVILEGES ON jpabegin.* TO 'jpauser'@'localhost';
GRANT ALL PRIVILEGES ON jpabegin.* TO 'jpauser'@'%';

create table jpabegin.user (
  email varchar(50) not null primary key,
  name varchar(50),
  create_date datetime
) engine innodb character set utf8mb4;
```

### 엔티티 – `jpabasic.reserve.domain.User`
```java
@Entity
@Table(name = "user")
public class User {
    @Id
    private String email;
    private String name;

    @Column(name = "create_date")
    private LocalDateTime createDate;

    protected User() {          // JPA가 사용하는 기본 생성자
    }

    public User(String email, String name, LocalDateTime createDate) {
        this.email = email;
        this.name = name;
        this.createDate = createDate;
    }

    public String getEmail() { return email; }
    public String getName() { return name; }
    public LocalDateTime getCreateDate() { return createDate; }

    public void changeName(String newName) {   // setter 대신 의미 있는 변경 메서드
        this.name = newName;
    }
}
```

| 어노테이션 | 역할 |
|---|---|
| `@Entity` | JPA가 관리하는 엔티티 |
| `@Table(name = "user")` | 매핑할 테이블 이름 |
| `@Id` | PK 필드 (`email`) — 직접 할당 방식 |
| `@Column(name = "create_date")` | 필드명과 컬럼명이 다를 때 컬럼명 지정 |

### 매핑 포인트
- `name` 필드처럼 어노테이션이 없어도 **필드명 = 컬럼명**으로 자동 매핑
- `createDate`(카멜 케이스) ↔ `create_date`(스네이크 케이스)는 자동 변환되지 않으므로 `@Column`으로 지정
- `LocalDateTime` ↔ DATETIME/TIMESTAMP 컬럼
- 기본 생성자는 `protected` (JPA 전용), 실제 생성은 파라미터 있는 생성자 사용
- setter가 없고 `changeName()` 같은 **의도가 드러나는 메서드**로만 상태 변경
- `@Id`가 **필드**에 붙어 있으므로 **필드 접근 방식**

## 6. 저장 · 조회 · 수정 시연

세 Main 클래스는 모두 같은 틀을 가진다.

```java
EntityManagerFactory emf = Persistence.createEntityManagerFactory("jpabegin");

EntityManager entityManager = emf.createEntityManager();
EntityTransaction transaction = entityManager.getTransaction();
try {
    transaction.begin();
    // ... JPA 작업
    transaction.commit();
} catch (Exception ex) {
    ex.printStackTrace();
    transaction.rollback();
} finally {
    entityManager.close();
}
emf.close();
```

### 저장 – `UserSaveMain`
```java
User user = new User("user@user.com", "user", LocalDateTime.now());
entityManager.persist(user);
```
- `persist()` → 비영속 → **영속** 상태, 커밋 시점에 `INSERT`
- 같은 email로 다시 실행하면 PK 중복 예외 → catch에서 `rollback()`

### 조회 – `UserGetMain`
```java
User user = entityManager.find(User.class, "user@user.com");
if (user == null) {
    System.out.println("User 없음");
} else {
    System.out.printf("User 있음: email=%s, name=%s, createDate=%s\n",
            user.getEmail(), user.getName(), user.getCreateDate());
}
```
- `find(엔티티클래스, PK)` → 없으면 **null** 반환
- 실습 코드는 조회도 트랜잭션 안에서 실행 (조회만 한다면 트랜잭션 없이도 가능)

### 수정 – `UserUpdateMain`
```java
User user = entityManager.find(User.class, "user@user.com");
if (user == null) {
    System.out.println("User 없음");
} else {
    String newName = "이름" + (System.currentTimeMillis() % 100);
    user.changeName(newName);    // 값만 바꿨는데
}
transaction.commit();            // 커밋 시점에 UPDATE 자동 실행
```
- `em.update()` 같은 메서드는 **없다**. 영속 상태 엔티티의 변경은 커밋(flush) 시 **변경 감지(Dirty Checking)** 로 자동 반영
- 조회한 엔티티가 영속 상태여야 하므로 **같은 EntityManager / 트랜잭션 안에서** 조회 후 변경

### 실행 순서
1. MySQL에 `jpabegin` DB와 `jpauser` 계정, `user` 테이블 준비
2. `UserSaveMain` → INSERT
3. `UserGetMain` → 저장된 값 확인
4. `UserUpdateMain` → 이름 변경 후 `UserGetMain`으로 다시 확인

## 7. 핵심 객체 요약

| 객체 | 역할 | 생명주기/특징 |
|---|---|---|
| `Persistence` | EMF 생성용 유틸 | persistence-unit 이름으로 설정 로딩 |
| `EntityManagerFactory` | EM 생성 | 앱당 **1개**, 생성 비용 큼, 스레드 안전 (`emf.close()`로 종료) |
| `EntityManager` | 엔티티 저장/조회/삭제 | 작업당 생성 후 `close()`, **스레드 안전하지 않음** |
| `EntityTransaction` | 트랜잭션 제어 | 변경 작업은 반드시 트랜잭션 안에서 |

## 8. 핵심 포인트

- JPA 3.0 → `jakarta.persistence` / Hibernate **6.0**
- 설정 파일은 `META-INF/persistence.xml`, 프로퍼티는 `jakarta.persistence.jdbc.*`
- 커넥션 풀은 `hibernate-hikaricp` + `hibernate.hikari.*` 설정
- 엔티티 = `@Entity` + `@Id` + **기본 생성자**
- 수정은 **변경 감지**로 자동 처리 → 개발자는 SQL 대신 객체를 다룬다
- 예외 시 `rollback()`, 마지막엔 반드시 `entityManager.close()` / `emf.close()`
