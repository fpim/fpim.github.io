<!--
.. title: Easy (and effective) python type checker
.. slug: easy-and-effective-python-type-checker
.. date: 2019-09-01 13:25:58 UTC+08:00
.. tags: 
.. category: 
.. link: 
.. description: 
.. type: text
-->


#The Problem

Types have been implicitly handled by python. Flexible as it may seem, developers often find it causing confusions 
when managing a large project, especially for those coming from a strongly typed language.

##Annotation

Newer versions of python (3.5+) allow you to put type hints into at function definition. 
However type checking is not supported by python and it is up to python developer to implement 
their own runtime type checking functionality, according to [PEP 484](https://www.python.org/dev/peps/pep-0484/#non-goals):

>While the proposed typing module will contain some building blocks for runtime 
type checking -- in particular the get_type_hints() function -- third party 
packages would have to be developed to implement specific runtime type checking 
functionality, for example using decorators or metaclasses. Using type hints 
for performance optimizations is left as an exercise for the reader.

Although there are open source libraries like [mypy](http://mypy-lang.org/) 
that do type checking for you, this article aims to present you with the minimum knowledge you need to know (and a hack) 
for you to implement your own type checking, if you want to avoid the unnecessary dependencies brought by a full-blown library.

 
###The Basic Type Hinting
 
 The following function intends to take two integer as arguments and return the sum of them,
 you can specify the type of the function arguments by adding `:type` and type of return value by adding `->type`:
 
```python
def add(a:int, b:int)->int:
    return a + b

add(1,2)
# 3
```
However python did next to nothing with the type of value you passed in:

```python
add('nah ', 'i dont care')
# 'nah i dont care'
```

In fact what python did is that it added the type hinting information into the function's 
`__annotations__` attribute:

```pyhton
add.__annotations__
# {'a': <class 'int'>, 'b': <class 'int'>, 'return': <class 'int'>}
```

Accessing magic function is bad, sometime it doesn't handle edge cases, fortunately the `typing` module comes with a 
handy function `get_type_hints` for you to access object's annotations:

```pyhton
import typing
typing.get_type_hints(add)
# {'a': <class 'int'>, 'b': <class 'int'>, 'return': <class 'int'>}
```
 
###Signature
 
To preform type checking, you will need to analyse the functions' signature in runtime. The `inspect.signature` module 
provides you with a convenient utility ([Signature object](https://docs.python.org/3/library/inspect.html#inspect.Signature)) to do so.

```python
import inspect
sig = inspect.signature(add)
sig
<Signature (a: int, b: int) -> int>
sig.bind_partial(1,2)
# <BoundArguments (a=1, b=2)>
sig.bind_partial(b=2,a=2)
# <BoundArguments (a=2, b=2)>
sig.bind_partial(b=2,a=2).arguments
# OrderedDict([('a', 2), ('b', 2)])
```

The `bind_partial` method of the `signature` object map arguments to their corresponding signature, we can 
use it together with the annotation information to create a simple function decorator that does type checking:

```python
from functools import wraps
def type_check(fn):
	sig = inspect.signature(fn)
	annotation = typing.get_type_hints(fn)
	return_type = annotation.pop('return',None)
	@wraps(fn)
	def wrapped(*args,**kwargs):
		if len(annotation) > 0:
			arguments = sig.bind_partial(*args,**kwargs).arguments
			assert all(isinstance(arguments[k],v) for k,v in annotation.items())
		return_value = fn(*args,**kwargs)
		if return_type:
			assert isinstance(return_value,return_type)
		return return_value
	return wrapped
	
@type_check
def add(a:int, b:int)->int:
    return a + b
    
add(1,2)
# 3

add('1',2)
#Traceback (most recent call last):
#  File "<stdin>", line 1, in <module>
#  File "/Projects/aw/shit.py", line 37, in wrapped
#    assert all(isinstance(arguments[k],v) for k,v in annotation.items())
#AssertionError
```

The above is the most basic type checker you can create, however this type checker involved the use of the `inspect` module 
which the notoriously slow.

###Let's timeit

```python
import inspect
from timeit import timeit
import typing
from functools import wraps

def type_check(fn):
	sig = inspect.signature(fn)
	annotation = typing.get_type_hints(fn)
	return_type = annotation.pop('return',None)
	@wraps(fn)
	def wrapped(*args,**kwargs):
		if annotation:
			arguments = sig.bind_partial(*args,**kwargs).arguments
			assert all(isinstance(arguments[k],v) for k,v in annotation.items())
		return_value = fn(*args,**kwargs)
		if return_type:
			assert isinstance(return_value,return_type)
		return return_value
	return wrapped

def useless_wrapper(fn):
	@wraps(fn)
	def wrapped(*args,**kwargs):
		return fn(*args,**kwargs)
	return wrapped


def add(a:int, b:int)->int:
    return a + b

base_add = useless_wrapper(add)
tc_add = type_check(add)

t= timeit('add(1,1)',
	   setup='from __main__ import add', number=100000)
print('it takes ',t,' seconds to run add 100000 times')


t= timeit('base_add(1,1)',
	   setup='from __main__ import base_add', number=100000)
print('it takes ',t,' seconds to run base_add 100000 times')


t= timeit('tc_add(1,1)',
	   setup='from __main__ import tc_add', number=100000)
print('it takes ',t,' seconds to run tc_add 100000 times')
```

Result:

```python
#it takes  0.013391613000000024  seconds to run add 100000 times
#it takes  0.029804532999999994  seconds to run base_add 100000 times
#it takes  0.708789169  seconds to run tc_add 100000 times
```

Adding a function decorator add a little overhead while the type checker 
put huge overhead onto the original function. It is due to that the `bind_partial` method
dynamically analyses where `*args` and `**kwarg` would be mapped to the signature, in native python code, while the handling of
`*args` and `**kwarg` of an actual function call is optimized in c, now what if we can leverage that?

### The Hack

```python
fn_s = """
def magic_func {0}:
    {1}
                """

def type_check_fast(fn):
	sig = inspect.signature(fn)
	annotation = typing.get_type_hints(fn)
	return_type = annotation.pop('return',None)
	if annotation:
		assert_str = 'assert ' + ' and '.join(["isinstance({k},{v})".format(k=k,v=v.__name__) for k,v in annotation.items()])
		print('compiling:\n', fn_s.format(sig, assert_str))
		exec(fn_s.format(sig,assert_str))
		func = locals()['magic_func']
	@wraps(fn)
	def deced(*args,**kwargs):
		if annotation:
			func(*args,**kwargs)
		return_value = fn(*args,**kwargs)
		if return_type:
			assert isinstance(return_value,return_type)
		return return_value
	return deced
```

Yes, `exec` is used here. The trick is to compile a function with signature that follows the target function's,
and construct assert statement dynamically, so that the string

```python
fn_s = """
def magic_func {0}:
    {1}
                """
```

got formatted to:

```python
#def magic_func (a: int, b: int) -> int:
#    assert isinstance(a,int) and isinstance(b,int)
```

The string is then evaluated by the `exec` statement.

The new function defined in the local scope is then accessable with `locals()['magic_func']`.

Let's put it all together:

```python
import inspect
from timeit import timeit
import typing
from functools import wraps
fn_s = """
def magic_func {0}:
    {1}
                """

def type_check_fast(fn):
	sig = inspect.signature(fn)
	annotation = typing.get_type_hints(fn)
	return_type = annotation.pop('return',None)
	if annotation:
		assert_str = 'assert ' + ' and '.join(["isinstance({k},{v})".format(k=k,v=v.__name__) for k,v in annotation.items()])
		print('compiling:\n', fn_s.format(sig, assert_str))
		exec(fn_s.format(sig,assert_str))
		func = locals()['magic_func']
	@wraps(fn)
	def deced(*args,**kwargs):
		if annotation:
			func(*args,**kwargs)
		return_value = fn(*args,**kwargs)
		if return_type:
			assert isinstance(return_value,return_type)
		return return_value
	return deced

def type_check(fn):
	sig = inspect.signature(fn)
	annotation = typing.get_type_hints(fn)
	return_type = annotation.pop('return',None)
	@wraps(fn)
	def wrapped(*args,**kwargs):
		if annotation :
			arguments = sig.bind_partial(*args,**kwargs).arguments
			assert all(isinstance(arguments[k],v) for k,v in annotation.items())
		return_value = fn(*args,**kwargs)
		if return_type:
			assert isinstance(return_value,return_type)
		return return_value
	return wrapped

def useless_wrapper(fn):
	@wraps(fn)
	def wrapped(*args,**kwargs):
		return fn(*args,**kwargs)
	return wrapped


def add(a:int, b:int)->int:
    return a + b

base_add = useless_wrapper(add)
tc_add = type_check(add)
fast_tc_add = type_check_fast(add)

t= timeit('add(1,1)',
	   setup='from __main__ import add', number=100000)
print('it takes ',t,' seconds to run add 100000 times')


t= timeit('base_add(1,1)',
	   setup='from __main__ import base_add', number=100000)
print('it takes ',t,' seconds to run base_add 100000 times')


t= timeit('tc_add(1,1)',
	   setup='from __main__ import tc_add', number=100000)
print('it takes ',t,' seconds to run tc_add 100000 times')


t= timeit('fast_tc_add(1,1)',
	   setup='from __main__ import fast_tc_add', number=100000)
print('it takes ',t,' seconds to run fast_tc_add 100000 times')
```

Result:

```python
#compiling:
# 
#def magic_func (a: int, b: int) -> int:
#    assert isinstance(a,int) and isinstance(b,int)
#                
#it takes  0.013479943000000001  seconds to run add 100000 times
#it takes  0.030140912  seconds to run base_add 100000 times
#it takes  0.713209548  seconds to run tc_add 100000 times
#it takes  0.07377745000000002  seconds to run fast_tc_add 100000 times
```

The new type checker is 10 time faster than the origional one. Given 
that adding a "useless" decorator (invoking one extra function) adds 0.017 second of overhead, 
we achieved 0.07 second with essentially two extra function invoked with `fast_tc_add`.

### Hack 2 - auto type check

Adding a decorator to every function you have written is very, very ugly. What if we can:

1. At the end of each module, access all variables declared in the local scope.
2. For all variables belongs to the "current" module and is function type:
3. Wrap those functions with the type_check decorator.

Save the following as `tc.py`:

```python
import inspect
import typing
from functools import wraps
fn_s = """
def magic_func {0}:
    {1}
                """

def type_check_fast(fn):
	sig = inspect.signature(fn)
	annotation = typing.get_type_hints(fn)
	return_type = annotation.pop('return',None)
	if annotation:
		assert_str = 'assert ' + ' and '.join(["isinstance({k},{v})".format(k=k,v=v.__name__) for k,v in annotation.items()])
		exec(fn_s.format(sig,assert_str))
		func = locals()['magic_func']
	@wraps(fn)
	def deced(*args,**kwargs):
		if annotation:
			func(*args,**kwargs)
		return_value = fn(*args,**kwargs)
		if return_type:
			assert isinstance(return_value,return_type)
		return return_value
	return deced

def auto_dec(name,dic_locals):
	for k,v in dic_locals.items():
		if hasattr(v,'__module__') and v.__module__ == name and inspect.isfunction(v):
			dic_locals[k] = type_check_fast(v)
```

Then, in another `.py` file, put `auto_dec(__name__,locals())` after all function are decleared:

```python
from tc import auto_dec

def add(a:int,b:int)->int:
    return a+b

def otherfunc(a:int,b:int)->int:
    return a+b

def otherotherfunc(a:int,b:int)->int:
    return a+b

auto_dec(__name__,locals())

if __name__ == '__main__' :
    print(add(1,2))
    print(otherfunc(1,2))
    print(otherotherfunc('nah','got string'))
```

Result:

```python
3
3
Traceback (most recent call last):

  File "/Projects/aw/test.py", line 17, in <module>
    print(otherotherfunc('nah','got string'))
  File "/Projects/aw/tc.py", line 20, in deced
    func(*args,**kwargs)
  File "<string>", line 3, in magic_func
AssertionError
```
Source code of this post can be found [here](https://github.com/fpim/type_checker).