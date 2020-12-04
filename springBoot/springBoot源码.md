

## SpringBoot源码



## 1. SpingBoot注解

`@SpringBootApplication`注解是springboot的核心注解，它主要包括三个注解，分别是

- **@SpringBootConfiguration：**它包含了`@Configuration`注解，是配置类注解。
- **@EnableAutoConfiguration：**主动装配注解，它最主要的功能是通过`@Improt`注解导入了`AutoConfigurationImportSelector`组件。

- **@ComponentScan：**扫描包注解。

