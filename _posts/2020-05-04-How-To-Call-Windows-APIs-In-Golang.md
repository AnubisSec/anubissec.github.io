---
layout: single
title: How To Call Windows APIs in Golang
date: 2020-05-04
classes: wide
tags:
    - Programming
    - Golang
    - Windows
    - Malware Development
---

Well, it's been quite a while since my last post, but it feels good to be back again. I've taken a break from doing exploit development stuff since getting my OSCE, I don't have much of passion for it anymore. I'm now focusing more on reversing and creating malware. And this blog post is a result of me going down this path.

I've been learning a lot about Golang, trying to learn a new language and trying to future-proof my skill sets for a while. Go has actually been a blast to learn, but I needed to start tying it into my goal of learning how to develop malware. In this pursuit, I wanted to see the full realm of how Go can be utitlized to conduct malicious functions on users machines, and I soon after found out that you can call Windows APIs (similar to C programming) from within Go. 
Though I found this out, there is VERY LITTLE documentation about this, if any at all. I found one blog (which I will link soon after this introduction) that talks about it, but is already a tad dated and not super easy to follow for people like me (non-developers). Wanting to be the change I want in the world, I decided to make this blog. 

One shoutout I have to make is to this [blog](https://medium.com/jettech/breaking-all-the-rules-using-go-to-call-windows-api-2cbfd8c79724) which is the one I mentioned above and had the most robust explanation on this topic that I could find. 

Without much more rambling, let's get into the meat of this blog!


Different Libraries to Call Windows APIs
------------------------------------------

There are quite a few libraries within Golang that allow you to call windows APIs. To just name the ones I know of, there are [syscall](https://golang.org/pkg/syscall/), [windows](https://godoc.org/golang.org/x/sys/windows), [w32](https://godoc.org/github.com/TheTitanrain/w32), and [C](https://golang.org/cmd/cgo/). The last one doesn't truly allow you to call APIs directly, but it allows you to write C code within your Golang source, and also help you convert data types to assist with these other libraries.

I had quite of a bit of issue with each of these, and really had the best luck with windows and syscall, so let's talk about this one first.

Changing Windows Background Image in C++
-----------------------------------------

My idea to test using Windows API's was to try and change the background in my development VM. Let's see what this looks like in C++ first and then we will investigate porting it over to Go.

```cpp
#include <windows.h>
#include <iostream>


int main() {
    const wchar_t *path = L"C:\\image.png";
    int result;
    result = SystemParametersInfoW(SPI_SETDESKWALLPAPER, 0, (void *)path, SPIF_UPDATEINIFILE);
    std::cout << result;        
    return 0;
}
```

I'm going to assume a level of comfortability with reading C code if you're reading this, so I wont go into the structure of this code, but as you can see, you can use the `SystemParameterInfoW()` function to channge the wallpaper in Windows. One of the greatest things Microsft has given to the people are it's MSDN documenation. Here is the struture of `SystemParameterInfoW()` from MSDN documenation:

```cpp
BOOL SystemParametersInfoW(
  UINT  uiAction,
  UINT  uiParam,
  PVOID pvParam,
  UINT  fWinIni
);
```
All we need to really know for this, is the value for `uiAction` to change the background (which is 0x0014 (which represents `SPI_SETDESKWALLPAPER`)), `uiParam` can be 0, `pvParam` will be the path to the image we want to change, and `fWinIni` will be set to `0x001A` which is defined as `SPIF_UPDATEINIFILE` from the documentation. Knowing all this, let's start setting up the Go version of this!


Changing Windows Background Image in Golang
--------------------------------------------

Now that we are equipped with how we can call this function, we can start piecing together our Go file. All we need is one file, so we start out with our skeleton file, with some imports:

```go
package main


import (

	"fmt"
	"unsafe"

	"golang.org/x/sys/windows"
)

func main() {

}
```

Let's go over the imports here. `fmt` for us to print some text to the terminal, `unsafe` allows us to bypass the safety of declaring types within Go programs, and finally `golang.org/x/sys/windows` is what will allow us to call Windows APIs.

Looking over the documentation of the `windows` library, since the `SystemParameterInfoW()` function is not explicitly defined by the creator of this library, we have to manually open a handle to the DLL that holds this function, and then create a variable that points to this function. In the `windows` library, the way to open a handle to a DLL is:

```go
user32DLL				= windows.NewLazyDLL("user32.dll")
```
Which will then add to a variable declaration block with the pointer to the function we want from this DLL:

```go
var (
    user32DLL				= windows.NewLazyDLL("user32.dll")
    procSystemParamInfo	= user32DLL.NewProc("SystemParametersInfoW")
)
```

So now we have to variables, and we will call upon the second one later on and load our parameters into it to have it change our wallpaper. 
Now within our `main` function, we first need to define the image path of where the picture we want to change our wallpaper to is located. For me it's located under `C:\Users\User\Pictures\image.jpg`. You have to wrap this path within the `UTF16PtrFromString()` function to change it's type so that it will be accepted into the C function, since Windows APIs get a little tricky with Golang strings. This will definiely be an issue doing more complicated APIs, which I will have another blog about in the future. 

I then have the binary print out to the user that it is in fact changing the background (but Go binaries execute so quickly probably doesn't matter too much, I just like having print statements!) Finally, we use the `Call()` function within this `windows` library to invoke the API and have it execute and change our wallpaper. Here is the final `main` function in our code:

```go
func main()  {
	imagePath, _ := windows.UTF16PtrFromString(`C:\Users\User\Pictures\image.jpg`)
	fmt.Println("[+] Changing background now...")
	procSystemParamInfo.Call(20, 0, uintptr(unsafe.Pointer(imagePath)), 0x001A)

}
```
So as you can see above, in the last line, we use the `procSystemParamInfo` variable we declared which points to the API we want to use, then pair that with the `Call()` function explained earlier, and then load it up with the parameters we discussed towards the begining. You will see two additional wrappers around the `imagePath` variable on the last line as well. This is a super hacky way to get windows API's to accept golang variable types, first making the whole parameter a `uintptr` which is just a pointer that a C function will accept, and then wrapping the `UTF16PtrFromString` string in `unsafe.Pointer()` function to then allow Go to bypass the safety of type conversion since we are doing several unorthodox conversions.

Let's compile this with `go build` from within the directory of the source code and run the exe that gets built.

![](/assets/images/change.gif)


Thanks for reading this! Feel free to hit me up on twitter if you have any questions about this! Happy hacking!
