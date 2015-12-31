title: Spring Validation BindingResult/Errors always return false
tags: spring
categories: spring
---

### solution:
http://stackoverflow.com/questions/9968900/form-validation-isnt-picking-up-the-validations-on-my-account-object

### Dependency:
```xml
        <dependency>
            <groupId>javax.validation</groupId>
            <artifactId>validation-api</artifactId>
            <version>1.1.0.Final</version>
        </dependency>

        <dependency>
            <groupId>org.hibernate</groupId>
            <artifactId>hibernate-validator</artifactId>
            <version>5.1.3.Final</version>
        </dependency>
```

### Create validator bean:

```xml
<bean id="globalValidator" class="org.springframework.validation.beanvalidation.LocalValidatorFactoryBean" />
```
