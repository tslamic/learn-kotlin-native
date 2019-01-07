---
layout: page
title: What is Kotlin/Native?
comments: true
---

Kotlin/Native is a technology capable of converting your Kotlin code into standalone binaries that can run on Android, iOS, macOS, Windows, Linux, embedded systems, and more.

It uses [LLVM][1] to achieve this. LLVM stands for Low Level Virtual Machine. The name is unfortunate, as it has little to do with virtual machines. It's actually a collection of modular and reusable compiler and toolchain technologies. Through a series of steps, the Kotlin/Native compiler transforms Kotlin code into LLVM [intermediate representation][3], or IR for short.

IR is a strongly typed [RISC][4] instruction set which abstracts away the underlying platform. Here's a simple _Hello, World_ in LLVM IR:

```llvm
@.str = private unnamed_addr constant [15 x i8] c"Hello, World!\0A\00", align 1

; Function Attrs: noinline nounwind optnone ssp uwtable
define i32 @main() #0 {
  %1 = alloca i32, align 4
  store i32 0, i32* %1, align 4
  %2 = call i32 (i8*, ...) @printf(i8* getelementptr inbounds ([15 x i8], [15 x i8]* @.str, i32 0, i32 0))
  ret i32 0
}

declare i32 @printf(i8*, ...) #1
```

The LLVM compilers, which understand IR, then create binaries for the desired platforms. Perhaps the name makes a bit more sense now - while LLVM needs no virtual machine, the compilation process through IR resembles one.

Kotlin/Native has no problem interacting with C, Objective C, and Swift. In fact, it provides a rich set of libraries with platform-specific APIs such as POSIX, OpenGL, Core Foundation, and more. To get a device name using Apple's [UIKit][5] framework, for example, this is all it takes:

```kotlin
import platform.UIKit.UIDevice

fun deviceName(): String = UIDevice.currentDevice.name
```

There are currently [no plans][2] to support C++.

Kotlin/Native also comes with a standard library which is _almost_ the same as the one on the JVM. The platform-specific constructs and objects, e.g. `File`, are not included, but you can use POSIX, Core Foundation, or other libraries provided out-of-the-box, to overcome this.

Finally, Kotlin/Native features fully automatic memory management, which means you don't have to worry _too much_ about memory. However, when allocating objects via platform-specific means, for example a C library, you need to clean up manually. The standard library offers nice helper functions to keep this as painless as possible.

Some examples of Kotlin/Native in the wild are [Kotlin Coroutines][6], the JetBrains [KotlinConf app][7], and more.

[1]: https://llvm.org/
[2]: https://discuss.kotlinlang.org/t/kotlin-interop-with-c/7121
[3]: https://en.wikipedia.org/wiki/Intermediate_representation
[4]: https://en.wikipedia.org/wiki/Reduced_instruction_set_computer
[5]: https://developer.apple.com/documentation/uikit
[6]: https://kotlinlang.org/docs/reference/coroutines-overview.html
[7]: https://github.com/JetBrains/kotlinconf-app
