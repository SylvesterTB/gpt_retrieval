# Deferred error handling

After analyzing the implications a possible introduction of deferrred error handling would have to our project, we decided not to change it in general.
This file documents the insights, we have gained during analysis, and can be used as a guide, when you really want to use deferred error handling.
It provides examples as well as it mentions pitfalls, which you need to keep in mind for implementation.

In general, deferred error-handling has the advantage, that it is always executed.
This includes cases, where an unexpected `panic` occurs.
However, deferred error handling becomes tricky in case of more complex functions.

Let's start with some short examples, what the error handling could look like, and then increase the complexity step by step.

Let's look at the following example:

```go
func deferredErrorHandling() error {
    defer handleError() // always executed

    if err := firstFunction(); err != nil {
        return util.WrapErrorWithContextInfo("firstFunction", err)
    }
    return nil
}
```

**Pitfall No.1**: A deferred function is always executed!

In some cases this is desired, e.g. for closing files, which should be done, no matter if an error occurred or not.
However, deferred error handling is only desired in case of an actual failure.

In this case it means, that `handleError` is also called in case of no error.

You could overcome this issue, by simply moving the `defer` call within the if statement,
which, however, comes with the drawback, that business code and error handling are no longer strictly separated.

```go
func deferredErrorHandling() error {
    if err := firstFunction(); err != nil {
        defer handleError() // errors are discarded
        return util.WrapErrorWithContextInfo("firstFunction", err)
    }
    return nil
}
```

**Pitfall No.2**: A deferred cleanup function might also cause errors!

We might want to know whether the `handleError` was successful, or not.
Depending on that, we might continue with a recovered state of the program, or terminate the running process.

Let's take a look at the following:

```go
func deferredErrorHandling() error {
    if err := firstFunction(); err != nil {
        defer func() error {
            if err := handleError(); err != nil {
                return util.WrapErrorWithContextInfo("error handling failed as well", err) // this will never be returned
            }
            return nil
        }()
        return util.WrapErrorWithContextInfo("firstFunction", err)
    }
    return nil
}
```

**Pitfall No.3**: A return value in a deferred function is ignored!

Even though we defined a return value in the deferred function, the result of it will be discarded.
To handle this as well, we need to use named return values, which can be overridden by the deferred function:

```go
func deferredErrorHandling() (err error) {
    if err := firstFunction(); err != nil {
        defer func() {
            if err2 := handleError(); err2 != nil {
                err = util.WrapErrorWithContextInfo(
                    fmt.Sprintf("error handling failed as well with err: %v", err2), err)
            }
        }()
        return util.WrapErrorWithContextInfo("firstFunction", err)
    }
    return nil
}
```

But in this case, the question stays open, if that is a real improvement of the error-handling, as we just have a defer statement in the same place, like we would have called the error handling directly.

If desired, the deferred error handling could also be written in the beginning like this:

```go
func deferredErrorHandling() (err error) {
    defer func() {
        if err != nil {
            if err2 := handleError(); err2 != nil {
                err = util.WrapErrorWithContextInfo(
                    fmt.Sprintf("error handling failed as well with err: %v", err2), err)
            }
        }
    }()

    if err := firstFunction(); err != nil {
        return util.WrapErrorWithContextInfo("firstFunction", err)
    }
    return nil
}
```

In the last case, we would have gained a separation of the regular code, and the code for error handling.
On the other hand, the readability of the code did also decrease.

Now let's assume, that a `secondFunction()`, needs some different error handling.
In this case, the question will arise, how to deal with that in a deferred error handling function.

If you had fixed errors you could check like this:

```go
func deferredErrorHandling() (err error) {
    defer func() {
        if err == errFirstFunction {
            if err1 := handleFirstError(); err2 != nil {
                err = util.WrapErrorWithContextInfo(
                    fmt.Sprintf("error handling of first function failed as well with err: %v", err1), err)
            }
        }
        if err == errSecondFunction {
            if err2 := handleSecondError(); err2 != nil {
                err = util.WrapErrorWithContextInfo(
                    fmt.Sprintf("error handling of second function failed as well with err: %v", err2), err)
            }
        }
    }()

    if err := firstFunction(); err != nil {
        return errFirstFunction
    }
    if err := secondFunction(); err != nil {
        return errSecondFunction
    }
    return nil
}
```

One thing to notice here is, that the defer function becomes quite complex, and wouldn't even work like this,
if we didn't use `errFirstFunction` and `errSecondFunction` as constants, but rather use `util.WrapErrorWithContextInfo("firstFunction", err)`.
One more problem arises here, in case that we wouldn't directly return `errFirstFunction` but rather continue with the common process in case, that the error-handling was successful.

Using `direct` error handling, this would be something like this:

```go
func directErrorHandling() error {
    if err := firstFunction(); err != nil {
        if err1 := handleFirstError() {
            return util.WrapErrorWithContextInfo(
                fmt.Sprintf("error handling of first function failed as well with err: %v", err1), err)
        }
    }
    if err := secondFunction(); err != nil {
        if err2 := handleSecondError() {
            return util.WrapErrorWithContextInfo(
                fmt.Sprintf("error handling of second function failed as well with err: %v", err2), err)
        }
    }
    return nil
}
```

In this case, you could say that it is best, if the code would be split up in smaller functions (`firstFunctionWithDeferredErrorHandling` and `secondFunctionWithDeferredErrorHandling`),
which include their error handling, so that it would stay similar to the examples above:

```go
func deferredErrorHandling() error {
    if err := firstFunctionWithDeferredErrorHandling(); err != nil {
        return util.WrapErrorWithContextInfo("firstFunctionWithDeferredErrorHandling", err)
    }
    if err := secondFunctionWithDeferredErrorHandling(); err != nil {
        return util.WrapErrorWithContextInfo("secondFunctionWithDeferredErrorHandling", err)
    }
}
```

## Panics

Finally, we want to have a closer look about deferred error handling in case of panics.

```go
func deferredErrorHandling() (err error) {
    defer func() {
        if errPanic := recover(); errPanic != nil {
            err = handlePanic()
        }
    }()
    panic("How could this have happend?")
}
```

To catch `panics`, we need a deferred error handling, like in the example above.
Here we are also using a `named return value` ("err error"), to return the failure to the calling function.
This can, of course, be combined with the error handling we've seen before.

## Conclusion

Deferred error handling has the advantage, that it can be executed even if a panic occurs.
However, the readability of the code decreases for more complex functions.
The most important part is, that the error handling does not contain complex logic, or many different function calls, which are included in the usual workflow,
but rather be extracted into a single function call, which could then do a possibly complex recovery.

Therefore, we currently see the use case of deferred functions mainly to do some cleanup,
like closing files in the end, which should be done in any case, or for handling panics.
However, panics should in general be avoided.
In our case, they are therefore only handled on the gRPC server interface.
