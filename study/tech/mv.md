# 좋아요로 시작된 인덱스·캐시·MV 구조 개선 여정

> TL:DR :  인덱스, 캐시, MV 모두 도구일 뿐이고, 도구 선택은 항상 트레이드오프와 기회비용 위에서 결정

## 서론. 이 글을 작성하게 된 이유 <a href="#undefined" id="undefined"></a>



우선 제 상품 엔티티를 간단하게 소개하겠습니다.

```java
public class ProductEntity extends BaseEntity {
    private long likeCount = OL
}
```

처음 좋아요 기능은 이렇게 구현했습니다.

1. `like_count` 컬럼을 Product 테이블에 추가
2. 좋아요가 등록될 때, JPQL을 사용해 직접 증가 쿼리를 실행
3. 동시에 Product 엔티티를 JPA 영속성 컨텍스트에서 관리하며 상태 변경 감지(dirty checking) 수행

```java
public class LikeService {
    public LikeEntity upsertLike(UserEntity user, ProductEntity product) {
        productRepository.incrementLikeCount(product.getId());
        return likeRepository.save(LikeEntity.createEntity(user.getId(), product.getId()));
    }
}


public interface ProductJpaRepository extends JpaRepository<ProductEntity, Long> {
    @Query("UPDATE ProductEntity p SET p.likeCount = p.likeCount + 1 WHERE p.id = :productId")
    @Modifying
    void incrementLikeCount(@Param("productId") Long productId);
}
```

이 구조에서는 매 좋아요 등록 시 JPQL 업데이트를 사용함에 있어 영속성 컨텍스트와 동기화 안된다는 문제점과 \
상품 엔티티의 좋아요 수라는 비 정규화를 함에 있어서 좋아요 이벤트를 직접 처리 하였고, 즉 이게 도메인 책임 왜곡 \
이 아닐까 라는 생각이 들었습니다.&#x20;

더구나 좋아요 수 필드에 인덱스를 걸어 읽기 성능을 높였다 한들, 잦은 업데이트가 인덱스 재구성 비용을 높여 쓰기 부담을 증가시키고, 전체 응답 시간과 DB CPU 사용량이 함께 상승하는 문제를 야기 하는등 시간이 지나면 지날수록 기술 부채라는 느낌이 강했습니다.

이번 글에서는 캐시와 인덱스를 활용한 정합성과 실시간성 간에 트레이드 오프를 고민하면서 진행한 과정을 글로 소개를 하고자 합니다.



## 1장. 10만 개 상품과 인덱스, 그리고 카디널리티

좋아요는 계기였을 뿐이고, 돌고 돌아 성능을 생각하다 보니 제가 궁금했던 건 이거였습니다.

“인덱스를 잘 걸면 조회 성능은 얼마나 좋아질까?”\
“그리고, 인덱스를 걸었는데도 왜 어떤 쿼리는 여전히 느릴까?”

이 질문을 조금 더 구체적으로 만들기 위해, 제 상품 데이터를 이렇게 가정해 봤습니다.

* 총 상품 수: 100,000개
* Case A: 한 브랜드에 100,000개가 몰려 있는 경우
* Case B: 100개의 브랜드에 100,000개가 고르게 분포된 경우 (브랜드당 약 1,000개)

엔티티는 대략 이런 형태입니다.

```java
@Entity
public class ProductEntity extends BaseEntity {

    @Column(name = "brand_id", nullable = false)
    private Long brandId;

    @Column(name = "name", nullable = false, length = 200)
    private String name;

    @Column(name = "like_count", nullable = false)
    private Long likeCount = 0L;

    // createdAt, price, stock 등 기타 필드는 생략
}
```

이 상태에서 아래와 같은 조회 API들을 만들어 두고, 인덱스 유무에 따라 `EXPLAIN`과 응답 시간을 비교해 봤습니다.

```http
# 브랜드 필터만
GET /api/v1/products?brandId=1&size=20&page=0

# 브랜드 + 좋아요순
GET /api/v1/products?brandId=1&size=20&page=0&sort=likeCount,desc

# 전체 좋아요순
GET /api/v1/products?size=20&page=0&sort=likeCount,desc

# 상품명 검색
GET /api/v1/products?productName=테스트&size=20&page=0
```

