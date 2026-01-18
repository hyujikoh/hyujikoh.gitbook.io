# JPA/Hibernate MultiTenancy 활용한 다중 스키마 지원 아키텍처 구현

### 개요 <a href="#undefined" id="undefined"></a>

재직하고 있는 회사의 솔루션의 요구사항 중 하나인, **하나의 애플리케이션이 여러 고객(테넌트)의 데이터를 논리적으로 분리하여 관리**해야 하는 상황이 존재합니다. 이때 여러 가지 멀티테넌시 전략 중 **"공유 데이터베이스, 분리된 스키마(Shared Database, Separate Schemas)"** 방식은 효율성과 관리 용이성 측면에서 많은 이점을 제공하기 때문에 기존에 운영하고 있는 솔루션을 고도화를 위해 실제 구현및 적용을 해봤습니다.

이 글에서는 **Spring Boot 3.2.5, Java 17, MySQL 환경**에서 JPA/Hibernate의 멀티테넌시 기능을 활용하여, **HTTP 요청 헤더에 기반한 동적 스키마 전환 로직**을 구현하는 방법을 다룹니다.

### 요구사항

* **하나의 DB 인스턴스 안에 여러 스키마 존재**
* **HTTP 헤더 `datasource`에 스키마명을 담아 요청**
* **DispatcherServlet 진입 전 헤더 값 추출 및 스키마 유효성 검증**
* **존재하지 않는 스키마일 경우 예외 처리**
* **존재하는 스키마일 경우 해당 스키마로 동적 전환**
* **요청 완료 후 ThreadLocal 정리로 멱등성 보장**
* **수백 개의 스키마 추가/제거에도 코드 변경 없는 확장성**

### 왜 Hibernate 멀티테넌시를 선택했나? <a href="#hibernate" id="hibernate"></a>

처음에는 `AbstractRoutingDataSource`를 고려했지만, 다음과 같은 한계가 있었습니다.

### AbstractRoutingDataSource의 한계

* **모든 스키마별 DataSource Bean을 미리 등록해야 함**
* **스키마가 많아질수록 설정 코드 복잡성 증가**
* **새로운 스키마 추가 시 코드 변경 및 재배포 필요**
* **각 DataSource마다 별도 커넥션 풀로 인한 리소스 낭비**

> 물론 내가 좀더 조사를 해보고 관련된 레퍼런스 및 문서를 꼼꼼하게 봤다면 ARDS 를 이용해 구현이 가능했겠지만, 아쉽게도 다음 기회를 통해 해보는 시간이 있었음 바란다.&#x20;

이러한 이유 때문에 보다 간결하면서 설정이 가능한 레퍼런스를 찾다 선택한 것이 MultiTenancy 였습니다.

### Hibernate 멀티테넌시의 강점

* **단 하나의 DataSource Bean만 관리 (이게 선택의 큰 요인)**
* **프레임워크 수준의 동적 스키마 전환**
* **수백 개 스키마 추가에도 코드 변경 불필요**&#x20;
* **비즈니스 로직과 멀티테넌시 로직의 완전한 분리**
* **ThreadLocal 기반 멱등성 보장**

### 서비스 핵심 컴포넌트 <a href="#undefined" id="undefined"></a>

전체 아키텍처는 다음과 같은 핵심 컴포넌트들로 구성된다:

1. **SchemaValidationInterceptor**: HTTP 요청 가로채기 및 스키마 검증
2. **SchemaContextHolder**: ThreadLocal 기반 스키마 컨텍스트 관리
3. **TenantIdentifierResolver**: Hibernate에 현재 테넌트 식별자 제공
4. **SchemaMultiTenantConnectionProvider**: 동적 스키마 전환 핵심 로직
5. **JpaConfig**: Spring Bean을 Hibernate에 주입하는 설정

### 구현 과정 <a href="#undefined" id="undefined"></a>

### 1. application.yml 설정

기본 DataSource 설정과 Hibernate 멀티테넌시 설정을 정의 합니다. 중요한 점은 `multi_tenant_connection_provider`와 `tenant_identifier_resolver`를 YAML에 직접 지정하지 않고, `JpaConfig`에서 Spring Bean으로 주입한다는 것입니다.&#x20;

