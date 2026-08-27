---
inclusion: manual
---

# HTML / CSS / JS 퍼블리싱 표준

그누보드5 기업 사이트 퍼블리싱 기준.
빌드 도구가 없는 공유호스팅 환경이므로 **브라우저가 바로 읽는 코드**를 직접 작성한다.
Sass, TypeScript, 번들러를 쓰지 않는다.

## 1. 파일 구조와 로드 순서

```
g5-custom/
├── css/
│   ├── reset.css       # 브라우저 기본 스타일 초기화
│   ├── page.css        # 공통 + 메인 페이지
│   ├── sub.css         # 서브 페이지 공통 및 개별
│   ├── board.css       # 게시판 / Q&A
│   └── animation.css   # 스크롤 애니메이션 상태 클래스
├── js/
│   ├── page_script.js  # 공통 + 메인 (헤더, 메뉴, 메인 슬라이더)
│   ├── sub_script.js   # 서브 페이지 (탭, 아코디언, 폼)
│   └── animation.js    # 스크롤 감지 및 클래스 토글
└── img/
```

CSS 로드 순서는 고정이다. 뒤에 오는 파일이 앞을 덮어쓰는 구조에 의존한다.

```html
<link rel="stylesheet" href="<?php echo PRJ_CSS; ?>/reset.css">
<link rel="stylesheet" href="<?php echo PRJ_CSS; ?>/page.css">
<link rel="stylesheet" href="<?php echo PRJ_CSS; ?>/sub.css">
<link rel="stylesheet" href="<?php echo PRJ_CSS; ?>/board.css">
<link rel="stylesheet" href="<?php echo PRJ_CSS; ?>/animation.css">
```

JS는 `</body>` 직전에 둔다. jQuery가 항상 먼저다.

```html
<script src="https://code.jquery.com/jquery-3.5.1.min.js"></script>
<script src="https://cdn.jsdelivr.net/npm/swiper@11/swiper-bundle.min.js"></script>
<script src="<?php echo PRJ_JS; ?>/page_script.js"></script>
<script src="<?php echo PRJ_JS; ?>/sub_script.js"></script>
<script src="<?php echo PRJ_JS; ?>/animation.js"></script>
```

- 경로는 하드코딩하지 않고 `PRJ_CSS` / `PRJ_JS` / `PRJ_IMG` 상수를 쓴다.
- 그누보드가 자체적으로 jQuery를 로드하는 페이지가 있다. 중복 로드 시 플러그인이 깨지므로 확인한다.
- 파일을 임의로 추가하지 않는다. 위 5개 CSS, 3개 JS 구성을 유지한다.

## 2. 외부 리소스

| 리소스 | 사용 방식 |
|---|---|
| Pretendard | CDN (가변 폰트, 동적 서브셋) |
| Swiper | CDN v11 |
| jQuery | CDN 3.5.1 |

```html
<link rel="stylesheet" as="style" crossorigin
      href="https://cdn.jsdelivr.net/gh/orioncactus/pretendard@v1.3.9/dist/web/variable/pretendardvariable-dynamic-subset.min.css">
<link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/swiper@11/swiper-bundle.min.css">
```

```css
body {
    font-family: 'Pretendard Variable', Pretendard, -apple-system,
                 'Malgun Gothic', 'Apple SD Gothic Neo', sans-serif;
}
```

- CDN 버전을 고정한다. `@latest` 를 쓰지 않는다. 예고 없이 바뀌면 운영 중 사이트가 깨진다.
- 폰트가 로드되기 전 텍스트가 사라지지 않도록 시스템 폰트를 폴백에 반드시 넣는다.
- 관공서 납품 등 외부 통신이 제한되는 환경이면 폰트와 라이브러리를 로컬에 업로드한다.

## 3. HTML 작성 규칙

```html
<!DOCTYPE html>
<html lang="ko">
<head>
    <meta charset="utf-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>페이지명 | 회사명</title>
    <meta name="description" content="페이지 설명">
</head>
```

