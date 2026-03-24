---
name: new-use-case
description: >
  Generates a complete, production-ready use case for the centralized-users FastAPI microservice.
  Use this skill whenever the user wants to create a new use case, says "crea un use case",
  "agrega un use case", "implementa la lógica para X", "nuevo use case para X",
  "necesito la lógica de negocio para X", or anything about implementing a feature
  that requires orchestrating services, adapters, and DB queries.
  Also trigger for "scaffoldea el use case", "genera el use case de X", or when the user
  describes a new business operation that involves fetching data, validating it, and
  calling external services (Learning, Talent) or updating the DB.
  Always use this skill — manually written use cases in this project frequently miss
  the adapter injection pattern (breaking testability), use wrong rollback compensation,
  or put DB queries directly in the use case instead of a repository.
---

# New Use Case — centralized-users

This skill defines how to create use cases in this project. Follow these conventions
precisely — do not invent alternative patterns.

## Quick Orientation: What is a Use Case?

A use case is the **business logic orchestration layer**. It:
- Fetches needed data from DB (via a repository)
- Validates business rules
- Calls external services (Learning/Talent adapters)
- Writes results back to DB (Centralized)
- Optionally publishes PubSub events

Use cases are called from **mutations** (never directly from queries unless a query triggers complex logic).

---

## File Structure

### Simple use case (single operation, no sub-flows)
```
apps/users/use_cases/<feature_name>.py   ← single file
```

### Complex use case (multiple files, sub-flows, or reusable validators)
```
apps/users/use_cases/<feature_name>/
├── __init__.py
├── <feature_name>.py      ← main use case class (always required)
├── repository.py          ← DB queries (always required for complex)
├── types.py               ← Pydantic models / enums (optional)
└── validators.py          ← validation helpers (optional)
```

**Rule of thumb:** If the use case needs more than 2-3 DB queries OR has a repository shared with other use cases → use the directory structure with a `repository.py`.

---

## Use Case Class Skeleton

```python
# apps/users/use_cases/<feature_name>/<feature_name>.py
import logging

from sqlalchemy.orm import Session

from crehana_centralized_users.apps.users.adapters import (
    SomeAdapter,
    AnotherAdapter,
)
from crehana_centralized_users.apps.users.exceptions import (
    OrganizationNotFoundError,
    UserNotFoundError,
)
from crehana_centralized_users.apps.users.use_cases.<feature_name>.repository import (
    <FeatureName>Repository,
)
from crehana_centralized_users.utils.pubsub.publisher import PubSubPublisher

logger = logging.getLogger(__name__)


class <FeatureName>:
    def __init__(
        self,
        session: Session,
        repository=None,
        some_adapter: SomeAdapter | None = None,
        another_adapter: AnotherAdapter | None = None,
        pubsub_publisher=None,  # only if publishing events
    ):
        self.session = session
        self.repository = repository or <FeatureName>Repository(session)
        self.some_adapter = some_adapter or SomeAdapter()
        self.another_adapter = another_adapter or AnotherAdapter()
        self.pubsub_publisher = pubsub_publisher or PubSubPublisher()

    async def execute(
        self,
        organization_slug: str,
        user_id: int,
        # ... other input params
    ):
        # Step 1: store inputs as instance variables
        self.organization_slug = organization_slug
        self.user_id = user_id

        # Step 2: fetch from DB
        self.__get_organization_from_db()
        self.__get_user_from_db()

        # Step 3: validate business rules
        self.__validate_some_condition()

        # Step 4: execute (with rollback compensation)
        await self.__execute_action()
```

---

## Key Conventions

### 1. All dependencies are injected with `or DefaultValue()` defaults

Every adapter, repository, and pubsub_publisher is optional in `__init__` with a default. This is what makes the use case testable — tests inject mocks without needing to patch.

```python
# CORRECT — injectable
def __init__(self, session, some_adapter=None):
    self.some_adapter = some_adapter or SomeAdapter()

# WRONG — hardcoded, breaks testability
def __init__(self, session):
    self.some_adapter = SomeAdapter()
```

### 2. `execute()` assigns inputs to `self` first

```python
async def execute(self, organization_slug: str, user_id: int, ...):
    self.organization_slug = organization_slug
    self.user_id = user_id
    # ... then call private methods
```

### 3. Private methods use `__` prefix (name mangling)

All internal steps are private:
- `__get_<entity>_from_db()` — DB fetches, sets `self.<entity>`
- `__validate_<rule>()` — raises exception if invalid
- `__<action>_<service>()` — calls one adapter (e.g., `__deactivate_user_talent()`)
- `__<action>_centralized()` — writes to DB (always last)

### 4. DB fetch methods raise exceptions on not-found

```python
def __get_organization_from_db(self):
    organization = self.repository.get_organization_by_slug(self.organization_slug)
    if not organization:
        raise OrganizationNotFoundError()
    self.organization = organization  # store on self for other methods to use

def __get_user_from_db(self):
    user = self.repository.get_user_by_id(self.user_id)
    if not user:
        raise UserNotFoundError()
    self.user = user
```

### 5. DB writes use `session.flush()`, never `session.commit()`

```python
def __update_centralized(self):
    self.user.organization.status = UserOrganizationStatus.INACTIVE
    self.session.flush()  # CORRECT
    # self.session.commit()  # NEVER
```

---

## Rollback Compensation Pattern

When calling multiple external services (Talent → Learning → Centralized DB), use the compensation pattern: if a later step fails, undo earlier steps.

