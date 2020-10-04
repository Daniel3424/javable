---
layout: post
title: "ETag with Spring"
author: "티거"
comment: "true"
tags: ["spring", "etag"]
toc: true
---

## ETag란 무엇일까요?

> ETag 또는 Entity Tag는 월드 와이드 웹 프로토콜인 HTTP의 일부다. 그것은 HTTP가 웹 캐시 유효성 검사를 위해 제공하는 몇 가지 메커니즘 중 하나로, 클라이언트가 조건부 요청을 할 수 있게 한다.
>
> ...
>
> ETag는 웹 서버가 URL에서 찾은 리소스의 특정 버전에 할당한 불투명한 식별자다. 만약 그 URL의 리소스 표현이 변경된다면, 새롭고 다른 ETag가 할당된다. 이러한 방식으로 사용되는 ETag는 지문과 유사하며, 한 자원의 두 가지 표현이 동일한지 여부를 결정하기 위해 빠르게 비교할 수 있다.
>
> [위키백과](https://en.wikipedia.org/wiki/HTTP_ETag)

간단하게 말하면 ETag(entity tag)는 웹 서버가 주어진 URL의 콘텐츠가 변경되었는지 알려주고 이를 반환하는 HTTP 응답 헤더입니다.

## 왜 사용할까요?

먼저 캐시는 왜 사용할까요? 

> 캐시는 컴퓨터 과학에서 데이터나 값을 미리 복사해 놓는 임시 장소를 가리킨다. 캐시는 캐시의 접근 시간에 비해 원래 데이터를 접근하는 시간이 오래 걸리는 경우나 값을 다시 계산하는 시간을 절약하고 싶은 경우에 사용한다. 캐시에 데이터를 미리 복사해 놓으면 계산이나 접근 시간 없이 더 빠른 속도로 데이터에 접근할 수 있다.
>
> [위키백과](https://ko.wikipedia.org/wiki/%EC%BA%90%EC%8B%9C)

캐시를 사용하면 불필요한 요청을 줄이면서 서버의 부하를 줄일 수 있고, 미리 캐시에 저장해 놓은 값을 사용함으로써 빠른 응답을 할 수 있습니다.

ETag는 저희가 사용하는 캐시가 유효한지 검증하기 위해 사용합니다. 서버의 리소스가 변경된다면 어떨까요? 그러면 저희가 저장해 놓은 캐시의 데이터와 서버의 리소스 데이터는 다른 값이겠죠? 캐시가 서버에게 리소스가 변경되었는지 안 되었는지 물어보는 것을 **캐시 유효성 검사**라고 합니다. 저희는 ETag를 사용하여 **캐시 유효성 검사**를 하는 겁니다.

## 클라이언트와 서버 간 통신을 어떻게 하는지 알아볼까요?

먼저 첫 요청을 보냅니다.

```http
curl -H "Accept: application/json" 
     -i http://localhost:8080/spring-boot-rest/foos/1
```

그러면 서버는 `ETag`를 응답 header에 담아서 보냅니다.

```http
HTTP/1.1 200 OK
ETag: "f88dd058fe004909615a64f01be66a7"
Content-Type: application/json;charset=UTF-8
Content-Length: 52
```

클라이언트는 재요청할 때 `ETag`를 header의 `If-None-Match`에 담아 요청을 보냅니다. 여기서 `If-None-Match`는 뭘까요? ETag를 사용할 때 Conditional headers로  `If-None-Match`와 `If-Match`가 있습니다. 

간단하게 설명하면

[If-None-Match](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/If-None-Match) - 클라이언트에서 캐싱된 ETag와 서버의 ETag가 다를 때 요청을 처리합니다.

[If-Match](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/If-Match)  - 클라이언트에서 캐싱된 ETag와 서버의 ETag가 같을 때 요청을 처리합니다.

```http
curl -H "Accept: application/json" 
     -H 'If-None-Match: "f88dd058fe004909615a64f01be66a7"'
     -i http://localhost:8080/spring-boot-rest/foos/1
```

리소스가 바뀌지 않았기 때문에 서버는 `304 Not Modified`를 응답합니다. `ETag`는 이전 요청에 대한 응답과 같습니다.

> [304 Not Modified](https://developer.mozilla.org/ko/docs/Web/HTTP/Status/304)  - 이것은 캐시를 목적으로 사용됩니다. 이것은 클라이언트에게 응답이 수정되지 않았음을 알려주며, 그러므로 클라이언트는 계속해서 응답의 캐시된 버전을 사용할 수 있습니다.

```http
HTTP/1.1 304 Not Modified
ETag: "f88dd058fe004909615a64f01be66a7"
```

이제 리소스를 바꿔줍니다.

```http
curl -H "Content-Type: application/json" 
     -i -X PUT --data '{ "id":1, "name":"Transformers2"}' 
     http://localhost:8080/spring-boot-rest/foos/1
```

요청에 대한 응답을 확인합니다.

```http
HTTP/1.1 200 OK
ETag: "d41d8cd98f00b204e9800998ecf8427e" 
Content-Length: 0
```

지난 요청을 다시 합니다. 요청을 다시 할 때는 마지막으로 가지고 있던 ETag를 담아서 보낼 것입니다. 

```http
curl -H "Accept: application/json" 
     -H 'If-None-Match: "f88dd058fe004909615a64f01be66a7"' 
     -i http://localhost:8080/spring-boot-rest/foos/1
```

클라이언트에서 보낸 ETag와 서버의 ETag가 다르기 때문에 요청을 처리합니다. 리소스가 바뀌었으니 새로운 ETag를 header에 담아 보냅니다. 새로운 요청을 처리했기 때문에 서버는 `200 OK`를 응답합니다.

```http
HTTP/1.1 200 OK
ETag: "03cb37ca667706c68c0aad4cb04c3a211"
Content-Type: application/json;charset=UTF-8
Content-Length: 56
```

## ETag를 사용하지 않은 API vs 사용한 API

ETag 사용 예시와 ETag를 사용한 API와 사용하지 않은 API를 비교를 설명하기 위해 간단하게 Controller를 작성하였습니다.

```java
@RequestMapping("/posts")
@RestController
public class PostController {

    // ...
    
    @GetMapping("/no-etag")
    public ResponseEntity<List<PostResponse>> findAllWhenNoETag() {
        return ResponseEntity.ok().body(postService.findAll());
    }
    
    @GetMapping("/etag")
    public ResponseEntity<List<PostResponse>> findAllWhenETag() {
        return ResponseEntity.ok().body(postService.findAll());
    }
    
    // ...
}
```

그리고 ETag 설정으로 ShallowEtagHeaderFilter를 Bean으로 등록해줍니다.

```java
@Configuration
public class ETagHeaderFilter {

    @Bean
    public ShallowEtagHeaderFilter shallowEtagHeaderFilter() {
        return new ShallowEtagHeaderFilter();
    }
}
```

추가 필터를 구성할 필요 없다면 위의 코드와 같이 작성하셔도 됩니다. 하지만 저는 ETag를 사용한 API와 사용하지 않은 API를 비교하기 위해 필터를 사용하겠습니다.

추가 필터 구성을 하고 싶다면 다음과 같이 설정해주면 됩니다.

```java
@Configuration
public class ETagHeaderFilter {

    @Bean
    public FilterRegistrationBean<ShallowEtagHeaderFilter> shallowEtagHeaderFilter() {
        FilterRegistrationBean<ShallowEtagHeaderFilter> filterRegistrationBean
                = new FilterRegistrationBean<>( new ShallowEtagHeaderFilter());
        filterRegistrationBean.addUrlPatterns("/posts/etag");
        filterRegistrationBean.setName("PostAPIFilter");
        return filterRegistrationBean;
    }
}
```

현재 PostController에서 `/posts/etag`만  etag를 사용한다는 설정입니다. 만약 `/post/`에 대해 전부 ETag를 설정하고 싶다면  `filterRegistrationBean.addUrlPatterns("/posts/*")` 이렇게 설정하시면 됩니다.

그럼 이제 `/no-etag`와 `/etag`를 호출해 볼까요? 호출하면 네트워크 상에 어떤 일이 일어날까요?

![image](https://user-images.githubusercontent.com/45934117/94986209-cb10ab80-0597-11eb-9b8d-d88597fcc56e.png)

얼핏 보면 둘의 차이가 안 보입니다. 하지만 Response Headers를 보면 차이를 볼 수 있습니다.

![image](https://user-images.githubusercontent.com/45934117/94986113-e929dc00-0596-11eb-84c1-7f12b318c509.png)

두 응답의 차이를 볼 수 있는 곳은 ETag일 것입니다. `/etag`는 ETag를 사용하고 있기 때문에 응답으로 ETag를 header에 해시값으로 보내줍니다. 이는 재요청할 때 header의 `If-None-Match`의 값으로 보내 줄 것입니다.

```http
If-None-Match: "0fad8e1b47f45fa4ce7fef400e87c9289"
```

이렇게 ETag를 `/etag` 요청 header의 `If-None-Match`에 담아 재요청해 보겠습니다.

![image](https://user-images.githubusercontent.com/45934117/94986192-af0d0a00-0597-11eb-8966-f7123a1fd879.png)

`etag`를 보면 앞서 설명했듯이 같은 요청에 대해서 304 상태 코드를 응답합니다. 이는 서버에서 캐시 유효성 검사를 한 결과 변경되지 않았기 때문입니다. 

여기서 봐야 할 것은 사이즈입니다. `no-etag`는 재요청에 대해서 `796B -> 796B`인 반면에 `etag`는 `820B -> 145B` 입니다. 이유는 ETag를 사용하지 않으면 했던 일을 똑같이 또 하지만, ETag를 사용하면 같은 요청에 대해서 변경된 리소스가 없다면 304 상태 코드와 ETag를 header에 담아 보내줄 뿐 요청에 대한 리소스를 또 보내지 않습니다.

이제 ETag를 사용했을 때와 사용하지 않을 때 차이가 있는 것을 아시겠죠?😊😊

## Test Code 작성

테스트할 때 중요하게 볼 것은 두 가지라고 생각합니다. 

첫 번째, 첫 요청을 보낼 때 응답에 "ETag"를 가졌는지
두 번째, header "If-None-Match"에 받은 etag 값을 넣고 같은 요청을 또 보낼 때 `304 Not Modified`를 응답하는지
추가로 리소스를 변경한 다음 다시 요청 보냈을 때 `200 OK`를 응답하는지 본다면 더 좋을 것 같습니다.

```java
@Autowired
private MockMvc mockMvc;

@Test
void findAll_ETag() throws Exception {
    create(); // 먼저 데이터를 만들어 줍니다.

    String url = "/posts/etag";

    // 첫 번째 요청을 보냅니다.
    MvcResult mvcResult = this.mockMvc.perform(get(url))
        .andDo(print())
        .andExpect(status().isOk()) // 첫 요청이기 때문에 200 OK 
        .andExpect(header().exists("ETag")) // ETag를 사용하고 있기 때문에 header가 ETag를 가지고 있는지 확인해 줍니다.
        .andReturn();

    String etag = mvcResult.getResponse().getHeader("ETag");

    // 두 번째 요청을 보냅니다.
    mvcResult = this.mockMvc.perform(get(url).header("If-None-Match", etag)) // 응답받은 ETag를 해더에 담아 보냅니다.
        .andDo(print())
        .andExpect(status().isNotModified()) // 유효성 검사를 하고 변경이 안되었기때문에 304 Not Modified
        .andExpect(header().exists("ETag"))
        .andReturn();

    update(); // 리소스를 변경합니다.

    etag = mvcResult.getResponse().getHeader("ETag");

    this.mockMvc.perform(get(url).header("If-None-Match", etag)) // 두 번째 응답에 대한 ETag를 header에 담아 보냅니다.
        .andDo(print())
        .andExpect(status().isOk()) // 리소스가 변경되었기 때문에 200 OK 
        .andExpect(header().exists("ETag"))
        .andReturn();
}

// ...

```

## 마무리

ETag가 무조건 좋은 것은 아니다. 만약 여러 대의 서버를 운영하고 있다면 같은 콘텐츠이지만 ETag가 다를 수 있기 때문이다. 따라서 ETag를 사용한다면 이러한 문제점을 인지하고 사용해야 할 것이다.

😊😊글을 읽으면서 제가 잘못 알고 있는 점, 틀린 점, 추가했으면 하는 점 등 아낌없는 피드백 부탁합니다.😊😊

## 참고자료

[ETags for REST with Spring](https://www.baeldung.com/etags-for-rest-with-spring)