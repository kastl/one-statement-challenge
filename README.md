# One Statement, Two Semicolons

A port of Jim Lehmer's code challenge at [dullroar/code-challenge](https://github.com/dullroar/code-challenge),
written in Java, Kotlin, Python and Go so the languages can be compared. The original was a 2008 C# 3.0 class
exercise, modernized to .NET 10 in 2026.

The constraint: `main` contains exactly one statement, a `for` loop with an empty body. The loop header does all the
work. This is a stunt in expression mechanics. It is not an architectural pattern and it is not good production
code.

## What every version does

All four programs do the same thing, in the same order:

1. Open a local SQLite database (`demo-timecards.db`) and create a `time_cards` table if it is missing.
2. Loop. Each pass adds 10 to 19 random hours, stages a made-up time card with the next category (Meetings, Coding,
   Debugging, Learning), and starts a background thread that saves the staged cards.
3. Stop when the hours total more than 40 or the categories run out. That is three or four passes.
4. Show a window that says **DEMO ONLY** with the total simulated hours. It pulses (fades in and out) until you close
   it.
5. Wait for the saves to finish, then exit.

The cards are flagged `is_demo`. Nothing here connects to a real time-reporting system. Delete a version's `data`
folder to start over; git ignores them.

## Run

| Version | You need | Run | Window | Data file |
| --- | --- | --- | --- | --- |
| Java | JDK 25 (Gradle wrapper does the rest) | `.\gradlew.bat runJava` | Swing dialog | `data/demo-timecards.db` |
| Kotlin | JDK 25 | `.\gradlew.bat runKotlin` | Swing dialog | `data/demo-timecards.db` (shared with Java) |
| Python | Python 3 only (`tkinter` and `sqlite3` are built in) | `.\gradlew.bat runPython` or `py python\program.py` | Tk window | `python/data/demo-timecards.db` |
| Go | Go toolchain (`winget install GoLang.Go`); the first run downloads one module | `.\gradlew.bat runGo` or `cd go; go run .` | Page in your default browser | `go/data/demo-timecards.db` |

Notes:

- The Gradle tasks appear in IntelliJ's Gradle tool window under Tasks > application. `bootRun` and `bootJar` use the
  Java version.
- Java and Kotlin need a desktop session. Their data path is relative to the working directory, which the Gradle run
  tasks set to the project folder. If you run the JAR from somewhere else, `data` is created there.
- Go: if a running IDE doesn't see `go` after installing it, restart the IDE so it picks up the new PATH. In the Go
  version, press Close in the page or close the tab to finish.
- Gradle builds only the Java and Kotlin code. `python/` and `go/` are self-contained folders beside it.

## Layout

| Version | Program | Other files |
| --- | --- | --- |
| Java | [Program.java](src/main/java/challenge1/javaversion/Program.java) | `DemoTimeCard.java`, `DemoTimeCardRepository.java` |
| Kotlin | [Program.kt](src/main/kotlin/challenge1/kotlinversion/Program.kt) | `DemoTimeCard.kt`, `DemoTimeCardRepository.kt` |
| Python | [program.py](python/program.py) | Entity and repository classes sit in the same file, above `main` |
| Go | [main.go](go/main.go) | [helpers.go](go/helpers.go): entity, repository and the expression helpers; `go.mod`, `go.sum` |

As in the original, the entity and the data-access type are the only code outside `main`. The exception is Go, whose
`helpers.go` also holds small helper functions (explained below).

## How the one statement is built, by language

Every version splits the work three ways: an **initializer** that builds the loop's state, a **condition** that
decides whether to keep going (and, on the last check, shows the window), and an **update** that does one pass of
work. Comment banners in each source file mark the boundaries.

### Java

The literal original shape: `for (init; condition; update) { }`, with the two semicolons.

- **Initializer.** `var i = new Object() { ... }`. An anonymous class is the state holder, because the `for`
  initializer allows only one declaration. `var` keeps the anonymous type so its fields stay reachable. Fields: the
  Spring Boot context, the repository, the hours list, a category iterator, a lock, and an XML fragment.
- **Condition.** `total <= 40 && categories.hasNext() ? true : (BooleanSupplier) () -> ...`. When the loop should
  end, the else branch builds the window, shows it, and turns the result into `false` with `Double.isNaN(int)`, the
  original's trick.
