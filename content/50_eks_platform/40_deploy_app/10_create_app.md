+++
title = "3.1 Create a Sample App"
chapter = false
weight = 10
+++

To simulate the experience of a backend engineer deploying to your Kubernetes platform, we'll write a small application in Go, that we want to use to deploy to a Kubernetes clyster.

## Step 1 &mdash; Create App

Create a new directory in your Pulumi project called `app`:

```
mkdir platform-app
```

Inside that directory, add a `main.go` file which will serve a simple webserver that returns `Hello, world!`:

```go
package main

import (
	"fmt"
	"github.com/gin-gonic/gin"
	"net/http"
	"os"
)

func main() {
	r := gin.New()
	r.Use(gin.Logger())
	r.Use(gin.Recovery())

	r.GET("/", func(c *gin.Context) {
		c.String(http.StatusOK, "Hello, world!")
	})

	var listenPort string

	port, defined := os.LookupEnv("LISTEN_PORT")

	if defined {
		listenPort = port
	} else {
		listenPort = "80"
	}

	r.Run(fmt.Sprintf(":%s", listenPort))
}

```

You'll also need to create a `go.mod` file inside the `app` directory:

```
go mod init platform-app
```

You can go test your application starts by using `go run`:

```
LISTEN_PORT=8080 go run platform-app/main.go
```

You should see your web server start, like so:

```
[GIN-debug] [WARNING] Running in "debug" mode. Switch to "release" mode in production.
 - using env:	export GIN_MODE=release
 - using code:	gin.SetMode(gin.ReleaseMode)

[GIN-debug] GET    /                         --> main.main.func1 (3 handlers)
[GIN-debug] Listening and serving HTTP on :8080
```

## Step 2 &mdash; Create a Docker Image

Next, we need to add a Dockerfile so we can push this Image to a registry. Inside the `app` directory, add a `Dockerfile that looks like this:

```dockerfile
FROM golang:buster as builder
WORKDIR /app
COPY . .
ENV CGO_ENABLED=0
RUN go mod download \
    && go build -o /app/webserver

# Runtime container
FROM scratch
WORKDIR /app
ENV LISTEN_PORT=80
COPY --from=builder /app/webserver /app/webserver

ENTRYPOINT [ "/app/webserver" ]
```

We're using a Docker multi-stage build here to ensure we keep the image size as low as possible. This is the end of building our application.

Now we have our docker container, we need to deploy it to our Kubernetes cluster.

