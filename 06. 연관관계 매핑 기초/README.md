# 06. 다양한 연관관계 매핑

# 요약
연관관계의 주인
- FK를 갖고 있는 곳!
- mappedBy가 없는 곳!

상황별 Annotation 정리

1. @ManyToOne
	- 주인
		- @ManyToOne
		- @JoinColumn(name = "COLUMN_NAME")
	-  !주인
		- @OneToMany(mappedBy = "ownerField")
2. ~~@OneToMany (NOT RECOMMENDED)~~
	-  !주인
		- @OneToMany
		- @JoinColumn(name = "TEAM_ID")
	- 주인  
		- @ManyToOne
		- @JoinColumn(
		  name = "COLUMN_NAME", insertable = false, updatable = false)
3. @OneToOne
	- 주인
		- @OneToOne
		- @JoinColumn(name = "COLUMN_NAME")
	-  !주인
		- @OneToOne(mappedBy = "ownerField")
4. ~~@ManyToMany (NOT RECOMMENDED)~~
	- 주인
		- @ManyToMany
		-   @JoinColumn(name = "MID_TABLE_NAME",
				joinColumns = "MY_COLUMN_NAME",
				inverseJoinColumns = @JoinColumn(name = "YOUR_COLUMN_NAME")
			)
	-  !주인
		- @ManyToMany(mappedBy = "ownerField")
5. → @ManyToMany의 대안 : @ManyToOne 두개 사용!


# 6.1 다대일
## 다대일 단방향

```java 
@Entity
class Member{
	...
	@ManyToOne
	@JoinColumn(name = "TEAM_ID")
	private Team team;
	...
}
```

```java 
class Team{
	... // member와 관련된 필드가 없다.
}
```

- @ManyToOne : 다대일 관계에서 '다' 에 붙임
- @JoinColumn : 상대방 column 지정한다. 생략 가능
	- name : 테이블에서 FK column 이름 / 생략시 `{상대방 table명}_{상대방 PK 컬럼명}`


## 다대일 양방향
```java 
@Entity
class Member{
	...
	@ManyToOne
	@JoinColumn(name = "TEAM_ID")
	private Team team;
	...
	// 연관관계 편의 메소드  
	public void setTeam(Team team) {  
	    // 기존 team 리스트에서 제거  
	    if (this.team != null) {  
	        this.team.getMembers().remove(this);  
	    }  
	    this.team = team;  
	    // 새로운 team에 추가  
	    if (this.team != null && !team.getMembers().contains(team)) {  
	        team.getMembers().add(this);  
	    }  
	}
	...
}
```

```java 
class Team{
	...
	@OneToMany(mappedBy = "team") // mappedBy = 반대편 @ManyToOne의 변수명
	private List<Member> members = new ArrayList<Member> ();
	
	public void addMember(Member member) {  
	    this.members.add(member);  
	  
	    if (member.getTeam() != this) {  // 무한루프 체크
	        member.setTeam(this);  
	    }  
	}
	...
}
```

- 연관관계의 주인 설정 == **항상 FK가 있는** 곳!
- @OneToMany : 다대일 에서 '일'에 붙이는 Annotation
	- mappedBy : 반대편 @ManyToOne의 변수명
- 연관관계 편의 메소드 : 객체가 테이블과 같이 항상 서로를 참조하게 하기 위한 메소드
	- 이전 연결 객체로부터 데이터 삭제 필요! (`this.team.getMembers().remove(this)`)
	- 무한루프 체크 필요

# 6.2 일대다
> [!note] 일대다 보다는 다대일이 좋다!
## 일대다 단방향
```java 
class Team{
	...
	@OneToMany
	@JoinColumn(name = "TEAM_ID")
	private List<Member> members = new ArrayList<Member> ();
	...
}
```

```java 
@Entity
class Member{
	...
}
```

- "일" 에서 JoinColumn을 사용할 경우, FK는 "다" 테이블에 저장된다.
- @JoinColumn : 다대일 에서와 달리생략 불가능하다
	- name : Member 테이블에서 FK column 이름

## 일대다 양방향
```java 
class Team{
	...
	@OneToMany
	@JoinColumn(name = "TEAM_ID")
	private List<Member> members = new ArrayList<Member> ();
	...
}
```

```java 
@Entity
class Member{
	...
	@ManyToOne
	@JoinColumn(name = "TEAM_ID", insertable = false, updatable = false)
	private Team team;
	...
}
```
- "다" 엔티티에도 @JoinColumn을 추가한다.
- 이때 Team과 Member 모두에서 FK를 관리하므로  문제 발생
	- -> `insertable = false, updatable = false` 설정 필요!

# 6.3 일대일
## 주 테이블에 외래 키, 단방향