먼저 아무 인덱스도 없이 조회하면, 예상대로 대부분 풀 스캔이 일어나고 응답 속도 인덱스를 적용후 요청 및 응답 속성

그렇다면 브랜드 인덱스의 기준인 brandId 가 1개 일때는 어떻게 되는지 한번 조사해봤습니다.

| 케이스           | 인덱스 유무 | 응답 속도 |
| ------------- | ------ | ----- |
| 1개 브랜드에 10만   | O      | 850ms |
| 100개 브랜드에 10만 | X      | 953ms |
| 100개 브랜드에 10만 | O      | 351ms |

인덱스 자체는 걸려 있지만 사실상 인덱스 없는 케이스와 동일한 응답 결과가 나왔습니다. \
\
인덱스가 걸려있다는 기준하여 케이스 별로 정리하면 다음과 같이 나왔습니다.

* Case A (한 브랜드에 10만 개 몰림):
  * `brand_id = 1` 조건으로 걸러도, 결국 10만 개 가까운 로우를 스캔.
  * 인덱스를 타긴 하지만, “거의 대부분”을 읽어야 해서 체감 성능이 거의 좋아지지 않습니다.
* Case B (100개 브랜드에 균등 분포):
  * `brand_id = 1`이면 대략 1,000개 상품이 존재.
  * 같은 인덱스여도, 조회해야 하는 로우 수가 10만 개 → 1,000개로 줄어들면서 인덱스 최적화가 되었음.

인덱스가 있느냐 없느냐보다, 그 인덱스가 실제로 데이터를 얼마나 **잘 골라주느냐**가 더 중요했습니다.

그렇다면 Case A 의 근본적인 원인은 무엇이었을까요 ?&#x20;

카디널리티(cardinality)의 문제였습니다.

* 한 브랜드에 10만 개가 몰려 있는 컬럼은, 사실상 “카디널리티가 낮은 컬럼”처럼 동작합니다.
* 100개 브랜드에 적당히 분산된 경우는 카디널리티가 상대적으로 높아서 인덱스 효과가 큽니다.

정리하면, 인덱스를 “걸었느냐/안 걸었느냐” 수준으로만 보다가, 아래와 같은 케이스를 고려해야한다는걸 체감했습니다.

* “이 컬럼이 실제 데이터 분포에서 얼마나 선별적으로 동작하는지”
* “이 인덱스가 전체 100,000개 중 몇 개까지 걸러줄 수 있는지”

***

## 2장. 캐시, 그리고 ‘정말 필요한 곳’에만 도입한다는 것 <a href="#id-2" id="id-2"></a>

인덱스를 테스트하면서 느낀 건 하나였습니다.

* 인덱스를 잘 걸어도, 트래픽이 몰리면 한계는 온다.
* 결국 “읽기 비용 자체를 없애는” 레이어가 하나 더 필요하다 → 캐시.

그래서 캐시 구조를 한 번 설계해 보기로 했습니다.\
로컬 캐시, 글로벌 캐시, 1차/2차 캐시 구조까지 개념은 한 번 훑어봤지만, \
이번에 제가 선택한 건 아주 단순한 형태였습니다.

* 1차 캐시(Caffeine 같은 JVM 로컬 캐시)는 “아 이런 게 있구나” 정도로 개념만 익히고,
* 실제 운영에는 글로벌 캐시(Redis) 하나만 두고, 그 위에 정책을 입히는 방식으로 접근했습니다.

***

#### 2-1. 캐시를 설계할 때의 기본 생각

처음에는 페이지네이션 조합마다 캐시를 다 걸 수도 있다고 생각했습니다.

* brandId, page, size, sort 조합별로 캐시 키를 만들어서
* 모든 목록 API 응답을 캐시해 두면 빠르지 않을까?&#x20;

하지만 트래픽 패턴을 떠올려보면 답은 명확했습니다.

* 대부분의 사용자는 1페이지만 보고,
* 2\~3페이지는 가끔,
* 그 이후는 거의 들어오지 않는다.

그럼 굳이 “안 입는 옷”까지 다 옷걸이에 걸어둘 필요는 없습니다.

