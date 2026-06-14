# Low Level Design (LLD) — 6-Week Mastery Plan

> **How to use this as a prompt:** Copy any week's section and paste it to an AI assistant with:
> _"Teach me this topic step by step with theory, full working code in Java/Python, and practice problems. Ask me quiz questions at the end to check my understanding."_

---

## Prerequisites Checklist

Before starting, make sure you are comfortable with:
- [ ] OOP concepts: Classes, Objects, Inheritance, Polymorphism, Encapsulation, Abstraction
- [ ] Basic Data Structures: Arrays, LinkedList, HashMap, Stack, Queue, Tree
- [ ] UML basics: Class diagrams, Sequence diagrams
- [ ] One OOP language fluency: Java / Python / C++

---

## Overall Plan at a Glance

| Week | Theme | Focus |
|---|---|---|
| 1 | Foundation | OOP Deep Dive + SOLID Principles |
| 2 | All Design Patterns | Creational + Structural + Behavioral (all 23 GoF patterns) |
| 3 | Real-World Problems (Part 1) | Developer Tools, Backend & Infrastructure Systems |
| 4 | Real-World Problems (Part 2) | Consumer Apps, Business & Domain Systems |
| 5 | Advanced LLD (Part 1) | Concurrency, Thread Safety, Caching, Pooling |
| 6 | Advanced LLD (Part 2) | Distributed Patterns, Event-Driven Design + Mock Interviews |

---

## Week 1 — OOP Fundamentals & SOLID Principles

### Topics
- Deep dive into OOP: Abstraction, Encapsulation, Inheritance, Polymorphism
- SOLID Principles:
  - **S** — Single Responsibility Principle (SRP)
  - **O** — Open/Closed Principle (OCP)
  - **L** — Liskov Substitution Principle (LSP)
  - **I** — Interface Segregation Principle (ISP)
  - **D** — Dependency Inversion Principle (DIP)
- UML Class Diagrams: associations, aggregation, composition, dependency
- Composition vs Inheritance — when to use which

### Real Example Problems

#### General / Everyday Systems
1. **Violation Spotter:** A `UserManager` class handles login, email alerts, and DB writes — identify all violated SOLID principles and refactor.
2. **Shape Area Calculator (OCP):** Add new shapes (Hexagon, Pentagon) without touching existing `AreaCalculator` logic.
3. **Payment Processor (LSP):** `CreditCard`, `Wallet`, and `UPI` must be substitutable for `PaymentMethod` without altering behavior.

#### Software Developer Specific
4. **CI/CD Pipeline (SRP):** A `BuildPipeline` class does code checkout, compilation, testing, artifact upload, and Slack notification — split it into focused classes.
5. **Plugin System (OCP):** Design a code analyzer that supports plugins (LintPlugin, SecurityPlugin, MetricsPlugin) — adding a new plugin must not change the core engine.
6. **Logger Interface (ISP):** Separate `Loggable`, `Auditable`, `Traceable` interfaces so a simple service only implements what it needs, not a fat `ILogger` interface.
7. **Repository Pattern (DIP):** Design a `UserService` that depends on `IUserRepository` (interface), allowing easy swap between `MySQLUserRepository` and `MongoUserRepository` in tests.

### Prompt Template for Week 1
```
Teach me SOLID principles in Low Level Design.
For each principle (S, O, L, I, D):
1. Explain with a real-world analogy
2. Show a BAD code example that violates it
3. Show a GOOD refactored version
4. Give one software developer-specific mini-problem to solve (e.g., related to APIs, CI/CD, logging)

Language: Java (or Python)
End with: 3 quiz questions to check my understanding.
```

---

## Week 2 — All Design Patterns (Creational + Structural + Behavioral)

> **Strategy for the week:** Study each category on separate days.
> Day 1–2: Creational | Day 3–4: Structural | Day 5–6: Behavioral | Day 7: Mixed practice + revision

---

### Part A — Creational Patterns (5 Patterns)

> **Intent:** Control how objects are created. Decouple construction logic from usage.

#### 2A.1 Singleton Pattern
- Only one instance exists; global access point
- **Use when:** Config manager, Logger, DB connection pool, Thread pool
- **Pitfall:** Not thread-safe without `synchronized` or double-checked locking

| Developer Example | Consumer Example |
|---|---|
| `ApplicationConfig` reads `app.properties` once at startup | Single `PrintSpooler` in an OS |
| `MetricsRegistry` (Micrometer/Prometheus) — one registry per JVM | Game's `GameManager` |
| `FeatureFlagService` — one in-memory flag cache | `ShoppingCart` session singleton |

**Problems to Solve:**
1. Design a thread-safe `Logger` that writes to file — only one instance must exist across all threads.
2. Design a `ConfigurationManager` that lazily loads `config.yaml` and exposes properties — singleton with lazy initialization.
3. Design a `ServiceRegistry` for microservices where each service registers itself; only one registry per process.

---

#### 2A.2 Factory Method Pattern
- Defer instantiation to subclasses; client code doesn't know the concrete class
- **Use when:** Object creation logic varies based on input/type

| Developer Example | Consumer Example |
|---|---|
| `DatabaseFactory` returns `MySQLDB` or `PostgresDB` based on config | `NotificationFactory` returns `EmailNotif` or `SMSNotif` |
| `SerializerFactory` returns `JsonSerializer` or `XmlSerializer` | `VehicleFactory` returns `Car`, `Bike`, `Truck` |
| `CloudStorageFactory` returns `S3Client` or `GCSClient` or `AzureBlobClient` | `PaymentFactory` returns `Stripe`, `PayPal`, `Razorpay` |

**Problems to Solve:**
1. Design a `ParserFactory` that returns the right parser (JSON, XML, CSV, YAML) based on a file extension string.
2. Design a `DatabaseConnectionFactory` that returns connections for MySQL, PostgreSQL, and SQLite.
3. Design a `LogHandlerFactory` for a logging framework: `ConsoleHandler`, `FileHandler`, `RemoteHandler`.

---

#### 2A.3 Abstract Factory Pattern
- Create families of related objects without specifying concrete classes
- **Use when:** Multiple product families must work together

| Developer Example | Consumer Example |
|---|---|
| `CloudProviderFactory` creates matching `VM`, `Storage`, `Network` objects for AWS/GCP/Azure | `UIFactory` creates matching `Button`, `Checkbox`, `Dialog` for Windows/Mac |
| `TestDoubleFactory` creates matching `MockRepo`, `MockMailer`, `MockCache` for unit tests | `FurnitureFactory` creates `Chair`, `Table`, `Sofa` for Victorian/Modern style |

**Problems to Solve:**
1. Design a cross-cloud infrastructure factory: `AWSFactory` and `GCPFactory` each produce compatible `ComputeInstance`, `ObjectStorage`, and `LoadBalancer`.
2. Design a test framework factory: `UnitTestFactory` vs `IntegrationTestFactory` create matching `TestRunner`, `Asserter`, and `Reporter`.

---

#### 2A.4 Builder Pattern
- Construct complex objects step by step; same process can create different representations
- **Use when:** Object has many optional fields, or construction is multi-step

| Developer Example | Consumer Example |
|---|---|
| `QueryBuilder` — `.select().from().where().orderBy().limit()` | `PizzaBuilder` — choose crust, sauce, size, toppings |
| `HttpRequestBuilder` — set URL, headers, body, timeout, retries | `ResumeBuilder` — add sections one by one |
| `DockerContainerBuilder` — set image, ports, volumes, env vars, memory | `HouseBuilder` — floors, rooms, garage, pool |

