---
layout: post
title: "Spring MVC Redirect OutOfMemory 문제"
date: 2020-08-10
categories: [Spring]
tags: [spring, mvc, redirect, oom, troubleshooting]
---

운영 환경에서 KMC 본인인증 서비스 운영 중에 OutOfMemory(OOM)이 발생하여 서버가 종료되는 장애가 발생했다. 다행히 무거운 트래픽의 API가 아니고 HA(High Availability)로 구성되어 있어 큰 문제는 없었지만, 즉시 트러블슈팅이 필요했다.

<!-- more -->

## 문제 상황

### 문제가 된 코드

```java
@RequestMapping("/request")
public String requestForm(@ModelAttribute @Valid AwesomeRequest request) {
  log.info("{}", request);
  String cert = 한국모바일인증클라이언트.encryptRequest(request.get사용자ID());
  return "redirect:" + "어떤 URL" + cert;
}
```

이 API는 사용자가 KMC(한국모바일인증)에 요청하면 모바일 인증 페이지로 리다이렉트(302 응답코드)를 응답하는 기능이다. 문제는 **매번 다른 `cert` 값**이 생성되면서 고유한 redirect URL이 만들어진다는 점이었다.

## 원인 분석

### Spring Framework의 뷰 해결 메커니즘

Spring Framework의 뷰 해결 메커니즘은 **AbstractCachingViewResolver**를 여러 뷰 리졸버의 기본 클래스로 사용한다. 이 리졸버는 **뷰 이름과 로케일을 기반으로 한 키**를 사용하여 HashMap에 뷰를 캐시한다.

```java
public View resolveViewName(String viewName, Locale locale) throws Exception {
	if (!isCache()) {
		return createView(viewName, locale);
	}
	else {
		Object cacheKey = getCacheKey(viewName, locale);
		synchronized (this.viewCache) {
			View view = this.viewCache.get(cacheKey);
			if (view == null && (!this.cacheUnresolved || !this.viewCache.containsKey(cacheKey))) {
				// Ask the subclass to create the View object.
				view = createView(viewName, locale);
				if (view != null || this.cacheUnresolved) {
					this.viewCache.put(cacheKey, view); // <- 메모리 누수 발생!!
					if (logger.isTraceEnabled()) {
						logger.trace("Cached view [" + cacheKey + "]");
					}
				}
			}
			return view;
		}
	}
}
```

### 메모리 누수의 핵심

캐시 키(key)를 작성할 때 `viewName`을 사용하는데, 이 `viewName`이 **뷰를 요청할 때마다 다르다면** 메모리에 축적된다. 

실제로는 다음과 같은 과정을 거친다:

1. Controller에서 `return "redirect:" + dynamicUrl` 형태로 리턴
2. View 클래스로 변환 작업 진행
3. `org.springframework.beans.factory.config.BeanPostProcessor` 구현체 동작
4. 그 중 하나인 `AnnotationAwareAspectJAutoProxyCreator` 클래스가 `ConcurrentHashMap<Object, Boolean>` 타입 객체에 **key: viewName, value: 필요 여부(boolean)** 형태로 **갯수 제한 없이 저장**

### 힙 덤프 분석 결과

Memory Analyzer Tool (MAT)로 확인한 결과:


