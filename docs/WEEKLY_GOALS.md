# 📋 WEEKLY GOALS - 52 Week Learning Roadmap

> **Philosophy:** Build → Create Problem → Analyze Solutions → Implement Best Approach

---

## 🎯 Quick Reference

| Phase | Weeks | Goal | Services |
|-------|-------|------|----------|
| **1: Foundation** | 1-9 | Build core microservices & discover patterns | 7 services |
| **2: Observability** | 10-17 | Add tracing, logging, metrics | - |
| **3: Event-Driven** | 18-34 | Implement CQRS, Event Sourcing | - |
| **4: Production** | 35-52 | Kubernetes, scaling, security | - |

---

# 🔷 PHASE 1: FOUNDATION (Weeks 1-9)

## Week 1: User Service - Race Condition Discovery

**Goal:** Build User Service CRUD, then deliberately break it with concurrent registration

### 📝 Pseudocode

```pseudocode
CLASS UserService
  FUNCTION register(email, password, fullName)
    IF user_exists_in_database(email)
      THROW UserAlreadyExistsException
    END IF
    
    CREATE new_user with email, hash(password), fullName
    SAVE new_user to database
    RETURN new_user_dto
  END FUNCTION
END CLASS

// Problem: What if two requests check at the SAME TIME?
// Both see email doesn't exist, both insert, one fails!
```

### 🔴 Problem Statement

**Concurrent Registration Race Condition**

Scenario:
```
Time T1: Request A checks if user@email.com exists → NO
Time T1+1ms: Request B checks if user@email.com exists → NO
Time T1+2ms: Request A inserts user@email.com → SUCCESS
Time T1+3ms: Request B tries to insert user@email.com → DUPLICATE KEY ERROR ❌
```

**Why it matters:** Users getting errors while registering leads to support tickets!

### 🟢 Solution Analysis

| Approach | Pros | Cons | Best For |
|----------|------|------|----------|
| **Unique constraint only** | Simple DB check | Race condition still possible | Learning purpose |
| **Check + Unique constraint** | Application + DB safety | Small gap still exists | Not ideal |
| **Unique constraint + Exception handling** | Catches race condition | Need try-catch | Basic production |
| **Database-level uniqueness + optimistic locking** | Thread-safe, scalable | More complex | ⭐ Best approach |

### ✅ Best Approach: Unique Constraint + Exception Handling

```java
@Entity
@Table(uniqueConstraints = @UniqueConstraint(columnNames = "email"))
public class User {
    @Id
    private UUID id;
    
    @Column(unique = true, nullable = false)
    private String email;
    
    @Version  // Optimistic locking
    private Long version;
}

@Service
public class UserService {
    public UserDTO register(RegisterRequest request) {
        try {
            User user = new User();
            user.setEmail(request.getEmail());
            user.setPasswordHash(passwordEncoder.encode(request.getPassword()));
            userRepository.save(user);
            return mapToDTO(user);
        } catch (DataIntegrityViolationException e) {
            // Another thread/request already registered this email
            throw new UserAlreadyExistsException("Email already registered");
        }
    }
}
```

### 📚 Deliverables

- [x] User entity with email unique constraint
- [x] UserRepository interface
- [x] UserService with register/getProfile/updateProfile
- [x] UserController with REST endpoints
- [x] ConcurrentRegistrationTest that exposes race condition
- [x] Exception handling with DataIntegrityViolationException
- [x] Documentation in LEARNING_LOG.md

### 🧪 Test to Write

```java
@Test
void testConcurrentRegistrationWithSameEmail() throws InterruptedException {
    String email = "concurrent@test.com";
    CountDownLatch latch = new CountDownLatch(2);
    AtomicInteger successCount = new AtomicInteger(0);
    List<Exception> exceptions = Collections.synchronizedList(new ArrayList<>());

    Runnable registerTask = () -> {
        try {
            RegisterRequest req = new RegisterRequest(email, "John", "pass123");
            userService.register(req);
            successCount.incrementAndGet();
        } catch (UserAlreadyExistsException e) {
            exceptions.add(e);
        } finally {
            latch.countDown();
        }
    };

    new Thread(registerTask).start();
    new Thread(registerTask).start();

    latch.await();

    assertEquals(1, successCount.get(), "Only 1 registration should succeed");
    assertEquals(1, exceptions.size(), "Other should fail");
}
```

