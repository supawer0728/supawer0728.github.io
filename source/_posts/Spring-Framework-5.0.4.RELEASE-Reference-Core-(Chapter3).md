---
title: Spring Framework 5.0.4.RELEASE Reference Core (Chapter3)
date: 2018-03-11 02:40:01
tags: [spring, reference]
categories:
  - spring
  - reference
---

# 글에 앞서서

- 본문은 Spring Framework Version 5의 습득을 위한 글이다.
- 이 글을 상업적 목적으로 쓰지 않았다
{% blockquote 원본 https://docs.spring.io/spring/docs/5.0.4.RELEASE/spring-framework-reference/ %}
Authors
Rod Johnson , Juergen Hoeller , Keith Donald , Colin Sampaleanu , Rob Harrop , Thomas Risberg , Alef Arendsen , Darren Davison , Dmitriy Kopylenko , Mark Pollack , Thierry Templier , Erwin Vervaet , Portia Tung , Ben Hale , Adrian Colyer , John Lewis , Costin Leau , Mark Fisher , Sam Brannen , Ramnivas Laddad , Arjen Poutsma , Chris Beams , Tareq Abedrabbo , Andy Clement , Dave Syer , Oliver Gierke , Rossen Stoyanchev , Phillip Webb , Rob Winch , Brian Clozel , Stephane Nicoll , Sebastien Deleuze

Copyright © 2004-2016

Copies of this document may be made for your own use and for distribution to others, provided that you do not charge any fee for such copies and further provided that each copy contains this Copyright Notice, whether distributed in print or electronically.
{% endblockquote %}

# Validation, Data Binding, and Type Conversion

# 3.2. Validation using Spring’s Validator interface

## 3.2. Validation using Spring’s Validator interface
<!-- more -->
```java
@lombok.Data
public class Person {
    private String name;
    private int age;
}
```
```java
public class PersonValidator implements Validator {
    @Override
    public boolean supports(Class clazz) {
        return Person.class.equals(clazz);
    }
    @Override
    public void validate(Object obj, Errors e) {
        ValidationUtils.rejectIfEmpty(e, "name", "name.empty");
        Person p = (Person) obj;
        if (p.getAge() < 0) {
            e.rejectValue("age", "negativevalue");
        } else if (p.getAge() > 110) {
            e.rejectValue("age", "too.darn.old");
        }
    }
}
```
```java
public interface Errors {
  void rejectValue(String field, String errorCode);
  void rejectValue(String field, String errorCode, String defaultValue);
}
```
# 3.3. Resolving codes to error messages

## 3.3. Resolving codes to error messages

```
public interface BindingResult extends Errors {}

public interface Errors {
  //...
  List<FieldError> getFieldErrors();
  //...
}

public class FieldError extends ObjectError {}
public class ObjectError extends DefaultMessageSourceResolvable {}
```

```
FieldError error = bindingResult.getFieldErrors.get(0);
String message = messageSource.getMessage(error, Locale.KOREAN);
```

`errors.rejectValue("a", "field.required")`가 실행됐을 때, `FieldError` 안의 codes 순서

1. `field.required.argumentName.a`
2. `field.required.a`
3. `field.required.java.lang.Integer`
4. `field.required`

# 3.4. Bean manipulation and the BeanWrapper

`Bean` : `property(getter, setter)`를 가지고 있는 오브젝트

## 3.4.1. Setting and getting basic and nested properties

```java
@Data
public class Company {
    private String name;
    private Employee managingDirector;
}

@Data
public class Employee {
    private String name;
    private float salary;
}
```
```java
BeanWrapper company = new BeanWrapperImpl(new Company());
// setting the company name..
company.setPropertyValue("name", "Some Company Inc.");
// ... can also be done like this:
PropertyValue value = new PropertyValue("name", "Some Company Inc.");
company.setPropertyValue(value);

// ok, let's create the director and tie it to the company:
BeanWrapper jim = new BeanWrapperImpl(new Employee());
jim.setPropertyValue("name", "Jim Stravinsky");
company.setPropertyValue("managingDirector", jim.getWrappedInstance());

// retrieving the salary of the managingDirector through the company
Float salary = (Float) company.getPropertyValue("managingDirector.salary");
```

## 3.4.2 Built-in PropertyEditor implementations

### Registering additional custom PropertyEditors

```java
@AllArgsConstructor
@Data
public class Genre {
  private String name;
}

@Data
public class Book {
  private Genre genre;
}
```
```xml
<bean id="novel" class="Book">
    <property name="genre" value="fiction"/>
</bean>
```
```java
public class GenreEditor extends PropertyEditorSupport {
    public void setAsText(String text) {
        setValue(new Genre(text.toUpperCase()));
    }
}
```
```xml
<bean class="org.springframework.beans.factory.config.CustomEditorConfigurer">
    <property name="customEditors">
        <map>
            <entry key="example.ExoticType" value="example.ExoticTypeEditor"/>
        </map>
    </property>
</bean>
```

사용 예) `POST https://localhost:8080/books/1 "{"genre" : "fiction"}"`
```java
@Data
public BookRequest {
  private Genre genre;
}
```

