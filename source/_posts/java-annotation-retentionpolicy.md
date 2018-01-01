title: java Annotation的RetentionPolicy介绍
date: 2016-11-04 00:01:06
tags:
- java
categories:
- java

---------------

Java Annotation对应的Retention有3种，在RetentionPolicy中定义，有3种：


1. SOURCE. 注解保留在源代码中，但是编译的时候会被编译器所丢弃。比如@Override, @SuppressWarnings
2. CLASS. 这是默认的policy。注解会被保留在class文件中，但是在运行时期间就不会识别这个注解。
3. RUNTIME. 注解会被保留在class文件中，同时运行时期间也会被识别。所以可以使用反射机制获取注解信息。比如@Deprecated


<!--more-->

## RUNTIME

大部分情况下，我们都是使用RUNTIME这个Policy。

下面就是一个RUNTIME Annotation的例子。

先定义Annotation：

	@Retention(RetentionPolicy.RUNTIME)
	@Target(ElementType.TYPE)
	public @interface MyClassRuntimeAnno {
    	String name();
    	int level() default 1;
	}
	
	
然后在CLASS前面使用这个Annotation。

	@MyClassRuntimeAnno(name = "simple", level = 10)
	public class SimpleObj {
	}
	
最后写一个testcase通过反射可以获取这个类的Annotation进行后续操作。

	@Test
    public void testGetAnnotation() {
        Annotation[] annotations = SimpleObj.class.getAnnotations();
        System.out.println(Arrays.toString(annotations));
        MyClassRuntimeAnno myClassAnno = SimpleObj.class.getAnnotation(MyClassRuntimeAnno.class);
        System.out.println(myClassAnno.name() + ", " + myClassAnno.level());
        System.out.println(myClassAnno == annotations[0]);
    }
    
## SOURCE

