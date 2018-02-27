java_reflection_and_class_loading
---------------------------------
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

Using URL classloader
---------------------
URL classloader is used to load classes. Let's make a jar out of the helper class explained in classpath section above. This program will demonstrate how to load that class using URLClassLoader.

	import java.net.MalformedURLException;
	import java.net.URL;
	import java.net.URLClassLoader;

	public class URLClassLoaderDemo {
		public static void main(String[] args) {
			URL url = null;
			try {
				url = new URL("file:///c:/lib/helper.jar");
				URLClassLoader ucl = new URLClassLoader(new URL[]{url});
				Class clazz=ucl.loadClass("sample.helper.Helper");
				Object obj=clazz.newInstance();
				System.out.println(obj.toString());
			} catch (MalformedURLException e) {
				e.printStackTrace();
			} catch (ClassNotFoundException e) {
				e.printStackTrace();
			} catch (InstantiationException e) {
				e.printStackTrace();
			} catch (IllegalAccessException e) {
				e.printStackTrace();
			}
		}
	}

*****	
Note: clazz.newInstance() is returning type of Object. We should not typecast that to sample.helper.Helper. If we use the actual type, it defeats the whole purpose of URLClassLoader because it uses application class loader to load the type.

*****
Note: The problem with the above approach of using our own class loader is, we cannot invoke any method present in the class that is loaded, because the return type is object. If we try invoke using cast it will be loaded using application class loader as mentioned above. so, the alternative to the above problem is to split the code into implementation and interface, so that Interface definitions go on the classpath and implementation is loaded through classpath. Let's see an example.

Let's write an interface.

	package sample.helper;
	public interface IHelper {
		public String getMessage();
	}

create a jar file from the above and name it as interfaces.jar.

Let's modify Helper class to implement IHelper. Now to compile Helper, we need interfaces.jar in the classpath. The command is,

javac -cp /path/to/interfaces.jar -d classes -sourcepath src C:\Users\damart1\Documents\evaluation_test\sample\src\main\java\sample\helper\Helper.java

Now re-create helper.jar with modified Helper.class.

Now modify the class loader program to cast the object to IHelper type.
	
	IHelper obj= (IHelper) clazz.newInstance();
	System.out.println(obj.getMessage());

To run the program,
java -cp /path/to/interfaces.jar sample.main.URLClassLoaderDemo

Writing custom class loader
---------------------------
When writing custom class loader few points need to be considered.  
1)custom classloader can't load all classes that application needs, it may delegate to the system classloader. (system classloader means to application, extension or bootstrap classloaders)  
2)If the system classloader can't load the class, then our custom classloader has to load the class bytes from the external location(either from DB or file system)

Let's write an example which will load bytes from a file system. Few key points to note when writing custom classloader.

1)Write a class that extends java.lang.ClassLoader and overwrite findClass() method.  
2)findClass() method should first give a chance to System classloader to load the class.  
3)If system classloader can't load the class, then load the class bytes from external location(file system, DB etc)  
4)once bytes are loaded, invoke defineClass() method present in java.lang.ClassLoader

Here is the code.

	package sample.main.classloaders;

	import java.io.File;
	import java.io.FileInputStream;
	import java.io.FileNotFoundException;
	import java.io.IOException;

	public class FileClassLoader extends ClassLoader {

		private ClassLoader parent;
		private String location;

		public FileClassLoader(String location) {
			this(ClassLoader.getSystemClassLoader(), location);
		}

		public FileClassLoader(ClassLoader parent, String location) {
			super(parent);
			this.parent = parent;
			this.location = location;
		}

		@Override
		public Class<?> findClass(String name) throws ClassNotFoundException {
			Class cls = null;
			try{
				cls = parent.loadClass(name);
			}catch (ClassNotFoundException cnfe) {
				byte[] classBytes= loadClassFromFileSystem(name);
				return defineClass(name, classBytes, 0, classBytes.length);
			}
			return cls;
		}

		private byte[] loadClassFromFileSystem(String location) {
			File file = new File(location);
			FileInputStream fis=null;
			byte[] classBytes = null;
			try {
				try {
					fis = new FileInputStream(file);
					classBytes = new byte[fis.available()];
					fis.read(classBytes);
				} finally {
					if(fis != null){
						fis.close();
					}
				}
			} catch (FileNotFoundException e) {
				e.printStackTrace();
			} catch (IOException e) {
				e.printStackTrace();
			}
			return classBytes;
		}

	}

Loading different versions of Same Class
----------------------------------------
Loading different versions of same class is also called side by side deployment.

Note: There may different reasons for doing side by side deployment. For example, your IDE may have different versions of XML Parsers. Each version may load different version of the same class.

Class loaders provide isolation. i.e. Two class loaders can load two different versions of the same class.

Class.forName()
---------------
Class.forName() loads the class by taking the name of the class as a parameter. It also has  additional parameter which takes a boolean value(whether to define the class or not means whether to verify if the class bytes are legal) and a thrid parameter which takes a classloader to load the class with.

Here is an example:

			URL url = new URL("file:///c:/lib/helper.jar");
			
			URLClassLoader ucl = new URLClassLoader(new URL[]{url});
			Class clazz= Class.forName("sample.helper.Helper",true,ucl);
			IHelper obj= (IHelper) clazz.newInstance();
			
			URL url2 = new URL("file:///c:/lib/helper.jar");
			URLClassLoader ucl2 = new URLClassLoader(new URL[]{url2});
			Class clazz2 = Class.forName("sample.helper.Helper",true,ucl2);
			IHelper obj2= (IHelper) clazz2.newInstance();

			System.out.printf("clazz == clazz2 %b\n", clazz == clazz2);
			System.out.printf("obj.class == obj2.class %b\n", obj.getClass() == obj2.getClass());

