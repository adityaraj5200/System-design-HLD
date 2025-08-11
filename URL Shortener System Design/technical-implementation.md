# URL Shortener - Technical Implementation Guide

## Technology Stack

### Backend
- **Language**: Java 17+
- **Framework**: Spring Boot 3.x
- **Database**: PostgreSQL 14+
- **Cache**: Redis 7+
- **Message Queue**: RabbitMQ

### Infrastructure
- **Containerization**: Docker + Docker Compose
- **Load Balancer**: NGINX
- **Monitoring**: Prometheus + Grafana

## Core Implementation

### URL Service
```java
@Service
@Transactional
public class UrlService {
    
    private final UrlRepository urlRepository;
    private final RedisTemplate<String, Url> redisTemplate;
    private final AnalyticsService analyticsService;
    
    public UrlService(UrlRepository urlRepository, 
                     RedisTemplate<String, Url> redisTemplate,
                     AnalyticsService analyticsService) {
        this.urlRepository = urlRepository;
        this.redisTemplate = redisTemplate;
        this.analyticsService = analyticsService;
    }
    
    public Url createShortUrl(String originalUrl, String userId) {
        String shortCode = generateShortCode();
        Url url = new Url();
        url.setId(UUID.randomUUID());
        url.setShortCode(shortCode);
        url.setOriginalUrl(originalUrl);
        url.setUserId(userId);
        url.setActive(true);
        url.setCreatedAt(LocalDateTime.now());
        
        urlRepository.save(url);
        cacheUrl(url);
        
        return url;
    }
    
    public String redirectToOriginal(String shortCode) {
        Url cachedUrl = getCachedUrl(shortCode);
        if (cachedUrl != null) {
            analyticsService.trackClick(shortCode);
            return cachedUrl.getOriginalUrl();
        }
        
        Url url = urlRepository.findByShortCode(shortCode)
            .orElseThrow(() -> new UrlNotFoundException("URL not found"));
            
        if (!url.isActive()) {
            throw new UrlNotFoundException("URL is not active");
        }
        
        cacheUrl(url);
        analyticsService.trackClick(shortCode);
        
        return url.getOriginalUrl();
    }
}
```

### Database Schema
```sql
CREATE TABLE urls (
    url_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    short_code VARCHAR(50) UNIQUE NOT NULL,
    original_url TEXT NOT NULL,
    user_id UUID REFERENCES users(user_id),
    is_active BOOLEAN DEFAULT true,
    expires_at TIMESTAMP,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_urls_short_code ON urls(short_code);
CREATE INDEX idx_urls_user_id ON urls(user_id);
```

### Docker Compose
```yaml
version: '3.8'
services:
  app:
    build: .
    ports: ["8080:8080"]
    depends_on: [postgres, redis]
    environment:
      - SPRING_PROFILES_ACTIVE=docker
      - SPRING_DATASOURCE_URL=jdbc:postgresql://postgres:5432/urlshortener
      - SPRING_REDIS_HOST=redis
      - SPRING_REDIS_PORT=6379
    
  postgres:
    image: postgres:14-alpine
    environment:
      POSTGRES_DB: urlshortener
      POSTGRES_USER: user
      POSTGRES_PASSWORD: password
    ports: ["5432:5432"]
    volumes:
      - postgres_data:/var/lib/postgresql/data
      
  redis:
    image: redis:7-alpine
    ports: ["6379:6379"]
    volumes:
      - redis_data:/data

volumes:
  postgres_data:
  redis_data:
```

### NGINX Configuration
```nginx
upstream app_servers {
    server 127.0.0.1:8080;
    server 127.0.0.1:8081;
}

server {
    listen 80;
    server_name short.ly;
    
    location /api/ {
        proxy_pass http://app_servers;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
    
    location ~ ^/[A-Za-z0-9]{6,}$ {
        proxy_pass http://app_servers;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

## Testing Strategy

### Unit Tests
```java
@ExtendWith(MockitoExtension.class)
class UrlServiceTest {
    
    @Mock
    private UrlRepository urlRepository;
    
    @Mock
    private RedisTemplate<String, Url> redisTemplate;
    
    @Mock
    private AnalyticsService analyticsService;
    
    @InjectMocks
    private UrlService urlService;
    
    @Test
    void shouldCreateShortUrl() {
        // Given
        String originalUrl = "https://example.com";
        String userId = "user123";
        
        when(urlRepository.save(any(Url.class))).thenAnswer(invocation -> {
            Url url = invocation.getArgument(0);
            url.setId(UUID.randomUUID());
            return url;
        });
        
        // When
        Url result = urlService.createShortUrl(originalUrl, userId);
        
        // Then
        assertThat(result.getShortCode()).hasLength(6);
        assertThat(result.getOriginalUrl()).isEqualTo(originalUrl);
        assertThat(result.getUserId()).isEqualTo(userId);
        verify(urlRepository).save(any(Url.class));
    }
}
```

### Entity Classes
```java
@Entity
@Table(name = "urls")
public class Url {
    @Id
    @GeneratedValue(strategy = GenerationType.UUID)
    private UUID id;
    