## 3.4.2 Built-in PropertyEditor implementations

### Using PropertyEditorRegistrars

```java
public final class CustomPropertyEditorRegistrar implements PropertyEditorRegistrar {
    public void registerCustomEditors(PropertyEditorRegistry registry) {
        registry.registerCustomEditor(ExoticType.class, new ExoticTypeEditor());
    }
}
```
```xml
<bean class="org.springframework.beans.factory.config.CustomEditorConfigurer">
    <property name="propertyEditorRegistrars">
        <list>
            <ref bean="customPropertyEditorRegistrar"/>
        </list>
    </property>
</bean>

<bean id="customPropertyEditorRegistrar" class="com.foo.editors.spring.CustomPropertyEditorRegistrar"/>
```
```java
public final class RegisterUserController extends SimpleFormController {

    private final PropertyEditorRegistrar customPropertyEditorRegistrar;

    public RegisterUserController(PropertyEditorRegistrar propertyEditorRegistrar) {
        this.customPropertyEditorRegistrar = propertyEditorRegistrar;
    }

    protected void initBinder(HttpServletRequest request,
            ServletRequestDataBinder binder) throws Exception {
        this.customPropertyEditorRegistrar.registerCustomEditors(binder);
    }
}
```

# 3.5. Spring Type Conversion

## 3.5.1. Converter SPI

```java
public interface Converter<S, T> {
    T convert(S source);
}
```

## 3.5.2. ConverterFactory

```java
public interface ConverterFactory<S, R> {
    <T extends R> Converter<S, T> getConverter(Class<T> targetType);
}
```

```java
final class StringToEnumConverterFactory implements ConverterFactory<String, Enum> {

    public <T extends Enum> Converter<String, T> getConverter(Class<T> targetType) {
        return new StringToEnumConverter(targetType);
    }

    private final class StringToEnumConverter<T extends Enum> implements Converter<String, T> {

        private Class<T> enumType;

        public StringToEnumConverter(Class<T> enumType) {
            this.enumType = enumType;
        }

        public T convert(String source) {
            return (T) Enum.valueOf(this.enumType, source.trim());
        }
    }
}
```
## 3.5.5. Configuring a ConversionService

```xml
<bean id="conversionService" class="org.springframework.context.support.ConversionServiceFactoryBean">
    <property name="converters">
        <set>
            <bean class="example.MyCustomConverter"/>
        </set>
    </property>
</bean>
```

## 3.5.6. Using a ConversionService programmatically

```java
@Service
public class MyService {
    @Autowired
    public MyService(ConversionService conversionService) {
        this.conversionService = conversionService;
    }
    public void doIt() {
        this.conversionService.convert(...)
    }
}
```

```java
public interface ConversionService {

    boolean canConvert(Class<?> sourceType, Class<?> targetType);
    <T> T convert(Object source, Class<T> targetType);
    boolean canConvert(TypeDescriptor sourceType, TypeDescriptor targetType);
    Object convert(Object source, TypeDescriptor sourceType, TypeDescriptor targetType);
}
```

# Spring Field Formatting

## 3.6.1. Formatter SPI

