
# Microservices & System Design - In-Depth Interview Answers

---

## MICROSERVICES QUESTIONS

### 1. What is an API Gateway, and why do we use it?

**Problem Context:**
In a microservices architecture, we have multiple services (e.g., User Service, Order Service, Product Service, Payment Service). Each service runs on different ports and IPs. If a frontend application (React/SPA) directly called each service, it would face several challenges.

**In-Depth Answer:**
"An API Gateway is a single entry point for all client requests in a microservices architecture. It sits between the client and the backend services, acting as a reverse proxy that routes requests to the appropriate services."

**Key Functions:**
- **Routing:** Maps client requests to the correct microservice based on the URL path (e.g., `/api/users/*` → User Service, `/api/orders/*` → Order Service)
- **Authentication & Authorization:** Validates JWT tokens/OAuth2 credentials at the edge, ensuring only authenticated requests pass through. This prevents each microservice from implementing auth logic separately
- **Rate Limiting:** Protects downstream services from abuse by limiting requests per client (using token bucket or sliding window algorithms)
- **Load Balancing:** Distributes requests across multiple instances of each service
- **Logging & Monitoring:** Centralizes request logs, metrics, and tracing
- **Response Aggregation:** Combines responses from multiple services into a single response (reduces client round trips)

**Implementation Example:**
```java
// Spring Cloud Gateway Configuration
@Configuration
public class GatewayConfig {
    @Bean
    public RouteLocator customRouteLocator(RouteLocatorBuilder builder) {
        return builder.routes()
            .route("user_service", r -> r
                .path("/api/users/**")
                .filters(f -> f
                    .requestRateLimiter(config -> config.setRateLimiter(redisRateLimiter()))
                    .addRequestHeader("X-Service-Name", "user-service"))
                .uri("lb://USER-SERVICE"))
            .route("order_service", r -> r
                .path("/api/orders/**")
                .uri("lb://ORDER-SERVICE"))
            .build();
    }
}
```

Benefits:

· Single Point of Entry: Reduces client complexity—the frontend only needs to know one URL
· Security Layer: Adds an extra security barrier before requests hit the business logic
· Cross-Cutting Concerns: Centralizes concerns like logging, metrics, and authentication

---

2. How do microservices communicate with each other? (Synchronous vs. Asynchronous)

In-Depth Answer:
"Microservices need to communicate with each other to fulfill business transactions. There are two primary communication patterns: synchronous and asynchronous."

Synchronous Communication:

· Pattern: Request-Response (Blocking)
· Protocols: REST (HTTP/HTTPS), gRPC, GraphQL
· Use Cases:
  · When you need an immediate response (e.g., payment validation)
  · Simple CRUD operations
  · Low latency requirements

Example:

```java
// Using RestTemplate (Synchronous)
@Service
public class OrderService {
    @Autowired
    private RestTemplate restTemplate;
    
    public Order createOrder(OrderRequest request) {
        // Blocking call to User Service to validate user
        User user = restTemplate.getForObject(
            "http://USER-SERVICE/api/users/" + request.getUserId(), 
            User.class
        );
        // Business logic continues only after getting response
        return orderRepository.save(order);
    }
}
```

Asynchronous Communication:

· Pattern: Event-Driven (Fire and Forget)
· Protocols: Kafka, RabbitMQ, AWS SQS/SNS
· Use Cases:
  · Email/SMS notifications
  · Order status updates
  · Logging and analytics
  · When immediate response is not required

Example:

```java
// Using Kafka (Asynchronous)
@Service
public class PaymentService {
    @Autowired
    private KafkaTemplate<String, PaymentEvent> kafkaTemplate;
    
    public void processPayment(Payment payment) {
        paymentRepository.save(payment);
        // Fire event and continue - doesn't wait for consumers
        PaymentEvent event = new PaymentEvent(payment.getId(), payment.getOrderId(), "PAID");
        kafkaTemplate.send("payment-events", event);
        // Method returns immediately
    }
}

// Separate consumer updates order status
@Component
public class PaymentEventListener {
    @EventListener
    public void handlePaymentEvent(PaymentEvent event) {
        // Updates order status in Order Service
        orderService.updateOrderStatus(event.getOrderId(), "PAID");
    }
}
```

Hybrid Approach (Consumer-Driven Contract):

· Use synchronous for atomic operations (e.g., inventory check before order placement)
· Use asynchronous for eventual consistency flows (e.g., sending confirmation emails)

---

3. How do you handle distributed transactions? (The Saga Pattern)

Problem Context:
In a monolithic application, we use a single database and ACID transactions. In microservices, each service has its own database, making distributed transactions impossible with standard 2PC (Two-Phase Commit) due to availability and performance concerns.

In-Depth Answer:
"Distributed transactions across multiple microservices require a Saga pattern—a sequence of local transactions. Each local transaction updates its own database and publishes an event or calls the next service. If a step fails, compensating transactions are executed to rollback the changes."

Two Types of Sagas:

1. Choreography-Based Saga:

· Each service publishes events when it completes its work
· Other services listen and react
· No central coordinator

Example:

```
Order Created → Payment Service processes payment → Inventory Service reserves stock → Shipping Service schedules delivery
```

Implementation:

```java
// Order Service publishes event
@Service
public class OrderService {
    @Autowired
    private KafkaTemplate<String, OrderEvent> kafkaTemplate;
    
    @Transactional
    public Order createOrder(Order order) {
        order.setStatus("PENDING");
        orderRepository.save(order);
        // Fire event
        OrderEvent event = new OrderEvent(order.getId(), "ORDER_CREATED");
        kafkaTemplate.send("order-events", event);
    }
}

// Payment Service listens and processes
@Component
public class PaymentEventListener {
    @EventListener
    public void handleOrderCreated(OrderEvent event) {
        try {
            paymentService.processPayment(event.getOrderId());
            // Publish payment success event
        } catch (Exception e) {
            // Publish payment failed event → triggers compensation
        }
    }
}

// Compensation (Rollback)
@Component
public class CompensationHandler {
    @EventListener
    public void handlePaymentFailed(PaymentFailedEvent event) {
        orderService.updateOrderStatus(event.getOrderId(), "CANCELLED");
        inventoryService.releaseReservedStock(event.getOrderId());
    }
}
```

2. Orchestration-Based Saga:

· A central coordinator (orchestrator) tells services what to do
· Orchestrator tracks state and handles failures
· More centralized control, easier to manage

Example:

```java
@Component
public class OrderSagaOrchestrator {
    @Autowired
    private OrderService orderService;
    @Autowired
    private PaymentServiceClient paymentClient;
    @Autowired
    private InventoryServiceClient inventoryClient;
    
    public void executeSaga(Order order) {
        try {
            // Step 1: Create order
            orderService.createOrder(order);
            
            // Step 2: Process payment
            paymentClient.processPayment(order.getPaymentDetails());
            
            // Step 3: Reserve inventory
            inventoryClient.reserveStock(order.getItems());
            
            // Step 4: Complete order
            orderService.completeOrder(order.getId());
            
        } catch (PaymentFailedException e) {
            // Compensate: Cancel order
            orderService.cancelOrder(order.getId());
        } catch (InventoryReservationFailedException e) {
            // Compensate: Refund payment
            paymentClient.refundPayment(order.getPaymentDetails().getTransactionId());
            orderService.cancelOrder(order.getId());
        }
    }
}
```

Challenges:

· Idempotency: Ensure operations can be safely retried
· Monitoring: Track saga state to identify stuck transactions
· Eventual Consistency: Users may see intermediate states

---

4. What is the Circuit Breaker pattern, and why is it important?

Problem Context:
In a microservices ecosystem, if one service fails (e.g., slow response, timeout, throws exceptions), calls to it will block threads, exhaust connection pools, and cause cascading failures across the system.

In-Depth Answer:
"The Circuit Breaker pattern prevents a failure in one service from cascading to the entire system. It monitors calls to a service and if failures exceed a threshold, it 'opens the circuit,' immediately returning fallback responses instead of making the actual call."

Three States of Circuit Breaker:

```
     ┌─────────────┐
     │   CLOSED    │ (Normal: calls go through)
     └──────┬──────┘
            │ Failure threshold exceeded
            ▼
     ┌─────────────┐
     │   OPEN      │ (Failing fast: no calls)
     └──────┬──────┘
            │ Timeout elapsed
            ▼
     ┌─────────────┐
     │  HALF-OPEN  │ (Test: allow limited calls)
     └─────────────┘
```

Implementation with Resilience4j:

