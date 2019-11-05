

## 4.5版本屏蔽控制台日志输出

在自己的日志配置文件（logback-spring.xml）中添加如下配置：

```xml
	<logger name="RocketmqRemoting" additivity="false">
		<level value="WARN"/>
		<appender-ref ref="CONSOLE"/>
	</logger>

	<!-- CONSOLE如下 -->
	<!-- 控制台的日志输出样式 -->
	<property name="CONSOLE_LOG_PATTERN"
			  value="%clr(%d{yyyy-MM-dd HH:mm:ss.SSS}){faint} %clr(${LOG_LEVEL_PATTERN:-%5p}) %clr(${PID:- }){magenta} %clr(---){faint} %clr([%15.15t]){faint} %m%n${LOG_EXCEPTION_CONVERSION_WORD:-%wEx}}" />
	<!-- 控制台输出 -->
	<appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
		<filter class="ch.qos.logback.classic.filter.ThresholdFilter">
			<level>INFO</level>
		</filter>
		<!-- 日志输出编码 -->
		<encoder>
			<pattern>${CONSOLE_LOG_PATTERN}</pattern>
			<!--<charset>utf8</charset>-->
		</encoder>
	</appender>
```

