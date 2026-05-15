## <font style="color:rgb(51, 51, 51);">问题背景</font>
<font style="color:#DF2A3F;">报错: The specified log seems to be in use already: tmlog in ./\. Make sure that no other instance is running, or kill any pending process if needed.</font>

<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2026/png/26718473/1774433119898-4f53564c-097f-4178-9e5f-852857839af8.png)

<font style="color:rgb(51, 51, 51);">在使用 Spring Boot 3.3 + Atomikos 实现分布式事务时，发现事务日志文件（tmlog）总是生成在项目根目录，而不是期望的模块目录下。多个模块的 tmlog 文件混在一起，难以区分和管理。</font>

<font style="color:rgb(51, 51, 51);"></font>

> 在多模块的情况下,因为<font style="color:rgb(51, 51, 51);">tmlog默认都是在父级模块下,所以每个模块都是共用一个tmlog,因为共用一个tmlog所以所有模块在同一时间只能启动一个,不能同时运行,但是部署在服务器不存在这个问题,因为在服务器上部署时每个模块的jar包不在想他目录,所以不会出现这个问题,主要解决的就是开发环境想要同时启动多个模块的问题,方便调试</font>
>

## <font style="color:rgb(51, 51, 51);">环境信息</font>
+ **<font style="color:rgb(51, 51, 51);">JDK</font>**<font style="color:rgb(51, 51, 51);">: 21</font>
+ **<font style="color:rgb(51, 51, 51);">Spring Boot</font>**<font style="color:rgb(51, 51, 51);">: 3.3.0</font>
+ **<font style="color:rgb(51, 51, 51);">Atomikos</font>**<font style="color:rgb(51, 51, 51);">: 6.0.0 (transactions-spring-boot3-starter)</font>
+ **<font style="color:rgb(51, 51, 51);">项目结构</font>**<font style="color:rgb(51, 51, 51);">: Maven 多模块</font>

## <font style="color:rgb(51, 51, 51);">问题现象</font>
<font style="color:rgb(51, 51, 51);">默认配置下，所有模块的事务日志都输出到项目根目录：</font>

```shell
dog-auto-servers/
├── tmlog.lck
├── tmlog0.log
├── tmlog1.log
└── task-formula-ai/
└── task-auto-test/
```

## <font style="color:rgb(51, 51, 51);">错误尝试</font>
### <font style="color:rgb(51, 51, 51);">方式一：自定义 TransactionManagerConfig Bean</font>
<font style="color:rgb(51, 51, 51);">最初尝试通过 Java 配置类手动创建 Bean：</font>

```java
@Configuration
public class TransactionManagerConfig {

    @Bean(name = "atomikosTransactionManager", initMethod = "init", destroyMethod = "close")
    public UserTransactionManager atomikosTransactionManager() throws Throwable {
        UserTransactionManager userTransactionManager = new UserTransactionManager();
        userTransactionManager.setForceShutdown(false);
        // ❌ 这样设置无效！
        return userTransactionManager;
    }

    @Bean(name = "transactionManager")
    public JtaTransactionManager transactionManager() throws Throwable {
        return new JtaTransactionManager(userTransaction(), atomikosTransactionManager());
    }
}
```

**<font style="color:rgb(51, 51, 51);">结果</font>**<font style="color:rgb(51, 51, 51);">：YAML 中的配置完全不生效，日志仍在根目录。</font>

### <font style="color:rgb(51, 51, 51);">方式二：在 YAML 中配置但保留自定义 Bean</font>
```yaml
spring:
  jta:
    atomikos:
      properties:
        log-base-dir: ./logs/atomikos
        log-base-name: task-formula-ai-tmlog
        enable-logging: true
        tm-unique-name: task-formula-ai
```

```java
@Configuration
public class TransactionManagerConfig {

    @Bean(name = "atomikosTransactionManager")
    public UserTransactionManager atomikosTransactionManager() throws Throwable {
        // ❌ 仍然无效！因为自定义 Bean 覆盖了自动配置
        UserTransactionManager userTransactionManager = new UserTransactionManager();
        return userTransactionManager;
    }
}
```

**<font style="color:rgb(51, 51, 51);">结果</font>**<font style="color:rgb(51, 51, 51);">：配置依然不生效。</font>**<font style="color:rgb(51, 51, 51);">原因：自定义 Bean 会覆盖 Spring Boot 的自动配置</font>**<font style="color:rgb(51, 51, 51);">。</font>

## <font style="color:rgb(51, 51, 51);">正确解决方案</font>
### <font style="color:rgb(51, 51, 51);">✅</font><font style="color:rgb(51, 51, 51);"> 方案：删除自定义配置类，使用 Spring Boot 自动配置</font>
#### <font style="color:rgb(51, 51, 51);">1. 删除 TransactionManagerConfig.java</font>
<font style="color:rgb(51, 51, 51);">直接删除自定义的配置类文件：</font>

