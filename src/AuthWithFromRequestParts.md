# User Authentication with Axum: Session Handling and Redirection

This document explains the implementation of user authentication in an Axum-based web application using session cookies. The authentication flow involves extracting user data from session cookies, handling missing or invalid sessions, and redirecting users when necessary.

## Key Concepts

### Axum Framework
[Axum](https://github.com/tokio-rs/axum) is a web framework built on top of [Tokio](https://tokio.rs/) and [Hyper](https://hyper.rs/), designed for building fast, reliable, and scalable APIs. It provides tools to handle requests, responses, routing, and extracting data from requests.

### `AuthRedirect` Struct
The `AuthRedirect` struct is used as a custom rejection type to handle cases where user authentication is missing or invalid. In Axum, when a request cannot be processed (for example, due to a failed authentication), a rejection type is returned.

The `AuthRedirect` struct triggers a temporary redirect response to the root URL (`/`), guiding unauthenticated users back to a login or home page.

### `IntoResponse` Implementation for `AuthRedirect`
The `IntoResponse` trait is implemented for the `AuthRedirect` struct. This allows `AuthRedirect` to be returned as a response, which in this case is a redirect to the root URL:

```rust
impl IntoResponse for AuthRedirect {
    fn into_response(self) -> Response {
        println!("AuthRedirect called.");
        Redirect::temporary("/").into_response()
    }
}
```

When the `AuthRedirect` rejection is triggered, the user is redirected to the root path `/` with a temporary redirect (HTTP 302).

### `FromRequestParts` for `User`
The `FromRequestParts` trait is implemented for extracting a `User` from the request parts. This trait allows the application to pull specific data from the request, such as cookies or headers.

In this implementation, we use `FromRequestParts` to extract the user from the session cookie:

```rust
impl<S> FromRequestParts<S> for User
where
    S: Send + Sync,
{
    type Rejection = AuthRedirect;

    async fn from_request_parts(parts: &mut Parts, _state: &S) -> Result<Self, Self::Rejection> {
        let cookies = parts
            .extract::<TypedHeader<headers::Cookie>>()
            .await
            .map_err(|_| AuthRedirect)?;

        // Get session from cookie
        let session_cookie = cookies
            .get(SESSION_COOKIE_NAME.as_str())
            .ok_or(AuthRedirect)?;
        let store_guard = SESSION_STORE.lock().await;
        let session = store_guard
            .get_store()
            .get(session_cookie)
            .await
            .map_err(|_| AuthRedirect)?;

        // Get user data from session
        let stored_session = session.ok_or(AuthRedirect)?;
        Ok(stored_session.user)
    }
}
```

### Workflow of `FromRequestParts` for `User`
1. Extract cookies from the request using the `TypedHeader<headers::Cookie>` extractor.
2. Attempt to find the session cookie by the predefined `SESSION_COOKIE_NAME`.
3. Look up the session in the `SESSION_STORE`, which is a shared store of session data.
4. If the session is found, return the `user` associated with the session.
5. If any of these steps fail (cookie not found, session not valid, etc.), an `AuthRedirect` rejection is triggered, and the user is redirected.

### `OptionalFromRequestParts` for `User`
The `OptionalFromRequestParts` trait is also implemented for the `User` type. This version attempts to extract the user, but instead of rejecting the request, it returns an `Option<User>`, meaning the user may or may not be present in the request.

```rust
impl<S> OptionalFromRequestParts<S> for User
where
    S: Send + Sync,
{
    type Rejection = Infallible;

    async fn from_request_parts(
        parts: &mut Parts,
        _state: &S,
    ) -> Result<Option<Self>, Self::Rejection> {
        match <User as FromRequestParts<S>>::from_request_parts(parts, _state).await {
            Ok(res) => Ok(Some(res)),
            Err(AuthRedirect) => Ok(None),
        }
    }
}
```

### Workflow of `OptionalFromRequestParts` for `User`
1. This method calls `from_request_parts` (which extracts the user) but returns an `Option<User>` instead of failing outright.
2. If the user is found, it returns `Some(user)`.
3. If the `AuthRedirect` rejection occurs (indicating an unauthenticated user), it returns `None` without triggering an error.

### How It Works Together
- **User Authentication**: The system checks the request for a session cookie and retrieves the associated user if the session is valid.
- **Session Handling**: If a valid session is found, the corresponding user is returned. If the session is invalid or missing, the user is redirected to the root URL (`/`).
- **Flexible User Handling**: The `OptionalFromRequestParts` trait allows for handling requests where the user may or may not be authenticated, making the code flexible for different routes or middleware.

## Conclusion
This implementation demonstrates how to handle user authentication with Axum using session cookies. The system ensures that users are redirected if they are not authenticated while providing a flexible approach for optional authentication in certain parts of the application.

## References
- [Axum Documentation](https://docs.rs/axum/)
- [Tokio Documentation](https://tokio.rs/docs/)
- [Hyper Documentation](https://docs.rs/hyper/)
```
