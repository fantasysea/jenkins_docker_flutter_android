# docker搭建android+jenkins+flutter环境

## 步骤:

### 配置Dockerfile

这里使用的是**flutter_linux_1.20.4-stable**版本,android sdk 需要commandlinetools的最新版本.

```
FROM jenkins/jenkins:lts

USER root
RUN apt update && apt upgrade -y && apt install wget unzip && mkdir /opt/android && cd /opt/android  && wget https://dl.google.com/android/repository/commandlinetools-linux-6609375_latest.zip && unzip commandlinetools-linux-6609375_latest.zip && mkdir /opt/flutter && cd /opt/flutter  && wget https://storage.googleapis.com/flutter_infra/releases/stable/linux/flutter_linux_1.20.4-stable.tar.xz && tar xf flutter_linux_1.20.4-stable.tar.xz

ENV ANDROID_HOME=/opt/android
ENV FLUTTER_HOME=/opt/flutter/flutter

ENV PATH=${PATH}:${ANDROID_HOME}/tools/bin:${ANDROID_HOME}/tools:${ANDROID_HOME}/platform-tools:${FLUTTER_HOME}/bin

ENV PUB_HOSTED_URL=https://pub.flutter-io.cn
ENV FLUTTER_STORAGE_BASE_URL=https://storage.flutter-io.cn

RUN yes | /opt/android/tools/bin/sdkmanager --sdk_root=$ANDROID_HOME "platform-tools" && yes | /opt/android/tools/bin/sdkmanager --sdk_root=$ANDROID_HOME "build-tools;28.0.2" && yes | /opt/android/tools/bin/sdkmanager --sdk_root=$ANDROID_HOME "platforms;android-28" && yes | /opt/android/tools/bin/sdkmanager --sdk_root=$ANDROID_HOME --update
RUN yes | /opt/android/tools/bin/sdkmanager --sdk_root=${ANDROID_HOME} --licenses

RUN chmod -R 777  ${ANDROID_HOME}

RUN chmod -R 777 ${FLUTTER_HOME}

RUN echo ANDROID_HOME="$ANDROID_HOME" >> /etc/environment

RUN flutter config  --no-analytics

RUN yes "y" | flutter doctor --android-licenses


RUN yes "y" | flutter upgrade

USER jenkins


```


### Dockerfile使用说明:

```
#1:build镜像,看到最后
docker build -t myasjenkins:v5 .

#2:运行一个容器,设置jenkisn的挂载目录,其实android和flutter也可以挂载本地
docker run -d --name myasjenkins -p 8080:8080 -p 50000:50000 -v /home/jenkins:/var/jenkins_home  myasjenkins:v5

#3:进入容器产看flutter doctor能否运行成功
docker exec -it myasjenkins bash

```
### 2.常规设置jenkins配置
需要注意的是运行起来的时候,jenkins初始密码会打印在控制台

### 3:使用
配置一个自由风格的工程,添加shell,直接使用shell去执行打包

```
#bin/bash -1

echo '开始'
echo ${WORKSPACE}
flutter doctor -v
flutter pub get
#Archive需执行以下命令
# flutter clean
flutter packages get
# flutter build ios --release --no-codesign

flutter build apk --release

# ...其他动作

echo 'Flutter 结束'

```
执行的时候可能会加载Gradle很慢,你可以在开始的Dockerfile中手动加入内置的gradle
列入我代理下载了一个5.4.6版本的包,解压之后直接copy进来

1:首先需要下载:*https://downloads.gradle-dn.com/distributions/gradle-5.6.4-bin.zip*,解压到gradle-5.6.4文件夹,和Dockerfile同一个目录
```
#>ls

gradle-5.6.4  Dockerfile
```
然后修改下Dockerfile
```
RUN mkdir -p /opt/gradles/5.6.4

COPY gradle-5.6.4 /opt/gradles/5.6.4

```
你去配置jenkins的gradle的时候记得选择手动配置,路径写 **/opt/gradles/5.6.4**

## 其他坑
今天本来弄flutter的时候使用本地挂目录,但是一直没成功就改用直接下载到镜像,后来弄好镜像之后会过头解决本地的问题,发现flutter怎么build都不成功,一直提示:
```
If you are deploying the app to the Play Store, it's recommended to use app bundles or split the APK to reduce the APK size.
    To generate an app bundle, run:
        flutter build appbundle --target-platform android-arm,android-arm64,android-x64
        Learn more on: https://developer.android.com/guide/app-bundle
    To split the APKs per ABI, run:
        flutter build apk --target-platform android-arm,android-arm64,android-x64 --split-per-abi
        Learn more on:  https://developer.android.com/studio/build/configure-apk-splits#configure-abi-split
                                                                        
ERROR: JAVA_HOME is set to an invalid directory: /usr/lib/jvm/java-1.8.0-openjdk-1.8.0.262.b10-0.el7_8.x86_64
                                                                        
Please set the JAVA_HOME variable in your environment to match the      
location of your Java installation.                                     
                                                                        
Running Gradle task 'assembleRelease'...                                
Running Gradle task 'assembleRelease'... Done                       0.1s
Gradle task assembleRelease failed with exit code 1

```

后来发现用android sdk 需要commandlinetools的时候需要手动配置**JAVA_HOME**,但是我配置之后一直提示我需要jdk而不是jre,我也明白他们的区别,但是我反复看了文件夹目录确实是有 **/usr/lib/jvm/java-1.8.0-openjdk-1.8.0.262.b10-0.el7_8.x86_64**,之前找了很久发现需要安装 **yum install java-1.8.0-openjdk-devel** 版本的才行,不然就是jre,虽然链接上写这jdk,但是其实是假的.然后跑去容器里面看了下,他们也有配置**JAVA_HOME**环境,这个浪费了很多时间.jdk包好jre,jdk的bin目录比jre的bin目录多了很多文件,所以开始的时候一直执行有问题.

```
➜  app l /usr/lib/jvm/
total 4.0K
drwxr-xr-x.  3 root root  187 Sep 24 11:34 .
dr-xr-xr-x. 38 root root 4.0K Sep 23 10:10 ..
drwxrwxrwx.  3 root root   17 Sep 24 11:34 java-1.8.0-openjdk-1.8.0.262.b10-0.el7_8.x86_64
lrwxrwxrwx.  1 root root   21 Sep 24 11:34 jre -> /etc/alternatives/jre
lrwxrwxrwx.  1 root root   27 Sep 24 11:34 jre-1.8.0 -> /etc/alternatives/jre_1.8.0
lrwxrwxrwx.  1 root root   35 Sep 24 11:34 jre-1.8.0-openjdk -> /etc/alternatives/jre_1.8.0_openjdk
lrwxrwxrwx.  1 root root   51 Sep 24 11:34 jre-1.8.0-openjdk-1.8.0.262.b10-0.el7_8.x86_64 -> java-1.8.0-openjdk-1.8.0.262.b10-0.el7_8.x86_64/jre
lrwxrwxrwx.  1 root root   29 Sep 24 11:34 jre-openjdk -> /etc/alternatives/jre_openjdk
➜  app pwd

```
