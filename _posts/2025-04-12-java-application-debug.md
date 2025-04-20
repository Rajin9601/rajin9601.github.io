---
layout: post
title:  "Java Application 디버깅 이야기"
date:   2025-04-12 22:00:00 +0900
categories: dev
img-overlay: 0.1
comments: true
draft: true
---

# 상황 설명

어느 날부터 특정 서비스를 배포할 때 간헐적으로 Pod가 Ready 상태가 되지 않는 현상이 발생하기 시작했습니다. 구체적인 현상은 다음과 같았습니다.

- 간헐적(약 10% 확률)으로 Pod가 실행될 때 Ready 상태가 되지 않음
- Pod를 재시작하면 정상적으로 동작할 때도 있음
- 로그를 보면 JVM 시작 로그는 출력되나 Application 시작 로그는 보이지 않음
- 해당 현상은 두 개의 서비스에서만 재현됨

Application 시작 로그가 출력되지 않지만, Pod가 Crash 나는 것이 아니라 Ready 상태가 되지 않은 채 유지되고 있는 것으로 보아, 데드락이 의심되어 먼저 Stacktrace를 확인하였습니다.

# Java Stacktrace 확인

문제가 발생한 Pod에 접속하여 `jstack` 명령어를 통해 현재 실행 중인 Java 프로그램의 Stacktrace를 확인하였습니다. (해당 프로그램은 RajinApplication이라고 하겠습니다.)

```shell
jstack -l -e <pid>

....
"main" #1 [12] prio=5 os_prio=0 cpu=4154.15ms elapsed=160.23s allocated=156M defined_classes=4965 tid=0x00007f5ffdf0f060 nid=12 waiting on condition  [0x00007f5ffdef8000]
   java.lang.Thread.State: RUNNABLE
        at reactor.tools.agent.ReactorDebugAgent$1.transform(ReactorDebugAgent.java:97)
        at java.lang.instrument.ClassFileTransformer.transform(java.instrument@21.0.3/ClassFileTransformer.java:244)
        at sun.instrument.TransformerManager.transform(java.instrument@21.0.3/TransformerManager.java:188)
        at sun.instrument.InstrumentationImpl.transform(java.instrument@21.0.3/InstrumentationImpl.java:610)
        at java.lang.ClassLoader.defineClass1(java.base@21.0.3/Native Method)
        at java.lang.ClassLoader.defineClass(java.base@21.0.3/ClassLoader.java:1027)
		....
        at java.net.URLClassLoader.findClass(java.base@21.0.3/URLClassLoader.java:420)
        at java.lang.ClassLoader.loadClass(java.base@21.0.3/ClassLoader.java:593)
        - locked <0x00000000d2000000> (a org.springframework.boot.loader.launch.LaunchedClassLoader)
        at org.springframework.boot.loader.net.protocol.jar.JarUrlClassLoader.loadClass(JarUrlClassLoader.java:104)
        at org.springframework.boot.loader.launch.LaunchedClassLoader.loadClass(LaunchedClassLoader.java:91)
        at java.lang.ClassLoader.loadClass(java.base@21.0.3/ClassLoader.java:526)
        at .........RajinApplicationKt.main(....RajinApplication.kt:17)
		....
        at org.springframework.boot.loader.launch.Launcher.launch(Launcher.java:91)
        at org.springframework.boot.loader.launch.Launcher.launch(Launcher.java:53)
        at org.springframework.boot.loader.launch.JarLauncher.main(JarLauncher.java:58)
...
"dd-task-scheduler" #19 [35] daemon prio=5 os_prio=0 cpu=298.33ms elapsed=158.32s allocated=7532K defined_classes=32 tid=0x00007f5f9bd89f60 nid=35 waiting on condition  [0x00007f5f9bb4a000]
   java.lang.Thread.State: RUNNABLE
        at reactor.tools.agent.ReactorDebugAgent$1.transform(ReactorDebugAgent.java:97)
		...
(dd-trace-processor 등등 datadog 관련 thread 여러개가 똑같은 stacktrace 에서 waiting 중이였음)
...
```