![MAT Analyzer](https://github.com/user-attachments/assets/77a843ff-9a2c-48f8-8b4f-030b9db0ab0e)

- **Suspect 객체**: `java.util.concurrent.ConcurrentHashMap$Node`
- **메모리 점유 패턴**: RedirectView 관련 객체들이 대량으로 생성되어 있음
- **키값 패턴**: `org.springframework.web.servlet.view.RedirectView_redirect:{동적URL}` 형태

Map의 키값들을 확인해보니 RedirectView가 많이 생성되어 있었고, 각각 다른 동적 URL을 가지고 있었다.
![MAT Analyzer 2](https://github.com/user-attachments/assets/cedb8ca8-5ad4-4116-bd43-3e4ab509c09e)


## 해결 방법

### 올바른 구현 방식

이 문제는 302 리다이렉트 URL을 요청마다 다르게 보내야 한다면, **RedirectView** 혹은 **ModelAndView** 객체를 사용해야 한다.

#### 방법 1: RedirectView 사용

```java
@RequestMapping("/request")
public String requestForm(@ModelAttribute @Valid AwesomeRequest request) {
	log.info("{}", request);
  	String cert = 한국모바일인증클라이언트.encryptRequest(request.get사용자ID());
  	return new RedirectView(String.format(리다이렉트URLTemplate, cert));
}
```

#### 방법 2: ModelAndView 사용
```java
@RequestMapping("/request")
public ModelAndView requestForm(@ModelAttribute @Valid AwesomeRequest request) {
	log.info("{}", requestVo);
	String cert = 한국모바일인증클라이언트.encryptRequest(request.get사용자ID());
	ModelAndView modelAndView = new ModelAndView();
	RedirectView redirectView = new RedirectView();
	redirectView.setUrl(String.format(리다이렉트URLTemplate, cert));
	modelAndView.setView(redirectView);
	
	return modelAndView;

```


### 성능 개선 결과

개선 후 동일한 환경에서 테스트한 결과:

#### 기존 방식 (문제 있는 코드)
- **TPS**: 불안정하며 시간이 지날수록 감소
- **메모리**: 지속적으로 증가 후 FullGC 빈발
- **에러율**: 높음

#### 개선 방식 (RedirectView/ModelAndView 사용)
- **TPS**: 안정적이고 일정하게 유지
- **메모리**: 안정적인 패턴 유지, FullGC 없음
- **에러율**: 0%

## 추가 고려사항

### 언제 이 문제가 발생하는가?

- **고위험**: UUID, 난수, 타임스탬프 등이 포함된 동적 URL
- **저위험**: 고정된 URL 패턴 (예: `/home`, `/login` 등)

### 실제 운영 환경에서의 위험성
```java
// 위험한 코드 예시
@GetMapping("/link/{key}")
public String redirectLinkPage(@PathVariable String key) {
  	String dynamicUrl = generateDynamicUrl(key); // 매번 다른 URL 생성
	return "redirect:" + dynamicUrl; // <- 메모리 누수 위험!
}
```

이런 코드가 **대량의 트래픽**을 받으면:
1. 순간적으로 많은 고유 URL이 생성됨
2. 각 URL이 캐시에 저장됨
3. 캐시 크기 제한이 없어 메모리 사용량 급증
4. OutOfMemory 발생으로 서비스 장애

### 모니터링 포인트

운영 환경에서는 다음을 모니터링해야 한다:

- **Heap 메모리 사용량** 추이
- **FullGC 빈도** 및 시간
- **ConcurrentHashMap** 관련 메모리 점유율
- **RedirectView** 객체 생성 수

## 결론

Spring MVC에서 동적 URL로 redirect할 때는 반드시 **RedirectView** 또는 **ModelAndView**를 사용해야 한다. 단순해 보이는 `"redirect:" + dynamicUrl` 방식은 메모리 누수를 일으켜 심각한 서비스 장애로 이어질 수 있다[4][5].

특히 다음과 같은 상황에서는 더욱 주의해야 한다:
- **높은 트래픽** 환경
- **동적 파라미터**가 포함된 redirect URL
- **UUID, 랜덤값** 등이 포함된 URL

이 문제는 Spring Framework의 잘 알려진 이슈이지만, 여전히 많은 개발자들이 놓치기 쉬운 부분이다. 코드 리뷰와 성능 테스트를 통해 이런 잠재적 위험을 미리 발견하고 대응하는 것이 중요하다.

## 참고 자료

- [Spring Framework GitHub Issues](https://github.com/spring-projects/spring-framework/issues/14698)
- [Spring 공식 문서 - Redirect and Forward](https://docs.spring.io/spring-framework/docs/current/reference/html/web.html#mvc-redirecting)



