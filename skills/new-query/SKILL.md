---
name: new-query
description: >
  Generates a complete, production-ready GraphQL query for the centralized-users FastAPI microservice.
  Use this skill whenever the user wants to create a new query, says "crea una query", "agrega una query",
  "nueva query para X", "necesito una query que retorne X", "agrega un campo de consulta", or anything about
  adding a GraphQL query to the project. Also trigger for "scaffold a query", "agrega un field de query",
  "expón datos via GraphQL", or when the user describes a new data-fetching requirement that needs a
  GraphQL entry point. Always use this skill for query creation — manually generated queries in this project
  frequently miss registration in root_schema.py, use wrong pagination patterns, or mix concerns between
  the query layer and service layer.
---

# New GraphQL Query — centralized-users

This skill defines how to create GraphQL queries in this project. Follow these conventions precisely — do not invent alternative patterns.

## Quick Orientation: Layers Involved

A new query involves up to 4 touch points:

```
schema/queries/<name>.py       ← Query class (presentation layer — only this file is strictly required)
schema/nodes/<name>.py         ← Response node type (if it doesn't already exist)
schema/input/<name>.py         ← Input type (only if the query takes a structured input object)
root_schema.py                 ← Registration point (always required)
```

Before creating anything, **search for existing services and repositories** that already fetch the data you need. Never write inline SQLAlchemy queries in the query layer.

## File Placement

```
crehana_centralized_users/apps/users/schema/
├── queries/
│   └── <name>.py            ← New query class goes here
├── nodes/
│   └── <name>.py            ← New response node types go here (if new)
└── input/
    └── <name>.py            ← New input types go here (if needed)
```

## Query Class Skeleton

```python
# schema/queries/<name>.py
import logging
from typing import Optional

import strawberry
from crehana_graphql_strawberry.connections.nodes import CustomConnection

from crehana_centralized_users.apps.users.services.<service> import <Service>
from crehana_centralized_users.dependencies.graphql_context import CustomInfo

logger = logging.getLogger(__name__)


@strawberry.type
class <Name>Query:
    @strawberry.field(description="<Clear description of what this returns>")
    async def <field_name>(
        self,
        info: CustomInfo,
        organization_slug: str,
        # ... other args
    ) -> <ReturnType>:
        session = info.context.session
        service = <Service>(session)
        result = service.<method>(...)
        return result
```

**Always use `async def`** — all query methods in this project are async even when the underlying service is sync.

## Return Type Patterns

Choose the right return type based on what the query returns:

### 1. Paginated list (most common)

Use `CustomConnection[NodeType]` for cursor-based pagination:

```python
from crehana_graphql_strawberry.connections.nodes import CustomConnection

async def get_something(
    self,
    info: CustomInfo,
    organization_slug: str,
    search: Optional[str] = None,
    before: Optional[str] = None,
    after: Optional[str] = None,
    first: Optional[int] = None,
    last: Optional[int] = None,
) -> CustomConnection[SomethingNode]:
    session = info.context.session
    service = SomeService(session)
    result = service.get_items(organization_slug=organization_slug, search=search)
    return CustomConnection.from_iterable(
        result,
        total_count=len(result),
        after=after,
        before=before,
        first=first,
        last=last,
    )
```

Use `Connection[NodeType]` (from `crehana_centralized_users.utils.pagination`) for offset-based pagination when the service returns a separate count:

```python
from crehana_centralized_users.utils.pagination import Connection

return Connection.from_results(
    results=items,
    node=SomethingNode,
    total_count=total_count,
    after=after,
    before=before,
    first=first,
    last=last,
)
```

### 2. Single entity (nullable)

```python
async def get_something(
    self,
    info: CustomInfo,
    id: int,
) -> Optional[SomethingNode]:
    session = info.context.session
    service = SomeService(session)
    return service.get_by_id(id=id)
```

### 3. With error handling (union return type)

Use this pattern when the query can fail with known, structured errors:

```python
from subgraph_utils.common.function import get_graphql_operation_id
from crehana_centralized_users.apps.common.error_responses import (
    InternalErrorResponse,
    ValidationErrorResponse,
)

async def get_something(
    self,
    info: CustomInfo,
    input: SomethingInput,
) -> SomethingNode | ValidationErrorResponse | InternalErrorResponse:
    try:
        session = info.context.session
        service = SomeService(session)
        result = service.get(...)
        return SomethingNode(...)
    except SomeDomainError as exc:
        return ValidationErrorResponse(
            graphql_operation_id=getattr(
                info.context, "operation_id", get_graphql_operation_id()
            ),
            message=str(exc),
        )
    except Exception as exc:
        logger.exception("Error in get_something: %s", exc)
        return InternalErrorResponse(
            graphql_operation_id=getattr(
                info.context, "operation_id", get_graphql_operation_id()
            ),
            message="Internal Error",
        )
```

### 4. Simple list (no pagination)

```python
from typing import List

async def get_something(self, info: CustomInfo, ...) -> List[SomethingNode]:
    ...
    return items
```

## Input Types