```yaml
spring:
  config:
    activate:
      on-profile: development
  datasource:
    url: jdbc:mysql://localhost:3306/main_db?useSSL=false&allowPublicKeyRetrieval=true&serverTimezone=Asia/Seoul
    username: ${DB_USERNAME:admin}
    password: ${DB_PASSWORD:password}
    driver-class-name: com.mysql.cj.jdbc.Driver
    hikari:
      maximum-pool-size: 30
      auto-commit: false
  jpa:
    hibernate:
      ddl-auto: update
      default_batch_fetch_size: 100
    database-platform: org.hibernate.dialect.MySQLDialect
    properties:
      hibernate:
        multiTenancy: SCHEMA
        # connection provider와 resolver는 JpaConfig에서 주입
        connection.provider_disables_autocommit: true
        connection.handling_mode: DELAYED_ACQUISITION_AND_RELEASE_AFTER_TRANSACTION
```

### 2. ThreadLocal 기반 스키마 컨텍스트 관리

요청 스레드별로 현재 스키마 이름을 안전하게 저장하고 관리합니다.

```java
package com.example.multischema.context;

public class SchemaContextHolder {
    
    private static final ThreadLocal<String> CONTEXT = new ThreadLocal<>();
    
    public static void setCurrentSchema(String schema) {
        CONTEXT.set(schema);
    }
    
    public static String getCurrentSchema() {
        return CONTEXT.get();
    }
    
    public static void clear() {
        CONTEXT.remove();
    }
}
```

### 3. HTTP 요청 인터셉터 구현

Spring MVC의 `HandlerInterceptor`를 구현하여 모든 HTTP 요청을 가로채고 스키마 검증을 수행합니다.

```java
package com.example.multischema.interceptor;

import com.example.multischema.context.SchemaContextHolder;
import org.springframework.beans.factory.annotation.Qualifier;
import org.springframework.jdbc.core.JdbcTemplate;
import org.springframework.stereotype.Component;
import org.springframework.web.servlet.HandlerInterceptor;

import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;
import javax.sql.DataSource;

@Component
public class SchemaValidationInterceptor implements HandlerInterceptor {
    
    private final JdbcTemplate defaultJdbcTemplate;
    
    // 스키마가 없을 경우 설정하는 디폴트 스
    private static final String DEFAULT_SCHEMA_NAME = "main_db";
    
    public SchemaValidationInterceptor(@Qualifier("dataSource") DataSource defaultDataSource) {
        this.defaultJdbcTemplate = new JdbcTemplate(defaultDataSource);
    }
    
    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {
        String schemaName = request.getHeader("datasource");
        
        // 헤더가 없거나 비어있는 경우 기본 스키마 사용
        if (schemaName == null || schemaName.trim().isEmpty()) {
            SchemaContextHolder.setCurrentSchema(DEFAULT_SCHEMA_NAME);
            return true;
        }
        
        // MySQL information_schema를 통한 스키마 존재 여부 검증
        if (!isSchemaExists(schemaName)) {
            response.sendError(HttpServletResponse.SC_BAD_REQUEST, 
                "Invalid or non-existent schema: '" + schemaName + "'");
            return false;
        }
        
        // 유효한 스키마명을 ThreadLocal에 저장
        SchemaContextHolder.setCurrentSchema(schemaName);
        return true;
    }
    
    @Override
    public void afterCompletion(HttpServletRequest request, HttpServletResponse response, 
                              Object handler, Exception ex) throws Exception {
        // 요청 처리 완료 후 ThreadLocal 정리 (멱등성 보장)
        SchemaContextHolder.clear();
    }
    
    private boolean isSchemaExists(String schemaName) {
        String sql = "SELECT COUNT(*) FROM information_schema.schemata WHERE SCHEMA_NAME = ?";
        try {
            Integer count = defaultJdbcTemplate.queryForObject(sql, Integer.class, schemaName);
            return count != null && count > 0;
        } catch (Exception e) {
            System.err.println("Error checking schema existence for '" + schemaName + "': " + e.getMessage());
            return false;
        }
    }
}
```

### 4. Hibernate TenantIdentifierResolver

Hibernate가 현재 어떤 스키마를 사용해야 하는지 물어볼 때 호출되는 컴포넌트입니다.

