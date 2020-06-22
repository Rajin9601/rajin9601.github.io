---
layout: post
title:  "Live Template"
date:   2020-06-21 22:00:00 +0900
categories: dev
img-overlay: 0.1
comments: true
---

Android Studio는 IntelliJ 기반 IDE이기 때문에 IntelliJ에서 제공하는 여러 기능을 사용할 수 있다. 그중 Live Template는 boilerplate나 자주 쓰는 패턴의 코드들을 좀 더 적은 타자수로 칠 수 있도록 도와주는 기능이다. 

# Live Template 기능

Live Template이란, 사용자가 미리 설정한 축약어로 긴 코드 타이핑을 대신할 수 있는 기능이다. 길지만 자주 쓰는 코드가 있을 때 유용하게 쓸 수 있다.

{% include video-fig.html src="/image/live_template/fullscreen_demo" alt="video of live template in action" caption="Live Template 사용 영상" %}

# 기본 기능 설명

{% include image-fig-fullwidth.html src="/image/live_template/live_templates.png" alt="screenshot of live template setting" caption="constraint layout에서 꽉 차게 만드는 live template" %}

Template group 안에 Live Template이 있고, 각 Live Template이 축약어 하나를 의미한다.

* Abbreviation : 축약어. 실제로 코드에서 타이핑하는 건 이 내용이다.
* Description : 축약어 자체도 자동완성을 시켜주는데, 그때 보이는 내용입니다. 팀원과 공유할 땐 자세히 씁시다.
* Template Text : 축약어에 해당하는 내용.
* Applicable in .... : 축약어가 적용될 수 있는 파일 확장자 & scope. Kotlin의 경우 어디서 쓸 수 있는지 좀 더 자세히 설정할 수 있다.

# Variable

Template에서 똑같은 단어가 여러 번 등장해야 의미가 있는 경우도 있는데, 그 경우엔 Variable 기능을 사용하면 된다.

{% include image-fig.html src="/image/live_template/github_link.png" alt="usage of variable in github pr link" caption="github PR link를 쉽게 달기 위한 live template" %}

이렇게 하면  `$NUMBER$` 를 한 번만 적기만 해도, 두 군데 모두 적히게 된다.

## Variable Expression

실제 코드를 쓸 때는 완벽히 똑같은 단어가 아니라 조금씩 다르게 반복해서 써야 하는 경우도 있다.
예를 들면, 타다에서는 API를 param, result 객체로 해놓고 사용하는데, API를 하나 만들 때마다 다음과 같은 코드를 작성해야 한다.

{% include image-fig-fullwidth.html src="/image/live_template/test_api_service.png" %}

DriverAuthenticate API를 만드는데 함수명, param, result, url 들이 다 비슷하지만 조금씩 다르다. 이런 경우에는 Variable에 Expression을 넣어줌으로써 해결할 수 있다.

{% include image-fig.html src="/image/live_template/test_api.png" %}

이렇게 template text를 적고 `Edit variables`를 누르면 variable마다 expression을 적을 수 있다.

{% include image-fig.html src="/image/live_template/test_api_variables.png" %}

apiName을 적으면 나머지가 다 적히도록 funName과 apiPrefix를 만들어 주면 된다. expression에 사용할 수 있는 기능들은 IntelliJ에서 제공하는 기능이나 Groovy Script를 사용할 수 있고 [여기서](https://www.jetbrains.com/help/idea/template-variables.html) 확인할 수 있다.

그리고 그 둘에 `Skip if defined`를 체크하면 앞에 apiName만 쓰고도 모든 내용이 채워지게 된다.

{% include video-fig.html src="/image/live_template/test_api_demo" caption="testApi 작동 영상" %}

## Variable Order

하지만 이렇게 하면 내가 치는 내용이 단순한 string이기 때문에 오타를 칠 확률이 높아진다. 이런 문제를 원천봉쇄하기 위해서는 value가 아니라 Params이나 Result를 먼저 쓰도록 하면 된다. 그러면 IDE에서 제공하는 code completion을 통해 실제 있는 class를 쳐야 하고, 그러면 오타를 칠 가능성이 사라질 것이다.

그래서 Live Template를 다음과 같이 바꿔서 시도해보자

{% include image-fig.html src="/image/live_template/test_api_2.png" %}
{% include image-fig.html src="/image/live_template/test_api_2_variables.png" %}

apiParam 만 채워 넣으면 나머지 변수들이 채워지도록 설정했다. 하지만 실제 실행하면 다음과 같이 실행된다.

{% include video-fig.html src="/image/live_template/test_api_2_demo" caption="testApi2 영상. 커서가 apiParam으로 가지 않는다." %}

이렇게 해도 커서가 apiName으로 먼저 가서 문제가 해결되지 않는걸 볼 수 있다. 왜냐면 커서가 가는 순서는 Edit variables 창에서 보이는 variable 순서이기 때문이다.
Edit variables 창에서 왼쪽 밑에 있는 화살표를 통해 variable의 순서를 바꿔서 apiParam을 맨 위로 올리면 원하는 것처럼 잘 작동하는 것을 볼 수 있다.

{% include image-fig.html src="/image/live_template/test_api_3_variables.png" %}

{% include video-fig.html src="/image/live_template/test_api_3_demo" caption="원하는 대로 잘 작동한다!" %}

# 팀원들과 공유하기

IntelliJ에서 공유하려면 IDE에서 export를 해서 공유해야 한다[^1]. GUI로 일일이 해줘야 해서 귀찮기도 하고, 팀원들과 공유할 필요가 없는 Live Template Group도 모두 다 export/import가 되기 때문에, 직접 스크립트를 통해 공유하는 것이 좋을 것 같아 스크립트를 만들었다. 

단, Mac이라는 것, Android Studio라는 가정하에 작성했다.

(사용할 때는, script와 같은 폴더에 templates라는 폴더를 만들어서 그곳에 Live Template 설정 파일들을 넣고 git repo에서 관리하고 있다.)

<script src="https://gist.github.com/Rajin9601/2af5cee40247dbd2a9c7234baa6b28b2.js"></script>

<script src="https://gist.github.com/Rajin9601/9e90976679a461f69361a6a80703db8f.js"></script>

# Live Template, 이런 것까지 할 수 있다!

{% include video-fig.html src="/image/live_template/logf" caption="클래스, 함수 이름, parameter까지 로깅 하는 Live Template" %}

해당 Live Template를 만드는 방법은 [여기에](https://gist.github.com/Rajin9601/61bb1645cb9bb0c4837ba3f0f24ec1e9) 정리되어 있다.

[^1]: https://www.jetbrains.com/help/idea/sharing-live-templates.html