**Problems to Solve:**
1. Design a `SQLQueryBuilder` with methods: `select(cols)`, `from(table)`, `where(condition)`, `join(table)`, `orderBy(col)`, `limit(n)` — produce a final SQL string.
2. Design an `HttpRequest` builder that allows setting URL, method, headers (Map), body (optional), timeout, and retry count.
3. Design a `KafkaProducerConfig` builder: set broker, topic, acks, retries, serializer, compression type.

---

#### 2A.5 Prototype Pattern
- Clone an existing object instead of creating from scratch
- **Use when:** Object creation is expensive; need copies with slight variations

| Developer Example | Consumer Example |
|---|---|
| `RequestTemplate` clone with custom headers/body per API call | `GameEnemy` clone with different health values |
| `DocumentTemplate` clone for generating invoices with different data | `ResumeTemplate` clone per job application |
| `MicroserviceConfig` clone per environment (dev/staging/prod) | `UserProfile` clone for sandbox testing |

**Problems to Solve:**
1. Design a `ServerConfigPrototype` where base config (port, timeout, logging) is cloned per microservice and overridden.
2. Design a `ReportTemplate` prototype: clone a base report and change only title, date, and data source.

---

### Part B — Structural Patterns (7 Patterns)

> **Intent:** Compose classes/objects into larger structures while keeping them flexible and efficient.

---

#### 2B.1 Adapter Pattern
- Bridge two incompatible interfaces without changing either
- **Use when:** Integrating a third-party library, legacy system, or external API

| Developer Example | Consumer Example |
|---|---|
| Wrap `OldXMLParser` to match `IJsonParser` interface expected by new code | Wrap a 3-pin US plug to fit a 2-pin EU socket |
| Adapt `LegacyUserDAO` to new `UserRepository` interface | `MediaPlayerAdapter` plays MP4 using an existing VLC player |
| Wrap `AWSSDKv1Client` to match your internal `IStorageClient` contract | Adapt a foreign bank's API to your payment gateway contract |

**Problems to Solve:**
1. Your app uses `ILogger` with `log(level, message)`. A third-party library uses `Log4jLogger` with `write(severity, text)` — build an adapter.
2. Your code expects `IPaymentGateway.charge(amount, currency)`. A new vendor uses `VendorAPI.processPayment(data: Map)` — build an adapter.
3. Adapt a `LegacyCSVUserReader` (returns `String[][]`) to your new `IUserRepository` (returns `List<User>`).

---

#### 2B.2 Decorator Pattern
- Add behavior to objects dynamically at runtime without modifying the class
- **Use when:** You need flexible combinations of behaviors; subclassing creates class explosion

| Developer Example | Consumer Example |
|---|---|
| `GzipInputStream(BufferedInputStream(FileInputStream(file)))` | Coffee + Milk + Sugar + Caramel |
| HTTP middleware chain: `AuthDecorator → LoggingDecorator → RateLimitDecorator → Handler` | Notification: `BaseNotif + PriorityTag + EncryptedPayload` |
| `RetryDecorator(TimeoutDecorator(HTTPClient()))` | Pizza with multiple toppings |

**Problems to Solve:**
1. Design a `DataSource` decorator stack: `EncryptionDecorator(CompressionDecorator(FileDataSource))` — write/read applies decorators in order.
2. Design an HTTP client decorator chain: `RetryDecorator`, `LoggingDecorator`, `AuthDecorator` — each wraps the next.
3. Design a `TextFormatter` with decorators: `BoldDecorator`, `ItalicDecorator`, `UnderlineDecorator` — combinable at runtime.

---

#### 2B.3 Facade Pattern
- Provide a simplified, unified interface to a complex subsystem
- **Use when:** Hiding complexity, providing a clean API for a library or module

| Developer Example | Consumer Example |
|---|---|
| `OrderService.placeOrder()` internally calls InventoryService, PaymentService, NotificationService, ShippingService | `HomeTheater.watchMovie()` internally starts TV, projector, sound, dims lights |
| `DeploymentFacade.deploy()` internally calls Docker build, push, Helm upgrade, health check | `TravelBooking.book()` internally books flight, hotel, cab |
| AWS SDK — `S3Client.putObject()` hides multipart upload, retry, signature logic | `SmartHome.leaveHome()` locks doors, turns off lights, sets alarm |

**Problems to Solve:**
1. Design an `OrderFacade` that calls `InventoryService.reserve()`, `PaymentService.charge()`, `ShippingService.dispatch()`, and `EmailService.sendConfirmation()` behind one `placeOrder()` call.
2. Design a `CIPipelineFacade`: `runPipeline()` internally triggers `CodeCheckout`, `Build`, `Test`, `SecurityScan`, `Deploy`, `Notify`.
3. Design a `ReportFacade`: `generateMonthlyReport()` fetches from DB, applies filters, formats to PDF, and emails — hidden behind one method.

---

#### 2B.4 Proxy Pattern
- Control access to another object (security, lazy loading, caching, logging)
- **Types:** Virtual Proxy (lazy load), Protection Proxy (auth), Remote Proxy (RPC), Caching Proxy

| Developer Example | Consumer Example |
|---|---|
| `CachingProxy` wraps a `DatabaseRepository` — cache results, skip DB on cache hit | `VirtualProxy` loads a large image only on first `display()` |
| `AuthorizationProxy` wraps `AdminService` — checks role before allowing method calls | `ATM` is a proxy to the actual `BankAccount` on the server |
| `LoggingProxy` wraps any `Service` — logs every method call with duration | `CDN` is a proxy for the origin server |

**Problems to Solve:**
1. Design a `CachingDatabaseProxy` that wraps `IUserRepository` — caches `findById()` results in a `HashMap` with a TTL.
2. Design an `AuthorizationProxy` for `IAdminService` — checks if current user has `ADMIN` role before forwarding the call.
3. Design a `LoggingProxy` using Java dynamic proxy or Python `__getattr__` — logs method name, args, and response time for any service.

---

#### 2B.5 Composite Pattern
- Treat individual objects and compositions (trees) uniformly
- **Use when:** Part-whole hierarchies, tree structures (file system, org chart, UI components)

| Developer Example | Consumer Example |
|---|---|
| File system: `File` and `Folder` both implement `FileSystemItem` — `getSize()`, `delete()` | `MenuItem` and `Menu` both implement `MenuComponent` |
| `UIComponent` tree: `Panel` contains `Button`, `Label`, `TextField` — all respond to `render()` | Org chart: `Employee` and `Department` both have `getSalary()` |
| Build system: `TaskGroup` contains `Task` objects — all have `execute()` | HTML DOM: `Element` can contain other `Element` objects |

**Problems to Solve:**
1. Design a file system with `File` and `Directory` — both implement `FileSystemNode` with `getSize()`, `getName()`, `print(indent)`. A directory's size = sum of children.
2. Design a CI task tree: `StepGroup` and `Step` both implement `CITask` with `execute()` and `getStatus()`.
3. Design a composite `UIComponent` tree: `Screen → Panel → [Button, Label, InputField]` — all respond to `render()` and `onClick()`.

---

#### 2B.6 Bridge Pattern
- Decouple an abstraction from its implementation so both can vary independently
- **Use when:** You have 2 orthogonal dimensions of variation (avoid Cartesian product of subclasses)

| Developer Example | Consumer Example |
|---|---|
| `Notification` (Email/SMS/Push) × `MessageFormatter` (HTML/Plain/JSON) — bridge avoids 9 subclasses | `Shape` (Circle/Square) × `Color` (Red/Blue) |
| `DataExporter` (PDF/Excel/CSV) × `DataSource` (MySQL/Mongo/REST) | `RemoteControl` (Basic/Advanced) × `Device` (TV/Radio) |

