---
title: "Why async Functions in Rust with `tokio::spawn` Must Be `'static` (and What `move` Really Means)"
description: ""
pubDate: 'Jun 30 2026'
lang: en
heroImage: '../../assets/Rust_en.webp'
---


# Why async Functions in Rust with `tokio::spawn` Must Be `'static` (and What `move` Really Means)


I want to explain an important detail about asynchronous programming in Rust, especially when using `tokio::spawn`:  
**Why do async functions need to be** `**'static**`**?** And what does `move` mean in this context?

## async and Tokio

In Rust, when you write an `async` function, it **does not run immediately**. Instead, it returns a **Future** that needs to be executed by a runtime like **Tokio**.  
When building concurrent applications, such as handling multiple TCP connections, we often use `tokio::spawn` to run tasks **concurrently**.

## What is `tokio::spawn`?

`tokio::spawn` creates a **new asynchronous task** that runs on the Tokio runtime.  
For example, when building a TCP server, you typically call `tokio::spawn` for each new connection, so each connection can be handled independently without blocking others.

## Why does it require `'static`?

Any future you pass to `tokio::spawn` must satisfy these constraints:

- It must be `Send` (safe to move across threads)
- It must be `'static` (it doesn’t borrow data with a shorter lifetime)

Why? Because `tokio::spawn` may **schedule the task to run later** or even **move it to a different thread**.  
If your task references data that could be dropped before the task runs, it would cause **undefined behavior or a crash**. That’s why the future needs to be `'static`.

## The solution: `move` in async blocks

To make sure the task owns the data it needs, we use `move` when creating the async block.  
This transfers ownership of variables into the task, ensuring they outlive the async execution.

```rust

let (socket, addr) = listener.accept().await?;  
tokio::spawn(async move {  
    // socket is now owned by this task  
    // Safe to use even if the outer scope ends  
});

```

Without `move`, the async block would only borrow variables from the outer scope, which could lead to lifetime issues and compiler errors.

## Example: Building a simple TCP server with Tokio

```rust
use tokio::net::TcpListener;  
use tokio::io::{AsyncReadExt, AsyncWriteExt};  
  
#[tokio::main]  
async fn main() -> Result<(), Box<dyn std::error::Error>> {  
	let listener = TcpListener::bind("127.0.0.1:8080").await?;  
	println!("Server running on 127.0.0.1:8080");  
  
	loop {  
		let (mut socket, addr) = listener.accept().await?;  
		println!("New client: {}", addr);  
  
		tokio::spawn(async move {  
			let mut buf = [0u8; 1024];  
			loop {  
				let n = match socket.read(&mut buf).await {  
					Ok(0) => break, // Connection closed  
					Ok(n) => n,  
					Err(_) => break,  
				};  
  
				if socket.write_all(&buf[0..n]).await.is_err() {  
					break;  
				}  
			}  
		});  
	}  
}

```

## Key takeaways

- Anything passed to `tokio::spawn` must be `'static` because tasks may run much later or on another thread.
- Use `move` to transfer ownership of variables into the async block so the future does not borrow temporary data.
- If you see a lifetime or `'static` error, you likely need `move` or a different ownership strategy.

Hope this was helpful!