### 📖 What You Learn

- Unique constraints in databases
- Race conditions in multithreading
- DataIntegrityViolationException handling
- Concurrent testing patterns
- Why database constraints matter

---

## Week 2: JWT Authentication & Token Management

**Goal:** Add authentication layer, discover token expiration issues, implement refresh tokens

### 📝 Pseudocode

```pseudocode
CLASS JwtTokenProvider
  FUNCTION generateToken(userId, email)
    payload = { userId, email, issuedAt: NOW, expiresAt: NOW + 1hour }
    signature = HMAC_SHA256(payload + secret)
    RETURN base64(header + payload + signature)
  END FUNCTION
  
  FUNCTION validateToken(token)
    parts = token.split(".")
    payload = decode(parts[1])
    signature = HMAC_SHA256(parts[0] + parts[1] + secret)
    
    IF signature != parts[2]
      THROW InvalidTokenException
    END IF
    
    IF payload.expiresAt < NOW
      THROW TokenExpiredException
    END IF
    
    RETURN payload
  END FUNCTION
END CLASS
```

### 🔴 Problem Statement

**Token Expiration & Refresh Token Issue**

Scenario:
```
User logs in → Gets access token (expires in 1 hour)
After 50 minutes → Token still valid, everything works
After 65 minutes → Token expired, user must login again ❌

Better approach?
After 65 minutes → Use refresh token to get new access token
```

### 🟢 Solution: Refresh Token Pattern

```java
@Entity
public class RefreshToken {
    @Id
    private UUID id;
    
    @ManyToOne
    private User user;
    
    @Column(unique = true, nullable = false)
    private String token;
    
    private LocalDateTime expiresAt;
    
    private LocalDateTime revokedAt;  // For logout
}

@Service
public class AuthService {
    public LoginResponse login(LoginRequest request) {
        User user = authenticate(request.getEmail(), request.getPassword());
        
        String accessToken = jwtProvider.generateAccessToken(user);
        String refreshToken = generateRefreshToken(user);
        
        return LoginResponse.builder()
            .accessToken(accessToken)
            .refreshToken(refreshToken)
            .expiresIn(3600) // 1 hour
            .build();
    }
    
    public LoginResponse refreshAccessToken(String refreshTokenString) {
        RefreshToken refreshToken = refreshTokenRepository
            .findByToken(refreshTokenString)
            .orElseThrow(() -> new InvalidTokenException("Refresh token not found"));
        
        if (refreshToken.isExpired() || refreshToken.isRevoked()) {
            throw new InvalidTokenException("Refresh token is invalid");
        }
        
        User user = refreshToken.getUser();
        String newAccessToken = jwtProvider.generateAccessToken(user);
        
        return LoginResponse.builder()
            .accessToken(newAccessToken)
            .refreshToken(refreshTokenString)
            .expiresIn(3600)
            .build();
    }
}
```

### 📚 Deliverables

- [x] JwtTokenProvider with generate/validate methods
- [x] RefreshToken entity
- [x] AuthService with login/refresh methods
- [x] SecurityConfig with JWT filter
- [x] Protected endpoints with @PreAuthorize
- [x] Logout endpoint that revokes refresh token
- [x] Tests for token expiration
- [x] Tests for token refresh

### 📖 What You Learn

- JWT structure (header.payload.signature)
- Token expiration and validation
- Refresh token pattern for seamless UX
- Session vs token-based authentication
- Security best practices

---

## Week 3: Product Service - N+1 Query Problem

**Goal:** Build Product Service, then expose N+1 query problem, fix with JOIN FETCH

### 📝 Pseudocode

