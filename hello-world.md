---
layout: page
title: Hello, Kotlin/Native
comments: true
---

In this short tutorial, we're going to look at an all-time classic, a _Hello, World_ program. Before starting, you'll need to know the very basics of:

- Kotlin. If you're not familiar with Kotlin, you'll have a hard time with Kotlin/Native. Give [Kotlin Koans][1] a go first!
- Gradle. While it's possible to use the command line tools, Gradle is the recommended build tool for Kotlin/Native,
  and thus used throughout the tutorials on this website. Here's a [quick refresher][2] in case you feel rusty.

Create a new directory for this project. Inside the directory, initialize Gradle by executing

```bash
$ gradle init
```

Open `build.gradle` and paste the following:

```gradle
plugins {
    id "org.jetbrains.kotlin.konan" version "1.3.11"
}

konanArtifacts {
    program "hello"
}
```

Konan is the name of the Kotlin/Native plugin for Gradle. It will help us compile the program specified in `konanArtifacts`. The name of the program is `hello`, but you can change it to whatever you like.

By convention, the plugin expects source code to reside in `src/main/kotlin`. Go ahead and create a `src/main/kotlin/Hello.kt` file in your project directory. Then, add the following:

```kotlin
fun main(args: Array<String>) {
    println("Hello, Kotlin/Native!")
}
```

As you can see, there is no difference between a Kotlin/Native Hello World and a Kotlin Hello World. Under the hood, however, a few things do differ.

Strings, such as `"Hello, Kotlin/Native!"`, can be represented [differently][3] on various platforms. The Kotlin/Native [`String`][4] class, also known as `KString`, hides away this complexity. The same is true for `Array` type. According to the Kotlin/Native [interop][5] docs, it is represented as a `CPointer<T>?` class when interacting with a `C` library, which supports the `[]` operator for accessing values by index.

It's now clear that Kotlin/Native hides away some of the multiplatform complexities, and makes it easy for developers to just worry about the code they write.

To build the project, run `gradle build`. Once successfully built, the executable is placed in the `build/konan/bin/{platform}` folder, where `platform` denotes your underlying OS. On a Mac, your build folder might look like this:

![the build folder](/public/assets/hello-build.png)

The executable name is equal to the name of the program specified in `konanArtifacts`. The `.kexe` extension is an abbreviation for _Kotlin executable_. To run the program, you can either execute `gradle run`, or invoke the `.kexe` file directly.

[1]: https://play.kotlinlang.org/koans/overview
[2]: http://www.vogella.com/tutorials/Gradle/article.html
[3]: https://en.wikipedia.org/wiki/String_(computer_science)#Representations
[4]: https://github.com/JetBrains/kotlin-native/blob/master/runtime/src/main/kotlin/kotlin/String.kt
[5]: https://github.com/JetBrains/kotlin-native/blob/master/INTEROP.md
