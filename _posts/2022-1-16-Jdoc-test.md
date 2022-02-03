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

Once we have sources in place basically we want to compile and run them.  
The simplest thing I could think is binding everything to a String template which represent our source code wrapped in a common interface 
so if compile actually works we load it into class loader, create an instance cast it to Callable in this case (because of checked exceptions)  
and run it.

Binding is pretty simple; import everything from type where javadoc is written, add all doc test imports and wrap into Callable.

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

Once we have that we write everything into a file, must have a valid folder structure matching the package and it's ready for compile.
Dynamic compile sounds more interesting than it actually is - Java kills a bit of fun here with a clumsy [dynamic compiler API](https://docs.oracle.com/javase/7/docs/api/javax/tools/JavaCompiler.html).

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

Classpath here is a bit tricky to resolve because of project dependencies - so to keep it simple dependencies are copied into target and  
along with compiled classes that's all we need to run any code in doc test - ofc that can be improved and if a miracle happens someone's PR  
will address this issues on plugin side (we'll get later to that) to resolve all project dependencies without the overhead of copying.

Dynamic compile could also be improved to compile everything in-memory with a custom compiler file manager, so we don't need to create anything on FS.

Oh well... once we compile the source all that's left is to load it into class loader, create and instance and in our case 
invoke `call()`

```kotlin
    private fun createClassInstance(path: Path, fullClassName: String): Callable<*> {
        log.debug("Creating instance of $fullClassName")
        val paths = classpathElements.map { Path.of(it).toUri().toURL() }.toMutableList()
        paths += path.toUri().toURL()

        val classLoader = URLClassLoader.newInstance(paths.toTypedArray()).also {
            it.setDefaultAssertionStatus(true)
        }
        return classLoader.loadClass(fullClassName)
            .getDeclaredConstructor()
            .newInstance()
            as Callable<*>
    }
```

Core part is fail-fast so any compile or runtime errors will be propagated up call stack.

### Plugin

We need to run the core part somehow - creating a maven plugin is the first choice here, easy to use and integrate into build lifecycle.  
Nothing special on the plugin side just collects all classpath elements - in this case all in project target and invokes the core lib.

```kotlin
@Mojo(name = "jdoctest", defaultPhase = LifecyclePhase.VERIFY)
class JDocTestMojo : AbstractMojo() {

    @Parameter(defaultValue = ".")
    lateinit var docPath: String

    @Parameter(defaultValue = "\${project}", required = true, readonly = true)
    lateinit var project: MavenProject

    override fun execute() {
        try {
            val cp = project.runtimeClasspathElements.plus(projectDependencies())
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

Doc test validation is run on verify phase by default and support path/package opt-in validation just in case.



#### GH Repo
[Available here](https://github.com/dejankos/jdoctest)