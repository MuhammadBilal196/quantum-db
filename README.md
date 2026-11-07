# quantum-db

An embedded document and relational database engine written in the Quantum language. Store collections in memory or in a single JSON database file with schemas, unique and secondary hash indexes, Mongo-style query operators, SQL execution (`SELECT`, `INSERT`, `UPDATE`, `DELETE`, `CREATE TABLE`, `JOIN`, `GROUP BY`), ACID-style snapshot transactions, and formatted ASCII/Markdown/CSV/JSONL import and export.

```bash
qrun quantum-db-cli.sa init store.db.json --sample
qrun quantum-db-cli.sa sql "SELECT name, role, age FROM users WHERE age >= 25 ORDER BY age DESC" --db store.db.json
qrun quantum-db-cli.sa sql "SELECT role, COUNT(*) AS count, AVG(age) AS avg_age FROM users GROUP BY role" --db store.db.json
```

- **Pure Quantum.** Zero external dependencies; runs directly on `qrun`.
- **Dual Query Interface:** Query with **SQL statements** (`SELECT ... JOIN ... WHERE ... GROUP BY ... ORDER BY ...`) or **Mongo-style dicts** (`{"age": {"$gte": 21}, "role": {"$in": ["admin", "engineer"]}}`).
- **Schemas & Constraints:** Optional per-collection field types (`string`, `int`, `number`, `boolean`, `array`, `dict`), default values, `required`, `min`/`max`, `minLength`/`maxLength`, `enum`, primary keys, and `unique` hash indexes.
- **Joins & Aggregations:** `INNER JOIN`, `LEFT JOIN`, and `lookup` array joins across collections, plus `GROUP BY` aggregations (`COUNT`, `SUM`, `AVG`, `MIN`, `MAX`, `DISTINCT`).
- **Transactions:** Snapshot isolation with `Db.begin(db)`, `Db.commit(db)`, `Db.rollback(db)`, and `Db.transaction(db, fn)` that rolls back automatically on error.
- **Tables & Import/Export:** Render results as ASCII tables, GitHub Markdown tables, RFC-4180 CSV, JSON, or JSON Lines (`JSONL`).

## Install

```bash
qpm install quantum-db
```

The package installs into `quantum_qubits/quantum-db/`. The standalone library bundle is `dist/quantum-db.sa` and the command-line tool is `dist/quantum-db-cli.sa`.

> `qrun` below is the Quantum interpreter. If it isn't on your PATH, use its full path, e.g. `C:\QuantumLogicLabs\QuantumLanguage\seed\qrun.exe`.

## CLI Commands

```bash
qrun quantum-db-cli.sa init [file.db.json] [--sample]
qrun quantum-db-cli.sa sql "SELECT ..." [--db file.db.json] [--format table|md|csv|json]
qrun quantum-db-cli.sa query <collection> [--where '{"role":"admin"}'] [--sort age:desc] [--limit 10] [--format table|md|csv|json]
qrun quantum-db-cli.sa insert <collection> '{"name":"Ada","email":"ada@quantum.dev"}' [--db file.db.json]
qrun quantum-db-cli.sa import <collection> <file.csv|file.jsonl> [--db file.db.json]
qrun quantum-db-cli.sa export <collection> [--db file.db.json] [--format csv|md|json|jsonl|table] [--out file]
qrun quantum-db-cli.sa stats [--db file.db.json]
qrun quantum-db-cli.sa demo
qrun quantum-db-cli.sa help
```

Run with no arguments (or `demo`) to execute a live showcase of SQL queries, aggregations, joins, and ASCII tables on built-in sample data.

## Using the Library

Include `dist/quantum-db.sa` or bundle with `# @use quantum-db`:

