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

먼저 로컬 환경에서 문제를 재현하고자 하였으나, Datadog가 비활성화된 설정이어서 재현이 어려웠습니다. 이에 따라 Pod에 Remote Debug 설정을 추가하고, IntelliJ를 통해 Stacktrace 상에서 Local Variable과 함수 인자 등을 확인할 수 있었습니다.

확인된 상태는 다음과 같았습니다.

- main 스레드는 RajinApplication 클래스를 로딩 중이며, 이 과정에서 ReactorDebugAgent에 등록된 ClassFileTransformer 실행 중 Block 상태임
- `dd-`로 시작하는 Datadog 관련 스레드들도 클래스 로딩 중 ReactorDebugAgent의 Transformer 실행 중 Block 상태임

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

현재 데드락이 생기는 시나리오는 다음과 같습니다.

1. dd-java-agent.jar 가 돌면서 dd-trace-processor thread 를 만듬
2. main 함수가 실행되기 시작
3. dd-trace-processor 의 내용물이 멀티쓰레드 환경에서 main 함수의 내용물과 동시에 진행되고 있으며, dd-trace-processor 내용물 실행에 필요한 class loading 을 열심히 하고 있다.
4. main 함수의 `ReactorDebugAgent.init()` 이 실행되면서 이 순간 이후에 일어나는 모든 class loading 에서는 ReactorDebugAgent 내의 transformer 가 실행이 된다.
5. main 함수에서 RajinApplication 를 실행하기 위해 RajinApplication 의 class loading 이 시작됨
    1. **RajinApplication 의 classLoader 인 LaunchedClassLoader 에 lock 을 잡음**
6. dd-trace-processor 에서 class Loading 이 실행되는데, 4번에 의해서 이젠 ReactorDebugAgent 의 transformer 가 실행됨.
    1. 참고 : 3번에서는 ReactorDebugAgent.init() 이 실행되기 전이니 transformer 가 실행이 안되고 있었다.
    2. dd-trace-processor 에서 ReactorDebugAgent 의 transformer 의 내용물을 실행한다.
	3. ReactorDebugAgent 의 transformer 안에 있는 `ClassWriter cw = new ClassWriter(cr, ClassWriter.COMPUTE_MAXS);` 코드를 실행하기 위해 ClassWriter 의 class loading 을 시작한다.
    3. ClassWriter 의 classLoader 는 LaunchedClassLoader 이기 때문에 **LaunchedClassLoader 에 lock 을 잡으려고 시도 하는데 이미 main 에서 잡고 있기 때문에 blocked**
7. main 에서 5번에서 하고 있던 ClassLoading 에서  ReactorDebugAgent 의 transformer 가 실행되면서 ClassWriter 의 class loading 시도
    1. 이미 dd-trace-processor 에서 로딩을 하고 있기에, 해당 쓰레드에서 로딩을 끝내면 그 로딩된 결과물을 사용하기 위해서 **dd-trace-processor 에 진행하고 있는 ClassWriter 의 class loading 을 기다림.**

6번과 7번에 의해서 데드락이 형성이 됩니다.
- dd-trace-processor : ClassWriter 를 로딩을 위해서, LaunchedClassLoader 의 lock 을 잡으려고 함
- main : LaunchedClassLoader 의 lock 을 잡은 상태로 ClassWriter 로딩을 기다림

이 시나리오를 보면 데드락이 생길수 있고, 왜 간헐적으로 문제가 생겼는지도 알수 있습니다. main thread 에서 "RajinApplication 의 class loading 시작하면서 LaunchedClassLoader 의 락을 잡는 그 타이밍"과 "ReactorDebugAgent 의 transformer 가 시작하는 그 타이밍" 사이에 "dd-trace-processor thread 에서 ReactorDebugAgent의 transformer 가 실행"이 되면 생기는 문제이기에, 멀티 쓰레딩 환경에서 타이밍이 맞아 떨어져야 발생하는 문제이기 때문입니다.

