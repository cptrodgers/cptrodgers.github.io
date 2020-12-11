---
published: true
---

Actix is an Actor framework for Rust (According to the actix github page). Actor is a high level of concurrency model. The tips below that are useful for newbies who want to boost their performance in Actix.

+ Websocket with Actor
+ Async Actor and Sync Actor
+ Handle Async Function
+ System Service and Arbiter Service
+ Alternative System Service but in Arbiter Thread (Non-Block System)
+ Interval Task


# Tips:

### Websocket with Actor.

Actix works well with Websocket protocol. Each WebSocket is an actor with the potential to handle a Stream. This is the sample code to handle Websocket in Actix.

```rust
actix = "0.9.0"
actix-web-actors = "2.0.0"
actix-web = "2.0.0-rc"
```

```rust
use std::time::{Duration, Instant};

use actix::prelude::*;
use actix_web_actors::ws;

const HEARTBEAT_INTERVAL: Duration = Duration::from_secs(5);
const CLIENT_TIMEOUT: Duration = Duration::from_secs(30);

#[derive(Debug)]
pub struct Connection {
    heartbeat: Instant,
}

impl Connection {
    pub fn new() -> Self {
        Connection {
            heartbeat: Instant::now(),
        }
    }
}

// Add Actor for Connection
impl Actor for Connection {
    type Context = ws::WebsocketContext<Self>;

    fn started(&mut self, ctx: &mut Self::Context) {
        self.run_heartbeat_check(ctx);
    }
}

impl Connection {
		// Run interval to check state of connection
    fn run_heartbeat_check(&self, ctx: &mut ws::WebsocketContext<Self>) {
        ctx.run_interval(HEARTBEAT_INTERVAL, |act, ctx| {
            // check client heartbeats
            if Instant::now().duration_since(act.heartbeat) > CLIENT_TIMEOUT {
                warn!(
                    "Websocket Client heartbeat failed, disconnecting!",
                );
                // stop actor
                ctx.stop();
            } else {
                ctx.ping(b"");
            }
        });
    }
}

// Now, We need to handle ws message from client
// Each message will be handled by this Handler
impl StreamHandler<Result<ws::Message, ws::ProtocolError>> for Connection {
    fn handle(&mut self, item: Result<ws::Message, ws::ProtocolError>, ctx: &mut Self::Context) {
        match item {
            Ok(ws::Message::Binary(bytes)) => {
					      prinln!("Received Bytes Message");
								// Send Binary Message to Client
								ctx.binary(b"Hello from server");
            },
						Ok(ws::Message::Text(text) => {
								println!("Receied Text Message");
								// Send Text String to Client
								ctx.text("Hello from server");
						},
            Ok(ws::Message::Ping(msg)) => ctx.pong(&msg),
						// Tracking heartbeat of Websocket
            Ok(ws::Message::Pong(_)) => self.heartbeat = Instant::now(),
            _ => (),
        }
    }
}
```

### Async Actor and Sync Actor

According to the github page of Actix, It provides two types: Sync Actor and Async Actor. Each actor has its benefits. You need to know to choose the right Actor type.

**Async Actor:**

In Default: Actor type is Async Actor. It helps you handle async functions in the handler. It also processes the message in an async way.

```rust
use actix::prelude::*;

pub async fn print_one() {
    print!("One");
}

pub struct PrintActor;

impl Actor for PrintActor {
    type Context = Context<Self>;
}

#[derive(Message)]
#[rtype(result="()")]
pub struct PrintOne;

impl Handler<PrintOne> for PrintActor {
    type Result = ();

    fn handle(&mut self, _: PrintOne, ctx: &mut Self::Context) -> Self::Result {
        let task = async {
            print_one().await;
        };

        wrap_future::<_, Self>(task).wait(ctx);
    }
}
```

**Sync Actor:** 

Actix also supports Sync Actor. Sync Actor cannot handle async function as Sync Actor. Using Sync Actor is a rare use case because Async Actor does almost things better than Sync Actor. Sync Actor provides distributed messages. I prefer you to read this document because I don't have experience in this type.