```pseudocode
CLASS ProductService
  FUNCTION getAllProducts()
    // BAD: N+1 Problem
    products = SELECT * FROM products LIMIT 10
    FOR EACH product IN products
      product.reviews = SELECT * FROM reviews WHERE product_id = product.id
      product.category = SELECT * FROM categories WHERE id = product.category_id
    END FOR
    RETURN products
    
    // Result: 1 query for products + 10 queries for reviews + 10 for categories = 21 queries!
    
    // GOOD: JOIN FETCH
    products = SELECT p FROM Product p 
                 JOIN FETCH p.reviews
                 JOIN FETCH p.category
    RETURN products
    
    // Result: 1 optimized query!
  END FUNCTION
END CLASS
```

### 🔴 Problem Statement

**N+1 Query Problem**

```
GET /api/products
    ↓
1 query: SELECT * FROM products LIMIT 10
    ↓ (for each product)
10 queries: SELECT * FROM reviews WHERE product_id = ?
    ↓ (for each product)
10 queries: SELECT * FROM categories WHERE id = ?

Total: 21 database roundtrips! ❌
```

### 🟢 Solution: JOIN FETCH

```java
@Repository
public interface ProductRepository extends JpaRepository<Product, UUID> {
    @Query("""
        SELECT DISTINCT p FROM Product p
        LEFT JOIN FETCH p.reviews
        LEFT JOIN FETCH p.category
        WHERE p.active = true
        ORDER BY p.createdAt DESC
    """)
    Page<Product> findAllActiveWithDetails(Pageable pageable);
}

@Service
public class ProductService {
    public Page<ProductDTO> getAllProducts(Pageable pageable) {
        // Now only 1 query!
        Page<Product> products = productRepository.findAllActiveWithDetails(pageable);
        return products.map(this::mapToDTO);
    }
}
```

### 📊 Performance Comparison

| Approach | Query Count | Time | Memory |
|----------|------------|------|--------|
| N+1 (no optimization) | 21 | 500ms | High |
| JOIN FETCH | 1 | 50ms | Low |
| Caching (later) | 1 (then 0) | 5ms | Moderate |

### 📚 Deliverables

- [x] Product, Category, Review entities
- [x] MongoDB integration (for product metadata)
- [x] ProductRepository with pagination
- [x] Write test showing N+1 problem
- [x] Fix with JOIN FETCH
- [x] Add database indices
- [x] Performance comparison tests
- [x] Documentation

### 📖 What You Learn

- N+1 query problem identification
- JPQL JOIN FETCH syntax
- Database indexing strategies
- Query optimization techniques
- Performance testing

---

## Week 4: Cart Service - Distributed Transactions

**Goal:** Build Cart Service, expose distributed transaction problem with Saga pattern

### 📝 Pseudocode

```pseudocode
FUNCTION addToCart(userId, productId, quantity)
  // Problem: What if product stock runs out between check and update?
  
  product = SELECT FROM products WHERE id = productId
  IF product.stock < quantity
    THROW OutOfStockException
  END IF
  
  // Gap here! Another request could buy the last item
  
  UPDATE products SET stock = stock - quantity WHERE id = productId
  INSERT INTO cart_items (userId, productId, quantity)
  
  RETURN success
END FUNCTION

// What if cart service crashes after inventory updated but before inserting cart item?
// Inventory is lost, but cart is empty!
```

### 🔴 Problem Statement

**Distributed Transaction Inconsistency**

```
Request 1: Add 5 items to cart
  ✓ Check inventory (50 items available)
  ✓ Reserve from inventory (now 45)
  ✗ Service crashes before adding to cart

Result: Inventory reduced but item not in cart! ❌
```

### 🟢 Solution: Saga Pattern (Compensating Transactions)

