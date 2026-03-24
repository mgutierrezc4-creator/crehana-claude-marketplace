---
name: new-adapter
description: >
  Generates a complete, production-ready adapter for the centralized-users FastAPI microservice.
  Use this skill whenever the user wants to create a new adapter, says "crea un adapter",
  "agrega un adapter", "nuevo adapter para X", "necesito llamar a Learning/Talent para X",
  "necesito integrar con la API de X", or anything about calling an external service
  (Learning GraphQL API, Talent REST API, or any other HTTP integration).
  Also trigger for "implementa la llamada a X", "conecta con el servicio Y", or when
  the user needs to add an outbound HTTP call to Learning or Talent.
  Always use this skill — manually written adapters in this project frequently miss the
  dual try-except pattern, use wrong HTTP clients, or skip the Pydantic response model.
---

# New Adapter — centralized-users

This skill defines how to create adapters in this project. Follow these conventions
precisely — do not invent alternative patterns.

## Quick Orientation: What is an Adapter?

Adapters are the **outbound HTTP layer** — they encapsulate calls to external services
(Learning, Talent). They are never called directly from mutations or queries; they are
called from use cases or services.

```
apps/users/adapters/
├── learning/
│   ├── graphql/           ← .graphql query/mutation files for Learning API
│   └── <action>.py        ← Learning adapter (GraphQL over HTTP)
└── talent/
    └── <action>.py        ← Talent adapter (REST over HTTP)
```

---

## Choose the Right Pattern

There are **3 patterns** in this codebase. Choose based on target service:

| Target | Protocol | Pattern to use |
|--------|----------|----------------|
| Learning API | GraphQL | Learning pattern (GraphQLClient + JWT) |
| Talent API | REST | `APIAdapter` subclass (preferred) |
| Talent API (legacy) | REST | Manual httpx pattern (only when complex logic needed) |

---

## Pattern 1: Learning Adapter (GraphQL + JWT)

Use for any call to the Learning GraphQL API (`CREHANA_API_V4_B2B_URL` or `CREHANA_API_V2_B2B_URL`).

### File placement
```
apps/users/adapters/learning/<action>.py
apps/users/adapters/learning/graphql/<action>.graphql
```

### Skeleton

```python
# apps/users/adapters/learning/<action>.py
import logging

from pydantic import BaseModel, Extra, Field

from crehana_centralized_users.constants import CREHANA_JWT_COOKIE_NAME
from crehana_centralized_users.external_services.graphql import GraphQLClient
from crehana_centralized_users.settings.base import settings
from crehana_centralized_users.utils.generate_learning_jwt import generate_learning_jwt

logger = logging.getLogger(__name__)


# --- Response models (private, prefix with _) ---

class _SomeNestedModel(BaseModel):
    id: int
    some_field: str = Field(alias="someField")


class <Action>LearningResponse(BaseModel, extra=Extra.allow):
    success: bool | None = None
    error: str | None = None
    # Add response fields here — use camelCase aliases for API fields


# --- Adapter class ---

class <Action>LearningAdapter:
    def __init__(self):
        self.path = ""

    def execute(
        self,
        organization_slug: str,
        admin_learning_user_id: int,
        admin_learning_user_email: str,
        # ... additional params
    ) -> <Action>LearningResponse:
        crehana_token = generate_learning_jwt(
            user_id=admin_learning_user_id, email=admin_learning_user_email
        )
        client = GraphQLClient(
            url=settings.CREHANA_API_V4_B2B_URL,  # or V2 depending on the endpoint
            cookies={CREHANA_JWT_COOKIE_NAME: crehana_token},
        )

        try:
            response = client.execute_query_from_file(
                filename="<action>.graphql",
                variables={
                    "slug": organization_slug,
                    # ... other variables
                },
            )
            logger.debug(f"Response: {response}")
        except Exception as e:
            logger.exception(e)
            raise Exception("Unexpected error calling <action> in learning") from e

        try:
            parsed_response = <Action>LearningResponse.parse_obj(
                response["<graphqlOperationName>"]
            )
        except Exception as e:
            logger.exception(e)
            raise Exception("Error at parsing response from learning") from e

        return parsed_response
```

### GraphQL file pattern

