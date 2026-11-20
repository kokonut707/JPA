# Chapter 10. 값 콜렉션 매핑 – Map (@MapKeyColumn)

## 1. 개념

- `Map<키, 값>` 형태의 값 콜렉션을 **별도 테이블**에 저장한다.
- 콜렉션 테이블의 한 행이 Map의 **한 엔트리**이고, 키가 저장되는 컬럼을 `@MapKeyColumn`으로 지정한다.
- 값이 단순 값(`String`)이거나 `@Embeddable` 값 객체일 수 있다.

```
Document (엔티티)  ──  doc 테이블
 └─ Map<String, String> props  ──  doc_prop 테이블 (doc_id, name[키], value[값])
```

### 테이블 (`ddl.sql`)
```sql
create table jpabegin.doc (
  id varchar(20) not null primary key,
  title varchar(200),
  content varchar(200)
) engine innodb character set utf8mb4;

create table jpabegin.doc_prop (
  doc_id varchar(20),
  name varchar(50),              -- Map의 키
  value varchar(100),            -- Map의 값
  enabled boolean,               -- Document2(임베디드)에서만 사용
  primary key (doc_id, name)
) engine innodb character set utf8mb4;
```
- PK가 `(doc_id, name)` → 한 문서 안에서 **키(name)는 유일**, Map의 성격과 일치
- 주의: 레포의 `ddl.sql`에는 `doc` 테이블의 마지막 컬럼(`content varchar(200),`) 뒤에 **쉼표가 남아 있어** 그대로 실행하면 문법 오류가 난다. 위 노트에서는 쉼표를 제거했다.

## 2. 단순 값 Map 매핑 – `Document`

```java
@Entity
@Table(name = "doc")
public class Document {
    @Id
    private String id;
    private String title;
    private String content;

    @ElementCollection
    @CollectionTable(
            name = "doc_prop",
            joinColumns = @JoinColumn(name = "doc_id")
    )
    @MapKeyColumn(name = "name")
    @Column(name = "value")
    private Map<String, String> props = new HashMap<>();

    protected Document() {}
    ...
    public void setProp(String name, String value) { props.put(name, value); }
    public void removeProp(String name) { props.remove(name); }
}
```

| 어노테이션 | 역할 |
|---|---|
| `@ElementCollection` | 값 콜렉션 선언 (기본 LAZY) |
| `@CollectionTable(name, joinColumns)` | 콜렉션 테이블(`doc_prop`)과 FK(`doc_id`) |
| **`@MapKeyColumn(name = "name")`** | **Map의 키**가 저장될 컬럼 |
| `@Column(name = "value")` | **Map의 값**이 저장될 컬럼 |

- 필드는 `new HashMap<>()`으로 초기화
- 엔티티에 `setProp()`, `removeProp()` 같은 **의도가 드러나는 메서드**를 두고, Map 자체를 외부에 노출하지 않는 구조

## 3. 동작 – `MainDocument1`

### 저장
```java
Map<String, String> props = new HashMap<>();
props.put("p1", "v1");
props.put("p2", "v2");
Document doc = new Document(id, "제목", "내용", props);
em.persist(doc);
tx.commit();
```
- `doc` 1건 + `doc_prop` 엔트리 수만큼(p1, p2) INSERT

### 수정
```java
Document doc = em.find(Document.class, id);
doc.setProp("p1", "v1new");   // 기존 키의 값 변경
doc.setProp("p10", "v10");    // 새 키 추가
doc.removeProp("p2");         // 키 삭제
tx.commit();
```

| 변경 | 반영되는 쿼리 |
|---|---|
| 기존 키의 값 변경 (`p1`) | `UPDATE` (키로 행을 식별할 수 있으므로) |
| 새 키 추가 (`p10`) | `INSERT` |
| 키 삭제 (`p2`) | `DELETE` |

- Set/List와 달리 Map은 **키가 행의 식별자 역할**을 하므로 값만 바뀌면 삭제·삽입이 아니라 UPDATE가 가능하다. (실제 쿼리는 `org.hibernate.SQL` 로그로 확인)
- 컬렉션에 접근할 때(LAZY) `doc_prop`을 조회한 뒤 변경 내용을 비교해 반영한다.

## 4. @Embeddable 타입을 Map 값으로 사용 – `Document2`

값에 문자열 외에 **활성화 여부(`enabled`)** 같은 속성이 더 필요한 경우.

### 값 타입 – `PropValue`
```java
@Embeddable
@Access(AccessType.FIELD)
public class PropValue {
    private String value;
    private boolean enabled;

    protected PropValue() {}

    public PropValue(String value, boolean enabled) { ... }

    public String getValue() { return value; }
    public boolean isEnabled() { return enabled; }
}
```

### 엔티티 – `Document2`
```java
@Entity
@Table(name = "doc")
public class Document2 {
    @Id
    private String id;
    private String title;
    private String content;

    @ElementCollection
    @CollectionTable(
            name = "doc_prop",
            joinColumns = @JoinColumn(name = "doc_id")
    )
    @MapKeyColumn(name = "name")
    private Map<String, PropValue> props = new HashMap<>();

    public void setProp(String name, PropValue value) { props.put(name, value); }
    public void removeProp(String name) { props.remove(name); }
}
```
- 값이 `@Embeddable`이므로 **`@Column`을 컬렉션 필드에 붙이지 않는다.** 컬럼은 `PropValue`의 필드명(`value`, `enabled`)을 사용한다.
- 키는 그대로 `@MapKeyColumn(name = "name")`
- `Document`(String 값)와 `Document2`(PropValue 값)는 **같은 테이블**에 매핑되는 비교용 예제

### 사용 – `MainDocument2`
```java
props.put("p1", new PropValue("v1", true));
props.put("p2", new PropValue("v2", true));
...
doc.setProp("p1", new PropValue("v1new", true));    // 값 객체를 새로 만들어 교체
doc.setProp("p10", new PropValue("v10", false));
doc.removeProp("p2");
```
- `PropValue`는 setter가 없는 **불변 객체** → 값을 바꿀 때는 **새 객체를 `put`** 으로 교체한다.

## 5. Set / List / Map 비교

| 구분 | Set | List | Map |
|---|---|---|---|
| 식별 방식 | 요소 값 (`equals/hashCode`) | 순서 `idx` (`@OrderColumn`) | **키** (`@MapKeyColumn`) |
| 추가 컬럼 | 없음 | `idx` | 키 컬럼 (`name`) |
| 값 변경 시 | 삭제 + 삽입 | 위치 기준 갱신 | 키 기준 **UPDATE** |
| 테이블 PK 예 | `(role_id, perm)` | `(question_id, text)` | `(doc_id, name)` |

## 6. 정리

- Map 값 콜렉션: `@ElementCollection` + `@CollectionTable` + **`@MapKeyColumn`**
- 값 컬럼은 단순 값이면 `@Column`, 값 객체면 `@Embeddable`
- 키는 한 엔티티 안에서 유일해야 하며 (PK `(doc_id, name)`), 키를 기준으로 INSERT/UPDATE/DELETE가 결정된다.
- 값 객체는 **불변**으로 두고, 변경은 새 객체로 교체한다.