```java
@RestController
public class OrderController {
    
    @Autowired
    private ProductServiceClient productClient;
    
    @GetMapping("/orders/{id}/product-details")
    // Circuit Breaker with fallback
    @CircuitBreaker(name = "productService", fallbackMethod = "getProductDetailsFallback")
    public ProductDetails getProductDetails(@PathVariable Long id) {
        return productClient.getProductDetails(id);
    }
    
    // Fallback method
    public ProductDetails getProductDetailsFallback(Long id, Throwable ex) {
        // Returns cached or default response
        return new ProductDetails(id, "Unavailable (Service Down)");
    }
}

// Configuration
@Configuration
public class Resilience4jConfig {
    @Bean
    public CircuitBreakerConfig circuitBreakerConfig() {
        return CircuitBreakerConfig.custom()
            .failureRateThreshold(50)           // Open if 50% failures
            .slowCallRateThreshold(50)           // Open if 50% slow calls
            .slowCallDurationThreshold(Duration.ofSeconds(2))
            .permittedNumberOfCallsInHalfOpenState(5)
            .minimumNumberOfCalls(10)
            .waitDurationInOpenState(Duration.ofSeconds(10))
            .build();
    }
}
```

Benefits:

· Fault Tolerance: Gracefully handles service failures
· Resilience: Prevents cascading failures (chain reaction)
· Resource Protection: Saves connection pools and threads
· User Experience: Provides graceful degradation (fallback UI/messages)

---

5. How do you maintain data consistency across microservices?

In-Depth Answer:
"We maintain data consistency across microservices through Eventual Consistency rather than strong consistency. This means the system becomes consistent over time through async event handling, rather than guaranteeing immediate consistency."

Key Strategies:

1. Event-Driven Architecture:

```java
// Service publishes event
@Service
public class OrderService {
    public Order createOrder(Order order) {
        Order saved = orderRepository.save(order);
        // Eventual consistency: other services will be updated
        eventPublisher.publish(new OrderCreatedEvent(saved));
        return saved;
    }
}

// Other services handle events asynchronously
@Component
public class CustomerServiceEventHandler {
    @EventListener
    public void onOrderCreated(OrderCreatedEvent event) {
        // Eventually updates customer's order history
        customerRepository.updateOrderHistory(event.getCustomerId(), event.getOrderId());
    }
}
```

2. Event Sourcing (ES):

· Store each state change as an event
· Rebuild current state by replaying events

3. CQRS (Command Query Responsibility Segregation):

· Separate write model and read model
· Allow eventual consistency between them

4. Outbox Pattern:

```java
@Table
@Entity
public class OutboxMessage {
    @Id
    private UUID id;
    private String aggregateType;
    private String aggregateId;
    private String eventType;
    private String payload;
    private LocalDateTime createdAt;
    private String status; // PENDING, SENT
    private int retryCount;
}

// Transactionally save business data AND outbox record
@Transactional
public void createOrderAndPublish(Order order) {
    orderRepository.save(order);
    outboxRepository.save(new OutboxMessage(order));
    // Message will be sent asynchronously via polling
}
```

5. Compensation Events:

· When a transaction fails, publish compensation events to rollback changes

Best Practices:

· Design for eventual consistency by default
· Use idempotent consumers
· Implement idempotency keys in APIs
· Monitor event processing lag

---

6. What is Service Discovery (e.g., Netflix Eureka) and why is it needed?

Problem Context:
In cloud environments, microservices instances are dynamic—they scale up/down, crash, restart, change IP addresses. Hardcoding IPs is impossible.

In-Depth Answer:
"Service Discovery provides a dynamic registry where services register their network locations (IPs and ports). Other services can query the registry to find the current locations of the services they need to communicate with."

Two Components:

· Service Registry: Server that maintains the list of available service instances (e.g., Eureka, Consul, Zookeeper)
· Service Discovery Client: Library that services use to register and discover

Spring Cloud Netflix Eureka Example:

```java
// Eureka Server (Service Registry)
@SpringBootApplication
@EnableEurekaServer
public class ServiceRegistryApplication {
    public static void main(String[] args) {
        SpringApplication.run(ServiceRegistryApplication.class, args);
    }
}
```

```yaml
# application.yml for Eureka Server
server:
  port: 8761

eureka:
  client:
    register-with-eureka: false
    fetch-registry: false
  server:
    enable-self-preservation: false
```

```java
// Service Registering with Eureka
@SpringBootApplication
@EnableEurekaClient
public class UserServiceApplication {
    public static void main(String[] args) {
        SpringApplication.run(UserServiceApplication.class, args);
    }
}
```

```yaml
# application.yml for Service
spring:
  application:
    name: user-service
  eureka:
    client:
      service-url:
        defaultZone: http://localhost:8761/eureka/
    instance:
      lease-renewal-interval-in-seconds: 10
      lease-expiration-duration-in-seconds: 30
```

```java
// Discovery Client - Consuming Service
@Service
public class OrderService {
    @Autowired
    private DiscoveryClient discoveryClient;
    @Autowired
    private RestTemplate restTemplate;
    
    public User getUser(Long userId) {
        // Lookup service location dynamically
        List<ServiceInstance> instances = discoveryClient.getInstances("user-service");
        if (!instances.isEmpty()) {
            ServiceInstance instance = instances.get(0);
            String url = "http://" + instance.getHost() + ":" + 
                         instance.getPort() + "/api/users/" + userId;
            return restTemplate.getForObject(url, User.class);
        }
        throw new ServiceUnavailableException("No user service instances found");
    }
}

// Or using LoadBalancer (Spring Cloud)
@Bean
@LoadBalanced
public RestTemplate restTemplate() {
    return new RestTemplate();
}

// Now can use service name directly
User user = restTemplate.getForObject(
    "http://user-service/api/users/" + userId, 
    User.class
);
```

Why Needed:

· Dynamic Environments: Services can scale up/down without breaking integrations
· Self-Healing: Unhealthy instances are removed from the registry
· Decoupling: Services don't need to know each other's exact locations
· Load Balancing: Client-side load balancing across instances

---

7. How do you secure microservices? (JWT & OAuth2)

In-Depth Answer:
"Securing microservices involves multiple layers. We use API Gateway as the security edge, OAuth2 for authorization and delegation, and JWT for stateless authentication. Each microservice validates the JWT token to enforce fine-grained permissions."

Complete Security Flow:

```
┌─────────────┐   1. Login (Username/Password)    ┌─────────────────┐
│   React     │─────────────────────────────────────→  API Gateway    │
│   Frontend  │                                     │                 │
│             │    2. Redirect to Auth Server       │                 │
│             │─────────────────────────────────────→                 │
└─────────────┘    3. Auth Server validates         └─────────────────┘
        │                 ┌─────────────────────────────────────────────┐
        │                 │          Auth Server / Identity Provider   │
        │                 │          (OAuth2 / Keycloak / Custom)      │
        │                 │                                             │
        │                 │   - Validates credentials                  │
        │                 │   - Generates JWT (Access + Refresh Token) │
        │                 └─────────────────────────────────────────────┘
        │                           │
        │    4. Returns JWT Token   │
        │←──────────────────────────┘
        │
        │    5. Future requests: Bearer <JWT>
        │─────────────────────────────────────────────────────────────────▶ API Gateway
        │                                                                  │
        │                                                                  │ 6. Validate JWT
        │                                                                  │    (Signature, Expiry, 
        │                                                                  │     Scope/Permissions)
        │                                                                  ▼
        │                                                    ┌─────────────────────────┐
        │                                                    │   Microservice          │
        │                                                    │   (User/Order Service)  │
        │                                                    │                         │
        │                                                    │ 7. Validate JWT again   │
        │                                                    │    Extract User Context │
        │                                                    └─────────────────────────┘
```

Implementation Examples:

1. JWT Generation (Auth Server):

```java
@Component
public class JwtTokenProvider {
    @Value("${jwt.secret}")
    private String jwtSecret;
    @Value("${jwt.expiration}")
    private int jwtExpiration;
    @Value("${jwt.refresh-expiration}")
    private int refreshExpiration;
    
    public String generateAccessToken(String username, List<String> roles) {
        Date now = new Date();
        Date expiryDate = new Date(now.getTime() + jwtExpiration);
        
        return Jwts.builder()
            .setSubject(username)
            .claim("roles", roles)
            .setIssuedAt(now)
            .setExpiration(expiryDate)
            .signWith(SignatureAlgorithm.HS512, jwtSecret)
            .compact();
    }
    
    public String generateRefreshToken(String username) {
        Date now = new Date();
        Date expiryDate = new Date(now.getTime() + refreshExpiration);
        
        return Jwts.builder()
            .setSubject(username)
            .setIssuedAt(now)
            .setExpiration(expiryDate)
            .signWith(SignatureAlgorithm.HS512, jwtSecret)
            .compact();
    }
    
    public boolean validateToken(String token) {
        try {
            Jwts.parser().setSigningKey(jwtSecret).parseClaimsJws(token);
            return true;
        } catch (SignatureException | MalformedJwtException | ExpiredJwtException e) {
            return false;
        }
    }
}
```

