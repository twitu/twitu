# Over engineering assignment checking

Suppose you have to check 100 python programs that have implemented a game of guess the number. The program  generates a random number between 1 and 100. This is hidden from the user and they have to guess it. You have to make sure it behaves in a certain way. For e.g. it correctly terminates on a correct guess, it counts the number of guesses it takes you, it displays helpful messages like "too high" or "too low" for each wrong guess.

Here's a rough example of what a submitted file looks like.

```Python
import turtle
import random

hidden = random.randint(1, 100)
count = 0
guessed = False

while not guessed:
    guess = turtle.numinput("Enter your guess: ")
    
    if 0 < guess < 100:
        if guess < hidden:
            turtle.write("Too low")
            count += 1
        else if guess > hidden:
            turtle.write("Too high")
            count += 1
        else:
            turtle.write("You guessed correctly")
            turtle.write(f"You took {count} guesses")
    else:
        turtle.write("Enter a number between 1 and 100.")
        count += 1
```

Checking this has interesting challenges especially when there is no fixed spec or naming convention. Follow along to see some interesting Python wrangling we can do here.

## Running the module

It's a script that the runs the game when the file is run by Python. I can run the game by importing it my `test_runner.py`. Importing a python module executes all it's global statements from top to bottom. In this case the whole game is in the global scope so importing it starts the game.

```Python

def test():
    import game # this will run the game
```

This won't scale to 100 files though. I'll generate the filenames dynamically and import only works when we know the module name beforehand. `importlib` 🕺 to the rescue. This is a library for importing modules.

```Python
import importlib

def test(module_name: str):
    importlib.import_module(module_name)
```

## Mocking libraries

There's a problem, the game takes an input by calling a turtle method which opens up a graphical user interface. Unless it's given an input it'll stay stuck there hanging up the whole test runner. I want to avoid interacting with the UI, so I'll mock this functionality from the turtle library.

```Python
import importlib

def test(module_name: str):
    def mock_numinput(*args, **kwargs):
        return 42
        
    import turtle
    turtle.numinput = mock_numinput
    
    importlib.import_module(module_name)
```

Notice that the game module has it's own import call for `random`. But when we import it here it will reuse our modified version of the turtle. This is because all imports are stored in [`sys.modules`](https://docs.python.org/3/library/multiprocessing.html#contexts-and-start-methods). We've imported turtle and modified one of it's methods. Any new imports call will look up `sys.modules` and use this modified turtle.

What we've done is possible because in Python function are objects too and can be passed around as references. And secondly because there are no private fields so we can modify anything, including imported libraries. [^2] 

We'll do something similar with `random`.

```Python
import importlib

def test(module_name: str):
    def mock_numinput(*args, **kwargs):
        return 42
        
    def mock_randint(*args, **kwargs):
        return 42
        
    import turtle
    turtle.numinput = mock_numinput
    
    import random
    random.randint = mock_randint
    
    importlib.import_module(module_name)
```

## Handling errors and non-termination

Now we have a basic setup. If the game behaves correctly it'll terminate because the hidden number and guess (both doctored by us) are the same. The problem is that there could be faulty implementation which forgets a statement here, a statement there and this does not terminate or maybe throws an error.

There are two approaches to this. One to handle the non-termination, and another for handling errors. They can be combined together which I'll show later.

```Python
import importlib
import multiprocessing as mp

def test(module_name: str):
    # mock imports
    # ....
    
    p = mp.Process(target=importlib.import_module, args=[module_name])
    p.fork() # Note: Do not use start
    p.join(3) # wait for 3 seconds for child process to terminate
    
    if p.is_alive():
        print(f"{module} did not terminate")
```

Here we import the game module in a child process and wait for it to terminate. If that doesn't happen in 3 secs, the implementation has a bug.

An important point to note here is that we must fork the child process and not spawn it. According to the [docs](https://docs.python.org/3/library/multiprocessing.html#contexts-and-start-methods), spawn starts a fresh python interpreter process. This means the child process will no longer use our mocked implementations of the two libraries.

For the child process to use our mocked versions, we'll use fork which creates a child process identical to the parent process when it begins. This does not work with MacOS but there's a [workaround](https://www.reddit.com/r/learnpython/comments/g5372v/multiprocessing_with_fork_on_macos/).

By adding a try catch we can catch the submissions that give errors.

```Python
import importlib

def test(module_name: str):
    # mock imports
    # ....
    
    try:
        importlib.import_module(module_name)
    except Exception as e:
        print(f"{module_name} failed with exception")
        print(e)
```

The non-termination and error approach can be combined by propagating the error from the child process to the parent process. [^3]

## Concluding remarks

By further mocking the libraries and adding states we can do more complex checks like see what text was written on a certain guess. If the number of guesses are being counted correctly and so on.

However a point to note is that all these shenanigans are needed because the assignment isn't designed to be machine checkable. It's method calls are directly coupled with the UI, the variable names are not fixed and the output messages have no defined format. This makes it much harder. Some simple things to make this easier are -

* give predefined variable names and mention that they shouldn't be modified
* give predefined text messages or output codes that can be checked easily
* Get the students to write the game logic as a library rather than a script. This way the real turtle can easily be swapped out for a mocked/checking version of the library

On the other hand by not setting any rules, it allowed the students to be creative and there were a handful who came up with interesting and non-trivial implementations even for this small program.

Another potential approach that I'll try out some other time is to parse the code as an [ast](https://docs.python.org/3/library/ast.html) and query it to verify properties using [ast_selector](https://github.com/guilatrova/ast_selector). This sounds like a more robust approach. And is probably in-line with this larger trend of considering source code as data and interacting/querying/manipulating/training on it's structure. [^4]

[^1]: https://docs.python.org/3/library/sys.html#sys.modules

[^2]: For more robust testing consider using a library like [mock](https://docs.python.org/dev/library/unittest.mock.html)

[^3]: Helpful [SO answer](https://stackoverflow.com/a/33599967)

[^4]: GitHub's [semantic](https://github.com/github/semantic) and [copilot](https://github.com/features/copilot)
