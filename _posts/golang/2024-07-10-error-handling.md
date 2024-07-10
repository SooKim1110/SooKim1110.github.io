---
title: 부딪히며 배우는 Golang 백엔드 개발 - ① 에러 처리 및 응답 작성
date: 2024-07-10 22:30:00 +0900
categories: [Go]
tags: [Go, Golang, 에러처리, 백엔드]
---

회사에서 팀을 옮기면서 자바에서 Go로 백엔드를 구축하는 환경으로 바뀌었다.
자바와는 다른 점이 많다고 느끼면서도 결국 백엔드 개발의 본질은 비슷하다는 생각도 든다.

어쨌든 요즘 공부와 개발을 같이하고 있어서 ~~(원래 개발은 실전이지!)~~ 앞으로 직접 부딪혀가면서 겪었던 문제들과 해결 방법을 정리해두려고 한다.

첫 주제는 개발할 때 항상 고민하게 되는 **에러 처리 방법** 그리고 에러 발생시에 우아하게 **응답을 작성하는 방법**이다!

## 문제 상황

> Go의 Gin 프레임워크를 이용해 API 서버를 구현하고 있다. 
> 사용자 요청 처리 중 에러가 발생하는 경우에 사용자에게 적절한 에러 응답을 보내고 싶다.

> 근데 Go는 살펴보니 try catch 도 없고 `error`에 추가 정보나 스택 정보를 남기는 것도 쉽지 않다!
> 어떻게하면 각 계층에서 비즈니스 에러를 적절하게 관리하고 사용자에게 에러 응답을 잘 작성해줄 수 있을까?

## Go에서의 에러 처리