그래서 “데이터 온도”를 기준으로 캐싱 전략을 나누기로 했습니다.

* Hot: 정말 자주 조회되는 구간 → 적극적으로 캐시
* Warm: 어느 정도 조회되는 구간 → 필요시 캐시
* Cold: 거의 조회되지 않는 구간 → 캐시 안 함

이걸 코드로 정리한 게 CacheStrategy입니다.

```java
@Getter
@RequiredArgsConstructor
public enum CacheStrategy {

    // Hot: 배치 갱신, TTL 30분
    HOT(30, TimeUnit.MINUTES, true, CacheUpdateStrategy.BATCH_REFRESH),

    // Warm: Cache-Aside, TTL 10분
    WARM(10, TimeUnit.MINUTES, false, CacheUpdateStrategy.CACHE_ASIDE),

    // Cold: 캐시 미사용
    COLD(0, TimeUnit.MINUTES, false, CacheUpdateStrategy.NO_CACHE);

    private final long ttl;
    private final TimeUnit timeUnit;
    private final boolean useBatchUpdate;
    private final CacheUpdateStrategy updateStrategy;

    public boolean shouldCache() {
        return this != COLD;
    }

    public boolean shouldUseBatchUpdate() {
        return useBatchUpdate;
    }
}

```

* HOT: 1페이지, 인기순 같은 “핵심 목록”에 사용
* WARM: 2\~3페이지 정도처럼 꾸준히 조회되지만, Hot만큼은 아닌 곳에 사용
* COLD: 4페이지 이후, 거의 안 들어오는 영역은 캐시를 아예 사용하지 않음





#### 2-2. 키 설계: “페이지 전체”가 아니라 ID 리스트

캐시 키는 다음과 같이 설계했습니다.

```java
@Component
public class CacheKeyGenerator {

    private static final String PRODUCT_PREFIX = "product";
    private static final String DETAIL_PREFIX = "detail";
    private static final String IDS_PREFIX = "ids";
    private static final String PAGE_PREFIX = "page";

    // 상품 상세: product:detail:{productId}
    public String generateProductDetailKey(Long productId) {
        return "product:detail:" + productId;
    }

    // 상품 ID 리스트: product:ids:{strategy}:{brandId}:{page}:{size}:{sort}
    public String generateProductIdsKey(CacheStrategy strategy, Long brandId, Pageable pageable) {
        return new StringJoiner(":")
            .add(PRODUCT_PREFIX)
            .add(IDS_PREFIX)
            .add(strategy.name().toLowerCase())
            .add(brandId != null ? String.valueOf(brandId) : "null")
            .add(String.valueOf(pageable.getPageNumber()))
            .add(String.valueOf(pageable.getPageSize()))
            .add(generateSortString(pageable.getSort()))
            .toString();
    }

    // 특정 전략 전체 패턴: product:ids:{strategy}:*
    public String generateProductIdsPattern(CacheStrategy strategy) {
        return "product:ids:" + strategy.name().toLowerCase() + ":*";
    }

    // 정렬 정보 문자열화: likeCount_desc,id_asc
    private String generateSortString(Sort sort) { ... }
}

```

여기서 중요한 건 두 가지였습니다.

1. “상품 목록 전체 DTO”를 캐시하지 않는다
2. “상품 ID 리스트만” 캐시한다

그 이유는 간단합니다.

* 상품 상세 정보(가격, 재고, 좋아요 수 등)는 변경 가능성이 큽니다.
* 목록에서 개별 상품의 필드 하나가 바뀔 때마다,
  * “목록 전체 캐시”를 날리기 시작하면 관리가 너무 복잡해집니다.
* 반대로, ID 리스트만 캐시하면:
  * 목록의 순서는 유지하되,
  * 각 상품의 상세 정보는 별도 캐시/조회로 관리할 수 있습니다.

즉, “정렬과 필터링의 결과(=ID 목록)”와 “각 상품의 실제 내용”을 분리해서 캐싱한 구조를 이번에 설계하였습니다.

#### 2-3. 결국 이번에는 글로벌 캐시 하나로만 간다

캐시를 설계하면서 1차/2차 캐시 구조도 한 번 고민해봤습니다.

