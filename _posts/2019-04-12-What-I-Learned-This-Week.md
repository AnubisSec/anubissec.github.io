```
layout: post
title: Weekly Learning Post
date: 04/12/2019
```
Hello! This will be the first post of this series that I am doing in which I track the little things that I learn throughout the week.

This one will be somewhat short since I hatched this idea midway through the week, but I think it's a good start! So let's jump into some topics!



Anonymous Functions - Definition
-----------------------------

Anonymous functions in programming refers to when a function is defined without an identifier. Not too many resources mention this, but functions can be passed as arguments, and if this is the case, then it makes sense to not have a literal definition and make your code heavier than it needs to be. The idea of anonymous functions are language agnostic and exists within almost all major programming languages out there.

In this post I will dig a bit deeper into how this is implemented inside Golang. Let's look at some examples and dig deeper into this concept.


Anonymous Functions - Golang
----------------------------

So a normal function declaration looks a bit like this within Go:

```go
func testFunc(foo int, bar int) int {
	return foo + bar
}
```

Here we have a function being named called `testFunc` and it has two integer parameters `foo` and `bar` and this function just returns the two parameters added together. This is a fairly simple funciton that would take up some useless space within your code in a large scale development enviornment.

Now let's look at a similar code snippet, but implementing anonymous functions:

```go
var add := func(foo int, bar int) int {
	return foo + bar
}
```

So as you can see this is very similar but notice a few key things:

1.) We have declared a variable before the function, `add`
2.) This variable has a value of a function

Now if we wanted to use this, we would do something like `fmt.Println(add(24,22))`. What this would do is just add the numbers `24` and `22` and print out the result. One of the most valuable reasons to use anonymous functions is that, as soon as the parameters have returned, and the function is called again, the paramaters are written over each time it's called. So this way, we don't need to reassign any variable within an anonymous function any time it's called multiple times. Take a look at this other example pulled from `Go By Example`:

```go
func intSeq() func() int {
	i := 0
	return func() int {
		i ++
		return i
	}
}
```

Here a function, `intSeq` returns another (anonymous) function. And every new call to this function will make the variable `i` reset! Take a look at the next few lines along with their output:

```go
nextInt := intSeq()

fmt.Println(nextInt())		// Returns: 1
fmt.Println(nextInt())		// Returns: 2
fmt.Println(nextInt())		// Returns: 3
```

Make note that each call of `nextInt` keeps the count of `i`. Now let's assign `intSeq()` to another function and call that and see what it returns

```go
newInts := intSeq()

fmt.Println(newInts())		// Returns: 1
```

Look at that! It starts back at 1! This is so amazing to me and blows my mind thinking about the use cases of this in production. In the book that I'm using to learn about Go (since I don't know much at all), there is only a small section about this functionality, but as you can see you can really break down any componenet of any programming langauge to a deep level. 



Conclusion
-----------

This is really all I have for this week, but I'm hoping this will be a good start and hopefully a nice format for what's to come. Next week I will have a minimum of 3 topics to talk about, but this week will only have the one. 

Thanks for reading! See you next week! 