2. API Gateway JWT Validation Filter:

```java
@Component
public class JwtAuthenticationGatewayFilter implements GlobalFilter {
    @Autowired
    private JwtTokenProvider tokenProvider;
    
    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        String token = extractJwt(exchange.getRequest());
        
        if (token != null && tokenProvider.validateToken(token)) {
            String username = tokenProvider.getUsernameFromToken(token);
            List<String> roles = tokenProvider.getRolesFromToken(token);
            
            // Add user context to headers for downstream services
            ServerHttpRequest modifiedRequest = exchange.getRequest()
                .mutate()
                .header("X-User-Id", username)
                .header("X-User-Roles", String.join(",", roles))
                .build();
            
            return chain.filter(exchange.mutate().request(modifiedRequest).build());
        }
        
        // If token missing/invalid, return 401
        if (exchange.getRequest().getURI().getPath().startsWith("/api/secure")) {
            exchange.getResponse().setStatusCode(HttpStatus.UNAUTHORIZED);
            return exchange.getResponse().setComplete();
        }
        
        return chain.filter(exchange);
    }
    
    private String extractJwt(ServerHttpRequest request) {
        String authHeader = request.getHeaders().getFirst("Authorization");
        if (authHeader != null && authHeader.startsWith("Bearer ")) {
            return authHeader.substring(7);
        }
        return null;
    }
}
```

3. Microservice JWT Validation (Spring Security):

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {
    
    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .csrf().disable()
            .sessionManagement(session -> session.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
            .authorizeHttpRequests(authz -> authz
                .requestMatchers("/api/public/**").permitAll()
                .requestMatchers("/api/admin/**").hasRole("ADMIN")
                .requestMatchers("/api/orders/**").hasAnyRole("USER", "ADMIN")
                .anyRequest().authenticated()
            )
            .addFilterBefore(jwtAuthenticationFilter(), UsernamePasswordAuthenticationFilter.class);
        
        return http.build();
    }
    
    @Bean
    public JwtAuthenticationFilter jwtAuthenticationFilter() {
        return new JwtAuthenticationFilter();
    }
}

public class JwtAuthenticationFilter extends OncePerRequestFilter {
    @Autowired
    private JwtTokenProvider tokenProvider;
    
    @Override
    protected void doFilterInternal(HttpServletRequest request, 
                                    HttpServletResponse response,
                                    FilterChain filterChain) throws ServletException, IOException {
        String jwt = getJwtFromRequest(request);
        
        if (jwt != null && tokenProvider.validateToken(jwt)) {
            String username = tokenProvider.getUsernameFromToken(jwt);
            List<String> roles = tokenProvider.getRolesFromToken(jwt);
            
            List<GrantedAuthority> authorities = roles.stream()
                .map(role -> new SimpleGrantedAuthority("ROLE_" + role))
                .collect(Collectors.toList());
            
            Authentication auth = new UsernamePasswordAuthenticationToken(
                username, null, authorities
            );
            
            SecurityContextHolder.getContext().setAuthentication(auth);
        }
        
        filterChain.doFilter(request, response);
    }
}
```

OAuth2 Integration:

· Use Spring Boot OAuth2 Client for third-party login (Google, GitHub)
· Use Keycloak as dedicated OAuth2 Server
· Implement Refresh Token Rotation for better security

---

8. How do you handle centralized logging and tracing in microservices?

Problem Context:
With dozens of services, a single user request may be processed by multiple services. Debugging is nearly impossible without correlation and centralized logging.

In-Depth Answer:
"We implement distributed tracing by passing a unique Correlation ID/Trace ID across all service calls. All logs are centralized in a log aggregation system (ELK Stack or Loki) with the Trace ID as a mandatory field. This allows us to trace a user's request across all services."

Complete Architecture:

```
┌─────────────┐     Request with Trace ID Header     ┌──────────────────┐
│   React     │─────────────────────────────────────→│   API Gateway    │
│   Frontend  │     X-Trace-Id: abc-123              │   Generates Trace│
│             │                                     │   ID if missing  │
└─────────────┘                                     └──────────────────┘
                                                             │
                                    ┌────────────────────────┼────────────────────────┐
                                    │                        │                        │
                                    ▼                        ▼                        ▼
                          ┌─────────────────┐      ┌─────────────────┐      ┌─────────────────┐
                          │  User Service   │      │  Order Service  │      │  Payment Service│
                          │  Log: Trace ID  │      │  Log: Trace ID  │      │  Log: Trace ID  │
                          │  Span ID: u-1   │      │  Span ID: o-1   │      │  Span ID: p-1   │
                          └─────────────────┘      └─────────────────┘      └─────────────────┘
                                    │                        │                        │
                                    └────────────────────────┼────────────────────────┘
                                                             │
                                                             ▼
                                                  ┌─────────────────────────┐
                                                  │   Log Aggregator        │
                                                  │   (ELK Stack / Loki)    │
                                                  │                         │
                                                  │  All logs aggregated    │
                                                  │  Searchable by Trace ID │
                                                  └─────────────────────────┘
```

Implementation:

1. Adding Correlation ID to Logs:

```java
// Custom MDC Filter
@Component
public class CorrelationIdFilter implements Filter {
    private static final String CORRELATION_ID_HEADER = "X-Correlation-Id";
    private static final String CORRELATION_ID_KEY = "correlationId";
    
    @Override
    public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain) 
            throws IOException, ServletException {
        HttpServletRequest httpRequest = (HttpServletRequest) request;
        String correlationId = httpRequest.getHeader(CORRELATION_ID_HEADER);
        
        if (correlationId == null || correlationId.isEmpty()) {
            correlationId = UUID.randomUUID().toString();
        }
        
        // Set in MDC for logging
        MDC.put(CORRELATION_ID_KEY, correlationId);
        
        try {
            chain.doFilter(request, response);
        } finally {
            MDC.clear();
        }
    }
}
```

2. Logback Configuration (logback-spring.xml):

```xml
<configuration>
    <appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
        <encoder>
            <pattern>
                %d{yyyy-MM-dd HH:mm:ss.SSS} [%thread] %-5level %logger{36} - [%X{correlationId}] - %msg%n
            </pattern>
        </encoder>
    </appender>
    
    <appender name="JSON" class="ch.qos.logback.core.ConsoleAppender">
        <encoder class="ch.qos.logback.classic.encoder.JsonEncoder"/>
    </appender>
    
    <root level="INFO">
        <appender-ref ref="JSON"/>
    </root>
</configuration>
```

3. Distributed Tracing with Sleuth & Zipkin:

```xml
<!-- pom.xml -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-sleuth</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-zipkin</artifactId>
</dependency>
```

```yaml
# application.yml
spring:
  sleuth:
    sampler:
      probability: 1.0  # Sample 100% of traces
  zipkin:
    base-url: http://zipkin-server:9411
    sender:
      type: web
```

4. Propagating Trace ID Across Services:

```java
// RestTemplate interceptor
@Component
public class TracingRestTemplateInterceptor implements ClientHttpRequestInterceptor {
    @Override
    public ClientHttpResponse intercept(HttpRequest request, byte[] body, 
                                        ClientHttpRequestExecution execution) throws IOException {
        String traceId = MDC.get("traceId");
        if (traceId != null) {
            request.getHeaders().add("X-Trace-Id", traceId);
        }
        return execution.execute(request, body);
    }
}