- **Update.** A comma-separated list: lock, add hours, stage a card, unlock, then `new Thread(...).start()`.
- **Several steps in one expression.** `Stream.of(x).peek(...).peek(...)`.
- **Window.** A Swing `JDialog` described by Spring XML bean definitions built at runtime
  (`GenericXmlApplicationContext` over a string). A `javax.swing.Timer` bean fades the label colors. The modal dialog
  blocks until closed.
- **Saving.** Spring Data JPA `saveAll`, invoked through Spring's `ReflectionUtils`.

### Kotlin

Kotlin has no three-part `for` and no semicolons, so the loop iterates a sequence:
`for (pass in generateSequence(seed) { ... }) { }`.

- **Initializer.** An `object { ... }` expression is the sequence seed.
- **Condition.** `takeIf { if ... else ... }`. A null result ends the sequence. When the loop should end, the else
  branch shows the window and finishes with `.let { false }`, so `takeIf` returns null.
- **Update.** A chain of `?.also { ... }` calls.
- **Several steps in one expression.** `x.also { ... }.also { ... }`.
- **Window and saving.** The same Swing, Spring XML and Spring Data JPA as Java. The XML is one raw string with
  `${...}` templates instead of joined text blocks.
- **One behavior difference.** The Kotlin loop body runs once before the first condition check, where Java checks
  first. The body is empty, so both programs do the same things in the same order.

### Python

Python has no three-part `for`, no comma expressions and no statements inside expressions, so the loop is
`for _ in iter(step, False): ...`. Calling `step` again and again is the loop. It stops when `step` returns the
sentinel `False`.

- **Initializer.** A `SimpleNamespace` holds the state. It is passed to a lambda that builds `step`.
- **Condition.** One conditional expression: `(update ...) if total <= 40 and categories else (show the window ...)`.
- **Update.** A tuple of calls whose last element is `True`; `[-1]` picks it. Tuples evaluate left to right, which
  stands in for the comma list. It locks, adds hours, stages a card, unlocks, and starts a `threading.Thread`.
- **Empty body.** `...` is a bare constant expression. A `pass` would be a second statement.
- **Window.** A Tk window described by an XML string parsed at runtime with `xml.etree.ElementTree`, one widget per
  element via `getattr(tkinter, tag)`. `after()` repeats a timer that blends each label's color toward the
  background (Tk labels have no alpha). `mainloop()` blocks until the window closes.
- **Saving.** Standard-library `sqlite3`. `getattr(repo, "save_all")` is the reflection stand-in.
- **Caveat.** `main` is exactly one `for` statement, but the lambdas and comprehensions inside it are expressions
  only. Imports, the entity and the repository sit outside `main`.

### Go

Go is the strictest: a `for` clause allows only simple statements, and Go has no ternary, no comma expression and no
statements inside expressions. The loop is still a real three-part
`for i := &struct{...}{...}; condition; post { }` with the same two semicolons as Java.

- **Initializer.** `i := &struct{...}{...}`. An anonymous struct holds the state: the CSS fragment, hours, user,
  repository, staged cards, categories and a mutex.
- **Condition.** `total <= 40 && len(categories) > 0 || After(false, ShowDialog(...), WaitForSaves())`. The `||`
  stands in for the ternary: when the loop should end, the page is shown, the saves are awaited, and the result is
  `false`.
- **Update.** One call, `Seq(...)`, whose arguments run left to right. Go evaluates function calls in an argument
  list in order, so `Seq` stands in for the comma list: `Lock`, `Append` hours, `Append` a card, `Unlock`, `Spawn`.
- **The price.** Anything that needs a statement (`Lock`, `Append`, `Pop`, `Spawn`, `ShowDialog`) is a small
  value-returning function in `helpers.go`. The goroutine's body is one `Seq(...)` call.
- **Window.** Go has no standard-library GUI, so `ShowDialog` serves a page on a loopback port and opens it in the
  default browser. The pulse is a CSS animation fragment, as the original had an animation fragment. The page posts to
  `/closed` when you press Close or leave it, which unblocks the loop.
- **Saving.** `modernc.org/sqlite`, a pure-Go driver that needs no C compiler. `reflect`'s `MethodByName` calls
  `SaveAll`. `WaitForSaves` stops the process exiting before the goroutines finish.

## Side by side

