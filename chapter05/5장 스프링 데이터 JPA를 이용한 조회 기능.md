# 도메인 주도 개발 시작하기

## Chapter 5. 스프링 데이터 JPA를 이용한 조회 기능

> 스프링 데이터 JPA를 이용한 조회 기능에 대한 이해

## 개요

- 5.1 시작에 앞서
- 5.2 검색을 위한 스펙
- 5.3 스프링 데이터 JPA 를 이용한 스펙 구현
- 5.4 리포지터리/DAO에서 스펙 사용하기
- 5.5 스펙 조합
- 5.6 정렬 지정하기
- 5.7 페이징 처리하기
- 5.8 스펙 조합을 위한 스펙 빌더 클래스
- 5.9 동적 인스턴스 생성
- 5.10 하이버네이트 @Subselect 사용

<br><br>

## 5.1 시작에 앞서

> CQRS는 명령(command) 모델과 조회(Query) 모델을 분리하는 패턴이다.
>
> - 명령 모델은 상태를 변경하는 기능을 구현할 때 사용
>
> - 조회 모델은 데이터를 조회하는 기능을 구현할 때 사용

이 장에서는  조회 모델을 구현하는 방법에 대해 살펴본다.

<br><br>

## 5.2 검색을 위한 스펙

다양한 검색 조건을 조합해야할 때가 있는데, 매번 함수를 구현하는건 좋은 방법이 아니다.

Specification 은 검색 조건을 다양하게 조합해야 할 때 사용할 수 있는 인터페이스로 애그리거트가 특정 조건을 충족하는지 검사할 때 사용

```java
public interface Specification<T> {
	public boolean isSatisfiedBy(T agg);
}
```

agg 파라미터는 검사 대상이 되는 객체

- 레포지토리 : 애그리거트 루트
- DAO : 검색 결과로 리턴할 데이터 객체


```java
public List<Order> findAll(Specification<Order> spec) {
	List<Order> allOrders = findAll();
	return allOrders.stream()
			.filter(order -> sepc.isSatisfiedBy(order))
			.toList()
}
```

❌ 위와 같은 방법으로는 사용하지 않음. (성능, 메모리 등)

<br><br>

## 5.3 스프링 데이터 JPA 를 이용한 스펙 구현

Specification(이하 스펙) 인터페이스에서 지네릭 타입 파라미터는 JPA의 Entity를 의미, Predicate 는 JPA 크리테리아 API에서 조건을 표현할 때 사용

- spring data jpa 가 제공하는 Specification 인터페이스

```java
package org.springframework.data.jpa.domain;

public interface Specification<T> extends Serializable {

	long serialVersionUID = 1L;

	// not, where, and, or 매서드 생략

	@Nullable
	Predicate toPredicate(Root<T> root, CriteriaQuery<?> query, CriteriaBuilder criteriaBuilder);
}
```

- 구현 클래스 예시

```java
public class OrdererIdSpec implements Specification<OrderSummary> {
    private String ordererId;

    public OrdererIdSpec(String ordererId) {
        this.ordererId = ordererId;
    }

    @Override
    public Predicate toPredicate(Root<OrderSummary> root, 
																	CriteriaQuery<?> query,
																	CriteriaBuilder cb) {
        return cb.equal(root.get(OrderSummary_.ordererId), ordererId);
    }
}
```

💡 문자열은 오타 가능성이 있고 실행하기 전까지 오타가 있다는 것을 놓칠 수 있기 때문에 JPA 정적 메타 모델을 사용한다.

<br><br>

## 5.4 리포지터리/DAO에서 스펙 사용하기

스펙을 충족하는 엔티티검색할 경우 findAll() 메서드 이용

```java
public interface OrderSummaryDao extends Repository<OrderSummary, String> {
    List<OrderSummary> findAll(Specification<OrderSummary> spec);
}

// 스펙 객체 생성
Specification<OrderSummary> spec = new OrdererIdSpec("user1");

// findAll() 메서드 이용해서 검색
List<OrderSummary> results = orderSummaryDao.findAll(spec);
```

<br><br>

## 5.5 스펙 조합

스프링 데이터 JPA가 제공하는 스펙 인터페이스는 스펙을 조합할 수 있는 메서드를 제공하고 있다.

> - and, or 메서드를 이용하여 스펙을 조합
>
> - not 메서드는 조건을 반대로 적용할 때 사용
>
> - where 메서드는 null 을 전달하면 아무 조건도 생성하지 않는 객체를 리턴하기 때문에 NullPointerException이 발생하는것을 방지

