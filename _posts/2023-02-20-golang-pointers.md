---
title: Understanding Goland Pointers
date: 2023-02-20
categories: [Programming, Golang]
tags: [programming, golang, fundamentals, pointers, basics]
---

# Understanding Golang Pointers

Even if you are familiar with a language that has pointers like C pointers in go may still give you some trouble. This short guide will assume you have no previous knowledge of pointers to give you a ground up understanding of what they are and how they are used in golang.

## What is a pointer

In programming one of the most common things you do is create variables. When you do this the computer is reserving some portion of memory for the data you want to store. For example lets say I have a struct in go and create an variable of that type:

```golang
type Foo struct {
    ID int
    Bar string
}

var MyData = Foo{ ID: 23, Bar: "Hello" }
```

Then there is a part of memory that now has that data stored in it.

![Memory Screen Shot 1](/assets/img/golang-pointers-image-0.png)

Now lets say I have a function that can accept a `Foo` and update its data.

```golang
func FooFunction(data Foo) {
    data.Bar = "World"
}
FooFunction(MyData)
```

Now before we get to the difference between passing by value or passing by reference it's important to know that whenever you pass something to a function in go the function will receive a copy of whatever was passing in it's arguments. This will become more clear as we continue but for this version of the function we are essentially passing a duplicate of the full data to the function so now our memory looks like this.

![Memory Screen Shot 2](/assets/img/golang-pointers-image-1.png)

So when our function mutates the struct we instead get this version of the memory where the function modifies its copy instead of the original like we wanted.

![Memory Screen Shot 3](/assets/img/golang-pointers-image-2.png)

This is were pointers are helpful. If we change our function to instead take in a pointer to our strut.

```golang
func FooFunction(data *Foo) {
    data.Bar = "World"
}
FooFunction(&MyData)
```

Then when we pass our data to the function we will instead get a memory map that looks more like this.

![Memory Screen Shot 4](/assets/img/golang-pointers-image-3.png)

We have a pointer to the data which means when our function updates the data it will show in our original variable.

![Memory Screen Shot 5](/assets/img/golang-pointers-image-4.png)

That is the basics of pointers for golang and if all you needed to know was how to use them you can stop reading here. However if you want to look a little deeper or if you came from a language like C there is one more detail you may find intersting. Remember that I said anything that you pass to a function in go is a duplicate of its parameters. So the memory actually looks like this.

![Memory Screen Shot 6](/assets/img/golang-pointers-image-5.png)

It may seem like a small difference but it means that a go function cannot reassign pointers. Which means a function like this.

```golang
func ReassignPointer(data *Foo, newData *Foo) {
    data = newData
}
ReassignPointer(&MyData, &SomeOtherData)
```

Would not work as intended because you are not reassigning the original pointer but instead the duplicate of the pointer. A small detail that may save you a few hours of debugging.

I hope you enjoyed this quick walkthrough of go pointers and good luck on your go journey.