```java
package com.example.multischema.config.datasource;

import com.example.multischema.context.SchemaContextHolder;
import org.hibernate.context.spi.CurrentTenantIdentifierResolver;
import org.springframework.stereotype.Component;

@Component
public class TenantIdentifierResolver implements CurrentTenantIdentifierResolver {
    
    private static final String DEFAULT_SCHEMA_NAME = "main_db";
    
    @Override
    public String resolveCurrentTenantIdentifier() {
        String currentSchema = SchemaContextHolder.getCurrentSchema();
        return (currentSchema != null) ? currentSchema : DEFAULT_SCHEMA_NAME;
    }
    
    @Override
    public boolean validateExistingCurrentSessions() {
        return true;
    }
}
```

### 5. 멀티테넌트 커넥션 provider 구현

해당 컴포넌트는 **동적 스키마 전환을 수행하기 위한 핵심 컴포넌트 입니다**. 단일 DataSource에서 커넥션을 가져온 후, `USE <schema_name>` 쿼리를 실행하여 스키마를 동적으로 변경합니다. 이를 통해 수백개의 스키마를 처리하여도 문제없이 동적으로 변환이 가능합니다.

```java
package com.example.multischema.config.datasource;

import org.hibernate.engine.jdbc.connections.spi.MultiTenantConnectionProvider;
import org.springframework.beans.factory.annotation.Qualifier;
import org.springframework.stereotype.Component;

import javax.sql.DataSource;
import java.sql.Connection;
import java.sql.SQLException;
import java.sql.Statement;

@Component
public class SchemaMultiTenantConnectionProvider implements MultiTenantConnectionProvider {
    
    private final DataSource dataSource;
    private static final String DEFAULT_SCHEMA_NAME = "main_db";
    
    public SchemaMultiTenantConnectionProvider(@Qualifier("dataSource") DataSource dataSource) {
        this.dataSource = dataSource;
    }
    
    @Override
    public Connection getAnyConnection() throws SQLException {
        return dataSource.getConnection();
    }
    
    @Override
    public void releaseAnyConnection(Connection connection) throws SQLException {
        try {
            // 커넥션 풀 반환 전 기본 스키마로 리셋
            try (Statement stmt = connection.createStatement()) {
                stmt.execute("USE " + DEFAULT_SCHEMA_NAME);
            }
        } catch (SQLException e) {
            System.err.println("Failed to reset schema to '" + DEFAULT_SCHEMA_NAME + 
                "' before releasing connection: " + e.getMessage());
        } finally {
            connection.close();
        }
    }
    
    @Override
    public Connection getConnection(Object object) throws SQLException {
        String tenantIdentifier = (String) object;
        final Connection connection = getAnyConnection();
        
        try {
            // USE 쿼리로 스키마 변경 (setSchema() 대신 사용)
            try (Statement stmt = connection.createStatement()) {
                stmt.execute("USE " + tenantIdentifier);
            }
        } catch (SQLException e) {
            throw new SQLException("Could not alter JDBC connection to specified schema [" + 
                tenantIdentifier + "]", e);
        }
        
        return connection;
    }
    
    @Override
    public void releaseConnection(Object object, Connection connection) throws SQLException {
        releaseAnyConnection(connection);
    }
    
    @Override
    public boolean supportsAggressiveRelease() {
        return false;
    }
    
    @Override
    public boolean isUnwrappableAs(Class<?> unwrapType) {
        return false;
    }
    
    @Override
    public <T> T unwrap(Class<T> unwrapType) {
        return null;
    }
}
```

### 6. JPA 설정 클래스

Spring이 관리하는 Bean 인스턴스들을 Hibernate EntityManagerFactory에 직접 주입을 하는데, 이전에 yaml 파일에 작성하였을때 발생하는 오류를 Hibernate가 직접 인스턴스화 하여 해결을 합니다.

