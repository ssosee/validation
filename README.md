# ✅ validation
## MVC 2편 (김영한)
**목표: 스프링과 타임리프가 제공하는 검증 기능을 활용해보자.**

## 검증1 -Validation

**컨트롤러의 중요한 역할중 하나는 `HTTP 요청`이 정상인지 검증**하는 것!

그리고 정상 로직 보다 이런 검증로직을 잘 개발하는것이 더 중요 할 수도 있다.!!

### 클라이언트 검증 VS 서버 검증
* 클라이언트 검증은 조작할 수 있으므로 보안에 취약
* 서버만으로 검증하면, 즉각적인 고객 사용성 부족
* 둘을 적절히 섞어서 사용하되, 최종적으로 서버 검증은 필수
* API 방식을 사용하면 API 스펙을 잘 정의해서 검증 오류를 API 응답 결과에 잘 남겨줘야 한다.

### 검증 요구사항
* 타입 검증
  * 가격, 수량에 문자가 들어가면 검증 오류 처리
* 필드 검증
  * 상품명: 필수, 공백 X
  * 가격: 1,000원 이상, 1백만원 이하
  * 수량: 최대 9,999개
* 특정 필드의 범위를 넘어서는 검증
  * 가격 * 수량의 합은 10,000원 이상

### 상품 저장 성공
![img.png](img.png)
### 상품 저장 실패
![img_1.png](img_1.png)
<br><br>

**검증오류 보관**

`Map<String, String> errors = new HashMap<>();`

검증시 오류가 발생하면 어떤 검증에서 오류가 발생했는지 정보를 담아둔다...!

```java
@PostMapping("/add")
public String addItem(@ModelAttribute Item item, RedirectAttributes redirectAttributes, Model model) {

        //검증 오류 결과 보관
        Map<String, String> errors = new HashMap<>();

        //검증 로직
        if(!StringUtils.hasText(item.getItemName())) {
        errors.put("itemName", "상품 이름은 필수 입니다.");
        }
        if(item.getPrice() == null || item.getPrice() < 1000 || item.getPrice() > 1000000) {
        errors.put("price", "가격은 1,000 ~ 1,000,000 까지 허용합니다.");
        }
        if(item.getQuantity() == null || item.getQuantity()>= 9999) {
        errors.put("quantity", "상품 수량은 최대 9,999 까지 허용 합니다.");
        }

        //특정 필드가 아닌 복합 룰 검증
        if(item.getQuantity() != null && item.getPrice() != null) {
        int resultPrice = item.getPrice() * item.getQuantity();
        if (resultPrice < 10000) {
        errors.put("globalError", "가격 * 수량의 합은  10,000원 이상을 허용합니다. 현재 값 = "+resultPrice);
        }
        }

        //검증에 실패하면 다시 입력 폼으로
        if(!errors.isEmpty()) {
        log.info("errors = {}", errors);
        model.addAttribute("errors", errors);
        return "/validation/v1/addForm";
        }

        //성공 로직

        Item savedItem = itemRepository.save(item);
        redirectAttributes.addAttribute("itemId", savedItem.getId());
        redirectAttributes.addAttribute("status", true);
        return "redirect:/validation/v1/items/{itemId}";
        }
```

**Safe Navigation Operator**

`errors?.`는 `errors`가 `null`일때 `NullpointerException`이 발생하는 대신, `null`을 반환하는 문법이다.

`th:if` 에서 `null`은 실패로 처리되므로 오류 메시지가 출력되지 않는다.
```html
<div th:if="${errors?.containsKey('globalError')}">
  <p class="field-error" th:text="${errors['globalError']}">전체 오류 메시지</p>
</div>
```

**필드 오류 처리**
```html
<input type="text" th:classappend="${errors?.containsKey('itemName')} ? 'fielderror' : _" class="form-control">
```

**필드 오류 처리 -입력 폼 색상 적용**
```html
<input type="text" class="form-control field-error">
```

**필드 오류 처리 -메시지**
```html
<div class="field-error" th:if="${errors?.containsKey('itemName')}" th:text="${errors['itemName']}">
 상품명 오류
</div>
```

