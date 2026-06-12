---
name: sqlalchemy-patterns
description: |
  SQLAlchemy 2.0 ORM patterns for FastAPI: declarative mapped classes with Mapped/mapped_column, async session with AsyncSession, relationships with explicit lazy loading, Alembic migration workflow integration. Used by fastapi-architect (model definitions) and alembic-specialist (column type finalization and migration generation). Activated automatically by fastapi-plugin/stack.md.

  Use this skill to:
  - Write SQLAlchemy 2.0 declarative models with Mapped[T] annotations and mapped_column().
  - Manage async database sessions with AsyncSession and async_sessionmaker.
  - Define relationships with explicit lazy loading strategy (lazy="selectin" or "raise").
  - Integrate with Alembic for migration autogeneration.

  Do NOT use this skill for:
  - FastAPI routing and Pydantic schemas — see fastapi-plugin:fastapi-conventions.
  - Alembic migration execution (that's alembic-specialist's job) — this skill covers definitions.
  - Python idioms — see python-foundation:python-conventions.
---

# SQLAlchemy 2.0 Patterns

## Detection

Read `pyproject.toml` before writing any model code:

```bash
grep -E "sqlalchemy" pyproject.toml
```

- SQLAlchemy **2.0+**: use `Mapped`/`mapped_column` syntax (shown throughout this skill). This is the assumed baseline.
- SQLAlchemy **1.x**: use `Column()`/`relationship()` style. Mark a comment in the code noting the legacy version; do not silently mix styles.

Always prefer 2.0 style for new code. Never mix 1.x `Column()` and 2.0 `mapped_column()` in the same model.

---

## Declarative base and mapped classes

Define one `Base` class per project. All models inherit from it.

```python
from datetime import datetime
from decimal import Decimal
from typing import Optional

from sqlalchemy import DateTime, ForeignKey, Numeric, String, func
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column, relationship


class Base(DeclarativeBase):
    pass


class User(Base):
    __tablename__ = "users"

    id: Mapped[int] = mapped_column(primary_key=True)
    email: Mapped[str] = mapped_column(String(255), unique=True, index=True)
    display_name: Mapped[str] = mapped_column(String(100))
    hashed_password: Mapped[str] = mapped_column(String(255))
    is_active: Mapped[bool] = mapped_column(default=True)
    created_at: Mapped[datetime] = mapped_column(
        DateTime(timezone=True), server_default=func.now()
    )

    orders: Mapped[list["Order"]] = relationship(
        "Order", back_populates="user", lazy="selectin"
    )
```

Key rules:
- `Mapped[T]` without `Optional` means `NOT NULL`. `Mapped[Optional[T]]` means nullable.
- `mapped_column()` without a SQLAlchemy type uses Python type inference — always provide the type explicitly (e.g., `String(255)`) for alembic-specialist to finalize correctly.
- Use `server_default=func.now()` for database-side default timestamps, not Python-side `default=datetime.utcnow`.

---

## Column type guidance

| Python type | SQLAlchemy column type | Notes |
|---|---|---|
| `str` | `String(N)` | Always set length; never bare `String` |
| `Decimal` | `Numeric(precision, scale)` | Never `Float` for money or precise values |
| `datetime` | `DateTime(timezone=True)` | Always set `timezone=True` |
| `int` | `Integer` or `BigInteger` | Use `BigInteger` for large tables (users, events) |
| `bool` | `Boolean` | |
| `UUID` | `Uuid` (SA 2.0+) or `String(36)` | `Uuid` stores as native UUID on PostgreSQL |
| enum | `Enum(MyEnum, native_enum=False)` | `native_enum=False` for DB portability |
| `float` | `Float` | Only for non-monetary approximations (lat/lon, scores) |

---

## Async session setup

```python
# app/db/session.py
from collections.abc import AsyncGenerator

from sqlalchemy.ext.asyncio import AsyncSession, async_sessionmaker, create_async_engine

from app.core.config import settings

engine = create_async_engine(
    settings.DATABASE_URL,
    echo=False,
    pool_pre_ping=True,
    pool_size=10,
    max_overflow=20,
)

AsyncSessionLocal = async_sessionmaker(
    engine,
    class_=AsyncSession,
    expire_on_commit=False,
)


async def get_db() -> AsyncGenerator[AsyncSession, None]:
    async with AsyncSessionLocal() as session:
        try:
            yield session
            await session.commit()
        except Exception:
            await session.rollback()
            raise
```

Use `expire_on_commit=False` so that model attributes remain accessible after a commit without triggering lazy loads — important in async contexts where implicit IO is not allowed.

Use `pool_pre_ping=True` to detect stale connections before use.

---

## Querying patterns

```python
from sqlalchemy import select
from sqlalchemy.ext.asyncio import AsyncSession
from sqlalchemy.orm import selectinload

from app.users.models import User


async def get_user_by_id(db: AsyncSession, user_id: int) -> User | None:
    result = await db.execute(select(User).where(User.id == user_id))
    return result.scalar_one_or_none()


async def get_user_by_email(db: AsyncSession, email: str) -> User | None:
    result = await db.execute(select(User).where(User.email == email))
    return result.scalar_one_or_none()


async def list_users(db: AsyncSession, skip: int = 0, limit: int = 20) -> list[User]:
    result = await db.execute(select(User).offset(skip).limit(limit))
    return list(result.scalars().all())


async def get_user_with_orders(db: AsyncSession, user_id: int) -> User | None:
    result = await db.execute(
        select(User)
        .options(selectinload(User.orders))
        .where(User.id == user_id)
    )
    return result.scalar_one_or_none()


async def create_user(db: AsyncSession, email: str, hashed_password: str, display_name: str) -> User:
    user = User(email=email, hashed_password=hashed_password, display_name=display_name)
    db.add(user)
    await db.flush()  # assigns id without committing; session.commit() happens in get_db()
    return user
```