- 문서 구조는 시맨틱 태그로 잡는다. `header`, `nav`, `main`, `section`, `footer`
- 모든 `img` 에 `alt` 를 넣는다. 장식용 이미지는 `alt=""` 로 비운다.
- 아이콘만 있는 버튼에는 텍스트 대체를 제공한다.
  ```html
  <button type="button" class="menu_btn">
      <span class="blind">전체메뉴 열기</span>
  </button>
  ```
  ```css
  .blind {
      position: absolute; width: 1px; height: 1px;
      margin: -1px; padding: 0; overflow: hidden;
      clip: rect(0, 0, 0, 0); border: 0;
  }
  ```
- 클릭 동작은 `a` 가 아니라 `button` 을 쓴다. 이동이 아니면 링크가 아니다.
- 인라인 `style`, 인라인 `onclick` 을 남기지 않는다.
- `form` 요소에는 `label` 을 연결한다. `for` 와 `id` 가 짝을 이뤄야 한다.

## 4. 네이밍 규칙

**snake_case 를 쓴다. BEM 을 쓰지 않는다.**

```css
/* 권장 */
.main_visual { }
.main_visual_title { }
.notice_list { }
.notice_item { }
.qa_status { }
.btn_submit { }

/* 지양 - BEM */
.main-visual__title { }
.notice-list--active { }

/* 지양 - camelCase, kebab-case 혼용 */
.mainVisual { }
.main-visual { }
```

### 접두사 규칙

영역을 접두사로 구분한다. 계층은 밑줄로 이어 붙인다.

| 접두사 | 용도 | 예 |
|---|---|---|
| `main_` | 메인 페이지 영역 | `main_visual`, `main_about` |
| `sub_` | 서브 페이지 공통 | `sub_visual`, `sub_title` |
| `board_` | 게시판 공통 | `board_wrap`, `board_paging` |
| `qa_` | Q&A | `qa_list`, `qa_status` |
| `btn_` | 버튼 | `btn_submit`, `btn_more` |
| `form_` | 폼 요소 | `form_input`, `form_label` |

### 상태 클래스

상태는 `is_` 접두사로 통일한다. JS가 붙이고 떼는 클래스임을 이름으로 구분한다.

```css
.is_active { }    /* 현재 선택/활성 */
.is_open { }      /* 열림 */
.is_fixed { }     /* 고정 */
.is_show { }      /* 노출 */
.is_hide { }      /* 숨김 */
```

```javascript
$('.tab_btn').on('click', function () {
    $('.tab_btn').removeClass('is_active');
    $(this).addClass('is_active');
});
```

### 파일과 이미지

- 파일명, 이미지명, `id`, `name` 속성 모두 snake_case.
- 이미지는 용도를 앞에 둔다. `main_visual_01.jpg`, `sub_about_bg.png`, `ico_arrow_right.svg`
- **소문자만 쓴다.** 서버가 Linux이므로 대소문자를 구분한다.
  로컬에서 되고 서버에서 404가 나는 가장 흔한 원인이다.

## 5. 반응형

PC를 기준으로 만들고 아래로 좁혀 내려간다 (desktop first).

| 구간 | 범위 | 비고 |
|---|---|---|
| PC | `1200px` 이상 | 기준 레이아웃 |
| 태블릿 | `768px ~ 1199px` | 2단 축소, 메뉴 축약 |
| 모바일 | `767px` 이하 | 1단, 햄버거 메뉴 |

```css
/* 기본: PC */
.inner { width: 1200px; margin: 0 auto; }

/* 태블릿 */
@media all and (max-width: 1199px) {
    .inner { width: 100%; padding: 0 30px; }
}

/* 모바일 */
@media all and (max-width: 767px) {
    .inner { padding: 0 16px; }
}
```

### 규칙

- 미디어쿼리는 `max-width` 로 통일한다. `min-width` 와 섞지 않는다.
- 브레이크포인트는 위 3개만 쓴다. 임의로 추가하지 않는다.
- 미디어쿼리는 각 CSS 파일 **하단에 모아** 작성한다. 선택자마다 흩어 놓지 않는다.
- 고정 너비 대신 `max-width` 와 `%` 를 쓴다.
- 이미지에는 전역으로 대응을 걸어둔다.
  ```css
  img { max-width: 100%; height: auto; }
  ```
