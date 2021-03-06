---
type: doc
layout: reference
category: "Syntax"
title: "Type-Safe Groovy-Style Builders"
---


The concept of [builders](http://groovy.codehaus.org/Builders) is rather popular in the *Groovy* community. 
Builders allow for defining data in a semi-declarative way. Builders are good for [generating XML](http://groovy.codehaus.org/GroovyMarkup), 
[laying out UI components](http://groovy.codehaus.org/GroovySWT), 
[describing 3D scenes](http://www.artima.com/weblogs/viewpost.jsp?thread=296081) and more...

For many use cases, Kotlin allows to *type-check* builders, which makes them even more attractive than the 
dynamically-typed implementation made in *Groovy* itself.

For the rest of the cases, Kotlin supports Dynamic types builders.

## A type-safe builder example

Consider the following code that is taken from [here](http://groovy.codehaus.org/Builders) and slightly adapted:

``` kotlin
import html.*

fun result(args : Array<String>) =
  html {
    head {
      title {+"XML encoding with Kotlin"}
    }
    body {
      h1 {+"XML encoding with Kotlin"}
      p  {+"this format can be used as an alternative markup to XML"}

      // an element with attributes and text content
      a(href = "http://jetbrains.com/kotlin") {+"Kotlin"}

      // mixed content
      p {
        +"This is some"
        b {+"mixed"}
        +"text. For more see the"
        a(href = "http://jetbrains.com/kotlin") {+"Kotlin"}
        +"project"
      }
      p {+"some text"}

      // content generated by
      p {
        for (arg in args)
          +arg
      }
    }
  }
```

This is a completely legitimate Kotlin code. 
You can play with this code online (modify it and run in the browser) [here](http://kotlin-demo.jetbrains.com/?folder=Longer_examples&name=HTML_Builder).

## How it works

Let's walk through the mechanisms of implementing type safe builders in Kotlin. First of all we need to define the model we want to build, in this case we need to model HTML tags. It is easily done with a bunch of classes. For example, `HTML` is a class that describes the `<html>` tag, i.e. it defines children like `<head>` and `<body>`.
(See its declaration [below](#declarations).)

Now, let's recall why we can say something like this in the code:

``` kotlin
html {
 // ...
}
```

This is actually a function call that takes a [function literal](lambdas.html) as an argument (see [this page](lambdas.html#higher-order-functions) for details). Actually, this function is defined as follows:

``` kotlin
fun html(init : HTML.() -> Unit) : HTML {
  val html = HTML()
  html.init()
  return html
}
```

This function takes one parameter named `init`, which is itself a function. Actually, it is an [extension function](extensions.html) that has a receiver of type `HTML`
(and returns nothing interesting, i.e. `Unit`). So, when we pass a function literal to as an argument to `html`, it is typed as an extension function literal, and 
there's a *this* reference available:

``` kotlin
html {
  this.head { /* ... */ }
  this.body { /* ... */ }
}
```

(`head` and `body` are member functions of `html`.)

Now, *this* can be omitted, as usual, and we get something that looks very much like a builder already:

``` kotlin
html {
  head { /* ... */ }
  body { /* ... */ }
}
```

So, what does this call do? Let's look at the body of `html` function as defined above. It creates a new instance of `HTML`, then it initializes it by calling the 
function that is passed as an argument (in our example this boils down to calling `head` and `body` on the `HTML` instance), and then it returns this instance. 
This is exactly what a builder should do.

The `head` and `body` functions in the `HTML` class are defined similarly to `html`. 
The only difference is that they add the built instances to the `children` collection of the enclosing `HTML` instance:

``` kotlin
fun head(init : Head.() -> Unit) {
  val head = Head()
  head.init()
  children.add(head)
  return head
}

fun body(init : Body.() -> Unit) {
  val body = Body()
  body.init()
  children.add(body)
  return body
}
```

Actually these two functions do just the same thing, so we can have a generic version, `initTag`:

``` kotlin
  protected fun initTag<T: Element>(tag: T, init: T.() -> Unit): T {
    tag.init()
    children.add(tag)
    return tag
  }
```

So, now our functions are very simple:

``` kotlin
fun head(init : Head.() -> Unit) = initTag(Head(), init)

fun body(init : Body.() -> Unit) = initTag(Body(), init)
```

And we can use them to build `<head>` and `<body>` tags. 


One other thing to be discussed here is how we add text to tag bodies. In the example above we say something like

``` kotlin
html {
  head {
    title {+"XML encoding with Kotlin"}
  }
  // ...
}
```

So basically, we just put a string inside a tag body, but there is this little "+" in front of it, so it is a function call that invokes a prefix "plus" operation. 
That operation is actually defined by an extension function `plus` that is a member of the `TagWithText` abstract class (a parent of `Title`):

``` kotlin
fun String.plus() {
  children.add(TextElement(this))
}
```

So, what the prefix "+" does here is it wraps a string into an instance of `TextElement` and adds it to the `children` collection, so that it becomes a proper part of the tag tree.

All this is defined in a package `html` that is imported at the top of the builder example above. In the next section you can read through the full definition of this namespace.

## Full definition of the `html` namespace

This is how the namespace `html` is defined (only the elements used in the example above). It builds an HTML tree. It makes heavy use of [Extension functions](extensions.html) and 
[Extension function literals](lambdas.html#extension-function-literals).

<a href='declarations'></a>

``` kotlin
package html

import java.util.ArrayList
import java.util.HashMap

trait Element {
    fun render(builder: StringBuilder, indent: String)

    fun toString(): String? {
        val builder = StringBuilder()
        render(builder, "")
        return builder.toString()
    }
}

class TextElement(val text: String): Element {
    override fun render(builder: StringBuilder, indent: String) {
        builder.append("$indent$text\n")
    }
}

abstract class Tag(val name: String): Element {
    val children: ArrayList<Element> = ArrayList<Element>()
    val attributes = HashMap<String, String>()

    protected fun initTag<T: Element>(tag: T, init: T.() -> Unit): T {
        tag.init()
        children.add(tag)
        return tag
    }

    override fun render(builder: StringBuilder, indent: String) {
        builder.append("$indent<$name${renderAttributes()}>\n")
        for (c in children) {
            c.render(builder, indent + "  ")
        }
        builder.append("$indent</$name>\n")
    }

    private fun renderAttributes(): String? {
        val builder = StringBuilder()
        for (a in attributes.keySet()) {
            builder.append(" $a=\"${attributes[a]}\"")
        }
        return builder.toString()
    }
}

abstract class TagWithText(name: String): Tag(name) {
    fun String.plus() {
        children.add(TextElement(this))
    }
}

class HTML(): TagWithText("html") {
    fun head(init: Head.() -> Unit) = initTag(Head(), init)

    fun body(init: Body.() -> Unit) = initTag(Body(), init)
}

class Head(): TagWithText("head") {
    fun title(init: Title.() -> Unit) = initTag(Title(), init)
}

class Title(): TagWithText("title")

abstract class BodyTag(name: String): TagWithText(name) {
    fun b(init: B.() -> Unit) = initTag(B(), init)
    fun p(init: P.() -> Unit) = initTag(P(), init)
    fun h1(init: H1.() -> Unit) = initTag(H1(), init)
    fun a(href: String, init: A.() -> Unit) {
        val a = initTag(A(), init)
        a.href = href
    }
}

class Body(): BodyTag("body")

class B(): BodyTag("b")
class P(): BodyTag("p")
class H1(): BodyTag("h1")
class A(): BodyTag("a") {
    public var href: String
        get() = attributes["href"]!!
        set(value) {
            attributes["href"] = value
        }
}

fun html(init: HTML.() -> Unit): HTML {
    val html = HTML()
    html.init()
    return html
}
```


### Appendix. Making Java classes nicer

In the code above there's something that looks very nice:

``` kotlin
  class A() : BodyTag("a") {
    var href : String
      get() = attributes["href"]!!
      set(value) { attributes["href"] = value }
  }
```

We access the `attributes` map as if it were an "associative array": just with the `[]` operation. By [convention](operator-overloading.html) this compiles to a call to 
`get(K)` or `set(K, V)`, all right. But we said that `attributes` was a *Java* `Map`, i.e. it does NOT have a `set(K, V)`. 
This problem is easily fixable in Kotlin:

``` kotlin
  fun <K, V> Map<K, V>.set(key : K, value : V) = this.put(key, value)
```
So, we simply define an [extension function](extensions.html) `set(K, V)` that delegates to vanilla `put` and make a Kotlin operator available for a *Java* class.
