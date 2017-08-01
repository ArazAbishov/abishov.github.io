---
layout: post
title: >
  Implementing a bot for Slack in Rust, Rocket and Anterofit - Part 1
date: 2017-07-27 15:00:00 +0000
---

Rust is a systems programming language which enables developers to write safe and fast code without sacrificing high-level language constructs. At first, it seems that Rust is targeting only performance critical use cases, but the original intention is far more ambitious. Frameworks like Rocket, Serde and Anterofit make Rust a good fit for the web application development as well.

This series of blog posts will be dedicated to building simple slack bot for searching github repositories. Developing a bot in systems programming language might seem to be a crazy idea, but the intention is to showcase how powerful Rust is. The whole implementation is about 150 lines of code, which is quite amazing.

The first part is dedicated to the implementation of a command line utility using the libraries Anterofit and Serde. Following that, the next post focuses on integration of the Rocket framework, which will turn the command line utility into functioning web app. The last blog post will be about managing and running the bot behind a NGINX reverse proxy on a RaspberryPi.

## Preparing development environment
One of the dependencies - Rocket, requires the nightly version of Rust compiler. We will need to make sure that it is installed and configured correctly.

> In order to keep the development environment reproducible and isolated, I highly recommend using [Vagrant](https://www.vagrantup.com/). Especially when you are working simultaneously on multiple projects, that require different versions of the compiler.  

In case if you don't have `rustup` installed on your machine, execute following command:

```bash
curl https://sh.rustup.rs -sSf | sh

# source environment variables after installation
source $HOME/.cargo/env
```

Now we are going to switch to the nightly compiler and set it as the default version:

```bash
# checking for updates
rustup update

# install nightly compiler
rustup install nightly

# set nightly compiler as default one
rustup default nightly
```

Remember that you can always switch back to the stable compiler by executing `rustup default stable`.

Hyper – a HTTP library which is used both by the Rocket and Anterofit, has a dependency on OpenSSL native libraries that we need to install system-wide:

```bash
# check for updates
sudo apt-get update

# install dependencies
sudo apt-get install libssl-dev
sudo apt-get install pkg-config
```

> In case if you are running OSX, refer to this [discussion](https://github.com/hyperium/hyper/issues/935) to install necessary dependencies.

## Setting up a project
Now we can finally start working on the project. First, we need to create a template application using [cargo](http://doc.crates.io/guide.html).

```bash
# `--bin` flag tells cargo that we want to
# create a binary project, not a library.
cargo new hexocat-bot --bin
```

In order to call Github APIs, we need to wire in a new dependency – Anterofit. Anterofit is a library which allows to gracefully consume REST APIs by declaring services as traits, which encapsulate all necessary information about the target endpoint. It has integration with the most popular serialization library for Rust called Serde, which makes parsing JSON a breeze.

In order to compile Anterofit as a part of the project, we need to modify the  `Cargo.toml` file.

```toml
[package]
name = "hexocat-bot"
version = "0.1.0"
authors = ["Araz Abishov <araz.abishov.gsoc@gmail.com>"]

[dependencies]
anterofit = "0.1.1"

# serialization library
serde = "0.9"
serde_json = "0.9"
serde_derive = "0.9"
```

The `Cargo.toml` file encloses information about the project, including metadata about the name, version and the authors. Dependencies are declared under the block with the same name. Since we are going to use Anterofit in conjunction with Serde, we also need to explicitly declare a dependency on it.

## Working with GitHub endpoints
Since we want to search only by repositories, we are going to work with one [endpoint](https://developer.github.com/v3/search/#search-repositories). Here is an example of the search query:

```bash
curl https://api.github.com/search/repositories?q=retrofit&per_page=10
```

We have specified two query parameters:
 - `q` - repository we are looking for
 - `per_page` - limiting the size of the page

In order to be able to parse and map the response body to `struct`s, we first need to declare them within the `main.rs` file. Each response from GitHub is wrapped into a model which provides useful metadata to clients, like a total count of search hits, completeness of results and actual search items. Each result item is a repository, which also contains information about the hosting organization. In order to keep this example lean, we are going to declare only the properties which we need.

```rust
extern crate serde;
extern crate serde_json;

#[macro_use]
extern crate serde_derive;

#[derive(Deserialize)]
struct Owner {
    login: String,
    html_url: String
}

#[derive(Deserialize)]
struct Repository {
    name: String,
    html_url: String,
    description: String,
    owner: Owner
}

#[derive(Deserialize)]
struct SearchResult {
    items: Vec<Repository>
}
```

As you might have noticed, there is an `attribute` specified for each of the models - `#[derive(Deserialize)]`. This way we tell Serde that we want to generate code for deserializing this model.

Now we can jump in and declare some services. This is an example of how GitHub's search endpoint can be represented as an Anterofit service:

```rust
#[macro_use]
extern crate anterofit;

service! {
    trait GitHubService {
        fn search(&self, q: String, p: u32) -> SearchResult {
            GET("/search/repositories");
            query!{ "q" => q, "per_page" => p }
        }
    }
}
```

A trait declaration is placed within Anterofit's `service` macro. Later on during compilation phase, Rust's compiler will generate the actual implementation of the specified trait. Since we work with a single endpoint, there is only one `search` function declared. It returns a `SearchResult` instance and takes in two parameters: a search keyword and a page size.

The body of search function consists of the two parts:
- The `GET` function invocation specifies both the HTTP verb and takes in a string which represents a relative path to the endpoint.
- The `query` macro maps the query parameter names to the arguments of the `search` function.

Now let's take a look at how to initialize and consume the service we have just defined.

```rust
use anterofit::{Adapter, Url};
use anterofit::net::intercept::AddHeader;
use useragent::UserAgentHeader;

mod useragent;

fn prepare_response_body(repos: Vec<Repository>) -> String {
    return repos.iter()
        .map(|repo| format!("{0} by {1}: {2}",
            repo.name, repo.owner.login, repo.html_url))
        .collect::<Vec<String>>()
        .join("\n");
}

fn main() {
    let service = Adapter::builder()
        .base_url(Url::parse("https://api.github.com").unwrap())
        .interceptor(AddHeader(UserAgentHeader("hexocat-bot".to_string())))
        .serialize_json()
        .build();

    let response = match service.search("linux".to_string(), 10).exec().block() {
        Ok(result) => prepare_response_body(result.items),
        Err(error) => "Oops, something went wrong.".to_string()
    };

    println!("{}", response);
}
```

The GitHub service is initialized through Anterofit's Adapter, which requires us to specify the parameters listed below:

 - `base_url` - base url which will be appended to the relative path of the GitHubService
 - `interceptor` - a powerful abstraction which allows clients to modify requests / responses flowing through Anterofit. Here it is used to add a `UserAgent` header, which is required by GitHub API. If you want to take a closer look at `UserAgent` header implementation, here is a complete [example](https://github.com/ArazAbishov/hexocat-bot/tree/post-part-1).
 - `serialize_json` - flags Anterofit that we want to parse JSON within request or response body.

Finally, invoking `build()` will return an instance of the GitHubService. Now searching GitHub is as easy as calling any function in Rust. In sake of simplicity, the search keyword and page size parameters are hardcoded. After a successful call to the endpoint, the service will return an instance of the `SearchResult` struct, which in a turn will be 'prettified' by the `prepare_response_body` function.

Now we can finally compile and execute the app by running `cargo run` and see the output in the terminal.

## Passing arguments dynamically
The app with the hardcoded search keyword is not very useful. In order to let user to specify it dynamically, we are going to use command line arguments.

```rust
use std::env;

fn main() {
    // When running app through cargo, the first argument
    // is a path to the binary being executed. Hence, if repository
    // name is provided, the argument count must be at least two.
    if env::args().count() < 2 {
        println!("Please, specify repository name you would like to find.");
        return;
    }

    // Extract the last argument as a search keyword.
    let repository = env::args().last().unwrap();

    // Building an instance of GitHubService.
    let service = Adapter::builder()
        .base_url(Url::parse("https://api.github.com").unwrap())
        .interceptor(AddHeader(UserAgentHeader("hexocat-bot".to_string())))
        .serialize_json()
        .build();

    let response = match service.search(repository.to_string(), 10).exec().block() {
        Ok(result) => prepare_response_body(result.items),
        Err(error) => "Oops, something went wrong.".to_string()
    };

    println!("{}", response);
}
```

In order to make sure that the repository name is provided, there is an argument count check right in the beginning of the main function. In case the user has specified at least one argument, it will be used as a search keyword:

```bash
# running app using cargo
# note that application arguments must be passed after --
cargo run -- retrofit
```

In case you want to execute binary directly:

```bash
# navigate to the folder with the executable
cd target/debug

# run it
./hexocat-bot retrofit
```

## Wrapping up
In this blog post we have learned how to consume REST APIs using Anterofit. You can find the source code of hexocat-bot on [GitHub](https://github.com/ArazAbishov/hexocat-bot/tree/post-part-1). In the next part, we are going to use Rocket to serve requests from Slack.
