---
title:  "JPA OneToOne 관계에서의 N+1 이슈 해결 : PersistentAttributeInterceptable"
excerpt: "JPA OneToOne 관계에서의 N+1 이슈 해결"

categories:
  - JPA
tags:
  - JPA, OneToOne, N+1, LazyToOne, PersistentAttributeInterceptable
---
# 한줄 요약
JPA에서 양방향 OneToOne 연관관계 사용시 지연로딩이 먹히지 않는 문제는 부모 엔티티(관계형 DB 테이블 구조상 부모)에서 `PersistentAttributeInterceptable`를 상속받게하고, 자식 엔티티(연관관계의 주인)에 `@LazyToOne` 어노테이션을 달아줌으로써 해결할 수 있다.  
#  
# 배경
다음은 가상의 쇼핑몰에서 상품과 상품 부가정보 엔티티로 1:1 양방향 연관관계를 맺고있다.
### 상품(Goods) : 상품의 기본정보를 담고있다. 정말 기본만...
```java
@Entity
@Getter @Setter
@Builder
@NoArgsConstructor @AllArgsConstructor
public class Goods {
    @Id
    @GeneratedValue
    private Long id;

    private String name;

    @OneToOne(mappedBy = "goods", fetch = FetchType.LAZY)
    private GoodsExtra goodsExtra;
}
```
### 상품 부가정보(GoodsExtra) : 종합 쇼핑몰이라 가정하고, 옵셔널한 부가 정보들이 들어있다. ISBN이나 해외통관여부와 같은
```java
@Entity
@Getter
@Builder
@NoArgsConstructor @AllArgsConstructor
public class GoodsExtra {
    @Id
    @GeneratedValue
    private Long id;

    @OneToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "goods_id")
    private Goods goods;

    private Boolean isImported;
}
```

