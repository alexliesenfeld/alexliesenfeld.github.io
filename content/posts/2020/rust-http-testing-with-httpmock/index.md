+++
author = "Alexander Liesenfeld"
title = "Rust HTTP Testing with httpmock"
date = "2020-03-05"
description = "Rust HTTP Testing with httpmock"
tags = [
    "rust",
    "http",
    "mock",
    "test"
]
+++

HTTP mocking libraries allow you to simulate HTTP responses so you can easier test code that depends on third-party APIs. This article shows how you can use [httpmock](https://github.com/alexliesenfeld/httpmock) to do this in Rust.

# Why HTTP-Mocking?
At some point, every developer had the need to test calls to external APIs. This is especially important in (micro)service-oriented environments where services typically require access to several external dependencies, such as APIs, authentication providers, data sources, etc. These services are not always available to you. This is where mocking/stubbing tools can fill the gaps.

HTTP mocking libraries usually provide you a simple HTTP server and tools to configure it for custom request/response scenarios. We will see examples of such scenarios in the following sections.

# The App
Let's suppose we are building a Rust app that will create a Github repository on our behalf. To perform these operations, we will use the Github REST API. We will then write some automated tests to verify correct behavior by simulating HTTP responses using `httpmock`.

## Let's start!
Let's first create a new cargo package for our app and name it `github_api_client`:
```sh
cargo new github_api_client --bin
```

We'll also need some libraries, so let's add them to Cargo.toml. We'll use 
- `isahc` as our HTTP client library,
- `serde_json` for easy JSON serialization and deserialization, and 
- `custom_error` to create custom error types easily.

Cargo.toml:
```toml
[dependencies]
isahc = { version = "0.9.8", features = ["json"] }
serde_json = "1.0"
anyhow = "1.0.34"
```

## The Client

Now let's write some code that will allow us to access the Github REST API. We will create a structure named `GithubClient` that will contain all the logic required to access the Github API.

```rust
use isahc::{RequestExt, ResponseExt, prelude::Request};
use serde_json::{json, Value};
use anyhow::{Result,ensure};

pub struct GithubClient {
    base_url: String,
    token: String,
}

impl GithubClient {
    pub fn new(token: &str, base_url: &str) -> GithubClient {
        GithubClient { base_url: base_url.into(), token: token.into() }
    }

    pub fn create_repo(&self, name: &str) -> Result<String> {
        let mut response = Request::post(format!("{}/user/repos", self.base_url))
            .header("Authorization", format!("token {}", self.token))
            .header("Content-Type", "application/json")
            .body(json!({ "name": name, "private": true }).to_string())?
            .send()?;

        let json_body: Value = response.json()?;

        ensure!(response.status().as_u16() == 201, "Unexpected status code");
        ensure!(json_body["html_url"].is_string(), "Missing html_url in response");

        return Ok(json_body["html_url"].as_str().unwrap().into());
    }
}

fn main() {
    let github = GithubClient::new("<github-token>", "https://api.github.com");
    let url = github.create_repo("myRepo").expect("Cannot create repo");
    println!("Repo URL: {}", url);
}
```

The only method our client provides is `create_repo`. It takes the repository name as an argument and returns a `Result` containing the repository URL as a string value. To keep this example simple, we use the `anyhow` crate for generic error handling.

## The Problem

Now that we have a functional app, we need to write some tests to make sure it doesn’t have any obvious errors.

The tricky part is to find a good target for mocking so we can test the client behavior in different scenarios. In our case, the HTTP client (such as the `Request::post` method in our client implementation) looks like a good place to start.

Unfortunately, mocking HTTP clients is very tedious. This is because in a larger application we would need to reimplement a big chunk of the HTTP clients API to be able to simulate request/response scenarios. So what to do?

## The Solution

To test our Github API client conveniently, we can use an HTTP mocking library. Such libraries can help us verify that HTTP requests sent by our client are correct and allow us to simulate HTTP responses from the Github API.

At the time of writing there are 4 noteworthy Rust libraries that can help us with this:

- `mockito`
- `httpmock`
- `httptest`
- `wiremock`.

The following comparison matrix shows how the libraries compare to each other:

| Library     | Execution  | Custom Matchers | Mock- able APIs | Sync API | Async API | Stand-alone Mode 
| ----------- | ---------- | ------- | ---------------- | --------------- | --------------- | -------- |
| mockito     | serial     | no              | 1             | yes      | no        | no              
| **httpmock**    | **parallel**   | **yes**             | **∞**    | **yes**      | **yes**       | **yes**             
| httptest    | parallel   | yes             | ∞     | yes      | no        | no              
| wiremock | parallel   | yes             | ∞     | no       | yes       | no              

According to the comparison matrix the most complete package is currently provided by httpmock. For this reason, we will use this one for the rest of this article (and also because I am the developer 😜).

# Creating Mocks

In this section, we'll write some tests to verify our Github API client works as expected. Let’s first add the `httpmock` crate to our Cargo.toml:
```rust
[dev-dependencies]
httpmock = "0.5"
```

Now we’re all set. Let’s create a test that will make sure “the good path” in our client implementation works as expected:

{{< highlight rust "linenos=table" >}}
#[cfg(test)]
mod tests {
    use crate::GithubClient;
    use httpmock::MockServer;
    use serde_json::json;

    #[test]
    fn create_repo_success_test() {
        // Arrange
        let server = MockServer::start();
        let mock = server.mock(|when, then| {
            when.method("POST")
                .path("/user/repos")
                .header("Authorization", "token TOKEN")
                .header("Content-Type", "application/json");
            then.status(201)
                .json_body(json!({ "html_url": "http://example.com" }));
        });
        let client = GithubClient::new("TOKEN", &server.base_url());

        // Act
        let result = client.create_repo("myRepo");

        // Assert
        mock.assert();
        assert_eq!(result.is_ok(), true);
        assert_eq!(result.unwrap(), "http://example.com");
    }
}
{{< / highlight >}}

To easier grasp what this test does, we arranged it following the [AAA (Arrange-Act-Assert) pattern](https://medium.com/@pjbgf/title-testing-code-ocd-and-the-aaa-pattern-df453975ab80) (look at the comments).

#### Arrange

The first step in our test function is to create a `MockServer` instance (line 10). Next, we create a `Mock` object on the `MockServer` with all our request and response requirements (lines 11–18). 

Notice how we used the `when` variable to define HTTP request requirements (lines 12-15). We used the `then` variable to define what data the corresponding HTTP response will contain (lines 16–17). 

The mock server will only respond as we specified if it receives an HTTP request that meets all the request criteria from the `when` part. Otherwise, it will respond with an error message and HTTP status code `404`.

*Important*: Observe how we set the base URL in our client to point to the mock server instead of the real Github API (line 19).

#### Act

Next, we trigger the method that is under test (line 22) which is the method `create_repo` from our `GithubClient` structure.

#### Assert

At last, we use the `assert` method provided by the mock object (line 25). This method makes sure that the mock server received exactly one HTTP request that matched all the mock requirements. If not, it will fail the test with a detailed problem description (see next section).

## Verification

`Mock` objects provide an `assert` method which ensures our app did actually send a request to the mock server and that it matched the mock spec (the `when` part). If not, this method will fail the test. In this case `httpmock` will try to find a request in its request journal that is most similar to the mock spec. It will then identify the differences between the two so you can easily spot unexpected values. 

To demonstrate this feature, we will modify our client to send `text/plain` in the `content-type` header. If we rerun the test, we'll see that this change is detected and the test fails now with the following message:

```
At least one request has been received, but none exactly matched the mock specification.
Here is a comparison with the most similar request (request number 1): 
1 : Expected header with name 'Content-Type' and value 'application/json' to be present in the request but it wasn't.
------------------------------------------------------------------------------------------
Expected:                [key=equals, value=equals]   Content-Type=application/json
Actual (closest match):                               content-type=text/plain
```

Depending on the IDE you are using, you will also be able to see the differences between the expected and actual value in a differences viewer. For example, this is how it would look like in IntelliJ or CLion:

{{< figure src="./intellij.jpeg" caption="Differences viewer in IntelliJ or CLion." width="600px">}}


# Conclusion

This article showed how `httpmock` can be used to test HTTP-based API clients in Rust. On one hand, it allowed us to verify that our app is sending HTTP requests which contain all the required information. On the other hand, we could simulate HTTP responses to see if our app behaves as expected.

You can find the source code from this article [on Github](https://github.com/alexliesenfeld/blog-rust-http-testing-with-httpmock).