// Feign Client interceptor
@Component
public class FeignTracingInterceptor implements RequestInterceptor {
    @Override
    public void apply(RequestTemplate template) {
        String traceId = MDC.get("traceId");
        if (traceId != null) {
            template.header("X-Trace-Id", traceId);
        }
    }
}
```

5. Centralized Logging with ELK Stack:

· Logstash: Ingests logs, parses them (JSON), and sends to Elasticsearch
· Elasticsearch: Stores and indexes logs for fast searching
· Kibana: Visualization and dashboarding

Monitoring with Grafana/Loki:

· Loki for log aggregation (more efficient than ELK)
· Grafana for querying traces and logs together

Benefits:

· Single Source of Truth: All logs in one place
· Debugging: Search by Trace ID to see complete request flow
· Performance Monitoring: Analyze response times by service
· Alerting: Set alerts on error patterns

---

9. What is CQRS (Command Query Responsibility Segregation)?

Problem Context:
In traditional CRUD systems, the same model handles both reads and writes. This can cause issues when the read requirements differ from the write requirements (e.g., complex reporting vs. simple data entry).

In-Depth Answer:
"CQRS is a pattern that separates the data mutation operations (Commands/Writes) from the data retrieval operations (Queries/Reads) into different models. This allows each side to be optimized independently."

Full Architecture:

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                               APPLICATION                                      │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                 │
│    ┌──────────────────────────────┐          ┌──────────────────────────────┐   │
│    │        Command Side          │          │        Query Side            │   │
│    │        (Writes)              │          │        (Reads)              │   │
│    ├──────────────────────────────┤          ├──────────────────────────────┤   │
│    │  ┌──────────────────────┐    │          │  ┌──────────────────────┐    │   │
│    │  │  Commands            │    │          │  │  Queries             │    │   │
│    │  │  - CreateOrder       │    │          │  │  - GetOrderById      │    │   │
│    │  │  - UpdateOrder       │    │          │  │  - GetOrdersByDate   │    │   │
│    │  │  - CancelOrder       │    │          │  │  - GetUserOrders     │    │   │
│    │  └──────────┬───────────┘    │          │  └──────────┬───────────┘    │   │
│    │             │                │          │             │                │   │
│    │             ▼                │          │             ▼                │   │
│    │  ┌──────────────────────┐    │          │  ┌──────────────────────┐    │   │
│    │  │  Command Model       │    │          │  │  Query Model         │    │   │
│    │  │  (Write Model)       │    │          │  │  (Read Model)        │    │   │
│    │  └──────────┬───────────┘    │          │  └──────────┬───────────┘    │   │
│    │             │                │          │             │                │   │
│    └─────────────┼────────────────┘          └─────────────┼────────────────┘   │
│                  │                                        │                     │
│                  ▼                                        ▼                     │
│    ┌──────────────────────────────────────────────────────────────────────┐     │
│    │                                                                      │     │
│    │    ┌──────────────┐    Event Store    ┌──────────────┐             │     │
│    │    │ Write DB     │───────────────────│ Read DB      │             │     │
│    │    │ (Transactional)│                   │ (Read-Optimized)│          │     │
│    │    │ - Normalized  │                   │ - Denormalized│             │     │
│    │    │ - ACID        │                   │ - Cached     │             │     │
│    │    │ - Consistent  │                   │ - Fast       │             │     │
│    │    └──────────────┘                   └──────────────┘             │     │
│    │         ▲                                   │                     │     │
│    └─────────┼───────────────────────────────────┼─────────────────────┘     │
│              │                                   │                            │
│          Events                              Events                          │
│          Published                           Consumed                        │
│              │                                   │                            │
│              └───────────────────────────────────┘                            │
│                                                                               │
└───────────────────────────────────────────────────────────────────────────────┘
```

Implementation Example:

```java
// ===== Command Side (Writes) =====
@RestController
@RequestMapping("/api/orders")
public class OrderCommandController {
    @Autowired
    private OrderCommandService commandService;
    
    @PostMapping
    public ResponseEntity<Order> createOrder(@RequestBody CreateOrderCommand command) {
        Order created = commandService.handle(command);
        return ResponseEntity.created(URI.create("/api/orders/" + created.getId())).body(created);
    }
}

@Service
public class OrderCommandService {
    @Autowired
    private OrderWriteRepository orderWriteRepo;
    @Autowired
    private EventPublisher eventPublisher;
    
    @Transactional
    public Order handle(CreateOrderCommand command) {
        // 1. Validate business rules
        validateOrder(command);
        
        // 2. Create order entity (Write model)
        Order order = new Order();
        order.setCustomerId(command.getCustomerId());
        order.setTotalAmount(command.getTotalAmount());
        order.setStatus("CREATED");
        order.setCreatedAt(LocalDateTime.now());
        
        // 3. Save to write database
        Order saved = orderWriteRepo.save(order);
        
        // 4. Publish domain event
        eventPublisher.publish(new OrderCreatedEvent(saved));
        
        return saved;
    }
}

// ===== Query Side (Reads) =====
@RestController
@RequestMapping("/api/orders")
public class OrderQueryController {
    @Autowired
    private OrderReadService readService;
    
    @GetMapping("/{id}")
    public OrderDTO getOrder(@PathVariable Long id) {
        return readService.getOrder(id);
    }
    
    @GetMapping
    public List<OrderDTO> getOrders(@RequestParam LocalDate start, @RequestParam LocalDate end) {
        return readService.getOrdersBetweenDates(start, end);
    }
}

@Service
public class OrderReadService {
    @Autowired
    private OrderReadRepository readRepo;
    
    public OrderDTO getOrder(Long id) {
        // Query optimized for reads, direct DB fetch with joins
        return readRepo.findOrderWithDetails(id);
    }
    
    public List<OrderDTO> getOrdersBetweenDates(LocalDate start, LocalDate end) {
        return readRepo.findOrdersByDateRange(start, end);
    }
}

// ===== Read Model (Denormalized) =====
public class OrderReadModel {
    @Id
    private Long id;
    private Long customerId;
    private String customerName;      // Denormalized customer data
    private String customerEmail;     // Denormalized customer data
    private Double totalAmount;
    private String status;
    private LocalDateTime createdAt;
    private List<OrderItemReadModel> items;  // Denormalized items
}

// ===== Event Consumer (Syncs Read Model) =====
@Component
public class OrderEventConsumer {
    @Autowired
    private OrderReadRepository readRepo;
    
    @EventListener
    public void handleOrderCreated(OrderCreatedEvent event) {
        // Update read model with denormalized data
        OrderReadModel readModel = new OrderReadModel();
        readModel.setId(event.getOrderId());
        readModel.setCustomerId(event.getCustomerId());
        readModel.setCustomerName(event.getCustomerName());  // Fetch from User Service
        readModel.setTotalAmount(event.getTotalAmount());
        readModel.setStatus("CREATED");
        readModel.setCreatedAt(event.getCreatedAt());
        
        readRepo.save(readModel);
    }
}
```

When to Use CQRS:

· Complex domain logic where reads and writes have different performance requirements
· Read-heavy applications requiring optimized queries
· Multiple data sources for reads (different projections)
· Event Sourcing applications
· Applications where data consistency can be eventual

---

10. How do you handle database migration or updates in a microservices architecture?

Problem Context:
In microservices, each service has its own database. Database schema changes must be handled carefully to avoid downtime and ensure all instances are updated.

In-Depth Answer:
"We use automated schema migration tools like Flyway or Liquibase bundled within each microservice. The migration runs automatically on startup, ensuring the database schema matches the version of the service code. We follow version-controlled migration scripts to handle database updates safely."

Flyway Implementation:

```xml
<!-- pom.xml -->
<dependency>
    <groupId>org.flywaydb</groupId>
    <artifactId>flyway-core</artifactId>
</dependency>
```

Migration Scripts:

```sql
-- V1__initial_schema.sql
CREATE TABLE orders (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    customer_id BIGINT NOT NULL,
    total_amount DECIMAL(10,2) NOT NULL,
    status VARCHAR(20) NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- V2__add_indexes.sql
CREATE INDEX idx_orders_customer_id ON orders(customer_id);
CREATE INDEX idx_orders_created_at ON orders(created_at);

-- V3__add_version_column.sql
ALTER TABLE orders ADD COLUMN version INT DEFAULT 0;
```

Configuration:

```yaml
# application.yml
spring:
  flyway:
    enabled: true
    baseline-on-migrate: true
    baseline-version: 0
    locations: classpath:db/migration
    validate-on-migrate: true
```

Database Migration Strategy:

1. Blue-Green Deployment:

· Deploy new version (green) alongside current (blue)
· Green runs migrations to its own database
· Switch traffic to green after validation

2. Rolling Updates with Backward Compatibility:

```sql
-- Step 1: Add new column with default
ALTER TABLE orders ADD COLUMN new_column VARCHAR(255) DEFAULT '';

-- Step 2: Deploy code that reads both old and new columns
-- Step 3: Backfill data
UPDATE orders SET new_column = old_column;
-- Step 4: Deploy code that only reads new column
-- Step 5: Remove old column
ALTER TABLE orders DROP COLUMN old_column;
```

3. Database Migration with Ease:

```java
@Configuration
public class FlywayConfig {
    @Bean
    public FlywayMigrationStrategy flywayMigrationStrategy() {
        return flyway -> {
            flyway.setBaselineOnMigrate(true);
            flyway.setLocations("classpath:db/migration");
            flyway.setTable("schema_history");
            flyway.migrate();
        };
    }
}
```

Best Practices:

· Version Control: Migration scripts in Git, versioned
· Automated Rollbacks: Have rollback scripts ready
· Test Migrations: Run migrations on staging first
· Database Backups: Always backup before production migration
· Idempotent Scripts: Use IF NOT EXISTS to make scripts safe

