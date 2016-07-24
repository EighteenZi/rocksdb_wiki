If you are a Java developer working with JNI code, debugging it can be particularly hard, for example if you are experiencing an unexpected SIGSEGV and don't know why.

There are several techniques which we can use to try and help get to the bottom of these:

# Interpreting hs_err_pid files
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
make[1]: *** [test] Abort trap: 6
make: *** [jtest] Error 2
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