### 문제점
* 뷰 템플릿에서 중복 처리가 많다.
* 타입 오류 처리 불가.
  * `Item` 의 `price` , `quantity` 같은 숫자 필드는 타입이 `Integer` 이므로 문자
    타입으로 설정하는 것이 불가능하다. 숫자 타입에 문자가 들어오면 오류가 발생한다. 그런데 이러한 오류는
    스프링MVC에서 컨트롤러에 진입하기도 전에 예외가 발생하기 때문에, 컨트롤러가 호출되지도 않고, `400`
    예외가 발생하면서 오류 페이지를 띄워준다.
* 타입 오류가 발생해도 고객이 입력한 문자를 화면에 남겨야 함.
  * 만약 컨트롤러가 호출된다고 가정해도 `Item` 의 `price` 는 `Integer` 이므로 문자를 보관할 수가 없다.
    결국 문자는 바인딩이 불가능하므로 고객이 입력한 문자가 사라지게 되고, 고객은 본인이 어떤 내용을
    입력해서 오류가 발생했는지 이해하기 어렵다.

## 스프링이 제공하는 검증 오류 처리 방법

### BindingResult1

* `BindingResult`이 핵심이다.
* 반드시 `@ModelAttribute` 바로 뒤에 `BindingResult` 를 작성해야 한다.!!
* `BindingResult` 는 `Model` 에 자동으로 포함
```java
public String addItem(@ModelAttribute Item item, BindingResult bindingResult) {}
```

```java
public String addItemV1(@ModelAttribute Item item, BindingResult bindingResult, RedirectAttributes redirectAttributes, Model model) {

        //검증 오류 결과 보관
        Map<String, String> errors = new HashMap<>();

        //검증 로직
        if(!StringUtils.hasText(item.getItemName())) {
            bindingResult.addError(new FieldError("item", "itemName", "상품이름은 필수 입니다."));
        }
        if(item.getPrice() == null || item.getPrice() < 1000 || item.getPrice() > 1000000) {
            bindingResult.addError(new FieldError("item", "price", "가격은 1,000 ~ 1,000,000 까지 허용합니다."));
        }
        if(item.getQuantity() == null || item.getQuantity()>= 9999) {
            bindingResult.addError(new FieldError("item", "quantity", "상품 수량은 최대 9,999 까지 허용 합니다."));
        }

        //특정 필드가 아닌 복합 룰 검증
        if(item.getQuantity() != null && item.getPrice() != null) {
            int resultPrice = item.getPrice() * item.getQuantity();
            if (resultPrice < 10000) {
                bindingResult.addError(new ObjectError("item", "가격 * 수량의 합은  10,000원 이상을 허용합니다. 현재 값 = "+resultPrice));
            }
        }

        //검증에 실패하면 다시 입력 폼으로
        if(bindingResult.hasErrors()) {
          log.info("bindingResult = {}", bindingResult);
          return "/validation/v2/addForm";
        }

        //성공 로직
        Item savedItem = itemRepository.save(item);
        redirectAttributes.addAttribute("itemId", savedItem.getId());
        redirectAttributes.addAttribute("status", true);
        return "redirect:/validation/v2/items/{itemId}";
}
```

**필드오류 -FieldError**
```java
public FieldError(String objectName, String field, String defaultMessage) {}
    
    //...//
    
if (!StringUtils.hasText(item.getItemName())) {
    bindingResult.addError(new FieldError("item", "itemName", "상품 이름은 필수입니다."));
}
```
필드에 오류가 있으면 `FieldError` 객체를 생성해서 `bindingResult` 에 담아두면 된다.
* `objectName` : `@ModelAttribute` 이름
* `field` : 오류가 발생한 필드 이름
* `defaultMessage` : 오류 기본 메시지


**글로벌 오류 -ObjectError**
```java
public ObjectError(String objectName, String defaultMessage) {}

    //...//
        
bindingResult.addError(new ObjectError("item", "가격 * 수량의 합은 10,000원 이상이어야 합니다. 현재 값 = " + resultPrice));
```
특정 필드를 넘어서는 오류가 있으면 `ObjectError` 객체를 생성해서 `bindingResult` 에 담아두면 된다.
* `objectName` : `@ModelAttribute` 의 이름
* `defaultMessage` : 오류 기본 메시지

**글로벌 오류 처리**
```html
<div th:if="${#fields.hasGlobalErrors()}">
    <p class="field-error" th:each="err : ${#fields.globalErrors()}" th:text="${err}">전체 오류 메시지</p>
</div>
```

**필드 오류 처리**
```html
<input type="text" id="itemName" th:field="*{itemName}" th:errorclass="field-error" class="form-control" placeholder="이름을 입력하세요">
<div class="field-error" th:errors="*{itemName}">
 상품명 오류
</div>
```