TransactionManagerConfig.java

#### <font style="color:rgb(51, 51, 51);">2. 在 application.yml 中配置</font>
```yaml
spring:
  application:
    name: task-formula-ai  # 服务名称
  jta:
    atomikos:
      properties:
        log-base-dir: ./logs/atomikos           # 事务日志目录（相对路径）
        log-base-name: ${spring.application.name}-tmlog  # 日志文件基础名称
        enable-logging: true                    # 启用磁盘日志
        tm-unique-name: ${spring.application.name}       # 事务管理器唯一标识
        max-actives: 10000                      # 最大活动事务数
        default-jta-timeout: 300000             # 默认事务超时时间（毫秒）
        force-shutdown-on-vm-exit: false        # VM 关闭时是否强制关闭
```

#### <font style="color:rgb(51, 51, 51);">3. 多模块配置示例</font>
<font style="color:rgb(51, 51, 51);">为每个模块设置不同的服务名称，日志文件就会自动区分开：</font>

**<font style="color:rgb(51, 51, 51);">task-formula-ai:</font>**

```yaml
spring:
  application:
    name: task-formula-ai
```

**<font style="color:rgb(51, 51, 51);">task-auto-test:</font>**

```yaml
spring:
  application:
    name: task-auto-test
```

#### <font style="color:rgb(51, 51, 51);">4. 重启验证</font>
<font style="color:rgb(51, 51, 51);">清理并重启应用后，日志文件结构变为：</font>

```shell
dog-auto-servers/
├── task-formula-ai/
│   └── logs/
│       └── atomikos/
│           ├── task-formula-ai-tmlog.lck
│           └── task-formula-ai-tmlog0.log
└── task-auto-test/
    └── logs/
        └── atomikos/
            ├── task-auto-test-tmlog.lck
            └── task-auto-test-tmlog0.log
```

## <font style="color:rgb(51, 51, 51);">核心原理</font>
### <font style="color:rgb(51, 51, 51);">为什么自定义 Bean 会导致配置失效？</font>
<font style="color:rgb(51, 51, 51);">Spring Boot 的自动配置类 </font>`<font style="color:rgb(51, 51, 51);background-color:rgb(243, 244, 244);">AtomikosJtaConfiguration</font>`<font style="color:rgb(51, 51, 51);"> 使用了条件注解：</font>

```java
@Configuration
@ConditionalOnMissingBean(UserTransactionManager.class)
public class AtomikosJtaConfiguration {

    @Bean
    @ConfigurationProperties(prefix = "spring.jta.atomikos.properties")
    public UserTransactionManager userTransactionManager() {
        // 只有没有自定义 Bean 时才会创建
        // 这里会自动读取 YAML 配置
    }
}
```

<font style="color:rgb(51, 51, 51);">当你手动定义了 </font>`<font style="color:rgb(51, 51, 51);background-color:rgb(243, 244, 244);">UserTransactionManager</font>`<font style="color:rgb(51, 51, 51);"> Bean 后：</font>

+ `<font style="color:rgb(51, 51, 51);background-color:rgb(243, 244, 244);">@ConditionalOnMissingBean</font>`<font style="color:rgb(51, 51, 51);"> 条件不满足</font>
+ <font style="color:rgb(51, 51, 51);">自动配置类不会生效</font>
+ <font style="color:rgb(51, 51, 51);">YAML 中的 </font>`<font style="color:rgb(51, 51, 51);background-color:rgb(243, 244, 244);">spring.jta.atomikos.properties.*</font>`<font style="color:rgb(51, 51, 51);"> 配置不会被读取</font>
+ <font style="color:rgb(51, 51, 51);">Atomikos 使用默认配置（日志输出到当前目录）</font>

