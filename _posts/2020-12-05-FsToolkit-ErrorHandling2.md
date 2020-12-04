---
layout: post
title: "FsToolkit.Errorhandling 2.0"
published: true
---

# FsToolkit.Errorhandling 2.0

### TOC

- [Introduction](#Introduction)
- [Applicatives](#Applicatives)
- [Ply](#Ply)
- [Source overload](#Source-overload)
- [Conclusion](#Conclusion)

## Introduction

I'm happy to announce [FsToolkit.ErrorHandling 2.0](https://github.com/demystifyfp/FsToolkit.ErrorHandling/releases/tag/2.0.0).  I'll go over some of the major changes that this release has brought. I'll be covering things library consumers can use such as:
  - [Applicatives](https://devblogs.microsoft.com/dotnet/announcing-f-5/#applicative-computation-expressions) in [F# 5.0](https://devblogs.microsoft.com/dotnet/announcing-f-5/)
  - [Ply](https://github.com/crowded/ply)

And I'll be covering for creators of computation expressions:
  - [Source member](https://stackoverflow.com/questions/35286541/why-would-you-use-builder-source-in-a-custom-computation-expression-builder) (I can't find a better resource on it than this StackOverflow post.)

## Applicatives

[Applicatives](https://devblogs.microsoft.com/dotnet/announcing-f-5/#applicative-computation-expressions) were introduced [F# 5.0](https://devblogs.microsoft.com/dotnet/announcing-f-5/). We initially added support during F# 5.0 for it [PR #75](https://github.com/demystifyfp/FsToolkit.ErrorHandling/pull/75) which was released in [1.3.0](https://github.com/demystifyfp/FsToolkit.ErrorHandling/releases/tag/1.3.0) but it's worth covering it here in the 2.0 announcement.  

We've added support to the `result` `asyncResult` `taskResult` and `jobResult` builders, so you can take advantage of it like:

```fsharp
let myThing : Result<int*int*int ,string> = result {
    let! red = Ok 42
    and! green = Ok 100
    and! blue = Ok 0
    return red,green,blue
}

Result: 
Ok (42,100,0)
```

The main attraction here is this is a performance benefit. The one downside with this current implementation is that if there are multiple errors you will only get the first error.

```fsharp
let myThing : Result<int*int*int ,string> = result {
    let! red = Ok 42
    and! green = Error "Can't be negative"
    and! blue = Error "'daba dee daba die' isn't a color"
    return red,green,blue
}

Result:
Error "Cant be negative"
```

It would be nice to have a list of all possible errors.  This is where the `validation` CE steps in. The type is encoded like

```fsharp
type Validation<'a,'err> = Result<'a, 'err list>
```

So you can see we're still using a `Result` under the hood but now we're going to have a `list` of `'err` for the return type. So if you change the `result` to `validation`

```fsharp
let myThing : Result<int*int*int ,string> = valitation {
    let! red = Ok 42
    and! green = Error "Can't be negative"
    and! blue = Error "'daba dee daba die' isn't a color"
    return red,green,blue
}

Result:
Error ["Cant be negative"; "'daba dee daba die' isn't a color"]
```

Currently there is no implementation for `asyncValidation`, `taskValidation`, or `jobValidation`. I haven't seen the use case for this but I'm open to Pull Requests for them!

## Ply

Ply was added in [PR #97](https://github.com/demystifyfp/FsToolkit.ErrorHandling/pull/97) thanks to [Nino Floris](https://github.com/NinoFloris). This is the breaking change that makes us go to 2.0 since we're replacing [TaskBuilder.fs](https://github.com/rspeele/TaskBuilder.fs) with [Ply](https://github.com/crowded/ply). Ply "is a high performance TPL library for F#" which also brings along a bigger set of [Computation Expressions](https://github.com/crowded/ply#builders). (copying for brevity)


| builder          | return type   | tfm                           | namespace                            |
|---------------|---------------|-------------------------------|--------------------------------------|
| `task`        | Task<'T>      | netstandard2.0, netcoreapp2.1 | FSharp.Control.Tasks.Builders        |
| `vtask`       | ValueTask<'T> | netcoreapp2.1                 | FSharp.Control.Tasks.Builders        |
| `unitTask`    | Task          | netstandard2.0, netcoreapp2.1 | FSharp.Control.Tasks.Builders        |
| `unitVtask`   | ValueTask     | netcoreapp2.1                 | FSharp.Control.Tasks.Builders        |
| `uvtask`      | ValueTask<'T> | netcoreapp2.1                 | FSharp.Control.Tasks.Builders.Unsafe |
| `uunitTask`   | Task          | netcoreapp2.1                 | FSharp.Control.Tasks.Builders.Unsafe |
| `uunintVtask` | ValueTask     | netcoreapp2.1                 | FSharp.Control.Tasks.Builders.Unsafe |
| `uply`        | Ply<'T>       | netstandard2.0,netcoreapp2.1  | FSharp.Control.Tasks.Builders.Unsafe |

Under the covers we're only using the `task` builder since it's the most flexible but in higher performance scenarios you may want to look at the other builders and decide if making your own `vtaskResult` implementation makes sense.

On the initial PR for Ply, we were using Context Insentive tasks (e.g. `ConfigureAwait(false)`). In [Issue #106](https://github.com/demystifyfp/FsToolkit.ErrorHandling/issues/106#issuecomment-728674251) it was brought up this can have performance impacts and there's some compelling evidence as to why not to use `ConfigureAwait(false)`. With [PR #107](https://github.com/demystifyfp/FsToolkit.ErrorHandling/pull/107) by [Swoorup](https://github.com/Swoorup) we are now using Context Sensitive Tasks. I'm still unclear as to what effects this may have on code-bases. From my understanding, this may affect Desktop UI application but not for Server applications.  Please open any issues if this does impact your applications. 

To convert from using TaskBuilder.fs (assuming you don't have anything else pulling in Taskbuilder.fs) `FSharp.Control.Tasks.V2.ContextInsensitive` with `FSharp.Control.Tasks`.

## Source overload

The last thing I'd like to cover doesn't affect consumers of this library but is something that authors of Computation Expression libraries may be unfamiliar with. 

I was investigating how [FSharp.Control.FusionTasks](https://github.com/kekyo/FSharp.Control.FusionTasks) was handling its binding overloads and I came across an overload I was not familiar with, the [Source method](https://github.com/kekyo/FSharp.Control.FusionTasks/blob/master/FSharp.Control.FusionTasks/AsyncExtensions.fs#L194-L210). I came across [this StackOverflow post](https://stackoverflow.com/questions/35286541/why-would-you-use-builder-source-in-a-custom-computation-expression-builder) that explained it.

The main issue I faced with the current implementation of the `result` and friends was maintainability.  I had to add _so many_ overloads for `BindResult` and `MergeSources` to get things to work correctly for different types, such as `Choice`. What the `Source` member does is [remove a lot of that repetitive, unmaintainable code](https://github.com/demystifyfp/FsToolkit.ErrorHandling/pull/83/files). Essentially, `Source` lets an author teach a Computation Expression how to covert different types to a type it knows natively.

## Conclusion

This was a pretty big release with regards to changes.  Please take it for a spin!