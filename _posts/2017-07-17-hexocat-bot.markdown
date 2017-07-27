---
layout: post
title: >
  Implementing slash command for Slack in Rust, Rocket and Anterofit - Part 1
date: 2017-07-17 22:00:00 +0000
---

Rust is a systems programming language which enables developers to write safe and fast code without sacrificing high-level language constructs. At first it seems that Rust is targeting only performance critical use cases, but original intention is far more ambitious. Frameworks like Rocket, Serde and Anterofit make Rust a good fit for the web application development as well.

This blogpost is all about building a simple slack bot, which allows to search github repositories. Obviously, you don't need to develop a simple chatbot in systems programming language, but intention of this blogpost is to show how powerful Rust is. The whole implementation is less 150 lines code, which is quite amazing.

In order to make this blog post self-sufficient, we will build a simple command line utility for searching repositories. In the second part, command line app will be turned into a small server built on top of the `Rocket` framework.

- bla bla about Rust
- scope of the blogpost
- slowly shift to the actual topic

## Preparing development environment
- mention usage of vagrant
-

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

In order to be able to call Github APIs, we need to wire in a new dependency – Anterofit. Anterofit is a library which allows to gracefully consume REST APIs by declaring services as traits, which encapsulate all necessary information about target endpoint. It has integration with the most popular serialization library for Rust called Serde, which makes parsing JSON a breeze.

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
- models, serialization / deserialization
- defining github service
- user agent header
- preparing body
- outputting to console
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
service! {
    trait GitHubService {
        fn search(&self, q: String, p: u32) -> SearchResult {
            GET("/search/repositories");
            query!{ "q" => q, "per_page" => p }
        }
    }
}
```

A trait declaration is placed within Anterofit's `service` macro. Later on during compilation phase, Rust's compiler will generate actual implementation of the specified trait. Since we need to work with only one endpoint, there is only one `search` function declared. It returns `SearchResult` instance and takes in two parameters: a search keyword and a page size.

The function body invokes two macros, where the first one is used to specify HTTP verb, while the second one is used for processing query parameters. `GET` macro both specifies HTTP verb and takes in a string which represents a relative path to the endpoint. `query` macro maps query parameter names to the arguments of the `search` function.

Now let's take a look on how to initialize and consume the service we have just defined.

```rust
fn main() {
    // When running app through cargo, the first argument
    // is a path to the binary being executed. Hence, if repository
    // name is provided, the argument count must be at least two.
    if env::args().count() < 2 {
        println!("Please, specify repository name you would like to find.");
        return;
    }

    let service = Adapter::builder()
        .base_url(Url::parse("https://api.github.com").unwrap())
        .interceptor(AddHeader(UserAgentHeader("hexocat-bot".to_string())))
        .serialize_json()
        .build();

    let repository = env::args().last().unwrap();
    let response = match service.search(repository.to_string(), 10).exec().block() {
        Ok(result) => prepare_response_body(result.items),
        Err(error) => "Oops, something went wrong.".to_string()
    };

    println!("{}", response);
}
```

In order to make sure that repository name is provided, there is an argument check right in the beginning of the main function. Right after it, there is an initialization of the GitHubService using Anterofit's Adapter.

 - `base_url` - base url which will be appended to relative path of the GitHubService
 - `interceptor` - a powerful abstraction which allows clients to modify requests / responses flowing through Anterofit. Here it is used to add a `UserAgent` header, which is required by GitHub API.
 - `serialize_json` - flags Anterofit that we want to parse / serialize JSON within request / response body.

 After invoking `.build()`, we will get an instance of the GitHubService. The repository name from arguments is passed as a search keyword to the service, along with the page size parameter. Resulting `SearchResult` payload is processed by `prepare_response_body` function which prettifies output.

 ```rust
 fn prepare_response_body(repos: Vec<Repository>) -> String {
     return repos.iter()
         .map(|repo| format!("{0} by {1}: {2}",
             repo.name, repo.owner.login, repo.html_url))
         .collect::<Vec<String>>()
         .join("\n");
 }
 ```

## Interacting with application (CLI arguments)

## Wrapping up
Mention what has been done. Mention what you're going to write about in future.

## Future
 - Serving requests with Rocket (using ngrok for development)
 - Deploying and managing the bot on Ubuntu under nginx
 - Deploying the bot on RaspberryPi

## ToDo
 - Prepare source code on GitHub (including ReadMe and probably CI)
 - Possibly write some tests for the app
