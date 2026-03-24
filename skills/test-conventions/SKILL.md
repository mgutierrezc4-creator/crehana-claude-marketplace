---
name: test-conventions
description: >
  Testing patterns and conventions for the centralized-users FastAPI microservice.
  Use this skill whenever the user asks to write tests, add test coverage, create test files,
  or test any component (mutations, use cases, PubSub handlers, services, repositories).
  Also use when the user says "agrega tests", "escribe tests para", "necesito testear",
  "test the handler", "add test coverage", or anything related to writing or fixing tests
  in this project. Even if the user just says "testea esto", use this skill.
---

# Test Conventions — centralized-users

This skill defines how to write tests in this project. Follow these conventions precisely — do not invent alternative patterns.

## Golden Rules

1. **Use the `session` fixture** from conftest.py. Never create your own DB session.
2. **Use `session.flush()`**, never `session.commit()` — the session fixture manages transactions with automatic rollback.
3. **Use `pytestmark = pytest.mark.anyio`** at module level for async test files. Do not use per-function `@pytest.mark.anyio`.
4. **Reuse existing factory functions** before writing new ones. Search with Grep first in `tests/factories/`, then in nearby `mocks.py` files.
5. **Place tests in the correct directory** based on what you're testing (see structure below).
6. **Each test creates only the data it needs** — no god-fixtures that create an entire world of entities. If a test only needs one user and one org, create only that.
7. **Test each layer independently** — services get their own tests, not just indirect coverage through use cases.

## Test File Placement

```
tests/
├── factories/                # Centralized entity factories (preferred)
│   ├── __init__.py
│   ├── organization.py       # create_organization()
│   ├── user.py               # create_user(), create_user_organization()
│   ├── custom_field.py       # create_custom_field(), create_custom_field_value()
│   └── excel_import.py       # create_excel_import(), create_bulk_upload_chunk()
├── apps/users/
│   ├── mutations/            # GraphQL mutation tests
│   │   └── test_mutation_<name>.py
│   ├── use_cases/
│   │   └── <use_case>/
│   │       ├── test_<name>.py           # Use case unit tests
│   │       ├── mocks.py                 # Use-case-specific builders (message builders, etc.)
│   │       └── pubsub/
│   │           ├── conftest.py          # PubSub-specific fixtures (CapturingPublisher, etc.)
│   │           └── test_<handler>_pubsub_handler.py
│   ├── services/             # Service-layer tests (isolated)
│   │   └── test_<service>.py
│   ├── repository/           # Repository tests
│   │   └── test_<entity>_repository.py
│   └── adapters/             # Adapter tests
│       └── test_<adapter>.py
```

### Where to put factory functions

**Entity factories** (create_organization, create_user, etc.) go in `tests/factories/`. These are generic creators for any entity that multiple test files need. If a `tests/factories/` directory doesn't exist yet, create it.

**Use-case-specific builders** (build_batch_message, build_valid_row, etc.) stay in the `mocks.py` near the tests that use them, because they encode domain-specific message structures — not generic entity creation.

The distinction: if the function creates a DB entity, it belongs in `tests/factories/`. If it builds a dict/payload specific to a use case, it belongs in the local `mocks.py`.

## Naming Conventions

- **Test files**: `test_<what_youre_testing>.py`
  - Mutations: `test_mutation_<mutation_name>.py`
  - PubSub handlers: `test_<handler_name>_pubsub_handler.py`
  - Services: `test_<service_name>.py`
- **Test functions**: `test_<action>_<condition>_<expected_result>`
  - Happy path: `test_create_user_with_valid_data`
  - Error case: `test_create_user_with_duplicate_email_raises_error`
  - Numbered scenarios: `test_h1_1_five_rows_produce_one_chunk` (for multi-scenario handler tests)
- **Fixtures**: descriptive names like `populate_data`, `mock_step_function`

## Async Test Pattern

```python
import pytest

pytestmark = pytest.mark.anyio

async def test_something(session):
    # test code here
    pass
```

The `anyio_backend` fixture in root conftest returns `"asyncio"`. Pytest config uses `asyncio_mode = "strict"`.

## Database Fixtures — How They Work

The root `conftest.py` provides a `session` fixture (function scope) that:
- Opens a transaction with a nested savepoint
- Rolls back everything after each test
- Overrides `get_db` dependency so all code under test uses this same session

This is why you must use `flush()` (writes to DB within the transaction) and never `commit()` (would finalize the transaction and break isolation).

```python
# CORRECT
def test_something(session):
    org = Organization(name="test", slug="test-slug")
    session.add(org)
    session.flush()  # data is visible within this transaction

# WRONG - never do this
def test_something(session):
    session.commit()  # breaks transaction isolation
```

## Pattern: Centralized Factories

Entity factories live in `tests/factories/` and follow a consistent pattern. Each factory receives `session` and `**overrides`, uses faker for defaults, and returns the created object.

