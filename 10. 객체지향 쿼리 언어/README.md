# 10. 객체지향 쿼리 언어

# 10.1 객체지향 쿼리 소개


### 목적

- 생성/수정에서도 엔티티 객체를 중심으로 수행했던 것과 같이
- 검색에서도 테이블이 아닌 엔티티 객체를 대상으로 검색해야 한다.
- → JPQL은 엔티티 객체를 대상으로 쿼리를 수행 (SQL은 테이블 대상 쿼리)

### 종류

- JPQL : 표준 문법
- JPA Criteria, Query DSL : JPQL 크리에이터
- 네이티브 SQL : DB 종속적인 query를 보내야 할 때
- JDBC API or MyBatis, JdbcTemplate 등등 같이 사용 가능!

### JPQL

예시
```java
List<Member> resultList = em.createQuery(  
        "select m from Member m where m.name like '%kim%'",  
        Member.class  
).getResultList();
```
- Member : Member 테이블이 아닌 Member 엔티티를 나타냄
- m : column이 아닌 객체 자체를 가져옴


### Criteria (사용 x)
예시
```java
// Criteria  
CriteriaBuilder cb = em.getCriteriaBuilder();  
CriteriaQuery<Member> query = cb.createQuery(Member.class);   
  
// 루트 클래스 (조회를 시작할 지점 설정)  
Root<Member> m = query.from(Member.class);  
// 쿼리 생성
CriteriaQuery<Member> cq = query.select(m).where(cb.equal(m.get("name"), "kim"));  
List<Member> criteriaResult = em.createQuery(cq).getResultList();
```

장점
- JPQL은 string → 동적 쿼리 구성이 어렵다!
- 동적 쿼리를 쉽게 작성 가능 (자바 코드를 통해 JPQL 빌드)
- 오타 → 컴파일 에러 → 수정에 용이하다.

단점
- 매우 복잡해진다 → 유지보수 어려움
- SQL스럽지 않음

### QueryDSL (사용 권장)

오픈소스 [링크](http://querydsl.com/)

예시
```java
public void hello() {  
    QMember m = QMember.member;  
    List<Member> result = queryFactory  
            .select(m)  
            .from(m)  
            .where(m.name.like("kim"))  
            .fetch();  
}
```
- 오타 → 컴파일 에러 → 수정에 용이하다.
- 자바코드 이므로 재사용성 용이
- **실무 사용 권장!!**

### 네이티브 SQL
`em.createNativeQuery()` 이용!
```java
em.createNativeQuery("select MEMBER_ID, city, street, zipcode, USERNAME from MEMBER")  
	.getResultList();
```

### 결론
> [!note] 
> JPQL 을 잘 쓰자!
> +QueryDSL을 사용하자


# 10.2 JPQL

> [!abstract]
> - JPQL은 객체지향 쿼리 언어이다. 따라서 테이블을 대상으로 하는 것이 아니라 엔티티 객체를 대상으로 쿼리한다.
> - JPQL은 SQL을 추상화해서 특정 데이터베이스 SQL을 추상화하지 않는다.
> - JPQL은 결국 SQL로 변환된다.

### 문법
```java
select_문 ::=
	select_절
	from_절
	where_절
	groupby_절
	having_절
	orderby_절

update_문 ::= update_절 [where_절]
delete_문 ::= delete_절 [where_절]
```
- 엔티티와 속성은 대소문자 구분
- JPQL 키워드는 대소문자 구분 x
- 엔티티 이름, !테이블이름
- 별칭은 필수, as는 선택

### 집합, 통계
```sql
SELECT
	COUNT(m),
	SUM(m.age),
	AVG(m.age),
	MAX(m.age),
	MIN(m.age)
FROM ...
```

### TypeQuery, Query
```java
TypeQuery query = em.createQuery({JPQL}, {ClassName}.class)
Query query = em.createQuery({JPQL})
```
- TypeQuery 쓰기 어려운 경우?
	- select절로 타입이 명확하지 않은 것을 요청할때
	- `SELECT m, COUNT(m) FROM ...`

### 결과 조회
- `query.getResultList()`
	- 결과 하나 이상 : 리스트 반환
	- 결과 없음 : 빈 리스트 반환
- `query.getSingleList()`
	- 결과 하나 : 하나의 객체 반환
	- 결과 둘이상 :  javax.persistence.NonUniqueResultException
	- 결과 없음 : javax.persistence.NoResultException

### 파라미터 바인딩
- `:파라미터명`
- `query.setParameter("파라미터명", parameterValue);`
```java
Member member = em.createQuery("select m from Member m where m.username = :username", Member.class );  
	.setParameter("username", "이름1")
	.getSingleResult();
```

### 프로젝션
- SELECT 절에 조회할 대상을 지정하는 것
- 대상 : 엔티티, 임베디드 타입, 스칼라 타입(숫자,문자,데이터)
- 엔티티
	- `SELECT m FROM Member m`
	- `SELECT m.team FROM Member m`
		- → JPQL과 SQL의 쿼리문이 크게 달라지므로 비추!
- 임베디드 타입
	- `SELECT m.address FROM Member m`
		- embedded는 다른 엔티티에 소속되어 있으므로 직접 불러올 수 없다.
		- ~~`SELECT a FROM Address a` ~~
- 스칼라 타입(숫자,문자,데이터)
	- `SELECT m.username, m.age FROM Member m`

### 여러 값 조회
