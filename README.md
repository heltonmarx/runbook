Interviews Runbook
===

# Go

## Stack and Heap

 - **Stack**: stores local variables within a function frame. Fast, automatic, no GC involved.
 - **Heap**: stores objects that "escape" their function scope. Managed by Go's garbage collector.

The key concept is escape analysis: The Go compiler decides at compile time where each variable lives.

Goes to **stack** when:
 - Variable is only used inside the current function
 - Small, fixed-size arrays/slices
 - No reference leaves the scope

Goes to **heap** when:
 - Function returns a pointer (`return &User{}`). Object must outlive the frame.
 - Assigned to an `interface{}`/`any`. Boxing forces heap
 - Large or dynamically-sized `make([]byte, n)`
 - Closures capturing variables by reference
 - Goroutines capturing outer variables


To inspect what escapes in code:
```bash
go build -gcflags="-m" ./...
```

One Go-specific detail worth noting:
Each goroutine's stack starts small (2-8KB) and grows dynamically - unlike C or Java where stack size is fixed.
This is what allows Go to run thousands of goroutines efficiently.

## M:N Scheduler

Go uses an **M:N threading model**.

M goroutines mapped onto N OS threads, managed by its own runtime scheduler.
This is fundamentally different from Java or Python where each thread maps 1:1 to an OS thread.

The three key actors are:
- **G**: goroutine
- **M**: machine (OS Thread)
- **P**: processor (scheduling context)

### How it works in practice:
`GOMAXPROCS` controls how many *Ps* exist (defaults to number of CPU cores). 
Each P has a **local run queue** of goroutines ready to run, 
and there's also a **global run queue** as a fallback.

Each P must be attached to an M (Os thread) to actually execute goroutines.

**Work Stealing** is the key to efficiency: 

Is P2's local queue is empty, it steals half the goroutines from P1's queue. 
This keeps all threads busy without doing anything.

**Syscall Handling** is where it gets clever. 
When a goroutine blocks on a syscall (file I/O, network, etc.):

 1. The P detaches from M3 (which is now stuck waiting on the OS)
 2. P picks up a free M (or the runtime creates a new one)
 3. P continues running other goroutines while M3 waits
 4. When the syscall returns, M3 tries to reacquire a P. If none is free, the goroutine goes to the global queue and M3 goes to sleep.

This is why can have thousands of goroutines with just a handfull of OS threads.
A goroutine waiting on I/O doesn't hold an OS thread hostage. It simply parks itself and the thread moves on.

**Goroutine stack** starts at 2-8 KB and grows on demand (up to 1GB by default),
unlike OS threads which get a fixed ~1-8 MB stack up front. That's what makes it cheap to spawn 100K
goroutines.

## Garbage Collector

### How GC works when a goroutine is created and ended ?

When `go func()` is called, the runtime allocates a `G` struct on the *heap* to represent the goroutine, 
and gives it a small contiguous stack (2-8 KB, also on the *heap*).
The `G` is placed in the local run queue of the current `P`.
This is cheap - no OS thread is created.

When running, locals stay on the stack (no GC involvement), but any object that escapes
(returned pointer, captured by closure, assigned to interface, etc.) goes to the *heap* and 
becomes the GC's responsibility, During a GC mark phase, a **write barrier** os activated, any new pointer
writes are intercepted so the GC doesn't miss live objects being created mid-scan.

The GC itself runs mostly concurrently with goroutines using a **tri-color mark-and-sweep** algorithm.
Every heap object is conceptually one of three colors:
 * White: not yet visited, will be freed if still white at sweep time.
 * Grey: reachable but its children haven't been scanned yet
 * Black: fully scanned, definitely alive.

The GC starts from "roots" (globals, stack frames of all goroutines) and progressively turns
everyting reachable from white -> grey -> black. Anything left white at the end is garbage and gets swept.

There are only two tiny **stop-the-world (STW)** pauses: 
at mark start (to snapshot roots) and
at mark termination (to finalize). 
Both are typically under **1ms**.
The bulk of marking happens concurrently. 
Notably, if a goroutine is allocating very fast, it may be asked to do **GC assist work** helping mark
objects as part of its own execution to keep up with allocation pressure.

When the goroutine finishes, the runtime does three things: 
1. the stack is returned to a pool (not freed - reused for the next goroutine), 
2. the `G` struct itself is pooled too (same reason),
3. and heap objects the goroutine was referencing are not freed immediately.

They're freed in the next GC cycle once the mark phase confirms no other goroutine holds a reference to them.

One subtle point: **the GC scans goroutine stacks** as roots during the mark phase. 
This means all goroutines are briefly paused at safe points (function call boundaries) so the GC
can snapshot their stack, this is called a **stack scan**. 

Goroutines that are blocked (on channels, syscalls, `select`) are easier to scan since they're already parked.

## Channels

A channel is a **runtime struct** (`hchan`) with a circular buffer, two queues of waiting goroutines, and a mutex.

The `hchan` struct (defined in `runtime/chan.go`) holds everything:
* A pointer to the circular buffer
* `sendx`/`recvx` indices tracking where the next write/read goes.
* `qcount` (items currently in the buffer)
* `cap` (buffer capacity)
* two linked lists of waiting goroutines (`sendq` and `recvq`)
* A `mutex` protecting the whole thing.

### The four send/receive paths:

When do you `ch <- v`, the runtime acquires the mutex and checks in order: 
* Is there a goroutine waiting in `recvq` ? 
* If yes, it copies the value directly into that goroutine's stack memory calls `goready` to make it runnable, the buffer is never touched, this is the fastest path.
* If no receiver is waiting but the buffer has space, the value is copied into `buf[sendx]` and the sender continues.
* If the buffer is full, the runtime wraps the goroutine and its value pointer into a `sudog` struct, appends it to `sendq`, and calls `gopark`. The goroutine is removed from the run queue and parks until receiver wakes it.

Receiving with `v := <- ch` is the mirror image: check `sendq` first (dequeue and copy directly), then check the buffer, then park into `recvq`.

The `sudog` struct is key detail: It's a wrapper that holds a pointer to the blocked `G`, a pointer to the value being sent/received, and linkage for the queue. It's pooled by the runtime to avoid allocations on every channel operation.

Direct stack-to-stack copy (when sendq/recvq has as waiter) is a subtle optimization:
The sender copies directly into the receiver's stack frame, skipping the buffer entirely.
This means for an unbuffered channel, `make (chan int)`, there is literally never any buffer involved -- every send blocks until a receiver is ready, and the value goes straight across.

`close(ch)` sets a `closed` flag, then wakes up all goroutines in `recvq` (they get the zero value and `false` for the `ok` flag), and panics any goroutines in `sendq` since sending on a closed channel is a runtime panic.

`select` whit multiple channels works by locking all involved channels simultaneously (in a consistent address order to avoid deadlock), scanning all cases, and if none are ready, enqueuing a `sudog` into every channel's queue at once.
When any one fires, it dequeues itself from all the others.

