---
layout: post
title: "How to manage the local submodule without URL like github.com."
date: 2019-03-25
excerpt: "Create project like following."
tags: [Go, Module, go mod, GO1111MOULE ]
comments: false
---

How to manage the local submodule without URL like “github.com~~” 
=========

The best practice I found
------

1. Create project like following.

myapp/main.go
``` go
   package main

   import (
       "fmt"
        hello "myapp/hello"
   )

   func main() {
       fmt.Println("Module Test")
          hello.Hello()
   }
```

2. Create ```myapp/hello/hello.go```

``` go
   package hello

   import "fmt"

   func Hello() {
      fmt.Println("Test")
   }
```

3. Build dependency files. 

``` bash
$ go mod init myapp
$ go run main.go
```

4. No need to commit go.mod. ;-)


Why it works?
------

Generally, Go assumes that all modules will be served via git repository or service like github. Yes, it’s big revolution. Because it’s the main stream that uses the open source module via git hosting service like github.com.

But there are exceptions. If you need to build the home brew application in your private business with the private git hosting service e.g. gitlab or github enterprise, you don't need to use the module name with git hosting service like github.com.

In this case, I used the latest module management rule of Go. How you can locate your source code outside of ```GOPATH```. So any module can be linked outside of ```GOPATH```.

To guide the localtion of modules, we used ```go.mod``` and generated this file by using ```go mod init myapp ```. 


## Should I keep go.mod and go.sum under version control? 

**My answer is YES**. In 1st time, you don't have ```go.mod``` and ```go.sum``` . Then you need to run ```go mod init myapp ``` . ```go.mod`` file has all version info of your dependencies and `go.sum` is a checksum of dependencies. 

In some case, the latest dependency can have a problem like missing modules or bugs.  Then you need ```go.mod``` file to assign the right version of a dependency. 



When we should activate `GO1111MOULE` ?
------

According to  [golang wiki](https://github.com/golang/go/wiki/Modules), ___GO111MODULE is used when you work under GOPATH__.  

> Note that outside of GOPATH, you do not need to set `GO111MODULE` to activate module mode. Alternatively, if you want to work in your GOPATH:

```
$ export GO111MODULE=on                         # manually active module mode
$ cd $GOPATH/src/<project path>                 # e.g., cd $GOPATH/src/you/hello
```

# Reference

- [How to define a module](https://github.com/golang/go/wiki/Modules#how-to-use-modules)

# Thanks to
* [Sangkil](https://www.linkedin.com/in/상길-박-b6ab145a/),he gave the idea of this article. 