Go를 공부하는데 공식 Go Blog 만한게 없다. [Error handling and Go](https://go.dev/blog/error-handling-and-go)를 읽으면 Go에서는 에러 핸들링을 어떻게 하는지 감을 잡을 수 있다.

자바에는 `Exception`이 존재한다면 Go에는 에러 처리를 위해 `error` 라는 타입이 존재하는데, 인터페이스 타입으로 `Error()` 함수가 구현되어 있는 모든 struct를 error 타입이라고 볼 수 있다.
```go
type error interface {
    Error() string
}
```

새로운 에러를 만들기 위해 `errors.New()`를 사용해줄 수 있는데, 정말 단순한 코드이다. `fmt.Println()`으로 에러를 출력해주면 에러의 `Error()`를 호출하여 리턴한다.
```go
// errors.New()
func New(text string) error {
    return &errorString{text}
}

type errorString struct {
    s string
}

func (e *errorString) Error() string {
    return e.s
}

err := errors.New("invalid argument")
fmt.Println(err) // output: invalid argument
```

애플리케이션에서 발생할 수 있는 기본적인 에러들은 이렇게 `errors.New()` 로 선언해두고 관리 할 수 있다.
```go
var (
    ErrRequestFail    = errors.New("request failed")
    ErrAuthDeclined   = errors.New("authorization declined")
    ErrTimeout        = errors.New("request timed-out")
)
```

Go에서 발생한 에러에 추가적인 정보를 넣기 위한 기본적인 방법은 래핑이다. `fmt.Errorf()` 등을 사용하면 위의 에러들을 래핑해가면서 추가적인 정보들을 포맷팅해서 넣어줄 수 있다.
그치만, 기본적으로 에러는 스트링일 뿐이다.

이런 에러가 발생했을 때 service 단에서는 어떻게 처리해줄 수 있을까?
자바의 `Exception`은 throw하고 try-catch 블록을 사용해주는 방식인 반면, Go에서는 단순히 함수의 반환값으로 `error`를 반환해주고 호출처에서 이 값을 받아 처리해줘야한다.

예를 들어서 service에서 사용하는 ApiClient에서 `ErrTimeout` 에러가 발생하면 함수의 반환값으로 에러를 전달해준다. 그럼 전달 받은 service 에서 이를 적절하게 처리해주면 된다.
```go
// service
signupDate, err := apiClient.GetSignupDate(userId)
if err != nil {
	// 에러 처리
}
```

## 커스텀 에러 만들기
내부적으로 `ErrTimeout` 에러가 발생할 경우에 사용자에게는 이를 실제 내부적인 에러를 노출하기 보다는 500 응답을 주고 싶다고 생각해보자.
사실 API 서버에서는 이런 경우처럼 특정 에러가 발생한 경우에 사용자에게 특정 http status, message 등으로 응답하는 경우가 많다.
그래서 밑단의 에러를 적절한 비즈니스 에러로 변환해줄 필요성이 있다.

어떻게 기본적인 에러를 비즈니스 에러로 변환해줄 수 있을까?

### 1) 커스텀 에러 struct 추가 
비즈니스 에러를 다루기 위해 새로운 Error 구현체를 하나 만들어서 API 서버에서 사용할 수 있다.
해당 struct에는 에러에 추가적인 정보를 넣을 수 있도록 필요한 필드들을 넣을 수 있다.

```go
type AppError struct {
    code ErrorCode  // 별도로 정의한 애플리케이션에서 발생할 수 있는 비즈니스 에러 종류들
    Err   error
    Message string
}

func (e *AppError) Error() string {
    return fmt.Sprintf("Code: %d, Message: %s, InternalError: %v", e.Code, e.Message, e.Error)
}

type ErrorCode int

const (
    InvalidInput ErrorCode = iota + 1
    Unauthorized
    NotFound
    InternalError
)
```

커스텀 에러인 `AppError`는 어떤 비즈니스 에러인지 나타내기 위해 ErrorCode 필드를 포함하고 있고, 기존에 발생한 에러에 대한 정보를 담기 위한 Err 필드를 가지고 있다.
그 외에는 각 애플리케이션에서 에러 처리를 위해 필요한 정보들을 더 추가해주면 된다.

여기서 `Error()` 함수를 구현해주었기 때문에 `AppError`는 error 타입으로 사용할 수 있다.
그러면 이제 service에서 적절하게 원하는 `AppError`로 기본 에러를 변환해줄 수 있다. 해당 에러를 controller에서 처리할 수 있도록 리턴해준다.
```go
// service
func getUserInfo(userId int) (User, error) {
    signupDate, err := apiClient.GetSignupDate(userId)
    if errors.Is(err, ErrTimeout) {
        return nil, &AppError{
            Code:    InternalError,
            Error:   err
        }
    }	
    // ...
}

```

### 2) 커스텀 에러에 trace 정보 넣어주기
커스텀 에러를 만들어서 추가적인 정보를 넣어주긴 했는데 한 가지 부족한 정보가 있다. 어디서 해당 에러가 발생한건지에 대한 정보가 없어서 디버깅하기 어렵다는 것이다.
자바처럼 호출 스택을 바로 알 수 있으면 좋을텐데 Go에서는 직접 정보를 추가해줘야한다.
방법을 찾다가 좋은 d2 글을 발견해서 참고해서 구현했다. ([Golang, 그대들은 어떻게 할 것인가 - 3. error 래핑](https://d2.naver.com/helloworld/2690202))

정보를 넣어줄 수 있는 첫번째 방법은 github.com/pkg/errors 패키지의 `errors.Wrap()` 을 활용하는 것이다.
해당 함수로 에러를 래핑해주면 에러에 호출 스택 정보를 포함해줄 수 있다.
```go
func GetUser() error {
    return errors.Wrap(ErrRequestFail, "get user info failed")	// 호출 스택 정보가 포함된 에러가 생성된다.
}

err := GetUser()
fmt.Printf("%+v", err) // 전체 호출 스택이 함께 출력된다.
```

포맷팅 지시자 `%+v`를 사용해서 출력하면 에러 메세지와 함께 전체 호출 스택을 출력하기 때문에 에러 발생 위치를 파악하기 쉽다.
다만, `errors.Wrap()`을 반복적으로 사용하게 될 경우 호출 스택 정보가 중첩되기 때문에 커스텀 에러로 만들어줄 때 한번만 `errors.Wrap()`을 사용해주는 것이 좋다.
또 한가지 단점은 `Wrap()` 자체가 파라미터로 메세지를 받기 때문에 추가 에러 메세지를 넣고 싶지 않아도 빈 스트링이라도 넣어줘야한다는 것이다.

위 방법을 사용해도 좋지만, 잘못하여 `errors.Wrap()`을 여러번 사용하게 되면 호출 스택이 불필요하게 길어질 수 있다.
그래서, 두번째 방법으로 직접 에러 발생 위치에 대한 정보 구해서 에러를 래핑해줄 수 있다.
사실 에러가 발생한 파일의 위치와 함수 이름의 정보만 있으면 디버깅할 때 충분히 유용하게 사용할 수 있어서, 이 정보만 구해서 넣어주면 된다.

Go의 `runtime.Caller(skip int) ` 함수를 사용하면 호출처에서 skip만큼의 상위 호출 스택의 정보를 가져올 수 있다.
이 함수를 사용해주면 커스텀하게 pc, 파일 이름, 함수 위치 등의 정보를 얻을 수 있다.
또한 `runtime.FuncForPC(pc int)` 함수를 사용하면 함수 이름의 정보도 얻을 수 있다.
이런 정보를 조합해서 에러를 한 번 더 래핑해주면 커스텀한 trace 정보가 포함된 에러를 만들어줄 수 있다.

## Gin에 에러 핸들러 등록하기

여기까지 왔다면 이제 controller 에서 에러를 받아서 적절한 에러 응답을 작성하기만 해주면 된다.
각 controller 마다 직접 response를 작성해주는 코드를 넣어도 되겠지만 에러에 대한 처리 방법은 유사한 경우가 많으므로 공통적으로 처리해주면 좋을 것 같다.

이를 위해 Gin에서 에러 핸들러를 미들웨어로 등록해서 에러들을 공통적으로 처리하는 구조를 사용할 수 있다.
자바의 Global Error Handler 와 유사하다고 볼 수 있다.

service 단에서 에러가 발생했을 때 `AppError`를 생성해준 후 controller 단으로 바로 리턴해준다. 
그러면 controller 에서는 받은 에러를 바로 Gin Context에 넣어주기만 하고 리턴한다.
그러면 Gin 미들웨어로 에러 핸들러를 등록해두어서 여기에서 공통적으로 에러를 응답에 작성해주는 처리를 해줄 수 있다.

```go
// controller
user, err := ctrl.userService.GetUser(userId)
if err != nil {
    c.Error(err)
    return
}
```

아래와 같이 미들웨어를 작성하고 등록해줄 수 있다.
```go
r := Gin.Default()
r.Use(ErrorHandler()) // 미들웨어 등록

// Error Handler
func ErrorHandler() Gin.HandlerFunc {
    return func(c *Gin.Context) {
        c.Next()

    // 에러 가져오기
    err := c.Errors.Last()
    if err == nil {
        return
    }
    
    // AppError 타입인지 확인하여 처리
    var appError *AppError
    if errors.As(lastError.Err, &appError) {
    ok := errors.As(err, &appError)
    if !ok {
        logger.Errorf("Failed to handle error. Error not AppError. error: %s", err.Error())
        // 적절한 응답 작성
    }
	
	// AppError -> 적절한 응답 작성
}
```

여기서 `c.Next()`는 Gin 프레임워크에서 현재 미들웨어에서 다음 등록된 미들웨어나 핸들러 함수로 제어를 넘기는 역할을 한다.

이 때, 미들웨어가 등록된 순서도 중요하다.
처음에는 이해가 잘 안갔는데 미들웨어가 여러개 등록되어 있고 `c.Next()` 위치의 전과 후에 코드가 작성되어 있다고 할 때, 미들웨어가 등록된 순서로 전 코드는 순서대로 실행되고, 후 코드는 등록된 순서의 역순으로 실행된다고 이해하면 쉽다.

여기서 에러 핸들링은 실제 controller의 처리가 끝난 후에 실행이 되어야하므로 `c.Next()` 후에 작성되어야한다.

## 참고자료
[The Go Blog - Error handling and Go](https://go.dev/blog/error-handling-and-go)  
[Golang, 그대들은 어떻게 할 것인가 - 3. error 래핑](https://d2.naver.com/helloworld/2690202)  
[Golang, 그대들은 어떻게 할 것인가 - 4. error 핸들링](https://d2.naver.com/helloworld/6507662)  
[Don’t just check errors, handle them gracefully](https://dave.cheney.net/2016/04/27/dont-just-check-errors-handle-them-gracefully)