---
title: 부딪히며 배우는 Golang 백엔드 개발 - ② Gin에서 미들웨어 작성하기
date: 2024-07-17 22:40:00 +0900
categories: [Programming Languages, Go]
tags: [Go, Golang, 미들웨어, 백엔드, Gin]
---

두번째 주제는 **Gin 프레임워크에서 미들웨어를 작성하는 방법**이다.
첫번째 글에서 미들웨어로 공통 에러 핸들러를 만들어주었는데, 미들웨어가 무엇이고 어떻게 사용하면 좋은지 좀 더 자세히 알아보자.

생각보다 Gin의 미들웨어 흐름 제어에 관해 자세하게 적은 글이 없어서 `c.Next()`, `c.Abort()` 등을 활용했을 때의 동작 방식을 자세히 적어보려고 한다.

## 문제 상황

> Go의 Gin 프레임워크를 이용해 API 서버를 구현하고 있다. 
> 사용자 인증, 로깅, 에러 처리와 같이 모든 서비스에 공통적으로 적용되어야하는 작업들이 각 컨트롤러 코드에 중복해서 존재한다.

> 이런 중복 코드는 어떻게 제거할 수 있을까? 이럴 때 Gin에서 미들웨어를 사용할 수 있다고 하는데 미들웨어는 무엇이고 어떻게 동작하는 것일까?

## 미들웨어란?
Go 웹 프레임워크에서 말하는 미들웨어는 말 그대로 중간에서 모든 요청에 공통적으로 적용되어야하는 작업들을 담당해주는 함수 체인이다.
매 API 요청마다 수행되어야하는 사용자 인증이라던가 로깅과 같은 처리, 지난 글에서 본 공통 에러처리 등을 매번 컨트롤러에 중복해서 넣어주면 관리하기 쉽지 않을 것이다.

이럴 때 미들웨어로 특정 관심사를 분리하게 되면 유지보수가 쉬워지고 의존성도 분리되므로 테스트하기도 편해질 것이다.

Gin의 미들웨어는 어떻게 보면 Spring AOP랑 조금 유사하다고 볼 수도 있다. AOP를 활용하면 비즈니스 로직에서 공통 기능을 분리시켜 메소드 호출 전후에 적용할 수 있다.
예를 들어 요청에 대한 전후처리를 `HandlerInterceptor`를 구현하고 등록해서 사용할 수도 있고 `@ExceptionHandler` annotation이 존재해서 글로벌 에러 핸들러를 만들어줄 수도 있다.
Spring security에서는 다양한 `Filter`를 사용하여 인증 및 권한 부여를 담당해줄 수 있다.

이렇게 각 언어, 프레임워크마다 방식은 조금 다르지만 미들웨어를 통한 공통 관심사 분리는 개발에 필수적인 접근 방식이다.

## Gin의 미들웨어

그러면 Gin에서 실제로 어떻게 미들웨어를 사용할 수 있는지 살펴보자.

![img1](/assets/images/programminglanguage/golang/gin-middleware/img1.png){: w="600" }

위에서 미들웨어를 함수 체인이라고 설명했는데, 그림에서 볼 수 있듯이 요청이 들어오면, 이를 실제로 처리하는 핸들러 로직이 수행되기 전후에 등록된 미들웨어들의 체인을 모두 거쳐가게 된다.


### 기본 미들웨어 작성법