```java
package com.example.multischema.config;

import com.example.multischema.config.datasource.SchemaMultiTenantConnectionProvider;
import com.example.multischema.config.datasource.TenantIdentifierResolver;
import org.hibernate.cfg.AvailableSettings;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.orm.jpa.JpaVendorAdapter;
import org.springframework.orm.jpa.LocalContainerEntityManagerFactoryBean;
import org.springframework.orm.jpa.vendor.HibernateJpaVendorAdapter;

import javax.sql.DataSource;
import java.util.HashMap;
import java.util.Map;

@Configuration
public class JpaConfig {
    
    @Autowired
    private DataSource dataSource;
    
    @Autowired
    private SchemaMultiTenantConnectionProvider multiTenantConnectionProvider;
    
    @Autowired
    private TenantIdentifierResolver tenantIdentifierResolver;
    
    @Bean
    public LocalContainerEntityManagerFactoryBean entityManagerFactory() {
        LocalContainerEntityManagerFactoryBean em = new LocalContainerEntityManagerFactoryBean();
        em.setDataSource(dataSource);
        em.setPackagesToScan("com.example.multischema.domain"); // 모든 엔티티 패키지
        
        JpaVendorAdapter vendorAdapter = new HibernateJpaVendorAdapter();
        em.setJpaVendorAdapter(vendorAdapter);
        
        Map<String, Object> properties = new HashMap<>();
        
        // 멀티테넌시 핵심 설정
        properties.put("hibernate.multiTenancy", "SCHEMA");
        properties.put(AvailableSettings.MULTI_TENANT_CONNECTION_PROVIDER, multiTenantConnectionProvider);
        properties.put(AvailableSettings.MULTI_TENANT_IDENTIFIER_RESOLVER, tenantIdentifierResolver);
        
        // 기본 JPA/Hibernate 설정
        properties.put(AvailableSettings.HBM2DDL_AUTO, "update");
        properties.put(AvailableSettings.DIALECT, "org.hibernate.dialect.MySQLDialect");
        properties.put(AvailableSettings.DEFAULT_BATCH_FETCH_SIZE, 100);
        
        em.setJpaPropertyMap(properties);
        return em;
    }
}
```

### 7. 웹 설정 클래스

인터셉터를 Spring MVC 체인에 등록합니다.

```java
package com.example.multischema.config;

import com.example.multischema.interceptor.SchemaValidationInterceptor;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.servlet.config.annotation.InterceptorRegistry;
import org.springframework.web.servlet.config.annotation.WebMvcConfigurer;

@Configuration
public class WebConfig implements WebMvcConfigurer {
    
    private final SchemaValidationInterceptor schemaValidationInterceptor;
    
    @Autowired
    public WebConfig(SchemaValidationInterceptor schemaValidationInterceptor) {
        this.schemaValidationInterceptor = schemaValidationInterceptor;
    }
    
    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(schemaValidationInterceptor).addPathPatterns("/**");
    }
}
```

### 8. 엔티티 예시

`createdAt`, `updatedAt` 컬럼에 MySQL 기본값을 명시적으로 설정하여 DB 레벨에서 기본값이 보장되도록 합니다.

```java
package com.example.multischema.domain.common;

import jakarta.persistence.Column;
import jakarta.persistence.EnumType;
import jakarta.persistence.Enumerated;
import jakarta.persistence.MappedSuperclass;
import lombok.Getter;
import org.hibernate.annotations.CreationTimestamp;
import org.hibernate.annotations.UpdateTimestamp;

import java.time.LocalDateTime;

@Getter
@MappedSuperclass
public class BaseEntity {
    
    @Enumerated(EnumType.STRING)
    @Column(name = "status", nullable = false, length = 10)
    protected Status status = Status.ACTIVE;
    
    @Enumerated(EnumType.STRING)
    @Column(name = "is_deleted", nullable = false, length = 10)
    protected IsDeleted isDeleted = IsDeleted.N;
    
    @CreationTimestamp
    @Column(name = "created_at", nullable = false, updatable = false,
            columnDefinition = "TIMESTAMP DEFAULT CURRENT_TIMESTAMP")
    private LocalDateTime createdAt;
    
    @UpdateTimestamp
    @Column(name = "updated_at", nullable = false,
            columnDefinition = "TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP")
    private LocalDateTime updatedAt;
    
    public void updateStatus(Status status) {
        this.status = status;
    }
    
    public void delete() {
        this.isDeleted = IsDeleted.Y;
    }
    
    public enum Status {
        ACTIVE, INACTIVE
    }
    
    public enum IsDeleted {
        Y, N
    }
}
```



