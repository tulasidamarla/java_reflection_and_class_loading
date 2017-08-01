# java_reflection_and_class_loading
-----------------------------------
Understanding Classpath
-----------------------
To understand classpath, lets write two sample programs. A helper class and a main class in different packages like this.

	package sample.helper;
	public class Helper {
		public String getMessage(){
			return "Message from Helper";
		}
	}

	package sample.main;
	import sample.helper.Helper;
	public class Main {
		public static void main(String[] args) {
			Helper helper = new Helper();
			System.out.println(helper.getMessage());
		}
	}	

If you want to compile these classes into any custom directory(like classes) from command line here are the steps:
1)javac -d classes -sourcepath src C:\Users\damart1\Documents\evaluation_test\sample\src\main\java\sample\helper\Helper.java
2)Set classpath(set CLASSPATH=%CLASSPATH%;/path/to/classes) to the classes directory otherwise Main class will not be compiled because it can't find Helper.class. 
3)javac -d classes -sourcepath src C:\Users\damart1\Documents\evaluation_test\sample\src\main\java\sample\main\Main.java

To run the main program, java sample.main.Main

Note: -d to specify where to place generated class files.
	  -sourcepath to specify where to find input source files 
*****
Note: Setting classpath using the above command(step 2) globally is not a good idea, because it only exists for the current terminal window. There is a better way to do it using 'cp' argument like this for step 3.

javac -cp /path/to/classes -d classes -sourcepath src C:\Users\damart1\Documents\evaluation_test\sample\src\main\java\sample\main\Main.java

To run the main program, java -cp classes sample.main.Main

Note: Using cp command we can set multiple directories, jar files to classpath. In windows multiple locations/jar files are seperated with semicolon, in unix they are seperated by colon.

Let's create a jar file using jar command.
1)cd classes
2)jar -cvf helper.jar /path/to/Helper.class
3)copy it some directory say lib
4)remove Helper.class file

Now, if we try to compile Main using step 3, we will get NoClassDefFoundError. we can compile Main using the below command.

javac -cp /path/to/classes;/path/to/lib/helper.jar -d classes -sourcepath src C:\Users\damart1\Documents\evaluation_test\sample\src\main\java\sample\main\Main.java

Basic concepts of class loading
-------------------------------
1)Core classes -- Classes present in java packages like java.lang,java.util etc.
2)Extension classes -- Classes that oracle want to ship with java that should be part of every application but are not part of the core classes. Also, classes that are shipped through third party that has to be part of every java application but are not necessiarily core classes, for example we often find cryptography classes as part of extension classes.
3)Delegation -- when a class loader first attempts to load a class it is typically delegated to what is known as parent class loader. Parent class loader is delegated to its parent class loader until some class loader can able to load.

Note: In the classpath example above, we have seen that classes are loaded from classpath. Apart from this, there are two other locations from where classes are loaded.
1)location of core classes, typically from /path/to/java/jdkorjre/lib/rt.jar(rt.jar is runtime jar which contains core java classes)
2)location of extension libaries, typically from /path/to/java/jdkorjre/lib/ext/

Note: Unlike application classes, the above two are not required to be passed in the classpath.

In the classpath example, to compile Main class we need to set Helper.class or helper.jar file path to classpath. If we copy the jar to ext directory, then we don't need to setup helper.jar with -cp argument. 

Note: ext directory is present in both jdk and jre directories. Be sure which one you are using.

***
Note: we can also set the ext directory explicitly using below command.
java -cp classes -Djava.ext.dirs=c:\lib sample.main.Main

Delegation
----------
There is a hierarchy in class loaders. A class loader may delegate to it's parent class loader.
Parent may load class. A class loader loads class only once. Once a class loader loads a class, it is cached. 

Note: If we load a basic java console application, it works with 3 class loaders.
1)Bootstrap: Written in C
2)Extension
3)Application

Application class loader asks to load a class to extension class loader. Extension class loader asks to load the class from bootstrap class loader.

Note: If bootstrap class loader finds the class in rt.jar it will load. Otherwise, it will pass that to extension class loader. Extension class loader tries to load the classes from ext directory jar files. If it can't find it is passed to Application class loader. Application class loader tries to load classes from classpath directories if it can, otherwise it will fail with "NoClassDefFoundError".

Lets see an example to see the Class loaders:

	import java.net.URL;
	import java.net.URLClassLoader;
	public class Delegation {
		public static void main(String[] args) {
			URLClassLoader loader = (URLClassLoader) ClassLoader.getSystemClassLoader();
			do{
				System.out.println(loader);
				for(URL url: loader.getURLs()){
					System.out.printf("\t %s \n", url.getPath());
				}
			}while((loader = (URLClassLoader) loader.getParent()) != null);
			System.out.println("Boot strap class loader");
		}
	}

	