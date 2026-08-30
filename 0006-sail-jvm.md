---
layout: default
title: "Sail desde la JVM: un test kit en Maven Central y lo que encontré por el camino"
date: 2026-08-30
categories: Blog
---

# ⛵ Sail desde la JVM: un test kit en Maven Central y lo que encontré por el camino

Llevo unos cuantos posts hablando de [Sail](https://github.com/lakehq/sail) desde Python. La pregunta que faltaba era la evidente: **¿puedo coger mis tests de Spark en Scala y apuntarlos a Sail?**

La respuesta corta es que sí, y que corre muchísimo más de lo que esperaba. La respuesta larga es este post: un launcher publicado en Maven Central, un template con dos motores, y un hallazgo que me ha cambiado cómo escribo Spark — y que no va de Sail, va de Spark.

---

## 🕳️ El hueco

Sail habla **Spark Connect**. Y Spark publica un cliente JVM, `spark-connect-client-jvm`, que habla ese mismo protocolo. Así que sobre el papel la pieza está: cliente Scala, servidor Rust, gRPC en medio.

El problema es arrancarlo. Sail es un binario de Rust **distribuido como wheel de Python**. Desde Python haces `pip install pysail` y `sail spark server`, y ya. Desde la JVM no hay absolutamente nada que levante un servidor: no hay artefacto en Maven, no hay clase que llamar, no hay nada.

Eso es lo que hace [`sail-testkit`](https://github.com/devel0pez-com/sail-testkit). Es la pieza que faltaba, y no es más que eso: arranca el proceso, te da la URL, y lo mata cuando termina la suite.

Y ya **está en Maven Central**:

```scala
libraryDependencies += "com.devel0pez" %% "sail-testkit" % "0.1.0" % Test
```

---

## 📦 Cómo se usa

Mezclas `SailSuite` en una suite de ScalaTest y tienes un `spark` que habla con Sail:

```scala
import com.devel0pez.sail.testkit.SailSuite
import org.scalatest.funsuite.AnyFunSuite

class MyEtlSpec extends AnyFunSuite with SailSuite {
  test("agrega por clave") {
    val out = MyEtl.transform(spark.read.parquet("src/test/resources/input"))
    assert(out.count() == 3)
  }
}
```

`sbt test` y ya. Un servidor por suite, arrancado en `beforeAll` y parado en `afterAll`. Si prefieres manejar tú la sesión, el launcher está suelto:

```scala
SailServer.withServer { server =>
  val spark = SparkSession.builder().remote(server.url).getOrCreate()
  ...
}
```

Hay una decisión de diseño de la que estoy contento: **`SailSuite` no configura nada de la sesión**. Ni modo ANSI, ni zona horaria, ni nada que cambie lo que una query significa. Eso son decisiones de tu proyecto, y un test kit que las tomara por su cuenta estaría respondiendo a una pregunta que nadie le ha hecho. Hay un hook y punto:

```scala
override protected def configureSession(session: SparkSession): Unit =
  session.conf.set("spark.sql.ansi.enabled", "true")
```

Ese hook, por cierto, no salió de pensarlo bien: salió de consumir el artefacto desde el template Scala vía `publishLocal` y darme cuenta de que estaba peleándome con mi propia librería.

### Los detalles que cuestan una tarde

Tres cosas del launcher que parecen tontería y no lo son:

**El pipe hay que drenarlo.** Sail escribe a stderr. Si nadie lee ese pipe, se llena y Sail se bloquea en su propio logging. El síntoma es un cuelgue sin causa visible en ningún sitio. Descartar la salida también lo evitaría, pero entonces un servidor que se muere se lleva el motivo con él.

**Así que se guardan las últimas 20 líneas** y se meten en el mensaje de error. Un servidor que revienta al arrancar dice por qué:

```
The Sail server exited while starting up (code 1). It printed:
  error: failed to bind: Address already in use (os error 48)
```

**El hilo lector es daemon.** Un test kit no puede ser nunca el motivo por el que una JVM se niega a salir.

---

## 🧨 Los requisitos que no están escritos en ningún sitio

Esta es la sección que me habría ahorrado la mitad del trabajo.

**`pyspark` tiene que estar instalado al lado de `pysail`.** Sail resuelve *su* versión de Spark preguntándole a ese módulo, en su propio lado. Sin él, `spark.version` te contesta:

```
invalid argument: failed to get PySpark version:
ModuleNotFoundError: No module named 'pyspark'
```

**Java 17+ y los `--add-opens`.** Spark y Arrow entran en internals de la JDK por reflexión, y eso está cerrado desde Java 17. `spark-submit` pasa esos flags por ti; sbt no. Sin ellos, el primer `collect()` muere con `sun.misc.Unsafe ... not available` — y es un fallo de *runtime*, así que un build que compila limpio se muere igual en la primera fila.

```scala
Test / fork := true,
Test / javaOptions ++= Seq(
  "--add-opens=java.base/java.nio=ALL-UNNAMED",
  "--add-opens=java.base/sun.nio.ch=ALL-UNNAMED",
  "-Dio.netty.tryReflectionSetAccessible=true"
)
```

**Cliente y servidor viajan en pareja.** Un `versions.json` es la única fuente de verdad, y lo leen `build.sbt` (Scala y Spark) y `flake.nix` (qué se instala en el venv del servidor). Si subes Spark y te olvidas de pysail, no falla: empieza a contestar cosas raras.

---

## 🧪 Y con esto, ¿corre el corpus de Sail?

Aquí es donde el juguete se volvió interesante. Sail tiene su propia suite de conformidad: ficheros `.feature` de Gherkin, SQL puro, comparados contra lo que Spark de verdad devuelve. Está escrita para pytest-bdd. Si el launcher funciona, ese mismo corpus tendría que correr **desde un cliente JVM** con Cucumber.

Corre. `spark/function/` son **4.972 escenarios** contados como los ejecuta Cucumber (uno por fila de `Examples`; declarados en los ficheros son 2.960), y salen por la puerta con un informe agrupado **por causa**, no por escenario. Eso último importa: una coerción que falta tumba cientos de escenarios, y cientos de issues para un solo bug no ayuda a nadie.

Lo que aprendí ahí no fue sobre Sail. Fue sobre lo fácil que es reportar bugs que no existen.

**Spark es el oráculo, Sail es el sujeto.** Los valores esperados se capturaron contra Spark real, así que un escenario que falla *contra Spark* es un bug de mi harness y de nada más. Correr esa línea base primero debería dar ~99%. Lo que falle ahí es mío.

**El corpus y el binario tienen que venir de la misma release.** Correr el corpus de HEAD contra pysail 0.7.0 produjo **96 "divergencias"**, incluida `sum` sobre strings — que se había arreglado diez días después de la release, 64 commits antes. Con el corpus fijado a `v0.7.0`, esas 96 se fueron a cero. No es hipotético: me pasó.

**Y luego los bugs del harness que parecían bugs del motor.** Cada uno costó horas y cada uno produjo cientos de fallos falsos:

| lo que estaba mal | fallos que provocó |
|---|---|
| `query result` lee la tabla que el servidor renderiza con `show()`, no hace `collect()` | 292 |
| `config` hay que restaurarlo al acabar cada escenario | 113 |
| Cucumber desescapa `\\` en tablas de `Examples`; pytest-bdd no | 10 |
| una celda vacía de Gherkin es string vacío, no NULL | (todas las de string vacío) |

El de `config` es el más bonito de todos: **un** escenario que ponía `spark.sql.session.timeZone = America/New_York` y no lo devolvía desplazaba las horas de todos los resultados posteriores.

Como el corpus solo afirma un tipo donde el escenario dice `query schema` (901 de los 4.972), el resto compara filas como texto — y ahí `decimal(29,2)` y `decimal(20,2)` se renderizan igual y pasan. Volcando los schemas de ambos motores y diffeando: **4.639 queries en común, 4.599 idénticas, 40 distintas**, en unas seis familias. Esas 40 siguen sin revisar.

---

## 🛠️ El template: dos motores, un solo `shared/`

La otra mitad de esto es [`template-nix-sail-scala`](https://github.com/davidlghellin/template-nix-sail-scala), hermano del [template de Python](0003-template-nix-sail.md). Misma premisa: Nix fija el entorno, CI corre los tests contra **los dos motores**, y el código de ejemplo tiene forma de ETL de verdad.

```bash
nix develop   # JDK 21, sbt, scalafmt y el servidor Sail
t             # tests contra AMBOS backends
tc            # solo Spark clásico
ts            # solo Sail
```

La diferencia con la versión Python: allí el motor se elige **en tiempo de ejecución** con `SPARK_BACKEND`, aquí **en tiempo de compilación**. Y no es un capricho: `spark-sql` y `spark-connect-client-jvm` traen los dos la clase `org.apache.spark.sql.SparkSession`, así que no pueden compartir classpath. De ahí dos subproyectos.

Lo que *no* está duplicado es el código: `shared/` se compila dos veces, una contra cada cliente. Spark 4 movió la API común a `spark-sql-api`, así que las mismas transformaciones y **los mismos specs** sirven para los dos. Solo cambia de dónde sale la sesión:

| | classic | connect |
| --- | --- | --- |
| Dependencia | `spark-sql` | `spark-connect-client-jvm` |
| Sesión | `.master("local[1]")` | `.remote(server.url)` |
| Motor | JVM | Sail (Rust) |

Y una cosa que quise comprobar en vez de asumir, porque el objetivo último es poder tirar Spark clásico del todo. El classpath entero del lado Spark en el subproyecto `connect` es:

```
spark-connect-client-jvm  spark-connect-shims  spark-common-utils
spark-sketch  spark-unsafe  spark-variant  spark-tags
```

Ni `spark-sql`, ni `spark-core`, ni Hadoop, ni Hive. El jar del cliente lleva la API dentro. `classic` está ahí como oráculo contra el que comparar, no como algo de lo que `connect` dependa.

---

## 🎯 El hallazgo: no es DataFrame contra Dataset

Aquí está la parte por la que escribo el post.

El resumen fácil de todo esto sería *"los DataFrames funcionan en Sail, los Datasets fallan"*. Es falso, y además lleva justo al movimiento equivocado. El eje que importa es **columnas contra closures**:

| | corre en Sail | pushdown y pruning | tipado |
|---|---|---|---|
| DataFrame + columnas | sí | sí | no |
| **Dataset + columnas** | **sí** | **sí** | **sí** |
| Dataset + closures | no | no | sí |

La fila del medio no cede nada respecto a la primera. Es la primera **más los tipos, gratis**. Los encoders se derivan en el **cliente**, así que `as[T]`, `Seq[T].toDS()`, `Dataset[T]`, `Option[T]` y el `collect()` tipado cruzan Connect sin despeinarse.

Lo que no cruza es un **closure**:

| | en Sail | qué contesta |
|---|---|---|
| `ds.map(lambda)` | ✗ | `wildcard with plan ID` |
| `ds.filter(lambda)` | ✗ | `wildcard with plan ID` |
| `ds.flatMap(lambda)` | ✗ | `wildcard with plan ID` |
| `ds.groupByKey(lambda)` | ✗ | `Scala UDF is not supported yet` |
| `ds.queryExecution` | ✗ | `UNSUPPORTED_CONNECT_FEATURE.DATASET_QUERY_EXECUTION` |

Los cuatro primeros son la misma causa: Connect manda el lambda como **bytecode de la JVM**, y al otro lado hay Rust. No hay nada que lo deserialice ni nada que lo ejecute. El quinto es otra cosa: una limitación de Connect que comparte cualquier servidor Connect, no un hueco de Sail.

Fíjate en que solo `groupByKey` dice el motivo de verdad. Los otros tres mueren antes, en un wildcard, y el mensaje bueno que Sail ya tiene escrito no llega a leerse: `resolve_map_partitions` resuelve los argumentos de la UDF antes de mirar qué tipo de UDF es. Es un arreglo pequeño y autocontenido en Sail, y el mensaje está fijado en un test que se pondrá rojo el día que lo arreglen.

La regla que predice todo lo demás: **lo que se puede expresar como columna viaja; lo que necesita bytecode ejecutándose en el servidor, no.**

---

## 📉 Y ahora la parte incómoda

La lectura obvia de esa tabla es que Sail es menos capaz: clásico corre `ds.map(_.amount * 2)` y Sail no.

Medir qué hace clásico *de verdad* con eso cambia la lectura. Leyendo una tabla parquet de cinco columnas:

| closure, en clásico | columnas leídas | forma de columna | columnas leídas |
|---|---|---|---|
| `filter(_.amount > 50)` | 5 de 5 | `filter(col(...))` | 2 de 5 |
| `map(_.amount)` | 5 de 5 | `select(col(...))` | 1 de 5 |
| `groupByKey(_.country)` | 5 de 5 | `groupBy(col(...))` | 1 de 5 |
| `flatMap(...)` | 5 de 5 | `explode(...)` | 1 de 5 |
| `reduce(_ + _)` | 5 de 5 | `sum(col(...))` | 1 de 5 |

Ninguno de esos cinco closures menciona la columna `day`. Los cinco la cargan.

(La fila del `filter` lee dos y no una porque el predicado necesita `amount` además de la columna que se proyecta: no se puede filtrar por una columna sin leerla. Las otras cuatro proyectan una sola.)

Y con un filtro en juego se va también el pushdown. Sobre la misma tabla de cinco columnas, quedándote una (`branch`) y filtrando por otra (`amount`):

| cómo está escrito | filtros empujados | columnas leídas |
|---|---|---|
| `filter(col("amount") > 50)` | `IsNotNull, GreaterThan(amount,50.00)` | 2 de 5 |
| `filter(_.amount > 50)` (lambda) | **ninguno** | **5 de 5** |

Un closure cuesta tres cosas a la vez, y clásico te las cobra las tres en silencio. No llega ningún predicado al fichero, así que se leen filas solo para tirarlas. No llega ninguna proyección, así que se leen todas las columnas para satisfacer un lambda que toca una. Y cada fila se deserializa a un objeto JVM para que el closure tenga contra qué correr. Con dos filas esto es invisible; con mil millones es la diferencia entre leer 200 GB y leer 12.

Ojo a la primera fila de esa tabla: es un `Dataset[Sale]`, completamente tipado, con pushdown y pruning intactos. **No es el Dataset el que pierde las optimizaciones — es el closure.** Catalyst no puede ver dentro del bytecode, así que no puede razonar sobre él, ni moverlo, ni empujarlo a ningún sitio.

Sail, además, hace las mismas optimizaciones y las escribe en el plan:

```
DataSourceExec: file_groups={...},
  projection=[branch, amount],
  predicate=amount@3 > Some(5000),18,2,
  pruning_predicate=amount_null_count@1 != row_count@2
                    AND amount_max@0 > Some(5000),18,2
```

Pruning de columnas, pushdown de predicado, y pruning de row-groups por las estadísticas del parquet. DataFusion escribe la tercera en el plan; Catalyst se la guarda.

Así que:

| | `filter(_.amount > 50)` |
|---|---|
| clásico | corre — 5 columnas de 5, sin pushdown, cada fila deserializada, **en silencio** |
| Sail | se niega |

Visto desde el rendimiento en vez de desde las features, **el que te trata mal es clásico**: te da el camino lento sin mencionarlo, y solo te enteras leyendo un plan, cosa que casi nadie hace. Un fallo ruidoso se arregla esa misma tarde; un job que lee 200 GB en vez de 12 puede vivir años en producción.

Y es también el argumento en contra de construir el camino de UDFs JVM dentro de Sail. Serían meses de trabajo — un puente JNI, un ciclo de vida de JVM, un jar auxiliar — para entregar un modo de ejecución **más lento que la alternativa que ya funciona**. El mensaje de error merece arreglarse. La feature que hay detrás, para todo lo que una columna sepa expresar, merece rechazarse.

Donde el rechazo sí cuesta algo real: un closure que llama a Scala arbitrario — una librería, un lookup, lógica que ninguna columna deletrea — no tiene forma de columna a la que caer. Ese es un límite de verdad, no un favor disfrazado.

> ¿Cómo de vivible es la regla? **Ninguno de los seis ETLs del template contiene un solo closure de Dataset.** Ni el de DataFrames ni los cinco tipados. Los closures solo viven en los specs que existen para enseñar lo que cuestan.

---

## 🪄 ¿Y no se puede traducir el closure solo?

Es la pregunta obvia a estas alturas. Si `_.amount * 2` y `col("amount") * 2` significan lo mismo, ¿por qué tengo que reescribir una en la otra a mano?

La vía difícil es analizar **bytecode**, que es donde este problema lleva atascado desde siempre: para cuando el closure es bytecode, ya has perdido casi todo lo que necesitabas saber de él. Desde un macro de Scala no hace falta llegar ahí. En tiempo de compilación tienes delante el **AST tipado del lambda**, que es un punto de partida incomparablemente mejor. Eso es lo que hay en `macros/`, y funciona:

```scala
Expr.of[Sale](_.amount * 2)   // se convierte en  col("amount") * lit(2)
```

Y envolviendo el `Dataset` en un tipo propio, el call site queda **idéntico al código que falla en Sail**:

```scala
val sales = TypedDataset(spark.table("sales").as[Sale])
sales.filter(_.amount > 50).map(s => Doubled(s.country, s.amount * 2)).dataset
```

Eso compila a una proyección. Hace falta el wrapper porque `Dataset.map` **no se puede interceptar**: un miembro siempre gana a una conversión implícita, así que una extensión llamada `map` compilaría, resolvería a la de Spark y seguiría fallando igual. Sobre un tipo mío, `map` es un nombre como cualquier otro — la misma regla por la que `Storage` no puede llamar `write` a su escritor.

### Lo interesante son los rechazos

Un traductor *casi* correcto es peor que ningún traductor. El modo de fallo que importa no es que no compile: es que conteste algo **plausible y distinto**. Así que el macro no rechaza por sintaxis, rechaza por **semántica**:

| lambda | veredicto | por qué |
|---|---|---|
| `_.userId % 2` | compila | medido: `1` en los dos motores |
| `_.userId / 2.0` | compila | medido: `2.5` en los dos |
| `_.userId / 2` | **no compila** | Scala da `2`; Spark da `2.5`, y de tipo Double |
| `s.country + s.branch` | **no compila** | `+` concatena Strings en Scala, es aritmética en Spark |
| `s.product.toUpperCase` | **no compila** | llamada a método: fuera del subconjunto |
| `_.tariff == "X"` | compila, pero a `<=>` | `===` propaga NULL; el `==` de Scala contesta `false` |

La última fila es mi favorita. Traducir `==` a `===` era lo evidente y era **incorrecto**, y solo se nota sobre una columna nullable. El spec lo prueba con dos filas, una con `tariff` a NULL.

Todo eso se afirma con `assertDoesNotCompile`, que es donde descansa el diseño entero: si el rechazo llegara en tiempo de ejecución no sería mejor que lo que Sail ya hace.

> El mejor argumento a favor de fallar así es un bug que tuvo el propio macro. Una versión anterior casaba cualquier llamada cuya aridad coincidiera con el número de campos, así que `s => swapped(s.a, s.b)` compilaba como `Doubled(s.a, s.b)`: ignoraba lo que `swapped` hacía y contestaba otra cosa. Exactamente el fallo que el spike existe para prevenir, cometido dentro del spike. Ahora comprueba que la llamada sea el constructor, y en los tests vive un `notAConstructor` cuyo único trabajo es que eso no vuelva.

Y sí, recupera lo que el closure perdía: `filterExpr` empuja el mismo predicado que la columna escrita a mano — 2 columnas de 5 — y `mapExpr` lee 1 de 5. Las cifras de la forma de columna, con la forma del lambda.

### Y aun así no resuelve el problema

Porque para usarlo **hay que tocar el código**, que es justo lo contrario de la premisa de Sail: cambias el servidor y dejas el job en paz. Un macro que exige reescribir el job no te ahorra reescribir el job.

Como ejercicio sí contesta algo, y no es poco: la traducción es **posible y barata** para el subconjunto que de verdad se usa, siempre que estés dispuesto a rechazar todo lo demás en tiempo de compilación. La parte cara nunca fue traducir. Fue decidir qué no traducir.

---

## 🪤 Dos trampas de Spark que no tienen nada que ver con Sail

Salieron montando todo esto y no van de motores. Van de Spark, y las dos son silenciosas.

### `as[T]` decodifica por nombre; `insertInto` escribe por posición

`as[T]` casa las columnas **por nombre**, pero no reordena el schema. Un frame cuyas columnas vienen como `(family, name, code)` se convierte en un `Dataset[Product]` cuyo schema **sigue en ese orden**, mientras `collect()` te devuelve valores `Product` perfectamente correctos. Así que todas las aserciones que se te ocurriría escribir pasan:

```scala
reversed.as[Product].collect().head          // Product(P1, Widget, TOOLS)  ✓
reversed.as[Product].schema.fieldNames       // (family, name, code)        ✗
```

Y después `insertInto` casa **por posición** y te escribe la familia en la columna del código. No salta nada. Ni un warning.

El typeclass `Conform` cierra el agujero. `Dataset.to(StructType)` es el motor —reordena, descarta lo que sobra y comprueba tipos— y el typeclass añade la guarda que a `to` le falta, **antes** de llamarlo:

```scala
wide.conformTo[Product]                    // reordena, descarta lo que Product no declara
short.conformTo[Product]                   // ConformError: missing columns: family
wide.conformTo[Product](Conform.exact)     // ConformError: unexpected columns: junk
```

Que la comprobación vaya antes y no después no es cosmético, y aquí sí reaparece Sail: ante una columna que falta **los dos motores no se comportan igual** (está en la tabla de la sección siguiente). Comprobar antes es lo único que hace que contesten lo mismo.

### `withColumn` en bucle, y el README que lo contaba mal

`withColumn` añade una columna envolviendo el plan en un `Project` más, así que llamado en bucle anida. Hasta ahí, folklore conocido. Lo que el README de mi propio template afirmaba —y estaba mal— es que ese anidamiento llegaba a la query ejecutada. No llega:

| plan | 5 `withColumn` encadenados | un solo `select` |
|---|---|---|
| analizado | 6 `Project` | 2 |
| optimizado | 1 | 1 |
| físico | 1 | 1 |

Catalyst lo aplana antes de ejecutar. El coste no es una query peor: es un **camino más largo hasta la misma query**, que paga el analizador en cada operación posterior. Con cadenas suficientemente largas es una forma conocida de reventar su pila, con una traza que no menciona el bucle que la causó.

Lo cuento porque la corrección salió de medirlo, no de pensarlo mejor. Y porque el número vive ahora en un spec en vez de en la memoria de nadie.

---

## 🔬 Dónde discrepan de verdad los dos motores

Todo medido contra `pysail` 0.7.0 con el cliente JVM 4.2.0, y cada línea fijada por un test que se pone rojo si deja de ser cierta:

| | clásico | Sail |
|---|---|---|
| cast inválido | `CAST_INVALID_INPUT` | `Cast error: Cannot cast string ...`, envuelto en `CONNECT_CLIENT_UNEXPECTED_MISSING_SQL_STATE` |
| `to(schema)` sin una columna | **se la inventa**, a NULL | **se niega**: `field not found in input schema` |
| `DECIMAL(18,2) * 2` | `decimal(20,2)` | `decimal(29,2)` |
| `DECIMAL(38,18) * 2` | `decimal(38,16)` | `decimal(38,18)` |
| `DECIMAL(38,18) / 2` | `decimal(38,18)` | `decimal(38,22)` |
| el plan | Catalyst: `Project`, `Filter` | DataFusion: `ProjectionExec`, `FilterExec` |

Lo de los decimales tiene un diagnóstico más fino que el que enseña la tabla: los dos coinciden siempre que **ambos operandos declaran su precisión**. `DECIMAL(18,2) * DECIMAL(1,0)` es `(20,2)` en los dos; `* DECIMAL(10,0)` es `(29,2)` en los dos. Solo los separa el **literal pelado**: Catalyst estrecha el `2` al decimal más pequeño que lo contiene, Sail lo mantiene al ancho de un `Int`. Los valores salen iguales de todas formas; lo que se mueve es el schema, que es invisible hasta que el resultado se encuentra con una tabla.

Y fíjate en la fila de `to(schema)`, porque va al revés de lo que uno esperaría: **el estricto es Sail**. Clásico se inventa la columna y la rellena de nulls.

### Nada se marca como skipped

La tentación con estas divergencias es marcar los tests como `ignore` o `pending` para que el build salga verde. El coste es que se quedan **callados para siempre**: el día que Sail cierre el hueco, nada te lo dice, y el skip sobrevive a su motivo durante años.

Así que aquí no se salta nada. Los dos brazos siguen vivos:

```scala
// Corre en clásico, se espera que falle en Sail — afirmado, no tolerado.
failsOnSail()(sales.map(_.amount * 2).collect())
```

`failsOnSail` **afirma el fallo**. Si Sail algún día corre el lambda, el test se pone **rojo** y dice que la expectativa está caducada. Que es exactamente el punto: un build verde que ha dejado de comprobar nada es peor que uno rojo.

---

## 📮 Publicar en Maven Central en 2026

Un par de cosas que me costaron y que no están donde deberían.

**El Central Portal no es el default del plugin.** `sbt-sonatype` sigue apuntando a `oss.sonatype.org`, el OSSRH legacy, que dejó de admitir namespaces nuevos cuando lo jubilaron. Sin esto, `sbt ci-release` firma y sube correctamente a un host que nunca va a publicar nada:

```scala
ThisBuild / sonatypeCredentialHost := xerial.sbt.Sonatype.sonatypeCentralHost
```

**`sbt-ci-release` termina llamando a `sonaRelease`**, un comando que ninguna versión publicada de `sbt-sonatype` define. El build hace todo bien y se muere en el último paso con *"Not a valid command"*. Hay que dárselo:

```scala
addCommandAlias("sonaRelease", "sonatypeCentralUpload")
```

Y ahí `sonatypeCentralUpload` en vez de `sonatypeCentralRelease` a propósito: deja el deployment en `VALIDATED`, de modo que firmas, sources, javadoc y POM se pueden mirar en el Portal antes de que nada llegue a Central. De ahí es un clic publicar, o *Drop* y nunca existió. Maven Central es **inmutable**: una versión que sale no se puede reemplazar ni retirar.

**El namespace no tiene que coincidir con el repo.** `com.devel0pez` se verifica con un registro DNS TXT en el dominio; la URL de GitHub es solo metadata del POM. Por eso el groupId es `com.devel0pez` y el repo vive en `devel0pez-com/sail-testkit`, sin problema. Con `io.github.*` habrían quedado atados.

**`versionScheme` importa más de lo que parece.** `early-semver`, no `semver-spec`: bajo semver estricto toda release `0.x` puede romper lo que quiera, así que un salto `0.1.0 → 0.1.1` no prometería nada. `early-semver` mantiene el dígito de patch con significado antes del 1.0.0.

Y las releases van **por tag**, nada publica porque una rama se mueva. `sbt-dynver` lee la versión del tag, así que `v0.1.0` es `0.1.0`. El `workflow_dispatch` se queda para lo único bueno que tenía el trigger por rama: lanzarlo a mano sobre un commit sin tag produce un `-SNAPSHOT` y prueba credenciales, firma y subida sin dejar nada inmutable en Central.

---

## ⚠️ Lo que conviene saber antes de apuntar tu suite a Sail

- **Los lambdas tipados no van.** `map`, `filter(_.x)`, `flatMap`, `groupByKey`, UDFs de Scala. Una suite escrita a base de lambdas tipados no va a correr, y mejor saberlo antes de gastar la tarde.
- **RDDs tampoco**, pero eso no es de Sail: no hay API de RDD sobre Connect, punto.
- **`queryExecution` es solo de clásico.** Es lo primero a lo que echa mano cualquier código que inspeccione planes.
- **No hagas `match` sobre error classes.** Los dos motores coinciden en la *semántica* (los dos rechazan el cast) pero no en la identidad del error.
- **Una case class declarada dentro de la clase de test rompe los encoders** por reflexión. Ese fallo es del test, no de Sail. Decláralas a nivel de fichero.
- **IntelliJ ignora `Test / javaOptions`.** Su runner de ScalaTest no pasa los `--add-opens`, así que el primer `collect()` desde el gutter muere con `InaccessibleObjectException`. O marcas *use sbt shell for builds and tests*, o copias los flags a las VM options de la run configuration.

---

## 💭 Reflexión

Tres cosas me llevo de esto.

**El trabajo de verdad no estuvo en el motor, estuvo en el harness.** De los cientos de fallos del corpus, los que resultaron ser divergencias reales eran un puñado. Los demás eran míos: un `collect()` donde tocaba un `show()`, una config sin restaurar, unos backslashes que Cucumber se comía. Reportar un bug de motor sin haber descartado eso primero es hacerle perder la tarde a otro.

**Sail no es menos capaz, es más honesto.** Cada operación que Sail rechaza es exactamente la que en clásico te estaba costando un scan completo sin avisar. Eso no lo sabía antes de medirlo, y ha cambiado cómo escribo Spark en Scala independientemente de qué motor haya debajo. La regla es simple: evita los closures — no porque Sail no pueda con ellos, sino porque nunca fueron el camino rápido. Sail es solo el primer motor que lo dice en voz alta.

**Y una cosa sobre publicar.** Que la pieza que faltaba fuera tan pequeña — un `ProcessBuilder`, un puerto libre, un hilo drenando un pipe y un trait de ScalaTest — y que aun así no existiera, dice algo sobre lo lejos que puede estar un ecosistema de otro. Sail lleva tiempo corriendo Spark Connect perfectamente; simplemente nadie había escrito las 200 líneas de Scala que hacen falta para que la JVM lo arranque.

Esto lo hablé en el Slack de LakeSail antes de empezar. Su respuesta fue que un repo aparte está bien, que un launcher JVM "sería interesante", que no pueden mantener un componente JVM in-tree por el CI extra que implica, pero que ejemplos ayudarían. Así que lo siguiente es un PR de documentación a `lakehq/sail` — *"Using Sail from the JVM"* — con el arranque del servidor, los flags de Arrow, el requisito de `pyspark` y la tabla de la API tipada.

Estado: **0.1.0**. Temprano. Hace una cosa y los tests la cubren, pero no ha pasado por mucho más que eso. Issues y PRs bienvenidos. Y si algún día aparece un kit JVM upstream, este debería apartarse.

- 📦 [`com.devel0pez:sail-testkit_2.13`](https://central.sonatype.com/artifact/com.devel0pez/sail-testkit_2.13) en Maven Central
- 🐙 [devel0pez-com/sail-testkit](https://github.com/devel0pez-com/sail-testkit)
- ⛵ [davidlghellin/template-nix-sail-scala](https://github.com/davidlghellin/template-nix-sail-scala)

---

## 📚 Posts anteriores

[Sail + PySpark](0001-sail.md) | [Nix + Sail](0002-nix-sail.md) | [Template Nix](0003-template-nix-sail.md) | [Maintainer en nixpkgs](0004-sail-nixpkgs-maintainer.md) | [NixOS en mis rpi3](0005-nixos-rpi3.md)