- 게시판 목록은 모바일에서 열을 숨긴다. 조회수, 작성자를 먼저 정리한다.
  ```css
  @media all and (max-width: 767px) {
      .notice_hit { display: none; }
  }
  ```
- 터치 대상은 최소 44px 확보한다.
- 실제 기기에서 확인한다. 브라우저 축소만으로는 모바일 주소창 높이 변화나 터치 동작을 못 잡는다.

## 6. CSS 작성 규칙

```css
/* ========================================
   서브 비주얼
   ======================================== */
.sub_visual {
    position: relative;
    height: 400px;
    background: url('../img/sub_visual_bg.jpg') no-repeat center / cover;
}
.sub_visual_title {
    font-size: 42px;
    font-weight: 700;
    color: #fff;
}
```

- 섹션 구분 주석을 넣는다. 파일이 길어져도 찾을 수 있어야 한다.
- 선택자 깊이는 3단계까지. 그 이상은 클래스를 새로 만든다.
- `!important` 를 쓰지 않는다. 그누보드 기본 CSS를 덮어야 할 때만 예외로 허용하고 주석으로 이유를 남긴다.
- `id` 선택자로 스타일을 주지 않는다. 클래스를 쓴다.
- 색상, 폰트 크기 등 반복 값은 `:root` 변수로 뽑는다.
  ```css
  :root {
      --color_primary: #0b57d0;
      --color_text: #222;
      --color_border: #e5e5e5;
  }
  .btn_submit { background: var(--color_primary); }
  ```
- 이미지 경로는 CSS 파일 기준 상대 경로다. `css/` 에서 `img/` 로 가려면 `../img/` 다.

## 7. JS 작성 규칙

파일별 역할을 지킨다.

| 파일 | 담당 |
|---|---|
| `page_script.js` | 헤더 고정, 전체메뉴, 메인 슬라이더, 상단 이동 |
| `sub_script.js` | 탭, 아코디언, 폼 검증, 서브 전용 인터랙션 |
| `animation.js` | 스크롤 감지와 상태 클래스 토글만 |

```javascript
$(function () {

    // 헤더 스크롤 고정
    var $header = $('.header');
    $(window).on('scroll', function () {
        if ($(window).scrollTop() > 100) {
            $header.addClass('is_fixed');
        } else {
            $header.removeClass('is_fixed');
        }
    });

    // 모바일 전체메뉴
    $('.menu_btn').on('click', function () {
        $('body').toggleClass('is_open');
    });

});
```

- 함수와 변수는 snake_case. jQuery 객체 변수는 `$` 를 앞에 붙인다.
- 스타일을 JS로 직접 조작하지 않는다. 클래스만 토글하고 스타일은 CSS에 둔다.
  ```javascript
  // 지양
  $('.box').css('opacity', 1);
  // 권장
  $('.box').addClass('is_show');
  ```
- `scroll` 과 `resize` 핸들러는 반드시 스로틀링한다. 없으면 저사양 기기에서 멈춘다.
  ```javascript
  var scroll_timer = null;
  $(window).on('scroll', function () {
      if (scroll_timer) return;
      scroll_timer = setTimeout(function () {
          // 처리
          scroll_timer = null;
      }, 100);
  });
  ```
- 선택자를 반복 호출하지 않고 변수에 담는다.
- 요소가 없는 페이지에서도 에러가 나지 않게 존재 확인을 넣는다.
  ```javascript
  if ($('.main_slider').length) {
      new Swiper('.main_slider', { loop: true });
  }
  ```
- `console.log` 를 남기지 않는다.

## 8. 애니메이션 규칙

스크롤 진입 시 요소를 나타내는 방식으로 통일한다.
CSS가 상태를 정의하고, JS는 클래스만 붙인다.

### 마크업