```java
@Service
public class CartService {
    @Transactional
    public void addToCart(UUID userId, UUID productId, int quantity) {
        // Step 1: Add to cart
        CartItem cartItem = new CartItem();
        cartItem.setUserId(userId);
        cartItem.setProductId(productId);
        cartItem.setQuantity(quantity);
        cartItemRepository.save(cartItem);
        
        // Step 2: Call Product Service to reserve inventory
        try {
            InventoryReserveRequest request = new InventoryReserveRequest(productId, quantity);
            productServiceClient.reserveInventory(request);
        } catch (Exception e) {
            // Compensating transaction: Remove from cart
            cartItemRepository.delete(cartItem);
            throw new CartOperationFailedException("Failed to reserve inventory", e);
        }
    }
}

@Service
public class ProductService {
    @Transactional
    public void reserveInventory(UUID productId, int quantity) {
        Product product = productRepository.findById(productId).orElseThrow();
        
        if (product.getStock() < quantity) {
            throw new OutOfStockException("Insufficient inventory");
        }
        
        product.setStock(product.getStock() - quantity);
        productRepository.save(product);
    }
}
```

### 📚 Deliverables

- [x] Cart and CartItem entities
- [x] CartService with add/remove operations
- [x] Implement stock validation
- [x] Write concurrent add_to_cart test
- [x] Expose race condition
- [x] Implement compensating transaction
- [x] Integration test for saga

### 📖 What You Learn

- Distributed transactions challenges
- Saga pattern introduction
- Compensating transactions
- Service-to-service communication
- Transaction boundaries in microservices

---

## Week 5: Order Service - Full Saga Pattern Implementation

**Goal:** Complete Saga orchestration for entire order flow (Order → Payment → Inventory)

### 📝 Pseudocode

```pseudocode
CLASS OrderSaga
  FUNCTION executeOrderSaga(userId, cartItems, shippingAddress)
    // Step 1: Create order
    order = createOrder(userId, cartItems, shippingAddress)
    IF order is NULL
      THROW OrderCreationFailedException
    END IF
    
    // Step 2: Process payment
    TRY
      paymentResult = paymentService.processPayment(order.getId(), order.totalPrice)
    CATCH PaymentFailedException
      COMPENSATE: cancelOrder(order.getId())
      THROW
    END TRY
    
    // Step 3: Update inventory
    TRY
      inventoryService.reserveItems(order.items)
    CATCH OutOfStockException
      COMPENSATE: refundPayment(paymentResult.transactionId)
      COMPENSATE: cancelOrder(order.getId())
      THROW
    END TRY
    
    // Step 4: Create shipment
    TRY
      shipmentService.scheduleShipment(order.getId())
    CATCH ShipmentException
      COMPENSATE: releaseInventory(order.items)
      COMPENSATE: refundPayment(paymentResult.transactionId)
      COMPENSATE: cancelOrder(order.getId())
      THROW
    END TRY
    
    RETURN order_with_confirmation
  END FUNCTION
END CLASS
```

### 🔴 Problem Statement

**Distributed Order Processing**

```
Order placement needs to coordinate 3 services:
1. Payment Service (Process payment)
2. Inventory Service (Reserve items)
3. Shipment Service (Schedule delivery)

If ANY step fails, EVERYTHING should rollback
But these are DIFFERENT DATABASES!
❌ Can't use traditional database transactions
```

### 🟢 Solution: Orchestrated Saga Pattern

