---
title: Spring Framework의 구조에 대해
author: abruption
date: 2021-04-21 14:10:00 +0900
categories: [Leature, Spring]
tags: [Java, Spring, Framework]
---

## Spring MVC 구성 주요 컴포넌트
![](https://linked2ev.github.io/assets/img/devlog/201908/mvc-s1.png)

- DispatcherServlet: 모든요 청을 최초로 받아들이는 Servlet
- HandlerMapping: 클라이언트의 요청을 처리할 Controller를 찾는 작업
- Controller: Client의 요청을 처리
- ModelAndView: View에게 값을 전달하기 위해 사용되는 객체
- ViewResolver: View를 찾는 작업을 처리
- View: 화면구성

### Spring의 동작원리
- 라이브러리 : Application의 실행을 위해 특정 기능들을 미리 구현해 놓은 부분 (Ex. pom.xml, Maven) 
- 설정 파일 : Library를 해당 프로젝트에 맞게 활용하기 위해서 설정되는 값 정보 (Ex. Controller)
- 어노테이션 : 자바 코드에서 Spring Framework의 기능을 작동시키기 위한 @로 시작하는 회색글자

<br />

## Annotation Type
@RequestParam : 넘어오는 데이터를 받아오기 위해서 사용되는 Annotation 
- 파라미터가 변수의 이름이 다를 경우도 활용
- 예외 방지를 위한 DefaultValue 지정 가능
- 필수 여부를 선택 가능(ex. required=false)

@RequestMapping : URL을 컨트롤러의 메소드와 Mapping할 때 사용하는 Annotation

@Service : 보통 DAO로부터 받아온 데이터를 전달 받아 가공하는 역할을 하는 클래스 
- Service로 선언되면 Spring Framework에서 서비스로 인식
- 클래스로 선언 시 이름은 로직 + Service로 선언 (Ex. BoardService)
- 선언 방법 : @Service

@Component : Controller, Service, Repository 외 스프링에서 관리하는 객체로 등록하여 사용할 경우 활용 
- 선언 방법 : @Component

@Autowired : 객체의 의존성을 자동으로 할당하는 수단
- 해당 어노테이션으로 할당되는 객체는 하나만 생성되어 관리가 된다.
- 그렇다고 전역 변수로 활용되는 것은 아니다.
**※ BoardService boardService = new boardService()로 객체 생성 시에는 숫자가 각각 올라가지만, @Autowired 어노테이션을 사용할 시에는 객체를 공유한다 → Singleton 방식으로 객체가 관리됨**

@MapperScan : mapper.xml 파일들이 바라볼 기본 패키지 위치를 지정해주는 Annotation
- Mapper 스캔을 위한 어노테이션 (Ex. @MapperScan(value = {“com.inhatc.spring.mapper”})

<br />

## Spring Framework의 객체관리 - Singleton
- Singleton : 객체 활용 시 최초에 한번만 메모리를 할당하고 처음 할당한 메모리 내에서 객체를 활용하는 것
  - 메모리의 낭비 방지, 전역에서 사용되는 메모리 영역이 동일하기 때문에 데이터 공유가 편리함 
  - 객체의 로딩 시각을 줄여줌
  - 객체간 결합도가 높아질 수 있음(객체 지향에 맞지 않음)
  - 테스트가 어려워질 수 있음
- 활용 방법 : @Service, @Component, @Repository 등으로 선언한 클래스를 객체화 할 경우 @Autowired를 사용하여 접근