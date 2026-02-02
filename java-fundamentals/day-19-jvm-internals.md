# Day 19: JVM Internals

## Mục tiêu
- JVM Architecture
- Class loading
- Bytecode
- JIT Compilation

---

## 1. JVM Architecture

```
┌─────────────────────────────────────────────────────────┐
│                    Java Application                      │
├─────────────────────────────────────────────────────────┤
│                    JVM (Java Virtual Machine)            │
│  ┌─────────────────────────────────────────────────┐    │
│  │              Class Loader Subsystem              │    │
│  │   Bootstrap → Extension → Application Loader    │    │
│  └─────────────────────────────────────────────────┘    │
│  ┌─────────────────────────────────────────────────┐    │
│  │                  Runtime Data Areas              │    │
│  │   Method Area │ Heap │ Stack │ PC │ Native Stack│    │
│  └─────────────────────────────────────────────────┘    │
│  ┌─────────────────────────────────────────────────┐    │
│  │                 Execution Engine                 │    │
│  │   Interpreter │ JIT Compiler │ Garbage Collector│    │
│  └─────────────────────────────────────────────────┘    │
│  ┌─────────────────────────────────────────────────┐    │
│  │              Native Method Interface             │    │
│  └─────────────────────────────────────────────────┘    │
├─────────────────────────────────────────────────────────┤
│                    Native Libraries                      │
└─────────────────────────────────────────────────────────┘
```

---

## 2. Class Loading

### 2.1. ClassLoader Hierarchy

```
Bootstrap ClassLoader (rt.jar, core Java)
       ↓
Extension ClassLoader (ext/*.jar)
       ↓
Application ClassLoader (classpath)
       ↓
Custom ClassLoaders
```

### 2.2. Loading Process

```
1. Loading    - Find and load .class file
2. Linking
   - Verification  - Check bytecode validity
   - Preparation   - Allocate memory for static vars
   - Resolution    - Resolve symbolic references
3. Initialization - Execute static initializers
```

### 2.3. Custom ClassLoader

```java
public class CustomClassLoader extends ClassLoader {
    @Override
    protected Class<?> findClass(String name) throws ClassNotFoundException {
        byte[] classData = loadClassData(name);
        if (classData == null) {
            throw new ClassNotFoundException(name);
        }
        return defineClass(name, classData, 0, classData.length);
    }

    private byte[] loadClassData(String name) {
        String path = name.replace('.', '/') + ".class";
        try (InputStream is = getResourceAsStream(path)) {
            return is.readAllBytes();
        } catch (IOException e) {
            return null;
        }
    }
}
```

### 2.4. Class Loading Info

```java
Class<?> clazz = String.class;

// Get classloader
ClassLoader loader = clazz.getClassLoader();
// null for bootstrap classes (core Java)

// Get parent loader
ClassLoader parent = loader.getParent();

// Load class dynamically
Class<?> dynamicClass = Class.forName("com.example.MyClass");

// With specific classloader
Class<?> loaded = loader.loadClass("com.example.MyClass");
```

---

## 3. Bytecode

### 3.1. Viewing Bytecode

```bash
# Compile
javac HelloWorld.java

# View bytecode
javap -c HelloWorld.class

# Verbose output
javap -v HelloWorld.class
```

### 3.2. Simple Example

```java
public class Simple {
    public int add(int a, int b) {
        return a + b;
    }
}
```

```
Bytecode:
  public int add(int, int);
    Code:
       0: iload_1      // Load int from local variable 1 (a)
       1: iload_2      // Load int from local variable 2 (b)
       2: iadd         // Add two ints
       3: ireturn      // Return int
```

### 3.3. Common Bytecode Instructions

```
// Load/Store
iload_0   - Load int from local var 0
aload_0   - Load reference from local var 0
iconst_1  - Push int constant 1
bipush 10 - Push byte 10 as int
ldc "str" - Push constant from pool

// Arithmetic
iadd      - Integer add
isub      - Integer subtract
imul      - Integer multiply
idiv      - Integer divide

// Objects
new       - Create object
dup       - Duplicate top stack value
invokespecial - Call constructor/private method
invokevirtual - Call instance method
invokestatic  - Call static method

// Control
if_icmpge - If int compare >=
goto      - Unconditional jump
return    - Return void
ireturn   - Return int
areturn   - Return reference
```

---

## 4. JIT Compilation

### 4.1. How JIT Works

```
Source Code → Bytecode → Interpreter (slow start)
                      ↓
              Hot Code Detection
                      ↓
              JIT Compilation → Native Code (fast)
```

### 4.2. Compilation Tiers