```java
@Service
@RequiredArgsConstructor
@Slf4j
public class OrderSagaOrchestrator {
    private final OrderRepository orderRepository;
    private final PaymentServiceClient paymentClient;
    private final InventoryServiceClient inventoryClient;
    private final ShipmentServiceClient shipmentClient;
    private final OrderEventPublisher eventPublisher;

    @Transactional
    public OrderResponse executeOrderSaga(CreateOrderRequest request) {
        Order order = createOrder(request);
        log.info("Order created: {}", order.getId());

        try {
            // Step 1: Process Payment
            PaymentResponse payment = paymentClient.processPayment(
                order.getId(),
                order.getTotalPrice()
            );
            order.setPaymentId(payment.getId());
            order.setStatus(OrderStatus.PAYMENT_CONFIRMED);
            orderRepository.save(order);
            eventPublisher.publishPaymentCompleted(order.getId());
            log.info("Payment processed for order: {}", order.getId());

            // Step 2: Reserve Inventory
            try {
                inventoryClient.reserveItems(order.getId(), order.getItems());
                order.setStatus(OrderStatus.INVENTORY_RESERVED);
                orderRepository.save(order);
                eventPublisher.publishInventoryReserved(order.getId());
                log.info("Inventory reserved for order: {}", order.getId());
            } catch (OutOfStockException e) {
                // Compensate: Refund payment
                paymentClient.refundPayment(order.getPaymentId());
                order.setStatus(OrderStatus.CANCELLED);
                orderRepository.save(order);
                throw new SagaExecutionException("Inventory not available", e);
            }

            // Step 3: Schedule Shipment
            try {
                ShipmentResponse shipment = shipmentClient.scheduleShipment(order.getId());
                order.setShipmentId(shipment.getId());
                order.setStatus(OrderStatus.CONFIRMED);
                orderRepository.save(order);
                eventPublisher.publishOrderConfirmed(order.getId());
                log.info("Order confirmed: {}", order.getId());
            } catch (ShipmentException e) {
                // Compensate: Release inventory and refund
                inventoryClient.releaseItems(order.getId());
                paymentClient.refundPayment(order.getPaymentId());
                order.setStatus(OrderStatus.CANCELLED);
                orderRepository.save(order);
                throw new SagaExecutionException("Shipment scheduling failed", e);
            }

            return mapToResponse(order);

        } catch (Exception e) {
            log.error("Saga failed for order: {}", order.getId(), e);
            order.setStatus(OrderStatus.FAILED);
            orderRepository.save(order);
            throw new SagaExecutionException("Order processing failed", e);
        }
    }

    private Order createOrder(CreateOrderRequest request) {
        Order order = new Order();
        order.setUserId(request.getUserId());
        order.setItems(request.getItems());
        order.setTotalPrice(calculateTotal(request.getItems()));
        order.setStatus(OrderStatus.PENDING);
        order.setShippingAddress(request.getShippingAddress());
        return orderRepository.save(order);
    }

    private BigDecimal calculateTotal(List<OrderItem> items) {
        return items.stream()
            .map(item -> item.getPrice().multiply(BigDecimal.valueOf(item.getQuantity())))
            .reduce(BigDecimal.ZERO, BigDecimal::add);
    }
}
```

### 📚 Deliverables

- [x] Order entity with status enum
- [x] OrderSagaOrchestrator
- [x] Service clients for Payment, Inventory, Shipment
- [x] Compensating transactions for each step
- [x] Event publishing for saga events
- [x] Test: Successful saga execution
- [x] Test: Payment failure with compensation
- [x] Test: Inventory failure with compensation
- [x] Test: Shipment failure with full compensation

### 📖 What You Learn

- Saga orchestration pattern
- Compensating transactions
- Distributed consistency models
- Event-driven saga coordination
- Service client implementations

---

## Week 6: Payment Service & Idempotency Pattern

**Goal:** Build Payment Service with idempotency to prevent duplicate charges

### 📝 Pseudocode

```pseudocode
FUNCTION processPayment(orderId, amount, idempotencyKey)
  // Check if payment already processed with this key
  existingPayment = SELECT FROM payments WHERE idempotency_key = idempotencyKey
  IF existingPayment EXISTS
    RETURN existingPayment  // Don't charge again!
  END IF
  
  // Save payment intent first (BEFORE calling gateway)
  payment = CREATE new Payment {
    orderId, amount, status: PENDING, idempotencyKey
  }
  SAVE payment
  
  // Call payment gateway
  TRY
    result = paymentGateway.charge(amount)
    payment.status = COMPLETED
    payment.transactionId = result.transactionId
    SAVE payment
  CATCH PaymentFailedException
    payment.status = FAILED
    SAVE payment
    THROW
  END TRY
  
  RETURN payment
END FUNCTION
```

### 🔴 Problem Statement

**Duplicate Payments**

```
User clicks "Pay" button
→ Network timeout, but payment WAS processed
→ User clicks "Pay" again
→ Customer charged TWICE! ❌
```

### 🟢 Solution: Idempotency Key