* 1차 캐시: 애플리케이션 내부 로컬 캐시 (JVM 메모리 기반)
* 2차 캐시: Redis 같은 글로벌 캐시

개념상으론 “로컬 캐시로 먼저 막고, 없으면 글로벌 캐시로 떨어지는” 구조가 이상적으로 보였습니다.\
하지만 지금 단계에서 제가 풀고 싶은 문제를 다시 생각해 보니, 우선순위가 조금 달랐습니다.

* 로컬 + 글로벌 두 단계를 한 번에 도입하면
  * 동기화, 무효화 전파, 메모리 관리, 장애 시나리오까지 한 번에 고민해야 하고
* 지금은 “캐시 아키텍처를 화려하게 만드는 것”보다
  * “어디에, 어떤 키로, 어떤 정책으로 캐시를 쓰는 게 맞는지”를 먼저 검증하는 게 더 중요했습니다.

그래서 이번에는 의도적으로 단순한 선택을 했습니다.

* 1차/2차 캐시는 개념만 이해해두고
* 실제 구현은 글로벌 캐시(예: Redis) 한 단계만 사용
* 그 위에 Hot / Warm / Cold + CacheUpdateStrategy 조합으로만 정책을 나누는 방향

결국 캐시는 “모든 조회를 빠르게 하는 도구”라기보다,

* 트래픽이 몰리고, 비용이 크고, 정말 자주 조회되는 구간 중에서

선택적으로 찍어 누르는 도구라는 쪽에 좀 더 가깝다고 느꼈습니다.

그래서 캐시 구조는 이렇게 정리됐습니다.

* 글로벌 캐시 1단계
* 각 조건에 맞는 키 구조로 설계
* Hot(1페이지, 인기/핵심 구간), Warm(2\~3페이지), Cold(그 외)는 CacheStrategy로 표현

이렇게 캐싱과 인덱스에 대해서 이해를 하였으니 저의 기술 부채를 해결할수 있을것 같았습니다.

***

## 3장. 좋아요, 상품, 브랜드를 한 번에 보는 통계용 MV 설계

앞에서 인덱스와 캐시를 다루면서, 결국 이런 결론에 도달했습니다.

* 좋아요 수, 브랜드 정보, 상품 기본 정보는 조회할 때 항상 함께 필요하다.
* 그런데 매번 Product, Brand, Like를 조인해서 조회하는 건 비용이 크다.
* 좋아요는 실시간 1초 단위 정확도까지 필요하지는 않다.

그래서 조회 전용 통계 테이블을 따로 두기로 했습니다.

#### 3-1. 처음 MV는 ‘상품 + 좋아요’만 담았지만, 부족했다

처음에는 아주 단순하게 시작했습니다.

* `product_id`
* `like_count`

정도만 가진 “좋아요 통계용 MV”를 만들었는데, 실제로 써보니 이렇게 되었습니다.

* 결국 상품 이름, 가격, 브랜드명도 다 필요하다.
* 요청마다 다시 Product, Brand를 또 조인해야 한다.
* “통계용 테이블”이라기보다, 그저 like\_count만 따로 뺀 느낌이었다.

#### 3-2. 상품 + 브랜드 + 좋아요를 합친 MV 테이블

어차피 조회할 때 항상 같이 쓰는 정보라면, 아예 한 번에 조인해서 “조회 전용” 테이블에 담자. 라는 의도로\
최종적으로 만든 MV 엔티티는 대략 이런 형태입니다.

```java
@Entity
@Table(name = "product_materialized_view", indexes = {
    @Index(name = "idx_pmv_brand_id", columnList = "brand_id"),
    @Index(name = "idx_pmv_like_count", columnList = "like_count"),
    @Index(name = "idx_pmv_brand_like", columnList = "brand_id, like_count"),
    @Index(name = "idx_pmv_name", columnList = "name"),
    @Index(name = "idx_pmv_updated_at", columnList = "last_updated_at")
})
public class ProductMaterializedViewEntity extends BaseEntity {

    @Column(name = "product_id", nullable = false, unique = true)
    private Long productId;

    // 상품 정보
    private String name;
    private String description;
    @Embedded
    private Price price;
    private Integer stockQuantity;

    // 브랜드 정보
    private Long brandId;
    private String brandName;

    // 좋아요 정보
    private Long likeCount;

    // 각 도메인의 변경 시각
    private ZonedDateTime productUpdatedAt;
    private ZonedDateTime likeUpdatedAt;
    private ZonedDateTime brandUpdatedAt;

    // MV 자체가 마지막으로 동기화된 시간
    private ZonedDateTime lastUpdatedAt;
}

```

