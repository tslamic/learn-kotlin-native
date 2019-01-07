---
layout: page
title: Hello, libcurl
---

In this tutorial, we're going to create a simple program using a C library `libcurl`. Before starting, you'll need to know the very basics of:

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
    program "http"
}
```

Next, we need to get our hands on a `libcurl` library. Your OS might already have `curl` installed. If so, you need to find the path to `curl.h` header file. On a Mac, it is usually located in `/usr/include/curl/curl.h`. If you don't have it installed, or cannot find the `curl.h` file, don't worry. Download and extract the [latest version][3], then install the library following [the documentation][5]. The header file is in the `include/curl` directory.

Kotlin/Native comes with a handy tool called `cinterop`. It takes a C library and wraps Kotlin code around it, so you can interact with it seamlessly. To use it, we need a _definition file_ which provides information about the library we want to create bindings for. Each library gets its own definition file, named the same as the library.

Create a new file called `libcurl.def` in your root project directory. Then, add a line denoting the header file:

```bash
headers = /absolute/path/to/your/curl/curl.h
```

There's one more executable that will come handy: `curl-config`. If you didn't have to install the library, it's most likely already part of your path. Otherwise, you'll find it in the library folder you extracted. Run the following:

```bash
$ curl-config --libs
```

The output is the linking information we need in the definition file. If you didn't install the library, it might be as simple as `-lcurl`. If you did install it, it might look like `-L/usr/local/lib -lcurl -lldap -lz`. Copy it, and add it to the definitiona file as a `linkerOpts` value, so your `libcurl.def` looks like this:

```bash
headers = /absolute/path/to/your/curl/curl.h
linkerOpts = -L/usr/local/lib -lcurl -lldap -lz # "curl-config --libs" output
```

While `cinterop` tool can do a [lot more][4], this is all we need to run our program. We now need to tell Gradle about our definition file and include the generated bindings in our program. Here's the updated `build.gradle`:

```gradle
plugins {
    id "org.jetbrains.kotlin.konan" version "1.3.11"
}

konanArtifacts {
    interop "libcurl", {
        defFile "libcurl.def"
    }
    program "http", {
        libraries {
            artifact "libcurl"
        }
    }
}
```

The `interop` closure - when executed - will generate Kotlin bindings based on the provided `defFile`. To use the bindings in your program, simply add them as an `artifact` inside a `libraries` closure.

Before building, add a simple `main` stub to satisfy the compiler. Create a `src/main/kotlin/Http.kt` file with an empty main function:

```kotlin
fun main(args: Array<String>) {
    // Empty for now.
}
```

Now run `gradle build`. Once successfully built, inspect the `build` folder content:

![curl build](/public/assets/curl-build.png)

The `libcurl.kt` contains the library bindings. Time to put them to use! A simple `HTTP GET` request using `libcurl` can be found [here][6], and look like this:

```c
#include <stdio.h>
#include <curl/curl.h>

int main(void)
{
  CURL *curl;
  CURLcode res;

  curl = curl_easy_init();
  if(curl) {
    curl_easy_setopt(curl, CURLOPT_URL, "https://example.com");
    curl_easy_setopt(curl, CURLOPT_FOLLOWLOCATION, 1L);

    res = curl_easy_perform(curl);
    if(res != CURLE_OK)
      fprintf(stderr, "curl_easy_perform() failed: %s\n", curl_easy_strerror(res));

    curl_easy_cleanup(curl);
  }
  return 0;
}
```

Here's how the above looks in Kotlin/Native, using `libcurl` bindings:

```kotlin
import libcurl.*
import kotlinx.cinterop.*

fun main(args: Array<String>) {
  val curl = curl_easy_init()
  if (curl != null) {
    curl_easy_setopt(curl, CURLOPT_URL, "http")
    curl_easy_setopt(curl, CURLOPT_FOLLOWLOCATION, 1L)

    val res = curl_easy_perform(curl)
    if (res != CURLE_OK) {
      val err: CPointer<ByteVar>? = curl_easy_strerror(res)
      val message: String = err?.toKString() ?: "unknown error"
      println("curl_easy_perform() failed: ${message}")
    }

    curl_easy_cleanup(curl)
  }
}
```

As you can see, there is very little difference between the languages. There are a few things to note in the Kotlin code:

- to start using the `libcurl` bindings, all we need to do is import them,
- the `libcurl` function names remain exactly the same (e.g. `curl_easy_init`)
- the types, such as `CURL` or `CURLcode` can be deferred by the compiler.

While unnecessary, we've also introduced two types inside the `if` statement. They are part of `kotlinx.cinterop` library. Pointers and arrays in Kotlin/Native are mapped to `CPointer<T>?`. The [docs][7] for `curl_easy_strerror` state the return type of the function is `const char*`. The [interop docs][8] tell us that's represented as `ByteVar` in Kotlin/Native, therefore the `err` is of type `CPointer<ByteVar>?`.

To transform a `C` string represented as `CPointer<ByteVar>?` to a Kotlin string, we can use the `CPointer<ByteVar>.toKString()` extension method. Refer to the [docs][9] for more information.

Finally, note the `curl_easy_cleanup(curl)` function. As discussed in [a brief introduction](kotlin-native), when interacting with a C library, you need to manually clean up the allocated objects.

[1]: https://play.kotlinlang.org/koans/overview
[2]: http://www.vogella.com/tutorials/Gradle/article.html
[3]: https://github.com/curl/curl/releases
[4]: https://kotlinlang.org/docs/reference/native/c_interop.html
[5]: https://github.com/curl/curl/blob/master/docs/INSTALL.md
[6]: https://curl.haxx.se/libcurl/c/simple.html
[7]: https://curl.haxx.se/libcurl/c/curl_easy_strerror.html
[8]: https://github.com/JetBrains/kotlin-native/blob/master/INTEROP.md
[9]: https://kotlinlang.org/api/latest/jvm/stdlib/kotlinx.cinterop/to-k-string.html