```java
@Entity
public class Payment {
    @Id
    private UUID id;
    
    @Column(unique = true, nullable = false)
    private String idempotencyKey;
    
    private UUID orderId;
    private BigDecimal amount;
    
    @Enumerated(EnumType.STRING)
    private PaymentStatus status;
    
    private String transactionId;
    private LocalDateTime createdAt;
}

@Service
@RequiredArgsConstructor
public class PaymentService {
    private final PaymentRepository paymentRepository;
    private final PaymentGatewayClient gatewayClient;

    @Transactional
    public PaymentResponse processPayment(
        UUID orderId,
        BigDecimal amount,
        String idempotencyKey) {
        
        // Check if payment already exists
        Optional<Payment> existingPayment = paymentRepository
            .findByIdempotencyKey(idempotencyKey);
        
        if (existingPayment.isPresent()) {
            log.info("Idempotent retry detected, returning existing payment");
            return mapToResponse(existingPayment.get());
        }
        
        // Save payment intent FIRST
        Payment payment = new Payment();
        payment.setIdempotencyKey(idempotencyKey);
        payment.setOrderId(orderId);
        payment.setAmount(amount);
        payment.setStatus(PaymentStatus.PENDING);
        payment = paymentRepository.save(payment);
        
        try {
            // Call payment gateway
            ChargeResponse response = gatewayClient.charge(
                amount,
                idempotencyKey  // Also send to gateway for double idempotency
            );
            
            payment.setTransactionId(response.getTransactionId());
            payment.setStatus(PaymentStatus.COMPLETED);
            paymentRepository.save(payment);
            
            log.info("Payment processed: {}", payment.getId());
            return mapToResponse(payment);
            
        } catch (PaymentGatewayException e) {
            payment.setStatus(PaymentStatus.FAILED);
            paymentRepository.save(payment);
            throw new PaymentProcessingException("Payment failed", e);
        }
    }
}
```

### 📚 Deliverables

- [x] Payment entity with idempotency key
- [x] PaymentService with idempotency check
- [x] PaymentController with REST endpoints
- [x] Mock payment gateway for testing
- [x] Tests for successful payment
- [x] Tests for idempotent retry
- [x] Tests for payment failure
- [x] Integration with Order Saga

### 📖 What You Learn

- Idempotency pattern
- At-least-once delivery semantics
- Payment processing best practices
- Error handling in payment systems
- Unique constraints for preventing duplicates

---

## Week 7: Notification Service & Async Messaging

**Goal:** Implement RabbitMQ-based async notification system

### 📝 Pseudocode

```pseudocode
EVENT OrderCreatedEvent
  orderId: UUID
  userId: UUID
  orderTotal: Money
  items: List<OrderItem>
END EVENT

CLASS NotificationService
  FUNCTION onOrderCreated(event: OrderCreatedEvent)
    // Send email asynchronously
    emailTemplate = LOAD_TEMPLATE("order_confirmation.html")
    email = COMPOSE_EMAIL {
      to: event.userId.email,
      subject: "Order Confirmation: " + event.orderId,
      body: emailTemplate.render(event)
    }
    emailGateway.SEND_ASYNC(email)
    
    // Send SMS
    sms = COMPOSE_SMS {
      to: event.userId.phoneNumber,
      message: "Order confirmed. Ref: " + event.orderId
    }
    smsGateway.SEND_ASYNC(sms)
    
    // Save notification record
    notification = CREATE Notification {
      userId: event.userId,
      type: EMAIL + SMS,
      status: PENDING
    }
    SAVE notification
  END FUNCTION
END CLASS
```

### 🟢 Solution: Event-Driven Architecture

