---
name: sqlalchemy-patterns
description: |
  SQLAlchemy ORM patterns for Flask: Flask-SQLAlchemy extension setup, declarative models with db.Model, synchronous db.session queries, relationships with explicit lazy loading, Flask-Migrate integration for schema migrations. Used by flask-architect (model definitions) and flask-migrate-specialist (column finalization and migration). Activated automatically by flask-plugin/stack.md.

  Use this skill to:
  - Write Flask-SQLAlchemy models with db.Model base and properly typed columns.
  - Query the database with db.session and SQLAlchemy 2.0-style select() statements.
  - Define relationships with explicit lazy loading strategy.
  - Integrate Flask-Migrate for Alembic-based migrations managed via flask db commands.

  Do NOT use this skill for:
  - Flask routing and template/API patterns — see flask-plugin:flask-conventions.
  - Migration execution (flask db migrate, flask db upgrade) — that's flask-migrate-specialist's job.
  - Python idioms — see python-foundation:python-conventions.
---

# SQLAlchemy Patterns for Flask

## Detection

Read `pyproject.toml` or `requirements.txt` before writing any model code:

```bash
grep -E "flask.sqlalchemy|sqlalchemy" requirements.txt pyproject.toml
```

Determine the Flask-SQLAlchemy version:

- **Flask-SQLAlchemy 3.x** (released 2022+): Uses SQLAlchemy 2.0 under the hood. Supports `Mapped`/`mapped_column` style with `db.Model`. This is the baseline for new projects.
- **Flask-SQLAlchemy 2.x** (legacy): Uses `db.Column()` style. Still common in existing projects.

Always use **Flask-SQLAlchemy 3.x style** for new code. When working in an existing project using 2.x style throughout, match the existing style. Never mix `db.Column()` and `mapped_column()` in the same model.

---

## Extension setup

Define extensions at module level in `app/extensions.py` and call `.init_app(app)` in the factory. This avoids circular imports and allows multiple app instances for testing.

```python
# app/extensions.py
from flask_migrate import Migrate
from flask_sqlalchemy import SQLAlchemy

db = SQLAlchemy()
migrate = Migrate()
```

Initialize in the factory:

```python
# app/__init__.py
from app.extensions import db, migrate


def create_app(config_name: str = "development") -> Flask:
    app = Flask(__name__)
    app.config.from_object(config_by_name[config_name])

    db.init_app(app)
    migrate.init_app(app, db)

    return app
```

---

## Model definition

### Flask-SQLAlchemy 3.x with Mapped (preferred for new code)

```python
from datetime import datetime
from decimal import Decimal
from typing import Optional

from sqlalchemy import DateTime, ForeignKey, Numeric, String, func
from sqlalchemy.orm import Mapped, mapped_column, relationship

from app.extensions import db


class User(db.Model):
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
        "Order", back_populates="user", lazy="select"
    )

    def __repr__(self) -> str:
        return f"<User id={self.id} email={self.email!r}>"
```

Key rules:
- `Mapped[T]` without `Optional` means `NOT NULL`. `Mapped[Optional[T]]` means nullable.
- Always provide the SQLAlchemy type explicitly (e.g., `String(255)`) — flask-migrate-specialist uses this to finalize column lengths, precision, and constraints.
- Use `server_default=func.now()` for database-side default timestamps, not `default=datetime.utcnow` (Python-side defaults are not reflected in DB schema).

### Flask-SQLAlchemy 2.x with db.Column (legacy)

```python
from app.extensions import db


class User(db.Model):
    __tablename__ = "users"

    id = db.Column(db.Integer, primary_key=True)
    email = db.Column(db.String(255), unique=True, nullable=False)
    display_name = db.Column(db.String(100), nullable=False)
    hashed_password = db.Column(db.String(255), nullable=False)
    is_active = db.Column(db.Boolean, default=True)
    created_at = db.Column(db.DateTime(timezone=True), server_default=db.func.now())

    orders = db.relationship("Order", back_populates="user", lazy="select")

    def __repr__(self) -> str:
        return f"<User id={self.id} email={self.email!r}>"
```

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

## Querying patterns (synchronous)

Flask-SQLAlchemy uses synchronous sessions. All queries are blocking — no `await`, no async generators.

```python
from sqlalchemy import select

from app.extensions import db
from app.users.models import User


def get_user_by_id(user_id: int) -> User | None:
    return db.session.get(User, user_id)


def get_user_by_email(email: str) -> User | None:
    return db.session.execute(
        select(User).where(User.email == email)
    ).scalar_one_or_none()


def list_users(skip: int = 0, limit: int = 20) -> list[User]:
    return list(
        db.session.execute(select(User).offset(skip).limit(limit)).scalars().all()
    )


def create_user(email: str, hashed_password: str, display_name: str) -> User:
    user = User(email=email, hashed_password=hashed_password, display_name=display_name)
    db.session.add(user)
    db.session.flush()  # assigns id without committing
    return user


def delete_user(user: User) -> None:
    db.session.delete(user)
```

**Session lifecycle in Flask:** Flask-SQLAlchemy automatically calls `db.session.remove()` at the end of each request via `teardown_appcontext`. This closes the session and returns the connection to the pool. You do not need to call `db.session.close()` manually in view functions.

