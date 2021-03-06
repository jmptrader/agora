This article explains how to call agora from a Go program and how to build a native module in Go to expose it to agora programs.

The agora programming language is built in Go and is meant to be easily embedded into Go programs. Agora itself is a collection of Go packages, [documented on godoc.org][godoc] from the source code comments.

The `bytecode` package contains types and functions required to convert to and from the bytecode format and to hold the in-memory bytecode representation. It is used by both the `runtime` and the `compiler`.

The `runtime` package contains the definition of the supported types, the virtual machine to execute the instructions, the built-in functions, the execution context, and as a sub-package, the stdlib. It is essentially everything that is required at runtime.

The `compiler` package contains the various parts of the compiler - the scanner and tokens, the parser and the code emitter - as well as the assembler and disassembler.

Finally, the `cmd` subdirectory contains the package for the command-line tool.

## Embedding agora

### The execution context

The first thing that is required to run an agora module within Go is an **execution context**. This is actually an instance of `runtime.Ctx` obtained via a call to `runtime.NewCtx(ModuleResolver, Compiler)`.

A `ModuleResolver` and a `Compiler` must be passed to the execution context. Those are two interfaces defined like this:

```
type ModuleResolver interface {
	Resolve(string) (io.Reader, error)
}
type Compiler interface {
	Compile(string, io.Reader) (*bytecode.File, error)
}
```

Conveniently, the agora runtime provides a ready-to-use module resolver, `runtime.FileResolver`, that maps the module identifier to a file in the file system, relative to the current working directory. It can easily be replaced by any type that implements the `ModuleResolver` interface, for example to load from http or from the database, etc. There is no specific "constructor", it can be created simply using `new(runtime.FileResolver)` or using the literal notation.

A compiler is also provided with the `compiler.Compiler` struct. This is the agora source code compiler. The assembler also implements the `runtime.Compiler` interface, so it is possible to pass a `compiler.Asm` struct to the execution context as compiler and it will not complain. Note, however, that it will only work if the source code found by the module resolver is actually in assembler code format! For most use cases, the `compiler.Compiler` should be used.

A working execution context looks like this:

```Go
import (
    "github.com/PuerkitoBio/agora/compiler"
    "github.com/PuerkitoBio/agora/runtime"
)

func main() {
    ctx := runtime.NewCtx(new(runtime.FileResolver),
        new(compiler.Compiler))
}
```

But there are other fields that may be customized on the context, namely:

* Stdout, Stdin, Stderr : allows setting custom streams, defaults to the standard streams.
* Arithmetic : an implementation of the `Arithmetic` interface, which defines functions for all arithmetic operations, namely `Add`, `Sub`, `Mul`, `Div`, `Mod` and `Unm`. By default, the standard arithmetic implementation is used.
* Comparer : an implementation of the `Comparer` interface, which defines a single `Cmp` function to compare two values, returning 1 if the first value is greater, 0 if both values are equal, and -1 if the first value is lower. By default, the standard comparer implementation is used.
* Debug : a boolean field indicating if the execution context should output debug messages, including those generated by calls to the built-in `debug` in the agora code.

By default, the execution context imports only the built-in functions (the core of the language). Native modules, such as the stdlib, must be registered explicitly via a call to `Ctx.RegisterNativeModule(nativeModule)`. For example:

```Go
func main() {
    ctx := runtime.NewCtx(new(runtime.FileResolver),
        new(compiler.Compiler))
    ctx.RegisterNativeModule(new(stdlib.FmtMod))
}
```

### The module

Once an execution context is ready to use, the next step is to load an agora module in it. That's the responsibility of the `Ctx.Load(id string)` method. It takes a string value representing a module, and the module resolver turns it into actual module data. If the module found is already in bytecode format (the default file resolver checks first for a ".agorac" file - for compiled agora - and uses it if it exists, before looking for a ".agora" source code file), then it is simply loaded into memory, otherwise it is compiled and loaded.