Channel operations are **not lock-free** -- there's a real mutex inside. The performance advantage over raw mutexes comes from the scheduler integration: a blocked goroutine doesn't burn a thread, it just parks.

What happens without goroutine control:
### 1. Goroutine Leak
```go
func leak() {
    ch := make(chan int)
    go func() {
        val := <- ch // blocks forever
        fmt.Println(val)
    }()
    // function returns, but the goroutine stays stuck in memory.
}

func main() {
    for i := 0; i < 100000; i++ {
        leak() // 100k stuck goroutines
    }
}
```
Each goroutine starts with **~8KB of stack** and grows as needed. With 100K idle goroutines, you easily consume **gigabytes of RAM** without doing anything useful.

### 2. Race Condition
```go
var counter int

func main() {
    for i := 0; i < 1000; i++ {
        go func() { counter++ }() // read + increment + write IS NOT atomic
    }
    time.Sleep(time.Second)
    fmt.Println(counter) // unpredictable result
    // Solution: sync.Mutex, sync.RWMutex, or channels
}
```

### 3. Deadlock
```go
func main() {
    ch := make(chan int)

    go func() { ch <- 1 }()
    go func() { ch <- 2 }()
    // nobody reads the channel - Go detects and panics:
    // "all goroutines are asleep - deadlock!"
}
```

### 4. Starvation
```go
var mu sync.Mutex

func greedy() {
    for {
        mu.Lock()
        // long work holding the mutex
        time.Sleep(1 * time.Second)
        mu.Unlock()
    }
}

for starved() {
    mu.Lock() // never gets the lock
    defer mu.Unlock()
    fmt.Printf("never get here")
}

func main() {
    go greedy()
    go greedy()
    go starved()
    time.Sleep(5 * time.Second)
}
```

### 5. Goroutine explosion + OOM
```go
func processRequest(r Request) {
    // no concurrency limit
    for _, item := range r.Items {
        go process(item) // 1 request with 10K items = 10K goroutines
    }
}
```
In production, a single request with a large payload can bring down the entire service.

## How to control correctly

### 1. Worker Pool with semaphore
```go
func main() {
    sem := make(chan struct {}, 10) // maximum 10 simultaneous goroutines
    var wg sync.WaitGroup

    for i := range 1000 {
        wg.Add(1)
        sem <- struct{}{} // blocks if 10 goroutines are already running

        go func(id int) {
            defer wg.Done()
            defer func() { <- sem } () // releases slot when done

            process(id)
        }(i)
    }
    wg.Wait()
}
```
### 2. `errgroup` with limit

```go
import "golang.org/x/sync/errgroup"

func main() {
    g, ctx := errgroup.WithContext(context.Background())
    g.SetLimit(10) // max 10 goroutines

    for i := range 1000 {
        id := i
        g.Go(func() error {
            return process(ctx, id)
        })
    }
    if err := g.Wait(); err != nil {
        log.Fatal(err)
    }
}
```

### 3. Detect leaks in tests
```go
import "go.uber.org/goleak"

func TestMyFunction(t *testing.T) {
    defer goleak.VerifyNone(t) // fails if goroutines remain open

    myFunctionThatShouldCleanUp()
}
```

### Risk summary

| Problem           | Symptom                         | Solution                          |
| ----------------- | ------------------------------- | --------------------------------- |
| Goroutine leak    | Memory grows endlessly          | `goleak`, contexts with cancel    |
| Race condition    | Unpredictable result            | `sync.Mutex`, `atomic`, channels  |
| Deadlock          | `panic: all goroutines asleep`  | Lock review, timeouts             |
| Starvation        | Goroutine never executes        | Fairness in lock design           |
| OOM/explosion     | Service crashes under load      | Worker pool, semaphore, `errgroup`|


## Behavior of closed channels in Go

| Operation                   | Result                                        |
| --------------------------- | --------------------------------------------- |
| Read from a closed channel  | Returns zero value + `false` immediately      |
| Write to a closed channel   | `panic` at runtime                            |
| Close an already closed channel | `panic` at runtime                        |


## Behavior of an uninitialized (`nil`) channel in Go

| Operation                  | Result                                                      |
| -------------------------- | ----------------------------------------------------------- |
| Read from a `nil` channel  | Blocks *forever* (`panic: all goroutines are asleep`)       |
| Write to a `nil` channel   | Blocks *forever* (`panic: all goroutines are asleep`)       |
| Close a `nil` channel      | `panic` at runtime                                          |


A `nil` channel has a valid intentional use: disabling a `case` inside a `select`:
```go
func merge(ctx context.Context, a, b <- chan int) <- chan int {
    out := make(chan int)
    go func() {
        defer close(out)
        for a != nil || b != nil {
            select {
                case v, ok := <- a:
                    if !ok { a = nil; continue} // disables this case
                    out <- v
                case v, ok := <- b
                    if !ok { b = nil; continue} // disables this case
                    out <- v
                case <- ctx.Done():
                    return
            }
        }
    }()
    return out
}
```

The difference between `sync.Mutex` and `sync.RWMutex`:
- `sync.Mutex`: Simple Mutual Exclusion, allows only `one goroutine at a time`, for either reading or writing.
- `sync.RWMutex`: Read/Write Mutex, allows multiple simultaneous reads, but exclusive writes.

| Situation              | `Mutex`  | `RWMutex`           |
| ---------------------- | -------- | ------------------- |
| Simultaneous reads     | blocks   | allows              |
| Simultaneous writes    | blocks   | blocks              |
| Overhead               | lower    | slightly higher     |

### `panic` and `recover`

* `panic` interrupts the normal execution flow of the current goroutine, unwinding the stack until the program terminates, unless a `recover` intercepts it.
* `recover` captures an ongoing panic and allows the program to continue. Only works inside a `defer`.

```go
func safeCall() {
    defer func() {
        if r := recover(); r != nil {
            fmt.Println("panic captured: ", r)
        }
    }()
    panic("something went wrong")
}
```
### When to use `recover`?

1. HTTP Server — prevent a panic in a handler from bringing down the entire process.
```go
func recoveryMiddleware(next http.Handler) http.Handler{
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        defer func() {
            if r := recover(); r != nil {
                log.Println("panic recovered: ", r)
                http.Error(w, "internal server error", http.StatusInternalServerError)
            }
        }()
        next.ServeHTTP(w, r)
    })
}
```

2. Worker pools — prevent a panicking job from killing the entire worker.
```go
func safeWorker(job Job) (err error) {
    defer func() {
        if r := recover(); r != nil {
            return fmt.Errorf("worker panic: %v", r)
       }
    }()
    return job.Process()
}
```

# SOLID
SOLID is an acronym for five object-oriented software design principles that make systems more maintainable,
scalable, and testable.
* S: Single Responsibility Principle (SRP)
* O: Open/Closed Principle (OCP)
* L: Liskov Substitution Principle (LSP)
* I: Interface Segregation Principle (ISP)
* D: Dependency Inversion Principle (DIP)

## S - Single Responsibility Principle (SRP)
A struct/module should have only one reason to change.
Each component should be responsible for a single part of the functionality.