    @Column(name = "short_code", unique = true, nullable = false)
    private String shortCode;
    
    @Column(name = "original_url", nullable = false)
    private String originalUrl;
    
    @Column(name = "user_id")
    private String userId;
    
    @Column(name = "is_active")
    private boolean active = true;
    
    @Column(name = "expires_at")
    private LocalDateTime expiresAt;
    
    @Column(name = "created_at")
    private LocalDateTime createdAt;
    
    // Getters and setters
}

@Entity
@Table(name = "users")
public class User {
    @Id
    @GeneratedValue(strategy = GenerationType.UUID)
    private UUID id;
    
    @Column(unique = true, nullable = false)
    private String email;
    
    @Column(name = "password_hash", nullable = false)
    private String passwordHash;
    
    @Column(name = "created_at")
    private LocalDateTime createdAt;
    
    // Getters and setters
}
```

### Repository Interface
```java
@Repository
public interface UrlRepository extends JpaRepository<Url, UUID> {
    Optional<Url> findByShortCode(String shortCode);
    List<Url> findByUserIdAndActiveOrderByCreatedAtDesc(String userId, boolean active);
    boolean existsByShortCode(String shortCode);
}
```

### Controller
```java
@RestController
@RequestMapping("/api/v1")
@Validated
public class UrlController {
    
    private final UrlService urlService;
    
    public UrlController(UrlService urlService) {
        this.urlService = urlService;
    }
    
    @PostMapping("/urls")
    public ResponseEntity<UrlResponse> createShortUrl(@Valid @RequestBody CreateUrlRequest request) {
        Url url = urlService.createShortUrl(request.getOriginalUrl(), request.getUserId());
        return ResponseEntity.status(HttpStatus.CREATED)
            .body(UrlResponse.from(url));
    }
    
    @GetMapping("/urls")
    public ResponseEntity<Page<UrlResponse>> getUserUrls(
            @RequestParam(defaultValue = "0") int page,
            @RequestParam(defaultValue = "20") int size) {
        Page<Url> urls = urlService.getUserUrls(page, size);
        Page<UrlResponse> response = urls.map(UrlResponse::from);
        return ResponseEntity.ok(response);
    }
}
```

### Performance Optimization
- **Caching**: Redis for URL mappings with Spring Data Redis
- **Indexing**: Database indexes on frequently queried fields
- **Connection Pooling**: HikariCP for database connection pooling
- **Rate Limiting**: Spring Boot Actuator with custom rate limiting
- **Async Processing**: @Async for analytics tracking
- **Connection Pooling**: Redis connection pooling with Lettuce

### Application Properties
```properties
# Database Configuration
spring.datasource.url=jdbc:postgresql://localhost:5432/urlshortener
spring.datasource.username=user
spring.datasource.password=password
spring.datasource.hikari.maximum-pool-size=20
spring.datasource.hikari.minimum-idle=5

# Redis Configuration
spring.redis.host=localhost
spring.redis.port=6379
spring.redis.timeout=2000ms
spring.redis.lettuce.pool.max-active=8
spring.redis.lettuce.pool.max-idle=8

# JPA Configuration
spring.jpa.hibernate.ddl-auto=validate
spring.jpa.show-sql=false
spring.jpa.properties.hibernate.dialect=org.hibernate.dialect.PostgreSQLDialect
```

### Dockerfile
```dockerfile
# Multi-stage build for Java application
FROM maven:3.9.5-openjdk-17 AS builder

WORKDIR /app
COPY pom.xml .
COPY src ./src

# Build the application
RUN mvn clean package -DskipTests

# Runtime stage
FROM openjdk:17-jre-slim

WORKDIR /app

# Install dumb-init for proper signal handling
RUN apt-get update && apt-get install -y dumb-init && rm -rf /var/lib/apt/lists/*

# Create non-root user
RUN groupadd -r appuser && useradd -r -g appuser appuser

# Copy the built JAR
COPY --from=builder --chown=appuser:appuser /app/target/*.jar app.jar

# Switch to non-root user
USER appuser

# Expose port
EXPOSE 8080

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD curl -f http://localhost:8080/actuator/health || exit 1

# Start application
ENTRYPOINT ["dumb-init", "--"]
CMD ["java", "-jar", "app.jar"]
```

### Maven Dependencies
```xml
<dependencies>
    <!-- Spring Boot Starter Web -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    
    <!-- Spring Boot Starter Data JPA -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-jpa</artifactId>
    </dependency>
    
    <!-- Spring Boot Starter Data Redis -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-redis</artifactId>
    </dependency>
    
    <!-- PostgreSQL Driver -->
    <dependency>
        <groupId>org.postgresql</groupId>
        <artifactId>postgresql</artifactId>
        <scope>runtime</scope>
    </dependency>
    
    <!-- Spring Boot Starter Validation -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-validation</artifactId>
    </dependency>
    
    <!-- Spring Boot Starter Actuator -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-actuator</artifactId>
    </dependency>
</dependencies>
```

This implementation provides a solid foundation for a scalable URL shortener system using Spring Boot and Java.
