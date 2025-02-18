# melody

![Build Status](https://github.com/olahol/melody/actions/workflows/test.yml/badge.svg)
[![Codecov](https://img.shields.io/codecov/c/github/olahol/melody)](https://app.codecov.io/github/olahol/melody)
[![Go Report Card](https://goreportcard.com/badge/github.com/olahol/melody)](https://goreportcard.com/report/github.com/olahol/melody)
[![GoDoc](https://godoc.org/github.com/olahol/melody?status.svg)](https://godoc.org/github.com/olahol/melody)

> :notes: Minimalist websocket framework for Go.

Melody is websocket framework based on [github.com/gorilla/websocket](https://github.com/gorilla/websocket)
that abstracts away the tedious parts of handling websockets. It gets out of
your way so you can write real-time apps. Features include:

* [x] Clear and easy interface similar to `net/http` or Gin.
* [x] A simple way to broadcast to all or selected connected sessions.
* [x] Message buffers making concurrent writing safe.
* [x] Automatic handling of sending ping/pong heartbeats that timeout broken sessions.
* [x] Store data on sessions.

## Install

```bash
go get github.com/olahol/melody
```

## [Example: chat](https://github.com/olahol/melody/tree/master/examples/chat)

[![Chat](https://cdn.rawgit.com/olahol/melody/master/examples/chat/demo.gif "Demo")](https://github.com/olahol/melody/tree/master/examples/chat)

Using [Gin](https://github.com/gin-gonic/gin):
```go
package main

import (
	"github.com/gin-gonic/gin"
	"github.com/olahol/melody"
	"net/http"
)

func main() {
	r := gin.Default()
	m := melody.New()

	r.GET("/", func(c *gin.Context) {
		http.ServeFile(c.Writer, c.Request, "index.html")
	})

	r.GET("/ws", func(c *gin.Context) {
		m.HandleRequest(c.Writer, c.Request)
	})

	m.HandleMessage(func(s *melody.Session, msg []byte) {
		m.Broadcast(msg)
	})

	r.Run(":5000")
}
```

Using [Echo](https://github.com/labstack/echo):
```go
package main

import (
	"github.com/labstack/echo/v4"
	"github.com/labstack/echo/v4/middleware"
	"github.com/olahol/melody"
	"net/http"
)

func main() {
	e := echo.New()
	m := melody.New()

	e.Use(middleware.Logger())
	e.Use(middleware.Recover())

	e.GET("/", func(c echo.Context) error {
		http.ServeFile(c.Response().Writer, c.Request(), "index.html")
		return nil
	})

	e.GET("/ws", func(c echo.Context) error {
		m.HandleRequest(c.Response().Writer, c.Request())
		return nil
	})

	m.HandleMessage(func(s *melody.Session, msg []byte) {
		m.Broadcast(msg)
	})

	e.Logger.Fatal(e.Start(":5000"))
}
```

## [Example: gophers](https://github.com/olahol/melody/tree/master/examples/gophers)

[![Gophers](https://cdn.rawgit.com/olahol/melody/master/examples/gophers/demo.gif "Demo")](https://github.com/olahol/melody/tree/master/examples/gophers)

```go
package main

import (
	"net/http"
	"strings"

	"github.com/gin-gonic/gin"
	"github.com/google/uuid"
	"github.com/olahol/melody"
)

type GopherInfo struct {
	ID, X, Y string
}

func main() {
	r := gin.Default()
	m := melody.New()

	r.GET("/", func(c *gin.Context) {
		http.ServeFile(c.Writer, c.Request, "index.html")
	})

	r.GET("/ws", func(c *gin.Context) {
		m.HandleRequest(c.Writer, c.Request)
	})

	m.HandleConnect(func(s *melody.Session) {
		ss, _ := m.Sessions()

		for _, o := range ss {
			value, exists := o.Get("info")

			if !exists {
				continue
			}

			info := value.(*GopherInfo)

			s.Write([]byte("set " + info.ID + " " + info.X + " " + info.Y))
		}

		id := uuid.NewString()
		s.Set("info", &GopherInfo{id, "0", "0"})

		s.Write([]byte("iam " + id))
	})

	m.HandleDisconnect(func(s *melody.Session) {
		value, exists := s.Get("info")

		if !exists {
			return
		}

		info := value.(*GopherInfo)

		m.BroadcastOthers([]byte("dis "+info.ID), s)
	})

	m.HandleMessage(func(s *melody.Session, msg []byte) {
		p := strings.Split(string(msg), " ")
		value, exists := s.Get("info")

		if len(p) != 2 || !exists {
			return
		}

		info := value.(*GopherInfo)
		info.X = p[0]
		info.Y = p[1]

		m.BroadcastOthers([]byte("set "+info.ID+" "+info.X+" "+info.Y), s)
	})

	r.Run(":5000")
}
```

### [More examples](https://github.com/olahol/melody/tree/master/examples)

## [Documentation](https://godoc.org/github.com/olahol/melody)

## Contributors

<a href="https://github.com/olahol/melody/graphs/contributors">
	<img src="https://contrib.rocks/image?repo=olahol/melody" />
</a>

## FAQ

If you are getting a `403` when trying  to connect to your websocket you can [change allow all origin hosts](http://godoc.org/github.com/gorilla/websocket#hdr-Origin_Considerations):

```go
m := melody.New()
m.Upgrader.CheckOrigin = func(r *http.Request) bool { return true }
```
