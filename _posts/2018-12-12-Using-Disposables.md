---
layout: post
title: "(Ab)using Disposables for Integration Testing"
published: true
---

# (Ab)using Disposables for Integration Testing

## Introduction


Have you written unit tests against your code but still had some type of issue in production? For example, your code reads and writes to a file. In your unit tests parsing and printing are great. But, when you go to production, something happens out of your control. File permissions, encoding, and faulty assumptions about third-party tools are all common issues. Or in another case, wouldn’t it be great to test those queries you’ve written for your database? I know you’ve written an [anti-corruption layer to create your domain objects](https://fsharpforfunandprofit.com/ddd/). But still, shouldn't we be doing integration tests in some of these scenarios?

Integration tests seem to get a bad reputation, and to be fair it’s understandable why. They tend to be hard to setup, brittle, and high maintenance when underlying assumptions change about your code base. Another big issue with them tends to be isolation and cleanup. There is a solution to some of these problems. We can create disposable objects that generate unique, isolated environments and clean themselves up. 


## What are disposables?

Dotnet has this concept of the ["disposable" pattern](https://docs.microsoft.com/en-us/dotnet/standard/design-guidelines/dispose-pattern). Digging deep into this is outside the scope of this blog. Here's the sales pitch: wouldn’t it be nice for something to clean itself up after it has left a scope and you don’t have to remember to call it yourself? Now, lets see a super simple example in F#: 


```FSharp
let myFunction () =
    use myresource = new Resource()
    myresource.DoOperation()
```

This function news up an object called `Resource` then calls `DoOperation` on that object. After that function is called, the `Dispose` method of the `Resource` object will be called.  Many dotnet developers are familiar doing this pattern with database connections, commands, and readers. For more in-depth examples please refer to [F# for Fun and Profit](https://fsharpforfunandprofit.com/posts/let-use-do/#use-bindings).


## The setup

My F# test framework of choice is [Expecto](https://github.com/haf/expecto). By default it runs all tests in parallel so it’s difficult to use any global or shared state.  Our goal is to have isolated environment for our tests, so this constraint helps us. To aid testing using disposables with async, I had to create a few helpers. 

```FSharp
type ParameterizedTest<'a> =
    | Sync of string * ('a -> unit)
    | Async of string * ('a -> Async<unit>)

let testCase' name test =
     ParameterizedTest.Sync(name,test)

let testCaseAsync' name test  =
    ParameterizedTest.Async(name,test)

let inline testFixture'<'a> setup =
    Seq.map (fun ( parameterizedTest : ParameterizedTest<'a>) ->
        match parameterizedTest with
        | Sync (name, test) ->
            testCase name <| fun () -> test >> async.Return |> setup |> Async.RunSynchronously
        | Async (name, test) ->
            testCaseAsync name <| setup test

    )
```

## Disposable Directory 

For the first example given, let's create a disposable directory.  It's requirements are that it should be isolated, cleanup after itself, and tell us which directory it created.

```FSharp
[<AllowNullLiteral>]
type private DisposableDirectory (directory : string) =
    static member Create() =
        let tempPath = IO.Path.Combine(IO.Path.GetTempPath(), Guid.NewGuid().ToString("n"))
        printfn "Creating directory %s" tempPath
        IO.Directory.CreateDirectory tempPath |> ignore
        new DisposableDirectory(tempPath)
    member x.DirectoryInfo = IO.DirectoryInfo(directory)
    interface IDisposable with
        member x.Dispose() =
            printfn "Deleting directory %s" x.DirectoryInfo.FullName
            IO.Directory.Delete(x.DirectoryInfo.FullName,true)
```

I defined a function to create the object `Create`, it uses dotnet's `GetTempPath` to find the designated temporary directory for your operating system, generates a guid and then creates that directory.  

The implementation of the IDisposable will delete this directory.  The `true` part of the parameter is for recursive deletion.

For exposing the directory path, I like using `DirectoryInfo`. We as an industry still have an obsession with primitives and its better to [use the type system](https://fsharpforfunandprofit.com/posts/designing-with-types-single-case-dus/) to describe your intent.

The print statements are for verbosity of this demo.

Lets show off the tests.


```FSharp
let withDisposableDirectory (test) = async {
  use disposableDirectory = DisposableDirectory.Create()
  do! test disposableDirectory.DirectoryInfo
}

let disposableDirectoryTests = [
  testCase' "Write file to directory" <| fun (directory : IO.DirectoryInfo) ->
        let fileName = IO.Path.Combine(directory.FullName, "Foo.txt")
        let contents = [
          "hello"
          "world"
        ]
        IO.File.WriteAllLines(fileName,contents)
  testCaseAsync' "Write file to directory async" <| fun (directory : IO.DirectoryInfo) -> async {
        let fileName = IO.Path.Combine(directory.FullName, "Foo.txt")
        let contents = [
          "hello"
          "mars"
        ]
        do! IO.File.WriteAllLinesAsync(fileName,contents) |> Async.AwaitTask
  }
]

[<Tests>]
let tests = 
  testList "Disposable Directory tests" [
    yield! testFixture' withDisposableDirectory disposableDirectoryTests 
  ]
```

The function `withDisposableDirectory` is a helper to create disposable directories for each test.

`disposableDirectoryTests` is a list of `testCase'` or `testCaseAsync'` that takes in a `IO.DirectoryInfo` so the test has access to the file path it created.

Finally, we're telling `expecto` to make these as tests in the last section.

Lets run them and see the output:

```
[15:45:30 INF] EXPECTO? Running tests... <Expecto>
Creating directory /var/folders/14/mp4bnvkn3fq5mqcgqscm5c040000gn/T/eeb2aac569b8417a8035c2d10c08c3bb
Creating directory /var/folders/14/mp4bnvkn3fq5mqcgqscm5c040000gn/T/bfa68a425def41e88e8c227f87cfd4b5
Deleting directory /var/folders/14/mp4bnvkn3fq5mqcgqscm5c040000gn/T/eeb2aac569b8417a8035c2d10c08c3bb
Deleting directory /var/folders/14/mp4bnvkn3fq5mqcgqscm5c040000gn/T/bfa68a425def41e88e8c227f87cfd4b5
[15:45:30 INF] EXPECTO! 2 tests run in 00:00:00.1009410 for Disposable Directory tests – 2 passed, 0 ignored, 0 failed, 0 errored. Success! <Expecto>
````

We can see our print statements telling us where it created the directories and when it deleted them. Each test has their own isolated folder. Neat!


## Disposable Database

For databases, we're going to take a similar approach.  I'll be using Postgres with Npgsql in this example. Setting up Postgres is outside the scope of this blog post.  We'll be doing something similar to the Disposable Directory example.  


```FSharp
type private DisposableDatabase (superConn : NpgsqlConnectionStringBuilder, databaseName : string) =
    static member Create(connStr) =
        let databaseName = System.Guid.NewGuid().ToString("n")
        DatabaseTestHelpers.createDatabase (connStr |> string) databaseName
        new DisposableDatabase(connStr,databaseName)
    member x.SuperConn = superConn
    member x.Conn =
        let builder = x.SuperConn |> DatabaseTestHelpers.cloneConnectionString
        builder.Database <- x.DatabaseName
        builder
    member x.DatabaseName = databaseName
    interface IDisposable with
        member x.Dispose() =
            DatabaseTestHelpers.dropDatabase (superConn |> string) databaseName

```

The `Create` function generates us a new database name, then creates that database with another helper called `createDatabase`.

The `Dispose` method will delete that database.

We're exposing both the `SuperConn` (The connection with the correct permissions to create that database) and the `Conn` (the connection to the database).

Since we'll want to run migrations we'll have to add a hook to running migrations

```FSharp

let withDisposableDatabase connectionString runMigration (test) = async {
  use disposableDirectory = DisposableDatabase.Create(connectionString)
  runMigration disposableDirectory.Conn
  do! test disposableDirectory.Conn
}
```

In our case, we'll be using [Simple.Migrations](https://github.com/canton7/Simple.Migrations).

```FSharp
let runMigration (migrationsAssembly : Assembly) (connStr : NpgsqlConnectionStringBuilder) =
    use connection = new NpgsqlConnection(connStr.ToString())
    let databaseProvider = new PostgresqlDatabaseProvider(connection)
    let migrator = new SimpleMigrator(migrationsAssembly, databaseProvider)
    migrator.Load()
    migrator.MigrateToLatest()
```


Now that the boiler plate is out of the way, we can write some tests!

```FSharp
let disposableDirectoryTests = [
  testCase' "Write to database" <| fun (connStr : NpgsqlConnectionStringBuilder) ->
        use conn = new NpgsqlConnection(connStr.ToString())
        use cmd = new NpgsqlCommand("INSERT INTO animals (name, birthday) VALUES ('Spunky', now())", conn)
        let added = cmd.ExecuteNonQuery()
        Expect.equal added 1 "Should have added 1 row"
       
  testCaseAsync' "Read count from database" <| fun (connStr : NpgsqlConnectionStringBuilder) -> async {
        use conn = new NpgsqlConnection(connStr.ToString())
        use cmd = new NpgsqlCommand("SELECT COUNT(*) FROM animals)", conn)
        let! reader = cmd.ExecuteReaderAsync() |> Async.AwaitTask
        let! canRead = reader.ReadAsync() |> Async.AwaitTask
        let count = reader.GetInt64(0)
        Expect.equal count 0L "Should have 0 rows"
  }
]
```

Every test will now get its own isolated environment to run queries in. Neat!


## Conclusion

By now you can see it does take some work to get integration tests setup. But with some effort, you can make them less brittle and behave as isolated environments. You could expand on this to even include spinning up and down Docker containers, or virtual machines with Vagrant. Now I admit, this feels like _abusing_ disposables. Yet there are benefits in doing this for test purposes.