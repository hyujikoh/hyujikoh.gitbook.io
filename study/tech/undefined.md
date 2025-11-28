---
hidden: true
---

# 이 데이터는 핫데이터 인가 ?

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

```
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

이번 글에서는 정합성과 실시간성 간에 트레이드 오프를 고민하면서 진행한 과정을 글로 소개를 하고자 합니다.

