# Analysis of testing styles in Elixir

## Table of Contents
- Critical thinking
- Definitions
- The Sample
- Example implementations
- Fake vs. Mock
- Reusability
- Intrusiveness to Code
- Test setup
- Concurrency
- Static vs Dynamic Typing
- What is my preference and why?
- Is an ideal achievable?
- Feedback

## Critical Thinking
My intention is to seek the fact of the matter regarding the advantages and disadvantages of approaches to testing with Elixir.   I don’t have an opinion on what the fact of the matter turns out to be, I am curious about what the fact of the matter is.  While I consider it wise to seek out perspectives of people more experienced than me or smarter than me, experience and intelligence are just tools, they don’t make something true.  This is my attempt to judge between better and worse ways of testing in Elixir, and document that in a manner such that the reasons why are self-evident.

## Definitions
To communicate clearly, I need to be precise about the distinctions between these terms as they relate to the different styles of testing.  My intention is to be compatible with common usage, while narrowing the definition to achieve the necessary precision to be clear.

- Module under test: Module containing the logic being tested, which may collaborate with other modules.
- Collaborator: Module containing behavior depended on by the module under test.
- Stub: A replacement for a collaborator, implemented based on the needs of a particular test.
- Fake: A replacement for a collaborator, implemented as a simulation based on the needs of any  test whos module under test collaborates with the module being faked
- Mock: A replacement for a collaborator, implemented to allow outputs to be precisely specified and inputs to be recorded for verification by the test.

## The Sample
This is an extension of the typical “hello, world” example designed to be as simple as possible to implement, as hard as possible to test, and to be complete regarding types of testing needs.  I read a file name from the command line arguments, read the greeting target from the file, emit a greeting to that target to the console, and emit how long the process took.  Here is the sample:

```elixir
defmodule HelloApp do
  def main() do
    time_unit = :microsecond
    microseconds_before = System.monotonic_time time_unit
    target = System.argv |> List.first |> File.read!
    IO.puts "Hello, #{target}!"
    microseconds_after = System.monotonic_time time_unit
    microseconds_duration = microseconds_after - microseconds_before
    IO.puts "Took #{microseconds_duration} microseconds"
  end
end
```

It is my opinion that until you know how to get everything in this sample under 100% test coverage in the language you are using, you have no business writing production code in that language because you lack the proper knowledge to write maintainable code.

## Example Implementations

- https://github.com/SeanShubin/testability_using_function_references
- https://github.com/SeanShubin/testability_using_fakes
- https://github.com/SeanShubin/testability_using_mocks
- https://github.com/SeanShubin/testability_using_mox

Note that I don’t include the function references approach in my analysis because while they are mechanically different from the fakes example, I don’t find them sufficiently conceptually different to address them separately.

## Fake vs. Mock
Mocks are generic with respect to both the collaborator being mocked and the test.  Fakes are generic to the test but specific to the collaborator.  This means there is some up front work to write the fake, but since the fake is under control of the programmer rather than the framework, the test turns out to be much easier to write and read once the fake is written.  Fakes also lend themselves to the more intuitive given/when/then style of programming that flows in a natural order, and is actually testing the results.  With mocks you are more tightly coupled to implementation details, and it looks more like the code is being exercised rather than having its results checked.  With mocks it is also harder to change implementation details while leaving the behavior the same, because the mocking framework itself forces you to couple yourself to implementation details.  With a fake you are necessary coupled to SOME implementation details, but the rest gives you the flexibility to make the test very easy to write and understand.  Consider the flow here:

```elixir
defmodule HelloAppTest do
  use ExUnit.Case, async: true
  test "say hello to world" do
    Fake.with_file_io_system fn file, io, system ->
      # given
      file.set_file_contents("the-file.txt", "world")
      system.set_command_line_arguments(["the-file.txt"])
      system.set_times_in_microseconds([1000, 1234])

      # when
      HelloApp.main(file, io, system)

      # then
      actual = io.get_lines_emitted_to_console()
      expected = ["Hello, world!", "Took 234 microseconds"]
      assert expected == actual
    end
  end
end
```

The first 3 lines tell the test how to behave in a manner specific to the collaborator.
The next line HelloApp.main actually executes the behavior
The next 3 lines verify the behavior after the fact.

Now consider the same thing with a mock style

```elixir
defmodule HelloAppTest do
  use ExUnit.Case, async: true

  import Mox

  setup :verify_on_exit!

  test "say hello to world" do
    IOProxy.Mock
    |> expect(:puts, 1, fn string ->
      assert string == "Hello, world!"
      :ok
    end)
    |> expect(:puts, 1, fn string ->
      assert string == "Took 234 microseconds"
      :ok
    end)

    FileProxy.Mock
    |> expect(:read!, 1, fn path ->
      assert path == "greeting-target.txt"
      "world"
    end)

    SystemProxy.Mock
    |> expect(:argv, 1, fn -> ["greeting-target.txt"] end)
    |> expect(:monotonic_time, 1, fn _unit -> 1000 end)
    |> expect(:monotonic_time, 1, fn _unit -> 1234 end)

    HelloApp.main()
  end
end
```

