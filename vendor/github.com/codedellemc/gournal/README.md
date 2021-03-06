# Gournal [![GoDoc](https://godoc.org/github.com/codedellemc/gournal?status.svg)](https://godoc.org/github.com/codedellemc/gournal) [![Build Status](https://travis-ci.org/emccode/gournal.svg?branch=master)](https://travis-ci.org/emccode/gournal) [![Go Report Card](https://goreportcard.com/badge/github.com/codedellemc/gournal)](https://goreportcard.com/report/github.com/codedellemc/gournal) [![codecov](https://codecov.io/gh/emccode/gournal/branch/master/graph/badge.svg)](https://codecov.io/gh/emccode/gournal)
Gournal (pronounced "Journal") is a Context-aware logging framework
that introduces the Google [Context type](https://bgournal.golang.org/context) as
a first-class parameter to all common log functions such as Info, Debug, etc.

## Getting Started
Instead of being Yet Another Go Log library, Gournal actually takes its
inspiration from the Simple Logging Facade for Java
([SLF4J](http://www.slf4j.org/)). Gournal is not attempting to replace anyone's
favorite logger, rather existing logging frameworks such as
[Logrus](github.com/Sirupsen/logrus), [Zap](github.com/uber-go/zap), etc. can
easily participate as a Gournal Appender.

The following
[example](https://github.com/codedellemc/gournal/tree/master/examples/01/main.go)
is a simple program that uses Logrus as a Gournal Appender to emit some log
data:

```go
package main

import (
	"github.com/codedellemc/gournal"
	"github.com/codedellemc/gournal/logrus"
)

func main() {
	ctx := gournal.Background()
	ctx = gournal.WithValue(ctx, gournal.LevelKey(), gournal.InfoLevel)
	ctx = gournal.WithValue(ctx, gournal.AppenderKey(), logrus.New())

	gournal.Info(ctx, "Hello %s", "Bob")

	gournal.WithFields(map[string]interface{}{
		"size":     1,
		"location": "Austin",
	}).Warn(ctx, "Hello %s", "Mary")
}
```

To run the above example, clone this project and execute the following from the
command line:

``` bash
$ make run-exmaple-1
INFO[0000] Hello Bob                                    
WARN[0000] Hello Mary                                    location=Austin size=1
```

## Go 1.7 Support
Prior to Go 1.7 the Context type was not part of the Go standard library
(StdLib). Instead the Go Context was provided via the `golang.org/x/net/context`
package. However, due to the growing importance of the type, it is supported by
Go 1.7 as part of the StdLib in the `context` package.

Since Go interfaces use duck-typing, where an interface is defined is not
entirely important. However, if a Go 1.7 application imports the old package
`golang.org/x/net/context`, the application will be needlessly introducing
an additional dependency since the Go 1.7 StdLib includes the Context type
already.

Yet, Gournal cannot just immediately abandon support for the old import path
as not all applications are able to adopt Go 1.7 on day one. Therefore Gournal
needs to support both pre-1.7 and 1.7 import paths for the Context type. To
achieve this Gournal defines the following type:

```go
type Context interface {
	gournal.Context
}
```

The above type is defined in two files. One file is activated for pre-Go 1.7
toolchains and uses the old import path. The other file is activated for Go 1.7
and uses the StdLib import path. For convenience Gournal also redefines the
`WithValue` and `Background` functions.

Because Gournal's Appender interface uses Gournal's Context interface
definition, implementations of the Appender interface will also be required to
reference `gournal.Appender` when defining the `Append` function. However,
any object that satisfies a Google Context, pre or post 1.7, satisfies a
Gournal Context as well since the latter is just an empty interface that
inherits one of the former.

## Compatability
Gournal provides ready-to-use Appenders for the following logging frameworks:

  * [Logrus](https://github.com/codedellemc/gournal/tree/master/logrus)
  * [Zap](https://github.com/codedellemc/gournal/tree/master/zap)
  * [Google App Engine](https://github.com/codedellemc/gournal/tree/master/gae)
  * [`gournal.Logger`](https://github.com/codedellemc/gournal/tree/master/stdlib)
  * [`io.Writer`](https://github.com/codedellemc/gournal/tree/master/iowriter)

With little overhead, Gournal leverages the Google Context type to provide an
elegant solution to the absence of features that are commonly found in
languages that employ thread-local storage. And not only that, but using
Gournal helps avoid logger-lock-in. Replacing the underlying implementation
of a Gournal Appender is as simple as placing a different Appender object
into the Context.

## Performance
Gournal has minimal impact on the performance of the underlying logger
framework.

Benchmark | Logger | Time | Malloc Size | Malloc Count
-----|--------|-----------|------|-------------|-------------
Native without Fields | `gournal.Logger` | 1024 ns/op | 16 B/op | 1 allocs/op
       | Logrus | 4118 ns/op | 832 B/op | 19 allocs/op
       | Zap | 1347 ns/op | 0 B/op | 0 allocs/op
       | Google App Engine | 1302 ns/op | 32 B/op | 1 allocs/op
Gournal without Fields | `gournal.Logger` | 1230 ns/op | 16 B/op | 1 allocs/op
       | Logrus | 3784 ns/op | 832 B/op | 19 allocs/op
       | Zap | 1448 ns/op | 0 B/op | 0 allocs/op
       | Google App Engine | 1641 ns/op | 32 B/op | 1 allocs/op
Gournal with Fields | `gournal.Logger` | 4424 ns/op | 881 B/op | 18 allocs/op
       | Logrus | 6467 ns/op | 1746 B/op | 31 allocs/op
       | Zap | 3160 ns/op | 641 B/op | 8 allocs/op
       | Google App Engine | 5196 ns/op | 1041 B/op | 20 allocs/op

The above benchmark information (results may vary) was generated using the
following command:

```bash
$ make benchmark
GOOS=darwin GOARCH=amd64 go install .
GOOS=darwin GOARCH=amd64 go install ./gae
GOOS=darwin GOARCH=amd64 go install ./logrus
GOOS=darwin GOARCH=amd64 go install ./stdlib
GOOS=darwin GOARCH=amd64 go install ./zap
./.gaesdk/1.9.40/go_appengine/goapp test -cover -coverpkg 'github.com/codedellemc/gournal' -c -o benchmarks/benchmarks.test ./benchmarks
warning: no packages being tested depend on github.com/codedellemc/gournal
benchmarks/benchmarks.test -test.run Benchmark -test.bench . -test.benchmem 2> /dev/null
PASS
BenchmarkNativeStdLibWithoutFields-8 	 1000000	      1024 ns/op	      16 B/op	       1 allocs/op
BenchmarkNativeLogrusWithoutFields-8 	  300000	      4118 ns/op	     832 B/op	      19 allocs/op
BenchmarkNativeZapWithoutFields-8    	 1000000	      1347 ns/op	       0 B/op	       0 allocs/op
BenchmarkNativeGAEWithoutFields-8    	 1000000	      1302 ns/op	      32 B/op	       1 allocs/op
BenchmarkGournalStdLibWithoutFields-8	 1000000	      1230 ns/op	      16 B/op	       1 allocs/op
BenchmarkGournalLogrusWithoutFields-8	  500000	      3784 ns/op	     832 B/op	      19 allocs/op
BenchmarkGournalZapWithoutFields-8   	 1000000	      1448 ns/op	       0 B/op	       0 allocs/op
BenchmarkGournalGAEWithoutFields-8   	 1000000	      1641 ns/op	      32 B/op	       1 allocs/op
BenchmarkGournalStdLibWithFields-8   	  300000	      4424 ns/op	     881 B/op	      18 allocs/op
BenchmarkGournalLogrusWithFields-8   	  300000	      6467 ns/op	    1745 B/op	      31 allocs/op
BenchmarkGournalZapWithFields-8      	  500000	      3160 ns/op	     641 B/op	       8 allocs/op
BenchmarkGournalGAEWithFields-8      	  300000	      5196 ns/op	    1041 B/op	      20 allocs/op
coverage: 21.6% of statements in github.com/codedellemc/gournal
```

Please keep in mind that the above results will vary based upon the version of
dependencies used. This project uses
[Glide](https://github.com/Masterminds/glide/) to pin dependencies such as
Logrus and Zap.

## Configuration
Gournal is configured primarily via the Context instances supplied to the
various logging functions. However, if a supplied argument is nil or is missing
the Appender or Level, there are some default, global variables that can
supplement the missing pieces.

Global Variable | Default Value | Description
-----------------|---------------|-----------
`DefaultLevel`  | `ErrorLevel` | Used when a Level is not present in a Context.
`DefaultAppender` | `nil` | Used when an Appender is not present in a Context.
`DefaultContext` | `gournal.Background()` | Used when a log method is invoked with a nil Context.

Please note that there is no default value for `DefaultAppender`. If this
field is not assigned and log function is invoked with a nil `Context` or one
absent an `Appender` object, a panic will occur.

## Features
Gournal provides several features on top of the underlying logging framework
that is doing the actual logging:

 * [Concurrent Logging Frameworks](#concurrent-logging-frameworks)
 * [Global Context Fields](#global-context-fields)
 * [Multiple Log Levels](#multiple-log-levels)

### Concurrent Logging Frameworks
The following
[example](https://github.com/codedellemc/gournal/tree/master/examples/02/main.go)
illustrates how to utilize the Gournal `DefaultAppender` as well as multiple
logging frameworks in the same program:

```go
package main

import (
	"github.com/codedellemc/gournal"
	"github.com/codedellemc/gournal/logrus"
	"github.com/codedellemc/gournal/zap"
)

func main() {
	// Make a Zap-based Appender the default appender for when one is not
	// present in a Context, or when a nill Context is provided to a logging
	// function.
	gournal.DefaultAppender = zap.New()

	// The following call fails to provide a valid Context argument. In this
	// case the DefaultAppender is used.
	gournal.WithFields(map[string]interface{}{
		"size":     2,
		"location": "Boston",
	}).Error(nil, "Hello %s", "Bob")

	ctx := gournal.Background()
	ctx = gournal.WithValue(ctx, gournal.LevelKey(), gournal.InfoLevel)

	// Even though this next call provides a valid Context, there is no
	// Appender present in the Context so the DefaultAppender will be used.
	gournal.Info(ctx, "Hello %s", "Mary")

	ctx = gournal.WithValue(ctx, gournal.AppenderKey(), logrus.New())

	// This last log function uses a Context that has been created with a
	// Logrus Appender. Even though the DefaultAppender is assigned and is a
	// Zap-based logger, this call will utilize the Context Appender instance,
	// a Logrus Appender.
	gournal.WithFields(map[string]interface{}{
		"size":     1,
		"location": "Austin",
	}).Warn(ctx, "Hello %s", "Alice")
}
```

To run the above example, clone this project and execute the following from the
command line:

```bash
$ make run-example-2
{"level":"error","ts":1470251785.437946,"msg":"Hello Bob","size":2,"location":"Boston"}
{"level":"info","ts":1470251785.4379828,"msg":"Hello Mary"}
WARN[0000] Hello Alice                                   location=Austin size=1
```

### Global Context Fields
Another nifty feature of Gournal is the ability to provide a Context with
fields that will get emitted along-side every log message, whether they are
explicitly provided with log message or not. This feature is illustrated
in the
[example](https://github.com/codedellemc/gournal/tree/master/examples/03/main.go)
below:

```go
package main

import (
	"github.com/codedellemc/gournal"
	"github.com/codedellemc/gournal/logrus"
)

func main() {
	ctx := gournal.Background()
	ctx = gournal.WithValue(ctx, gournal.LevelKey(), gournal.InfoLevel)
	ctx = gournal.WithValue(ctx, gournal.AppenderKey(), logrus.New())

	ctx = gournal.WithValue(
		ctx,
		gournal.FieldsKey(),
		map[string]interface{}{
			"name":  "Venus",
			"color": 0x00ff00,
		})

	// The following log entry will print the message and the name and color
	// of the planet.
	gournal.Info(ctx, "Discovered planet")

	ctx = gournal.WithValue(
		ctx,
		gournal.FieldsKey(),
		func() map[string]interface{} {
			return map[string]interface{}{
				"galaxy":   "Milky Way",
				"distance": 42,
			}
		})

	// The following log entry will print the message and the galactic location
	// and distance of the planet.
	gournal.Info(ctx, "Discovered planet")

	// Create a Context with the FieldsKey that points to a function which
	// returns a Context's derived fields based upon what data was provided
	// to a the log function.
	ctx = gournal.WithValue(
		ctx,
		gournal.FieldsKey(),
		func(ctx gournal.Context,
			lvl gournal.Level,
			fields map[string]interface{},
			args ...interface{}) map[string]interface{} {

			if v, ok := fields["z-value"].(int); ok {
				delete(fields, "z-value")
				return map[string]interface{}{
					"point": struct {
						x int
						y int
						z int
					}{1, -1, v},
				}
			}

			return map[string]interface{}{
				"point": struct {
					x int
					y int
				}{1, -1},
			}
		})

	// The following log entry will print the message and two-dimensional
	// location information about the planet.
	gournal.Info(ctx, "Discovered planet")

	// This log entry, however, will print the message and the same location
	// information, however, because the function used to derive the Context's
	// fields inspects the field's "z-value" key, it will add that data to the
	// location information, making it three-dimensional.
	gournal.WithField("z-value", 3).Info(ctx, "Discovered planet")
}
```

To run the above example, clone this project and execute the following from the
command line:

```bash
$ make run-example-3
INFO[0000] Discovered planet                             color=65280 name=Venus
INFO[0000] Discovered planet                             distance=42 galaxy=Milky Way
INFO[0000] Discovered planet                             point={x:1 y:-1}
INFO[0000] Discovered planet                             point={x:1 y:-1 z:3}
```

### Multiple Log Levels
Instead of creating multiple logger instances that exist and consume resources
for no other reason than to have multiple log levels, Gournal supports multiple
log levels as well as ensuring that no resources are wasted if a log entry does
not meet the level qualifications:

```go
package main

import (
	"fmt"

	"github.com/codedellemc/gournal"
	"github.com/codedellemc/gournal/logrus"
)

// myString is a custom type that has a custom fmt.Format function.
// This function should *not* be invoked unless the log level is such that the
// log message would actually get emitted. This saves resources as fields
// and formatters are not invoked at all unless the log level allows an
// entry to be logged.
type myString string

func (s myString) Format(f fmt.State, c rune) {
	fmt.Println("* INVOKED MYSTRING FORMATTER")
	fmt.Fprint(f, string(s))
}

func main() {
	ctx := gournal.Background()
	ctx = gournal.WithValue(ctx, gournal.AppenderKey(), logrus.New())

	counter := 0

	// Set up a context fields callback that will print a loud message to the
	// console so it is very apparent when the function is invoked. This
	// function should *not* be invoked unless the log level is such that the
	// log message would actually get emitted. This saves resources as fields
	// and formatters are not invoked at all unless the log level allows an
	// entry to be logged.
	getCtxFieldsFunc := func() map[string]interface{} {
		counter++
		fmt.Println("* INVOKED CONTEXT FIELDS")
		return map[string]interface{}{"counter": counter}
	}
	ctx = gournal.WithValue(ctx, gournal.FieldsKey(), getCtxFieldsFunc)

	var name myString = "Bob"

	// Log "Hello Bob" at the INFO level. This log entry will not get emitted
	// because the default Gournal log level (configurable by
	// gournal.DefaultLevel) is ERROR.
	//
	// Additionally, we should *not* see the messages produced by the
	// myString.Format and getCtxFieldsFunc functions.
	gournal.Info(ctx, "Hello %s", name)

	// Keep a reference to the context that has the original log level.
	oldCtx := ctx

	// Set the context's log level to be INFO.
	ctx = gournal.WithValue(ctx, gournal.LevelKey(), gournal.InfoLevel)

	// Note the log level has been changed to INFO. This is also a marker to
	// show that the previous log and messages generated by the functions should
	// not have occurred prior to this statement in the terminal.
	fmt.Println("* CTX LOG LEVEL INFO")

	name = "Mary"

	fields := map[string]interface{}{
		"length":   8,
		"location": "Austin",
	}

	// Log "Hello Mary" with some field information. We should not only see
	// the messages from the myString.Format and getCtxFieldsFunc functions,
	// but the field "size" from the getCtxFieldsFunc function should add the
	// field "counter" to the fields provided directly to this call.
	gournal.WithFields(fields).Info(ctx, "Hello %s", name)

	// Log "Hello Mary" again with the exact same info, except use the original
	// context that did not have an explicit log level. Since the default log
	// level is still ERROR, nothing will be emitted, not even the messages that
	// indicate the myString.Format or getCtxFieldsFunc functions are being
	// invoked.
	gournal.WithFields(fields).Info(oldCtx, "Hello %s", name)

	// Update the default log level to INFO
	gournal.DefaultLevel = gournal.InfoLevel
	fmt.Println("* DEFAULT LOG LEVEL INFO")

	// Log "Hello Mary" again with the exact same info, even use the original
	// context that did not have an explicit log level. However, since the
	// default log level is now INFO, the entry will be emitted, along with the
	// messages from the myString.Format or getCtxFieldsFunc functions are being
	// invoked.
	//
	// Note the counter value has only be incremented once since the function
	// was not invoked when the log level did not permit the entry to be logged.
	gournal.WithFields(fields).Info(oldCtx, "Hello %s", name)
}
```

To run the above example, clone this project and execute the following from the
command line:

```bash
$ make run-example-4
go run ./examples/04/main.go
* CTX LOG LEVEL INFO
* INVOKED CONTEXT FIELDS
* INVOKED MYSTRING FORMATTER
INFO[0000] Hello Mary                                    counter=1 length=8 location=Austin
* DEFAULT LOG LEVEL INFO
* INVOKED CONTEXT FIELDS
* INVOKED MYSTRING FORMATTER
INFO[0000] Hello Mary                                    counter=2 length=8 location=Austin
```