### SRP Violation
```go
type UserService struct { db *sql.DB }

func (s *UserService) CreateUser(name, email string) error {
    if email == "" { return errors.New("invalid email") }
    s.db.Exec("INSERT INTO users...", name, email)
    smt.SendEmail("smtp.example.com", nil, "noreply@", []string{email}, []byte("welcome")),
    return nil
}
```

### Applying SRP
```go
type UserRepository struct { db *sql.DB }
func (r *UserRepository) Save(u User) error { /* persists only */ }

type EmailService struct { host string }
func (e *EmailService) SendWelcome(to string) error { /* sends email only */ }

type UserService struct {
    repo *UserRepository
    mailer *EmailService
}

func (s *UserService) CreateUser(name, email string) error {
    if err := s.repo.Save(User{Name: name, Email: email}); err != nil { return err }
    return s.mailer.SendWelcome(email)
}
```

## O - Open/Closed Principle (OCP)
Software entities should be open for extension but closed for modification.

### Example with interfaces
```go
type Notifier interface {
    Notify(msg string) error
}

type SlackNotifier struct { webhook string }
type (n SlackNotifier) Notify(msg string) error { /* POST to slack */ return nil }

type EmailNotifier struct { smtp string }
func (n EmailNotifier) Notify(msg string) error { /* sends email */ return nil }

type AlertService struct { notifiers []Notifier }
func(s *AlertService) Alert( msg string ) error {
    for _, n := range s.notifiers {
        if err := n.Notify(msg); err != nil { return err }
    }
}
```

## L - Liskov Substitution Principle (LSP)
Subtypes must be substitutable for their base types without altering the expected behavior of the program.
```go
type Storage interface {
    Save(key string, val []byte) error
    Load(key string) ([]byte, error)
}

type RedisStorage struct { client *redis.Client }
func (s RedisStorage) Save(key string, val []byte) error {...}
func (s RedisStorage) Load(key string) ([]byte, error) {...}


type S3Storage struct { client *redis.Client }
func (s S3Storage) Save(key string, val []byte) error {...}
func (s S3Storage) Load(key string) ([]byte, error) {...}

type CacheService struct { store Storage }
func (c *CacheService) Set (key string, val []byte) error { return s.store.Save(k, v) }
```

## I - Interface Segregation Principle (ISP)
No client should be forced to depend on interfaces it does not use.
Prefer small and specific interfaces.

### "Fat" interface (violation)
```go
type MessageBroker interface {
    Publish(topic string, msg []byte) error
    Subscribe(topic string) (<-chan []byte, error)
    CreateTopic(name string) error
    DeleteTopic(name string) error
    ListTopics() ([]string, error)
}
```

### Segregated interfaces
```go
type Publisher interface {
    Publish(topic string, msg []byte) error
}

type Subscriber interface {
    Subscribe(topic string) (<-chan []byte, error)
}

type TopicManager interface {
    CreateTopic(name string) error
    DeleteTopic(name string) error
}

// Notification service only needs to publish
type NotificationService struct { pub Publisher}

// Worker only needs to consume messages
type ConsumerWorker struct { sub Subscriber }
```

## D - Dependency Inversion Principle (DIP)
High-level modules should not depend on low-level modules. Both should depend on abstractions.

### Dependency injection in Go
```go
type OrderRepository interface {
    FindByID(id string) (*Order, error)
    Save(o *Order) error
}

type OrderService struct {
    repo OrderRepository
}

func (s *OrderService) ProcessOrder(id string) error {
    order, err := s.repo.FindByID(id)
    if err != nil { return err }
    order.Status = "processed"
    return s.repo.Save(order)
}

type PostgresOrderRepo struct { db *sql.DB }
func (p *PostgresOrderRepo) FindByID(id string) (*Order, error) {...}
func (p *PostgresOrderRepo) Save(o *Order) error{ ... }
```

# CAP Theorem
The CAP Theorem (Brewer, 2000) states that a distributed system can only simultaneously guarantee two of three properties:
*Consistency*, *Availability*, and *Partition Tolerance*.

| Combination | Consistency | Availability    | Partition Tolerance |
| ----------- | ----------- | --------------- | ------------------- |
| CP          | Yes         | Not guaranteed  | Yes                 |
| AP          | Eventual    | Yes             | Yes                 |
| CA          | Yes         | Yes             | No                  |

In practice, network partitions are unavoidable in distributed systems. The real choice is between CP and AP; CA is theoretical.

## C - Consistency
Every node returns the most recent data. After a write, all subsequent reads reflect that value.

### Example: Consistent read with etcd (CP)
```go
func readConsistent(
    ctx context.Context,
    client *clientv3.Client,
    key string)
    (string, error) {
    ctx, cancel := context.WithTimeout(ctx, 3 * time.Second)
    defer cancel()

    resp, err := client.Get(ctx, key, clientv3.WithSerializable())
    switch {
        case err != nil:
            return "", err
        case len(resp.Kvs) == 0:
            return "", nil
    }
    return string(resp.Kvs[0].Value), nil
}
```

## A - Availability
Every request receives a response (not necessarily with the most recent data). The system remains operational even with partial failures.

### Example: Eventually consistent read with DynamoDB (AP)
```go
func readEventuallyConsistent(
    ctx context.Context,
    svc *dynamodb.DynamoDB,
    table string,
    key string,
) (string, error) {
    input := &dynamodb.GetItemInput{
        TableName: aws.String(table),
        Key: map[string]*dynamodb.AttributeValue{
            "id": { S: aws.String(key) },
        }
        // ConsistentRead: false (default) = eventual consistency
        // Higher availability, lower latency, lower cost
        ConsistentRead: aws.Bool(false),
    }
    result, err := svc.GetItem(input)
    if err != nil { return "", err}
    return aws.StringValue(result.Item("value").S), nil
}
```

## P - Partition Tolerance
The system continues to function even when messages between nodes are lost or delayed. In real networks, this is practically mandatory.

## Real-world examples of systems and their CAP trade-offs

| System        | CAP    | Use in your stack      | When to use                       |
| ------------- | ------ | ---------------------- | --------------------------------- |
| etcd/Zookeeper| CP     | kubernetes, config     | Coordination, leader election     |
| PostgreSQL    | CP     | Transactional data     | Strong consistency, ACID          |
| MongoDB       | CP/AP* | Flexible               | Documents, variable schema        |
| DynamoDB      | AP     | AWS services, Ring     | Global high availability          |
| Kafka         | CP     | Messaging              | Replicated event log              |
| Cassandra     | AP     | Write-heavy scale      | Time series, IoT                  |
| Redis         | CP     | Cache                  | Cache, sessions, rate limiting    |