### BindingResult2

* 스프링이 제공하는 검증 오류를 보관하는 객체, 검증 오류가 발생하면 여기에 보관
* `BindingResult` 가 있으면 `@ModelAttribute`에 데이터 바인딩 시 오류가 발생해도 컨트롤러가 호출된디ㅏ.!

**예) `@ModelAttribute` 에 바인딩 시 타입 오류가 발생하면?**
* `BindingResult` 가 없으면 ->  `400 오류` 발생하면서 컨트롤러가 호출되지 않고, 오류 페이지로 이동
* `BindingResult` 가 있으면 -> 오류 정보(`FieldError`)를 `BindingResult` 에 담아서 컨트롤러를 정상 호출

**BindingResult 에 오류를 적용하는 방법 3가지**
* `@ModelAttribute` 의 객체에 타입 오류 등으로 바인딩이 실패하는 경우 스프링이 `FieldError` 생성해서 `BindingResult` 에 넣어준다.
* 개발자가 직접 넣어준다...
* Validator 사용

현재 오류 발생시 사용자 입력한 내용 사라지고 있다. 이문제를 해결해보자!

### FieldError, ObjectError
* 사용자 입력 오류 메시지가 화면에 남도록 하자.!
* 이를 위해서 `FieldError`, `ObjectError` 에 대해서 자세히 알아보자!
```java
@PostMapping("/add")
public String addItemV2(@ModelAttribute Item item, BindingResult bindingResult, RedirectAttributes redirectAttributes, Model model) {

    //검증 로직
    if(!StringUtils.hasText(item.getItemName())) {
        bindingResult.addError(new FieldError("item", "itemName", item.getItemName(), false, null, null, "상품이름은 필수 입니다."));
    }
    if(item.getPrice() == null || item.getPrice() < 1000 || item.getPrice() > 1000000) {
        bindingResult.addError(new FieldError("item", "price", item.getPrice(), false, null, null, "가격은 1,000 ~ 1,000,000 까지 허용합니다."));
    }
    if(item.getQuantity() == null || item.getQuantity() >= 9999) {
        bindingResult.addError(new FieldError("item", "quantity", item.getQuantity(), false, null, null, "상품 수량은 최대 9,999 까지 허용 합니다."));
    }

    //특정 필드가 아닌 복합 룰 검증
    if(item.getQuantity() != null && item.getPrice() != null) {
        int resultPrice = item.getPrice() * item.getQuantity();
        if (resultPrice < 10000) {
            bindingResult.addError(new ObjectError("item", null, null, "가격 * 수량의 합은  10,000원 이상을 허용합니다. 현재 값 = "+resultPrice));
        }
    }

    //검증에 실패하면 다시 입력 폼으로
    if(bindingResult.hasErrors()) {
        log.info("bindingResult = {}", bindingResult);
        return "/validation/v2/addForm";
    }

    //성공 로직
    Item savedItem = itemRepository.save(item);
    redirectAttributes.addAttribute("itemId", savedItem.getId());
    redirectAttributes.addAttribute("status", true);
    return "redirect:/validation/v2/items/{itemId}";
}
```

**FieldError 생성자**

```java
public FieldError(String objectName, String field, String defaultMessage) {};

public FieldError(String objectName, 
                  String field, 
                  @Nullable Object rejectedValue, 
                  boolean bindingFailure, 
                  @Nullable String[] codes, 
                  @Nullable Object[] arguments, 
                  @Nullable String defaultMessage) {};
```

**ObjectError 생성자**
```java
public ObjectError(String objectName, String defaultMessage) {};
    
public ObjectError(String objectName, 
                  @Nullable String[] codes, 
                  @Nullable Object[] arguments, 
                  @Nullable String defaultMessage) {};
```

**파라미터 목록**
* `objectName` : 오류가 발생한 객체 이름
* `field` : 오류 필드
* `rejectedValue` : 사용자가 입력한 값(거절된 값)
* `bindingFailure` : 타입 오류 같은 바인딩 실패인지, 검증 실패인지 구분 값
* `codes` : 메시지 코드
* `arguments` : 메시지에서 사용하는 인자
* `defaultMessage` : 기본 오류 메시지

