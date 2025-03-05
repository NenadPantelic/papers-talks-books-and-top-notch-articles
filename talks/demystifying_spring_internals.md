# Demystifying Spring Internals

- `@SpringBootAplication` = `@EnableAutoConfiguration` + `@ComponentScan`
- `@Component` -> more specialized form = `@RestController`, `@Controller`, `@Service`, `@Repository`
- scanning works in a way the SB is actually looking into packages and subpackages in which there are classes which are annotated with of the above mentioned annotationsDemystifying Spring Internals
- component scan actually triggers the bean creation
- it starts with the bean definition - a blueprint for a bean instance
- `BeanDefinition` (metadata):
  1. Lazy - should the bean be created lazily (`@Lazy`)
  2. Primary - if we have multiple bean of the same type, this annotation defines which bean is a primary/default one for that type
  3. Scope -
  4. Type
  5. Factory method name
  6. Init method name
  7. Destroy method name
- `BeanFactory` - creates bean instances by using the bean defintions
- `BeanDefinitionRegistry` - registers the bean definitions
- `DefaultListenableBeanFactory` - implements both interfaces (`BeanFactory` and `BeanDefinitionRegistry`) - manages bean definitions and bean instances
- these components sit in `spring-beans.jar`

- `ApplicationContext` (`spring-context.jar`) is an implemention of the `BeanFactory`. If offers some enhancements to `BeanFactory`:
  1. `ApplicationEvent`
  2. `Environment`
  3. `MessageSource`
  4. `@Configuration`

MessageSource - internationalization

## Application Context

- `AbstractApplicationContext` - basic implementation that detects special beans like post processors
- `GenericApllicationContext` - extends `AbstractApplicationContext`, contains a `DefaultListableBeanFactory` (contains bean definitions and allows getting instances from those bean definitions)
- `AnnotationConfigApplicationContext` - provides `@Configuration` and support for scanning

- Example:

```java
@Configuration
class ApplicationConfiguration {

    @Bean
    public HelloMessageProvider messageProvider() {
        return new HelloMessageProvider();
    }BeanDefinition
}
```

- `@Configuration` is also `@Component` - gets picked by component scanning
- when those configuration classes are picked by component scanning, they are scanned for bean methods and when a bean method is found, a bean definition is created from it - turns the method signature into a `BeanDefinition`

```java
type = HelloMessageProvider
factoryBean = "applicationConfiguration"
factoryMethodName = "messageProvider"
```

- method `messageProvider` is invoked when the instance of `HelloMessageProvider` is needed
- type is taken from the method signature and not from the method return statement, so it's best to be most accurate when defining a bean
- when some bean should be instantiated, all its dependency beans must be created first

- `AnnotationConfigApplicationContext` uses `ConfigurationClassPostProcessor`
- it inherits `BeanDefinitionRegistryPostProcessor` - allows to modify the bean definition registry by adding more definitions to it
- `BeanDefinitionRegistryPostProcessor` inherits `BeanFactoryPostProcessor` - allows to modify the bean factory

- Another way of registering bean definitions

```java
class HelloNameProvider {

    public String getName() {
        return "SpringOne";
    }
}

class HelloMessageProvider {

    private final HelloNameProvider nameProvider;

    HelloMessageProvider(HelloNameProvider nameProvider) {
        this.nameProvider = nameProvider;
    }

    public String getMessage() {
        return "Hello " + nameProvider.getName() + "!;
    }
}

```

```java

@Configuration
class HelloConfiguration {

    @Bean
    HelloMessageProvider messageProvider(HelloProvider nameProvider) {
        return new HelloMessageProvider(nameProvider);
    }

    // static not to cause any early initialization issues

    @Bean
    static HelloPostProcessor postProcessor() {
        return new HelloPostProcessor();
    }

    static class HelloPostProcessor implements BeanDefinitionRegistryPostProcessor {

        @Override
        public void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry) throws BeanException {
            RootBeanDefinition beanDefinition = new RootBeanDefinition(HelloNameProvider.class);
            beanDefinition.setInstanceSupplier(HelloNameProvider::new);
            registry.registerBeanDefinition("nameProvider", beanDefinition);
        }

        @Override
        public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeanException {

        }
    }
}

```

### GraalVM

- introduced in Spring Boot 3 and Spring 6 in Nov 22
- it needs to know about all class that will end up in the final native image ahead of time
- it analyzes the call stack and discards all classes, fields and methods that are not used directly
- to support GraalVM, we would have to use reflection or write better code, so

  - transfrom this

  ```java

  @Configuration
  class ApplicationConfiguration {

      @Bean
      public HelloMessageProvider messageProvider() {
          return new HelloMessageProvider();
      }
  }
  ```

  - into something more like this

  ```java
      BeanDefintion bd = new RootBeanDefinition(HelloMessageProvider.class, HelloMessageProvider::new);
      registry.registerBeanDefinition("helloMessageProvider", bd);
  ```

  - transforming a code with annotations to one without is not so easy

- an alterantive way
  1. since Spring knows how to create bean definitions, run the app so we have the bean definitions
  2. stop an app before we create the bean instances
  3. create code that can recreate the bean definitions
  4. let GraalVM do its thing