## Applying CAP in a real service
Example of a service that consciously chooses between CP and AP per endpoint:
```go
type OrderService struct {
    db OrderRepository   // CP - Postgres
    cache OrderRepository // AP - Redis eventual
}

// Payment requires strong consistency (CP)
func (s *OrderService) ProcessPayment(orderID string) error {
    order, err := s.pgRepo.FindByID(orderID) // consistent read
    if err != nil { return err }
    order.Status = "paid"
    return s.pgRepo.Save(order) // durable write
}

// Order listing accepts slightly stale data (AP)
func (s *OrderService) ListOrders(userID string) ([]*Order, error) {
    orders, err := s.cacheRepo.FindByUser(userID)
    if err != nil {
        // fallback to CP if cache fails
        return s.pgRepo.FindByUser(userID)
    }
    return orders, nil
}
```

## CQRS Pattern
Separating Command (CP: consistent write) from Query (AP: read with eventual cache) is an elegant way to consciously apply CAP in Go services.

# Connecting SOLID and CAP

SOLID principles and the CAP Theorem complement each other in microservices architectures in Go:
* SRP + CAP: Small, cohesive services allow more surgical CAP choices — a payment service can be CP while the catalog is AP.
* OCP + CAP: Go interfaces allow swapping CP implementations for AP ones (e.g. Redis → etcd) without changing business code.
* DIP + CAP: Injecting repositories as abstractions makes it easy to change the CAP trade-off by environment (test: in-memory, prod: Postgres CP).
* ISP + CAP: Segregated interfaces (Reader/Writer) allow implementing AP reads and CP writes independently.

# REST

| Method   | Operation           | Idempotency | Safe |
| -------- | ------------------- | ----------- | ---- |
| `GET`    | read                | yes         | yes  |
| `POST`   | create              | no          | no   |
| `PUT`    | full replacement    | yes         | no   |
| `PATCH`  | partial update      | no          | no   |
| `DELETE` | removal             | yes         | no   |

## Idempotency
Multiple identical calls produce the same result. **Safe** does not alter state on the server.

1. Idempotency Keys
The client generates a unique key per operation and sends it in the header. The server stores the result of the first execution and reuses it for duplicate requests.
Where to store keys:
* Redis: ideal — native TTL, low latency, atomic operations.
* PostgreSQL: when strong consistency is required.
* DynamoDB: native TTL, scales well.

2. Transactions with Explicit State Control
Instead of detecting duplicates, model the operation with states that make reprocessing safe.

3. Conditional Updates — Optimistic Locking
Uses version or ETag to ensure that an update only occurs if the expected state is still valid. Prevents duplicate updates and race conditions simultaneously.

4. Idempotency in Message Queues / Event-Driven
Consumers must be idempotent because messages can be delivered more than once (Kafka, SQS, Pub/Sub guarantee at-least-once by default).

### Idempotency types
| Strategy                    | Best for                       | Complexity  | Guarantee         |
| --------------------------- | ------------------------------ | ----------- | ----------------- |
| Idempotency Keys            | HTTP APIs, payments            | medium      | strong            |
| Unique Constraints          | creation operations            | low         | strong (DB)       |
| Optimistic Locking          | concurrent updates             | medium      | strong            |
| Deduplication in consumers  | message queues                 | low/medium  | eventual          |
| Outbox pattern              | guaranteed event delivery      | high        | exactly once      |

## HTTP Status Codes
 - 2xx Success
 - 4xx Client error
 - 5xx Server error

## URI Design
Best practices:

| Method | Path                | Description                              |
| ------ | ------------------- | ---------------------------------------- |
| GET    | /v1/users           | list users                               |
| GET    | /v1/users/42        | fetch user by ID                         |
| POST   | /v1/users           | create user                              |
| PUT    | /v1/users/42        | full user update                         |
| PATCH  | /v1/users/42        | update specific fields                   |
| DELETE | /v1/users/42        | remove user                              |
| GET    | /v1/users/42/orders | nested resource (orders for user 42)     |

### General rules
* Clients should generate UUIDs v4 for idempotency keys, never the server
* TTL on keys: store long enough to cover retry windows
* Idempotency is not a transaction: it prevents duplication, it does not guarantee atomicity
* Document which endpoints are idempotent: it is a contract with the API client
* Always test the retry path: simulate timeouts and partial failures in integration tests.

## Pagination

### Offset-based — simple, but inefficient for large datasets
`GET /users?page=2&limit=20`

### Cursor-based — more efficient, ideal for large volumes
`GET /users?cursor=eyJpZCI6MTAwfQ&limit=20`

Standard response:

```json
{
    "data": [...],
    "pagination": {
        "total": 1000,
        "page": 2,
        "limit": 20,
        "next_cursor": "eyJpZCI6MTIwfQ"
    }
}
```

# DRY

DRY is a software development principle coined by Andrew Hunt and David Thomas in the book **The Pragmatic Programmer**.
The core idea is not just "don't copy code" — it is deeper: every business decision, rule, or logic must exist in a single place.
When you need to change something, you change it in one place only.

DRY is not about eliminating all visual code repetition — it is about ensuring that every rule, decision, or piece of system knowledge lives in a single authoritative place. When well applied, it reduces bugs, simplifies maintenance, and makes changes safer. When poorly applied, it creates artificial coupling between things that should be independent.

# Postgres

## Excessive reads and high connection count

### Why are too many connections a problem?
Each connection in PostgreSQL is a **separate OS process**, not a thread — a process.
Each one consumes between 5–10MB of memory just to exist.
With 500 open connections, you already have 2.5–5GB consumed before any query runs.

Additionally, PostgreSQL has a `max_connections` limit (default 100). When this limit is reached, new connections immediately receive an error:
```
FATAL: sorry, too many clients already
```

#### Connection Pooling with PgBouncer
The primary solution for a high number of connections. PgBouncer sits between the application and the database — the application opens hundreds of connections to PgBouncer, but it maintains only a small pool of real connections to PostgreSQL.
* Application (500 connections) → PgBouncer → PostgreSQL (20 real connections)

#### Operation Modes

| Mode          | Behavior                               | Ideal for              |
| ------------- | -------------------------------------- | ---------------------- |
| `session`     | dedicated connection per client session| maximum compatibility  |
| `transaction` | connection released after each transaction | most cases         |
| `statement`   | connection released after each statement | rarely used          |

#### Read Replicas to distribute reads
For excessive reads, the most effective solution is to create read replicas and route read queries to them.

#### Read query optimization
Before scaling horizontally, ensure queries are efficient.

```sql
-- Identify slow queries
SELECT query,
       calls,
       mean_exec_time,
       total_exec_time,
       rows
FROM pg_stat_statements
ORDER BY mean_exec_time DESC
LIMIT 20;

-- Analyze the execution plan of a suspicious query
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT)
SELECT * FROM orders
WHERE user_id = 42 AND status = 'pending'
ORDER BY created_at DESC;

-- Warning signs in EXPLAIN:
-- Seq Scan on large table    → missing index
-- Nested Loop with many rows → inefficient join
-- Buffers: low hit           → cache miss, lots of I/O
```

#### Indexes for frequently read queries:

