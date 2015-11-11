+++
date = "2015-08-09T14:15:53-05:00"
title = "Inverted Switch"
tags = ["go"]
author = "Tim Gossett"
+++

Go is great for building APIs. It's common for APIs to perform some validation on the data structures used in request bodies. 

### some structure

Assume we're handling a request with the following body:

```json
{
  "thing_id": 123,
  "foo": "fizz",
  "bar": "buzz",
  "some_date": "2015-08-09T07:25:43-05:00"
}
```

This can be unmarshalled onto the following Go struct:

```go
type SomeRequest struct {
	ThingID  int64 `json:"thing_id"`
	Foo, Bar string
	SomeDate time.Time `json:"some_date"`
}
```

`package json` will do the heavy lifting to parse the date and convert the JSON numeric into an `int64`. If the client had supplied something other than an ISO8601 date string for `some_date`, `package json` would squawk. The same goes for values in `thing_id` that cannot be converted to an `int64`.

### validate me

Now let's validate that the client supplied data for each attribute in the struct.

There are some great packages available to define constraints with struct annotations. However, I prefer hand-rolling validation with a few simple funcs.

```go
func (req SomeRequest) validate() error {
	if req.Foo == "" {
		return errInvalidRequest
	}

	if req.Bar == "" {
		return errInvalidRequest
	}

	if req.ThingID == 0 {
		return errInvalidRequest
	}

	if req.SomeDate.UnixNano() == 0 {
		return errInvalidRequest
	}

	return nil
}
```

This naïve approach gets the job done, but we can do better. [gocyclo](https://github.com/fzipp/gocyclo) calculates the cyclomatic complexity of the func above at 5.

### inverted switch

The first two conditionals are testing for the same condition in two different attributes. In my view, the most elegant way to detect if one thing matches one of a few other things is with a `switch`. I'm accustomed to using a variable as the control expression and using hard-coded literals or constants as the cases; but we can invert that pattern:

```go
switch "" {
case req.Foo, req.Bar:
        return errInvalidRequest
}
```

The second two conditionals are dealing with attributes of two different types: one is an `int64` and the other is a [`time.Time`](https://golang.org/pkg/time/#Time). However, `time.Time` can covert itself to an `int64` with [`(time.Time).UnixNano()`](https://golang.org/pkg/time/#Time.UnixNano). We can use that to apply the same pattern here:

```go
switch int64(0) {
case req.ThingID, req.SomeDate.UnixNano():
        // nope
}
```

The `validate` func now looks like this:

```go
func (req SomeRequest) validate() error {
	switch "" {
	case req.Foo, req.Bar:
		return errInvalidRequest
	}

	switch int64(0) {
	case req.ThingID, req.SomeDate.UnixNano():
		return errInvalidRequest
	}

	return nil
}
```

Much better.

As a bonus, `gocyclo` now calculates complexity at 3.

```
$ gocyclo .
3 main (SomeRequest).validate
```