**Problems to Solve:**
1. Design a messaging system: `Notification` abstraction (with `send()`) and `MessageChannel` implementor (Email, SMS, Push). Add a `UrgentNotification` without touching channel code.
2. Design a `ReportExporter` (PDF, CSV, Excel) × `DataSource` (SQL, NoSQL, API) — bridge so any combination works without a subclass for each pair.

---

#### 2B.7 Flyweight Pattern
- Share common state among many fine-grained objects to reduce memory
- **Use when:** Large number of similar objects; can extract shared (intrinsic) state

| Developer Example | Consumer Example |
|---|---|
| `CharacterStyle` objects (font, size, color) shared across millions of characters in a document editor | Tree objects in a forest game — share `TreeType` (texture, color, model) |
| `ConnectionMetadata` shared across all pooled DB connections (host, port, driver) | Bullet objects in a shooter game — share shape, color, sprite |
| `IconDescriptor` shared across thousands of UI icons in a file manager | Particle objects in a particle system simulation |

**Problems to Solve:**
1. Design a text editor where each character in a document has position (extrinsic) but shares `CharacterStyle` font/size/color (intrinsic) — avoid creating a new style object per character.
2. Design a game map renderer: `TreeType` (name, color, texture) is the flyweight shared by thousands of `Tree` instances with unique x, y positions.

---

### Part C — Behavioral Patterns (11 Patterns)

> **Intent:** Define how objects communicate, assign responsibilities, and encapsulate algorithms.

---

#### 2C.1 Observer Pattern
- Define a one-to-many dependency — when one object changes, all dependents are notified
- **Use when:** Event-driven systems, pub-sub, UI event listeners, reactive systems

| Developer Example | Consumer Example |
|---|---|
| CI/CD: `BuildEvent` notifies `SlackNotifier`, `EmailNotifier`, `DashboardUpdater` | Stock app: price change notifies multiple widgets |
| Kafka consumer group: topic change notifies all subscribers | Weather app: temperature update notifies all displays |
| Spring `@EventListener` / Node.js `EventEmitter` | Social media: new post notifies all followers |

**Problems to Solve:**
1. Design a `MetricsCollector` that notifies `AlertManager`, `Dashboard`, and `AuditLogger` when a metric breaches a threshold.
2. Design a `BuildSystem` where a build status change (`SUCCESS`, `FAILED`) notifies `SlackChannel`, `EmailService`, and `PRStatusUpdater`.
3. Design a stock price observer where `MobileApp`, `EmailAlert`, and `TradingBot` subscribe to price changes on a stock.

---

#### 2C.2 Strategy Pattern
- Define a family of algorithms, encapsulate each one, and make them interchangeable at runtime
- **Use when:** Multiple variations of an algorithm; avoid if-else chains

| Developer Example | Consumer Example |
|---|---|
| `AuthStrategy`: `JWTStrategy`, `OAuthStrategy`, `APIKeyStrategy` | `SortStrategy`: BubbleSort, QuickSort, MergeSort |
| `CompressionStrategy`: `GzipCompressor`, `ZstdCompressor`, `LZ4Compressor` | `PaymentStrategy`: Stripe, PayPal, Crypto |
| `RoutingStrategy`: `RoundRobin`, `LeastConnections`, `WeightedRandom` | `DiscountStrategy`: Flat, Percentage, BuyOneGetOne |

**Problems to Solve:**
1. Design a file compression service where strategy (Gzip, Zstd, Brotli) is chosen at runtime based on file type and size.
2. Design an `AuthenticationService` that supports swappable strategies: `UsernamePassword`, `OAuth2`, `APIKey`, `SAML`.
3. Design a load balancer with pluggable routing strategies: `RoundRobin`, `LeastConnections`, `IPHash`.

---

#### 2C.3 Command Pattern
- Encapsulate a request as an object — supports undo/redo, queuing, logging, and macro commands
- **Use when:** Undo/redo, task queues, macro recording, transactional operations

| Developer Example | Consumer Example |
|---|---|
| Database transaction: `InsertCommand`, `UpdateCommand`, `DeleteCommand` with rollback | Text editor Undo/Redo |
| Job queue: serialize `SendEmailCommand`, `ResizeImageCommand` and dispatch to workers | Remote control buttons as commands |
| Microservice saga: `ReserveInventoryCommand`, `ChargePaymentCommand` — compensating commands on failure | Smart home: `TurnOnLightCommand`, `SetThermostatCommand` |

**Problems to Solve:**
1. Design a text editor with `InsertTextCommand`, `DeleteTextCommand`, `ReplaceTextCommand` — all support `execute()` and `undo()`.
2. Design a job queue system: `Command` objects (EmailJob, ResizeJob, BackupJob) are serialized and dispatched to workers asynchronously.
3. Design a database migration tool: each migration is a `Command` with `up()` and `down()` — run/rollback individual migrations.

---

#### 2C.4 Iterator Pattern
- Provide a way to sequentially access elements of a collection without exposing its internal structure
- **Use when:** Custom data structures need traversal; hide collection internals

| Developer Example | Consumer Example |
|---|---|
| Custom `PaginatedResultIterator` that fetches next DB page on `next()` | Java's `Iterator<T>`, Python's `__iter__` |
| `DirectoryTreeIterator` — depth-first or breadth-first traversal | Playlist iterator — next/previous song |
| `LogFileIterator` — reads log entries one by one from compressed file | Newsfeed infinite scroll iterator |

**Problems to Solve:**
1. Design a `PaginatedAPIIterator` that fetches pages lazily from a REST API on `hasNext()`/`next()` — hides pagination logic from the caller.
2. Design a `BinaryTreeIterator` that supports both in-order and pre-order traversal without exposing the tree structure.

---

#### 2C.5 State Pattern
- Allow an object to alter its behavior when its internal state changes — the object will appear to change its class
- **Use when:** Object behavior changes dramatically based on state; avoids large if-else or switch

| Developer Example | Consumer Example |
|---|---|
| `OrderStateMachine`: `Placed → Confirmed → Shipped → Delivered → Returned` | ATM: `Idle → CardInserted → PINEntered → Dispensing` |
| `TrafficLight`: `Green → Yellow → Red → Green` | Vending Machine: `Idle → HasMoney → Dispensing → OutOfStock` |
| `CI Job`: `Queued → Running → Passed / Failed / Cancelled` | Document: `Draft → Review → Published → Archived` |

**Problems to Solve:**
1. Design an `Order` state machine with states: `Placed → Confirmed → Packed → Shipped → Delivered` — each state only allows valid transitions.
2. Design a `TrafficLight` system with states `Red`, `Green`, `Yellow` — each state knows its next state and duration.
3. Design a CI/CD `PipelineJob` with states: `Queued → Running → Passed / Failed / Cancelled` — only valid transitions allowed.

---

#### 2C.6 Chain of Responsibility Pattern
- Pass a request along a chain of handlers — each handler decides to process or forward
- **Use when:** Multiple handlers may process a request; decouple sender from receiver

| Developer Example | Consumer Example |
|---|---|
| HTTP middleware: `AuthMiddleware → RateLimitMiddleware → LoggingMiddleware → Handler` | Customer support: `Tier1 → Tier2 → Manager` |
| Request validation pipeline: `InputSanitizer → AuthValidator → SchemaValidator → BusinessRuleValidator` | Expense approval: `TeamLead → Manager → Director → CFO` |
| Exception handling: `AppExceptionHandler → HttpExceptionHandler → GlobalExceptionHandler` | Help desk ticket escalation |

