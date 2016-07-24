If you are a Java developer working with JNI code, debugging it can be particularly hard, for example if you are experiencing an unexpected SIGSEGV and don't know why.

There are several techniques which we can use to try and help get to the bottom of these:

# Interpreting hs_err_pid files
If the JVM crashes whilst executing our native C++ code via JNI, then it will typically write a crash report to a file named like `hs_err_pid76448.log` in the same location that the `java` process was launched from.

Such a *hs_err* crash report might look like:
```hs_err
#
# A fatal error has been detected by the Java Runtime Environment:
#
#  SIGSEGV (0xb) at pc=0x00007fff87283132, pid=76448, tid=5891
#
# JRE version: Java(TM) SE Runtime Environment (7.0_80-b15) (build 1.7.0_80-b15)
# Java VM: Java HotSpot(TM) 64-Bit Server VM (24.80-b11 mixed mode bsd-amd64 compressed oops)
# Problematic frame:
# C  [libsystem_c.dylib+0x1132]  strlen+0x12
#
# Failed to write core dump. Core dumps have been disabled. To enable core dumping, try "ulimit -c unlimited" before starting Java again
#
# If you would like to submit a bug report, please visit:
#   http://bugreport.java.com/bugreport/crash.jsp
# The crash happened outside the Java Virtual Machine in native code.
# See problematic frame for where to report the bug.
#

---------------  T H R E A D  ---------------

Current thread (0x00007fc3c2007800):  JavaThread "main" [_thread_in_native, id=5891, stack(0x000070000011a000,0x000070000021a000)]

siginfo:si_signo=SIGSEGV: si_errno=0, si_code=1 (SEGV_MAPERR), si_addr=0x00000000fffffff0

Registers:
RAX=0x00000000ffffffff, RBX=0x00000000ffffffff, RCX=0x00000000ffffffff, RDX=0x00000000ffffffff
RSP=0x0000700000216450, RBP=0x0000700000216450, RSI=0x0000000000000007, RDI=0x00000000fffffff0
R8 =0x00000000fffffffc, R9 =0x00007fc3c143f8e8, R10=0x00000000ffffffff, R11=0x0000000102ac7f40
R12=0x00007fc3c288a600, R13=0x00007fc3c1442bf8, R14=0x00007fc3c288a638, R15=0x00007fc3c288a638
RIP=0x00007fff87283132, EFLAGS=0x0000000000010206, ERR=0x0000000000000004
  TRAPNO=0x000000000000000e

Top of Stack: (sp=0x0000700000216450)
0x0000700000216450:   0000700000216490 0000000112bb538a
0x0000700000216460:   0000000000000000 0000000000000000
0x0000700000216470:   0000000000000000 00007fc3c289c800
0x0000700000216480:   00007fc3c288a638 00007fc3c143c0e8
0x0000700000216490:   00007000002166f0 0000000112bcd82c
0x00007000002164a0:   303733312e34333a 00007fc3c288a608
0x00007000002164b0:   0000700000216ca8 697265766f636552
0x00007000002164c0:   206d6f726620676e 74736566696e616d
0x00007000002164d0:   4d203a656c696620 2d54534546494e41
0x00007000002164e0:   00007fc3c289c800 00007fc3c143a870
0x00007000002164f0:   0000000000000060 00007fc3c1530e40
0x0000700000216500:   00007fc300000006 0000000100c2d000
0x0000700000216510:   00007fc3c1500000 0000700000216df0
0x0000700000216520:   0000700000216550 0000700000216df8
0x0000700000216530:   0000700000216e00 00007fc3c28d7000
0x0000700000216540:   0000000100c30a00 0000000000000006
0x0000700000216550:   0000700000216590 00007fff91b94154
0x0000700000216560:   00007fc3c28a6808 0000000000000004
0x0000700000216570:   0000000100c47a00 0000000100c2d000
0x0000700000216580:   0000000100c48e00 0000000000001400
0x0000700000216590:   0000700000216680 00007fff91b90ee5
0x00007000002165a0:   0000000000000001 00007fc3c1530e46
0x00007000002165b0:   00007000002166a0 00007fff91b90a26
0x00007000002165c0:   0000000000001400 0000000100c30a00
0x00007000002165d0:   00007000002166c0 00007fff91b90a26
0x00007000002165e0:   00007fc3c28a6808 0000000000000006
0x00007000002165f0:   0000000100c47a00 0000000100c2d000
0x0000700000216600:   0000000000001400 0000000100c30a00
0x0000700000216610:   0000700000216700 00000000000006e8
0x0000700000216620:   0000000000001002 0000000000c31e00
0x0000700000216630:   0000000000001002 0000000100c48e00
0x0000700000216640:   ff80000000001002 00000000c153ffff 

Instructions: (pc=0x00007fff87283132)
0x00007fff87283112:   0e 01 f3 0f 7f 44 0f 01 5d c3 90 90 90 90 55 48
0x00007fff87283122:   89 e5 48 89 f9 48 89 fa 48 83 e7 f0 66 0f ef c0
0x00007fff87283132:   66 0f 74 07 66 0f d7 f0 48 83 e1 0f 48 83 c8 ff
0x00007fff87283142:   48 d3 e0 21 c6 74 17 0f bc c6 48 29 d7 48 01 f8 

Register to memory mapping:

RAX=0x00000000ffffffff is an unknown value
RBX=0x00000000ffffffff is an unknown value
RCX=0x00000000ffffffff is an unknown value
RDX=0x00000000ffffffff is an unknown value
RSP=0x0000700000216450 is pointing into the stack for thread: 0x00007fc3c2007800
RBP=0x0000700000216450 is pointing into the stack for thread: 0x00007fc3c2007800
RSI=0x0000000000000007 is an unknown value
RDI=0x00000000fffffff0 is an unknown value
R8 =0x00000000fffffffc is an unknown value
R9 =0x00007fc3c143f8e8 is an unknown value
R10=0x00000000ffffffff is an unknown value
R11=0x0000000102ac7f40 is at entry_point+0 in (nmethod*)0x0000000102ac7e10
R12=0x00007fc3c288a600 is an unknown value
R13=0x00007fc3c1442bf8 is an unknown value
R14=0x00007fc3c288a638 is an unknown value
R15=0x00007fc3c288a638 is an unknown value


Stack: [0x000070000011a000,0x000070000021a000],  sp=0x0000700000216450,  free space=1009k
Native frames: (J=compiled Java code, j=interpreted, Vv=VM code, C=native code)
C  [libsystem_c.dylib+0x1132]  strlen+0x12
C  [librocksdbjni-osx.jnilib+0x1c38a]  rocksdb::InternalKeyComparator::InternalKeyComparator(rocksdb::Comparator const*)+0x4a
C  [librocksdbjni-osx.jnilib+0x3482c]  rocksdb::ColumnFamilyData::ColumnFamilyData(unsigned int, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&, rocksdb::Version*, rocksdb::Cache*, rocksdb::WriteBufferManager*, rocksdb::ColumnFamilyOptions const&, rocksdb::DBOptions const*, rocksdb::EnvOptions const&, rocksdb::ColumnFamilySet*)+0x7c
C  [librocksdbjni-osx.jnilib+0x382dd]  rocksdb::ColumnFamilySet::CreateColumnFamily(std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&, unsigned int, rocksdb::Version*, rocksdb::ColumnFamilyOptions const&)+0x7d
C  [librocksdbjni-osx.jnilib+0x12ca6f]  rocksdb::VersionSet::CreateColumnFamily(rocksdb::ColumnFamilyOptions const&, rocksdb::VersionEdit*)+0xaf
C  [librocksdbjni-osx.jnilib+0x12d831]  rocksdb::VersionSet::Recover(std::__1::vector<rocksdb::ColumnFamilyDescriptor, std::__1::allocator<rocksdb::ColumnFamilyDescriptor> > const&, bool)+0xc51
C  [librocksdbjni-osx.jnilib+0x80df4]  rocksdb::DBImpl::Recover(std::__1::vector<rocksdb::ColumnFamilyDescriptor, std::__1::allocator<rocksdb::ColumnFamilyDescriptor> > const&, bool, bool, bool)+0x244
C  [librocksdbjni-osx.jnilib+0x9c0aa]  rocksdb::DB::Open(rocksdb::DBOptions const&, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&, std::__1::vector<rocksdb::ColumnFamilyDescriptor, std::__1::allocator<rocksdb::ColumnFamilyDescriptor> > const&, std::__1::vector<rocksdb::ColumnFamilyHandle*, std::__1::allocator<rocksdb::ColumnFamilyHandle*> >*, rocksdb::DB**)+0xb0a
C  [librocksdbjni-osx.jnilib+0x9b138]  rocksdb::DB::Open(rocksdb::Options const&, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&, rocksdb::DB**)+0x448
C  [librocksdbjni-osx.jnilib+0x1234c]  std::__1::__function::__func<rocksdb::Status (*)(rocksdb::Options const&, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&, rocksdb::DB**), std::__1::allocator<rocksdb::Status (*)(rocksdb::Options const&, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&, rocksdb::DB**)>, rocksdb::Status (rocksdb::Options const&, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&, rocksdb::DB**)>::operator()(rocksdb::Options const&, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&, rocksdb::DB**&&)+0x1c
C  [librocksdbjni-osx.jnilib+0xd84b]  rocksdb_open_helper(JNIEnv_*, long, _jstring*, std::__1::function<rocksdb::Status (rocksdb::Options const&, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&, rocksdb::DB**)>)+0x8b
C  [librocksdbjni-osx.jnilib+0xd9ab]  Java_org_rocksdb_RocksDB_open__JLjava_lang_String_2+0x4b
j  org.rocksdb.RocksDB.open(JLjava/lang/String;)J+0
j  org.rocksdb.RocksDB.open(Lorg/rocksdb/Options;Ljava/lang/String;)Lorg/rocksdb/RocksDB;+9
j  org.rocksdb.util.BytewiseComparatorTest.openDatabase(Ljava/nio/file/Path;Lorg/rocksdb/AbstractComparator;)Lorg/rocksdb/RocksDB;+28
j  org.rocksdb.util.BytewiseComparatorTest.java_vs_java_directBytewiseComparator()V+37
v  ~StubRoutines::call_stub
V  [libjvm.dylib+0x2dc898]  JavaCalls::call_helper(JavaValue*, methodHandle*, JavaCallArguments*, Thread*)+0x22a
V  [libjvm.dylib+0x2dc668]  JavaCalls::call(JavaValue*, methodHandle, JavaCallArguments*, Thread*)+0x28
V  [libjvm.dylib+0x468428]  Reflection::invoke(instanceKlassHandle, methodHandle, Handle, bool, objArrayHandle, BasicType, objArrayHandle, bool, Thread*)+0x9fc
V  [libjvm.dylib+0x46888e]  Reflection::invoke_method(oopDesc*, Handle, objArrayHandle, Thread*)+0x16e
V  [libjvm.dylib+0x329247]  JVM_InvokeMethod+0x166
j  sun.reflect.NativeMethodAccessorImpl.invoke0(Ljava/lang/reflect/Method;Ljava/lang/Object;[Ljava/lang/Object;)Ljava/lang/Object;+0
j  sun.reflect.NativeMethodAccessorImpl.invoke(Ljava/lang/Object;[Ljava/lang/Object;)Ljava/lang/Object;+87
j  sun.reflect.DelegatingMethodAccessorImpl.invoke(Ljava/lang/Object;[Ljava/lang/Object;)Ljava/lang/Object;+6
j  java.lang.reflect.Method.invoke(Ljava/lang/Object;[Ljava/lang/Object;)Ljava/lang/Object;+57
j  org.junit.runners.model.FrameworkMethod$1.runReflectiveCall()Ljava/lang/Object;+15
j  org.junit.internal.runners.model.ReflectiveCallable.run()Ljava/lang/Object;+1
j  org.junit.runners.model.FrameworkMethod.invokeExplosively(Ljava/lang/Object;[Ljava/lang/Object;)Ljava/lang/Object;+10
j  org.junit.internal.runners.statements.InvokeMethod.evaluate()V+12
j  org.junit.runners.ParentRunner.runLeaf(Lorg/junit/runners/model/Statement;Lorg/junit/runner/Description;Lorg/junit/runner/notification/RunNotifier;)V+17
j  org.junit.runners.BlockJUnit4ClassRunner.runChild(Lorg/junit/runners/model/FrameworkMethod;Lorg/junit/runner/notification/RunNotifier;)V+30
j  org.junit.runners.BlockJUnit4ClassRunner.runChild(Ljava/lang/Object;Lorg/junit/runner/notification/RunNotifier;)V+6
j  org.junit.runners.ParentRunner$3.run()V+12
j  org.junit.runners.ParentRunner$1.schedule(Ljava/lang/Runnable;)V+1
j  org.junit.runners.ParentRunner.runChildren(Lorg/junit/runner/notification/RunNotifier;)V+44
j  org.junit.runners.ParentRunner.access$000(Lorg/junit/runners/ParentRunner;Lorg/junit/runner/notification/RunNotifier;)V+2
j  org.junit.runners.ParentRunner$2.evaluate()V+8
j  org.junit.runners.ParentRunner.run(Lorg/junit/runner/notification/RunNotifier;)V+20
j  org.junit.runners.Suite.runChild(Lorg/junit/runner/Runner;Lorg/junit/runner/notification/RunNotifier;)V+2
j  org.junit.runners.Suite.runChild(Ljava/lang/Object;Lorg/junit/runner/notification/RunNotifier;)V+6
j  org.junit.runners.ParentRunner$3.run()V+12
j  org.junit.runners.ParentRunner$1.schedule(Ljava/lang/Runnable;)V+1
j  org.junit.runners.ParentRunner.runChildren(Lorg/junit/runner/notification/RunNotifier;)V+44
j  org.junit.runners.ParentRunner.access$000(Lorg/junit/runners/ParentRunner;Lorg/junit/runner/notification/RunNotifier;)V+2
j  org.junit.runners.ParentRunner$2.evaluate()V+8
j  org.junit.runners.ParentRunner.run(Lorg/junit/runner/notification/RunNotifier;)V+20
j  org.junit.runner.JUnitCore.run(Lorg/junit/runner/Runner;)Lorg/junit/runner/Result;+37
j  org.junit.runner.JUnitCore.run(Lorg/junit/runner/Request;)Lorg/junit/runner/Result;+5
j  org.junit.runner.JUnitCore.run(Lorg/junit/runner/Computer;[Ljava/lang/Class;)Lorg/junit/runner/Result;+6
j  org.junit.runner.JUnitCore.run([Ljava/lang/Class;)Lorg/junit/runner/Result;+5
j  org.rocksdb.test.RocksJunitRunner.main([Ljava/lang/String;)V+93
v  ~StubRoutines::call_stub
V  [libjvm.dylib+0x2dc898]  JavaCalls::call_helper(JavaValue*, methodHandle*, JavaCallArguments*, Thread*)+0x22a
V  [libjvm.dylib+0x2dc668]  JavaCalls::call(JavaValue*, methodHandle, JavaCallArguments*, Thread*)+0x28
V  [libjvm.dylib+0x31004e]  jni_invoke_static(JNIEnv_*, JavaValue*, _jobject*, JNICallType, _jmethodID*, JNI_ArgumentPusher*, Thread*)+0xe6
V  [libjvm.dylib+0x3092d5]  jni_CallStaticVoidMethodV+0x9c
V  [libjvm.dylib+0x31c28e]  checked_jni_CallStaticVoidMethod+0x16f
C  [java+0x30fe]  JavaMain+0x91d
C  [libsystem_pthread.dylib+0x399d]  _pthread_body+0x83
C  [libsystem_pthread.dylib+0x391a]  _pthread_body+0x0
C  [libsystem_pthread.dylib+0x1351]  thread_start+0xd

Java frames: (J=compiled Java code, j=interpreted, Vv=VM code)
j  org.rocksdb.RocksDB.open(JLjava/lang/String;)J+0
j  org.rocksdb.RocksDB.open(Lorg/rocksdb/Options;Ljava/lang/String;)Lorg/rocksdb/RocksDB;+9
j  org.rocksdb.util.BytewiseComparatorTest.openDatabase(Ljava/nio/file/Path;Lorg/rocksdb/AbstractComparator;)Lorg/rocksdb/RocksDB;+28
j  org.rocksdb.util.BytewiseComparatorTest.java_vs_java_directBytewiseComparator()V+37
v  ~StubRoutines::call_stub
j  sun.reflect.NativeMethodAccessorImpl.invoke0(Ljava/lang/reflect/Method;Ljava/lang/Object;[Ljava/lang/Object;)Ljava/lang/Object;+0
j  sun.reflect.NativeMethodAccessorImpl.invoke(Ljava/lang/Object;[Ljava/lang/Object;)Ljava/lang/Object;+87
j  sun.reflect.DelegatingMethodAccessorImpl.invoke(Ljava/lang/Object;[Ljava/lang/Object;)Ljava/lang/Object;+6
j  java.lang.reflect.Method.invoke(Ljava/lang/Object;[Ljava/lang/Object;)Ljava/lang/Object;+57
j  org.junit.runners.model.FrameworkMethod$1.runReflectiveCall()Ljava/lang/Object;+15
j  org.junit.internal.runners.model.ReflectiveCallable.run()Ljava/lang/Object;+1
j  org.junit.runners.model.FrameworkMethod.invokeExplosively(Ljava/lang/Object;[Ljava/lang/Object;)Ljava/lang/Object;+10
j  org.junit.internal.runners.statements.InvokeMethod.evaluate()V+12
j  org.junit.runners.ParentRunner.runLeaf(Lorg/junit/runners/model/Statement;Lorg/junit/runner/Description;Lorg/junit/runner/notification/RunNotifier;)V+17
j  org.junit.runners.BlockJUnit4ClassRunner.runChild(Lorg/junit/runners/model/FrameworkMethod;Lorg/junit/runner/notification/RunNotifier;)V+30
j  org.junit.runners.BlockJUnit4ClassRunner.runChild(Ljava/lang/Object;Lorg/junit/runner/notification/RunNotifier;)V+6
j  org.junit.runners.ParentRunner$3.run()V+12
j  org.junit.runners.ParentRunner$1.schedule(Ljava/lang/Runnable;)V+1
j  org.junit.runners.ParentRunner.runChildren(Lorg/junit/runner/notification/RunNotifier;)V+44
j  org.junit.runners.ParentRunner.access$000(Lorg/junit/runners/ParentRunner;Lorg/junit/runner/notification/RunNotifier;)V+2
j  org.junit.runners.ParentRunner$2.evaluate()V+8
j  org.junit.runners.ParentRunner.run(Lorg/junit/runner/notification/RunNotifier;)V+20
j  org.junit.runners.Suite.runChild(Lorg/junit/runner/Runner;Lorg/junit/runner/notification/RunNotifier;)V+2
j  org.junit.runners.Suite.runChild(Ljava/lang/Object;Lorg/junit/runner/notification/RunNotifier;)V+6
j  org.junit.runners.ParentRunner$3.run()V+12
j  org.junit.runners.ParentRunner$1.schedule(Ljava/lang/Runnable;)V+1
j  org.junit.runners.ParentRunner.runChildren(Lorg/junit/runner/notification/RunNotifier;)V+44
j  org.junit.runners.ParentRunner.access$000(Lorg/junit/runners/ParentRunner;Lorg/junit/runner/notification/RunNotifier;)V+2
j  org.junit.runners.ParentRunner$2.evaluate()V+8
j  org.junit.runners.ParentRunner.run(Lorg/junit/runner/notification/RunNotifier;)V+20
j  org.junit.runner.JUnitCore.run(Lorg/junit/runner/Runner;)Lorg/junit/runner/Result;+37
j  org.junit.runner.JUnitCore.run(Lorg/junit/runner/Request;)Lorg/junit/runner/Result;+5
j  org.junit.runner.JUnitCore.run(Lorg/junit/runner/Computer;[Ljava/lang/Class;)Lorg/junit/runner/Result;+6
j  org.junit.runner.JUnitCore.run([Ljava/lang/Class;)Lorg/junit/runner/Result;+5
j  org.rocksdb.test.RocksJunitRunner.main([Ljava/lang/String;)V+93
v  ~StubRoutines::call_stub

---------------  P R O C E S S  ---------------

Java Threads: ( => current thread )
  0x00007fc3c2001000 JavaThread "Service Thread" daemon [_thread_blocked, id=19971, stack(0x00007000010ca000,0x00007000011ca000)]
  0x00007fc3c181f800 JavaThread "C2 CompilerThread1" daemon [_thread_blocked, id=19459, stack(0x0000700000fc7000,0x00007000010c7000)]
  0x00007fc3c3834000 JavaThread "C2 CompilerThread0" daemon [_thread_blocked, id=18947, stack(0x0000700000ec4000,0x0000700000fc4000)]
  0x00007fc3c382f000 JavaThread "Signal Dispatcher" daemon [_thread_blocked, id=15887, stack(0x0000700000dc1000,0x0000700000ec1000)]
  0x00007fc3c2846000 JavaThread "Finalizer" daemon [_thread_blocked, id=14339, stack(0x0000700000c3b000,0x0000700000d3b000)]
  0x00007fc3c2845000 JavaThread "Reference Handler" daemon [_thread_blocked, id=13827, stack(0x0000700000b38000,0x0000700000c38000)]
=>0x00007fc3c2007800 JavaThread "main" [_thread_in_native, id=5891, stack(0x000070000011a000,0x000070000021a000)]

Other Threads:
  0x00007fc3c2842800 VMThread [stack: 0x0000700000a35000,0x0000700000b35000] [id=13315]
  0x00007fc3c2023000 WatcherThread [stack: 0x00007000011cd000,0x00007000012cd000] [id=20483]

VM state:not at safepoint (normal execution)

VM Mutex/Monitor currently owned by a thread: None

Heap
 PSYoungGen      total 76800K, used 2642K [0x00000007aaa80000, 0x00000007b0000000, 0x0000000800000000)
  eden space 66048K, 4% used [0x00000007aaa80000,0x00000007aad14888,0x00000007aeb00000)
  from space 10752K, 0% used [0x00000007aeb00000,0x00000007aeb00000,0x00000007af580000)
  to   space 10752K, 0% used [0x00000007af580000,0x00000007af580000,0x00000007b0000000)
 ParOldGen       total 174592K, used 1284K [0x0000000700000000, 0x000000070aa80000, 0x00000007aaa80000)
  object space 174592K, 0% used [0x0000000700000000,0x00000007001412e0,0x000000070aa80000)
 PSPermGen       total 21504K, used 6746K [0x00000006fae00000, 0x00000006fc300000, 0x0000000700000000)
  object space 21504K, 31% used [0x00000006fae00000,0x00000006fb496848,0x00000006fc300000)

Card table byte_map: [0x0000000105a65000,0x000000010628f000] byte_map_base: 0x000000010228e000

Polling page: 0x0000000100ecf000

Code Cache  [0x0000000102a65000, 0x0000000102cd5000, 0x0000000105a65000)
 total_blobs=314 nmethods=63 adapters=213 free_code_cache=48648Kb largest_free_block=49809216

Compilation events (10 events):
Event: 0.355 Thread 0x00007fc3c181f800   59 %           java.lang.String::startsWith @ 42 (72 bytes)
Event: 0.360 Thread 0x00007fc3c181f800 nmethod 59% 0x0000000102adea50 code [0x0000000102adeba0, 0x0000000102adee98]
Event: 0.360 Thread 0x00007fc3c3834000   60             java.lang.String::startsWith (72 bytes)
Event: 0.362 Thread 0x00007fc3c3834000 nmethod 60 0x0000000102ade610 code [0x0000000102ade760, 0x0000000102ade918]
Event: 0.416 Thread 0x00007fc3c181f800   61             java.util.Properties$LineReader::readLine (452 bytes)
Event: 0.420 Thread 0x00007fc3c3834000   62             java.lang.String::substring (79 bytes)
Event: 0.420 Thread 0x00007fc3c181f800 nmethod 61 0x0000000102adfd10 code [0x0000000102adfec0, 0x0000000102ae0458]
Event: 0.422 Thread 0x00007fc3c3834000 nmethod 62 0x0000000102ae3dd0 code [0x0000000102ae3f40, 0x0000000102ae42d8]
Event: 0.490 Thread 0x00007fc3c181f800   63             java.util.Arrays::fill (21 bytes)
Event: 0.506 Thread 0x00007fc3c181f800 nmethod 63 0x0000000102ae4550 code [0x0000000102ae46a0, 0x0000000102ae47f8]

GC Heap History (10 events):
Event: 0.392 GC heap before
{Heap before GC invocations=2 (full 1):
 PSYoungGen      total 76800K, used 2327K [0x00000007aaa80000, 0x00000007b0000000, 0x0000000800000000)
  eden space 66048K, 0% used [0x00000007aaa80000,0x00000007aaa80000,0x00000007aeb00000)
  from space 10752K, 21% used [0x00000007aeb00000,0x00000007aed45c10,0x00000007af580000)
  to   space 10752K, 0% used [0x00000007af580000,0x00000007af580000,0x00000007b0000000)
 ParOldGen       total 174592K, used 8K [0x0000000700000000, 0x000000070aa80000, 0x00000007aaa80000)
  object space 174592K, 0% used [0x0000000700000000,0x0000000700002000,0x000000070aa80000)
 PSPermGen       total 21504K, used 5852K [0x00000006fae00000, 0x00000006fc300000, 0x0000000700000000)
  object space 21504K, 27% used [0x00000006fae00000,0x00000006fb3b7270,0x00000006fc300000)
Event: 0.409 GC heap after
Heap after GC invocations=2 (full 1):
 PSYoungGen      total 76800K, used 0K [0x00000007aaa80000, 0x00000007b0000000, 0x0000000800000000)
  eden space 66048K, 0% used [0x00000007aaa80000,0x00000007aaa80000,0x00000007aeb00000)
  from space 10752K, 0% used [0x00000007aeb00000,0x00000007aeb00000,0x00000007af580000)
  to   space 10752K, 0% used [0x00000007af580000,0x00000007af580000,0x00000007b0000000)
 ParOldGen       total 174592K, used 2133K [0x0000000700000000, 0x000000070aa80000, 0x00000007aaa80000)
  object space 174592K, 1% used [0x0000000700000000,0x0000000700215678,0x000000070aa80000)
 PSPermGen       total 21504K, used 5851K [0x00000006fae00000, 0x00000006fc300000, 0x0000000700000000)
  object space 21504K, 27% used [0x00000006fae00000,0x00000006fb3b6c90,0x00000006fc300000)
}
Event: 0.490 GC heap before
{Heap before GC invocations=3 (full 1):
 PSYoungGen      total 76800K, used 3302K [0x00000007aaa80000, 0x00000007b0000000, 0x0000000800000000)
  eden space 66048K, 5% used [0x00000007aaa80000,0x00000007aadb9ab0,0x00000007aeb00000)
  from space 10752K, 0% used [0x00000007aeb00000,0x00000007aeb00000,0x00000007af580000)
  to   space 10752K, 0% used [0x00000007af580000,0x00000007af580000,0x00000007b0000000)
 ParOldGen       total 174592K, used 2133K [0x0000000700000000, 0x000000070aa80000, 0x00000007aaa80000)
  object space 174592K, 1% used [0x0000000700000000,0x0000000700215678,0x000000070aa80000)
 PSPermGen       total 21504K, used 6249K [0x00000006fae00000, 0x00000006fc300000, 0x0000000700000000)
  object space 21504K, 29% used [0x00000006fae00000,0x00000006fb41a7d0,0x00000006fc300000)
Event: 0.491 GC heap after
Heap after GC invocations=3 (full 1):
 PSYoungGen      total 76800K, used 288K [0x00000007aaa80000, 0x00000007b0000000, 0x0000000800000000)
  eden space 66048K, 0% used [0x00000007aaa80000,0x00000007aaa80000,0x00000007aeb00000)
  from space 10752K, 2% used [0x00000007af580000,0x00000007af5c8000,0x00000007b0000000)
  to   space 10752K, 0% used [0x00000007aeb00000,0x00000007aeb00000,0x00000007af580000)
 ParOldGen       total 174592K, used 2133K [0x0000000700000000, 0x000000070aa80000, 0x00000007aaa80000)
  object space 174592K, 1% used [0x0000000700000000,0x0000000700215678,0x000000070aa80000)
 PSPermGen       total 21504K, used 6249K [0x00000006fae00000, 0x00000006fc300000, 0x0000000700000000)
  object space 21504K, 29% used [0x00000006fae00000,0x00000006fb41a7d0,0x00000006fc300000)
}
Event: 0.491 GC heap before
{Heap before GC invocations=4 (full 2):
 PSYoungGen      total 76800K, used 288K [0x00000007aaa80000, 0x00000007b0000000, 0x0000000800000000)
  eden space 66048K, 0% used [0x00000007aaa80000,0x00000007aaa80000,0x00000007aeb00000)
  from space 10752K, 2% used [0x00000007af580000,0x00000007af5c8000,0x00000007b0000000)
  to   space 10752K, 0% used [0x00000007aeb00000,0x00000007aeb00000,0x00000007af580000)
 ParOldGen       total 174592K, used 2133K [0x0000000700000000, 0x000000070aa80000, 0x00000007aaa80000)
  object space 174592K, 1% used [0x0000000700000000,0x0000000700215678,0x000000070aa80000)
 PSPermGen       total 21504K, used 6249K [0x00000006fae00000, 0x00000006fc300000, 0x0000000700000000)
  object space 21504K, 29% used [0x00000006fae00000,0x00000006fb41a7d0,0x00000006fc300000)
Event: 0.506 GC heap after
Heap after GC invocations=4 (full 2):
 PSYoungGen      total 76800K, used 0K [0x00000007aaa80000, 0x00000007b0000000, 0x0000000800000000)
  eden space 66048K, 0% used [0x00000007aaa80000,0x00000007aaa80000,0x00000007aeb00000)
  from space 10752K, 0% used [0x00000007af580000,0x00000007af580000,0x00000007b0000000)
  to   space 10752K, 0% used [0x00000007aeb00000,0x00000007aeb00000,0x00000007af580000)
 ParOldGen       total 174592K, used 1286K [0x0000000700000000, 0x000000070aa80000, 0x00000007aaa80000)
  object space 174592K, 0% used [0x0000000700000000,0x00000007001418c8,0x000000070aa80000)
 PSPermGen       total 21504K, used 6249K [0x00000006fae00000, 0x00000006fc300000, 0x0000000700000000)
  object space 21504K, 29% used [0x00000006fae00000,0x00000006fb41a788,0x00000006fc300000)
}
Event: 0.513 GC heap before
{Heap before GC invocations=5 (full 2):
 PSYoungGen      total 76800K, used 3302K [0x00000007aaa80000, 0x00000007b0000000, 0x0000000800000000)
  eden space 66048K, 5% used [0x00000007aaa80000,0x00000007aadb9ad8,0x00000007aeb00000)
  from space 10752K, 0% used [0x00000007af580000,0x00000007af580000,0x00000007b0000000)
  to   space 10752K, 0% used [0x00000007aeb00000,0x00000007aeb00000,0x00000007af580000)
 ParOldGen       total 174592K, used 1286K [0x0000000700000000, 0x000000070aa80000, 0x00000007aaa80000)
  object space 174592K, 0% used [0x0000000700000000,0x00000007001418c8,0x000000070aa80000)
 PSPermGen       total 21504K, used 6263K [0x00000006fae00000, 0x00000006fc300000, 0x0000000700000000)
  object space 21504K, 29% used [0x00000006fae00000,0x00000006fb41df00,0x00000006fc300000)
Event: 0.513 GC heap after
Heap after GC invocations=5 (full 2):
 PSYoungGen      total 76800K, used 96K [0x00000007aaa80000, 0x00000007b0000000, 0x0000000800000000)
  eden space 66048K, 0% used [0x00000007aaa80000,0x00000007aaa80000,0x00000007aeb00000)
  from space 10752K, 0% used [0x00000007aeb00000,0x00000007aeb18000,0x00000007af580000)
  to   space 10752K, 0% used [0x00000007af580000,0x00000007af580000,0x00000007b0000000)
 ParOldGen       total 174592K, used 1286K [0x0000000700000000, 0x000000070aa80000, 0x00000007aaa80000)
  object space 174592K, 0% used [0x0000000700000000,0x00000007001418c8,0x000000070aa80000)
 PSPermGen       total 21504K, used 6263K [0x00000006fae00000, 0x00000006fc300000, 0x0000000700000000)
  object space 21504K, 29% used [0x00000006fae00000,0x00000006fb41df00,0x00000006fc300000)
}
Event: 0.513 GC heap before
{Heap before GC invocations=6 (full 3):
 PSYoungGen      total 76800K, used 96K [0x00000007aaa80000, 0x00000007b0000000, 0x0000000800000000)
  eden space 66048K, 0% used [0x00000007aaa80000,0x00000007aaa80000,0x00000007aeb00000)
  from space 10752K, 0% used [0x00000007aeb00000,0x00000007aeb18000,0x00000007af580000)
  to   space 10752K, 0% used [0x00000007af580000,0x00000007af580000,0x00000007b0000000)
 ParOldGen       total 174592K, used 1286K [0x0000000700000000, 0x000000070aa80000, 0x00000007aaa80000)
  object space 174592K, 0% used [0x0000000700000000,0x00000007001418c8,0x000000070aa80000)
 PSPermGen       total 21504K, used 6263K [0x00000006fae00000, 0x00000006fc300000, 0x0000000700000000)
  object space 21504K, 29% used [0x00000006fae00000,0x00000006fb41df00,0x00000006fc300000)
Event: 0.525 GC heap after
Heap after GC invocations=6 (full 3):
 PSYoungGen      total 76800K, used 0K [0x00000007aaa80000, 0x00000007b0000000, 0x0000000800000000)
  eden space 66048K, 0% used [0x00000007aaa80000,0x00000007aaa80000,0x00000007aeb00000)
  from space 10752K, 0% used [0x00000007aeb00000,0x00000007aeb00000,0x00000007af580000)
  to   space 10752K, 0% used [0x00000007af580000,0x00000007af580000,0x00000007b0000000)
 ParOldGen       total 174592K, used 1284K [0x0000000700000000, 0x000000070aa80000, 0x00000007aaa80000)
  object space 174592K, 0% used [0x0000000700000000,0x00000007001412e0,0x000000070aa80000)
 PSPermGen       total 21504K, used 6263K [0x00000006fae00000, 0x00000006fc300000, 0x0000000700000000)
  object space 21504K, 29% used [0x00000006fae00000,0x00000006fb41df00,0x00000006fc300000)
}

Deoptimization events (5 events):
Event: 0.059 Thread 0x00007fc3c2007800 Uncommon trap: reason=unstable_if action=reinterpret pc=0x0000000102ac9928 method=java.lang.String.hashCode()I @ 14
Event: 0.080 Thread 0x00007fc3c2007800 Uncommon trap: reason=unstable_if action=reinterpret pc=0x0000000102ac52e8 method=java.lang.String.indexOf([CII[CIII)I @ 134
Event: 0.321 Thread 0x00007fc3c2007800 Uncommon trap: reason=unstable_if action=reinterpret pc=0x0000000102ac6080 method=java.lang.String.indexOf(II)I @ 49
Event: 0.329 Thread 0x00007fc3c2007800 Uncommon trap: reason=unstable_if action=reinterpret pc=0x0000000102ad1f60 method=java.lang.String.startsWith(Ljava/lang/String;I)Z @ 25
Event: 0.581 Thread 0x00007fc3c2007800 Uncommon trap: reason=unstable_if action=reinterpret pc=0x0000000102ac42cc method=sun.nio.cs.UTF_8$Encoder.encode([CII[B)I @ 33

Internal exceptions (10 events):
Event: 0.537 Thread 0x00007fc3c2007800 Threw 0x00000007aaa91690 at /Users/java_re/workspace/7u-2-build-macosx-x86_64/jdk7u80/2329/hotspot/src/share/vm/prims/jvm.cpp:1319
Event: 0.540 Thread 0x00007fc3c2007800 Threw 0x00000007aaa95310 at /Users/java_re/workspace/7u-2-build-macosx-x86_64/jdk7u80/2329/hotspot/src/share/vm/prims/jvm.cpp:1319
Event: 0.540 Thread 0x00007fc3c2007800 Threw 0x00000007aaa98390 at /Users/java_re/workspace/7u-2-build-macosx-x86_64/jdk7u80/2329/hotspot/src/share/vm/prims/jvm.cpp:1319
Event: 0.540 Thread 0x00007fc3c2007800 Threw 0x00000007aaa9b910 at /Users/java_re/workspace/7u-2-build-macosx-x86_64/jdk7u80/2329/hotspot/src/share/vm/prims/jvm.cpp:1319
Event: 0.541 Thread 0x00007fc3c2007800 Threw 0x00000007aaa9e988 at /Users/java_re/workspace/7u-2-build-macosx-x86_64/jdk7u80/2329/hotspot/src/share/vm/prims/jvm.cpp:1319
Event: 0.541 Thread 0x00007fc3c2007800 Threw 0x00000007aaaa49d0 at /Users/java_re/workspace/7u-2-build-macosx-x86_64/jdk7u80/2329/hotspot/src/share/vm/prims/jvm.cpp:1319
Event: 0.542 Thread 0x00007fc3c2007800 Threw 0x00000007aaaa93f0 at /Users/java_re/workspace/7u-2-build-macosx-x86_64/jdk7u80/2329/hotspot/src/share/vm/prims/jvm.cpp:1319
Event: 0.542 Thread 0x00007fc3c2007800 Threw 0x00000007aaaaf140 at /Users/java_re/workspace/7u-2-build-macosx-x86_64/jdk7u80/2329/hotspot/src/share/vm/prims/jvm.cpp:1319
Event: 0.548 Thread 0x00007fc3c2007800 Threw 0x00000007aaac7e78 at /Users/java_re/workspace/7u-2-build-macosx-x86_64/jdk7u80/2329/hotspot/src/share/vm/prims/jvm.cpp:1319
Event: 0.580 Thread 0x00007fc3c2007800 Threw 0x00000007aab035a8 at /Users/java_re/workspace/7u-2-build-macosx-x86_64/jdk7u80/2329/hotspot/src/share/vm/prims/jvm.cpp:1319

Events (10 events):
Event: 0.552 loading class 0x00007fc3c152f830 done
Event: 0.552 loading class 0x00007fc3c38b2ed0
Event: 0.552 loading class 0x00007fc3c38b2ed0 done
Event: 0.552 loading class 0x00007fc3c38b1ea0
Event: 0.552 loading class 0x00007fc3c38b1ea0 done
Event: 0.579 loading class 0x00007fc3c3300390
Event: 0.579 loading class 0x00007fc3c3300390 done
Event: 0.581 Thread 0x00007fc3c2007800 Uncommon trap: trap_request=0xffffff75 fr.pc=0x0000000102ac42cc
Event: 0.581 Thread 0x00007fc3c2007800 DEOPT PACKING pc=0x0000000102ac42cc sp=0x0000700000215bd0
Event: 0.581 Thread 0x00007fc3c2007800 DEOPT UNPACKING pc=0x0000000102a9e045 sp=0x0000700000215b88 mode 2


Dynamic libraries:
0x0000000006634000      /System/Library/Frameworks/Cocoa.framework/Versions/A/Cocoa
0x0000000006634000      /System/Library/Frameworks/Security.framework/Versions/A/Security
0x0000000006634000      /System/Library/Frameworks/ApplicationServices.framework/Versions/A/ApplicationServices
0x0000000006634000      /usr/lib/libz.1.dylib
0x0000000006634000      /usr/lib/libSystem.B.dylib
0x0000000006634000      /usr/lib/libobjc.A.dylib
0x0000000006634000      /System/Library/Frameworks/CoreFoundation.framework/Versions/A/CoreFoundation
0x0000000006634000      /System/Library/Frameworks/Foundation.framework/Versions/C/Foundation
0x0000000006634000      /System/Library/Frameworks/AppKit.framework/Versions/C/AppKit
0x0000000006634000      /System/Library/Frameworks/CoreData.framework/Versions/A/CoreData
0x0000000006634000      /System/Library/PrivateFrameworks/RemoteViewServices.framework/Versions/A/RemoteViewServices
0x0000000006634000      /System/Library/PrivateFrameworks/UIFoundation.framework/Versions/A/UIFoundation
0x0000000006634000      /usr/lib/libScreenReader.dylib
0x0000000006634000      /System/Library/Frameworks/Accelerate.framework/Versions/A/Accelerate
0x0000000006634000      /System/Library/Frameworks/IOSurface.framework/Versions/A/IOSurface
0x0000000006634000      /System/Library/Frameworks/AudioToolbox.framework/Versions/A/AudioToolbox
0x0000000006634000      /System/Library/Frameworks/AudioUnit.framework/Versions/A/AudioUnit
0x0000000006634000      /System/Library/PrivateFrameworks/DataDetectorsCore.framework/Versions/A/DataDetectorsCore
0x0000000006634000      /System/Library/PrivateFrameworks/DesktopServicesPriv.framework/Versions/A/DesktopServicesPriv
0x0000000006634000      /System/Library/Frameworks/Carbon.framework/Versions/A/Frameworks/HIToolbox.framework/Versions/A/HIToolbox
0x0000000006634000      /System/Library/Frameworks/QuartzCore.framework/Versions/A/QuartzCore
0x0000000006634000      /System/Library/Frameworks/Carbon.framework/Versions/A/Frameworks/SpeechRecognition.framework/Versions/A/SpeechRecognition
0x0000000006634000      /usr/lib/libauto.dylib
0x0000000006634000      /usr/lib/libicucore.A.dylib
0x0000000006634000      /usr/lib/libxml2.2.dylib
0x0000000006634000      /System/Library/PrivateFrameworks/CoreUI.framework/Versions/A/CoreUI
0x0000000006634000      /System/Library/Frameworks/CoreAudio.framework/Versions/A/CoreAudio
0x0000000006634000      /System/Library/Frameworks/DiskArbitration.framework/Versions/A/DiskArbitration
0x0000000006634000      /usr/lib/liblangid.dylib
0x0000000006634000      /System/Library/PrivateFrameworks/MultitouchSupport.framework/Versions/A/MultitouchSupport
0x0000000006634000      /System/Library/Frameworks/IOKit.framework/Versions/A/IOKit
0x0000000006634000      /usr/lib/libDiagnosticMessagesClient.dylib
0x0000000006634000      /System/Library/Frameworks/CoreServices.framework/Versions/A/CoreServices
0x0000000006634000      /System/Library/PrivateFrameworks/PerformanceAnalysis.framework/Versions/A/PerformanceAnalysis
0x0000000006634000      /System/Library/PrivateFrameworks/GenerationalStorage.framework/Versions/A/GenerationalStorage
0x0000000006634000      /System/Library/Frameworks/OpenGL.framework/Versions/A/OpenGL
0x0000000006634000      /System/Library/PrivateFrameworks/Sharing.framework/Versions/A/Sharing
0x0000000006634000      /System/Library/Frameworks/CoreGraphics.framework/Versions/A/CoreGraphics
0x0000000006634000      /System/Library/Frameworks/CoreImage.framework/Versions/A/CoreImage
0x0000000006634000      /System/Library/Frameworks/CoreText.framework/Versions/A/CoreText
0x0000000006634000      /System/Library/Frameworks/ImageIO.framework/Versions/A/ImageIO
0x0000000006634000      /System/Library/PrivateFrameworks/Backup.framework/Versions/A/Backup
0x0000000006634000      /usr/lib/libextension.dylib
0x0000000006634000      /usr/lib/libarchive.2.dylib
0x0000000006634000      /System/Library/Frameworks/CFNetwork.framework/Versions/A/CFNetwork
0x0000000006634000      /System/Library/Frameworks/SystemConfiguration.framework/Versions/A/SystemConfiguration
0x0000000006634000      /usr/lib/libCRFSuite.dylib
0x0000000006634000      /usr/lib/libc++.1.dylib
0x0000000006634000      /usr/lib/libc++abi.dylib
0x0000000006634000      /usr/lib/system/libcache.dylib
0x0000000006634000      /usr/lib/system/libcommonCrypto.dylib
0x0000000006634000      /usr/lib/system/libcompiler_rt.dylib
0x0000000006634000      /usr/lib/system/libcopyfile.dylib
0x0000000006634000      /usr/lib/system/libcorecrypto.dylib
0x0000000006634000      /usr/lib/system/libdispatch.dylib
0x0000000006634000      /usr/lib/system/libdyld.dylib
0x0000000006634000      /usr/lib/system/libkeymgr.dylib
0x0000000006634000      /usr/lib/system/liblaunch.dylib
0x0000000006634000      /usr/lib/system/libmacho.dylib
0x0000000006634000      /usr/lib/system/libquarantine.dylib
0x0000000006634000      /usr/lib/system/libremovefile.dylib
0x0000000006634000      /usr/lib/system/libsystem_asl.dylib
0x0000000006634000      /usr/lib/system/libsystem_blocks.dylib
0x0000000006634000      /usr/lib/system/libsystem_c.dylib
0x0000000006634000      /usr/lib/system/libsystem_configuration.dylib
0x0000000006634000      /usr/lib/system/libsystem_coreservices.dylib
0x0000000006634000      /usr/lib/system/libsystem_coretls.dylib
0x0000000006634000      /usr/lib/system/libsystem_dnssd.dylib
0x0000000006634000      /usr/lib/system/libsystem_info.dylib
0x0000000006634000      /usr/lib/system/libsystem_kernel.dylib
0x0000000006634000      /usr/lib/system/libsystem_m.dylib
0x0000000006634000      /usr/lib/system/libsystem_malloc.dylib
0x0000000006634000      /usr/lib/system/libsystem_network.dylib
0x0000000006634000      /usr/lib/system/libsystem_networkextension.dylib
0x0000000006634000      /usr/lib/system/libsystem_notify.dylib
0x0000000006634000      /usr/lib/system/libsystem_platform.dylib
0x0000000006634000      /usr/lib/system/libsystem_pthread.dylib
0x0000000006634000      /usr/lib/system/libsystem_sandbox.dylib
0x0000000006634000      /usr/lib/system/libsystem_secinit.dylib
0x0000000006634000      /usr/lib/system/libsystem_trace.dylib
0x0000000006634000      /usr/lib/system/libunc.dylib
0x0000000006634000      /usr/lib/system/libunwind.dylib
0x0000000006634000      /usr/lib/system/libxpc.dylib
0x0000000006634000      /usr/lib/libenergytrace.dylib
0x0000000006634000      /usr/lib/libbsm.0.dylib
0x0000000006634000      /usr/lib/system/libkxld.dylib
0x0000000006634000      /usr/lib/libxar.1.dylib
0x0000000006634000      /usr/lib/libsqlite3.dylib
0x0000000006634000      /usr/lib/libpam.2.dylib
0x0000000006634000      /usr/lib/libOpenScriptingUtil.dylib
0x0000000006634000      /usr/lib/libbz2.1.0.dylib
0x0000000006634000      /usr/lib/liblzma.5.dylib
0x0000000006634000      /System/Library/Frameworks/CoreServices.framework/Versions/A/Frameworks/FSEvents.framework/Versions/A/FSEvents
0x0000000006634000      /System/Library/Frameworks/CoreServices.framework/Versions/A/Frameworks/CarbonCore.framework/Versions/A/CarbonCore
0x0000000006634000      /System/Library/Frameworks/CoreServices.framework/Versions/A/Frameworks/Metadata.framework/Versions/A/Metadata
0x0000000006634000      /System/Library/Frameworks/CoreServices.framework/Versions/A/Frameworks/OSServices.framework/Versions/A/OSServices
0x0000000006634000      /System/Library/Frameworks/CoreServices.framework/Versions/A/Frameworks/SearchKit.framework/Versions/A/SearchKit
0x0000000006634000      /System/Library/Frameworks/CoreServices.framework/Versions/A/Frameworks/AE.framework/Versions/A/AE
0x0000000006634000      /System/Library/Frameworks/CoreServices.framework/Versions/A/Frameworks/LaunchServices.framework/Versions/A/LaunchServices
0x0000000006634000      /System/Library/Frameworks/CoreServices.framework/Versions/A/Frameworks/DictionaryServices.framework/Versions/A/DictionaryServices
0x0000000006634000      /System/Library/Frameworks/CoreServices.framework/Versions/A/Frameworks/SharedFileList.framework/Versions/A/SharedFileList
0x0000000006634000      /System/Library/Frameworks/NetFS.framework/Versions/A/NetFS
0x0000000006634000      /System/Library/PrivateFrameworks/NetAuth.framework/Versions/A/NetAuth
0x0000000006634000      /System/Library/PrivateFrameworks/login.framework/Versions/A/Frameworks/loginsupport.framework/Versions/A/loginsupport
0x0000000006634000      /System/Library/PrivateFrameworks/TCC.framework/Versions/A/TCC
0x0000000006634000      /usr/lib/libmecabra.dylib
0x0000000006634000      /System/Library/Frameworks/ApplicationServices.framework/Versions/A/Frameworks/ATS.framework/Versions/A/ATS
0x0000000006634000      /System/Library/Frameworks/ApplicationServices.framework/Versions/A/Frameworks/ColorSync.framework/Versions/A/ColorSync
0x0000000006634000      /System/Library/Frameworks/ApplicationServices.framework/Versions/A/Frameworks/HIServices.framework/Versions/A/HIServices
0x0000000006634000      /System/Library/Frameworks/ApplicationServices.framework/Versions/A/Frameworks/LangAnalysis.framework/Versions/A/LangAnalysis
0x0000000006634000      /System/Library/Frameworks/ApplicationServices.framework/Versions/A/Frameworks/PrintCore.framework/Versions/A/PrintCore
0x0000000006634000      /System/Library/Frameworks/ApplicationServices.framework/Versions/A/Frameworks/QD.framework/Versions/A/QD
0x0000000006634000      /System/Library/Frameworks/ApplicationServices.framework/Versions/A/Frameworks/SpeechSynthesis.framework/Versions/A/SpeechSynthesis
0x0000000006634000      /System/Library/Frameworks/Metal.framework/Versions/A/Metal
0x0000000006634000      /System/Library/Frameworks/Accelerate.framework/Versions/A/Frameworks/vImage.framework/Versions/A/vImage
0x0000000006634000      /System/Library/Frameworks/Accelerate.framework/Versions/A/Frameworks/vecLib.framework/Versions/A/vecLib
0x0000000006634000      /System/Library/Frameworks/Accelerate.framework/Versions/A/Frameworks/vecLib.framework/Versions/A/libvDSP.dylib
0x0000000006634000      /System/Library/Frameworks/Accelerate.framework/Versions/A/Frameworks/vecLib.framework/Versions/A/libvMisc.dylib
0x0000000006634000      /System/Library/Frameworks/Accelerate.framework/Versions/A/Frameworks/vecLib.framework/Versions/A/libLAPACK.dylib
0x0000000006634000      /System/Library/Frameworks/Accelerate.framework/Versions/A/Frameworks/vecLib.framework/Versions/A/libBLAS.dylib
0x0000000006634000      /System/Library/Frameworks/Accelerate.framework/Versions/A/Frameworks/vecLib.framework/Versions/A/libLinearAlgebra.dylib
0x0000000006634000      /System/Library/Frameworks/Accelerate.framework/Versions/A/Frameworks/vecLib.framework/Versions/A/libSparseBLAS.dylib
0x0000000006634000      /System/Library/PrivateFrameworks/GPUCompiler.framework/libmetal_timestamp.dylib
0x0000000006634000      /System/Library/Frameworks/OpenGL.framework/Versions/A/Libraries/libCoreFSCache.dylib
0x0000000006634000      /System/Library/PrivateFrameworks/IOAccelerator.framework/Versions/A/IOAccelerator
0x0000000006634000      /System/Library/Frameworks/ApplicationServices.framework/Versions/A/Frameworks/ATS.framework/Versions/A/Resources/libFontParser.dylib
0x0000000006634000      /System/Library/Frameworks/ApplicationServices.framework/Versions/A/Frameworks/ATS.framework/Versions/A/Resources/libFontRegistry.dylib
0x0000000006634000      /System/Library/PrivateFrameworks/AppleVPA.framework/Versions/A/AppleVPA
0x0000000006634000      /System/Library/PrivateFrameworks/AppleJPEG.framework/Versions/A/AppleJPEG
0x0000000006634000      /System/Library/Frameworks/ImageIO.framework/Versions/A/Resources/libJPEG.dylib
0x0000000006634000      /System/Library/Frameworks/ImageIO.framework/Versions/A/Resources/libTIFF.dylib
0x0000000006634000      /System/Library/Frameworks/ImageIO.framework/Versions/A/Resources/libPng.dylib
0x0000000006634000      /System/Library/Frameworks/ImageIO.framework/Versions/A/Resources/libGIF.dylib
0x0000000006634000      /System/Library/Frameworks/ImageIO.framework/Versions/A/Resources/libJP2.dylib
0x0000000006634000      /System/Library/Frameworks/ImageIO.framework/Versions/A/Resources/libRadiance.dylib
0x0000000006634000      /System/Library/Frameworks/CoreVideo.framework/Versions/A/CoreVideo
0x0000000006634000      /System/Library/Frameworks/OpenGL.framework/Versions/A/Libraries/libGLU.dylib
0x0000000006634000      /System/Library/Frameworks/OpenGL.framework/Versions/A/Libraries/libGFXShared.dylib
0x0000000006634000      /System/Library/Frameworks/OpenGL.framework/Versions/A/Libraries/libGL.dylib
0x0000000006634000      /System/Library/Frameworks/OpenGL.framework/Versions/A/Libraries/libGLImage.dylib
0x0000000006634000      /System/Library/Frameworks/OpenGL.framework/Versions/A/Libraries/libCVMSPluginSupport.dylib
0x0000000006634000      /System/Library/Frameworks/OpenGL.framework/Versions/A/Libraries/libCoreVMClient.dylib
0x0000000006634000      /usr/lib/libcompression.dylib
0x0000000006634000      /usr/lib/libcups.2.dylib
0x0000000006634000      /System/Library/Frameworks/Kerberos.framework/Versions/A/Kerberos
0x0000000006634000      /System/Library/Frameworks/GSS.framework/Versions/A/GSS
0x0000000006634000      /usr/lib/libresolv.9.dylib
0x0000000006634000      /usr/lib/libiconv.2.dylib
0x0000000006634000      /System/Library/PrivateFrameworks/Heimdal.framework/Versions/A/Heimdal
0x0000000006634000      /usr/lib/libheimdal-asn1.dylib
0x0000000006634000      /System/Library/Frameworks/OpenDirectory.framework/Versions/A/OpenDirectory
0x0000000006634000      /System/Library/PrivateFrameworks/CommonAuth.framework/Versions/A/CommonAuth
0x0000000006634000      /System/Library/Frameworks/OpenDirectory.framework/Versions/A/Frameworks/CFOpenDirectory.framework/Versions/A/CFOpenDirectory
0x0000000006634000      /System/Library/Frameworks/SecurityFoundation.framework/Versions/A/SecurityFoundation
0x0000000006634000      /System/Library/PrivateFrameworks/LanguageModeling.framework/Versions/A/LanguageModeling
0x0000000006634000      /usr/lib/libmarisa.dylib
0x0000000006634000      /usr/lib/libChineseTokenizer.dylib
0x0000000006634000      /usr/lib/libcmph.dylib
0x0000000006634000      /System/Library/Frameworks/ServiceManagement.framework/Versions/A/ServiceManagement
0x0000000006634000      /System/Library/Frameworks/ServiceManagement.framework/Versions/A/ServiceManagement
0x0000000006634000      /usr/lib/libxslt.1.dylib
0x0000000006634000      /System/Library/Frameworks/Carbon.framework/Versions/A/Frameworks/Ink.framework/Versions/A/Ink
0x0000000006634000      /usr/lib/libFosl_dynamic.dylib
0x0000000006634000      /System/Library/PrivateFrameworks/FaceCore.framework/Versions/A/FaceCore
0x0000000006634000      /System/Library/Frameworks/OpenCL.framework/Versions/A/OpenCL
0x0000000006634000      /System/Library/PrivateFrameworks/CrashReporterSupport.framework/Versions/A/CrashReporterSupport
0x0000000006634000      /System/Library/PrivateFrameworks/IconServices.framework/Versions/A/IconServices
0x0000000006634000      /System/Library/PrivateFrameworks/Apple80211.framework/Versions/A/Apple80211
0x0000000006634000      /System/Library/Frameworks/CoreWLAN.framework/Versions/A/CoreWLAN
0x0000000006634000      /System/Library/Frameworks/IOBluetooth.framework/Versions/A/IOBluetooth
0x0000000006634000      /System/Library/PrivateFrameworks/CoreWiFi.framework/Versions/A/CoreWiFi
0x0000000006634000      /System/Library/Frameworks/CoreBluetooth.framework/Versions/A/CoreBluetooth
0x0000000006634000      /System/Library/PrivateFrameworks/ChunkingLibrary.framework/Versions/A/ChunkingLibrary
0x0000000006634000      /System/Library/PrivateFrameworks/DebugSymbols.framework/Versions/A/DebugSymbols
0x0000000006634000      /System/Library/PrivateFrameworks/CoreSymbolication.framework/Versions/A/CoreSymbolication
0x0000000006634000      /System/Library/PrivateFrameworks/Symbolication.framework/Versions/A/Symbolication
0x0000000006634000      /System/Library/PrivateFrameworks/SpeechRecognitionCore.framework/Versions/A/SpeechRecognitionCore
0x0000000102000000      /Library/Java/JavaVirtualMachines/jdk1.7.0_80.jdk/Contents/Home/jre/lib/server/libjvm.dylib
0x0000000006634000      /usr/lib/libstdc++.6.dylib
0x0000000100e92000      /Library/Java/JavaVirtualMachines/jdk1.7.0_80.jdk/Contents/Home/jre/lib/libverify.dylib
0x0000000100e9f000      /Library/Java/JavaVirtualMachines/jdk1.7.0_80.jdk/Contents/Home/jre/lib/libjava.dylib
0x0000000100ed9000      /Library/Java/JavaVirtualMachines/jdk1.7.0_80.jdk/Contents/Home/jre/lib/libzip.dylib
0x0000000111201000      /System/Library/Frameworks/JavaVM.framework/Frameworks/JavaRuntimeSupport.framework/JavaRuntimeSupport
0x000000011121c000      /System/Library/Frameworks/JavaVM.framework/Versions/A/Frameworks/JavaNativeFoundation.framework/Versions/A/JavaNativeFoundation
0x0000000111231000      /System/Library/Frameworks/JavaVM.framework/Versions/A/JavaVM
0x0000000006634000      /System/Library/Frameworks/Carbon.framework/Versions/A/Carbon
0x0000000100ff4000      /System/Library/PrivateFrameworks/JavaLaunching.framework/Versions/A/JavaLaunching
0x0000000006634000      /System/Library/Frameworks/Carbon.framework/Versions/A/Frameworks/CommonPanels.framework/Versions/A/CommonPanels
0x0000000006634000      /System/Library/Frameworks/Carbon.framework/Versions/A/Frameworks/Help.framework/Versions/A/Help
0x0000000006634000      /System/Library/Frameworks/Carbon.framework/Versions/A/Frameworks/ImageCapture.framework/Versions/A/ImageCapture
0x0000000006634000      /System/Library/Frameworks/Carbon.framework/Versions/A/Frameworks/OpenScripting.framework/Versions/A/OpenScripting
0x0000000006634000      /System/Library/Frameworks/Carbon.framework/Versions/A/Frameworks/Print.framework/Versions/A/Print
0x0000000006634000      /System/Library/Frameworks/Carbon.framework/Versions/A/Frameworks/SecurityHI.framework/Versions/A/SecurityHI
0x0000000112b99000      /Users/aretter/code/rocksdb/java/target/librocksdbjni-osx.jnilib
0x0000000113150000      /usr/local/opt/snappy/lib/libsnappy.1.dylib
0x0000000113159000      /usr/local/opt/lz4/lib/liblz4.1.dylib
0x00000001140c3000      /Library/Java/JavaVirtualMachines/jdk1.7.0_80.jdk/Contents/Home/jre/lib/libnio.dylib
0x00000001140d2000      /Library/Java/JavaVirtualMachines/jdk1.7.0_80.jdk/Contents/Home/jre/lib/./libnet.dylib

VM Arguments:
jvm_args: -ea -Xcheck:jni -Djava.library.path=target 
java_command: org.rocksdb.test.RocksJunitRunner org.rocksdb.BackupableDBOptionsTest org.rocksdb.BackupEngineTest org.rocksdb.BlockBasedTableConfigTest org.rocksdb.util.BytewiseComparatorTest org.rocksdb.CheckPointTest org.rocksdb.ColumnFamilyOptionsTest org.rocksdb.ColumnFamilyTest org.rocksdb.ComparatorOptionsTest org.rocksdb.ComparatorTest org.rocksdb.CompressionOptionsTest org.rocksdb.DBOptionsTest org.rocksdb.DirectComparatorTest org.rocksdb.DirectSliceTest org.rocksdb.util.EnvironmentTest org.rocksdb.FilterTest org.rocksdb.FlushTest org.rocksdb.InfoLogLevelTest org.rocksdb.KeyMayExistTest org.rocksdb.LoggerTest org.rocksdb.MemTableTest org.rocksdb.MergeTest org.rocksdb.MixedOptionsTest org.rocksdb.NativeLibraryLoaderTest org.rocksdb.OptionsTest org.rocksdb.PlainTableConfigTest org.rocksdb.ReadOnlyTest org.rocksdb.ReadOptionsTest org.rocksdb.RocksDBTest org.rocksdb.RocksEnvTest org.rocksdb.RocksIteratorTest org.rocksdb.RocksMemEnvTest org.rocksdb.util.SizeUnitTest org.rocksdb.SliceTest org.rocksdb.SnapshotTest org.rocksdb.TransactionLogIteratorTest org.rocksdb.TtlDBTest org.rocksdb.StatisticsCollectorTest org.rocksdb.WriteBatchHandlerTest org.rocksdb.WriteBatchTest org.rocksdb.WriteBatchThreadedTest org.rocksdb.WriteOptionsTest org.rocksdb.WriteBatchWithIndexTest
Launcher Type: SUN_STANDARD

Environment Variables:
JAVA_HOME=/Library/Java/JavaVirtualMachines/jdk1.7.0_80.jdk/Contents/Home
PATH=/opt/local/bin:/opt/local/sbin:/usr/local/bin:/usr/local/scala/bin:/usr/local/phabricator/arcanist/bin:/Users/aretter/Library/Android/sdk/ndk-bundle:/opt/local/bin:/opt/local/sbin:/usr/local/bin:/usr/bin:/bin:/usr/sbin:/sbin:/opt/X11/bin:/usr/local/share/dotnet:/Library/TeX/texbin
SHELL=/bin/bash
DISPLAY=/private/tmp/com.apple.launchd.1OqLopSBFQ/org.macosforge.xquartz:0

Signal Handlers:
SIGSEGV: [libjvm.dylib+0x52e8c9], sa_mask[0]=0xfffefeff, sa_flags=0x00000043
SIGBUS: [libjvm.dylib+0x52e8c9], sa_mask[0]=0xfffefeff, sa_flags=0x00000042
SIGFPE: [libjvm.dylib+0x42022e], sa_mask[0]=0xfffefeff, sa_flags=0x00000042
SIGPIPE: [libjvm.dylib+0x42022e], sa_mask[0]=0xfffefeff, sa_flags=0x00000042
SIGXFSZ: [libjvm.dylib+0x42022e], sa_mask[0]=0xfffefeff, sa_flags=0x00000042
SIGILL: [libjvm.dylib+0x42022e], sa_mask[0]=0xfffefeff, sa_flags=0x00000042
SIGUSR1: SIG_DFL, sa_mask[0]=0x00000000, sa_flags=0x00000000
SIGUSR2: [libjvm.dylib+0x41fd20], sa_mask[0]=0x00000000, sa_flags=0x00000042
SIGHUP: [libjvm.dylib+0x41dfd5], sa_mask[0]=0xfffefeff, sa_flags=0x00000042
SIGINT: [libjvm.dylib+0x41dfd5], sa_mask[0]=0xfffefeff, sa_flags=0x00000042
SIGTERM: [libjvm.dylib+0x41dfd5], sa_mask[0]=0xfffefeff, sa_flags=0x00000042
SIGQUIT: [libjvm.dylib+0x41dfd5], sa_mask[0]=0xfffefeff, sa_flags=0x00000042


---------------  S Y S T E M  ---------------

OS:Bsduname:Darwin 15.6.0 Darwin Kernel Version 15.6.0: Thu Jun 23 18:25:34 PDT 2016; root:xnu-3248.60.10~1/RELEASE_X86_64 x86_64
rlimit: STACK 65532k, CORE 0k, NPROC 709, NOFILE 10240, AS infinity
load average:3.39 2.99 2.70

CPU:total 8 (4 cores per cpu, 2 threads per core) family 6 model 70 stepping 1, cmov, cx8, fxsr, mmx, sse, sse2, sse3, ssse3, sse4.1, sse4.2, popcnt, avx, avx2, aes, erms, ht, tsc, tscinvbit

Memory: 4k page, physical 16777216k(4194304k free)

/proc/meminfo:


vm_info: Java HotSpot(TM) 64-Bit Server VM (24.80-b11) for bsd-amd64 JRE (1.7.0_80-b15), built on Apr 10 2015 11:25:43 by "java_re" with gcc 4.2.1 (Based on Apple Inc. build 5658) (LLVM build 2336.11.00)

time: Sun Jul 24 17:19:34 2016
elapsed time: 0 seconds
```

