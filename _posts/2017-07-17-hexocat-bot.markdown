---
layout: post
title: >
  Building a simple chatbot for slack in Rust using Rocket and Anterofit frameworks
date: 2017-07-17 22:00:00 +0000
---

Rust is a systems programming language which enables developers to write safe and fast code without sacrificing high-level language constructs. At first it seems that Rust is targeting only performance critical use cases, but original intention is far more ambitious. Frameworks like Rocket, Serde and Anterofit make Rust a good fit for web application development as well.

This blogpost is all about building a simple slack bot, which allows to search github repositories. Obviously, you don't need to develop a simple chatbot in systems programming language, but intention of this blogpost is to show how powerful Rust is. The whole implementation is less 150 lines code, which is quite amazing.

## Preparing development environment
One of the dependencies - Rocket, depends on the nightly version of Rust compiler. We will need to make sure that it is installed and configured correctly.

In case if you don't have `rustup` installed on your machine, execute following command:

```bash
curl https://sh.rustup.rs -sSf | sh

# source environment variables after installation
source $HOME/.cargo/env
```

No we are going to switch to nightly compiler and set it as default:

```bash
# checking for updates
rustup update

# install nightly compiler
rustup install nightly

# set nightly compiler as default one
rustup default nightly
```

Remember that you always can switch back to the stable compiler by executing `rustup default stable`.

Hyper – a HTTP library which is used both by Rocket and Anterofit, has a dependency on OpenSSL native libraries which we need to install in the system:

```bash
# check for updates
sudo apt-get update

# install dependencies
sudo apt-get install libssl-dev
sudo apt-get install pkg-config
```

## Setting up a project
Now we finally can start working on the project. First, we need to create a skeleton application using `cargo`.

```bash
# `--bin` flag tells cargo that we want to
# create a binary project, not a library.
cargo new hexocat-bot --bin
```

In order to be able to call Github APIs, we need to wire in a new dependency – Anterofit. Anterofit is a library which allows to gracefully consume REST APIs by declaring services as traits, which encapsulate all necessary information about target endpoint. It has integration with the most popular serialization library for Rust called Serde, which makes parsing Json a breeze.

In order to compile Anterofit as a part of the project, we need to modify `Cargo.toml` file.

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

`Cargo.toml` file encloses information about the project, including metadata about the name, version and authors. Dependencies are declared under the block of the same name. Since we are going to use Anterofit in conjunction with Serde, we also need to explicitly declare dependency on it.

## Working with GitHub endpoints
Since we want to search only by repositories, we are going to work with one [endpoint](https://developer.github.com/v3/search/#search-repositories). Here is an example of the search query:

```bash
curl https://api.github.com/search/repositories?q=retrofit&per_page=10
```

We have specified two query parameters:
 - `q` - repository we are looking for
 - `per_page` - limiting the size of the page

In order to be able to parse and map response body to `struct`s, we first need to declare them. Each response from GitHub is wrapped into a model which provides useful metadata to clients, like a total count of search hits, completeness of results and actual search items. Each result item is a repository, which also contains information about hosting organization. In order to keep this example lean, we are going to declare only those properties which we need.

```rust
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

If you have noticed, there is an `attribute` specified for each of the models - `#[derive(Deserialize)]`. This way we tell serde that we want to generate code for deserializing this model.

Now we can jump in and declare some services. Here is an example of how GitHub's search endpoint can be represented as Anterofit service:

```rust
...

service! {
    trait GithubService {
        fn search(&self, q: String, p: u32) -> SearchResult {
            GET("/search/repositories");
            query!{ "q" => q, "per_page" => p }
        }
    }
}

...
```

A trait declaration is placed within Anterofit's `service` macro. Later on during compilation phase, Rust's compiler will generate actual implementation of the specified trait. Since we need to work with only one endpoint, there is only one `search` function declared. It returns `SearchResult` instance and takes in two parameters: a search keyword and a page size.

The function body invokes two macros, where the first one is used to specify HTTP verb, while the second one is used for processing query parameters. `GET` macro both specifies HTTP verb and takes in a string which represents a relative path to the endpoint. `query` macro maps query parameter names to the arguments of the `search` function.

Now we can take a look on how to consume the service we have just defined. For now let's assume that `GitHubService` has already been initialized somewhere.

```rust

```

// write about initialization of the service
// write about userheader implementation

## Structure   
 - Requirements / dependencies
 - Anterofit (write a simple query to the GitHub APIs)
 - Rocket (use for exposing to slack)
 - Using ngrok for development with slack

## Future
 - Deploying and managing the bot on Ubuntu under nginx
 - Deploying the bot on RaspberryPi
