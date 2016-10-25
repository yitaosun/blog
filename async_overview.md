# Async Overview

## `async`/`await` Internals

### The Setup

`async`/`await` is .NET's the most recent attempt to manage asynchronous programming.  On the surface, the
code looks similar to normal, synchronous functions.  The fun part is how a compiler breaks the function down
into manageable executable pieces for threads to consume, and how to coordinate transitions between threads.

Take this sample code from (MSDN)[https://msdn.microsoft.com/en-us/library/mt674882.aspx],

```
async Task<int> AccessTheWebAsync()
{
    HttpClient client = new HttpClient();

    Task<string> getStringTask = client.GetStringAsync("http://msdn.microsoft.com");

    DoIndependentWork();

    string urlContents = await getStringTask;

    return urlContents.Length;
}
```

This is a pretty straighforward piece of asynchronous programming.  The execution flow breaks down into

- Thread1 enters `AccessTheWebAsync`
- Thread1 schedules `client.GetStringAsync(("http://msdn.microsoft.com")` to execute on Thread2
- Thread1 blocks and waits at `await getStringTask`
- Thread2 executes `client.GetStringAsync(("http://msdn.microsoft.com")` and notifies Thread1
- Thread1 continues and returns

_insert rationale here_

What is actually executed?

```
async Task<int> AccessTheWebAsync()
{
    HttpClient client;
    Task<string> getStringTask;
    string urlContents;
    // Section 1
    {
        client = new HttpClient();

        getStringTask = client.GetStringAsync("http://msdn.microsoft.com");

        DoIndependentWork();

        // Thread1 blocks
        urlContents = await getStringTask;
    }

    // Section 2
    {
        return urlContents.Length;
    }
}
```

I've taken the liberty to re-organize the function into 2 parts.  Thread1 executes Section 1 continuously
until it reaches the end of Section 1, blocks, and resumes execution into Section 2.  Section 1 and Section 2
can be thought of as regions where Thread1 has un-interrupted execution.  This is how the compiler understands
this function, and how compilers understand asynchronous functions in general, as sections of continuous
execution where transitions between the sections are separated by thread joins.  Naturally, the compiler
breaks the function into a state machine so it can keep track of which section is executing and the state
of the function when it enters and leaves a section.

Looking at the compiled byte-code, this function will have 2 parts.  First, the function itself.  Second,
the generated class that stores the function state.

## Instrumentation

Instrumenting this function is trivial, but to keep track of the top-level function enter and exit, the state-
machine must be taken into account.