# Notes on API design in Go

At work, I have a working student who's implementing some features in the various Golang applications that I build and maintain. I'm trying to pass some of my experience with real-world programming on to him, and [one particular pull request review](https://github.com/sapcc/go-bits/pull/2) escalated into a blog post on API design, so I might as well share it here for archival purposes (and to fill the desolate wasteland that is my RSS feed).

The concern of the pull request was to add a function to a utility library that implements exponential backoff. His proposed API looked like:

```go
//Retry takes a function (action) that returns an error, and two int64 values (x, y) as
//parameters and creates a retry loop with an exponential backoff such that on failure (error return),
//the action is called again after x seconds and this is incremented by a factor of 2 until y minutes
//then it is keeps on repeating after y minutes till action succeeds (no error).
func Retry(action func() error, x, y time.Duration) { ...  }
```

## The original comment

When you have a function that takes another function, I like to place the function argument at the end of the argument list. When the function argument is a long anonymous function, the other arguments otherwise get separated from the function call:

```go
err := doSomething(function() error {
  if aLotOfThings.Happen() {
    in.ThisFunction(AndMaybe {
       There: "are",
       More: "braces",
       And: "stuff",
    })
  }
  then(Lines(100).DownBelow())
  itIsNotClear = true
}, 2, 4) //...which function call these arguments belong to
```

vs.

```go
err := doSomething(2, 4, function() error {
  ...
  ...
  ...
})
```

Finally, I would suggest a different API design altogether. The problem with function arguments is that it's sometimes difficult to tell from context what the arguments mean. An extreme example (which I sadly cannot find on Google right now) is the Win32 API function for starting a new process which takes 22 arguments and it looks something like:

```
StartProcess("C:\Windows\system32\rundll.exe", null, null, null, null, null, null, null, null, true, true, false, false, null, false, true);
```

So unless you have the API docs open on your other monitor, it's pretty impossible to tell what each of these arguments mean. To be able to give meaningful names to arguments, it's common practice in Go APIs to collect all these switches and configuration options into an Options struct:

```go
type RetryOptions struct {
  BackoffFactor int
  MaxInterval time.Duration
}
func Retry(opts RetryOptions, action func() error) { ... }

//usage example:
Retry(RetryOptions { BackoffFactor: 2, MaxInterval: 5 * time.Second }, func() {
  ...
})
```

That's much more verbose, but also much more obvious when you're reading it. It also has the advantage that you can later add new fields to `RetryOptions` without breaking existing users of your API. (If you look at the API of [Schwift](https://godoc.org/github.com/majewsky/schwift), you'll see these Options types all over the place for this reason.)

Finally, here's the API that I would propose:

```go
type RetryStrategy interface {
  RetryUntilSuccessful(action func() error)
}

type ExponentialBackoff struct {
  Factor int
  MaxInterval time.Duration
}
func (eb ExponentialBackoff) RetryUntilSuccessful(action func() error) { ... }

//usage example:
ExponentialBackoff {
  Factor: 2,
  MaxInterval: 5 * time.Second,
}.RetryUntilSuccessful(func() error {
  ...
})
```

This makes it easy to later add other implementations for `type RetryStrategy` (e.g. one that just retries for a given number of times and then gives up). It also allows other parts of the program to take a RetryStrategy as a parameter.

## The morale

A good API design is based on two things: First, there are some fundamental values that the developer tries to optimize for. In my case, that's extensibility and backwards-compatibility. (I've been bitten by updates with breaking changes a few times too often and don't want to install this pain on others, or on myself.) Second, the language features inform how to translate these values into an API design. In Python, I wouldn't work with Options types, because Python provides kwargs which serve the same purpose and are more idiomatic.
