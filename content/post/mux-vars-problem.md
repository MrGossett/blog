+++
date = "2015-04-14T09:22:04-04:00"
title = "The mux.Vars Problem"
tags = ["go"]
author = "Tim Gossett"
+++

We've begun to use `github.com/gorilla/mux` at work as a router for many of our services. I ran into a snag when writing some unit tests, and learned a bit about the design of `mux` in the process.

### A quick detour

When I say "unit tests", I mean this:

> ... a software development process in which the smallest testable parts of an application, called units, are individually and independently scrutinized for proper operation.
> [definition from techtarget.com](http://searchsoftwarequality.techtarget.com/definition/unit-testing)

More directly, I mean calling a single function with inputs formed specifically to test the behavior of that function. When I "unit test" handler funcs, I call the handler directly; I don't want to involve other parts of the system (like a `mux.Router`).

### The `mux.Vars` problem

`github.com/gorilla/mux` is a Golang HTTP router. Among other features, it provides a nice way to extract parameters from a request's path, using `mux.Vars` which is a `map[string]string`. However, the way it works makes it impossible to unit test handlers (in a pure sense).

Here's an example:

```go
router.Methods("GET").Path("/posts/{year}/{month}/{day}").HandlerFunc(ShowPost)
```

This sets up a route for `GET /posts/{year}/{month}/{day}` where the `year`, `month`, and `day` segments of the request path are parsed as variables in `mux.Vars`.

Then, in the `ShowPost` handler, the `year`, `month`, and `day` variables are accessed like this:

```go
vars := mux.Vars(req)

year  := vars["year"]
month := vars["month"]
day   := vars["day"]
```

Let's get a little insight into how this works. Here's the [definition of the `mux.Vars`](https://github.com/gorilla/mux/blob/master/mux.go#L259) func:

```go
// Vars returns the route variables for the current request, if any.
func Vars(r *http.Request) map[string]string {
	if rv := context.Get(r, varsKey); rv != nil {
		return rv.(map[string]string)
	}
	return nil
}
```

This is using `github.com/gorilla/context`, which maintains a global `map[*http.Request]map[interface{}]interface{}`. When `mux` is routing a request, it parses the parameters from the request path, forms the `map[string]string` for `Vars`, and does `context.Set(req, varsKey, vars)`, which stashes `vars` in the global map, using `req` and then `varsKey` as keys.

In the func above, those vars are fetched with `context.Get(r, varsKey)` as a `map[interface{}]interface{}`, and then type-asserted back into a `map[string]string`.

When unit-testing a handler, if the handler uses `mux.Vars` to get at request params (as we did in the example above), then we'll need to craft a `map[string]string` to use with the test. We then would want to set that map in the request's context with something like `context.Set(testReq, someKey, testMap)`. So... where do we get `someKey`?

Back in that [same file](https://github.com/gorilla/mux/blob/master/mux.go#L251) in `github.com/gorilla/mux`, here's the definition of `varsKey`:

```go
type contextKey int

const (
	varsKey contextKey = iota
	routeKey
)
```

`varsKey` is of type `contextKey`, which is unexported; that means it's not accessible outside of the `github.com/gorilla/mux` package. This means we can't craft the right type of key to set our test vars in the context of a test request. That means we have to have `github.com/gorilla/mux` process the test request. So we have to create a `mux.Router` wired up with the handler under test, and call `ServeHTTP` on that router. That's closer to integration testing than pure unit testing of the handler.

This all would be solved if either the `varsKey` var or the `contextKey` type were exported. I'm consdering submitting a pull request to `github.com/gorilla/mux` to do just that; if I do, I'll link to it here.