---
layout: post
title: "Cool Golang slog.Logger tricks"
---

## Intro
For years in Go, I've used many different logging libraries, from `logrus` to `zap`.
After the release of `slog` into Go standard library, I immediately embraced it.
It's clever design made it so versatile and easy to use, and has already accumulated an enormous amount of libraries in the ecosystem.

## Trick 1

Try have a dedicate package in your project for logging attributes.
 - you have broad view of what attributes are available, especially when you are setting up logging indexes and queries in your dashboards.
 - it enforces people using a consistent way of naming logging attributes, especially in large teams, you won't end up with  `userID`, `user_id`, `UserID` in your logs at the same time.
 - it provides a easy gateway of implementing something even greater, which I'll show you in trick number 2.

e.g,
```go
package logging 

func Error(err error) slog.Attr {
    return slog.String("error", err.Error())
}

func UserID(id int) slog.Attr {
    return slog.Int("user_id", id)
}
```

## Trick 2

Combining logging attributes and error logs

```go
package logging

type ErrorWithAttrs struct {
	attrs []slog.Attr
	err   error
}

func (e *ErrorWithAttrs) Error() string {
	return e.err.Error()
}

func Errorf(format string, args ...any) error {
	var attrs []slog.Attr
	for _, arg := range args {
		if attr, ok := arg.(slog.Attr); ok {
			attrs = append(attrs, attr)
		}
	}

	return &ErrorWithAttrs{
		attrs: attrs,
		err:   fmt.Errorf(format, args...),
	}
}

func Error(err error) slog.Attr {
	attrs := AttrsFromError(err)
	if len(attrs) == 0 {
		return slog.String("error", err.Error())
	}
	attrs = append(attrs, slog.String("error", err.Error()))
	args := lo.Map(attrs, func(attr slog.Attr, _ int) any {
		return attr
	})
	return slog.Group("", args...)
}
```

This way, you will be able to do something like this:

```go

func SearchDB(userID int) (*User, error) {
    return nil, fmt.Errorf("user %v not found in database due to: %w", logging.UserID(userID), err)
}

func main() {
    _, err := SearchDB(123)
    myLogger.ErrorContext(ctx, "failed to get user", logging.Error(err))
}
```

This  will produce a log entry like this:

```
{"level":"error","msg":"failed to get user","user_id":123,"error":"user user_id=123 not found in database due to: <original error>"}}
```

You get the benefits of:
 * a. able to get all related details in error message
 * b. has a separate attribute field where you can index and query on in dashboards like kibana.


# Trick 3

A less easy way of using loggers all over the places. Sometimes it's just pain to pass loggers around, especially when you want to log as much as possible.

```go
package logging

import (
	"context"
	"log/slog"
	"sync"
)

var (
	globalHandler slog.Handler
	pendingLogs   []slog.Record
	pendingLogsMu sync.Mutex
)

func Provide(handler slog.Handler) {
	pendingLogsMu.Lock()
	for _, record := range pendingLogs {
		_ = handler.Handle(context.Background(), record)
	}
	pendingLogs = nil
	pendingLogsMu.Unlock()
	globalHandler = handler
}

func New(name string) *slog.Logger {
	handler := &placeholderHandler{
		attrs: []slog.Attr{
			slog.String("instrument", name),
		},
	}
	return slog.New(handler)
}

type placeholderHandler struct {
	attrs  []slog.Attr
	groups []string

	once    sync.Once
	handler slog.Handler
}

func (h *placeholderHandler) init() {
	h.once.Do(func() {
		handler := globalHandler
		for _, group := range h.groups {
			handler = handler.WithGroup(group)
		}
		h.handler = handler.WithAttrs(h.attrs)
	})
}

func (h *placeholderHandler) Enabled(ctx context.Context, level slog.Level) bool {
	if globalHandler == nil {
		return true
	}

	return globalHandler.Enabled(ctx, level)
}

func (h *placeholderHandler) Handle(ctx context.Context, record slog.Record) error {
	if globalHandler == nil {
		pendingLogsMu.Lock()
		pendingLogs = append(pendingLogs, record)
		pendingLogsMu.Unlock()
		return nil
	}

	h.init()
	return h.handler.Handle(ctx, record)
}

func (h *placeholderHandler) WithAttrs(attrs []slog.Attr) slog.Handler {
	if globalHandler != nil {
		h.init()
		return h.handler.WithAttrs(attrs)
	}

	return &placeholderHandler{
		attrs:  append(append([]slog.Attr{}, attrs...), h.attrs...),
		groups: append([]string{}, h.groups...),
	}
}

func (h *placeholderHandler) WithGroup(name string) slog.Handler {
	if globalHandler != nil {
		h.init()
		return h.handler.WithGroup(name)
	}

	return &placeholderHandler{
		attrs:  append([]slog.Attr{}, h.attrs...),
		groups: append(append([]string{}, h.groups...), name),
	}
}
```

with this, you can create a logger like this:

```go
var logger = logging.New("my-special-library").WithAttrs([]slog.Attr{
    slog.String("version", "1.0.0"),
})

func DoSomething() {
    logger.Info("doing something")
}
```

and just provide a handler at the entry point of your application once and for all:

```go
logging.Provide(slog.NewJSONHandler(os.Stdout, nil))
```