**Problems to Solve:**
1. Design an HTTP request pipeline: `AuthHandler → RateLimitHandler → InputValidationHandler → BusinessLogicHandler` — each can short-circuit.
2. Design an expense approval chain: `< $100: TeamLead`, `< $1000: Manager`, `< $10000: Director`, `> $10000: CFO`.
3. Design a log level chain: `DEBUG → INFO → WARN → ERROR` — each handler processes messages at its level and above.

---

#### 2C.7 Template Method Pattern
- Define the skeleton of an algorithm in a base class; subclasses fill in the steps
- **Use when:** Multiple classes share the same algorithm structure but differ in specific steps

| Developer Example | Consumer Example |
|---|---|
| `DataMigration`: `connect() → fetchData() → transform() → load() → disconnect()` — each migration overrides transform | `BeverageRecipe`: `boilWater() → brew() → pourInCup() → addCondiments()` |
| `ReportGenerator`: `fetchData() → processData() → renderOutput()` — PDF, Excel, HTML override renderOutput | `GameAI`: `takeTurn()` → `collectResources()` → `buildStructures()` → `attack()` |
| `BaseTestCase`: `setUp() → runTest() → tearDown()` (JUnit/pytest follows this) | `OnlineOrderProcess`: `selectItem() → checkout() → pay() → confirm()` |

**Problems to Solve:**
1. Design a `DataPipeline` base class: `extract() → transform() → load()` — `CSVPipeline`, `APIPipeline`, `DBPipeline` override individual steps.
2. Design a `TestFramework` base: `beforeAll() → setUp() → runTest() → tearDown() → afterAll()` — test classes override `runTest()` and optionally hook steps.
3. Design a `DocumentParser`: `openFile() → parseContent() → extractMetadata() → close()` — `PDFParser`, `WordParser`, `MarkdownParser` override parsing steps.

---

#### 2C.8 Mediator Pattern
- Define an object that encapsulates how a set of objects interact — reduces direct references between them
- **Use when:** Many objects communicate in complex ways; reduce coupling

| Developer Example | Consumer Example |
|---|---|
| `EventBus` / `MessageBroker` mediates between microservices | Air traffic control tower mediates between aircraft |
| `ChatRoom` mediates between `User` objects — users don't talk directly | UI `DialogMediator` coordinates `TextBox`, `ListBox`, `Button` |
| `Orchestrator` in saga pattern mediates between individual services | Stock exchange mediates between buyers and sellers |

**Problems to Solve:**
1. Design an in-memory `EventBus` where microservices publish events and multiple subscribers receive them — services never call each other directly.
2. Design a `ChatRoom` mediator where `User` objects send messages through the room — users are unaware of each other's references.

---

#### 2C.9 Memento Pattern
- Capture and restore an object's internal state without violating encapsulation
- **Use when:** Undo/redo, snapshots, save-game states

| Developer Example | Consumer Example |
|---|---|
| `CodeEditor` save checkpoints — restore to a previous state | Word processor document undo history |
| `DatabaseTransaction` savepoint — rollback to a savepoint on error | Game save/load checkpoint |
| `ConfigSnapshot` — save config state before a deploy; rollback on failure | Photoshop history panel |

**Problems to Solve:**
1. Design a code editor with save/restore: `Editor.save()` returns a `Memento`; `Editor.restore(memento)` rolls back to that state.
2. Design a transaction system: before executing a batch of DB writes, save a snapshot; on failure, restore.

---

#### 2C.10 Visitor Pattern
- Add new operations to existing class hierarchies without modifying them
- **Use when:** Need to add operations to a stable class hierarchy; separate algorithm from object structure

| Developer Example | Consumer Example |
|---|---|
| `ASTVisitor`: `TypeCheckVisitor`, `CodeGenVisitor`, `PrettyPrintVisitor` traverse an AST | `TaxVisitor` calculates tax differently per item type |
| `ReportVisitor` generates different report formats for each node type in a document tree | `AreaVisitor` calculates area of different shapes |
| `SerializerVisitor` serializes different node types of a config tree to JSON/XML | Zoo app: `FeedingVisitor` behaves differently per animal |

**Problems to Solve:**
1. Design a compiler AST with `NumberNode`, `AddNode`, `MultiplyNode` — add a `EvaluatorVisitor` and `PrintVisitor` without changing node classes.
2. Design an order system with `PhysicalItem`, `DigitalItem`, `ServiceItem` — add `TaxCalculatorVisitor` and `ShippingCostVisitor`.

---

#### 2C.11 Interpreter Pattern
- Define a grammar for a language and provide an interpreter to process sentences
- **Use when:** Scripting, DSLs, query languages, expression evaluation

| Developer Example | Consumer Example |
|---|---|
| SQL `WHERE` clause parser: `AND`, `OR`, `NOT`, `EQUALS` expressions | Boolean logic evaluator |
| Math expression evaluator: `3 + 5 * (2 - 1)` | Search query parser: `author:John AND tag:java` |
| Feature flag DSL: `env == 'prod' AND rollout < 0.5` | Regular expression engine |

**Problems to Solve:**
1. Design a boolean expression interpreter: parse and evaluate `"true AND (false OR true)"`.
2. Design a simple search query interpreter: parse `"title:LLD AND tag:patterns NOT difficulty:hard"` into a filter object.

---

### Week 2 — Combined Prompt Template

```
Teach me ALL design patterns organized by category (Creational, Structural, Behavioral).

For each pattern:
1. One-line intent
2. Real-world software developer analogy (CI/CD, APIs, databases, editors)
3. Class diagram described textually
4. Full working code example in [Java/Python]
5. When to USE vs when NOT to USE
6. Contrast with 1 commonly confused pattern

After covering all patterns in a category, give me:
- A mixed problem that uses 2 patterns from that category together
- 3 quiz questions to test my understanding

Start with Creational: Singleton, Factory, Abstract Factory, Builder, Prototype.
```

---

## Week 3 — Real-World Problems: Developer Tools & Backend Systems

> **Focus:** Systems that software developers build, maintain, and use daily.
> **Goal:** Practice applying multiple patterns in the context of backend infrastructure, tooling, and developer platforms.

---

### Problem 3.1: Design a Logging Framework (like Log4j / SLF4J)

**Actors:** `Logger`, `LogLevel`, `Appender` (Console, File, Remote), `LogFormatter`, `LogRecord`, `LogManager`

**Features:**
- Multiple log levels: TRACE, DEBUG, INFO, WARN, ERROR, FATAL
- Multiple appenders simultaneously (write to file AND console AND remote)
- Custom formatters (JSON, Plain text, XML)
- Logger hierarchy (package-based inheritance of log levels)
- Async logging option for performance

**Patterns Used:**
| Pattern | Application |
|---|---|
| Singleton | `LogManager` — one registry per JVM |
| Chain of Responsibility | Log level filtering chain |
| Decorator | Appender wrapping (AsyncAppender wraps FileAppender) |
| Observer | Appenders as observers of log events |
| Template Method | Base `Appender.append()` calls `format() → write() → flush()` |
| Strategy | Pluggable `LogFormatter` |

**Design Steps:**
1. Define `Logger`, `LogRecord`, `LogLevel` enum
2. Define `IAppender` interface: `append(LogRecord)`
3. Build `ConsoleAppender`, `FileAppender`, `RollingFileAppender`
4. Build `LogFormatter` with `JsonFormatter` and `PlainTextFormatter`
5. Build `LogManager` as Singleton with `getLogger(name)`

**Extension Problems:**
- Add `AsyncAppender` decorator that buffers log records in a queue
- Add log level filtering per appender
- Add MDC (Mapped Diagnostic Context) for tracing request IDs across log lines

---

### Problem 3.2: Design a Rate Limiter Service

