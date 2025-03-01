# Spring Boot: Under the Hood

- Spring: unroasted chicken
- Spring Boot: roaster chicken
- a lot of configuration (XML) in Spring
- an idea is very simple: starter dependencies rely on `spring-boot-autoconfigure`
- e.g. `spring-boot-starter-data-jpa` relies on the m
  - in its `META-INF` there is `spring.factories` file which contains the key `org.springframework.boot.autoconfigure.EnableAutoConfiguration` with a list of autoconfiguration classes
  - there is also `org.springframework.boot.autoconfigure.data.jpa.JpaRepositoriesAutoConfiguration`

### Annotations

```java
@Configuration(
    proxyBeanMethods = false
)
@ConditionalOnBean({DataSource.class})
@ConditionalOnClass({JpaRepository.class})
@ConditionalOnMissingBean({JpaRepositoryFactoryBean.class, JpaRepositoryConfigExtension.class})
@ConditionalOnProperty(
    prefix = "spring.data.jpa.repositories",
    name = {"enabled"},
    havingValue = "true",
    matchIfMissing = true
)
@Import({JpaRepositoriesRegistrar.class})
@AutoConfigureAfter({HibernateJpaAutoConfiguration.class, TaskExecutionAutoConfiguration.class})
public class JpaRepositoriesAutoConfiguration {
```

```java
@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.TYPE, ElementType.METHOD})
@Documented
@Conditional(ProfileCondition.class)
public @interface Profile {
  String[] value;
}

```

- `@Conditional` & `Condition`

  - @Conditional contains the array of conditions (Condition type)
  - `Condition` (only one boolean method) defines whether the bean method will be called or not depending on its `matches` method
    - `ProfileCondition`
    - `@ConditionalOnBean` - if the bean exists
    - `@ConditionalOnClass` - if the class is loaded
    - `@ConditionalOnMissingClass` - if the class is missing
    - `@ConditionalOnMissingBean` - if the bean is missing
    - `@ConditionalOnProperty` - if the property is defined
    - .....
    - plus - `AllNestedConditions` (AND), `AnyNestedConditions` (OR), `NoneNestedCondition` (NOT)
  - if the condition is met, the bean is created and added to Spring context

- `@PostConstruct` - Spring calls this method only once, just after the initialization of bean properties
- `@PreDestroy` - Spring calls this method only once, just before the bean removal from the application context
- a good way to make a bean configuration based on the presence of the driver:

```java
@Bean
@ConditionalOnMissingBean(XStream.class)
public XStream xstream(Optional<HierarchicalStreamDriver> driver) {
  // if the custom config code does not define a HierarchicalStreamDriver bean, the XppDriver will be used
  return new XStream(driver.orElse(new XppDriver()));
}

```
