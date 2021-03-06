## Docker打包部署


### 环境准备

> 配置企业内部`Docker Registry`

#### 本地`/etc/hosts`文件

```
192.168.1.158	docker.apiacmed.com
```

#### `Docker Registry` 控制台

```
    http://docker.apiacmed.com
    用户名： acmedback 
    密码：   Acmedcare123

```

----

### 项目目录`src/assembly` 下添加`Maven`的`assembly`插件

> assembly-descriptor.xml

```xml

<assembly
   xmlns="http://maven.apache.org/plugins/maven-assembly-plugin/assembly/1.1.3"
   xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
   xsi:schemaLocation="http://maven.apache.org/plugins/maven-assembly-plugin/assembly/1.1.3 http://maven.apache.org/xsd/assembly-1.1.3.xsd">
    <id>spring-cloud-assembly</id>
    <formats>
        <format>zip</format>
    </formats>
    <includeBaseDirectory>false</includeBaseDirectory>
    <fileSets>
        <!--bin -->
        <fileSet>
            <directory>src/main/bin</directory>
            <outputDirectory>bin</outputDirectory>
            <includes>
                <include>*.sh</include>
            </includes>
            <fileMode>0755</fileMode>
            <lineEnding>unix</lineEnding>
        </fileSet>

        <fileSet>
            <directory>src/main/resources</directory>
            <outputDirectory>config</outputDirectory>
            <includes>
                <include>*.properties</include>
                <include>logback.xml</include>
                <include>banner.txt</include>
            </includes>
            <lineEnding>unix</lineEnding>
        </fileSet>

        <!--artifact -->
        <fileSet>
            <directory>target</directory>
            <outputDirectory>./</outputDirectory>
            <includes>
                <include>${project.artifactId}-*.jar</include>
            </includes>
            <fileMode>0755</fileMode>
        </fileSet>
    </fileSets>
</assembly>

```


### `pom.xml` 添加Docker打包插件

```xml

<plugin>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-maven-plugin</artifactId>
</plugin>

<plugin>
    <groupId>com.spotify</groupId>
    <artifactId>docker-maven-plugin</artifactId>
    <version>0.4.13</version>
    <configuration>
        <imageName>docker.apiacmed.com/acmedback/${project.artifactId}:${project.version}</imageName>
        <dockerDirectory>src/main/docker</dockerDirectory>
        <resources>
            <resource>
                <targetPath>/</targetPath>
                <directory>${project.build.directory}</directory>
                <include>*.zip</include>
            </resource>
        </resources>
    </configuration>
</plugin>

<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-assembly-plugin</artifactId>
    <executions>
        <execution>
            <phase>package</phase>
            <goals>
                <goal>single</goal>
            </goals>
            <configuration>
                <finalName>${project.artifactId}-${project.version}</finalName>
                <appendAssemblyId>false</appendAssemblyId>
                <descriptors>
                    <descriptor>src/assembly/assembly-descriptor.xml</descriptor>
                </descriptors>
            </configuration>
        </execution>
    </executions>
</plugin>

```
----

### 项目`Nacos`服务发现配置

#### 添加`Nacos`服务发现的依赖

```xml

<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
</dependency>

<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-alibaba-nacos-config</artifactId>
</dependency>

```

#### 添加`Nacos`服务发现的配置文件 `bootstrap-nacos.properties`

```properties


## Nacos Discovery Server
spring.cloud.nacos.discovery.server-addr=nacos.acmedcare.com:8848

## Nacos Config Server
spring.cloud.nacos.config.server-addr=nacos.acmedcare.com:8848

```

#### 

----
### 项目目录`src/main/docker` 创建 `Dockerfile`

> 以下配置文件中 `microservices-spring-cloud-app` ，即为项目的 `artifactId`

```dockerfile
FROM docker.apiacmed.com/env/openjdk-acmed:8-jre-alpine
MAINTAINER Elve.Xu <iskp.me@gmail.com>

ENV VERSION 2.1.1.BUILD-SNAPSHOT

RUN echo "http://mirrors.aliyun.com/alpine/v3.6/main" > /etc/apk/repositories \
    && echo "http://mirrors.aliyun.com/alpine/v3.6/community" >> /etc/apk/repositories \
    && apk update upgrade \
    && apk add --no-cache procps unzip curl bash tzdata \
    && ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime \
    && echo "Asia/Shanghai" > /etc/timezone

ADD microservices-spring-cloud-app-${VERSION}.zip /microservices-spring-cloud-app/microservices-spring-cloud-app-${VERSION}.zip

RUN unzip /microservices-spring-cloud-app/microservices-spring-cloud-app-${VERSION}.zip -d /microservices-spring-cloud-app \
    && rm -rf /microservices-spring-cloud-app/microservices-spring-cloud-app-${VERSION}.zip \
    && sed -i '$d' /microservices-spring-cloud-app/bin/startup.sh \
    && echo "tail -f /dev/null" >> /microservices-spring-cloud-app/bin/startup.sh

EXPOSE 19090

CMD ["/microservices-spring-cloud-app/bin/startup.sh"]

```

### 项目目录`src/main/bin` 创建启动脚本`startup.sh` & `shutdown.sh`

> 注意修改启动脚本中变量参数

```bash
SERVICE_NAME=microservices-spring-cloud-app   ### 修改为项目名称
SERVICE_VERSION=2.1.1.BUILD-SNAPSHOT          ### 修改为项目版本
```

----

### 打包项目

```bash

## 切换目录
cd distribution

## 打包项目
mvn clean install -DskipTests=true

## 编译镜像
mvn docker:build

## 登录Docker Registry
docker login docker.apiacmed.com
> Username: acmedback   ## 输入用户名
> Password: Acmedcare123  ## 输入密码

## 推送镜像到远程仓库 
docker push docker.apiacmed.com/acmedback/${项目名称}:${项目版本}

```

----

### 登录指定的服务器进行容器启动

> ssh xx.xx.xx.xx

----

### 启动项目

```bash

## 启动命令模板
docker run -p 19090:19090 \
    --net docker-br0 --ip 172.172.1.117 \
    --add-host nacos.acmedcare.com:172.172.1.3 \ 
    --add-host mysql.acmedcare.com:172.172.1.101 \ 
    --add-host trace.acmedcare.com:192.168.1.151 \ 
    --add-host redis.acmedcare.com:172.172.1.101 \ 
    -d -v /tmp/logs:/tmp/logs \ 
    -v /tmp/logs/${项目名称}:/${项目名称}/logs \ 
    --name ${项目名称} \ 
    docker.apiacmed.com/acmedback/${项目名称}:${项目版本}

> 注意容器的`--ip`地址指定

## 示例
docker run -p 19090:19090 \
    --net docker-br0 --ip 172.172.1.117 \
    --add-host nacos.acmedcare.com:172.172.1.3 \ 
    --add-host mysql.acmedcare.com:172.172.1.101 \ 
    --add-host trace.acmedcare.com:192.168.1.151 \ 
    --add-host redis.acmedcare.com:172.172.1.101 \ 
    -d -v /tmp/logs:/tmp/logs \ 
    -v /tmp/logs/microservices-spring-cloud-app:/microservices-spring-cloud-app/logs \ 
    --name microservices-spring-cloud-app \ 
    docker.apiacmed.com/acmedback/microservices-spring-cloud-app:2.1.1.BUILD-SNAPSHOT



```
 