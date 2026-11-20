# Chapter 9. 값 콜렉션 매핑 – List (@OrderColumn)

## 1. 개념

- Chapter 8의 `Set`과 마찬가지로 **값 콜렉션**이지만, `List`는 **순서**가 있다는 점이 다르다.
- 순서를 DB에 보존하려면 **순서를 저장할 컬럼(`@OrderColumn`)** 이 필요하다.
- 요소가 단순 값(`String`)이거나 `@Embeddable` 값 객체일 수 있다.

```
Question (엔티티)  ──  question 테이블
 └─ List<String> choices  ──  question_choice 테이블 (question_id, idx, text)
```

### 테이블 (`ddl.sql`)
```sql
create table jpabegin.question (
  id varchar(20) not null primary key,
  text varchar(200)
) engine innodb character set utf8mb4;

create table jpabegin.question_choice (
  question_id varchar(20),
  idx int,                       -- 리스트 순서(인덱스)를 저장하는 컬럼
  text varchar(50),
  input boolean,                 -- Question2(임베디드)에서만 사용
  primary key (question_id, text)
) engine innodb character set utf8mb4;
```
- `idx`가 `@OrderColumn`이 매핑되는 컬럼 (0부터 시작)
- 참고: 예제 DDL의 PK는 `(question_id, text)`라서 `idx`가 PK에 포함되지 않는다. 같은 질문 안에서 같은 `text`가 중복될 수 없다.

## 2. 단순 값 List 매핑 – `Question`

```java
@Entity
@Table(name = "question")
public class Question {
    @Id
    private String id;
    private String text;

    @ElementCollection(fetch = FetchType.EAGER)
    @CollectionTable(
            name = "question_choice",
            joinColumns = @JoinColumn(name = "question_id")
    )
    @OrderColumn(name = "idx")
    @Column(name = "text")
    private List<String> choices = new ArrayList<>();

    protected Question() {}
    ...
    public void setChoices(List<String> choices) { this.choices = choices; }
}
```

| 어노테이션 | 역할 |
|---|---|
| `@ElementCollection` | 값 콜렉션 선언 (예제는 `fetch = EAGER`) |
| `@CollectionTable` | 콜렉션 테이블(`question_choice`)과 FK(`question_id`) |
| **`@OrderColumn(name = "idx")`** | 리스트의 **인덱스를 저장할 컬럼** 지정 |
| `@Column(name = "text")` | 요소(String) 값 컬럼 |

### @OrderColumn이 필요한 이유
- `@OrderColumn`이 없으면 `List`는 순서를 저장하지 않는 **bag**(순서 없는 컬렉션)처럼 다뤄진다. 조회할 때마다 순서가 달라질 수 있다.
- `@OrderColumn`을 쓰면 JPA가 **요소의 위치(0, 1, 2 …)를 `idx` 컬럼에 저장**하고, 조회 시 `idx` 순으로 읽어 리스트를 복원한다.
- 요소를 중간에 삽입/삭제하면 뒤쪽 요소의 `idx`도 함께 갱신해야 하므로, **리스트 앞/중간을 자주 바꾸는 용도에는 비용이 크다.**

## 3. 동작 – `MainQuestion1`

### 저장
```java
Question q = new Question(id, "질문", List.of("보기1", "보기2"));
em.persist(q);
tx.commit();
```
- `question`에 1건, `question_choice`에 요소 수만큼 INSERT (`idx`는 0, 1 로 저장)

### 조회
```java
Question q = em.find(Question.class, id);
logger.info("보기 개수: {}", q.getChoices().size());
```
- `fetch = EAGER`이므로 엔티티를 조회할 때 **콜렉션도 함께 조회**된다. (기본값 LAZY였다면 콜렉션에 접근할 때 조회)

### 수정 – 리스트 교체
```java
Question q = em.find(Question.class, id);
q.setChoices(List.of("답1", "답2"));
tx.commit();
```
- 리스트를 통째로 바꾸면 기존 행을 지우고 새 요소를 INSERT하는 방식으로 반영된다. (정확한 쿼리는 `org.hibernate.SQL` 로그로 확인)

### 삭제 – 엔티티 삭제 시 콜렉션 데이터
```java
Question q = em.find(Question.class, id);
em.remove(q);
tx.commit();
```
- 값 콜렉션은 소유 엔티티에 종속되므로, 엔티티를 삭제하면 **콜렉션 테이블(`question_choice`)의 행이 먼저 삭제된 뒤 엔티티(`question`)가 삭제**된다.
- 콜렉션 요소를 따로 지우는 코드를 작성할 필요가 없다.

## 4. @Embeddable 타입 List 매핑 – `Question2`

보기에 텍스트 외에 **입력 여부(`input`)** 같은 추가 속성이 필요한 경우.

### 값 타입 – `Choice`
```java
@Embeddable
@Access(AccessType.FIELD)
public class Choice {
    private String text;
    private boolean input;

    protected Choice() {}

    public Choice(String text, boolean input) { ... }

    public String getText() { return text; }
    public boolean isInput() { return input; }
}
```
- `@Access(AccessType.FIELD)` : **필드 접근**을 명시. `boolean input`의 getter가 `isInput()`처럼 이름 규칙이 달라도 필드 기준으로 매핑되게 한다.

### 엔티티 – `Question2`
```java
@Entity
@Table(name = "question")
public class Question2 {
    @Id
    private String id;
    private String text;

    @ElementCollection(fetch = FetchType.EAGER)
    @CollectionTable(
            name = "question_choice",
            joinColumns = @JoinColumn(name = "question_id")
    )
    @OrderColumn(name = "idx")
    private List<Choice> choices = new ArrayList<>();
    ...
}
```
- 요소가 `@Embeddable`이므로 **컬렉션 필드에는 `@Column`을 붙이지 않는다.** 컬럼은 `Choice`의 필드명(`text`, `input`)을 그대로 사용한다.
- `Question`(String 리스트)과 `Question2`(Choice 리스트)는 **같은 테이블**(`question`, `question_choice`)에 매핑되는 비교용 예제이고, 둘 다 persistence.xml에 등록되어 있다.

### 사용 – `MainQuestion2`
```java
new Question2(id, "질문", List.of(new Choice("보기1", false), new Choice("보기2", false)));
...
q.setChoices(List.of(new Choice("답1", false), new Choice("답2", false)));
```
- 저장 → 조회 → 리스트 교체 → 삭제 흐름은 `MainQuestion1`과 동일

## 5. Set과 List 비교

| 구분 | `Set` (jpa-08) | `List` (jpa-09) |
|---|---|---|
| 순서 | 없음 | 있음 (`@OrderColumn`으로 저장) |
| 중복 | 불가 | 가능 (단, 예제 PK는 중복을 막음) |
| 추가 컬럼 | 없음 | `idx`(순서 컬럼) |
| 값 객체 요소 | `equals/hashCode` **필수** | 순서(`idx`)로 구분하지만, 동등성 비교를 위해 구현해 두는 것이 안전 |
| 중간 삽입/삭제 | 해당 행만 | 뒤쪽 요소의 `idx`도 갱신 |

## 6. 정리

- `List` 값 콜렉션: `@ElementCollection` + `@CollectionTable` + **`@OrderColumn`**
- `@OrderColumn`은 리스트의 **인덱스를 컬럼에 저장**해 순서를 복원한다.
- 엔티티를 삭제하면 콜렉션 테이블의 데이터도 함께 삭제된다.
- 값 객체 요소는 `@Embeddable`로 만들고, 필드 접근은 `@Access(AccessType.FIELD)`로 명시한다.
