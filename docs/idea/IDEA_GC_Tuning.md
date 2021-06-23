# idea GC 调优

> 方案来自网络



## 1

```
-server
-XX:MetaspaceSize=128M
-XX:MaxMetaspaceSize=512M
-XX:+AlwaysPreTouch
-Xms128m
-Xmx512M
-XX:ReservedCodeCacheSize=512m
-XX:+UseG1GC
-XX:+UseStringDeduplication
-XX:AutoBoxCacheMax=20000
-ea
-Dsun.io.useCanonCaches=false
-Dsun.awt.keepWorkingSetOnMinimize=true
-Djava.net.preferIPv4Stack=true
-Djdk.http.auth.tunneling.disabledSchemes=""
-Djsse.enablesSNIExtension=false
-XX:+HeapDumpOnOutOfMemoryError
-XX:-OmitStackTraceInFastThrow
-Dfile.encoding=UTF-8
-Duser.name=tshit11
```

## 2

>https://gist.github.com/EdwardBeckett/4bbb1bcaad3f900e7305


```
-server
-Xms2g
-Xmx2g
-Xss16m
-XX:+UseConcMarkSweepGC
-XX:+CMSParallelRemarkEnabled
-XX:ConcGCThreads=4
-XX:ReservedCodeCacheSize=128m
-XX:+AlwaysPreTouch
-XX:+TieredCompilation
-XX:+UseCompressedOops
-XX:SoftRefLRUPolicyMSPerMB=50
-Dsun.io.useCanonCaches=false
-Djava.net.preferIPv4Stack=true
-Djsse.enableSNIExtension=false
-ea
```



## 3

> https://gist.github.com/judepereira/5f4276fea45f2e0237e90ba5e4319f61

```
-Xmx2048m
-Xss256k
-XX:+UseG1GC
-XX:InitiatingHeapOccupancyPercent=65
-XX:G1HeapRegionSize=2
-XX:MaxGCPauseMillis=100
```



## 4

> https://awesomeopensource.com/project/FoxxMD/intellij-jvm-options-explained

```
-server
-ea
-Xmx[memoryValue, If total memory < 2GB then at least 1/4 total memory. If > 2GB then 1-4 GB. (See note below)]
-Xms[memoryValue, at least 1/2 Xmx. Can be = to Xmx]
-XX:+UseG1GC
-XX:ReservedCodeCacheSize=[between 128m and 256m, depending on how much free physical memory you have available]
-XX:+OmitStackTraceInFastThrow
-Dsun.io.useCanonCaches=false
-XX:SoftRefLRUPolicyMSPerMB=50
```



## 5

> https://medium.com/stochastic-stories/tuning-my-intellij-ide-8255781f6a0d

```
-Xms1G
-XX:ReservedCodeCacheSize=480m
-XX:+UseCompressedOops
-Dfile.encoding=UTF-8
-XX:+UseG1GC
-XX:MaxGCPauseMillis=100
-XX:+PerfDisableSharedMem
-XX:SoftRefLRUPolicyMSPerMB=50
-ea
-Dsun.io.useCanonCaches=false
-Djava.net.preferIPv4Stack=true
-XX:+HeapDumpOnOutOfMemoryError
-XX:-OmitStackTraceInFastThrow
-Xverify:none

-XX:ErrorFile=$USER_HOME/java_error_in_idea_%p.log
-XX:HeapDumpPath=$USER_HOME/java_error_in_idea.hprof
-Xbootclasspath/a:../lib/boot.jar
```



## 6

> https://www.cnblogs.com/nevermorewang/p/10061377.html

```
-Xms512m
-Xmn512m
-Xmx2048m
-XX:ReservedCodeCacheSize=240m
-XX:+UseCompressedOops
-Dfile.encoding=UTF-8
-XX:+UseG1GC   //使用G1收集器，好处是并行收集
-XX:+UseNUMA  //优先使用速度较快的内存
-XX:SoftRefLRUPolicyMSPerMB=50
-ea
-Dsun.io.useCanonCaches=false
-Djava.net.preferIPv4Stack=true
-Djdk.http.auth.tunneling.disabledSchemes=""
-XX:+HeapDumpOnOutOfMemoryError
-XX:-OmitStackTraceInFastThrow
-Xverify:none
 
-XX:ErrorFile=$USER_HOME/java_error_in_idea_%p.log
-XX:HeapDumpPath=$USER_HOME/java_error_in_idea.hprof
```



## 7

> 官方配置，用来恢复

```
-Xms1024m
-Xmx2048m
-XX:ReservedCodeCacheSize=300m
-XX:+UseConcMarkSweepGC
-XX:SoftRefLRUPolicyMSPerMB=50
-ea
-XX:CICompilerCount=2
-Dsun.io.useCanonPrefixCache=false
-Djdk.http.auth.tunneling.disabledSchemes=""
-XX:+HeapDumpOnOutOfMemoryError
-XX:-OmitStackTraceInFastThrow
-Djdk.attach.allowAttachSelf=true
-Dkotlinx.coroutines.debug=off
-Djdk.module.illegalAccess.silent=true
#
```