| | Java | Kotlin | Python | Go |
| --- | --- | --- | --- | --- |
| The one statement | `for (init; cond; update) {}` | `for (pass in generateSequence(seed) {...}) {}` | `for _ in iter(step, False): ...` | `for init; cond; post {}` |
| Real three-part `for` | Yes | No | No | Yes |
| State holder | Anonymous class in a `var` | `object { }` as the seed | `SimpleNamespace` | Anonymous struct |
| Condition | `? :` expression | `takeIf { if ... else ... }` | Conditional expression | `a && b \|\| After(false, ...)` |
| Update (several steps) | Comma-separated expressions | Chain of `?.also { }` | Tuple, `[-1]` | `Seq(...)` call |
| Ending the loop | Condition returns `false` | `takeIf` yields null | `step` returns the sentinel `False` | Condition returns `false` |
| Empty body | `{ }` | `{ }` | `...` | `{ }` |
| Window | Swing from Spring XML | Swing from Spring XML | Tk from ElementTree XML | Browser page on loopback |
| Pulse | `javax.swing.Timer` | `javax.swing.Timer` | Tk `after()` timer | CSS animation |
| Blocks until closed | Modal `JDialog` | Modal `JDialog` | `mainloop()` | Wait for `/closed` |
| Database | Spring Data JPA, Hibernate, SQLite | Same | Standard-library `sqlite3` | `modernc.org/sqlite` |
| Background save | `Thread` | `Thread` | `threading.Thread` | Goroutine via `Spawn` |
| Lock | `ReentrantLock` | `ReentrantLock` | `threading.RLock` | `sync.Mutex` |
| "Reflection" | Spring `ReflectionUtils` | Spring `ReflectionUtils` | `getattr` | `reflect.MethodByName` |
| Code outside `main` | Entity, repository | Entity, repository | Entity, repository | Entity, repository and helper functions |
| Extra install | JDK 25 | JDK 25 | None | Go toolchain |

## What changed from the .NET version

| .NET version | Java and Kotlin | Python | Go |
| --- | --- | --- | --- |
| Anonymous object as iterator state | Anonymous class or object expression | `SimpleNamespace` | Anonymous struct |
| EF Core `DbContext` and SQLite | Spring Data JPA, Hibernate and SQLite | `sqlite3` | `modernc.org/sqlite` |
| `TimeCards.Add` then `SaveChanges` | A `pending` list, then `saveAll` | A `pending` list, then `save_all` | A `pending` slice, then `SaveAll` |
| WPF window from `XamlReader.Load` on a runtime XAML string | Swing window from `GenericXmlApplicationContext` on a runtime Spring XML string | Tk window from an XML string via `ElementTree` | HTML page built from a string, shown in the browser |
| `XElement` holding a `DoubleAnimation` | XML fragment holding a `Timer` bean | XML fragment holding the timer interval | CSS `@keyframes` fragment |
| Opacity animation on the `Viewbox` | Timer that fades the label colors | Timer that blends label colors | CSS opacity animation |
| `Application.Run` blocks until the window closes | Modal `JDialog.setVisible(true)` | `Tk.mainloop()` | Wait for the page's `/closed` request |
| LINQ to Objects `Sum` | `Stream` `sum` or `sum()` | `sum(...)` | A small `Sum` function |
| SQLite file under `%LOCALAPPDATA%` | `data/` in the project | `python/data/` | `go/data/` |
| Named `Mutex` | `ReentrantLock` | `threading.RLock` | `sync.Mutex` |
| `MethodInfo.Invoke` | Spring `ReflectionUtils.invokeMethod` | `getattr` | `reflect` `MethodByName` |
| `Random.Shared` | `ThreadLocalRandom.current()` | `random.Random()` | `math/rand/v2` |
| `ISOWeek.GetWeekOfYear` | `IsoFields.WEEK_OF_WEEK_BASED_YEAR` | `date.isocalendar().week` | `time.ISOWeek()` |
| `Environment.UserName` | `System.getProperty("user.name")` | `getpass.getuser()` | `USERNAME` or `USER` environment variable |

## From the original XAML to each port's markup

The .NET version builds its window from a XAML string at runtime with `XamlReader.Load`. XAML only runs on .NET, so
each port builds its UI from a different markup string at runtime instead:

- **Java and Kotlin** use standard Spring bean XML. Spring usually uses it to wire up services, and here it creates
  Swing widgets.
