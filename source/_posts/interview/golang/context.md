---
title: Context in golang
date: 2020-11-27 22:37:18
tags:
- Golang
- Interview
---

# Context In Golang

## Overview

the context in golang has some functions and types.

We can see them in [document](https://golang.org/pkg/context/)

context package functions:
```golang
// WithCancel returns a copy of parent with a new Done channel. 
func WithCancel(parent Context) (ctx Context, cancel CancelFunc)

// WithDeadline returns a copy of the parent context with 
// the deadline adjusted to be no later than d.
func WithDeadline(parent Context, d time.Time) (Context, CancelFunc)

// WithTimeout returns WithDeadline(parent, time.Now().Add(timeout)).
func WithTimeout(parent Context, timeout time.Duration) (Context, CancelFunc)
```

2 types:
```golang
type CancelFunc
type Context
    // Background returns a non-nil, empty Context.
    func Background() Context
    // TODO returns a non-nil, empty Context.
    func TODO() Context
    // WithValue returns a copy of parent in which the value associated with key is val.
    func WithValue(parent Context, key, val interface{}) Context
```

important type -- context
```golang
type Context interface {
	Deadline() (deadline time.Time, ok bool)
	Done() <-chan struct{}
	Err() error
	Value(key interface{}) interface{}
}
```

So we can sure there are three roles of context:

1. Carry deadlines
2. Cancellation signals
3. Save Request-scoped values

## When we use it?

In the actual situation, we will meet some questions:

* how to cancel a goroutine
* how to cancel a RPC/HTTP request
* how to trace a RPC/HTTP request
* ...

Due to a RPC/HTTP request is goroutine in Golang, so we can use context to cancel the request by canceling goroutine.

Context can be put in a binary tree, actually, it has parent and child node.

So when we want to cancel a request cross many goroutine, we can use context tree.

Every node in the tree will sub the chan `Done`, when a node received the signal, the children will invoke `cancel()`.

And type context has a map struct, so we can save k-v data in it.

But it is better to save immutable information like:

1. request id
2. trace id
3. user auth token

## Create a context
First we can create a empty context by `Background()` and `TODO()`.

There are same function. 

But using `TODO()` that means you don't know which context to use ot it is not yet available.

A empty context is never canceled, has no values, and has no deadline.

We usually use them in following:

* main function
* initialization
* test case
* top-level Context for incoming requests

When we have a context, we can use `WithCancel()`, `WithDeadline()` and `WithTimeout()` to create a child one.

We can see the source of `WithCancel()`:
```golang
func WithCancel(parent Context) (ctx Context, cancel CancelFunc) {
	c := newCancelCtx(parent)
	propagateCancel(parent, &c)
	return &c, func() { c.cancel(true, Canceled) }
}
```

The `propagateCancel` will connect parent and child of cancel function.

First, if a context is empty, can be created by `Background()`, `Done()`, or is never canceled, can be created by `WithValue(emptyCtx, key, value)`, the `propagateCancel` will do nothing, parent will have no effect on children.

And then, the function will use `select` to sub parent chan `Done`, if parent is done, child will cancel immediately.

If not, the child will put into parent's a list of children.

If developer custom the type of context, the function will run a goroutine to sub chan `child.Done` and `parent.Done`.

## Pass Value

Sometimes, we want to trace a request in RPC/HTTP. We can put a trace id in headers or meta data.

In goroutine, we can pass some immutable information in context.

And we should control the key, and use `GetXXX()` and `WithXXX()` to get and set value.

```golang
type privateCtxType string
var (
  reqID = privateCtxType("req-id")
)
func GetRequestID(ctx context.Context) (int, bool) {
  id, exists := ctx.Value(reqID).(int)
  return id, exists
}
func WithRequestID(ctx context.Context, reqId int) context.Context {
  return context.WithValue(ctx, reqID, reqId)
}

```

## Using

There are currently two ways to integrate Context objects into your API:

* The first parameter of a function call
* Optional config on a request structure

> A great mental model of using Context is that it should flow through your program. Imagine a river or running water. This generally means that you don’t want to store it somewhere like in a struct. Nor do you want to keep it around any more than strictly needed. Context should be an interface that is passed from function to function down your call stack, augmented as needed. Ideally, a Context object is created with each request and expires when the request is over.

## Different with Gin Context

Gin use context to extend middleware.

The framework is as like `koa` in Node.js.

The middleware will be invoked twice in request and response.

And if we want to pass value in middleware, we can put k-v data in context.

But also, context of gin had implement the golang context interface.