---

SYSTEM DESIGN QUESTIONS

11. How would you design a scalable notification system?

Problem Statement:
Design a system that sends notifications (Email, SMS, Push) to millions of users reliably.

In-Depth Answer:
"We'll design a system that handles high-volume asynchronous notification delivery with proper queueing, retries, and monitoring."

Complete System Design:

```
┌──────────────────────────────────────────────────────────────────────────────────────┐
│                          Notification Service Architecture                          │
├──────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                      │
│  ┌─────────────────┐                                                               │
│  │   Service A     │                                                               │
│  │  (Order, Payment)│                                                               │
│  └────────┬────────┘                                                               │
│           │                                                                          │
│           ▼                                                                          │
│  ┌──────────────────────────────────────────────────────────────────────┐           │
│  │                    Notification API Gateway                        │           │
│  │                    (Authentication, Validation, Rate Limiting)     │           │
│  └──────────────┬──────────────────────────────────────────────────────┘           │
│                 │                                                                     │
│                 ▼                                                                     │
│  ┌──────────────────────────────────────────────────────────────────────┐           │
│  │                    Message Broker (Kafka / RabbitMQ)                 │           │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐              │           │
│  │  │ email-topic  │  │  sms-topic   │  │ push-topic   │              │           │
│  │  └──────────────┘  └──────────────┘  └──────────────┘              │           │
│  └──────────────┬──────────────────────────────────────────────────────┘           │
│                 │                                                                     │
│                 ▼                                                                     │
│  ┌──────────────────────────────────────────────────────────────────────┐           │
│  │                    Notification Workers                              │           │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐              │           │
│  │  │ Email Worker │  │  SMS Worker  │  │ Push Worker  │              │           │
│  │  └──────────────┘  └──────────────┘  └──────────────┘              │           │
│  └──────────────┬──────────────────────────────────────────────────────┘           │
│                 │                                                                     │
│                 ▼                                                                     │
│  ┌──────────────────────────────────────────────────────────────────────┐           │
│  │                    External Providers                                │           │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐              │           │
│  │  │ SendGrid     │  │  Twilio      │  │ Firebase    │              │           │
│  │  │ AWS SES      │  │  AWS SNS     │  │  APNS/FCM   │              │           │
│  │  └──────────────┘  └──────────────┘  └──────────────┘              │           │
│  └──────────────┬──────────────────────────────────────────────────────┘           │
│                 │                                                                     │
│                 ▼                                                                     │
│  ┌──────────────────────────────────────────────────────────────────────┐           │
│  │                    Storage & Monitoring                              │           │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐              │           │
│  │  │ Notification │  │  Metrics     │  │  Logging     │              │           │
│  │  │   DB         │  │  (Prometheus)│  │  (ELK Stack) │              │           │
│  │  └──────────────┘  └──────────────┘  └──────────────┘              │           │
│  └──────────────────────────────────────────────────────────────────────┘           │
│                                                                                      │
└──────────────────────────────────────────────────────────────────────────────────────┘
```

Implementation Details:

```java
// ===== 1. Notification Request DTO =====
@Data
public class NotificationRequest {
    private String userId;
    private String type;  // EMAIL, SMS, PUSH
    private String subject;
    private String content;
    private Map<String, String> metadata;
    private String templateId;
    private Map<String, Object> templateVariables;
}

// ===== 2. API Controller =====
@RestController
@RequestMapping("/api/notifications")
public class NotificationController {
    @Autowired
    private NotificationService notificationService;
    
    @PostMapping
    public ResponseEntity<NotificationResponse> sendNotification(
            @Valid @RequestBody NotificationRequest request) {
        String notificationId = notificationService.queueNotification(request);
        return ResponseEntity.accepted()
            .body(new NotificationResponse(notificationId, "Queued for delivery"));
    }
}

// ===== 3. Service Layer =====
@Service
public class NotificationService {
    @Autowired
    private KafkaTemplate<String, NotificationMessage> kafkaTemplate;
    @Autowired
    private NotificationRepository notificationRepository;
    
    public String queueNotification(NotificationRequest request) {
        // 1. Validate user exists
        // 2. Check rate limits
        // 3. Create notification record
        Notification notification = new Notification();
        notification.setId(UUID.randomUUID().toString());
        notification.setUserId(request.getUserId());
        notification.setType(request.getType());
        notification.setStatus("PENDING");
        notification.setCreatedAt(LocalDateTime.now());
        
        notificationRepository.save(notification);
        
        // 4. Send to appropriate topic
        NotificationMessage message = new NotificationMessage(
            notification.getId(),
            request.getUserId(),
            request.getContent(),
            request.getTemplateId(),
            request.getTemplateVariables()
        );
        
        String topic = getTopicForType(request.getType());
        kafkaTemplate.send(topic, message);
        
        return notification.getId();
    }
    
    private String getTopicForType(String type) {
        switch (type) {
            case "EMAIL": return "email-notifications";
            case "SMS": return "sms-notifications";
            case "PUSH": return "push-notifications";
            default: throw new IllegalArgumentException("Invalid type");
        }
    }
}

// ===== 4. Notification Worker =====
@Component
public class EmailNotificationWorker {
    @Autowired
    private SendGridClient sendGridClient;
    @Autowired
    private NotificationRepository notificationRepository;
    
    @KafkaListener(topics = "email-notifications", groupId = "email-workers")
    public void processEmailNotification(NotificationMessage message) {
        try {
            // 1. Render template
            String emailHtml = renderTemplate(message.getTemplateId(), 
                                             message.getTemplateVariables());
            
            // 2. Send via provider
            String providerMessageId = sendGridClient.sendEmail(
                message.getUserId(),
                message.getSubject(),
                emailHtml
            );
            
            // 3. Update status
            notificationRepository.updateStatus(
                message.getNotificationId(), 
                "SENT", 
                providerMessageId
            );
            
        } catch (Exception e) {
            // 4. Retry or DLQ
            log.error("Failed to send email: {}", message.getNotificationId(), e);
            notificationRepository.updateStatus(
                message.getNotificationId(), 
                "FAILED", 
                e.getMessage()
            );
            // Re-throw for Kafka retry
            throw new RuntimeException(e);
        }
    }
}

// ===== 5. Rate Limiting =====
@Component
public class RateLimitService {
    @Autowired
    private RedisTemplate<String, Integer> redisTemplate;
    
    public boolean checkRateLimit(String userId, String type, int limit, int windowSeconds) {
        String key = "rate:" + type + ":" + userId;
        Integer count = redisTemplate.opsForValue().get(key);
        
        if (count == null) {
            redisTemplate.opsForValue().set(key, 1, Duration.ofSeconds(windowSeconds));
            return true;
        }
        
        if (count >= limit) {
            return false;
        }
        
        redisTemplate.opsForValue().increment(key);
        return true;
    }
}
```

Key Components:

· Message Queue: Decouple API from processing (Kafka for persistence, high throughput)
· Workers: Process notifications asynchronously, retry on failure
· Templates: Store templates for consistent formatting
· DLQ (Dead Letter Queue): Handle undeliverable messages
· Monitoring: Track delivery rates, failures, and latencies

---

12. How do you scale a system handling sudden traffic spikes?

In-Depth Answer:
"To handle sudden traffic spikes, we use a combination of horizontal scaling, caching, and database scaling techniques."

Strategies:

1. Horizontal Scaling:

```yaml
# Kubernetes auto-scaling
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: order-service-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: order-service
  minReplicas: 3
  maxReplicas: 20
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Pods
    pods:
      metric:
        name: http_requests
      target:
        type: AverageValue
        averageValue: 1000
```

2. Caching Layer:

```java
// Redis caching
@Service
public class ProductService {
    @Autowired
    private RedisTemplate<String, Product> redisTemplate;
    @Autowired
    private ProductRepository productRepository;
    
    @Cacheable(value = "products", key = "#id")
    public Product getProduct(Long id) {
        return productRepository.findById(id)
            .orElseThrow(() -> new RuntimeException("Product not found"));
    }
}

// Multi-level caching
// - CDN for static assets
// - Redis for frequent reads
// - Database for source of truth
```

3. Read Replicas:

```yaml
spring:
  datasource:
    url: jdbc:mysql://master-db:3306/appdb
    read-only-url: jdbc:mysql://replica-db:3306/appdb
    read-write-url: jdbc:mysql://master-db:3306/appdb
```