**Always call external services in this order: Talent first → Learning second → Centralized DB last**

```python
async def __execute_action(self):
    # Step A: Talent
    try:
        await self.__action_talent()
    except Exception as e:
        raise e from e

    # Step B: Learning (compensate A if this fails)
    try:
        self.__action_learning()
    except Exception as e:
        await self.__revert_talent()   # ← compensation for step A
        raise e from e

    # Step C: Centralized DB (compensate A + B if this fails)
    try:
        self.__action_centralized()
    except Exception as e:
        await self.__revert_talent()   # ← compensation for step A
        self.__revert_learning()       # ← compensation for step B
        raise e from e
```

**If only calling one external service** (no compensation needed):
```python
async def __execute_action(self):
    try:
        await self.__action_talent()
    except Exception as e:
        raise e from e
    self.__action_centralized()
```

**If error in external service is non-fatal** (log + continue):
```python
async def __execute_action(self):
    try:
        await self.__action_talent()
    except Exception as e:
        logger.warning("Talent call failed, continuing: %s", e)
        # don't re-raise — soft failure

    self.__action_centralized()  # always run
```

---

## Repository Skeleton

```python
# apps/users/use_cases/<feature_name>/repository.py
from crehana_centralized_models.models import Organization, User, UserOrganization
from sqlalchemy import select
from sqlalchemy.orm import Session, contains_eager


class <FeatureName>Repository:
    def __init__(self, session: Session):
        self.session = session

    def get_organization_by_slug(self, organization_slug: str):
        query = (
            select(Organization)
            .filter(Organization.slug == organization_slug)
        )
        return self.session.execute(query).scalars().first()

    def get_user_by_id(self, user_id: int):
        query = select(User).filter(User.id == user_id)
        return self.session.execute(query).scalars().first()

    # Add only the queries this use case actually needs
    def create_<entity>(self, ...):
        entity = Entity(...)
        self.session.add(entity)
        self.session.flush()
        return entity

    def update_<entity>(self, entity_id: int, **fields):
        query = select(Entity).filter(Entity.id == entity_id)
        entity = self.session.execute(query).scalars().first()
        for key, value in fields.items():
            setattr(entity, key, value)
        self.session.flush()
        return entity
```

**Rules for repositories:**
- Only include queries this specific use case needs — not a general CRUD for the entity
- Use `session.flush()` (not `commit()`)
- Never add business logic here — just DB access
- Use `scalars().first()` for single results, `scalars().all()` or `scalars().unique().all()` for lists

---

## Optional: Input Types

When the use case receives complex structured input, define a Pydantic model:

```python
# apps/users/use_cases/<feature_name>/types.py
from pydantic import BaseModel


class <FeatureName>Input(BaseModel):
    organization_slug: str
    user_id: int
    # ... other fields
```

Use it in `execute()`:
```python
async def execute(self, input: <FeatureName>Input):
    self.organization_slug = input.organization_slug
    self.user_id = input.user_id
    ...
```

---

## How to Call a Use Case from a Mutation

```python
# In the mutation resolver
async def my_mutation(self, info: CustomInfo, input: MyInput) -> MyResponse:
    session = info.context.session
    use_case = <FeatureName>(session=session)
    try:
        await use_case.execute(
            organization_slug=input.organization_slug,
            user_id=input.user_id,
        )
        return SuccessResponse(...)
    except CentralizedUsersError as e:
        logger.exception(e)
        return ErrorResponse(...)
    except Exception as e:
        logger.exception(e)
        return ErrorResponse(...)
```

---

## Real Examples from the Codebase

| Use case | Pattern | File |
|---|---|---|
| `DeactivateUser` | Full compensation (Talent→Learning→DB) | `use_cases/deactivate_user/deactivate_user.py` |
| `RemoveUserAdminRole` | Full compensation with revert | `use_cases/remove_user_admin_role/remove_user_admin_role.py` |
| `AssignUserToPermissionGroup` | Repository + adapter + pubsub | `use_cases/assign_user_to_permission_group/assign_user_to_permission_group.py` |
| `UpdateUserCustomField` | Repository + multiple adapters, soft failures | `use_cases/update_user_custom_field/update_user_custom_field.py` |

---

## Checklist — Before Submitting

- [ ] All adapters and repository injected via `__init__` with `or Default()` fallbacks
- [ ] `execute()` assigns all inputs to `self` before calling private methods
- [ ] DB fetches in `__get_*_from_db()` methods — never inline in `execute()`
- [ ] DB fetch methods raise appropriate exceptions when entity not found
- [ ] External service calls follow order: Talent → Learning → Centralized DB
- [ ] Rollback compensation applied if calling 2+ external services
- [ ] DB writes use `session.flush()`, never `session.commit()`
- [ ] Repository in separate `repository.py` if 3+ DB queries needed
- [ ] No business logic in the repository

---

## Anti-Patterns — What NOT to Do

1. **Do not put SQLAlchemy queries inline in the use case** — they go in a repository
2. **Do not hardcode adapter instantiation** — always inject via `__init__` param
3. **Do not call `session.commit()`** — use `session.flush()`
4. **Do not ignore exceptions silently** — either re-raise or log with warning
5. **Do not call Centralized DB before external services** — DB write is always last
6. **Do not skip rollback compensation** when calling 2+ external services
7. **Do not put validation logic inside `execute()`** — extract to `__validate_*()` private methods
8. **Do not call use cases from other use cases** — use shared services instead
9. **Do not put HTTP calls or adapter logic inside the repository** — repository is DB only
