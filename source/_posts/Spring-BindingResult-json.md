---
title: Spring BindingResult를 json으로 받기
date: 2018-03-10 08:58:29
tags: [spring, json, bindingresult]
categories: 
  - [practice]
  - [spring]
---

# 서론

Spring은 Controller에서 Validation을 한 후, 유효하지 못한 값이 존재할 때, Error(BindingResult)에 그 내용을 담아서, JSP, FreeMarker 등의 View Template Engine으로 오류 내용을 MessageSource로 국제화하여 보여줄 수 있도록 지원을 하고 있다.

하지만 의외로 그러한 국제화 메시지 지원을 Json 응답으로 보여주려고 할 때에는 편리한 수단이 잘 보이지 않고, Reference를 뒤져봐도 딱 맘에 드는 방법을 알려주고 있지 않는다. 때문에 Json으로 국제화된 오류 내용을 받을 수 있도록, View의 내용을 커스터마이징 하는 방법에 대해 알아보려 한다.

<!-- more -->

# 기본 동작 소스 코드

**의존성**

spring-boot : 2.0.0.RELEASE
lombok : 1.16.18

**AdderController.java**

아래는 `POST /add?a=1&b=2`를 요청했을 때, 유효성 검사를 한 후 `{"result":3}`를 반환하는 소스이다.

```java
@RestController
@RequestMapping("/add")
public class AdderController {
    private final Validator adderRequestValidator;
    @Autowired
    public AdderController(@Qualifier("adderRequestValidator") Validator adderRequestValidator) {
        this.adderRequestValidator = adderRequestValidator;
    }
    @PostMapping
    public AdderResult add(AdderRequest request, BindingResult bindingResult) throws BindException {
        adderRequestValidator.validate(request, bindingResult);
        if (bindingResult.hasErrors()) {
            throw new BindException(bindingResult);
        }
        return new AdderResult(request.getA() + request.getB());
    }
}
```

**AdderRequestValidator.java**

검증에 대한 구현이다. `a`나 `b`가 비어있으면 `field.required` 코드 값으로 `errors`의 `filedErrors`에 오류 내용이 추가된다.

```java
@Component
public class AdderRequestValidator implements Validator {
    @Override
    public boolean supports(Class<?> clazz) {
        return AdderRequest.class.isAssignableFrom(clazz);
    }
    @Override
    public void validate(Object o, Errors errors) {
        AdderRequest request = AdderRequest.class.cast(o);
        if (request.getA() == null) {
            errors.rejectValue("a", "field.required");
        }
        if (request.getB() == null) {
            errors.rejectValue("b", "field.required");
        }
    }
}
```

**error.xml**

`field.required`의 내용을 해석하여, 국제화할 xml property를 정의했다.

```xml
<?xml version="1.0" encoding="UTF-8" standalone="no"?>
<!DOCTYPE properties SYSTEM "http://java.sun.com/dtd/properties.dtd">
<properties>
    <entry key="field.required.adderRequest.a">a를 입력해주세요</entry>
    <entry key="field.required.adderRequest.b">b를 입력해주세요</entry>
</properties>
```

## 유효하지 못한 요청이 있을 때

아무런 커스터마이징 없이 `POST /add?a=1`로 요청을 보내면, `BindException`의 내용을 View로 응답한다.
그 내용은 아래와 같다.

```json
{
  "timestamp": 1519659376474,
  "status": 400,
  "error": "Bad Request",
  "exception": "org.springframework.validation.BindException",
  "errors":[{
    "codes":[
      "field.required.adderRequest.b",
      "field.required.b",
      "field.required.java.lang.Integer",
      "field.required"
    ],
    "arguments": null,
    "defaultMessage": null,
    "objectName": "adderRequest",
    "field": "b",
    "rejectedValue": null,
    "bindingFailure": false,
    "code": "field.required"
  }],
  "message": "Validation failed for object='adderRequest'. Error count: 1",
  "path": "/add"
}
```