```java
package com.example.multischema.domain.product;

import com.example.multischema.domain.common.BaseEntity;
import jakarta.persistence.*;
import lombok.*;

@Entity
@Table(name = "products")
@Getter
@NoArgsConstructor(access = AccessLevel.PROTECTED)
@ToString
public class Product extends BaseEntity {
    
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @Column(name = "product_code", length = 20, unique = true, nullable = false)
    private String productCode;
    
    @Column(name = "product_name", length = 100, nullable = false)
    private String productName;
    
    @Column(name = "price", nullable = false)
    private Integer price;
    
    @Column(name = "description", length = 500)
    private String description;
    
    @Builder
    public Product(String productCode, String productName, Integer price, String description) {
        this.productCode = productCode;
        this.productName = productName;
        this.price = price;
        this.description = description;
    }
    
    public void updateProduct(String productName, Integer price, String description) {
        this.productName = productName;
        this.price = price;
        this.description = description;
    }
}
```

### 9. Repository 및 Service 예시

```java
package com.example.multischema.domain.product;

import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.stereotype.Repository;

import java.util.List;
import java.util.Optional;

@Repository
public interface ProductRepository extends JpaRepository<Product, Long> {
    
    Optional<Product> findByProductCode(String productCode);
    
    List<Product> findByProductNameContaining(String productName);
    
    List<Product> findByStatus(BaseEntity.Status status);
}
```

```java
package com.example.multischema.domain.product;

import lombok.RequiredArgsConstructor;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.util.List;

@Service
@RequiredArgsConstructor
@Transactional(readOnly = true)
public class ProductService {
    
    private final ProductRepository productRepository;
    
    public Page<Product> getAllProducts(Pageable pageable) {
        return productRepository.findAll(pageable);
    }
    
    public Product getProductById(Long id) {
        return productRepository.findById(id)
                .orElseThrow(() -> new RuntimeException("Product not found with id: " + id));
    }
    
    public Product getProductByCode(String productCode) {
        return productRepository.findByProductCode(productCode)
                .orElseThrow(() -> new RuntimeException("Product not found with code: " + productCode));
    }
    
    @Transactional
    public Product createProduct(String productCode, String productName, Integer price, String description) {
        Product product = Product.builder()
                .productCode(productCode)
                .productName(productName)
                .price(price)
                .description(description)
                .build();
        
        return productRepository.save(product);
    }
    
    @Transactional
    public Product updateProduct(Long id, String productName, Integer price, String description) {
        Product product = getProductById(id);
        product.updateProduct(productName, price, description);
        return product;
    }
    
    @Transactional
    public void deleteProduct(Long id) {
        Product product = getProductById(id);
        product.delete();
    }
}
```

### 10. Controller 예시

```java
package com.example.multischema.api;

import com.example.multischema.domain.product.Product;
import com.example.multischema.domain.product.ProductService;
import lombok.RequiredArgsConstructor;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/api/products")
@RequiredArgsConstructor
public class ProductController {
    
    private final ProductService productService;
    
    @GetMapping
    public ResponseEntity<Page<Product>> getAllProducts(Pageable pageable) {
        Page<Product> products = productService.getAllProducts(pageable);
        return ResponseEntity.ok(products);
    }
    
    @GetMapping("/{id}")
    public ResponseEntity<Product> getProductById(@PathVariable Long id) {
        Product product = productService.getProductById(id);
        return ResponseEntity.ok(product);
    }
    
    @GetMapping("/code/{productCode}")
    public ResponseEntity<Product> getProductByCode(@PathVariable String productCode) {
        Product product = productService.getProductByCode(productCode);
        return ResponseEntity.ok(product);
    }
    
    @PostMapping
    public ResponseEntity<Product> createProduct(@RequestBody CreateProductRequest request) {
        Product product = productService.createProduct(
                request.getProductCode(),
                request.getProductName(),
                request.getPrice(),
                request.getDescription()
        );
        return ResponseEntity.ok(product);
    }
    
    @PutMapping("/{id}")
    public ResponseEntity<Product> updateProduct(@PathVariable Long id, 
                                               @RequestBody UpdateProductRequest request) {
        Product product = productService.updateProduct(
                id,
                request.getProductName(),
                request.getPrice(),
                request.getDescription()
        );
        return ResponseEntity.ok(product);
    }
    
    @DeleteMapping("/{id}")
    public ResponseEntity<Void> deleteProduct(@PathVariable Long id) {
        productService.deleteProduct(id);
        return ResponseEntity.ok().build();
    }
    
    // DTO 클래스들
    public static class CreateProductRequest {
        private String productCode;
        private String productName;
        private Integer price;
        private String description;
        
        // getters and setters
        public String getProductCode() { return productCode; }
        public void setProductCode(String productCode) { this.productCode = productCode; }
        public String getProductName() { return productName; }
        public void setProductName(String productName) { this.productName = productName; }
        public Integer getPrice() { return price; }
        public void setPrice(Integer price) { this.price = price; }
        public String getDescription() { return description; }
        public void setDescription(String description) { this.description = description; }
    }
    
    public static class UpdateProductRequest {
        private String productName;
        private Integer price;
        private String description;
        
        // getters and setters
        public String getProductName() { return productName; }
        public void setProductName(String productName) { this.productName = productName; }
        public Integer getPrice() { return price; }
        public void setPrice(Integer price) { this.price = price; }
        public String getDescription() { return description; }
        public void setDescription(String description) { this.description = description; }
    }
}
```

