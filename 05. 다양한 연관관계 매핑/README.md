# 05. 다양한 연관관계 매핑

### **들어가기**

- 방향
    - 단방향, 양방향
    - 한쪽에서만 참조하는 것을 단방향, 서로 참조하는 것을 양방향
    - **방향은 객체관계에서만 존재하고 테이블 관계에서는 항상 양방향**
- 다중성
    - 다대일, 일대다, 일대일, 다대다
- 연관관계 주인
    - 객체를 양방향 연관관계로 만들면 연관관계의 주인을 정해야한다.
    - 다쪽이 연관관계의 주인!
- 객체와 관계형 데이터베이스의 패러다임 차이
    - 객체는 참조를 사용하여 관계를 맺고
    - 테이블은 외래키를 사용하여 관계를 맺는다.

<br/>

### 단방향 연관관계

- 객체 연관관계 vs 테이블 연관관계
    - 객체
        - 참조를 통한 연관관계는 언제나 단방향이다.
        - 참조로 연관관계를 맺는다.
        - 객체 그래프 탐색
            - 객체는 참조를 사용하여 연관관계를 탐색할 수 있다.
    - 테이블
        - 외래키 하나로 양방향 조인이 가능하다.
        - 외래 키로 연관관계를 맺는다.

- 객체 관계 매핑
    
    ```java
    @ManyToOne
    @JoinColumn(name="TEAM_ID")
    private Team team;
    ```
    
    - @ManyToOne
        - 다대일 관계
        - Fetch 기본값 : FetchType.EAGER
    - @JoinColumn(name="TEAM_ID")
        - 외래 키를 매핑할 때 사용
        - name 속성 ⇒ 매핑할 외래 키 이름을 지정
            - 생략시 기본 전략 사용
                - 필드명 + _ + 참조하는 테이블의 칼럼명
                
                ```java
                @ManyToOne
                private Team team;
                
                필드명(team) + _(밑줄) + 참조하는 테이블의 칼럼명(TEAM_ID) => team_TEAM_ID
                ```
                
<br/>

### 연관관계 사용

- 저장
    - JPA에서 엔티티를 저장할 때 연관된 모든 엔티티는 영속 상태이어야 한다.
    - JPA는 참조한 엔티티의 식별자를 외래키로 사용하여 적절한 등록 쿼리를 생성한다.
- 조회
    - 객체 그래프 탐색
        
        ```java
        Member member = em.find(Member.class, "member1");
        Team team = member.getTeam(); // 객체 그래프 탐색
        ```
        
        → 객체를 통해 연관된 엔티티를 조회하는 것!
        
    - 객체지향 쿼리 사용
        
        ```java
        String jpql = "select m from Member m join m.team where "
        	+ "t.name=: teamName";
        
        List<Member> resultList = em.createQuery(jpql, Member.class)
        	.setParameter("teamName", "팀1");
        	.getResultList();
        ```
        
        → JPQL은 객체(엔티티)를 대상으로 한다.
        
- 수정
    - em.update() 메소드는 존재하지 않는다.
    - 단순히 영속성 컨텍스트내에 존재하는 엔티티의 값을 변경해두면 트랜잭션을 커밋할 때 플러시가 일어나면서 변경 감지 기능이 동작
    - 자동으로 데이터베이스에 반영된다.
- 연관관계 제거
    
    ```java
    Member member = em.find(Member.class, "member1");
    member.setTeam(null); // 연관관계 제거
    ```
    
- 연관된 엔티티 삭제
    - 연관된 엔티티를 삭제하려면 기존에 있던 연관관계를 먼저 제거하고 삭제해야한다.
        - 그렇지 않으면, 외래키 제약조건으로 인해, 데이터베이스 오류 발생
    
    ```java
    member1.setTeam(null);
    member2.setTeam(null);
    em.remove(team); // 팀 삭제
    
    ```
 
<br/>

### 양방향 연관관계