**When to commit:** call `db.session.commit()` in the view function or service after all writes are complete for the request. Use `db.session.flush()` inside a unit of work to get the generated PK without committing.

```python
@orders_bp.route("/", methods=["POST"])
@login_required
def create_order_view():
    data = order_create_schema.load(request.get_json())
    order = create_order(data)
    db.session.commit()
    return jsonify(order_schema.dump(order)), 201
```

---

## Relationships

Define relationships with **explicit** `lazy` and `cascade` settings.

```python
from sqlalchemy.orm import Mapped, mapped_column, relationship


class User(db.Model):
    __tablename__ = "users"

    id: Mapped[int] = mapped_column(primary_key=True)

    # One-to-many: user has many orders
    # lazy="select" — loads orders with a separate SELECT when accessed (default)
    orders: Mapped[list["Order"]] = relationship(
        "Order",
        back_populates="user",
        lazy="select",
        cascade="all, delete-orphan",
    )

    # One-to-one: user has one profile
    # lazy="joined" — loads profile with a JOIN in the same query
    profile: Mapped[Optional["UserProfile"]] = relationship(
        "UserProfile",
        back_populates="user",
        lazy="joined",
        uselist=False,
        cascade="all, delete-orphan",
    )


class Order(db.Model):
    __tablename__ = "orders"

    id: Mapped[int] = mapped_column(primary_key=True)
    user_id: Mapped[int] = mapped_column(ForeignKey("users.id", ondelete="CASCADE"))

    user: Mapped["User"] = relationship("User", back_populates="orders", lazy="select")
    lines: Mapped[list["OrderLine"]] = relationship(
        "OrderLine",
        back_populates="order",
        lazy="select",
        cascade="all, delete-orphan",
    )
```

**Lazy loading strategy guide (synchronous Flask context):**
- `lazy="select"` — loads the related collection with a separate `SELECT` when the attribute is first accessed. This is the SQLAlchemy default and is safe in synchronous Flask (unlike async FastAPI, where it can block the event loop).
- `lazy="joined"` — loads the relation with a JOIN in the same query. Best for one-to-one relations or small, always-needed collections.
- `lazy="subquery"` — loads the relation with a subquery. Valid in synchronous Flask (unlike async where it is not supported). Useful for loading collections alongside the parent.
- `lazy="dynamic"` — **deprecated in SQLAlchemy 2.0**. Do not use. Replace with explicit `select()` queries.

To prevent N+1 queries on list endpoints, use `options(joinedload(...))` or `options(selectinload(...))` at query time:

```python
from sqlalchemy.orm import joinedload, selectinload


def list_users_with_orders() -> list[User]:
    return list(
        db.session.execute(
            select(User).options(selectinload(User.orders))
        ).scalars().all()
    )
```

---

## Flask-Migrate integration

Flask-Migrate wraps Alembic. Running `flask db init` scaffolds the `migrations/` directory including `migrations/env.py` and `migrations/alembic.ini`. Do not hand-write `env.py` — Flask-Migrate generates a synchronous configuration automatically.

```
migrations/
    alembic.ini
    env.py          — generated by flask db init; imports db.metadata
    script.py.mako
    versions/       — generated migration scripts live here
```

For autogenerate to detect all models, Flask-Migrate's `env.py` must import all model modules before `target_metadata = db.metadata`. A common pattern is to import all models in `app/models/__init__.py` or in the app factory before `db.init_app(app)`:

```python
# app/__init__.py
def create_app(config_name: str = "development") -> Flask:
    app = Flask(__name__)
    app.config.from_object(config_by_name[config_name])

    db.init_app(app)
    migrate.init_app(app, db)

    # Import models so Flask-Migrate sees them in db.metadata
    with app.app_context():
        from app.users import models as _user_models  # noqa: F401
        from app.orders import models as _order_models  # noqa: F401

    _register_blueprints(app)
    return app
```

Flask-migrate-specialist runs `flask db migrate` and `flask db upgrade`. Flask-architect only **defines the models**. Never call `flask db` commands from flask-architect.

---

## Anti-patterns

| Anti-pattern | Problem | Correct approach |
|---|---|---|
| `String` without length | Alembic autogenerate produces `VARCHAR` with no length; PostgreSQL uses `TEXT` | Always `String(N)` |
| `lazy="dynamic"` | Deprecated in SQLAlchemy 2.0; raises a warning | Use `lazy="select"` with explicit `selectinload()` for large collections |
| `Float` for monetary values | IEEE 754 rounding errors on financial calculations | `Numeric(precision, scale)` |
| `db.session.commit()` in every helper function | Makes unit testing harder; scattered transaction boundaries | Commit at the end of the request in the view or service layer |
| Importing models only in routers | Flask-Migrate's `env.py` never sees them; `--autogenerate` misses tables | Import all models in the app factory or `app/models/__init__.py` |
| `default=datetime.utcnow` | Python-side default — not reflected in DB schema; utcnow is deprecated | `server_default=func.now()` with `DateTime(timezone=True)` |
| Mixing `db.Column()` and `mapped_column()` | Produces inconsistent metadata; confuses tooling | Use one style per project; prefer `mapped_column()` for new code |
