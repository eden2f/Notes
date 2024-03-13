# Mybatis 代码生成器Mybatis-Generator使用详解

> [Throwable : Mybatis代码生成器Mybatis-Generator使用详解](https://www.cnblogs.com/throwable/p/12046848.html)

## 前提
最近在做创业项目的时候因为有比较多的新需求，需要频繁基于`DDL`生成`Mybatis`适合的实体、`Mapper`接口和映射文件。其中，代码生成器是`MyBatis Generator(MBG)`，用到了`Mybatis-Generator-Core`相关依赖，这里通过一篇文章详细地分析这个代码生成器的使用方式。本文编写的时候使用的`Mybatis-Generator`版本为`1.4.0`，其他版本没有进行过调研。
## 引入插件
`Mybatis-Generator`的运行方式有很多种：

- 基于`mybatis-generator-core-x.x.x.jar`和其`XML`配置文件，通过命令行运行。
- 通过`Ant`的`Task`结合其`XML`配置文件运行。
- 通过`Maven`插件运行。
- 通过`Java`代码和其`XML`配置文件运行。
- 通过`Java`代码和编程式配置运行。
- 通过`Eclipse Feature`运行。

这里只介绍通过`Maven`插件运行和通过`Java`代码和其`XML`配置文件运行这两种方式，两种方式有个特点：都要提前编写好`XML`配置文件。个人感觉`XML`配置文件相对直观，后文会花大量篇幅去说明`XML`配置文件中的配置项及其作用。这里先注意一点：默认的配置文件为`ClassPath:generatorConfig.xml`。
### 通过编码和配置文件运行
通过编码方式去运行插件先需要引入`mybatis-generator-core`依赖，编写本文的时候最新的版本为：
```xml
<dependency>
    <groupId>org.mybatis.generator</groupId>
    <artifactId>mybatis-generator-core</artifactId>
    <version>1.4.0</version>
</dependency>
```
假设编写好的`XML`配置文件是`ClassPath`下的`generator-configuration.xml`，那么使用代码生成器的编码方式大致如下：
```java
List<String> warnings = new ArrayList<>();
// 如果已经存在生成过的文件是否进行覆盖
boolean overwrite = true;
File configFile = new File("ClassPath路径/generator-configuration.xml");
ConfigurationParser cp = new ConfigurationParser(warnings);
Configuration config = cp.parseConfiguration(configFile);
DefaultShellCallback callback = new DefaultShellCallback(overwrite);
MyBatisGenerator generator = new MyBatisGenerator(config, callback, warnings);
generator.generate(null);
```
### 通过Maven插件运行
如果使用`Maven`插件，那么不需要引入`mybatis-generator-core`依赖，只需要引入一个`Maven`的插件`mybatis-generator-maven-plugin`：
```xml
<plugins>
    <plugin>
        <groupId>org.mybatis.generator</groupId>
        <artifactId>mybatis-generator-maven-plugin</artifactId>
        <version>1.4.0</version>
        <executions>
            <execution>
                <id>Generate MyBatis Artifacts</id>
                <goals>
                    <goal>generate</goal>
                </goals>
            </execution>
        </executions>
        <configuration>
            <!-- 输出详细信息 -->
            <verbose>true</verbose>
            <!-- 覆盖生成文件 -->
            <overwrite>true</overwrite>
            <!-- 定义配置文件 -->
            <configurationFile>${basedir}/src/main/resources/generator-configuration.xml</configurationFile>
        </configuration>
    </plugin>
</plugins>
```
`mybatis-generator-maven-plugin`的更详细配置和可选参数可以参考：[Running With Maven](http://mybatis.org/generator/running/runningWithMaven.html)。插件配置完毕之后，使用下面的命令即可运行：
```shell
mvn mybatis-generator:generate
```
## XML配置文件详解
`XML`配置文件才是`Mybatis-Generator`的核心，它用于控制代码生成的所有行为。所有非标签独有的公共配置的`Key`可以在`mybatis-generator-core`的`PropertyRegistry`类中找到。下面是一个相对完整的配置文件的模板：
```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE generatorConfiguration
  PUBLIC "-//mybatis.org//DTD MyBatis Generator Configuration 1.0//EN"
  "http://mybatis.org/dtd/mybatis-generator-config_1_0.dtd">

<generatorConfiguration>

  <properties resource="db.properties"/>

  <classPathEntry location="/Program Files/IBM/SQLLIB/java/db2java.zip" />

  <context id="DB2Tables" targetRuntime="MyBatis3">

    <jdbcConnection driverClass="COM.ibm.db2.jdbc.app.DB2Driver"
        connectionURL="jdbc:db2:TEST"
        userId="db2admin"
        password="db2admin">
    </jdbcConnection>

    <plugin type="org.mybatis.generator.plugins.SerializablePlugin"/>

    <commentGenerator>
        <property name="suppressDate" value="true"/>
        <property name="suppressAllComments" value="true"/>
    </commentGenerator>

    <javaTypeResolver>
      <property name="forceBigDecimals" value="false" />
    </javaTypeResolver>

    <javaModelGenerator targetPackage="test.model" targetProject="\MBGTestProject\src">
      <property name="enableSubPackages" value="true" />
      <property name="trimStrings" value="true" />
    </javaModelGenerator>

    <sqlMapGenerator targetPackage="test.xml"  targetProject="\MBGTestProject\src">
      <property name="enableSubPackages" value="true" />
    </sqlMapGenerator>

    <javaClientGenerator type="XMLMAPPER" targetPackage="test.dao"  targetProject="\MBGTestProject\src">
      <property name="enableSubPackages" value="true" />
    </javaClientGenerator>

    <table schema="DB2ADMIN" tableName="ALLTYPES" domainObjectName="Customer" >
      <property name="useActualColumnNames" value="true"/>
      <generatedKey column="ID" sqlStatement="DB2" identity="true" />
      <columnOverride column="DATE_FIELD" property="startDate" />
      <ignoreColumn column="FRED" />
      <columnOverride column="LONG_VARCHAR_FIELD" jdbcType="VARCHAR" />
    </table>

  </context>
</generatorConfiguration>
```
配置文件中，最外层的标签为`<generatorConfiguration>`，它的子标签包括：

- 0或者1个`<properties>`标签，用于指定全局配置文件，下面可以通过占位符的形式读取`<properties>`指定文件中的值。
- 0或者N个`<classPathEntry>`标签，`<classPathEntry>`只有一个`location`属性，用于指定数据源驱动包（`jar`或者`zip`）的绝对路径，具体选择什么驱动包取决于连接什么类型的数据源。
- 1或者N个`<context>`标签，用于运行时的解析模式和具体的代码生成行为，所以这个标签里面的配置是最重要的。

下面分别列举和分析一下`<context>`标签和它的主要子标签的一些属性配置和功能。
### context标签
`<context>`标签在`mybatis-generator-core`中对应的实现类为`org.mybatis.generator.config.Context`，它除了大量的子标签配置之外，比较主要的属性是：

-  `id`：`Context`示例的唯一`ID`，用于输出错误信息时候作为唯一标记。 
-  `targetRuntime`：用于执行代码生成模式。 

defaultModelType：控制 Domain 类的生成行为。执行引擎为 MyBatis3DynamicSql 或者 MyBatis3Kotlin 时忽略此配置，可选值： 

- `conditional`：默认值，类似`hierarchical`，但是只有一个主键的时候会合并所有属性生成在同一个类。
- `flat`：所有内容全部生成在一个对象中。
- `hierarchical`：键生成一个XXKey对象，Blob等单独生成一个对象，其他简单属性在一个对象中。

`targetRuntime`属性的可选值比较多，这里做个简单的小结：

| 属性 | 功能描述 |
| --- | --- |
| `MyBatis3DynamicSql` | 默认值，兼容`JDK8+`
和`MyBatis 3.4.2+`
，不会生成`XML`
映射文件，忽略`<sqlMapGenerator>`
的配置项，也就是`Mapper`
全部注解化，依赖于`MyBatis Dynamic SQL`
类库 |
| `MyBatis3Kotlin` | 行为类似于`MyBatis3DynamicSql`
，不过兼容`Kotlin`
的代码生成 |
| `MyBatis3` | 提供基本的基于动态`SQL`
的`CRUD`
方法和`XXXByExample`
方法，会生成`XML`
映射文件 |
| `MyBatis3Simple` | 提供基本的基于动态`SQL`
的`CRUD`
方法，会生成`XML`
映射文件 |
| `MyBatis3DynamicSqlV1` | 已经过时，不推荐使用 |

笔者偏向于把`SQL`文件和代码分离，所以一般选用`MyBatis3`或者`MyBatis3Simple`。例如：
```xml
<context id="default" targetRuntime="MyBatis3">
```
`<context>`标签支持0或N个`<property>`标签，`<property>`的可选属性有：

| property属性 | 功能描述 | 默认值 | 备注 |
| --- | --- | --- | --- |
| `autoDelimitKeywords` | 是否使用分隔符号括住数据库关键字 | `false` | 例如`MySQL`
中会使用反引号括住关键字 |
| `beginningDelimiter` | 分隔符号的开始符号 | `"` |  |
| `endingDelimiter` | 分隔符号的结束号 | `"` |  |
| `javaFileEncoding` | 文件的编码 | `系统默认值` | 来源于`java.nio.charset.Charset` |
| `javaFormatter` | 类名和文件格式化器 | `DefaultJavaFormatter` | 见`JavaFormatter`
和`DefaultJavaFormatter` |
| `targetJava8` | 是否JDK8和启动其特性 | `true` |  |
| `kotlinFileEncoding` | `Kotlin`
文件编码 | `系统默认值` | 来源于`java.nio.charset.Charset` |
| `kotlinFormatter` | `Kotlin`
类名和文件格式化器 | `DefaultKotlinFormatter` | 见`KotlinFormatter`
和`DefaultKotlinFormatter` |
| `xmlFormatter` | `XML`
文件格式化器 | `DefaultXmlFormatter` | 见`XmlFormatter`
和`DefaultXmlFormatter` |

### jdbcConnection标签
`<jdbcConnection>`标签用于指定数据源的连接信息，它在`mybatis-generator-core`中对应的实现类为`org.mybatis.generator.config.JDBCConnectionConfiguration`，主要属性包括：

| 属性 | 功能描述 | 是否必须 |
| --- | --- | --- |
| `driverClass` | 数据源驱动的全类名 | `Y` |
| `connectionURL` | `JDBC`
的连接`URL` | `Y` |
| `userId` | 连接到数据源的用户名 | `N` |
| `password` | 连接到数据源的密码 | `N` |

### commentGenerator标签
`<commentGenerator>`标签是可选的，用于控制生成的实体的注释内容。它在`mybatis-generator-core`中对应的实现类为`org.mybatis.generator.internal.DefaultCommentGenerator`，可以通过可选的`type`属性指定一个自定义的`CommentGenerator`实现。`<commentGenerator>`标签支持0或N个`<property>`标签，`<property>`的可选属性有：

| property属性 | 功能描述 | 默认值 |
| --- | --- | --- |
| `suppressAllComments` | 是否生成注释 | `false` |
| `suppressDate` | 是否在注释中添加生成的时间戳 | `false` |
| `dateFormat` | 配合`suppressDate`
使用，指定输出时间戳的格式 | `java.util.Date#toString()` |
| `addRemarkComments` | 是否输出表和列的`Comment`
信息 | `false` |

笔者建议保持默认值，也就是什么注释都不输出，生成代码干净的实体。
### javaTypeResolver标签
`<javaTypeResolver>`标签是`<context>`的子标签，用于解析和计算数据库列类型和`Java`类型的映射关系，该标签只包含一个`type`属性，用于指定`org.mybatis.generator.api.JavaTypeResolver`接口的实现类。`<javaTypeResolver>`标签支持0或N个`<property>`标签，`<property>`的可选属性有：

| property属性 | 功能描述 | 默认值 |
| --- | --- | --- |
| `forceBigDecimals` | 是否强制把所有的数字类型强制使用`java.math.BigDecimal`
类型表示 | `false` |
| `useJSR310Types` | 是否支持`JSR310`
，主要是`JSR310`
的新日期类型 | `false` |

如果`useJSR310Types`属性设置为`true`，那么生成代码的时候类型映射关系如下（主要针对日期时间类型）：

| 数据库（JDBC）类型 | Java类型 |
| --- | --- |
| `DATE` | `java.time.LocalDate` |
| `TIME` | `java.time.LocalTime` |
| `TIMESTAMP` | `java.time.LocalDateTime` |
| `TIME_WITH_TIMEZONE` | `java.time.OffsetTime` |
| `TIMESTAMP_WITH_TIMEZONE` | `java.time.OffsetDateTime` |

引入`mybatis-generator-core`后，可以查看`JavaTypeResolver`的默认实现为`JavaTypeResolverDefaultImpl`，从它的源码可以得知一些映射关系：
```java
BIGINT --> Long
BIT --> Boolean
INTEGER --> Integer
SMALLINT --> Short
TINYINT --> Byte
......
```
有些时候，我们希望`INTEGER`、`SMALLINT`和`TINYINT`都映射为`Integer`，那么我们需要覆盖`JavaTypeResolverDefaultImpl`的构造方法：
```java
public class DefaultJavaTypeResolver extends JavaTypeResolverDefaultImpl {

    public DefaultJavaTypeResolver() {
        super();
        typeMap.put(Types.SMALLINT, new JdbcTypeInformation("SMALLINT",
                new FullyQualifiedJavaType(Integer.class.getName())));
        typeMap.put(Types.TINYINT, new JdbcTypeInformation("TINYINT",
                new FullyQualifiedJavaType(Integer.class.getName())));
    }
}
```
注意一点的是这种自定义实现`JavaTypeResolver`接口的方式使用编程式运行`MBG`会相对方便，如果需要使用`Maven`插件运行，那么需要把上面的`DefaultJavaTypeResolver`类打包到插件中。
### javaModelGenerator标签
`<javaModelGenerator标签>`标签是`<context>`的子标签，主要用于控制实体（`Model`）类的代码生成行为。它支持的属性如下：

| 属性 | 功能描述 | 是否必须 | 备注 |
| --- | --- | --- | --- |
| `targetPackage` | 生成的实体类的包名 | `Y` | 例如`club.throwable.model` |
| `targetProject` | 生成的实体类文件相对于项目（根目录）的位置 | `Y` | 例如`src/main/java` |

`<javaModelGenerator标签>`标签支持0或N个`<property>`标签，`<property>`的可选属性有：

| property属性 | 功能描述 | 默认值 | 备注 |
| --- | --- | --- | --- |
| `constructorBased` | 是否生成一个带有所有字段属性的构造函数 | `false` | `MyBatis3Kotlin`
模式下忽略此属性配置 |
| `enableSubPackages` | 是否允许通过`Schema`
生成子包 | `false` | 如果为`true`
，例如包名为`club.throwable`
，如果`Schema`
为`xyz`
，那么实体类文件最终会生成在`club.throwable.xyz`
目录 |
| `exampleTargetPackage` | 生成的伴随实体类的`Example`
类的包名 | - | - |
| `exampleTargetProject` | 生成的伴随实体类的`Example`
类文件相对于项目（根目录）的位置 | - | - |
| `immutable` | 是否不可变 | `false` | 如果为`true`
，则不会生成`Setter`
方法，所有字段都使用`final`
修饰，提供一个带有所有字段属性的构造函数 |
| `rootClass` | 为生成的实体类添加父类 | - | 通过`value`
指定父类的全类名即可 |
| `trimStrings` | `Setter`
方法是否对字符串类型进行一次`trim`
操作 | `false` | - |

### javaClientGenerator标签
`<javaClientGenerator>`标签是`<context>`的子标签，主要用于控制`Mapper`接口的代码生成行为。它支持的属性如下：

| 属性 | 功能描述 | 是否必须 | 备注 |
| --- | --- | --- | --- |
| `type` | `Mapper`
接口生成策略 | `Y` | `<context>`
标签的`targetRuntime`
属性为`MyBatis3DynamicSql`
或者`MyBatis3Kotlin`
时此属性配置忽略 |
| `targetPackage` | 生成的`Mapper`
接口的包名 | `Y` | 例如`club.throwable.mapper` |
| `targetProject` | 生成的`Mapper`
接口文件相对于项目（根目录）的位置 | `Y` | 例如`src/main/java` |

`type`属性的可选值如下：

- `ANNOTATEDMAPPER`：`Mapper`接口生成的时候依赖于注解和`SqlProviders`（也就是纯注解实现），不会生成`XML`映射文件。
- `XMLMAPPER`：`Mapper`接口生成接口方法，对应的实现代码生成在`XML`映射文件中（也就是纯映射文件实现）。
- `MIXEDMAPPER`：`Mapper`接口生成的时候复杂的方法实现生成在`XML`映射文件中，而简单的实现通过注解和`SqlProviders`实现（也就是注解和映射文件混合实现）。

注意两点：

- `<context>`标签的`targetRuntime`属性指定为`MyBatis3Simple`的时候，`type`只能选用`ANNOTATEDMAPPER`或者`XMLMAPPER`。
- `<context>`标签的`targetRuntime`属性指定为`MyBatis3`的时候，`type`可以选用`ANNOTATEDMAPPER`、`XMLMAPPER`或者`MIXEDMAPPER`。

`<javaClientGenerator>`标签支持0或N个`<property>`标签，`<property>`的可选属性有：

| property属性 | 功能描述 | 默认值 | 备注 |
| --- | --- | --- | --- |
| `enableSubPackages` | 是否允许通过`Schema`
生成子包 | `false` | 如果为`true`
，例如包名为`club.throwable`
，如果`Schema`
为`xyz`
，那么`Mapper`
接口文件最终会生成在`club.throwable.xyz`
目录 |
| `useLegacyBuilder` | 是否通过`SQL Builder`
生成动态`SQL` | `false` |  |
| `rootInterface` | 为生成的`Mapper`
接口添加父接口 | - | 通过`value`
指定父接口的全类名即可 |

### sqlMapGenerator标签
`<sqlMapGenerator>`标签是`<context>`的子标签，主要用于控制`XML`映射文件的代码生成行为。它支持的属性如下：

| 属性 | 功能描述 | 是否必须 | 备注 |
| --- | --- | --- | --- |
| `targetPackage` | 生成的`XML`
映射文件的包名 | `Y` | 例如`mappings` |
| `targetProject` | 生成的`XML`
映射文件相对于项目（根目录）的位置 | `Y` | 例如`src/main/resources` |

`<sqlMapGenerator>`标签支持0或N个`<property>`标签，`<property>`的可选属性有：

| property属性 | 功能描述 | 默认值 | 备注 |
| --- | --- | --- | --- |
| `enableSubPackages` | 是否允许通过`Schema`
生成子包 | `false` | - |

### plugin标签
`<plugin>`标签是`<context>`的子标签，用于引入一些插件对代码生成的一些特性进行扩展，该标签只包含一个`type`属性，用于指定`org.mybatis.generator.api.Plugin`接口的实现类。内置的插件实现见[Supplied Plugins](http://mybatis.org/generator/reference/plugins.html)。例如：引入`org.mybatis.generator.plugins.SerializablePlugin`插件会让生成的实体类自动实现`java.io.Serializable`接口并且添加`serialVersionUID`属性。
### table标签
<table>标签是<context>的子标签，主要用于配置要生成代码的数据库表格，定制一些代码生成行为等等。它支持的属性众多，列举如下：

| 属性 | 功能描述 | 是否必须 | 备注 |
| --- | --- | --- | --- |
| tableName | 数据库表名称 | Y | 例如t_order |
| schema | 数据库Schema | N | - |
| catalog | 数据库Catalog | N | - |
| alias | 表名称标签 | N | 如果指定了此值，则查询列的时候结果格式为alias_column |
| domainObjectName | 表对应的实体类名称，可以通过.指定包路径 | N | 如果指定了bar.User，则包名为bar，实体类名称为User |
| mapperName | 表对应的Mapper接口类名称，可以通过.指定包路径 | N | 如果指定了bar.UserMapper，则包名为bar，Mapper接口类名称为UserMapper |
| sqlProviderName | 动态SQL提供类SqlProvider的类名称 | N | - |
| enableInsert | 是否允许生成insert方法 | N | 默认值为true，执行引擎为MyBatis3DynamicSql或者MyBatis3Kotlin时忽略此配置 |
| enableSelectByPrimaryKey | 是否允许生成selectByPrimaryKey方法 | N | 默认值为true，执行引擎为MyBatis3DynamicSql或者MyBatis3Kotlin时忽略此配置 |
| enableSelectByExample | 是否允许生成selectByExample方法 | N | 默认值为true，执行引擎为MyBatis3DynamicSql或者MyBatis3Kotlin时忽略此配置 |
| enableUpdateByPrimaryKey | 是否允许生成updateByPrimaryKey方法 | N | 默认值为true，执行引擎为MyBatis3DynamicSql或者MyBatis3Kotlin时忽略此配置 |
| enableDeleteByPrimaryKey | 是否允许生成deleteByPrimaryKey方法 | N | 默认值为true，执行引擎为MyBatis3DynamicSql或者MyBatis3Kotlin时忽略此配置 |
| enableDeleteByExample | 是否允许生成deleteByExample方法 | N | 默认值为true，执行引擎为MyBatis3DynamicSql或者MyBatis3Kotlin时忽略此配置 |
| enableCountByExample | 是否允许生成countByExample方法 | N | 默认值为true，执行引擎为MyBatis3DynamicSql或者MyBatis3Kotlin时忽略此配置 |
| enableUpdateByExample | 是否允许生成updateByExample方法 | N | 默认值为true，执行引擎为MyBatis3DynamicSql或者MyBatis3Kotlin时忽略此配置 |
| selectByPrimaryKeyQueryId | value指定对应的主键列提供列表查询功能 | N | 执行引擎为MyBatis3DynamicSql或者MyBatis3Kotlin时忽略此配置 |
| selectByExampleQueryId | value指定对应的查询ID提供列表查询功能 | N | 执行引擎为MyBatis3DynamicSql或者MyBatis3Kotlin时忽略此配置 |
| modelType | 覆盖<context>的defaultModelType属性 | N | 见<context>的defaultModelType属性 |
| escapeWildcards | 是否对通配符进行转义 | N | - |
| delimitIdentifiers | 标记匹配表名称的时候是否需要使用分隔符去标记生成的SQL | N | - |
| delimitAllColumns | 是否所有的列都添加分隔符 | N | 默认值为false，如果设置为true，所有列名会添加起始和结束分隔符 |

<table>标签支持0或N个<property>标签，<property>的可选属性有：

| property属性 | 功能描述 | 默认值 | 备注 |
| --- | --- | --- | --- |
| constructorBased | 是否为实体类生成一个带有所有字段的构造函数 | false | 执行引擎为MyBatis3Kotlin的时候此属性忽略 |
| ignoreQualifiersAtRuntime | 是否在运行时忽略别名 | false | 如果为true，则不会在生成表的时候把schema和catalog作为表的前缀 |
| immutable | 实体类是否不可变 | false | 执行引擎为MyBatis3Kotlin的时候此属性忽略 |
| modelOnly | 是否仅仅生成实体类 | false | - |
| rootClass | 如果配置此属性，则实体类会继承此指定的超类 | - | 如果有主键属性会把主键属性在超类生成 |
| rootInterface | 如果配置此属性，则实体类会实现此指定的接口 | - | 执行引擎为MyBatis3Kotlin或者MyBatis3DynamicSql的时候此属性忽略 |
| runtimeCatalog | 指定运行时的Catalog | - | 当生成表和运行时的表的Catalog不一样的时候可以使用该属性进行配置 |
| runtimeSchema | 指定运行时的Schema | - | 当生成表和运行时的表的Schema不一样的时候可以使用该属性进行配置 |
| runtimeTableName | 指定运行时的表名称 | - | 当生成表和运行时的表的表名称不一样的时候可以使用该属性进行配置 |
| selectAllOrderByClause | 指定字句内容添加到selectAll()方法的order by子句之中 | - | 执行引擎为MyBatis3Simple的时候此属性才适用 |
| trimStrings | 实体类的字符串类型属性会做trim处理 | - | 执行引擎为MyBatis3Kotlin的时候此属性忽略 |
| useActualColumnNames | 是否使用列名作为实体类的属性名 | false | - |
| useColumnIndexes | XML映射文件中生成的ResultMap使用列索引定义而不是列名称 | false | 执行引擎为MyBatis3Kotlin或者MyBatis3DynamicSql的时候此属性忽略 |
| useCompoundPropertyNames | 是否把列名和列备注拼接起来生成实体类属性名 | false | - |

<table>标签还支持众多的非property的子标签：

- 0或1个<generatedKey>用于指定主键生成的规则，指定此标签后会生成一个<selectKey>标签：
```xml
<!-- column：指定主键列 -->
<!-- sqlStatement：查询主键的SQL语句，例如填写了MySql，则使用SELECT LAST_INSERT_ID() -->
<!-- type：可选值为pre或者post，pre指定selectKey标签的order为BEFORE，post指定selectKey标签的order为AFTER -->
<!-- identity：true的时候，指定selectKey标签的order为AFTER -->
<generatedKey column="id" sqlStatement="MySql" type="post" identity="true" />
```

- 0或1个<domainObjectRenamingRule>用于指定实体类重命名规则：
```xml
<!-- searchString中正则命中的实体类名部分会替换为replaceString -->
<domainObjectRenamingRule searchString="^Sys" replaceString=""/>
<!-- 例如 SysUser会变成User -->
<!-- 例如 SysUserMapper会变成UserMapper -->
```

- 0或1个<columnRenamingRule>用于指定列重命名规则：
```xml
<!-- searchString中正则命中的列名部分会替换为replaceString -->
<columnRenamingRule searchString="^CUST_" replaceString=""/>
<!-- 例如 CUST_BUSINESS_NAME会变成BUSINESS_NAME（useActualColumnNames=true） -->
<!-- 例如 CUST_BUSINESS_NAME会变成businessName（useActualColumnNames=false） -->
```

- 0或N个<columnOverride>用于指定具体列的覆盖映射规则：
```xml
<!-- column：指定要覆盖配置的列 -->
<!-- property：指定要覆盖配置的属性 -->
<!-- delimitedColumnName：是否为列名添加定界符，例如`{column}` -->
<!-- isGeneratedAlways：是否一定生成此列 -->
<columnOverride column="customer_name" property="customerName" javaType="" jdbcType="" typeHandler="" delimitedColumnName="" isGeneratedAlways="">
   <!-- 覆盖table或者javaModelGenerator级别的trimStrings属性配置 -->
   <property name="trimStrings" value="true"/>
<columnOverride/>
```

- 0或N个<ignoreColumn>用于指定忽略生成的列：
```xml
<ignoreColumn column="version" delimitedColumnName="false"/>
```
## 实战
如果需要深度定制一些代码生成行为，建议引入mybatis-generator-core并且通过编程式执行代码生成方法，否则可以选用Maven插件。假设我们在本地数据local有一张t_order表如下：
```sql
CREATE TABLE `t_order`
(
    id           BIGINT UNSIGNED PRIMARY KEY COMMENT '主键',
    order_id     VARCHAR(64)    NOT NULL COMMENT '订单ID',
    create_time  DATETIME       NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
    amount       DECIMAL(10, 2) NOT NULL DEFAULT 0 COMMENT '金额',
    order_status TINYINT        NOT NULL DEFAULT 0 COMMENT '订单状态',
    UNIQUE uniq_order_id (`order_id`)
) COMMENT '订单表';
```
假设项目的结构如下：
```shell
mbg-sample
  - main
   - java
    - club
     - throwable
   - resources
```
下面会基于此前提举三个例子。编写基础的XML配置文件：
```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE generatorConfiguration
        PUBLIC "-//mybatis.org//DTD MyBatis Generator Configuration 1.0//EN"
        "http://mybatis.org/dtd/mybatis-generator-config_1_0.dtd">
<generatorConfiguration>
    <!-- 驱动包绝对路径 -->
    <classPathEntry
            location="I:\Develop\Maven-Repository\mysql\mysql-connector-java\5.1.48\mysql-connector-java-5.1.48.jar"/>

    <context id="default" targetRuntime="这里选择合适的引擎">

        <property name="javaFileEncoding" value="UTF-8"/>

        <!-- 不输出注释 -->
        <commentGenerator>
            <property name="suppressDate" value="true"/>
            <property name="suppressAllComments" value="true"/>
        </commentGenerator>

        <jdbcConnection driverClass="com.mysql.jdbc.Driver"
                        connectionURL="jdbc:mysql://localhost:3306/local"
                        userId="root"
                        password="root">
        </jdbcConnection>


        <!-- 不强制把所有的数字类型转化为BigDecimal -->
        <javaTypeResolver>
            <property name="forceBigDecimals" value="false"/>
        </javaTypeResolver>

        <javaModelGenerator targetPackage="club.throwable.entity" targetProject="src/main/java">
            <property name="enableSubPackages" value="true"/>
        </javaModelGenerator>

        <sqlMapGenerator targetPackage="mappings" targetProject="src/main/resources">
            <property name="enableSubPackages" value="true"/>
        </sqlMapGenerator>

        <javaClientGenerator type="这里选择合适的Mapper类型" targetPackage="club.throwable.dao" targetProject="src/main/java">
            <property name="enableSubPackages" value="true"/>
        </javaClientGenerator>

        <table tableName="t_order"
               enableCountByExample="false"
               enableDeleteByExample="false"
               enableSelectByExample="false"
               enableUpdateByExample="false"
               domainObjectName="Order"
               mapperName="OrderMapper">
            <generatedKey column="id" sqlStatement="MySql"/>
        </table>
    </context>
</generatorConfiguration>
```
### 纯注解
使用纯注解需要引入mybatis-dynamic-sql：
```xml
<dependency>
    <groupId>org.mybatis.dynamic-sql</groupId>
    <artifactId>mybatis-dynamic-sql</artifactId>
    <version>1.1.4</version>
</dependency>
```
需要修改两个位置：
```xml
<context id="default" targetRuntime="MyBatis3DynamicSql">
...

<javaClientGenerator type="ANNOTATEDMAPPER"
...
```
运行结果会生成三个类：
```java
// club.throwable.entity
public class Order {
    @Generated("org.mybatis.generator.api.MyBatisGenerator")
    private Long id;

    @Generated("org.mybatis.generator.api.MyBatisGenerator")
    private String orderId;

    @Generated("org.mybatis.generator.api.MyBatisGenerator")
    private Date createTime;

    @Generated("org.mybatis.generator.api.MyBatisGenerator")
    private BigDecimal amount;

    @Generated("org.mybatis.generator.api.MyBatisGenerator")
    private Byte orderStatus;

    @Generated("org.mybatis.generator.api.MyBatisGenerator")
    public Long getId() {
        return id;
    }

    @Generated("org.mybatis.generator.api.MyBatisGenerator")
    public void setId(Long id) {
        this.id = id;
    }

    @Generated("org.mybatis.generator.api.MyBatisGenerator")
    public String getOrderId() {
        return orderId;
    }

    @Generated("org.mybatis.generator.api.MyBatisGenerator")
    public void setOrderId(String orderId) {
        this.orderId = orderId;
    }

    @Generated("org.mybatis.generator.api.MyBatisGenerator")
    public Date getCreateTime() {
        return createTime;
    }

    @Generated("org.mybatis.generator.api.MyBatisGenerator")
    public void setCreateTime(Date createTime) {
        this.createTime = createTime;
    }

    @Generated("org.mybatis.generator.api.MyBatisGenerator")
    public BigDecimal getAmount() {
        return amount;
    }

    @Generated("org.mybatis.generator.api.MyBatisGenerator")
    public void setAmount(BigDecimal amount) {
        this.amount = amount;
    }

    @Generated("org.mybatis.generator.api.MyBatisGenerator")
    public Byte getOrderStatus() {
        return orderStatus;
    }

    @Generated("org.mybatis.generator.api.MyBatisGenerator")
    public void setOrderStatus(Byte orderStatus) {
        this.orderStatus = orderStatus;
    }
}

// club.throwable.dao
public final class OrderDynamicSqlSupport {
    @Generated("org.mybatis.generator.api.MyBatisGenerator")
    public static final Order order = new Order();

    @Generated("org.mybatis.generator.api.MyBatisGenerator")
    public static final SqlColumn<Long> id = order.id;

    @Generated("org.mybatis.generator.api.MyBatisGenerator")
    public static final SqlColumn<String> orderId = order.orderId;

    @Generated("org.mybatis.generator.api.MyBatisGenerator")
    public static final SqlColumn<Date> createTime = order.createTime;

    @Generated("org.mybatis.generator.api.MyBatisGenerator")
    public static final SqlColumn<BigDecimal> amount = order.amount;

    @Generated("org.mybatis.generator.api.MyBatisGenerator")
    public static final SqlColumn<Byte> orderStatus = order.orderStatus;

    @Generated("org.mybatis.generator.api.MyBatisGenerator")
    public static final class Order extends SqlTable {
        public final SqlColumn<Long> id = column("id", JDBCType.BIGINT);

        public final SqlColumn<String> orderId = column("order_id", JDBCType.VARCHAR);

        public final SqlColumn<Date> createTime = column("create_time", JDBCType.TIMESTAMP);

        public final SqlColumn<BigDecimal> amount = column("amount", JDBCType.DECIMAL);

        public final SqlColumn<Byte> orderStatus = column("order_status", JDBCType.TINYINT);

        public Order() {
            super("t_order");
        }
    }
}

@Mapper
public interface OrderMapper {
    @Generated("org.mybatis.generator.api.MyBatisGenerator")
    BasicColumn[] selectList = BasicColumn.columnList(id, orderId, createTime, amount, orderStatus);

    @Generated("org.mybatis.generator.api.MyBatisGenerator")
    @SelectProvider(type=SqlProviderAdapter.class, method="select")
    long count(SelectStatementProvider selectStatement);

    @Generated("org.mybatis.generator.api.MyBatisGenerator")
    @DeleteProvider(type=SqlProviderAdapter.class, method="delete")
    int delete(DeleteStatementProvider deleteStatement);

    @Generated("org.mybatis.generator.api.MyBatisGenerator")
    @InsertProvider(type=SqlProviderAdapter.class, method="insert")
    @SelectKey(statement="SELECT LAST_INSERT_ID()", keyProperty="record.id", before=true, resultType=Long.class)
    int insert(InsertStatementProvider<Order> insertStatement);

    @Generated("org.mybatis.generator.api.MyBatisGenerator")
    @SelectProvider(type=SqlProviderAdapter.class, method="select")
    @Results(id="OrderResult", value = {
        @Result(column="id", property="id", jdbcType=JdbcType.BIGINT, id=true),
        @Result(column="order_id", property="orderId", jdbcType=JdbcType.VARCHAR),
        @Result(column="create_time", property="createTime", jdbcType=JdbcType.TIMESTAMP),
        @Result(column="amount", property="amount", jdbcType=JdbcType.DECIMAL),
        @Result(column="order_status", property="orderStatus", jdbcType=JdbcType.TINYINT)
    })
    Optional<Order> selectOne(SelectStatementProvider selectStatement);

    @Generated("org.mybatis.generator.api.MyBatisGenerator")
    @SelectProvider(type=SqlProviderAdapter.class, method="select")
    @Results(id="OrderResult", value = {
        @Result(column="id", property="id", jdbcType=JdbcType.BIGINT, id=true),
        @Result(column="order_id", property="orderId", jdbcType=JdbcType.VARCHAR),
        @Result(column="create_time", property="createTime", jdbcType=JdbcType.TIMESTAMP),
        @Result(column="amount", property="amount", jdbcType=JdbcType.DECIMAL),
        @Result(column="order_status", property="orderStatus", jdbcType=JdbcType.TINYINT)
    })
    List<Order> selectMany(SelectStatementProvider selectStatement);

    @Generated("org.mybatis.generator.api.MyBatisGenerator")
    @UpdateProvider(type=SqlProviderAdapter.class, method="update")
    int update(UpdateStatementProvider updateStatement);

    @Generated("org.mybatis.generator.api.MyBatisGenerator")
    default long count(CountDSLCompleter completer) {
        return MyBatis3Utils.countFrom(this::count, order, completer);
    }

    @Generated("org.mybatis.generator.api.MyBatisGenerator")
    default int delete(DeleteDSLCompleter completer) {
        return MyBatis3Utils.deleteFrom(this::delete, order, completer);
    }

    @Generated("org.mybatis.generator.api.MyBatisGenerator")
    default int deleteByPrimaryKey(Long id_) {
        return delete(c -> 
            c.where(id, isEqualTo(id_))
        );
    }

    @Generated("org.mybatis.generator.api.MyBatisGenerator")
    default int insert(Order record) {
        return MyBatis3Utils.insert(this::insert, record, order, c ->
            c.map(id).toProperty("id")
            .map(orderId).toProperty("orderId")
            .map(createTime).toProperty("createTime")
            .map(amount).toProperty("amount")
            .map(orderStatus).toProperty("orderStatus")
        );
    }

    @Generated("org.mybatis.generator.api.MyBatisGenerator")
    default int insertSelective(Order record) {
        return MyBatis3Utils.insert(this::insert, record, order, c ->
            c.map(id).toProperty("id")
            .map(orderId).toPropertyWhenPresent("orderId", record::getOrderId)
            .map(createTime).toPropertyWhenPresent("createTime", record::getCreateTime)
            .map(amount).toPropertyWhenPresent("amount", record::getAmount)
            .map(orderStatus).toPropertyWhenPresent("orderStatus", record::getOrderStatus)
        );
    }

    @Generated("org.mybatis.generator.api.MyBatisGenerator")
    default Optional<Order> selectOne(SelectDSLCompleter completer) {
        return MyBatis3Utils.selectOne(this::selectOne, selectList, order, completer);
    }

    @Generated("org.mybatis.generator.api.MyBatisGenerator")
    default List<Order> select(SelectDSLCompleter completer) {
        return MyBatis3Utils.selectList(this::selectMany, selectList, order, completer);
    }

    @Generated("org.mybatis.generator.api.MyBatisGenerator")
    default List<Order> selectDistinct(SelectDSLCompleter completer) {
        return MyBatis3Utils.selectDistinct(this::selectMany, selectList, order, completer);
    }

    @Generated("org.mybatis.generator.api.MyBatisGenerator")
    default Optional<Order> selectByPrimaryKey(Long id_) {
        return selectOne(c ->
            c.where(id, isEqualTo(id_))
        );
    }

    @Generated("org.mybatis.generator.api.MyBatisGenerator")
    default int update(UpdateDSLCompleter completer) {
        return MyBatis3Utils.update(this::update, order, completer);
    }

    @Generated("org.mybatis.generator.api.MyBatisGenerator")
    static UpdateDSL<UpdateModel> updateAllColumns(Order record, UpdateDSL<UpdateModel> dsl) {
        return dsl.set(id).equalTo(record::getId)
                .set(orderId).equalTo(record::getOrderId)
                .set(createTime).equalTo(record::getCreateTime)
                .set(amount).equalTo(record::getAmount)
                .set(orderStatus).equalTo(record::getOrderStatus);
    }

    @Generated("org.mybatis.generator.api.MyBatisGenerator")
    static UpdateDSL<UpdateModel> updateSelectiveColumns(Order record, UpdateDSL<UpdateModel> dsl) {
        return dsl.set(id).equalToWhenPresent(record::getId)
                .set(orderId).equalToWhenPresent(record::getOrderId)
                .set(createTime).equalToWhenPresent(record::getCreateTime)
                .set(amount).equalToWhenPresent(record::getAmount)
                .set(orderStatus).equalToWhenPresent(record::getOrderStatus);
    }

    @Generated("org.mybatis.generator.api.MyBatisGenerator")
    default int updateByPrimaryKey(Order record) {
        return update(c ->
            c.set(orderId).equalTo(record::getOrderId)
            .set(createTime).equalTo(record::getCreateTime)
            .set(amount).equalTo(record::getAmount)
            .set(orderStatus).equalTo(record::getOrderStatus)
            .where(id, isEqualTo(record::getId))
        );
    }

    @Generated("org.mybatis.generator.api.MyBatisGenerator")
    default int updateByPrimaryKeySelective(Order record) {
        return update(c ->
            c.set(orderId).equalToWhenPresent(record::getOrderId)
            .set(createTime).equalToWhenPresent(record::getCreateTime)
            .set(amount).equalToWhenPresent(record::getAmount)
            .set(orderStatus).equalToWhenPresent(record::getOrderStatus)
            .where(id, isEqualTo(record::getId))
        );
    }
}
```
### 极简XML映射文件
极简XML映射文件生成只需要简单修改配置文件：
```xml
<context id="default" targetRuntime="MyBatis3Simple">
...

<javaClientGenerator type="XMLMAPPER"
...
```
生成三个文件：
```java
// club.throwable.entity
public class Order {
    private Long id;

    private String orderId;

    private Date createTime;

    private BigDecimal amount;

    private Byte orderStatus;

    public Long getId() {
        return id;
    }

    public void setId(Long id) {
        this.id = id;
    }

    public String getOrderId() {
        return orderId;
    }

    public void setOrderId(String orderId) {
        this.orderId = orderId;
    }

    public Date getCreateTime() {
        return createTime;
    }

    public void setCreateTime(Date createTime) {
        this.createTime = createTime;
    }

    public BigDecimal getAmount() {
        return amount;
    }

    public void setAmount(BigDecimal amount) {
        this.amount = amount;
    }

    public Byte getOrderStatus() {
        return orderStatus;
    }

    public void setOrderStatus(Byte orderStatus) {
        this.orderStatus = orderStatus;
    }
}

// club.throwable.dao
public interface OrderMapper {
    int deleteByPrimaryKey(Long id);

    int insert(Order record);

    Order selectByPrimaryKey(Long id);

    List<Order> selectAll();

    int updateByPrimaryKey(Order record);
}

// mappings
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE mapper PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN" "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
<mapper namespace="club.throwable.dao.OrderMapper">
    <resultMap id="BaseResultMap" type="club.throwable.entity.Order">
        <id column="id" jdbcType="BIGINT" property="id"/>
        <result column="order_id" jdbcType="VARCHAR" property="orderId"/>
        <result column="create_time" jdbcType="TIMESTAMP" property="createTime"/>
        <result column="amount" jdbcType="DECIMAL" property="amount"/>
        <result column="order_status" jdbcType="TINYINT" property="orderStatus"/>
    </resultMap>
    <delete id="deleteByPrimaryKey" parameterType="java.lang.Long">
        delete
        from t_order
        where id = #{id,jdbcType=BIGINT}
    </delete>
    <insert id="insert" parameterType="club.throwable.entity.Order">
        <selectKey keyProperty="id" order="AFTER" resultType="java.lang.Long">
            SELECT LAST_INSERT_ID()
        </selectKey>
        insert into t_order (order_id, create_time, amount,
        order_status)
        values (#{orderId,jdbcType=VARCHAR}, #{createTime,jdbcType=TIMESTAMP}, #{amount,jdbcType=DECIMAL},
        #{orderStatus,jdbcType=TINYINT})
    </insert>
    <update id="updateByPrimaryKey" parameterType="club.throwable.entity.Order">
        update t_order
        set order_id     = #{orderId,jdbcType=VARCHAR},
            create_time  = #{createTime,jdbcType=TIMESTAMP},
            amount       = #{amount,jdbcType=DECIMAL},
            order_status = #{orderStatus,jdbcType=TINYINT}
        where id = #{id,jdbcType=BIGINT}
    </update>
    <select id="selectByPrimaryKey" parameterType="java.lang.Long" resultMap="BaseResultMap">
        select id, order_id, create_time, amount, order_status
        from t_order
        where id = #{id,jdbcType=BIGINT}
    </select>
    <select id="selectAll" resultMap="BaseResultMap">
        select id, order_id, create_time, amount, order_status
        from t_order
    </select>
    <resultMap id="BaseResultMap" type="club.throwable.entity.Order">
        <id column="id" jdbcType="BIGINT" property="id"/>
        <result column="order_id" jdbcType="VARCHAR" property="orderId"/>
        <result column="create_time" jdbcType="TIMESTAMP" property="createTime"/>
        <result column="amount" jdbcType="DECIMAL" property="amount"/>
        <result column="order_status" jdbcType="TINYINT" property="orderStatus"/>
    </resultMap>
    <delete id="deleteByPrimaryKey" parameterType="java.lang.Long">
        delete
        from t_order
        where id = #{id,jdbcType=BIGINT}
    </delete>
    <insert id="insert" parameterType="club.throwable.entity.Order">
        <selectKey keyProperty="id" order="BEFORE" resultType="java.lang.Long">
            SELECT LAST_INSERT_ID()
        </selectKey>
        insert into t_order (id, order_id, create_time,
        amount, order_status)
        values (#{id,jdbcType=BIGINT}, #{orderId,jdbcType=VARCHAR}, #{createTime,jdbcType=TIMESTAMP},
        #{amount,jdbcType=DECIMAL}, #{orderStatus,jdbcType=TINYINT})
    </insert>
    <update id="updateByPrimaryKey" parameterType="club.throwable.entity.Order">
        update t_order
        set order_id     = #{orderId,jdbcType=VARCHAR},
            create_time  = #{createTime,jdbcType=TIMESTAMP},
            amount       = #{amount,jdbcType=DECIMAL},
            order_status = #{orderStatus,jdbcType=TINYINT}
        where id = #{id,jdbcType=BIGINT}
    </update>
    <select id="selectByPrimaryKey" parameterType="java.lang.Long" resultMap="BaseResultMap">
        select id, order_id, create_time, amount, order_status
        from t_order
        where id = #{id,jdbcType=BIGINT}
    </select>
    <select id="selectAll" resultMap="BaseResultMap">
        select id, order_id, create_time, amount, order_status
        from t_order
    </select>
</mapper>
```
### 编程式自定义类型映射
笔者喜欢把所有的非长整型的数字，统一使用Integer接收，因此需要自定义类型映射。编写映射器如下：
```java
public class DefaultJavaTypeResolver extends JavaTypeResolverDefaultImpl {

    public DefaultJavaTypeResolver() {
        super();
        typeMap.put(Types.SMALLINT, new JdbcTypeInformation("SMALLINT",
                new FullyQualifiedJavaType(Integer.class.getName())));
        typeMap.put(Types.TINYINT, new JdbcTypeInformation("TINYINT",
                new FullyQualifiedJavaType(Integer.class.getName())));
    }
}
```
此时最好使用编程式运行代码生成器，修改XML配置文件：
```xml
<javaTypeResolver type="club.throwable.mbg.DefaultJavaTypeResolver">
        <property name="forceBigDecimals" value="false"/>
</javaTypeResolver>
...
```
运行方法代码如下：
```java
public class Main {

    public static void main(String[] args) throws Exception {
        List<String> warnings = new ArrayList<>();
        // 如果已经存在生成过的文件是否进行覆盖
        boolean overwrite = true;
        ConfigurationParser cp = new ConfigurationParser(warnings);
        Configuration config = cp.parseConfiguration(Main.class.getResourceAsStream("/generator-configuration.xml"));
        DefaultShellCallback callback = new DefaultShellCallback(overwrite);
        MyBatisGenerator generator = new MyBatisGenerator(config, callback, warnings);
        generator.generate(null);
    }
}
```
数据库的order_status是TINYINT类型，生成出来的文件中的orderStatus字段全部替换使用Integer类型定义。
## 小结
本文相对详尽地介绍了Mybatis Generator的使用方式，具体分析了XML配置文件中主要标签以及标签属性的功能。因为Mybatis在Java的ORM框架体系中还会有一段很长的时间处于主流地位，了解Mybatis Generator可以简化CRUD方法模板代码、实体以及Mapper接口代码生成，从而解放大量生产力。Mybatis Generator有不少第三方的扩展，例如tk.mapper或者mybatis-plus自身的扩展，可能附加的功能不一样，但是基本的使用是一致的。
## 参考资料

- [MyBatis Generator](http://mybatis.org/generator/index.html)