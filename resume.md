---
layout: resume
title: Resume
permalink: /resume/
---

<div id="resume-header">
    <div id="profile">
    </div>
    <div id="info">
        <div id="info-name">김진형 <sup>Rajin</sup></div>
        {% include icon-github.html username=site.github_username %} <br>
        {% include icon-twitter.html username=site.twitter_username %} <br>
        <a href="mailto:rajin9601@gmail.com">rajin9601@gmail.com</a>
    </div>
</div>

# Introduction

웹, 앱, 서버를 포함한 다양한 개발을 해왔습니다. 개발하면서 느껴지는 불편함을 개선하는 것이 전체 생산성 향상에 큰 도움이 된다고 생각합니다. 그래서 개발자들이 쉽게 실수하지 않을 만한 환경을 만들기 위해 노력합니다. 이를테면 깨진 유리창을 만들지 않기 위해 전체적인 코드 퀄리티를 신경 쓰며 비교적 큰 규모의 리팩토링을 자주 해왔습니다. 어려운 문제에 대해 동료들과 토론하고 현재 상황에 가장 적합한 해결책을 도출하는 것을 즐깁니다.

좋은 제품을 개발하기 위해서는 기획, 개발, 디자인 파트 구성원 간의 활발한 소통이 중요하다고 생각합니다. 기획의 흐름을 파악하는 동시에 개발적인 맥락을 이야기함으로써 오해 없는 소통을 추구하며, 이를 바탕으로 최선의 선택을 하려 노력합니다. 기획·개발 단계에서 어려운 부분이 있다면 목표를 향한 새로운 해결책을 적극적으로 제시합니다.

# Experience

## 타다 서버 개발자 <sup>2022.4 ~ 2023.8</sup>

