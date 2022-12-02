# Multiple versions of alembic migration in a single schema

In this proof of concept, I want to use multiple alembic migrations in a single schema.

A reasonable use case is:

- You have a system with multiple plugins
- Each plugin has a different set of tables, independent from the other.
- You want to maintain the plugin versioning as well as their database schema migration.

# Some technology we use

- [SQLALchemy](https://www.sqlalchemy.org/): Python database toolkit and ORM
- [Alembic](https://alembic.sqlalchemy.org/en/latest/): Lightweight database migration tool for usage with the SQLAlchemy

# How to use

```bash
# prepare venv
if [ ! -d venv ]
then
    python -m venv venv
fi
source venv/bin/activate

# install packages
pip install -r requirements.txt

# generate migrations for `one` and `two` modules
./generate.sh

# do migration for `one` and `two` modules
./migrate.sh
```

# What to expect

- There will be two migration files:
    - `/one/alembic/versions/<migration-message>.py`
    - `/two/alembic/versions/<migration-message>.py`
- There will be four tables in `anu.db`:
    - `alembic_version_one`
    - `alembic_version_two`
    - `books`
    - `users`

# How to set up 

First, prepare env and install the necessary packages (SQLAlchemy and Alembic)

```bash
# prepare venv
if [ ! -d venv ]
then
    python -m venv venv
fi
source venv/bin/activate

# install packages
pip install -r requirements.txt
```

Next, create two packages, `one` and `two`. The directory structure is as follows:

```
one/
  __init__.py
  model/
    book.py
two/
  __init__.py
  model/
    user.py
```

In `book.py` and `user.py`, you define SQLAlchemy models as follows:

```python
# location: one/model/book.py
from sqlalchemy import Column
from sqlalchemy import ForeignKey
from sqlalchemy import Integer
from sqlalchemy import String
from sqlalchemy.orm import declarative_base
from sqlalchemy.orm import relationship

Base = declarative_base()

class Book(Base):
    __tablename__ = "books"

    id = Column(Integer, primary_key=True)
    title = Column(String(30))
    author = Column(String)
```

```python
# location: two/model/user.py
from sqlalchemy import Column
from sqlalchemy import ForeignKey
from sqlalchemy import Integer
from sqlalchemy import String
from sqlalchemy.orm import declarative_base
from sqlalchemy.orm import relationship

Base = declarative_base()

class User(Base):
    __tablename__ = "users"

    id = Column(Integer, primary_key=True)
    name = Column(String(30))
    address = Column(String)
```

Once you have the models, you can init alembic:

```bash
alembic init one/alembic
alembic init two/alembic
```

Now you should have a file named `alembic.ini` and two migration versions named `one/alembic` and `two/alembic`.

Let's update your `alembic.ini` as follow:

```ini
[alembic]

# template used to generate migration file names; The default value is %%(rev)s_%%(slug)s
# Uncomment the line below if you want the files to be prepended with date and time
# see https://alembic.sqlalchemy.org/en/latest/tutorial.html#editing-the-ini-file
# for all available tokens
# file_template = %%(year)d_%%(month).2d_%%(day).2d_%%(hour).2d%%(minute).2d-%%(rev)s_%%(slug)s

# sys.path path, will be prepended to sys.path if present.
# defaults to the current working directory.
prepend_sys_path = .

# timezone to use when rendering the date within the migration file
# as well as the filename.
# If specified, requires the python-dateutil library that can be
# installed by adding `alembic[tz]` to the pip requirements
# string value is passed to dateutil.tz.gettz()
# leave blank for localtime
# timezone =

# max length of characters to apply to the
# "slug" field
# truncate_slug_length = 40

# set to 'true' to run the environment during
# the 'revision' command, regardless of autogenerate
# revision_environment = false

# set to 'true' to allow .pyc and .pyo files without
# a source .py file to be detected as revisions in the
# versions/ directory
# sourceless = false

# version location specification; This defaults
# to alembic/versions.  When using multiple version
# directories, initial revisions must be specified with --version-path.
# The path separator used here should be the separator specified by "version_path_separator" below.
# version_locations = %(here)s/bar:%(here)s/bat:alembic/versions

# version path separator; As mentioned above, this is the character used to split
# version_locations. The default within new alembic.ini files is "os", which uses os.pathsep.
# If this key is omitted entirely, it falls back to the legacy behavior of splitting on spaces and/or commas.
# Valid values for version_path_separator are:
#
# version_path_separator = :
# version_path_separator = ;
# version_path_separator = space
version_path_separator = os  # Use os.pathsep. Default configuration used for new projects.

# the output encoding used when revision files
# are written from script.py.mako
# output_encoding = utf-8

sqlalchemy.url = driver://user:pass@localhost/dbname

databases = one, two

[post_write_hooks]
# post_write_hooks defines scripts or Python functions that are run
# on newly generated revision scripts.  See the documentation for further
# detail and examples

# format using "black" - use the console_scripts runner, against the "black" entrypoint
# hooks = black
# black.type = console_scripts
# black.entrypoint = black
# black.options = -l 79 REVISION_SCRIPT_FILENAME

# Logging configuration
[loggers]
keys = root,sqlalchemy,alembic

[handlers]
keys = console

[formatters]
keys = generic

[logger_root]
level = WARN
handlers = console
qualname =

[logger_sqlalchemy]
level = WARN
handlers =
qualname = sqlalchemy.engine

[logger_alembic]
level = INFO
handlers =
qualname = alembic

[handler_console]
class = StreamHandler
args = (sys.stderr,)
level = NOTSET
formatter = generic

[formatter_generic]
format = %(levelname)-5.5s [%(name)s] %(message)s
datefmt = %H:%M:%S


[DEFAULT]
# sys.path path, will be prepended to sys.path if present.
# defaults to the current working directory.
prepend_sys_path = .

# template used to generate migration files
file_template = %%(year)d-%%(month).2d-%%(day).2d_%%(hour).2d-%%(minute).2d_%%(rev)s_%%(slug)s

# max length of characters to apply to the "slug" field
truncate_slug_length = 60

[one]
# sqlalchemy.url = sqlite:///one.db
sqlalchemy.url = sqlite:///anu.db
script_location = ./one/alembic/
# version_locations = ./one/versions


[two]
# sqlalchemy.url = sqlite:///two.db
sqlalchemy.url = sqlite:///anu.db
script_location = ./two/alembic/
# version_locations = ./two/versions
```

This tells alembic that you want to use two migration versions:

- `one`: located on `./one/alembic`
- `two`: located on `./two/alembic`

Finally, let's set up your `env.py`

```python
# location: one/alembic/env.py
from logging.config import fileConfig

from sqlalchemy import engine_from_config
from sqlalchemy import pool

from alembic import context

from one.model.book import Base
from one.model.book import Book

# this is the Alembic Config object, which provides
# access to the values within the .ini file in use.
config = context.config

# Interpret the config file for Python logging.
# This line sets up loggers basically.
if config.config_file_name is not None:
    fileConfig(config.config_file_name)

# add your model's MetaData object here
# for 'autogenerate' support
# from myapp import mymodel
# target_metadata = mymodel.Base.metadata
target_metadata = Base.metadata

# other values from the config, defined by the needs of env.py,
# can be acquired:
# my_important_option = config.get_main_option("my_important_option")
# ... etc.


def run_migrations_offline() -> None:
    """Run migrations in 'offline' mode.

    This configures the context with just a URL
    and not an Engine, though an Engine is acceptable
    here as well.  By skipping the Engine creation
    we don't even need a DBAPI to be available.

    Calls to context.execute() here emit the given string to the
    script output.

    """
    url = config.get_main_option("sqlalchemy.url")
    context.configure(
        url=url,
        target_metadata=target_metadata,
        literal_binds=True,
        dialect_opts={"paramstyle": "named"},
        version_table='alembic_version_one',
    )

    with context.begin_transaction():
        context.run_migrations()


def run_migrations_online() -> None:
    """Run migrations in 'online' mode.

    In this scenario we need to create an Engine
    and associate a connection with the context.

    """
    connectable = engine_from_config(
        config.get_section(config.config_ini_section),
        prefix="sqlalchemy.",
        poolclass=pool.NullPool,
    )

    with connectable.connect() as connection:
        context.configure(
            connection=connection, target_metadata=target_metadata,
            version_table='alembic_version_one',
        )

        with context.begin_transaction():
            context.run_migrations()


if context.is_offline_mode():
    run_migrations_offline()
else:
    run_migrations_online()
```

```python
# location: two/alembic/env.py
from logging.config import fileConfig

from sqlalchemy import engine_from_config
from sqlalchemy import pool

from alembic import context

from two.model.user import Base
from two.model.user import User

# this is the Alembic Config object, which provides
# access to the values within the .ini file in use.
config = context.config

# Interpret the config file for Python logging.
# This line sets up loggers basically.
if config.config_file_name is not None:
    fileConfig(config.config_file_name)

# add your model's MetaData object here
# for 'autogenerate' support
# from myapp import mymodel
# target_metadata = mymodel.Base.metadata
target_metadata = Base.metadata

# other values from the config, defined by the needs of env.py,
# can be acquired:
# my_important_option = config.get_main_option("my_important_option")
# ... etc.


def run_migrations_offline() -> None:
    """Run migrations in 'offline' mode.

    This configures the context with just a URL
    and not an Engine, though an Engine is acceptable
    here as well.  By skipping the Engine creation
    we don't even need a DBAPI to be available.

    Calls to context.execute() here emit the given string to the
    script output.

    """
    url = config.get_main_option("sqlalchemy.url")
    context.configure(
        url=url,
        target_metadata=target_metadata,
        literal_binds=True,
        dialect_opts={"paramstyle": "named"},
        version_table='alembic_version_two',
    )

    with context.begin_transaction():
        context.run_migrations()


def run_migrations_online() -> None:
    """Run migrations in 'online' mode.

    In this scenario we need to create an Engine
    and associate a connection with the context.

    """
    connectable = engine_from_config(
        config.get_section(config.config_ini_section),
        prefix="sqlalchemy.",
        poolclass=pool.NullPool,
    )

    with connectable.connect() as connection:
        context.configure(
            connection=connection, target_metadata=target_metadata,
            version_table='alembic_version_two',
        )

        with context.begin_transaction():
            context.run_migrations()


if context.is_offline_mode():
    run_migrations_offline()
else:
    run_migrations_online()
```

Some notable changes are:

- You import the model and the base class

    ```python
    from one.model.book import Base
    from one.model.book import Book
    ```

- You tell alembic to use `Base` as metadata, thus only comparing the existing database with any child class of `Base`. This is necessary if you want to generate migration versions:

    ```python
    target_metadata = Base.metadata
    ```

- You tell alembic to not do anything to any table it doesn't know about:

    ```python
    def include_object(object, name, type_, reflected, compare_to):
        if type_ == "table" and reflected and compare_to is None:
            return False
        else:
            return True

    def run_migrations_offline() -> None:
        # ...
        context.configure(
            url=url,
            target_metadata=target_metadata,
            literal_binds=True,
            dialect_opts={"paramstyle": "named"},
            version_table='alembic_version_two',
            include_object = include_object,
        )
        # ...
    
    def run_migrations_online() -> None:
        # ...
        context.configure(
            connection=connection,
            target_metadata=target_metadata,
            version_table='alembic_version_two',
            include_object = include_object,
        )
        # ...
    ```

Done. Now you are ready to generate migration versions:

```bash
alembic --name one revision -m "new rev for schema 1" --autogenerate
alembic --name two revision -m "new rev for schema 2" --autogenerate
```

and execute them

```bash
alembic --name one upgrade head
alembic --name two upgrade head
```

# Resources

- [Managing Alembic Migrations with a single alembic.ini & env.py](https://learningtotest.com/2021/06/17/managing-alembic-migrations-with-a-single-alembic-ini-env-py/)
- [Tell Flask-Migrate / Alembic to NOT drop any tables it doesn't know about](https://stackoverflow.com/questions/57631160/tell-flask-migrate-alembic-to-not-drop-any-tables-it-doesnt-know-about)