---
layout: post title: JDocTest - test your documentation examples description: Compile and run example code in javadoc
tags: java javadoc ast compile kotlin comments: true
---

Some languages (Rust, Python...) have this feature where you can execute example code from comments .  
For Rust there is rustdoc and the idea behind it is really simple:

`rustdoc supports executing your documentation examples as tests. This makes sure that examples within your documentation are up to date and working.`

and an example doc from `std::collections::HashMap` looks like this:

```
  /// Inserts a key-value pair into the map.
    ///
    /// If the map did not have this key present, [`None`] is returned.
    ///
    /// If the map did have this key present, the value is updated, and the old
    /// value is returned. The key is not updated, though; this matters for
    /// types that can be `==` without being identical. See the [module-level
    /// documentation] for more.
    ///
    /// [module-level documentation]: crate::collections#insert-and-complex-keys
    ///
    /// # Examples
    ///
    /// ```
    /// use std::collections::HashMap;
    ///
    /// let mut map = HashMap::new();
    /// assert_eq!(map.insert(37, "a"), None);
    /// assert_eq!(map.is_empty(), false);
    ///
    /// map.insert(37, "b");
    /// assert_eq!(map.insert(37, "c"), Some("b"));
    /// assert_eq!(map[&37], "c");
    /// ```
```

Pretty simple and self-explanatory while for the JVM world you usually need to search for some examples if you want to  
see how it's used; this of course is a trivial example, but you get the idea.

So an equivalent example for Java would be: 
```java 
    /**
     * <pre>
     *     {@code
     *      import java.util.HashMap;
     *      HashMap<Integer, String> map = new HashMap<Integer, String>();
     *      assert map.put(37, "a") == null;
     *      assert map.isEmpty() == false;
     *
     *      map.put(37, "b");
     *      assert map.put(37, "c") == "b";
     *      assert map.get(37) == "c";
     *     }
     * </pre>
     */
```

Well not really an equivalent... `assert_eq!` macro is part of Rust std while assertions are an opt-in runtime feature in Java;  
but to keep dependencies at minimum the best approach is to just enable them when documentation code is run.   

Speaking of dependencies I want everything that's in current class scope to be implicitly available in documentation code  
to reduce boilerplate... but it also makes sense for example code to depend on something outside of current scope so   
documentation code imports must be supported as well including those from external dependencies.

Can't think of a better name than JDocTest based on what I've just described. 

### Parsing source into AST

I had no idea where to start with AST in Java; it's not really something one uses often.
There are multiple what seems like abandoned projects, other require subscription for any relevant piece of 
documentation but at the end I've decided to go with [Spoon](https://spoon.gforge.inria.fr/) 

`open-source library to analyze, rewrite, transform, transpile Java source code. It parses source files to build a well-designed AST with powerful analysis and transformation API`

And it really is just like they advise it; finding all Javadoc comments and traversing parents it's extremely simple.

I also want users to opt-in when jdoctest should be run and based on how javadocs are usually structured; 
at least in some newer versions it makes sense to just introduce a `<javadoc>` element and wrap all code examples.

```java 
    /**
     * <jdoctest>
     * <pre>
     *     {@code
     *      System.out.println("hello!")
     *     }
     * </pre>
     * </jdoctest>
     */
```

Pretty simple example should be easy to understand how the journey goes from parsing to compiling and running.  

Taken from source lib a high level overview how javadocs are parsed:

```kotlin
    internal fun extract() =
        buildModel().getElements(javadocFilter) // Build AST and find only comments
            .filter { it.isJavadoc() } // interested only in javadocs
            .filter { it.isDocTest() } // interested only in javadoc with <javadoc> elements
            .mapNotNull { jDoc ->
                extractTypeData(jDoc)?.let { // extract type data -> class, interface, enum or record
                    DocTestContext(it, parseJavadoc(jDoc)) // parse code into a model that eventually can be compiled
                }
            }
```

There is a bit more behind the scenes; for any type info (class, interface, enum or record) AST traversal is required  
and at some point type is erased from imports but other than that really nothing special and the outcome of this is:  

```
DocTestContext(  
    typeInfo=TypeInfo(  
            package=io.github.dejankos.valid,   
            name=ExampleInterface, 
            imports=[]
    ), 
    docsCode=[
            DocTestCode(
                docTestImports=[],
                docTestCode=[System.out.println("hello!")],
                originalContent=System.out.println("hello!")
            )
        ]
)
```
Notice how docsCode is an array type - that because we can have multiple code examples for the same API and  
you probably want to run them separately.

### Dynamic compile


### Plugin