```java
public interface Specification<T> extends Serializable {

	default Specification<T> and(@Nullable Specification<T> other) {
		return SpecificationComposition.composed(this, other, CriteriaBuilder::and);
	}

	default Specification<T> or(@Nullable Specification<T> other) {
		return SpecificationComposition.composed(this, other, CriteriaBuilder::or);
	}

	static <T> Specification<T> not(@Nullable Specification<T> spec) {

		return spec == null //
				? (root, query, builder) -> null //
				: (root, query, builder) -> builder.not(spec.toPredicate(root, query, builder));
	}

	static <T> Specification<T> where(@Nullable Specification<T> spec) {
		return spec == null ? (root, query, builder) -> null : spec;
	}

	@Nullable
	Predicate toPredicate(Root<T> root, CriteriaQuery<?> query, CriteriaBuilder criteriaBuilder);
}

// 사용 예시
Specification<OrderSummary> spec = OrderSummarySpecs.ordererId("user1")
                .and(OrderSummarySpecs.orderDateBetween(from, to));

Specification<OrderSummary> spec = Specification.not(OrderSummarySpecs.ordererId("user1"));

Specification<OrderSummary> spec = Specification.where(createNullableSpec()).and(createOtherSpec());

```

<br><br>

## 5.6 정렬 지정하기

스프링 데이터 JPA 는 두가지 방법으로 정렬을 지정할 수 있다.

> - 메서드 이름에 OrderBy를 사용해서 정렬 기준 지정
> - Sort를 인자로 전달

```java
public interface OrderSummaryDao extends Repository<OrderSummary, String> {
    List<OrderSummary> findByOrdererIdOrderByNumberDesc(String ordererId);
    List<OrderSummary> findByOrdererIdOrderByOrderDateDescNumberAsc(String ordererId);
}
```

💡 방법은 간단하지만 정렬 기준이 많아진다면 메서드 이름이 길어질 수 있다. 또한, 메서드 이름으로 순서가 정해지기 때문에 상황에 따라 순서를 변경할 수 없다.

```java
public interface OrderSummaryDao extends Repository<OrderSummary, String> {
    List<OrderSummary> findByOrdererId(String ordererId, Sort sort);
}

// 사용 예시
Sort sort = Sort.by("number").ascending();
List<OrderSummary> results = orderSummaryDao.findByOrdererId("user1",sort);
```

💡 두 개 이상의 정렬 순서를 지정 시 Sort#and() 매서드 사용.

<br><br>

## 5.7 페이징 처리하기

스프링 데이터 JPA는 페이징 처리를 위해 Pageable 타입을 이용한다. Pageable 타입 파라미터를 사용하면 페이징을 자동으로 처리해 준다.

```java
import org.springframework.data.domain.Pageable;

public interface MemberDataDao extends Repository<MemberData, String> {
    
  List<MemberData> findByNameLike(String name, Pageable pageable);
	Page<MemberData> findAll(Specification<MemberData> spec, Pageable pageable);
	// name 프로퍼티 기준으로 like를 검색한 결과를 name 기준 오름차순정렬 후 처음 3개를 조회
	List<memberData> findFirst3ByNameLikeOrderByName(String name);

}

// 사용 예시
PageRequest pageReq = PageRequest.of(1, 10);
List<MemberData> uesr = memberDataDao.findByNameLike("사용자%", pageReq);
public static PageRequest of(int page, int size, Sort sort) {
	return new PageRequest(page, size, sort);
}
```

💡 Pageable 타입 객체는 PageRequest 클래스를 이용해 생성하고, PageRequest.of()매서드의 첫 번째 인자는 페이지의 번호, 두 번째 인자는 한 페이지의 개수를의미한다. 또한, Sort 를 사용하여 정렬 순서를 지정할 수 있다.

Page객체는 전체 개수, 페이지 개수 등 페이징 처리에 필요한 데이터도 함께 제공한다.

```java
Page<MemberData> page = memberDataDao.findByBlocked(false, pageReq);
List<Todo> content = page.getContent(); // 조회 결과 목록
long totalElements = page.getTotalElements(); // 조건에 해당하는 전체 개수
int totalPages = page.getTotalPages(); // 전체 페이지 번호
int number = page.getNumber(); // 현재 페이지 번호
int numberOfElements = page.getNumberOfElements(); // 조회 결과 개수
int size = page.getSize(); // 페이지 크기
```

<br><br>

## 5.8 스펙 조합을 위한 스펙 빌더 클래스

스펙을 조합해서 사용하는 경우 스펙빌더 클래스를 생성하여 메서드 호출 체인으로 연속된 변수 할당을 줄이고 코드 가독성을 높일 수 있다. 추가로 필요한 메서드를 추가해서 사용하면 된다.