```python
# @use quantum-db

let db = Db.create({"name": "app"})

# 1. Create a collection with schema and unique index
Db.createCollection(db, "users", {
    "unique": ["email"],
    "indexes": ["role"],
    "schema": {
        "name":  "string!",
        "email": "string!",
        "role":  {"type": "string", "default": "member"},
        "age":   {"type": "int", "min": 13}
    }
})

# 2. Insert & update documents
Db.insertMany(db, "users", [
    {"name": "Ada Lovelace", "email": "ada@quantum.dev", "role": "admin", "age": 28},
    {"name": "Bilal Raza", "email": "bilal@quantum.dev", "role": "engineer", "age": 24}
])
Db.update(db, "users", {"email": "bilal@quantum.dev"}, {"$inc": {"age": 1}})

# 3. Query with filter operators or SQL
let adults = Db.find(db, "users", {
    "where": {"age": {"$gte": 21}},
    "orderBy": "age:desc",
    "select": ["id", "name", "role", "age"]
})
print(DbFormat.table(adults, null))

let rows = Db.sql(db, "SELECT role, COUNT(*) AS total FROM users GROUP BY role")
print(DbFormat.markdown(rows, ["role", "total"]))

# 4. Save and reopen
Db.save(db, "app.db.json")
let loaded = Db.open("app.db.json")
```

## Query & Update Operators

| Category | Operators |
|---|---|
| **Comparison** | `$eq`, `$ne`, `$gt`, `$gte`, `$lt`, `$lte`, `$between` (`[lo, hi]`) |
| **Array & Set** | `$in`, `$nin`, `$contains`, `$all`, `$size` |
| **String** | `$startsWith`, `$endsWith`, `$contains`, `$icontains`, `$like` (`%`, `_`), `$ilike` |
| **Logical & Meta** | `$and`, `$or`, `$not`, `$exists`, `$type` |
| **Update (`Db.update`)** | `$set`, `$inc`, `$mul`, `$min`, `$max`, `$push`, `$pull`, `$unset` |
| **Aggregation (`metrics`)** | `$count`, `$sum`, `$avg`, `$min`, `$max`, `$distinct` |

## Module API Reference

| Global | Functions |
|---|---|
| `Db` | `create(options)` · `open(path)` · `save(db, path)` · `createCollection(db, name, options)` · `dropCollection(db, name)` · `listCollections(db)` · `createIndex(db, col, field, unique)` · `insert(db, col, doc)` · `insertMany(db, col, docs)` · `get(db, col, id)` · `has(db, col, id)` · `count(db, col, where)` · `find(db, col, query)` · `findOne(db, col, where)` · `update(db, col, where, changes)` · `updateById(db, col, id, changes)` · `upsert(db, col, where, doc)` · `remove(db, col, where)` · `removeById(db, col, id)` · `clear(db, col)` · `begin(db)` · `commit(db)` · `rollback(db)` · `transaction(db, fn)` · `sql(db, sqlText)` · `importCsv(db, col, csv)` · `exportCsv(db, col, query)` · `stats(db)` · `sampleDatabase()` |
| `DbSql` | `execute(db, sqlText)` · `parse(sqlText)` · `tokenize(sqlText)` |
| `DbQuery` | `matches(doc, where)` · `applyChanges(doc, changes)` · `project(rows, select)` · `executeJoin(left, right, spec)` · `aggregate(rows, spec)` · `likeMatch(text, pattern, ignoreCase)` |
| `DbSchema` | `normalize(schema)` · `prepare(schema, doc)` · `validate(schema, doc)` · `assertValid(schema, doc)` |
| `DbFormat` | `table(rows, cols)` · `markdown(rows, cols)` · `csv(rows, cols)` · `parseCsv(text, options)` · `jsonl(rows)` · `parseJsonl(text)` |
| `DbUtil` | `clone(v)` · `equals(a, b)` · `getPath(obj, path)` · `setPath(obj, path, v)` · `sortRecords(rows, orderSpec)` · `autoValue(str)` |

## Development

```
index.sa                 bundle entry (lists modules in dependency order)
src/                     util, schema, query, sql, format, engine
bin/quantum-db.sa        command-line tool
dist/                    pre-bundled single files: library and CLI
test/all.test.sa         90 tests; test/run-tests.sa is the bundled runner
```

```bash
qrun test/run-tests.sa
```

## License

MIT © 2026 Muhammad Bilal Raza
