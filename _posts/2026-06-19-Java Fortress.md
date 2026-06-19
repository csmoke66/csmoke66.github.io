---  
title: Java Fortress  
date: 2026-06-19 10:05:00 -0400  
categories: [Security]  
tags: [Java, Programming, Security]  
---  
> This article is undergoing active changes. Once it is complete, a repository will be hosted for the modified JVM I've made.
{: .prompt-danger }
> This article assumes Arch Linux usage. Adjust where required.
{: .prompt-warning }

We're going to explore turning a Java application into a highly secured fortress with minimal effort by using a modified JDK. This is a prime example of how effective client sided security can be achieved in an elegant way with minimal effort.

## Idea
----
What happens if we modify the underlying semantics that allows the JVM to function? This includes constant numbers (instruction opcodes, header data,) and in more extreme cases compiler output, and possibly vast changes to the file formats. By doing this, existing tooling will not work against the generated files (such as decompilers, or bytecode introspection libraries.)

## Setup
----
First we pull the package, and set up the source code:
  
```sh
git clone --recurse-submodules https://gitlab.archlinux.org/archlinux/packaging/packages/java-openjdk.git
cd java-openjdk
makepkg -o
```

Then we configure it, and test build:
```sh
cd src/the_jdk_src_folder
bash configure
make JOBS=$(nproc)
```
## Patching
----
I've spent almost 20 years working with Java, and JVMs, but I would be lying if I said I knew where to begin with this task. So our first step is locating what we're interested in. We're going to use the [JVM specifications](https://docs.oracle.com/javase/specs/jvms/se8/html/index.html) for locating what we need. The JVM specifications are crucial for understanding how things are done, and what we need to do to make analysis more difficult.
### .class Header
We need to locate 2 parts for every modification:
1. The encoding (compiler)
2. The decoding (JVM runtime, possibly JIT)
This is because in order for this to work, the compiler needs to generate our proprietary data, and the JVM needs to be able to work with it.  
  
Looking at the [specifications](https://docs.oracle.com/javase/specs/jvms/se8/html/jvms-4.html), we can see `0xCAFEBABE` are the first 4 bytes of every .class file. We're going to make an assumption they don't access the individual bytes. Searching for the magic number used for identifying .class files, we find quite a few results:
```sh
❯ grep -rn "CAFEBABE" .  
./hotspot/cpu/x86/vm_version_x86.hpp:744:  static int ymm_test_value()    { return 0xCAFEBABE; }  
./hotspot/cpu/x86/vm_version_x86.hpp:745:  static jlong egpr_test_value()   { return 0xCAFEBABECAFEBABELL; }  
./hotspot/share/classfile/classFileParser.cpp:100:#define JAVA_CLASSFILE_MAGIC              0xCAFEBABE  
./hotspot/share/prims/jvmtiClassFileReconstituter.cpp:901:  write_u4(0xCAFEBABE);  
./java.base/share/classes/java/lang/Integer.java:1676:     * compress(0xCAFEBABE, 0xFF00FFF0) == 0xCABAB  
./java.base/share/classes/java/lang/Integer.java:1680:     * at positions 1, 2, 3, 6 and 7 of {@code 0xCAFEBABE}. The selected digits  
./java.base/share/classes/java/lang/Integer.java:1719:     * sag(0xCAFEBABE, 0xFF00FFF0) == 0xCABABFEE  
./java.base/share/classes/java/lang/Long.java:1678:     * compress(0xCAFEBABEL, 0xFF00FFF0L) == 0xCABABL  
./java.base/share/classes/java/lang/Long.java:1682:     * at positions 1, 2, 3, 6 and 7 of {@code 0xCAFEBABE}. The selected digits  
./java.base/share/classes/java/lang/Long.java:1721:     * sag(0x00000000_CAFEBABEL, 0xFFFFFFFF_FF00FFF0L) == 0x00000000_CABABFEEL  
./java.base/share/classes/java/lang/classfile/ClassFile.java:785:    int MAGIC_NUMBER = 0xCAFEBABE;  
./java.base/share/classes/java/nio/X-Buffer.java.template:243: *     bb.putInt(0xCAFEBABE);  
./java.base/share/classes/java/nio/X-Buffer.java.template:251: *     bb.putInt(0xCAFEBABE).putShort(3).putShort(45);  
./java.base/share/classes/jdk/internal/classfile/impl/ClassReaderImpl.java:73:        if (classfileLength < 4 || readInt(0) != 0xCAFEBABE) {  
./java.base/share/classes/jdk/internal/module/ModuleInfo.java:188:        if (magic != 0xCAFEBABE)  
./java.xml/share/classes/com/sun/org/apache/bcel/internal/Const.java:39:    public static final int JVM_CLASSFILE_MAGIC = 0xCAFEBABE;  
./jdk.compiler/share/classes/com/sun/tools/javac/jvm/ClassFile.java:67:    public static final int JAVA_MAGIC = 0xCAFEBABE;  
./jdk.hotspot.agent/share/classes/sun/jvm/hotspot/tools/jcore/ClassWriter.java:88:        dos.writeInt(0xCAFEBABE);
```
After playing around with a few different ways of accomplishing with we want, I landed on the most viable way: create a secondary decoding/encoding pass while leaving the first intact. This is because OpenJDK bootstraps itself, and making outright modifications to the format is not feasible.

After creating secondary paths for all the decoding/encoding results found above, we compile a simple hello world Java application. We toggle our custom format encoding with the JVM argument `jdk.experimentalWrite`. There's 2 decoding/encoding paths, and decoding does not need a switch. It decides what path to take based on the new magic number.
```java
import java.lang.System;

public class HelloWorld {
    public static void main(String[] args) {
        System.out.println("Hello, World!");
    }
}
```
```sh
❯ /home/x/Documents/code/java-openjdk/src/jdk26u-jdk-26.0.1-8/build/linux-x86_64-server-release/jdk/bin/javac -J-Djdk.experimentalWrite=true HelloWorld.java
```
We can see that the magic number has been modified:
![modified_class_format_id](assets/modified_class_format_id.png)
And we can run this modified class file using our patched JVM:
```sh
❯ /home/x/Documents/code/java-openjdk/src/jdk26u-jdk-26.0.1-8/build/linux-x86_64-server-release/jdk/bin/java HelloWorld  
Hello, World!
```
### Onwards
Now that we've made a custom file header, and we have split the decoding/encoding paths into 2, we have full reign over the entire file format. We can change the order of data. We can obfuscate the data using simple mathematical operations. We can do anything.

> This section is incomplete for now.
{: .prompt-danger }

### Result
----
The result of all of this is that our binary is completely unusable by existing reverse engineering tools. The difficulty of someone recovering the original data back depends on how much time you put in, and how creative you get.
![failed_decompiler](assets/failed_decompiler.png)

## Second Layer Protection
----
So you've done all your modifications to your JVM. Existing tooling no longer works and without reverse engineering your JVM they cannot inspect things easily. But we want to make them work for that. I recommend applying secondary level protection such as Themida, Code Virtualizer, or a custom VM within your custom decoding/encoding paths. This will ensure that 99.9% of people will not have the skills to inspect your class files. And that remaining 0.1% probably don't want to deal with the headache.

## Acknowledgements
----
The developers of [Stalcraft](https://store.steampowered.com/app/1818450/STALCRAFT_X/), for making me go through the hell that was reversing their implementation of this.