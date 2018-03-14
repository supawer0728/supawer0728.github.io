---
title: Spring BindingResult를 json으로 받기
date: 2018-03-10 08:58:29
tags: [spring, json, bindingresult]
categories: 
  - [practice]
  - [spring]
---

# 개요

Spring은 Controller에서 Validation을 한 후, 유효하지 못한 값이 존재할 때,
Error(BindingResult)에 그 내용을 담아서,
JSP, FreeMarker 등의 View Template Engine으로
오류 내용을 MessageSource로 국제화하여 보여줄 수 있도록 지원을 하고 있다

하지만 의외로 그러한 국제화 메시지 지원을 Json 응답으로 보여주려고 할 때에는
편리한 수단이 잘 보이지 않고, Reference를 뒤져봐도 딱 맘에 드는 방법을 알려주고 있지 않는다

<!-- more -->

## 소스 코드

`POST /add?a=1&b=2`를 요청했을 때 `{"result":3}`를 반환하는 소스

**AdderController.java**

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

```xml
<?xml version="1.0" encoding="UTF-8" standalone="no"?>
<!DOCTYPE properties SYSTEM "http://java.sun.com/dtd/properties.dtd">
<properties>
    <entry key="field.required.adderRequest.a">a를 입력해주세요</entry>
    <entry key="field.required.adderRequest.b">b를 입력해주세요</entry>
</properties>
```

## 유효하지 못한 요청이 있을 때

`POST /add?a=1`로 요청을 보내면, BindException의 내용을 아래와 같이 내려준다

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

응답에 내용에 `error,xml`에서 정의한 메시지 내용이 내려가지 않고 있다
(defaultMessage를 적어줬다면을 써먹을 수는 있겠으나, 국제화가 적용되지 않는다)


클라이언트에서 언어 관련 resource를 들고 있고 codes를 적절히 대조해서 가져올 수 있다면 다행이지만,
클라이언트가 하나가 아니라면 국제화 처리하는 로직에 중복이 발생하므로
서버쪽에서 내려주는 것이 효율적으로 판단했다

## 서버에서 국제화 메시지로 변경하여 내려주기

**응답모델 정의**

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
messageSource.getMessage(MessageSourceResolvable, Locale) API를 사용해서
여기서 MessageSourceResolvable을 FieldError가 구현을 하고 있어서 저렇게 가져올 수 있다

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

기본적으로 Spring은 BindException에 대해서 DefaultHandlerExceptionResolver로 Exception의 내용을 json으로 보여주기만 한다
국제화된 메시지를 가져오기 위해선는 BindException에서 FieldError들을 가져와 messageSource에 던져줘야 한다

소스 : https://github.com/supawer0728/spring-bindingresult-json