```graphql
# apps/users/adapters/learning/graphql/<action>.graphql
mutation <ActionName>($slug: String!, $userId: Int!) {
  <graphqlOperationName>(slug: $slug, userId: $userId) {
    success
    error
    # ... fields matching the response model
  }
}
```

**Key rules for Learning adapters:**
- `execute()` is **synchronous** (no `async def`) — `GraphQLClient` uses `requests` internally
- Always pass `admin_learning_user_id` + `admin_learning_user_email` to `generate_learning_jwt`
- API version: use `V4` for modern endpoints, `V2` for legacy ones (check existing adapters for the right version)
- The `.graphql` filename must match what you pass to `execute_query_from_file`
- `response["<graphqlOperationName>"]` — the key must match the GraphQL operation name exactly

---

## Pattern 2: Talent Adapter via `APIAdapter` (preferred)

Use for REST calls to Talent API when the logic is straightforward (most cases).

### File placement
```
apps/users/adapters/talent/<action>.py
```

### Skeleton

```python
# apps/users/adapters/talent/<action>.py
import logging
from typing import Any

from pydantic import BaseModel, Extra, Field

from crehana_centralized_users.utils.adapter import APIAdapter

logger = logging.getLogger(__name__)


class <Action>TalentResponse(BaseModel, extra=Extra.allow):
    # Define response fields — use camelCase aliases for API fields
    some_field: str | None = Field(default=None, alias="someField")
    another_field: int | None = None


class <Action>TalentAdapter(APIAdapter):
    api_endpoint = "learningintegration/<path>"  # relative to TALENT_API_BASE_URL
    method = "POST"  # GET | POST | PUT | PATCH | DELETE
    pydantic_parser = <Action>TalentResponse

    async def execute(self, param1: str, param2: int) -> <Action>TalentResponse:
        data = {
            "field1": param1,
            "field2": param2,
            # camelCase keys as Talent API expects
        }
        return await super().execute(params={}, data=data)
```

**When `api_endpoint` needs URL parameters** (e.g., `/users/{user_id}/action`):

```python
class <Action>TalentAdapter(APIAdapter):
    api_endpoint = "learningintegration/users/{talent_organization_id}/action"
    method = "POST"
    pydantic_parser = <Action>TalentResponse

    async def execute(self, talent_organization_id: int, email: str) -> <Action>TalentResponse:
        data = {"email": email}
        return await super().execute(
            params={"talent_organization_id": talent_organization_id},
            data=data,
        )
```

`params` dict values are interpolated into `api_endpoint` via `.format(**params)`.

**Key rules for `APIAdapter` subclass:**
- `base_url` defaults to `settings.TALENT_API_BASE_URL` — no need to override
- `pydantic_parser` handles response parsing automatically
- Built-in retry with exponential backoff (3 tries, 30s max) via `@backoff.on_exception`
- `execute()` is **async**

---

## Pattern 3: Talent Adapter (manual httpx — legacy)

Use only when you need complex logic that `APIAdapter` can't express (e.g., branching HTTP methods, list responses, special error handling).

```python
# apps/users/adapters/talent/<action>.py
import logging

import httpx
from pydantic import BaseModel, Extra, Field

from crehana_centralized_users.settings.base import settings
from crehana_centralized_users.utils.http_client import httpx_client_maker

logger = logging.getLogger(__name__)


class <Action>TalentResponse(BaseModel, extra=Extra.allow):
    some_field: str | None = None
    camel_case_field: str | None = Field(default=None, alias="camelCaseField")


class <Action>TalentAdapter:
    def __init__(
        self, base_url: str | None = None, client: httpx.AsyncClient | None = None
    ):
        self.base_url = base_url or settings.TALENT_API_BASE_URL
        self.client = client or httpx_client_maker()

    async def execute(self, param1: str, param2: int) -> <Action>TalentResponse | None:
        url = f"{self.base_url}/learningintegration/<path>/{param1}"

        try:
            logger.debug(f"Request: {url}")
            response = await self.client.post(
                url=url,
                json={"field": param2},
            )
            logger.info(f"Response: {response.status_code} | {response.text}")
            response.raise_for_status()
        except Exception as e:
            logger.exception(e)
            raise Exception("Error calling <action> in talent") from e

        try:
            parsed_response = <Action>TalentResponse.parse_obj(response.json())
        except Exception as e:
            logger.exception(e)
            raise Exception("Error parsing response from Talent") from e

        return parsed_response

    async def aclose(self):
        await self.client.aclose()
```