핵심 포인트는:

* Product, Brand, Like에서 조회에 필요한 필드만 모아서 한 테이블에 담았고,
* 자주 쓰는 조회 패턴에 맞춰 인덱스를 걸어뒀다는 점입니다.
  * 브랜드 필터링: `idx_pmv_brand_id`
  * 좋아요 순 정렬: `idx_pmv_like_count`
  * 브랜드 + 좋아요 순: `idx_pmv_brand_like`

이제 목록/상세 조회 시에는

* 원본 테이블 3개를 조인하지 않고,
* 이 MV 테이블만 조회하면서 캐시까지 태우는 구조가 됩니다.

#### 3-3. MV 갱신은 “2분마다 배치 + 변경분만 싱크”

실시간 1초 정확도를 포기하는 대신, **2분 간격 배치**로 MV를 갱신하기로 했습니다.

* 기준: `lastBatchTime` 이후에 변경된 데이터만 조회
* 변경 기준:
  * 상품이 바뀌었거나
  * 브랜드가 바뀌었거나
  * 좋아요가 바뀌었거나

이를 위해 QueryDSL로 다음과 같은 동기화용 쿼리를 만들었습니다.

```java
public List<ProductMVSyncDto> findChangedProductsForSync(ZonedDateTime lastBatchTime) {
    QProductEntity product = QProductEntity.productEntity;
    QBrandEntity brand = QBrandEntity.brandEntity;
    QLikeEntity like = QLikeEntity.likeEntity;

    return queryFactory
        .select(new QProductMVSyncDto(
            product.id,
            product.name,
            product.description,
            product.price.originPrice,
            product.price.discountPrice,
            product.stockQuantity,
            product.updatedAt,
            brand.id,
            brand.name,
            brand.updatedAt,
            like.id.count().coalesce(0L),
            like.updatedAt.max()
        ))
        .from(product)
        .leftJoin(brand).on(product.brandId.eq(brand.id))
        .leftJoin(like).on(like.productId.eq(product.id)
                           .and(like.deletedAt.isNull()))
        .where(
            product.updatedAt.after(lastBatchTime)
                .or(brand.updatedAt.after(lastBatchTime))
                .or(like.updatedAt.after(lastBatchTime)),
            product.deletedAt.isNull(),
            brand.deletedAt.isNull()
        )
        .groupBy( /* 생략 */ )
        .fetch();
}

```

그리고 배치에서는 이 DTO 리스트만 돌면서:

* 기존 MV가 있으면 변경 여부를 비교해서 업데이트
* 없으면 새로 INSERT

하는 식으로 동작합니다.

```java
@Transactional
public BatchUpdateResult syncMaterializedView() {
    List<ProductMVSyncDto> changed = mvRepository.findChangedProductsForSync(lastBatchTime);

    // 기존 MV 조회
    Map<Long, ProductMaterializedViewEntity> existing =
        mvRepository.findByIdIn(productIds).stream()
            .collect(toMap(ProductMaterializedViewEntity::getProductId, mv -> mv));

    for (ProductMVSyncDto dto : changed) {
        ProductMaterializedViewEntity mv = existing.get(dto.getProductId());

        if (mv == null) {
            // 신규 생성
            mvRepository.save(ProductMaterializedViewEntity.fromDto(dto));
        } else if (ProductMaterializedViewEntity.hasActualChangesFromDto(mv, dto)) {
            // 실제 값이 바뀐 경우에만 업데이트
            syncMVFromDto(mv, dto);
        }
    }

    lastBatchTime = ZonedDateTime.now();
}

```

#### 3-4. MV 위에 캐시를 입힌 조회 흐름

목록 조회는 이제 이렇게 흘러갑니다.

