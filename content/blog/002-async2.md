+++
title       = "async2"
date        = 2025-07-21
description = "A breakdown of the async2 experiment in .NET — how moving state machines into the runtime enables up to 38× faster exception handling and cuts async/await overhead."
+++

# Introduction
Before we get into the proposed changes to how async works in C#, let's step back and ask: why does any of this matter? What's actually wrong with the current model, and why are the .NET team looking for ways to improve it?

Let's start with the basic question: what problem does async actually solve?

Imagine a typical web API that queries a database. With single-threaded synchronous execution, the next request can only be processed after the previous one finishes — which is inefficient, because all the work falls on one CPU core while the rest sit idle.

![image](/assets/async2/image_o.png)

Modern CPUs have many physical and logical cores (HyperThreading/SMT). To make better use of the hardware and allow our application to handle requests in parallel, we could spin up a new OS thread for each request, letting multiple cores work simultaneously without waiting for previous requests to finish.

![image](/assets/async2/image_1v.png)

But here we hit another problem: spawning a thread is expensive. Creating a new thread (on x86_64/Linux):
- requires a syscall to the OS kernel, meaning a switch to kernel mode
- allocates a fair amount of memory:
    - kernel structures for managing the thread (`task_struct` and associated data)
    - TLS (Thread Local Storage) — for thread-local variables
    - TCB (Thread Control Block) — housekeeping info about the thread (thread ID, scheduling, signal mask)
    - kernel stack (16 KB, 2 pages, typically 8 KB each)
    - thread stack (8 MB by default, configurable via `pthread_attr_setstacksize()` or `ulimit -s`)

![image](/assets/async2/image_7.png)

And managing a large number of threads doesn't just cost memory and creation time. The scheduler ends up spending most of its cycles just managing threads — picking the next one to run, switching contexts, maintaining CPU caches — which can easily negate any parallelism gains.

To avoid overwhelming the scheduler, the natural move is to pre-create a fixed pool of threads and hand them tasks to execute.

![image](/assets/async2/image_e.png)

Which looks roughly like this:
```
while (true) 
{
	var task = tasksQueue.GetBlocking();
	
	task.Execute();
}
```

That's a reasonable approach. But most applications aren't CPU-bound — they're IO-bound: network calls (databases, HTTP/gRPC, Kafka/RabbitMQ), disk access. Consider fetching a user by ID from a database:
```
public void SomeMethod(Guid userGuid)
{
	// do something

	var user = usersRepository.GetUser(userGuid);

	usersService.DoSomethingWithUser(user);
}

```


![image](/assets/async2/image_p.png)

When a blocking operation (a network or disk call) is invoked, a syscall suspends the thread. The kernel marks the thread as unrunnable, parks it, and switches to running another thread. When the kernel later gets an interrupt indicating the data is ready — say, a network response arrived — the thread is queued for execution again and eventually resumes.

The result: the thread sits idle most of the time, and the constant context switches (plus the associated interrupt storms and cache flushes) add real overhead.

# The current situation
This is exactly what async is for. Tasks decouple work from threads — a single thread can switch between multiple tasks whenever one can't make progress. I'll keep this section brief, since there are already much more detailed writeups on how the current async implementation works under the hood.

Making our earlier example async looks like this:
```
public async Task HelloWorldAsync()
{
    Console.Write("Text: ");

    await _helloWorldService.HelloAsync();
    
    Console.Write(", ");
    
    await _helloWorldService.WorldAsync();
    
    Console.WriteLine("!");
}

```

We added the `async` modifier to the signature and `await` before the method calls. Clean on the surface. But what's hiding underneath?

> Generated code