Use `scalar_one_or_none()` for single-row queries. Use `scalars().all()` for multi-row queries. Use `flush()` inside a unit of work to get the generated PK without committing — let the `get_db()` dependency commit on session exit.

---

## Relationships

Define relationships with **explicit** `lazy` and `cascade` settings. Never rely on defaults.

```python
from sqlalchemy.orm import Mapped, mapped_column, relationship


class User(Base):
    __tablename__ = "users"

    id: Mapped[int] = mapped_column(primary_key=True)

    # One-to-many: user has many orders
    # lazy="selectin" — loads orders automatically with a second SELECT; safe for small collections
    orders: Mapped[list["Order"]] = relationship(
        "Order",
        back_populates="user",
        lazy="selectin",
        cascade="all, delete-orphan",
    )

    # One-to-one: user has one profile
    # lazy="raise" — raises if accessed without explicit selectinload(); prevents accidental N+1
    profile: Mapped[Optional["UserProfile"]] = relationship(
        "UserProfile",
        back_populates="user",
        lazy="raise",
        uselist=False,
        cascade="all, delete-orphan",
    )


class Order(Base):
    __tablename__ = "orders"

    id: Mapped[int] = mapped_column(primary_key=True)
    user_id: Mapped[int] = mapped_column(ForeignKey("users.id", ondelete="CASCADE"))

    # Many-to-one: order belongs to user
    user: Mapped["User"] = relationship("User", back_populates="orders", lazy="selectin")

    # One-to-many: order has many lines
    lines: Mapped[list["OrderLine"]] = relationship(
        "OrderLine",
        back_populates="order",
        lazy="selectin",
        cascade="all, delete-orphan",
    )
```

**Lazy loading strategy guide:**
- `lazy="selectin"` — loads the related collection automatically with a separate `SELECT IN` query. Best for small-to-medium collections that are always needed.
- `lazy="raise"` — raises `MissingGreenlet` if accessed without explicit eager loading. Forces explicit `selectinload()` calls at query time. Best for large or rarely-needed collections to prevent N+1.
- `lazy="dynamic"` — **deprecated in SQLAlchemy 2.0**, do not use.
- Never use `lazy="subquery"` in async context — it uses a subquery that is not supported by async drivers.

---

## Alembic integration

### alembic.ini

Point `script_location` to the `alembic/` directory and configure the async database URL:

```ini
[alembic]
script_location = alembic
sqlalchemy.url = driver://user:pass@localhost/dbname
```

The URL in `alembic.ini` is overridden in `env.py` — do not put production credentials here.

### env.py — async pattern

```python
# alembic/env.py
import asyncio
from logging.config import fileConfig

from alembic import context
from sqlalchemy.ext.asyncio import create_async_engine

from app.core.config import settings
from app.db.base import Base  # import all models so Base.metadata is populated

config = context.config
fileConfig(config.config_file_name)

target_metadata = Base.metadata


def run_migrations_offline() -> None:
    context.configure(
        url=settings.DATABASE_URL,
        target_metadata=target_metadata,
        literal_binds=True,
        dialect_opts={"paramstyle": "named"},
    )
    with context.begin_transaction():
        context.run_migrations()


async def run_async_migrations() -> None:
    connectable = create_async_engine(settings.DATABASE_URL)
    async with connectable.connect() as connection:
        await connection.run_sync(do_run_migrations)
    await connectable.dispose()


def do_run_migrations(connection):
    context.configure(connection=connection, target_metadata=target_metadata)
    with context.begin_transaction():
        context.run_migrations()


def run_migrations_online() -> None:
    asyncio.run(run_async_migrations())


if context.is_offline_mode():
    run_migrations_offline()
else:
    run_migrations_online()
```

**Import all models** before `target_metadata = Base.metadata` so that Alembic sees them in the metadata and can autogenerate accurate migrations. A common pattern is a `app/db/base.py` that imports every model:

```python
# app/db/base.py
from app.db.session import Base  # noqa: F401 — must import Base
from app.users.models import User  # noqa: F401
from app.orders.models import Order, OrderLine  # noqa: F401
```

`Base.metadata` is the source of truth for `--autogenerate`. Any model not imported before `target_metadata` is set will be invisible to Alembic.

---

## Anti-patterns

| Anti-pattern | Problem | Correct approach |
|---|---|---|
| `session.execute(select(...))` in a sync context with `AsyncSession` | Implicit IO in async — raises `MissingGreenlet` | Always `await session.execute(...)` |
| `String` without length | Alembic autogenerate produces `VARCHAR` with no length; some databases use `TEXT` or reject it | Always `String(N)` |
| `lazy="dynamic"` | Deprecated in SQLAlchemy 2.0; raises warning | Use `lazy="selectin"` or explicit `selectinload()` |
| `session.commit()` in a router handler | Couples transport layer to transaction lifecycle | Let the `get_db()` dependency commit on `yield` exit |
| `Float` for monetary values | IEEE 754 rounding errors on financial calculations | `Numeric(precision, scale)` |
| Bare `relationship()` with no `lazy=` | Defaults to `lazy="select"` (sync lazy load) — raises in async context | Always set `lazy="selectin"` or `lazy="raise"` |
| Importing models only in routers | Alembic never sees them; `--autogenerate` misses tables | Always import all models in `app/db/base.py` |