### <font style="color:rgb(51, 51, 51);">Atomikos 配置属性映射</font>
| **<font style="color:rgb(51, 51, 51);">YAML 属性</font>** | **<font style="color:rgb(51, 51, 51);">Atomikos 系统属性</font>** | **<font style="color:rgb(51, 51, 51);">说明</font>** |
| :--- | :--- | :--- |
| `<font style="color:rgb(51, 51, 51);background-color:rgb(243, 244, 244);">log-base-dir</font>` | `<font style="color:rgb(51, 51, 51);background-color:rgb(243, 244, 244);">com.atomikos.icatch.log_base_dir</font>` | <font style="color:rgb(51, 51, 51);">日志目录</font> |
| `<font style="color:rgb(51, 51, 51);background-color:rgb(243, 244, 244);">log-base-name</font>` | `<font style="color:rgb(51, 51, 51);background-color:rgb(243, 244, 244);">com.atomikos.icatch.log_base_name</font>` | <font style="color:rgb(51, 51, 51);">日志文件基础名称</font> |
| `<font style="color:rgb(51, 51, 51);background-color:rgb(243, 244, 244);">tm-unique-name</font>` | `<font style="color:rgb(51, 51, 51);background-color:rgb(243, 244, 244);">com.atomikos.icatch.tm_unique_name</font>` | <font style="color:rgb(51, 51, 51);">事务管理器唯一标识</font> |
| `<font style="color:rgb(51, 51, 51);background-color:rgb(243, 244, 244);">enable-logging</font>` | `<font style="color:rgb(51, 51, 51);background-color:rgb(243, 244, 244);">com.atomikos.icatch.enable_logging</font>` | <font style="color:rgb(51, 51, 51);">是否启用日志</font> |
| `<font style="color:rgb(51, 51, 51);background-color:rgb(243, 244, 244);">max-actives</font>` | `<font style="color:rgb(51, 51, 51);background-color:rgb(243, 244, 244);">com.atomikos.icatch.max_actives</font>` | <font style="color:rgb(51, 51, 51);">最大活动事务数</font> |
| `<font style="color:rgb(51, 51, 51);background-color:rgb(243, 244, 244);">default-jta-timeout</font>` | `<font style="color:rgb(51, 51, 51);background-color:rgb(243, 244, 244);">com.atomikos.icatch.default_jta_timeout</font>` | <font style="color:rgb(51, 51, 51);">默认事务超时</font> |


## <font style="color:rgb(51, 51, 51);">如果必须使用自定义 Bean 怎么办？</font>
<font style="color:rgb(51, 51, 51);">某些场景下可能需要自定义 Bean（比如添加额外的初始化逻辑），这时可以通过设置系统属性来配置：</font>

```java
@Configuration
public class TransactionManagerConfig {

    @Value("${spring.jta.atomikos.properties.log-base-dir:./logs/atomikos}")
    private String logDir;

    @Value("${spring.application.name:app}")
    private String appName;

    @Bean(name = "atomikosTransactionManager", initMethod = "init", destroyMethod = "close")
    public UserTransactionManager atomikosTransactionManager() throws Throwable {
        // ⚠️ 必须在 init() 调用之前设置系统属性
        System.setProperty("com.atomikos.icatch.log_base_dir", logDir);
        System.setProperty("com.atomikos.icatch.tm_unique_name", appName);
        System.setProperty("com.atomikos.icatch.log_base_name", appName + "-tmlog");
        System.setProperty("com.atomikos.icatch.enable_logging", "true");

        UserTransactionManager userTransactionManager = new UserTransactionManager();
        userTransactionManager.setForceShutdown(false);
        userTransactionManager.init();
        return userTransactionManager;
    }
}
```

## <font style="color:rgb(51, 51, 51);">最佳实践建议</font>
1. **<font style="color:rgb(51, 51, 51);">优先使用 Spring Boot 自动配置</font>**
    - <font style="color:rgb(51, 51, 51);">遵循约定优于配置原则</font>
    - <font style="color:rgb(51, 51, 51);">减少自定义代码</font>
    - <font style="color:rgb(51, 51, 51);">配置更直观、易维护</font>
2. **<font style="color:rgb(51, 51, 51);">多模块项目务必设置唯一的服务名称</font>**

```yaml
spring:
  application:
    name: ${project.artifactId}  # 或使用具体名称
```

**<font style="color:rgb(51, 51, 51);">使用相对路径配置日志目录</font>**

3. <font style="color:rgb(51, 51, 51);">log-base-dir: ./logs/atomikos  # 相对于工作目录</font>

**<font style="color:rgb(51, 51, 51);">生产环境使用绝对路径</font>**

4. <font style="color:rgb(51, 51, 51);">log-base-dir: /var/logs/${spring.application.name}/atomikos</font>
5. **<font style="color:rgb(51, 51, 51);">确保每个模块的 </font>**`**<font style="color:rgb(51, 51, 51);background-color:rgb(243, 244, 244);">tm-unique-name</font>**`**<font style="color:rgb(51, 51, 51);"> 唯一</font>**
    - <font style="color:rgb(51, 51, 51);">避免多个实例冲突</font>
    - <font style="color:rgb(51, 51, 51);">便于事务恢复和故障排查</font>