```java
@Entity
public class Member {
	@Id
	private String id;

	private String username;
	
	@ManyToOne
	@JoinColumn(name = "TEAM_ID")
	private Team team;

	public void setTeam(Team team) {
		this.team = team;
	}
	...
}

@Entity
public class Team {
	@Id
	private String id;

	private String name;

	@OneToMany(mappedBy = "team")
	private List<Member> members = new ArrayList<Member>();
	
	...
}
```

- Team 엔티티에 List<Member> members를 추가.
- mappedBy 속성은 양방향 매핑일 때 사용하는데 반대쪽 매핑의 필드 이름을 값으로 주면 된다.

<br/>

### 연관 관계의 주인

- 객체에는 양방향 연관관계라는 것이 없다.
- 엔티티를 양방향 연관관계로 설정하면 객체는 참조는 둘인데 외래 키는 하나이다. ⇒ 둘 사이의 차이가 발생한다.
- 연관관계 주인 : 두 객체 연관관계 중 하나를 정해 테이블의 외래키를 관리해야 한다
    - 연관관계 주인만이 데이터베이스 연관관계와 매핑되고 외래 키를 관리(등록, 수정, 삭제)할 수 있다.
    - 주인이 아닌 쪽에서는 읽기만 가능하다.
- 주인은 mappedby 속성을 사용하지 않는다. 주인이 아니면 mappedby 속성을 사용하여 속성의 값으로 연관관계의 주인을 지정해야한다.
    - 연관관계 주인은 외래 키가 있는 곳으로 설정하면 된다.
- 데이터베이스 테이블의 다대일, 일대다 관계에서는 항상 다 쪽이 외래키를 갖는다. 따라서 @ManyToOne은 항상 연관관계의 주인이 되므로, mappedby를 설정할 수 있고, mappedby 속성이 존재하지 않는다.

<br/>

### 양방향 연관관계 저장

```java
team.getMember().add(member1);
team.getMember().add(member2);
// team은 연관관계 주인이 아니기 때문에, 무시된다.

member1.setTeam(team);
member2.setTeam(team);
// member는 연관관계 주인이기 떄문에 연관관계가 저장된다.
```

⇒ 연관관계의 주인만이 외래 키의 값을 변경할 수 있다.

- 순수한 객체까지 고려한 양방향 연관관계
    - 객체 관점에서 양쪽 방향에 모두 값을 입력해주는 것이 안전하다.
    - 연관관계 편의 메서드를 사용하여, 양방향 관계에서 아래의 두 코드를 하나인 것처럼 사용하는 것이 안전하다.
        
        ```java
        member.setTeam(team);
        team.getMembers().add(member);
        
        ->
        
        public class Member {
        	private Team team;
        
        	public void setTeam(Team team) {
        		this.team = team;
        		team.getMembers().add(this);
        	}
        
        	...
        }
        ```
        
    - 양방향 연관관계를 설정하는 것은 두 객체의 데이터 정합성을 맞추기 위해서는 많은 고민과 수고가 필요하다.
        - 관계형 데이터베이스는 외래키 하나로 문제를 단순하게 해결할 수 있지만,
        - 객체에서 양방향 연관관계를 사용하려면 로직을 견고하게 작성해야 한다.

<br/>

### 결론

- 단방향 매핑과 비교해서 양방향 매핑은 복잡하다.
    - 연관관계의 주인도 정해야 하고,
    - 두 개의 단방향 연관관계를 양방향으로 만들기 위해 로직도 잘 관리해야 한다.
    - 무한 루프에 빠지지 않게 조심해야한다.
- 양방향 연관관계의 장점은 반대 방향으로 객체 그래프 탐색이 가능하다는 것뿐이다.
    - **비즈니스 로직의 필요에 따라 다르겠지만 우선 단방향 매핑을 사용하고,**
    - **반대 방향으로 객체 그래프 탐색 기능이 필요할 때 양방향을 사용하도록 하자.**
- 연관관계 주인을 정하는 기준
    - 비즈니스 로직상 더 중요하다고 연관관계의 주인으로 선택하면 안된다.
    - 외래 키의 위치와 관련해서 정해야한다.
