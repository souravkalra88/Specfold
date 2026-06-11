# SpecFold v2 — Python Persistence from Spec

You implement **database models, repositories/data access, and Alembic migrations** from `specfold/<feature>/spec.md` for the **current slice** identified in `progress.md`.

---

## Before Coding

1. If **`specfold/<feature>/repo_kb.md`** exists or is attached, **read it first**.
2. If **`specfold/<feature>/progress.md`** exists, **read it** to identify the **current slice**.
3. Extract **schema**, **constraints**, **indexes**, and **migration/backfill** needs from the spec for the current slice.
4. Inspect existing **models**, **session** usage, and **Alembic** layout (prefer paths from `repo_kb.md` when listed).

---

## While Coding

### Follow Rules
- **`.cursor/rules/python.mdc`** (ORM, migrations, transactions, N+1)
- **`.cursor/rules/specfold-v2.mdc`** — Incremental slicing, backward compatibility
- Global SpecFold rules: scope discipline, no invented requirements

### SQLAlchemy Best Practices

#### Models
- **Declarative base:** Use SQLAlchemy 2.0 style
  ```python
  from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column
  from sqlalchemy import String, Numeric, DateTime
  from datetime import datetime
  from uuid import UUID, uuid4
  
  class Base(DeclarativeBase):
      pass
  
  class Order(Base):
      __tablename__ = "orders"
      
      id: Mapped[UUID] = mapped_column(primary_key=True, default=uuid4)
      user_id: Mapped[UUID] = mapped_column(nullable=False, index=True)
      total_amount: Mapped[Decimal] = mapped_column(Numeric(10, 2), nullable=False)
      status: Mapped[str] = mapped_column(String(20), nullable=False, default="pending")
      created_at: Mapped[datetime] = mapped_column(DateTime, nullable=False, default=datetime.utcnow)
      
      # Relationships
      items: Mapped[list["OrderItem"]] = relationship(back_populates="order")
  ```
- **Type hints:** Use `Mapped[]` for all columns
- **Indexes:** Add for query patterns
  ```python
  user_id: Mapped[UUID] = mapped_column(nullable=False, index=True)
  status: Mapped[str] = mapped_column(String(20), index=True)
  ```
- **Composite indexes:** For common queries
  ```python
  __table_args__ = (
      Index('idx_user_status', 'user_id', 'status'),
  )
  ```

#### Repositories
- **Separate data access:** Keep ORM queries in repository layer
  ```python
  class OrderRepository:
      def __init__(self, session: AsyncSession):
          self.session = session
      
      async def create(self, order: Order) -> Order:
          self.session.add(order)
          await self.session.flush()
          return order
      
      async def get_by_id(self, order_id: UUID) -> Order | None:
          return await self.session.get(Order, order_id)
      
      async def list_by_user(self, user_id: UUID, page: int = 1, limit: int = 20) -> list[Order]:
          offset = (page - 1) * limit
          result = await self.session.execute(
              select(Order)
              .where(Order.user_id == user_id)
              .order_by(Order.created_at.desc())
              .limit(limit)
              .offset(offset)
          )
          return list(result.scalars().all())
  ```
- **Avoid N+1 queries:** Use `selectinload` or `joinedload`
  ```python
  result = await self.session.execute(
      select(Order)
      .options(selectinload(Order.items))
      .where(Order.user_id == user_id)
  )
  ```

#### Transactions
- **Service layer:** Handle transactions in service, not repository
  ```python
  class OrderService:
      async def create_order(self, order_data: CreateOrderRequest) -> Order:
          async with self.session.begin():  # Transaction starts
              order = Order(...)
              order = await self.order_repo.create(order)
              
              for item_data in order_data.items:
                  item = OrderItem(order_id=order.id, ...)
                  await self.item_repo.create(item)
              
              await self.session.commit()  # Transaction commits
              return order
  ```

### Alembic Migrations

#### Creating Migrations
- **Autogenerate:** Let Alembic detect changes
  ```bash
  alembic revision --autogenerate -m "Add orders table"
  ```
- **Review migration:** Always review generated SQL
  ```python
  def upgrade() -> None:
      op.create_table(
          'orders',
          sa.Column('id', sa.UUID(), nullable=False),
          sa.Column('user_id', sa.UUID(), nullable=False),
          sa.Column('total_amount', sa.Numeric(10, 2), nullable=False),
          sa.Column('status', sa.String(20), nullable=False),
          sa.Column('created_at', sa.DateTime(), nullable=False),
          sa.PrimaryKeyConstraint('id')
      )
      op.create_index('idx_orders_user_id', 'orders', ['user_id'])
  ```

#### Backward Compatibility (Zero-Downtime)
- **Additive migrations:** Add columns as nullable first
  ```python
  # Step 1: Add nullable column
  def upgrade():
      op.add_column('orders', sa.Column('total_amount_v2', sa.Numeric(10, 2), nullable=True))
  
  # Step 2 (separate migration): Backfill data
  def upgrade():
      op.execute("UPDATE orders SET total_amount_v2 = total_amount WHERE total_amount_v2 IS NULL")
  
  # Step 3 (separate migration): Make not-nullable, drop old column
  def upgrade():
      op.alter_column('orders', 'total_amount_v2', nullable=False)
      op.drop_column('orders', 'total_amount')
  ```
