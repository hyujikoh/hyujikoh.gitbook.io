# WIL: 이벤트로 관심사 분리와 의사소통 방법

### 이번주 한 일 <a href="#undefined" id="undefined"></a>

주문/결제/쿠폰/좋아요 기능을 스프링 이벤트로 리팩토링.

```
createOrderForCardPayment() → 도메인 이벤트 + 응용 이벤트 분리
├── PaymentEntity.completeWithEvent() → PaymentCompletedEvent 발행
├── LikeEntity.createWithEvent() → LikeChangedEvent 발행
├── UserBehaviorTracker.trackOrderCreate() → UserBehaviorEvent 발행
└── DataPlatformEventHandler → AFTER_COMMIT + @Async 처리
```

### 배운 것 <a href="#undefined" id="undefined"></a>

**스프링 이벤트 = 로컬 이벤트**

* 한 애플리케이션 내에서 관심사 분리 도구
* `@TransactionalEventListener(AFTER_COMMIT) + @Async`로 트랜잭션 완전 분리

**도메인 이벤트 vs 응용 이벤트**

```
도메인: "결제가 완료됐다" (PaymentEntity가 직접 발행)
응용: "사용자가 주문했다" (파사드에서 발행)
인프라: 데이터 플랫폼/분석 전송 (AFTER_COMMIT에서 처리)
```

### 깨달은 점 <a href="#undefined" id="undefined"></a>

**이벤트 = 누군가 관심 있는 과거의 사건**\
응용 메서드가 모든 관심사를 직접 조율하는 대신:

```
❌ 응용 메서드: productService.deductStock() + couponService.consume() + tracker.track()
✅ 도메인: registerEvent(PaymentCompletedEvent)
   ↓ 관심 있는 쪽만 구독
```

**핵심 질문**\
"이 일은 여기서 끝까지 책임져야 하나? 아니면 '일어났다'고만 알릴까?"

**결과**

* 응용 메서드 가벼워짐
* 트랜잭션 경계 명확 (AFTER\_COMMIT)
* 장애 격리 (비동기 + 실패 무시)

### 다음주 계획 <a href="#undefined" id="undefined"></a>

* 이벤트 재처리 큐 (Outbox 패턴)
* 리스너 구조화 (Envelope 패턴으로 단순화)
