[TOC]

## Compile a Program with Debugging Symbols

使用GDB之前必须完成。通过GDB options来对代码进行编译。

`gcc -ggdb -Wall -o gdbtest gdbtest.c`

`-g`：This option adds debugging info in the operating system's native format.

`-ggdb`：It produces debugging information compatible for use by GDB.

`-Wall`：It enables all warnings in the code.

## Start a Program with GDB

### Run a program without any argument.

`gdb program`

### Run a program with arguments.

two ways you can feed arguments to your program in the GDB debugger. 

1. `gdb --args program arg1 arg2 ... argN  `

2. `$ gdb program  `

   `(gdb) r arg1 arg2 ... argN`

（Run GDB with <--silent> option to hide the extra output it emits on the console.）

## Print Source Code in GDB Console

use the <***l (or list)\***> command which prints <*ten lines*> of source code at a time.

### Display lines after a line number

`list 12`

### Display lines after a function

`list CheckValidEmail`

（**GDB debugger usually shows a few lines back of the point you requested.**）

## Set up Breakpoints

### Break into a line or a Function.

`break (b as shortcut) linenum`

`b function`

### Break into a line which is relative to the current line.

`b +linenum`

### Break into a Function in a given file.

`b filename:function`

### Break on to a line in a given file.

`b filename:linenum`

### Break upon matching memory address.

If you have a program without debug symbols, then you can’t use any of the above options. Instead, the gdb allows you to specify a break point for **memory addresses**.

`b *(memory address)`

### Break after a condition.

`b <...> if condition`

## Print Debug Info

### print backtrace after the breakpoint

`backtrace (or bt as shortcut)`

`info stack`

### execute a function to the end after a breakpoint

`fin`

### print the current stack of the executing program

`where`

### print the line number in GDB while debugging

`frame`

## Trace Variables

### Print standard variable (int, char, etc.)

`p <<variable>>`

### Print structure variable

`p <<structure>>`

### Print pointer variable

`p <<*ptr>>`

### Print a Macro

`p/x DBG_FLAG`

### Print an Array

`p arr`

`p *&arr[96]@5`

### Add Watchers

`watch <<variable>>`

## Continue, Step-in or Next Operations

`c` to continue execution.

`n` for executing the next line.

`s` to step into the function.

## Skip/Ignore Breakpoints

`break test.cpp:18`

`info breakpoints`

`ignore 1 1000`

`run`

`info breakpoints`

## Remove Breakpoints & Quit from GDB

### Deleting a Breakpoint

`d <<breakpoint num>>`

### Quitting from the GDB debugger

`q`



