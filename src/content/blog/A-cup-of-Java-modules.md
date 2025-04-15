---
title: 'A cup of Java: Modules'
description: 'A quick exploration of Java modules'
pubDate: 'Apr 15 2025'
---

In this post, I do a quick exploration of Java modules.

Although they have been introduced in 2017 with the delayed Java 9 release,
the use of modules took a while to take off. 

So what are modules and what do you get from using them?

Until modules were a thing, Java let you organize and structure your code into methods, classes and packages.

Classes and packages have public and private API. But the distinction between private and public API is not 
air-tight and using reflection at runtime it is easy to circumvent it.

Modules are the next level of code organization.

A module has a unique name, some dependencies (other modules it depends on), a public API that it exports 
and a list of services that it uses or provides to others.
It does not ****** ook that much but the real novelty is that with modules the distinction between public and private
API is real: only public types that are exported are usable outside a module and you can only access types using
reflection if explicitly allowed by the module.

By using modules, you increase readability and maintainability and drastically limit accidental use of 
private APIs.

But to really understand the benefits, it always good to look at some example.
In this post, I play with a toy application made of 3 *parts*:
- a service api,
- an implementation of that service
- and a client application of the service.

The code for this post can be found [here](https://github.com/Applequist/JavaModules)

## The old and dirty way

In this first part we look at the code structured using **plain JARs** (as opposed to **modular JARs**).

The code is available [here] and is made of 3 artifacts:
- `example.api` has 1 package `org.example.api` with a public interface:
```GreetService.java
public interface GreetService {
    void greet();
}
```
- `example.impl` **depends on** `example.api` and has 1 package `org.example.impl` with a public implementation of `GreetService`:
```RudeGreeter.java
public class RudeGreeter implements GreetService {
    @Overrides
    public void greet() {
        ...
    }
}
```
- finalyy `example.app` **depends on** both `example.api` and `example.impl` and has 1 package `org.example.app` with a `Main` class:
```Main.java
public class Main {
    public static void main(String[] args) {
        GreetService greeter = new RudeGreeter();
        greeter.greet();
    }
}
```

Not the most intersting code but you get the idea...

So the way to compile, package and run this code using the old way is to use the class-path:
```sh
$ maven clean package 
$ java -cp example.api/target/org.example.api-1.0-SNAPSHOT.jar:example.impl/target/org.example.impl-1.SNAPSHOT.jar:example.app/target/org.example.app-1.0-SNAPSHOT.jar org.example.app.Main
```

Now the thing is although the api, the impl and the app are in separate JARs, JARs have *no boundaries* and as we said the distinction between 
private and public API is only enforce at compile time. 

To demonstrate this, lets add a private implemantation `PrivateGreeter` of `GreetService` in `example.impl`:
```PrivateGreeter.java
package org.example.impl;

import org.example.api.GreetService;

class PrivateGreeter implements GreetService {
    @Override
    public void greet() {
        ...
    }
}
```

And let's use that in `org.example.app.Main#main()` using reflection:
```Main.java
package org.example.app;

public class Main {
    public static void main(String[] args) {
        ...

        // Use reflection to instantiate a package-private class
        try {
            Constructor<?> defaultCtor = Class.forName("org.example.impl.PrivateGreeter")
                    .getDeclaredConstructor();
            defaultCtor.setAccessible(true);
            GreetService privateGreeter = (GreetService)defaultCtor.newInstance();
            privateGreeter.greet();
        } catch (InstantiationException e) {
            throw new RuntimeException(e);
        } catch (IllegalAccessException e) {
            throw new RuntimeException(e);
        } catch (InvocationTargetException e) {
            throw new RuntimeException(e);
        } catch (NoSuchMethodException e) {
            throw new RuntimeException(e);
        } catch (ClassNotFoundException e) {
            throw new RuntimeException(e);
        }
    }

}
```

And running again:

```sh
$ maven clean package 
$ java -cp example.api/target/org.example.api-1.0-SNAPSHOT.jar:example.impl/target/org.example.impl-1.SNAPSHOT.jar:example.app/target/org.example.app-1.0-SNAPSHOT.jar org.example.app.Main
[...]
Hey shit face!
Shhhh, hey just pretend I'm not here
```

## Automatic modules

Before going further, there are a few things you need to know: although we are loading our classes from the class path, under the hood
they are still part of a special module: the **unamed module**, which actually has no name and which exports all public types and 
opens all packages.

Since it has no name, the *unamed module* cannot be *required* by any other modules... except by *automatic modules*.

What are automatic modules? Basically it is a *plain JAR* loading from the **module-path**. The java platform figures out a module name
for the JAR from the `Automatic-Module-Name` property of the JAR's `MANIFEST.MF` if any or from the JAR's filename otherwise.

So let's turn our `example.app` artifact into an automatic module.

We modify the `pom.xml` to add a custom `MANIFEST.MF` as follow:
```MANIFEST.MF
Manifest-Version: 1.0
Build-Jdk-Spec: 17
Automatic-Module-Name: example.app
```

Now to launch our app, we use the `-p example.app/target` to add the `example.app` JAR in the module-path. The other classes 
are still loaded using the class-path and hence end up in the *unamed module*.

```sh
$ mvn clean package 
$ java -p example.app/target -cp example.api/target/classes:example.impl/target/classes -m example.app/org.example.app.Main
```

## Using proper modules

Our next step is to turn our artifact into proper java modules by adding `module-info.java` files in all of them:

Our `example.api` is pretty simple and only exports its public API package:
```
module example.api {
    exports org.example.api;
}
```

The `example.impl` module requires module `example.api` as it uses its `GreetService` interface and exports its `org.example.impl`
package as its public `RudeGreeter` is still used in our main function.
```
module example.impl {
    requires example.api;

    exports org.example.impl;
}
```

Finally our `example.app` module requires both `example.api` and `example.impl` and exports its main class package:
```
module example.app {
    requires example.api;
    requires example.impl;

    exports org.example.app;
}
```

And now we launch everything from the module path:
```sh
$ mvn clean compile
$ java -p example.api/target/classes:example.impl/target/classes:example.app/target/classes -m example.app/org.example.app.Main
```

However we now have a problem:
```
Hey shit face!
Exception in thread "main" java.lang.reflect.InaccessibleObjectException: Unable to make public org.example.impl.PrivateGreeter() accessible: module example.impl does not "opens org.example.impl" to module example.app
	at java.base/java.lang.reflect.AccessibleObject.checkCanSetAccessible(AccessibleObject.java:354)
	at java.base/java.lang.reflect.AccessibleObject.checkCanSetAccessible(AccessibleObject.java:297)
	at java.base/java.lang.reflect.Constructor.checkCanSetAccessible(Constructor.java:188)
	at java.base/java.lang.reflect.Constructor.setAccessible(Constructor.java:181)
	at example.app@1.0-SNAPSHOT/org.example.app.Main.main(Main.java:17)
```

Now that we use proper module for `example.impl`, the package-private class `PoliteGreeter` is not **opened** for reflection as mentioned
by the exception message. 
We can easily fix that by opening `org.example.impl` in `example.impl`'s `module-info.java`:
```
module example.impl {
    requires example.api;

    exports org.example.impl;

    opens org.example.impl;
}
```

Recompile and relaunch. All good?

But there is still a major flaw in our code: our app instead of simply depending on the `GreetService` abstraction, also depends
on its `RudeGreeter` implementation. 

Let's fix that now using `ServiceLoader`.

## Module uses and provides

To decouple our app from the service implementation, we can make use the Java platform `ServiceLoader` facility.

For that we need first to declare that our `example.app` module **uses** the `example.api` `GreetService` service interface:
```
module example.app {
    requires org.example.api;

    exports org.example.app;

    uses org.example.api.GreetService;
}
```

And then to tell the module system that our `example.impl` *provides* an implementation of the `GreetService` interface:
```
module example.impl {
    requires example.api;

    exports org.example.impl;

    provides org.example.api.GreetService with org.example.impl.RudeGreeter, org.example.impl.PoliteGreeter;
}
```

Recompile and relaunch:
```sh
$ mvn clean compile
$ java -p example.api/target/classes:example.impl/target/classes:example.app/target/classes -m example.app/org.example.app.Main
[...]
Hey shit face!
Hello dear friend
```

Tada! No more dependency on implementation details! Services implementations are automatically loaded using the `ServiceLoader` API.