Notice that it is not even possible to do the given/when/then style with mocks.  This is because unlike the fake, in the mock the behavior for the test is necessarily coupled with the expected results.  We are also embedding the actual function calls to the collaborator, rather than hiding those details behind an abstraction.  There are two forms of noise here, the unnatural order of mixing test behavior with assertions, and extra details from the collaborator that the test doesn’t care about.


## Reusability

As fakes get reused across many tests, both their library of functions they are faking, and the library of functions used for test setup, gradually increase.  These features are added once than used over and over again for each test that interacts with that collaborator.

Since mocks are not an abstraction, but a thin generated wrapper around functions, there is not much opportunity to reuse mocking code from one test in another.

## Intrusiveness to the code

The “mock” library is the clear winner here, albeit at the cost of not being able to run the tests in parallel.  Not a single change needed to the code under test.

Mox requires splitting each collaborator into a specification and an implementation, as well as some configuration to set up the process dictionary.

Fakes require the signatures to be changed to method inject the fake implementations, a consequence of needing a state machine to manage fake behavior.

## Test setup

The mock library is pretty easy.  You have to learn how to create the mock specifications, the mock specifications directly integrate with implementation details, so there is no opportunity for abstraction and reuse.  Also doesn’t allow for the more intuitive given/when/then style of testing.  A particular oddity I ran across was the need to specify expected output to IO.puts in two places.  One to make sure I get a sensible error message on something unexpected, like this:

```elixir
IO,
[],
[
  puts: fn
    ("Hello, world!")->:ok
    ("Took 234 microseconds")->:ok
    (unexpected) -> flunk("unexpected call: IO.puts(#{inspect unexpected})")
  end
]
```

And another to make sure the invocations actually happen, like this:

```elixir
do
  HelloApp.main()
  assert_called IO.puts("Hello, world!")
  assert_called IO.puts("Took 234 microseconds")
end
```

Most of the Mox setup was in the configuration.  You have to switch out to the generated implementations in the test environment and generate the actual mock implementations.

For fakes the work is in creating the fake implementation for each collaborator, and it takes a bit of macro magic to make sure each fake is maintaining its state in a separately named process id.

## Concurrency

The mock library does not support concurrency, presumably because the mock library is globally switching out the implementation of the collaborator

Mox supports concurrency by making use of Elixir’s process dictionary.

The fakes I use support concurrency by having a dynamic name, importing behavior via a macro, and naming their process in charge of state based on the dynamic name.  The fakes are then method injected into the module under test.  Perhaps there is a better way to do this but I couldn’t think of one.

## Static vs Dynamic typing

Mox and mock both have static typing

Due to the use of method injection for overriding with fakes, the solution with fakes loses static typing.

## What is my preference and why?

I am going to make a principled argument based on the following values.
Being easy to maintain is more important than being easy to write, because you only initially write code once, but you repeatedly maintain it indefinitely.
Code should clearly express its intent, that is, when you look at the code, it should be obvious what the code is doing, even without knowing the language.

If you accept these values as the most important, I think it is the inescapable conclusion that fakes are the best solution of the ones I have presented.

While  “clearly express intent” might seem a subjective measure, it is actually more objective than it appears at first, and there are some tricks to reveal this.  You can consider how easy it is for someone who does not know the implementation details or even the language to figure out what the goal of the test is.  The practice of test driven design is also particularly revealing. When you start by designing the behavior you want instead of the implementation of how to get the behavior you want, you necessarily have code that more clearly expresses intent, because you started with intention rather than implementation.

The fundamental problem with mocks is that since they are generated, they focus your attention on implementation details rather than what you need to accomplish.  Top down design is the opposite, you start with what you need, make invocations as if you have what you need, and then that tells you what functions to create.

Since you are in control of the fakes, the code for setting up fakes is more intuitive.  You could design the fakes top down, invoking setup and verification functions that you wish the fake had, and then modify the fakes to actually have those functions.  You can give the functions descriptive names that speak to your intention rather than underlying the implementation details.  Consider again the following test.  In what order would these lines appear?

