# 编译安装最新版 Git

<!-- TOC -->

- [编译安装最新版 Git](#编译安装最新版-git)
    - [完整步骤](#完整步骤)
        - [获取代码并编译](#获取代码并编译)
        - [卸载旧版 Git](#卸载旧版-git)
        - [将编译好的最新版的目录添加到环境变量](#将编译好的最新版的目录添加到环境变量)
    - [后遗症](#后遗症)
    - [遇到的问题](#遇到的问题)
        - [`git-compat-util.h:280:25: fatal error: openssl/ssl.h: No such file or directory`](#git-compat-utilh28025-fatal-error-opensslsslh-no-such-file-or-directory)
        - [`Can't locate ExtUtils/MakeMaker.pm in @INC`](#cant-locate-extutilsmakemakerpm-in-inc)
        - [`/bin/sh: line 1: asciidoc: command not found`](#binsh-line-1-asciidoc-command-not-found)
        - [`/bin/sh: line 1: xmlto: command not found`](#binsh-line-1-xmlto-command-not-found)
        - [`/bin/sh: line 1: docbook2x-texi: command not found`](#binsh-line-1-docbook2x-texi-command-not-found)
    - [部分参考](#部分参考)

<!-- /TOC -->

## 完整步骤

Tips： 为了保证编译一次成功，可以将『遇到的问题』部分中须安装的依赖提前安装好。

### 获取代码并编译

```bash
$ cd ~/workspace/git                     
$ git clone git@github.com:git/git.git
$ cd git
$ git checkout v2.11.0 ; 当前最新版
$ git checkout -b v2.11 ; 创建新分支
$ make prefix=/usr all doc info
$ sudo mv ../git /opt
```
### 卸载旧版 Git 

- `yum remove git`

### 将编译好的最新版的目录添加到环境变量

```
file: ~/.bashrc

PATH=/opt/git:$PATH
export PATH
```
- `. ~/.bashrc`  //使环境变量生效

```
[rainstorm@localhost blog]$ git --version
git version 2.11.0
```

## 后遗症

1. 在运行命令时可能会出现 `no such file...` 这类的问题，只要把 `/opt/git/` 目录下相应的文件复制到错误指定的位置即可

```
/usr/libexec/git-core/git-sh-setup: line 343: cd: .git: No such file or directory

$ sudo mkdir /usr/libexec/git-core
$ sudo cp /opt/git/git-sh-setup /usr/libexec/git-core
```

## 遇到的问题

### `git-compat-util.h:280:25: fatal error: openssl/ssl.h: No such file or directory`

```bash
[rainstorm@localhost git]$ make prefix=/opt/git all doc info
GIT_VERSION = 2.11.0
    * new build flags
    CC credential-store.o
In file included from cache.h:4:0,
                 from credential-store.c:1:
git-compat-util.h:280:25: fatal error: openssl/ssl.h: No such file or directory
 #include <openssl/ssl.h>
                         ^
compilation terminated.
make: *** [credential-store.o] Error 1

```

- `# yum groupinstall 'Development Tools'`

- `# yum install openssl-devel curl-devel expat-devel gettext-devel zlib-devel`

### `Can't locate ExtUtils/MakeMaker.pm in @INC`

```
    SUBDIR perl
/usr/bin/perl Makefile.PL PREFIX='/usr' INSTALL_BASE='' --localedir='/usr/share/locale'
Can't locate ExtUtils/MakeMaker.pm in @INC (@INC contains: /usr/local/lib64/perl5 /usr/local/share/perl5 /usr/lib64/perl5/vendor_perl /usr/share/perl5/vendor_perl /usr/lib64/perl5 /usr/share/perl5 .) at Makefile.PL line 3.
BEGIN failed--compilation aborted at Makefile.PL line 3.
make[1]: *** [perl.mak] Error 2
make: *** [perl/perl.mak] Error 2

```

- `# yum install perl-ExtUtils-CBuilder perl-ExtUtils-MakeMaker`

### `/bin/sh: line 1: asciidoc: command not found`

```
    ASCIIDOC git-show-branch.html
/bin/sh: line 1: asciidoc: command not found
make[1]: *** [git-show-branch.html] Error 127
make[1]: Leaving directory `/home/rainstorm/workspace/git/git/Documentation'
make: *** [doc] Error 2

```
- `# yum install asciidoc`

### `/bin/sh: line 1: xmlto: command not found`

```
    XMLTO git-show-branch.1
/bin/sh: line 1: xmlto: command not found
make[1]: *** [git-show-branch.1] Error 127
make[1]: Leaving directory `/home/rainstorm/workspace/git/git/Documentation'
make: *** [doc] Error 2

```
- `# yum install xmlto`

### `/bin/sh: line 1: docbook2x-texi: command not found`

```
    DB2TEXI user-manual.texi
/bin/sh: line 1: docbook2x-texi: command not found
make[1]: *** [user-manual.texi] Error 127
make[1]: Leaving directory `/home/rainstorm/workspace/git/git/Documentation'
make: *** [info] Error 2

```

- `# yum install docbook2X` //注意 最后 X 为大写
- `$ cd /usr/bin`
- `sudo ln -s /usr/bin/db2x_docbook2texi /usr/bin/docbook2x-texi`

## 部分参考

1. [How to install latest version of git in Redhat 6.x/7.x](http://stackoverflow.com/questions/32709471/how-to-install-latest-version-of-git-in-redhat-6-x-7-x)
1. [ Can't locate ExtUtils/MakeMaker.pm in @INC 错误的解决方式 ](http://blog.csdn.net/w87848608/article/details/13997271)
1. [How to install latest version of git on CentOS 6.x/7.x](http://stackoverflow.com/questions/21820715/how-to-install-latest-version-of-git-on-centos-6-x-7-x)