```
[NullableContext(1)]
[Nullable(0)]
public class AsyncAwaitDemo
{
  private readonly HelloWorldService _helloWorldService;

  [AsyncStateMachine(typeof (AsyncAwaitDemo.<HelloWorldAsync>d__1))]
  [DebuggerStepThrough]
  public Task HelloWorldAsync()
  {
    AsyncAwaitDemo.<HelloWorldAsync>d__1 stateMachine = new AsyncAwaitDemo.<HelloWorldAsync>d__1();
    stateMachine.<>t__builder = AsyncTaskMethodBuilder.Create();
    stateMachine.<>4__this = this;
    stateMachine.<>1__state = -1;
    stateMachine.<>t__builder.Start<AsyncAwaitDemo.<HelloWorldAsync>d__1>(ref stateMachine);
    return stateMachine.<>t__builder.Task;
  }

  public AsyncAwaitDemo()
  {
    this._helloWorldService = new HelloWorldService();
    base..ctor();
  }

  [CompilerGenerated]
  private sealed class <HelloWorldAsync>d__1 : 
  /*[Nullable(0)]*/
  IAsyncStateMachine
  {
    public int <>1__state;
    public AsyncTaskMethodBuilder <>t__builder;
    [Nullable(0)]
    public AsyncAwaitDemo <>4__this;
    private TaskAwaiter <>u__1;

    public <HelloWorldAsync>d__1()
    {
      base..ctor();
    }

    void IAsyncStateMachine.MoveNext()
    {
      int num1 = this.<>1__state;
      try
      {
        TaskAwaiter awaiter1;
        int num2;
        TaskAwaiter awaiter2;
        if (num1 != 0)
        {
          if (num1 != 1)
          {
            Console.Write("Text: ");
            awaiter1 = this.<>4__this._helloWorldService.HelloAsync().GetAwaiter();
            if (!awaiter1.IsCompleted)
            {
              this.<>1__state = num2 = 0;
              this.<>u__1 = awaiter1;
              AsyncAwaitDemo.<HelloWorldAsync>d__1 stateMachine = this;
              this.<>t__builder.AwaitUnsafeOnCompleted<TaskAwaiter, AsyncAwaitDemo.<HelloWorldAsync>d__1>(ref awaiter1, ref stateMachine);
              return;
            }
          }
          else
          {
            awaiter2 = this.<>u__1;
            this.<>u__1 = new TaskAwaiter();
            this.<>1__state = num2 = -1;
            goto label_9;
          }
        }
        else
        {
          awaiter1 = this.<>u__1;
          this.<>u__1 = new TaskAwaiter();
          this.<>1__state = num2 = -1;
        }
        awaiter1.GetResult();
        Console.Write(", ");
        awaiter2 = this.<>4__this._helloWorldService.WorldAsync().GetAwaiter();
        if (!awaiter2.IsCompleted)
        {
          this.<>1__state = num2 = 1;
          this.<>u__1 = awaiter2;
          AsyncAwaitDemo.<HelloWorldAsync>d__1 stateMachine = this;
          this.<>t__builder.AwaitUnsafeOnCompleted<TaskAwaiter, AsyncAwaitDemo.<HelloWorldAsync>d__1>(ref awaiter2, ref stateMachine);
          return;
        }
label_9:
        awaiter2.GetResult();
      }
      catch (Exception ex)
      {
        this.<>1__state = -2;
        this.<>t__builder.SetException(ex);
        return;
      }
      this.<>1__state = -2;
      this.<>t__builder.SetResult();
    }

    [DebuggerHidden]
    void IAsyncStateMachine.SetStateMachine(IAsyncStateMachine stateMachine)
    {
    }
  }
}

```

The async method gets split into parts at compile time:
```
public async Task HelloWorldAsync()
{
	// state -1
    Console.Write("Text: ");

    await _helloWorldService.HelloAsync();
    
	// state 0
    Console.Write(", ");
    
    await _helloWorldService.WorldAsync();
    
	// state 1
    Console.WriteLine("!");
}

```

The method itself is replaced by something that looks nothing like the original:
```
[AsyncStateMachine(typeof (AsyncAwaitDemo.<HelloWorldAsync>d__1))]
[DebuggerStepThrough]
public Task HelloWorldAsync()
{
  AsyncAwaitDemo.<HelloWorldAsync>d__1 stateMachine = new AsyncAwaitDemo.<HelloWorldAsync>d__1();
  stateMachine.<>t__builder = AsyncTaskMethodBuilder.Create();
  stateMachine.<>4__this = this;
  stateMachine.<>1__state = -1;
  stateMachine.<>t__builder.Start<AsyncAwaitDemo.<HelloWorldAsync>d__1>(ref stateMachine);
  return stateMachine.<>t__builder.Task;
}
```

Here:
- A state machine is created
- A builder is created and set
- The `this` context is captured
- The state machine is started and its task is returned

The task comes from `AsyncTaskMethodBuilder`; the result is set when the state machine finishes running.

The state machine is advanced via `IAsyncStateMachine.MoveNext`, called for the first time by `AsyncTaskMethodBuilder.Start`. In our example: "Text: " is written to the console, then `HelloAsync()` is called, we get an awaiter, and check whether it's already completed (as would be the case for `Task.CompletedTask`). If the task isn't done yet, the state is updated (set to 0) and `AwaitUnsafeOnCompleted` is called on the builder, passing the awaiter and the state machine. Roughly speaking, this sets up a continuation on the awaiter that calls `MoveNext` when it completes.