```elixir
defmodule HelloAppTest do
  use ExUnit.Case, async: true
  test "say hello to world" do
    Fake.with_file_io_system fn file, io, system ->
      # given
1      file.set_file_contents("the-file.txt", "world")
2      system.set_command_line_arguments(["the-file.txt"])
3      system.set_times_in_microseconds([1000, 1234])

      # when
4      HelloApp.main(file, io, system)

      # then
5      actual = io.get_lines_emitted_to_console()
6      expected = ["Hello, world!", "Took 234 microseconds"]
7      assert expected == actual
    end
  end
end

```

I always start with line 7, which makes me think about what exactly I am expecting.
That forces me to write line 6, to define the expectation.
Now I have to think about where I get the thing that needs to match the expectation, which leads me to line 5.  Notice that unlike a mock, here I think about a sensible name for what I need, in this case “get_lines_emitted_to_console”, not the implementation details.  
Something has to actually do the work, leading me to line 4.
Then the test failures force me to set up the fakes to support lines 1-3, but again, unlike the mock, I give sensible names to what I need first, then I implement each thing in the fake.

Notice the function set_times_in_microseconds does not expose the implementation detail of getting the microseconds from System.monotonic_time.  This means I could actually get the time from somewhere else and not change the test, because the test is verifying proper behavior, not a particular implementation.  While the test does expose that the time comes from System somehow, the goal is to minimize coupling to implementation details, and while this is not perfect, it is better than mocks.

I do have to implement the fakes myself, which unlike mocks, aren’t generated.  But the tradeoff here is that the first time I implement something in the fake I have it forever, so I pay the cost once, and I benefit from that clarity in every test that uses this fake in the future.

I certainly don’t like the loss of static typing and the extra work of injecting the fakes into the function, but according to the principles I started with, I do not value these things as much as I value code that clearly expresses its intent.  The need for static typing is mitigated by being test driven with top down design in my approach, as what I miss from type checking I catch with the test.  Method injecting the collaborators does indeed look weird, but that weirdness is reflective of calling out the fact that I have non-determinism.  With this method injection, it is obvious at a glance what functions have nondeterminism, without this method injection, it is easy to mistake methods with nondeterminsm for pure functions.  Maybe non-pure functions should look weird to incentivize being more deliberate about when to use them.


## Is an ideal achievable?

What I really want is the best of all these approaches, so is that achievable?  I don’t know, it depends on the limitations of the Elixir language, but a lot of the necessary parts seem to already exist.  Ideally, I would like the subject of the test to remain unchanged, as the “mock” solution does.  I would like to avoid the configuration and extra classes required by Mox.  I would like the type safety Mox gives you.  I would like to avoid injecting collaborators into functions like I have to do with fakes.  I would like to be able to run my tests asynchronously.  So I think the ideal test would look something like this:

```elixir
Fake.with_replacements %{IO => FakeIO, System => FakeSystem, File => FakeFile} ->
      // given
      FakeFile.set_file_contents("the-file.txt", "world")
      FakeSystem.set_command_line_arguments(["the-file.txt"])
      FakeSystem.set_times_in_microseconds([1000, 1234])

      // when
      HelloApp.main()

      // then
      actual = FakeIO.get_lines_emitted_to_console()
      expected = ["Hello, world!", "Took 234 microseconds"]
      assert expected == actual
end
```

Then each fake would be given its separate state maintained within the scope of the code block, so it can be run concurrently.  I have type safety because the Fake modules have more functions than the real ones.  I still have to write my fakes, and I can even write different fakes for the same underlying modules.

Behind the scenes, the fakes could be given unique process names to maintain their state, emulating what object oriented languages do when they have multiple instances of the same class.  In my examples I simulated this by dynamically naming a class, inlining the implementation as a macro, and naming the process after the dynamically generated class name.  I am not particularly attached to this implementation, but it is the only way I could think of to get safe concurrency.

```elixir
  def with_file(f) do
    id = System.unique_integer([:positive])

    {_, file, _, _} = defmodule String.to_atom("FileFake#{id}") do
      use FileFake
    end

    file.start()
    result = f.(file)
    file.stop()
    result
  end
```

## Feedback

I have tried to take everybody’s feedback into account, and if you have further feedback after reading my analysis, I would love to hear it, especially if you disagree. I appreciate the work all of you have done to help me understand the best way to test nondeterminism in elixir.

If you disagree, I will be curious about the type of disagreement.  Did I get my facts wrong?  Is my reasoning flawed?  Am I valuing the wrong things?

If you find my analysis upsetting, it may actually be the case that you are overly attached to doing things the way you are used to doing them.  Experience can work against you by blinding you to alternatives that are different from what you are used to.  I am not trying to upset anyone.  I am exploring the possibility that better ways of testing in elixir have not been tried due to not realizing they exist, or stagnation in advances in Elixir testing libraries.  Remember that just because I don’t have as much experience as you, or may not be as smart as you, that does not make me wrong.
