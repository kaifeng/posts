---
layout: post
title: opencit 2.2编译指导
author: kaifeng
date: 2020-03-17
categories: tpm
---

opencit以java代码为主，混合了少量非java代码（openssl，tpm-tools，trousers等C源码编译，版本比较旧），使用ant和maven作为工程管理工具。
最初应该是使用的ant，后来加入了maven，并以ant驱动maven。
编译opencit参照官方文档即可成功（https://github.com/opencit/opencit/wiki/Open-CIT-2.2-Product-Guide 3.2和3.3节），本文以容器中编译为例，使用ubuntu:14.04镜像，使用虚机也是一样的。

踩过的坑在此记录一下，以助来者。

# 0.0 整体步骤

```
 # cd /root/opencit-external-artifacts
 # git checkout release-cit-2.2
 # ant
 # cd ..
 # cd /root/opencit-util
 # git checkout release-cit-2.2
 # ant
 # cd ..
 # cd /root/opencit-contrib
 # git checkout release-cit-2.2
 # ant
 # cd ..
 # cd /root/opencit
 # git checkout release-cit-2.2
 # ant
```

Binary Locations

Attestation Server Binary:
/root/opencit/installers/mtwilson-server/target/mtwilson-server-2.2-SNAPSHOT-jdk_glassfish_monit.bin

Trust Agent Binary:
/root/opencit/installers/mtwilson-trustagent/target/mtwilson-trustagent-2.2-SNAPSHOT.bin

Windows Trust Agent Binary:
/root/opencit/packages/trustagent-windows-installer/target/mtwilson-trustagent-windows-installer-2.2-SNAPSHOT.exe

TPM Patched Tools Binary:
/root/opencit-contrib/cit-patched-tpm-tools/target


## 0.1 maven代理配置

maven不使用http_proxy这样的环境变量，有自己的代理配置，内网环境需要配置。

如果按手册搭建环境，其全局配置文件在/usr/local/apache-maven-3.3.1/conf/settings.xml

编辑``<proxies>``下面的字段。


## 0.2 maven连接repo报protocol version错的解决

这是java7的bug，需要在MAVEN_OPTS中加入 ``-Dhttps.protocols=TLSv1.2`` 参数


# 1. 编译外部依赖库


要求的tomcat，glassfish，vijava，jdk的对应版本目前还可以下载到。
tpm-tools 1.3.8也可以下载到，但这个版本比较旧，最终应该会替换成更高新的版本。
monit 5.5已经下载不到了，能下载到的最低版本是5.9，因此只能使用5.9版本来编译。

所需要的包如下
```
apache-maven-3.3.1-bin.zip
apache-tomcat-7.0.34.tar.gz
glassfish-4.0.zip
monit-5.9.tar.gz
tpm-tools-1.3.8.tar.gz
vijava55b20130927.zip
jdk-7u51-linux-x64.tar.gz
```

代码就自己去clone github

编译opencit-external-artifacts/monit/pom.xml，将5.5替换为5.9

```
root@0f32ef4ebbb5:~/opencit-external-artifacts# git diff monit/pom.xml
diff --git a/monit/pom.xml b/monit/pom.xml
index f7a0932..50a5e96 100644
--- a/monit/pom.xml
+++ b/monit/pom.xml
@@ -3,9 +3,9 @@

     <groupId>com.mmonit</groupId>
     <artifactId>monit</artifactId>
-    <version>5.5</version>
+    <version>5.9</version>

-    <name>Monit 5.5</name>
+    <name>Monit 5.9</name>
     <packaging>pom</packaging>

     <properties>

@@ -14,7 +14,7 @@

         <version>${project.version}</version>
         <classifier>linux-src</classifier>
         <packaging>tgz</packaging>
-        <file>monit-5.5-linux-src.tgz</file>
+        <file>monit-5.9-linux-src.tgz</file>
     </properties>
     <build>

root@0f32ef4ebbb5:~/opencit-external-artifacts#
```

执行ant编译，应该可以成功。


# 2. 编译opencit-util库

需要升级一个依赖再执行编译，因为jotm-core旧版本依赖的jarcob已经没有对应的版本了，maven取不到。

```
root@0f32ef4ebbb5:~/opencit-util# git diff
diff --git a/util/mtwilson-util-jpa/pom.xml b/util/mtwilson-util-jpa/pom.xml
index 40bd897..9c388a1 100644
--- a/util/mtwilson-util-jpa/pom.xml
+++ b/util/mtwilson-util-jpa/pom.xml
@@ -128,7 +128,7 @@
         <dependency>
             <groupId>org.ow2.jotm</groupId>
             <artifactId>jotm-core</artifactId>
-            <version>2.2.3</version>
+            <version>2.3.1-M1</version>
         </dependency>
         <dependency>
             <groupId>jaxen</groupId>

@@ -137,4 +137,4 @@

         </dependency>
     </dependencies>
-</project>
\ No newline at end of file
+</project>

root@0f32ef4ebbb5:~/opencit-util#
```

# 3. 编译opencit-contrib

这里主要是c代码编译，没有遇到问题。


# 4. 编译opencit

