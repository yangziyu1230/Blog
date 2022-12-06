---
title: 编译JDK
date: 2022-12-06 09:59:19
tags: jvm
categories: jvm
---

{% note::本内容由[《深入理解JVM虚拟机》](https://book.douban.com/subject/6522893/)和笔者自己的实践所得，有条件建议直接看书。阅读本内容需要亿点点c++及编译原理基础。 %}

## 环境信息

编译环境使用VMware Workstation创建的虚拟机，配置为16C24G，具体信息如下：

<table border=1>
    <tr>
        <td>操作系统</td>
        <td>Ubuntu 18.04.6 LTS</td>
    </tr>
    <tr>
        <td>内核版本</td>
        <td>4.15.0-200-generic</td>
    </tr>
    <tr>
        <td>gcc version</td>
        <td>7.5.0</td>
    </tr>
</table>

## 依赖安装

```bash
sudo apt install aptitude
sudo aptitude install build-essential 
sudo aptitude install libfreetype6-dev 
sudo aptitude install libcups2-dev
sudo aptitude install libx11-dev libxext-dev libxrender-dev libxrandr-dev libxt-dev 
sudo aptitude install libasound2-dev 
sudo aptitude install libffi-dev
sudo aptitude install autoconf
sudo aptitude install unzip 
sudo aptitude install zip
sudo aptitude install libx11-dev libxext-dev libxrender-dev libxrandr-dev libxtst-dev libxt-dev
sudo aptitude install libfontconfig1-dev
sudo aptitude install gdb
```

## 编译

jdk12源代码从github获取，链接地址如下：[jdk12-ga](https://codeload.github.com/openjdk/jdk/zip/refs/tags/jdk-12-ga)，请自行想办法下载。

编译FastDebug版、仅含Server模式的HotSpot虚拟机 。

```bash
bash configure --enable-debug --with-jvm-variants=server
```

{% note::若使用gcc8进行编译需增加--disable-warnings-as-errors参数 %}

执行过程中会检查工具链及依赖，若有缺失会给出安装命令。若顺利执行则会输出如下所示的各种摘要信息：

```
A new configuration has been successfully created in
/home/ubuntu/jdk-jdk-12-ga/build/linux-x86_64-server-fastdebug
using configure arguments '--enable-debug --with-jvm-variants=server'.

Configuration summary:
* Debug level:    fastdebug
* HS debug level: fastdebug
* JVM variants:   server
* JVM features:   server: 'aot cds cmsgc compiler1 compiler2 epsilongc g1gc graal jfr jni-check jvmci jvmti management nmt parallelgc serialgc services shenandoahgc vm-structs zgc' 
* OpenJDK target: OS: linux, CPU architecture: x86, address length: 64
* Version string: 12-internal+0-adhoc.ubuntu.jdk-jdk-12-ga (12-internal)

Tools summary:
* Boot JDK:       openjdk version "11.0.15.1" 2022-04-22 LTS OpenJDK Runtime Environment (build 11.0.15.1+2-LTS) OpenJDK 64-Bit Server VM (build 11.0.15.1+2-LTS, mixed mode)  (at /home/ubuntu/jdk-11.0.15.1-full)
* Toolchain:      gcc (GNU Compiler Collection)
* C Compiler:     Version 7.5.0 (at /usr/bin/gcc)
* C++ Compiler:   Version 7.5.0 (at /usr/bin/g++)

Build performance summary:
* Cores to use:   16
* Memory limit:   24079 MB
```

进行编译：

```bash
make images
```

编译完成后查看jdk信息：

```bash
cd $BASE_DIR/build/linux-x86_64-server-fastdebug/jdk/bin && ./java -version
openjdk version "12-internal" 2019-03-19
OpenJDK Runtime Environment (fastdebug build 12-internal+0-adhoc.ubuntu.jdk-jdk-12-ga)
OpenJDK 64-Bit Server VM (fastdebug build 12-internal+0-adhoc.ubuntu.jdk-jdk-12-ga, mixed mode)
```

## 调试环境搭建

### 安装VSCode

登录 https://code.visualstudio.com/ 网址进行下载，安装时一直下一步就可。

### 连接Linux服务器

登录服务器，编辑/etc/sysctl.conf，新增如下内容：

```
fs.inotify.max_user_watches=524288
```

执行`sudo sysctl -p`使其生效。

打开VSCode，点击左下角的打开远程窗口，点击Connect to Host，输入相关信息连接至远程服务器。若感觉每回都输入密码过于麻烦，可以设置免密。

```bash
 ssh-keygen -t rsa
 ssh-copy-id -i ~/.ssh/id_rsa.pub ubuntu@remoteAddr
```

安装相关c/c++插件，使用vscode打开jdk源码所在的文件夹，等待配置完成。为了能够在编辑器里实现debug，我们需要在项目路径新建.vscode文件夹，并且新建c_cpp_properties.json、launch.json和tasks.json文件，具体内容如下：

```json  c_cpp_properties.json
{
    "configurations": [
        {
            "name": "Linux",
            "includePath": [
                "${workspaceFolder}/**"
            ],
            "defines": [],
            "compilerPath": "/usr/bin/g++",
            "cStandard": "c11",
            "cppStandard": "gnu++14",
            "intelliSenseMode": "linux-gcc-x64",
            "configurationProvider": "ms-vscode.makefile-tools"
        }
    ],
    "version": 4
}
```

```json  launch.json
{
    // 使用 IntelliSense 了解相关属性。 
    // 悬停以查看现有属性的描述。
    // 欲了解更多信息，请访问: https://go.microsoft.com/fwlink/?linkid=830387
    "version": "0.2.0",
    "configurations": [
        {
            "name": "java -version",
            "preLaunchTask": "make images",
            "type": "cppdbg",
            "request": "launch",
            "program": "${workspaceFolder}/build/linux-x86_64-server-fastdebug/jdk/bin/java",
            "args": [
                "-version"
            ],
            "stopAtEntry": false,
            "cwd": "${fileDirname}",
            "environment": [],
            "externalConsole": false,
            "MIMode": "gdb",
            "setupCommands": [
                {
                    "description": "为 gdb 启用整齐打印",
                    "text": "-enable-pretty-printing",
                    "ignoreFailures": true
                },
                {
                    "description":  "将反汇编风格设置为 Intel",
                    "text": "-gdb-set disassembly-flavor intel",
                    "ignoreFailures": true
                }
            ]
        }

    ]
}
```

```json  tasks.json
{
	"version": "2.0.0",
	"tasks": [
		{
			"type": "cppbuild",
			"label": "make images",
			"command": "make",
			"args": [
				"images"
			],
			"options": {
				"cwd": "${workspaceFolder}"
			},
			"group": "build",
			"detail": "make"
		}
	]
}
```

以上是执行命令java -version的相关配置，若需要执行其他命令可自行修改launch.json文件。

src/java.base/share/native/libjli/java.c的JavaMain(void * _args)方法为程序入口，我们在此方法打上断点进行调试，可以看到能够正常调试。

![](https://blog-1255608703.cos.ap-hongkong.myqcloud.com/deeply-understand-the-jvm-virtual-machine-chapter1-build-your-jdk/jvm-debug-example.png)

HotSpot JVM 在内部将 SEGV 信号用于一些不同的事情（NullPointerException、safepoints、...），因此在调试过程中会出现SIGSEGV Segmentation fault错误。若不想让其出现这些东西，请执行以下命令：

```bash
echo "handle SIGSEGV pass noprint nostop" >> $HOME/.gdbinit
```