```
Tier 0 - Interpreted
Tier 1 - C1 compiled (simple optimizations)
Tier 2 - C1 compiled with profiling
Tier 3 - C1 compiled with full profiling
Tier 4 - C2 compiled (aggressive optimizations)
```

### 4.3. JIT Options

```bash
# Print compilation
-XX:+PrintCompilation

# Disable JIT
-Djava.compiler=NONE

# Compilation thresholds
-XX:CompileThreshold=10000

# AOT (Ahead of Time) - Java 9+
jaotc --output lib.so --class-name MyClass
```

---

## 5. Method Dispatch

```java
// Static dispatch (compile time)
public static void print(String s) { }
public static void print(Integer i) { }

String s = "hello";
print(s);  // Resolved at compile time

// Dynamic dispatch (runtime)
Animal animal = new Dog();
animal.speak();  // Resolved at runtime via vtable
```

---

## 6. String Pool

```java
// Interning
String s1 = "Hello";           // Pool
String s2 = "Hello";           // Same pool reference
String s3 = new String("Hello"); // Heap
String s4 = s3.intern();       // Pool

s1 == s2;  // true
s1 == s3;  // false
s1 == s4;  // true

// Pool location
// Java 7+: Heap (not PermGen)
// Allows dynamic resizing and GC
```

---

## 7. Runtime Data Areas

```java
// Method Area (Metaspace)
// - Class structures
// - Field/method data
// - Runtime constant pool

// Heap
// - Object instances
// - Arrays

// Stack (per thread)
// - Stack frames per method call
// - Local variables
// - Operand stack
// - Frame data

// PC Register (per thread)
// - Address of current instruction

// Native Method Stack
// - For native (C/C++) methods
```

---

## 8. Diagnostic Tools

```bash
# JDK Tools
java -XX:+PrintFlagsFinal -version  # All JVM flags
java -XX:+PrintCommandLineFlags     # Active flags

# Flight Recorder
java -XX:StartFlightRecording=duration=60s,filename=recording.jfr

# Native Memory Tracking
java -XX:NativeMemoryTracking=summary
jcmd <pid> VM.native_memory summary

# Class histogram
jmap -histo <pid>

# Thread dump
jstack <pid>

# GC info
jstat -gc <pid> 1000

# JConsole, VisualVM, JMC for GUI monitoring
```

---

## 9. Performance Tips

```java
// 1. Warm-up JIT before benchmarking
for (int i = 0; i < 10000; i++) {
    methodToBenchmark();
}
long start = System.nanoTime();
// Actual measurement

// 2. Avoid reflection in hot paths
// Reflection is slower than direct calls

// 3. Be aware of escape analysis
// Objects that don't escape may be stack-allocated

// 4. Use appropriate collections
// ArrayList vs LinkedList based on access patterns

// 5. Minimize synchronization
// Use concurrent collections when possible

// 6. Profile before optimizing
// Use JMH for microbenchmarks
```

---

## 10. Summary

```
┌─────────────────────────────────────────────────────────┐
│                     JVM Key Concepts                     │
├─────────────────────────────────────────────────────────┤
│  Class Loading: Bootstrap → Extension → Application     │
│  Memory: Heap (objects) + Stack (method calls)          │
│  Execution: Interpreter → JIT → Native Code             │
│  GC: Young Gen (frequent) → Old Gen (major)             │
│  Bytecode: Platform-independent intermediate code       │
│  JIT: Hot code compiled to native for performance       │
└─────────────────────────────────────────────────────────┘
```

---

## Bài tập thực hành

### Bài 1: ClassLoader Explorer
List all loaded classes và classloaders.

### Bài 2: Bytecode Analyzer
Đọc và explain bytecode của simple methods.

### Bài 3: JIT Warmup
Demonstrate JIT warmup effect với benchmarks.

---

## Tổng kết khóa học

Chúc mừng bạn đã hoàn thành 19 ngày học Java Fundamentals!

### Recap

| Phase | Days | Topics |
|-------|------|--------|
| Basic | 1-7 | Syntax, OOP, Collections, Exceptions |
| Intermediate | 8-13 | Generics, Lambda, Streams, I/O, DateTime, Threading |
| Advanced | 14-19 | Concurrency, Async, Reflection, Patterns, JVM |

### Next Steps
- Spring Framework
- Testing (JUnit, Mockito)
- Build tools (Maven, Gradle)
- Web development (Spring Boot)
- Microservices

---

## Navigation

- [← Day 18: Memory & GC](./day-18-memory-gc.md)
- [Overview →](./00-overview.md)