```python
# tests/factories/organization.py
from faker import Faker
from models.organization.organization import Organization

faker = Faker()

def create_organization(session, **overrides):
    defaults = dict(
        name=faker.company(),
        slug=faker.slug(),
        country_code="PE",
    )
    defaults.update(overrides)
    org = Organization(**defaults)
    session.add(org)
    session.flush()
    return org
```

```python
# tests/factories/user.py
from faker import Faker
from models.crehana.user import User

faker = Faker()

def create_user(session, organization_id=None, **overrides):
    defaults = dict(
        email=faker.email(),
        first_name=faker.first_name(),
        last_name=faker.last_name(),
    )
    if organization_id:
        defaults["organization_id"] = organization_id
    defaults.update(overrides)
    user = User(**defaults)
    session.add(user)
    session.flush()
    return user
```

When writing new tests:
1. First search `tests/factories/` for an existing factory
2. If it doesn't exist, create it there — not inline in the test file
3. For use-case-specific builders (message payloads, not DB entities), use local `mocks.py`

## Pattern: Minimal Test Data

Each test should create only the entities it actually needs. This makes tests faster, easier to understand, and less fragile.

```python
# GOOD — creates exactly what this test needs
def test_deactivate_user(session):
    org = create_organization(session=session)
    user = create_user(session=session, organization_id=org.id)

    service = DeactivateUser(session=session)
    service.execute(user_id=user.id, org_id=org.id)

    refreshed = session.query(User).get(user.id)
    assert refreshed.is_active is False

# BAD — creates an entire world when it only needs 1 user and 1 org
def test_deactivate_user(session, populated_items):
    # populated_items has enterprise, 3 orgs, 5 users, permission groups...
    service = DeactivateUser(session=session)
    service.execute(
        user_id=populated_items.user_1.id,
        org_id=populated_items.organization_1.id,
    )
```

Use NamedTuple fixtures only when a specific test or small group of tests genuinely need multiple related entities:

```python
from typing import NamedTuple

class OrgWithAdmin(NamedTuple):
    organization: Organization
    admin: User

@pytest.fixture()
def org_with_admin(session):
    org = create_organization(session=session)
    admin = create_user(session=session, organization_id=org.id, is_admin=True)
    return OrgWithAdmin(organization=org, admin=admin)
```

## Pattern: Service-Layer Tests

Services contain business logic and should be tested independently — not only via use cases. This is important because when a use case test fails, you need to know whether the bug is in the orchestration (use case) or the logic (service).

```python
# tests/apps/users/services/test_user_organization_service.py
import pytest
from tests.factories.organization import create_organization
from tests.factories.user import create_user

pytestmark = pytest.mark.anyio

class TestUserOrganizationService:
    """Tests for the service layer, isolated from use cases."""

    async def test_assign_user_to_organization(self, session):
        org = create_organization(session=session)
        user = create_user(session=session)

        service = UserOrganizationService(session=session)
        result = service.assign_user(user_id=user.id, organization_id=org.id)

        assert result is not None
        uo = session.query(UserOrganization).filter_by(
            user_id=user.id, organization_id=org.id
        ).first()
        assert uo is not None
        assert uo.status == UserOrganizationStatus.ACTIVE

    async def test_assign_duplicate_raises_error(self, session):
        org = create_organization(session=session)
        user = create_user(session=session, organization_id=org.id)

        service = UserOrganizationService(session=session)
        with pytest.raises(UserAlreadyInOrganizationError):
            service.assign_user(user_id=user.id, organization_id=org.id)
```

When to test at service layer vs use case layer:
- **Service test**: the method has branching logic, calculations, or complex queries
- **Use case test**: you're testing the orchestration — that the right services are called in the right order with the right data

## Pattern: GraphQL Mutation Tests

Use `schema.execute()` directly — not HTTP calls through `async_client`.

```python
from types import SimpleNamespace

mutation = """
mutation doSomething($orgSlug: String!, $userId: Int!) {
    centralized_users {
        do_something(input: { org_slug: $orgSlug, user_id: $userId }) {
            success
            response { code message }
        }
    }
}
"""

async def test_mutation_happy_path(session, populate_data):
    result = await schema.execute(
        mutation,
        variable_values={"orgSlug": "test", "userId": 1},
        context_value=SimpleNamespace(session=session),
    )
    assert result.errors is None
    assert result.data["centralized_users"]["do_something"]["success"] is True
```

## Pattern: Use Case Tests

Instantiate the use case class directly, inject `session` and mocked dependencies.

```python
def test_use_case_happy_path(session):
    org = create_organization(session=session)
    user = create_user(session=session, organization_id=org.id)

    use_case = MyUseCase(
        session=session,
        pubsub_publisher=MagicMock(),
    )

    result = use_case.execute(input=MyUseCaseInput(
        organization_slug=org.slug,
        user_id=user.id,
    ))

    assert result.success is True
```