Then the `runtime.Module` is created (actually, this is an interface; a `runtime.agoraModule` is created) and it is cached by the execution context and returned, so that future requests for the same module ID are very cheap.

The module interface looks like this:

```Go
type Module interface {
	ID() string
	Run(...Val) (Val, error)
}
```

Once the module is obtained, executing it is a simple matter of calling `Module.Run(...)`. Everything that is passed into agora must be represented as a `runtime.Val` interface. The value returned from the execution is also a `runtime.Val`, and any runtime error is returned as second value:

```Go
func main() {
    ctx := runtime.NewCtx(new(runtime.FileResolver),
        new(compiler.Compiler))
    ctx.RegisterNativeModule(new(stdlib.FmtMod))
    mod, err := ctx.Load("mymodule")
    if err != nil {
        fmt.Println("Error:", err)
        os.Exit(1)
    }
    ret, err := mod.Run()
    if err != nil {
        fmt.Println("Error:", err)
        os.Exit(2)
    }
    fmt.Println(ret)
}
```

Once a module has been executed, its return value is cached, so that it is only executed once.All `import`s of the same module receive the same return value.

### The value

As mentioned, all values in the runtime are `runtime.Val` implementations. The `Val` interface is defined as follows:

```Go
type Val interface {
	Converter
}
type Converter interface {
	Int() int64
	Float() float64
	String() string
	Bool() bool
	Native() interface{}
}
```

Obviously, some operations are not allowed on certain types (for example adding `nil`), so some operations may raise an error. Apart from the obvious operations, `Native()` returns the underlying native Go value (like the `string` for a string value).

The supported value types are the following, all defined in the `runtime` package:

* Func (more on this later)
* Bool
* Number
* Object
* String
* null

Any other value that implements the `runtime.Val` interface but that isn't any of the predefined, known values, is referred to as a "custom" value, and has little support regarding arithmetic and comparison operations.

All the primitive types are augmented versions of the corresponding Go type:

```Go
type Bool bool
type Number float64
type String string
```

So creating a corresponding `runtime.Val` is simply a matter of converting the Go value to the desired agora value, i.e.:

```Go
agoraBool := runtime.Bool(true)
agoraNumf := runtime.Number(3.1415)
agoraNumi := runtime.Number(42)
agoraString := runtime.String("hi, there!")
```

The `null` value is an empty struct and a single instance, `runtime.Nil`, is created to represent all `nil` values in agora.

The function and the object types are special in that they are *reference* values, as opposed to the other types being passed by value (copied).

The `Func` is actually another interface defined as follows:

```Go
type Func interface {
	Val
	Call(this Val, args ...Val) Val
}
```

So it adds the `Call` method to the common `Val` behaviour. There are two implementations of this interface, `runtime.agoraFunc` and `runtime.NativeFunc`. Only the native function can be created via the native Go API, the agora functions are created internally by the runtime when executing an agora module.

The `Object` is an interface defined as follows:

```
type Object interface {
	Val
	Get(Val) Val  // Get a field value
	Set(Val, Val) // Set a field value, or remove a field if value is nil
	Len() Val 		// Get the length of the object
	Keys() Val 		// Get the keys of the object
	callMethod(Val, ...Val) Val
	callMetaMethod(string, ...Val) (Val, bool)
}
```

It is created by the `runtime.NewObject()` function. Using anonymous struct embedding, it is possible to create custom `Object`s in native modules (see for example the `runtime/stdlib.file` struct in /runtime/stdlib/os.go).

To pretty-print a value for debugging purpose (when running in `Debug` mode, and executing `debug` statements), a `Val` may implement the `Dumper` interface, which defines a single function, `Dump() string`. All predefined agora types implement this interface. If a value does not implement `Dumper`, it is printed using the "%v" `fmt` flag.

## Building a native module