需要先修改配置，主要是将monit的版本依赖改为5.9

```
root@0f32ef4ebbb5:~/opencit# git diff installers/MonitLinuxInstaller/pom.xml
diff --git a/installers/MonitLinuxInstaller/pom.xml b/installers/MonitLinuxInstaller/pom.xml
index 4f5ab14..8cc5b3e 100644
--- a/installers/MonitLinuxInstaller/pom.xml
+++ b/installers/MonitLinuxInstaller/pom.xml
@@ -3,7 +3,7 @@

     <groupId>com.intel.mtwilson.linux</groupId>
     <artifactId>monit</artifactId>
-    <version>5.5</version>
+    <version>5.9</version>^M

     <packaging>pom</packaging>
     <name>Monit Linux Installer</name>

@@ -45,7 +45,7 @@

                 <dependency>
                     <groupId>com.mmonit</groupId>
                     <artifactId>monit</artifactId>
-                    <version>5.5</version>
+                    <version>5.9</version>^M
                     <type>tgz</type>
                     <classifier>linux-src</classifier>
                 </dependency>

@@ -78,11 +78,11 @@

                                         <artifactItem>
                                             <groupId>com.mmonit</groupId>
                                             <artifactId>monit</artifactId>
-                                            <version>5.5</version>
+                                            <version>5.9</version>^M
                                             <type>tgz</type>
                                             <classifier>linux-src</classifier>
                                             <outputDirectory>${makeself.directory}</outputDirectory>
-                                            <destFileName>monit-5.5-linux-src.tar.gz</destFileName>
+                                            <destFileName>monit-5.9-linux-src.tar.gz</destFileName>^M
                                         </artifactItem>
                                     </artifactItems>
                                     <overWriteReleases>false</overWriteReleases>

root@0f32ef4ebbb5:~/opencit#
```

```
root@0f32ef4ebbb5:~/opencit# git diff installers/mtwilson-server/pom.xml
diff --git a/installers/mtwilson-server/pom.xml b/installers/mtwilson-server/pom.xml
index a1864bf..8c3ef39 100644
--- a/installers/mtwilson-server/pom.xml
+++ b/installers/mtwilson-server/pom.xml
@@ -61,7 +61,7 @@
                 <dependency>
                     <groupId>com.intel.mtwilson.linux</groupId>
                     <artifactId>monit</artifactId>
-                    <version>5.5</version>
+                    <version>5.9</version>^M
                     <type>bin</type>
                 </dependency>
                 <dependency>

@@ -275,7 +275,7 @@

                                         <artifactItem>
                                             <groupId>com.intel.mtwilson.linux</groupId>
                                             <artifactId>monit</artifactId>
-                                            <version>5.5</version>
+                                            <version>5.9</version>^M
                                             <type>bin</type>
                                         </artifactItem>
                                         <!-- Web services -->

root@0f32ef4ebbb5:~/opencit#
```

这个修改是解决编译javadoc时报错的问题，规避maven的问题（bug？不知道）

```
root@0f32ef4ebbb5:~/opencit# git diff integration/mtwilson-client-java7/pom.xml
diff --git a/integration/mtwilson-client-java7/pom.xml b/integration/mtwilson-client-java7/pom.xml
index 70607d6..19e17bb 100644
--- a/integration/mtwilson-client-java7/pom.xml
+++ b/integration/mtwilson-client-java7/pom.xml
@@ -96,6 +96,17 @@

                     </systemProperties>
                 </configuration>
             </plugin>
+            <!-- ADDED BY ME -->
+            <plugin>^M
+                <groupId>org.apache.maven.plugins</groupId>^M
+                <artifactId>maven-site-plugin</artifactId>^M
+                <version>3.3</version>
+            </plugin>^M
+            <plugin>^M
+                <groupId>org.apache.maven.plugins</groupId>^M
+                <artifactId>maven-project-info-reports-plugin</artifactId>^M
+                <version>2.1.2</version>
+            </plugin>^M
         </plugins>
     </build>

root@0f32ef4ebbb5:~/opencit#
```

打印出这些就表示编译成功了，可以从手册中列出的位置取得版本。

```
     [exec] [INFO] ------------------------------------------------------------------------
     [exec] [INFO] Reactor Summary:
     [exec] [INFO]
     [exec] [INFO] mtwilson-server-zip ................................ SUCCESS [  8.958 s]
     [exec] [INFO] mtwilson-packages .................................. SUCCESS [  0.018 s]
     [exec] [INFO] Trust Agent Windows Installer ...................... SUCCESS [ 27.036 s]
     [exec] [INFO] ------------------------------------------------------------------------
     [exec] [INFO] BUILD SUCCESS
     [exec] [INFO] ------------------------------------------------------------------------
     [exec] [INFO] Total time: 36.550 s
     [exec] [INFO] Finished at: 2020-03-17T02:41:09+00:00
     [exec] [INFO] Final Memory: 32M/342M
     [exec] [INFO] ------------------------------------------------------------------------


all:
BUILD SUCCESSFUL
Total time: 40 minutes 16 seconds

root@0f32ef4ebbb5:~/opencit#
```
