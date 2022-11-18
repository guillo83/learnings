---
title: Java resource loading cookbook
date: 2022-11-16
---

# Quick recipes
## Load resource in the same package as the calling code
Use CallingClass.class.getResourceAsStream("fileName.ext"), where "fileName.ext" is in the same package than CallingClass. The folder structure of your project is similar to this:
```
- YourModule
  - SourcesRoot
    - com/myPackage/path <- The CallingClass package is com.myPackage.path
       - CallingClass.java
  - ResourcesRoot
     - com/myPackage/path <- The path of fileName.ext (relative to ResourcesRoot) should match CallingClass.java path (relative to SourcesRoot)
        - fileName.ext 
```

Here is an example:

```
package com.myPackage.path;

public class CallingClass {
  public static void main(String[] args) {
    InputStream useClass = CallingClass.class.getResourceAsStream("samePackage.txt");
    // printStream(useClass);
  }
}
```

Folder project stucture:

```
- MyModule
  - src
    - com/myPackage/path
       - CallingClass.java
  - resources
     - com/myPackage/path
        - samePackage.txt 
```

## Load resource in a different package as the calling code
### Option 1: Use CallingClass.class.getResourceAsStream("/full/path/fileName.ext")
Notice that the resource path starts with '/'.

Here is an example:

```
package com.myPackage.path;

public class CallingClass {
  public static void main(String[] args) {
    InputStream useClass = CallingClass.class.getResourceAsStream("/com/other/package/message.txt");
    // printStream(useClass);
  }
}
```

Folder project stucture:

```
- MyModule
  - src
    - com/myPackage/path
       - CallingClass.java
  - resources
     - /com/other/package
        - message.txt 
```

### Option 2: Use CallingClass.class.getClassLoader().getResourceAsStream("full/path/fileName.ext")
Notice that the resources name doesn't start with '/'.

Here is an example:

```
package com.myPackage.path;

public class CallingClass {
  public static void main(String[] args) {
    InputStream useClass = CallingClass.class.getClassLoader().getResourceAsStream("/com/other/package/message.txt");
    // printStream(useClass);
  }
}
```

Folder project stucture:

```
- MyModule
  - src
    - com/myPackage/path
       - CallingClass.java
  - resources
     - /com/other/package
        - message.txt 
```

# More in depth: How to load a resource in Java

Loading a resource in Java can be somewhat confusing the first time. After you read and understand the documentation, it is fairly obvious how to load resources.

## How Java loads resources?
The short answer is using getResourceAsStream API (or any other getResource* API). The interesting part is that there are two distinct implementations of the getResourceAsStream API and if you use the wrong one you will not be able to load your resource.

## Class.getResourceAsStream
You access this API by invoking YourCurrentClass.class.getResourceAsStream("resourceName"). 

As the documentation of this API mentions, the absolute resource name is constructed from the given resource name using this algorithm:

- If the name begins with a '/' ('\u002f'), then the absolute name of the resource is the portion of the name following the '/'.
- Otherwise, the absolute name is of the following form:
modified_package_name/name
Where the modified_package_name is the package name of this object with '/' substituted for '.' ('\u002e').

As an example, lets take the following piece of code:

```
package com.gdev.libs;

public class ResourceLoader {
  public static void main(String[] args) {
    InputStream useClass = ResourceLoader.class.getResourceAsStream("samePackage.txt");
    printStream(useClass);
  }

  private static void printStream(InputStream stream) {
    if (stream == null) {
      System.out.println("Resource is null!!");
    }

    String text = new BufferedReader(new InputStreamReader(stream))
            .lines()
            .collect(Collectors.joining(System.lineSeparator()));
    System.out.println(text);
  }
}
```

The modified_package_name in this case will be the package of `ResourceLoader` class (`com.gdev.libs`), hence the absolute resource name for "samePackage.txt" will be:
```
${RESOURCES_ROOT}/com/gdev/libs/samePackage.txt
```
`${RESOURCES_ROOT}` folder is the folder marked on your modules as "Resources root". On IntelliJ, you mark this folder by right click on the folder > "Mark Directory as" > "Resources root". On Image 1, such folder is called "resources".

You can also give the API the absolute resource name as follows:
```
InputStream useClass = ResourceLoader.class.getResourceAsStream("/com/gdev/libs/samePackage.txt");
```
Notice that the resource name starts with '/', which as per the documentation, is the correct way to pass the absolute name of the resource.

## ClassLoader.getResourceAsStream
You access this API by invoking YourCurrentClass.class.getClassLoader().getResourceAsStream("resourceName").

The way the `resourceName` is resolved can get a bit more complex (from the documentation):

...first search the parent class loader for the resource; if the parent is null the path of the class loader built-in to the virtual machine is searched. That failing, this method will invoke findResource(String) to find the resource.

Lets take the following piece of code:

```
package com.gdev.libs;

public class ResourceLoader {
  public static void main(String[] args) {
    InputStream useClassLoader = ResourceLoader.class.getClassLoader().getResourceAsStream("atResourcesRoot.txt");
    // printStream(useClassLoader);
  }
}
```

In the previous code, chain of class loaders for `ResourceLoader` class is `AppClassLoader` > `PlatformClassLoader` > `BootClassLoader`. This chain is important to mention because the implementation of `ClassLoader.getResourceAsStream` implicitely will use `BootLoader.findResource`, which will return the URL to the given resource in any of the modules defined to the boot loader and the boot class path. Although the full logic get pretty complex, for the practical purposes of our example above, the resource "atResourcesRoot.txt" will get resolved to:

```
${RESOURCES_ROOT}/atResourcesRoot.txt
```
Notice that the resource name ("atResourcesRoot.txt") was resolved as an absolute path even when it doesn't start with '/'.  


https://docs.oracle.com/javase/9/docs/api/java/lang/ClassLoader.html
https://www.baeldung.com/java-classloaders