**오류 발생시 사용자 입력 값 유지**
```java
new FieldError("item", "price", item.getPrice(), false, null, null, "가격은 1,000 ~ 1,000,000 까지 허용합니다.");
```
* 사용자가의 입력 데이터가 컨트롤러의 `@ModelAttribute` 에 바인딩되는 시점에 오류가 발생하면 모델 객체에 사용자 입력값을 유지하기 어려움.
  * e.g) 
  * 가격에 숫자가 아닌 문자가 입력되면 가격은 `Integer` 타입이므로 문자를 보관할 수 잇는 방법이 없다.
  * 그래서 오류가 발생한 경우 사용자 입력 값을 보관하는 별도의 방법이 필요하다.
  * 이렇게 보관한 사용자 입력 값을 검증 오류 발생시 화면에 다시 출력.
* `FieldError` 는 오류 발생시 사용자 입력 값을 저장하는 기능을 제공
* 여기서 `rejectValue` 가 바로 오류 발생시 사용자 입력 값을 저장하는 필드
* `bindingFailure` 는 타입 오류 같은 바인딩이 실패했는지 여부를 적어주면 된다. 여기서는 바인딩이 실패한 것이 아니기 때문에 `false` 를 사용

**타임리프의 사용자 입력 값 유지**

`th:field="*{price}"`

타임리프의 `th:field` 는 매우 똑똑하게 동작하는데, 
**정상 상황에서는 모델 객체의 값을 사용**하지만, 
**오류가 발생하면 `FieldError` 에서 보관한 값을 사용**해서 값을 출력한다.

**스프링의 바인딩 오류 처리**

타입 오류로 바인딩에 실패하면 스프링은 `FieldError` 를 생성하면서 사용자가 입력한 값을 넣어둔다.
그리고 해당 오류를 `BindingResult` 에 담아서 컨트롤러를 호출한다. 따라서 타입 오류 같은 바인딩
실패시에도 사용자의 오류 메시지를 정상 출력할 수 있다.

### 오류 코드와 메시지 처리1
**FieldError 생성자**

```java
public FieldError(String objectName, String field, String defaultMessage) {};

public FieldError(String objectName, 
                  String field, 
                  @Nullable Object rejectedValue, 
                  boolean bindingFailure, 
                  @Nullable String[] codes, 
                  @Nullable Object[] arguments, 
                  @Nullable String defaultMessage) {};
```
**파라미터 목록**
* `objectName` : 오류가 발생한 객체 이름
* `field` : 오류 필드
* `rejectedValue` : 사용자가 입력한 값(거절된 값)
* `bindingFailure` : 타입 오류 같은 바인딩 실패인지, 검증 실패인지 구분 값
* `codes` : 메시지 코드
* `arguments` : 메시지에서 사용하는 인자
* `defaultMessage` : 기본 오류 메시지

`FieldError`, `ObjectError` 의 생성자는 `codes`, `arguments` 를 제공한다. 
이것은 **오류 발생시 오류 코드로 메시지를 찾기 위해 사용**

**errors 메시지 파일 생성**

messages.properties 를 사용해도 되지만, 오류 메시지를 구분하기 쉽게 errors.properties 라는 별도의 파일로 관리해보자.

**스프링 부트 메시지 설정 추가**

생략하면 messages.properties 를 기본으로 인식한다.

`application.properties`
```properties
spring.messages.basename=messages,errors
```

`src/main/resources/errors.properties`
```properties
required.item.itemName=상품 이름은 필수입니다.
range.item.price=가격은 {0} ~ {1} 까지 허용합니다.
max.item.quantity=수량은 최대 {0} 까지 허용합니다.
totalPriceMin=가격 * 수량의 합은 {0}원 이상이어야 합니다. 현재 값 = {1}
```

