## Spring多数据源配置

### 配置文件

在项目中新建配置文件 MultiDataSourceConfig.java

```java
package com.example.springweb.config;

import org.springframework.boot.jdbc.DataSourceBuilder;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.Primary;

import javax.sql.DataSource;

/**
 * 多数据源配置
 *
 * @Author zhanglin
 * @Date 2019/2/19 22:50
 * @Verson 1.0
 */
@Configuration
public class MultiDataSourceConfig {

    @Bean
    @Primary // 设置主数据源
    public DataSource masterDataSource() {
        DataSourceBuilder dataSourceBuilder = DataSourceBuilder.create();
        DataSource dataSource = dataSourceBuilder
                .driverClassName("com.mysql.cj.jdbc.Driver")
                .url("jdbc:mysql://localhost:3306/test?serverTimezone=GMT&characterEncoding=utf-8")
                .username("root")
                .password("")
                .build();
        return dataSource;
    }

    @Bean
    public DataSource salveDataSource() {
        DataSourceBuilder dataSourceBuilder = DataSourceBuilder.create();
        DataSource dataSource = dataSourceBuilder
                .driverClassName("com.mysql.cj.jdbc.Driver")
                .url("jdbc:mysql://localhost:3306/test2?serverTimezone=GMT&characterEncoding=utf-8")
                .username("root")
                .password("")
                .build();
        return dataSource;
    }
}
```

### 多数据源的使用

* 手写Connection方式

  ```java
  package com.example.springweb.dao;
  
  import com.example.springweb.domain.User;
  import org.springframework.beans.factory.annotation.Qualifier;
  import org.springframework.stereotype.Repository;
  
  import javax.sql.DataSource;
  import java.sql.Connection;
  import java.sql.PreparedStatement;
  import java.sql.SQLException;
  
  /**
   * @Author zhanglin
   * @Date 2019/2/19 22:59
   * @Verson 1.0
   */
  @Repository
  public class UserDao {
  
      private final DataSource masterDataSource;
  
      private final DataSource salveDataSource;
  
      // 构造器方式注入
      public UserDao(@Qualifier("masterDataSource") DataSource masterDataSource,
                     @Qualifier("salveDataSource") DataSource salveDataSource) {
          this.masterDataSource = masterDataSource;
          this.salveDataSource = salveDataSource;
      }
  
      public boolean save(User user) {
          boolean result = false;
          Connection connection = null;
          try {
              connection = salveDataSource.getConnection();
              connection.setAutoCommit(false); // 设置非自动提交
              PreparedStatement statement = connection.prepareStatement(
                      "INSERT INTO user(username, password) VALUES (?, ?)");
              statement.setString(1, user.getUsername());
              statement.setString(2, user.getPassword());
              result = statement.executeUpdate() > 0;
          } catch (SQLException e) {
              e.printStackTrace();
          } finally {
              if (connection != null) {
                  try {
                      connection.commit(); // 在非自动提交的情况下，需要手动commit
                      connection.close();
                  } catch (SQLException e1) {
                      e1.printStackTrace();
                  }
              }
          }
  
          return result;
      }
  }
  ```

### 思考：当配合jpa、mybatis时如何使用多数据源？



## Spring Boot 工程改造成多模块以及打包（Spring Boot版本：2.1.3.RELEASE）

### 改造成多模块：计划三个模块，web，domain，repository

* 新建web模块，把所有代码copy到web模块

* 新建repository模块，把web模块中关于repository中的代码copy到repository中，并在web模块中添加repository依赖

* 新建domain模块，把web模块中关于domain中的代码copy到repository中，并在repository添加domain的依赖。这样，在web模块中就间接依赖domain模块

  

### 打包

* 把主工程的build插件移动到web模块下的pom.xml文件中，如下

  ```xml
  <build>
      <plugins>
          <plugin>
              <groupId>org.springframework.boot</groupId>
              <artifactId>spring-boot-maven-plugin</artifactId>
              <configuration>
                  <!-- 主类（手工添加） -->
                  <mainClass>com.example.springweb.SpringwebApplication</mainClass>
              </configuration>
          </plugin>
      </plugins>
  </build>
  ```

* 打成jar包-

  * 执行打包命令：`mvn -Dmaven.test.skip -U clean package`
* 打成war包
  * 在web模块的pom.xml文件中添加`<packaging>war</packaging>`
  * 执行命令：`mvn -Dmaven.test.skip -U clean package`

### 启动

* 常规启动：在jar目录执行命令->`java -jar web-0.0.1-SNAPSHOT.jar --Dserver.port=0`

  > 注意： --Dserver.port=0，启动一个随机端口

* 目录方式启动（该方式解决老旧的jar不支持Spring Boot的情况）：

  * 将jar文件解压
  * 进入解压的目录，执行命令`java org.springframework.boot.loader.JarLauncher`
  * 如果是war包，则命令变为`java org.springframework.boot.loader.WarLauncher`