보시다시피 main 스레드가 ReactorDebugAgent에서 이상한 지점에서 Block되고 있었습니다. 해당 코드에서 명시적으로 락을 잡는 부분은 보이지 않았습니다.

```java
// ReactorDebugAgent의 97번 줄
ClassReader cr = new ClassReader(bytes);
ClassWriter cw = new ClassWriter(cr, ClassWriter.COMPUTE_MAXS);
```

데드락은 보통 명시적인 락 또는 synchronized 블록에서 발생한다고 생각했기에, 특수한 상황이라 판단하였습니다. 특히 `Thread.State`가 `RUNNABLE`인데, `waiting on condition`이라는 점이 이상하게 느껴졌습니다.

# Java Remote Debugging

먼저 로컬 환경에서 문제를 재현해보고자 IntelliJ와 같은 IDE를 활용하여 여러 정보를 확인하려 하였으나, 로컬에서는 재현이 어려웠습니다.  
이는 로컬 환경 설정에서 Datadog가 실행되지 않도록 구성되어 있었기 때문으로 보이며, 해당 프로그램을 로컬에서 제대로 실행시키는 것 자체가 쉽지 않았습니다.

이에 따라 Pod에 설정을 추가하여 원격 디버깅이 가능하도록 구성하였고, IntelliJ를 통해 현재 스택 트레이스 상에서 로컬 변수와 함수 인자에 어떤 값이 들어있는지를 확인할 수 있었습니다.  
그 결과, 각 스레드의 상태는 다음과 같았습니다.

- `main` 스레드는 `RajinApplication` 클래스를 로딩 중이며, 그 과정에서 `ReactorDebugAgent`에 등록된 `ClassFileTransformer`가 실행되다가 블로킹되어 있습니다.
- `dd-`로 시작하는 Datadog 관련 스레드들 역시 프로그램을 실행하는 도중, 특정 클래스를 로딩하는 과정에서 `ReactorDebugAgent`의 `ClassFileTransformer`가 실행되며 블로킹된 상태입니다.

# Stacktrace 기반 조사