When a query has many filter parameters, group them into an input type:

```python
# schema/input/<name>.py
import strawberry
from typing import Optional


@strawberry.input
class Get<Name>Input:
    organization_slug: str
    status: Optional[str] = None
    keyword: Optional[str] = None
```

Use the input in the query:
```python
async def get_something(self, info: CustomInfo, input: GetSomethingInput) -> ...:
    ...
```

## Response Node Types

Response nodes go in `schema/nodes/<name>.py`:

```python
# schema/nodes/<name>.py
import strawberry
from typing import Optional


@strawberry.type
class <Name>Node:
    id: int
    name: str
    email: Optional[str] = None
```

**Search first**: many node types already exist (e.g., `CentralizedUserOrganizationNode`, `CentralizedUserNode`). Reuse them rather than creating duplicates.

## Permissions

Unlike mutations, most queries **do not declare explicit `permission_classes`** on `@strawberry.field`. The query simply accesses `info.context.session` and delegates to the service layer.

If a query truly needs to be gated (rare), you can use `@strawberry.field(permission_classes=[...])`. Available permission classes:

```python
from crehana_centralized_users.auth.permissions import (
    IsAuthenticatedCentralized,
    IsAuthenticatedLearning,
    IsAdminCentralized,
)
```

## Registration in root_schema.py

**This step is always required.** Add the new query class to `Query` in `root_schema.py`:

### Option A: Flat (direct on Query type)

```python
# root_schema.py
from crehana_centralized_users.apps.users.schema.queries.<name> import <Name>Query

@strawberry.type
class Query(
    UserQuery,
    ...
    <Name>Query,   # ← add here
):
    pass
```

### Option B: Namespaced under `centralized_users`

For queries that belong in the `centralized_users` namespace (accessed as `centralizedUsers { ... }` from the client):

```python
# root_schema.py

@strawberry.type
class CentralizedUserOrganizationQueries(
    CentralizedUserOrganizationQuery,
    <Name>Query,   # ← add here to namespace it
):
    pass
```

Check existing queries to see which namespace is most appropriate for your use case.

## Checklist — Before Submitting

- [ ] Query class decorated with `@strawberry.type`
- [ ] All methods use `async def`
- [ ] `info: CustomInfo` is the first parameter after `self`
- [ ] Description provided in `@strawberry.field(description="...")`
- [ ] No inline SQLAlchemy queries — delegate to a service or repository
- [ ] Return type is explicit and correct (paginated, nullable, union, list)
- [ ] New node types created in `schema/nodes/` if needed (and not duplicating existing ones)
- [ ] Registered in `root_schema.py` `Query` class

## Anti-Patterns — What NOT to Do

1. **Do not write SQLAlchemy queries inline** in the query method — use existing services/repositories
2. **Do not return raw SQLAlchemy model instances** — wrap in node types
3. **Do not use sync `def`** — always `async def` for query methods
4. **Do not skip registration** in `root_schema.py` — the query won't be reachable
5. **Do not duplicate node types** — search `schema/nodes/` and `schema/queries/` before creating new ones
6. **Do not put business logic** in the query layer — it belongs in services or use cases
7. **Do not access context outside** of the method body (e.g., at class-level) — context is request-scoped

## Real Examples from the Codebase

### Simple paginated query

```python
@strawberry.type
class HeadquarterQuery:
    @strawberry.field(description="Return a list of User's related to a Headquarter")
    async def get_headquarter_users(
        self,
        info: CustomInfo,
        organization_slug: str,
        headquarter_id: int,
        search_by_user_name: Optional[str] = None,
        before: Optional[str] = None,
        after: Optional[str] = None,
        first: Optional[int] = None,
        last: Optional[int] = None,
    ) -> CustomConnection[CentralizedUserNode]:
        service = HeadquarterService(info.context.session)
        response = service.get_headquarter_users(
            organization_slug=organization_slug,
            headquarter_id=headquarter_id,
            search_by_user_name=search_by_user_name,
        )
        return CustomConnection.from_iterable(
            response, total_count=len(response),
            after=after, before=before, first=first, last=last,
        )
```

### Query with union error response

```python
@strawberry.type
class CentralizedUserOrganizationQuery:
    @strawberry.field(description="Get all user organizations")
    async def get_centralized_user_organizations(
        self,
        info: CustomInfo,
        input: CentralizedGetUserOrganizationsInput,
    ) -> Union[Connection[CentralizedUserOrganization], ValidationErrorResponse, InternalErrorResponse]:
        try:
            values = input.to_pydantic()
            service = CentralizedUserOrganizationService(info.context.session)
            result = service.get_centralized_user_organizations(...)
            return Connection.from_results(results=result, node=CentralizedUserOrganization, ...)
        except ValidationError as exc:
            return ValidationErrorResponse(
                graphql_operation_id=get_graphql_operation_id(), message=format_pydantic_errors(exc)
            )
        except Exception as exc:
            logger.exception("Error: %s", exc)
            return InternalErrorResponse(
                graphql_operation_id=get_graphql_operation_id(), message="Internal Error"
            )
```