```java
// Event definition
@Data
@AllArgsConstructor
public class OrderCreatedEvent {
    private UUID orderId;
    private UUID userId;
    private BigDecimal orderTotal;
    private List<OrderItemDTO> items;
    private LocalDateTime createdAt;
}

// Event publisher (in Order Service)
@Service
@RequiredArgsConstructor
public class OrderEventPublisher {
    private final RabbitTemplate rabbitTemplate;
    private final ObjectMapper objectMapper;

    public void publishOrderCreated(Order order) {
        OrderCreatedEvent event = new OrderCreatedEvent(
            order.getId(),
            order.getUserId(),
            order.getTotalPrice(),
            mapItems(order.getItems()),
            order.getCreatedAt()
        );
        
        rabbitTemplate.convertAndSend(
            "orders.exchange",
            "order.created",
            objectMapper.writeValueAsString(event)
        );
        
        log.info("OrderCreatedEvent published: {}", order.getId());
    }
}

// Event subscriber (in Notification Service)
@Service
@RequiredArgsConstructor
@Slf4j
public class OrderEventSubscriber {
    private final EmailService emailService;
    private final SmsService smsService;
    private final NotificationRepository notificationRepository;

    @RabbitListener(queues = "notification.order-created.queue")
    public void handleOrderCreated(String message) throws IOException {
        OrderCreatedEvent event = objectMapper.readValue(message, OrderCreatedEvent.class);
        log.info("Received OrderCreatedEvent: {}", event.getOrderId());

        try {
            // Send email
            emailService.sendOrderConfirmationEmail(event);
            
            // Send SMS
            smsService.sendOrderConfirmationSms(event);
            
            // Save notification
            Notification notification = new Notification();
            notification.setOrderId(event.getOrderId());
            notification.setUserId(event.getUserId());
            notification.setType(NotificationType.ORDER_CONFIRMATION);
            notification.setStatus(NotificationStatus.SENT);
            notificationRepository.save(notification);
            
            log.info("Notifications sent for order: {}", event.getOrderId());
            
        } catch (Exception e) {
            log.error("Failed to send notifications for order: {}", event.getOrderId(), e);
            // Message will be requeued (RabbitMQ default)
            throw new RuntimeException(e);
        }
    }
}

// RabbitMQ Configuration
@Configuration
public class RabbitMqConfig {
    public static final String ORDERS_EXCHANGE = "orders.exchange";
    public static final String ORDER_CREATED_QUEUE = "notification.order-created.queue";
    public static final String ORDER_CREATED_ROUTING_KEY = "order.created";

    @Bean
    public TopicExchange ordersExchange() {
        return new TopicExchange(ORDERS_EXCHANGE, true, false);
    }

    @Bean
    public Queue orderCreatedQueue() {
        return QueueBuilder.durable(ORDER_CREATED_QUEUE)
            .withArgument("x-dead-letter-exchange", "orders.dlx")
            .withArgument("x-dead-letter-routing-key", "order.created.dlq")
            .build();
    }

    @Bean
    public Binding orderCreatedBinding(Queue orderCreatedQueue, TopicExchange ordersExchange) {
        return BindingBuilder.bind(orderCreatedQueue)
            .to(ordersExchange)
            .with(ORDER_CREATED_ROUTING_KEY);
    }
}
```

### 📚 Deliverables

- [x] Notification entity
- [x] Event publishing from Order Service
- [x] Event subscribing in Notification Service
- [x] RabbitMQ queue configuration
- [x] Email service integration
- [x] SMS service integration
- [x] Retry logic for failed messages
- [x] Dead letter queue for permanent failures

### 📖 What You Learn

- Event-driven architecture
- Message queue patterns
- RabbitMQ exchange/queue binding
- Publisher/subscriber pattern
- Async notification handling

---

## Week 8: API Gateway - Rate Limiting & Circuit Breaker

**Goal:** Implement centralized gateway with resilience patterns

### 📝 Pseudocode

```pseudocode
CLASS ApiGateway
  FUNCTION handleRequest(request)
    // Step 1: Rate limiting
    IF rateLimiter.isExceeded(request.clientId)
      RETURN 429 Too Many Requests
    END IF
    
    // Step 2: Routing
    targetService = ROUTE(request.path)
    
    // Step 3: Circuit breaker
    TRY
      response = circuitBreaker.call(() -> {
        FORWARD_REQUEST_TO(targetService, request)
      })
      RETURN response
    CATCH CircuitOpenException
      RETURN 503 Service Unavailable (with fallback)
    END TRY
  END FUNCTION
END CLASS
```