```java
public class SpecBuilder {
    public static <T> Builder<T> builder(Class<T> type) {
        return new Builder<T>();
    }

    public static class Builder<T> {
        private List<Specification<T>> specs = new ArrayList<>();

        public Builder<T> and(Specification<T> spec) {
            specs.add(spec);
            return this;
        }

        public Builder<T> ifHasText(String str,
                                    Function<String, Specification<T>> specSupplier) {
            if (StringUtils.hasText(str)) {
                specs.add(specSupplier.apply(str));
            }
            return this;
        }

        public Builder<T> ifTure(Boolean cond,
                                 Supplier<Specification<T>> specSupplier) {
            if (cond != null && cond.booleanValue()) {
                specs.add(specSupplier.get());
            }
            return this;
        }

        public Specification<T> toSpec() {
            Specification<T> spec = Specification.where(null);
            for (Specification<T> s : specs) {
                spec = spec.and(s);
            }
            return spec;
        }
    }
}

// 사용 예시
Specification<MemberData> spec = SpecBuilder.builder(MemberData.class)
                .ifTure(searchRequest.isOnlyNoyBlocked(),
                        () -> MemberDataSpecs.nonBlocked())
                .ifHasText(searchRequest.getName(),
                        name -> MemberDataSpecs.nameLike(searchRequest.getName()))
                .toSpec();
```

<br><br>

## 5.9 동적 인스턴스 생성

JPA는 쿼리 결과에서 임의의 객체를 동적으로 생성할 수 있는 기능을 제공하고있다.

> - JPQL(Java Persistence Query Language) : Entity 객체를 조회하는 객체지향 쿼리

```java
@Query("""
        select new com.myshop.order.query.dto.OrderView(
            o.number, o.state, m.name, m.id, p.name
        )
        from Order o join o.orderLines ol, Member m, Product p
        where o.orderer.memberId.id = :ordererId
        and o.orderer.memberId.id = m.id
        and index(ol) = 0
        and ol.productId.id = p.id
        order by o.number.number desc
        """)
List<OrderView> findOrderView(String ordererId);
```

- JPQL의 select 절에 생성할 인스턴스의 완전한 클래스 이름을 지정하고 괄호 안에 생성자에 인자로 전달할 값을 지정한다.

```java
package com.myshop.order.query.dto;

public class OrderView {

    private final String number;
    private final OrderState state;
    private final String memberName;
    private final String memberId;
    private final String productName;

    public OrderView(OrderNo number, OrderState state, String memberName, MemberId memberId, String productName) {
        this.number = number.getNumber();
        this.state = state;
        this.memberName = memberName;
        this.memberId = memberId.getId();
        this.productName = productName;
    }
// getter 생량
}
```

💡 동적인스턴스의 장점은 객체기준으로 쿼리를 작성하면서 지연 /  즉시 로딩과 같은 고민을 하지 않아도 원하는 형태의 데이터를 조회할 수 있다.

<br>

<br>

## 5.10 하이버네이트 @Subselect 사용

@Subselect, @Immutable, @Synchronize 는 하이버네이트 전용 애너테이션으로 쿼리 결과를 @Entity 로 매핑할 수 있는 유용한 기능이다.

```java
@Entity
@Immutable
@Subselect(
        """
                select o.order_number as number,
                o.version,
                o.orderer_id,
                o.orderer_name,
                o.total_amounts,
                o.receiver_name,
                o.state,
                o.order_date,
                p.product_id,
                p.name as product_name
                from purchase_order o inner join order_line ol
                    on o.order_number = ol.order_number
                    cross join product p
                where
                ol.line_idx = 0
                and ol.product_id = p.product_id"""
)
@Synchronize({"purchase_order", "order_line", "product"})
public class OrderSummary {
    @Id
    private String number;
    private long version;
    @Column(name = "orderer_id")
    private String ordererId;
    @Column(name = "orderer_name")
    private String ordererName;
    @Column(name = "total_amounts")
    private int totalAmounts;
    @Column(name = "receiver_name")
    private String receiverName;
    private String state;
    @Column(name = "order_date")
    private LocalDateTime orderDate;
    @Column(name = "product_id")
    private String productId;
    @Column(name = "product_name")
    private String productName;

    protected OrderSummary() {
    }

// getter 생략
}
```

- @Subselect : DBMS가 여러 테이블을 조인해서 조회한 결과를 한 테이블 처럼 보여주기 위한 용도로 뷰를 사용하는 것처럼 쿼리 실행 결과를 매핑할 테이블처럼 사용한다.
- @Immutable : 엔티티의 매핑 필드 / 프로퍼티가 변경되어도 DB에 반영하지 않는다. (@Subselect로 조회한 @Entity는 수정할 수 없음)
- @Synchronize : 엔티티를 로딩하기 전에 지정한 테이블과 관련된 변경이 발생하면 flush를 먼저 진행해 변경된 내용이 반영된다.