**Actors:** `RateLimiter`, `TokenBucket`, `LeakyBucket`, `SlidingWindowCounter`, `RateLimitRule`, `ClientIdentifier`

**Features:**
- Support multiple algorithms: Token Bucket, Leaky Bucket, Fixed Window, Sliding Window
- Per-user, per-API-key, per-IP rate limiting
- Configurable rules per endpoint
- Thread-safe for concurrent requests

**Patterns Used:**
| Pattern | Application |
|---|---|
| Strategy | Pluggable algorithm (TokenBucket, SlidingWindow) |
| Singleton | `RateLimiterRegistry` — global rule store |
| Factory | `RateLimiterFactory.create(type, config)` |
| Decorator | `LoggingRateLimiter` wraps any `IRateLimiter` |

**Design Steps:**
1. Define `IRateLimiter` interface: `boolean allowRequest(clientId, endpoint)`
2. Implement `TokenBucketLimiter`, `SlidingWindowLimiter`
3. Implement `RateLimiterRegistry` to store per-client limiters
4. Implement `RateLimiterFactory` to create limiters by config
5. Thread-safe with `ReentrantLock` or `AtomicInteger`

**Extension Problems:**
- Add distributed rate limiting using Redis counters
- Add burst allowance (allow 10 requests instantly, then enforce 1/sec)

---

### Problem 3.3: Design a Caching System (like Redis / Guava Cache)

**Actors:** `Cache`, `CacheEntry`, `EvictionPolicy`, `CacheLoader`, `CacheStats`

**Features:**
- LRU (Least Recently Used) and LFU (Least Frequently Used) eviction
- TTL (Time-To-Live) per entry
- Cache-aside, write-through, write-behind loading strategies
- Thread-safe access
- Stats: hit rate, miss rate, eviction count

**Patterns Used:**
| Pattern | Application |
|---|---|
| Strategy | Pluggable `EvictionPolicy` (LRU, LFU, FIFO) |
| Decorator | `StatsCollectingCache` wraps `BasicCache` |
| Template Method | `AbstractCache.get()` calls `checkTTL() → lookup() → onMiss()` |
| Proxy | `CachingProxy` sits in front of `DataRepository` |
| Observer | Cache eviction event notifies listeners |

**Data Structures:**
- LRU: `HashMap` + `DoublyLinkedList` → O(1) get/put
- LFU: `HashMap<key, node>` + `HashMap<freq, LinkedHashSet>` → O(1) get/put

**Extension Problems:**
- Add write-behind (async persist to DB on put)
- Add read-through (auto-load from DB on cache miss using `CacheLoader`)
- Add segmented locking for reduced contention in concurrent access

---

### Problem 3.4: Design a Plugin / Extension System (like VSCode Extensions)

**Actors:** `PluginRegistry`, `Plugin`, `PluginLoader`, `PluginContext`, `EventHook`, `ExtensionPoint`

**Features:**
- Register and discover plugins at runtime
- Plugins contribute to extension points (commands, UI, parsers)
- Plugin lifecycle: load, activate, deactivate, unload
- Plugins communicate through events, not directly

**Patterns Used:**
| Pattern | Application |
|---|---|
| Observer | Plugins subscribe to IDE events (fileSaved, buildCompleted) |
| Strategy | Plugins implement `ILanguageProvider`, `IFormatter` strategies |
| Factory | `PluginLoader` creates plugin instances from manifests |
| Mediator | `EventBus` mediates between plugins |
| Composite | Plugin contributes a tree of `Command` objects |

**Extension Problems:**
- Design a dependency resolver: Plugin B requires Plugin A to be loaded first
- Design a plugin permission system: plugins declare required capabilities in manifest

---

### Problem 3.5: Design a CI/CD Pipeline Engine (like GitHub Actions / Jenkins)

**Actors:** `Pipeline`, `Stage`, `Step`, `Job`, `Trigger`, `Artifact`, `Runner`, `PipelineExecutor`

**Features:**
- Define pipelines as DAGs of stages and steps
- Parallel and sequential stage execution
- Artifacts passed between stages
- Trigger on push, PR, schedule, or manual
- Retry logic, timeout, and cancellation per step

**Patterns Used:**
| Pattern | Application |
|---|---|
| Composite | `Pipeline → [Stage → [Step]]` — all implement `IExecutable` |
| Command | Each `Step` is a command with `execute()` and optionally `undo()` |
| Chain of Responsibility | Pre-execution hooks: AuthCheck → ResourceCheck → ConcurrencyCheck |
| Observer | `PipelineEventBus` notifies Slack, Email, Dashboard on status changes |
| Strategy | `ExecutionStrategy`: Sequential vs Parallel runner |
| Builder | `PipelineBuilder.stage("build").step("compile").step("test").build()` |
| Template Method | `BaseStep.run()` calls `preExecute() → execute() → postExecute()` |

**Extension Problems:**
- Design a DAG scheduler that handles step dependencies
- Design a retry policy per step with exponential backoff

---

### Problem 3.6: Design a Dependency Injection Container (like Spring IoC)

**Actors:** `BeanDefinition`, `BeanFactory`, `ApplicationContext`, `Scope` (Singleton, Prototype), `DependencyResolver`

**Features:**
- Register beans by type or name
- Auto-wire dependencies by type
- Support Singleton and Prototype scopes
- Lifecycle callbacks: `@PostConstruct`, `@PreDestroy`
- Circular dependency detection

**Patterns Used:**
| Pattern | Application |
|---|---|
| Factory | `BeanFactory.getBean(type)` |
| Singleton | Singleton-scoped beans cached in registry |
| Proxy | AOP proxies wrap beans for transaction, logging |
| Template Method | `AbstractBeanFactory.getBean()` — resolveDefinition → create → inject → init |

**Extension Problems:**
- Add `@Qualifier` support to choose among multiple candidates of the same type
- Add lazy initialization — bean only created when first requested

---

### Week 3 — Prompt Template
```
I want to practice designing backend/developer-tool systems for LLD interviews.

Problem: [Design a Rate Limiter Service]

Walk me through this design as a senior engineer:
Step 1: Ask me 5 clarifying questions about requirements
Step 2: Define the core entities and their attributes
Step 3: Define the interfaces and their contracts
Step 4: Identify which design patterns fit and why
Step 5: Code the core classes with real method implementations in [Java/Python]
Step 6: Walk through a concrete use case end-to-end
Step 7: Give me 2 extension problems to challenge my design

Be my interviewer — push back if my design has flaws.
```

---

## Week 4 — Real-World Problems: Consumer Apps & Business Systems

> **Focus:** End-user applications, domain-driven systems, and business workflows.
> **Goal:** Practice entity modeling, state machines, and pattern combinations in real product scenarios.

---

### Problem 4.1: Design a Parking Lot System

**Actors:** `ParkingLot`, `ParkingFloor`, `ParkingSpot`, `Vehicle`, `Ticket`, `EntryGate`, `ExitGate`, `PricingStrategy`

**Features:**
- Multi-level parking floors
- Spot types: Compact, Regular, Large, Handicapped, Electric
- Vehicle types: Bike, Car, Van, Truck
- Entry/exit gate with ticket generation
- Dynamic pricing: hourly, flat, overnight

**Patterns Used:** Singleton (ParkingLot), Factory (Vehicle), Strategy (Pricing), Observer (spot availability alerts)

**State Machine:** `AVAILABLE → RESERVED → OCCUPIED → AVAILABLE`

**Extension Problems:**
- Add electric vehicle charging spot tracking
- Add a monthly subscription pass system

---

### Problem 4.2: Design a Hotel Booking System (Booking.com / MakeMyTrip)

