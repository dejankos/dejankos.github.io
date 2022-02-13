---
layout: post 
title: Compile and run example code in Javadoc with JDocTest
description: JDocTest - test your documentation examples
tags: java javadoc ast compile kotlin rust
comments: true
---

![_config.yml]({{ site.baseurl }}/images/jdoctest.jpg)

Some languages (Rust, Python...) have this feature where you can execute example code from comments.
For Rust there is rustdoc and the idea behind it is really simple:

*`Rustdoc supports executing your documentation examples as tests. 
This makes sure that examples within your documentation are up to date and working.`*  

And an example doc from `std::collections::HashMap` looks like this:

```rust
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

Pretty simple and self-explanatory. On the other hand, in the JVM world you usually need to search for some examples if you want to
see how the code is used. This, of course, is a trivial example, but you get the idea.

Well... There is also [JEP-413](https://openjdk.java.net/jeps/413) targeting Java 18 of which existence I was completely unaware 
of at the time of developing this project. It has some nice features such as syntax highlights, linkage, and IDE support, but also some
code limitations which makes JDocTest currently way cooler.
Still, it's nice to see some progress in this area. 

And back to our example... So an equivalent example for Java would be: 
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

Well, not really an equivalent... *`assert_eq!`* macro is a part of Rust std while assertions are an opt-in runtime feature in Java.
But to keep dependencies at a minimum and not to force the user to use some third-party library, the best approach is to
just enable dependencies when documentation code is run. This way we can perform assertions out of the box but also have the flexibility 
to use some third-party library. It's really up to the user.

Speaking of dependencies, I want everything in the current class' scope to be implicitly available in documentation code
to reduce boilerplate. But it also makes sense for example code to depend on something outside the current scope so
documentation code imports must be supported, either from the current project or some third party.

### Parsing source into AST

I had no idea where to start with AST in Java; it's not really something one uses often.
There are multiple (what seem like) abandoned projects and others require a subscription for any relevant piece of 
documentation so in the end, I've decided to go with [Spoon](https://spoon.gforge.inria.fr/).

Based on docs, it's an:  
*`open-source library to analyze, rewrite, transform, transpile Java source code. 
It parses source files to build a well-designed AST with powerful analysis and transformation API`.*

And it really is just like they advertise it: finding all Javadoc comments and traversing AST is extremely simple.
So Spoon it is.  

One of its very basic features is the ability to opt-in when jdoctest should be run. And based on how javadocs are usually structured 
at least in some newer versions, it makes sense to just introduce a `<javadoc>` element and wrap all code examples.

```java 
/**
 * <jdoctest>
 * <pre>
 *     {@code
 *      System.out.println("hello!");
 *     }
 * </pre>
 * </jdoctest>
 */
```

With this simple example, it should be easy to understand how the journey from parsing to compiling and running goes.  

Taken from source lib, this is a higher-level overview of how javadocs are parsed:
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

private fun parseJavadoc(comment: CtJavaDoc): List<DocTestCode> {
    var state = DocTestState.NONE
    val res = mutableListOf<DocTestCode>()
    for (e in comment.javadocElements) {
        when (e) {
            is JavadocSnippet -> {
                if (e.toText().contains("<jdoctest>")) {
                    state = DocTestState.OPEN
                    continue
                }
                if (e.toText().contains("</jdoctest>")) {
                    state = DocTestState.CLOSED
                    continue
                }
            }
            is JavadocInlineTag -> {
                if (state == DocTestState.OPEN && e.type == JavadocInlineTag.Type.CODE) {
                    res += extractDocCode(e)
                }
            }
        }
    }

    return when (state) {
        DocTestState.CLOSED -> res
        else -> throw ParseException(
            "JDocTest parse error; javadoc fragment ${comment.docComment}",
            comment.content
        )
    }
}

private fun extractDocCode(javadoc: JavadocInlineTag) =
    javadoc.content.lineSequence()
        .fold(listOf<String>() to listOf<String>()) { acc, line ->
            if (line.contains("import"))
                acc.first + line to acc.second
            else
                acc.first to acc.second + line
        }
        .run {
            val (imports, code) = this
            DocTestCode(imports, code, javadoc.content)
        }
```

