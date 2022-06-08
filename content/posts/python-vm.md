+++
title = "Python internals for fun (& profit?)"
date = "2021-08-08T12:36:11+03:00"
author = "vlaghe"
authorTwitter = "vlagh3" #do not include @
cover = ""
tags = ["python"]
keywords = ["", ""]
description = "A deep dive in how python compiles & executes code under the hood. On this journey, I will debunk how the python virtual machine works, common misconceptions, tips for an improved efficiency & take a look @ how we can inject code or do other nasty stuff with what we've learned"
showFullContent = false
readingTime = true
hideComments = false
+++

# Introduction

Today I want to speak about python. I recently, realized that I've been using python on a day-to-day basis, but never actually wondered into the magic that happens under the hood, whenever I run a python piece of code. Therefore, as a quest to fulfill my curiosity and maybe use what I learn in my advantage I started researching how python 
does its magic. 

> **DISCLAIMER**: The information in this article is strictly based on my personal research and understanding of the subject which is in no way, shape or form one of a professional. As a result, if you notice something wrong please ping me and upon consensus I will change the flawed section


# So how do machines work?

Very simply put, in order to create a program you write source code in a language. That, is further translated into instructions that the *CPU* can read *(machine code)*. This process is done by a compiler *(e.g gcc)*.
```bash
						gcc prog.c -o prog		
		Source Code		-----compiler----->		Machine Code
```

