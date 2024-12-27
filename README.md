# Local AWS Lambda environments (with Go)

> [!NOTE]
> This document is a DRAFT, and is IN-PROGRESS. It will be updated over time as we learn more.

## Overview

AWS has open-sourced their [AWS Lambda runtimes as Docker images](https://github.com/aws/aws-lambda-base-images). However, in typical AWS fashion, they "open-sourced" them [in a broken state](https://gallery.ecr.aws/lambda/provided). So we've had to fork and patch to get them to work outside of Amazon as they should.

Since we work with Go, we primarily care about the `provided.al2023` runtime. So [this is what we're patching and rebuilding](https://github.com/northwood-labs/lambda-provided-al2023).

However, the runtime alone is not enough to make local Lambda functions work the way you expect.

There is also a way to do this with the AWS SAM CLI, however (a) I'm not a fan of AWS SAM, and (b) this can be done with normal Docker Desktop, which will help you understand the pieces better and not be locked into a single vendor's local tooling. (In my experience, AWS has a habit of breaking things and neither noticing nor responding to issues people post in their GitHub repos.)

## Dockerfile

While you could do this in all sorts of different ways (e.g., Kubernetes, Podman, `nerdctl`, AWS SAM), I chose to solve the local Lambda runtime with [Docker Compose] running in [Docker Desktop] since they're good for local development.

I started with a [multi-stage Dockerfile](https://docs.docker.com/build/building/multi-stage/).

In the first stage, we download the _[AWS Lambda Runtime Interface Emulator](https://github.com/aws/aws-lambda-runtime-interface-emulator)_ (RIE), compiled for Linux and our current CPU architecture. In the second stage, we will put this in front of our Lambda executable that we've written ourselves.

### Downloading Runtime Interface Emulator

In our Dockerfile, our first stage leverages [download-asset](https://github.com/northwood-labs/download-asset) to download the correct GitHub release asset for our current CPU architecture.

> [!TIP]
> [download-asset](https://github.com/northwood-labs/download-asset) **requires** a valid `$GITHUB_TOKEN` environment variable in order to raise the GitHub rate limit. [Create a new Personal Access Token](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/managing-your-personal-access-tokens#creating-a-personal-access-token-classic), no scopes required (we only need to _authenticate_ you). We do this **securely** using [Docker Secrets](https://docs.docker.com/compose/how-tos/use-secrets/), instead of **insecurely** using `--build-args` or plain-text secrets in source code. We show the Docker Compose definition later in this document.

```Dockerfile
# syntax=docker/dockerfile:1
FROM golang:1-alpine AS go-installer

RUN go install github.com/northwood-labs/download-asset@latest
RUN --mount=type=secret,id=github_token \
    GITHUB_TOKEN="$(cat /run/secrets/github_token)" \
    download-asset get \
        --owner-repo aws/aws-lambda-runtime-interface-emulator \
        --tag latest \
        --intel64 x86_64 \
        --arm64 arm64 \
        --pattern 'aws-lambda-rie-{{.Arch}}' \
        --write-to-bin aws-lambda-rie \
    ;

# Rename `aws-lambda-rie-arm64` or `aws-lambda-rie-x86_64` to a platform-neutral `aws-lambda-rie`.
RUN mv /usr/local/bin/aws-lambda-rie* /usr/local/bin/aws-lambda-rie
```

### Identify the SHA digest of the Docker image

It is **always** more secure to refer to a remote Docker image by SHA digest than by tag. This is because a SHA digest is _immutable_, and cannot be changed after-the-fact like a Docker tag can.

It's a _little_ more work for **a lot** more security.

We want to pull the SHA digest for the `:latest` tag on the `ghcr.io/northwood-labs/lambda-provided-al2023` image. This is a [multi-platform image](https://docs.docker.com/build/building/multi-platform/) that has an _Intel64_ version and an _ARM64_ version.

> [!TIP]
> [View the GitHub Actions workflow](https://github.com/northwood-labs/lambda-provided-al2023/blob/main/.github/workflows/build-and-push.yml) which constructs this multi-platform Docker image from AWS source code.

```bash
docker pull ghcr.io/northwood-labs/lambda-provided-al2023:latest
docker images --digests ghcr.io/northwood-labs/lambda-provided-al2023 --format '{{ .Digest }}'
```

This gave me a result of `sha256:2b947c7c1e18392ce6b1b311ba1715a9b043a6fb5bb6572e914764e946321382`. So we'll use this instead of `:latest`. Over time, as `:latest` is updated, it will point to a different SHA digest. But this command will get you the current value.

### Downloading the AWS Lambda `provided.al2023` image

```Dockerfile
# syntax=docker/dockerfile:1
FROM ghcr.io/northwood-labs/lambda-provided-al2023@sha256:2b947c7c1e18392ce6b1b311ba1715a9b043a6fb5bb6572e914764e946321382

# Copy a file from the previous stage
COPY --from=go-installer /usr/local/bin/aws-lambda-rie /usr/local/bin/aws-lambda-rie
COPY entrypoint.sh /entrypoint.sh

# Ensure that these are executable
RUN chmod 0755 /usr/local/bin/aws-lambda-rie /entrypoint.sh

ENTRYPOINT ["/entrypoint.sh"]
```

### Contents of `entrypoint.sh`

We will be compiling our own Lambda function and will save it to `/var/runtime/bootstrap`.

These are the contents of `entrypoint.sh`. If we're running inside a Lambda Docker image, use the _Runtime Interface Emulator_ (from the first stage). Otherwise, don't.

```bash
#!/bin/sh
if [ -z "${AWS_LAMBDA_RUNTIME_API}" ]; then
    exec /usr/local/bin/aws-lambda-rie /var/runtime/bootstrap
else
    exec /var/runtime/bootstrap
fi
```

## Docker Compose

Here is our `docker-compose.yml` file. Modern Docker Compose no longer uses `version:` as a top-level YAML key.

```yaml
---
services:
  lambda:
    # Name of the container when it is running.
    container_name: localdev-lambda

    # Instructions which tell BuildKit how to build the image, passing secrets
    # SECURELY to the Dockerfile.
    build:
      context: .
      dockerfile: ./Dockerfile
      secrets:
        - github_token # See below for definition

    # Set shared memory limit when using `docker compose`.
    shm_size: 128mb

    # Stay running. Restart on failure.
    restart: always

    # Basic Linux-y and permission stuff.
    privileged: false
    pid: host
    sysctls:
      net.core.somaxconn: 1024

    # Environment variables used by the running Docker environment.
    # https://github.com/aws/aws-lambda-runtime-interface-emulator
    environment:
      _LAMBDA_SERVER_PORT: 8080
      AWS_LAMBDA_FUNCTION_TIMEOUT: 30      # Web timeout
      AWS_LAMBDA_FUNCTION_MEMORY_SIZE: 128 # Lambda function memory limit (logged; not enforced)
      LOG_LEVEL: DEBUG                     # Logging for the Runtime Interface Emulator

    # Mount a local directory inside the running Docker container.
    volumes:
      - ./var-runtime:/var/runtime:ro

    # Inside, the container runs on port 8080. But we want to expose it on
    # port 9000 to our host machine.
    ports:
      - 9000:8080

    # Enable running containers to communicate with services on the host machine.
    # Only works in Docker Desktop for local development. Don't do this with
    # containers you don't trust.
    extra_hosts:
      - host.docker.internal:host-gateway

# Define a secret here to read from the builder's environment variables, and
# pass them SECURELY into Docker BuildKit so that the Dockerfile can access it.
secrets:
  github_token:
    name: GITHUB_TOKEN
    environment: GITHUB_TOKEN

# Configure a bridge network to connect this (and other containers inside this
# file) together.
networks:
  dst-network:
    driver: bridge
```

## Compiling Go into a Lambda function

1. When you deploy a Lambda function to the real AWS Lambda service, and you're deploying a compiled executable for a Lambda function, it gets stored inside `/var/runtime`.

1. We know that later, we'll mount a local `var-runtime` directory (that is **not** a typo) into the running Docker environment at `/var/runtime`, so we'll compile our Lambda executable into our local `var-runtime` directory.

    ```yaml
    volumes:
      - ./var-runtime:/var/runtime:ro
    ```

1. Here, we'll compile our Lambda function into the correct local directory using the appropriate build flags. We do not specify a value for `GOARCH` because we want to build for whatever the current CPU architecture is, since we're running this locally.

    ```bash
    CGO_ENABLED=0 GOOS=linux go build \
        -a -trimpath \
        -ldflags="-s -w" \
        -tags lambda.norpc \
        -o localdev/var-runtime/bootstrap \
        . ;
    ```

## Running Docker Compose

From the directory containing your `docker-compose.yml` file, run:

```bash
docker compose up
```

The Docker image will build (installing the things you need), and the Lambda RIE process will start.

In our `docker-compose.yml` file, we specified that the Lambda service inside Docker (port `8080`) should be exposed to the host machine on port `9000`.

The endpoint for the local Lambda environment (exposed by RIE) will be:

```plain
http://localhost:9000/2015-03-31/functions/function/invocations
```

And in this case, whatever you send to this endpoint will be sent to your Lambda function to process.

But what if you have something sitting in-front of your Lambda function? Something like API Gateway? The payload that API Gateway sends to your Lambda function (when running in the cloud) is not the same bare-bones payload that RIE sends to your local Lambda function.

## Simulating API Gateway

When AWS ships software, they have a tendency to behave like IKEA. They give you the pieces you need for a solution, but don't actually provide you a solution that you can start using immediately.

The same is true for local Lambda environments. There is not, at the time of this writing, any out-of-the-box solution for simulating API Gateway. This GitHub issue entitled “[Emulate API Gateway payload in event object](https://github.com/aws/aws-lambda-runtime-interface-emulator/issues/64)” calls this out and has some helpful suggestions, but they all essentially involve setting up a reverse proxy which does what you want.

A reverse proxy sits between you and the service you're talking to, and (in this case) translates the _shape_ of the request and response into/out-of API Gateway format.

### Writing a reverse proxy

The [aws/aws-lambda-go] package simplifies writing Lambda functions in Go, and is a must-have library for this purpose. It provides the shapes of a variety of AWS services which communicate back to Lambda, including [API Gateway](https://github.com/aws/aws-lambda-go/blob/main/events/README_ApiGatewayEvent.md). [See the documentation](https://pkg.go.dev/github.com/aws/aws-lambda-go/events).

The flow looks like this:

1. Write a small web server (I used [Gin]) which simulates the endpoints you have with API Gateway sitting in front of Lambda. This is your _reverse proxy_.

1. When you send a web request to your reverse proxy, the reverse proxy reads your HTTP method, query string parameters, request body, etc., and rewrites those inputs as an [API Gateway payload][events.APIGatewayProxyRequest].

1. Your reverse proxy POSTs that [API Gateway-shaped payload][events.APIGatewayProxyRequest] to the endpoint for _Runtime Interface Emulator_ and your  Lambda function running in a Docker environment.

    ```plain
    http://localhost:9000/2015-03-31/functions/function/invocations
    ```

1. _Runtime Interface Emulator_ receives the payload, and invokes your Lambda function. This is automatic if you followed all of the instructions above to wire things together appropriately.

1. Your Lambda function, leveraging [aws/aws-lambda-go], will receive that payload, and do whatever your Lambda function is supposed to do. It will respond with an [events.APIGatewayProxyResponse] struct.

1. The reverse proxy receives the response from the Lambda. That response is an `events.APIGatewayProxyResponse` struct. The reverse proxy reads the status code and response body, then returns those back to the caller.

    Using [Gin], it looks something like this. Different frameworks for different languages will look different.

    ```go
    c.JSON(result.StatusCode, body)
    ```

> [!NOTE]
> An [example implementation](https://github.com/northwood-labs/devsec-tools/blob/605872a0285acd392670e7dba7cafc33e767ee7d/cmd/serve.go) can be found in the `devsec-tools` project.

### Debugging

I wrote a tool several years ago which _simulates_ an API Gateway request body. When API Gateway sits in front of a Lambda function, this is the payload that gets sent to the Lambda function on each request.

* The `headers` block is faked.
* The `requestContext` block is faked.
* `httpMethod`, `path`, `resource`, `body`, and `queryStringParameters` are all real.

#### GET example

```bash
curl -XGET "https://debug.ryanparman.com/json?abc=123&def=456"
```

#### POST example

```bash
curl -XPOST "https://debug.ryanparman.com/json" \
    --header 'Content-Type: application/x-www-form-urlencoded' \
    --data-urlencode "abc=123" \
    --data-urlencode "def=456" \
    ;
```

```bash
curl -XPOST "https://debug.ryanparman.com/json" \
    --header 'Content-Type: application/json; charset=utf-8' \
    --data $'{"abc": 123, "def": 456}' \
    ;
```

#### Format with go-spew

If you are using Go, you might find it helpful to replace `/json` with `/dump`. The "dump" format is produced by a tool called [go-spew](https://github.com/davecgh/go-spew), and shows you the data types of the payload.

[Docker Compose]: https://docs.docker.com/compose
[Docker Desktop]: https://docker.com/desktop
[Gin]: https://github.com/gin-gonic/gin#readme
[aws/aws-lambda-go]: https://pkg.go.dev/github.com/aws/aws-lambda-go
[events.APIGatewayProxyResponse]: https://pkg.go.dev/github.com/aws/aws-lambda-go/events#APIGatewayProxyResponse
[events.APIGatewayProxyRequest]: https://pkg.go.dev/github.com/aws/aws-lambda-go/events#APIGatewayProxyRequest
