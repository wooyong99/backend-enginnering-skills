# 외부 시스템 API 연동 Infrastructure 구조 가이드

> **목적**: 일관되고 유지보수 가능한 외부 API 연동 구조  
> **대상**: 백엔드 개발자 및 AI 개발 어시스턴트  
> **원칙**: Port-Adapter 패턴 + 단일 책임 원칙

---

## 📋 목차

1. [Infrastructure 레이어 구조](#infrastructure-레이어-구조)
2. [각 컴포넌트 역할](#각-컴포넌트-역할)
3. [구현 패턴](#구현-패턴)
4. [예외 처리 전략](#예외-처리-전략)
5. [테스트 전략](#테스트-전략)
6. [실전 예제](#실전-예제)

---

## Infrastructure 레이어 구조

### 표준 디렉토리 구조

```
infrastructure/
└── {외부시스템명}/
    ├── adapter/
    │   └── {시스템명}Adapter.java
    ├── apiclient/
    │   ├── {시스템명}ApiClient.java
    │   └── {시스템명}FeignClient.java (선택적)
    ├── properties/
    │   └── {시스템명}Properties.java
    ├── config/
    │   └── {시스템명}Config.java
    ├── constant/
    │   ├── {시스템명}ApiPath.java
    │   └── {시스템명}ErrorCode.java
    ├── dto/
    │   ├── request/
    │   │   └── {Api명}Request.java
    │   └── response/
    │       └── {Api명}Response.java
    ├── mapper/
    │   └── {시스템명}Mapper.java
    └── exception/
        └── {시스템명}Exception.java
```

### 실제 예시 (결제 시스템)

```
infrastructure/
└── payment/
    ├── adapter/
    │   └── PortOnePaymentAdapter.java
    ├── apiclient/
    │   ├── PortOneApiClient.java
    │   └── PortOneFeignClient.java
    ├── properties/
    │   └── PortOneProperties.java
    ├── config/
    │   └── PortOneConfig.java
    ├── constant/
    │   ├── PortOneApiPath.java
    │   └── PortOneErrorCode.java
    ├── dto/
    │   ├── request/
    │   │   ├── PaymentApprovalRequest.java
    │   │   ├── PaymentCancellationRequest.java
    │   │   └── PaymentStatusRequest.java
    │   └── response/
    │       ├── PaymentApprovalResponse.java
    │       ├── PaymentCancellationResponse.java
    │       └── PaymentStatusResponse.java
    ├── mapper/
    │   └── PortOneMapper.java
    └── exception/
        ├── PortOneException.java
        ├── PaymentApprovalFailedException.java
        └── PaymentTimeoutException.java
```

---

## 각 컴포넌트 역할

### 1. Adapter

**역할**: Core Layer의 Port 인터페이스를 구현하여 외부 시스템과 도메인을 연결

**책임**:

- Port 인터페이스 구현
- 도메인 객체 ↔ 외부 API DTO 변환 조율
- 외부 API 호출 조율
- 예외 변환 (외부 예외 → 도메인 예외)

**명명 규칙**: `{시스템명}Adapter.java`

**예시**:

```java
/**
 * PortOne 결제 시스템 어댑터
 *
 * Core Layer의 PaymentProcessor 인터페이스를 구현하여
 * PortOne API와 통신합니다.
 */
@Component
@RequiredArgsConstructor
public class PortOnePaymentAdapter implements PaymentProcessor {

    private final PortOneApiClient apiClient;
    private final PortOneMapper mapper;

    /**
     * 결제를 승인합니다.
     *
     * @param request 도메인 결제 요청
     * @return 결제 승인 결과
     * @throws PaymentApprovalFailedException 결제 승인 실패
     * @throws PaymentTimeoutException 결제 서비스 응답 시간 초과
     */
    @Override
    public PaymentResult approve(PaymentRequest request) {
        try {
            // 1. 도메인 → API DTO 변환
            PaymentApprovalRequest apiRequest = mapper.toApprovalRequest(request);

            // 2. 외부 API 호출
            PaymentApprovalResponse apiResponse = apiClient.approve(apiRequest);

            // 3. API DTO → 도메인 변환
            return mapper.toPaymentResult(apiResponse);

        } catch (PortOneException e) {
            // 4. 예외 변환
            throw new PaymentApprovalFailedException(
                "결제 승인 중 오류가 발생했습니다", e
            );
        } catch (TimeoutException e) {
            throw new PaymentTimeoutException(
                "결제 서비스 응답 시간 초과", e
            );
        }
    }

    @Override
    public PaymentStatus getStatus(String transactionId) {
        try {
            PaymentStatusRequest apiRequest = mapper.toStatusRequest(transactionId);
            PaymentStatusResponse apiResponse = apiClient.getStatus(apiRequest);
            return mapper.toPaymentStatus(apiResponse);

        } catch (PortOneException e) {
            throw new PaymentServiceException(
                "결제 상태 조회 중 오류가 발생했습니다", e
            );
        }
    }
}
```

### 2. ApiClient

**역할**: 실제 HTTP 통신을 담당하는 클라이언트

**책임**:

- HTTP 요청/응답 처리
- 인증 처리 (API Key, OAuth 등)
- 재시도 로직
- 타임아웃 처리
- 로깅

**명명 규칙**: `{시스템명}ApiClient.java`

**구현 방식**:

- RestTemplate 사용
- WebClient 사용
- Feign Client 사용

**RestTemplate 예시**:

```java
/**
 * PortOne API 통신 클라이언트
 */
@Component
@RequiredArgsConstructor
@Slf4j
public class PortOneApiClient {

    private final RestTemplate restTemplate;
    private final PortOneProperties properties;

    /**
     * 결제 승인 API 호출
     */
    public PaymentApprovalResponse approve(PaymentApprovalRequest request) {
        String url = properties.getBaseUrl() + PortOneApiPath.PAYMENT_APPROVAL;

        // 요청 로깅
        log.info("PortOne 결제 승인 요청: {}", request);

        try {
            // HTTP 요청
            HttpHeaders headers = createHeaders();
            HttpEntity<PaymentApprovalRequest> entity =
                new HttpEntity<>(request, headers);

            ResponseEntity<PaymentApprovalResponse> response =
                restTemplate.exchange(
                    url,
                    HttpMethod.POST,
                    entity,
                    PaymentApprovalResponse.class
                );

            // 응답 로깅
            log.info("PortOne 결제 승인 응답: {}", response.getBody());

            return response.getBody();

        } catch (HttpClientErrorException e) {
            // 4xx 에러 처리
            log.error("PortOne API 클라이언트 에러: {}", e.getResponseBodyAsString());
            throw new PortOneClientException(
                "PortOne API 호출 실패: " + e.getStatusCode(),
                e
            );

        } catch (HttpServerErrorException e) {
            // 5xx 에러 처리
            log.error("PortOne API 서버 에러: {}", e.getResponseBodyAsString());
            throw new PortOneServerException(
                "PortOne 서버 오류: " + e.getStatusCode(),
                e
            );

        } catch (ResourceAccessException e) {
            // 타임아웃, 연결 실패 등
            log.error("PortOne API 접근 에러", e);
            throw new PortOneConnectionException(
                "PortOne 서버 연결 실패",
                e
            );
        }
    }

    /**
     * HTTP 헤더 생성
     */
    private HttpHeaders createHeaders() {
        HttpHeaders headers = new HttpHeaders();
        headers.setContentType(MediaType.APPLICATION_JSON);
        headers.set("Authorization", "Bearer " + properties.getApiKey());
        headers.set("X-Request-ID", UUID.randomUUID().toString());
        return headers;
    }
}
```

**Feign Client 예시**:

```java
/**
 * PortOne Feign 클라이언트
 */
@FeignClient(
    name = "portone",
    url = "${portone.base-url}",
    configuration = PortOneFeignConfig.class
)
public interface PortOneFeignClient {

    @PostMapping(PortOneApiPath.PAYMENT_APPROVAL)
    PaymentApprovalResponse approve(
        @RequestHeader("Authorization") String authorization,
        @RequestBody PaymentApprovalRequest request
    );

    @GetMapping(PortOneApiPath.PAYMENT_STATUS)
    PaymentStatusResponse getStatus(
        @RequestHeader("Authorization") String authorization,
        @RequestParam("transactionId") String transactionId
    );

    @PostMapping(PortOneApiPath.PAYMENT_CANCELLATION)
    PaymentCancellationResponse cancel(
        @RequestHeader("Authorization") String authorization,
        @RequestBody PaymentCancellationRequest request
    );
}
```

### 3. Properties

**역할**: 외부 시스템 연동을 위한 설정 값 관리

**책임**:

- API 엔드포인트 URL
- API 키, 시크릿 등 인증 정보
- 타임아웃 설정
- 재시도 설정

**명명 규칙**: `{시스템명}Properties.java`

**예시**:

```java
/**
 * PortOne API 설정
 */
@Configuration
@ConfigurationProperties(prefix = "portone")
@Validated
@Getter
@Setter
public class PortOneProperties {

    /**
     * API 기본 URL
     */
    @NotBlank
    private String baseUrl;

    /**
     * API 키
     */
    @NotBlank
    private String apiKey;

    /**
     * API 시크릿
     */
    @NotBlank
    private String apiSecret;

    /**
     * 연결 타임아웃 (밀리초)
     */
    @Min(1000)
    private int connectTimeout = 5000;

    /**
     * 읽기 타임아웃 (밀리초)
     */
    @Min(1000)
    private int readTimeout = 10000;

    /**
     * 재시도 최대 횟수
     */
    @Min(0)
    @Max(5)
    private int maxRetries = 3;

    /**
     * 재시도 간격 (밀리초)
     */
    @Min(100)
    private int retryInterval = 1000;
}
```

**application.yml**:

```yaml
portone:
  base-url: https://api.portone.io
  api-key: ${PORTONE_API_KEY}
  api-secret: ${PORTONE_API_SECRET}
  connect-timeout: 5000
  read-timeout: 10000
  max-retries: 3
  retry-interval: 1000
```

### 4. Config

**역할**: 외부 시스템 연동을 위한 Bean 설정

**책임**:

- RestTemplate/WebClient 생성 및 설정
- Interceptor 등록
- 타임아웃 설정
- 에러 핸들러 설정

**명명 규칙**: `{시스템명}Config.java`

**예시**:

```java
/**
 * PortOne API 설정
 */
@Configuration
@RequiredArgsConstructor
public class PortOneConfig {

    private final PortOneProperties properties;

    /**
     * PortOne 전용 RestTemplate
     */
    @Bean
    public RestTemplate portOneRestTemplate() {
        // Connection Factory 설정
        HttpComponentsClientHttpRequestFactory factory =
            new HttpComponentsClientHttpRequestFactory();
        factory.setConnectTimeout(properties.getConnectTimeout());
        factory.setReadTimeout(properties.getReadTimeout());

        // RestTemplate 생성
        RestTemplate restTemplate = new RestTemplate(factory);

        // Interceptor 등록
        restTemplate.setInterceptors(List.of(
            new PortOneLoggingInterceptor(),
            new PortOneAuthInterceptor(properties)
        ));

        // Error Handler 등록
        restTemplate.setErrorHandler(new PortOneErrorHandler());

        return restTemplate;
    }

    /**
     * 재시도 템플릿
     */
    @Bean
    public RetryTemplate portOneRetryTemplate() {
        return RetryTemplate.builder()
            .maxAttempts(properties.getMaxRetries())
            .fixedBackoff(properties.getRetryInterval())
            .retryOn(PortOneRetryableException.class)
            .build();
    }
}
```

**Feign Config 예시**:

```java
/**
 * PortOne Feign 설정
 */
@Configuration
public class PortOneFeignConfig {

    @Bean
    public RequestInterceptor portOneRequestInterceptor(
            PortOneProperties properties) {
        return template -> {
            template.header("Authorization", "Bearer " + properties.getApiKey());
            template.header("X-Request-ID", UUID.randomUUID().toString());
        };
    }

    @Bean
    public ErrorDecoder portOneErrorDecoder() {
        return (methodKey, response) -> {
            if (response.status() >= 400 && response.status() < 500) {
                return new PortOneClientException(
                    "PortOne API 클라이언트 에러: " + response.status()
                );
            }
            if (response.status() >= 500) {
                return new PortOneServerException(
                    "PortOne API 서버 에러: " + response.status()
                );
            }
            return new PortOneException("PortOne API 에러");
        };
    }

    @Bean
    public Retryer portOneRetryer(PortOneProperties properties) {
        return new Retryer.Default(
            properties.getRetryInterval(),
            properties.getRetryInterval() * 5,
            properties.getMaxRetries()
        );
    }
}
```

### 5. Constant

**역할**: API 관련 상수 정의

**책임**:

- API 경로 정의
- 에러 코드 정의
- 고정된 값 정의

**명명 규칙**:

- `{시스템명}ApiPath.java`
- `{시스템명}ErrorCode.java`

**ApiPath 예시**:

```java
/**
 * PortOne API 경로
 */
public final class PortOneApiPath {

    private PortOneApiPath() {
        throw new AssertionError("인스턴스 생성 불가");
    }

    // Base
    public static final String API_V1 = "/api/v1";

    // Payment
    public static final String PAYMENT = API_V1 + "/payments";
    public static final String PAYMENT_APPROVAL = PAYMENT + "/approve";
    public static final String PAYMENT_CANCELLATION = PAYMENT + "/cancel";
    public static final String PAYMENT_STATUS = PAYMENT + "/status";

    // Refund
    public static final String REFUND = API_V1 + "/refunds";
    public static final String REFUND_REQUEST = REFUND + "/request";
    public static final String REFUND_STATUS = REFUND + "/status";
}
```

**ErrorCode 예시**:

```java
/**
 * PortOne 에러 코드
 */
@Getter
@RequiredArgsConstructor
public enum PortOneErrorCode {

    // 인증 에러
    INVALID_API_KEY("AUTH001", "유효하지 않은 API 키"),
    EXPIRED_API_KEY("AUTH002", "만료된 API 키"),

    // 결제 에러
    PAYMENT_DECLINED("PAY001", "결제 거부"),
    INSUFFICIENT_BALANCE("PAY002", "잔액 부족"),
    INVALID_CARD("PAY003", "유효하지 않은 카드"),

    // 시스템 에러
    INTERNAL_ERROR("SYS001", "내부 서버 오류"),
    SERVICE_UNAVAILABLE("SYS002", "서비스 이용 불가"),
    TIMEOUT("SYS003", "요청 시간 초과"),

    // 기타
    UNKNOWN_ERROR("ERR999", "알 수 없는 오류");

    private final String code;
    private final String message;

    public static PortOneErrorCode fromCode(String code) {
        return Arrays.stream(values())
            .filter(ec -> ec.code.equals(code))
            .findFirst()
            .orElse(UNKNOWN_ERROR);
    }
}
```

### 6. DTO (Request/Response)

**역할**: 외부 API와 주고받는 데이터 구조 정의

**책임**:

- API 스펙에 맞는 필드 정의
- 직렬화/역직렬화
- 기본 검증

**명명 규칙**:

- `{Api명}Request.java`
- `{Api명}Response.java`

**Request 예시**:

```java
/**
 * 결제 승인 요청
 */
@Getter
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class PaymentApprovalRequest {

    /**
     * 주문 ID
     */
    @JsonProperty("order_id")
    @NotBlank
    private String orderId;

    /**
     * 결제 금액
     */
    @JsonProperty("amount")
    @NotNull
    @Min(0)
    private Long amount;

    /**
     * 통화
     */
    @JsonProperty("currency")
    @NotBlank
    private String currency;

    /**
     * 결제 수단
     */
    @JsonProperty("payment_method")
    @NotBlank
    private String paymentMethod;

    /**
     * 카드 정보
     */
    @JsonProperty("card_info")
    private CardInfo cardInfo;

    /**
     * 고객 정보
     */
    @JsonProperty("customer_info")
    @NotNull
    private CustomerInfo customerInfo;

    @Getter
    @Builder
    @NoArgsConstructor
    @AllArgsConstructor
    public static class CardInfo {
        @JsonProperty("card_number")
        private String cardNumber;

        @JsonProperty("expiry_month")
        private String expiryMonth;

        @JsonProperty("expiry_year")
        private String expiryYear;

        @JsonProperty("cvv")
        private String cvv;
    }

    @Getter
    @Builder
    @NoArgsConstructor
    @AllArgsConstructor
    public static class CustomerInfo {
        @JsonProperty("name")
        private String name;

        @JsonProperty("email")
        private String email;

        @JsonProperty("phone")
        private String phone;
    }
}
```

**Response 예시**:

```java
/**
 * 결제 승인 응답
 */
@Getter
@NoArgsConstructor
@AllArgsConstructor
public class PaymentApprovalResponse {

    /**
     * 거래 ID
     */
    @JsonProperty("transaction_id")
    private String transactionId;

    /**
     * 주문 ID
     */
    @JsonProperty("order_id")
    private String orderId;

    /**
     * 결제 상태
     */
    @JsonProperty("status")
    private String status;

    /**
     * 승인 번호
     */
    @JsonProperty("approval_number")
    private String approvalNumber;

    /**
     * 승인 시각
     */
    @JsonProperty("approved_at")
    private LocalDateTime approvedAt;

    /**
     * 결제 금액
     */
    @JsonProperty("amount")
    private Long amount;

    /**
     * 에러 코드 (실패 시)
     */
    @JsonProperty("error_code")
    private String errorCode;

    /**
     * 에러 메시지 (실패 시)
     */
    @JsonProperty("error_message")
    private String errorMessage;
}
```

### 7. Mapper

**역할**: 도메인 객체 ↔ API DTO 변환

**책임**:

- 도메인 → Request DTO 변환
- Response DTO → 도메인 변환
- 필드 매핑
- 타입 변환

**명명 규칙**: `{시스템명}Mapper.java`

**예시**:

```java
/**
 * PortOne 객체 변환기
 */
@Component
public class PortOneMapper {

    /**
     * 도메인 결제 요청 → API 승인 요청 변환
     */
    public PaymentApprovalRequest toApprovalRequest(PaymentRequest domain) {
        return PaymentApprovalRequest.builder()
            .orderId(domain.getOrderId().getValue())
            .amount(domain.getAmount().getAmount().longValue())
            .currency(domain.getAmount().getCurrency().getCurrencyCode())
            .paymentMethod(mapPaymentMethod(domain.getPaymentMethod()))
            .cardInfo(mapCardInfo(domain.getCardInfo()))
            .customerInfo(mapCustomerInfo(domain.getCustomer()))
            .build();
    }

    /**
     * API 승인 응답 → 도메인 결제 결과 변환
     */
    public PaymentResult toPaymentResult(PaymentApprovalResponse api) {
        if ("APPROVED".equals(api.getStatus())) {
            return PaymentResult.success(
                PaymentId.of(api.getTransactionId()),
                Money.of(
                    BigDecimal.valueOf(api.getAmount()),
                    Currency.getInstance("KRW")
                ),
                api.getApprovalNumber(),
                api.getApprovedAt()
            );
        } else {
            return PaymentResult.failed(
                PortOneErrorCode.fromCode(api.getErrorCode()),
                api.getErrorMessage()
            );
        }
    }

    /**
     * 도메인 → API 상태 요청 변환
     */
    public PaymentStatusRequest toStatusRequest(String transactionId) {
        return PaymentStatusRequest.builder()
            .transactionId(transactionId)
            .build();
    }

    /**
     * API 상태 응답 → 도메인 상태 변환
     */
    public PaymentStatus toPaymentStatus(PaymentStatusResponse api) {
        return switch (api.getStatus()) {
            case "PENDING" -> PaymentStatus.PENDING;
            case "APPROVED" -> PaymentStatus.APPROVED;
            case "DECLINED" -> PaymentStatus.DECLINED;
            case "CANCELLED" -> PaymentStatus.CANCELLED;
            default -> PaymentStatus.UNKNOWN;
        };
    }

    // Private helper methods

    private String mapPaymentMethod(PaymentMethod domain) {
        return switch (domain) {
            case CREDIT_CARD -> "CARD";
            case BANK_TRANSFER -> "BANK";
            case VIRTUAL_ACCOUNT -> "VACCOUNT";
            default -> throw new IllegalArgumentException(
                "지원하지 않는 결제 수단: " + domain
            );
        };
    }

    private PaymentApprovalRequest.CardInfo mapCardInfo(CardInfo domain) {
        if (domain == null) {
            return null;
        }

        return PaymentApprovalRequest.CardInfo.builder()
            .cardNumber(domain.getNumber())
            .expiryMonth(domain.getExpiryMonth())
            .expiryYear(domain.getExpiryYear())
            .cvv(domain.getCvv())
            .build();
    }

    private PaymentApprovalRequest.CustomerInfo mapCustomerInfo(Customer domain) {
        return PaymentApprovalRequest.CustomerInfo.builder()
            .name(domain.getName())
            .email(domain.getEmail().getValue())
            .phone(domain.getPhone().getValue())
            .build();
    }
}
```

### 8. Exception

**역할**: 외부 시스템 관련 예외 정의

**책임**:

- 외부 시스템 에러를 도메인 예외로 변환
- 에러 메시지 관리
- 에러 코드 관리

**명명 규칙**: `{시스템명}Exception.java`

**예시**:

```java
/**
 * PortOne 관련 예외 Base
 */
public class PortOneException extends RuntimeException {

    private final String errorCode;

    public PortOneException(String message) {
        super(message);
        this.errorCode = null;
    }

    public PortOneException(String message, Throwable cause) {
        super(message, cause);
        this.errorCode = null;
    }

    public PortOneException(String errorCode, String message) {
        super(message);
        this.errorCode = errorCode;
    }

    public PortOneException(String errorCode, String message, Throwable cause) {
        super(message, cause);
        this.errorCode = errorCode;
    }

    public String getErrorCode() {
        return errorCode;
    }
}

/**
 * PortOne API 클라이언트 에러 (4xx)
 */
public class PortOneClientException extends PortOneException {
    public PortOneClientException(String message) {
        super(message);
    }

    public PortOneClientException(String message, Throwable cause) {
        super(message, cause);
    }
}

/**
 * PortOne API 서버 에러 (5xx)
 */
public class PortOneServerException extends PortOneException {
    public PortOneServerException(String message) {
        super(message);
    }

    public PortOneServerException(String message, Throwable cause) {
        super(message, cause);
    }
}

/**
 * 결제 승인 실패
 */
public class PaymentApprovalFailedException extends PortOneException {
    public PaymentApprovalFailedException(String message) {
        super(message);
    }

    public PaymentApprovalFailedException(String message, Throwable cause) {
        super(message, cause);
    }

    public PaymentApprovalFailedException(String errorCode, String message) {
        super(errorCode, message);
    }
}

/**
 * 결제 서비스 타임아웃
 */
public class PaymentTimeoutException extends PortOneException {
    public PaymentTimeoutException(String message) {
        super(message);
    }

    public PaymentTimeoutException(String message, Throwable cause) {
        super(message, cause);
    }
}
```

---

## 구현 패턴

### 전체 흐름

```
1. Core Layer (Use Case)
   ↓ Port 인터페이스 호출

2. Adapter
   ↓ 도메인 → Request 변환 (Mapper)

3. ApiClient
   ↓ HTTP 요청

4. External API
   ↓ HTTP 응답

5. ApiClient
   ↓ 예외 처리

6. Adapter
   ↓ Response → 도메인 변환 (Mapper)
   ↓ 예외 변환

7. Core Layer (Use Case)
   ↓ 비즈니스 로직 처리
```

### 코드 흐름 예시

```java
// 1. Core Layer - Use Case
@UseCase
@RequiredArgsConstructor
public class ProcessPaymentUseCase {

    private final PaymentProcessor paymentProcessor;  // Port

    @Transactional
    public PaymentResult execute(ProcessPaymentCommand command) {
        // 도메인 객체 생성
        PaymentRequest request = PaymentRequest.create(command);

        // Port 호출 (실제로는 Adapter 호출됨)
        PaymentResult result = paymentProcessor.approve(request);

        return result;
    }
}

// 2. Infrastructure - Adapter
@Component
@RequiredArgsConstructor
public class PortOnePaymentAdapter implements PaymentProcessor {

    private final PortOneApiClient apiClient;
    private final PortOneMapper mapper;

    @Override
    public PaymentResult approve(PaymentRequest request) {
        // 도메인 → API DTO 변환
        PaymentApprovalRequest apiRequest = mapper.toApprovalRequest(request);

        // API 호출
        PaymentApprovalResponse apiResponse = apiClient.approve(apiRequest);

        // API DTO → 도메인 변환
        return mapper.toPaymentResult(apiResponse);
    }
}

// 3. Infrastructure - ApiClient
@Component
@RequiredArgsConstructor
public class PortOneApiClient {

    private final RestTemplate restTemplate;
    private final PortOneProperties properties;

    public PaymentApprovalResponse approve(PaymentApprovalRequest request) {
        String url = properties.getBaseUrl() + PortOneApiPath.PAYMENT_APPROVAL;

        HttpHeaders headers = createHeaders();
        HttpEntity<PaymentApprovalRequest> entity =
            new HttpEntity<>(request, headers);

        ResponseEntity<PaymentApprovalResponse> response =
            restTemplate.exchange(
                url,
                HttpMethod.POST,
                entity,
                PaymentApprovalResponse.class
            );

        return response.getBody();
    }
}
```

---

## 예외 처리 전략

### 예외 계층

```
Exception
└── RuntimeException
    └── PortOneException (Infrastructure)
        ├── PortOneClientException (4xx)
        ├── PortOneServerException (5xx)
        ├── PortOneConnectionException (연결 실패)
        └── PaymentApprovalFailedException (비즈니스 에러)
```

### 예외 변환 패턴

```java
@Component
public class PortOneApiClient {

    public PaymentApprovalResponse approve(PaymentApprovalRequest request) {
        try {
            return callApi(request);

        } catch (HttpClientErrorException e) {
            // 4xx → 클라이언트 에러
            throw new PortOneClientException(
                "PortOne API 호출 실패: " + e.getStatusCode(),
                e
            );

        } catch (HttpServerErrorException e) {
            // 5xx → 서버 에러
            throw new PortOneServerException(
                "PortOne 서버 오류: " + e.getStatusCode(),
                e
            );

        } catch (ResourceAccessException e) {
            // 타임아웃, 연결 실패
            throw new PortOneConnectionException(
                "PortOne 서버 연결 실패",
                e
            );
        }
    }
}
```

---

## 테스트 전략

### 1. Adapter 테스트 (단위 테스트)

```java
class PortOnePaymentAdapterTest {

    private PortOnePaymentAdapter adapter;
    private PortOneApiClient apiClient;
    private PortOneMapper mapper;

    @BeforeEach
    void setUp() {
        apiClient = mock(PortOneApiClient.class);
        mapper = new PortOneMapper();
        adapter = new PortOnePaymentAdapter(apiClient, mapper);
    }

    @Test
    void approve_success() {
        // Given
        PaymentRequest request = PaymentRequestFixture.create();
        PaymentApprovalResponse apiResponse = PaymentApprovalResponseFixture.success();

        when(apiClient.approve(any())).thenReturn(apiResponse);

        // When
        PaymentResult result = adapter.approve(request);

        // Then
        assertThat(result.isSuccess()).isTrue();
        assertThat(result.getTransactionId()).isNotNull();
    }
}
```

### 2. ApiClient 테스트 (통합 테스트 - WireMock)

```java
@SpringBootTest
class PortOneApiClientIntegrationTest {

    @Autowired
    private PortOneApiClient apiClient;

    private WireMockServer wireMockServer;

    @BeforeEach
    void setUp() {
        wireMockServer = new WireMockServer(8089);
        wireMockServer.start();
        configureFor("localhost", 8089);
    }

    @AfterEach
    void tearDown() {
        wireMockServer.stop();
    }

    @Test
    void approve_success() {
        // Given
        stubFor(post(urlEqualTo("/api/v1/payments/approve"))
            .willReturn(aResponse()
                .withStatus(200)
                .withHeader("Content-Type", "application/json")
                .withBody("""
                    {
                        "transaction_id": "txn-123",
                        "status": "APPROVED"
                    }
                    """)));

        PaymentApprovalRequest request = PaymentApprovalRequestFixture.create();

        // When
        PaymentApprovalResponse response = apiClient.approve(request);

        // Then
        assertThat(response.getTransactionId()).isEqualTo("txn-123");
        assertThat(response.getStatus()).isEqualTo("APPROVED");
    }
}
```

---

## 실전 예제

### 완전한 외부 시스템 연동 예제

**디렉토리 구조**:

```
infrastructure/
└── portone/
    ├── adapter/
    │   └── PortOnePaymentAdapter.java
    ├── apiclient/
    │   └── PortOneApiClient.java
    ├── properties/
    │   └── PortOneProperties.java
    ├── config/
    │   └── PortOneConfig.java
    ├── constant/
    │   ├── PortOneApiPath.java
    │   └── PortOneErrorCode.java
    ├── dto/
    │   ├── request/
    │   │   └── PaymentApprovalRequest.java
    │   └── response/
    │       └── PaymentApprovalResponse.java
    ├── mapper/
    │   └── PortOneMapper.java
    └── exception/
        ├── PortOneException.java
        └── PaymentApprovalFailedException.java
```

이 구조를 따르면 일관되고 유지보수 가능한 외부 시스템 연동을 구현할 수 있습니다.

---

## 체크리스트

### 신규 외부 시스템 연동 시

- [ ] infrastructure/{시스템명} 디렉토리 생성
- [ ] adapter/ 디렉토리 및 Adapter 클래스 생성
- [ ] apiclient/ 디렉토리 및 ApiClient 클래스 생성
- [ ] properties/ 디렉토리 및 Properties 클래스 생성
- [ ] config/ 디렉토리 및 Config 클래스 생성
- [ ] constant/ 디렉토리 및 상수 클래스 생성
- [ ] dto/request, dto/response 디렉토리 및 DTO 클래스 생성
- [ ] mapper/ 디렉토리 및 Mapper 클래스 생성
- [ ] exception/ 디렉토리 및 예외 클래스 생성

### 구현 검증

- [ ] Adapter가 Port 인터페이스를 구현하는가?
- [ ] ApiClient가 HTTP 통신만 담당하는가?
- [ ] Mapper가 변환 로직만 담당하는가?
- [ ] 예외가 적절히 변환되는가?
- [ ] 테스트 코드가 작성되었는가?

---

이 가이드는 외부 시스템 API 연동 시 일관된 구조를 유지하기 위한 실용적인 지침을 제공합니다.
