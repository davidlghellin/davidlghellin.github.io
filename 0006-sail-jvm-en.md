---
layout: default
title: "Sail from the JVM: a test kit on Maven Central, and what I found on the way"
date: 2026-08-30
categories: Blog
---

# ⛵ Sail from the JVM: a test kit on Maven Central, and what I found on the way

I have spent a few posts now talking about [Sail](https://github.com/lakehq/sail) from Python. The question left over was the obvious one: **can I take my Spark tests in Scala and point them at Sail?**

The short answer is yes, and that far more of them run than I expected. The long answer is this post: a launcher published on Maven Central, a template with two engines, and a finding that has changed how I write Spark — and it is not about Sail, it is about Spark.

---

## 🕳️ The gap

Sail speaks **Spark Connect**. And Spark publishes a JVM client, `spark-connect-client-jvm`, that speaks the same protocol. So on paper the pieces are all there: Scala client, Rust server, gRPC in between.

The problem is starting it. Sail is a Rust binary **shipped as a Python wheel**. From Python you `pip install pysail`, run `sail spark server`, and that is that. From the JVM there is nothing at all that brings a server up: no artifact on Maven, no class to call, nothing.

That is what [`sail-testkit`](https://github.com/devel0pez-com/sail-testkit) is. It is the missing piece and nothing more than that: it starts the process, hands you the URL, and kills it when the suite ends.

And it is **on Maven Central**:

```scala
libraryDependencies += "com.devel0pez" %% "sail-testkit" % "0.1.0" % Test
```

---

## 📦 How you use it

Mix `SailSuite` into a ScalaTest suite and you get a `spark` that talks to Sail:

```scala
import com.devel0pez.sail.testkit.SailSuite
import org.scalatest.funsuite.AnyFunSuite

class MyEtlSpec extends AnyFunSuite with SailSuite {
  test("aggregates by key") {
    val out = MyEtl.transform(spark.read.parquet("src/test/resources/input"))
    assert(out.count() == 3)
  }
}
```

`sbt test` and that is it. One server per suite, started in `beforeAll` and stopped in `afterAll`. If you would rather own the session yourself, the launcher stands alone:

```scala
SailServer.withServer { server =>
  val spark = SparkSession.builder().remote(server.url).getOrCreate()
  ...
}
```

There is one design decision I am happy with: **`SailSuite` configures nothing about the session**. Not ANSI mode, not a time zone, nothing that changes what a query means. Those are your project's decisions, and a test kit that quietly made them would be answering a question nobody asked it. There is a hook, and that is all:

```scala
override protected def configureSession(session: SparkSession): Unit =
  session.conf.set("spark.sql.ansi.enabled", "true")
```

That hook, incidentally, did not come from thinking it through. It came from consuming the artifact from the Scala template via `publishLocal` and finding myself fighting my own library.

### The details that cost an afternoon

Three things about the launcher that look trivial and are not:

**The pipe has to be drained.** Sail logs to stderr. If nobody reads that pipe it fills up and Sail blocks on its own logging. The symptom is a hang with no cause visible anywhere. Discarding the output would avoid that too, but then a server that dies takes the reason with it.

**So it keeps the last 20 lines** and puts them in the failure. A server that dies on startup says why:

```
The Sail server exited while starting up (code 1). It printed:
  error: failed to bind: Address already in use (os error 48)
```

**The reader thread is a daemon.** A test kit must never be the reason a JVM refuses to exit.

---

## 🧨 The requirements that are written down nowhere

This is the section that would have saved me half the work.

**`pyspark` has to be installed next to `pysail`.** Sail resolves *its* Spark version by asking that module, on its own side. Without it, `spark.version` answers:

```
invalid argument: failed to get PySpark version:
ModuleNotFoundError: No module named 'pyspark'
```

**Java 17+ and the `--add-opens` flags.** Spark and Arrow reach into JDK internals by reflection, and that has been sealed off since Java 17. `spark-submit` passes those flags for you; sbt does not. Without them the first `collect()` dies with `sun.misc.Unsafe ... not available` — and it is a *run-time* failure, so a build that compiles cleanly still dies on its first row.

```scala
Test / fork := true,
Test / javaOptions ++= Seq(
  "--add-opens=java.base/java.nio=ALL-UNNAMED",
  "--add-opens=java.base/sun.nio.ch=ALL-UNNAMED",
  "-Dio.netty.tryReflectionSetAccessible=true"
)
```

**Client and server travel in pairs.** A `versions.json` is the single source of truth, read by `build.sbt` (Scala and Spark) and by `flake.nix` (what goes into the server's venv). If you bump Spark and forget pysail, nothing fails — it just starts answering odd things.

---

## 🧪 And with that, does Sail's corpus run?

This is where the toy got interesting. Sail has its own conformance suite: Gherkin `.feature` files, pure SQL, compared against what real Spark returns. It is written for pytest-bdd. If the launcher works, that same corpus ought to run **from a JVM client** under Cucumber.

It does. `spark/function/` is **4,972 scenarios** counted the way Cucumber runs them (one per `Examples` row; declared in the files they are 2,960), and they come out the other end as a report grouped **by cause**, not by scenario. That last part matters: one missing coercion fails hundreds of scenarios, and hundreds of issues for one bug helps nobody.

What I learned there was not about Sail. It was about how easy it is to report bugs that do not exist.

**Spark is the oracle, Sail is the subject.** The expected values were captured against real Spark, so a scenario failing *against Spark* is a bug in my harness and in nothing else. Running that baseline first should sit at ~99%. Whatever fails there is mine.

**The corpus and the binary must come from the same release.** Running HEAD's corpus against pysail 0.7.0 produced **96 "divergences"**, including `sum` over strings — which had been fixed ten days after the release, 64 commits earlier. With the corpus pinned to `v0.7.0`, those 96 went to zero. Not hypothetical: it happened to me.

**And then the harness bugs that looked like engine bugs.** Each one cost hours and each produced hundreds of false failures:

| what was wrong | failures it caused |
|---|---|
| `query result` reads the table the *server* renders with `show()`, it does not `collect()` | 292 |
| `config` has to be restored after each scenario | 113 |
| Cucumber unescapes `\\` in `Examples` tables; pytest-bdd does not | 10 |
| an empty Gherkin cell is an empty string, not NULL | (every empty-string one) |

The `config` one is my favourite: **one** scenario that set `spark.sql.session.timeZone = America/New_York` and left it set shifted every later result by hours.

Since the corpus only asserts a type where a scenario says `query schema` (901 of the 4,972), everything else compares rows as text — and there `decimal(29,2)` and `decimal(20,2)` render identically and pass. Dumping both engines' schemas and diffing them: **4,639 queries in common, 4,599 identical, 40 different**, in about six families. Those 40 are still unreviewed.

---

## 🛠️ The template: two engines, one `shared/`

The other half of this is [`template-nix-sail-scala`](https://github.com/davidlghellin/template-nix-sail-scala), sibling to the [Python template](0003-template-nix-sail-en.md). Same premise: Nix pins the environment, CI runs the tests against **both engines**, and the example code is shaped like a real ETL.

```bash
nix develop   # JDK 21, sbt, scalafmt and the Sail server
t             # tests against BOTH backends
tc            # classic Spark only
ts            # Sail only
```

The difference from the Python version: there the engine is chosen **at run time** with `SPARK_BACKEND`, here **at compile time**. And that is not a whim: `spark-sql` and `spark-connect-client-jvm` both ship the class `org.apache.spark.sql.SparkSession`, so they cannot share a classpath. Hence two subprojects.

What is *not* duplicated is the code: `shared/` is compiled twice, once against each client. Spark 4 moved the common API into `spark-sql-api`, so the same transformations and **the same specs** serve both. Only where the session comes from changes:

| | classic | connect |
| --- | --- | --- |
| Dependency | `spark-sql` | `spark-connect-client-jvm` |
| Session | `.master("local[1]")` | `.remote(server.url)` |
| Engine | JVM | Sail (Rust) |

And one thing I wanted to check rather than assume, because the eventual goal is being able to drop classic Spark entirely. The whole Spark-side classpath of the `connect` subproject is:

```
spark-connect-client-jvm  spark-connect-shims  spark-common-utils
spark-sketch  spark-unsafe  spark-variant  spark-tags
```

No `spark-sql`, no `spark-core`, no Hadoop, no Hive. The client jar carries the API inside it. `classic` is there as an oracle to compare against, not as something `connect` leans on.

---

## 🎯 The finding: it is not DataFrame against Dataset

Here is the part I am writing this post for.

The easy summary of all this would be *"DataFrames work on Sail, Datasets fail"*. It is false, and worse, it leads to exactly the wrong move. The axis that matters is **columns against closures**:

| | runs on Sail | pushdown and pruning | typed |
|---|---|---|---|
| DataFrame + columns | yes | yes | no |
| **Dataset + columns** | **yes** | **yes** | **yes** |
| Dataset + closures | no | no | yes |

The middle row gives up nothing to the first. It is the first row **plus the types, for free**. Encoders are derived on the **client**, so `as[T]`, `Seq[T].toDS()`, `Dataset[T]`, `Option[T]` and typed `collect()` cross Connect without breaking a sweat.

What does not cross is a **closure**:

| | on Sail | what it answers |
|---|---|---|
| `ds.map(lambda)` | ✗ | `wildcard with plan ID` |
| `ds.filter(lambda)` | ✗ | `wildcard with plan ID` |
| `ds.flatMap(lambda)` | ✗ | `wildcard with plan ID` |
| `ds.groupByKey(lambda)` | ✗ | `Scala UDF is not supported yet` |
| `ds.queryExecution` | ✗ | `UNSUPPORTED_CONNECT_FEATURE.DATASET_QUERY_EXECUTION` |

The first four are one cause: Connect ships the lambda as **JVM bytecode**, and on the far side there is Rust. Nothing to deserialise it and nothing to run it. The fifth is something else: a Connect limitation any Connect server shares, not a gap in Sail.

Notice that only `groupByKey` names the real reason. The other three die earlier, on a wildcard, and the accurate message Sail already has written is never reached: `resolve_map_partitions` resolves the UDF's arguments before it looks at what kind of UDF it is. That is a small, self-contained fix in Sail, and the message is pinned in a test that will go red the day it lands.

The rule that predicts everything else: **anything expressible as a column travels; anything needing bytecode executed on the server does not.**

---

## 📉 And now the uncomfortable part

The obvious reading of that table is that Sail is less capable: classic runs `ds.map(_.amount * 2)` and Sail does not.

Measuring what classic actually *does* with it changes the reading. Reading a five-column parquet table:

| closure, on classic | columns read | column form | columns read |
|---|---|---|---|
| `filter(_.amount > 50)` | 5 of 5 | `filter(col(...))` | 2 of 5 |
| `map(_.amount)` | 5 of 5 | `select(col(...))` | 1 of 5 |
| `groupByKey(_.country)` | 5 of 5 | `groupBy(col(...))` | 1 of 5 |
| `flatMap(...)` | 5 of 5 | `explode(...)` | 1 of 5 |
| `reduce(_ + _)` | 5 of 5 | `sum(col(...))` | 1 of 5 |

Not one of those five closures mentions the `day` column. All five load it.

(The `filter` row reads two rather than one because the predicate needs `amount` on top of the column being projected: you cannot filter on a column without reading it. The other four project a single column.)

And with a filter in play the pushdown goes too. Over the same five-column table, keeping one column (`branch`) and filtering on another (`amount`):

| how it is written | filters pushed down | columns read |
|---|---|---|
| `filter(col("amount") > 50)` | `IsNotNull, GreaterThan(amount,50.00)` | 2 of 5 |
| `filter(_.amount > 50)` (lambda) | **none** | **5 of 5** |

A closure costs three things at once, and classic charges all three silently. No predicate reaches the file, so rows are read only to be discarded. No projection reaches it either, so every column is read to satisfy a lambda that touches one. And each row is deserialised into a JVM object so the closure has something to run against. On two rows this is invisible; on a billion it is the difference between reading 200 GB and reading 12.

Look at the first row of that table: it is a `Dataset[Sale]`, fully typed, with pushdown and pruning intact. **It is not the Dataset that loses the optimisations — it is the closure.** Catalyst cannot see inside bytecode, so it cannot reason about it, move it, or push it anywhere.

Sail, moreover, does the same optimisations and writes them into the plan:

```
DataSourceExec: file_groups={...},
  projection=[branch, amount],
  predicate=amount@3 > Some(5000),18,2,
  pruning_predicate=amount_null_count@1 != row_count@2
                    AND amount_max@0 > Some(5000),18,2
```

Column pruning, predicate pushdown, and row-group pruning from the parquet statistics. DataFusion writes the third one into the plan; Catalyst keeps it to itself.

So:

| | `filter(_.amount > 50)` |
|---|---|
| classic | runs — 5 columns of 5, no pushdown, every row deserialised, **silently** |
| Sail | refuses |

Seen from performance rather than from features, **the one treating you badly is classic**: it hands you the slow path without mentioning it, and you only find out by reading a plan, which almost nobody does. A loud failure gets fixed the same afternoon; a job reading 200 GB instead of 12 can live in production for years.

It is also the argument against building the JVM UDF path into Sail. It would be months of work — a JNI bridge, a JVM lifecycle, a helper jar — to deliver an execution mode that is **slower than the alternative that already works**. The error message is worth fixing. The feature behind it, for anything a column can express, is worth refusing.

Where the refusal does cost something real: a closure calling arbitrary Scala — a library, a lookup, logic no column can spell — has no column form to fall back on. That is a genuine limit, not a disguised favour.

> How livable is the rule? **Not one of the six ETLs in the template contains a single Dataset closure.** Not the DataFrame one, not the five typed ones. The closures live only in the specs that exist to show what they cost.

---

## 🪄 Can the closure not just be translated?

It is the obvious question by now. If `_.amount * 2` and `col("amount") * 2` mean the same thing, why do I have to rewrite one into the other by hand?

The hard road is analysing **bytecode**, which is where this problem has always got stuck: by the time the closure is bytecode you have lost almost everything you needed to know about it. From a Scala macro you never have to go there. At compile time you have the **typed AST of the lambda** in front of you, which is an incomparably better starting point. That is what lives in `macros/`, and it works:

```scala
Expr.of[Sale](_.amount * 2)   // becomes  col("amount") * lit(2)
```

And wrapping the `Dataset` in a type of my own, the call site reads **exactly like the code that fails on Sail**:

```scala
val sales = TypedDataset(spark.table("sales").as[Sale])
sales.filter(_.amount > 50).map(s => Doubled(s.country, s.amount * 2)).dataset
```

That compiles to a projection. The wrapper is needed because `Dataset.map` **cannot be intercepted**: a member always beats an implicit conversion, so an extension named `map` would compile, resolve to Spark's, and go on failing exactly as before. On a type of mine, `map` is a name like any other — the same rule that stops `Storage` calling its writer `write`.

### The refusals are the interesting part

A translator that is *nearly* right is worse than no translator. The failure mode that matters is not failing to compile: it is answering something **plausible and different**. So the macro refuses on **semantics**, not on syntax:

| lambda | verdict | why |
|---|---|---|
| `_.userId % 2` | compiles | measured: `1` on both engines |
| `_.userId / 2.0` | compiles | measured: `2.5` on both |
| `_.userId / 2` | **does not compile** | Scala gives `2`; Spark gives `2.5`, typed Double |
| `s.country + s.branch` | **does not compile** | `+` concatenates Strings in Scala, is arithmetic in Spark |
| `s.product.toUpperCase` | **does not compile** | a method call: outside the subset |
| `_.tariff == "X"` | compiles, but to `<=>` | `===` propagates NULL; Scala's `==` answers `false` |

That last row is my favourite. Translating `==` to `===` was the obvious move and it was **wrong**, and it only shows up on a nullable column. The spec proves it with two rows, one with `tariff` NULL.

All of it is asserted with `assertDoesNotCompile`, which is where the whole design rests: if the refusal arrived at run time it would be no better than what Sail already does.

> The best argument for failing this way is a bug the macro itself had. An earlier version matched any call whose arity equalled the field count, so `s => swapped(s.a, s.b)` compiled as `Doubled(s.a, s.b)`: it ignored what `swapped` did and answered something else. Exactly the failure the spike exists to prevent, committed inside the spike. It now checks that the call is the constructor, and a `notAConstructor` lives in the tests whose only job is making sure that does not come back.

And yes, it recovers what the closure was losing: `filterExpr` pushes the same predicate as the hand-written column — 2 columns of 5 — and `mapExpr` reads 1 of 5. The column form's numbers, in the lambda's shape.

### And it still does not solve the problem

Because using it means **changing the code**, which is the exact opposite of Sail's premise: swap the server, leave the job alone. A macro that demands rewriting the job does not save you rewriting the job.

As an exercise it does answer something, and it is not nothing: the translation is **possible and cheap** for the subset people actually use, as long as you are willing to refuse everything else at compile time. The expensive part was never translating. It was deciding what not to translate.

---

## 🪤 Two Spark traps that have nothing to do with Sail

Both turned up while building this, and neither is about engines. They are about Spark, and both are silent.

### `as[T]` decodes by name; `insertInto` writes by position

`as[T]` matches columns **by name**, but does not reorder the schema. A frame whose columns arrive as `(family, name, code)` becomes a `Dataset[Product]` whose schema is **still in that order**, while `collect()` hands back perfectly correct `Product` values. So every assertion you would think to write passes:

```scala
reversed.as[Product].collect().head          // Product(P1, Widget, TOOLS)  ✓
reversed.as[Product].schema.fieldNames       // (family, name, code)        ✗
```

And then `insertInto` matches **by position** and writes the family into the code column. Nothing raises. Not even a warning.

The `Conform` typeclass closes the hole. `Dataset.to(StructType)` is the engine — it reorders, drops extras and type-checks — and the typeclass adds the guard `to` lacks, **before** calling it:

```scala
wide.conformTo[Product]                    // reorders, drops what Product does not declare
short.conformTo[Product]                   // ConformError: missing columns: family
wide.conformTo[Product](Conform.exact)     // ConformError: unexpected columns: junk
```

The check running before rather than after is not cosmetic, and here Sail reappears: on a missing column **the two engines do not behave the same** (it is in the next section's table). Checking first is the only thing that makes them answer alike.

### `withColumn` in a loop, and the README that had it wrong

`withColumn` adds one column by wrapping the plan in one more `Project`, so in a loop it nests. So far, known folklore. What my own template's README claimed — and had wrong — is that the nesting reached the executed query. It does not:

| plan | 5 chained `withColumn` | one `select` |
|---|---|---|
| analyzed | 6 `Project` | 2 |
| optimized | 1 | 1 |
| physical | 1 | 1 |

Catalyst flattens it before execution. The cost is not a worse query: it is a **longer walk to the same query**, paid by the analyzer on every operation that follows. Long enough chains are a known way to blow its stack, with a trace that says nothing about the loop that caused it.

I mention it because the correction came from measuring it, not from thinking harder. And because the number now lives in a spec rather than in anybody's memory.

---

## 🔬 Where the two engines really differ

All measured against `pysail` 0.7.0 with the 4.2.0 JVM client, and every line pinned by a test that goes red if it stops being true:

| | classic | Sail |
|---|---|---|
| invalid cast | `CAST_INVALID_INPUT` | `Cast error: Cannot cast string ...`, wrapped in `CONNECT_CLIENT_UNEXPECTED_MISSING_SQL_STATE` |
| `to(schema)` missing a column | **invents it**, filled with NULL | **refuses**: `field not found in input schema` |
| `DECIMAL(18,2) * 2` | `decimal(20,2)` | `decimal(29,2)` |
| `DECIMAL(38,18) * 2` | `decimal(38,16)` | `decimal(38,18)` |
| `DECIMAL(38,18) / 2` | `decimal(38,18)` | `decimal(38,22)` |
| the plan | Catalyst: `Project`, `Filter` | DataFusion: `ProjectionExec`, `FilterExec` |

The decimal rows have a sharper diagnosis than the table shows: the two agree whenever **both operands declare their precision**. `DECIMAL(18,2) * DECIMAL(1,0)` is `(20,2)` on both; `* DECIMAL(10,0)` is `(29,2)` on both. Only the **bare literal** parts them: Catalyst narrows the `2` to the smallest decimal that holds it, Sail keeps it at an `Int`'s width. The values come out equal either way; what moves is the schema, which is invisible until the result meets a table.

And notice the `to(schema)` row, because it runs against expectation: **the strict one is Sail**. Classic invents the column and fills it with nulls.

### Nothing is marked skipped

The tempting move with these divergences is to mark the tests `ignore` or `pending` so the build goes green. The cost is that they go **quiet forever**: the day Sail closes the gap, nothing tells you, and the skip outlives its reason by years.

So nothing here skips. Both arms stay live:

```scala
// Works on classic, expected to fail on Sail — asserted, not tolerated.
failsOnSail()(sales.map(_.amount * 2).collect())
```

`failsOnSail` **asserts the failure**. If Sail ever runs the lambda, the test goes **red** and says the expectation is stale. Which is the point: a green build that has stopped checking anything is worse than a red one.

---

## 📮 Publishing to Maven Central in 2026

A couple of things that cost me time and are not where they should be.

**The Central Portal is not the plugin's default.** `sbt-sonatype` still points at `oss.sonatype.org`, the legacy OSSRH, which stopped taking new namespaces when it was sunset. Without this, `sbt ci-release` signs and stages perfectly into a host that will never publish it:

```scala
ThisBuild / sonatypeCredentialHost := xerial.sbt.Sonatype.sonatypeCentralHost
```

**`sbt-ci-release` ends by calling `sonaRelease`**, a command no published version of `sbt-sonatype` defines. The build does everything right and dies on the last step with *"Not a valid command"*. You have to supply it:

```scala
addCommandAlias("sonaRelease", "sonatypeCentralUpload")
```

And `sonatypeCentralUpload` rather than `sonatypeCentralRelease` on purpose: it leaves the deployment at `VALIDATED`, so signatures, sources, javadoc and the POM can be inspected in the Portal before anything reaches Central. From there it is one click to publish, or *Drop* and it never existed. Maven Central is **immutable**: a version that goes out cannot be replaced or withdrawn.

**The namespace does not have to match the repo.** `com.devel0pez` is verified with a DNS TXT record on the domain; the GitHub URL is only POM metadata. That is why the groupId is `com.devel0pez` while the repo lives at `devel0pez-com/sail-testkit`, with no problem. `io.github.*` would have tied them together.

**`versionScheme` matters more than it looks.** `early-semver`, not `semver-spec`: under strict semver every `0.x` release is allowed to break anything, so a `0.1.0 → 0.1.1` bump would promise nothing. `early-semver` keeps the patch digit meaningful before 1.0.0.

And releases go **by tag** — nothing publishes because a branch moved. `sbt-dynver` reads the version from the tag, so `v0.1.0` is `0.1.0`. `workflow_dispatch` keeps the one good thing the branch trigger had: running it by hand on an untagged commit produces a `-SNAPSHOT` and exercises credentials, signing and upload without leaving anything immutable on Central.

---

## ⚠️ What to know before pointing your suite at Sail

- **Typed lambdas will not run.** `map`, `filter(_.x)`, `flatMap`, `groupByKey`, Scala UDFs. A suite written on typed lambdas will not run, and it is better to know before spending the afternoon.
- **Neither will RDDs**, but that is not Sail: there is no RDD API over Connect at all.
- **`queryExecution` is classic-only.** It is the first thing any plan-inspecting code reaches for.
- **Do not `match` on error classes.** The two engines agree on the *semantics* (both refuse the cast) but not on the identity of the error.
- **A case class declared inside the test class breaks encoders** by reflection. That failure is the test's fault, not Sail's. Declare them at file level.
- **IntelliJ ignores `Test / javaOptions`.** Its ScalaTest runner does not pass the `--add-opens`, so the first `collect()` from the gutter dies with `InaccessibleObjectException`. Either tick *use sbt shell for builds and tests*, or copy the flags into the run configuration's VM options.

---

## 💭 Reflection

Three things I take from this.

**The real work was not in the engine, it was in the harness.** Of the hundreds of corpus failures, the ones that turned out to be real divergences were a handful. The rest were mine: a `collect()` where a `show()` belonged, a config left unrestored, backslashes Cucumber had eaten. Reporting an engine bug without ruling that out first is wasting somebody else's afternoon.

**Sail is not less capable, it is more honest.** Every operation Sail refuses is exactly the one that was costing you a full scan on classic without saying so. I did not know that before measuring it, and it has changed how I write Spark in Scala regardless of which engine is underneath. The rule is simple: avoid closures — not because Sail cannot run them, but because they were never the fast path. Sail is just the first engine that says so out loud.

**And one thing about publishing.** That the missing piece was this small — a `ProcessBuilder`, a free port, a thread draining a pipe and a ScalaTest trait — and that it still did not exist, says something about how far one ecosystem can sit from another. Sail has been serving Spark Connect perfectly well for a while; nobody had simply written the 200 lines of Scala the JVM needs to start it.

I raised this in the LakeSail Slack before starting. Their answer was that a separate repo is fine, that a JVM launcher "would be interesting", that they cannot maintain a JVM component in-tree because of the extra CI it implies, but that examples would help. So the next thing is a docs PR to `lakehq/sail` — *"Using Sail from the JVM"* — with the server startup, the Arrow flags, the `pyspark` requirement and the typed API table.

Status: **0.1.0**. Early. It does one thing and the tests cover it, but it has not been through much beyond that. Issues and PRs welcome. And if a JVM kit ever lands upstream, this one should give way to it.

- 📦 [`com.devel0pez:sail-testkit_2.13`](https://central.sonatype.com/artifact/com.devel0pez/sail-testkit_2.13) on Maven Central
- 🐙 [devel0pez-com/sail-testkit](https://github.com/devel0pez-com/sail-testkit)
- ⛵ [davidlghellin/template-nix-sail-scala](https://github.com/davidlghellin/template-nix-sail-scala)

---

## 📚 Previous posts

[Sail + PySpark](0001-sail.md) | [Nix + Sail](0002-nix-sail.md) | [Nix template](0003-template-nix-sail-en.md) | [Maintainer at nixpkgs](0004-sail-nixpkgs-maintainer-en.md) | [NixOS on my rpi3s](0005-nixos-rpi3-en.md)
