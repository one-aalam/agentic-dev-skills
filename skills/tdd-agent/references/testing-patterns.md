# Testing Patterns Reference

Stack-specific test patterns for the tdd-agent skill.
Load the relevant section based on the project stack detected in CONTEXT.md.

---

## TypeScript + Jest

### File naming
- Source: `src/auth/middleware.ts`
- Test: `src/auth/middleware.test.ts` (colocated) or `tests/auth/middleware.test.ts`

### Test structure
```typescript
import { describe, it, expect, beforeEach, afterEach, jest } from '@jest/globals';
import { <moduleUnderTest> } from './<module>';

describe('<module name>', () => {
  let <dependencies>;

  beforeEach(() => {
    <setup — create fresh instances, reset mocks>
    jest.clearAllMocks();
  });

  afterEach(() => {
    <teardown — close connections, restore mocks>
  });

  describe('<behaviour group>', () => {
    it('<sentence describing the behaviour>', async () => {
      // Arrange
      const input = <value>;
      const expected = <value>;

      // Act
      const result = await <functionUnderTest>(input);

      // Assert
      expect(result).toEqual(expected);
    });

    it('throws when <error condition>', async () => {
      await expect(<functionUnderTest>(invalidInput))
        .rejects.toThrow('<error message>');
    });
  });
});
```

### Mocking external dependencies
```typescript
jest.mock('../external-service', () => ({
  callExternalApi: jest.fn(),
}));

import { callExternalApi } from '../external-service';
const mockCallExternalApi = callExternalApi as jest.MockedFunction<typeof callExternalApi>;

it('handles external service failure', async () => {
  mockCallExternalApi.mockRejectedValueOnce(new Error('Service unavailable'));
  await expect(sut.doThing()).rejects.toThrow('Service unavailable');
});
```

### Testing HTTP endpoints (supertest)
```typescript
import request from 'supertest';
import { app } from '../app';

it('returns 401 for unauthenticated request', async () => {
  const response = await request(app)
    .get('/api/protected')
    .set('Content-Type', 'application/json');

  expect(response.status).toBe(401);
  expect(response.body).toMatchObject({ error: 'Unauthorized' });
});
```

### Run commands
```bash
npx jest
npx jest src/auth/middleware.test.ts
npx jest --coverage --testPathPattern src/auth/
npx jest --watch
```

---

## Python + pytest

### File naming
- Source: `src/auth/middleware.py`
- Test: `tests/auth/test_middleware.py`

### Test structure
```python
import pytest
from unittest.mock import MagicMock, patch
from src.auth.middleware import <function_under_test>

class TestMiddleware:
    @pytest.fixture(autouse=True)
    def setup(self):
        self.dependency = MagicMock()
        yield

    def test_authenticates_valid_token(self):
        token = "valid-jwt-token"
        expected_user_id = "user-123"
        result = authenticate_request(token, self.dependency)
        assert result.user_id == expected_user_id

    def test_raises_for_expired_token(self):
        with pytest.raises(AuthenticationError, match="Token expired"):
            authenticate_request("expired-token", self.dependency)

    @pytest.mark.parametrize("invalid_token", [
        None, "", "malformed", "a.b",
    ])
    def test_raises_for_invalid_token_formats(self, invalid_token):
        with pytest.raises(AuthenticationError):
            authenticate_request(invalid_token, self.dependency)
```

### Fixtures
```python
# conftest.py
import pytest
from src.app import create_app
from src.db import get_test_db

@pytest.fixture(scope="session")
def app():
    return create_app(testing=True)

@pytest.fixture
def client(app):
    return app.test_client()

@pytest.fixture
def db():
    database = get_test_db()
    yield database
    database.rollback()
    database.close()
```

### Run commands
```bash
pytest
pytest tests/auth/test_middleware.py
pytest --cov=src --cov-report=term-missing tests/
pytest tests/auth/test_middleware.py::TestMiddleware::test_authenticates_valid_token -v
pytest -x
```

---

## Go standard testing

### File naming
- Source: `internal/auth/middleware.go`
- Test: `internal/auth/middleware_test.go` (same package, colocated)

### Test structure
```go
package auth_test

import (
    "testing"
    "github.com/<org>/<repo>/internal/auth"
)

func TestAuthenticateRequest_ValidToken(t *testing.T) {
    token := "valid-jwt-token"
    want := "user-123"
    got, err := auth.AuthenticateRequest(token)
    if err != nil {
        t.Fatalf("unexpected error: %v", err)
    }
    if got.UserID != want {
        t.Errorf("got UserID %q, want %q", got.UserID, want)
    }
}

// Table-driven test
func TestAuthenticateRequest_InvalidFormats(t *testing.T) {
    cases := []struct{ name, token string }{
        {"empty string", ""},
        {"malformed", "malformed"},
        {"missing signature", "a.b"},
    }
    for _, tc := range cases {
        t.Run(tc.name, func(t *testing.T) {
            _, err := auth.AuthenticateRequest(tc.token)
            if err == nil {
                t.Errorf("expected error for token %q, got nil", tc.token)
            }
        })
    }
}
```

### Run commands
```bash
go test ./...
go test ./internal/auth/...
go test -coverprofile=coverage.out ./...
go tool cover -func=coverage.out
go test -v ./internal/auth/...
go test -run TestAuthenticateRequest_ValidToken ./internal/auth/
```

---

## Rust + cargo test

### File naming
- Unit: inline `#[cfg(test)]` module in `src/auth/middleware.rs`
- Integration: `tests/auth_integration.rs`

### Test structure
```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn authenticates_valid_token() {
        let token = "valid-jwt-token";
        let result = authenticate_request(token);
        assert!(result.is_ok());
        assert_eq!(result.unwrap().user_id, "user-123");
    }

    #[test]
    fn returns_error_for_expired_token() {
        let result = authenticate_request("expired-token");
        assert!(result.is_err());
        assert!(matches!(result.unwrap_err(), AuthError::TokenExpired));
    }
}
```

### Run commands
```bash
cargo test
cargo test authenticates_valid_token
cargo test -- --nocapture
cargo tarpaulin --out Html
```

---

## Common Anti-Patterns to Avoid

| Anti-pattern | Why bad | Instead |
|---|---|---|
| `expect(spy).toHaveBeenCalled()` as the only assertion | Tests implementation, not behaviour | Assert on observable output/effect |
| Modifying test assertions to make them pass | Hides bugs | Fix the implementation |
| `beforeAll` with database mutations | Tests share state, fail randomly | Use `beforeEach` with rollback |
| Sleeping/waiting for time in tests | Brittle, slow | Use fake timers or dependency injection |
| Testing private functions directly | Brittle to refactors | Test through public interface |
| `catch` blocks that swallow errors silently | Hides failures | Re-throw or assert on the error |
| Single massive test that tests everything | Hard to debug failures | One concept per test |