** TODO **

# ASAN
[ASAN](https://github.com/google/sanitizers/wiki/AddressSanitizer) (Google Address Sanitizer) attempts to detect a whole range of memory and range issues and can be compiled into your code, at runtime, it will report some memory or buffer-range violations.

## Mac (Apple LLVM 7.3.0)
1. Set JDK 7 as required by RocksJava
    ```bash
    export JAVA_HOME=/Library/Java/JavaVirtualMachines/jdk1.7.0_80.jdk/Contents/Home
    ```

2. Ensure a clean start:
    ```bash
    make clean jclean
    ```

3. Compile the Java test suite with ASAN compiled in:
    ```bash
    DEBUG_LEVEL=2 COMPILE_WITH_ASAN=true make jtest_compile
    ```

4. Execute the entire Java Test Suite:
    ```bash
    make jtest_run
    ```

   or for a single test (e.g. `ComparatorTest`), execute:

    ```bash
    cd java
    java -ea -Xcheck:jni -Djava.library.path=target -cp "target/classes:target/test-classes:test-libs/junit-4.12.jar:test-libs/hamcrest-core-1.3.jar:test-libs/mockito-all-1.10.19.jar:test-libs/cglib-2.2.2.jar:test-libs/assertj-core-1.7.1.jar:target/*" org.rocksdb.test.RocksJunitRunner org.rocksdb.ComparatorTest```
    ```

If ASAN detects an issue, you will see output similar to the following:
```bash
Run: org.rocksdb.BackupableDBOptionsTest testing now -> destroyOldData 
Run: org.rocksdb.BackupEngineTest testing now -> deleteBackup 
=================================================================
==80632==ERROR: AddressSanitizer: unknown-crash on address 0x7fd93940d6e8 at pc 0x00011cebe075 bp 0x70000020ffe0 sp 0x70000020ffd8
WRITE of size 8 at 0x7fd93940d6e8 thread T0
    #0 0x11cebe074 in rocksdb::PosixLogger::PosixLogger(__sFILE*, unsigned long long (*)(), rocksdb::Env*, rocksdb::InfoLogLevel) posix_logger.h:47
    #1 0x11cebc847 in rocksdb::PosixLogger::PosixLogger(__sFILE*, unsigned long long (*)(), rocksdb::Env*, rocksdb::InfoLogLevel) posix_logger.h:53
    #2 0x11ce9888c in rocksdb::(anonymous namespace)::PosixEnv::NewLogger(std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&, std::__1::shared_ptr<rocksdb::Logger>*) env_posix.cc:574
    #3 0x11c09a3e3 in rocksdb::CreateLoggerFromOptions(std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&, rocksdb::DBOptions const&, std::__1::shared_ptr<rocksdb::Logger>*) auto_roll_logger.cc:166
    #4 0x11c3a8a55 in rocksdb::SanitizeOptions(std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&, rocksdb::DBOptions const&) db_impl.cc:143
    #5 0x11c3ac2f3 in rocksdb::DBImpl::DBImpl(rocksdb::DBOptions const&, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&) db_impl.cc:307
    #6 0x11c3b38b4 in rocksdb::DBImpl::DBImpl(rocksdb::DBOptions const&, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&) db_impl.cc:350
    #7 0x11c4497bc in rocksdb::DB::Open(rocksdb::DBOptions const&, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&, std::__1::vector<rocksdb::ColumnFamilyDescriptor, std::__1::allocator<rocksdb::ColumnFamilyDescriptor> > const&, std::__1::vector<rocksdb::ColumnFamilyHandle*, std::__1::allocator<rocksdb::ColumnFamilyHandle*> >*, rocksdb::DB**) db_impl.cc:5665
    #8 0x11c447b74 in rocksdb::DB::Open(rocksdb::Options const&, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&, rocksdb::DB**) db_impl.cc:5633
    #9 0x11bff8ca4 in rocksdb::Status std::__1::__invoke_void_return_wrapper<rocksdb::Status>::__call<rocksdb::Status (*&)(rocksdb::Options const&, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&, rocksdb::DB**), rocksdb::Options const&, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&, rocksdb::DB**>(rocksdb::Status (*&&&)(rocksdb::Options const&, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&, rocksdb::DB**), rocksdb::Options const&&&, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&&&, rocksdb::DB**&&) __functional_base:437
    #10 0x11bff89ff in std::__1::__function::__func<rocksdb::Status (*)(rocksdb::Options const&, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&, rocksdb::DB**), std::__1::allocator<rocksdb::Status (*)(rocksdb::Options const&, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&, rocksdb::DB**)>, rocksdb::Status (rocksdb::Options const&, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&, rocksdb::DB**)>::operator()(rocksdb::Options const&, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&, rocksdb::DB**&&) functional:1437
    #11 0x11bff269b in std::__1::function<rocksdb::Status (rocksdb::Options const&, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&, rocksdb::DB**)>::operator()(rocksdb::Options const&, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&, rocksdb::DB**) const functional:1817
    #12 0x11bfd6edb in rocksdb_open_helper(JNIEnv_*, long, _jstring*, std::__1::function<rocksdb::Status (rocksdb::Options const&, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&, rocksdb::DB**)>) rocksjni.cc:37
    #13 0x11bfd723e in Java_org_rocksdb_RocksDB_open__JLjava_lang_String_2 rocksjni.cc:55
    #14 0x10be77757  (<unknown module>)
    #15 0x10be6b174  (<unknown module>)
    #16 0x10be6b232  (<unknown module>)
    #17 0x10be654e6  (<unknown module>)
    #18 0x10b6dc897 in JavaCalls::call_helper(JavaValue*, methodHandle*, JavaCallArguments*, Thread*) (libjvm.dylib+0x2dc897)
    #19 0x10b6dc667 in JavaCalls::call(JavaValue*, methodHandle, JavaCallArguments*, Thread*) (libjvm.dylib+0x2dc667)
    #20 0x10b868427 in Reflection::invoke(instanceKlassHandle, methodHandle, Handle, bool, objArrayHandle, BasicType, objArrayHandle, bool, Thread*) (libjvm.dylib+0x468427)
    #21 0x10b86888d in Reflection::invoke_method(oopDesc*, Handle, objArrayHandle, Thread*) (libjvm.dylib+0x46888d)
    #22 0x10b729246 in JVM_InvokeMethod (libjvm.dylib+0x329246)
    #23 0x10be77757  (<unknown module>)
    #24 0x10be6b232  (<unknown module>)
    #25 0x10be6b232  (<unknown module>)
    #26 0x10be6b8e0  (<unknown module>)
    #27 0x10be6b232  (<unknown module>)
    #28 0x10be6b232  (<unknown module>)
    #29 0x10be6b232  (<unknown module>)
    #30 0x10be6b232  (<unknown module>)
    #31 0x10be6b057  (<unknown module>)
    #32 0x10be6b057  (<unknown module>)
    #33 0x10be6b057  (<unknown module>)
    #34 0x10be6b057  (<unknown module>)
    #35 0x10be6b057  (<unknown module>)
    #36 0x10be6b057  (<unknown module>)
    #37 0x10be6b057  (<unknown module>)
    #38 0x10be6b705  (<unknown module>)
    #39 0x10be6b705  (<unknown module>)
    #40 0x10be6b057  (<unknown module>)
    #41 0x10be6b057  (<unknown module>)
    #42 0x10be6b057  (<unknown module>)
    #43 0x10be6b057  (<unknown module>)
    #44 0x10be6b057  (<unknown module>)
    #45 0x10be6b057  (<unknown module>)
    #46 0x10be6b057  (<unknown module>)
    #47 0x10be6b057  (<unknown module>)
    #48 0x10be6b705  (<unknown module>)
    #49 0x10be6b705  (<unknown module>)
    #50 0x10be6b057  (<unknown module>)
    #51 0x10be6b057  (<unknown module>)
    #52 0x10be6b057  (<unknown module>)
    #53 0x10be6b057  (<unknown module>)
    #54 0x10be6b232  (<unknown module>)
    #55 0x10be6b232  (<unknown module>)
    #56 0x10be6b232  (<unknown module>)
    #57 0x10be6b232  (<unknown module>)
    #58 0x10be654e6  (<unknown module>)
    #59 0x10b6dc897 in JavaCalls::call_helper(JavaValue*, methodHandle*, JavaCallArguments*, Thread*) (libjvm.dylib+0x2dc897)
    #60 0x10b6dc667 in JavaCalls::call(JavaValue*, methodHandle, JavaCallArguments*, Thread*) (libjvm.dylib+0x2dc667)
    #61 0x10b71004d in jni_invoke_static(JNIEnv_*, JavaValue*, _jobject*, JNICallType, _jmethodID*, JNI_ArgumentPusher*, Thread*) (libjvm.dylib+0x31004d)
    #62 0x10b7092d4 in jni_CallStaticVoidMethodV (libjvm.dylib+0x3092d4)
    #63 0x10b71c28d in checked_jni_CallStaticVoidMethod (libjvm.dylib+0x31c28d)
    #64 0x109fdd0fd in JavaMain (java+0x1000030fd)
    #65 0x7fff8df9c99c in _pthread_body (libsystem_pthread.dylib+0x399c)
    #66 0x7fff8df9c919 in _pthread_start (libsystem_pthread.dylib+0x3919)
    #67 0x7fff8df9a350 in thread_start (libsystem_pthread.dylib+0x1350)

AddressSanitizer can not describe address in more detail (wild memory access suspected).
SUMMARY: AddressSanitizer: unknown-crash posix_logger.h:47 in rocksdb::PosixLogger::PosixLogger(__sFILE*, unsigned long long (*)(), rocksdb::Env*, rocksdb::InfoLogLevel)
Shadow bytes around the buggy address:
  0x1ffb27281a80: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
  0x1ffb27281a90: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
  0x1ffb27281aa0: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
  0x1ffb27281ab0: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
  0x1ffb27281ac0: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
=>0x1ffb27281ad0: 00 00 00 00 00 00 00 00 00 04 00 00 00[04]00 00
  0x1ffb27281ae0: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
  0x1ffb27281af0: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
  0x1ffb27281b00: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
  0x1ffb27281b10: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
  0x1ffb27281b20: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
Shadow byte legend (one shadow byte represents 8 application bytes):
  Addressable:           00
  Partially addressable: 01 02 03 04 05 06 07 
  Heap left redzone:       fa
  Heap right redzone:      fb
  Freed heap region:       fd
  Stack left redzone:      f1
  Stack mid redzone:       f2
  Stack right redzone:     f3
  Stack partial redzone:   f4
  Stack after return:      f5
  Stack use after scope:   f8
  Global redzone:          f9
  Global init order:       f6
  Poisoned by user:        f7
  Container overflow:      fc
  Array cookie:            ac
  Intra object redzone:    bb
  ASan internal:           fe
  Left alloca redzone:     ca
  Right alloca redzone:    cb
==80632==ABORTING
make[1]: *** [run_test] Abort trap: 6
make: *** [jtest_run] Error 2

```


The output from ASAN shows a stack-trace with file names and line numbers of our C++ code that led to the issue, hopefully this helps shed some light on where the issue is occurring and perhaps why.

Unfortunately all of those `(<unknown module>)` are execution paths inside the JVM, ASAN cannot discover them because the JVM we are using was not itself build with support for ASAN. We could attempt to build our own JVM from the OpenJDK project and include ASAN, but at the moment that process for Mac OS X seems to be broken: https://github.com/hgomez/obuildfactory/issues/51.


** TODO ** Note the path of the DSO for libasan on Mac OS X: `/Library/Developer/CommandLineTools/usr/lib/clang/7.3.0/lib/darwin/libclang_rt.asan_osx_dynamic.dylib`


## Linux (CentOS 7) (GCC 4.8.5)
1. Set JDK 7 as required by RocksJava
    ```bash
    export JAVA_HOME="/usr/lib/jvm/java-1.7.0-openjdk"
    export PATH="${PATH}:${JAVA_HOME}/bin"
    ```
   You might also need to run `sudo alternatives --config java` and select OpenJDK 7.

2. Perform a clean build where ASAN is compiled in (from your RocksDB folder):
    ```bash
    make clean jclean
    DEBUG_LEVEL=2 COMPILE_WITH_ASAN=true make rocksdbjava
    ```
***TODO***