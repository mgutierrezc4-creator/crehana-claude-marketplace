---
name: new-pubsub-handler
description: Generates a complete, production-ready Google Cloud Pub/Sub handler for the centralized-users FastAPI microservice. Use this skill whenever the user wants to add a new PubSub event handler, create a new subscription, handle a new event type, says "agrega un handler de pubsub", "nueva suscripción de pubsub", "implementa el handler para X", "necesito procesar el evento X", or anything about consuming or handling Pub/Sub messages. Always use this skill — manually written PubSub handlers in this project frequently miss the re-raise pattern (causing silent ACKs on failure), skip the publisher injection needed for testing, or register in the wrong place.
---

## Your job

Generate all the files needed for a new PubSub handler in this project. Always use the **new pattern** (`EVENT_HANDLERS` dict in `event_handler.py`) — never add `if/elif` branches to existing handlers.

Before writing code, run a short interview (see below). Then generate the files and show the user exactly what to register where.

---

## Interview

Ask only what you can't infer. Fill in sensible defaults when possible. Never ask more than one question at a time.

Gather:
1. **Event name** — the new event to handle (e.g. `sync_user_custom_fields`). Will become an entry in `PubSubEvents`.
2. **Case** — is this:
   - **A**: New event on an **existing subscription** (most common — just adds a handler + event type)
   - **B**: A **brand new subscription** (needs its own subscription ID in settings + entry in `SUBSCRIPTION_MAP`)
3. **App and use case** — where the handler logic lives (e.g. `users / create_user`, `users / assign_user_to_permission_group`)
4. **Input fields** — the fields expected in `message_data["data"]` (name + type)
5. **Publishes downstream?** — does this handler need to publish events to another topic after processing? If yes: which event and topic name.
6. For **Case B only**: subscription ID (e.g. `subs-sync-user-custom-fields`) and a short name for `SUBSCRIPTION_MAP` (e.g. `sync-user-custom-fields`)

---

## File structure

**Case A — new event on existing subscription:**
```
apps/{app}/use_cases/{use_case}/pubsub/{event}.py   ← input model + service class + handler function
```
Plus two registration edits:
- `utils/pubsub/events.py` — add entry to `PubSubEvents` enum
- `utils/pubsub/event_handler.py` — add entry to `EVENT_HANDLERS` dict

**Case B — brand new subscription:**
Everything in Case A, plus:
- `settings/base.py` — add `PUBSUB_SUBSCRIPTION_{EVENT}_ID`
- `pubsub_new.py` — add entry to `SUBSCRIPTION_MAP` and create the subscription handler function

---

## Patterns to follow exactly

### Input model

Use Pydantic v1 (`BaseModel`). Always use `.parse_obj()`, never direct instantiation from dict.

```python
from pydantic import BaseModel


class {Event}Input(BaseModel):
    organization_id: int
    user_id: int
    # ... fields from the interview
```

Group related optional fields with `Optional[X] = None` when data comes from cross-service context (auth user IDs, learning user data, etc.).

### Handler file (`apps/{app}/use_cases/{use_case}/pubsub/{event}.py`)

```python
import logging

from sqlalchemy.orm import Session

from crehana_centralized_users.utils.pubsub.publisher import PubSubPublisher

logger = logging.getLogger(__name__)


class {Event}Input(BaseModel):
    # ... fields


class {Event}Service:
    def __init__(self, session: Session, pubsub_publisher=None):
        self.session = session
        self.pubsub_publisher = pubsub_publisher or PubSubPublisher()
        # initialize any other services here

    def execute(self, input_data: {Event}Input) -> None:
        # orchestrate business logic via private methods
        self.__do_something()
        # if publishes downstream:
        # self.__publish_downstream()

    def __do_something(self):
        pass

    # Only if publishes downstream:
    def __publish_downstream(self):
        self.pubsub_publisher.run(
            data={...},
            event_type=PubSubEvents.{NEXT_EVENT}.value,
            topic_name=f"{topic-name}-{settings.APP_ENVIRONMENT}",
        )


async def {event}_pubsub_handler(
    message_data: dict,
    session: Session,
) -> None:
    try:
        input_data = {Event}Input.parse_obj(message_data)
        service = {Event}Service(session=session)
        await service.execute(input_data)  # or service.execute() if sync
        session.commit()
    except Exception as e:
        logger.exception(e)
        raise e  # re-raise so the decorator NACKs the message
```