[https://docs.rs/actix/0.10.0/actix/sync/struct.SyncArbiter.html](https://docs.rs/actix/0.10.0/actix/sync/struct.SyncArbiter.html)

### Handle Async Function

The common question when using async result is how to handle async function and return the result in Actor. Actix provide a solution is ResponseActFuture.

```rust
use actix::prelude::*;

pub async fn get_one() -> Result<i32, ()> {
    Ok(1)
}

pub struct GetActor;

impl Actor for GetActor {
    type Context = Context<Self>;
}

#[derive(Message)]
#[rtype(result="Result<i32, ()>")]
pub struct GetOne;

impl Handler<GetOne> for GetActor {
    type Result = ResponseActFuture<Self, Result<i32, ()>>;
		
		// ResponseActFuture is wrapper of Future
		// It will address an Phantom Type for this future.
    fn handle(&mut self, _: GetOne, _: &mut Self::Context) -> Self::Result {
        Box::new(wrap_future::<_, Self>(get_one()))
    }
}
```

### Wait and Spawn

In async context (async actor), actix provide two efficient way to handle non-return message that spawns another thread to handle this task. It's similar to another concurrent library, spawn and wait. In the context of Actor, It's relating to process messages instead of thread.

- Spawn will spawn a new thread to handle the current task and don't stop the process of the new message. This is efficient in case you don't care about ordering messages.

```rust
use actix::prelude::*;
use tokio::time::{sleep, Duration};

pub async fn print_one() {
    print!("One");
}

pub struct PrintActor;

impl Actor for PrintActor {
    type Context = Context<Self>;
}

#[derive(Message)]
#[rtype(result="()")]
pub struct PrintOne;

impl Handler<PrintOne> for PrintActor {
    type Result = ();

    fn handle(&mut self, _: PrintOne, ctx: &mut Self::Context) -> Self::Result {
        let task = async {
						sleep(Duration::from_millis(500)).await;
            print_one().await;
        };
				
				// Actix will spawn and continue processing new message although
				// we want to sleep in 0.5s
        wrap_future::<_, Self>(task).spawn(ctx);
    }
}
```

- Waiting is another level of spawn. It will spawn a new thread to handle the current task and stop the process of new incoming messages until this task complete. This is useful in case you have an important task that needs to complete before processing another message. For example: Check authentication.

```rust
use actix::prelude::*;
use tokio::time::{sleep, Duration};

pub async fn print_one() {
    print!("One");
}

pub struct PrintActor;

impl Actor for PrintActor {
    type Context = Context<Self>;
}

#[derive(Message)]
#[rtype(result="()")]
pub struct PrintOne;

impl Handler<PrintOne> for PrintActor {
    type Result = ();

    fn handle(&mut self, _: PrintOne, ctx: &mut Self::Context) -> Self::Result {
        let task = async {
						sleep(Duration::from_millis(500)).await;
            print_one().await;
        };

				// Actix will spawn new thread and wait for complete this task
				// It means incoming message should wait at least 0.5s
				// for complete this task
        wrap_future::<_, Self>(task).wait(ctx);
    }
}
```

### System Service and Arbiter Service

Service is a useful actor concept in Actix. Service can be access in an acceptable space, it like a constant Actor.

- System Service: This Service Actor can be accessed in any space of our code, similar to a global constant.
- Arbiter Service: This Service Actor can only be accessed in Artiber Thread Space, similar scope constant.

Service Actor is useful for Database Pool, Connection Counter, Authenticator. 

```rust
use actix::prelude::*;

#[derive(Default)]
pub struct ConnectionCounter {
    count: i32,
}

impl Actor for ConnectionCounter {
    type Context = Context<Self>;
}

// This is required by SystemService
impl Supervised for ConnectionCounter {}
// Make Actor to Actor Service. 
// Basicall, Actix will register this address to a hashmap
impl SystemService for ConnectionCounter {}

#[derive(Message)]
#[rtype(result="()")]
pub struct CountConnection;

impl Handler<CountConnection> for ConnectionCounter {
    type Result = ();
    
    fn handle(&mut self, _: CountConnection, _: &mut Self::Context) -> Self::Result {
        self.count += 1;
    }
}

fn count_conn() {
		// from_registry() will return Addr<ConnectionCounter>, 
		// so, you can easy to send messeage to this actor
    ConnectionCounter::from_registry().do_send(CountConnection);
}
```

### Alternative System Service but in different thread

Now, this is some advance than before tips. Let me analyze the logic in System Service. One of my questions is "Where will System Service run on?". The answer is System Thread. Almost spaces of the main processing of our system. It means, If you make some stupid logic in System Service such as loop forever in the bare thread, your system will be stopped by your system service. It's a bad idea.

To be clear, I faced this problem while developing Axie Infinity Battle. Our System has a DBPool Actor. DBPool is a Database Pool - It manages Connection with Postgres and Redis. Inside ConnectionPool of r2d2 lib, it has a block code that waits until a new connection is available. It's dangerous in case your system is scaling with a lot of users, the pool is full and it will block the current thread (in this case system thread).

For this problem, we can use another approach that similar to SystemService but it will be run on a different thread. I prefer to use lazy_static in this case.

```rust
lazy_static = "1.4.0"
actix = "0.9.0"
```

```rust
lazy_static! {
		// Create a static pointer that store Address of DBPool
		// Use Mutex to serve accessing from multiple threads.
    static ref POOL_ADDR: Mutex<Option<Addr<DBPool>>> = Mutex::new(None);
}

pub fn start_pool() {
		// Create an arbiter thread
    let arbiter = Arbiter::new();
		// Start actor in arbiter thread and take address
    let pool_addr = DBPool::start_in_arbiter(&arbiter, |_| DBPool::default());
		// Assign to Pool Address pointer in lazy_static
    *POOL_ADDR.lock().unwrap() = Some(pool_addr);
}

pub struct DBPool {
    postgres: Pool<ConnectionManager<PgConnection>>,
}

impl DBPool {
    pub fn get_address() -> Addr<Self> {
        POOL_ADDR.lock().unwrap().clone().unwrap()
    }
}

impl Default for DBPool {
    fn default() -> Self {
	      let manager = ConnectionManager::<PgConnection>::new("database_url");
			  let postgres_pool = r2d2::Pool::builder()
		        .max_size(20)
		        .build(manager)
		        .expect("Build connection pool fails");
        Self {
            postgres: postgres_pool,
        }
    }
}

// From another actor
let db_address = DBPool::get_address()
```

### Interval Task

Sometimes, you want to loop interval in a period to do something. Actix also provides a good interface to make it work. For example, Check collection is live.

```rust

impl Actor for Connection {
    type Context = ws::WebsocketContext<Self>;

    fn started(&mut self, ctx: &mut Self::Context) {
		// This task run after 15s.
        ctx.run_interval(Duration::from_secs(15), |act, ctx| {
            if Instant::now().duration_since(act.heartbeat) > CLIENT_TIMEOUT {         
                // Disconnected, Stop this Actor
                ctx.stop();
            } else {
								// Connecting, Send a Ping message to keep alive.
                ctx.ping(b"");
            }
        });
    }
}

```
