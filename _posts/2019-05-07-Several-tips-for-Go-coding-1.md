---
layout: post
title: "Several tips for Go coding-1"
date: 2019-05-07
excerpt: "It's summarized note that I need to remember for Golang."
tags: [Go, Tips]
comments: false
---


# Several tips for Go coding - 1

It's summarized note that I need to remember for Golang. 

# Playground

 We need to test our code snippet is working or not quickly. Use the playground [https://play.golang.org/](https://play.golang.org/). 

# Structure

Declare the structure like this. 

``` go
    //Due date can be nil because there is a task without a due date.
    type Task struct {
            title string
            done  bool
            due   *time.Time
    }
    
    myTask := Task{"homework", false, nil}
    
    var currentTime = time.Now()
    var yourTask = Task{"homework", false, &currentTime}
    
    var hisTask = Task{title: "cleanup_codes", done: false}
```


# Enum

- **Use 'enum' when you wanna use bool.** Because you'll get so many incidents that you modify your code frequently as service is growing up.
- We can change the task structure like following.
  * Use ```iota``` to represent enum type.

``` go
    type status int // How to define the value?
    
    const (
        UNKNOWN status = iota //<== will be 0
        TODO                  //<== will be 1
        DONE                  //<== will be 2
    )
    
    type Task struct {
        title  string
        status status // <== This
        due    *time.Time
    }
    
    myTask := Task{"homework", TODO, nil}
    
    var currentTime = time.Now()
    var yourTask = Task{"homework", DONE, &currentTime}
    
    var hisTask = Task{title: "cleanup_codes", status: UNKNOWN}
```    

- Effective Go introduces a very smart way to define the constant.

``` go
    type ByteSize float 64
    
    const (
        _ = iota // ignore first value
      KB ByteSize = 1 << (10 * iota) //2^10 = 1024 
      MB   // 2^20 = 1024*1024
      GB   // 2^30 = 1024*1024*1024
      .....
    )
```

# Table-driven tests

To test many cases in batch, you can declare the table as test cases. And run the unit test like following. 

This is the tip that you manage many test cases.

``` go

    package main
    
    import (
        "testing"
    )
    
    
    func fib(n int) int {
        if n == 0 {
            return 0
        } else if n == 1 {
            return 1
        } else {
            return fib(n-1) + fib(n-2)
        }
    }
    
    
    func TestFib(t *testing.T) {
        cases := []struct {
          in, want int     // Test case dat structure. 
        }{
                { 0, 0 },     // Set the test cases in the table.
                { 5, 5 },
                { 6, 8 },
        }
      for _, c := range cases {  // Travese table and run mutiple cases. 
        got := fib(c.in)
        if got != c.want {
        t.Errorf("Fib(%d) == %d, want %d", c.in, got, c.want)
        }
      }
    
    }
```

You can read more articles from [this](https://github.com/golang/go/wiki/TableDrivenTests). It's a very common case to implement many unit tests very effectively.
