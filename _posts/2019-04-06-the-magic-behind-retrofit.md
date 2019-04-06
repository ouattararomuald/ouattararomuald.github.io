---
layout: post
author: Romuald Ouattara
excerpt_separator: <!--more-->
categories: [Http]
tags: [Android, Java, Retrofit, Http]
---

The first time I used [retrofit](https://github.com/square/retrofit) I was very impressed.
You give it an interface and every time you call a method on that interface something happens. How is that possible?
This is what this article is about.

<!--more-->

## How to use retrofit

I shamelessly copied the retrofit's [documentation](http://square.github.io/retrofit/).

Let's say you are writing a Github client. If you are doing so you will need to consume the Github API
at a moment and make HTTP requests. If you use retrofit, all you have to do is to create an interface:

```java
public interface GitHubService {
  @GET("users/{user}/repos")
  Call<List<Repo>> listRepos(@Path("user") String user);
}
```

Then you let retrofit generates an implementation of the interface.

```java
Retrofit retrofit = new Retrofit.Builder()
    .baseUrl("https://api.github.com/")
    .build();

GitHubService service = retrofit.create(GitHubService.class);
```

The magic happens on the last line

```java
GitHubService service = retrofit.create(GitHubService.class);
```

The documentation states that retrofit generates an implementation of the interface `GitHubService`.
But the truth is that there is no class implementing this interface, nor a class generated through
annotation processing mechanism. Then how is it possible?

## Proxy

According to Oracle's documentation 

> [Proxy](https://docs.oracle.com/javase/10/docs/api/java/lang/reflect/Proxy.html) 
> provides static methods for creating objects that act like instances of interfaces but allow for customized method invocation.

This is good to know but practice is better. Consider the following interface:

```kotlin
interface RssReader {
  fun readRss(url: String): String
  fun readAtom(url: String): String
}
```

What we want is to create a class that will act as a proxy of the given interface.
This means that each time we will invoke a method on our interface, the proxy will download the syndication feed
at the given `url` and return a `String`. It will give something like this:

```kotlin
fun main() {
  val syndication = Syndication()
  val rssReader: RssReader = syndication.create(RssReader::class.java)
  val messageRss = rssReader.readRss("file://url-rss-1")
  val messageAtom = rssReader.readAtom("file://url-atom-2")

  println(messageRss)
  println("--------------------------------")
  println(messageAtom)
}
```

**With `Syndication`**

```kotlin
class Syndication {

  @Suppress("UNCHECKED_CAST") fun <T> create(reader: Class<T>): T {
    return Proxy.newProxyInstance(reader.classLoader, arrayOf(reader),
        object : InvocationHandler {
          @Throws(Throwable::class)
          override fun invoke(proxy: Any, method: Method, args: Array<Any>?): Any {
            // If the method is a method from Object then defer to normal invocation.
            return if (method.declaringClass == Any::class.java) {
              method.invoke(this, args)
            } else {
              val sb = StringBuilder()

              sb.append("The method ${method.name}(...) was invoked.\n")
              sb.append("...with ${args?.size} argument(s):\n")
              for (i in 0 until(args?.size ?: 0)) {
                sb.append("   > argument: ${args?.get(i)}")
              }

              return sb.toString()
            }
          }
        }) as T
  }
}
```

The method `create` above creates a **Proxy** instance for the given class loader (`RssReader`).
Each time a method of the interface `RssReader` is invoked we do whatever we want with the argument 
passed and return what we want depending on the invoked method. In our example we just built a `String` 
to return. But in real world application, we would make an http request to download the feed, parse 
the result and return an object of the same return type as the invoked method.

Just in case you wondered, If you run the `main` method above, it will produce the output below:

```
The method readRss(...) was invoked.
...with 1 argument(s):
   > argument: file://url-rss-1
--------------------------------
The method readAtom(...) was invoked.
...with 1 argument(s):
   > argument: file://url-atom-2
```

As you can see Proxies are a very powerful but unfortunately only a handful of people know about them.
They are used in [Hibernate](http://hibernate.org/) for lazy loading entities, [Spring](https://spring.io/)
for [Aspect Oriented Programming](https://en.wikipedia.org/wiki/Aspect-oriented_programming).

After I read retrofit's code and I had an idea of a library that use the same concept
to download syndication feed. The library is called 
[Syndication](https://github.com/ouattararomuald/syndication), is available on GitHub and of course
**contributions** are welcome.