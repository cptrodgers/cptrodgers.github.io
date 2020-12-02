---
published: true
---
# Rust: Use futures@0.1 and futures@0.3 together

Content of this post is helping from [https://rust-lang.github.io/futures-rs/blog/2019/04/18/compatibility-layer.html](https://rust-lang.github.io/futures-rs/blog/2019/04/18/compatibility-layer.html) and my experience while developing [Axie Infinity](https://axieinfinity.com/) Game Server on Actix.

Thank to @future-rs team and:

- **[@tinaun](https://www.github.com/tinaun)**
- **[@Nemo157](https://www.github.com/Nemo157)**
- **[@cramertj](https://www.github.com/cramertj)**
- **[@LucioFranco](https://www.github.com/LucioFranco)**

# Introduction

Rust has multiple ways to handle concurrent jobs. You can spawn OS thread by std libraries of Rust by [spawn](https://doc.rust-lang.org/std/thread/fn.spawn.html) function. If you want a cheap thread than an OS thread, you can use the third library to spawn future tasks (async task) or green thread such as [tokio](https://tokio.rs/) runtime. 

In this blog, I'm addressing to Async processing problem in Rust. In the async context, Rust doesn't have a runtime inside, It relies on third libraries' runtime. We have 2 promising competitors here: tokio and async-std. I cannot talk to you about what is best. It depends on your application dependencies depending. For example, actix (Actor Concurrent Model) build on tokio, you should choose tokio run time for your project. You feel hard? Don't worry, Rust keeps it standard and compatible between multiple third libraries. It defined the interface for Async Function called Future trait. There are 2 big versions of the Future trait:

In the first version, Rust provides future feature with 2 associated types: Item and Error and you cannot use `async/await` syntax. The Item and Error are similar to Result. It means the interface of Future seems duplicated logic with Result and in case that you don't want to return Error you also need to declare Error Type for Future result - sound bad.

```rust
fn get_one() -> impl futures::Future<Item = i32, Error = ()> {
	futures::future::ok(1)
}

fn main() {
	let task = get_one().then(|res| {
		match res {
		    Ok(one) => println!("{}", one),
		    _ => (),
		}
	})
	
	tokio::run(task)
}
```

The second version (currently version) of Future Trait is used one associated type called Output and you can use async/await syntax. If you want to return Result. You just assign Result enum to Output - sound great.

```rust
use tokio; // 0.3.4

// async is sugar syntax, this function will return impl Future<Output = i32>
async fn get_one() -> i32 {
    1
}

#[tokio::main]
async fn main() {
	let one = get_one().await;
	// We don't need to handle Error as Future@0.1
	// Code is readable than Future@0.1
	println!("{}", one);
}
```

Yeh. Async Await syntax. Readable code. All problems solved? No. The problem is a lot of outdated libraries that are using futures@0.1. The transformation from the first version to the third version takes more time. There are some struggles:

- The interface is changed. We cannot use two libs based on two different Future Traits. If you update a library, you need to check the Future trait of it.
- One selling point of Rust is Fearless Concurrent which means concurrent programming with Rust is easy. But, a new guy who comes to this young language will confuse and PANIC while coding and implement Rust dependencies to their Rust project.

Don't worry, the future-rs team provides a way to use 2 future traits together in the same project. It means you can use a lot of user interfaces of futures@0.3 in the project base on futures@0.1 runtime. For easy example:

```rust
let future03 = async {
    println!("Running on the pool");
};

let future01 = future03
    .unit_error()
    .boxed()
    .compat();
```

# Compatible layer Using (Actix example)

Wall of text? We need practice. I'm a fan of practical. We have 2 cases here: executors are 0.1 and executors is 0.3.

First step in two cases, add 2 interfaces in your Cargo.toml. It's needed because I'm not sure about your std futures.

```rust
futures01 = { package = "futures", version = "0.1.6" } // Add futures@0.1
futures = { package = "futures", version = "0.3.2", features = ["compat"] }
// Add futures@0.3 with compat feature.
```

## Case 1: Based on futures@0.1 executors

If your runtime is base on futures@0.1 executors. It seems outdated or legacy at current time (this post is written). For example, we use `actix-web@1.0.9` . This actix support futures@0.1.

```rust
use actix_web::{web, App, Responder, HttpServer, Error};
use futures01::Future; // Import future@0.1 trait. 
use futures::future::{FutureExt, TryFutureExt}; // Import compat function from futures@0.3
use futures::compat::Future01CompatExt;

// Simple async function with futures@0.3
async fn hello_3() {
    println!("Hello in Async 0.3");
}

// Simple async function with futures@0.1
fn hello_1() -> impl Future<Item = (), Error = ()> {
    println!("Hello in Async 0.1");
    futures01::future::ok(())
}

fn hello() -> impl Future<Item = impl Responder, Error = Error> {
    let task = async {
        hello_3().await;
        hello_1().compat().await; // From futures0.1 to futures0.3, we just use compat fn
    
    Ok("Hello")
    }; // task variable is future0.3;

    // To make compat from 0.3 to 0.1, We need pin future.
    // In actix, we need boxed_local because it doesn't have Send trait.
    task.boxed_local().compat()
}

fn main() -> std::io::Result<()> {
    HttpServer::new(|| App::new().service(
        web::resource("/").to_async(hello))
    )
    .bind("127.0.0.1:8080")?
    .run()
}
```

## Case 2:  Based on futures@0.3 executors

This is happy easy case now. We have runtime 0.3 and lib is 0.1. For example actix-web

```rust
use actix_web::{get, web, App, HttpServer, Responder};
use futures01::Future; // Import future@0.1 trait. 
use futures::compat::Future01CompatExt;

fn hello_1() -> impl Future<Item = (), Error = ()> {
    println!("Hello in Async 0.1");
    futures01::future::ok(())
}

#[get("/")]
async fn hello() -> impl Responder {
    hello_1.compat().await;
    
    "Hello"
}

#[actix_web::main]
async fn main() -> std::io::Result<()> {
    HttpServer::new(|| App::new().service(index))
        .bind("127.0.0.1:8080")?
        .run()
        .await
}
```

## Others:

### Wrong Tokio Runtime.

Sometimes, you're compiling code and received an error about wrong tokio runtime. Such as [reqwest error](https://github.com/tokio-rs/tokio/issues/1412). You need to make compatible your third party lib and current runtime.

### Boxed and Boxed Local

Why we have both `.boxed` and `.boxed_local` . In short, 2 functions make the same logic. Pin your future, it will notify the compiler doesn't move this memory. The boxed function use for the future that implements Send. The boxed_local function use for the future that doesn't implement Send. For example, the actix-web controller future function doesn't implement Send. So, your output function should be not implement Send to make it compatible with the actix-web controller signature. You should use boxed_local.
