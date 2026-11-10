# One Statement, Two Semicolons (Java and Kotlin + Spring)

A Spring Boot port of Jim Lehmer's code challenge at
[dullroar/code-challenge](https://github.com/dullroar/code-challenge), written twice: once in Java and once in
Kotlin, so the two languages can be compared. The original was a 2008 C# 3.0 class exercise, modernized to .NET 10
in 2026.

The constraint is the same: `main` contains exactly one statement, a `for` loop with an empty body. The
initializer, condition and update expressions do all the work. This is a stunt in expression mechanics. It is not
an architectural pattern and it is not good production code.

## Run

You need a JDK 25 toolchain and a desktop session. Everything else comes through the Gradle wrapper.

```
.\gradlew.bat runJava
.\gradlew.bat runKotlin
```

A window titled "Only One STATEMENT (Java + Spring demo)" or "(Kotlin + Spring demo)" appears and pulses. Closing
it ends the program. `bootRun` and `bootJar` use the Java version.

The window says **DEMO ONLY**. Each run generates three or four made-up time cards and saves them to a local SQLite
file at `data/demo-timecards.db` inside this project. Both versions share that file. The app has no connection to a
real time-reporting system. Delete the `data` folder to start over; git ignores it.

The path is relative to the working directory, which the Gradle run tasks set to the project folder. If you run the
JAR directly from somewhere else, `data` is created there.

## Layout

| Version | Program | Helper types |
| --- | --- | --- |
| Java | [Program.java](src/main/java/challenge1/javaversion/Program.java) | `DemoTimeCard.java`, `DemoTimeCardRepository.java` |
| Kotlin | [Program.kt](src/main/kotlin/challenge1/kotlinversion/Program.kt) | `DemoTimeCard.kt`, `DemoTimeCardRepository.kt` |

Each version has its own package, Spring Boot application class, entity and repository. As in the original, the
entity and the data-access type are the only code outside `main`.

## How the one statement works

Comment arrows mark the boundaries between the three sections in both files.

- **Initializer.** One anonymous object holds the loop's state: the Spring Boot application context, the
  repository, the list of hours, the category iterator, a lock and an XML fragment.
- **Condition.** While the hours total 40 or less and a category remains, the loop continues. Otherwise it builds
  the window, shows it, waits for it to close, and returns `false`.
- **Update.** Each pass adds 10 to 19 random hours, stages a time card, and starts a background thread that saves
  the staged cards through reflection.

## Java compared with Kotlin

| | Java | Kotlin |
| --- | --- | --- |
| The one statement | `for (init; condition; update) {}`, with the two semicolons | `for (pass in generateSequence(seed) { ... }) {}`. Kotlin has no three-part `for` and no semicolons |
| Initializer | Anonymous class held in a `var` | Object expression as the sequence seed |
| Condition | The `for` condition, a `? :` expression | `takeIf { if ... else ... }`; a null result ends the sequence |
| Update | Comma-separated expressions | A chain of `?.also { ... }` calls |
| Running several steps inside one expression | `Stream.of(x).peek(...).peek(...)` | `x.also { ... }.also { ... }` |
| Turning the result into `false` | `Double.isNaN(int)`, the original's trick | `.let { false }` |
| XML string | Text blocks joined with `+` | One raw string with `${...}` templates |
| Summing the hours | `stream().mapToDouble(Double::doubleValue).sum()` | `sum()` |
| Entity | Class with fields and two constructors | Class with a primary constructor |

One behavior differs in principle. The Kotlin loop body runs once before the first condition check, where Java
checks first. The body is empty, so the two programs do the same things in the same order.

## What changed from the .NET version

| .NET version | These versions |
| --- | --- |
| Anonymous object as iterator state | Anonymous class (Java) or object expression (Kotlin) |
| EF Core `DbContext` and SQLite | Spring Boot, Spring Data JPA, Hibernate and SQLite |
| `TimeCards.Add` then `SaveChanges` | A `pending` list, then `CrudRepository.saveAll` |
| WPF window from `XamlReader.Load` on a runtime XAML string | Swing window from `GenericXmlApplicationContext` on a runtime Spring XML string |
| `XElement` holding a `DoubleAnimation` | XML fragment holding a `javax.swing.Timer` bean |
| Opacity animation on the `Viewbox` | Timer that fades the label colors |
| `Application.Run` blocks until the window closes | Modal `JDialog.setVisible(true)` blocks until the window closes |
| LINQ to Objects `Sum` | `Stream` `sum` (Java) or `sum()` (Kotlin) |
| SQLite file under `%LOCALAPPDATA%` | SQLite file in the project's `data` folder |
| Named `Mutex` | `ReentrantLock` |
| `MethodInfo.Invoke` | Spring `ReflectionUtils.invokeMethod` |
| `Random.Shared` | `ThreadLocalRandom.current()` |
| `ISOWeek.GetWeekOfYear` | `IsoFields.WEEK_OF_WEEK_BASED_YEAR` |
| `Environment.UserName` | `System.getProperty("user.name")` |

## License

MIT, carrying the original copyright notice. See [LICENSE](LICENSE).
