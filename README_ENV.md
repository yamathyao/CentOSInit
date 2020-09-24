### 安装 openjdk

查看现有 jdk  
`rpm -qa |grep java`  
`rpm -qa |grep jdk` 

如果有就批量删除  
`rpm -qa | grep java | xargs rpm -e --nodeps` 

安装  
`yum install java-1.8.0-openjdk* -y`  

安装 maven  

配置环境变量  
```
export JAVA_HOME=/usr/lib/jvm/java-1.8.0-openjdk-1.8.0.242.b08-0.el7_7.x86_64
export CLASSPATH=.:$JAVA_HOME/jre/lib/rt.jar:$JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar
export MAVEN_HOME=/opt/apache-maven-3.6.3
export PATH=$PATH:$JAVA_HOME/bin:$MAVEN_HOME/bin
```
