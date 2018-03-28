# go-backoff

Backoff algorithm and helpers for Go

[![Build Status](https://travis-ci.org/lestrrat-go/backoff.png?branch=master)](https://travis-ci.org/lestrrat-go/backoff)

[![GoDoc](https://godoc.org/github.com/lestrrat-go/backoff?status.svg)](https://godoc.org/github.com/lestrrat-go/backoff)

# SYNOPSIS

```go

import "github.com/lestrrat-go/backoff"

func Func(arg Foo) (Result, error) { ... }

var policy = backoff.NewExponential(
  backoff.WithInterval(500*time.Millisecond), // base interval
  backoff.WithJitterFactor(0.05), // 5% jitter
  backoff.WithMaxRetries(25),
)
func RetryFunc(arg Foo) (Result, error) {
  b, cancel := policy.Start(context.Background())
  defer cancel()

  for retry.Continue(b) {
    res, err := Func(arg)
    if err == nil {
      return res, nil
    }
  }
  return nil, errors.New(`tried very hard, but no luck`)
}
```

## Simple Usage

```go
func ExampleRetry() {
  count := 0
  e := backoff.ExecuteFunc(func(_ context.Context) error {
    // This is a silly example that succeeds on every 10th try
    count++
    if count%10 == 0 {
      return nil
    }
    return errors.New(`dummy`)
  })

  ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
  defer cancel()

  p := backoff.NewExponential()
  if err := backoff.Retry(ctx, p, e); err != nil {
    log.Printf("failed to call function after repeated tries")
  }
}
```

# DESCRIPTION

This library is an implementation of backoff algorithm for retrying operations
in an idiomatic Go way. It respects `context.Context` natively, and the critical
notifications are done through *channel operations*, allowing you to write code 
that is both more explicit and flexibile.

It also exports a utility function `Retry`, for simple operations.

For a longer discussion, [please read this article](https://medium.com/@lestrrat/yak-shaving-with-backoff-libraries-in-go-80240f0aa30c)

# TODO

* Refine the timeouts generated by exponential backoff algorithm (help wanted!)

# PRIOR ARTS

## [github.com/cenkalti/backoff](https://github.com/cenkalti/backoff) 

This library is featureful, but one thing that gets to me is the fact that
it essentially forces you to create a closure over the operation to be retried.

## [github.com/jpillora/backoff](https://github.com/jpillora/backoff)

This library is a very simple implementation of calculating backoff durations.
I wanted it to let me know when to stop too, so it was missing a few things.

# DUMB BENCHMARK

```
make benchmark
go test -run=none -tags bench -bench . -benchmem -benchtime 20s
goos: darwin
goarch: amd64
pkg: github.com/lestrrat-go/backoff
Benchmark/cenkalti-4         	       3	6828914804 ns/op	    1205 B/op	      24 allocs/op
Benchmark/lestrrat-4         	       5	4842358461 ns/op	    1022 B/op	      20 allocs/op
PASS
ok  	github.com/lestrrat-go/backoff	70.711s
```