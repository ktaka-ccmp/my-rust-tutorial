# FromRequestParts in Axum: Comprehensive Guide

## Overview
`FromRequestParts` is a trait in Axum used to extract data from the request parts (headers, URI, method, etc.) without consuming the request body. It's particularly useful for accessing request metadata while preserving the body for other operations.

## Definition and Core Concepts

```rust
pub trait FromRequestParts<S>: Sized {
    type Rejection: IntoResponse;

    fn from_request_parts(
        parts: &mut Parts,
        state: &S,
    ) -> impl Future<Output = Result<Self, Self::Rejection>> + Send;
}
```

### Key Components
- `parts`: Represents the components of an HTTP request (headers, URI)
- `state`: A shared application state passed to the extractor
- `Rejection`: The type returned when extraction fails, convertible to an HTTP response

## When and How Extractors Are Called

### Automatic Invocation
Extractors in Axum are automatically called when a request is processed. You don't need to explicitly invoke them:

1. Axum calls appropriate extractors for parameters defined in your handler function
2. Extractors are called in order from left to right in the handler's parameter list
3. Only one body-consuming extractor is allowed, and it must be the last parameter

### Requirements
- The type must be an argument in the handler function for the extractor to be called
- The handler's parameter type must implement `FromRequestParts`

## Implementation Examples

### Example 1: Custom Header Extractor

```rust
use axum::{
    extract::FromRequestParts,
    http::{request::Parts, HeaderMap},
    response::IntoResponse,
};
use futures::future::BoxFuture;

struct CustomHeader(String);

#[derive(Debug)]
struct MissingHeader;

impl IntoResponse for MissingHeader {
    fn into_response(self) -> axum::response::Response {
        axum::response::Response::builder()
            .status(400)
            .body("Missing required header".into())
            .unwrap()
    }
}

impl<S> FromRequestParts<S> for CustomHeader {
    type Rejection = MissingHeader;

    fn from_request_parts(
        parts: &mut Parts,
        _state: &S,
    ) -> BoxFuture<'static, Result<Self, Self::Rejection>> {
        let headers = parts.headers.clone();
        Box::pin(async move {
            if let Some(header_value) = headers.get("X-Custom-Header") {
                Ok(CustomHeader(header_value.to_str().unwrap().to_string()))
            } else {
                Err(MissingHeader)
            }
        })
    }
}

// Handler using the custom extractor
async fn handler(custom_header: CustomHeader) -> String {
    format!("Custom Header: {}", custom_header.0)
}
```

### Example 2: User Authentication Extractor

```rust
impl<S> FromRequestParts<S> for User
where
    MemoryStore: FromRef<S>,
    S: Send + Sync,
{
    type Rejection = AuthRedirect;

    async fn from_request_parts(parts: &mut Parts, state: &S) -> Result<Self, Self::Rejection> {
        let store = MemoryStore::from_ref(state);
        
        // Extract cookies from request
        let cookies = parts
            .extract::<TypedHeader<headers::Cookie>>()
            .await
            .map_err(|_| AuthRedirect)?;

        // Get session cookie
        let session_cookie = cookies.get(COOKIE_NAME).ok_or(AuthRedirect)?;

        // Load session from store
        let session = store
            .load_session(session_cookie.to_string())
            .await
            .unwrap()
            .ok_or(AuthRedirect)?;

        // Get user from session
        let user = session.get::<User>("user").ok_or(AuthRedirect)?;

        Ok(user)
    }
}
```

### Example 3: Multiple Extractors

```rust
use axum::{
    extract::State,
    http::{Method, Uri},
    Router,
};

async fn handler(
    method: Method,
    uri: Uri,
    State(state): State<String>,
    CustomHeader(header): CustomHeader,
) -> String {
    format!("Method: {}, URI: {}, State: {}, Header: {}", 
            method, uri, state, header)
}
```

## Comparison: Middleware vs. FromRequestParts

| Feature | Middleware | FromRequestParts |
|---------|------------|------------------|
| Scope | Global (all requests) | Handler-specific |
| Use Case | Logging, security policies, global validation | Extracting specific request data |
| Execution | Before handler | Before handler |
| Response Modification | Yes | No |

## When to Use FromRequestParts

Use `FromRequestParts` when:
1. You only need access to request metadata (headers, URI, method)
2. The request body must remain untouched for other handlers or middleware
3. You want to implement lightweight custom extractors
4. You need handler-specific data extraction
5. You're implementing authentication or session management

Use `FromRequest` instead when:
1. You need to parse the request body
2. You're working with JSON payloads
3. You need access to the complete request

## Common Use Cases
- Session management
- Authentication/Authorization
- Header validation
- Request metadata extraction
- Cookie processing
- URI parameter extraction

## Best Practices
1. Keep extractors focused on a single responsibility
2. Handle rejections gracefully with meaningful error responses
3. Consider performance implications when cloning data
4. Use type-safe approaches when possible
5. Implement proper error handling and rejections
6. Document extractor behavior and requirements

# Appendix A: Advanced Examples

## A.1 Rate Limiting Extractor

