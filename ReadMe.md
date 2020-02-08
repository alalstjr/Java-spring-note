--------------------
# Java Spring Note
--------------------

# 목차

- [1. Validator 검증](#Validator-검증)
    - [1. Validator 필드 검증](#Validator-필드-검증)
        - [1. Reflection-API](#Reflection-API)

# Validator 검증

## Validator 필드 검증

~~~
@Getter
@Setter
public class AccountSaveDTO {
    ...

    @NotBlank(message = "비밀번호를 작성해 주세요.")
    private String password;

    @NotBlank(message = "비밀번호 확인을 작성해 주세요.")
    private String passwordRe;

    ...
}
~~~

사용자로부터 받은 password 와 passwordRe 가 동일하지 않다면 검증 실패로 에러메세지를 출력하도록 
커스텀 Validator 어노테이션을 제작하는 방법을 필기하였습니다.

위와같은 경우 `메소드 두개를 비교`해야 하는 경우이므로 해당 메소드의 어노테이션이 아니라
`필드를 검증해야하는 어노테이션을 제작`해야 합니다.

> @interface.PasswordMatch

~~~
@Documented
@Constraint(validatedBy = PasswordMatchValidator.class)
@Target({ ElementType.TYPE, ElementType.ANNOTATION_TYPE })
@Retention(RetentionPolicy.RUNTIME)
public @interface PasswordMatch {
    String message() default "비밀번호가 일치하지 않습니다.";

    String password();

    String passwordRe();

    Class<?>[] groups() default { };

    Class<? extends Payload>[] payload() default { };
}
~~~

> PasswordMatchValidator.class

~~~
public class PasswordMatchValidator implements ConstraintValidator<PasswordMatch, Object> {

    private String password;
    private String passwordRe;
    private String message;

    /*
     * initialize() 메소드는 어노테이션으로 받은 값을 해당 필드에 초기화 선언을 합니다.
     * */
    @Override
    public void initialize(PasswordMatch constraintAnnotation) {
        password = constraintAnnotation.password();
        passwordRe = constraintAnnotation.passwordRe();
        message = constraintAnnotation.message();
    }

    @Override
    public boolean isValid(Object value, ConstraintValidatorContext context) {

        boolean valid = true;

        try {
            Object passwordCheck = BeanUtils.getProperty(
                    value,
                    password
            );
            Object passwordReCheck = BeanUtils.getProperty(
                    value,
                    passwordRe
            );

            valid = passwordCheck == null && passwordReCheck == null || passwordCheck != null && passwordCheck.equals(passwordReCheck);
        }
        catch (Exception e) {
            e.printStackTrace();
        }

        if (!valid) {
            context
                    .buildConstraintViolationWithTemplate(message)
                    .addPropertyNode(password)
                    .addConstraintViolation()
                    .disableDefaultConstraintViolation();

            // context.buildConstraintViolationWithTemplate(
            //      MessageFormat.format("Email {0} already exists!", value)
            //      직접 메세지를 선언해 주기위해서 MessageFormat 사용할 수 도 있다.
            // )
        }

        return valid;
    }
}
~~~

[ConstraintValidator](https://docs.oracle.com/javaee/7/api/javax/validation/ConstraintValidator.html)

`ConstraintValidator` 인터페이스를 상속받음으로서 커스텀 Validator 설계 도면을 생성합니다.

`initialize()` 메소드는 어노테이션으로 받은 값을 `해당 필드에 초기화 선언`을 합니다.
constraintAnnotation 값은 어노테이션의 Value 값이 전달되어 옵니다.

`isValid()` 메소드에서 검증을 실시합니다.
Object value 값으로 어노테이션이 선언된 필드값이 전달되어 옵니다.
해당 Object의 객체 정보를 받아와서 메소드값을 비교하여 검증을 실시하면 됩니다 만
전달된 `Object의 구체적인 클래스 타입을 알지 못해 접근할 수 가 없습니다.`
접근하기 위해서 `리플렉션(reflection)`을 활용하여 그 클래스의 메소드, 타입, 변수에 접근해야 합니다.

리플렉션 API로 `Apache Common BeanUtils` 활용하였습니다.

[ConstraintValidatorContext](https://docs.oracle.com/javaee/7/api/javax/validation/ConstraintValidatorContext.html)

addConstraintViolation 메소드를 통해서 에러 메시지와 검증한 node key 값을 넘겨줍니다.

disableDefaultConstraintViolation 메소드는
ConstraintViolation제약 조건에 선언 된 메시지 템플릿을 사용 하는 기본 개체 생성을 비활성화합니다 .
다른 위반 메시지를 설정하거나 ConstraintViolation 다른 속성을 기반으로 생성하는 데 유용 합니다.

### Reflection API

BeanUtils 를 사용하기 위해서는 `프로퍼티(property) 에 접근할 수 있는 set, get 메소드가 제공`되어야 한다.
즉, firstName 프로퍼티에 접근하기 위한 getFirstName(), setFirstName(String) 메소드가 있어야 한다.

BeanUtils 에서 가장 자주 쓰이는 유틸클래스는 PropertyUtils 클래스이며 
PropertyUtils 클래스는 3가지 프로퍼티 유형에 따른 접근 방법을 제공한다.

Simple - Employess의 firstName, lastName 프로퍼티 처럼 `하나의 값을 인자로 전달`하는 경우
Indexed - Employess의 subordinate 프로퍼티 처럼 값을 가져오기 위해 `index 정보가 요구`되는 경우
Mapped - Employess의 address 프로퍼티 처럼 값을 가져오기 위해 `key 정보가 요구`되는 경우

- JavaBean 프로퍼티에 접근하는 방법

- Simple

~~~
Employee emp = new Employee();

PropertyUtils.setSimpleProperty( emp, firstName, "Hong");
<=> emp.setFirstName( "Hong");

PropertyUtils.getSimpleProperty( emp, firstName);
<=> emp.getFirstName();

<위, 아래 두개의 코드가 하는 결과는 동일하다>
~~~

- Indexed

~~~
PropertyUtils.getIndexedProperty( emp, "subordinate[0]");
PropertyUtils.getIndexedProperty( emp, "subordinate", 0);
=> 위 두 메소드의 결과는 같다. getIndexedProperty 를 사용할 메소드는 int index 인자를 갖고 있어야 한다.
subordinate[0] 는 getSubordinate 메소드로 리턴되는 결과의 0 번째 라는 의미이다.

PropertyUtils.setIndexedProperty( emp, "subordinate[0]", subordinate);
PropertyUtils.setIndexedProperty( emp, "subordinate", 0, subordinate);
=> setIndexedProperty 메소드드 또한 int index 인자를 갖고 있어야 한다.
~~~

- Mapped

~~~
PropertyUtils.getMappedProperty( emp, "address(home)");
PropertyUtils.getMappedProperty( emp, "address", "home");
=> 위 두 메소드의 실행 결과는 같다

PropertyUtils.setMappedProperty( emp, "address(home)", address);
PropertyUtils.setMappedProperty( emp, "address", "home", address);
~~~

- 2.3 Nested Property Access

~~~
String city = emp.getAddress( "home").getCity();
위 처럼 Adress -> home -> city 로 여러번의 걸쳐 정보를 가져오는 경우 
String city = (String)PropertyUtils.getNestedProperty( emp, "address(home).city");

위에 설명한 indexed, mapped 를 혼합하여 사용이 가능한다.
String city = (String)PropertyUtils.getProperty( emp, "subordinate[3].address(home).city");
~~~

- 출처

[Field Matching Bean Validation Annotation Example](https://memorynotfound.com/field-matching-bean-validation-annotation-example/)
[자바의 리플렉션](https://brunch.co.kr/@kd4/8)
[Apache Common BeanUtils](https://ismydream.tistory.com/169)
[Spring validator & Hibernate validator - 선언적인 방식 유효성 검증기](https://syaku.tistory.com/346)
