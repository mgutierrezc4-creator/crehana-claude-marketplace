---
name: new-mutation
description: Generates a complete, production-ready GraphQL mutation for the centralized-users FastAPI microservice. Use this skill whenever the user wants to create a new mutation, says "crea una mutation", "agrega una mutation", "nueva mutation para X", "necesito una mutation que haga X", or anything about adding a GraphQL mutation to the project. Also trigger for "scaffold a mutation", "genera la estructura para X", or when the user describes a new feature that clearly needs a GraphQL entry point. Always use this skill for mutation creation — manually generated mutations in this project frequently miss permissions, use wrong patterns, or break namespace conventions.
---

## Your job

Generate all the files needed for a new GraphQL mutation in this project. Every mutation you generate must follow the conventions below — they exist for good reasons (auth, observability, consistency) and are not optional.

Before writing any code, conduct a short interview to gather what you need (see "Interview" below). Then generate the files, and finally show the user exactly where to register the mutation.

---

## Interview

Ask only what you can't infer. If the user has already given you most of the information, fill in sensible defaults and confirm. Never ask more than one question at a time.

Gather:
1. **Mutation name** (snake_case, e.g. `update_user_position`)
2. **App** (`users`, `teams`, `corporations`, or another existing app)
3. **Description** — short string for `description=` in the decorator
4. **Input fields** — name and type for each field (e.g. `user_id: int`, `organization_slug: str`)
5. **Namespace** — which existing namespace should this mutation live in, or does it need a new one? (See "Namespace rules" below)
6. **Response type** — simple success/fail, or does it need to return specific data? (See "Response patterns" below)
7. **Permission level** — default is full admin stack; ask only if the mutation is for non-admin users or needs super_admin
8. **Repository?** — only ask if the mutation clearly needs its own DB queries not covered by existing services

If the user asks for a mutation as part of an epic or feature, search for an existing namespace in that app before proposing a new one.

---

## Namespace rules

Namespaces group mutations by domain so the GraphQL schema stays organized. The key idea: mutations from the same epic or feature area should share a namespace.

**Before proposing a new namespace**, search the target app for existing ones:
```
Glob: apps/{app}/schema/mutations/*.py
```
Look for classes ending in `Namespace` or `NamespaceMutation`. If a suitable one exists, add the new mutation there.

**When to create a new namespace**: only when the mutation is genuinely a new domain with no existing home. A namespace is just a `@strawberry.type` class with a `@strawberry.field` that returns the grouped mutations type:

```python
@strawberry.type
class MyDomainNamespaceMutation:
    @strawberry.field(description="Namespace for my-domain mutations.")
    def my_domain(self) -> "MyDomainMutations":
        return MyDomainMutations()
```

Always tell the user which namespace you added to (or created) and how to register it in `root_schema.py`.

---

## File structure

For a mutation named `{domain}` in `apps/{app}/`:

```
schema/
  input/{domain}.py        ← @strawberry.input with all fields
  nodes/{domain}.py        ← @strawberry.type response (if new response needed)
  mutations/{domain}.py    ← the mutation class
services/{domain}.py       ← business logic
repository/{domain}.py     ← DB queries (only if needed)
```

Generate all four (or five) files. Never put business logic in the mutation resolver — it belongs in the service.

---

## Patterns to follow exactly

### Input (`schema/input/{domain}.py`)

All fields as typed attributes. Group related fields into sub-inputs when there are more than ~5 fields.

```python
import strawberry


@strawberry.input
class {Domain}Input:
    organization_slug: str
    user_id: int
    # ... fields the user specified
```

No loose parameters in the mutation signature — the input class is the single source of truth for what the mutation accepts.

### Response node (`schema/nodes/{domain}.py`)

**Simple response** (success/fail, most common):
```python
import strawberry
from crehana_centralized_users.utils.nodes import ResponseNodeExtra


@strawberry.type
class {Domain}Response:
    graphql_operation_id: strawberry.ID
    success: bool
    response: ResponseNodeExtra
```

**Rich response** (when the mutation needs to return specific data, e.g. an ID or a list):
```python
import strawberry
from typing import Optional, Union
from crehana_centralized_users.utils.responses import (
    CentralizedMutationInternalError,
    CentralizedMutationValidationError,
)


@strawberry.type
class {Domain}Response:
    # specific fields the user needs
    entity_id: int


# The return type annotation on the mutation becomes:
# Union[{Domain}Response, CentralizedMutationValidationError, CentralizedMutationInternalError]
```

Use the simple pattern by default. Use the rich/Union pattern when the mutation must return specific data the client will use (not just success/fail).

### Mutation class (`schema/mutations/{domain}.py`)