```sql
-- Simple index
CREATE INDEX CONCURRENTLY idx_orders_user_id ON orders(user_id);

-- Composite index (respect the column order in the query)
CREATE INDEX CONCURRENTLY idx_orders_user_status
ON orders(user_id, status)
WHERE status = 'pending';  -- partial index — smaller, faster

-- Index for sorting
CREATE INDEX CONCURRENTLY idx_orders_created_at
ON orders(created_at DESC);
```

`CONCURRENTLY` is essential in production — it creates the index without blocking reads and writes on the table.

#### Application cache for repetitive reads
Queries that return the same data frequently should not reach the database at all.

## Excessive writes impacting reads

### Why do writes block reads?
PostgreSQL uses MVCC (Multi-Version Concurrency Control) — normal reads do not block writes and vice versa. But there are situations where intense writes cause serious problems for reads:
* Lock contention: `UPDATE`/`DELETE` acquire row-level locks that block other writes and may block reads depending on the isolation level.
* Table bloat: MVCC creates old versions of rows (dead tuples) that VACUUM needs to clean up; excess dead tuples make Seq Scans much slower.
* WAL pressure: intense writes generate a lot of WAL, impacting disk I/O shared with reads.
* Checkpoint storms: PostgreSQL forces periodic flushes to disk that degrade overall performance.

#### Separating reads and writes with CQRS
The cleanest separation — reads and writes have completely different models and routes.
* Write path: API → command → Primary DB (optimized writes)
* Read path: API → query → Read Replica (optimized reads, different indexes)

#### Asynchronous writes with buffer (Kafka/Queue)
For write spikes, buffer the operations in a queue and write to the database in controlled batches.
* API → Kafka → Consumer → Postgres (batch writes, controlled rate)

**Real gain**: 1000 individual `INSERT`s in separate transactions vs 1 `INSERT` of 1000 rows in one transaction — the batch version can be 10–50x faster.

#### Table partitioning
For tables with extremely high write volume (logs, events, metrics), partitioning distributes data and reduces contention.

```sql
-- Table partitioned by date range
CREATE TABLE events (
    id         BIGSERIAL,
    user_id    INT NOT NULL,
    type       VARCHAR(50),
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
) PARTITION BY RANGE (created_at);

-- Monthly partitions
CREATE TABLE events_2026_01 PARTITION OF events
FOR VALUES FROM ('2026-01-01') TO ('2026-02-01');

CREATE TABLE events_2026_02 PARTITION OF events
FOR VALUES FROM ('2026-02-01') TO ('2026-03-01');

-- Each partition has its own indexes
CREATE INDEX ON events_2026_01(user_id, created_at);
CREATE INDEX ON events_2026_02(user_id, created_at);

-- Dropping old data becomes an instant DROP TABLE (no VACUUM needed)
DROP TABLE events_2025_01;  -- much faster than DELETE
```

#### Tuning VACUUM and autovacuum
With many writes, dead tuples accumulate quickly. The default autovacuum may not be able to keep up.

```sql
-- More aggressive configuration for high-write tables
ALTER TABLE events SET (
    autovacuum_vacuum_scale_factor = 0.01,   -- vacuums when 1% of rows are dead (default: 20%)
    autovacuum_vacuum_cost_delay = 2,         -- less pause between operations (default: 20ms)
    autovacuum_vacuum_insert_scale_factor = 0.05
);

-- Manual VACUUM on a critical table (without blocking reads/writes)
VACUUM (ANALYZE, VERBOSE) events;

-- View autovacuum status in real time
SELECT schemaname, tablename,
       last_autovacuum,
       last_autoanalyze,
       autovacuum_count
FROM pg_stat_user_tables
WHERE tablename = 'events';
```

#### Optimizing indexes to avoid penalizing writes
Every index on a table has a cost on every `INSERT`, `UPDATE`, and `DELETE`. High-write tables should not have unnecessary indexes.

```sql
-- Find indexes that are never used
SELECT schemaname, tablename, indexname, idx_scan
FROM pg_stat_user_indexes
WHERE idx_scan = 0
  AND indexname NOT LIKE '%pkey%'  -- don't remove PKs
ORDER BY schemaname, tablename;

-- Remove unused indexes (with care)
DROP INDEX CONCURRENTLY idx_events_unused;

-- HOT updates — when an UPDATE doesn't change indexed columns,
-- Postgres can update in-place without moving the row or touching indexes.
-- A lower fillfactor leaves space on the page for HOT updates:
ALTER TABLE events SET (fillfactor = 70);
-- 70% fill per page, 30% space for in-place updates
```

### Overview of solutions:

| Problem              | Solution             | Impact                            |
| -------------------- | -------------------- | --------------------------------- |
| Too many connections | PgBouncer + pgxpool  | High — resolves immediately       |
| Intense reads        | Read replicas        | High — horizontal scaling         |
| Slow queries         | EXPLAIN + indexes    | High — eliminates root cause      |
| Repetitive data      | Cache (Redis)        | High — removes load from DB       |
| Write spikes         | Kafka + batch writes | High — decouples and smooths load |
| Write locks          | CQRS + replicas      | High — separates the paths        |
| Table bloat          | Autovacuum tuning    | Medium — prevents degradation     |
| Growing volume       | Partitioning         | Medium — scales over time         |
| Too many indexes     | Index audit          | Medium — reduces write overhead   |

# Redis

## Redis Functions

### 1. Cache
The most common use — Redis stores query results, API responses,
or any data that is expensive to compute, serving subsequent requests directly from memory.

#### Cache strategies

* Cache-Aside (Lazy Loading): read from cache; if not found, fetch from the database and update the cache.
* Write-Through: write to cache and database at the same time.
* Cache Invalidation with Tags (Surrogate keys): associate multiple keys to a tag.

### 2. Pub/Sub
Redis works as a lightweight message broker; publishers send messages to channels
and subscribers receive them in real time.

Important limitation: Pub/Sub in Redis is *fire-and-forget* — if a subscriber is
offline when the message arrives, it is lost.
There is no persistence.
For messages that cannot be lost, use *Streams*.

### 3. Streams — Persistent Event Log
Streams are the Kafka of Redis — an append-only event log with consumer groups,
acknowledgment, and message replay.

```
Stream "orders"
 | -> id: 17100001-0 { event: "created", order_id: 42 }
 | -> id: 17100002-0 { event: "paid", order_id: 42}
 | -> id: 17100003-0 { event: "shipped", order_id: 42}
 | -> id: 17100004-0 { event: "created", order_id: 42}
```

#### Producer
```go
type Producer struct {
    client *redis.Client
}

func (c *Producer) Produce(ctx context.Context, event OrderEvent) error {
    args := &redis.XAddArgs{
        Stream: "orders",
        MaxLen: 10000,
        Aprox: true,
        Values: map[string]any{
            "event": event.Type,
            "order_id": event.OrderID,
            "payload": marshal(event),
        }
    }
    return c.client.XAdd(ctx, args).Err()
}
```