SOURCE这个policy表示注解保留在源代码中，但是编译的时候会被编译器所丢弃。 由于在编译的过程中这个注解还被保留着，所以在编译过程中可以针对这个policy进行一些操作。比如在自动生成java代码的场景下使用。最常见的就是[lombok](https://projectlombok.org/)的使用了，可以自动生成field的get和set方法以及toString方法，构造器等；消除了冗长的java代码。


SOURCE这个policy可以使用jdk中的javax.annotation.processing.*包中的processor处理器进行注解的处理过程。



以1个编译过程中会打印类中的方法的例子来说明SOUCRE这个policy的作用：



首先定义一个Printer注解：

	@Retention(RetentionPolicy.SOURCE)
	@Target(ElementType.METHOD)
	public @interface Printer {
	}


然后一个类的方法使用这个注解：

	public class SimpleObject {

    	@Printer
    	public void methodA() {

    	}
    	
    	public void methodB() {

    	}

	}

创建对应的Processor：

	@SupportedAnnotationTypes({"me.format.annotaion.Printer"})
    public class PrintProcessor extends AbstractProcessor {
        @Override
        public boolean process(Set<? extends TypeElement> annotations, RoundEnvironment roundEnv) {
            Messager messager = processingEnv.getMessager();
            messager.printMessage(Diagnostic.Kind.NOTE, "start to use PrintProcessor ..");
  
    
            Set<? extends Element> rootElements = roundEnv.getRootElements();
            messager.printMessage(Diagnostic.Kind.NOTE, "root classes: ");
            for(Element root : rootElements) {
                messager.printMessage(Diagnostic.Kind.NOTE, ">> " + root.toString());
            }
            messager.printMessage(Diagnostic.Kind.NOTE, "annotation: ");
            for(TypeElement te : annotations) {
                messager.printMessage(Diagnostic.Kind.NOTE, ">>> " + te.toString());
                Set<? extends Element> elements = roundEnv.getElementsAnnotatedWith(te);
                for(Element ele : elements) {
                    messager.printMessage(Diagnostic.Kind.NOTE, ">>>> " + ele.toString());
                }
            }
    
            return true;
        }
        @Override
        public SourceVersion getSupportedSourceVersion() {
            return SourceVersion.latestSupported();
        }
    }

然后先使用javac编译Printer和PrintProcessor：

	javac -d classes src/main/java/me/format/annotation/Printer.java src/main/java/me/format/annotation/PrintProcessor.java

最后再使用javac中的processor参数处理：

	javac -cp classes -processor me.format.annotation.PrintProcessor -d classes src/main/java/me/format/annotation/SimpleObject.java


控制台打印出：

	注: start to use PrintProcessor ..
	注: root classes: 
	注: >> hello.annotation.SimpleObject
	注: annotation: 
	注: >>> hello.annotation.Printer
	注: >>>> methodA()


## CLASS

CLASS和RUNTIME的唯一区别是RUNTIME在运行时期间注解是存在的，而CLASS则不存在。

我们通过[asm](http://asm.ow2.org/)来获取class文件里的annotation。


首先定义注解：

policy为CLASS的注解。

	@Retention(RetentionPolicy.CLASS)
    @Target(ElementType.TYPE)
    public @interface Meta {
    
        String name();
    
    }

policy为RUNTIME的注解。

	@Retention(RetentionPolicy.RUNTIME)
    @Target(ElementType.TYPE)
    public @interface Header {
    
        int code();
    
    }

使用注解：

	@Meta(name = "obj")
    @Header(code = 200)
    public class AnnotationObject {
    
        private String val;
    
        public String getVal() {
            return val;
        }
    
        public void setVal(String val) {
            this.val = val;
        }
    }


编译这3个java文件得到字节码文件AnnotationObject.class：

	javac -d classes src/main/java/me/format/annotaion/AnnotationObject.java src/main/java/me/format/annotation/Meta.java src/main/java/me/format/annotation/Header.java


使用asm获取字节码文件中的注解：

	ClassNode classNode = new ClassNode();

    ClassReader cr = new ClassReader(new FileInputStream("classes/me/format/annotation/AnnotationObject.class"));

    cr.accept(classNode, 0);

    System.out.println("Class Name: " + classNode.name);
    System.out.println("Source File: " + classNode.sourceFile);

    System.out.println("invisible: ");
    AnnotationNode anNode = null;
    for (Object annotation : classNode.invisibleAnnotations) {
        anNode = (AnnotationNode) annotation;
        System.out.println("Annotation Descriptor : " + anNode.desc);
        System.out.println("Annotation attribute pairs : " + anNode.values);
    }

    System.out.println("visible: ");
    for (Object annotation : classNode.visibleAnnotations) {
        anNode = (AnnotationNode) annotation;
        System.out.println("Annotation Descriptor : " + anNode.desc);
        System.out.println("Annotation attribute pairs : " + anNode.values);
    }

打印出：

	Class Name: me/format/annotation/AnnotationObject
	Source File: AnnotationObject.java
	invisible: 
	Annotation Descriptor : Lme/format/annotation/Meta;
	Annotation attribute pairs : [name, obj]
	visible: 
	Annotation Descriptor : Lme/format/annotation/Header;
	Annotation attribute pairs : [code, 200]


其中policy为CLASS的注解编译完后不可见，而policy为RUNTIME的注解编译后可见。


同样，我们可以使用javap查看编译后的信息：

	javap -v me.format.annotation.AnnotationObject

会打印出注解的visible信息：
	
	#16 = Utf8               AnnotationObject.java
	#17 = Utf8               RuntimeVisibleAnnotations
	#18 = Utf8               Lhello/annotation/Header;
	#19 = Utf8               code
	#20 = Integer            200
	#21 = Utf8               RuntimeInvisibleAnnotations
	#22 = Utf8               Lhello/annotation/Meta;
	#23 = Utf8               name
	#24 = Utf8               obj

