# liquibase-linter: changesets for another database are not linted

Minimal project showing that `liquibase-linter-maven-plugin` never lints a changeset whose `dbms`
attribute excludes H2.

## The change log

[`db.changelog-master.xml`](src/main/resources/db/changelog/db.changelog-master.xml) holds three changesets
that break the same rule, `has-comment` (none has a `<comment>`). They differ only in `dbms`:

| Changeset | `dbms` | Should be reported |
| --- | --- | --- |
| `1-every-dbms` | — | yes |
| `2-postgresql-only` | `postgresql` | yes |
| `3-postgresql-and-h2` | `postgresql,h2` | yes |

The [linter configuration](src/test/resources/lqlint.json) enables only `has-comment`.

## Run

```sh
mvn verify
```

Expected: 3 violations. With `liquibase-linter-maven-plugin` 1.0.0 the build reports only 2 —
`2-postgresql-only` is missing:

```
[ERROR] 2 violation(s) found
[ERROR] src/main/resources/db/changelog/db.changelog-master.xml
[ERROR] 	changeSet '1-every-dbms'
[ERROR] 		'has-comment': Change set must have a comment
[ERROR] 	changeSet '3-postgresql-and-h2'
[ERROR] 		'has-comment': Change set must have a comment
```

Remove changesets 1 and 3 and the build even succeeds, although changeset 2 still breaks the rule.

## Why

The `check` goal parses the change log through an offline H2 connection hard-coded in
[`CheckMojo`](https://github.com/liquibase-linter/liquibase-linter/blob/main/liquibase-linter-maven-plugin/src/main/java/io/github/liquibaselinter/mavenplugin/CheckMojo.java):

```java
try (DatabaseConnection connection = new OfflineConnection("offline:h2", resourceAccessor)) {
    return new Liquibase(changeLogFile, resourceAccessor, connection);
}
```

While parsing, Liquibase keeps a changeset only if its `dbms` matches the database it parses for
([`DatabaseChangeLog.isDbmsMatch`](https://github.com/liquibase/liquibase/blob/v5.0.4/liquibase-standard/src/main/java/liquibase/changelog/DatabaseChangeLog.java)).
`2-postgresql-only` does not match H2, so it is dropped before any rule sees it. `3-postgresql-and-h2` matches
and is linted. The linter documentation does not mention this.

## Branches

- `main` — the released plugin, 1.0.0: shows the problem
- `fix` — the plugin built from the fix: shows the solution
