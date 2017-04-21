# KeyLang: A Scripting Language in the Key of C

The original documentation for KeyLang, which was written by [@clicheinternetalias](https://github.com/clicheinternetalias), can be found [here](https://rawgit.com/clicheinternetalias/keylang/master/key.html).

## Overview

KeyLang was designed to be small, simple, and easy to integrate and use. No attempts have been made to support any of the latest whiz-bang paradigms or theoretical stuff, just the basics. The primary design goal was to avoid malloc or GC during script execution.

Key features of KeyLang:

 - Simple syntax, similar to C, C++, Java, Javascript, etc., etc. (Well, not similar to C++. Nobody wants to be similar to C++.)
 - No run-time malloc or GC. The only allocations are during compilation or by user-defined functions.
 - No objects. No closures. No co-routines. No clever, mathy, or weird stuff. Just good old-fashioned programming.
 - No arrays, hash-maps, collections, lists, or sequences. No file I/O. You can add your own.
 - No threading support, but scripts can be paused and resumed, which is close enough.
 - Dynamic yet strict typing. Only integers, strings, and undefined are built-in.
 - Strings are atomic, immutable, interned, and mostly encoding-agnostic. (Anything consistent with ASCII, including UTF-8.)
 - Anonymous function expressions, but no closures.
 - No FFI, but easy-enough integration with native code and data.
 - User-defined functions and types. Syntactic support for accessing members and methods of compound types.
 - Easy to get up and running; the API isn't full of constructors and options for stuff that will end up being the same in every program anyway.
 - No actual boolean type, but undefined, zero, the empty string (""), and NULL custom types are false.
 - Great for video games. (Hopefully. I haven't actually used this for anything yet.)

## The Key Language

### Tokens

**Comments**: Block `/*...*/` and line `//...\n`.

**Integers**: Consistent with `strtol`: decimal, hexadecimal, and octal. Most likely 32-bit and signed.

**Strings**: Double-quoted or single-quoted. The only escapes recognized are newlines (\\r, \\n), tab (\\t), literals (\\\\), and embedded newlines (`\<newline>` is dropped from the string). No numeric escapes are recognized; are they even needed?

**Identifiers**: Letters, digits, underscore (\_), and any extended characters (Unicode or whatever). Identifiers cannot start with a digit.

**Keywords**: `break`, `continue`, `do`, `else`, `fn`, `for`, `if`, `is`, `isnot`, `return`, `typeof`, `undef`, `var`, `while`.

**Type names**: `undef`, `int`, `string`, `native` (C function), `function` (script function).

**Operators**: All of them, and then some. Have a precedence table:

| **Operator** | **Associativity** | **Description** | **Types**
| --- | --- | --- | ---
| | | **Primary** |
| `()` | l-to-r | sub-expression | any
| `{}` | l-to-r | object literal | any
| `[]` | l-to-r | array literal | any
| | | **Postfix Member** |
| `[]` | l-to-lr | subscript | any
| `.` | l-to-r | member | any
| | | **Postfix** |
| `()` | l-to-r | function call | native, function
| `++` | l-to-r | post-increment | int
| `--` | l-to-r | post-decrement | int
| | | **Prefix** |
| `~` | r-to-l | bitwise not | int
| `!` | r-to-l | logical not | any
| `+` | r-to-l | no operation | any
| `-` | r-to-l | negation | int
| `?` | r-to-l | is defined | any
| `typeof` | r-to-l | type of value as a string | any
| `++` | r-to-l | pre-increment | int
| `--` | r-to-l | pre-decrement | int
| | | **Infix Multiplicative** |
| `*` | l-to-r | multiplication | int
| `/` | l-to-r | division | int
| `%` | l-to-r | modulus | int
| | | **Infix Additive** |
| `+` | l-to-r | addition | int
| `-` | l-to-r | subtraction | int
| | | **Infix Bitwise** |
| `^` | l-to-r | bitwise exclusive-or | int
| `&` | l-to-r | bitwise and | int
| *pipe* | l-to-r | bitwise inclusive-or | int
| `<<` | l-to-r | shift left | int
| `>>` | l-to-r | signed shift right | int
| `>>>` | l-to-r | unsigned shift right | int
| | | **Infix Comparison** |
| `==` | l-to-r | equal | any
| `!=` | l-to-r | not equal | any
| `<` | l-to-r | less than | int, string
| `<=` | l-to-r | less than or equal | int, string
| `>` | l-to-r | greater than | int, string
| `>=` | l-to-r | greater than or equal | int, string
| `is` | l-to-r | is of a type | any
| `isnot` | l-to-r | is not of a type | any
| | | **Infix Logical (Short-Circuiting)** |
| `&&` | l-to-r | logical and | any
| *two pipes* | l-to-r | logical inclusive-or | any
| `??` | l-to-r | coalesce (evaluate right-hand side if the left is undefined) | any
| | | **Ternary** |
| `? :` | l-to-r | conditional | any
| | | **Assignment** |
| `=` | r-to-l | assignment | any
| `+=` | r-to-l | addition assignment | int
| `-=` | r-to-l | subtraction assignment | int
| `*=` | r-to-l | multiplication assignment | int
| `/=` | r-to-l | division assignment | int
| `%=` | r-to-l | modulus assignment | int
| `^=` | r-to-l | bitwise exclusive-or assignment | int
| `&=` | r-to-l | bitwise and assignment | int
| `&&=` | r-to-l | logical and assignment (replace value if current value is truthy) | any
| *pipe* `=` | r-to-l | bitwise inclusive-or assignment | int
| *two pipes* `=` | r-to-l | logical inclusive-or assignment (replace value if current value is falsey) | any
| `<<=` | r-to-l | shift left assignment | int
| `>>=` | r-to-l | signed shift right assignment | int
| `>>>=` | r-to-l | unsigned shift right assignment | int
| `??=` | r-to-l | coalesce assignment (replace value if current value is undefined) | any
| | | **Comma** |
| `,` | l-to-r | expression separator | any

There's also the rest-argument operator (`...`) which doesn't have precedence and is only valid following the last identifier in a function definition's parameter list.

(Note: You can use `?` with `!` for an “is undefined” operator like so: `!?expr`; I call this the “wtf” operator because “why the — is it undefined!?”)

**Custom operators**: Several syntactic constructs are supported via special functions, either native or script. These functions are listed in their own section.

### Grammar

Examples:

```
var foo = 5;
var bar = "happy times!";

fn doTheThing(a,b,c) {
  var d = a + b;
  return c * d;
}
fn stuff(a) return a + 5;
var baz = 3 + 5;
```

Statement types include:

> **if** (*expr*) *statement*
>
> **if** (*expr*) *statement* **else** *statement*

If the *expr* is not false, the first *statement* will be evaluated; otherwise, the second *statement* will be evaluated.

> **break**;
>
> **continue**;

Only allowed inside a `while` loop. `break` exits the loop; `continue` jumps back to the top, re-evaluating the loop's condition.

> **{ }**
>
> **{** *statements* **}**

Curly braces are used to group a set of statements together as a single statement. Useful for `if` and `while`, and pretty much required for functions. (Single-statement functions don't need the curly braces.)

> **return**;
>
> **return** *expr*;

Return a value from a function. The *expr* is optional; if a value is not given, the function returns `undef`.

> **var** *ident*;
>
> **var** *ident* = *expr*;
>
> **var** *ident*, *ident* = *expr*;

Declare locally-scoped variables inside a function; or, if at top-level, file-scoped variables.

The `var` statement may contain one or more identifiers, with optional values, separated by commas. Variables must be declared before use, and cannot be declared multiple times.

> **for (**;;**)** *statement*
>
> **for (** *expr*, *expr*, *expr* **)** *statement*
>
> **do** *statement* **while** (*expr*);
>
> **while** (*expr*) *statement*

Loops. As long as the condition is true, evaluate *statement*.

> ;
>
> *expr*;

Evaluate the expression, or do nothing.

> **fn** *ident*() *statement*
>
> **fn** *ident*(*arg1, arg2*) *statement*
>
> **fn** *ident*(*arg1, arg2...*) *statement*

Declare a function with name *ident*. Functions can have any number of parameters. Functions can be declared inside of other functions and can also be declared with or without a name inside an expression (similar to Javascript) but do not have access to the enclosing lexical scope( i.e., no closures).

A function may be called with any number of arguments. Missing parameters will be set to `undef`. If the last parameter is specified as a rest-argument (`...`), its value depends on the return value of the special function `__array__`, which will be called, regardless of the number of arguments passed to it. An example of the rest-arguments:

```
fn one(a...) { return a; }
fn rest(a,b,c...) { return c; }
one(); // []
one(1); // [1]
one(1,2); // [1,2]
rest(1,2); // []
rest(1,2,3); // [3]
rest(1,2,3,4); // [3,4]
```

If the special function `__array__` is not defined, then calling a function taking rest-arguments will result in a runtime error (specifically, an error about `__array__` being undefined).

### Variable Scope

Values and functions defined through the `key_global_*` functions are available to all script environments. Values declared at top-level in a file are file-scoped. Values declared inside a function are locally-scoped to that function. There is no block-level scope.

A function defined inside of another function does not have access to variables in the containing functions (i.e., no closures).

Names are not allowed to shadow names in outer scopes. A locally-scoped variable cannot have the same name as a file-scoped variable, and a file-scoped variable cannot have the same name as a globally-scoped variable. Nested functions may shadow names defined in the enclosing function's scopes isn't available to the nested function.

```
// 'global_foo' is defined globally
var foo = 5;
fn bar(foo) { // error: 'foo' redeclared
  var a = "local";
  fn stuff() {
    var a = 5; // okay
    var foo = 4; // error
    var global_foo; // error
  }
}
```

### Default Functions

The default functions are only provided if `key_global_defaults` is called. The supporting native functions are available for defining directly under other names.

> fn **fail(**...**)**;

Prints all arguments to `key_errmsg` then stops the script's execution, returning the error to the script's caller.

> fn **debug(**..**)**;

Prints all arguments to `stderr`, followed by a newline.

> fn **trace()**;

Prints the current stack trace to `stderr`.

> fn **pause()**;

Suspend execution of the script.

### Special Functions

Special functions provide syntactic support for custom compound types, and may be implemented as script functions or as native functions. These are not defined by `key_global_defaults`.

> fn **__member__**(*object*, *member*, *value*);
>
> keyret **__member__**(keyenv \* *e*, keyval \* *rv*, int *argc*, const keyval \* *argv*);

Retrieve or assign a member value in an object. If the function receives three arguments, then the assignment form is being invoked. Either way, the value of the member must be returned. This function handles subscript (`[]`, `[] =`) and member (`.`, `. =`) operators; there is no way to tell the difference between the two syntaxes.

Compound assignments (`+=`, etc.) and the increment operators are handled by the scripting language, with the final simple assignment performed by the `__member__` function.

> fn **__array__**(*value*, ...);
>
> keyret **__array__**(keyenv \* *e*, keyenv \* *rv*, int *argc*, const keyval \* *argv*);

Create an array from an array literal expression (`[]`) or from rest-arguments.

When called to handle rest-arguments, `__array__` only receives the values that should be collected into the rest-argument. If not enough arguments are passed to the rest-argument function, `__array__` will receive no arguments (*`argc`* will be 0). There is no way for the `__array__` function to tell if it's being called to handle rest-arguments or an array literal.

> fn **__object__**(*key*, *value*, *...*);
>
> keyret **__object__**(keyenv \* *e*, keyval \* *rv*, int *argc*, const keyval \* *argv*);

Create an object from an object literal expression (`{ }`). The keys may be of any type, including `undef`. There is no way to tell the difference between a string key and an identifier key. There is always an even number arguments.

### Notes

Conversion to boolean type: Undefined, zero, the empty string (`""`), and custom types of value `NULL` are false. Anything else is true.

Boolean results are represented by the integers 0 (false) or 1 (true).

## The C API

The `key.h` header includes `stddef.h` because I like `size_t`. There's also a certain joy in knowing `NULL` is defined. (Don't worry, dearest macro; my love is as constant as thou.)

### Types

> typedef enum keyret **keyret**;

An enumeration returned by various functions, indicating error and continuation status.

| **Key** | **Description**
| --- | ---
| **KEY_OK** | The operation completed successfully.
| **KEY_PAUSE** | The script is done for now but can be continued later ( only returned from `keyenv_run` and native functions.)
| **KEY_ERROR** | There was an error. A descriptive message with be in `key_errmsg`.

> typedef enum keytype **keytype**;

An enumeration containing the type of the value stored in the `type` member of a `keyval`. New types can be added by calling `keyval_custom_type`.

| **Key** | **Description**
| --- | ---
| **KEY_UNDEF** | The value is undefined.
| **KEY_INT** | The value is an integer, which can be found in the `ival` member of the `keyval`.
| **KEY_STR** | The value is an interned string, which can be found in the `sval` member of the `keyval`.

> typedef struct keyval **keyval**;

A tagged union treated as a value type, containing the various types and their values. Members include:

| **Member** | **Description**
| --- | ---
| **keytype type;** | The type of the value.
| **int ival;** | An integer value, if `type` is `KEY_INT`.
| **const char \* sval;** | A string, most likely interned, if `type` is `KEY_STR`.
| **void \* pval;** | A pointer to a value of custom type, if `type` is a custom type handle.

> typedef struct keyenv **keyenv**;

A script environment, containing the symbols defined in a script file and the execution context information. Only one thread of execution is allowed per file, but interrupt (signal-like) functions may be invoked.

> typedef keyret (**\*key_native_fn**)(keyenv \* *e*, keyval \* *rv*, int *argc*, const keyval \* *argv*);

The signature of native callback functions. The environment of the running script is in `e`. The return value of the function or operator is stored in the `keyval` pointed to by `rv`. The arguments to the function or operator are in `argv` which is `argc` members in length. The first argument is in `argv[0]`.

If a function is assigned to a member of a custom object and is called as a method, the custom object will be passed as the first argument, which is consistent with Uniform Function Call Syntax.

The function may return `KEY_PAUSE` to pause execution of the script or `KEY_ERROR` to stop. If everything finished successfully, `KEY_OK` should be returned.

### Global Variables

> char **key_errmsg[]**;

A short message describing the most recent error. At least it's not *exactly* like `errno`.

### Macros

> \#define **keyval_undef()**

Returns a `keyval` of type `KEY_UNDEF`.

> \#define **keyval_int(** *V* **)**

Returns a `keyval` of type `KEY_INT` with `ival` with value *`V`*.

> \#define **keyval_str(** *V* **)**

Returns a `keyval` of type `KEY_STR` with `sval` value *`V`*. Interning the string before passing it to this macro is encouraged but not required.

> \#define **keyval_custom(** *T*, *V* **)**

Returns a `keyval` of type *`T`* with `pval` value *`V`*.

### Error Handling Functions

> keyret **key_error(** const char \* *fmt*, *...* **)**;

Record an error in `key_errmsg` and returns `KEY_ERROR`.

> keyret **key_type_error(** keyval *a* **)**;

Record an error for an object of the incorrect type. Always returns `KEY_ERROR`.

> keyret **key_type_require(** keyval *a*, keytype *type* **)**;

Record an error if the object's type is not `type`. Returns `KEY_OK` if the types match, `KEY_ERROR` if not.

> keyret **key_type_require_fn(** keyval *a* **)**;

Record an error if the object's type is not a script function or native function. Returns `KEY_OK` if it is a function, `KEY_ERROR` if not.

> size_t **keyenv_stack_trace(** const keyenv \* *e*, char \* *buf*, size_t *buflen* **)**;

Write a stack trace for the environment into the buffer. It's just a list of function names formatted as "`in name:\n`". The message in `key_errmsg` is not appended nor modified. `buf` may be `NULL` if `buflen` is 0. The return value is the number of bytes needed for the resulting string.

> void **key_print_stats(** void **)**;

Print a list of usage statistics to `stdout`. If the compile-time option `TRACK_USAGES` was disabled, this function will only print the size of the `keyenv` struct. If enabled, it will print the amount of string internment room used, the number of global variables defined, and the maximum stack lengths for all environments (the environments must have been destroyed with `keyenv_delete` for their maximum lengths to be recorded).

### Type Functions

> keytype **keyval_custom_type(** const char \* *name* **)**;

Register a new `keytype` value for use as the `type` of a `keyval`. Returns 0 if the type could not be registered. The name must be unique among the type names and will be interned.

> int **keyval_tobool(** keyval *a* **)**

Converts a `keyval` to a boolean. Undefined, integer zero, the empty string, and custom types with value `NULL` are false. Everything else is true.

> int **keyval_equal(** keyval *a*, keyval *b* **)**;

Compares two `keyval` values for equality. No type conversion is performed. Two arguments of different type never compare equal. Custom types are compared for reference equality only.

### String Functions

> keyret **key_intern(** const char \*\* *dst*, const char \* *src* **)**;

Inserts the string `src` into the intern table. A pointer to the interned string's location will be returned in `dst`. Strings passed to Key functions do not have to be interned.

> size_t **key_escape(** char \* *dst*, size_t *dstlen*, const char \* *src* **)**;

Converts the string *src* to have back-slashed escapes consistent with KeyLang string literals. If `dstlen` is 0, `dst` may be `NULL`. The return value is the size needed to store the resulting string (i.e., the index of the terminating nil byte).

### Environment Functions

> keyret **keyenv_new(** keyenv \*\* *ep*, const char \* *src*, const char \* *fname*, int *line* **)**;

Allocate an initialize a new environment with the KeyLang source code `src`. The `fname` and `line` arguments are used for error messages during compilation. `line` should be 1 for the first line of a file.

The contents of the file will not be executed. The top-level of code (mostly variable and function definitions) will be queued for execution. `keyenv_run` should be called until `KEY_PAUSE` is no longer returned before performing any other operation on the environment.

```
keyenv * e;
/* load file */
keyenv_new(&e, "fn main() return 5;", "main.key", 1);
/* define 'main' */
while (keyenv_run(e) == KEY_PAUSE) /**/;
/* call 'main' */
keyenv_push(e, "main", 0, NULL);
while (keyenv_run(e) == KEY_PAUSE) /**/;
```

> void **keyenv_delete(** keyenv \* *e* **)**;

Deallocate the resources for an environment.

> keyret **keyenv_run(** keyenv \* *e* **)**;

Execute a script until it completes (`KEY_OK`), encounters an error (`KEY_ERROR`) or pauses (`KEY_PAUSE`).

This function may be used to resume a paused environment. The return value of the pausing function will be `undef`.

> int **keyenv_running(** const keyenv \* *e* **)**;

Returns true if the environment has not completed executing. This is true for environments that have been paused or have encountered an error.

> void **keyenv_reset(** keyenv \* e **)**;

Resets the execution context so that it's not in the middle of executing anything.

> keyret **keyenv_resume(** keyenv \* *e*, keyval *rv* **)**;

Return a value to a paused environment. This should only be called on an environment that has been paused by returning `KEY_PAUSE` from a native function.

> keyret **keyenv_push(** keyenv \* *e*, const char \* *name*, int *argc*, const keyval \* *argv* **)**;

Start a call to a script function in an environment. The environment must not be running. The function call won't be performed until `keyenv_run` is called. This should only be used on environments that are not currently running.

> keyret **keyenv_interrupt(** keyenv \* *e*, const char \* *name*, int *argc*, const keyval \* *argv* **)**;

Start a call to a script function in an environment. The environment may be running. The return value will not be available. The function call won't be performed until `keyenv_run` is called. Resumed execution will continue past the interrupt's completion. This is useful for inserting events into a script environment.

> keyret **keyenv_callback(** keyenv \* *e*, keyval \* *rv*, keyval *func*, int *argc*, const keyval \* *argv* **)**;

Run a script function or native function to completion. The function cannot pause. The function is called immediately and its return value is placed into `rv`. This is useful for implementing native functions that take functions as arguments, such as sorting functions.

> keyret **keyenv_pop(** keyenv \* *e*, keyval \* *rv* **)**;

Pop a return value from an environment. This should only be called once, on an environment that has finished executing a function called by `keyenv_push`.

> int **keyenv_has(** const keyenv \* *e*, const char \* *name* **)**;

Returns true if `name` is found in the environment. If the environment is running, the currently active local scope will be checked before the file scope.

> keyret **keyenv_get(** const keyenv \* *e*, const char \* *name*, keyval \* *valp* **)**;

Retrives the value of `name`. If the environment is running, the currently active local scope will be checked before the file scope.

> keyret **keyenv_set(** keyenv \* *e*, const char \* *name*, keyval *val* **)**;

Sets the value of `name` if found. If the environment is running, the currently active local scope will be checked before the file scope.

> keyret **keyenv_def(** keyenv \* *e*, const char \* *name*, keyval *val* **)**;

Defines `name` and its value if not already defined. If the environment is running, `name` will be in the currently active local scope, otherwise `name` is file scope.

### Global Environment Functions

> int **key_global_has(** const char \* *name* **)**;

Returns true if `name` is defined as a global value.

> keyret **key_global_get(** const char \* *name*, keyval \* *valp* **)**;

Retrieves the value of `name` if it is defined as a global value, or an error if it is not found.

> keyret **key_global_set(** const char \* *name*, keyval *val* **)**;

Sets the value of `name` to `val` if it is defined as a global value, or returns an error if it is not defined.

> keyret **key_global_def(** const char \* *name*, keyval *val* **)**;

Defines the variable `name` with the value of `val` if not defined as a global value, or returns an error if it is already defined.

> keyret **key_global_fndef(** const char \* *name*, key_native_fn *eval* **)**;

Define the variable `name` to the native function `eval` if not defined as a global value, or returns an error if it is already defined.

> keyret **key_global_defaults(** void **)**;

Called to define the default function set.

### Native Functions

The default native function set. These are all defined by `key_global_defaults` but may be defined using any name with `key_global_fndef`.

> keyret **key_default_fail(** keyenv \* *e*, keyval \* *rv*, int *argc*, const keyval \* *argv* **)**;

Defined as `fail` by `key_global_defaults`.

> keyret **key_default_debug(** keyenv \* *e*, keyval \* *rv*, int *argc*, const keyval \* *argv* **)**;

Defined as `debug` by `key_global_defaults`.

> keyret **key_default_trace(** keyenv \* *e*, keyval \* *rv*, int *argc*, const keyval \* *argv* **)**;

Defined as `trace` by `key_global_defaults`.

> keyret **key_default_pause(** keyenv \* *e*, keyval \* *rv*, int *argc*, const keyval \* *argv* **)**;

Defined as `pause` by `key_global_defaults`.

## Compile-time Options

These macros are defined and can be redefined in the `key.c` file. They are not part of the run-time API.

> \#define **ERROR_BUFLEN** (128)

The length of the error message buffer in `key_errmsg`, in bytes.

> \#define **TOKEN_BUFLEN** (1024)

The maximum length of a script token, in bytes. This includes string literals.

> \#define **INTERN_BUFLEN** (16 * 1024)

Storage space for interned string, in bytes.

> \#define **GLOBAL_MAX** (128)

Maximum number of global variables and functions that can be defined.

> \#define **OPER_DEPTH** (128)

Size of a script environment's operand stack. Bigger numbers allow more complex expressions.

> \#define **SCOPE_DEPTH** (256)

Size of a script environment's file and local scope stack. Bigger numbers allow deeper nesting of function calls with lots of local variables.

> \#define **EXEC_DEPTH** (128)

Size of the execution stack. Bigger numbers allow deeper nesting of script function calls and loops.

> \#define **TRACK_USAGES** (0)

Enable or disable recording usage statistics for many of the compile-time options. Call `key_print_stats` to print the statistics.
