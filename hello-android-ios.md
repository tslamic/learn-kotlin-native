---
layout: page
title: Hello, Android & iOS
comments: true
---

In this tutorial, we're going to create a multiplatform project with Kotlin code running on both Android and iOS. Before starting, you'll need to know the very basics of:

- Kotlin. If you're not familiar with Kotlin, you'll have a hard time with Kotlin/Native. Give [Kotlin Koans][1] a go first!
- Gradle. While it's possible to use the command line tools, Gradle is the recommended build tool for Kotlin/Native,
  and thus used throughout the tutorials on this website. Here's a [quick refresher][2] in case you feel rusty.
- Android and iOS app development.

Open Android Studio and create a new _Empty Activity_ project. Make sure to include Kotlin support. Next, create a new directory called `common` in your project root. Open `settings.gradle` and add it to the list of includes, so it looks like this:

```gradle
include ':app', ':common'
```

We've just created a new module named `common`. Sync the project.

Both `app` and `common` module require the Kotlin plugin, so adjust the root `build.gradle`. Move the `buildscript` closure into the `allproject` closure, so the script looks like this:

```gradle
allprojects {
    buildscript {
        ext.kotlin_version = '1.3.0'
        repositories {
            google()
            jcenter()
        }
        dependencies {
            classpath 'com.android.tools.build:gradle:3.2.1'
            classpath "org.jetbrains.kotlin:kotlin-gradle-plugin:$kotlin_version"
        }
    }
    repositories {
        google()
        jcenter()
    }
}

task clean(type: Delete) {
    delete rootProject.buildDir
}
```

Sync and build the project to ensure nothing is broken.

The `common` directory will contain code shared between both platforms. To create a multiplatform module, we first need to tell Gradle about it. Following the [docs][3], create a new file called `build.gradle` in the `common` directory, and add this:

```gradle
apply plugin: 'kotlin-multiplatform'

kotlin {
    targets {
        final def iOSTarget = System.getenv('SDK_NAME')?.startsWith("iphoneos") \
                              ? presets.iosArm64 : presets.iosX64

        fromPreset(iOSTarget, 'iOS') {
            compilations.main.outputKinds('FRAMEWORK')
        }

        fromPreset(presets.jvm, 'android')
    }

    sourceSets {
        commonMain.dependencies {
            api 'org.jetbrains.kotlin:kotlin-stdlib-common'
        }

        androidMain.dependencies {
            api 'org.jetbrains.kotlin:kotlin-stdlib'
        }
    }
}

// workaround for https://youtrack.jetbrains.com/issue/KT-27170
configurations {
    compileClasspath
}
```

Ensure the sync is successful. If you check your build logs, you should see this:

![Multiplatform Sync](/public/assets/multiplatform-sync.png)

which means Gradle recognized our multiplatform project. Just as every other Gradle plugin, the `kotlin-multiplatform` plugin expects source code in certain directories. Create a new file, `common/src/commonMain/kotlin/Common.kt` and add the following:

```kotlin
expect fun platformName(): String

fun sayHello(): String = "Hello, ${platformName()}"
```

The code should look familiar, with the exception of one keyword: `expect`. It denotes a [platform-specific declaration][4], where a common module _expects_ all specified targets in `build.gradle` to implement it. Expected declarations are not restricted to methods, you can apply it to interfaces, classes, annotations, variables, etc.

The `common/build.gradle` has two platforms or targets: iOS and Android. Therefore the Kotlin/Native compiler expects both platforms to implement the `platformName` method, which is highlighted by this lint warning:

![Expect Warning](/public/assets/expect-warning.png)

Let's handle Android first. Create a new file: `common/src/androidMain/kotlin/AndroidCommon.kt` and add the following:

```kotlin
actual fun platformName(): String = "Android"
```

Note the `actual` keyword - this is how you tell the compiler this is the _actual_ implementation of the _expected_ declaration. If you go back to the `Common.kt` file, the lint warning should now only complain about the `common_iOSMain` target, which means we've successfully covered one platform.

Now create a file `common/src/iosMain/kotlin/IosCommon.kt`. Before adding any code, make sure to open the Gradle tool window, and refresh the `common` module:

![Gradle Refresh](/public/assets/gradle-refresh.png)

This ensures the `common` module knows about the external `iOS` libraries provided by Kotlin/Native. Finally, add some code:

```kotlin
import platform.UIKit.UIDevice

actual fun platformName(): String = UIDevice.currentDevice.name
```

Again, note the `actual` keyword. More importantly, check the `import` statement. We're importing [`UIDevice`][5], which wouldn't be very interesting if it weren't for the fact that `UIDevice` is part of [`UIKit`][6], Apple's framework for event-driven user interface for iOS and tvOS apps. As explained in the [Kotlin/Native introduction](kotlin-native), libraries such as `UIKit` are available out of the box, with a single `import` statement.