errors 에 등록한 메시지를 사용하도록 코드를 변경해보자..!!😄
```java
@PostMapping("/add")
public String addItemV3(@ModelAttribute Item item, BindingResult bindingResult, RedirectAttributes redirectAttributes, Model model) {

    //검증 로직
    if(!StringUtils.hasText(item.getItemName())) {
        bindingResult.addError(new FieldError("item", "itemName", item.getItemName(), false, new String[]{"required.item.itemName"}, null, null));
    }
    if(item.getPrice() == null || item.getPrice() < 1000 || item.getPrice() > 1000000) {
        bindingResult.addError(new FieldError("item", "price", item.getPrice(), false, new String[]{"range.item.price"}, new Object[]{1000, 1000000}, null));
    }
    if(item.getQuantity() == null || item.getQuantity() >= 9999) {
        bindingResult.addError(new FieldError("item", "quantity", item.getQuantity(), false, new String[]{"max.item.quantity"}, new Object[]{9999}, null));
    }

    //특정 필드가 아닌 복합 룰 검증
    if(item.getQuantity() != null && item.getPrice() != null) {
        int resultPrice = item.getPrice() * item.getQuantity();
        if (resultPrice < 10000) {
            bindingResult.addError(new ObjectError("item", new String[]{"totalPriceMin"}, new Object[]{10000, resultPrice}, null));
        }
    }

    //검증에 실패하면 다시 입력 폼으로
    if(bindingResult.hasErrors()) {
        log.info("bindingResult = {}", bindingResult);
        return "/validation/v2/addForm";
    }

    //성공 로직
    Item savedItem = itemRepository.save(item);
    redirectAttributes.addAttribute("itemId", savedItem.getId());
    redirectAttributes.addAttribute("status", true);
    return "redirect:/validation/v2/items/{itemId}";
}
```

**코드 변경**
```java
//range.item.price=가격은 {0} ~ {1} 까지 허용합니다.
new FieldError("item", "price", item.getPrice(), false, new String[] {"range.item.price"}, new Object[]{1000, 1000000}
```
* `codes` : `required.item.itemName` 를 사용해서 메시지 코드를 지정한다. 메시지 코드는 하나가 아니라
배열로 여러 값을 전달할 수 있는데, **순서대로 매칭해서 처음 매칭되는 메시지가 사용**된다.
* `arguments` : `Object[]{1000, 1000000}` 를 사용해서 코드의 `{0}` , `{1}` 로 치환할 값을 전달한다.


### 오류 코드와 메시지 처리2
* 인간의 욕심은 끝이 없고.. ㅋㅋ
* FieldError, ObjectError는 다루기 너무 번거롭다..
* 오류 코드를 좀더 자동화 할수 없을까? 🤔

컨트롤러에서 `BindingResult` 는 검증해야 할 객체인 `target` 바로 다음에 온다.
따라서 `BindingResult` 는 이미 본인이 검증해야 할 객체인 `target` 을 알고 있다.

다음을 컨트롤러에서 실행해보면 알 수 있다.
```java
log.info("objectName={}", bindingResult.getObjectName());
log.info("target={}", bindingResult.getTarget());
//...//
//검증 로직
//특정 필드가 아닌 복합 룰 검증
```

출력 결과
```text
objectName=item //@ModelAttribute name
target=Item(id=null, itemName=상품, price=100, quantity=1234)
```

### `rejectValue()`, `reject`
`BindingResult` 가 제공하는 `rejectValue()`, `reject()` 를 사용하면 
`FieldError`, `ObjectError` 를 직접 생성하지 않고, 깔끔하게 검증 오류를 다룰 수 있다.

`rejectValue()`, `reject` 를 사용해서 기존 코드를 단순화 해보자!

```java
//@PostMapping("/add")
public String addItemV4(@ModelAttribute Item item, BindingResult bindingResult, RedirectAttributes redirectAttributes, Model model) {

    log.info("objectName={}", bindingResult.getObjectName());
    log.info("target={}", bindingResult.getTarget());

    // 검증 로직
    // ValidationUtils.rejectIfEmptyOrWhitespace(bindingResult, "itemName", "required");
    if(!StringUtils.hasText(item.getItemName())) {
        //bindingResult.addError(new FieldError("item", "itemName", item.getItemName(), false, new String[]{"required.item.itemName"}, null, null));
        bindingResult.rejectValue("itemName", "required");
    }
    if(item.getPrice() == null || item.getPrice() < 1000 || item.getPrice() > 1000000) {
        bindingResult.rejectValue("price", "range", new Object[]{1000, 1000000}, null);
    }
    if(item.getQuantity() == null || item.getQuantity() >= 9999) {
        bindingResult.rejectValue("quantity", "max", new Object[]{9999}, null);
    }

    // 특정 필드가 아닌 복합 룰 검증
    if(item.getQuantity() != null && item.getPrice() != null) {
        int resultPrice = item.getPrice() * item.getQuantity();
        if (resultPrice < 10000) {
            bindingResult.reject("totalPriceMin", new Object[]{10000, resultPrice}, null);
        }
    }

    //검증에 실패하면 다시 입력 폼으로
    if(bindingResult.hasErrors()) {
        log.info("bindingResult = {}", bindingResult);
        return "/validation/v2/addForm";
    }

    //성공 로직
    Item savedItem = itemRepository.save(item);
    redirectAttributes.addAttribute("itemId", savedItem.getId());
    redirectAttributes.addAttribute("status", true);
    return "redirect:/validation/v2/items/{itemId}";
}
```