There is a bit more happening behind the scenes: for any type (class, interface, enum, or record), AST traversal is required
and at some point, type is erased from imports. Other than that, nothing more is needed to parse this code, and here is the outcome of it:  

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
                docTestCode=[System.out.println("hello!");],
                originalContent=System.out.println("hello!");
            )
    ]
)
```

Notice how docsCode is an array type. That's because we can have multiple code examples for the same API and
the user probably wants to run them separately.

### Dynamic compile

Once we have sources in place basically we just need to compile and run them.
We start by binding everything to a String template which represents our source code wrapped in a common interface,
Callable in this case. Although we obviously don't return any value here, Callable is the best choice because it already has 
a top-level checked exception in the method signature.
So if compile actually works, we load the compiled class into the class loader, create an instance, cast it to Callable, and run it.

Binding is just as described: imports from the current scope and possibly from documentation tests with code wrapped in Callable:

```kotlin
private fun bindDocTestCode(typeInfo: TypeInfo, docTestCode: DocTestCode) =
    """
        package ${typeInfo.`package`};
        ${typeInfo.imports.joinAsImportMultiline()}
        ${docTestCode.docTestImports.joinMultiline()}
        
        public class ${typeInfo.name}_JDocTest implements java.util.concurrent.Callable {
            public Object call() throws Exception {
                ${docTestCode.docTestCode.joinMultiline()}
                return null;
            }
        }
    """
```

Once we have that, we write everything into a file having a valid folder structure matching the package name, and it's ready for compiling.
The actual Java source file from a temp folder we use for compiling looks like this:

```java
package io.github.dejankos.valid;

public class ExampleInterface_JDocTest implements java.util.concurrent.Callable {
    public Object call() throws Exception {
        System.out.println("hello!");
        return null;
    }
}
```

Dynamic compiling sounds more interesting than it actually is; Java kills a bit of fun with a clumsy [dynamic compiler API](https://docs.oracle.com/javase/7/docs/api/javax/tools/JavaCompiler.html).

```kotlin
private fun compileClassSource(source: Path): DiagnosticCollector<JavaFileObject> {
    val compiler = ToolProvider.getSystemJavaCompiler()
    val diagnostics = DiagnosticCollector<JavaFileObject>()
    val fileManager = compiler.getStandardFileManager(diagnostics, null, null)

    val optionList = listOf("-classpath", classpath)
    fileManager.use {
        val fileObject = fileManager.getJavaFileObjects(source)
        compiler.getTask(null, fileManager, diagnostics, optionList, null, fileObject).call()
    }

    return diagnostics
}
```

Classpath here is a bit tricky to resolve because of the third-party dependencies, so to keep it simple dependencies are just copied into the target folder and
along with compiled project classes that's all we need to run any code in doc test. Of course, that can be improved, but it's easier to hope that someone's PR
will address these issues on the plugin side (later we'll get to that) and resolve all project dependencies without the overhead of copying.

Dynamic compiling could also be improved to compile everything in-memory with a custom compiler file manager, so we don't need to create anything on the file system.

Oh, well... Once the source file is compiled all that's left is loading it into the class loader, creating an instance, and, in our case, 
invoking `call()`:

```kotlin
private fun createClassInstance(path: Path, fullClassName: String): Callable<*> {
    log.debug("Creating instance of $fullClassName")
    val paths = classpathElements.map { Path.of(it).toUri().toURL() }.toMutableList() + path.toUri().toURL()

    val classLoader = URLClassLoader.newInstance(paths.toTypedArray()).also {
        it.setDefaultAssertionStatus(true)
    }
    return classLoader.loadClass(fullClassName)
        .getDeclaredConstructor()
        .newInstance()
        as Callable<*>
}
```

The core part is fail-fast so any compile or runtime errors will be propagated up the call stack.

### Plugin

We need to run the core part somehow. Creating a maven plugin is the first choice here because it's easy to use and integrate into the build lifecycle.
Nothing special on the plugin side; it just collects all the classpath elements (in this case, all the elements in the project target), and invokes the core lib.

```kotlin
@Mojo(name = "jdoctest", defaultPhase = LifecyclePhase.VERIFY)
class JDocTestMojo : AbstractMojo() {

    @Parameter(defaultValue = ".")
    lateinit var docPath: String

    @Parameter(defaultValue = "\${project}", required = true, readonly = true)
    lateinit var project: MavenProject

    override fun execute() {
        try {
            val cp = project.runtimeClasspathElements + projectDependencies()
            JDocTest().processSources(docPath, cp)
        } catch (e: RuntimeException) {
            throw MojoExecutionException(e.message, e.cause ?: e)
        }
    }

    private fun projectDependencies() = project.basedir?.listFiles()
        ?.filter { it.path.contains("target") }
        ?.flatMap { target ->
            target?.listFiles()
                ?.find { it.path.contains("dependency") }
                ?.listFiles()
                ?.map { it.absolutePath } ?: emptyList()
        } ?: throw IllegalStateException("Can't access project base dir")
}
```

Doc test validation is run on verify phase by default and supports path/package opt-in validation just in case.

#### Conclusion

This project is probably one of those that are fun to code but nobody will actually use it since JEP-413 will gradually introduce almost the same thing on a different level 
into Java 18. More work on top of JEP-413 is already planned.

Nevertheless, Github repository [available here](https://github.com/dejankos/jdoctest) containing sources and examples.