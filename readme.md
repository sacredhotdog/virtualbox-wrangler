

### VirtualBox SDK Reference

Taken from the chapter 'Using Java API':
---
#### Introduction
VirtualBox can be controlled by a Java API, both locally (COM / XPCOM) and from remote (SOAP) clients. 
As with the Python bindings, a generic glue layer tries to hide all platform differences, allowing for source and 
binary compatibility on different platforms.

#### Requirements
To use the Java bindings, there are certain requirements depending on the platform. First of all, you need 
JDK 1.5 (Java 5) or later. Also please make sure that the version of the VirtualBox API `.jar` file exactly matches the 
version of VirtualBox you use. To avoid confusion, the VirtualBox API provides versioning in the Java package name, 
e.g. the package is named `org.virtualbox_3_2` for VirtualBox version 3.2.

**XPCOM (for all platforms excluding Microsoft Windows)**

A Java bridge based on JavaXPCOM is shipped with VirtualBox. 
The class path must contain `vboxjxpcom.jar` and the `vbox.homeproperty` must be set to location where the VirtualBox 
binaries are. Please make sure that the JVM bitness matches the bitness of VirtualBox you use as the XPCOM bridge 
relies on native libraries.

Start your application like this:
 
```$ java -cp vboxjxpcom.jar -Dvbox.home=/opt/virtualbox MyProgram```


**COM (for Microsoft Windows)**

We rely on Jacob - a generic Java to COM bridge - which has to be installed 
seperately. See [the Jacob project](http://sourceforge.net/projects/jacob-project/) for installation instructions. 
Also, the VirtualBox provided `vboxjmscom.jar` must be in the class path.

Start your application like this:
 
```$ java -cp vboxjmscom.jar;c:\jacob\jacob.jar -Djava.library.path=c:\jacob MyProgram```


**SOAP (all platforms)**

Java 6 is required, as it comes with built-in support for SOAP via the JAX-WS library. 
Also, the VirtualBox-provided `vbojws.jar` must be in the class path.  In the SOAP case, it’s possible to create 
several `VirtualBoxManager` instances to communicate with multiple VirtualBox hosts.

Start your application like this:
 
```$ java -cp vboxjws.jar MyProgram```

Exception handling is also generalized by the generic glue layer, so that all methods could throw `VBoxException` 
containing human-readable text (see `getMessage()` method) along with the wrapped original exception (see 
`getWrapped()` method).

#### Example
This example shows a simple use case of the Java API. Differences for SOAP vs. local version are minimal, and limited 
to the connection setup phase (see the `ws` variable). In the SOAP case it’s possible to create several 
VirtualBoxManager instances to communicate with multiple VirtualBox hosts.

```java
import org.virtualbox_4_3.*;

// ...

VirtualBoxManager mgr = VirtualBoxManager.createInstance(null);
boolean ws = false; // set to true if we need the SOAP version

if (ws) {
    String url = "http://myhost:18034";
    String user = "test";
    String passwd = "test";
    mgr.connect(url, user, passwd);
}

IVirtualBox vbox = mgr.getVBox();
System.out.println("VirtualBox version: " + vbox.getVersion() + "\n");

// Get the name of the first VM
String m = vbox.getMachines().get(0).getName();
System.out.println("\nAttempting to start VM ’" + m + "’");

// Start the VM
mgr.startVm(m, null, 7000);

if (ws)
    mgr.disconnect();

mgr.cleanup();
```
For more a complete example, see `TestVBox.java`, shipped with the SDK. It contains exception handling and error 
printing code, which is important for reliable, larger scale projects. It is good practice in long-running API clients 
to process system events every now and then in the main thread (this does not work in other threads). As a rule of 
thumb it makes sense to process them every few 100ms to every few seconds). This is done by calling 
`mgr.waitForEvents(0);`.

This avoids the accumulation of a large number of system events, which can utilise a significant amount of memory - and 
as they also play a role in object clean-up, it helps freeing additional memory in a timely manner which is used by the 
API implementation itself. Java’s garbage collection approach already needs more memory due to the delayed freeing of 
memory used by no longer accessible objects, and not processing the system events exacerbates memory usage issues.

The `TestVBox.java` example code sprinkles such lines over the code to achieve the desired effect. In multi-threaded 
applications it can be called from the main thread periodically. Sometimes it’s possible to use the non-zero timeout 
variant of the method, which then waits the specified number of milliseconds for events, processing them immediately as 
they arrive. It achieves better runtime behavior than separate sleeping / processing.
