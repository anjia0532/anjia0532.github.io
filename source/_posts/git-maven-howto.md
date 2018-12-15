---
title: 使用github搭建自己的maven库
date: 2018-10-11 10:24:40
tags: [github,maven,central,mvn-deploy]
---



建议使用maven中央仓库进行发布，不过我嫌步骤太繁琐了，还需要审核，所以才用github来做。发布中央仓库的可以参考[Maven 发布自己的项目到 Maven 中央仓库](https://www.cnblogs.com/binarylei/p/8628245.html)



使用github分两种，一种是mvn install 或者deploy到本地路径，然后git add && commit && push ，一种是[maven-plugins](https://github.com/github/maven-plugins#readme)



<!--more-->



两种方案都需要在github创建相应的repo，具体创建步骤不多说，自行百度。



## 1. maven-plugins

参考 https://stackoverflow.com/a/14013645

1.修改 `~/.m2/settings.xml`

```xml
<!-- NOTE: MAKE SURE THAT settings.xml IS NOT WORLD READABLE! -->
<settings>
  <servers>
    <server>
      <id>github</id>
      <username>YOUR-USERNAME</username>
      <password>YOUR-PASSWORD</password>
    </server>
  </servers>
</settings>
```

2.修改pom.xml

```xml
<properties>
    <!-- github server corresponds to entry in ~/.m2/settings.xml -->
    <github.global.server>github</github.global.server>
</properties>

<distributionManagement>
    <repository>
        <id>internal.repo</id>
        <name>Temporary Staging Repository</name>
        <url>file://${project.build.directory}/mvn-repo</url>
    </repository>
</distributionManagement>

	<build>
		<plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-deploy-plugin</artifactId>
                <version>2.8.2</version>
                <configuration>
                    <altDeploymentRepository>internal.repo::default::file://${project.build.directory}/mvn-repo</altDeploymentRepository>
                </configuration>
            </plugin>
		</plugins>
	</build>
```

3.修改pom.xml，配置maven-plugins

```xml
<build>
    <plugins>
        <plugin>
            <groupId>com.github.github</groupId>
            <artifactId>site-maven-plugin</artifactId>
            <version>0.12</version>
            <configuration>
                <message>Maven artifacts for ${project.version}</message>  <!-- git commit message -->
                <noJekyll>true</noJekyll>                                  <!-- disable webpage processing -->
                <outputDirectory>${project.build.directory}/mvn-repo</outputDirectory> <!-- matches distribution management repository url above -->
                <branch>refs/heads/master</branch>                       <!-- remote branch name -->
                <includes><include>**/*</include></includes>
                <repositoryName>YOUR-REPOSITORY-NAME</repositoryName>      <!-- github repo name -->
                <repositoryOwner>YOUR-GITHUB-USERNAME</repositoryOwner>    <!-- github username  -->
                <force>false</force> <!-- force commit or no -->
                <merge>true</merge> <!-- merge or no -->
            </configuration>
            <executions>
              <!-- run site-maven-plugin's 'site' target as part of the build's normal 'deploy' phase -->
              <execution>
                <goals>
                  <goal>site</goal>
                </goals>
                <phase>deploy</phase>
              </execution>
            </executions>
        </plugin>
    </plugins>
</build>
```

4.提交到github

```bash
mvn clean deploy
```

## 2. mvn deploy && git add && commit && push

参考 https://blog.csdn.net/u010442302/article/details/74639357

1.clone github repo

```bash
git clone https://github.com/YOUR-USERNAME/YOUR-PROJECT-NAME /home/my/code/maven-repo/
```

2.命令行执行mvn deploy

```bash
mvn deploy -DaltDeploymentRepository=my-mvn-repo::default::file:/home/my/code/maven-repo/
```

3.push到github

```bash
cd /home/my/code/maven-repo/
git add .
git commit -m 'Maven artifacts for 0.0.1'
git push -u origin master
```



## 3. 在项目中使用github repo

```xml
<repositories>
    <repository>
        <id>YOUR-PROJECT-NAME-mvn-repo</id>
        <url>https://raw.githubusercontent.com/YOUR-USERNAME/YOUR-PROJECT-NAME/master/</url>
        <snapshots>
            <enabled>true</enabled>
            <updatePolicy>always</updatePolicy>
        </snapshots>
    </repository>
</repositories>
```



## 总结

如果可以修改pom建议用maven-plugins，如果不能修改pom，则使用命令行`mvn deploy&&git add&&commit&&push`