**Critical rules for the handler function:**
- **Always re-raise** the exception — without this, the message gets ACKed even on failure and is silently lost
- **`session.commit()` belongs in the handler**, not the service — the service should not know it's in a PubSub context
- **`session.close()` in `finally`** only if the handler creates its own session (Case B); for Case A, `event_handler.py` manages the session lifecycle

### Event registration (`utils/pubsub/events.py`)

Add the new event to the `PubSubEvents` StrEnum:

```python
class PubSubEvents(StrEnum):
    # ... existing events
    {EVENT_NAME} = "{event_name}"
```

Use snake_case for the value. The key matches the value in uppercase by convention.

### Handler registration (`utils/pubsub/event_handler.py`)

Add one entry to `EVENT_HANDLERS`:

```python
from crehana_centralized_users.apps.{app}.use_cases.{use_case}.pubsub.{event} import (
    {event}_pubsub_handler,
)

EVENT_HANDLERS: dict = {
    # ... existing handlers
    PubSubEvents.{EVENT_NAME}: {event}_pubsub_handler,
}
```

### Case B only: new subscription

**`settings/base.py`** — add the subscription ID:
```python
PUBSUB_SUBSCRIPTION_{EVENT}_ID: str = "subs-{event-name}-dev"
```

**`pubsub_new.py`** — create the subscription handler and register it:

```python
# 1. New subscription handler (wraps event_handler.async_handler)
@PubSubDecorator(timeout=600, encoding=PubSubEncoding.JSON)
async def {event}_subscription_handler(message: Message, message_data: dict):
    session = next(get_db())
    try:
        await async_handler(message_data, session)
        # NOTE: do NOT call session.commit() here — the inner handler already does it
    except Exception as ex:
        logger.error(ex, exc_info=True)
        raise  # re-raise so PubSubDecorator NACKs the message
    finally:
        session.close()

# 2. Add to SUBSCRIPTION_MAP
SUBSCRIPTION_MAP = {
    # ... existing entries
    "{subscription-short-name}": (
        settings.PUBSUB_SUBSCRIPTION_{EVENT}_ID,
        {event}_subscription_handler,
    ),
}
```

---

## Non-negotiable rules

- **Always re-raise exceptions in the handler** — the `PubSubDecorator` NACKs only when an exception propagates out. Swallowing it means the message is ACKed and silently lost.
- **`session.commit()` in the handler, never in the service** — the service is reusable across contexts (HTTP, PubSub, tests); committing belongs to the caller.
- **`pubsub_publisher=None` always in `__init__`** — even if the service doesn't publish downstream today. It costs nothing and allows tests to inject a mock without patching.
- **Pydantic `.parse_obj()`** — never unpack `message_data` manually; the model validates and provides clear error messages.
- **`logger = logging.getLogger(__name__)`** — one per file.
- **Always use `PubSubEvents` enum** — never hardcode event name strings outside of `events.py`.

---

## Registration checklist

After generating the files, always show this checklist to the user:

**Case A:**
- [ ] Add `{EVENT_NAME} = "{event_name}"` to `PubSubEvents` in `utils/pubsub/events.py`
- [ ] Add `PubSubEvents.{EVENT_NAME}: {event}_pubsub_handler` to `EVENT_HANDLERS` in `utils/pubsub/event_handler.py`

**Case B (everything above plus):**
- [ ] Add `PUBSUB_SUBSCRIPTION_{EVENT}_ID` to `settings/base.py`
- [ ] Add subscription handler + `SUBSCRIPTION_MAP` entry to `pubsub_new.py`
- [ ] Add `"{subscription-short-name}"` to `PUBSUB_INCLUDE_SUBSCRIPTIONS` env var in deployment config