1. 컨트롤러 → 서비스에서 `ProductSearchFilter` 구성
2. 캐시 전략(HOT/WARM/COLD)에 따라 분기
3. HOT/WARM인 경우:
   * `product:ids:{strategy}:{brandId}:{page}:{size}:{sort}` 키로 ID 리스트 캐시 조회
   * 캐시 히트 → ID로 MV 테이블 조회
   * 캐시 미스 → MV 테이블에서 조회 후 ID 리스트 캐싱
4. COLD인 경우:
   * 그냥 MV 테이블만 조회 (캐시 없음)

서비스 레벨 코드는 대략 이런 형태입니다.

```java
@Transactional(readOnly = true)
public Page<ProductMaterializedViewEntity> getMVEntitiesByStrategy(
        ProductSearchFilter filter,
        CacheStrategy strategy
) {
    return switch (strategy) {
        case HOT -> getProductsWithCache(filter, CacheStrategy.HOT);
        case WARM -> getProductsWithCache(filter, CacheStrategy.WARM);
        default -> findBySearchFilter(filter);
    };
}

public Page<ProductMaterializedViewEntity> findBySearchFilter(ProductSearchFilter filter) {
    return mvRepository.findBySearchFilter(filter);
}

/**
 * 캐시를 사용하여 MV 엔티티를 조회합니다.
 */
private Page<ProductMaterializedViewEntity> getProductsWithCache(
        ProductSearchFilter filter,
        CacheStrategy strategy
) {
    Long brandId = filter.brandId();
    Pageable pageable = filter.pageable();

    // 1. 캐시에서 ID 리스트 조회
    Optional<List<Long>> cachedIds = productCacheService.getProductIdsFromCache(
            strategy, brandId, pageable
    );

    if (cachedIds.isPresent()) {
        log.debug("{} 캐시 히트 - brandId: {}, page: {}", strategy, brandId, pageable.getPageNumber());
        return findByIdsAsPage(cachedIds.get(), pageable);
    }

    log.debug("{} 캐시 미스 - brandId: {}, page: {}", strategy, brandId, pageable.getPageNumber());

    // 2. 통합된 searchFilter 기반 조회로 변경
    Page<ProductMaterializedViewEntity> mvProducts = mvRepository.findBySearchFilter(filter);

    // 3. ID 리스트 캐싱
    List<Long> productIds = mvProducts.getContent().stream()
            .map(ProductMaterializedViewEntity::getProductId)
            .toList();

    productCacheService.cacheProductIds(strategy, brandId, pageable, productIds);

    return mvProducts;
}
```

#### 3-5. Hot 캐시는 배치로 미리 채워둔다

핫 구간(예: 1페이지 인기 상품, 브랜드별 인기순 상위 페이지)은\
TTL + Cache-Aside만으로는 부족할 수 있습니다.\
그래서 따로 배치를 하나 더 두었습니다.

* 2분마다: MV 테이블 동기화
* 50분마다: Hot 캐시 갱신 (인기 상품, 브랜드별 인기 상위 N페이지)

핵심 로직만 보면:

```java
@Scheduled(fixedDelay = 120000)
public void syncMaterializedView() {
    BatchUpdateResult result = mvService.syncMaterializedView();

    if (result.hasChanges()) {
        cacheService.evictCachesAfterMVSync(
            result.getChangedProductIds(),
            result.getAffectedBrandIds()
        );
    }
}

@Scheduled(fixedRate = 50 * 60 * 1000, initialDelay = 60 * 1000)
public void refreshHotCache() {
    // 인기 상품 상위 100개
    Pageable top100 = PageRequest.of(0, 100, Sort.by(DESC, "likeCount"));
    Page<ProductMaterializedViewEntity> popular =
        mvService.findBySearchFilter(new ProductSearchFilter(null, null, top100));

    cacheService.cacheProductIds(HOT, null, top100,
        popular.getContent().stream().map(ProductMaterializedViewEntity::getProductId).toList());

    // 브랜드별 1~3페이지 인기순도 비슷한 방식으로 갱신
}

```

이렇게 해두면:

* MV는 2분 단위로 최신에 가깝게 유지됩니다.
* 가장 많이 조회되는 핫 구간은 별도의 캐시 배치로 항상 “준비된 상태”를 유지할 수 있습니다

