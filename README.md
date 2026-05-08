# Product Recommendation Platform — Java 25 Microservices

These repositories solves the **Product Recommendation Service** challenge using Java 25, Spring Boot, Spring Security,JUnit, Mockito, Docker, Kubernetes, Spring JPA, MySQL, Spring Cache, Spring Actuator, Resilience4j and Lombok.

---
Repositories:
---

https://github.com/rembertalcantara/user-service

https://github.com/rembertalcantara/product-service

https://github.com/rembertalcantara/recommendation-service

---
## Architecture

```text
Client
  |
  | Basic Auth
  v
recommendation-service :8083
  |-- GET  /recommendations?userId=123&category=electronics
  |-- POST /recommendations/feedback
  |-- MySQL feedback persistence
  |-- calls user-service and product-service concurrently
  |
  | REST + Resilience4j
  +--> user-service :8081
  |      |-- wraps WireMock GET /api/users/{userId}/profile
  |      |-- weekly-ish cache TTL for user preferences
  |
  +--> product-service :8082
         |-- wraps WireMock GET /api/products/category/{category}
         |-- short cache TTL only for catalog response; pricing/inventory are volatile

WireMock :8089 simulates external services.
MySQL :3306 stores feedback.
Actuator exposes health/metrics/prometheus endpoints.
```

## Why this design

- **Microservices**: recommendation-service orchestrates; user-service and product-service encapsulate downstream External API/WireMock integration.
- **Resilience**: downstream calls use Resilience4j retry, timeout, circuit breaker, and fallback.
- **Cache strategy**:
  - User profile: cached longer because profile/preference data changes infrequently.
- **Observability**: actuator, structured logs.
- **Security**: basic authentication enabled for business APIs; actuator health is public.

## Run locally

Requirements:

- JDK 25
- Maven 3.5+
- Docker 
- Kubernetes

```bash
docker compose up --build

kubectl apply -f k8s/k8s.yml
```

### Credentials

```text
username: admin
password: admin123
```

## Test the API

```bash
curl -u admin:admin123 "http://localhost:8080/recommendations?userId=123"

curl -u admin:admin123 "http://localhost:8080/recommendations?userId=123&category=electronics"

curl -u admin:admin123 -X POST "http://localhost:8080/recommendations/feedback" \
  -H "Content-Type: application/json" \
  -d '{
    "userId":"8870324580",
    "productId":"PROD-1234",
    "feedback":"LIKED",
    "comment":"Good recommendation"
  }'
```

## Health and docs

```bash
curl http://localhost:8081/actuator/health
curl http://localhost:8082/actuator/health
curl http://localhost:8083/actuator/health
```

## Performance analysis

The main bottleneck is the User Profile Service latency of 1200–1800ms. The Product Catalog Service is faster but may be called once per preferred category. This implementation mitigates those bottlenecks with:

1. long-lived cache for user profile preferences;
2. circuit breaker and fallback to avoid cascading failures;
3. graceful degradation: if one category fails, other categories can still return recommendations.

## Implementation notes

The external WireMock service is intentionally kept outside the microservices. The user-service and product-service expose internal normalized APIs and protect the recommendation-service from downstream response shape changes.
