---
title: 'Apache Flink的快速开始'
category: ETL 
tags: [ETL, Flink]
---
## 1. Flink环境准备
单机模式使用FLink非常简单，只需要下载[Flink](https://flink.apache.org/zh/downloads.html)压缩包，然后执行`./bin/start-cluster.sh`启动服务。在本地`http://localhost:8081/`就可以打开WebUI的界面。

## 2. 测试数据库准备
作为ETL任务，我们需要指定输入源Source与输出源Sink。这里我使用本地MySQL同时作为输入和输出数据源。  

### 2.1 建立数据库
```SQL
-- 输入数据库
CREATE DATABASE `flink_source`;
CREATE TABLE `user` (
  `id` bigint(20) NOT NULL AUTO_INCREMENT,
  `username` varchar(100) NOT NULL,
  `first_name` varchar(100) DEFAULT NULL,
  `last_name` varchar(100) DEFAULT NULL,
  `age` int(11) NOT NULL,
  `user_info` varchar(200) DEFAULT NULL,
  `last_update_time` datetime DEFAULT NULL,
  PRIMARY KEY (`id`)
)
-- 输出数据库
CREATE DATABASE `flink_sink`;
CREATE TABLE `user_sink` (
  `id` bigint(20) NOT NULL AUTO_INCREMENT,
  `username` varchar(100) NOT NULL,
  `first_name` varchar(100) DEFAULT NULL,
  `last_name` varchar(100) DEFAULT NULL,
  `age` int(11) NOT NULL,
  `user_info` varchar(200) DEFAULT NULL,
  `last_update_time` datetime DEFAULT NULL,
  PRIMARY KEY (`id`)
)
```

### 2.2准备测试数据
这里在输入源里插入两条数据，其中一条年龄17，另一条18。我们期望读取并filter出年龄大于等于18的user。
```SQL
INSERT INTO flink_source.`user` (username,first_name,last_name,age,user_info,last_update_time) VALUES
	 ('test18','test_first18','test_last18',18,'info18','2020-01-13 23:18:00.0'),
	 ('test17','test_first17','test_last17',17,'info17','2020-01-13 23:17:00.0');
```

## 3. 编写Flink程序

### 3.1 导入maven依赖
新建一个maven项目，然后加入依赖，如下。
```xml
<!-- pom.xml -->
<dependencies>
    <dependency>
        <groupId>org.apache.flink</groupId>
        <artifactId>flink-java</artifactId>
        <version>1.12.0</version>
    </dependency>
    <dependency>
        <groupId>org.apache.flink</groupId>
        <artifactId>flink-streaming-java_2.12</artifactId>
        <version>1.12.0</version>
        <scope>provided</scope>
    </dependency>
    <dependency>
        <groupId>org.apache.flink</groupId>
        <artifactId>flink-connector-jdbc_2.12</artifactId>
        <version>1.12.0</version>
        <scope>provided</scope>
    </dependency>
    <dependency>
        <groupId>mysql</groupId>
        <artifactId>mysql-connector-java</artifactId>
        <version>8.0.22</version>
    </dependency>
    <dependency>
        <groupId>org.projectlombok</groupId>
        <artifactId>lombok</artifactId>
        <version>1.18.16</version>
        <scope>provided</scope>
    </dependency>
</dependencies>
```
### 3.2 编写Java程序
1. 对于输入，我们分别建立Entity对应表结构，建立Function来获取数据源。
```Java
// UserEntity.java
@Data
@Builder
public class UserEntity implements Serializable {
    private Long id;
    private String username;
    private String firstName;
    private String lastName;
    private Integer age;
    private String userInfo;
    private LocalDateTime lastUpdateTime;
}
```
UserSourceFunction继承了Flink的RichSourceFunction，重写了关键方法，open()函数负责在读取数据库前调用，可以用来初始化数据库。run()函数是数据源的核心方法，sourceContext.collect()收集数据源发出的数据并交给Flink分布式去处理。cancel()做了错误处理。
```Java
// UserSourceFunction.java
public class UserSourceFunction extends RichSourceFunction<UserEntity> {
    final String MYSQL_DRIVER = "com.mysql.cj.jdbc.Driver";
    final String URL = "jdbc:mysql://localhost:3306/";
    final String DB_NAME = "flink_source";
    final String DB_NAME_TABLE_NAME = "flink_source.user";
    final String USERNAME = "root";
    final String PASSWORD = "dev";

    Connection connect;
    PreparedStatement ps;

    @Override
    public void open(Configuration parameters) throws Exception {
        super.open(parameters);
        Class.forName(MYSQL_DRIVER);
        connect = DriverManager.getConnection(URL + DB_NAME, USERNAME, PASSWORD);
        ps = connect.prepareStatement("select id, username, first_name, last_name, age, user_info, last_update_time from " + DB_NAME_TABLE_NAME);
    }

    @Override
    public void run(SourceContext<UserEntity> sourceContext) throws Exception {
        ResultSet resultSet = ps.executeQuery();
        while (resultSet.next()) {
            sourceContext.collect(UserEntity.builder()
                    .id(resultSet.getLong(1))
                    .username(resultSet.getString(2))
                    .firstName(resultSet.getString(3))
                    .lastName(resultSet.getString(4))
                    .age(resultSet.getInt(5))
                    .userInfo(resultSet.getString(6))
                    .lastUpdateTime(resultSet.getTimestamp(7).toLocalDateTime())
                    .build());
        }
    }

    @Override
    public void cancel() {
        try {
            super.close();
            if (Objects.nonNull(connect)) {
                connect.close();
            }
            if (Objects.nonNull(ps)) {
                ps.close();
            }
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```

2. 对于输出，因为我们是表对表，entity可以复用。所以我们只需要写输出源就可以了。输出源同样也复写open()，而与run()函数和cancel()函数对应，输出源的函数叫做invoke()和close()，我这里不多赘述。
```Java
// UserSinkFunction.java
public class UserSinkFunction extends RichSinkFunction<UserEntity> {
    @Override
    public void open(...){...}

    @Override
    public void invoke(UserEntity value, Context context) throws Exception {
        ps.setLong(1, value.getId());
        ps.setString(2, value.getUsername());
        ps.setString(3, value.getFirstName());
        ps.setString(4, value.getLastName());
        ps.setInt(5, value.getAge());
        ps.setString(6, value.getUserInfo());
        ps.setTimestamp(7, java.sql.Timestamp.valueOf(value.getLastUpdateTime()));

        ps.executeUpdate();
    }

    @Override
    public void close() {...}
}
```

3. 处理逻辑的核心方法是在main函数中，而Flink的Job是按主函数来区分的，如果一个Jar文件有多个主函数，也可以分别作为多个Flink Job触发。我们建立一个FlinkApplication类，写上主函数，这将作为Flink执行的入口。  
主函数代码在#1处需要获取Flink环境，这是Flink运行的基础。在#2处使用了lambda的语法写了一个filter来实现我们的逻辑。然后在最后声明执行。
   
```Java
public class FlinkApplication {
    public static void main(String[] args) throws Exception {
        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment(); // #1

        SingleOutputStreamOperator<UserEntity> user = env.addSource(new UserSourceFunction()).name("flink_source.user");
        SingleOutputStreamOperator<UserEntity> filterUser = user.filter(userEntity -> userEntity.getAge() >= 18).name ("filterUserAgeLagerThan18"); // #2
        filterUser.addSink(new UserSinkFunction()).name("flink_sink.user_sink");

        env.execute("FlinkApplicationUserToUserSink");
    }
}
```

### 3.3 将依赖也打包到Jar，并指定默认Class

由于maven不会默认将依赖打包到最终的Jar包中，我们需要在pom文件中使用打包插件，同时将Jar的默认MainClass指定，这样Flink会自动识别Jar的MainClass。

```xml
<!-- pom.xml -->
<build>
    <plugins>
        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-shade-plugin</artifactId>
            <version>3.2.4</version>
            <executions>
                <execution>
                    <phase>package</phase>
                    <goals>
                        <goal>shade</goal>
                    </goals>
                    <configuration>
                        <artifactSet>
                            <excludes>
                                <exclude>com.google.code.findbugs:jsr305</exclude>
                                <exclude>org.slf4j:*</exclude>
                                <exclude>log4j:*</exclude>
                            </excludes>
                        </artifactSet>
                        <filters>
                            <filter>
                                <artifact>*:*</artifact>
                                <excludes>
                                    <exclude>META-INF/*.SF</exclude>
                                    <exclude>META-INF/*.DSA</exclude>
                                    <exclude>META-INF/*.RSA</exclude>
                                </excludes>
                            </filter>
                        </filters>
                        <transformers>
                            <transformer
                                    implementation="org.apache.maven.plugins.shade.resource.ManifestResourceTransformer">
                                <mainClass>io.github.uhan6.FlinkApplication</mainClass>
                            </transformer>
                        </transformers>
                    </configuration>
                </execution>
            </executions>
        </plugin>
    </plugins>
</build>
```

## 4. 部署运行
访问`http://localhost:8081/`，点击左下角的`"Submit New Job"`，然后点击右上角的`"Add New"`选择我们打包好的Jar并上传，待上传完成后点击这个Jar会扩展出`"Submit"`按钮，点击提交这个Job。

![](../assets/posts/2021-01/Apache%20Flink的快速开始_4_1.png)

这时会跳转到Job的监控页面，待成功后我们会发现数据库的记录按照要求同步了age>=18的那条记录。

![](../assets/posts/2021-01/Apache%20Flink的快速开始_4_2.png)

如果任务不幸失败，可以点击TaskManagers下的LOG链接去查看报错信息。

![](../assets/posts/2021-01/Apache%20Flink的快速开始_4_3.png)

## 5. 总结
在初次使用Flink的过程中，我只是简单的使用JDBC做了表对表的拷贝，实际上Flink支持的功能强大的多。希望之后继续努力学习进步。  
此外，可以[在此](https://github.com/uhan6/get-start-apache-flink/tree/branch_20210114)找到本文的全部代码。