Once you've worked through state 0, the rest follow the same pattern. The final state is -2. If an exception was caught, it's set on the Task; if everything completed normally, `SetResult()` is called.

# async2
## Background: Green threads
The story starts in 2023 with an experiment on green thread support, motivated by the release of Project Loom in the JVM. The main goal was to understand how hard it would be to add green threads to .NET and what problems would come with it.

Green threads are threads implemented in user space — scheduled and suspended not by the OS kernel but by the runtime itself.

The current async/await implementation in C# is based on coroutines: functions that cooperate and can yield the thread to other coroutines. The difference is one of abstraction level:
- Coroutines operate on functions that switch on native OS threads
- Green threads implement their own threading model, managed by the runtime — transparent to the functions themselves, which just see a regular thread

Since green threads look like ordinary threads to the code running in them, functions can be written as plain synchronous code — no `async` modifier, no `await` suspension points. This would eliminate the "colored functions" problem. But let's talk about that problem first.

Consider this:
```
// blue
public static int GetNumber()
{
	return 1;
}

// red
public static async Task<int> GetNumberAsync()
{
	var number = GetNumber();
	var inc = await GetIncrease();
	
	return numb + inc;
}

```

Two methods — a sync one and an async one. You can call the sync method from the async one, but not the other way around (well, technically you can, but you'll block the thread and need to be very careful about error handling). The compatibility matrix:

| Caller / Callee | sync |  async |
|:---------------------|:------------|:--------------|
|          sync |   yes |    no |
|         async |   yes |     yes |

Async functions also require their own syntax — you have to know the result is coming "sometime in the future" and `await` it explicitly.

Green threads let you forget about function color entirely. Every function is just a function. No different semantics, no restrictions on who can call what.

A performance comparison on a simple ASP.NET Core app showed a modest throughput drop (keep in mind this was an early experiment without significant optimization):

|   ASP.NET Plaintext |   Async | Green threads |
|:---------------------------|:---------------|:---------------------|
| Requests per second | 178,620 |       162,019 |

The experiment also surfaced several real problems:
- Calling async (TAP) methods from green threads forced the sync-over-async antipattern
- P/Invoke performance degraded significantly: 100M P/Invoke calls went from 300ms to 1800ms
- Interop with code-protection mechanisms like shadow stacks was complex


The conclusion: you could make green threads as fast as, or even faster than, current async — but it wouldn't solve performance problems in specific scenarios. Adding yet another paradigm to the language would create confusion and require compatibility layers. The .NET team decided to focus on runtime-level async support rather than introducing a wholly new model.

## Runtime handled asyncs
As we've seen, the current async/await implementation is completely invisible to the runtime. From the runtime's perspective, there are only callbacks queued for execution when an async operation completes.

Let's look at what runtime-level async support actually means. I'll refer to it as async2 for simplicity.

async2 is an experiment exploring whether the async programming model in .NET could be improved by moving async state machine generation into the runtime itself. The stated goals:
- Determine whether runtime-generated async state machines are meaningfully faster than the existing compiler-generated model
- Determine whether an async variant with some semantic differences could remain compatible with existing code written in the original style


A new method modifier — `async2` — was introduced for the experiment, though this is temporary. If the approach ships, it'll use the existing `async` keyword. The requirement to return a wrapper type (`Task`/`ValueTask`) was also kept.

### Compatibility with async1
One critical constraint was compatibility with the existing async/await (TAP-based) model. In any future version of .NET that ships async2, you'd need seamless calling between async2 → async1 and async1 → async2. Fragmenting the codebase into "async1 code" and "async2 code" would just create confusion and slow adoption.

Think of async1 and async2 as two worlds that need bridges between them. Those bridges are *thunk functions*, which make the two models interoperable.

Starting with async1 → async2 (where an async1 function calls an async2 function): when the runtime sees this call, it generates a "glue function" (thunk) on the fly that:
- Saves the current `ExecutionContext` and `SynchronizationContext`
- Initiates the call to the target async2 function
- Converts the native result to a `Task`/`ValueTask` object

```
// Pseudocode thunk for Task<int> -> async2 int
Task<int> ThunkAsync(int param)
{
    var state = new RuntimeTaskState<int>();
    try 
    {
        int result = TargetAsync2Method(param);
        return state.FromResult(result);
    }
    catch (Exception ex)
    {
        return state.FromException(ex);
    }
}
```

If the target async2 function suspends, the thunk wraps it in a Task, which gets returned up the call stack and awaited by the classic state machine.

For async2 → async1 (an async2 function calling an async1 function), there's also a thunk that:
- Adapts `Task.GetAwaiter()` to runtime primitives
- Uses the low-level `AwaitAwaiterFromRuntimeAsync` methods

Which looks something like:
```
public async2 int GetDataAsync()
{
    var awaiter = _httpClient.GetAsync(...).GetAwaiter();
    if (!awaiter.IsCompleted)
    {
        RuntimeHelpers.AwaitAwaiterFromRuntimeAsync(awaiter);
    }
    return awaiter.GetResult();
}
```

There are some intentional semantic differences that can affect behavior:
- `ExecutionContext`
Unlike async1, where it's automatically captured on entry to an `async` method, in async2 the context propagates up the call stack. This means:
    - Changes to `AsyncLocal<T>` inside async2 are visible to the caller

    ```
    async Task DoOuterAsync() 
    {
        AsyncLocal<string> local = new AsyncLocal<string> { Value = "Outer" };
        await DoInnerAsync();
        Console.WriteLine($"DoOuter: {local.Value}"); // async1: "Outer", async2: "Inner"
    }
    
    async Task DoInnerAsync() 
    {
        AsyncLocal<string>.Value = "Inner";
        await Task.Yield();
    }
    ```
    - A `CaptureContext` mode was added to emulate async1 behavior, but it comes with a 20–30% performance cost
- `SynchronizationContext`
Managed via the `[ConfigureAwait]` attribute:
    ```
    [assembly: ConfigureAwait(false)]
    ```

In async2, the context is captured **at the point where control is returned**, not at method call time as in async1. The developers consider this unlikely to be a problem in practice.

Not all async2 features translate cleanly to async1 for compatibility purposes. For instance, the stack-unwind approach allows byref-like parameters to be passed to async functions. But when those parameters cross the async1 ⟷ async2 boundary, the thunk function throws `InvalidProgramException`.

Also, by default, async2 methods are hidden from reflection via `Type.GetMethods()` — you can find them only by adding `BindingFlags.Async2Visible`.

### Stack unwind (tasklets)
This approach exploits the fact that a function is already a state machine: the instruction pointer (IP) serves as the resume index, and the current state is the stack frame plus saved registers. Several JIT changes were needed to implement this prototype:

1. Prohibit binding `InlinedCallFrames` to the stackframe chain across suspends (or disable P/Invoke inlining in these methods)
2. Pass frame pointers as ByRef
3. Report all local variable pointers as ByRef
4. Force `AwaitAwaiterFromRuntimeAsync` to be treated as an async2 method even when not explicitly marked as such (due to compiler limitations)


For suspension/resumption, the function's stack needs to be unwound and saved (along with local variables and registers).

`AwaitAwaiterFromRuntimeAsync` handles this — it captures every stack frame from itself up to `ThunkAsync` or `ResumptionFunc` and packs them into `Tasklet` objects, forming a linked list. Conceptually:
```
struct Tasklet
{
    public IntPtr StackPointer; // Stack pointer (SP)
    public IntPtr InstructionPointer; // Instruction pointer (IP)
    public object[] LocalVariables; // Local variables
    public object[] GCReferences; // References to heap objects
}
```

Tasklets are registered with the GC but handled specially to account for ByRef pointers. After unwinding, the tasklets and the awaiter are stored in a thread-local variable for use by `ThunkAsync`/`ResumptionFunc`.

So what are `ThunkAsync`/`ResumptionFunc`?

As noted above, `ThunkAsync` is the compatibility shim between async1 and async2. In the stack-unwind approach, it specifically handles returning a `Task`/`ValueTask` — necessary only for async1 compatibility, since the underlying approach doesn't actually need wrapper types. Without that constraint, you could get rid of the red/blue function distinction entirely. But as with green threads, changes of that magnitude are too painful to introduce all at once. The function looks roughly like this (as a conceptual sketch, not actual implementation):

```
Task<ReturnType> ThunkAsync(ParameterType param1, ParameterType2 param2, ...)
{
    // A variant of this helper type will be defined for each Task/ValueTask/Task<TResult>/ValueTask<TResult>
    System.Runtime.CompilerServices.RuntimeTaskState<ReturnType> runtimeTaskState = new;

    // If the Task is parameterized, this is the return value type.
    ReturnType result;

    runtimeTaskState.Push();
    try
    {
        try
        {
            // NOTE: there's a special case for instance methods on value types. For them,
            // the thunk creates a boxed instance of the value `this` and calls the method on it
            result = TargetMethod(param1, param2, ...);
        }
        catch (Exception ex)
        {
            return runtimeTaskState.FromException(ex);
        }
        return runtimeTaskState.FromResult(result);
    }
    finally
    {
        runtimeTaskState.Pop();
    }
}
```

`ResumptionFunc` is the function that resumes suspended operations.

The dispatcher takes the collection of Tasklets and resumes the one at the top of the stack. When it returns, the next Tasklet is popped and executed. The structures are maintained so the EH (exception handler) stack walker can locate the list of stack frames still held by a Tasklet.
```
static void ResumptionFunc(Stack<Tasklet> tasklets)
{
	while ((var tasklet = tasklets.Pop()) != null)
	{
		// Restore the stack frame
		RestoreStackPointer(tasklet.StackPointer);
		RestoreInstructionPointer(tasklet.InstructionPointer);
		RestoreLocalVariables(tasklet.LocalVariables);
		RestoreGCReferences(tasklet.GCReferences);
		
		// Transfer control
		ContinueExecution();
	}
}

```

### JIT-compiled state machine (Continuations)
In this prototype, state machine generation is moved from the compiler to the runtime. When an async2 function awaits another async2 function, the JIT generates suspension code to capture the current state into a continuation. It also generates code for each suspension point that can resume execution from that continuation.

A `Continuation` holds the current state, resume flags, local variables, heap references, and a pointer to the next `Continuation` to execute after this one completes — forming a linked list.
```
internal sealed unsafe class Continuation
{
	public Continuation? Next;
	public delegate*<Continuation, Continuation?> Resume;
	public uint State;
	public CorInfoContinuationFlags Flags;
	public byte[]? Data;
	public object[]? GCData;
}
```

At runtime, the original code is modified to accept an additional `Continuation` parameter representing the current function state on resumption. If this is the first call, the continuation is null. The runtime also generates a helper method used for resumption.

Two runtime methods are provided:
```
// Get the Continuation created by the called method
[Intrinsic]
internal static Continuation? Async2CallContinuation() => null;

// Suspend the function with the provided Continuation
[Intrinsic]
private static void SuspendAsync2(Continuation continuation) => throw new UnreachableException();
```

Consider this example:
```
static async2 Task<int> Foo(int a, int b)
{
	if (a == null || b == null)
	{
		return 0;
	}

	await Task.Yield();

	int result = a + b;

	await Task.Yield();

	return result;
}
```

The JIT transforms this during compilation. The compiled form — remembering that the JIT works in machine instructions, not C# — looks conceptually like:
```
static int Foo(Continuation? continuation, int a, int b)
{
    if (continuation == null)
    {
        // First call
        if (a == 0 || b == 0)
        {
            return 0;
        }

        var awaiter1 = Task.Yield().GetAwaiter();
        if (!awaiter1.IsCompleted)
        {
            // Suspend
            var continuation = new Continuation
            {
                State = 1,
                Data = SaveStackAsBytes(), // Save the stack
                Resume = &IL_STUB_AsyncResume_Foo // Pointer to the resume method
            };
           	
			// handles continuation and sets its reference
			// in register RCX (for amd64)
			StubHelpers.SuspendAsync2(continuation);

			return default;
        }

        // synchronous continuation
        awaiter1.GetResult();
        int result = a + b;

        var awaiter2 = Task.Yield().GetAwaiter();
        if (!awaiter2.IsCompleted)
        {
            // Suspend
            var continuation = new Continuation
            {
                State = 2,
                Data = SaveStackAsBytes(),
                Resume = &IL_STUB_AsyncResume_Foo
            };
            
			StubHelpers.SuspendAsync2(continuation);
			
			return default;
        }

        // synchronous continuation
        awaiter2.GetResult();
		
        return result;
    }
    else
    {
        // Resume execution
        switch (continuation.State)
        {
			// Resume after the first await
            case 1:
                // restore stack values and heap references
                RestoreStack(continuation);

                int result = a + b;

                // Second await
                var awaiter2 = Task.Yield().GetAwaiter();
                if (!awaiter2.IsCompleted)
                {

            		// Suspend
            		var continuation = new Continuation
            		{
                		State = 2,
                		Data = SaveStackAsBytes(),
                		Resume = &IL_STUB_AsyncResume_Foo
            		};
                    
					StubHelpers.SuspendAsync2(continuation);
					
					return default;
                }

                // synchronous continuation
                awaiter2.GetResult();

                return result;
			// Resume after the second await
            case 2:
                
                RestoreStack(continuation);

                return result;
        }
    }
}

```

Notice that the `Resume` field of `Continuation` holds a pointer to `IL_STUB_AsyncResume_Foo`. This is another runtime-generated stub:
```
static Continuation? IL_STUB_AsyncResume_Foo(Continuation continuation)
{
	delegate*<Continuation, int, int, int> foo = &Foo;
	int result = foo(continuation, 0, 0);

	// Get the continuation returned by the previous async2 function call
	Continuation? newContinuation = StubHelpers.Async2CallContinuation();

	if (newContinuation == null) // the method completed
	{
		// Write the value for the next continuation to consume
		Unsafe.Write(ref continuation.Next.Data[index], result);
	}
	
	return newContinuation
}

```

This casts a function pointer to a delegate where the first parameter is a `Continuation` — since the stub is only called on resumption, the continuation is never null here.

The function is then called, executing the synchronous portion of the code. Default values are passed as arguments because the actual data will be restored from the `Continuation` onto the stack. Those arguments still need to be there so the stack frame is set up correctly for the restore to work.

If `Foo` needs to suspend again, the reference to the new `Continuation` is placed in register `RCX` (on amd64). The stub retrieves it via `Async2CallContinuation`, which returns whatever continuation the called function created — or null if it didn't.

Two cases, then:
- `newContinuation == null`: the function finished. The return value is written into the next continuation's data
- `newContinuation != null`: the function needs to suspend again. The stub returns the new continuation, which will be passed back as an argument on the next resumption

Since `Continuation` is a normal parameter, no special intrinsic is needed to pass it to the target function. The resume stub just calls the target via `calli` with a signature that includes the continuation, the same way IL instantiation stubs work.

## Span/byref support

The current situation: C# doesn't allow byref or byref-like types in async functions in any form — as parameters, local variables, or spans. The reason is that byrefs can't be captured as display types. The restriction applies to all uses, even ones that wouldn't actually require capture.

There are a few cases where C# does handle temporary byrefs by decomposing the expression that produces them, capturing the pieces, and replaying them at the point of use:
```
staticArray[i].Field += await Somehting(); 
```

This handles some cases, but the generated code is far from ideal.

In async2, byref and span support isn't a flat "no" — but it comes with caveats. There are roughly two kinds of byrefs an async implementation needs to handle:

1. **byrefs pointing to the heap** are tricky because GC byref reporting is only supported on the stack. Any alternative storage needs a scheme for reporting saved variables for marking and updating. The main concern is performance — with potentially thousands of concurrent tasks, O(n) algorithms during GC pauses become a real problem. Some mitigations were explored and measured as part of this experiment.
2. **byrefs pointing to the stack** don't need GC reporting, but they need to be adjusted when the containing frame is reactivated — since the object they point to is either in the current frame (now at a different stack position) or in one of the still-suspended caller frames (so not on the stack at all). When `ref Span` params are involved, updates at suspension time are also needed to keep byref chains safely traversable.

It would be a poor programming model if only one kind of byref were allowed and not the other, since the distinction may only be known at runtime.

The experiment considered two implementation strategies:
- **Stack unwind with 1:1 stack capture for suspended frames.** In this model, heap-pointing or async-caller-pointing byrefs/spans can be supported, since there's a mechanism for reporting them to the GC. There are performance concerns, but they're not unsolvable.
- **State machine with managed storage for captured variables.** The current implementation doesn't support byref/byref-like capture, but it uses standard GC reporting mechanisms — which brings real advantages, like no O(n) algorithms during GC pauses. The developers have ideas about capturing byrefs and transforming them as `{object, offset}` at the next GC cycle, to avoid the constant O(n) cost.

## Performance
To weigh the trade-offs and find the bottlenecks, the team ran several benchmarks.

### Cross-model call overhead (async1 ↔ async2)
Since both async models need to interoperate, their interaction cost matters.

The test calls an async method in a loop, each iteration invoking a near-empty async method. The test runs for a fixed duration and counts iterations. The caller always returns `Task`; the callee reads a global variable and, if that variable equals 0, awaits `Task.Yield()`. (It never equals 0.)

![async-mincallcost](/assets/async2/async-mincallcost.svg)

Key takeaways:
- async1 ↔ async2 interop is not problematically slow
- async2 calling async1 is actually faster than async1 calling async1
- async2-to-async2 call cost is substantially lower
- async1 calling async2 in the stack-unwind model was significantly slower (not shown on the graphs)


### Exception handling at various stack depths

One of the main goals was to speed up exception handling in async functions, which has known problems today. The test loops for 250 ms, counting iterations. Each iteration recursively calls a function to a given stack depth (optionally through `try`/`finally` blocks), suspends the async method by awaiting `Task.Yield()` or not, and finally either returns or throws a freshly created exception.

These benchmarks were only run for the JIT state machine implementation — the stack-unwind variant is noticeably slower and doesn't support exception handling.

![image](/assets/async2/image.png)

Notable observations:
- Throwing an exception is extremely expensive when the code path could have completed synchronously
- The overhead from suspending an async function and bouncing to the thread pool is significant
- Exception handling in runtime-generated async functions is dramatically faster. As stack depth increases, the runtime async model doesn't suffer the same stack-traversal penalty as the regular exception handling stack walker

At 64 frames of stack depth:
- **Synchronous path (NoSuspend)**: async2 handles exceptions **8× faster** (3,936 ops/s vs 484 for async1)
- **Suspended path (Suspend)**: the gap reaches **38×** (17,565 ops/s vs 458)


### **Raw async performance: `async Task` vs runtime-generated async functions**
The graph below shows raw exception-throw performance at various stack depths.

![perf-of-throwing](/assets/async2/perf-of-throwing.svg)

There's no meaningful difference between `ValueTask` and `Task`. async2 is noticeably faster — the design avoids the costly stack traversal, and the curve is dominated by the cost of suspending execution to handle the exception.

The next graph shows runtime async performance relative to classic async for specific code patterns.

![async-vs-async2-relative-performance](/assets/async2/async-vs-async2-relative-performance.svg)

Runtime-generated async is consistently faster. Numbers above 100% indicate a speedup. The graph compares runtime-generated async against compiler-generated async for the specific code under test.

### **Impact on `ExecutionContext` and `SynchronizationContext` capture/restore**
The Async2Capture test variant shows the performance impact of capturing and restoring `ExecutionContext` and `SynchronizationContext` in the runtime async model. Higher bars are better; if capture were free, everything would sit at 100%.

![async2-vs-async2capture-relative-performance](/assets/async2/async2-vs-async2capture-relative-performance.svg)

Comparing current async against runtime async with context capture:

![async-vs-async2capture-relative-performance](/assets/async2/async-vs-async2capture-relative-performance.svg)

### **Impact of `try`/`finally` blocks on exception-throw performance**
![throw-perf-with-different-number-of-finally-on-stack](/assets/async2/throw-perf-with-different-number-of-finally-on.svg)

Much of the exception-handling performance gain in the async2 prototype comes from fast stack traversal during exception dispatch. In the runtime-generated model, frames with no exception handlers can be skipped with a simple flag check. This is the opposite of compiler-generated IL async code, where the compiler emits a `catch` even when there's no `try` block written in the C# source.

### **GC impact (heap size, pauses)**
A straightforward test: keep 10,000 suspended tasks alive at a given moment, each with 100 stack frames holding at least 1 object and 1 integer live across an `await` (and therefore requiring capture). At the point of suspension, the test forces several Gen0 GC cycles and measures average GC time and managed heap size. To limit variables, the test is single-threaded and uses Workstation GC.

|                                   | Heap size | Peak working set | GC pause (Gen0) |
|:----------------------------------|:-----------------|:------------------------|:-----------------------|
|                     async1 |    130 Mb |           148 Mb |          120 us |
|      async2 (Stack unwind) |     28 Mb |           499 Mb |          480 us |
| async2 (JIT state-machine) |    157 Mb |           328 Mb |          100 us |

A few things worth noting here:

1. Initially, GC pause times for async2 with stack unwind were around 48,690 us — completely unacceptable for Gen0 collections. After implementing proper Tasklet reporting to the GC, that came down significantly.
2. The stack-unwind variant uses much less heap memory. That's because it uses native `malloc` (unmanaged memory). But the working set is larger than the others, since the entire stack and registers need to be stored.
3. The JIT state machine heap size depends on the JIT optimization level. With TieredCompilation off: 157 MB. With it enabled, it peaks around 300 MB in early tiers before settling to ~100 MB later.

# Alternative approaches
## Go
Goroutines are one of Go's cornerstones. They're stackful coroutines with a dynamically-sized stack (starting minimal, growing as needed). Go has no async/await. Everything executes synchronously; blocking happens on channels (`chan`) or special operations that the runtime intercepts. When a blocking call happens (say, a disk read), the runtime makes a non-blocking call instead — through epoll/kqueue or a syscall on a separate thread — and parks the goroutine until the result is available.

That's how Go avoided colored functions while still getting a modern, efficient concurrency model.

## Java
Project Loom shipped with Java 21, bringing virtual threads to the JVM. Virtual threads can suspend and resume, and can switch native threads between suspensions — but from the developer's perspective, they're just threads. No modifiers, no "async" functions, no await points. The runtime intercepts blocking operations and runs them on separate threads, parking the virtual thread that called them.

The underlying mechanism in Go and Project Loom is the same: the runtime intercepts blocking operations and executes them in a non-blocking way. The difference is how long it took. Go had this from the start; for the JVM, changes of this scale took five years:
- Project Loom started in 2017
- Preview in 2021
- Preview feature in JDK 19 (STS) in 2022
- Generally available in JDK 21 (LTS) — though widespread adoption across the ecosystem and libraries will take a few more years

# Conclusion
The async2 experiment represents a significant step in .NET's async story — one aimed at improving performance and simplifying the model compared to current async/await (async1). The results are compelling: up to 38× faster exception handling at deep call stacks, and meaningfully lower overhead for async-to-async calls. The green threads experiment, inspired by Project Loom, was ruled out due to performance and compatibility concerns — pointing the .NET team toward improving the existing async model rather than adding a new one.

Two implementation approaches were explored: stack unwind (tasklets) and JIT-generated state machines (continuations). The JIT approach proved preferable — better compatibility with the existing .NET infrastructure, lower overhead, and stronger performance across most scenarios. Stack unwind supports more complex constructs like byref and span, but proved less practical due to GC pause times and implementation complexity.

For now, async2 remains an experiment in runtimelab on the `feature/async2-experiment` branch, which is still being maintained — unlike `feature/green-threads`. It uses the temporary `async2` modifier and provides async1 compatibility through thunk functions, but limitations around byref, span, and similar constructs still need work. The .NET team's bet is on gradual improvement of the existing model, minimizing risk while keeping compatibility with existing code.

The long-term picture: async2 results may be integrated into future .NET versions incrementally, giving developers a faster and more intuitive async experience. The ultimate goal — removing the need for an explicit sync/async split — would make the programming model significantly more natural.

UPD.: Thanks to @Nagg for pointing out in the comments that runtime async support has already been merged ([here](https://github.com/dotnet/runtime/pull/114675) and [here](https://github.com/dotnet/runtime/pull/114861)) upstream — that's where development is now happening.

For those who want to go deeper, here are the main sources used:
1. [Runtime Handled Tasks Experiment](https://github.com/dotnet/runtimelab/blob/feature/async2-experiment/docs/design/features/runtime-handled-tasks.md)
2. [Green Threads Technical Report](https://github.com/dotnet/runtimelab/blob/feature/green-threads/docs/design/features/greenthreads.md)

 --- 
Sources:
- [https://habr.com/ru/articles/470830/](https://habr.com/ru/articles/470830/)
- [https://steven-giesel.com/blogPost/59752c38-9c99-4641-9853-9cfa97bb2d29](https://steven-giesel.com/blogPost/59752c38-9c99-4641-9853-9cfa97bb2d29)
- [https://github.com/dotnet/runtimelab/issues/2398](https://github.com/dotnet/runtimelab/issues/2398)
- [https://steven-giesel.com/blogPost/720a48fd-0abe-4c32-83ac-26926d501895/the-state-machine-in-c-with-asyncawait](https://steven-giesel.com/blogPost/720a48fd-0abe-4c32-83ac-26926d501895/the-state-machine-in-c-with-asyncawait)
- [https://steven-giesel.com/blogPost/69dc05d1-9c8a-4002-9d0a-faf4d2375bce/c-lowering](https://steven-giesel.com/blogPost/69dc05d1-9c8a-4002-9d0a-faf4d2375bce/c-lowering)
- [https://journal.stuffwithstuff.com/2015/02/01/what-color-is-your-function/](https://journal.stuffwithstuff.com/2015/02/01/what-color-is-your-function/)