```java
@Configuration
public class DatabaseRoutingConfig {
    @Bean
    public DataSource routingDataSource() {
        Map<Object, Object> targetDataSources = new HashMap<>();
        targetDataSources.put("master", masterDataSource());
        targetDataSources.put("replica", replicaDataSource());
        
        RoutingDataSource routingDataSource = new RoutingDataSource();
        routingDataSource.setDefaultTargetDataSource(masterDataSource());
        routingDataSource.setTargetDataSources(targetDataSources);
        return routingDataSource;
    }
}

// Read-Only transaction (goes to replica)
@Transactional(readOnly = true)
public List<Product> getProducts() {
    return productRepository.findAll();
}
```

4. Load Balancing:

· Round Robin, Least Connections, IP Hash
· Implement Circuit Breaker to fail fast

5. Asynchronous Processing:

· Move heavy processing to background jobs
· Use message queues to smooth traffic spikes

---

13. SQL vs. NoSQL: How do you choose the right database for a system?

In-Depth Answer:
"The choice between SQL and NoSQL depends on the specific requirements of the application, including data structure, consistency needs, and scalability requirements."

Decision Framework:

Factor Choose SQL Choose NoSQL
Data Structure Structured with predefined schema Unstructured, flexible schema
Relationships Complex relationships (JOINs) Few or no relationships
ACID Compliance Required (banking, payments) Not strictly required
Scalability Vertical scaling, complex sharding Horizontal scaling (built-in)
Query Complexity Complex queries with aggregations Simple key-value lookups
Transactions Multi-row transactions Single-record transactions
Use Cases ERP, CRM, Financial systems IoT, Social feeds, Caching

Example Scenarios:

1. Payment System (SQL):

```sql
CREATE TABLE payments (
    id BIGINT PRIMARY KEY,
    order_id BIGINT NOT NULL REFERENCES orders(id),
    amount DECIMAL(10,2) NOT NULL,
    status VARCHAR(20) NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- ACID required
BEGIN TRANSACTION;
UPDATE accounts SET balance = balance - 100 WHERE user_id = 1;
INSERT INTO payments (order_id, amount, status) VALUES (123, 100, 'COMPLETED');
UPDATE orders SET status = 'PAID' WHERE id = 123;
COMMIT;
```

2. User Activity Feed (NoSQL - MongoDB):

```json
// Denormalized feed items
{
  "_id": "user_123_feed",
  "userId": "123",
  "items": [
    {
      "type": "POST",
      "postId": "post_456",
      "author": {
        "id": "789",
        "name": "John Doe",
        "avatar": "url"
      },
      "content": "Hello world",
      "timestamp": "2024-01-01T12:00:00Z"
    }
  ],
  "page": 1,
  "nextCursor": "2024-01-01T12:00:00Z"
}
```

Hybrid Approach:

· SQL for transactional data
· NoSQL for caching and analytics
· Elasticsearch for search

---

14. What is a Load Balancer, and what are its common algorithms?

In-Depth Answer:
"A Load Balancer distributes incoming network traffic across multiple backend servers to ensure no single server becomes overwhelmed, improving application availability and responsiveness."

Common Algorithms:

1. Round Robin:

· Distributes requests sequentially
· Simple, but doesn't account for server load

2. Least Connections:

· Directs traffic to server with fewest active connections
· Good for varying request processing times

3. IP Hash:

· Hashes client IP to consistently route to the same server
· Useful for session stickiness

4. Weighted Algorithms:

· Assign weights to servers based on capacity
· Route more traffic to powerful servers

Implementation Example (Nginx):

```nginx
http {
    upstream backend {
        # Round Robin (default)
        server backend1.example.com weight=3;
        server backend2.example.com weight=2;
        server backend3.example.com;
        
        # Least Connections
        # least_conn;
        
        # IP Hash
        # ip_hash;
    }
    
    server {
        listen 80;
        location / {
            proxy_pass http://backend;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
        }
    }
}
```

Load Balancer Types:

· Layer 4 (Transport): Distributes based on IP and port (fast)
· Layer 7 (Application): Distributes based on content (URL, header) - more intelligent

---

15. How would you design a URL Shortener service (like Bitly)?

In-Depth Answer:
"Designing a URL shortener involves three main components: shortening logic, storage, and redirection. The key is to be able to handle high read load (redirects) efficiently."

Complete Architecture:

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                        URL Shortener System Design                                  │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                     │
│  ┌──────────────┐                                                                  │
│  │   Client     │                                                                  │
│  └──────┬───────┘                                                                  │
│         │                                                                           │
│         ▼                                                                           │
│  ┌──────────────────────────────────────────────────────────────────────┐           │
│  │                    API Gateway                                       │           │
│  │                    Rate Limiting, Authentication                     │           │
│  └──────────────┬──────────────────────────────────────────────────────┘           │
│                 │                                                                     │
│                 ▼                                                                     │
│  ┌──────────────────────────────────────────────────────────────────────┐           │
│  │                    URL Service                                       │           │
│  │  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐           │           │
│  │  │ Shorten API  │    │  Redirect    │    │  Analytics   │           │           │
│  │  └──────────────┘    └──────────────┘    └──────────────┘           │           │
│  └──────────────┬──────────────────────────────────────────────────────┘           │
│                 │                                                                     │
│                 ▼                                                                     │
│  ┌──────────────────────────────────────────────────────────────────────┐           │
│  │                    Database Layer                                    │           │
│  │  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐           │           │
│  │  │   MySQL      │    │    Redis     │    │    Kafka     │           │           │
│  │  │   (Metadata) │    │   (Cache)    │    │  (Analytics) │           │           │
│  │  └──────────────┘    └──────────────┘    └──────────────┘           │           │
│  └──────────────────────────────────────────────────────────────────────┘           │
│                                                                                     │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

1. Shortening Logic (Base62 Encoding):

```java
public class UrlShortener {
    private static final String BASE62 = "0123456789ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz";
    private static final int BASE = 62;
    
    public String encode(long number) {
        StringBuilder sb = new StringBuilder();
        while (number > 0) {
            sb.append(BASE62.charAt((int) (number % BASE)));
            number /= BASE;
        }
        return sb.reverse().toString();
    }
    
    public long decode(String shortCode) {
        long result = 0;
        for (char c : shortCode.toCharArray()) {
            result = result * BASE + BASE62.indexOf(c);
        }
        return result;
    }
}
```

2. Service Implementation:

```java
@Service
public class UrlService {
    @Autowired
    private UrlRepository urlRepository;
    @Autowired
    private RedisTemplate<String, String> redisTemplate;
    @Autowired
    private KafkaTemplate<String, String> kafkaTemplate;
    
    // Generate short URL
    public String shortenUrl(String longUrl) {
        // 1. Check if URL already exists
        Optional<UrlMapping> existing = urlRepository.findByLongUrl(longUrl);
        if (existing.isPresent()) {
            return existing.get().getShortCode();
        }
        
        // 2. Generate unique ID (Snowflake ID)
        long id = snowflakeIdGenerator.nextId();
        
        // 3. Encode to Base62
        String shortCode = encode(id);
        
        // 4. Save mapping
        UrlMapping mapping = new UrlMapping();
        mapping.setId(id);
        mapping.setShortCode(shortCode);
        mapping.setLongUrl(longUrl);
        mapping.setCreatedAt(LocalDateTime.now());
        urlRepository.save(mapping);
        
        // 5. Cache immediately (for fast redirect)
        redisTemplate.opsForValue().set(
            "url:" + shortCode, 
            longUrl, 
            Duration.ofHours(24)
        );
        
        // 6. Publish analytics event
        kafkaTemplate.send("url-analytics", shortCode);
        
        return shortCode;
    }
    
    // Redirect
    public String getLongUrl(String shortCode) {
        // 1. Check cache first (read-heavy)
        String cachedUrl = redisTemplate.opsForValue().get("url:" + shortCode);
        if (cachedUrl != null) {
            return cachedUrl;
        }
        
        // 2. Cache miss - check database
        UrlMapping mapping = urlRepository.findByShortCode(shortCode);
        if (mapping == null) {
            throw new RuntimeException("URL not found");
        }
        
        // 3. Cache it for next time
        redisTemplate.opsForValue().set(
            "url:" + shortCode, 
            mapping.getLongUrl(), 
            Duration.ofHours(24)
        );
        
        // 4. Increment click count (async)
        incrementClickCount(mapping.getId());
        
        return mapping.getLongUrl();
    }
}
```

3. Database Schema:

```sql
CREATE TABLE url_mapping (
    id BIGINT PRIMARY KEY,
    short_code VARCHAR(10) UNIQUE NOT NULL,
    long_url TEXT NOT NULL,
    user_id BIGINT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    expires_at TIMESTAMP,
    INDEX idx_short_code (short_code),
    INDEX idx_long_url (long_url(255))
);

CREATE TABLE url_analytics (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    url_id BIGINT NOT NULL,
    clicked_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    user_agent VARCHAR(255),
    ip_address VARCHAR(45),
    referrer VARCHAR(255),
    INDEX idx_url_id (url_id)
);
```