**오류 메시지가 정상 출력 된다..?!**

**`errors.properties` 에 있는 코드를 직접 입력하지 않았는데 어떻게 된 것일까?**

**rejectValue()**
```java
void rejectValue(
        @Nullable String field, String errorCode,
        @Nullable Object[] errorArgs, 
        @Nullable String defaultMessage
        );
```

`field` : 오류 필드명
`errorCode` : 오류 코드(이 오류 코드는 메시지에 등록된 코드가 아니다. 뒤에서 설명할 `messageResolver` 를 위한 오류 코드이다.)
`errorArgs` : 오류 메시지에서 `{0}` 을 치환하기 위한 값
`defaultMessage` : 오류 메시지를 찾을 수 없을 때 사용하는 기본 메시지

```java
bindingResult.rejectValue("price", "range", new Object[]{1000, 1000000}, null);
```
앞에서 `BindingResult` 는 **어떤 객체를 대상으로 검증하는지 `target`을 이미 알고 있다**고 했다. 
따라서 `target(item)` 에 대한 정보는 없어도 된다. **오류 필드명은 동일하게 `price` 를 사용**했다.

**축약된 오류 코드**
`FieldError()` 를 직접 다룰 때는 오류 코드를 `range.item.price` 와 같이 모두 입력했다. 

그런데 `rejectValue()` 를 사용하고 부터는 오류 코드를 `range` 로 간단하게 입력했다. 
그래도 오류 메시지를 잘 찾아서 출력한다. 무언가 규칙이 있는 것 처럼 보인다. 

이 부분을 이해하려면 `MessageCodesResolver` 를 이해해야 한다. 
왜 이런식으로 `오류 코드` 를 구성하는지 바로 다음에 자세히 알아보자.

### 오류 코드와 메시지 처리3
오류 코드를 만들때 자세히 만들 수 도 있고,

단순하게 만들 수도 있다.

```properties
# 자세히 만들기
required.item.itemName=상품 이름은 필수 입니다.
range.item.price=상품의 가격 범위 오류 입니다.

# 단순히 만들기
required=필수 값 입니다.
range=범위 오류 입니다.
```

**에러코드**를 만드는 가장 좋은 방법은 `범용성으로 사용하다가`,
세밀하게 작성해야 하는 경우는 세밀한 내용이 적용되도록 `메시지에 단계를 두는 방법`이다.

```properties
#Level1
required.item.itemName=상품 이름은 필수 입니다.

#Level2
required=필수 값 입니다.
```

스프링은 `MessageCodeResolver` 라는 것으로 이러한 기능을 지원!!

`MessageCodesResolver` 는 인터페이스이고 `DefaultMessageCodesResolver` 는 기본 구현체이다.

**DefaultMessageCodesResolver의 기본 메시지 생성 규칙**

**객체 오류**
```text
객체 오류의 경우 다음 순서로 2가지 생성
1.: code + "." + object name
2.: code

예) 오류 코드: required, object name: item
1.: required.item
2.: required
```

**필드 오류**
```text
필드 오류의 경우 다음 순서로 4가지 메시지 코드 생성
1.: code + "." + object name + "." + field
2.: code + "." + field
3.: code + "." + field type
4.: code

예) 오류 코드: typeMismatch, object name "user", field "age", field type: int
1. "typeMismatch.user.age"
2. "typeMismatch.age"
3. "typeMismatch.int"
4. "typeMismatch"
```

동작 방식
* `rejectValue()` , `reject()` 는 내부에서 MessageCodesResolver 를 사용한다. 여기에서 메시지
코드들을 생성한다.
* `FieldError` , `ObjectError` 의 생성자를 보면, 오류 코드를 하나가 아니라 여러 오류 코드를 가질 수 있다.
`MessageCodesResolver` 를 통해서 생성된 순서대로 오류 코드를 보관한다.
* 이 부분을 `BindingResult` 의 로그를 통해서 확인해보자. 
  * `codes [range.item.price, range.price, range.java.lang.Integer, range]`

## 검증2 -Validation
