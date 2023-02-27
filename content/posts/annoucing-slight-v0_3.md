---
title: "Annoucing Slight v0.3"
description: "Announcing new features and updates on Slight v0.3 release."
date: 2023-02-25 00:00:00 +0000 UTC
authorname: "Jiaxiao (Joe) Zhou"
author: "@jiaxiao_zhou"
authorlink: "https://twitter.com/jiaiao_zhou"
tags: ["slight", "spiderlightning", "Wasm", "WASI"]
---

We are excited to announce the release of Slight v0.3, the latest version of our WebAssembly runtime that utilizes SpiderLightning (also known as wasi-cloud) capabilities.

You can download the latest release by running the following command if you are using Linux or macOS:

```bash
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/deislabs/spiderlightning/main/install.sh)"
```

If you are using Windows, you can download the latest release by running this command:

```powershell
iex ((New-Object System.Net.WebClient).DownloadString('https://raw.githubusercontent.com/deislabs/spiderlightning/main/install.ps1'))
```

This release includes several new features, such as the ability to author HTTP clients as Wasm modules, publish or receive events, query databases with SQL, and much more. You can find all the [example Slight applications in our repository](https://github.com/deislabs/spiderlightning/tree/main/examples). Now, let’s dive into it!

### HTTP Client

Previously, Slight only supports the ability to serve HTTP requests. You can write applications in Rust or C to handle incoming HTTP requests and compile them to Wasm modules. And now, Slight supports the ability to make an outbound HTTP request to a URL. Take the following Rust program as an example:

```rust
wit_bindgen_rust::import("wit/http-client.wit");

#[register_handler]
fn handle_hello(_req: Request) -> Result<Response, HttpError> {
    let req = crate::http_client::Request {
        method: crate::http_client::Method::Get,
        uri: "https://some-random-api.ml/facts/dog",
        headers: &[],
        body: None,
        params: &[],
    };
    let res = crate::http_client::request(req).unwrap();
    println!("{:?}", res);

    let res = Response {
        status: res.status,
        headers: res.headers,
        body: res.body,
    };
    Ok(res)
}
```

You can use `wit-bindgen` tool to generate guest binding from `http-client.wit` WIT file.

> 💡 The WIT file can be downloaded using `slight add http-client@v0.3.1`

The `handle_hello` function is responsible for handling incoming HTTP requests. It achieves this by creating a `Request` object from the generated `http_client` module, which is then sent using the `request` function. The `request` function makes an HTTP request to the URI defined in the `Request` struct.

It is worth noting that the `request` function is also generated. Its purpose is to handle the actual sending of the `Request` object. Once the `request` function has completed this task, it returns a `Response` object, which serves as the return value for the `handle_hello` function.

Developers can easily interface with the HTTP request processing mechanism by using the `handle_hello` function. They can be confident that their HTTP requests will be handled appropriately. This function is a crucial part of any HTTP-based application.

Lastly, to enable both `HTTP` and `HTTP-client` capabilities, we need to update the configuration file. Below is an example slightfile.toml file.

```rust
specversion = "0.2"

[[capability]]
resource = "http"
name = "my-rest-api"
    # This capability does not require any configs

[[capability]]
resource = "http-client"
name = "something"
```

### Exchange Messaging

In this new release, we have combined the `message-queue` and `pub/sub` capabilities into one called `messaging`. The idea behind this move is that we think there are many overlaps and ambiguities to both capabilities and it made a difficult time for developers to choose one over another. Read's Dan's [proposal explaining the design of the new messaging capability](https://github.com/deislabs/spiderlightning/blob/main/proposals/abandoned/pubsub-mq-interfaces-proposal.md) for more details.

In short, the new messaging capability interface looks like the following in WIT.

```go
// producer interface
resource pub {
    /// creates a handle to a pub object
    static open: func(name: string) -> expected<pub, messaging-error>

    /// publish a message to a topic
    publish: func(msg: list<u8>, topic: string) -> expected<unit, messaging-error> 
}

/// consumer interface
resource sub {
    /// creates a handle to a sub object
    static open: func(name: string) -> expected<sub, messaging-error>

    /// subscribe to a topic
    subscribe: func(topic: string) -> expected<subscription-token, messaging-error> 

    /// pull-based message delivery
    receive: func(sub-tok: subscription-token) -> expected<list<u8>, messaging-error>
}
```

> 💡 The WIT file can be downloaded using `slight add messaging@v0.3.1`

A publisher service can target the `pub` resource to send messages to a topic. Similarly, a subscriber service can choose to subscribe to a topic and then pull the broker to receive messages. This way, the publisher and subscriber services can communicate with each other through the broker, without needing to know each other's details.

Below is an example in Rust showing how you can write a simple publisher service that compiles to Wasm. In this example, the service sends a message to a topic named "rust" and “global-chat” with the messages:

```rust
use messaging::*;
wit_bindgen_rust::import!("../../wit/messaging.wit");
wit_error_rs::impl_error!(messaging::MessagingError);

fn main() -> Result<()> {
    let ps = Pub::open("my-messaging")?;
    for i in 0..3 {
        println!("sending messages");
        ps.publish(format!("rust-value-{}", i).as_bytes(), "rust")?;
        ps.publish(format!("gc-value-{}", i).as_bytes(), "global-chat")?;
        thread::sleep(Duration::from_secs(3));
    }
    Ok(())
}
```

And an example showing how a subscriber service could be authored:

```rust
let ps = Sub::open("my-messaging")?;
let sub_tok = ps.subscribe("rust")?;
let sub_tok1 = ps.subscribe("global-chat")?;

for _ in 0..3 {
    loop {
        let msg = ps.receive(&sub_tok)?;
        if !msg.is_empty() {
            println!("received message from topic 'rust'> value: {:?}", String::from_utf8(msg));
            break;
        }
    }

    loop {
        let msg = ps.receive(&sub_tok1)?;
        if !msg.is_empty() {
            println!("received message from topic 'global-chat'> value: {:?}", String::from_utf8(msg));
            break;
        }
    }
}
```

One limitation of the current messaging interface and implementation is the lack of support for streaming events. This can be problematic in situations where the receive function is a blocking call that tries to pull the message broker for a new message. If the broker does not have any new messages, the receive function will be blocked, which can cause delays in message delivery. Additionally, the lack of streaming support can limit the scalability of the system, as the blocking call can cause bottlenecks in high-traffic scenarios. To address this limitation, we will be adding `time-to-expire` to the receive function in the near future and will be collaborating with the Wasm Component Model community to land `streams` feature in WIT.

### Using the SQL Capability

The latest release from Slight adds an exciting new capability to its portfolio of capabilities. The newly added SpiderLightning functionality enables WebAssembly programs to safely and generically interact with SQL databases. This capability is not only highly versatile, but also secure, as it includes functions for querying and modifying data using prepared statements, as well as handling errors in a way that ensures data integrity.

Below is an example of using the SQL service in Slight.

```rust
use sql::*;
wit_bindgen_rust::import!("wit/sql.wit");
wit_error_rs::impl_error!(sql::SqlError);

fn main() -> Result<()> {
    let sql = Sql::open("my-db")?;

    // create table if it doesn't exist
    sql.exec(&sql::Statement::prepare(
        "CREATE TABLE IF NOT EXISTS users (id SERIAL PRIMARY KEY, name TEXT NOT NULL)",
        &[],
    ))?;

    // add new user
    let rng = RNG::new(&Language::Elven).unwrap();
    let name = rng.generate_name();
    sql.exec(&sql::Statement::prepare(
        "INSERT INTO users (name) VALUES (?)",
        &[&name],
    ))?;

    // get all users
    let all_users = sql.query(&sql::Statement::prepare(
        "SELECT name FROM users",
        &[],
    ))?;
    dbg!(all_users);

    // get one user
    let one_user = sql.query(&sql::Statement::prepare("SELECT name FROM users WHERE id = ?", &["2"]))?;
    dbg!(one_user);

    // try sql injection
    assert!(sql
        .query(&sql::Statement::prepare("SELECT name FROM users WHERE id = ?", &["2 OR 1=1"]))
        .is_err());

    Ok(())
}
```

> 💡 The WIT file can be downloaded using `slight add sql@v0.3.1`

The SQL interface is paired with a slight configuration file to tell the host which SQL service is needed to provide this capability. Below is a sample `slightfile.toml` using the PostgreSQL database implementation, which is the only service Slight supports right now.

```toml
specversion = "0.2"

[[capability]]
resource = "sql.postgres"
name = "my-db"
    [capability.configs]
    POSTGRES_CONNECTION_URL = "${azapp.POSTGRES_CONNECTION_URL}"
```

### Thank you

We would like to take this opportunity to express our gratitude to all new contributors who have contributed to Slight!

As we mentioned in our **[Introducing SpiderLightning - A Cloud System Interface with WebAssembly](https://deislabs.io/posts/introducing-spiderlightning-and-slight/)** blog, we are excited for the future of WebAssembly Component Model and WASI. We firmly believe that the work the Bytecode Alliance has done will enable all developers to build more portable and much more secure applications that can run anywhere! Thank you, for all the Bytecode Alliance contributors and maintainers for their work!