MV 와 인덱스 , 캐싱을 조합해서 기능 구현을 하였을때 다음과 같은 기준으로 진행하였습니다.

* 조회는 MV + 인덱스 + 캐시로 극단적으로 빠르게, 쓰기/변경은 배치로 모아서 처리하자.”
* Product / Brand / Like 원본 테이블은 “사실 기록”에 집중
* `product_materialized_view`는 “조회에 최적화된 구조 + 인덱스 + 캐시”에 집중

좋아요 기능은 계기였고, 그걸 풀기 위해 도입한 MV + 캐시 구조가\
결국 “**읽기/쓰기 분리 + 통계성 데이터는 별도 테이블**”이라는 큰 방향으로 이어졌습니다.

## 4장. 인덱스, 캐시, MV - 어떤 순서로, 어떤 기준으로 선택할까

이제까지 과정을 진행하면서, 다음과 같이 정리가 잡혔습니다.

* 인덱스는 기본이지만, 데이터 분포(카디널리티)에 따라 효과가 극단적으로 다르다.
* 캐시는 트래픽이 몰리는 “핫 구간”에만 선택적으로 써야 한다.
* MV는 조인 비용이 큰 통계성 데이터에만 쓴다.

하지만 이 세 가지를 “한 번에 다 도입한다”는 건 아닙니다.\
문제의 본질에 따라 **순서와 기준**이 달라집니다.

#### 4-1. 문제 진단부터: “느린 이유가 뭔가?”

먼저 가장 중요한 건, **문제의 근본 원인**을 파악하는 겁니다.

* **Case 1: DB 조회 자체가 느리다** (풀 스캔, 조인 많음)\
  → **인덱스 우선**으로 근본 원인을 해결
* **Case 2: 조회는 빠르지만 호출량이 과다하다** (초당 1000건 이상)\
  → **캐시**로 DB 부하를 줄임
* **Case 3: 조회 시점 연산 비용이 크다** (복잡한 집계, 다중 조인)\
  → **MV**로 미리 계산된 결과를 저장

제가 좋아요 기능을 구현하면서 겪은 문제는 이 셋이 섞여 있었습니다.

* **인덱스 문제**: like\_count 인덱스를 걸어도 카디널리티가 낮아서 효과가 미미
* **캐시 문제**: 핫 구간(1페이지 인기순)에만 집중하지 않고, 페이지네이션 전체를 캐싱하려 함
* **MV 문제**: 매번 Product + Brand + Like를 조인해서 조회하는 비용

#### 4-2. 실제로 적용한 순서: 인덱스 → 캐시 → MV

제가 실제로 풀어간 순서는 이랬습니다.

**Step 1: 인덱스부터**\
먼저 like\_count, brand\_id에 인덱스를 걸어봤습니다.\
결과는 1장에서 봤듯이, 브랜드 분포에 따라 효과가 극명하게 갈렸습니다.\
“인덱스만으로는 한계가 있다”는 걸 확인했습니다.

**Step 2: 캐시로 핫 구간만 막기**\
인덱스 효과가 미미한 구간(예: 전체 좋아요순 1페이지)에 캐시를 넣어봤습니다.\
`product:ids:hot:*:0:20:likeCount_desc` 같은 키로 ID 리스트만 캐싱하니,\
DB 부하는 줄었지만 여전히 조인 비용은 남아 있었습니다.

**Step 3: MV로 조인 비용까지 없애기**\
결국 Product + Brand + Like를 미리 조인해서 `product_materialized_view`에 담았습니다.\
이제 조회는 MV 테이블만 타고, 그 위에 캐시까지 얹으니 응답 시간이 300ms → 15ms로 떨어졌습니다.

#### 4-3. 각 기술의 “적합한 영역”과 트레이드오프 (가성비)

이제 각 기술이 어떤 문제에 적합한지, 그리고 그 대가를 정리해보겠습니다.

**인덱스**

* **적합**: DB 내부 탐색 비용이 큰 쿼리 (WHERE, ORDER BY)
* **트레이드오프**: 쓰기 부담 증가 (UPDATE마다 인덱스 재구성)
* **한계**: 카디널리티가 낮으면 풀 스캔과 비슷한 효과