### 🟢 Solution: Spring Cloud Gateway + Resilience4j

```java
// Rate Limiting Configuration
@Configuration
public class RateLimitingConfig {
    @Bean
    public KeyResolver userKeyResolver() {
        return exchange -> Mono.just(
            exchange.getRequest()
                .getHeaders()
                .getFirst(HttpHeaders.AUTHORIZATION)
                .replaceAll("Bearer ", "")
        );
    }
}

// Application YAML
spring:
  cloud:
    gateway:
      routes:
        - id: user-service
          uri: http://localhost:8081
          predicates:
            - Path=/api/users/**
          filters:
            - name: RequestRateLimiter
              args:
                redis-rate-limiter:
                  replenishRate: 10
                  burstCapacity: 20
                key-resolver: "#{@userKeyResolver}"
            - name: CircuitBreaker
              args:
                name: userServiceCircuitBreaker
                fallbackUri: forward:/fallback/user-service

        - id: product-service
          uri: http://localhost:8082
          predicates:
            - Path=/api/products/**
          filters:
            - name: RequestRateLimiter
              args:
                redis-rate-limiter:
                  replenishRate: 20
                  burstCapacity: 50
            - name: CircuitBreaker
              args:
                name: productServiceCircuitBreaker

resilience4j:
  circuitbreaker:
    instances:
      userServiceCircuitBreaker:
        registerHealthIndicator: true
        slidingWindowSize: 10
        failureRateThreshold: 50
        waitDurationInOpenState: 30000  # 30 seconds
        slowCallRateThreshold: 50
        slowCallDurationThreshold: 2000  # 2 seconds
        
      productServiceCircuitBreaker:
        registerHealthIndicator: true
        slidingWindowSize: 20
        failureRateThreshold: 50
        waitDurationInOpenState: 20000
```

### 📚 Deliverables

- [x] API Gateway setup with Spring Cloud Gateway
- [x] Route configuration for all services
- [x] Rate limiting filter
- [x] Circuit breaker implementation
- [x] Fallback responses
- [x] Health check endpoint
- [x] Tests for rate limiting
- [x] Tests for circuit breaker

### 📖 What You Learn

- API Gateway pattern
- Rate limiting strategies
- Circuit breaker pattern
- Resilience patterns for microservices
- Gateway filter chains

---

## Week 9: Review, Consolidate, Fix Technical Debt

**Goal:** Review all Phase 1 code, improve quality, fix issues

### Tasks

- [ ] Code review each service
- [ ] Fix code style issues
- [ ] Add missing test coverage
- [ ] Update documentation
- [ ] Test complete integration
- [ ] Write Phase 1 summary
- [ ] Plan Phase 2

### 📖 What You Consolidate

- Complete microservices architecture
- Service-to-service communication
- Distributed transaction handling
- Async messaging patterns
- API Gateway with resilience
- JWT-based security

---

# 🔶 PHASE 2: OBSERVABILITY (Weeks 10-17)

## Week 10: Distributed Tracing with Jaeger

Implement request tracing across all services

## Week 11-12: Centralized Logging with ELK Stack

Elasticsearch, Logstash, Kibana for log aggregation

## Week 13-17: Monitoring & Metrics

Prometheus, Grafana, custom dashboards

---

# 🟠 PHASE 3: EVENT-DRIVEN (Weeks 18-34)

## Weeks 18-22: CQRS Pattern

Separate command and query responsibilities

## Weeks 23-28: Event Sourcing

Event store instead of state tables

## Weeks 29-34: Advanced Event Patterns

Projections, snapshots, event versioning

---

# 🟡 PHASE 4: PRODUCTION READY (Weeks 35-52)

## Weeks 35-40: Kubernetes & Scaling

K8s deployment, HPA, cluster setup

## Weeks 41-45: Security & OAuth2

Multi-tenant, role-based access, encryption

## Weeks 46-50: Performance Optimization

Database tuning, caching strategies, CDN

## Weeks 51-52: Capstone Project

ML integration, analytics, multi-region setup

---

**Remember:** Each week, document your learnings in LEARNING_LOG.md!