```java
public interface Formatter<T> extends Printer<T>, Parser<T> {
}

public interface Printer<T> {
    String print(T fieldValue, Locale locale);
}

public interface Parser<T> {
    T parse(String clientValue, Locale locale) throws ParseException;
}
```

```java
// 1000 -> "1,000"
public class IntegerCurrencyFormatter implements Formatter<Integer> {

    public String print(Date date, Locale locale) {
        if (date == null) {
            return "";
        }
        return insertComma(Integer.toString());
    }

    public Date parse(String formatted, Locale locale) throws ParseException {
        if (StringUtils.isBlank(formatted)) {
            return null;
        }
        return Integer.valueOf(removeComma(formatted));
    }
}
```

## 3.6.2. Annotation-driven Formatting

```java
public interface AnnotationFormatterFactory<A extends Annotation> {
    Set<Class<?>> getFieldTypes();
    Printer<?> getPrinter(A annotation, Class<?> fieldType);
    Parser<?> getParser(A annotation, Class<?> fieldType);

}
```

```java
public @interface DateTimeFormat {
  String pattern();
}

public class SomeDto {
  @DateTimeFormat(pattern = "yyyy.MM.dd HH:mm:ss")
  private Date now;
}

public final class DateTimeFormatterFactory
        implements AnnotationFormatterFactory<NumberFormat> {

    public Set<Class<?>> getFieldTypes() {
        return new HashSet<Class<?>>(asList(new Class<?>[] {
            Date.class }));
    }

    public Printer<Number> getPrinter(DateTimeFormat annotation, Class<?> fieldType) {
        return configureFormatterFrom(annotation, fieldType);
    }

    public Parser<Number> getParser(DateTimeFormat annotation, Class<?> fieldType) {
        return configureFormatterFrom(annotation, fieldType);
    }

    private Formatter<Number> configureFormatterFrom(DateTimeFormat annotation,
            Class<?> fieldType) {
        return new DateTimeFormatter(new SimpleTimeFormat(annotation.pattern()));
    }
}
```

## 3.7. Configuring a global date & time format

```java
@Configuration
public class AppConfig {

    @Bean
    public FormattingConversionService conversionService() {

        // Use the DefaultFormattingConversionService but do not register defaults
        DefaultFormattingConversionService conversionService = new DefaultFormattingConversionService(false);

        // Ensure @NumberFormat is still supported
        conversionService.addFormatterForFieldAnnotation(new NumberFormatAnnotationFormatterFactory());

        // Register date conversion with a specific global format
        DateFormatterRegistrar registrar = new DateFormatterRegistrar();
        registrar.setFormatter(new DateFormatter("yyyyMMdd"));
        registrar.registerFormatters(conversionService);

        return conversionService;
    }
}
```

# 3.8. Spring Validation

## 3.8.1. Overview of the JSR-303 Bean Validation API

```java
public class PersonForm {

    @NotNull
    @Size(max=64)
    private String name;

    @Min(0)
    private int age;

}
```

> Spring에서는 기본적으로 JSR-303 Validation의 구현체로 `Hibernate Validation`을 쓰고 있음

## 3.8.2. Configuring a Bean Validation Provider

```xml
<bean id="validator" class="org.springframework.validation.beanvalidation.LocalValidatorFactoryBean"/>
```

### Configuring Custom Constraints

`제약 애노테이션`과 `제약 로직`으로 이루어짐

```java
@Target({ElementType.METHOD, ElementType.FIELD})
@Retention(RetentionPolicy.RUNTIME)
@Constraint(validatedBy=MyConstraintValidator.class)
public @interface MyConstraint {
}

public class MyConstraintValidator implements ConstraintValidator {
    @Autowired;
    private Foo aDependency;
}
```

> 기본적으로 `LocalValidatorFactoryBean`은 Spring을 사용하여 `ConstraintValidator` 인스턴스를 생성하는 `SpringConstraintValidatorFactory`를 구성 
> 이를 통해 커스텀 `ConstraintValidator`는 다른 Spring bean처럼 의존성 주입함

### Spring-driven Method Validation

```xml
<bean class="org.springframework.validation.beanvalidation.MethodValidationPostProcessor"/>
```