#### Consumer
```go
type Consumer struct {
    client *redis.Client
}

func (c *Consumer) Subscribe(ctx context.Context) error {
    c.XGroupCreateMkStream(ctx, "orders", "processors", "$")
    for {
        msgs, _ := rdb.XReadGroup(ctx, &redis.XReadGroupArgs{
            Group:    "processors",
            Consumer: "worker-1",
            Streams:  []string{"orders", ">"},  // ">" = new messages only
            Count:    10,
            Block:    5 * time.Second,
        }).Result()

        for _, stream := range msgs {
            for _, msg := range stream.Messages {
                processEvent(msg)
                // confirms processing — removes from pending list
                rdb.XAck(ctx, "orders", "processors", msg.ID)
            }
        }
    }
}
```

### 4. Distributed Locks
Redis is widely used to implement distributed locks, ensuring that only one process executes a
critical section at a time in multi-server systems.

#### Implementation with SET NX (simple):
```go
type Locker struct {
    client *redis.Client
}

func (c *Locker) Lock(ctx context.Context, key string, ttl time.Duration) (bool, error) {
    return c.client.SetNX(ctx, "lock:" + key, "locked", ttl).Result()
}

func (c *Locker) Unlock(ctx context.Context, key string) error {
    return c.client.Del(ctx, "lock:" + key).Err()
}
```

#### Redlock: distributed lock with multiple instances:
For critical environments, Redlock uses N independent instances — the lock is only considered acquired if the majority confirm.

Library: [rsync](github.com/go-redsync/redsync)

```go
rs := redsync.New(pools....)
mutex := rs.NewMutex(key, redsync.WithExpiry(30 * time.Second), redsync.WithTries(3))
```

### 5. Rate Limiting
Redis is ideal for distributed rate limiting — shared request counting across multiple application instances.

#### Fixed Window
```go
type RateLimiter struct {
    client *redis.Client
}

func (r *RateLimiter) IsRateLimited(ctx context.Context, userID String, limit int) (bool, error) {
    key := fmt.Sprintf("rate:%s:%d", userID, time.Now().Unix()/60) // 1-minute window
    count, err := c.client.Incr(ctx, key).Result()
    if err != nil {
        return false, err
    }
    if count == 1 {
        rdb.Expire(ctx, key, 2 * time.Minute) // safety TTL
    }
    return count > int64(limit), nil
}
```

#### Sliding Window with Sorted Set
```go
func (r *RateLimiter) slidingWindowRateLimit(ctx context.Context, key string, limit int, window time.Duration) (bool, error) {
    now := time.Now()
    windowStart := now.Add(-window).UnixMilli()

    pipe := c.client.Pipeline()
    // remove entries outside the window
    pipe.ZRemRangeByScore(ctx, key, "0", strconv.FormatInt(windowStart, 10))
    // add the current request
    pipe.ZAdd(ctx, key, redis.Z{Score: float64(now.UnixMilli()), Member: now.UnixNano()})
    // count requests in the window
    countCmd := pipe.ZCard(ctx, key)
    pipe.Expire(ctx, key, window)
    pipe.Exec(ctx)

    return countCmd.Val() > int64(limit), nil
}
```

### 6. Session Store
Redis is the standard backend for user sessions in stateless applications.
The session lives in Redis, and any instance of the application can access it.

### 7. Leaderboards and Counters with Sorted Sets
Sorted Sets are perfect for real-time rankings — the score is updated atomically and the ordering is maintained automatically.

### 8. Job Queues
Redis can work as a simple job queue using Lists or Sorted Sets (for priority queues).

### 9. Geolocation
Redis has native commands for storing and querying geographic coordinates using Sorted Sets internally with geohash.

### 10. Bloom Filter and HyperLogLog — Probabilistic Structures
* HyperLogLog: count unique elements with fixed memory.
* Bloom Filter: check existence without false negatives.

## General Summary

| Function                  | Type                |
| ------------------------- | ------------------- |
| Cache                     | String              |
| Pub/Sub                   | Pub/Sub             |
| Event log                 | Streams             |
| Distributed lock          | String (SET NX)     |
| Rate Limiting             | String / Sorted Set |
| Session store             | String              |
| Leaderboard               | Sorted Set          |
| Job Queue                 | List / Sorted Set   |
| Geolocation               | Geo (Sorted Set)    |
| Unique counting           | HyperLogLog         |
| Existence check           | Bloom Filter        |

# Kafka

How to define and redefine the number of consumers in Kafka during consumer lag?

Number of active consumers <= number of topic partitions.
If consumers > partitions, the excess ones remain idle.
To reduce lag, you normally need more **partitions** (not just more consumers)
and then scale consumers up to that limit.

1. Diagnose the lag first (dashboards).
 LAG per partition, `CURRENT-OFFSET` vs `LOG-END-OFFSET`. If some partitions have high lag and others zero:
  - Distribution problem / hot partition.
  - Not just a consumer quantity issue.

2. Increase partitions (if necessary):
 Increasing partitions changes key hashing — may break key-based ordering for existing messages.

KEDA monitors lag and automatically adjusts replicas; each new pod joins the same consumer group
and rebalance distributes partitions.

**Summary:**
- High lag, few partitions: increase partitions + consumers.
- High lag, partitions > consumers: add consumers up to the number of partitions.
- High lag, partitions = consumers: optimize processing (batch, internal parallelism per consumer) or increase partitions.
- Uneven lag distribution: review key/hashing, possible hot partition.
- Automatic scaling: KEDA + lag threshold.

# Cache

## Write-Behind
Writes only to the cache synchronously, and persistence to the database happens later, asynchronously (via Kafka).
Faster for the caller, but risk of data loss if the cache goes down before flushing.

## Read-through (cache-aside)
The most common pattern with Redis.
The application checks the cache, and on a miss, fetches from the database and populates the cache.
Write-through ensures synchronous consistency between cache and database on every write, usually via key invalidation instead of direct update.

The choice depends on the criticality of the data:
 - For real-time game state, write-behind is acceptable;
 - For currency/inventory transactions, write-through or outbox-pattern is preferable to guarantee durability.

# Outbox Pattern
The Transactional Outbox Pattern is an alternative to the Dual-Write Pattern, designed to ensure
consistency between database writes and event publishing in distributed systems that use SQL databases.

In the context of CQRS, it is critically important that state change events are correctly propagated to query models.
Guaranteeing that a state change in the database and the publication of a corresponding event occur atomically means
that both must complete successfully or neither should execute.

The Outbox Pattern solves these problems by storing events in an outbox table within the same transactional database
used to persist state.

When a command is processed and a state change is made, a corresponding event is created in an outbox table within the same
database transaction.

An asynchronous intermediary service or process periodically reads events from the outbox table sequentially, publishes these
events to the messaging system, and then marks them as published or removes them from the table.

This pattern involves three main components:
 - The outbox table, the publishing process (also known as the message relay), and error handling — mandatory to make the pattern
truly resilient for the purpose for which it was designed.
 - The outbox table is a table in the transactional database where events are temporarily stored.
 - This table must be written within the same transaction as the data write to guarantee atomicity.
 - The publishing process is an asynchronous service or process that periodically reads events from the outbox table, publishes these events,
