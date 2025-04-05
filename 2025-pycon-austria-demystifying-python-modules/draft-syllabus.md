# Intro

This presentation is technically going to be about Python modules, but in reality, it will be about a side-quest that I went on when writing my bachelor's thesis. During my bachelor's thesis I wrote a Python application that was comparing the accuracy of different algorithms. When nearing the deadline, I noticed I was a couple of pages short. To address this, I also decided the compare the performance of the examined algorithms. At that time I also had a huge preference of unittest of pytest (not to worry, I cured myself from this). I liked how powerful it felt to inherit from TestCase and to use the setUp and tearDown methods. So I wanted to build a clone of unittest for benchmarking. The problem was that I had no idea how to dynamically import modules. In fact, I didn't even understand how the import statement worked and what am I even trying to import (what is a module, amirite?). So I had to find out.

# What sys.path and __init__.py actually do in project structure

Module is an object itself (like everything in python) that serves as an organizational unit of Python code. Modules have a namespace containing arbitrary Python objects. Usually a file.

A package is also a module that contains submodules or subpackages. So like a directory, really. 

Sys.path is a list of strings that specifies the search path for modules

* python -m module command line: prepend the current working directory.

* python script.py command line: prepend the script’s directory. If it’s a symbolic link, resolve symbolic links.

* python -c code and python (REPL) command lines: prepend an empty string, which means the current working directory.

**SHOW Example with inner/utils.**

What about __init__.py?

Python defines two types of packages, regular and namespaces package.

A regular package is typically implemented as a directory containing an __init__.py file. When a regular package is imported, this __init__.py file is implicitly executed, and the objects it defines are bound to names in the package’s namespace.

A namespace package is a bit more complex. With namespace packages, there is no parent/__init__.py file. In fact, there may be multiple parent directories found during import search, where each one is provided by a different portion. Thus parent/one may not be physically located next to parent/two. So, an implicit namespace package is created when a directory on the sys.path is found that does not contain an __init__.py file. This directory is treated as a namespace package, and its contents are added to the namespace of the package being imported.

Why should you care?

There are reasons like explicity, predictability, controlled imports etc.. but really, the one you probably most care about that it helps avoiding issue with discovery and distribution of your package.

**SHOW Example with inner/test_a.py**

# How Python finds and loads modules behind the scenes

So now that I understand a tad more about modules and packages, I need to understand how do I find and load my benchmarking module. How the unittest does it is that they start by looking at the current working directory and walk down from there. if it finds a file, then it immediately calls __import__ (which is what is basically called when using the import statement) which finds the top level module. Then it iterates through the submodules recursively until it finds something that looks like a test.

These use three main components:

    A finder. The job of a finder is to tell Python’s import mechanism if it knows about the type of file being imported or not. If it does, it returns…
    A spec. This specification will give the import mechanism details on how to load the file in question, using something called…
    A loader. Given a path for something to load, the loader will turn that original something into a chunk of compiled code that you can call from your Python script.

After importing the module, the loader will also cache the module in sys.modules. This is a dictionary that maps module names to modules that have already been loaded. This means that if you try to import a module that has already been imported, Python will not re-import it, but will instead return the cached version from sys.modules.

The loaded module is then placed in the module namespace, which is a dictionary that contains all the names defined in the module. This includes functions, classes, and variables. The module namespace is created when the module is loaded, and it is used to store all the names defined in the module.

**SHOW Example with `__import__` and `importlib.import_module`**

And that's how you can use the stuff you imported from other modules.

# Absolute vs. relative imports - when to use which, and why it matters

Absolute and relative imports in Python are very similar to the way we use absolute and relative paths in our file system. An absolute import specifies the full path to the module, starting from the top-level package. A relative import specifies the path to the module relative to the current module.

On the matter of when to use which - always prefer absolute imports. Relative imports have their use cases, like a bunch of loosely coupled script that you move around from a directory to directory, but generally it is cleaner and easier to move the files around (within the package).

There's also a specific case to be aware of in relative imports that seems like it should work, but it does not, that I want to showcase.

**SHOW Example with example-relative-absolute**

This happens because relative imports use a module's __name__ attribute to determine that module's position in the package hierarchy. If the module's name does not contain any package information (e.g. it is set to '__main__') then relative imports are resolved as if the module were a top level module

# The real cause of circular imports and how to fix them without hacks

Circular imports occur when two or more modules depend on each other. This can will throw an ImportError, otherwise the Python interpreter would fall into an infinite loop.

Circular imports are rather simple to fix using a few techniques:
* create a common module that contains the shared code and import it in both modules
* import modules inside functions or methods to delay the import
* import the whole module instead of a specific object

the first one is obvious and probably the correct choices. the other two are more of a hack and even though they are a common practice (especially the local import), they incur their own costs.

instead I would urge you to consider - maybe the modules are too tightly coupled? maybe you should merge them or split them up further?

# Conclusion

For an ending, since I talked about how my interest in this subject was started by the urge to put something extra in my bachelor thesis, I want to share the result of all of that extra research and effort put into writing the benchmarking tool:

**ADD screenshot of the single benchmark screenshot on my thesis**