- **Python** uses a made-up XML format. Each tag is named after a tkinter class (`<Tk>`, `<Label>`), and
  `getattr(tkinter, tag)` creates the widget.
- **Go** uses plain HTML and CSS, rendered by the browser.

| UI piece | Original XAML (C#) | Java / Kotlin (Spring XML) | Python (custom XML) | Go (HTML/CSS) |
| --- | --- | --- | --- | --- |
| Root of the markup | `<Viewbox>` with WPF namespaces | `<beans>` with Spring namespaces | `<ui>` (any name works) | `<!doctype html>` page |
| The window | Created in C#; the XAML becomes its `Content` | `<bean id="window" class="javax.swing.JDialog">` | `<Tk id="window" .../>` | The browser window |
| Window title | Set in C# code | `<property name="title" .../>` | `title="..."` attribute | `<title>...</title>` |
| Size | `Width='520' Height='320'` on the Viewbox | `size` property, `Dimension(520, 320)` | Not set (fits the content) | Not set (browser decides) |
| Layout container | `<Grid>` with two `RowDefinition`s | `<bean id="panel" class="javax.swing.JPanel">` with `BorderLayout` | None: each label is stacked with `pack="..."` | `<body>` with `text-align:center` |
| Background color | `Background='#172033'` on the Grid | `background` property, `#172033` | `bg="#172033"` on every element | `background:#172033` style |
| Padding and margins | `Margin='10'`, `Margin='28,24,28,0'`, `Margin='28'` | `createEmptyBorder(24, 28, 40, 28)` on the panel | `pack="fill=x,padx=28,pady=24"` | `padding:48px` |
| Yellow banner text | `<TextBlock Foreground='#FFCE56' FontSize='18' FontWeight='Bold' .../>` | `<bean id="banner" class="javax.swing.JLabel">`, `#FFCE56`, SansSerif bold 18 | `<Label id="banner" fg="#FFCE56" font="Helvetica 18 bold" .../>` | `<h2 style="color:#FFCE56">` |
| Hours total text | Second `<TextBlock Grid.Row='1' FontSize='24' Foreground='White' .../>` | `<bean id="total" class="javax.swing.JLabel">`, white, bold 24 | `<Label id="total" fg="#FFFFFF" font="Helvetica 24 bold" .../>` | `<h1>` |
| "Saved locally" text | Same TextBlock as the total, after two line breaks (`&#x0a;&#x0a;`) | Separate `<bean id="saved" class="javax.swing.JLabel">`, bold 18 | Separate `<Label id="saved" .../>`, bold 18 | Separate `<h3>` |
| Centered text | `TextAlignment='Center'` | `horizontalAlignment` property, `0` (center) | `pack="fill=x"` (labels center by default) | `text-align:center` |
| Placing children in the layout | Nesting inside `<Grid>`, plus `Grid.Row` | `MethodInvokingBean` calling `panel.add(label, "North" / "Center" / "South")` | `parent="window"` attribute | Nesting inside `<body>` |
| Close button | None (the window's X) | None (the window's X) | None (the window's X) | `<button onclick="done()">Close</button>` |
| Animation fragment, built separately and inserted | `XElement` holding `<DoubleAnimation From="1" To="0.45" Duration="0:0:1.2" AutoReverse="True" RepeatBehavior="Forever" .../>` | `xmlAnim` string holding `<bean id="anim" class="javax.swing.Timer">`, ticking every 40 ms | `xml_anim` string holding `<timer interval="40"/>` | `animCSS` string holding `@keyframes pulse { from { opacity: 1 } to { opacity: .45 } }` |
| What starts the animation | `<EventTrigger RoutedEvent='FrameworkElement.Loaded'>` and a `<Storyboard>`, all in the markup | `init-method="start"` in the XML; Java code fades the labels on each tick | Python code reads `interval` and schedules ticks with `after()` | `animation: pulse 1.2s ease-in-out infinite alternate`, all in CSS |
| What fades | The whole Viewbox's `Opacity` | Each label's text color, from 100% to 45% alpha | Each label's color, blended toward the background | `h1`, `h2` and `h3` opacity |

The XAML and the Go CSS describe the animation completely in markup. Swing and Tk can't declare an animation in
markup, so in Java, Kotlin and Python the markup only declares a timer, and code outside it does the fading.

## License

MIT, carrying the original copyright notice. See [LICENSE](LICENSE).
