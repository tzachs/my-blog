+++
categories = ["java"]
title = "IllegalAccessError - Using URLClassLoader and package private methods"
date = "2016-09-10"

+++

When working with URLClassLoader, one of the things we need to watch for, is the 'run-time packages'.
According to Java spec, a class is determined by it's binary name and it's class loader. Meaning, the class 'signature' is the binary name and the class loader used to load it. 

Once loaded, each class belongs to a single *run-time* package.

https://docs.oracle.com/javase/specs/jvms/se7/html/jvms-5.html#jvms-5.3

> At run time, a class or interface is determined not by its name alone, but by a pair: its binary name and its defining class loader. Each such class or interface belongs to a single run-time package. The run-time package of a class or interface is determined by the package name and defining class loader of the class or interface.



To demonstrate this problem (which I've encountered at work ofcourse), I've created the following example.

Full code is at https://github.com/tzachs/IllegalAccessErrorExample

# Example explained

I've created 3 modules (using maven) named IllegalAccessErrorA, IllegalAccessErrorB, IllegalAccessErrorC.

* IllegalAccessErrorA module
  * Has one class named IllegalAccessErrorA under package com.tzach.example
  * Has compile time dependency on module IllegalAccessErrorC
  * Packaged with module IllegalAccessErrorC
  * The sole purpose of IllegalAccessErrorA is to load module IllegalAccessErrorB using reflection and invoking a method in that module (the method can be usePrivatePackage or usePublic)

* IllegalAccessErrorB module
	* Has one class named IllegalAccessErrorB under package com.tzach.example
	* As IllegalAccessErrorA, has compile time dependency on IllegalAccessErrorC
	* Packaged with module IllegalAccessErrorC
	* Has 3 functions:
		* main() - calling new IllegalAccessErrorC().usePrivatePackage()
		* usePackagePrivate() - calling new IllegalAccessErrorC().usePrivatePackage();
		* usePublic() - calling new IllegalAccessErrorC().usePublic();

* IllegalAccessErrorC module
	* Has one class named IllegalAccessErrorC under package com.tzach.example
	* Has 2 methods:
		* void usePrivatePackage()
		* public void usePublic()

Both IllegalAccessErrorA and IllegalAccessErrorB have legal calls to IllegalAccessErrorC since they are in the same package.
If you run IllegalAccessErrorB module by itself, no error will be thrown by the JVM since both IllegalAccessErrorB and IllegalAccessErrorC classes are in the same **run-time package**, **BUT**, when running **IllegalAccessErrorA** module, using the argument **usePackagePrivate**, we get  **IllegalAccessError** thrown.

{{< highlight java >}}
C:\Work\IllegalAccessErrorExample\IllegalAccessErrorA>java -jar target\IllegalAccessErrorA-1.0-SNAPSHOT-jar-with-dependencies.jar ..\Illegal
AccessErrorB\target usePackagePrivate
Exception in thread "main" java.lang.reflect.InvocationTargetException
        at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
        at sun.reflect.NativeMethodAccessorImpl.invoke(Unknown Source)
        at sun.reflect.DelegatingMethodAccessorImpl.invoke(Unknown Source)
        at java.lang.reflect.Method.invoke(Unknown Source)
        at com.tzach.example.IllegalAccessErrorA.main(IllegalAccessErrorA.java:51)
Caused by: java.lang.IllegalAccessError: tried to access method com.tzach.example.IllegalAccessErrorC.usePrivatePackage()V from class com.tz
ach.example.IllegalAccessErrorB
        at com.tzach.example.IllegalAccessErrorB.usePackagePrivate(IllegalAccessErrorB.java:19)
        ... 5 more
{{< /highlight >}}

Why is that? Well, the module IllegalAccessErrorB was loaded through a child URLClassLoader by module IllegalAccessErrorA.
Meaning, the class IllegalAccessErrorB is in a different **run-time** package than IllegalAccessErrorC which was loaded when the 
program started, using the Application Class Loader.

This violates the 'accessibility' rule, thus, we get IllegalAccessError https://docs.oracle.com/javase/specs/jvms/se7/html/jvms-5.html#jvms-5.4.4

> A class or interface C is accessible to a class or interface D if and only if either of the following conditions is true:
> C and D are members of the same run-time package

**NOTICE**

Running the IllegalAccessErrorA with the argument usePublic, will run okay, since both methods are in the **runtime package**

{{< highlight java >}}
C:\Work\IllegalAccessErrorExample\IllegalAccessErrorA>java -jar target\IllegalAccessErrorA-1.0-SNAPSHOT-jar-with-dependencies.jar ..\Illegal
AccessErrorB\target usePublic
Ignore the next line since we called it from a public method
You are calling package private method!
{{< /highlight >}}

# In conclusion

When writing package private method, keep in mind that those classes are not to be used by reflection. If so, you need to wrap those methods with a public method.