4. Caching Strategy:

· Cache-aside pattern: Check Redis, then DB
· TTL: 24 hours for hot URLs
· Write-through: Write to cache when creating a new URL

5. High Availability:

· Read replicas for redirects
· CDN for static assets
· Eventual consistency for analytics

---

16. What is a Content Delivery Network (CDN) and when do you use it?

In-Depth Answer:
"A CDN is a globally distributed network of proxy servers that cache static assets (images, CSS, JS, videos) at edge locations. This reduces latency by serving content from the server geographically closest to the user."

How it Works:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           CDN Architecture                                  │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│    User in US                                                                  User in Asia
│        │                                                                          │
│        ▼                                                                          ▼
│  ┌─────────────┐    ┌──────────────────────────────┐    ┌─────────────┐      │
│  │  Edge Server│───▶│    Origin Server (AWS)       │◀───│  Edge Server│      │
│  │  (US)       │    │    (Actual Application)      │    │  (Asia)     │      │
│  └─────────────┘    └──────────────────────────────┘    └─────────────┘      │
│        │                    │                                  │             │
│        │ Cache Miss →       │                                  │ Cache Hit  │
│        │─────────────────▶  │                                  │◀────────────│
│        │                    │                                 │             │
│        │ Cache Hit ←        │                                 │             │
│        │◀───────────────────│─────────────────────────────────│             │
│                                                                             │
│    Edge Server Responses:                                                   │
│    - Cached assets served directly                                          │
│    - Dynamic content fetched from origin                                   │
│    - Geo-distributed for low latency                                       │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

When to Use:

· Static assets (images, CSS, JS)
· Large files (videos, downloads)
· Global applications with users worldwide
· To reduce origin server load
· For DDoS protection

Implementation:

```nginx
# Nginx config for static assets
location /static/ {
    expires 1y;
    add_header Cache-Control "public, immutable";
}

# CloudFront (AWS) distribution
Resources:
  CloudFrontDistribution:
    Type: AWS::CloudFront::Distribution
    Properties:
      DistributionConfig:
        Origins:
          - DomainName: my-origin.com
            Id: my-origin
        DefaultCacheBehavior:
          TargetOriginId: my-origin
          ViewerProtocolPolicy: redirect-to-https
          CachedMethods:
            - GET
            - HEAD
          CachePolicyId: 658327ea-f89d-4fab-a63d-7e88639e58f6  # CachingOptimized
```

---

17. How do you handle session management in a scaled fullstack application?

In-Depth Answer:
"In a distributed environment, we avoid sticky sessions on a single server. Instead, we use stateless authentication with JWT tokens stored client-side, or use a centralized session store like Redis to manage sessions across all server instances."

Approach 1: JWT (Stateless)

```java
@Configuration
public class SecurityConfig {
    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .sessionManagement(session -> 
                session.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
            .addFilterBefore(jwtAuthenticationFilter(), 
                           UsernamePasswordAuthenticationFilter.class);
        return http.build();
    }
}
```

Approach 2: Redis Session Store

```xml
<dependency>
    <groupId>org.springframework.session</groupId>
    <artifactId>spring-session-data-redis</artifactId>
</dependency>
```

```java
@Configuration
@EnableRedisHttpSession(maxInactiveIntervalInSeconds = 1800)
public class SessionConfig {
    @Bean
    public LettuceConnectionFactory connectionFactory() {
        return new LettuceConnectionFactory();
    }
}

// Spring Boot automatically manages sessions in Redis
@RestController
public class SessionController {
    @GetMapping("/user")
    public String getUser(HttpSession session) {
        String userId = (String) session.getAttribute("userId");
        // Session data available across all server instances
        return userId;
    }
}
```

Session Management Best Practices:

· JWT: Short-lived access tokens + refresh tokens
· Redis: TTL for automatic cleanup
· Logout: Explicitly clear session on logout
· Security: HttpOnly, Secure, SameSite cookies

---

18. How would you design a Rate Limiter?

In-Depth Answer:
"A Rate Limiter protects APIs from abuse by limiting the number of requests from a client within a specified time window. It's typically implemented at the API Gateway level using algorithms like Token Bucket or Sliding Window."

Common Algorithms:

1. Token Bucket:

```
┌──────────────────────────────────────┐
│         Token Bucket Algorithm        │
├──────────────────────────────────────┤
│                                        │
│    ┌─────────────────────────┐        │
│    │      Bucket             │        │
│    │   (Capacity: 10)        │        │
│    │  ┌───┐ ┌───┐ ┌───┐    │        │
│    │  │ T │ │ T │ │ T │... │        │
│    │  └───┘ └───┘ └───┘    │        │
│    └─────────────────────────┘        │
│              │                        │
│              ▼                        │
│         ┌────────┐                   │
│         │Request │                   │
│         │Takes 1 │                   │
│         │Token   │                   │
│         └────────┘                   │
│                                        │
│    - Tokens added at fixed rate        │
│    - Request consumes token            │
│    - No token → reject                 │
│                                        │
└──────────────────────────────────────┘
```

2. Sliding Window (Redis):

```java
@Component
public class RateLimiterService {
    @Autowired
    private RedisTemplate<String, Long> redisTemplate;
    
    public boolean isAllowed(String key, int limit, int windowSeconds) {
        long currentTime = System.currentTimeMillis();
        String windowKey = "rate:" + key;
        
        // Remove outdated entries
        redisTemplate.opsForZSet().removeRangeByScore(
            windowKey, 0, currentTime - windowSeconds * 1000
        );
        
        // Count current window requests
        Long count = redisTemplate.opsForZSet().zCard(windowKey);
        
        if (count == null || count < limit) {
            // Add current request
            redisTemplate.opsForZSet().add(windowKey, 
                                          String.valueOf(currentTime), 
                                          currentTime);
            redisTemplate.expire(windowKey, windowSeconds, TimeUnit.SECONDS);
            return true;
        }
        
        return false;
    }
}
```

3. API Gateway Rate Limiting:

```yaml
# Spring Cloud Gateway
spring:
  cloud:
    gateway:
      routes:
      - id: user-service
        uri: lb://user-service
        predicates:
        - Path=/api/users/**
        filters:
        - name: RequestRateLimiter
          args:
            redis-rate-limiter.replenishRate: 10
            redis-rate-limiter.burstCapacity: 20
            key-resolver: "#{@userKeyResolver}"
```

```java
@Component
public class UserKeyResolver implements KeyResolver {
    @Override
    public Mono<String> resolve(ServerWebExchange exchange) {
        String userId = exchange.getRequest().getHeaders().getFirst("X-User-Id");
        if (userId != null) {
            return Mono.just(userId);
        }
        // Fallback to IP
        String ip = exchange.getRequest().getRemoteAddress().getAddress().getHostAddress();
        return Mono.just(ip);
    }
}
```

---

19. What is the difference between Vertical and Horizontal Scaling?

In-Depth Answer:

Vertical Scaling (Scale Up):

```
┌─────────────────────────────────────────────────────────────┐
│                    Vertical Scaling                         │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│    Before:               After:                              │
│    ┌─────────────┐       ┌─────────────────────────────┐    │
│    │ Server      │       │         Server              │    │
│    │ 4 CPU       │       │       16 CPU                │    │
│    │ 8 GB RAM    │       │       32 GB RAM             │    │
│    │ 1 TB Disk   │       │       4 TB Disk             │    │
│    │ Single DB   │       │       Single DB             │    │
│    └─────────────┘       └─────────────────────────────┘    │
│                                                              │
│    Advantages:                                               │
│    - Simple to implement                                    │
│    - No data partitioning needed                           │
│    - No network overhead                                    │
│                                                              │
│    Disadvantages:                                           │
│    - Hardware limits (can't scale forever)                 │
│    - Single point of failure                               │
│    - Expensive (upgrading hardware)                        │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

Horizontal Scaling (Scale Out):

```
┌─────────────────────────────────────────────────────────────┐
│                    Horizontal Scaling                        │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│    Before:                   After:                          │
│    ┌─────────┐               ┌─────────┐ ┌─────────┐        │
│    │ Server 1│               │ Server 1│ │ Server 2│        │
│    │ 4 CPU   │               │ 4 CPU   │ │ 4 CPU   │        │
│    │ 8 GB    │     ───▶      │ 8 GB    │ │ 8 GB    │        │
│    └─────────┘               └─────────┘ └─────────┘        │
│                                  │         │                │
│                                  └────┬────┘                │
│                                       │                     │
│                               ┌───────────────┐            │
│                               │ Load Balancer │            │
│                               └───────────────┘            │
│                                                              │
│    Advantages:                                               │
│    - Virtually unlimited scaling                           │
│    - Fault tolerance                                       │
│    - Cost-effective                                         │
│                                                              │
│    Disadvantages:                                           │
│    - Complex architecture                                  │
│    - Data partitioning required                           │
│    - Network overhead                                     │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