```python
import logging

import strawberry
from subgraph_utils.common.function import get_graphql_operation_id

from crehana_centralized_users.apps.{app}.schema.input.{domain} import {Domain}Input
from crehana_centralized_users.apps.{app}.schema.nodes.{domain} import {Domain}Response
from crehana_centralized_users.apps.{app}.services.{domain} import {Domain}Service
from crehana_centralized_users.apps.users.exceptions import CentralizedUsersError
from crehana_centralized_users.auth.permissions import (
    IsAdminCentralized,
    IsAuthenticatedCentralized,
    IsAuthenticatedLearning,
)
from crehana_centralized_users.dependencies.graphql_context import CustomInfo
from crehana_centralized_users.utils.nodes import ResponseNodeExtra

logger = logging.getLogger(__name__)


@strawberry.type
class {Domain}Mutation:
    @strawberry.mutation(
        permission_classes=[
            IsAuthenticatedLearning,
            IsAuthenticatedCentralized,
            IsAdminCentralized,
        ],
        description="{description}",
    )
    async def {mutation_name}(
        self,
        info: CustomInfo,
        input: {Domain}Input,
    ) -> {Domain}Response:
        graphql_operation_id = get_graphql_operation_id()
        try:
            service = {Domain}Service(info.context.session)
            service.execute(input)
            return {Domain}Response(
                graphql_operation_id=graphql_operation_id,
                success=True,
                response=ResponseNodeExtra.success(),
            )
        except CentralizedUsersError as ex:
            logger.error(
                "graphql_operation_id: %s, message: %s",
                graphql_operation_id,
                str(ex),
                exc_info=True,
            )
            return {Domain}Response(
                graphql_operation_id=graphql_operation_id,
                success=False,
                response=ResponseNodeExtra.from_centralized_user_error(ex),
            )
        except Exception as ex:
            logger.error(
                "graphql_operation_id: %s, message: %s",
                graphql_operation_id,
                str(ex),
                exc_info=True,
            )
            return {Domain}Response(
                graphql_operation_id=graphql_operation_id,
                success=False,
                response=ResponseNodeExtra.from_exception(ex),
            )
```

**Permission levels** — substitute the appropriate classes:

| Level | Classes to use |
|---|---|
| Admin + Learning | `IsAuthenticatedLearning, IsAuthenticatedCentralized, IsAdminCentralized` |
| Admin solo centralized | `IsAuthenticatedCentralized, IsAdminCentralized` |
| Super admin | `IsAuthenticatedCentralized, IsSuperAdminCentralized` |
| Member + Learning | `IsAuthenticatedLearning, IsAuthenticatedCentralized` |
| JWT-based (internal) | `IsAuthenticatedCentralizedWithJWT, IsAdminCentralizedWithJWT` |

**When to include `IsAuthenticatedLearning`:**
Include it only if the mutation interacts with the learning system. Clear signals:
- The mutation publishes PubSub events to learning (`LEARNING_CREATE_USER`, `LEARNING_ASSIGN_CUSTOM_FIELD`, etc.)
- The resolver uses `info.context.user_session.session_user_id` to pass `auth_learning_user_id` to a service
- The operation synchronizes data between centralized and learning (user creation, deactivation, bulk uploads)

If the mutation operates **only on centralized data** (teams, licenses, org structure, internal roles, corporations), use the stack without Learning: `[IsAuthenticatedCentralized, IsAdminCentralized]`.

The reason: `IsAuthenticatedLearning` validates an active session in the learning system. If the operation doesn't touch learning, that requirement adds nothing and unnecessarily restricts who can call the mutation.

If unsure, ask the user whether the operation needs to sync with learning.

**Never generate a mutation without `permission_classes`**. If you're unsure which level to use, default to `[IsAuthenticatedCentralized, IsAdminCentralized]` and tell the user.

### Service (`services/{domain}.py`)

```python
from sqlalchemy.orm import Session

from crehana_centralized_users.apps.{app}.schema.input.{domain} import {Domain}Input


class {Domain}Service:
    def __init__(self, session: Session):
        self.session = session

    def execute(self, input: {Domain}Input) -> None:
        # business logic here
        pass
```

Keep the service focused on orchestration. If it needs to talk to the DB, inject a repository. If it needs to call an existing service, do so — don't duplicate queries that already exist elsewhere.

### Repository (`repository/{domain}.py`) — only if needed

```python
from sqlalchemy import select
from sqlalchemy.orm import Session
from crehana_centralized_models.models import SomeModel


class {Domain}Repository:
    def __init__(self, session: Session):
        self.session = session

    def get_by_id(self, entity_id: int) -> SomeModel | None:
        statement = select(SomeModel).where(SomeModel.id == entity_id)
        return self.session.execute(statement).scalar_one_or_none()
```

Only create a repository if the service needs DB queries that aren't covered by an existing service or repository. If you're not sure, ask — it's better to reuse than to duplicate.

---

## Non-negotiable rules

These exist to keep the codebase observable and secure:

- **`permission_classes` always present** — every mutation requires auth. No exceptions.
- **`input:` as single parameter** — no loose `file_url: str, organization_slug: str` style. Group everything into an input class.
- **`async def` always** — even if the service is synchronous today.
- **`graphql_operation_id = get_graphql_operation_id()`** — enables request tracing across services.
- **`logger = logging.getLogger(__name__)`** — one logger per file, named after the module.
- **Specific exception before generic** — always catch `CentralizedUsersError` (or other domain exceptions) before the bare `Exception`.

---

## Registration

After generating the files, always show the user:

1. Which namespace class to add the new `{Domain}Mutation` to (as a mixin or via `merge_types`)
2. If a new namespace was created, how to register it in `root_schema.py`

Example of adding to an existing namespace via `merge_types`:
```python
from crehana_centralized_users.apps.{app}.schema.mutations.{domain} import {Domain}Mutation

ExistingMutations = merge_types(
    "ExistingMutations",
    (ExistingMutation, {Domain}Mutation),  # add here
)
```

Example of registering a new namespace in `root_schema.py`:
```python
from crehana_centralized_users.apps.{app}.schema.mutations.{namespace} import {Namespace}NamespaceMutation

# Add to the schema's mutation types list
```

Point the user to the exact file and line where the registration goes.