## <font style="color:rgb(51, 51, 51);">验证清单</font>
+ <font style="color:rgb(51, 51, 51);">删除了自定义的 </font>`<font style="color:rgb(51, 51, 51);background-color:rgb(243, 244, 244);">TransactionManagerConfig</font>`<font style="color:rgb(51, 51, 51);"> 配置类</font>
+ <font style="color:rgb(51, 51, 51);">在 </font>`<font style="color:rgb(51, 51, 51);background-color:rgb(243, 244, 244);">application.yml</font>`<font style="color:rgb(51, 51, 51);"> 中正确配置了 </font>`<font style="color:rgb(51, 51, 51);background-color:rgb(243, 244, 244);">spring.jta.atomikos.properties.*</font>`
+ <font style="color:rgb(51, 51, 51);">为每个模块设置了不同的 </font>`<font style="color:rgb(51, 51, 51);background-color:rgb(243, 244, 244);">spring.application.name</font>`
+ <font style="color:rgb(51, 51, 51);">清理了旧的 tmlog 文件</font>
+ <font style="color:rgb(51, 51, 51);">重启应用后检查日志目录是否正确生成</font>
+ <font style="color:rgb(51, 51, 51);">验证事务功能是否正常</font>

## <font style="color:rgb(51, 51, 51);">常见问题 FAQ</font>
### <font style="color:rgb(51, 51, 51);">Q1: 配置后仍然在根目录生成 tmlog 怎么办？</font>
**<font style="color:rgb(51, 51, 51);">A:</font>**<font style="color:rgb(51, 51, 51);"> 检查以下几点：</font>

1. <font style="color:rgb(51, 51, 51);">确认已删除所有自定义的 </font>`<font style="color:rgb(51, 51, 51);background-color:rgb(243, 244, 244);">TransactionManagerConfig</font>`<font style="color:rgb(51, 51, 51);"> 类</font>
2. <font style="color:rgb(51, 51, 51);">清理 Maven 缓存：</font>`<font style="color:rgb(51, 51, 51);background-color:rgb(243, 244, 244);">mvn clean</font>`
3. <font style="color:rgb(51, 51, 51);">删除旧的 tmlog 文件</font>
4. <font style="color:rgb(51, 51, 51);">完全重启应用（不是热重载）</font>

### <font style="color:rgb(51, 51, 51);">Q2: 多个模块可以共享同一个事务管理器吗？</font>
**<font style="color:rgb(51, 51, 51);">A:</font>**<font style="color:rgb(51, 51, 51);"> 不建议。每个模块应该有自己独立的事务管理器，通过设置不同的 </font>`<font style="color:rgb(51, 51, 51);background-color:rgb(243, 244, 244);">tm-unique-name</font>`<font style="color:rgb(51, 51, 51);"> 来区分。</font>

### <font style="color:rgb(51, 51, 51);">Q3: 可以使用环境变量动态配置吗？</font>
**<font style="color:rgb(51, 51, 51);">A:</font>**<font style="color:rgb(51, 51, 51);"> 可以，支持环境变量替换：</font>

log-base-dir: ${ATOMIKOS_LOG_DIR:./logs/atomikos}

### <font style="color:rgb(51, 51, 51);">Q4: 生产环境如何配置？</font>
**<font style="color:rgb(51, 51, 51);">A:</font>**<font style="color:rgb(51, 51, 51);"> 在生产环境配置文件中指定绝对路径：</font>

```yaml
# application-prod.yml
spring:
  jta:
    atomikos:
      properties:
        log-base-dir: /data/logs/${spring.application.name}/atomikos
        enable-logging: true
```

## <font style="color:rgb(51, 51, 51);">总结</font>
<font style="color:rgb(51, 51, 51);">Spring Boot 的自动配置机制大大简化了 Atomikos 的配置，但前提是</font>**<font style="color:rgb(51, 51, 51);">不要手动覆盖自动配置的 Bean</font>**<font style="color:rgb(51, 51, 51);">。当遇到配置不生效的问题时，首先检查是否有自定义 Bean 覆盖了自动配置。</font>

**<font style="color:rgb(51, 51, 51);">核心要点</font>**<font style="color:rgb(51, 51, 51);">：</font>

+ <font style="color:rgb(51, 51, 51);">❌</font><font style="color:rgb(51, 51, 51);"> 自定义 </font>`<font style="color:rgb(51, 51, 51);background-color:rgb(243, 244, 244);">UserTransactionManager</font>`<font style="color:rgb(51, 51, 51);"> Bean → YAML 配置失效</font>
+ <font style="color:rgb(51, 51, 51);">✅</font><font style="color:rgb(51, 51, 51);"> 删除自定义 Bean → YAML 配置生效</font>
+ <font style="color:rgb(51, 51, 51);">🔧</font><font style="color:rgb(51, 51, 51);"> 必须自定义时 → 通过 </font>`<font style="color:rgb(51, 51, 51);background-color:rgb(243, 244, 244);">System.setProperty()</font>`<font style="color:rgb(51, 51, 51);"> 设置</font>

<font style="color:rgb(51, 51, 51);">希望这篇文章能帮助你少走弯路！</font><font style="color:rgb(51, 51, 51);">🎉</font>

  
 