GoodsExtra의 경우 상품마다 있을수도 없을수도 있다. 책이나 병행수입 상품과 같은게 아니라면 굳이 정보를 만들 필요가 없다. 상품의 기본정보만 제공해주다 때에 따라서 부가정보도 같이 제공해주어야 하는 도메인 특성이 있을 수 있다.(전 직장에서는 그랬다.) 매번 outer join해서 가져오기에는 불필요한 정보가 많고, 오용할 여지가 많다. 이럴 경우 좋은 방법이 지연로딩이다. 하지만 양방향 OneToOne 관계에서는 관계를 맺은 엔티티를 자동으로 불러오는 N+1 문제가 발생한다. 아래 테스트 코드를 보자.
```java
@SpringBootTest
@Transactional
class GoodsTest {
    @Autowired EntityManager em;

    @Test
    public void 일대일매핑() {
        Goods goods = Goods.builder()
                .name("코로나 방역 마스크")
                .build();

        GoodsExtra goodsExtra = GoodsExtra.builder()
                .isImported(Boolean.TRUE)
                .goods(goods)
                .build();

        goods.setGoodsExtra(goodsExtra);

        em.persist(goods);
        em.persist(goodsExtra);
        em.flush();
        em.detach(goods);
        em.detach(goodsExtra);

        Long goodsId = goods.getId();
        Goods result = em.find(Goods.class, goodsId);

        assertThat(result).isNotNull();
    }
}
```
뭐 좀 코드가 더러운 느낌이 없지않아 있는데, 아무튼 돌려보면 아래와 같이 Goods 테이블뿐만 아니라 Goods_Extra 테이블까지 조회한다. 내가 의도한 건 쿼리가 1번만 나가는 것인데, 2번 나간다. 
![N+1 발생상황](https://user-images.githubusercontent.com/56215503/134383965-dc36d99e-c391-4930-90e2-495f8c225e23.png)  

#

# 이유 : 가정이다. 공식문서를 찾을 수 없었다..
많이들 아시다싶이 JPA의 지연로딩은 프록시 패턴으로 구현되어 있다. 그리고 OneToOne 연관관계의 부모 엔티티에 자식 엔티티가 붙어 있을수도 있고 없을수도 있다. OneToMany라면 부모에게 자식들이 없더라도 `EmptyList`로 비어있는 값을 프록시가 대신해줄 수 있다. 컬렉션을 대신할 프록시 객체를 미리 할당해두고 지연로딩으로 불러온 결과가 `EmptyList`라도 '컬렉션이 비어있다'라는 의미를 갖는 엄연한 값으로 실제 값을 리턴해주면 된다.  
하지만 컬렉션이 아닌 단수?로 객체를 참조하는 상황에서는 값이 없는 상태를 뜻하는 `null`을 프록시 객체가 대신해주기 애매하다. `goodsExtra = null`일수도 있는데, 어떠한 프록시 객체(값이 있는)라도 goodsExtra에 대입되는 순간 `null`이 아니게 된다. 값이 없는 상태인데 값이 있는? 상태... 뭐 그런 상태인 것이다. 그렇기 때문에 무턱대고 프록시로 감싸줄 수 없다. 단방향 또는 자식 엔티티의 경우에는 해당 테이블에 관계를 맺는 테이블의 외래키가 매핑될 것 이니까, 해당 컬럼을 통해 null인지 알 수 있다. 실제로 테스트해보면 외래키가 null이면 해당 엔티티도 null로 매핑하여 바로 들고오고 아니면 프록시 상태로 들고온다.  
그 외래키로 지연로딩으로 객체를 불러왔는데 쿼리 결과가 없을 수 있지 않을 경우에는? 이란 의문도 드는데, 실제로 테이블간 외래키 제약을 설정하지 않았더라도 JPA에서는 연관관계로 매핑했으면 FK 제약이란 가정을 하는 것 같다.  
이유는 시간나는대로 계속 다듬어나가야 할 것 같다. 이유를 찾기 위해 아래 링크를 참조했다.  
[백기선님 블로그](https://www.whiteship.me/-ed-95-98-ec-9d-b4-eb-b2-84-eb-84-a4-ec-9d-b4-ed-8a-b8-onetoone--ec-97-b0-ea-b4-80--ea-b4-80-ea-b3-84-lazy-fetching-ec-9d-b4--ec-95-88--eb-a8-b9-ec-96/)  
[권남이님 블로그](https://kwonnam.pe.kr/wiki/java/jpa/one-to-one)

#

# 그래서 해결은
hibernate 5.2.1 이하에서는 `org.hibernate.bytecode.internal.javassist.FieldHandled` 인터페이스를 구현해서 쓰면 된다. 대부분 한국 블로그에 해결책으로 제시되어 있으니 참고하면 된다. 근데 이미 hibernate 버전이 5.2.1을 훌쩍 뛰어넘었다. 샘플 프로젝트는 5.4.3이다. FieldHandled 인터페이스가 아예 없어졌다. 5.1버전 이상에서는 `org.hibernate.engine.spi.PersistentAttributeInterceptable` 인터페이스를 구현하면 된다. 코드가 좀 더러워지긴 하는데 지연로딩의 이점을 얻을 수 있다.  
### 그래서 바뀐 코드
```java
@Entity
@Builder
@NoArgsConstructor @AllArgsConstructor
public class Goods implements PersistentAttributeInterceptable {
    @Id @Getter
    @GeneratedValue
    private Long id;

    private String name;

    @LazyToOne(LazyToOneOption.NO_PROXY)
    @OneToOne(mappedBy = "goods", fetch = FetchType.LAZY)
    private GoodsExtra goodsExtra;

    @Transient
    private PersistentAttributeInterceptor interceptor;

    public GoodsExtra getGoodsExtra() {
        if (interceptor != null) {
            return (GoodsExtra) interceptor.readObject(this, "goodsExtra", goodsExtra);
        }
        return goodsExtra;
    }

    void setGoodsExtra(final GoodsExtra goodsExtra) {
        if (interceptor != null) {
            this.goodsExtra = (GoodsExtra) interceptor.writeObject(this, "goodsExtra", this.goodsExtra, goodsExtra);
        }
        this.goodsExtra = goodsExtra;
    }

    @Override
    public PersistentAttributeInterceptor $$_hibernate_getInterceptor() {
        return interceptor;
    }

    @Override
    public void $$_hibernate_setInterceptor(final PersistentAttributeInterceptor interceptor) {
        this.interceptor = interceptor;
    }
}
```
hibernate의 프록시를 사용하지 않고, 인터셉터를 통해 지연로딩처럼 동작하게 한다. 객체 로딩 시점에 인터셉터를 초기화하고, `@LazyToOne` 어노테이션이 붙은 객체는 무시하여 아예 null로 할당해버린다.  
![N+1이 안터짐](https://user-images.githubusercontent.com/56215503/134525768-0b8df639-76fb-4af1-b66e-0ac9600d74a9.png)
일단 쿼리는 goods 테이블 조회만 1번 실행되었다.  
인터셉터 내부에 지연로딩 대상이 되는 객체를 지연로딩 시점에 저장한다. 그리고 별도로 구현한 getter 메서드에서는 인터셉터에 저장된 객체를 반환한다. 어차피 외부에서는 getter를 통해 해당 객체에 접근하기 때문에 이 메서드를 통해 `null`을 받거나 유효한 객체를 받을 수 있다. 위에서 문제가 되었던 `nullable`에 대한 문맥적 꼬임은 객체의 데이터에 대한 접근 해석때문에 발생한 것이고, 메서드를 통해서라면 실제 데이터에 구애받지 않고 원하는 것을 줄 수 있다.  
한가지 주의할 점은 일반적인 LazyLoading에서는 연관 객체를 참조한 시점이 아니라 그 안의 값을 참조할 때 쿼리가 날라가는데, 이 방식에서는 객체 참조 시점에 바로 쿼리가 날라간다는 것이다.  
![객체 참조 시점에 로딩](https://user-images.githubusercontent.com/56215503/134527910-9cc132d6-4ca0-444a-946b-8f3f66223f4b.png)
그냥 `result.getGoodsExtra()`만 해줬는데도, 쿼리가 날라가고 해당 연관객체가 메모리에 로딩되었다.  

PersistentAttributeInterceptable는 아래 링크를 참조하였다.
<https://github.com/jlmc/hibernate-tunings/tree/master/instrumentation>  
하이버네이트 사이트에 개제된 이슈  
<https://hibernate.atlassian.net/browse/HHH-12053>

#
틀린 내용이 있거나 다른 의견이 있으시면 피드백 부탁 드립니다.