It is possible to provide custom native Go modules to agora code. A good example of how to do this is the stdlib, in the `runtime/stdlib` package.

### The native module

The `runtime.NativeModule` interface augments the basic `runtime.Module` interface:

```Go
type NativeModule interface {
	Module
	SetCtx(*Ctx)
}
```

A native module should define a static, permanent ID, much like Go's import paths (i.e. "github.com/PuerkitoBio/my-native-module"), so the implementation of the `ID()` method is straight-forward. The `SetCtx()` method is called when the native module is registered with an execution context, and the native module should store this context if it needs it.

The `Run()` method is a little more involved. 

First, since the runtime panics when it encounters an error, the `Run()` method must be ready to recover from panics and return the error as second return value. This is such a common pattern that a helper function is made available, `runtime.PanicToError(*error)`. It is usually called in a `defer` statement, with a pointer to the (named) error return value as argument, so that if a panic is recovered, it is set in the error variable that will be returned by the function.

Second, it should cache its return value so that multiple imports of this module get the same value.

Putting it all together, a common implementation pattern for a native module looks like this:

```Go
import "github.com/PuerkitoBio/agora/runtime"

type MyMod struct {
    // The execution context
    ctx *runtime.Ctx
    // The returned value
    ob  runtime.Object
}
func (m *MyMod) ID() string {
    return "github.com/PuerkitoBio/mymod"
}
func (m *MyMod) SetCtx(ctx *runtime.Ctx) {
	m.ctx = ctx
}

// Not interested in any argument in this case. Note the named return values.
func (m *MyMod) Run(_ ...runtime.Val) (v runtime.Val, err error) {
    // Handle the panics, convert to an error
    defer runtime.PanicToError(&err)
    // Check the cache, create the return value if unavailable
    if m.ob == nil {
        // Prepare the object
        m.ob = runtime.NewObject()
        // Export some functions...
        m.ob.Set(runtime.String("MyFunc"), runtime.NewNativeFunc(m.ctx, "mymod.MyFunc", m.myFunc))
    }
    return m.ob, nil
}
```

To expose multiple functions (even though only one is added in this example), an object is created and the functions are set on its keys. The key value is the value to use to call the function. Primitive values can also be exposed, as well as sub-objects, after all this is the usual `runtime.Object` value. We'll look at the native function next, but first here is an example of how to use this module from within agora, assuming the native module has been registered in the execution context:

```
mymod := import("github.com/PuerkitoBio/mymod")
return mymod.MyFunc(4, 9)
```

### The native function

The `runtime.NativeFunc` type implements the standard `runtime.Val` interface, so it is a valid agora value that can be passed around like any other. It is created by calling `runtime.NewNativeFunc()` that takes an execution context, a name and a Go function as argument.

We've already seen the `runtime.Ctx` type, and the name is a string that is used when printing debugging information, but the Go function requires explanation. It must have the `runtime.FuncFn` type, which imposes the following signature:

```Go
type FuncFn func(...Val) Val
```

So this is a function that takes a variable number of `runtime.Val` values, and returns a single `runtime.Val`. Given this information, going back to our "MyMod" native module, the "MyFunc" implementation could look like this:

```Go
// Add two values together, return the sum
func (m *MyMod) myFunc(args ...runtime.Val) runtime.Val {
	runtime.ExpectAtLeastNArgs(2, args)
	return m.ctx.Arithmetic.Add(args[0], args[1])
}
```

The `runtime.ExpectAtLeastNArgs()` is a self-explanatory helper function provided by the `runtime` package that panics if the `args` slice doesn't have enough arguments (it can have more).

And that's pretty much all there is to it! This native Go function can now be exposed to agora code.

Next: [Bytecode format][bytecode]

[godoc]: http://godoc.org/github.com/PuerkitoBio/agora
[bytecode]: https://github.com/PuerkitoBio/agora/wiki/Bytecode-format

