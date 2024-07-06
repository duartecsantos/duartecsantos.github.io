---
layout: post
title: Using built-in objects to bypass a sandboxed environment
subtitle: 
tags: []
---

A little while ago I gave a talk at RootedCon about how we could use dunder methods in Python to bypass a sandboxed environment and thus achieve code execution. 
Basically, this talk followed closely what I wrote in my first blogpost, but it also included a brief reference to an alternative method for bypassing a sandboxed environment — this method is based on taking advantage of objects already instantiated in these environments and will be the subject of the blogpost you're reading right now.

In that talk, I mentioned that I had reported a vulnerability (Code Injection) which makes use of this type of bypass, but I did not go into detail because I had not yet received a response from the maintainers. 
Now I have — the package in question is Pandas (v2.2.2), one of the most widely used Python packages.

Okay, story time is over. 
The reported vulnerabilities were present in three functions: `pandas.eval`, `pandas.DataFrame.query`, and `pandas.DataFrame.eval`. 
All these functions receive a string representing a Python expression describing a particular operation on DataFrame columns, and evaluate that string — it's basically the built-in `eval` in steroids. 

You can get arbitrary code execution in all of them by leveraging dunder methods, such as:

```python
import pandas
pandas.eval('().__class__.__bases__[0].__subclasses__()[104].load_module("os").system("whoami")')
```

If you want to better understand how this payload works, you can check out my previous [blogpost](https://duartecsantos.github.io/2024-01-02-CVE-2023-50447/), which covers an interesting bypass that I used to get arbitrary code execution in the Pillow package (CVE-2023-50447).

In addition, in the functions `pandas.DataFrame.query` and `pandas.DataFrame.eval` it is possible to obtain arbitrary code execution in another way, and that's what we're going to focus on. 
The difference between these two functions and `pandas.eval` is that the expressions used in them can contain references to variables in the environment by prefixing them with an "@" character. 
Let's start with a PoC:

```python
import pandas
from os import system
df = pandas.DataFrame({'col1': [1, 2], 'col2': [3, 4]})
df.query('@system("whoami")')
```

```python
import pandas
df = pandas.DataFrame({'col1': [1, 2], 'col2': [3, 4]})
df.query('@pandas.compat.os.system("whoami")')
```

So here we have two example PoCs. 
The first is the easiest case to exploit, since in this environment there is a function that allows you to execute commands in a subshell (i.e. `system`) and as such you can simply call it to execute arbitrary code. 
The second one is a bit more interesting, because we're not relying on anything more than the bare minimum to run `pandas.DataFrame.query`/`pandas.DataFrame.eval` —  we're relying on Pandas being imported. 
What's happening?

Pandas, like every other piece of software on the face of the earth, imports other modules somewhere in its source code. 
For this PoC, we're interested in the fact that the `pandas\compat\__init__.py` file imports the `os` module that we can use to execute code. 
In fact, there are literally THOUSANDS of paths we can use that have `pandas` as their starting point and that take us to some function capable of executing code, so we don't necessarily need to reach the `os` module to get code execution, but it's an easy-to-understand example. 
Two of the shortest paths are `pandas.compat.os.system` and `pandas._testing.os.system`. `pandas.tseries.frequencies.np.lib._polynomial_impl.overrides.os.system` is an example of a slightly longer path. 
In the PoC, I chose `pandas.compat.os.system` because it's short and doesn't have leading underscores, which could be blocked by some filter that rejects dunders or very long expressions.

The process of finding these paths is the fun part of this method. 
While I was investigating Pandas, I created a virtual environment, and I was looking for a place to instrument a check for the objects present within the context of `pandas.DataFrame.query`/`pandas.DataFrame.eval`. 
I decided to put this instrumentation in the `Scope.__init__` function in `<venv_path>\Lib\site-packages\pandas\core\computation\scope.py`. 
According to the docstring, the `Scope` object is an "Object to hold scope, with a few bells to deal with some custom syntax and contexts added by pandas", and as such is perfect for what I wanted to find out. 
In this function, I had access to the `self.scope.maps` variable that contains the global objects present in the eval context. 
There, I wrote a pseudo-fuzzer that used some of Python's instrospection mechanisms (e.g. `dir()`) to help me find a chain of attributes that would lead me to a function capable of executing code (e.g. `system`). 
It's not a complicated process, but it's a fun one. 
And as I told you, I found thousands of possible paths.

And that's basically it! I'll take this opportunity to leave a challenge for the reader: try writing your own attribute fuzzer and let me know how it goes.

That said, in my opinion, by default it shouldn't be possible to use "@" to reference every existing variable in the environment. 
A simple and safer way to implement this functionality would be to have an empty sandbox by default, and have an allowlist in which you would need to explicitly add all the variables you would want to use within the context of these functions. 
This way, anyone using these Pandas functions with user-controlled values wouldn't automatically be vulnerable to Code Injection (assuming that the use of dunders is also blocked). 
When I reported this issue, the mantainers argued that it should be the responsibility of those using the package to validate the data passed to the vulnerable function, and not the responsibility of Pandas itself, so no fix was implemented. 
In the end, we settled on adding a warning to the documentation about the risks associated with using these functions. 
I don't completely agree with this decision, but I respect their choice and understand their reasoning.

Thanks for reading, I hope it was interesting! 🙂