ReactorDebugAgent 내의 Transformer 코드에서 여러 스레드가 Block되고 있었기 때문에 [해당 코드](https://github.com/reactor/reactor-core/blob/486152e0a1103caf9dd6ba50e7957c16b8dd268b/reactor-tools/src/main/java/reactor/tools/agent/ReactorDebugAgent.java#L97)의 내용을 읽었습니다만, ReactorDebugAgent 는 리액터 코드의 디버깅을 돕기 위해서, ClassFileTransformer 를 등록하는 내용 정도로만 보이고 문제가 될 만한 코드가 보이진 않았습니다.

그래서 Deadlock이 클래스 초기화 중에도 발생할 수 있는지를 조사하던 중 [이 글](https://www.farside.org.uk/201510/deadlocks_in_java_class_initialisation)을 발견하였습니다. 핵심은 다음과 같습니다:

> 클래스 초기화 중에 Deadlock이 걸릴 경우, 스레드 상태는 `waiting`임에도 불구하고 `RUNNABLE`로 표시될 수 있다.

```shell
"main" prio=10 tid=0x00007efd5000a000 nid=0x51ca in Object.wait() [0x00007efd59d45000]
   java.lang.Thread.State: RUNNABLE
```

현재 겪고 있는 문제와는 조금 다르긴 하지만 중요한 건, class initialization 도중에 데드락이 걸리면 waiting 상태이면서 Thread.State 는 Runnable 이라는 정보를 알게 되었습니다.
class initialization 관련 로그를 남겨야 겠다고 생각했고 [jvm arguments 로 남길수 있다는걸](https://bugs.openjdk.org/browse/JDK-8316229) 알게되어 `-Xlog:class+init=debug` 으로 로그를 보았습니다.

# JVM Log

```log
[debug][class,init] Thread "dd-trace-processor" linking reactor.tools.shaded.net.bytebuddy.jar.asm.ClassWriter
[info ][class,init] Start class verification for: reactor.tools.shaded.net.bytebuddy.jar.asm.ClassWriter
[debug][class,init] Thread "main" found reactor.tools.shaded.net.bytebuddy.jar.asm.ClassVisitor already linked
[debug][class,init] Thread "main" waiting for linking of reactor.tools.shaded.net.bytebuddy.jar.asm.ClassWriter by thread "dd-trace-processor"
// 더이상 main thread 의 내용은 출력되지 않는다.
...
[debug][class,init] Thread "dd-task-scheduler" waiting for linking of reactor.tools.shaded.net.bytebuddy.jar.asm.ClassWriter by thread "dd-trace-processor"
[debug][class,init] Thread "dd-telemetry" waiting for linking of reactor.tools.shaded.net.bytebuddy.jar.asm.ClassWriter by thread "dd-trace-processor"
// 더이상 dd-* thread 의 내용은 출력되지 않는다.
```

로그를 해석하면 다음과 같습니다.
- dd-trace-processor는 ClassWriter의 클래스 검증(class verification)을 시작했는데, end class verification 로그가 출력되지 않는 것으로 보아 ClassWriter의 클래스 검증이 무한 로딩 중인 것으로 추정됩니다.
- main, dd-task-scheduler 등 여러 스레드는 dd-trace-processor가 수행 중인 ClassWriter의 링크(linking)를 기다리고 있습니다.

결론적으로는 dd-trace-processor의 작업을 main이 기다리고 있으나, dd-trace-processor의 작업은 끝나지 않고 있으며 main보다 먼저 멈춰 있는 상태입니다.
클래스 검증이나 링크 과정을 기다리는 중인 작업은 JVM 내부에서 실행되고 있기 때문에, 그 내부에서 어떤 일이 발생하고 있기에 무한 로딩 중인지는 JVM 위에서 동작하는 jstack과 같은 도구로는 확인할 수 없습니다.
따라서 더 깊이, 현재 무슨 일이 벌어지고 있는지를 보기 위해서는 JVM 내부를 직접 살펴보아야 합니다.

# JVM 내부 보기 (gdb)

`gdb`를 사용하면 JVM 내부까지 확인할 수 있습니다. 단순한 스택 트레이스(stack trace)를 넘어 메모리의 내용까지 보고 싶다면, 심볼 테이블(symbol table)이 보존되도록 JDK를 직접 빌드하여 실행하지 않는 이상 확인이 어렵다고 판단하여, 우선 스택 트레이스만 추출하였습니다.

dd-trace-processor thread 의 stacktrace
```log
#0  __cp_end () at src/thread/x86_64/syscall_cp.s:29
#1  0x00007fc769521617 in __syscall_cp_c (nr=202, u=<optimized out>, v=<optimized out>, w=<optimized out>, x=<optimized out>, y=<optimized out>, z=0)
    at src/thread/pthread_cancel.c:33
#2  0x00007fc769520b49 in __futex4_cp (to=<optimized out>, val=2, op=128, addr=0x7fc701cfb7a4) at src/thread/__timedwait.c:24
#3  __timedwait_cp (addr=addr@entry=0x7fc701cfb7a4, val=val@entry=2, clk=clk@entry=1, at=at@entry=0x7fc701cfb7f0, priv=128, priv@entry=1) at src/thread/__timedwait.c:52
#4  0x00007fc7695219af in __pthread_cond_timedwait (c=0x7fc703d9f150, m=0x7fc703d9f128, ts=0x7fc701cfb7f0) at src/thread/pthread_cond_timedwait.c:100
#5  0x00007fc768c5627f in PlatformEvent::park_nanos(long) () from /opt/java/openjdk/lib/server/libjvm.so
#6  0x00007fc768c242c4 in ObjectMonitor::EnterI(JavaThread*) () from /opt/java/openjdk/lib/server/libjvm.so
#7  0x00007fc768c24c58 in ObjectMonitor::enter(JavaThread*) () from /opt/java/openjdk/lib/server/libjvm.so
#8  0x00007fc768e7434c in ObjectSynchronizer::enter(Handle, BasicLock*, JavaThread*) () from /opt/java/openjdk/lib/server/libjvm.so
#9  0x00007fc768e7d8e1 in SystemDictionary::resolve_instance_class_or_null(Symbol*, Handle, Handle, JavaThread*) () from /opt/java/openjdk/lib/server/libjvm.so
#10 0x00007fc768e7f56e in SystemDictionary::resolve_or_fail(Symbol*, Handle, Handle, bool, JavaThread*) () from /opt/java/openjdk/lib/server/libjvm.so
#11 0x00007fc76893142d in find_class_from_class_loader(JNIEnv_*, Symbol*, unsigned char, Handle, Handle, unsigned char, JavaThread*) () from /opt/java/openjdk/lib/server/libjvm.so
#12 0x00007fc768939a13 in JVM_FindClassFromClass () from /opt/java/openjdk/lib/server/libjvm.so
#13 0x00007fc7069a0d56 in load_class_global () from /opt/java/openjdk/lib/libverify.so
#14 0x00007fc7069a1abf in merge_fullinfo_types () from /opt/java/openjdk/lib/libverify.so
#15 0x00007fc7069a274b in pop_stack () from /opt/java/openjdk/lib/libverify.so
#16 0x00007fc7069a4b96 in VerifyClassForMajorVersion () from /opt/java/openjdk/lib/libverify.so
#17 0x00007fc768f26b37 in Verifier::inference_verify(InstanceKlass*, char*, unsigned long, JavaThread*) () from /opt/java/openjdk/lib/server/libjvm.so
#18 0x00007fc768f374fa in Verifier::verify(InstanceKlass*, bool, JavaThread*) () from /opt/java/openjdk/lib/server/libjvm.so
#19 0x00007fc768825313 in InstanceKlass::link_class_impl(JavaThread*) () from /opt/java/openjdk/lib/server/libjvm.so
#20 0x00007fc76882a893 in InstanceKlass::initialize_impl(JavaThread*) () from /opt/java/openjdk/lib/server/libjvm.so
#21 0x00007fc768847c90 in InterpreterRuntime::_new(JavaThread*, ConstantPool*, int) () from /opt/java/openjdk/lib/server/libjvm.so
#22 0x00007fc757b88535 in ?? ()
#23 0x00000000d29781a8 in ?? ()
#24 0x00007fc757b88500 in ?? ()
#25 0x00007fc701cfeea0 in ?? ()
#26 0x00007fc708614606 in ?? ()
#27 0x000000000000000c in ?? ()
#28 0x00007fc70863aa98 in ?? ()
#29 0x0000000000000000 in ?? ()
```

main
```log
#0  __cp_end () at src/thread/x86_64/syscall_cp.s:29
#1  0x00007fc769521617 in __syscall_cp_c (nr=202, u=<optimized out>, v=<optimized out>, w=<optimized out>, x=<optimized out>, y=<optimized out>, z=0) at src/thread/pthread_cancel.c:33
#2  0x00007fc769520b49 in __futex4_cp (to=<optimized out>, val=2, op=128, addr=0x7fc767f14924) at src/thread/__timedwait.c:24
#3  __timedwait_cp (addr=addr@entry=0x7fc767f14924, val=val@entry=2, clk=clk@entry=1, at=at@entry=0x0, priv=128, priv@entry=1) at src/thread/__timedwait.c:52
#4  0x00007fc7695219af in __pthread_cond_timedwait (c=0x7fc703aef290, m=0x7fc703aef268, ts=0x0) at src/thread/pthread_cond_timedwait.c:100
#5  0x00007fc768c5698b in PlatformMonitor::wait(unsigned long) () from /opt/java/openjdk/lib/server/libjvm.so
#6  0x00007fc768bfd9cc in Monitor::wait(unsigned long) () from /opt/java/openjdk/lib/server/libjvm.so
#7  0x00007fc76882348a in InstanceKlass::check_link_state_and_wait(JavaThread*) () from /opt/java/openjdk/lib/server/libjvm.so
#8  0x00007fc768825049 in InstanceKlass::link_class_impl(JavaThread*) () from /opt/java/openjdk/lib/server/libjvm.so
#9  0x00007fc76882a893 in InstanceKlass::initialize_impl(JavaThread*) () from /opt/java/openjdk/lib/server/libjvm.so
#10 0x00007fc768847c90 in InterpreterRuntime::_new(JavaThread*, ConstantPool*, int) () from /opt/java/openjdk/lib/server/libjvm.so
#11 0x00007fc757b88535 in ?? ()
#12 0x00000000d291dff8 in ?? ()
#13 0x00007fc757b88500 in ?? ()
#14 0x00007fc767f14d70 in ?? ()
#15 0x00007fc708614606 in ?? ()
#16 0x000000000000000c in ?? ()
#17 0x00007fc70863aa98 in ?? ()
#18 0x0000000000000000 in ?? ()
```

확인해보니 락을 잡기 시작하는 지점은 `SystemDictionary::resolve_instance_class_or_null`과 `InstanceKlass::check_link_state_and_wait`였습니다.  
이에 따라 `SystemDictionary`와 `InstanceKlass`의 코드를 살펴보며 의심되는 지점을 조사하였습니다.  
([주요 코드1](https://github.com/openjdk/jdk/blob/65646b5f81279a7fcef3ea04ef9894cf66f77a5a/src/hotspot/share/classfile/systemDictionary.cpp#L248-L255), [주요 코드2](https://github.com/openjdk/jdk/blob/65646b5f81279a7fcef3ea04ef9894cf66f77a5a/src/hotspot/share/classfile/systemDictionary.cpp#L600-L609))


```java
// from systemDictionary.cpp
Handle SystemDictionary::get_loader_lock_or_null(Handle class_loader) {
  // If class_loader is null or parallelCapable, the JVM doesn't acquire a lock while loading.
  if (is_parallelCapable(class_loader)) {
    return Handle();
  } else {
    return class_loader;
  }
}
```

```java
// from systemDictionary.cpp
  // Non-bootstrap class loaders will call out to class loader and
  // define via jvm/jni_DefineClass which will acquire the
  // class loader object lock to protect against multiple threads
  // defining the class in parallel by accident.
  // This lock must be acquired here so the waiter will find
  // any successful result in the SystemDictionary and not attempt
  // the define.
  // ParallelCapable class loaders and the bootstrap classloader
  // do not acquire lock here.
  Handle lockObject = get_loader_lock_or_null(class_loader);
```

해당 코드와 스택 트레이스를 통해 "JVM은 클래스 검증과정이나 클래스 로딩을 시작할 때, `ClassLoader`의 락을 잡는다"는 사실을 확인할 수 있었으며, 이로 인해 `main`과 `dd-trace-processor` 간에 데드락이 발생하고 있음을 추측할 수 있었습니다.

만약 `ClassLoader`가 `ParallelCapable`하다면 `ClassLoader`의 락을 잡지 않지만, `jstack`으로 확인한 `LaunchedClassLoader`는 `ParallelCapable`하지 않다는 사실을 IntelliJ의 Remote Debugging을 통해 확인할 수 있었습니다.

이러한 정보들을 모두 종합하여, 결론적으로 문제가 발생하는 시나리오를 완성할 수 있었습니다.


# 문제 시나리오

main 함수를 최대한 간단히 하면 다음과 같습니다.

```kotlin
func main() {
  ReactorDebugAgent.init()
  RajinApplication.start()
}
```

현재 데드락이 발생하는 시나리오는 다음과 같습니다.

1. `dd-java-agent.jar`가 실행되면서 `dd-trace-processor` 스레드를 생성합니다.
2. `main` 함수가 실행되기 시작합니다.
3. `dd-trace-processor`의 로직이 멀티스레드 환경에서 `main` 함수와 동시에 수행되며, 해당 로직에 필요한 클래스들을 로딩하고 있습니다.
4. `main` 함수에서 `ReactorDebugAgent.init()`이 호출되면서, 이 시점 이후에 발생하는 모든 클래스 로딩 과정에서는 `ReactorDebugAgent` 내부의 변환기(transformer)가 실행됩니다.
5. `main` 함수에서 `RajinApplication`을 실행하기 위해 해당 클래스의 로딩을 시작합니다.  
   - **이때 `RajinApplication`의 클래스 로더인 `LaunchedClassLoader`에 대한 락을 획득합니다.**
6. `dd-trace-processor`에서도 클래스 로딩이 발생하는데, 4번에서 설명한 바와 같이 이제 `ReactorDebugAgent`의 변환기가 실행됩니다.  
   1. 참고: 3번 시점에서는 아직 `ReactorDebugAgent.init()`이 실행되지 않았기 때문에 변환기가 적용되지 않았습니다.  
   2. `dd-trace-processor`에서 `ReactorDebugAgent`의 변환기 내부 로직이 실행됩니다.  
   3. 해당 변환기 안의 `ClassWriter cw = new ClassWriter(cr, ClassWriter.COMPUTE_MAXS);` 코드를 실행하기 위해 `ClassWriter` 클래스의 로딩을 시도합니다.  
   4. 이 `ClassWriter`의 클래스 로더는 `LaunchedClassLoader`이기 때문에, **락을 획득하려 하지만 이미 `main`이 보유하고 있어 대기 상태에 빠집니다.**
7. 한편 `main`에서도 5번에서 수행하던 클래스 로딩 중에 `ReactorDebugAgent`의 변환기가 실행되고, 이 또한 `ClassWriter`의 로딩을 시도하게 됩니다.  
   - 이미 `dd-trace-processor`가 해당 클래스를 로딩 중이므로, 해당 작업이 완료되기를 **기다리게 됩니다.**

결과적으로 6번과 7번의 교착 상황으로 인해 데드락이 형성됩니다.

- `dd-trace-processor`: `ClassWriter`를 로딩하기 위해 `LaunchedClassLoader`의 락을 획득하려고 시도함  
- `main`: 이미 `LaunchedClassLoader`의 락을 보유한 채 `ClassWriter`의 로딩이 완료되기를 기다림

이 시나리오를 통해 데드락이 발생할 수 있음을 알 수 있으며, 해당 문제가 간헐적으로 발생하는 이유도 설명됩니다.  
즉, `main` 스레드가 `RajinApplication` 클래스의 로딩을 시작하여 `LaunchedClassLoader`의 락을 획득하는 시점과 `ReactorDebugAgent`의 변환기가 활성화되는 시점 사이에  
`dd-trace-processor` 스레드에서 변환기가 실행된다면 이 문제가 발생하게 됩니다.  
멀티스레드 환경에서 이와 같은 타이밍이 맞아떨어져야만 데드락이 생기기 때문에, 현상이 간헐적으로 나타나는 것입니다.


# 문제 해결

그런데 이 문제는 애초에 멀티스레드 환경에서 클래스 로딩을 수행할 경우 발생할 수밖에 없는 구조적 문제입니다.  
`LaunchedClassLoader`가 실제로 `ParallelCapable`하다면 클래스 로더의 락을 획득하지 않기 때문에 이러한 문제가 발생하지 않았을 것입니다.

`LaunchedClassLoader`는 Spring Boot의 기본 클래스 로더이기 때문에, 병렬 클래스 로딩을 지원하지 않는다는 점이 매우 이상하게 느껴졌습니다.

실제로 코드를 보면 [`LaunchedClassLoader`는 스스로를 `ParallelCapable`로 등록하려고 시도합니다.](https://github.com/spring-projects/spring-boot/blob/e49a2daf38bccbb956043f1cd17bdf9296840871/spring-boot-project/spring-boot-tools/spring-boot-loader-classic/src/main/java/org/springframework/boot/loader/LaunchedURLClassLoader.java#L42-L49)

```java
public class LaunchedURLClassLoader extends URLClassLoader {
  private static final int BUFFER_SIZE = 4096;
  static {
    ClassLoader.registerAsParallelCapable();
  }
```

하지만 그럼에도 불구하고 `ParallelCapable`로 등록되지 않는 이유는, [`registerAsParallelCapable` 메서드가 동작하기 위해서는 해당 클래스 로더의 상위 클래스 또한 `ParallelCapable`이어야 하기 때문](https://github.com/openjdk/jdk/blob/f174bbd3baf351ae9248b70454b3bc5a89acd7c6/src/java.base/share/classes/java/lang/ClassLoader.java#L270-L284)입니다.


```java
// from ClassLoader.java
static boolean register(Class<? extends ClassLoader> c) {
  synchronized (loaderTypes) {
    if (loaderTypes.contains(c.getSuperclass())) {
      // register the class loader as parallel capable
      // if and only if all of its super classes are.
      // Note: given current classloading sequence, if
      // the immediate super class is parallel capable,
      // all the super classes higher up must be too.
      loaderTypes.add(c);
      return true;
    } else {
      return false;
    }
  }
}
```

Spring Boot의 코드와 코드 변경 이력을 살펴보면, 병렬 클래스 로딩을 의도하고 있었던 것으로 보입니다. 그러나 상위 클래스의 조건을 놓쳐 등록이 되지 않았던 것으로 판단됩니다. 이에 따라 [Spring Boot에 Pull Request를 보내어](https://github.com/spring-projects/spring-boot/pull/41665) 해당 문제를 수정하였습니다.

이후 해당 PR이 반영된 Spring Boot 버전을 사용하여 문제를 해결할 수 있었습니다.

## 번외: 왜 갑자기 문제가 생겼는가?

이 문제가 왜 갑자기 나타났는지를 추적해본 결과, [Spring Boot 3.2부터 기본적으로 사용하는 클래스 로더가 변경되었기 때문](https://github.com/spring-projects/spring-boot/wiki/Spring-Boot-3.2-Release-Notes?source=post_page-----a3656f8e69b4--------------------------------#nested-jar-support)이었습니다.

실제로 문제가 발생했던 서비스들은 모두 Spring Boot를 3.2 버전으로 업그레이드한 이후부터 해당 현상이 나타나기 시작하였습니다.

# 마무리

문제를 해결하기까지 많은 시행착오가 있었던 것도 사실입니다. 만약 `jstack`에서 확인된 스레드 상태가 Java 단의 락이 아니라 JVM 내부의 네이티브 코드 수준에서 발생할 수 있다는 사실을 알았더라면,  
또 JVM에서 클래스 로딩과 변환기(Transformer)가 어떻게 동작하는지를 이해하고 있었더라면,  
그리고 클래스 로더가 `ParallelCapable`하지 않다는 것이 어떤 의미인지 미리 알고 있었더라면,  
더 빠르게 디버깅을 마칠 수 있었을지도 모릅니다.

하지만 모든 것을 항상 알고 있을 수는 없습니다. 대신, 문제를 해결하는 길을 알고 있는 것이 더 중요하다고 생각합니다.  
~~(사실 요즘 같은 시대에는 `jstack` 출력만 ChatGPT에게 보여줘도 꽤 많은 것을 알 수 있습니다)~~

Java 애플리케이션에 문제가 발생했을 때, `jstack`, 원격 디버깅, JVM 로그 설정 방법, `gdb` 같은 도구를 사용할 수 있다는 사실만 알고 있어도 문제를 훨씬 빠르게 파악할 수 있습니다.

긴 글 읽어주셔서 감사합니다.