응답 내용에 `error.xml`에서 정의한 메시지 내용이 내려가지 않고 있다.
(defaultMessage를 정의하면 해당 값은 채워져 가겠지만, 국제화가 적용되지 않는다)

클라이언트에서 언어 관련 resource를 들고 있고 codes를 적절히 대조해서 가져올 수 있다면 다행이다.
하지만 클라이언트가 하나가 아니라면 국제화 처리하는 로직에 중복이 발생하므로 서버에서 내려주는 것이 효율적일 것 같다.

# 서버에서 국제화 메시지로 변경하여 내려주기

**응답모델 정의**

모델은 어떤 자료구조로 하든 상관없다. 각자 팀의 혹은 Client의 취향에 맞추어 개발하자.
여기서는 아래와 같은 json이 나오도록 정의한다.

```json
{
  "errors":[{
   "objectName": "adderRequest",
    "field": "b",
    "code": "field.required",
    "message": "b를 입력해주세요"
  }]
}
```

**ValidationResult**

오류의 목록(`errors`)를 가지고 있다. 필요하다면 공통된 속성을 추가로 정의할 수 있다.

```java
@Value
@AllArgsConstructor(access = AccessLevel.PRIVATE)
public class ValidationResult {
   private List<FieldErrorDetail> errors;

   public static ValidationResult create(Errors errors, MessageSource messageSource, Locale locale) {
      List<FieldErrorDetail> details =
            errors.getFieldErrors()
               .stream()
               .map(error -> FieldErrorDetail.create(error, messageSource, locale))
               .collect(Collectors.toList());
      return new ValidationResult(details);
   }
}
```

**FieldErrorDetail**

`FieldError`의 상세를 기술하는 클래스다.

```java
@Value
@AllArgsConstructor(access = AccessLevel.PRIVATE)
public class FieldErrorDetail {
   private String objectName;
   private String field;
   private String code;
   private String message;

   public static FieldErrorDetail create(FieldError fieldError, MessageSource messageSource, Locale locale) {
      return new FieldErrorDetail(
         fieldError.getObjectName(),
         fieldError.getField(),
         fieldError.getCode(),
         messageSource.getMessage(fieldError, locale)); // 이 부분이 포인트
   }
}
```
`messageSource.getMessage(MessageSourceResolvable, Locale)`를 사용해서 xml에 정의한 국제화 메시지를 가져올 수 있다.
이것이 가능한 이유는 [FieldError](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/validation/FieldError.html)가 `MessageSourceResolvable`을 구현하고 있기 때문이다.

## ExceptionHandler 정의

API서버라면 `@RestControllerAdvice` 등을 써서 컨트롤러 어드바이스에 등록시키는 것도 좋은 방법이다

```java
@ExceptionHandler(BindException.class)
@ResponseStatus(HttpStatus.BAD_REQUEST)
public ValidationResult handleBindException(BindException bindException, Locale locale) {
   return ValidationResult.create(bindException, messageSource, locale);
}
```

**응답**

```json
{
  "errors":[{
   "objectName": "adderRequest",
    "field": "b",
    "code": "field.required",
    "message": "b를 입력해주세요"
  }]
}
```
## 결론

기본적으로 Spring은 `BindException`에 대해서 Exception의 내용을 json으로 보여주기만 한다. 때문에 code값만 찍혀서 나오는데, 결국 code에 대응하는 국제화 메시지를 클라이언트에서 해석해야한다. 하지만 여러 클라이언트에 대응하기 위해서는 서버에서 국제화 코드를 해석해서 주는 것이 낫다.
국제화된 메시지를 가져오기 위해서는 `BindException`에서 `FieldError`들을 가져와 messageSource를 이용해야 한다. 위에서 설명한 커스터마이징을 거쳐야 원하는 국제화 메시지를 가져올 수 있다는 점이 조금 아쉽다.

소스 : https://github.com/supawer0728/spring-bindingresult-json