# database migrations

A migration (in scope of dbs) can mean 2 things:
- Schema (database) migration: incremental changes in the database schema, such as adding or modifying tables, columns, indexes, constraints
- Data migration (much bigger process that doesn't happen often): migrating from one database system to another, moving data, upgrading a database

version control for databases

The concept of db migrations is the exact same across any language and any tool, but the implementation detail differs slightly.

* problem:
application changes through dev (models, functions...)
while we can track these changes through version control systems, we need a way to track that external dependency (schema of the database)
=> question: how do we make sure the state/schema of the database matches the required state of the database ?!
(wait.. isn't the schema script also updated if found in the project !?)

A migration is just a script (sql, bash, js, python...) that takes a database from a state/schema to another. it's similar to a commit in a vcs.

A step that is needed is capturing the state (where we are in the migration lifecycle) using a version/id...

* examples:
    - Liquibase ? (paid | works with pretty much all kind of databases and programing languages) is probably the most famous one.
    - 'flyway' for Java.
    - 'Sequelize' for JS.

up is for running on the migration
down is for reverting the migration

data management:
youe api is stateless, but the database is stateful (you rarely gonna see containerized dbs).

code vs data separation:
1. migration within app/api:
    - needs manual intervention when app won't start due to migration failure
    - downtime if migration takes long
    - needs extra locking mechanims if multiple instances are running the migration
    - good for small self-contained apps and teams without dedicated database engineers

2. migration outside api/app:
    - isolates procedures, failure in migration can be solved earlier without a need to wait for the app to run
    - app code alays needs to be in sync with the schema
    - good if there are many cross-functional teams

* Parallel migrations (hybrid solution -> 3 options to handle multiple instances):
    1. run migrations outside of the app container (use separate container or job for your ci/cd pipeline - e.g. run-migrations.sh [with depends_on set in docker-compose] - before starting app containers)
    2. use kubernetes job/init containers
    3. DB locking mechanism

* Don'ts & Do's:
    - don't batch unrelated changes within a same migration (i.e. creating a table and inseting values into it are 2 different operations that must live in different migrations)
    - don't change models without creating a migration afterwards (must be same change in same commit)
    - do group migrations in separate folder by versions/features (i.e. migrations/users/...)
    - do use a dedicated db user for running migrations (not the default)
    - do ensure support for transactional DDL statements (ddl stats are transactions that can be reversible) -> (support is provided by db system itself) so that if the migration fails, all changes reflected on db must be reversed.
    - do proper testing (Testcontainers)
        1. drop the test db (if it exists)
        2. recreate it from scratch
        3. run all migrations
        4. (optional) seed it with test data
        5. verify the expected tables/columns exist

* Backward compatibility
    1. add non-breaking changes first, new columns nullable or with a default value
    2. use deprecation messages instead of droping tables right away
        1. when renaming a column, create a new one and temporarily update both old and new columns
        2. use views when possible (views are kind of aliases in sql).

* good to watch:
https://www.youtube.com/watch?v=jeWrbAiA1D0

---

## Migrate example (social network project -> golang sql migration)

* Requirements:

Every time the application runs, it creates the specific tables to make the project work properly.

Use provided folder structure (up & down files ?)
migrations/sqlite/... :
. 000001_create_users_table.down.sql
. 000001_create_users_table.up.sql
. 000002_create_posts_table.down.sql
. 000002_create_posts_table.up.sql
...

The folder structure is organized in a way that helps you to understand and use migrations, where you can apply it using a simple path, for example: file://backend/pkg/db/migrations/sqlite. It can be organized as you wish but do not forget that the application of migrations and the file organization will be tested.

Can use 'golang-migrate' package or other package that better suits your project.

The sqlite.go should present the connection to the database, the applying of the migrations and other useful functionalities that you may need to implement.

This migration system can help you manage your time and testing, by filling your database.

### provided golang packages:
    - [golang-migrate](https://github.com/golang-migrate/migrate/)
    - [sql-migration](https://pkg.go.dev/github.com/rubenv/sql-migrate)
    - [migration](https://pkg.go.dev/github.com/Boostport/migration)

* comparison between the 3 choices (generated by claude):

Here's a rundown of these three, based on what's currently out there:

> golang-migrate/migrate
The most widely used option — has around 12.8K GitHub stars. Key traits:
- Works as both a **CLI tool** and a **Go library**
- Supports a huge range of databases (Postgres, MySQL, SQLite, MongoDB, CockroachDB, etc.) via database drivers
- Supports many migration source formats (files, Go embed, S3, GitHub, etc.)
- Migrations are plain `.sql` files (up/down pairs), or you can write Go migrations too
- No ORM dependency — it's a standalone tool
- Very "batteries included" — this is the default choice for most Go teams doing SQL-first migrations

> sql-migrate (rubenv/sql-migrate)
A smaller, more focused competitor — has 2.8K GitHub stars (some sources report slightly higher, ~3K).
- Also CLI + library
- Migrations are `.sql` files with `-- +migrate Up` / `-- +migrate Down` annotations in the same file (rather than separate up/down files like golang-migrate)
- Built on top of `gorp` for the underlying DB layer
- Simpler feature set, smaller community, less actively maintained in recent years compared to golang-migrate

> "Boostport/migration"
This package is designed to be a simple and pragmatic migration library for Go, deliberately built to avoid the limitations its authors found in both sql-migrate and golang-migrate. Key traits:

- **Driver-per-module design**: each database driver lives in its own module (e.g., `driver/mysql`, `driver/postgres`, `driver/sqlite`, `driver/phoenix`), so you only pull in dependencies for the databases you actually use
- **Notably supports Apache Phoenix** — a database golang-migrate struggles with because it uses URL scheme to pick the driver, which conflicts with Phoenix's use of scheme for http/https
- Supports both **SQL file migrations** and **Go-code migrations** (via a `golang` driver) in the same project
- Native `go:embed` support for bundling migration files into your binary
- Much smaller adoption: imported by only 23 other packages, versus golang-migrate's much larger ecosystem

The README itself explains *why* it exists — the authors specifically evaluated sql-migrate and migrate (golang-migrate's old import path) before building their own. They found sql-migrate too tightly coupled to the gorp ORM, limiting database support, while migrate was hard to extend for embeddable migration files and used the DSN's URL scheme in a way that broke Apache Phoenix support.

> Practical recommendation
For a typical Go project, it's really a two-horse race:
- **golang-migrate** — best default choice: mature, database-agnostic, actively maintained, works well as a library inside your app or as a standalone CLI in CI/CD
- **sql-migrate** — fine if you want single-file up/down migrations and don't mind a smaller ecosystem

One other option worth knowing about even though you didn't ask: `pressly/goose`, which some teams prefer because it supports **Go-based migrations** (not just SQL) for cases where a migration needs custom logic that raw SQL can't express.

> Where this leaves you

| | golang-migrate | sql-migrate | Boostport/migration |
|---|---|---|---|
| Popularity | Very high (~12.8K★) | Moderate (~3K★) | Low (23 importers) |
| DB support | Broad, via URL scheme | Limited by gorp | Broad, modular drivers |
| Go-code migrations | Limited | No | Yes |
| Embeds migration files | Yes (various backends) | Limited | Yes, first-class `go:embed` |
| Best for | Most general-purpose projects | Simple gorp-based projects | Niche DBs (e.g., Phoenix) or when you want Go+SQL migrations together |

**Bottom line:** unless you specifically need Apache Phoenix support or want first-class Go-code migrations without adopting goose, golang-migrate remains the safer default — it's far more battle-tested and has a much larger community for support. Boostport/migration is a reasonable niche pick if its specific design solves a problem you actually have.

====================================================================

'docker-compose up' command