**캐시**

* **적합**: 반복 조회가 많은 핫 구간 (1페이지 인기순, 상세 정보)
* **트레이드오프**: 데이터 최신성 포기 (TTL, 무효화 복잡도)
* **한계**: 느린 쿼리의 근본 해결책은 아님

**MV**

* **적합**: 복잡한 조인/집계가 필요한 통계 조회
* **트레이드오프**: 배치 동기화 오버헤드, 실시간성 약화 (2분 지연)
* **한계**: 모든 데이터에 적용하면 저장소/동기화 비용 폭증

#### 4-4. “비용 vs 이득”으로 선택 (또 가성비)

결국 제가 내린 선택 기준은 간단합니다.

> “이 기술을 도입했을 때, 얻는 성능 이득이 유지/운영 비용보다 클 때만 쓴다.”

* **인덱스**: 핵심 조회 패턴만 선별적으로 (브랜드 + 좋아요순 조합 3개 정도만)
* **캐시**: 핫 구간(1\~3페이지)만, ID 리스트 형태로 최소화
* **MV**: 좋아요/브랜드 통계처럼 항상 같이 쓰이는 조인에만

그리고 순서는:

1. **인덱스**로 DB 자체 성능부터 확보
2. **캐시**로 트래픽 부하 분산
3. **MV**로 연산 비용까지 최적화

이 순서대로 쌓아가니, 각 레이어가 서로의 약점을 보완해주는 구조가 되었습니다.

#### 4-5. 앞으로의 숙제들

이번 작업을 통해 배운 건 많지만, 여전히 고민거리도 있습니다.

* **실시간성 vs 성능**: 2분 지연이 비즈니스적으로 괜찮은가?
* **캐시 무효화**: MV 변경 시 어떤 캐시를 얼마나 세밀하게 날릴지
* **모니터링**: 슬로우 쿼리, 캐시 히트율, 배치 지연 등을 어떻게 추적할지
* **확장성**: 상품이 100만 개, 브랜드가 1,000개가 되면?

좋아요 기능 하나를 풀면서 시작한 여정이, 결국 “읽기/쓰기 분리 + 계층적 최적화”라는 큰 그림으로 이어졌습니다.



## 5. 내 서비스는 가성비를 챙겼을까?&#x20;

이번 작업을 지나면서 딱 하나를 더 분명히 알게 됐습니다.

인프라 성능이 아무리 좋아져도,결국 **“가성비 있는 구조”를 설계하는 것**이 더 중요하다는 점입니다.

인덱스, 캐시, MV를 한 번에 다 쓰는 게 멋있어 보일 수 있지만, 실제로는 매번 이런 질문을 던져야 했습니다.

* 이 기능에 **얼마나 이득이 되는가?** (응답 시간, DB 부하, 개발/운영 편의)
* 그 이득을 얻기 위해 **얼마나 비용을 치르는가?** (복잡도, 장애 포인트, 모니터링/운영 부담)
* 지금 **정말 필요한가, 아니면 “있으면 좋겠음” 수준인가?**

이번에 제가 내린 선택들은 전부 이 기준에서 정리할 수 있습니다.

* 인덱스는 **카디널리티를 고려해, 진짜 효과 있는 컬럼에만** 걸었고
* 캐시는 **1\~3페이지 같은 핫 구간에만**, 그것도 **ID 리스트 형태**로만 사용했고
* MV는 **상품 + 브랜드 + 좋아요를 항상 같이 써야 하는 조회**에만 적용했습니다.

그 외의 것들은 “이론상으로는 있을 수 있는 최적화”였지만,\
지금 제가 가진 트래픽과 요구사항에서는 **기회비용만 키우는 군더더기**라고 판단해서 넣지 않았습니다.

결국 이번 글에서 하고 싶었던 이야기는 다음과 같습니다.

> “하고 싶은 최적화”보다 **“가져갈 만한 최적화”를 고르는 감각이 중요하다**\
> 인덱스, 캐시, MV 모두 도구일 뿐이고,\
> 도구 선택은 항상 트레이드오프와 기회비용 위에서 결정되어야 한다.

