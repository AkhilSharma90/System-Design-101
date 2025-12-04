---
slug: api-testing
title: API Testing
readTime: 10 min
orderIndex: 4
premium: false
---
# **API Testing: "Do Interfaces Work Correctly?"**

### **What is API Testing?**

API testing verifies that **endpoints return correct responses** for various inputs, focusing on the interface layer.

What API Testing Checks:

✓ Correct HTTP status codes (200, 404, 500\)

✓ Response format (JSON structure)

✓ Response time (performance)

✓ Authentication/Authorization

✓ Error handling

✓ Data validation

### **Real-World Analogy:**

**Like testing a restaurant's ordering window:**

Customer: "One burger, please"

Window: Returns burger \+ receipt ✅

Customer: "Ten burgers"

Window: Returns 10 burgers ✅

Customer: "One dragon" (invalid order)

Window: "Sorry, we don't have that" ✅

Customer: (no payment)

Window: "Payment required" ✅

Test the window interface thoroughly\!

### **API Test Example: User Management API**

Endpoint: GET /api/users/{id}

Test Case 1: Valid User ID
```json
Request: GET /api/users/123
Response:
  Status: 200 OK ✅
  Body: {"id": 123,
  "name": "John Doe",
  "email": "john@example.com"
  }
Headers: Content-Type: application/json ✅

Time: < 200ms ✅
```

 Test Case 2: Invalid User ID

 Request: GET /api/users/999999
 ```json

 Response:

 Status: 404 Not Found ✅

 Body: {    "error": "User not found",    "code": "USER_NOT_FOUND" }

 Time: < 100ms ✅
 ```

 Test Case 3: No Authentication

 Request: GET /api/users/123 (no auth header)

 Response:
 ```json

 Status: 401 Unauthorized ✅

 Body: {    "error": "Authentication required" }


 ```


Test Case 4: Malformed Request

Request: GET /api/users/abc (non-numeric ID)

Response:

Status: 400 Bad Request ✅
```json
Body: { "error": "Invalid user ID format",    "code": "INVALID_FORMAT"  }
```



### **API Testing Types:**

#### **1\. Contract Testing**

Verify API matches documented contract:

Contract says:

 POST /api/users

 ```json
 Body: { "name": string, "email": string }Returns: { "id": number, "name": string, "email": string }
 ```

Test:

✓ Request with correct fields → 201 Created

✓ Response has all required fields

✓ Field types match contract

✗ Request with extra fields → How does API handle?

✗ Request missing required field → 400 Bad Request?

Contract must match reality\!

#### **2\. Security Testing**

Endpoint: DELETE /api/users/{id}

Security Tests:

✓ Without auth token →

401 Unauthorized

✓ With regular user token →

403 Forbidden (can't delete others)

✓ With admin token →

200 OK (allowed)

✓ SQL injection attempt:

DELETE /api/users/1' OR '1' ='1 → Blocked

✓ XSS attempt:

POST /api/users {name: "\<script\>alert('xss')\</script\>"} → Sanitized


✓ Rate limiting: 1000 requests in 1 second →

429 Too Many Requests




#### **3\. Performance Testing**

Endpoint: GET /api/products

Performance Requirements:

- Response time < 200ms (95th percentile)

- Can handle 1000 requests/second

- Response size < 100KB

Tests:

✓ Single request: 150ms ✅

✓ 100 concurrent requests: Average 180ms ✅

✓ 1000 req/sec sustained: Average 195ms ✅

✓ Response compression enabled: 45KB ✅

### **API Testing Tools:**

Popular Tools:

1\. Postman

   \- GUI interface

   \- Collection runner

   \- Environment variables

   \- Automated testing

2\. curl (Command line)

```bash
   curl \-X POST  https://api.example.com/users  \
-H "Content-Type: application/json" \
-d '{"name":"John","email":"john@example.com"}'
```

3\. REST Assured (Java)


```java
   given()
    .auth().oauth2(token)
    .when()
      .get("/api/users/123")
    .then()
      .statusCode(200)
      .body("name", equalTo("John"))
```


4\. Pytest \+ Requests (Python)

```python
  def test_get_user():
    response = requests.get

    ('https://api.example.com/users/123')

    assert response.status_code == 200

    assert response.json()['name'] == 'John'
```

**Key characteristics:**

* 🔌 **Interface focused:** Test the API layer specifically
* 📊 **Multiple scenarios:** Happy path, errors, edge cases
* 🔒 **Security crucial:** Authentication, authorization, injection
* ⚡ **Performance matters:** Response time, throughput
