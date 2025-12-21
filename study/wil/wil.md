# WIL: 카프카를 통한 글로벌 이벤트 적용

### 배운 점과 아쉬운 점 (WIL: What I Learned) <a href="#wil-what-i-learned" id="wil-what-i-learned"></a>

### **잘한 점 & 배운 점**

**1. Kafka 보장 원칙 체득**

```
At-Least-Once + At-Most-Once = 실질적 Exactly-Once
Producer: Outbox로 "절대 유실 없음" 확보
Consumer: 멱등성으로 "중복 무시" 완성
```

트랜잭션 경계가 모든 걸 결정. DB 변경 + Outbox 저장 = 원자성 확보

**2. 점진적 개선 하는 자세 익힘**

```
다이렉트 발행 ❌ → Outbox → 멱등성 → 배치 → 동시성
한 번에 다 바꾸지 않고, 문제 발견 → 최소 해결 반복
```

**3. 성능 최적화 에 대한 이해**

```
메모리 락을 이용한 동시성 처리
개별 처리 → 배치 3000개
```

### **아쉬운 점 & 다음 도전**

**1. Kafka Config 심화 테스트 부족**

```
기본 설정만:
- producer.retries=3, acks=all, idempotence=true
- consumer.max.poll.records=3000, manual ack

다음엔 테스트해볼 것:
- linger.ms, batch.size (Producer throughput)
- fetch.min.bytes, fetch.max.wait (Consumer 효율)
- compression.type (네트워크 절약)
```

**2. 멀티 인스턴스 검증 미실시**

```
text현재: 단일 인스턴스 → 메모리 락 최적
미래: Redis 분산락 재도입 + Consumer 그룹 스케일링 필요
파티션 증가 (1→6, 12) + replicas 설정 검증
```

### 다음 계획

```
1️⃣ 다양한 Kafka Config 기반의 테스트 기록 (with Post) 
2️⃣ 멀티 인스턴스 환경 구축 후 테스트 (Docker Compose)
```