---

## Response Model Conventions

All response models follow the same rules:

```python
# Always use extra=Extra.allow — Talent/Learning may add new fields without warning
class MyResponse(BaseModel, extra=Extra.allow):
    # snake_case fields with camelCase aliases for API mapping
    success: bool | None = None
    user_id: int | None = Field(default=None, alias="userId")
    organization_id: int | None = Field(default=None, alias="organizationId")
    error: str | None = None

# Nested models: prefix private ones with _
class _UserDetail(BaseModel):
    id: int
    first_name: str = Field(alias="firstName")
    last_name: str = Field(alias="lastName")
```

**Use `@validator` when a field needs to be computed from other fields:**

```python
from pydantic import validator

class MyResponse(BaseModel, extra=Extra.allow):
    success: bool | None = None
    invalid_items: list[_Item] | None = Field(default=None, alias="invalidItems")
    error: str | None = None

    @validator("error", always=True, pre=True)
    def validate_error(cls, value, values, config, field):
        invalid_items = values.get("invalid_items")
        if invalid_items:
            return invalid_items[0].message
        return None
```

---

## Settings Reference

Available base URLs (from `settings`):

| Constant | Service |
|----------|---------|
| `settings.CREHANA_API_V4_B2B_URL` | Learning GraphQL v4 |
| `settings.CREHANA_API_V2_B2B_URL` | Learning GraphQL v2 (legacy) |
| `settings.TALENT_API_BASE_URL` | Talent REST API |

---

## Real Examples from the Codebase

### Learning (GraphQL)
```
apps/users/adapters/learning/create_user.py
apps/users/adapters/learning/activate_user.py
apps/users/adapters/learning/deactivate_user.py
apps/users/adapters/learning/update_user_custom_field.py
```

### Talent (APIAdapter — preferred)
```
apps/users/adapters/talent/update_user_email.py
```

### Talent (manual httpx — legacy)
```
apps/users/adapters/talent/create_user.py
apps/users/adapters/talent/deactivate_user.py
apps/users/adapters/talent/get_user_detail.py
```

---

## Checklist — Before Submitting

**Learning adapter:**
- [ ] Response model uses `extra=Extra.allow`
- [ ] `execute()` is **synchronous** (no `async`)
- [ ] JWT generated with `generate_learning_jwt(user_id=..., email=...)`
- [ ] `GraphQLClient` instantiated with correct URL version (V2 or V4)
- [ ] `.graphql` file created in `adapters/learning/graphql/`
- [ ] `response["<operationName>"]` key matches the GraphQL operation name exactly
- [ ] Dual try-except: one for HTTP call, one for parsing

**Talent adapter (APIAdapter):**
- [ ] Extends `APIAdapter`
- [ ] `api_endpoint` set as class attribute (relative path)
- [ ] `method` set as class attribute (`GET`/`POST`/`PUT`/`DELETE`)
- [ ] `pydantic_parser` set if response needs parsing
- [ ] `execute()` is **async** and calls `super().execute(params={}, data=...)`
- [ ] URL params passed via `params` dict, not string formatting

**Talent adapter (manual httpx):**
- [ ] `execute()` is **async**
- [ ] `__init__` accepts `base_url` and `client` for testability
- [ ] Dual try-except: one for HTTP call, one for parsing
- [ ] `aclose()` defined for cleanup
- [ ] Response model uses `extra=Extra.allow`

---

## Anti-Patterns — What NOT to Do

1. **Do not call adapters from mutations or queries** — adapters are called from use cases or services only
2. **Do not make Learning adapters async** — `GraphQLClient` is synchronous
3. **Do not hardcode URLs** — always use `settings.TALENT_API_BASE_URL` or computed URL properties
4. **Do not use a single generic try-except** — always have one for the HTTP call and a separate one for response parsing
5. **Do not skip `extra=Extra.allow`** on response models — external APIs add fields without notice
6. **Do not use `requests` for new Talent adapters** — use `httpx` (async) or `APIAdapter`
7. **Do not create the response model and adapter in separate files** — both go in the same file
8. **Do not name the response class without the service suffix** — always `<Action>LearningResponse` or `<Action>TalentResponse` to avoid confusion
