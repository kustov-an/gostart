This document assumes Go version 1.0.3.

## Table of contents ##

* [Motivation](#motivation)
* [The Go way](#canonical)
* [Problems with the current Go way](#problems)
* [FAQ](#faq)
  * [1. How do I start writing Go code?](#faq1)
  * [2. I've written some code. How do I run it?](#faq2)
  * [3. How do I split my package into multiple files?](#faq3)
  * [4. How do I split my package into multiple subpackages?](#faq4)
  * [5. How do I create a package for others to use (i.e. a non-main package)?](#faq5)
  * [6. How do I set up multiple workspaces?](#faq6)
  * [7. Can I create a package outside of $GOPATH?](#faq7)
  * [8. What if I want to hack on some (possibly throw-away) code outside of $GOPATH?](#faq8)
  * [9. How do I download remote packages?](#faq9)
  * [10. How do I distinguish between library packages and main packages?](#faq10)
  * [11. Can I import commands in my code?](#faq11)
  * [12. What if I don't want to use code hosting domains in my import paths?](#faq12)

<a name="motivation"/>
## Motivation ##

The **go tool** is bundled with Go distribution by default and it's convenient for automating common tasks such as getting dependencies, building, and testing your code. It's easy to use and provides a consistent command-line interface and it also expects you to respect a bunch of conventions, some of which I find peculiar and think they introduce a slight learning curve for some users and require a bit of getting used to.

While the conventions imposed by go tool might seem natural for a hardcore gopher, it takes effort for a newcomer to get up to speed with it. If you hit a wall trying to make it work for you and ask for help on #go-nuts channel or [golang-nuts group][2] showing your code layout and error messages `go get` or `go build` produces, you will most likely be told to first learn how to use go tool properly before you start coding in Go. Its documentation is actually pretty good, but it's not easy to absorb it all at once.

My experience was such that neither the recommended [initial reading][1], nor discussions on the mailing list cleared up the picture completely for me. I was only able to eventually grasp the go way by gathering tidbits from the net, through experimentation, and by looking at the go tool's source code.

In this article I'm going to explain the go way from an outsider's point of view. Assuming you're likely to encounter similar hurdles along your way, this guide should answer your questions and help you understand go tool's conventions. There is also a FAQ with code samples at the bottom.

<a name="canonical"/>
## The Go way ##

Here are some fundamentals you need to be aware of when using go tool.

### 1. Go tool is only compatible with code that resides in a workspace

This is the fundamental rule of writing Go code. There are some exceptions that apply to little programs and throw-away code, but in general everything you write in Go will reside in your workspace. So what is a workspace?

A workspace is a directory in your file system, the path to which is stored in the environment variable `GOPATH`. This directory has a certain structure, in the most basic terms it should have one subdirectory named `src`. Within the latter you create new subdirectories, one for each separate Go package.

This is described in more detail in the aforementioned [article][1] and in the FAQ near the end of the present document. To keep it simple, always create a new directory inside `$GOPATH/src` when you're starting a new Go project.

### 2. Go tool does not allow you to depend on specific versions of external packages

If you really need to have a specific version of a certain package, you'll need to fork that package and check it out at the specific version you desire. If you need to use different versions of a single package, you'll need to create a separate fork for each version.

Obviously, this approach doesn't scale and gets tedious once you need more than one version of any given package. Go's reasoning behind this is that versioning is so damn hard that your dependencies should better all be working for you at their latest versions and also at the latest versions of their respective dependencies, and so on recursively. While this is a sound advice, it is generally not possible to adhere to it in practice. You _will_ need to depend on specific versions of your immediate dependencies or dependencies of your dependencies, etc.

In a nutshell, don't try to emulate versioning with go tool, it just doesn't play well with it.

### 3. When working with Go tool, use fully qualified imports and always build import paths from $GOPATH

There is no such thing as local packages in Go. While local imports are possible to some extent, they're meant more for the Go devs themselves than for Go users.

Anything you import is relative to your `$GOPATH/src`. Thus, if you have a directory structure like the following one:

```shell
.
└── src
    └── gopher
        ├── main.go
        └── sub
            └── sub.go
```

in your _main.go_ file you'll need to import sub as follows:

```go
package main

import "gopher/sub"

func main() {
    sub.ExportedFunction()
}
```

If you'd like to use a go package that's publicly available from some code hosting site, the recommended approach is to import it like this:

```go
import "codehosting.com/path/to/package"

func main() {
    package.ExportedFunction()
}
```

Go tool let's you download remote packages either by passing their import path to `go get` or calling `go get ./...` inside your project's directory to recursively get all remote dependencies.

```shell
go get codehosting.com/path/to/package
```

The source for the downloaded package will end up in `$GOPATH/src/codehosting.com/path/to/package`. Go tool will also automatically build a static lib and put it in `$GOPATH/pkg/...`. To read more about this, run `go help get` and `go help gopath`. Also, see [question 9](#faq9).

  [1]: http://golang.org/doc/code.html
  [2]: http://groups.google.com/group/golang-nuts


<a name="problems"/>
## Problems with the current Go way ##

As much as I would like to say that go tool keeps it all simple and keeps you from getting into trouble, the reality proves otherwise. So far, I can come up with the following list of issues that might cause problems for some users:

  * no freedom to write go code anywhere in your file system
  * no support for setting up a reproducible development environment or packaging a locally set up environment
  * no support for managing dependency versions
  * URL-ish looking imports in your code

To elaborate a bit on the second point, go tool does not provide any way to create a reproducible environment for fool-proof deployments. If you tested your code locally, you can never be sure that it'll work during your next deploy, because one of the dependencies might introduce a breaking change during the time period between your testing and deployment. The only apparent solution to this is to package up your downloaded dependencies and copy them over to your production environment. Again, this will have to be done manually. You might find [goven][3] useful in this case.

So, this is the recommended Go way. Of course, you're not obliged to use go tool, you may stick to your favourite Makefiles that invoke Go compiler, linker, etc. directly if that's what works best for you. I only wish I could find a place in the official documentation describing Go's toolchain in detail.

To sum it up, there certainly exists justification for a 3rd party tool that would provide more flexible workflow, automate mundane tasks that are inevitable in practice and solve some of the problems that go tool wasn't designed to deal with.

  [3]: https://github.com/kr/goven

<a name="faq"/>
## FAQ ##

<a name="faq1"/>
### 1. How do I start writing Go code? ###

Before you start any coding, you should pick a directory that will become your Go workspace. All your Go code will reside there. Set the `GOPATH` environment variable to the path to that directory in your .bashrc or similar file for the shell you're using.

```shell
export GOPATH=/Users/alco/go
```

You'll also need to create a subdirectory named `src` inside your `$GOPATH`, this is where you'll be keeping your Go packages and commands. Here's what your initial directory structure is going to look like:

```shell
$ tree -L 2 $GOPATH
/Users/alco/go
└── src
    ├── example
    ├── gopher
    └── testy
```

Each of the subdirectories inside `src` represents a separate package or a command (see [next question](#faq2) describing the difference between packages and commands). Each one will contain at least one _.go_ file and may also have subdirectories of its own.

For more information about `GOPATH` and workspace directory structure, run `go help gopath`.


<a name="faq2"/>
### 2. I've written some code. How do I run it? ###

Navigate to your package's directory and use the go tool to build and run your code.

Let's assume you created a program in `$GOPATH/src/example` directory:

```shell
$ cd $GOPATH/src/example
$ ls
main.go

$ cat main.go
package main

import "fmt"

func main() {
    fmt.Println("Hello world!")
}
```

The quickest way to run it is using `go run`:

```shell
$ go run main.go
Hello world!
```

If your main package is split into multiple files (see [question 3](#faq3) for details on how to do that), you will need to pass them all as arguments to `go run`.

Soon, however, you'll want to produce a binary from your Go source that you can run as a standalone executable, without using go tool. Use `go build` for that, it will create an executable in the current directory.

You can also run `go install`, it will build the code and place the executable in `$GOPATH/bin`. You might want to add the latter to your `PATH` environment variable to be able to run your Go programs anywhere in the file system.

```shell
$ go build
$ ls
example main.go

$ ./example
Hello world!

$ go install
$ $GOPATH/bin/example
Hello world!
```

Here we defined a main package which has a `main` function that is the starting point of our Go program. Any Go program must have exactly one `main` function. By convention, these packages (the ones that declare `package main`) are called commands. In other words, we have built a command named _example_.

You can also define packages with other names, those are called simply packages. They are not intended to produce executable programs, but rather be included as part of some command that will provide a main package with a single `main` function defined in it.

See also: `go help run`, `go help build`, `go help install`.


<a name="faq3"/>
### 3. How do I split my package into multiple files? ###

Go treats files in a single directory as belonging to one package as long as they all have the same name in their `package` declarations.

Let's continue working on our example command. We'll add a second file named _helper.go_ and define a helper function in it.

```shell
$ cd $GOPATH/src/example
$ cat helper.go
package main

func privateHelperFunc() int {
    return 25
}

# now edit main.go to call this function
$ tail -3 main.go
func main() {
    fmt.Println("Hello world! My lucky number is", privateHelperFunc())
}
```

Now we cannot simply `go run main.go` because _main.go_ calls a function defined in another file. We can either pass all files as arguments to go run or build the current package and then run the produced binary.

```shell
$ go run main.go
# command-line-arguments
./main.go:6: undefined: privateHelperFunc

$ go run *.go
Hello world! My lucky number is 25

$ go build
$ ./example
Hello world! My lucky number is 25
```

Private (non-exported) functions and data are accessible in all files that belong to a single package.

A main package allows only one `main` function to be defined, so you'll need to choose one single file from your main package to put it in.

Finally, you can also split any other package into multiple files, not just the main ones. Go's standard packages use this ability pervasively.


<a name="faq4"/>
### 4. How do I split my package into multiple subpackages? ###

Subpackages are just separate packages that happen to reside in another package's directory. Go doesn't treat them in any special way, so import paths for subpackages are relative to your `$GOPATH/src`. Use a subpackage only when its functionality is tied to the main package which contains it and when it doesn't make sense to put that package on one level with other top-level packages.

Let's create a subdirectory in our example project called _math_ and create a file there named _math.go_.

```shell
$ cat math/math.go
package math

func Mul2(x int) int {
    return x * 2
}
```

Let's also edit _main.go_ to call Mul2().

```shell
$ cat main.go
package main

import (
    "fmt"
    "example/math"  // just "math" would not work, it would import std package math
)

func main() {
    fmt.Println("Hello world! My lucky number is", math.Mul2(privateHelperFunc()))
}

$ go run *.go
Hello world! My lucky number is 50
```

Note that if you were to replace `package math` in _math.go_ with `package main`, this would not make _math.go_ part of the same main package that both _main.go_ and _helper.go_ belong to. It would be treated as another main package and you would get an error when trying to run or import it because it doesn't define a main function.


<a name="faq5"/>
### 5. How do I create a package for others to use (i.e. a non-main package)? ###

In the previous questions we were mainly looking at writing so called commands — packages that declare `package main` and are meant to be built to produce an executable binary.

The other flavor of Go packages is used as libraries or modules in other languages, you can't build them into an executable. Their purpose is to be imported into another package (not necessarily main package) to provide useful functionality to that package.

You create a package the same way as you would create a command. The only difference is that instead of `package main` you write `package <some other name>`. All other rules described in previous questions apply to these packages as well: you may split one package into multiple files, but all files belonging to a package reside in a single directory.

Let's say we have created a package called _util_ that resides in `$GOPATH/src/util`.

```shell
$ cd $GOPATH/src/util
$ cat main.go   # file name is arbitrary and doesn't make any significance to Go
package util

import "math"

func Square(x float32) float32 {
    return x * x
}

func Circle(r float32) float32 {
    return math.Pi * r * r
}

func cube(x float32) float32 {
    return x * x * x
}

$ go build
# no output
```

In the context of a simple (non-main) package, `go build` is used to verify that the package compiles without errors. It doesn't produce any binary. To precompile the package for use by other package, you can run `go install`. This will put _util.a_ into `$GOPATH/pkg/<arch>/util.a`. Read more about the _pkg_ directory by running `go help gopath`.

In our util package, there are two exported functions (`Square` and `Circle`) and one private function (`cube`). The private function is only visible in files that are part of the package. Other packages can only call exported functions.

<a name="faq6"/>
### 6. How do I set up multiple workspaces? ###

Short answer — you don't. Go tool does expect you to work in a single workspace. You can add more than one to your `GOPATH` environment variable, but there's a gotcha: `go get` will always download new packages into the first location listed in `GOPATH`.

So while adding more than one path to `GOPATH` is rarely useful, you might still want to use multiple workspaces. You'll just need to change your `GOPATH` according to the workspace you're currently working in. There's definitely room here for some build tool that automates the bookkeeping.

See also [question 12](#faq12) for one example of using multiple workspaces.

<a name="faq7"/>
### 7. Can I create a package outside of $GOPATH? ###

No. You can change your `GOPATH` though, as described in the previous answer. But keep in mind that `$GOPATH` points to a workspace. You packages go into the _src_ directory inside a workspace.

<a name="faq8"/>
### 8. What if I want to hack on some (possibly throw-away) code outside of $GOPATH? ###

If you're not going to import anything outside of standard library or have one level of local imports, then it'll work for you with go tool. If, however, you need to use fully qualified imports, you have to move your code to a workspace or else you'll get problems when trying to `go get` your dependencies and other bad things might also happen.

Workarounds are possible for particular cases and those can be provided by another tool. In general, however, you have to stick to Go's conventions to make it work for you.

<a name="faq9"/>
### 9. How do I download remote packages? ###

To get all dependencies for the current package:

```shell
go get ./...
```

To download a particular remote package:

```shell
go get <package import path>  # see 'go help packages' for details

# for instance,
go get github.com/user/package
```

All downloaded packages end up in `$GOPATH/src`. They are also automatically built and installed into `$GOPATH/pkg`. You can skip installation by passing `-d` flag to `go get`:

```shell
go get -d github.com/user/package
```

<a name="faq10"/>
### 10. How do I distinguish between library packages and main packages? ###

Both kinds of packages live side by side in your workspace's _src_ directory, so there's no distinction at the file system level. There is a convention to call the former ones simply packages and the latter ones commands. So, if your package's first line reads `package main`, it's a command. Otherwise, it's just called a package.

You may also create separate workspaces for packages and commands if you like, but you'll need to remember to adjust `GOPATH` every time you switch between the two. See also [question 6](#faq6) for more details.

<a name="faq11"/>
### 11. Can I import commands in my code? ###

Sure, but you'll need to provide an alias during import so that the package's `main` function does not collide with your main function, if you're importing it into a main package.

```go
import chef "github.com/user/chef"
```

<a name="faq12"/>
### 12. What if I don't want to use code hosting domains in my import paths? ###

You can move stuff around inside `$GOPATH/src` after `go get` has downloaded your dependencies. This would go against the Go way though. Also, currently, the only fast way to distinguish other people's packages from your own is by looking at the directory structure inside your `$GOPATH/src`: remote packages will reside in one of the directories like _github.com_ or _code.google.com_.

Keeping remote packages in another directory, separate from your own packages, would certainly be great. And it can be emulated by specifying more than one path in `GOPATH`. Running `go get` will always download packages into the first directory listed in `GOPATH`, so you can then create your own packages in the directory which is second on the list of paths in `GOPATH`.
