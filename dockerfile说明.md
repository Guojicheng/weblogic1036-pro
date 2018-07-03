---
ebook:
  title: 项目交付
  authors: jicheng.guo
  base-font-size: 6
---

<!-- @import "[TOC]" {cmd="toc" depthFrom=1 depthTo=6 orderedList=false} -->
<!-- code_chunk_output -->

* [一、  Dockerfile 使用说明：](#一-dockerfile-使用说明)
	* [1.1  结构综述](#11-结构综述)
	* [1.2   Dockerfile 详述](#12-dockerfile-详述)
		* [1.2.1 第一个dokcerfile说明： Dockerfile.BASE1（基础镜像准备）](#121-第一个dokcerfile说明-dockerfilebase1基础镜像准备)
		* [1.2.2  第二个Dockerfile详细（weblogic镜像定制）](#122-第二个dockerfile详细weblogic镜像定制)
			* [1.2.2.1 第二个dockerfile构建目录结构及机制说明](#1221-第二个dockerfile构建目录结构及机制说明)
			* [1.2.2.2 第二个Dockerfile](#1222-第二个dockerfile)
			* [1.2.2.3 第二个Dockerfile 脚本功能说明](#1223-第二个dockerfile-脚本功能说明)
			* [1.2.2.4 第二个Dockerfile 构建使用](#1224-第二个dockerfile-构建使用)
		* [1.2.3 第三个 Dockerfile （加载应用程序 ，建议构建流水线使用）](#123-第三个-dockerfile-加载应用程序-建议构建流水线使用)
			* [1.2.3.1 第三个dockerfile目录说明](#1231-第三个dockerfile目录说明)
			* [1.2.3.2 Dockerfile 内容如下](#1232-dockerfile-内容如下)
			* [1.2.3.3 构建使用说明](#1233-构建使用说明)

<!-- /code_chunk_output -->

## 一、  Dockerfile 使用说明：

###  1.1  结构综述
  当前项目分为两个Dockerfile完成。功能分别描述如下：
*    **第一个Dockerfile.BASE1**
  用于基础镜像准备：基于centos 6.8 的官方docker 镜像安装svn （因为用户环境无法上网，故用些镜像作为正式构建weblogic image的基础镜像）
*   **第二个Dockerfile**
  用于构建最终用户使用的weblogic镜像。功能包括静默方式安装weblogic ，jdk 以及定制实现用户需要的Weblogic SSL支持功能 ，prod模式下的应用发布功能，配置文件的自动获取等功能。
*   ** 第三个dockerfile 在ciwebchat 文件夹。
    用于将用呢的程序打包到镜像applications目录内。建议用户使用此处文件夹内application 目录放置应用，并使用该dockerfile使用持续集成流水线内的dockerfile。（将此功能单独分开用于流水线的优势为功能简单，节约构建时间）
### 1.2   Dockerfile 详述
####  1.2.1 第一个dokcerfile说明： Dockerfile.BASE1（基础镜像准备）
* **Dockerfile.BASE1 内容**：
```
FROM centos:6.8
MAINTAINER jicheng <guojc@wise2c.com>
# 安装 svn
RUN  yum install -y subversion  \
  && yum clean all
```
* **使用命令**
在Dockerfile.BASE1文件夹下执行以下命令，构建名为centos6.8svnbase 的基础镜像
```
docker build -t centos6.8svnbase  -f Dockerfile.BASE1 .
```
#### 1.2.2  第二个Dockerfile详细（weblogic镜像定制）
#####  1.2.2.1 第二个dockerfile构建目录结构及机制说明
  当前构建目录wxglwebchat下总分为五个目录，分别是
Application  Dockerfile  Dockerfile.BASE1  keydir  weblogicinstall ，作用分别如下：
*  **Application 目录**  存放将要发布到weblogic的应用包，可为tar ,tag.gz 或目录形式。
* **weblogicinstall**  该目录下存放weblogic容器的静默安装应答文件，相关启动脚本及为实现SSL 定制发布等相关功能所写的定制脚本。
* **keydir** 存放用户SSL 证书及key文件信息
* **Dockerfile.BASE1**  用于生成基础镜像
* **Dockerfile**   用于构建weblogic定制镜像。

#####  1.2.2.2 第二个Dockerfile
```
FROM centos6.8svnbase
MAINTAINER jicheng <guojc@wise2c.com>
# 编码设置
ENV LANG=en_US.UTF-8  \
    LANGUAGE=en_US:en  \
    LC_ALL=en_US.UTF-8
#查看语言支持列表
RUN localedef --list-archive    #精简locale  \
  && cd /usr/lib/locale/  && mv locale-archive locale-archive.old  && localedef -i en_US -f UTF-8 en_US.UTF-8  # 添加中文支持（可选） \
  && localedef -i zh_CN -f UTF-8 zh_CN.UTF-8  && localedef -i zh_CN -f GB2312 zh_CN  && localedef -i zh_CN -f GB2312 zh_CN.GB2312 && localedef -i zh_CN -f GBK zh_CN.GBK \#下面这些也是可选的，可以>丰富中文支持（香港/台湾/新加坡）\
  && localedef -f UTF-8 -i zh_HK zh_HK.UTF-8 && localedef -f UTF-8 -i zh_TW zh_TW.UTF-8  && localedef -f UTF-8 -i zh_SG zh_SG.UTF-8  && localedef -i zh_CN -f GB18030 zh_CN.GB18030  \
  && mkdir -p /wxgl/weblogic
## 创建weblogic用户、配置权限及安装相关依赖
RUN  groupadd weblogic && useradd -g weblogic weblogic && echo "weblogic:weblogic" | chpasswd \
  && curl -o sudo-1.8.6p3-27.el6.x86_64.rpm  http://192.168.2.141:81/weblogic/sudo-1.8.6p3-27.el6.x86_64.rpm  \
  && rpm -ivh sudo-1.8.6p3-27.el6.x86_64.rpm \
  && rm -rf sudo-1.8.6p3-27.el6.x86_64.rpm \
  && echo "weblogic ALL=(ALL) NOPASSWD:ALL" >>/etc/sudoers
## 上传脚本文件及证书
COPY weblogicinstall/* /wxgl/weblogic/weblogicinstall/
COPY keydir  /home/weblogic/
WORKDIR /home/weblogic/
## 安装JDK 配置相关环境变量
RUN  mkdir -p /wxgl/weblogic/beainstall  \
 &&  chmod -R 755 /wxgl/weblogic/  \
 &&  curl -o jdk-6u45-linux-x64.bin http://192.168.2.141:81/weblogic/jdk-6u45-linux-x64.bin  \
 &&  chmod a+x  jdk-6u45-linux-x64.bin  \
 &&  /bin/bash ./jdk-6u45-linux-x64.bin  \
 &&  mv jdk1.6.0_45 /wxgl/weblogic/jdk1.6.0_45 \
 &&  rm -rf jdk-6u45-linux-x64.bin

ENV JAVA_HOME /wxgl/weblogic/jdk1.6.0_45/  \
    CLASSPATH ".:/wxgl/weblogic/jdk1.6.0_45/lib/dt.jar:$JAVA_HOME/lib/tools.jar" \
    PATH   $PATH:/wxgl/weblogic/jdk1.6.0_45/bin

# install weblogic for slient mode , and create domain

RUN curl -o wls1036_generic.jar http://192.168.2.141:81/weblogic/wls1036_generic.jar  &&  chmod -R 755 /wxgl/weblogic && /wxgl/weblogic/jdk1.6.0_45/bin/java  -jar wls1036_generic.jar -mode=silent -silent_xml=/wxgl/weblogic/weblogicinstall/silent.xml -log=/wxgl/weblogic/weblogicinstall/silent.log
RUN   rm wls1036_generic.jar /wxgl/weblogic/weblogicinstall/silent.xml && rm -rf wls1036_generic.jar && chown -R weblogic:weblogic /wxgl/  &&  chmod -R 0777 /wxgl/

ENV CONFIG_JVM_ARGS '-Djava.security.egd=file:/dev/./urandom'
CMD ["/wxgl/weblogic/weblogicinstall/entrypoint.sh"]
#COPY Application  /applications/
#RUN  chown -R weblogic:weblogic /applications \
#     && chmod -R 0777 /applications
#USER weblogic
```
***注意事项***
***当前dockefile中有多处curl 命令，从在线的http服务器拉取相关软件包括jdk 的包，wls1036_generic.jar的包等。使用crul是docker官方推荐的方式。相较ADD命令此方式可在安装后同时删除原软件 ，有效减小镜像大小。但同时在内网需要构建http服务器并放置相关软件***
*** 当前dockerfile里的最后四行被注释，将放置于独立的文件夹ciwebchat中，用于用户的持续集成中来将应用ADD到容器中。
#####  1.2.2.3 第二个Dockerfile 脚本功能说明
  在weblogicinstall 目录下，存放文件作用分别如下：
silent.xml  用于weblogic 静默安装应答
entrypoint.sh  weblogic容器启动脚本，它功能包括从svn拉取配置文件及替换应用内配置，调用create-wls-domain.py 脚本创建base domain , 配置weblogic 启动参数（USER_MEM_ARGS="-Xms1024m -Xmx1024m -Xmn768m 等），配置weblogic启动免密等功能。同时负责协调其它脚本完成相应功能
createapp.sh  调用deploy-application.py 部署应用
              调用create-datasourceonline2.py 部署数据源2
              调用 create-datasourceonline.py 部署数据源1
enable_ssl_server.py  wlst online 模式配置ssl 支持

#####  1.2.2.4 第二个Dockerfile 构建使用
目录下
```
docker build -t wxglweblogicnoapp .
```
镜像名noapp表示该镜像没有加载应用app程序 。

####  1.2.3 第三个 Dockerfile （加载应用程序 ，建议构建流水线使用）
#####  1.2.3.1 第三个dockerfile目录说明
在 ciwebchat 目录内仅有 Application目录及  Dockerfile 文件。
```
[root@rancher-kf-harbor02 ciwebchat]# ls
Application  Dockerfile
```
#####  1.2.3.2 Dockerfile 内容如下
```
FROM wxglweblogicnoapp
MAINTAINER jicheng <guojc@wise2c.com>

COPY Application  /applications/
RUN  chown -R weblogic:weblogic /applications \
     && chmod -R 0777 /applications *
USER weblogic
```
#####  1.2.3.3 构建使用说明
* 第一步，  将应用上传到此目录下的application 目录
* 第二步，  使用docker build 命令生成加载了应用的weblogic 镜像
```
docker build -t wxglweblogicaddapp .
```