- **Multi-step deploys:** For breaking changes
  1. Deploy code that writes to both old and new columns
  2. Backfill data
  3. Deploy code that reads from new column
  4. Drop old column

- **Index creation:** Use `CONCURRENTLY` for large tables (PostgreSQL)
  ```python
  from alembic import op
  
  def upgrade():
      op.create_index(
          'idx_orders_status',
          'orders',
          ['status'],
          postgresql_concurrently=True
      )
  ```

### Backward Compatibility
- **Nullable columns:** For new fields (or use default values)
- **No column renames:** Add new column, keep old one temporarily
- **No column drops:** Mark as deprecated, drop in future migration
- **Indexes:** Add indexes concurrently (PostgreSQL)

### Performance
- **Indexes:** Add for WHERE, ORDER BY, JOIN columns
- **Composite indexes:** For multi-column queries
- **Partial indexes:** For filtered queries (PostgreSQL)
  ```python
  op.create_index(
      'idx_orders_pending',
      'orders',
      ['user_id'],
      postgresql_where=sa.text("status = 'pending'")
  )
  ```
- **N+1 avoidance:** Use `selectinload`, `joinedload`
- **Pagination:** Always use `limit` + `offset` or cursor-based pagination

### Testing
- **pytest:** Test repository methods with test database
  ```python
  @pytest.mark.asyncio
  async def test_create_order__SF_001(db_session):
      repo = OrderRepository(db_session)
      order = Order(user_id=uuid4(), total_amount=Decimal('49.99'))
      
      created = await repo.create(order)
      
      assert created.id is not None
      assert created.total_amount == Decimal('49.99')
  ```
- **Migration tests:** Test up/down migrations
  ```python
  def test_migration_upgrade(alembic_config):
      command.upgrade(alembic_config, 'head')
      # Verify tables exist
  
  def test_migration_downgrade(alembic_config):
      command.downgrade(alembic_config, '-1')
      # Verify tables dropped
  ```
- **Coverage:** >80% for repositories

---

## Output

### Summary Format

```markdown
## Slice [N] Persistence Implementation: [Slice Name]

### Models Added/Modified
- `models/order.py` (new): Order model
- `models/order_item.py` (new): OrderItem model

### Repositories Added/Modified
- `repositories/order_repository.py` (new)

### Migrations Added
- `alembic/versions/001_add_orders_table.py`

### Acceptance Criteria Covered (Data)
- ✅ SF-005: Order stored in database (PostgreSQL)

### How to Verify
```bash
# Run migrations
alembic upgrade head

# Run tests
pytest tests/repositories/test_order_repository.py

# Verify schema
psql -d app_db -c "\d orders"
```

### Files Touched
- `app/models/order.py` (new)
- `app/repositories/order_repository.py` (new)
- `alembic/versions/001_add_orders_table.py` (new)
- `tests/repositories/test_order_repository.py` (new)

### Database Schema
```sql
CREATE TABLE orders (
    id UUID PRIMARY KEY,
    user_id UUID NOT NULL,
    total_amount NUMERIC(10, 2) NOT NULL,
    status VARCHAR(20) NOT NULL DEFAULT 'pending',
    created_at TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_orders_user_id ON orders(user_id);
CREATE INDEX idx_orders_status ON orders(status);
```

### Deploy Notes
1. Run migration: `alembic upgrade head`
2. No downtime required (additive changes only)
3. No data backfill needed

### Next Steps
1. Review models and migrations
2. Run tests: `pytest tests/repositories/`
3. Run migration in dev: `alembic upgrade head`
4. Update `progress.md`: Mark Slice [N] complete
5. Deploy to staging → production
```

---

## Traceability

Reference **`SF-…`** IDs in:
- **Model docstrings:** `# SF-005: Order entity with items`
- **Test names:** `test_create_order_in_database__SF_005()`
- **Migration comments:** `# SF-005: Add orders table`

---

## Quality Checklist

Before finishing:

- [ ] Implemented **only** the current slice
- [ ] **Type hints** on all model columns (`Mapped[]`)
- [ ] **Indexes** for query patterns
- [ ] **Relationships** defined correctly
- [ ] **Repository layer** for data access (no ORM in services)
- [ ] **Transactions** handled in service layer
- [ ] **Migrations** are backward-compatible (additive)
- [ ] **Tests** written (>80% coverage)
- [ ] **No N+1 queries** (use eager loading)
- [ ] **SF-…** IDs in models/tests

---

## Example Invocation

```
@.cursor/agents/python_persistence_from_spec.md
@specfold/user-orders/spec.md
@specfold/user-orders/progress.md
@specfold/user-orders/repo_kb.md

Implement current slice data layer (Slice 1: Order and OrderItem models)
Use SQLAlchemy 2.0 with Alembic migrations.
```
