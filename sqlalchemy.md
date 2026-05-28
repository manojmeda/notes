# Core

## Table of contents
- [Database](#database)
- [Engine](#engine)
- [Connection](#connection)
- [Table metadata](#table-metadata)
- [Declarative ORM](#declarative-orm)

## Database

### Transaction

- Provides a scope around a series of operations
- Transactions follow ACID model
- Relational databases usually use transactions for all operations. If they aren't apparent, it is probably using `autocommit` by default.

### ACID

- `Atomic` - Transactions are atomic i.e all changes which occur either succeed or fail as a single unit and there is no partial state.
-  `Consistency` - Consistency ensures that a transaction can only bring the database from one valid state to another, maintaining all predefined schema rules (unique, foreign key constraint etc.).
- `Isolation` - Ensures that concurrently running transactions do not interfere with each other, making them appear as if they ran sequentially. A database can operate transactions in different isolation levels such as Read committed, repeatable read, serializable each providing different levels of guarantees.
- `Durable` - Guarantees that once the transaction commits, its changes survive permanently in non-volatile storage, even in the event of a total system crash.

## Engine

Engine is factory for connections. It maintains a connection pool internally. Create an engine does not create the connections.


```python
from sqlalchemy import create_engine
engine = create_engine("sqlite://")

# postgres
engine = create_engine("postgresql+psycopg2://user:passwd@host/econdb")

# using URL object
from sqlalchemy import URL
url = URL.create("postgresql+psycopg2", 
                    username="appuser", 
                    password="testpasswd", 
                    host="localhost", 
                    port=5432, 
                    database="econdb")
engine = create_engine(url)
```

## Connection

sqlalchemy.engine.base.Connection is a proxy for DBAPI connection.
connection.connection.driver_connection holds the underlying DBAPI connection object.

```python
conn = engine.connect()
```

```shell
>> conn.connection.driver_connection
<sqlite3.Connection at 0x10813cb80>
```

It has an `execute` method that can run querues using the underlying DBAPI connection and cursor behind the scenes.

To invoke a textual query, use the `sqlalchemy.text()` construct, passed to `execute`.

```python
from sqlalchemy import text
stmt = text("select 'hello world' as greeting")
result = conn.execute(stmt)
```

The Result object is similar to a DBAPI cursor, but has more methods, transformations and automation.

```shell
>> result
<sqlalchemy.engine.cursor.CursorResult at 0x108982990>
```

Some of the methods include:

`first()` - Returns the first row (or None if no row) and close the result set. Returns `sqlalchemy.engine.row.Row` object. It acts mostly like a named tuple. It also has a dictionary interface available via an accessor called `._mapping`

```python
# first()
from sqlalchemy import text
stmt = text("select table_schema, table_name from information_schema.tables")
result = conn.execute(stmt)
row = result.first()
print(f"{row.table_schema}.{row.table_name}")
print(f"{row._mapping['table_schema']}.{row_mapping['table_name']}")

# Different ways of iterating through multiple rows
for row in result:
    print(row)

for table_schema, table_name in result:
    print(f"Table schema: {table_schema}, Table name: {table_name}")

rows = result.all()

# scalars() returns the first column of each row
for table_schema in result.scalars():
    print(table_schema)

list_of_scalars = result.scalars().all()
```

Connection has a `close` method, which does a rollback and releases the connection back to the pool.

```python
conn.close()
```

### Using context managers

```python
with engine.connect() as conn:
    conn.execute(text("create table countries (id serial primary key, name text, iso_code char(3))"))
    stmt = text("insert into countries (name, iso_code) values (:name, :iso_code)")
    conn.execute(stmt, {"name": "India", "iso_code": "IND"})
    conn.commit()

with engine.begin() as conn:
    conn.execute(text("create table countries (country_id serial primary key, name text, iso_code char(3))"))
    stmt = text("insert into countries (name, iso_code) values (:name, :iso_code)")
    conn.execute(stmt, {"name": "India", "iso_code": "IND"})

with engine.connect as conn:
    with conn.begin():
        conn.execute(text("create table countries (id serial primary key, name text, iso_code char(3))"))
        stmt = text("insert into countries (name, iso_code) values (:name, :iso_code)")
        conn.execute(stmt, {"name": "India", "iso_code": "IND"})
```

## Table metadata

```python
from sqlalchemy import Table, Column, MetaData
from sqlalchemy import Integer, String

metadata = MetaData()
countries_table = Table("countries", 
                    metadata, 
                    Column("id", Integer, primary_key=True),
                    Column("name", String), 
                    Column("iso_code", String(3)))
```

In practice the table metadata is not used much. In most ORM applications, `Table` is constructed indirectly using a style known as `Declarative ORM`

## Declarative ORM

- Integrates well with IDE typing tools, mypy etc.
- Creates typed SQL statements and result sets
- Integrates with python dataclasses
- Can be used with sqlalchemy core constructs (engine, connection ,execute etc.)

```python
from sqlalchemy.orm import MappedAsDataclass, DeclarativeBase, Mapped, mapped_column
from sqlalchemy import String
from datetime import datetime

class Base(MappedAsDataclass, DeclarativeBase):
    pass

class Country(Base):
    __tablename__ = "countries"

    country_id: Mapped[int] = mapped_column(primary_key=True, init=False)
    name: Mapped[str]
    iso_code: Mapped[str] = mapped_column(String(3))
    created: Mapped[datetime] = mapped_column(default_factory=datetime.utcnow)
```

- The `Mapped` type indicates to sqlalchemy that it is mapped to a database column.
- The `mapped_column` construct is optional and allows additional details about the database column
- Since `Country` is also a `dataclass`, it has access to methods like `__init__` and `__repr__`
- The `Table` objects are created internally and can be inspected using the `__table__` attribute 

```shell
>> Country.__table__
Table('countries', MetaData(), Column('country_id', Integer(), table=<countries>, primary_key=True, nullable=False), Column('name', String(), table=<countries>, nullable=False), Column('iso_code', String(length=3), table=<countries>, nullable=False), Column('created', DateTime(), table=<countries>, nullable=False), schema=None)
```

- Whether the `Table` objects are made directly or through declarative ORM, we can run CREATE TABLE statements using the method `create_all`

```python
from sqlalchemy import create_engine, URL, String, ForeignKey, DateTime
from sqlalchemy import text, func
from sqlalchemy.orm import MappedAsDataclass, DeclarativeBase
from sqlalchemy.orm import Mapped, mapped_column
from datetime import datetime

url = URL.create("postgresql+psycopg2", 
                    username="appuser", 
                    password="testpasswd", 
                    host="localhost", 
                    port=5432, 
                    database="econdb")
engine = create_engine(url, echo=True)

class Base(MappedAsDataclass, DeclarativeBase):
    pass

class Country(Base):
    __tablename__ = "countries"

    country_id: Mapped[int] = mapped_column(primary_key=True, init=False)
    name: Mapped[str] = mapped_column()
    iso_code: Mapped[str] = mapped_column(String(3))
    created_at: Mapped[datetime] = mapped_column(DateTime(timezone=True), init=False, server_default=text("timezone('utc', now())"))
    updated_at: Mapped[datetime] = mapped_column(DateTime(timezone=True), init=False, server_default=text("timezone('utc', now())"), onupdate=text("timezone('utc', now())"))

class Indicator(Base):
    __tablename__ = "indicators"

    indicator_id: Mapped[int] = mapped_column(primary_key=True, init=False)
    name: Mapped[str]
    code: Mapped[str]
    category: Mapped[str | None]
    description: Mapped[str | None]

class Observation(Base):
    __tablename__ = "observations"

    observation_id: Mapped[int] = mapped_column(primary_key=True, init=False)
    country_id: Mapped[int] = mapped_column(ForeignKey("countries.country_id"))
    indicator_id: Mapped[int] = mapped_column(ForeignKey("indicators.indicator_id"))
    value: Mapped[float]

with engine.begin() as conn:
    Base.metadata.create_all(conn)
```

## Insert

The `insert` function starts with the single argument, the table or table-representing ORM entity, that is the target of the insert.

```python
from sqlalchemy import insert

insert_stmt = insert(Country).values(name="India", iso_code="IND")

with engine.begin() as conn:
    conn.execute(insert_stmt)
```

You can pass the values directly to `execute` as well. You can pass a dictionary or a list of dictionaries.

```python
from sqlalchemy import insert

insert_stmt = insert(Country)

with engine.begin() as conn:
    conn.execute(insert_stmt, {"name": "India", "iso_code": "IND"})

with engine.begin() as conn:
    conn.execute(insert_stmt, [{"name": "India", "iso_code": "IND"},
                               {"name": "China", "iso_code": "CHN"}])
```

## Select

The arguments to `select()` are columns, tables (or a corresponding ORM class), other so called selectables such as aliases or subqueries, and sql expressions.

```python
from sqlalchemy import select

with engine.connect() as conn:
    stmt = select(Country.name, Country.iso_code)
    result = conn.execute(stmt)
    for row in result:
        print(f"Country name: {row.name}, Iso code: {row.iso_code}")

with engine.connect() as conn:
    stmt = select(Country) # selects all columns in the table
    result = conn.execute(stmt)
    for row in result:
        print(row)
```

## Joins

```python
with engine.connect() as conn:
    stmt = (
        select(
            Country.name.label("country_name"),
            Indicator.name.label("indicator_name"),
            Observation.value.label("observation_value"),
        )
        .join_from(Observation, Country)
        .join_from(Observation, Indicator)
    )
    result = conn.execute(stmt)
    for row in result:
        print(row)
```

## SQL expressions

- class attributes like Country.name are also the foundation of sql expressions. The `__eq__()` operator was overridden to produce an expression object.
- The generated expressions use **bound parameters** for all literal values
- The values for the parameters are embedded and come out during `Connection.execute()`
- The parameters can be seen using the `compile` method

```shell
>> stmt = Country.name == "India"

>> stmt
<sqlalchemy.sql.elements.BinaryExpression object at 0x10aa18b90>

>> print(stmt)
countries.name = :name_1

>> print((Country.name == "India").compile(compile_kwargs={"literal_binds": True}))
countries.name = 'India'
```

### `in` expressions

```shell
>> print(Country.name.in_(["India", "China"]))
countries.name IN (__[POSTCOMPILE_name_1])
```

### `where` clause

```shell
>> print(select(Country.name).where(Country.name.in_(["India", "China"])))
SELECT countries.name
FROM countries
WHERE countries.name IN (__[POSTCOMPILE_name_1])
```

The `where` clause can be use multiple times, criteria is joined by `AND`

```shell
>> print(select(Country.name).where(Country.name.in_(["India", "China"])).where(Country.country_id > 1))
SELECT countries.name
FROM countries
WHERE countries.name IN (__[POSTCOMPILE_name_1]) AND countries.country_id > :country_id_1
```

### order_by

```shell
>> print(select(Country.name).where(Country.name.in_(["India", "China"])).where(Country.country_id > 1).order_by(Country.name))
SELECT countries.name
FROM countries
WHERE countries.name IN (__[POSTCOMPILE_name_1]) AND countries.country_id > :country_id_1 ORDER BY countries.name
```

### add_columns

```shell
>> stmt = select(Country.name).where(Country.name.in_(["India", "China"])).where(Country.country_id > 1).ord
        ⋮ er_by(Country.name)

>> stmt = stmt.add_columns(literal("country id: ") + Country.country_id)

>> print(stmt)
SELECT countries.name, :param_1 + countries.country_id AS anon_1
FROM countries
WHERE countries.name IN (__[POSTCOMPILE_name_1]) AND countries.country_id > :country_id_1 ORDER BY countries.name
```

## Session

- `Session` is like `Connection` and provides mechanisms to interact with database, transactions etc.
- `sessionmaker()` is a factory to create sessions.

```python
from sqlalchemy.orm import sessionmaker

Session = sessionmaker(bind=engine)
session = Session()
result = session.execute(text("SELECT * FROM countries"))
for row in result:
    print(row)
session.close()

# using context manager
with Session() as session:
    result = session.execute(text("SELECT * FROM countries"))
    for row in result:
        print(row)

# insert
from sqlalchemy import insert

with Session() as session:
    result = session.execute(insert(Country), {"name": "United States of America", "iso_code": "USA"})
    session.commit()

with Session.begin() as session:
    result = session.execute(insert(Country), {"name": "United States of America", "iso_code": "USA"})
```

- Once the `Session` has established a connection, it is considered to be in a transaction. It uses the same `Connection` object until the transaction ends.
- To end the transaction and release the `Connection`, call `Session.commit()`, `Session.rollback()` or `Session.close()`

- The ORM way of doing things is as shown below:

```python
session = Session()
sl = Country(name="Sri Lanka", iso_code="SL")
session.add(sl) # doesn't insert into DB yet
print(session.new)
```

The process by which session emits insert, update and delete statements for objects is known as `flush`

The `flush` process occurs when:
- We run any sql statements with `execute()` or similar, before that sql statement is executed (known as `autoflush`)
- When any ORM instance runs a process known as `lazy loading` (also part of `autoflush`)
- When we call explicit method `Session.flush()`
- When we commit the transaction with `Session.commit()`, before the actual commit occurs

```python
session = Session()
stmt = select(Country).where(Country.iso_code == "IND")
result = session.execute(stmt)
row = result.first()
print(row[0]) # prints the Country object
```

The `Indentity Map` is a weak mapping of objects keyed to their class/primary key identity

```shell
>> dict(session.identity_map)
{(__main__.Country,
  (6,),
  None): Country(country_id=6, name='Sri Lanka', iso_code='SL', created_at=datetime.datetime(2026, 5, 12, 11, 45, 31, 638669, tzinfo=datetime.timezone.utc), updated_at=datetime.datetime(2026, 5, 12, 11, 45, 31, 638669, tzinfo=datetime.timezone.utc)),
 (__main__.Country,
  (7,),
  None): Country(country_id=7, name='Australia', iso_code='AUS', created_at=datetime.datetime(2026, 5, 12, 11, 55, 40, 837923, tzinfo=datetime.timezone.utc), updated_at=datetime.datetime(2026, 5, 12, 11, 55, 40, 837923, tzinfo=datetime.timezone.utc)),
 (__main__.Country,
  (3,),
  None): Country(country_id=3, name='India', iso_code='IND', created_at=datetime.datetime(2026, 5, 10, 11, 53, 39, 598085, tzinfo=datetime.timezone.utc), updated_at=datetime.datetime(2026, 5, 10, 11, 53, 39, 598085, tzinfo=datetime.timezone.utc))}
```

- `Session.scalars()`

```python
stmt = select(Country).where(Country.iso_code == "IND")
result = session.scalars(stmt)
country = result.first()
print(country)
```

- With objects both `pending` and `persistent` states, running any sql operations as well as any `Session.flush()` or `Session.commit()` call will `flush` all those changes before proceeding

```python
print(session.new)
print(session.dirty)
```

- `Session.query()`

Legacy 1.x syntax and is internally deprecated. Use `execute` with `select`.

```python
countries = session.query(Country).all()
countries = session.query(Country).order_by(Country.country_id).all()
countries = session.query(Country).order_by(Country.country_id.desc()).all()
country = session.query(Country).filter_by(id=3).first()
country = session.query(Country).filter_by(id=3).one_or_none()
```

- Explicit transactions are always present
- The session maintains a cached set of transaction state, consisting of rows
- A row is typically present in the session if it was selector or inserted in the span of that transaction
- Objects, when associated with session, are proxies for rows, represented uniquely on primary key identity.
- Changes to objects are pushed out to rows before each query, and at transaction end, using unit of work.
- An object is said to be `persistent` when it acts as a proxy to a row present in the transaction. This row is normally always known as a result of a select or an insert.
- With no transaction present, the state of the objects is expired. There is no view of the database data other than via a transaction.
- An object that is outside of the session, not yet corresponding to any row, is said to be `transient`.
- An object that is inside the session, but not yet corresponding to any row, is said to be `pending`.
- A previously persistent object that is no longer associated with a session is said to be `detached`. Detachment is useful for caching, but not much else.

### Unit of work

- Unit of work lazily flushes only those rows/columns that have changed, ordering to maintain consistency.