Think of machine code as just a series of 1's and 0's or on's and off's. What the numbers mean is up to what/who-ever is reading them, and in the processor's case, it takes action based on the number *(instruction)* it sees currently in the memory. Such actions might be writing a value to memory, modifying a value, jumping to another place, reading a value, etc. Different architectures have different recognizable instructions *([opcodes](https://en.wikipedia.org/wiki/Opcode))* based on the processor's manufacturer *(e.g [intel](http://www.c-jump.com/CIS77/CPU/x86/lecture.html#X77_0140_encoding_add_ecx_eax))*.

This construct was used since the early days of programming with languages such as: Pascal, C, C++. However, other modern programming languages *(e.g Java, Python)* use a slightly different strategy in order to make programs 
more portable across platforms. They translate the source code into bytecode, which is recognized and further executed by a "virtual" processor. By virtual I mean that the actual "processor" is just a program written in a lower-level language such as C or in the machine's hardware language itself. So, the same programs can run on any architecture that has the virtual machine program implemented on it which 'translates' that code dynamically. *(e.g python interpretr)*
```bash
																			/--> Windows
Source code	(.py) --python compiler--> Byte code (.pyc) ---PVM----> Machine Code
																			\--> Linux

```
# Almighty bytecode

As seen in the above diagram, python source code is compiled to python bytecode which is then read by the Python Virtual Machine. This bytecode is stored in `.pyc files`, which you can see within the `__pycache__`
directory on python 3.6 or higher. Python generates this directory for efficiency, so that it doesn't need to compile the source code every time you execute it.

## Say whaaaat 
Consider the following program:
```python
def maximum(aList):
	max = None if len(aList) == 0 else aList[0]
	for v in aList[1:]:
		if max < v:
			max = v
	return max
```

Since this is a function object we can have a look at it's attributes with: `dir(maximum)`
The focus for us is the `__code__` attribute, which is a code object and it contains everything python needs to execute the function.
```python
>>> maximum.__code__
<code object maximum at 0x7faad2d7f920, file "<stdin>", line 1>
```

So let's look at the code object's attributes:
```python
>>> dir(maximum.__code__)
['__class__', '__delattr__', '__dir__', '__doc__', '__eq__', '__format__', '__ge__', '__getattribute__', '__gt__', '__hash__', '__init__', '__init_subclass__', '__le__', '__lt__', '__ne__', '__new__', '__reduce__', '__reduce_ex__', '__repr__', '__setattr__', '__sizeof__', '__str__', '__subclasshook__', 'co_argcount', 'co_cellvars', 'co_code', 'co_consts', 'co_filename', 'co_firstlineno', 'co_flags', 'co_freevars', 'co_kwonlyargcount', 'co_lnotab', 'co_name', 'co_names', 'co_nlocals', 'co_posonlyargcount', 'co_stacksize', 'co_varnames', 'replace']
```

For the purposes of this article I will focus mainly on just a few of these:
* `co_varnames`	--	its local var names including parameters
* `co_names`	--	its global var names
* `co_consts`	--	its constants
* `co_code`		--	the actual bytecode

So for our example this would yeld:
```python
('aList', 'max', 'v')
('len',)
(None, 0, 1)
b't\x00|\x00\x83\x01d\x01k\x02r\x10d\x00n\x06|\x00d\x01\x19\x00}\x01|\x00d\x02d\x00\x85\x02\x19\x00D\x00]\x10}\x02|\x01|\x02k\x00r$|\x02}\x01q$|\x01S\x00'
```

In order to make sense out of that bytes object we firstly need to get decimal value of the bytecode, you can do that in python with the `ord()` function:
```python
>>> ord('t')
116
```

Ok, now we know that *t* is decimal 116, but that didn't help much. Luckily, there exist a module called `dis` which makes things easier. For example we could use the `dis.opname` which is a list that contains all the python bytecodes in a human-readeable format.
```python
>>> dis.opname[116]
'LOAD_GLOBAL'
```

You wouldn't want to do that every time so the `dis` module has a convenient way to show the bytecode with `dis.dis`:
```python
>>> dis.dis(add)

SOURCE CODE	 BYTECODE OFFSET		INDEX IN co_varnames
LINE NUMBER	 & INSTRUCTION			LIST & IT'S VALUE

  2           0 LOAD_FAST                0 (x)
              2 LOAD_FAST                1 (y)
              4 BINARY_ADD
              6 RETURN_VALUE
```

Some things to note here:
1. We can observe that each line of source code generates multiple lines of bytecode
2. Each offset is a multiple of 2. Well why? Because, some opcodes can receive an operand such as `LOAD_FAST`, whereas `BINARY_ADD` doesn't. However, since of python 3.6 every instruction will get an argument so that every opcode has 2 bytes. 
*(There are cases in which the argument gets larger than 2 bytes and it will be split in multiple bytes, but it's always going to be a multiple of 2)*

In order to make some sense out of any bytecode, we need to understand how the Python Virtual Machine *(PVM)* works.

## Ok but how does python actually uses bytecode
The PVM is written in C and you can find it [here](https://github.com/python/cpython). The most important thing to understand is that CPython is a stack-oriented virtual machine. A stack's primary operations are load/push and store/pop. We can load/push a value on the top of an upwardly-growing stack *(incrementing the stack pointer)* and we can store/pop a value from the top of the stack *(decrementing the stack pointer)*.

To better illustrate this let's have a look at an example. Consider the following hypothethical instructions:
```c
load 1
load 2
add
load 3
divide
```

Let's see how the stack behaves when each instruction is executed

```bash

				 load 1		 load 2		   add		 load 3		 divide
  |-------|		|-------|	|-------|	|-------|	|-------|	|-------|
2 |		  |		|		|	|		|	|		|	|		|	|		|
  |-------|		|-------|	|-------|	|-------|	|-------|	|-------|
1 |		  |		|		|	|	2	|	|		|	|	3	|	|		|
  |-------|		|-------|	|-------|	|-------|	|-------|	|-------|
0 |		  |		|	1	|	|	1	|	|	3	|	|	3	|	|	1	|
  |-------|		|-------|	|-------|	|-------|	|-------|	|-------|

stackp = -1		stackp=0	stackp=1	stackp=0	stackp=1	stackp=0
```


Now, Python's stack machine is a little bit more complicated than this hypothetical stack we created. In python:
- The objects are always stored in the heap. It's only the pointer to the object that is stored in the stack. For example let's say we want to load 2 integers onto the stack:
```bash
	   stack			  heap
	| ------- |       | -------- |
	| ------- | ----- | -------- |
	|         |       |          |
	| ------- |       | -------- |
	| addr b  | ----> | val of b |
	| ------- |       | -------- |
	| addr a  | ----> | val of a |
	| ------- |       | -------- |
```
- The call stack is the main structure of running a python program. It has one attribute *(a frame)* -- for each currently active function call. Every function call pushes a new frame onto the call stack and every time a call returns, its frame is popped off.

- In each frame there are 2 stacks:
	1. **the evaluation stack** -- where execution of a function's instructions occurs
	2. **the block stack** -- used by python to keep track of certain types of control structures such as: loops, try/excepts blocks, with blocks. This helps python to know which blocks are active at any given moment and take action accordingly. For example, if we use the `break` statement, the interpreter needs to know which loop to break, and that's done with the use of the block stack.


Suppose we have the following program:

```python
def add(x, y):
	return x+y

add(2,3)
```

So our dissasembled bytecode will look like this:
```python
co_consts   :   (<code object add at 0x7fbce771ec90, file "ex.py", line 1>, 'add', 5, 10, None)
co_names    :   ('add',)
co_varnames :   ()
co_code     :   b'd\x00d\x01\x84\x00Z\x00e\x00d\x02d\x03\x83\x02\x01\x00d\x04S\x00'

BYTECODE
  1         X 0 LOAD_CONST               0 (<code object add at 0x7fbce771ec90, file "ex.py", line 1>)
            X 2 LOAD_CONST               1 ('add')
            X 4 MAKE_FUNCTION            0
            X 6 STORE_NAME               0 (add)

  4         X 8 LOAD_NAME                0 (add)
              10 LOAD_CONST              2 (5)
              12 LOAD_CONST              3 (10)
              14 CALL_FUNCTION           2
            X 16 POP_TOP
            X 18 LOAD_CONST              4 (None)
            X 20 RETURN_VALUE

Disassembly of <code object add at 0x7fbce771ec90, file "ex.py", line 1>:
  2           0 LOAD_FAST                0 (x)
              2 LOAD_FAST                1 (y)
              4 BINARY_ADD
              6 RETURN_VALUE
```

For an easier understanding let's ignore the 'X' marked bytecodes and focus more on the others. So, we firstly push to the evaluation stack 2 consts from the `co_consts` list at index 2 and 3 which are our function's arguments *(values: 5, 10)*. 

After, we call the function, specifying the nr of arguments it has *(`CALL_FUNCTION 2`)*. At this point, a frame is going to be created for the `add` function which will have its own evaluation stack and block stack. Inside the add's evaluation stack the bytecode for our function will be executed. So we load the values at index *0* and *1* from the `co_varnames` list, which are our functions' arguments *(x, y)*. Then we add them up, return the value and destroy the frame.

To better understand this process, let's have a look at how these stacks behave and also at how the opcodes are actually defined within the interpreter.


- **BINARY_ADD**: load/push onto the stack the + of the two values on the top written `stack[stackp-1] = stack[stackp-1] + stack[stackp]; stackp -= 1` (turns the two values on the top of the stack into their sum)
- **LOAD_FAST N**: load/push onto the stack the value stored in `co_varnames[N]`, written `stackp += 1, stack[stackp] = co_varnames[N]`
- **LOAD_CONST N**: pushes `co_consts[N]` onto the stack.
- **LOAD_NAME N**: pushes the value associated with `co_names[N]` onto the stack.
- **RETURN_VALUE**: returns with TOS *(top of the stack)* to the caller of the function.
- **CALL_FUNCTION argc**: calls a callable object with positional arguments. `argc` indicates the number of positional arguments. The TOS contains positional arguments, with the right-most argument on top. `CALL_FUNCTION` pops all arguments and the callable object off the stack, calls the callable object with those arguments, and pushes the return value returned by the callable object.

*(check the dis module [docs](https://docs.python.org/3/library/dis.html#python-bytecode-instructions) for a complete definition of all the instructions)*

```bash
BYTECODE								Main Frame						
--------								----------						==== is pop
																		=--= is push
									 evaluation stack
LOAD_NAME 		0 (add)	=----|		|---------------|
LOAD_CONST 		2 (5)	=----|---=>	| 10			| =====>
LOAD_CONST		3 (10)	=----|---=>	|  5			| =====>
CALL_FUNCTION	2 			 |---=>	| <add func> 	| =====>
									| 15	  <-----|--------|
									|---------------|		 |
	|								|  block stack	|		 |
	|								|---------------|		 |
	|								|	.........	|		 |
	|								|---------------|		 |
	V														 | returns 
															 |	 the
LOAD_FAST	0 (x)	=-------|								 | 	   value
LOAD_FAST	1 (y)	=---|	|			add Frame			 |
BINARY_ADD				|	|			---------			 |
RETURN_VALUE			|	|								 |
						|	|		 evaluation stack		 |
						|	|		|---------------|		 |
						|---|----=>	| val of y (5)	| ===>	 |
							|----=>	| val of x (10)	| ===>	 |
									| 15	  ------|--------|
									| 				|
									|---------------|
									|  block stack	|
									|---------------|
									| 	.........	|
									|---------------|

```

If you want to explore more about how the PVM works, how frames are created, what are the C structures behind
the scenes, check out this [talk]() and the actual [source code]()
															 
															 
# Some fun

## dict() vs {}
There's a lot of debate on the internet about which is the more efficient way to define lists, dictionaries, etc.
From my point of view, I think one should understand that python is not a language created to run fast code
so if you need to develop complex software that runs fast do it in a lower language such as C/C++. However,
for the sake of the argument and to also explore more about how python behaves let's have a look at the bytecode:
```python
>>> dis.dis("{}")
  1           0 BUILD_MAP                0
              2 RETURN_VALUE
>>> dis.dis("dict()")
  1           0 LOAD_NAME                0 (dict)
              2 CALL_FUNCTION            0
              4 RETURN_VALUE
```

We can easily see that the first option is more efficient, since it doesn't call any function, thus it doesn't
need to push another frame onto the call stack.

## Should we use a main function or not?
I built 2 simple programs to test this:

naked.py

```python
for i in range(2**26):
	i

main_func.py
```

```python
def main():
	for i in range(2**26):
		i
main()
```

Most of us would assume that the second program would take longer to execute, however
if we check the exection time we see this:
```bash
main_func.py
________________________________________________________
Executed in    1.32 secs   fish           external 
   usr time  1312.34 millis  1300.00 micros  1311.04 millis 
   sys time   10.10 millis  159.00 micros    9.94 millis 


naked.py
________________________________________________________
Executed in    2.39 secs   fish           external 
   usr time    2.38 secs  365.00 micros    2.38 secs 
   sys time    0.01 secs   45.00 micros    0.01 secs 
```

So why is that? Well we should explore the bytecode:

```bash
>>> naked = open("naked.py").read()
>>> dis.dis(naked)
  1           0 LOAD_NAME                0 (range)
              2 LOAD_CONST               0 (67108864)
              4 CALL_FUNCTION            1
              6 GET_ITER
        >>    8 FOR_ITER                 8 (to 18)
             10 STORE_NAME               1 (i)

  2          12 LOAD_NAME                1 (i)
             14 POP_TOP
             16 JUMP_ABSOLUTE            8
        >>   18 LOAD_CONST               1 (None)
             20 RETURN_VALUE
>>> from main_func import main
>>> dis.dis(main)
  3           0 LOAD_GLOBAL              0 (range)
              2 LOAD_CONST               1 (67108864)
              4 CALL_FUNCTION            1
              6 GET_ITER
        >>    8 FOR_ITER                 8 (to 18)
             10 STORE_FAST               0 (i)

  4          12 LOAD_FAST                0 (i)
             14 POP_TOP
             16 JUMP_ABSOLUTE            8
        >>   18 LOAD_CONST               0 (None)
             20 RETURN_VALUE
>>> 
```

They are pretty much the same, but with a few important differences. And those are the following:
- `LOAD_NAME` VS `LOAD_GLOBAL`	*(offset 0 )*
- `STORE_NAME` VS `STORE_FAST`	*(offset 10)*
- `LOAD_NAME`  VS `LOAD_FAST`	*(offset 12)*

We know that `LOAD_NAME` is pushing values from `co_names` list onto the stack, which is pretty much the same as `LOAD_GLOBAL`. So no big difference there. Yet, we notice that the naked program is using the global namespace to load / store variables, while the `main_func` program is using it's local namespace. Since, the global namespace is bigger than the local one, finding stuff is harder. As a result, it takes more time to execute.

So **START USING YOUR MAIN FUNCTION PEOPLE :D**

## Injecting the code object
Now that we have an idea about how code objects work, let's try playing around with them. Suppose we have this function:
```python
def func():
	return 3

func
  co_varnames: ()
  co_names   : ()
  co_consts  : (None, 3) 

  co_code    : b'd\x01S\x00' 

  2           0 LOAD_CONST               1 (3)
              2 RETURN_VALUE
```

So what would happen if we modify the `co_consts` tuple, will this just return our value instead ? 
The answer is yes, computers are dumb they just do what they're told to, so in this case it would load whatever value is at index 1 in `co_consts` and return.
```python
In [9]: injConsts = (None, "wut happened?")
In [10]: code_obj = func.__code__
In [11]: code_obj = code_obj.replace(co_consts=injConsts)
In [12]: func.__code__ = code_obj
In [13]: func()
Out[14]: 'wut happened?'
```

Let's take it a step further and try to change the bytecode.
```python
def add(x, y):
	return x + y

add
  co_varnames: ('x', 'y')
  co_names   : ()
  co_consts  : (None,) 

  co_code    : b'|\x00|\x01\x17\x00S\x00' 



Source Line  m  operation/byte-code      operand (useful name/number)
---------------------------------------------------------------------
  2           0 LOAD_FAST                0 (x)
              2 LOAD_FAST                1 (y)
              4 BINARY_ADD
              6 RETURN_VALUE

```

We will try to change the `BINARY_ADD` instruction to `BINARY_multiply`. To do this, firstly we need to convert
the bytecode to int's so we can identify more easily each instruction:

```python
In [16]: code_obj = add.__code__
In [17]: bytecode = [x for x in code_obj.co_code]
In [18]: bytecode
Out[18]: [124, 0, 124, 1, 23, 0, 83, 0]
```

We know that the offset of our target is 4, so we change it with the opcode for `BINARY_MULTIPLY`:
```python
In [24]: bytecode[4] = dis.opmap['BINARY_MULTIPLY']
In [25]: bytecode
Out[25]: [124, 0, 124, 1, 20, 0, 83, 0]
```

Now we just need to replace the `co_code` attribute with our injected bytecode.
```python
In [26]: injCode = bytes(bytecode)
In [27]: code_obj = code_obj.replace(co_code=injCode)
In [28]: add.__code__ = code_obj
In [29]: add(3, 4)
Out[29]: 12
```

And voilà we modified the normal execution flow of the funcion.

Out there in the real world this aproach is not the best, since we completely destroy the original functionality of the targeted bytecode. Therefore, software might brake which may raise some alarms. So a better way do to this is to execute our code at the beginning and after return to the normal execution of the function. For the purposes of this article I will keep it simple and just execute the `ls` command, however keep in mind that you can basically execute any python code as long as you modify the other attributes of the code object accordingly (e.g co_varnames, co_consts, etc)

Consider our previous function `add`, but we know want to execute a system command before returning. I think the best way to visualize what we need to achieve is to look at the bytecode of our goal:
```python
def injadd(x, y):
	import os
	os.system('ls')
	return x + y

injadd
  co_varnames: ('x', 'y', 'os')
  co_names   : ('os', 'system')
  co_consts  : (None, 0, 'ls') 

  co_code    : b'd\x01d\x00l\x00}\x02|\x02\xa0\x01d\x02\xa1\x01\x01\x00|\x00|\x01\x17\x00S\x00' 

Source Line  m  operation/byte-code      operand (useful name/number)
---------------------------------------------------------------------
  2           0 LOAD_CONST               1 (0)
              2 LOAD_CONST               0 (None)
              4 IMPORT_NAME              0 (os)
              6 STORE_FAST               2 (os)

  3           8 LOAD_FAST                2 (os)
             10 LOAD_METHOD              1 (system)
             12 LOAD_CONST               2 ('ls')
             14 CALL_METHOD              1
             16 POP_TOP

  4          18 LOAD_FAST                0 (x)
             20 LOAD_FAST                1 (y)
             22 BINARY_ADD
             24 RETURN_VALUE
```

We notice that the bytecode changed significantly, which was expected. Firstly, we notice that we need to change
the all 3 variable tuples (co_varnames, co_names, co_consts):

```python
In [43]: injvnames = ('x', 'y', 'os')
In [44]: injnames = ('os', 'system')
In [45]: injconsts = (None, 0, 'ls')
In [47]: add_code = add_code.replace(co_varnames=injvnames)
In [48]: add_code = add_code.replace(co_names=injnames)
In [49]: add_code = add_code.replace(co_consts=injconsts)

add
  co_varnames: ('x', 'y', 'os')
  co_names   : ('os', 'system')
  co_consts  : (None, 0, 'whoami') 
```

That was pretty easy, now let's go and construct our injected bytecode and insert it before the function's bytecode
```python
# inj_code = (
#         "\x64",  LOAD_CONST    (0)
#         "\x01",  1
#         "\x64",  LOAD_CONST    (None)
#         "\x00",  0
#         "\x6c",  IMPORT_NAME   (os)
#         "\x00",  0
#         "\x7d",  STORE_FAST    (os)
#         "\x02",  2
#         "\x7c",  LOAD_FAST     (os)
#         "\x02",  2
#         "\xa0",  LOAD_METHOD   (system)
#         "\x01",  1
#         "\x64",  LOAD_CONST    (ls)
#         "\x02",  2
#         "\xa1",  CALL_METHOD
#         "\x01",  1
#         "\x01",  POP_TOP
#         "\x00",  POP_TOP
# )
In [53]: inj_code = b"\x64\x01\x64\x00\x6c\x00\x7d\x02\x7c\x02\xa0\x01\x64\x02\xa1\x01\x01\x00"
In [55]: print(f"inj_code : {bytes(inj_code)}")
In [56]: print(f"add_code : {bytes(add.__code__.co_code)}")
In [57]: print(f"result   : {bytes(inj_code + add.__code__.co_code)}")

inj_code : b'd\x01d\x00l\x00}\x02|\x02\xa0\x01d\x02\xa1\x01\x01\x00'
add_code : b'|\x00|\x01\x14\x00S\x00'
result   : b'd\x01d\x00l\x00}\x02|\x02\xa0\x01d\x02\xa1\x01\x01\x00|\x00|\x01\x14\x00S\x00'
```

All that's left to do is to change the `co_code` attribute and execute the function.
```python
In [57]: add_code = add_code.replace(co_code=inj_code + add_code.co_code)
In [58]: add.__code__ = add_code
In [59]: add(3,4)

disas.py  inject.py  main_func.py  naked.py  notes  __pycache__
Out[59]: 12
```

Victory !! We achieved code execution.

You can imagine how you could use this as a stealthy way to deploy backdoors, exfiltrate data, etc.
The nice thing about this aproach is that your code gets executed by the current running process, without spawning a suspicios subprocess that may raise some alerts. I haven't tested this yet, but I think that this kind of attack
might be more effective if a deserialization vulnerability is in place.

There are some things to note though:

1. This example uses a simple `add()` function. Hence, the interaction with different attributes are minimal *(e.g co_names was empty in our example)*. Since there weren't many interactions with some of the variable tuples, we could easily add our own and change the bytecode to load from those specific indexe's
<br>
2. More complex functions might be harder to inject because you need to set the position of your injected variables in such a way that is compatible with the bytecode already in place. Consequently, it might not be as easy as inserting your injected code at the start, instead you will also need to modify some of the original bytecode instructions accordingly.
<br>
3. Another thing that I noticed during my explorations is that function parameters play a big role in choosing how you position your injected variables.
<br>
4. You need access to the source code of your target

Currently, I'm trying to develop a way to automatically inject any piece of bytecode in any function. However, I 
ran into some problems due to the points I made above. So if there are some python gurus reading this, please
reach out to me at `vlaghew@protonmail.com` and we might develop something beautiful. Thanks !

# Conclusion

Since I just realized that this article got pretty lengthy I will stop here even if I wanted to aproach some other subjects such as: digging into the CPython Virtual Machine, Building an interpreter, etc. However, I might do that in separate articles. Until then, make sure to check out the awesome resources below which made the creation of this post possible and enlightened me about the unknown world of python vm. There are some great reads made by amazing people who share their knowledge, craft and experience.

As a final note, I encourage you to go and play around with the new tricks you've learned and maybe you'll find something interesting that can be used in your advantage. Learn by doing and don't forget to have fun !

Vlaghe out.




## Resources
- https://ruslanspivak.com/lsbasi-part1/
- http://www.bravegnu.org/blog/python-byte-code-hacks.html
- https://0xec.blogspot.com/2017/03/hacking-cpython-virtual-machine-to.html
- https://tenthousandmeters.com/blog/python-behind-the-scenes-4-how-python-bytecode-is-executed/
- https://tenthousandmeters.com/blog/python-behind-the-scenes-1-how-the-cpython-vm-works/
- https://github.com/python/cpython
- https://www.ics.uci.edu/~brgallar/week9_3.html
- https://files.eric.ed.gov/fulltext/EJ1164711.pdf
- https://people.cs.kuleuven.be/~stijn.volckaert/papers/2018_DIMVA_Interpreters.pdf
- [James Bennett - A Bit about Bytes: Understanding Python Bytecode - PyCon 2018](https://www.youtube.com/watch?v=cSSpnq362Bk)
- [CPython - Bytecode and Virtual Machine - Stephane Wirtel](https://www.youtube.com/watch?v=45BhX5wSeVs)