**Actors:** `Hotel`, `Room`, `RoomType`, `Guest`, `Booking`, `Payment`, `Review`, `SearchService`

**Features:**
- Search hotels by location, date, guests, price range
- Room types: Standard, Deluxe, Suite
- Real-time availability check
- Booking lifecycle: `Requested → Confirmed → CheckedIn → CheckedOut → Cancelled`
- Payment with hold/capture/refund
- Reviews and ratings post-checkout

**Patterns Used:** Builder (search query), Strategy (payment), State (booking lifecycle), Observer (notify on booking confirmation)

**Extension Problems:**
- Add dynamic pricing (prices increase as availability decreases)
- Add loyalty points system per booking

---

### Problem 4.3: Design an Elevator System

**Actors:** `Elevator`, `ElevatorController`, `Floor`, `FloorButton`, `CabinButton`, `Direction`, `ElevatorDoor`

**Features:**
- Multiple elevators in a building
- Up/down requests from each floor
- Floor selection from inside the cabin
- SCAN (elevator algorithm) scheduling
- Emergency stop and overload detection

**Patterns Used:** State (elevator states: Idle/Moving/DoorOpen/Emergency), Strategy (scheduling algorithm), Observer (floor arrival notifications), Singleton (ElevatorController)

**State Machine:** `IDLE → MOVING_UP / MOVING_DOWN → DOOR_OPENING → DOOR_OPEN → DOOR_CLOSING → IDLE`

**Extension Problems:**
- Add VIP floor priority routing
- Add energy-saving mode: park elevators at ground floor during off-hours

---

### Problem 4.4: Design a Chess Game

**Actors:** `Game`, `Board`, `Cell`, `Piece` (King, Queen, Rook, Bishop, Knight, Pawn), `Player`, `Move`, `MoveValidator`

**Features:**
- Full piece movement rules
- Check and checkmate detection
- En passant, castling, pawn promotion
- Undo last move
- Game history / move log