Gin의 미들웨어 사용 예시를 공식 문서에서 찾아볼 수 있다.
[미들웨어를 사용하기](https://gin-gonic.com/ko-kr/docs/examples/using-middleware/)

```go
func main() {
	// 기본 미들웨어를 포함하지 않는 라우터를 작성합니다.
	r := gin.New()

	// Global middleware
	// GIN_MODE=release로 하더라도 Logger 미들웨어는 gin.DefaultWriter에 로그를 기록합니다.
	// 기본값 gin.DefaultWriter = os.Stdout
	r.Use(gin.Logger())

	// Recovery 미들웨어는 panic이 발생하면 500 에러를 씁니다.
	r.Use(gin.Recovery())

	// 각 라우트 당 원하는만큼 미들웨어를 추가 할 수 있습니다.
	r.GET("/benchmark", MyBenchLogger(), benchEndpoint)

	// 권한 그룹
	// authorized := r.Group("/", AuthRequired())
	// 다음과 동일합니다:
	authorized := r.Group("/")
	// 그룹별로 미들웨어를 사용할 수 있습니다!
	// 이 경우 "authorized"그룹에서만 사용자 정의 생성된 AuthRequired() 미들웨어를 사용합니다.
	authorized.Use(AuthRequired())
	{
		authorized.POST("/login", loginEndpoint)
		authorized.POST("/submit", submitEndpoint)
		authorized.POST("/read", readEndpoint)

		// 중첩 그룹
		testing := authorized.Group("testing")
		testing.GET("/analytics", analyticsEndpoint)
	}

	// 서버가 실행 되고 0.0.0.0:8080 에서 요청을 기다립니다.
	r.Run(":8080")
}
```

가장 기본적인 문법은 `r.Use(gin.Logger())` 처럼 `Use()`로 미들웨어를 등록해주는 것이다.

실제 미들웨어는 다음과 같은 두 가지 방법으로 작성해줄 수 있다.

```go
// 1번 방식
func CustomMiddleware(c *gin.Context) {
  fmt.Println("커스텀 미들웨어 실행")
}

r.Use(CustomMiddleware)

// 2번 방식
func CustomMiddleware() gin.HandlerFunc {
// 기타 초기화 로직들
  return func(c *gin.Context) {
    fmt.Println("커스텀 미들웨어 실행")
  }
}

r.Use(CustomMiddleware())
```

첫번째 방식은 간단하게 함수에 미들웨어의 로직을 바로 적어주는 방식이다.

두번째 방식은 Gin의 `HandlerFunc`를 리턴하는 함수를 작성하는 방식으로, 첫번째 방식과 다르게 함수를 리턴하기 전에 초기화 로직들을 적어줄 수 있다.
특정 조건에 따라 다른 `HandlerFunc`를 사용해야하는 경우나 특정 로직을 실제 미들웨어 로직을 수행하기 전에 한번만 수행해야하는 경우에 활용하기 좋다.

### 흐름 제어: 이론
앞서 말했듯이 미들웨어는 보통 여러 개를 등록해두고 쓰게 된다.
그래서 체인의 제어가 필요한 경우, 예를 들어 미들웨어에서 다음 미들웨어로 제어권을 넘겨주고 싶은 경우나, 다음 미들웨어로 넘어가지 않고 동작을 멈추고 싶은 경우 등이 생길 수 있다.

이런 미들웨어 체인 흐름 제어를 할 수 있도록 Gin Context에서 두 가지 함수 `c.Next()`와 `c.Abort()`를 제공한다.

#### c.Next()
`c.Next()`는 미들웨어 내에서만 써야하는 함수로, 미들웨어 체인에 있는 다음 핸들러로 제어권을 넘긴다.
```go
// Next should be used only inside middleware.
// It executes the pending handlers in the chain inside the calling handler.
// See example in GitHub.
func (c *Context) Next() {
	c.index++
	for c.index < int8(len(c.handlers)) {
		c.handlers[c.index](c)
		c.index++
	}
}
```

`c.Next()`를 미들웨어의 맨 마지막에 작성해주고 바로 리턴해주는 예시 코드가 많은데, 사실 `c.Next()` 이후에 별도 로직이 없다면 `c.Next()`를 생략해도 자동으로 다음 미들웨어로 제어권이 넘어간다.
([관련된 이슈](https://github.com/gin-gonic/gin/issues/287)를 참고하면 좋다.)

그럼 언제 `c.Next()`를 사용해야할까? 
모든 체인이 수행되고 다시 현재 미들웨어로 제어권이 돌아왔을 때 수행되어야하는 추가 로직이 있어야하는 경우에 사용해줄 수 있다.

예를 들어 아래 코드처럼 수행 시간을 측정해야하는 경우 `c.Next()`를 작성하면 제어권이 넘어가서 나머지 모든 핸들러들이 수행되고, 다시 현재 미들웨어로 제어권이 넘어왔을 때 그 이후에 작성된 로직들이 수행되어 전후 시간을 측정할 수 있다.

```go
func latency(c *gin.Context) {
    time := time.Now()
   
    c.Next() // 체인의 나머지 핸들러들이 모두 수행된다.

    now := time.Now()
    diff = now.Sub(time)
    fmt.Println(diff)
}
```

#### c.Abort()
`c.Abort()`는 미들웨어 체인을 멈추고 싶은 경우에 사용하는 함수이다.

```go
// Abort prevents pending handlers from being called. Note that this will not stop the current handler.
// Let's say you have an authorization middleware that validates that the current request is authorized.
// If the authorization fails (ex: the password does not match), call Abort to ensure the remaining handlers
// for this request are not called.
func (c *Context) Abort() {
  c.index = abortIndex
}
```

주의해야하는 것이 `c.Abort()`가 불리자마자 미들웨어가 해당 위치에서 멈추는 것은 아니고, 미들웨어에 작성된 나머지 로직들은 모두 그대로 수행이 된다.
만약 해당 위치에서 바로 멈추길 원한다면 return을 해주어야한다.

그리고 `c.Abort()`를 하면 현재 미들웨어 이후에 등록된 미들웨어들이 불리는 것을 막는거라, 현재 미들웨어 이전에 등록된 미들웨어들을 그대로 수행이 된다.

유사한 함수들로 Abort하면서 특정 응답도 작성해주고 싶은 경우에 사용할 수 있는  `c.AbortWithStatus()`, `c.AbortWithStatusJSON()`, `c.AbortWithError()`도 제공된다.


### 흐름 제어: 실전
이제 흐름 제어 방법을 알았으니 실제 코드를 보며 동작 순서를 예측해보자.

먼저, 미들웨어들이 여러 개 등록되었을 때 어떤 순서로 동작이 될까?

```go
func main() {
    r := gin.Default()

    // 미들웨어 체인 설정
    r.Use(loggingMiddleware())
    r.Use(errorHandlerMiddleware())
    r.Use(authMiddleware())
}
```

기본적으로 동작 순서는 등록된 순서대로 동작한다. 여기서는 로깅, 에러 핸들러, 인증 미들웨어가 순서대로 동작한다.

#### Case 1
그러면 아래처럼 각 미들웨어가 `c.Next()` 전후에 로그를 남길 때 출력은 어떻게 남을지 예측해보자.

```go
func loggingMiddleware() gin.HandlerFunc {
    return func(c *gin.Context) {
        fmt.Println("Logging middleware before")
        c.Next()
        fmt.Println("Logging middleware after")
    }
}

func errorHandlerMiddleware() gin.HandlerFunc {
    return func(c *gin.Context) {
        fmt.Println("Error handler middleware before")
        c.Next()
        fmt.Println("Error handler middleware after")
    }
}

func authMiddleware() gin.HandlerFunc {
    return func(c *gin.Context) {
        fmt.Println("Auth middleware before")
        c.Next()
        fmt.Println("Auth middleware after")
    }
}

func main() {
    r := gin.Default()
    
    r.Use(loggingMiddleware())
    r.Use(errorHandlerMiddleware())
    r.Use(authMiddleware())
    
    r.GET("/ping", func(c *gin.Context) {
        fmt.Println("Handler function")
        c.JSON(http.StatusOK, gin.H{"message": "pong"})
    })
    
    r.Run()
}
```

앞서 공부한대로 `c.Next()`를 사용하면 다음 미들웨어로 제어권이 넘어가므로 출력은 이렇게 남을 것이다.
```
Logging middleware before
Error handler middleware before
Auth middleware before
Handler function
Auth middleware after
Error handler middleware after
Logging middleware after
```

그림으로 살펴보면 다음과 같이 동작한다고 이해할 수 있다.
![img2](/assets/images/programminglanguage/golang/gin-middleware/img2.png){: w="700" }
`c.Next()` 전 코드는 미들웨어가 등록된 순서대로 실행되고, `c.Next()` 후 코드는 미들웨어가 등록된 순서의 역순으로 실행된다고 이해하면 쉽다.

#### Case 2

이제 문제를 바꿔서 `errorHandlerMiddleware`에서 `c.Next()`가 아닌 `c.Abort()`로 흐름을 멈추면 어떻게 될지 생각해보자.

```go
func errorHandlerMiddleware() gin.HandlerFunc {
    return func(c *gin.Context) {
        fmt.Println("Error handler middleware before")
        c.Abort()
        fmt.Println("Error handler middleware after")
    }
}
```

답을 바로 공개하자면 아래와 같다.

```go
Logging middleware before
Error handler middleware before
Error handler middleware after // 왜 출력되었지?
Logging middleware after
```

왜 에러 핸들러 미들웨어의 `c.Abort()` 이후 로그도 출력이 되었을까?

앞서 설명했듯이 `c.Abort()`를 한다고 해당 미들웨어의 동작이 종료되는 것은 아니라서 로직은 계속 수행이 된다.
만약 미들웨어의 동작이 바로 종료되길 바라면 `c.Abort()` 직후에 return 을 해주어야한다.

그리고, 출력을 보면 로깅 미들웨어의 `c.Next()` 이후 로그도 출력이 되었음을 알 수 있다.
`c.Abort()`를 한다고 그 미들웨어에서 동작이 멈추는 것이 아니라, 제어권이 다시 이전 미들웨어로 넘어가기 때문에 `c.Abort()`가 작성된 미들웨어까지의 미들웨어들은 역순으로 다시 수행이 된다.

## 오픈소스 미들웨어
필요한 미들웨어들을 직접 구현해주어도 좋지만, 자주 쓰이는 미들웨어들은 [gin-gonic/contrib](https://github.com/gin-gonic/contrib)에서 오픈소스 패키지로도 제공이 되고 있다.

![img3](/assets/images/programminglanguage/golang/gin-middleware/img3.png){: w="400" }



이 중에서 몇가지 잘 쓰이는 것 같은 미들웨어를 소개해본다.

1. **[CORS middleware](https://github.com/gin-contrib/cors)**: CORS 설정을 쉽게 할 수 있는 미들웨어.
```go
func main() {
  router := gin.Default()
  // CORS for https://foo.com and https://github.com origins, allowing:
  // - PUT and PATCH methods
  // - Origin header
  // - Credentials share
  // - Preflight requests cached for 12 hours
  router.Use(cors.New(cors.Config{
    AllowOrigins:     []string{"https://foo.com"},
    AllowMethods:     []string{"PUT", "PATCH"},
    AllowHeaders:     []string{"Origin"},
    ExposeHeaders:    []string{"Content-Length"},
    AllowCredentials: true,
    AllowOriginFunc: func(origin string) bool {
      return origin == "https://github.com"
    },
    MaxAge: 12 * time.Hour,
  }))
  router.Run()
}
```

2. **[pprof](https://github.com/gin-contrib/pprof)**: 프로그램 성능 프로파일링을 도와주는 미들웨어. 성능 프로파일링 데이터를 수집하고 시각화하는 도구인 pprof의 포맷에 맞춰 런타임 프로파일링 데이터를 모아주는 역할.
```go
func main() {
  router := gin.Default()
  adminGroup := router.Group("/admin", func(c *gin.Context) {
    if c.Request.Header.Get("Authorization") != "foobar" {
    c.AbortWithStatus(http.StatusForbidden)
       return
    }
    c.Next()
  })
  pprof.RouteRegister(adminGroup, "pprof")
  router.Run(":8080")
}
```
프로파일 적용은 [d2글](https://d2.naver.com/helloworld/8404108) 도 참고하면 좋을 것 같다.

3. **[zap](https://github.com/gin-contrib/zap)**: gin에서 zap을 활용한 로깅을 할 수 있도록 도와주는 미들웨어.
```go
func main() {
  r := gin.New()
  logger, _ := zap.NewProduction()
  // ginzap 미들웨어 추가
  r.Use(ginzap.Ginzap(logger, time.RFC3339, true))
  // Panic 발생 시 에러 로그 기록
  r.Use(ginzap.RecoveryWithZap(logger, true))
  // 예제 ping 요청
  r.GET("/ping", func(c *gin.Context) {
    c.String(200, "pong "+fmt.Sprint(time.Now().Unix()))
  })
  // Panic 예제
  r.GET("/panic", func(c *gin.Context) {
    panic("An unexpected error happened!")
  })
  // 서버 시작
  r.Run(":8080")
}
```

그 외에도 세션 관리를 도와주는 미들웨어, casbin 기반 인증을 도와주는 미들웨어 등이 존재한다.

다만 오픈소스 미들웨어를 바로 사용할 경우에는 코드를 커스텀해주기 어려운 경우가 있어서 기존 코드와 호환이 안될 수도 있다. 이런 경우에는 해당 미들웨어 구현을 참고해서 커스텀하게 직접 구현하는 것이 나을 수 있다.

## 참고자료
[미들웨어를 사용하기](https://gin-gonic.com/ko-kr/docs/examples/using-middleware/)  
[Gin middleware examples](https://sosedoff.com/2014/12/21/gin-middleware.html)