and marks them as published or removes them from the table (preferably).

Error handling must include mechanisms to deal with publishing failures, such as retries and monitoring, to ensure that
all events are eventually published and only removed upon that confirmation.

# Trade-offs and Disadvantages of Microservices

## The fundamental problem
Microservices trade **accidental complexity** (growing monolithic code) for **distributed complexity** (network, consistency, operations). This trade-off is only worth it when domain complexity justifies it — in most cases, it does not justify it from the start.

````
"Don't start with microservices. Monoliths are not legacy -- they're a valid architecture" -- Martin Fowler.
````

## 1. Operational Complexity
With microservices you end up operating dozens or hundreds of independent processes.
Each service needs its own CI/CD pipeline, configuration, monitoring, alerts, and incident runbook.

### What becomes harder:
* Coordinated deploys when services have dependencies on each other.
* Tracing a bug that crosses 5 services without distributed tracing.
* Managing versions of internal APIs without breaking consumers.
* On-call becomes much more complex — which service does the alert point to?

**Real cost**: A team of 5 engineers maintaining 20 microservices will spend more time on infrastructure than on the product.

## 2. Data Consistency
In a monolith with a relational database, a transaction spans multiple tables with native ACID guarantees.
In microservices, each service has its own database, and distributed transactions are inherently difficult.

### Monolith:
```sql
BEGIN TRANSACTION
    INSERT INTO orders ...
    UPDATE inventory ...
    INSERT INTO payments...
COMMIT -- all or nothing, guaranteed
```

### Microservices:
```
    OrderService -> creates order
    InventoryService -> decrements stock
    PaymentService -> processes payment (x fails)
```

What to do? Distributed rollback? Compensation?
*Saga Pattern* solves it, but adds complexity.

### Consequences:
* Eventual consistency instead of strong consistency — the system may be temporarily inconsistent.
* *Saga Pattern* is necessary for multi-service operations, but is hard to test and debug.
* Duplicate data between services is inevitable, and falls out of *sync*.

## 3. Network Latency
Calls between services that were in-process function calls (nanoseconds) become network calls (milliseconds).

* Monolith: `user.GetProfile()` 0.1ms (local call)
* Microservices: `UserService.GetProfile()` 5ms (HTTP/gRPC) + `OrderService.GetOrders()` 8ms + `NotificationService.GetPrefs()` 6ms
    = 19ms in network overhead alone (without serial calls).

### Derived problems
* Timeouts cascade: if one service is slow, the entire chain suffers.
* Retry storms: chained retries amplify load on unstable services.
* Partial failures are more common and harder to handle than total failures.

## 4. Debugging and Observability
Tracing a request that passed through 8 services without proper tools is extremely difficult. Stack traces do not cross process boundaries.

### What you absolutely need
* **Distributed tracing**: Jaeger, Zipkin, Tempo — to follow a request end-to-end.
* **Correlation IDs**: propagated in all headers and logs.
* **Centralized logging**: ELK, Loki — because logs spread across 20 pods are unusable.
* **Service mesh**: Istio, Linkerd — for traffic visibility between services.

**The cost**: This observability stack has a significant operational and financial cost that many teams underestimate.

## 5. Integration tests become much more complex.

### Monolith:
* Simple unit tests.
* Integration tests with a single database, a single process.
* e2e tests — bring up the app, run the tests.

### Microservices:
* Unit tests per service.
* Integration tests need to mock external dependencies.
* Contract tests: Pact/consumer-driven contracts for APIs between services.
* e2e tests: need to bring up the entire platform — slow, fragile, and expensive.

**Practical consequence**: Teams tend to have less real integration coverage, more bugs appear in production in interactions between services, and staging environments become outdated or incomplete.

## 6. Communication overhead between teams
Conway's Law says that the architecture of a system reflects the communication structure of the organization. Microservices work well when teams own complete services, but require much more coordination when a feature spans multiple teams.

### Real problems:
* API contracts between services become dependencies between teams — changes need to be coordinated.
* Versioning internal APIs is laborious (`v1/`, `v2/` or header-based).
* "Who owns this service?" — in companies that grow quickly, ownership becomes unclear.
* Synchronization meetings between teams increase proportionally to the number of services.

## 7. Premature Over-engineering
The worst trade-off of all — adopting microservices before understanding the domain well results in poorly divided services that are worse than a monolith.
* Wrong premature split:
 UserService / UserProfileService / UserPreferenceService / UserAuthService. Extremely high implicit coupling, chatty APIs, unnecessary distributed transactions.

* Correct split comes after understanding bounded contexts: IdentityService / CatalogService / OrderService / FulfillmentService. Low coupling, high cohesion, clear business boundaries.

* Distributed monolith: the worst of both worlds — services that need to be deployed together because they are coupled, but carry all the operational complexity of microservices.

## Trade-off summary

| Dimension              | Monolith                    | Microservices                       |
| ---------------------- | --------------------------- | ----------------------------------- |
| Data consistency       | Native ACID                 | Eventual, Saga Pattern              |
| Internal latency       | Nanoseconds                 | Milliseconds                        |
| Deploy                 | Simple, all at once         | Complex, independent per service    |
| Debugging              | Linear stack trace          | Requires distributed tracing        |
| Scalability            | Scales everything together  | Scales per service                  |
| Team autonomy          | Conflict in same codebase   | Independent teams                   |
| Observability          | Simple                      | Requires dedicated stack            |
| e2e tests              | Simple                      | Complex and fragile                 |
| Operational cost       | Low                         | High                                |

### When microservices truly pay off
* Large, autonomous teams: more than 2 pizzas per service as a reference.
* Domains with very different scaling requirements: the search module needs 100x more resources than the settings module.
* Independent deploys are critical: different teams need to release features without coordinating.
* Domain boundaries are well understood: you already understand the business well enough to split it without mistakes.


# Leetcode

## 146. LRU Cache
```go
import (
    "container/list"
    "sync"
)

type Cache struct {
    mutex sync.Mutex
    cache map[int]*list.Element
    list *list.List
    limit int
}

type Entry struct {
    key int
    value int
}

func NewCache(capacity int) *Cache{
    return &Cache{
        cache: make(map[int]*list.Element)
        list: list.New(),
    }
}

func (c *Cache) Get(key int) int {
    c.mutex.Lock()
    defer c.mutex.Unlock()

    if element, ok := c.cache[key]; ok && element != nil {
        c.list.MoveToFront(element)
        return element.Value.(*Entry).value
    }
    return -1
}

func (c *Cache) Put(key int, value int) {
    c.mutex.Lock()
    defer c.mutex.Unlock()

    // if key exists, update value and move to front
    if element, ok := c.cache[key]; ok && element != nil {
        element.Value.(*Entry).value = value
        c.list.MoveToFront(element)
    }
    entry := &Entry{key: key, value: value}
    element := c.list.PushFront(entry)
    c.cache[key] = element

    if c.list.Len() > c.limit {
        element = c.list.Back()
        if element != nil {
            key := element.Value.(*Entry).key
            delete(c.cache, key)
            c.list.Remove(element)
        }
    }
}
```

