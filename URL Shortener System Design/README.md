# URL Shortener System Design

## Table of Contents
1. [Functional Requirements](#functional-requirements)
2. [Non-Functional Requirements](#non-functional-requirements)
3. [High-Level Design](#high-level-design)
4. [Database Design](#database-design)
5. [API Design](#api-design)
6. [Detailed Justifications](#detailed-justifications)
7. [Scale Considerations](#scale-considerations)
8. [Bonus Features](#bonus-features)

---

## 1. Functional Requirements (FR)

### Core Features
- **URL Shortening**: Convert long URLs to short, memorable URLs
- **URL Redirection**: Redirect short URLs to original long URLs
- **Custom Aliases**: Allow users to create custom short URLs
- **Analytics**: Track click counts, referrer information, geographic data
- **User Management**: User registration, authentication, and URL management
- **Expiration**: Set expiration dates for short URLs
- **QR Code Generation**: Generate QR codes for short URLs

### Inputs/Outputs
- **Input**: Long URL, optional custom alias, expiration date, user preferences
- **Output**: Short URL, QR code, analytics dashboard, click tracking

### Key User Interactions
- Create short URL (with/without custom alias)
- Access short URL (redirect to original)
- View analytics and click statistics
- Manage personal URL collection
- Set URL expiration and access controls

---

## 2. Non-Functional Requirements (NFR)

### Performance
- **Latency**: 
  - URL redirection: < 100ms (95th percentile)
  - URL creation: < 500ms (95th percentile)
  - Analytics retrieval: < 200ms (95th percentile)
- **Throughput**: 
  - URL redirections: 10,000+ requests/second
  - URL creations: 1,000+ requests/second

### Scalability
- **Horizontal Scaling**: Support 100M+ users and 1B+ URLs
- **Vertical Scaling**: Handle traffic spikes (10x normal load)
- **Geographic Distribution**: Global deployment with regional optimization

### Availability & Reliability
- **Uptime**: 99.9% availability (8.76 hours downtime/year)
- **Fault Tolerance**: Graceful degradation during partial failures
- **Data Durability**: 99.999% data retention

### Consistency
- **URL Redirection**: Strong consistency (always redirect to correct URL)
- **Analytics**: Eventual consistency (acceptable for reporting)
- **User Data**: Strong consistency for critical operations

### Security
- **URL Validation**: Prevent malicious URL injection
- **Rate Limiting**: Prevent abuse and DoS attacks
- **Authentication**: Secure user management
- **Data Privacy**: GDPR compliance for user data

### Cost-Effectiveness
- **Infrastructure**: Optimize for cost per request
- **Storage**: Efficient data compression and archiving
- **Bandwidth**: Minimize unnecessary data transfer

---

## 3. High-Level Design (HLD)

### Component Architecture

```
┌─────────────────┐    ┌──────────────────┐    ┌─────────────────┐
│   Web Client   │    │   Mobile App    │    │   API Client    │
└─────────┬───────┘    └────────┬─────────┘    └─────────┬───────┘
          │                      │                        │
          └──────────────────────┼────────────────────────┘
                                 │
                    ┌─────────────▼─────────────┐
                    │      Load Balancer       │
                    │    (NGINX/HAProxy)      │
                    └─────────────┬─────────────┘
                                  │
                    ┌─────────────▼─────────────┐
                    │      API Gateway         │
                    │   (Rate Limiting, Auth)  │
                    └─────────────┬─────────────┘
                                  │
                    ┌─────────────▼─────────────┐
                    │    URL Service           │
                    │  (Core Business Logic)   │
                    └─────────────┬─────────────┘
                                  │
          ┌───────────────────────┼───────────────────────┐
          │                       │                       │
┌─────────▼─────────┐  ┌─────────▼─────────┐  ┌─────────▼─────────┐
│   Analytics       │  │   User Service   │  │   Cache Layer    │
│   Service         │  │  (Auth, Profile) │  │   (Redis)        │
└─────────┬─────────┘  └─────────┬─────────┘  └─────────┬─────────┘
          │                      │                       │
          └──────────────────────┼───────────────────────┘
                                 │
                    ┌─────────────▼─────────────┐
                    │      Message Queue        │
                    │      (RabbitMQ/Kafka)    │
                    └─────────────┬─────────────┘
                                  │
                    ┌─────────────▼─────────────┐
                    │      Database Layer       │
                    │   (PostgreSQL + Redis)   │
                    └───────────────────────────┘
```

### Component Interactions

1. **Client → Load Balancer**: Distributes traffic across multiple API instances
2. **Load Balancer → API Gateway**: Handles authentication, rate limiting, and routing
3. **API Gateway → Services**: Routes requests to appropriate microservices
4. **Services → Cache**: Check cache before hitting database
5. **Services → Database**: Persistent storage for URLs and user data
6. **Services → Message Queue**: Asynchronous processing for analytics and notifications

### Design Justifications

**Microservices vs Monolith**: 
- **Chosen**: Microservices for high-scale version
- **Why**: Independent scaling, technology diversity, fault isolation
- **Trade-off**: Increased complexity, network overhead, distributed transactions

**Message Queue vs Direct Calls**:
- **Chosen**: Message queue for analytics and notifications
- **Why**: Decoupling, fault tolerance, scalability
- **Trade-off**: Eventual consistency, increased latency

**Java vs Node.js**:
- **Chosen**: Java with Spring Boot for enterprise-grade reliability
- **Why**: Strong typing, mature ecosystem, excellent performance, enterprise support
- **Trade-off**: Higher memory usage, longer startup time, but better runtime performance

---

## 4. Database Design

### Main Entities

```sql
-- Users table
CREATE TABLE users (
    user_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email VARCHAR(255) UNIQUE NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    is_active BOOLEAN DEFAULT true
);

-- URLs table
CREATE TABLE urls (
    url_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    short_code VARCHAR(50) UNIQUE NOT NULL,
    original_url TEXT NOT NULL,
    user_id UUID REFERENCES users(user_id),
    title VARCHAR(255),
    description TEXT,
    is_active BOOLEAN DEFAULT true,
    expires_at TIMESTAMP,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Clicks table (for analytics)
CREATE TABLE clicks (
    click_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    url_id UUID REFERENCES urls(url_id),
    ip_address INET,
    user_agent TEXT,
    referrer TEXT,
    country VARCHAR(2),
    city VARCHAR(100),
    clicked_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Custom aliases table
CREATE TABLE custom_aliases (
    alias_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    alias VARCHAR(50) UNIQUE NOT NULL,
    url_id UUID REFERENCES urls(url_id),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

### Database Choice: PostgreSQL

**Why PostgreSQL over alternatives:**

1. **vs MongoDB**: 
   - **Chosen**: PostgreSQL for ACID compliance and complex queries
   - **Why**: URL shorteners need strong consistency for redirects
   - **Trade-off**: Less flexible schema, but better for relational data

2. **vs Cassandra**:
   - **Chosen**: PostgreSQL for strong consistency
   - **Why**: Analytics queries need complex joins and aggregations
   - **Trade-off**: Less horizontal scaling, but better query performance

### Indexing Strategy

```sql
-- Primary indexes
CREATE INDEX idx_urls_short_code ON urls(short_code);
CREATE INDEX idx_urls_user_id ON urls(user_id);
CREATE INDEX idx_clicks_url_id ON clicks(url_id);
CREATE INDEX idx_clicks_clicked_at ON clicks(clicked_at);

-- Composite indexes
CREATE INDEX idx_urls_user_active ON urls(user_id, is_active);
CREATE INDEX idx_clicks_url_date ON clicks(url_id, clicked_at);
```

### Sharding Strategy

**Shard by user_id**: 
- Distributes load evenly
- Keeps user data together
- Simplifies queries per user

**Replication**: 
- Master-slave setup for read scaling
- 3 replicas for high availability

---

## 5. API Design

### RESTful Endpoints

#### Authentication
```
POST /api/v1/auth/register
POST /api/v1/auth/login
POST /api/v1/auth/refresh
POST /api/v1/auth/logout
```

#### URL Management
```
POST   /api/v1/urls                    # Create short URL
GET    /api/v1/urls                    # List user URLs
GET    /api/v1/urls/{id}              # Get URL details
PUT    /api/v1/urls/{id}              # Update URL
DELETE /api/v1/urls/{id}              # Delete URL
POST   /api/v1/urls/{id}/custom       # Set custom alias
```

#### Analytics
```
GET /api/v1/urls/{id}/analytics       # Get click analytics
GET /api/v1/urls/{id}/clicks          # Get click details
GET /api/v1/analytics/summary          # Get user summary
```

#### Redirection
```
GET /{shortCode}                       # Redirect to original URL
```

### Request/Response Examples

#### Create Short URL
```json
POST /api/v1/urls
{
    "original_url": "https://example.com/very/long/url",
    "custom_alias": "my-link",
    "expires_at": "2024-12-31T23:59:59Z",
    "title": "My Custom Link"
}

Response:
{
    "id": "uuid",
    "short_code": "my-link",
    "short_url": "https://short.ly/my-link",
    "original_url": "https://example.com/very/long/url",
    "created_at": "2024-01-01T00:00:00Z",
    "expires_at": "2024-12-31T23:59:59Z"
}
```

### Authentication & Rate Limiting

**JWT Authentication**:
- Access token: 15 minutes
- Refresh token: 7 days
- Secure HTTP-only cookies

**Rate Limiting**:
- Anonymous: 10 requests/minute
- Authenticated: 100 requests/minute
- Premium: 1000 requests/minute

---

## 6. Detailed Justifications

### Redis vs Memcached

**Chosen**: Redis
**Why**:
- Persistence (survives restarts)
- Rich data structures (sets, sorted sets for analytics)
- Built-in expiration and TTL
- Pub/sub for real-time features

**Trade-offs**:
- Higher memory usage
- More complex configuration

**Alternative problems**:
- Memcached: Data loss on restart, limited data structures

### PostgreSQL vs MongoDB

**Chosen**: PostgreSQL
**Why**:
- ACID compliance for critical operations
- Complex analytics queries with JOINs
- Mature ecosystem and tooling
- Better for structured data

**Trade-offs**:
- Less flexible schema
- Harder horizontal scaling

**Alternative problems**:
- MongoDB: Eventual consistency issues, complex aggregation queries

### Microservices vs Monolith

**Chosen**: Microservices for high-scale
**Why**:
- Independent scaling per service
- Technology diversity
- Fault isolation
- Team autonomy

**Trade-offs**:
- Distributed system complexity
- Network overhead
- Data consistency challenges

---

## 7. Scale Considerations

### Low-Scale Version (~1,000 users)

**Simplified Architecture**:
```
┌─────────────┐    ┌─────────────┐    ┌─────────────┐
│   Client    │───▶│   Monolith  │───▶│ PostgreSQL │
└─────────────┘    │   Service   │    │   (Local)  │
                   └─────────────┘    └─────────────┘
```

**What can be simplified**:
- Single monolith service
- Local PostgreSQL database
- No caching layer
- No message queues
- Single server deployment
- No load balancing

**Benefits**:
- Simpler development and deployment
- Lower operational overhead
- Easier debugging and monitoring

### High-Scale Version (Millions/Billions of users)

**Enhanced Architecture**:
```
┌─────────────┐    ┌─────────────┐    ┌─────────────┐    ┌─────────────┐
│   CDN       │───▶│ Load Bal.  │───▶│ API Gateway│───▶│ Services    │
└─────────────┘    └─────────────┘    └─────────────┘    └─────────────┘
                                                          │
                   ┌─────────────┐    ┌─────────────┐    │
                   │   Cache     │◀───│   Queue     │◀───┘
                   │  (Redis)    │    │ (Kafka)     │
                   └─────────────┘    └─────────────┘
                          │                   │
                   ┌─────────────┐    ┌─────────────┐
                   │ Database    │    │ Analytics   │
                   │ (Sharded)   │    │ (Warehouse) │
                   └─────────────┘    └─────────────┘
```

**Additional Components**:
- **CDN**: Global content distribution
- **Load Balancers**: Traffic distribution across regions
- **Horizontal Scaling**: Multiple service instances
- **Data Sharding**: Distribute data across multiple databases
- **Caching Strategy**: Multi-level caching (L1, L2, L3)
- **Async Processing**: Background job processing
- **Observability**: Comprehensive logging, metrics, and alerting

**Scaling Strategies**:
- **Database**: Read replicas, connection pooling, query optimization
- **Cache**: Redis clusters, cache warming, intelligent invalidation
- **Services**: Auto-scaling based on CPU/memory metrics
- **Storage**: Data archiving, compression, lifecycle management

---

## 8. Bonus Features

### Caching Strategy

**What to Cache**:
- **L1 (Application)**: Frequently accessed URLs, user sessions
- **L2 (Redis)**: URL mappings, analytics data, rate limit counters
- **L3 (CDN)**: Static assets, frequently accessed short URLs

**Cache Invalidation**:
- **TTL-based**: Automatic expiration for temporary data
- **Event-driven**: Invalidate cache on data updates
- **Pattern-based**: Bulk invalidation for user data

### Data Retention Policies

**URL Data**:
- Active URLs: Keep indefinitely
- Expired URLs: Archive after 1 year
- Deleted URLs: Soft delete, purge after 30 days

**Analytics Data**:
- Raw clicks: Keep for 1 year
- Aggregated data: Keep for 5 years
- User behavior: Anonymize after 2 years

### CI/CD Pipeline

```
┌─────────────┐    ┌─────────────┐    ┌─────────────┐    ┌─────────────┐
│   Code      │───▶│   Build     │───▶│   Test      │───▶│   Deploy    │
│   Commit    │    │   (Docker)  │    │   (Unit,    │    │   (Staging, │
└─────────────┘    └─────────────┘    │   Integration)│    │   Production)│
                                      └─────────────┘    └─────────────┘
```

**Pipeline Stages**:
1. **Code Quality**: Linting, formatting, security scanning
2. **Build**: Docker image creation, dependency management
3. **Test**: Unit tests, integration tests, performance tests
4. **Deploy**: Blue-green deployment, rollback capability

### Disaster Recovery

**Backup Strategy**:
- **Database**: Daily full backup + hourly incremental
- **Configuration**: Version-controlled infrastructure as code
- **Application**: Container images in multiple registries

**Recovery Procedures**:
- **RTO (Recovery Time Objective)**: 4 hours
- **RPO (Recovery Point Objective)**: 1 hour
- **Failover**: Automatic failover to secondary region
- **Rollback**: Quick rollback to previous stable version

**Monitoring & Alerting**:
- **Health Checks**: Service availability monitoring
- **Performance Metrics**: Response time, throughput, error rates
- **Business Metrics**: URL creation rate, click-through rates
- **Infrastructure**: CPU, memory, disk, network utilization

---

## Summary

This URL shortener system design provides a comprehensive solution that can scale from a simple single-server deployment to a globally distributed system serving millions of users. The key design principles focus on:

1. **Reliability**: Strong consistency for critical operations, fault tolerance
2. **Scalability**: Horizontal scaling, caching, and efficient data distribution
3. **Performance**: Optimized for low latency and high throughput
4. **Security**: Authentication, rate limiting, and data validation
5. **Cost-effectiveness**: Efficient resource utilization and intelligent caching

The system demonstrates how architectural decisions evolve based on scale requirements, with clear trade-offs and justifications for each major choice. The modular design allows for incremental scaling and technology adoption as the system grows.