# 문제 해결

근데 이 문제들은 애초에 멀티쓰레드 환경에서 class loading 을 하면 생길수 밖에 없는 문제입니다. LaunchedClassLoader 가 사실 ParallelCapable 이라면 lock 을 잡지 않을것이기 때문에 문제가 발생하지 않을 것입니다. LaunchedClassLoader 는 Spring Boot 의 기본 ClassLoader 이기에 Parallel class loading 을 지원하지 않는다는 것이 매우 이상했습니다.

실제로 코드를 보면 [LaunchedClassLoader 는 스스로를 ParallelCapable 로 취급하려고 합니다.](https://github.com/spring-projects/spring-boot/blob/e49a2daf38bccbb956043f1cd17bdf9296840871/spring-boot-project/spring-boot-tools/spring-boot-loader-classic/src/main/java/org/springframework/boot/loader/LaunchedURLClassLoader.java#L42-L49)

```java
public class LaunchedURLClassLoader extends URLClassLoader {
  private static final int BUFFER_SIZE = 4096;
  static {
    ClassLoader.registerAsParallelCapable();
  }
```

하지만 그럼에도 불구하고 ParallelCapable 이 아닌 이유는 [registerAsParallelCapable 이 동작하려면 그 class loader 의 super class 도 ParallelCapable 해야 합니다.](https://github.com/openjdk/jdk/blob/f174bbd3baf351ae9248b70454b3bc5a89acd7c6/src/java.base/share/classes/java/lang/ClassLoader.java#L270-L284)

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

Spring Boot 의 코드와 코드 history 를 살펴보니 parallel capable 로 등록하고 싶었던것 같은데, 잘못하여 등록이 안된 것 같아, [Spring Boot 에 PR](https://github.com/spring-projects/spring-boot/pull/41665) 을 날려 수정하였습니다.

그리고 PR 머지가 된 후의 Spring Boot 릴리즈를 사용하도록 하여 문제를 해결하였습니다.

## 번외 : 왜 갑자기 문제가 생긴것인가?

그러면 이 문제가 저희 서비스에 왜 갑자기 생긴것인지를 찾아보니 [Spring Boot 에서 사용하는 ClassLoader 가 Spring Boot 3.2 에서 바뀌었기 때문이였습니다.](https://github.com/spring-projects/spring-boot/wiki/Spring-Boot-3.2-Release-Notes?source=post_page-----a3656f8e69b4--------------------------------#nested-jar-support)

실제로 문제가 생겼던 서버들은 모두 Spring Boot 버전을 3.2 로 업그레이드 한 후로 부터 문제가 발생하기 시작했었습니다.

# 마무리

문제를 해결하기 위해서 많은 헛걸음질이 있었던 것도 사실입니다. 만약 jstack 에서 보였던 상태가 java 단의 lock 이 아니라 jvm 내부같은 native code 에서 lock 을 잡힌다면 생길수 있는 상태라는걸 알았더라면, JVM 에서 Class Loading 과 Transformer 가 어떤 식으로 동작하는지 알았더라면, ClassLoader 가 ParallelCapable 하지 않다는게 무슨 뜻인지 알고 있었더라면, 훨씬 빠르게 디버깅을 마무리 지을수 있었을것 같습니다.

하지만 언제나 모든 내용을 알고 있는 상태일수는 없기에 그 지식을 얻기 위한 길을 알고 있는것도 중요한 것 같습니다. ~~(사실 요즘같은 시대에는 ChatGPT 한테 jstack 내용물만 던져줘도 상당히 많은 것을 알수 있긴 합니다)~~
Java Program 에 문제가 생겼을 때, jstack, java remote debugging, JVM Log 키는 법, gdb 같은 툴들을 사용할수 있구나 라는 정보만 있더라도 빠르게 문제파악을 할수 있을것 같습니다. 긴 글 읽어주셔서 감사합니다.