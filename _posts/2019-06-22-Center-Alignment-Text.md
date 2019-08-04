---
layout: post
title:  "완벽한 텍스트 세로 중앙정렬을 위하여"
date:   2019-08-04 15:11:00 +0900
categories: dev
item-black: true
comments: true
---

VCNC 에서는 개발이 어느정도 마무리가 되면 기획/UX/개발 팀이 모두 모여 실제 구현물을 가지고 테스트를 해봅니다. 엣지 케이스들에서 어떻게 작동하는지, 실제 해보니 어떤 점이 불편했는지 를 확인하는 자리이지만 UI/UX 팀에서 원하는 대로 구현이 되었는지도 확인을 합니다. 호출 화면 리뉴얼을 할 시점에도 최종 테스트 회의를 하는 과정에서 UI 팀이 화면을 보더니 매의 눈으로 이상한 점을 발견했습니다.

> "선택된 결제카드 없음" 텍스트 높이가 왼쪽 아이콘이랑 가운데 정렬이 아닌거 같아요.

{% include image-fig.html src="/image/center_alignment/confirm_view_prev.png" alt="" caption="유심히 들여다 보면 보이긴 하네요. 아이콘의 중앙에 비해 '선' 글자가 밑에 있는거 같네요" %}

음? 가운데 정렬을 내가 몇 px 를 일부러 옮길리는 없는데... 라는 생각에 코드를 다시 봤지만 너무나도 확실하게 가운데 정렬을 시켰습니다. 실제로 layout editor 로 x, y 좌표를 봐도 정렬이 되어있습니다. 하지만 [Zeplin](https://zeplin.io/)에서 보이는 것과는 다르게 확실히 텍스트가 밑에 있었습니다. 무엇이 문제인지 알기 위해서 먼저 label 자체의 layout size를 확인해보았습니다. 

참고사항: UI/UX 팀이 개발팀에게 디자인을 전달하는 방식은 Sketch 로 디자인, Zeplin 으로 공유하고 있습니다.

# TextView 의 사이즈 로직

iOS 의 경우는 실험적으로 테스트를 하였고, Android 는 코드를 본 결과, TextView(Label) height 로직은 다음과 같습니다. (single line 기준)

* Android : includeFontPadding 이 true 이면 Top ↔ Bottom, false 이면 Ascent ↔ Descent 
* iOS : Ascent ↔ Descent
* Sketch & Zeplin : Ascent ↔ Descent 

{% include image-fig.html src="/image/center_alignment/font-metrics.png" alt="Font Metrics Explanation" caption="Typography y 축 정리. from https://proandroiddev.com/android-and-typography-101-5f06722dd611" %}

즉 디자인 팀이 TextView 의 y 관련 수치를 정확하게 요구를 한다면 Android 에서는 includeFontPadding = false 를 하는게 좋습니다.
하지만 타다의 경우, default 값이 includeFontPadding false 로 하였기 때문에 textview 의 사이즈는 sketch/zeplin 과 똑같았습니다.

Layout 사이즈의 문제가 아니라면 무엇이 문제일지 디자인팀과 이야기를 하다가 Sketch 에서 무언가를 발견하게 됩니다.

# Sketch, Zeplin 그리고 vertical text alignment

{% include image-fig.html src="/image/center_alignment/text-vertical-alignment.gif" alt="Text alignment option in sketch" caption="모든것의 원흉 vertical text alignment. from https://uxcellence.com/2017/ultimate-guide-to-sketch-symbols" %}

디자인 팀에서는 텍스트의 y 축이 중요할 때, Sketch 의 Vertical Text Align 을 쓰고 있었습니다. '선택된 결제카드 없음' 에서도 Vertical Text Align 을 center 로 되어있었지만, **Vertical Text Align 속성은 Zeplin 에서 보이지 않아** 개발팀은 모르고 있었던 것입니다. 게다가 Zeplin 에서 볼 때는 vertical text center 가 적용된 상태로 보이지만, 속성은 보이지 않습니다.

Vertical Text Align = Center 의 의미는 단순해 보이지만 굉장히 많은 조건과 케이스들이 있습니다. Sketch 와 동일하게 보이려면 Sketch 의 Baseline option이 무엇인지를 조사해야 되지만, Sketch 와 동일한 것보단 가운데 정렬이 되어있는 모습을 만들고 싶은것이므로 Spoqa han sans 에서는 Cap Height 로 잡았습니다.

{% include image-fig.html src="/image/center_alignment/spoqa-han-sans-metric.png" alt="Spoqa han sans metric of 결" caption="스포카 한 산스의 결 문자의 메트릭. from: https://fontdrop.info/" %}

Cap height 의 중앙과 한글의 중앙이 비슷하므로 view 의 가운데 정렬을 시키면, 볼때 가운데 정렬처럼 보일 것입니다.

# 구현
## iOS : CenterAlignedLabel

iOS 의 경우, Label 의 drawText method 를 override 하여, 원하는 offset 만큼 y 축을 이동시켜서 그리면 됩니다.
```
override func drawText(in rect: CGRect) {
    let offsetY = .... // 복잡한 로직이니 여기선 생략. gist 를 참고해주세요.
    super.drawText(in: rect.offsetBy(dx: 0, dy: offsetY))
}
```

구현체는 여기서 확인하실수 있습니다 : [CenterAlignedLabel.swift](https://gist.github.com/Rajin9601/803584407f8d4ce9953b6a91c84fdb87)

혹시 구현체에 문제나 질문이 있다면 gist 덧글로 달아주시면 감사하겠습니다.

## Android: CenterAlignedTextView 

Android 의 경우, 구현에는 2가지 어려움이 있었습니다.

### 1. text rendering 위치를 갈아끼우기 힘듬

drawText 처럼 text 렌더링 하는 부분만 갈아끼우기 힘들기 때문에 다른 편법을 써야 됩니다.
맨 처음에는 원하는 만큼 paddingBottom/Top 을 넣어서 하려고 했지만 만약 이 외의 용도로 padding 을 넣고 싶은 경우에는, 항상 center align 에 의한 padding 을 신경써서 코딩해야되기 때문에 귀찮습니다.

따라서 Span 을 사용하여 문제를 해결하였습니다. setText 를 할 때마다, text 전체에 Custom Span 을 걸고, Span 이 원하는 만큼 text rendering 위치를 바꾸게 하였습니다. 현재 쓰는 방법도 완벽하지 않고 엣지 케이스가 있을것이라고 생각되지만 padding 값을 직접 컨트롤 하는것보다는 확장성이 있다고 판단하였습니다.

### 2. Cap height 정보가 없음

Android 에서는 폰트 정보를 가져올수 있는 [Paint.FontMetrix](https://developer.android.com/reference/android/graphics/Paint.FontMetrics) 에 cap height 에 대한 정보를 가져올 수가 없습니다. 따라서 Cap height 대한 정보를 직접 만드는 수 밖에 없습니다.

만약 확장성있도록 모든 Font 에 대해서 작동하게 만들고 싶다면 [Paint.getTextBounds](https://developer.android.com/reference/android/graphics/Paint.html#getTextBounds(java.lang.String,%2520int,%2520int,%2520android.graphics.Rect)) 를 사용하여 Cap height 의 정의인 "X" 의 높이를 가져와서 사용하는 방법이 있습니다. 하지만 cap Height 는 font 정보에 들어있는 값이기 때문에 "X" 의 높이를 계산하더라도 실제 font 의 cap height 와 다를 수 있습니다. 따라서 타다에서는 스포카 한 산스 만 사용하고 있기 때문에 정확한 값을 미리 계산하여 저장해놓고, 만약을 위해 fallback 으로 "X" 높이를 계산하는 로직을 넣었습니다.

구현체는 여기서 확인하실수 있습니다 : [CenterAlignedTextView.kt](https://gist.github.com/Rajin9601/7591e7cfd4dc62742f6f8a8911ae8695)
혹시 구현체에 문제나 질문이 있다면 gist 덧글로 달아주시면 감사하겠습니다.

# TL;DR

{% include image-fig.html src="/image/center_alignment/confirm_view_compare.png" alt="Comparison" caption="위가 적용전, 밑이 적용후. 중앙정렬이 된 것을 볼수 있다." %}

* Sketch 의 vertical text alignment 옵션은 Zeplin 에서 보이지 않지만, 그림 상으로는 적용되어 보인다.
* vertical text alignment = center 는 cap height 을 중앙 정렬하도록 만든다.
* Android
  * TextView 의 includeFontPadding 은 false
  * 구현체 [gist](https://gist.github.com/Rajin9601/7591e7cfd4dc62742f6f8a8911ae8695)
* iOS
  * 구현체 [gist](https://gist.github.com/Rajin9601/803584407f8d4ce9953b6a91c84fdb87)
