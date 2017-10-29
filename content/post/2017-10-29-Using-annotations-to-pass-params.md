+++
draft = false
date = "2017-10-29T09:50:13+02:00"
tags = ["Java"]
categories = ["java"]
slug = ""
title = "Passing params using Java Annotations"

+++

**Scenario:**

I needed a way to track down the path my program was going through and also, in each method that was tracked, to know the methods status (Passed or failed) and other meta data on the method...

**First problem - How to track down the path of all the method?**  

I've used the example from [https://docs.oracle.com/javase/8/docs/technotes/guides/reflection/proxy.html ]() to have a pre / post method.

**Second problem - How to get the meta data of the method?**  

To track the methods, I've created the annotation TrackThis.
The Annoation for now has only 3 fields:  
UniqueID  
Tag - Meta Data I needed  
Status - Passed or Failed  

In DebugProxy class, under invoke() method, I've added the following code in finally section

``` 
Annotation[] annotations = obj.getClass().getDeclaredMethod(m.getName(), 	m.getParameterTypes()).getDeclaredAnnotations();
	if (annotations.length > 0) {
	    Annotation a = annotations[0];
	    if (a.annotationType() == TrackThis.class) {
	        String tag = ((TrackThis) a).tag();
	        String status = ((TrackThis) a).status();
	        long uniqueID = ((TrackThis) a).uniqueID();
	        System.out.println(String.format("%s, %s, %s",
	                        tag, status, uniqueID));
	    }
	} else {
		System.out.println("No annotations!");
}
```

The code retrieves the annotation from the invoked method, and checks if the annoation type is TrackThis. If so it prints the meta data.

**Third problem - How to change the valus of the annoation meta data?**

To change the annoation value, I've used the method changeAnnotationValue() in SimpleService.

**Running Main**

Running Main will give you the following output

```
Mouse, true, 1  
Mouse, false, 1  
Mouse, true, 11  
Mouse, false, 11  
Mouse, true, 10  
Mouse, false, 10
```

As you can see, the status changed according to the current state of the method.

**NOTICE**: I've used synchronized in DebugProxy there is only one instance of SimpleService class (if you don't use Singleton you probably don't have to use synchronized)
