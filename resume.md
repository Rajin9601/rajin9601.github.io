---
layout: resume
title: Resume
permalink: /resume/
---

<div id="resume-header">
    <div id="profile">
    </div>
    <div id="info">
        <div id="info-name">Jin-Hyung Kim <sup>Rajin</sup></div>
        {% include icon-github.html username=site.github_username %}
        {% include icon-twitter.html username=site.twitter_username %}
        <a href="mailto:rajin9601@gmail.com">rajin9601@gmail.com</a>
        <div id="status">Mobile Client Developer @ VCNC</div>
    </div>
</div>

## About myself

* I care about code/software quality & architecture and constantly refactor code.
* I love exchanging opinions about challenging engineering problem.
* I like making tools that improves other developers' productivity.
* I enjoy implementing complex animation that adds unique feeling to the application.
* I hate [dark pattern](https://darkpatterns.org/).

## Experiences

### Mobile Client Developer for [타다](https://tadatada.com/) <sup>2018.7 - </sup>

* Android <small>main</small> / iOS
    * Kotlin/Swift, [RIBs](https://github.com/uber/RIBs), RxJava/Swift, [gRPC](https://grpc.io/)
    * Participated in project's architecture design
        * see [blog post](http://engineering.vcnc.co.kr/2019/05/tada-client-development/) and [PPT](https://docs.google.com/presentation/d/e/2PACX-1vRBYACbRdO0rK71Ee-DHxL_TcjLLIpJnpD39S3OUPIupKQKZ_fV4ofq81oMY56yVLalDeTwflH1vkQ2/pub?start=false&loop=false&delayms=10000&slide=id.p) for details
    * App Links
        * [타다 Android](https://play.google.com/store/apps/details?id=kr.co.vcnc.tada), [타다 iOS](https://apps.apple.com/kr/app/%ED%83%80%EB%8B%A4/id1422751774), [타다 드라이버](https://play.google.com/store/apps/details?id=kr.co.vcnc.tada.driver), [핸들모아](https://play.google.com/store/apps/details?id=kr.co.vcnc.handlemoa)
    * Android Tech Lead since 2020.10
* Internal Library Works
    * Created map sdk's wrapper for convenient camera/marker control
    * Created SharedPreference wrapper
        * handles serialization, cache, crypto and Rx getter
    * [~~ButterKt~~](https://www.rajin.me/dev/2018/08/05/ButterKt.html) deprecated. Use ViewBinding.
    * Implemented [ScreenStack for RIBs](https://github.com/uber/RIBs/tree/master/android/libraries/rib-screen-stack-base)
    * Customized RIBs for animation
* Tooling
    * Created protocol buffer compiler to generate Kotlin/Swift/TS file
    * Resource import tool including auto-conversion from svg to vector drawable
    * Intellij plugin for creating boilerplate codes
    * custom lint checks (Android lint & detekt)

### Android Developer for [Between](https://between.us/) <sup>2017.7 - 2021.4</sup>

* Made a prototype for [Snap](https://youtu.be/LHHKSWS7oTg?t=114)
    * Apply filter(shader) in realtime & save it to file
* Developed voice chat <small>WebRTC, gRPC</small>
    * Participated in protocol design

### Internship at 42 Company <sup>2016.12 - 2017.1</sup>

* Developed simple contents crawler for Slide

### Internship at Ultracaption <sup>2015.07 - 08</sup>

* Developed iOS Application PLAIN

## Education

* Seoul National University, Majoring in Computer Science and Engineering <sup>2014 - 2021</sup>
    * Vice President of [WaffleStudio](https://www.facebook.com/wafflestudio/) <sup>2015 - 2016</sup>
    * 17th Place in ACM-ICPC Asia Daejeon as a team <sup>2015</sup>
    * Presidential Science Scholarship <sup>2014</sup>
* Gyeong-gi Science High School <sup>2011 - 2014</sup>
    * Gold in Korean Olympiad of Informatics, National <sup>2013</sup>

## Side-Projects

* [SNUTT2](https://snutt.kr/) 2015 - 2021.02
    * iOS Developer
    * [Link to AppStore](https://itunes.apple.com/kr/app/snutt-서울대학교-시간표-앱/id1215668309?mt=8), [Link to Github](https://github.com/wafflestudio/SNUTT-iOS)
* [21 Days](http://store.steampowered.com/app/607660/21_Days/)
    * Main Programmer
        * Created dialog system & parser

<div id="update-date">updated at 2021.10.02</div>