When to Use:

· Vertical: Small scale, quick fix, legacy applications
· Horizontal: Large scale, cloud-native applications, microservices

---

20. How would you design a real-time data flow chat application (frontend to backend)?

In-Depth Answer:
"Real-time chat requires a persistent bi-directional connection. We use WebSockets for this, with a Pub/Sub layer like Redis to route messages between different server instances."

Complete Architecture:

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                        Chat Application Architecture                                 │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                      │
│  ┌─────────────────────────────────────────────────────────────────────────────┐    │
│  │                               React Frontend                                 │    │
│  │  ┌─────────────────────┐    ┌─────────────────────┐                          │    │
│  │  │    WebSocket Client │    │    WebSocket Client │                          │    │
│  │  │    (User 1)         │    │    (User 2)         │                          │    │
│  │  └──────────┬──────────┘    └──────────┬──────────┘                          │    │
│  └─────────────┼───────────────────────────┼──────────────────────────────────┘    │
│                │                           │                                        │
│                ▼                           ▼                                        │
│  ┌─────────────────────────────────────────────────────────────────────────────┐    │
│  │                    Load Balancer (Nginx)                                    │    │
│  │                    (WebSocket Support)                                      │    │
│  └─────────────────────────────────────────────────────────────────────────────┘    │
│                │                           │                                        │
│                ▼                           ▼                                        │
│  ┌─────────────────────────┐  ┌─────────────────────────┐                        │
│  │    Chat Server 1        │  │    Chat Server 2        │                        │
│  │  ┌─────────────────────┐│  │  ┌─────────────────────┐│                        │
│  │  │  WebSocket Handler  ││  │  │  WebSocket Handler  ││                        │
│  │  │  Message Processing ││  │  │  Message Processing ││                        │
│  │  └─────────────────────┘│  │  └─────────────────────┘│                        │
│  └───────────┬─────────────┘  └───────────┬─────────────┘                        │
│              │                             │                                      │
│              └──────────────┬──────────────┘                                      │
│                             │                                                     │
│  ┌─────────────────────────┴─────────────────────────────────────────────────┐    │
│  │                         Redis (Pub/Sub)                                    │    │
│  │  ┌────────────────────────────────────────────────────────────────────┐   │    │
│  │  │  Channel: chat:room1         Channel: chat:room2                 │   │    │
│  │  │  [msg1, msg2, msg3]          [msg4, msg5, msg6]                 │   │    │
│  │  └────────────────────────────────────────────────────────────────────┘   │    │
│  └──────────────────────────────┬────────────────────────────────────────────┘    │
│                                 │                                                │
│                                 ▼                                                │
│  ┌─────────────────────────────────────────────────────────────────────────────┐ │
│  │                         Database (MongoDB/PostgreSQL)                       │ │
│  │  ┌────────────────────────────────────────────────────────────────────┐   │ │
│  │  │  Messages Collection/Table                                         │   │ │
│  │  │  { id, roomId, senderId, content, timestamp, readStatus }        │   │ │
│  │  └────────────────────────────────────────────────────────────────────┘   │ │
│  │  ┌────────────────────────────────────────────────────────────────────┐   │ │
│  │  │  Rooms Collection/Table                                           │   │ │
│  │  │  { id, name, participants, createdAt }                            │   │ │
│  │  └────────────────────────────────────────────────────────────────────┘   │ │
│  └─────────────────────────────────────────────────────────────────────────────┘ │
│                                                                                    │
└────────────────────────────────────────────────────────────────────────────────────┘
```

Backend Implementation:

```java
// ===== WebSocket Configuration =====
@Configuration
@EnableWebSocketMessageBroker
public class WebSocketConfig implements WebSocketMessageBrokerConfigurer {
    @Override
    public void configureMessageBroker(MessageBrokerRegistry config) {
        config.enableSimpleBroker("/topic");
        config.setApplicationDestinationPrefixes("/app");
    }
    
    @Override
    public void registerStompEndpoints(StompEndpointRegistry registry) {
        registry.addEndpoint("/ws")
            .setAllowedOrigins("*")
            .withSockJS();
    }
}

// ===== WebSocket Controller =====
@Controller
public class ChatController {
    @Autowired
    private ChatService chatService;
    @Autowired
    private SimpMessagingTemplate messagingTemplate;
    
    @MessageMapping("/chat.sendMessage")
    public void sendMessage(@Payload ChatMessage message) {
        // 1. Save message to database
        ChatMessage saved = chatService.saveMessage(message);
        
        // 2. Broadcast to room
        messagingTemplate.convertAndSend(
            "/topic/room/" + message.getRoomId(), 
            saved
        );
        
        // 3. Store in Redis for pub/sub (multi-instance)
        chatService.publishToRedis(message);
    }
    
    @MessageMapping("/chat.addUser")
    public void addUser(@Payload UserPresence presence) {
        // Notify everyone in room
        messagingTemplate.convertAndSend(
            "/topic/room/" + presence.getRoomId() + "/members",
            presence
        );
    }
}

// ===== Redis Pub/Sub for Multi-Instance Support =====
@Component
public class RedisMessageSubscriber implements MessageListener {
    @Autowired
    private SimpMessagingTemplate messagingTemplate;
    
    @Override
    public void onMessage(Message message, byte[] pattern) {
        ChatMessage chatMessage = deserialize(message);
        // Broadcast to local WebSocket subscribers
        messagingTemplate.convertAndSend(
            "/topic/room/" + chatMessage.getRoomId(),
            chatMessage
        );
    }
}

@Service
public class ChatService {
    @Autowired
    private RedisTemplate<String, ChatMessage> redisTemplate;
    
    public void publishToRedis(ChatMessage message) {
        redisTemplate.convertAndSend("chat-channel", message);
    }
}
```

Frontend Implementation:

```jsx
// React component with WebSocket
import React, { useState, useEffect } from 'react';
import { Client } from '@stomp/stompjs';

function ChatRoom({ roomId, user }) {
    const [messages, setMessages] = useState([]);
    const [input, setInput] = useState('');
    const [stompClient, setStompClient] = useState(null);
    
    useEffect(() => {
        // Connect to WebSocket
        const client = new Client({
            brokerURL: 'ws://localhost:8080/ws',
            connectHeaders: {
                login: user.id,
                passcode: user.token,
            },
            onConnect: () => {
                // Subscribe to room messages
                client.subscribe(`/topic/room/${roomId}`, (message) => {
                    const newMessage = JSON.parse(message.body);
                    setMessages(prev => [...prev, newMessage]);
                });
                
                // Send user presence
                client.publish({
                    destination: '/app/chat.addUser',
                    body: JSON.stringify({ userId: user.id, roomId: roomId }),
                });
            },
            onDisconnect: () => {
                // Clean up
            },
        });
        
        client.activate();
        setStompClient(client);
        
        // Cleanup on unmount
        return () => {
            if (client) {
                client.deactivate();
            }
        };
    }, [roomId, user.id]);
    
    const sendMessage = () => {
        if (input.trim() && stompClient) {
            const message = {
                roomId: roomId,
                senderId: user.id,
                senderName: user.name,
                content: input.trim(),
                timestamp: new Date().toISOString(),
            };
            
            stompClient.publish({
                destination: '/app/chat.sendMessage',
                body: JSON.stringify(message),
            });
            
            setInput('');
        }
    };
    
    return (
        <div className="chat-room">
            <div className="messages">
                {messages.map((msg, idx) => (
                    <Message key={idx} message={msg} />
                ))}
            </div>
            <div className="input-area">
                <input
                    value={input}
                    onChange={(e) => setInput(e.target.value)}
                    onKeyPress={(e) => e.key === 'Enter' && sendMessage()}
                />
                <button onClick={sendMessage}>Send</button>
            </div>
        </div>
    );
}
```

Key Design Decisions:

1. WebSockets: Persistent bi-directional communication
2. STOMP: Messaging protocol over WebSocket for easy pub/sub
3. Redis Pub/Sub: Enables horizontal scaling across multiple servers
4. Load Balancer: Nginx with WebSocket support
5. Database: PostgreSQL for persistence, Redis for caching
6. Presence: Track online/offline status
7. History: Load last N messages on join

Scalability Considerations:

· Use Redis for presence tracking across instances
· Horizontally scale WebSocket servers
· Store messages in MongoDB for chat history (write-heavy)
· Use CDN for file attachments

