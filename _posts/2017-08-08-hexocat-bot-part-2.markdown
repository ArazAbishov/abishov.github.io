---
layout: post
title: >
  Implementing a bot for Slack in Rust, Rocket and Anterofit - Part 2
date: 2017-08-08 15:00:00 +0000
---

In the [previous blog post]({% post_url 2017-07-27-hexocat-bot-part-1 %}) we have learned how to make calls to the GitHub APIs using Anterofit. In this part, we will focus on using the Rocket framework for serving requests, as well as using ngrok for exposing the hexocat-bot to the internet.

## Integrating Rocket
The first step is to add Rocket as a dependency. Open the `Cargo.toml` file and append following lines:

```toml
# web framework
rocket = "0.3.0"
rocket_codegen = "0.3.0"
```

Next, we have to add the `Rocket.toml` configuration file that specifies custom address, port and logging level:

```toml
[development]
address = "0.0.0.0"
log = "normal"
port = 2727

[staging]
address = "0.0.0.0"
log = "normal"
port = 2727

[production]
address = "0.0.0.0"
log = "critical"
port = 2727
```

In order to be able to handle requests from Slack, we will need to declare new models. All the properties which are sent and received are described in the Slack API [documentation](https://api.slack.com/slash-commands). The `SlackRequest` and `SlackResponse` models contain only those properties which are used in the bot:

```rust
#![feature(plugin, custom_derive)]
#![plugin(rocket_codegen)]

extern crate rocket;

// Encapsulates response body which is sent to
// Slack after searching repositories on GitHub.
#[derive(Serialize)]
struct SlackResponse {
    text: String,
    response_type: String,
}

// Struct that contains properties
// which Slack sends to the hexocat-bot.
#[derive(FromForm)]
struct SlackRequest {
    text: String,
    token: String
}
```

`#[derive(FromForm)]` attribute will signal Rocket to generate code for mapping `FormUrlEncoded` properties into the model instance.  

## Serving requests
Now we can have some fun. Let's declare an endpoint that Slack is going to call:

```rust
use std::io::Cursor;
use rocket::request::LenientForm;
use rocket::response::Response;
use rocket::http::{ContentType, Status};
use rocket::config::{self, ConfigError, Environment};

// Accepts a message body and embeds it into the SlackResponse
// instance. The latter one is wrapped into Response and
// returned to the caller.
fn prepare_response(text: String) -> Response<'static> {
    let body = serde_json::to_string(&SlackResponse {
        text: text,
        response_type: "in_channel".to_string()
    }).unwrap();

    return Response::build()
        .status(Status::Ok)
        .header(ContentType::JSON)
        .sized_body(Cursor::new(body))
        .finalize();
}

#[post("/", data = "<form_request>")]
fn hexocat(form_request: LenientForm<SlackRequest>) -> Response<'static> {
    return prepare_response("Yay, we got first response served.".to_string());
}

fn main() {
    rocket::ignite().mount("/hexocat/", routes![hexocat]).launch();
}
```

The `hexocat` function is a handler which will be called by Rocket when a request hits the server with the matching route and parameters. The `#[post("/", data = "<form_request>")]` attribute encapsulates the HTTP verb, route to the handler and the name of the `LenientForm` parameter.

The `prepare_response` function takes in the `text` parameter, which is a message that will be shown to the Slack user. The hardcoded `in_channel` response type signals Slack that message has to be shown to the all participants of the channel. The resulting `SlackResponse` instance is converted into the JSON string and set as a response body.

A careful reader will notice that implementation of the main function has changed significantly. There is no logic for processing command line arguments anymore. Initialization of the Anterofit's adapter now takes place within the `hexocat` function. The only responsibility of the main function is to initialize and mount the `hexocat` handler.

Now you can `cargo run` the server and execute a request against it. Note, that you have to supply text and token properties within the FormUrlEncoded body of the HTTP request. You can use postman or curl for testing the bot. Example of the curl request:

```bash
curl -X POST \
  http://0.0.0.0:2727/hexocat/ \
  -H 'content-type: application/x-www-form-urlencoded' \
  -d 'text=retrofit&token=test_token'
```

## Integrating the GitHub service
Since now we can consume requests from Slack, it is time to search repositories on GitHub! First, there is a check if a user has specified the repository to search. In case if not, the corresponding error message is returned. Following lines are taken from the part one implementation, where the GitHub service is initialized and invoked. The difference is that the search result now is wrapped into `Response` instance and returned to the client instead of the standard output.

```rust
#[post("/", data = "<request>")]
fn hexocat(request: LenientForm<SlackRequest>) -> Response<'static> {
    // Extract the SlackRequest instance
    // from the LenientForm wrapper.
    let slack_request = request.get();

    // In case if Slack request doesn't contain a search query,
    // return a meaningful message back to the user.
    if slack_request.text.trim().is_empty() {
        return prepare_response("Specify repository name to search. \
                For example: /hexocat linux".to_string());
    }

    // Construct an instance of the GitHub service.
    let service = Adapter::builder()
        .base_url(Url::parse("https://api.github.com").unwrap())
        .interceptor(AddHeader(UserAgentHeader("hexocat-bot".to_string())))
        .serialize_json()
        .build();

    // Query GitHub API and transform the response into human readable message.
    let repository = slack_request.text.to_lowercase().to_string();
    let response_body = match service.search(repository, 10).exec().block() {
        Ok(result) => prepare_response_body(result.items),
        Err(_) => "Oops, something went wrong.".to_string()
    };

    // Wrap the message into response and
    // return it to the client.
    return prepare_response(response_body);
}
```

Let's try to search something using curl:

```bash
curl -X POST \
  http://0.0.0.0:2727/hexocat/ \
  -H 'content-type: application/x-www-form-urlencoded' \
  -d 'text=retrofit&token=test_token'
```

If you see the search result, we made it! Since Slack [supports markdown](https://api.slack.com/docs/message-formatting), let's prettify the response message by updating the `prepare_response_body` function:

```rust
// Takes in a vector of Repositories and formats them
// using markdown supported by Slack.
fn prepare_response_body(repos: Vec<Repository>) -> String {
    return repos.iter()
        .map(|repo| format!("<{0}|{1}> by <{2}|{3}>\n{4}\n----", repo.html_url,
                repo.name, repo.owner.html_url, repo.owner.login, repo.description))
        .collect::<Vec<String>>()
        .join("\n\n");
}
```

## Guarding against requests from unknown sources
Since hexocat-bot will be exposed to the internet, there must be a mechanism which prevents unknown clients from calling it. Each Slack application has a unique token assigned to it, which is also included in every request that Slack is issuing against the target app. So our job is to match the token from the incoming request with the one provided by the host machine. In case if they do not match, the bot will respond with the Forbidden (403) status.

Let's define a function which verifies the incoming token. There is no need to perform any checks if the hexocat bot is running in the development environment.

```rust
// Returns true when running in the development environment
// or when incoming token matches the one in the configuration.
fn check_access(config: &Configuration, token: String) -> bool {
    return match config.environment {
        Environment::Development => true,
        Environment::Staging => config.token.eq(&token),
        Environment::Production => config.token.eq(&token)
    };
}
```

Next, we need a way to consume configuration parameters which are passed to the Rocket server from environment. In order to be able to access them later, we first need to store those parameters within the application state using [managed state](https://rocket.rs/guide/state/#managed-state):

```rust
use rocket::fairing::AdHoc;
use rocket::config::Environment;

// Encapsulates useful properties which are
// provided by Rocket on start (see main function).
struct Configuration {
    // Enum value which represents Environment
    // type: Development, Staging, or Production.
    environment: Environment,

    // Token that is used to verify Slack
    // requests coming from the internet.
    // This property is used only on Staging and
    // Production environments.
    token: String
}

// Updated main function..
fn main() {
    rocket::ignite()
        .attach(AdHoc::on_attach(|rocket| {
            // Rocket Config is passed here as a part
            // of the application state.
            // It does contain properties which we
            // will use later, like environment and token.
            let config = rocket.config().clone();
            return Ok(rocket.manage(Configuration {
                environment: config.environment,
                token: config.get_str("key").unwrap_or("").to_string()
            }));
        }))
        .mount("/hexocat/", routes![hexocat])
        .launch();
}
```

The stored state is consumed through the Rocket handlers, hence we have to update the `hexocat` function to accept a `State` instance:

```rust
use rocket::State;

// The 'allow' attribute here is just to silence the
// Rocket warning. The State<Configuration> is managed
// properly during rocket start.
#[allow(unmanaged_state)]
#[post("/", data = "<request>")]
fn hexocat(request: LenientForm<SlackRequest>, s: State<Configuration>) -> Response<'static> {
    // Extract the SlackRequest instance
    // from the LenientForm wrapper.
    let slack_request = request.get();

    // If access is not granted, return 403.
    if !check_access(s.inner(), slack_request.token.to_owned()) {
        return Response::build()
            .status(Status::Forbidden)
            .finalize();
    }

    // In case if Slack request doesn't contain a search query,
    // return a meaningful message back to the user.
    if slack_request.text.trim().is_empty() {
        return prepare_response("Specify repository name to search. \
                For example: /hexocat linux".to_string());
    }

    // Construct an instance of the GitHub service.
    let service = Adapter::builder()
        .base_url(Url::parse("https://api.github.com").unwrap())
        .interceptor(AddHeader(UserAgentHeader("hexocat-bot".to_string())))
        .serialize_json()
        .build();

    // Query GitHub API and transform the response into human readable message.
    let repository = slack_request.text.to_lowercase().to_string();
    let response_body = match service.search(repository, 10).exec().block() {
        Ok(result) => prepare_response_body(result.items),
        Err(_) => "Oops, something went wrong.".to_string()
    };

    // Wrap the message into response and
    // return it to the client.
    return prepare_response(response_body);
}
```

Now if you want to run a server in a production environment you have either to change `Rocket.toml` file to include extra properties or to expose them as environment variables:

```bash
# Prepare release version of the binary.
cargo build --release

# Export required environment variables.
export ROCKET_ENV=production cargo run
export ROCKET_KEY=your_slack_token
export ROCKET_PORT=2727

# Execute the binary to start server.
target/release/hexocat-bot
```  

> Unfortunately, we can't use the Rocket's [request guards](https://rocket.rs/guide/requests/#request-guards). Primarily because request guards are working only with HTTP headers, while Slack is sending token within HTTP body.

## Using ngrok for development
If we want to try out our bot through Slack, we first have to expose it to the internet. There is a great tool for this - [ngrok](https://ngrok.com/). All we have to do is to run the bot and point ngrok to the port it is running on:

> In case if you want to learn more about ngrok, check out this [tutorial](https://api.slack.com/tutorials/tunneling-with-ngrok) from Slack.

```bash
# Run the development version.
cargo run

# Bind ngrok to the port 2727.
./ngrok http 2727
```

The ngrok invocation will provide you a URL to which you will have to append `hexocat/` path in order to access the bot. The resulting URL will look something like this: [https://0658d623.ngrok.io/hexocat/](https://0658d623.ngrok.io/hexocat/). The last step is to create the slash command app on Slack with the given URL and try searching some GitHub repositories! You can find more information on creating a new slash command in the Slack [documentation](https://api.slack.com/slash-commands).

## Wrapping up
In this blog post we have integrated Rocket for serving requests from Slack, as well as the GitHub service from [part one]({% post_url 2017-07-27-hexocat-bot-part-1 %}). You can find the source code of the hexocat bot on [GitHub](https://github.com/ArazAbishov/hexocat-bot/tree/post-part-1). In the next part we are going to deploy the Slack behind the NGINX proxy server on RaspberryPi. 