## Spring Boot Validation

### 1、使用自带的注解验证

* 1）在pom.xml中增加依赖

  ```xml
  <dependency>
     <groupId>org.springframework.boot</groupId>
     <artifactId>spring-boot-starter-validation</artifactId>
  </dependency>
  ```

 * 2）在实体类上增加注解。如@NotBlank，@Max等，已定义注解有如下：
   * @AssertFalse
   * @AssertTrue
   * @DecimalMax
   * @DecimalMin
   * @Digits
   * @Email
   * @Future
   * @FutureOrPresent
   * @Max
   * @Min
   * @Negative
   * @NegativeOrZero
   * @NotBlank
   * @NotEmpty
   * @NotNull
   * @Null
   * @Past
   * @PastOrPresent
   * @Pattern
   * @Positive
   * @PositiveOrZero
   * @Size
 * 3）在Controller的参数里面增加@Valid注解

### 2、自定义验证注解

 * 1） 创建注解 `@ValidCardNumber`

   ```java
   package com.example.sourcelearn.validation.constraints;
   
   import com.example.sourcelearn.validation.ValidCardNumberConstraintValidator;
   
   import javax.validation.Constraint;
   import javax.validation.Payload;
   import java.lang.annotation.Documented;
   import java.lang.annotation.Repeatable;
   import java.lang.annotation.Retention;
   import java.lang.annotation.Target;
   
   import static java.lang.annotation.ElementType.FIELD;
   import static java.lang.annotation.RetentionPolicy.RUNTIME;
   
   @Target({ FIELD }) // 字段
   @Retention(RUNTIME)
   @Repeatable(ValidCardNumber.List.class)
   @Documented
   @Constraint(validatedBy = { ValidCardNumberConstraintValidator.class })
   public @interface ValidCardNumber {
   
       String message() default "{javax.validation.constraints.card.number.message}";
   
       Class<?>[] groups() default { };
   
       Class<? extends Payload>[] payload() default { };
   
       @Target({ FIELD })
       @Retention(RUNTIME)
       @Documented
       public @interface List {
           ValidCardNumber[] value();
       }
   }
   ```

* 2) 创建注解实现类  `ValidCardNumberConstraintValidator`

  ```java
  package com.example.sourcelearn.validation;
  
  import com.example.sourcelearn.validation.constraints.ValidCardNumber;
  import org.apache.commons.lang3.ArrayUtils;
  import org.apache.commons.lang3.StringUtils;
  
  import javax.validation.ConstraintValidator;
  import javax.validation.ConstraintValidatorContext;
  
  /**
   * @Author zhanglin
   * @Date 2019/3/10 11:01
   * @Verson 1.0
   */
  public class ValidCardNumberConstraintValidator
          implements ConstraintValidator<ValidCardNumber, String> {
  
      @Override
      public void initialize(ValidCardNumber constraintAnnotation) {
  
      }
  
      /**
       * 验证规则：GUPAO-开头，数字结尾
       * @param value
       * @param context
       * @return
       */
      @Override
      public boolean isValid(String value, ConstraintValidatorContext context) {
          // 这里的分割尽量不要用String#split
          // 这里用的Apache的分割
          String[] parts = StringUtils.split(value, "-");
  
          if (ArrayUtils.getLength(parts) != 2) {
              return false;
          }
  
          boolean isValidPrex = "GUPAO".equals(parts[0]);
          boolean isValidSuffix = StringUtils.isNumeric(parts[1]);
  
          return isValidPrex && isValidSuffix;
      }
  }
  ```

* 3） 在POJO类的字段上面增加注解

  ```java
  package com.example.sourcelearn.domain;
  
  import com.example.sourcelearn.validation.constraints.ValidCardNumber;
  import lombok.Data;
  import lombok.ToString;
  
  import javax.validation.constraints.NotBlank;
  
  
  @Data
  @ToString
  public class User {
      private Integer id;
  
      //@NotNull(message = "姓名不能为空")
      @NotBlank(message = "姓名不能为空")
      private String name;
  
      // 必须以 GUPAO-开头，数字结尾
      @ValidCardNumber
      private String cardNumber;
  
      public User() {
          //this(1, "小马哥");
      }
  
      public User(Integer id, String name) {
          this.id = id;
          this.name = name;
      }
  
  }
  ```

* 国际化：因为spring boot validation用的是hibernate，所以在resource文件下增加国际化资源配置文件`ValidationMessages.properties`和`ValidationMessages.properties`

  ```properties
  javax.validation.constraints.card.number.message=cardNumer must start with GUPAO,and must end with number
  ```

  > 注意：这里的键为`@ValidCardNumber` 中定义的message