### 11. 메인 애플리케이션 클래스

<pre class="language-java"><code class="lang-java"><strong>package com.example.multischema;
</strong>
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class MultiSchemaApplication {
    
    public static void main(String[] args) {
        SpringApplication.run(MultiSchemaApplication.class, args);
    }
}
</code></pre>

### 사용 방법 <a href="#undefined" id="undefined"></a>

### API 요청 예시

**기본 스키마 사용 (헤더 없음)**

```bash
GET http://localhost:8080/api/products?page=0&size=10
```

**특정 스키마 사용**

```bash
GET http://localhost:8080/api/products?page=0&size=10
datasource: tenant_a
```

**다른 스키마 사용**

```bash
POST http://localhost:8080/api/products
Content-Type: application/json
datasource: tenant_b

{
  "productCode": "PROD001",
  "productName": "Sample Product",
  "price": 10000,
  "description": "This is a sample product"
}
```

**존재하지 않는 스키마 (오류 발생)**

```bash
GET http://localhost:8080/api/products
datasource: non_existent_schema
# 응답: 400 Bad Request
```

### 트러블슈팅 <a href="#undefined" id="undefined"></a>

### 1. `Not a managed type` 오류

**원인**: JPA 엔티티 스캔 범위에 포함되지 않은 경우

**해결**: `JpaConfig`의 `setPackagesToScan`에 모든 엔티티 패키지 포함

```java
em.setPackagesToScan("com.example.multischema.domain");
```

### 2. 스키마 전환이 안 되는 경우

**원인**: 트랜잭션 시작 시점과 커넥션 획득 시점의 불일치

**해결**: `application.yml`에 지연된 커넥션 획득 설정 추가

```yaml
spring.jpa.properties.hibernate:
  connection.provider_disables_autocommit: true
  connection.handling_mode: DELAYED_ACQUISITION_AND_RELEASE_AFTER_TRANSACTION
```

### 결론 <a href="#undefined" id="undefined"></a>

해당 구조를 통해서 다음과 같은 장점을 얻었습니다.

* **높은 확장성**: 수백 개의 스키마 추가에도 코드 변경 불필요
* **낮은 결합성**: 비즈니스 로직과 멀티테넌시 로직의 완전한 분리
* **멱등성 보장**: ThreadLocal 관리로 요청 간 완전한 격리
* **간결한 설정**: 단일 DataSource로 모든 스키마 관리
* **안정성**: USE 쿼리 방식으로 DB 드라이버 호환성 문제 해결

물론 단점도 엄연히 존재 합니다.

* MySQL 에서만 테스트를 하였지만,  postgreql 에서는 정상 작동 여부를 확인 못했습니다. 즉 jpa 에 장점인 db 에 종속적인 부분이 생겼습니다.&#x20;
* 스키마가 존재하지 않을시에 예외 케이스를 작성하였습니다. 다만 사전에 검증 및 수행하는 로직을 분리하여 스키마 를 use 하기 전에 적용하였으면 보다 안전한 동작이 가능했다고 생각합니다.&#x20;
* 특정 스키마에서 트래픽이 몰릴경우에 서버에서 어떤 고객사 스키마로 호출되고 관리가 되는지 관리가 안되는 문제점이 존재합니다.&#x20;
