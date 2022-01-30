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

So the main idea behind this blog post is to be able to write something like this but for Java... written in Kotlin.

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
     *      assert map.get(37) == c;
     *     }
     * </pre>
     */
```