Note: As per the demo, The output is false, because classloaders are different, the Class references should be different. 

Implementing Factory Pattern using Java classloading
----------------------------------------------------
Factory pattern is a constructor pattern. Here is the sample code of Abstract factory pattern.

	public interface ICameraFactory {
		ICamera createCamera();
	}

	public interface ICamera {
		void takePhoto();
	}

	public class NikonCameraFactory implements ICameraFactory {
		public ICamera createCamera() {
			return new NikonCamera();
		}
	}	

	public class CanonCameraFactory implements ICameraFactory {
		public ICamera createCamera() {
			return new CanonCamera();
		}
	}

	public class NikonCamera implements ICamera {
		public void takePhoto() {
			System.out.println("Nikon photo taken");
		}
	}

	public class CanonCamera implements ICamera {
		public void takePhoto() {
			System.out.println("Canon photo taken");
		}
	}

	public class FactoryPatternDemo {
		public static void main(String[] args) {
			ICameraFactory factory = new NikonCameraFactory();
			ICamera camera = factory.createCamera();
			camera.takePhoto();
		}
	}
	
Note: The advantage of using Factory pattern is that we code against interfaces. one thing we can improve in the above program is that we have hardcoded the instantiation of the NikonCameraFactory. It should be configurable. Let's create a jar file from the above and name it as cameralib.jar. create a configuration file say config.json like below.

	{
		"factoryType" : "sample.factorypattern.demo.CanonCameraFactory",
		"location" : "file:///c:/lib/cameralib.jar"
	}

Write a Java class to read the configuration from a json file using jackson parser. Add the following dependency to your pom.xml

		<dependency>
			<groupId>com.fasterxml.jackson.core</groupId>
			<artifactId>jackson-databind</artifactId>
			<version>2.8.8</version>
		</dependency>
	
Here is the java code:

	import java.io.IOException;
	import java.nio.charset.StandardCharsets;
	import java.nio.file.FileSystems;
	import java.nio.file.Files;
	import java.nio.file.Path;
	import com.fasterxml.jackson.databind.ObjectMapper;

	public class Configuration {
		private String factoryType;
		private String location;
		
		public static Configuration loadConfiguration(String fileName) throws IOException{
			Path path = FileSystems.getDefault().getPath(fileName);
			String contents = new String(Files.readAllBytes(path), StandardCharsets.UTF_8);
			ObjectMapper mapper = new ObjectMapper();
			Configuration config = mapper.readValue(contents, Configuration.class);
			return config;
		}

		public String getFactoryType() {
			return factoryType;
		}

		public void setFactoryType(String factoryType) {
			this.factoryType = factoryType;
		}

		public String getLocation() {
			return location;
		}

		public void setLocation(String location) {
			this.location = location;
		}
	}

Write the main java class to test the factory pattern using configuration:

	try {
		Configuration config = Configuration.loadConfiguration("C:\\Users\\damart1\\Documents\\evaluation_test\\sample\\src\\main\\resources\\config.json");
		String location = config.getLocation();
		URL url = new URL(location);
		URLClassLoader classloader = new URLClassLoader(new URL[]{url});
		@SuppressWarnings("unchecked")
		Class<ICameraFactory> clazz=(Class<ICameraFactory>) Class.forName(config.getFactoryType(),true,classloader);
		ICameraFactory factory = clazz.newInstance();
		ICamera camera = factory.createCamera();
		camera.takePhoto();
		
	} catch (IOException e) {
		e.printStackTrace();
	} catch (ClassNotFoundException e) {
		e.printStackTrace();
	} catch (InstantiationException e) {
		e.printStackTrace();
	} catch (IllegalAccessException e) {
		e.printStackTrace();
	}

Note: The above code is highly configurable. i.e. If we make change in the factoryType property in json file and re run the above code, change will be reflected. So, no need to recompile the java code.

Hot Deployement
---------------
Classloaders give the ability to reload classes dynamically. Classloaders can also load new classes into the application without having to unload the application.

Let's write the server program:

	public interface IServer {
		public String getQuote();
	}
	
	public class ServerImpl implements IServer {
		public String getQuote() {
			return "When the going gets tough, the tough gets going, When the going gets rought, the tough gets rough";
		}
	}

Let's create a jar file with the above two and name it as server.jar.

Let's see the client program which loads the ServerImpl class from the jar using URLClassLoader.

	public class HotDeploymentDemoClient {
		
		private static ClassLoader cl;
		private static IServer server;
		
		public static void reloadServer() throws Exception{
			URL[] urls= new URL[]{new URL("file:///c:/lib/server.jar")};
			cl = new URLClassLoader(urls);
			server = (IServer) cl.loadClass("sample.hotdeployment.ServerImpl").newInstance();
		}

		public static void main(String[] args) throws Exception {
			BufferedReader reader = new BufferedReader(new InputStreamReader(System.in));
			reloadServer();
			while(true){
				System.out.println("Enter QUOTE, RELOAD or QUIT");
				String command = reader.readLine();
				if(command.toUpperCase().equals("QUIT")){
					return;
				}else if(command.toUpperCase().equals("QUOTE")){
					System.out.println(server.getQuote());
				} else if(command.toUpperCase().equals("RELOAD")){
					reloadServer();
				}
			}
		}
	}

Note: The above program takes user input to print Quote, or reload the ServerImpl class or to Quit. Print the quote first and then change the ServerImpl file and recompile and rejar and overwrite the jar into the same location. Now give reload command to the client program followed by quote to see your changes.

	