Checking `Common.kt`, all lint warning should now be gone. Our `common` module is ready to be consumed. Open `app/build.gradle` and add `common` as a project dependency:

```gradle
dependencies {
    implementation project(':common')
    // ...
}
```

Then, add an `id` to the `TextView` in `activity_main.xml`, perhaps something like `android:id="@+id/hello"`. To avoid a quirky Android Studio behavior, build the project. Once successfully built, sync the `app` module. Then, open `MainActivity.kt` and update the `TextView`:

```kotlin
import kotlinx.android.synthetic.main.activity_main.*
import sayHello

class MainActivity : AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)

        hello.text = sayHello()
    }
}
```

Run the app. Because `common/src/androidMain/kotlin/AndroidCommon.kt` specifies `platformName` to return `"Android"`, we should be greeted with a `Hello, Android` message once the app is launched.

![Hello, Android](/public/assets/hello-android.png)

To use the `common` module on iOS, there are a few [extra steps][7] we need to take. First, add the following task to the `common/build.gradle`:

```gradle
task packForXCode(type: Sync) {
    final File frameworkDir = new File(buildDir, "xcode-frameworks")
    final String mode = project.findProperty("XCODE_CONFIGURATION")?.toUpperCase() ?: 'DEBUG'

    inputs.property "mode", mode
    dependsOn kotlin.targets.iOS.compilations.main.linkTaskName("FRAMEWORK", mode)

    from { kotlin.targets.iOS.compilations.main.getBinary("FRAMEWORK", mode).parentFile }
    into frameworkDir

    doLast {
        new File(frameworkDir, 'gradlew').with {
            text = "#!/bin/bash\nexport 'JAVA_HOME=${System.getProperty("java.home")}'\ncd '${rootProject.rootDir}'\n./gradlew \$@\n"
            setExecutable(true)
        }
    }
}

tasks.build.dependsOn packForXCode
```

Sync, then rebuild the `common` project. Inspect the `build` folder once built, and ensure `xcode-framweworks` directory is in it.

![Xcode Frameworks](/public/assets/xcode-frameworks.png)

Next, create a new Xcode project. Choose a Single View app, let the product name be e.g. `ios`, and select the Android project root as the directory for the source code. Then add the `xcode-frameworks/common-framework` as an embedded binary:

![Embedded Binary](/public/assets/embedded-binaries.png)

Because Kotlin/Native produces native binaries and not LLVM bitcode, we need to tell that to Xcode:

![Bitcode](/public/assets/bitcode.png)

Now we need to - again - tell Xcode where to look for frameworks. This should be a relative path from `ios` root directory to the `xcode-frameworks` directory, for example: `$(SRCROOT)/../common/build/xcode-frameworks`.

![Search Paths](/public/assets/search-paths.png)

Finally, go to Build Phases, add a New Run Script Phase, drag it all the way to the top, and add the following code:

```bash
cd "$SRCROOT/../common/build/xcode-frameworks"
./gradlew :common:packForXCode -PXCODE_CONFIGURATION=${CONFIGURATION}
```

![Run Script](/public/assets/run-script.png)

Then, build the project. After it is successfully built, open `ViewController.swift` and replace its content with the following:

```swift
import UIKit
import common

class ViewController: UIViewController {
    override func viewDidLoad() {
        super.viewDidLoad()

        let label = UILabel(frame: CGRect(x: 0, y: 0, width: 300, height: 21))
        label.center = CGPoint(x: 160, y: 285)
        label.textAlignment = .center
        label.font = label.font.withSize(25)

        label.text = CommonKt.sayHello()

        view.addSubview(label)
    }
}
```

Note the `import common` statement. This is the actual `common` module we created at the beginning of this tutorial, using nothing but Kotlin. Also, `CommonKt.sayHello()` should be familiar - it's the method we defined in the `Common.kt` class.

Running the project on an iPhone XR iOS Simulator, the result is - unsurprisingly - a `"Hello, iPhone XR"` greeting:

![Hello, iOS](/public/assets/hello-ios.png)

[1]: https://play.kotlinlang.org/koans/overview
[2]: http://www.vogella.com/tutorials/Gradle/article.html
[3]: https://kotlinlang.org/docs/tutorials/native/mpp-ios-android.html#creating-an-android-project
[4]: https://kotlinlang.org/docs/reference/platform-specific-declarations.html
[5]: https://developer.apple.com/documentation/uikit/uidevice
[6]: https://developer.apple.com/documentation/uikit
[7]: https://kotlinlang.org/docs/tutorials/native/mpp-ios-android.html#setting-up-framework-dependency-in-xcode