- 차량호출 서비스 (80명~, 개발팀 15~25명, 서버팀 6~9명)
- 호출, 배차, 정산, 어드민 등 기존 비즈니스 로직 유지보수 및 문제 대응
- 서버 개발/운영시 불편함을 해소하기 위한 리팩토링 및 기능 개발
- AWS 인프라 / MySQL 데이터베이스 유지보수, 모니터링 및 장애 대응
- 4명으로 이루어진 사일로에서 드라이버 유효운행시간을 늘리기 위한 기능 서버 담당 <sup>2022.08 ~ 2023.02</sup>
- Tech
    - Kotlin, Spring Boot, gRPC, Protocol Buffers, MySQL, Hibernate, Redis, SQS
    - Kubernetes, Helm, Terraform
    - [관련 VCNC 개발블로그 글](https://blog-tech.tadatada.com/2019-01-28-tada-system-architecture)

### 티머니 단말기 연동 & 전용선 소통을 위한 추가적인 인프라 작업

- 티머니 단말기를 통한 결제를 할수 있도록 타다 서버와 티머니 서버간의 연동
- 정산시 타다 정보와 티머니측 정보가 일치하는지 확인하는 시스템 설계 
- 티머니와 전용선을 통한 HTTPS 소통을 위한 인프라 작업 진행
- 전용선 연결의 명확한 문제 해결을 위해 연결 로그 분석

### 무이자 대출 시스템 개발 및 운영

- VCNC 측에서 드라이버에게 무이자 대출을 해주고 원금은 12개월에 걸쳐서 운행 수익의 n% 를 상환수수료로 걷어 대출상환을 하도록 하는 시스템 설계 및 구현
- 1개월에 100만원까지만 상환수수료를 걷고 그 이상 운행하였을시엔 상환수수료를 상환하지 않는 구조 설계 및 구현
- 티머니를 통한 정산 시스템과 상환수수료를 연동
- 운영하며 나온 CS 대응을 위한 기능 구현

### 드라이버 등급제 개발 및 운영

- 드라이버의 운행유도를 위해 운행 횟수에 따라 등급을 부여하고 등급별로 추가 기능들을 제공
- 어뷰징을 방지하기 위한 정책 설계 및 등급유지조건 설계
- CS 대응을 위한 어드민 추가 개발

### CI 시간 최적화

- CI의 실행 시간이 어느 순간부터 계속 timeout에 걸리는 문제 해결
- async profiler를 통해 GC 실행시간이 병목임을 발견
- Memory dump 후 분석결과, [mockito-inline](https://github.com/mockito/mockito/issues/1614) 과 [spring 의 Cglib2AopProxy](https://github.com/spring-projects/spring-framework/issues/12663)에 메모리 leak 이 존재하는 것을 발견 & Work-around 적용

## 타다 클라이언트 개발자 <sup>2018.7 ~ 2022.4</sup>

- 클라이언트 팀 (5 ~ 6명 규모. 2020.10 부터 Android Tech Lead)
- 타다 시작시, iOS / 안드로이드 앱 아키텍쳐 설계
- 타다 라이더 / 드라이버 앱,  핸들모아 (대리운전 기사용 앱) 출시 및 유지보수
- 클라이언트 내부 라이브러리 & 툴링 개발 ([PPT 발표 참고](https://docs.google.com/presentation/d/e/2PACX-1vRBYACbRdO0rK71Ee-DHxL_TcjLLIpJnpD39S3OUPIupKQKZ_fV4ofq81oMY56yVLalDeTwflH1vkQ2/pub?start=false&loop=false&delayms=10000&slide=id.p))
- 인앱 브라우저를 최대한 네이티브처럼 보이기 위해 전체 구조 설계 & 앱쪽 구현
- Tech
    - Kotlin/Swift, [RIBs](https://github.com/uber/RIBs), RxJava/Swift, gRPC, Protocol Buffers, Moshi, [Motif](https://github.com/uber/motif)
    - [관련 VCNC 개발블로그 글](https://blog-tech.tadatada.com/2019-05-08-tada-client-development)

### RIBs 를 VCNC 상황에 맞게 수정 & 유지보수

- Uber 에서 만든 라이브러리 [RIBs](https://github.com/uber/RIBs) 을 VCNC 에서 사용하며 불편한 부분들을 수정하여 사용
- 애니메이션을 RIB 내부에서 지원하여 View와 ViewModel의 lifecycle 이 RIB 퇴장 애니메이션시 어색하지 않도록 수정
- 안드로이드 BackStack 에 해당하는 RIBs의 ScreenStack 구현
- Activity 단위로 이뤄졌던 firebase screen logging 을 RIBs 상황에 맞게 구현
- 이전 RIBs 에서 아쉬웠던 부분들을 고친 버전을 새롭게 개발하는 앱(핸들모아)에 적용해본후 동료들의 피드백을 받아 타다 앱에 적용

### protocol buffer generator code 커스텀화 & 유지보수

- Protocol Buffers 를 통해 생성된 java code 의 경우, Builder 패턴 사용과 독자적인 JSON serialize, deserialize 방식을 제공 중
- Kotlin data class 형태와 클라이언트에서 많이 사용하는 JSON Library 를 사용하기 위해 proto 파일이 다른 프로그래밍 언어로 transpile 되는 과정을 이해한 후, 원하는 형태의 generated code 를 만들기 위해서 protoc compiler plugin 을 수정함
- oneof 를 enum class 형태로 제공하도록 수정
- Protobuf 3.15 에 추가된 optional 을 지원하도록 수정
- Kotlin, Swift 버전 제작 후, TypeScript 에서 최소 용량을 가지기 위해 binary format 을 사용하지 않고 JSON 직렬화/역직렬화에 꼭 필요한 코드만 넣은 버전을 제작

### 네이티브 앱처럼 보이는 인앱 브라우저

- 앱의 기능의 일부를 웹으로 구현하기 위한 베이스 작업
- 인앱 브라우저를 네이티브 앱처럼 보이기 위한 구조 설계
- 웹과 앱간의 소통 api 설계 및 안드로이드 파트 구현
- [stackflow](https://github.com/daangn/stackflow) 와 화면 진입시 로딩/에러/애니메이션 처리에 집중

## 비트윈 안드로이드 개발자 <sup>2017.7 ~ 2018.7</sup>

타다에서 클라이언트 개발자로 일하기 전에, 비트윈에서 1년 간 안드로이드 앱 개발을 했습니다.

- 안드로이드 팀 (4명)
- [움짤기능](https://youtu.be/LHHKSWS7oTg?t=114) : Working Prototype 까지 작업
    - Shader를 사용해 필터를 입혀 실시간으로 보여주면서 파일로 저장.
- 음성통화 : WebRTC, gRPC 를 적용하여 개발
    - 프로토콜 디자인에 참여

## 그 외 경력

- 42 Company (2016.12 - 2017.1) - 컨텐츠 크롤러 개발 & 내부 어드민 효율화
- Ultracaption (2015.7 - 2015.8) - PLAIN iOS 앱 개발

# Skills

- Kotlin(Java), Swift, Typescript
- Spring Boot, gRPC, Protobuf, Hibernate, Redis, SQS, RxJava/Reactor
- Kubernetes, Helm, Terraform

# Presentations

- [Kotlin Refactoring Tool](https://docs.google.com/presentation/d/1PnXGOxkTa466qV2SUrlLVtQy_QmWQq21NOp35hXVX9s/edit?usp=sharing) <sup>2022.10</sup>
    - 큰 규모의 기계적 리팩토링을 위해 만든 Kotlin 코드를 프로그래밍으로 변경하는 툴 설명
- [타다 안드로이드 기반 작업](https://docs.google.com/presentation/d/e/2PACX-1vRBYACbRdO0rK71Ee-DHxL_TcjLLIpJnpD39S3OUPIupKQKZ_fV4ofq81oMY56yVLalDeTwflH1vkQ2/pub?start=false&loop=false&delayms=10000&slide=id.p) <sup>2018.12</sup>
    - 타다 클라이언트 개발시 했던 기반작업들 요약

# Education

- 서울대학교 컴퓨터공학 학사 <sup>2014 ~ 2021</sup>
    - 산업기능요원 <sup>2017 ~ 2020</sup>
    - [WaffleStudio](https://www.facebook.com/wafflestudio/) 동아리 활동 <sup>2015 ~ 2016</sup>
- 경기과학고등학교 <sup>2011 ~ 2014</sup>
    - 한국정보올림피아드 전국 본선 금상 <sup>2013</sup>

# Side-Projects

- [SNUTT2](https://snutt.kr/) <sup>2015 ~ 2021</sup>
    - 5명 중 iOS 담당.
    - [Link to AppStore](https://itunes.apple.com/kr/app/snutt-서울대학교-시간표-앱/id1215668309?mt=8), [Link to Github](https://github.com/wafflestudio/SNUTT-iOS)
- [21 Days](http://store.steampowered.com/app/607660/21_Days/)
    - 총 6명. 2명 프로그래머 중 리드
    - BIC 2016 Excellence in Narrative Finalist

<div id="update-date">updated at 2023.12.13</div>