**Patterns Used:** Command (undo/redo moves), Template Method (each piece's `isValidMove()`), Iterator (traverse board), Memento (save game state)

**Extension Problems:**
- Add a timer per player (chess clock)
- Add AI opponent using minimax (describe the strategy, not the algorithm)

---

### Problem 4.5: Design an Online Food Ordering System (Swiggy / Zomato)

**Actors:** `User`, `Restaurant`, `MenuItem`, `Cart`, `Order`, `DeliveryPartner`, `Payment`, `Review`

**Features:**
- Browse and search restaurants
- Cart management (add, remove, update quantity)
- Order placement with real-time tracking
- Delivery partner assignment
- Multiple payment methods
- Order state machine

**Patterns Used:** Observer (order tracking updates), Strategy (payment method), Facade (order placement), State (order lifecycle), Factory (payment processor)

**Order State Machine:** `Placed → Confirmed → Preparing → PickedUp → OutForDelivery → Delivered`

**Extension Problems:**
- Add surge pricing based on demand and weather
- Add group ordering (multiple users contribute to one cart)

---

### Problem 4.6: Design a Movie Ticket Booking System (BookMyShow / Fandango)

**Actors:** `Movie`, `Show`, `Screen`, `Seat`, `Booking`, `User`, `Payment`, `Theatre`

**Features:**
- Browse movies by city, date, genre
- View available seats on a screen layout
- Seat locking during booking (prevent double-booking)
- Payment and booking confirmation
- Cancellation with refund policy

**Patterns Used:** Singleton (BookingManager), State (seat: Available → Locked → Booked), Strategy (refund policy), Observer (notify waitlisted users on cancellation)

**Concurrency Challenge:** Two users select the same seat — how do you handle it?
- Optimistic locking: check version on confirm
- Pessimistic locking: lock seat row in DB during selection window
- Seat lock timeout: 5-minute hold, auto-release if payment not completed

**Extension Problems:**
- Add a waitlist for sold-out shows
- Design the seat map renderer (use Composite pattern)

---

### Problem 4.7: Design a Ride-Sharing App (Uber / Ola)

**Actors:** `Rider`, `Driver`, `Trip`, `Location`, `VehicleType`, `PricingEngine`, `MatchingService`, `Payment`

**Features:**
- Request ride by pickup/drop location
- Match nearest available driver
- Real-time trip tracking
- Dynamic surge pricing
- Rating after trip
- Multiple vehicle types: Auto, Sedan, SUV, Bike

**Patterns Used:** Strategy (driver matching: NearestFirst, RatingBased), Observer (trip status updates), State (trip lifecycle), Factory (vehicle type), Command (cancel/modify trip)

**Trip State Machine:** `Requested → DriverAssigned → DriverArrived → TripStarted → TripCompleted`

**Extension Problems:**
- Add carpooling (multiple riders in one trip)
- Design the pricing engine with surge multiplier

---

### Problem 4.8: Design a Library Management System

**Actors:** `Book`, `BookItem`, `Member`, `Librarian`, `Reservation`, `Loan`, `Fine`, `Catalog`

**Features:**
- Search books (title, author, ISBN, genre)
- Issue and return books
- Reservation queue for unavailable books
- Automatic fine calculation for overdue
- Member account with borrowing limits

**Patterns Used:** Observer (notify reserved user on book return), Builder (search query), Strategy (fine calculation — per-day vs flat), Singleton (Library instance)

**Extension Problems:**
- Add a recommendation system based on borrowing history
- Add e-book support with download tracking

---

### Week 4 — Prompt Template
```
Act as a FAANG interviewer. I'm solving an LLD problem.

Problem: [Design a Movie Ticket Booking System]

I'll design it step by step:
1. First I'll clarify requirements — correct any wrong assumptions
2. Then I'll list the core entities — point out missing ones
3. Then I'll draw the class hierarchy — highlight bad inheritance decisions
4. Then I'll apply design patterns — challenge me if a pattern is forced
5. Then I'll walk through a seat booking flow — point out race conditions or edge cases

After I'm done, score my design 1–10 on: correctness, extensibility, pattern usage, edge case handling.
Give specific improvement suggestions.
```

---

## Week 5 — Advanced LLD Part 1: Concurrency, Thread Safety & Performance

> **Focus:** Designing correct, high-performance, thread-safe systems from first principles.

---

### 5.1 Concurrency Fundamentals for LLD

**Core Concepts:**
- Thread safety: race conditions, visibility, atomicity
- Synchronization mechanisms: `synchronized`, `ReentrantLock`, `ReadWriteLock`, `StampedLock`
- `volatile` keyword — memory visibility without atomicity
- Atomic classes: `AtomicInteger`, `AtomicReference`, `AtomicLong`
- `ConcurrentHashMap`, `CopyOnWriteArrayList`, `BlockingQueue`

**Developer-Specific Problems:**
1. **Thread-Safe Counter:** Design a `RequestCounter` used by multiple threads to count HTTP requests per endpoint.
2. **Concurrent HashMap with Expiry:** Design a `TTLCache<K,V>` using `ConcurrentHashMap` where entries expire after a set duration.
3. **Read-Write Separated Repository:** Design a `UserRepository` that allows concurrent reads but exclusive writes using `ReadWriteLock`.

---

### 5.2 Thread-Safe Singleton (All Approaches)

**Approaches to cover:**
- Eager initialization
- Lazy initialization (broken without sync)
- Synchronized method (slow — lock on every call)
- Double-checked locking with `volatile` (correct + fast)
- Bill Pugh Singleton (static inner class — simplest and correct)
- Enum Singleton (serialization-safe)

**Problems to Solve:**
1. Implement all 6 approaches and explain their trade-offs.
2. Demonstrate how a Singleton can break with reflection — and how to prevent it.
3. Design a `FeatureFlagService` singleton that reloads flags from a config file every 60 seconds without locking reads.

---

### 5.3 Producer-Consumer & Blocking Queue

**Core Design:**
- `BlockingQueue<T>` with `put()` (waits if full) and `take()` (waits if empty)
- Implementation: circular buffer, `ReentrantLock` + `Condition`

**Problems to Solve:**
1. Implement `BlockingQueue<T>` from scratch with capacity, `put()`, `take()`, `size()` — thread-safe.
2. Design a `TaskExecutor` (simplified `ThreadPoolExecutor`): producers submit tasks to a queue; worker threads consume and execute.
3. Design a `LogAsyncAppender`: logs are placed in a `BlockingQueue` by application threads; a single background thread drains and writes to file.

---

### 5.4 Object Pool Pattern

**Core Design:**
- Pre-allocate N reusable objects
- `acquire()` blocks if no object available
- `release()` returns object to pool
- Idle timeout to shrink pool

**Problems to Solve:**
1. Design a `DatabaseConnectionPool` with `minSize`, `maxSize`, `acquireTimeout` — connections are reused across requests.
2. Design an `HttpClientPool` where a limited number of `HttpClient` instances are shared across API call threads.
3. Design a `WorkerThreadPool`: N worker threads are pre-created; tasks submitted to a queue are picked up by available workers.

---

### 5.5 LRU Cache (Thread-Safe, O(1))

**Data Structures:** `HashMap<key, DLinkedNode>` + `DoublyLinkedList`
- `get(key)` — O(1): find in map, move node to front
- `put(key, value)` — O(1): insert at front, evict tail if capacity exceeded

**Problems to Solve:**
1. Implement `LRUCache<K,V>` with O(1) `get()` and `put()` using `HashMap + DoublyLinkedList`.
2. Add thread safety with `ReentrantReadWriteLock` — concurrent reads, exclusive writes.
3. Extend to `LFUCache<K,V>` — evict least frequently used, with tie-broken by recency. (Uses `HashMap<key,node>` + `HashMap<freq, LinkedHashSet>`).

---

### 5.6 Rate Limiter — Deep Concurrency Design

**Algorithms:**
- **Token Bucket:** Tokens added at fixed rate; request consumes a token. Allows bursts.
- **Leaky Bucket:** Requests flow out at fixed rate. Smooths traffic.
- **Fixed Window Counter:** Count requests in fixed time window.
- **Sliding Window Log:** Store timestamp of each request in a window.

**Problems to Solve:**
1. Implement thread-safe `TokenBucketRateLimiter` using `AtomicLong` for token count — no `synchronized` blocks.
2. Implement `SlidingWindowRateLimiter` using a `ConcurrentLinkedDeque` of timestamps.
3. Design a distributed rate limiter API: each app node calls `checkLimit(userId, endpoint)` — state stored in Redis (describe the design, not Redis internals).

---

### Week 5 — Prompt Template
```
Teach me concurrent LLD implementation in Java (or Python with threading).

Topic: Thread-Safe LRU Cache

1. First explain the problem — why naive HashMap is not thread-safe
2. Show the non-thread-safe version with HashMap + DoublyLinkedList
3. Identify all race conditions
4. Show the fixed version with ReentrantReadWriteLock
5. Analyze throughput vs correctness trade-offs

Follow-up: Show the LFU Cache implementation.
Then give me a timed practice: implement BlockingQueue from scratch — 20 minutes.
```

---

## Week 6 — Advanced LLD Part 2: Distributed Patterns, Event-Driven & Mock Interviews

> **Focus:** Patterns and designs at the boundary of LLD and System Design — event sourcing, CQRS, sagas, and distributed object designs.

---

### 6.1 Event-Driven Architecture Patterns

**Core Patterns:**
- **Event Sourcing:** Store events as the source of truth; derive state by replaying
- **CQRS:** Separate read model (Query) and write model (Command)
- **Outbox Pattern:** Reliably publish events alongside DB writes
- **Saga Pattern:** Manage distributed transactions through compensating events

**LLD Components to Design:**
- `Event`, `EventStore`, `EventPublisher`, `EventHandler`, `Aggregate`
- `CommandBus`, `QueryBus`, `Projection`

**Problems to Solve:**
1. Design an `OrderAggregate` using Event Sourcing: events are `OrderPlaced`, `OrderConfirmed`, `OrderShipped`, `OrderCancelled` — derive current state by replaying.
2. Design a CQRS `UserService`: `UserCommandService` handles creates/updates; `UserQueryService` reads from a denormalized read model.
3. Design the Outbox Pattern: `OrderService.placeOrder()` saves order to DB and an outbox table atomically — a background `OutboxPublisher` reads and publishes events.

---

### 6.2 Pub-Sub Event Bus (In-Memory)

**Design Goals:**
- Decouple publishers from subscribers
- Support typed events
- Async delivery with a thread pool
- Dead-letter queue for failed handlers

**Core Interfaces:**
```
interface IEventBus {
    <T extends Event> void publish(T event);
    <T extends Event> void subscribe(Class<T> type, EventHandler<T> handler);
    <T extends Event> void unsubscribe(Class<T> type, EventHandler<T> handler);
}
```

**Problems to Solve:**
1. Design an in-memory `EventBus` with async dispatch using `ExecutorService`.
2. Add retry with backoff for failed handlers — after 3 retries move to dead-letter queue.
3. Add event filtering: handlers subscribe with a predicate, not just a type.

---

### 6.3 Circuit Breaker Pattern

**States:** `CLOSED → OPEN → HALF_OPEN → CLOSED`

**Logic:**
- CLOSED: requests pass through; track failure count
- OPEN: requests fail fast (no actual call made); reset timer starts
- HALF_OPEN: allow one probe request; success → CLOSED, failure → OPEN

**Problems to Solve:**
1. Design `CircuitBreaker<T>` wrapper: `execute(Callable<T> action)` — manages states internally with failure threshold and timeout.
2. Add metrics: track total calls, failures, state transitions.
3. Design a `ServiceProxyFactory` that wraps any service with a circuit breaker transparently.

---

### 6.4 Retry and Backoff Mechanism

**Strategies:**
- Fixed delay retry
- Exponential backoff: `delay = baseDelay * 2^attempt`
- Exponential backoff with jitter (avoid thundering herd)
- Max attempts + deadline enforcement

**Problems to Solve:**
1. Design a `RetryExecutor` with configurable `maxAttempts`, `baseDelayMs`, `backoffMultiplier`, and `retryableExceptions` list.
2. Add jitter to exponential backoff: `delay = random(0, baseDelay * 2^attempt)`.
3. Combine `CircuitBreaker + RetryExecutor` — retry only when circuit is CLOSED.

---

### 6.5 Distributed ID Generator (like Snowflake)

**Design:** Generate globally unique, sortable 64-bit IDs without coordination

**Bit Layout (Twitter Snowflake):**
```
[1 bit sign] [41 bits timestamp ms] [10 bits machine ID] [12 bits sequence]
```

**Problems to Solve:**
1. Implement `SnowflakeIdGenerator` in Java/Python — thread-safe sequence increment within the same millisecond.
2. What happens if the system clock goes backward? Design a safeguard.
3. Design a `UUIDv7` generator (time-ordered UUID) — how does it differ from Snowflake?

---

### 6.6 Mock Interview Problem Bank

#### Tier 1 — Easy (25–30 min)
| Problem | Key Patterns | Watch For |
|---|---|---|
| Design a Vending Machine | State, Singleton | State transitions, coin handling edge cases |
| Design a Notification System | Observer, Strategy, Factory | Channel fanout, retry on failure |
| Design an ATM Machine | State, Facade | Concurrent access to account, PIN retry limit |
| Design a Tic-Tac-Toe Game | Template Method, Iterator | Win detection, board representation |
| Design a URL Shortener (in-memory) | Singleton, Facade | Hash collision handling, redirect logic |

#### Tier 2 — Medium (40–45 min)
| Problem | Key Patterns | Watch For |
|---|---|---|
| Design a Hotel Booking System | Builder, State, Strategy | Double-booking, cancellation refund |
| Design an Online Shopping Cart | Composite, Observer, Strategy | Tax calculation, coupon stacking, inventory |
| Design Snake & Ladder Game | Command, Iterator | Multiple players, undo move |
| Design an Amazon Locker System | State, Factory, Observer | Locker assignment, expiry and auto-release |
| Design a Splitwise Clone | Observer, Strategy | Debt simplification, multi-currency |
| Design a Cricket Scorecard | Template Method, Observer | Over, wicket, innings state machine |

#### Tier 3 — Hard (55–60 min)
| Problem | Key Patterns | Watch For |
|---|---|---|
| Design a Stock Exchange | Mediator, Observer, Command | Order matching engine, partial fills |
| Design a Distributed Task Queue (in-process) | Command, Observer, Pool | Priority, retry, DLQ, concurrency |
| Design a Code Review Tool (like GitHub PRs) | Observer, State, Composite | PR state machine, inline comments, approval rules |
| Design a Real-Time Collaborative Editor | Command, Memento, Observer | Conflict resolution, undo/redo across users |
| Design an API Gateway (in-process) | Chain of Resp., Proxy, Strategy | Auth, rate limit, circuit break, routing |

---

### 6.7 Self-Assessment Rubric

Use this to grade your own or someone else's design (score each 1–5):

| Dimension | What to Evaluate |
|---|---|
| **Requirements Clarity** | Did you ask the right clarifying questions? |
| **Entity Modeling** | Are entities cohesive? Correct attributes? Missing actors? |
| **Class Hierarchy** | Appropriate use of inheritance vs composition? |
| **Pattern Application** | Right patterns chosen? Justified? Not over-engineered? |
| **Interface Design** | Are contracts clean? SRP followed? |
| **State Machines** | Are all states defined? Invalid transitions blocked? |
| **Concurrency** | Thread safety considered where needed? |
| **Extensibility** | Can a new feature be added without rewriting core? |
| **Edge Cases** | Null inputs, empty collections, concurrent access, timeouts |
| **Code Quality** | Readable, named well, no god classes |

**Total: /50** — Target 40+ for senior roles.

---

### Week 6 — Mock Interview Prompt Template
```
Act as a Staff Engineer interviewing me for a FAANG senior SDE role.

Problem: [Design an API Gateway (in-process)]

Conduct a realistic 45-minute interview:
- Minute 0–5: Ask me clarifying questions
- Minute 5–15: I'll define entities and interfaces — challenge any weak spots
- Minute 15–30: I'll write core classes and key methods — ask about design choices
- Minute 30–40: Walk through a request flow end-to-end — look for gaps
- Minute 40–45: Give me one extension problem

After the session:
1. Score me on the 10-dimension rubric (1–5 each)
2. Name the top 3 things I did well
3. Name the top 3 specific improvements with code examples
```

---

## Quick Reference: All Design Patterns

### Creational (5 patterns)
| Pattern | Intent | Developer Example | Interview Problem |
|---|---|---|---|
| Singleton | One instance, global access | LogManager, MetricsRegistry | Thread-safe config loader |
| Factory Method | Defer object creation to subclass | ParserFactory, DBFactory | Cloud provider client factory |
| Abstract Factory | Families of related objects | CloudProviderFactory | Test doubles factory |
| Builder | Step-by-step construction | SQLQueryBuilder, HttpRequest | Kafka config builder |
| Prototype | Clone existing objects | ServerConfigTemplate | Report template clone |

### Structural (7 patterns)
| Pattern | Intent | Developer Example | Interview Problem |
|---|---|---|---|
| Adapter | Bridge incompatible interfaces | Legacy API wrapper | Adapt old DAO to new repo |
| Decorator | Add behavior dynamically | HTTP middleware, I/O streams | Retry + logging + auth wrapper |
| Facade | Simplify complex subsystem | OrderFacade, CI pipeline | Deploy facade |
| Proxy | Control access | CachingProxy, AuthProxy | Logging proxy for any service |
| Composite | Tree part-whole structures | File system, CI task tree | Menu or UI tree |
| Bridge | Decouple abstraction + impl | Exporter × DataSource | Notif channel × formatter |
| Flyweight | Share intrinsic state | CharacterStyle in editor | Game map tile sharing |

### Behavioral (11 patterns)
| Pattern | Intent | Developer Example | Interview Problem |
|---|---|---|---|
| Observer | Notify dependents on change | Build event → Slack/Email | Metrics alert system |
| Strategy | Interchangeable algorithms | Auth strategy, compression | Load balancer routing |
| Command | Encapsulate request as object | DB migration up/down | Text editor undo/redo |
| Iterator | Sequential collection access | PaginatedAPIIterator | Binary tree traversal |
| State | Behavior changes with state | CI job, Order lifecycle | ATM state machine |
| Chain of Resp. | Pass request along chain | HTTP middleware | Expense approval chain |
| Template Method | Fixed skeleton, vary steps | DataPipeline ETL | Report generator |
| Mediator | Centralize communication | EventBus, ChatRoom | Microservice broker |
| Memento | Save and restore state | Editor checkpoint | Game save state |
| Visitor | Add ops without modifying classes | AST visitor, TaxVisitor | Compiler pass |
| Interpreter | Evaluate grammar/expressions | DSL evaluator, SQL parser | Boolean expression parser |

---

## Key Interview Tips

1. **Clarify first** — spend 3–5 min on requirements; wrong assumptions waste time.
2. **Entities before patterns** — get the class model right first, then apply patterns.
3. **Justify every pattern** — say WHY (extensibility, flexibility, decoupling), not just WHAT.
4. **Use enums for states** — `OrderStatus.PLACED` beats magic strings.
5. **Favor composition** — prefer `has-a` over `is-a`; easier to test and extend.
6. **Cover thread safety** — always ask "could this be accessed concurrently?"
7. **Design for extension** — show how to add a new feature without rewriting core.
8. **Name things well** — `PaymentStrategy` beats `IPay`; `UserCreatedEvent` beats `Event1`.
9. **Avoid god classes** — if a class has more than 5 responsibilities, split it.
10. **Show edge cases** — null input, zero quantity, expired TTL, concurrent write, invalid state transition.

---

## Recommended Resources

| Type | Resource |
|---|---|
| Book | "Head First Design Patterns" — Freeman & Freeman |
| Book | "Clean Code" — Robert C. Martin |
| Book | "Designing Data-Intensive Applications" — Martin Kleppmann |
| GitHub | [iluwatar/java-design-patterns](https://github.com/iluwatar/java-design-patterns) |
| GitHub | [prasadgujar/low-level-design-primer](https://github.com/prasadgujar/low-level-design-primer) |
| Practice | Educative — Grokking the LLD Interview |
| Practice | LeetCode — Object-Oriented Design problems |
| YouTube | Gaurav Sen — System Design |
| YouTube | Soumyajit Bhattacharyya — LLD Playlist |
| YouTube | NeetCode — Design problems |

---

*Plan Version: 2.0 | Updated: May 2026 | Covers: SOLID + 23 GoF Patterns + 14 Real Systems + 6 Advanced Topics + 20 Mock Problems*