## 1. Two Sum
Given an array of integers nums and an integer target,
return indices of the two numbers such that they add up to target.
```go
func twoSum(nums []int, target int) []int {
    hasher := make(map[int]int, len(nums))

    for i, value := range nums {
        if n, ok := hasher[value]; ok {
            return []int{n, i}
        }
        hasher[target - value] = i
    }
    return nums
}
```

## 3. Longest Substring Without Repeating Characters (Sliding Window Maximum)
Given a string s, find the length of the longest substring without duplicate characters.
```go
func lengthOfLongestSubstring(s string) int {
    maxLength, left := 0, 0
    m := make(map[rune]int)

    for right, c := range s {
        if pos, ok := m[c]; ok && pos >= left {
            left = pos + 1
        }
        maxLength = max(maxLength, (right - left) + 1)
        m[c] = right
    }
    return maxLength
}
```

## 15. Three Sum
Given an integer array nums, return all the triplets [nums[i], nums[j], nums[k]] such that i != j, i != k,
and j != k, and nums[i] + nums[j] + nums[k] == 0.
Notice that the solution set must not contain duplicate triplets.
```go
func threeSum(nums []int) [][]int {
	sort.Ints(nums)
	result := [][]int{}

	for i := 0; i < len(nums)-2; i++ {
		if i > 0 && nums[i] == nums[i-1] {
			continue
		}
		target, left, right := -nums[i], i+1, len(nums)-1

		for left < right {
			sum := nums[left] + nums[right]
			switch {
			case sum == target:
				result = append(result, []int{nums[i], nums[left], nums[right]})
                left, right = left + 1, right - 1
				for left < right && nums[left] == nums[left-1] {
					left++
				}
				for left < right && nums[right] == nums[right+1] {
					right--
				}
			case sum < target:
				left++
			case sum > target:
				right--
			}
		}
	}
	return result
}
```

## 239. Sliding Window Maximum
You are given an array of integers nums, there is a sliding window of size k
which is moving from the very left of the array to the very right.
You can only see the k numbers in the window. Each time the sliding window moves right by one position.
Return the max sliding window.
```go
func maxSlidingWindow(nums []int, k int) []int {
    deque, result := make([]int, 0), make([]int, 0)

    for i, v := range nums {
        // remove out-of-window indices
        if len(deque) > 0 && deque[0] <= (i - k) {
            deque = deque[1:]
        }
        // 2. Maintain decreasing order
        for len(deque) > 0 && nums[deque[len(deque)-1]] <= v {
            deque = deque[:len(deque) - 1]
        }
        // 3. insert current index at the tail
        deque = append(deque, i)
        // 4. The window is only complete starting from index k-1
        if i >= (k - 1) {
            result = append(result, nums[deque[0]])
        }
    }
    return result
}
```

## 104. Maximum Depth of Binary Tree
Given the root of a binary tree, return its maximum depth.
A binary tree's maximum depth is the number of nodes along the longest
path from the root node down to the farthest leaf node.
```go
type TreeNode struct {
	Val   int
	Left  *TreeNode
	Right *TreeNode
}

func maxDepth(root *TreeNode) int {
	if root == nil {
		return 0
	}

	var dfs func(root *TreeNode, depth int) int
	dfs = func(root *TreeNode, depth int) int {
		if root == nil {
			return depth
		}
		left := dfs(root.Left, depth+1)
		right := dfs(root.Right, depth+1)
		return max(left, right)
	}
	return dfs(root, 0)
}
```

## 21. Merge Two Sorted Lists
You are given the heads of two sorted linked lists list1 and list2.
Merge the two lists into one sorted list. The list should be made by splicing together the nodes of the first two lists.
Return the head of the merged linked list.
```go
func mergeTwoLists(list1 *ListNode, list2 *ListNode) *ListNode {
    dummy := &ListNode{}
    current := dummy

    for list1 != nil && list2 != nil {
        if list1.Val < list2.Val {
            current.Next = list1
            list1 = list1.Next
        } else {
            current.Next = list2
            list2 = list2.Next
        }
        current = current.Next
    }
    if list1 != nil {
        current.Next = list1
    } else {
        current.Next = list2
    }
    return dummy.Next
}
```

## 125. Valid Palindrome
A phrase is a palindrome if, after converting all uppercase letters into lowercase letters and removing all non-alphanumeric characters, it reads the same forward and backward. Alphanumeric characters include letters and numbers.

Given a string s, return true if it is a palindrome, or false otherwise.
```go
func isPalindrome(s string) bool {
	l, r := 0, len(s)-1
	s = strings.ToLower(s)

	for l < r {
		if !isAlphaNumeric(s[l]) {
			l = l + 1
			continue
		}
		if !isAlphaNumeric(s[r]) {
			r = r - 1
			continue
		}
		if s[l] != s[r] {
			return false
		}
		l, r = l+1, r-1
	}
	return true
}

func isAlphaNumeric(c byte) bool {
	return (c >= 'a' && c <= 'z') || (c >= '0' && c <= '9')
}
```

## Find Cipher
You're writing a puzzle helper app in which a user can enter a word, and the app
returns a list of words that are substitution ciphers of the user's word. The list
of substitution ciphers is derived from a large list of words available to the app.

A cipher is a code that can convert one string to another.

Two strings are considered substitution ciphers of each other if there exists a cipher
to convert from the first string to the second and there exists a cipher to convert
from the second string to the first. In other words, each letter in the first word can
be replaced by the SAME letter to convert it to the second word, AND each letter in the
second word can be replaced by the SAME letter to convert it to the first word.

A few examples: `banana`, `cololo`
`cololo` is a valid substitution cipher of banana because each character in `banana`
is replaced by the *same* character in cololo for every occurrence:
b : c
a : o
n : l
`banana` is also a valid substitution cipher of `cololo`:
c : b
o : a
l : n
`banana`, `cololl`
The first and second 'a' in banana are replaced by 'o' but the third 'a' is replaced by
'l' — hence not a valid substitution.

An example of the puzzle helper app:

Word list: [banana, abdbdb, cat, mom, tot]
Input: cololo
Output: [banana, abdbdb]

Input: pop
Output: [mom, tot]

Input: dog
Output: [cat]

```go
func Puzzle(input string, list []string) []string {
	res := []string{}
	for _, s := range list {
		ok := findCipher(s, input)
		if ok {
			res = append(res, s)
		}
	}
	return res
}

func findCipher(s string, input string) bool {
	if len(s) != len(input) {
		return false
	}
	forward := make(map[rune]rune)
	backward := make(map[rune]rune)

	for i := range len(s) {
		ci, cs := rune(input[i]), rune(s[i])
		if c, ok := forward[ci]; ok {
			if c != cs {
				return false
			}
		} else {
			forward[ci] = cs
		}
		if c, ok := backward[cs]; ok {
			if c != ci {
				return false
			}
		} else {
			backward[cs] = ci
		}
	}
	return len(backward) == len(forward)
}
```