For exception testing:
```python
def test_use_case_invalid_user(session):
    org = create_organization(session=session)

    use_case = MyUseCase(session=session, pubsub_publisher=MagicMock())

    with pytest.raises(UserNotFoundError):
        use_case.execute(input=MyUseCaseInput(
            organization_slug=org.slug,
            user_id=99999,
        ))
```

## Pattern: PubSub Handler Tests

PubSub tests are the most complex. The key tools:

### CapturingPublisher

Defined in `tests/apps/users/use_cases/create_user/pubsub/conftest.py`. Captures all published messages for assertion.

```python
publisher = CapturingPublisher()

service = MyPubSubHandler(session=session, pubsub_publisher=publisher)
service.execute(message_data={...})

# Assert what was published — verify BOTH count AND payload structure
bulk_msgs = publisher.messages_for_event(PubSubEvents.SOME_EVENT.value)
assert len(bulk_msgs) == 1

# Always validate the payload shape, not just the count
payload = bulk_msgs[0]["data"]
assert "user_id" in payload
assert payload["organization_id"] == org.id
assert payload["email"] == expected_email
```

### Reusable external service mocking

When multiple PubSub handler tests need the same external service mocks (audit log, segmentation, learning API), define them as fixtures in the PubSub conftest.py rather than repeating `@contextmanager` in each test file:

```python
# tests/apps/users/use_cases/<use_case>/pubsub/conftest.py
@pytest.fixture()
def mock_external_services():
    """Mocks all external services and yields a CapturingPublisher."""
    publisher = CapturingPublisher()
    with patch("path.to.PubSubPublisher", return_value=publisher):
        with patch("path.to.AuditLogClient") as mock_audit:
            mock_audit.return_value.send_pubsub_event_massive.return_value = None
            with patch("path.to.SegmentationService") as mock_seg:
                mock_seg.return_value.send_pubsub_event_massive.return_value = None
                with patch("path.to.LearningUserOrgService") as mock_learning:
                    mock_learning.return_value.get_user_organization_by_emails.return_value = []
                    yield publisher
```

Then in tests:
```python
async def test_handler_creates_users(session, mock_external_services):
    publisher = mock_external_services
    # ... test code ...
    learning_msgs = publisher.messages_for_event(PubSubEvents.LEARNING_CREATE_USER.value)
    assert len(learning_msgs) == 3
```

### Full PubSub handler test

```python
pytestmark = pytest.mark.anyio

async def test_handler_creates_users(session, mock_external_services):
    publisher = mock_external_services
    org = create_organization(session=session)
    admin = create_user(session=session, organization_id=org.id)
    excel_import = create_excel_import(session=session, organization_id=org.id)
    chunk = create_bulk_upload_chunk(session=session, excel_import_id=excel_import.id)
    session.flush()

    message = build_batch_message(
        org=org, admin_user=admin,
        excel_import_id=excel_import.id,
        chunk_id=chunk.id, n_users=3,
    )
    expected_emails = [u["email"] for u in message["users"]]

    await handler_function(message_data=message, session=session)

    # Verify DB state
    created = session.query(User).filter(User.email.in_(expected_emails)).all()
    assert len(created) == 3

    # Verify published events AND their payloads
    learning_msgs = publisher.messages_for_event(PubSubEvents.LEARNING_CREATE_USER.value)
    assert len(learning_msgs) == 3
    for msg in learning_msgs:
        assert "user_id" in msg["data"]
        assert "organization_id" in msg["data"]
        assert msg["data"]["organization_id"] == org.id
```

## Pattern: Mocking External HTTP Calls

- **httpx (async)**: Use `respx`
- **requests (sync)**: Use `responses`
- **AWS (boto3)**: Use `moto` decorators like `@mock_s3`

```python
import respx
from httpx import Response

async def test_external_call(session):
    respx.get("https://api.example.com/data").mock(
        return_value=Response(200, json={"result": "ok"})
    )
    # ... test code that makes httpx call ...
```

## Anti-Patterns — What NOT to Do

1. **Do not create DB engines or sessions** — use the `session` fixture
2. **Do not use `session.commit()`** — use `session.flush()`
3. **Do not mock the database** — use real DB with transaction rollback
4. **Do not write entity factories inline** in test files — put them in `tests/factories/`
5. **Do not use `@pytest.mark.asyncio`** per function — use `pytestmark = pytest.mark.anyio` at module level
6. **Do not test via HTTP client** for GraphQL mutations — use `schema.execute()`
7. **Do not duplicate existing fixtures** — check conftest.py files up the directory tree
8. **Do not put PubSub handler tests** in the use_cases root — they go in the `pubsub/` subdirectory
9. **Do not create god-fixtures** (TestBase/PopulatedItems pattern) for new tests — create only what each test needs using factories
10. **Do not only assert message count** on PubSub — always validate the payload structure too
11. **Do not test services only through use cases** — if a service has meaningful logic, test it directly
12. **Do not repeat external service mocks** across test files — extract them to conftest.py fixtures