```rust
use std::collections::HashMap;
use std::sync::Arc;
use tokio::sync::Mutex;
use std::time::{Duration, Instant};

struct RateLimit;

#[derive(Debug)]
struct RateLimitExceeded;

impl IntoResponse for RateLimitExceeded {
    fn into_response(self) -> Response {
        (
            StatusCode::TOO_MANY_REQUESTS,
            "Rate limit exceeded. Please try again later.",
        )
            .into_response()
    }
}

#[derive(Clone)]
struct RateLimiter {
    requests: Arc<Mutex<HashMap<String, Vec<Instant>>>>,
    max_requests: usize,
    window: Duration,
}

impl<S> FromRequestParts<S> for RateLimit
where
    RateLimiter: FromRef<S>,
    S: Send + Sync,
{
    type Rejection = RateLimitExceeded;

    async fn from_request_parts(
        parts: &mut Parts,
        state: &S,
    ) -> Result<Self, Self::Rejection> {
        let limiter = RateLimiter::from_ref(state);
        let ip = parts
            .headers
            .get("X-Forwarded-For")
            .and_then(|h| h.to_str().ok())
            .unwrap_or("unknown")
            .to_string();

        let now = Instant::now();
        let mut requests = limiter.requests.lock().await;
        
        if let Some(times) = requests.get_mut(&ip) {
            times.retain(|&t| now - t < limiter.window);
            if times.len() >= limiter.max_requests {
                return Err(RateLimitExceeded);
            }
            times.push(now);
        } else {
            requests.insert(ip, vec![now]);
        }
        
        Ok(RateLimit)
    }
}
```

## A.2 Content Type Validator

```rust
struct JsonContentType;

#[derive(Debug)]
struct InvalidContentType;

impl IntoResponse for InvalidContentType {
    fn into_response(self) -> Response {
        (
            StatusCode::UNSUPPORTED_MEDIA_TYPE,
            "Content-Type must be application/json",
        )
            .into_response()
    }
}

impl<S> FromRequestParts<S> for JsonContentType {
    type Rejection = InvalidContentType;

    fn from_request_parts(
        parts: &mut Parts,
        _state: &S,
    ) -> BoxFuture<'static, Result<Self, Self::Rejection>> {
        let content_type = parts
            .headers
            .get(CONTENT_TYPE)
            .and_then(|v| v.to_str().ok())
            .unwrap_or("");

        Box::pin(async move {
            if content_type.starts_with("application/json") {
                Ok(JsonContentType)
            } else {
                Err(InvalidContentType)
            }
        })
    }
}
```

# Appendix B: Authentication Examples

## B.1 Bearer Token Authentication

```rust
struct BearerToken(String);

#[derive(Debug)]
enum AuthError {
    Missing,
    Invalid,
}

impl IntoResponse for AuthError {
    fn into_response(self) -> Response {
        let (status, message) = match self {
            AuthError::Missing => (
                StatusCode::UNAUTHORIZED,
                "Authorization header missing",
            ),
            AuthError::Invalid => (
                StatusCode::UNAUTHORIZED,
                "Invalid bearer token",
            ),
        };
        
        (status, message).into_response()
    }
}

impl<S> FromRequestParts<S> for BearerToken {
    type Rejection = AuthError;

    fn from_request_parts(
        parts: &mut Parts,
        _state: &S,
    ) -> BoxFuture<'static, Result<Self, Self::Rejection>> {
        let auth_header = parts
            .headers
            .get(AUTHORIZATION)
            .and_then(|h| h.to_str().ok())
            .ok_or(AuthError::Missing)?;

        Box::pin(async move {
            if !auth_header.starts_with("Bearer ") {
                return Err(AuthError::Invalid);
            }
            
            let token = auth_header[7..].to_string();
            if token.is_empty() {
                return Err(AuthError::Invalid);
            }
            
            Ok(BearerToken(token))
        })
    }
}
```

## B.2 API Key Validation

```rust
struct ApiKey(String);

#[derive(Debug)]
enum ApiKeyError {
    Missing,
    Invalid,
}

impl<S> FromRequestParts<S> for ApiKey {
    type Rejection = ApiKeyError;

    fn from_request_parts(
        parts: &mut Parts,
        _state: &S,
    ) -> BoxFuture<'static, Result<Self, Self::Rejection>> {
        let headers = parts.headers.clone();
        
        Box::pin(async move {
            let api_key = headers
                .get("X-API-Key")
                .ok_or(ApiKeyError::Missing)?
                .to_str()
                .map_err(|_| ApiKeyError::Invalid)?;
            
            if api_key.len() < 32 {
                return Err(ApiKeyError::Invalid);
            }
            
            Ok(ApiKey(api_key.to_string()))
        })
    }
}
```

# Appendix C: Utility Examples

## C.1 Request ID Tracking

```rust
#[derive(Clone, Debug)]
struct RequestId(Arc<str>);

impl<S> FromRequestParts<S> for RequestId {
    type Rejection = std::convert::Infallible;

    fn from_request_parts(
        parts: &mut Parts,
        _state: &S,
    ) -> BoxFuture<'static, Result<Self, Self::Rejection>> {
        let request_id = parts
            .headers
            .get("X-Request-ID")
            .and_then(|h| h.to_str().ok())
            .map(|s| s.to_string())
            .unwrap_or_else(|| Uuid::new_v4().to_string());

        Box::pin(async move {
            Ok(RequestId(Arc::from(request_id)))
        })
    }
}
```

## C.2 User Agent Parser

```rust
struct UserAgent(String);

impl<S> FromRequestParts<S> for UserAgent {
    type Rejection = std::convert::Infallible;

    fn from_request_parts(
        parts: &mut Parts,
        _state: &S,
    ) -> BoxFuture<'static, Result<Self, Self::Rejection>> {
        let user_agent = parts
            .headers
            .get(header::USER_AGENT)
            .and_then(|h| h.to_str().ok())
            .unwrap_or("unknown")
            .to_string();

        Box::pin(async move {
            Ok(UserAgent(user_agent))
        })
    }
}
```

---

Note: All examples assume appropriate imports from `axum`, `tokio`, and other required crates. Error handling has been simplified for clarity in some cases.