애니메이션 대상에 `ani` 클래스를 붙인다. 방향과 지연은 추가 클래스로 지정한다.

```html
<div class="main_about_text ani ani_up">...</div>
<div class="main_about_img ani ani_left ani_delay_2">...</div>
```

### `animation.css`

```css
/* 초기 상태 */
.ani {
    opacity: 0;
    transition: opacity 0.8s ease, transform 0.8s ease;
}
.ani_up    { transform: translateY(40px); }
.ani_left  { transform: translateX(-40px); }
.ani_right { transform: translateX(40px); }
.ani_zoom  { transform: scale(0.92); }

/* 진입 후 */
.ani.is_show {
    opacity: 1;
    transform: none;
}

/* 순차 지연 */
.ani_delay_1 { transition-delay: 0.1s; }
.ani_delay_2 { transition-delay: 0.2s; }
.ani_delay_3 { transition-delay: 0.3s; }
.ani_delay_4 { transition-delay: 0.4s; }
```

### `animation.js`

`IntersectionObserver` 를 쓴다. 스크롤 이벤트로 매 프레임 계산하지 않는다.

```javascript
$(function () {

    var targets = document.querySelectorAll('.ani');
    if (!targets.length) return;

    // 미지원 브라우저는 즉시 노출한다. 콘텐츠가 안 보이는 것이 최악이다.
    if (!('IntersectionObserver' in window)) {
        $('.ani').addClass('is_show');
        return;
    }

    var observer = new IntersectionObserver(function (entries) {
        entries.forEach(function (entry) {
            if (!entry.isIntersecting) return;
            entry.target.classList.add('is_show');
            observer.unobserve(entry.target);   // 1회만 실행
        });
    }, {
        threshold: 0,
        rootMargin: '0px 0px -12% 0px'
    });

    targets.forEach(function (el) {
        observer.observe(el);
    });

});
```

### 규칙

- 애니메이션은 `opacity` 와 `transform` 만 쓴다.
  `width`, `height`, `top`, `left` 는 레이아웃을 다시 계산해 버벅인다.
- 지속시간은 `0.6s ~ 0.8s`. 그보다 길면 스크롤을 방해한다.
- 지연은 4단계까지만 쓴다. 그 이상은 사용자가 기다린다고 느낀다.
- 한 번 나타난 요소는 다시 숨기지 않는다. `unobserve` 로 관찰을 해제한다.
- **모바일에서는 애니메이션을 줄인다.** 좌우 이동은 특히 체감이 나쁘다.
  ```css
  @media all and (max-width: 767px) {
      .ani_left, .ani_right { transform: translateY(30px); }
  }
  ```
- 접근성 설정을 존중한다.
  ```css
  @media (prefers-reduced-motion: reduce) {
      .ani { opacity: 1; transform: none; transition: none; }
  }
  ```
- 첫 화면(first view) 요소에는 스크롤 애니메이션을 걸지 않는다.
  진입 판정 전에는 `opacity: 0` 이라 콘텐츠가 없는 것처럼 보인다.
- 게시판 목록 항목에 애니메이션을 걸지 않는다. 읽는 콘텐츠를 지연시키지 않는다.

## 9. 퍼블리싱 완료 체크리스트

- CSS 5개, JS 3개 구성을 지켰는가
- 클래스명이 snake_case 이고 BEM 이 섞이지 않았는가
- 파일명과 이미지명이 모두 소문자인가
- 브레이크포인트가 1200 / 768 / 767 세 구간만 쓰였는가
- 미디어쿼리가 파일 하단에 모여 있는가
- 모든 `img` 에 `alt` 가 있는가
- 아이콘 버튼에 `.blind` 텍스트가 있는가
- `scroll` / `resize` 핸들러에 스로틀링이 있는가
- `IntersectionObserver` 미지원 시 콘텐츠가 보이는가
- `prefers-reduced-motion` 대응이 있는가
- 첫 화면 요소에 스크롤 애니메이션이 걸려 있지 않은가
- `console.log` 와 인라인 `style` 이 남아 있지 않은가
- CDN 버전이 고정되어 있는가