```java 
@Entity
class Member{
	...
	@OneToOne
	@JoinColumn(name = "LOCKER_ID")
	private Locker locker;
	...
}
```

```java 
class Locker{
	...
}
```

## 주 테이블에 외래 키, 양방향
```java 
@Entity
class Member{
	...
	@OneToOne
	@JoinColumn(name = "LOCKER_ID")
	private Locker locker;
	...
}
```

```java 
class Locker{
	...
	@OneToOne(mappedBy = "locker")
	private Member member;
	...
}
```

## 외래 테이블에 외래 키, 단방향
- -> 사용 불가능

## 외래 테이블에 외래 키, 양방향
```java 
@Entity
class Member{
	...
	@OneToOne(mappedBy = "member")
	private Locker locker;
	...
}
```

```java 
class Locker{
	...
	@OneToOne
	@JoinColumn(name = "MEMBER_ID")
	private Member member;
	...
}
```

# 6.4 다대다
## 단방향
```java 
@Entity
class Member{
	...
	@ManyToMany
	@JoinColumn(name = "MEMBER_PRODUCT",
		joinColumns = "MEMBER_ID",
		inverseJoinColumns = @JoinColumn(name = "PRODUCT_ID")
	)
	private List<Product> products = new ArrayList<>();
	...
}
```

```java 
@Entity
class Product{
	...
}
```
- @ManyToMany
- @JoinColumn
	- name : 연결 테이블 이름 지정
	- joinColumns : 현재 방향 엔티티(회원)의 컬럼명
	- inverseJoinColumns : 반대 방향 엔티티(상품)의 컬럼명

## 양방향
```java 
@Entity
class Member{
	...
	@ManyToMany
	@JoinColumn(name = "MEMBER_PRODUCT",
		joinColumns = "MEMBER_ID",
		inverseJoinColumns = @JoinColumn(name = "PRODUCT_ID")
	)
	private List<Product> products = new ArrayList<>();

	// 연관관계 편의 메소드
	public void addProduct(Product product) {
		...
		products.add(product);
		product.getMembers().add(this);
	}
	...
}
```

```java 
@Entity
class Product{
	...	
	@ManyToMany(mappedBy = "products")
	private List<Member> products = new ArrayList<>();
	...
}
```
- mappedBy 로 연관관계의 주인 설정
- 연관관계의 주인 = Member

## 6.4 (2) 다대다 -> 일대다로 표현하기

### 첫 번째 방법 : 복합 키 
```java 
@Entity
class Member{
	...
	@OneToMany(mappedBy = "member")
	private List<MemberProduct> memberProducts;
	...
}
```

```java 
@Entity
class Product{
	...
}
```

```java 
@Entity 
@IdClass(MemberProductId.class)
class MemberProduct{
	@Id
	@ManyToOne
	@JoinColumn(name = "MEMBER_ID")
	private Member member;
	
	@Id
	@ManyToOne
	@JoinColumn(name = "PRODUCT_ID")
	private Product product;

	// 추가 필드
	private int orderAmount;
	...
}
```

```java 
class MemberProductId implements Serializable{
	private String member;
	private String product;

	@Override
	public boolean equals(Object o) {...}
	@Override
	public int hachCode(Object o) {...}
}
```
- `@IdClass(MemberProductId.class)` 
	- 복합 기본 키 를 만들기 위한 클래스
	- 별도의 식별자 클래스로 분리, Serializable 구현
	- 기본 생성자, equals, hashCode 필요

### 두 번째 방법 : 새로운 키 사용
```java 
@Entity 
class Order{
	@Id @GeneratedValue
	@Column(name = "ORDER_ID")
	private Long Id;
	
	@Id
	@ManyToOne
	@JoinColumn(name = "MEMBER_ID")
	private Member member;
	
	@Id
	@ManyToOne
	@JoinColumn(name = "PRODUCT_ID")
	private Product product;

	// 추가 필드
	private int orderAmount;
	...
}
```

### 비교!
**방법 1 식별 관계 : 받아온 식별자를 기본키 + 외래 키로 사용한다.**

- 데이터를 표현하는 더 나은 방식
	- 새로운 식별자(Auto increment)는 해당 column 데이터와 전혀 무관한 데이터이기 때문에 불필요한 데이터이다.
	- 중복 데이터 발생 가능성..!!
- 데이터의 조회/수정/삭제 성능 개선


**방법 2 비식별 관계 : 받아온 식별자는 외래 키로만 사용하고 새로운 식별자 추가**
- 생성이 간편
- 영구적으로 사용 가능하다
- 비즈니스에 의존하지 않는다
- JPA 코드 구현이 간편하다.
