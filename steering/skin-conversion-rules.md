---
inclusion: manual
---

# HTML → 그누보드5 스킨 변환 규칙

퍼블리싱한 정적 HTML을 그누보드5 스킨으로 옮기는 기준.
대상: `g5-custom/skin/board/{project}/`, `g5-custom/skin/qa/{project}/`

## 1. 기본 원칙

### 마크업을 지키고 반복 영역만 치환한다

퍼블리싱 산출물의 클래스명과 구조를 **그대로 유지**한다.
그누보드 기본 스킨의 마크업을 가져와 CSS를 맞추는 방향으로 가지 않는다.
반대로, 완성된 마크업에서 반복되는 행/카드만 PHP 루프로 바꾼다.

```html
<!-- 퍼블리싱 원본 -->
<ul class="notice_list">
    <li class="notice_item">
        <span class="notice_num">1</span>
        <a href="#" class="notice_subject">공지 제목입니다</a>
        <span class="notice_date">2026-08-27</span>
    </li>
    <li class="notice_item">...</li>   <!-- 더미 반복 -->
</ul>
```

```php
<!-- 변환 결과: li 하나만 남기고 루프로 감싼다 -->
<ul class="notice_list">
<?php for ($i = 0; $i < count($list); $i++) { ?>
    <li class="notice_item">
        <span class="notice_num"><?php echo $list[$i]['num']; ?></span>
        <a href="<?php echo $list[$i]['href']; ?>" class="notice_subject"><?php echo $list[$i]['subject']; ?></a>
        <span class="notice_date"><?php echo substr($list[$i]['datetime'], 0, 10); ?></span>
    </li>
<?php } ?>
</ul>
```

### 변환 전에 기본 스킨을 먼저 읽는다

변수명은 그누보드 버전에 따라 다르다. 추측하지 않는다.
`skin/board/basic/` 과 `skin/qa/basic/` 을 열어 **실제로 제공되는 변수를 확인한 뒤** 마크업을 교체한다.
이 문서의 예제도 그 확인을 대체하지 않는다.

### 필수 규칙

- 모든 스킨 파일 첫 줄에 직접 접근 차단을 넣는다.
  ```php
  <?php if (!defined('_GNUBOARD_')) exit; ?>
  ```
- PHP 태그는 `<?php` 를 쓴다. `<?` 단축 태그는 호스팅 설정에 따라 동작하지 않는다.
- 닫는 `?>` 는 HTML이 뒤따르지 않는 파일에서는 생략한다. 공백 출력으로 헤더가 깨진다.
- 스킨 안에서 DB에 직접 쿼리하지 않는다. 진입점이 넘겨준 변수를 쓴다.
- 인라인 `style` 과 인라인 `onclick` 을 남기지 않는다. CSS와 JS 파일로 분리한다.

## 2. 스킨 파일 구성

### 게시판 스킨 (`skin/board/{project}/`)

| 파일 | 역할 |
|---|---|
| `list.skin.php` | 목록 |
| `view.skin.php` | 상세 |
| `write.skin.php` | 쓰기 / 수정 폼 |
| `view_comment.skin.php` | 댓글 목록 (사용 시) |
| `write_comment.skin.php` | 댓글 폼 (사용 시) |
| `delete.skin.php` | 삭제 확인 (사용 시) |
| `search.skin.php` | 통합 검색 결과 (사용 시) |

공지 게시판만 쓰는 프로젝트라면 `list` / `view` / `write` 세 개로 충분하다.

### Q&A 스킨 (`skin/qa/{project}/`)

| 파일 | 역할 |
|---|---|
| `list.skin.php` | 문의 목록 |
| `view.skin.php` | 문의 상세 + 답변 |
| `write.skin.php` | 문의 작성 폼 |

## 3. 출력 이스케이프 규칙

어떤 변수를 그대로 출력해도 되는지 구분한다. 잘못 판단하면 XSS 또는 이중 이스케이프가 생긴다.

| 대상 | 처리 |
|---|---|
| `$list[$i]['subject']` | 진입점에서 이미 가공됨. 그대로 출력 |
| `$view['subject']`, `$view['content']` | 이미 가공됨. 그대로 출력 |
| `$board['bo_subject']` | `get_text()` 적용 |
| DB에서 직접 읽은 값 (`qa_name`, `qa_hp` 등) | `get_text()` 적용 |
| 사용자 입력 본문을 직접 출력할 때 | `nl2br(get_text($v))` |
| URL 파라미터로 들어온 값 | `get_text()` 적용 후 출력 |

```php
<!-- 가공된 값: 그대로 -->
<h2 class="board_title"><?php echo $view['subject']; ?></h2>

<!-- 원시 값: 이스케이프 -->
<span class="qa_name"><?php echo get_text($qa['qa_name']); ?></span>
```

## 4. 공지 목록 — `list.skin.php`

### 제공되는 주요 변수

| 변수 | 설명 |
|---|---|
| `$list` | 게시글 배열 |
| `$total_count` | 전체 글 수 |
| `$page` | 현재 페이지 |
| `$write_pages` | 페이징 HTML. 그대로 출력 |
| `$write_href` | 글쓰기 URL |
| `$list_href` | 목록 URL |
| `$is_checkbox` | 관리 체크박스 노출 여부 |
| `$notice_count` | 공지글 수 |

`$list[$i]` 주요 키: `num`, `wr_id`, `href`, `subject`, `name`, `datetime`, `wr_hit`,
`is_notice`, `icon_new`, `icon_file`, `comment_cnt`

### 변환 예제

```php
<?php
if (!defined('_GNUBOARD_')) exit;
?>
<div class="board_wrap notice_wrap">

    <div class="board_top">
        <p class="board_total">전체 <strong><?php echo number_format($total_count); ?></strong>건</p>

        <form name="fsearch" method="get" class="board_search">
            <input type="hidden" name="bo_table" value="<?php echo $bo_table; ?>">
            <label for="stx" class="blind">검색어</label>
            <input type="text" name="stx" id="stx" value="<?php echo isset($stx) ? get_text($stx) : ''; ?>"
                   class="search_input" placeholder="검색어를 입력하세요">
            <button type="submit" class="search_btn">검색</button>
        </form>
    </div>

    <ul class="notice_list">
    <?php if (count($list) === 0) { ?>
        <li class="notice_empty">등록된 게시물이 없습니다.</li>
    <?php } ?>

    <?php for ($i = 0; $i < count($list); $i++) { ?>
        <li class="notice_item<?php echo ($list[$i]['is_notice'] ? ' is_notice' : ''); ?>">
            <span class="notice_num">
                <?php echo ($list[$i]['is_notice'] ? '공지' : $list[$i]['num']); ?>
            </span>

            <div class="notice_body">
                <a href="<?php echo $list[$i]['href']; ?>" class="notice_subject">
                    <?php echo $list[$i]['subject']; ?>
                    <?php if ($list[$i]['comment_cnt']) { ?>
                        <span class="notice_comment"><?php echo $list[$i]['comment_cnt']; ?></span>
                    <?php } ?>
                    <?php echo $list[$i]['icon_new']; ?>
                    <?php echo $list[$i]['icon_file']; ?>
                </a>
            </div>

            <span class="notice_date"><?php echo substr($list[$i]['datetime'], 0, 10); ?></span>
            <span class="notice_hit"><?php echo number_format($list[$i]['wr_hit']); ?></span>
        </li>
    <?php } ?>
    </ul>

    <div class="board_paging"><?php echo $write_pages; ?></div>

    <?php if ($write_href) { ?>
    <div class="board_btn_area">
        <a href="<?php echo $write_href; ?>" class="btn_board btn_write">글쓰기</a>
    </div>
    <?php } ?>

</div>
```

### 주의점

- `<table>` 대신 `<ul>` 을 쓰면 반응형에서 유리하다. 모바일에서 열을 숨기거나 재배치하기 쉽다.
- 목록이 0건일 때의 마크업을 반드시 넣는다. 퍼블리싱 단계에서 빠지기 쉬운 부분이다.
- `$write_pages` 는 그누보드가 생성한 HTML이다. 페이징 디자인은 CSS로 맞춘다.
  마크업을 바꾸려면 `get_paging()` 을 대체하는 별도 함수를 만들어 쓴다.
- 검색 폼에 `bo_table` hidden 값을 빠뜨리면 검색 시 게시판을 잃는다.
- 공지글은 `is_notice` 로 구분해 클래스를 붙인다. 번호 자리에 "공지"를 출력한다.

## 5. 공지 상세 — `view.skin.php`

### 제공되는 주요 변수

| 변수 | 설명 |
|---|---|
| `$view` | 게시글 정보 배열 |
| `$view['subject']` / `$view['content']` | 가공 완료. 그대로 출력 |
| `$view['file']` | 첨부파일 배열 |
| `$view['link']` | 링크 배열 |
| `$prev_href` / `$next_href` | 이전글 / 다음글 URL |
| `$list_href` | 목록 URL |
| `$update_href` / `$delete_href` | 수정 / 삭제 URL (권한 있을 때만 값이 있음) |

### 변환 예제

```php
<?php
if (!defined('_GNUBOARD_')) exit;
?>
<div class="board_wrap notice_view">

    <div class="view_head">
        <h2 class="view_subject"><?php echo $view['subject']; ?></h2>
        <ul class="view_meta">
            <li class="view_date"><?php echo substr($view['wr_datetime'], 0, 10); ?></li>
            <li class="view_hit">조회 <?php echo number_format($view['wr_hit']); ?></li>
        </ul>
    </div>

    <?php if (isset($view['file']['count']) && $view['file']['count']) { ?>
    <ul class="view_file">
        <?php for ($i = 0; $i <= (int)$view['file'][count($view['file'])]; $i++) {
            if (empty($view['file'][$i]['source'])) continue; ?>
        <li>
            <a href="<?php echo $view['file'][$i]['href']; ?>" class="file_link">
                <?php echo $view['file'][$i]['source']; ?>
                <span class="file_size">(<?php echo $view['file'][$i]['size']; ?>)</span>
            </a>
        </li>
        <?php } ?>
    </ul>
    <?php } ?>

    <div class="view_content">
        <?php echo $view['content']; ?>
    </div>

    <ul class="view_nav">
        <?php if ($prev_href) { ?>
        <li class="nav_prev"><a href="<?php echo $prev_href; ?>">이전글</a></li>
        <?php } ?>
        <?php if ($next_href) { ?>
        <li class="nav_next"><a href="<?php echo $next_href; ?>">다음글</a></li>
        <?php } ?>
    </ul>

    <div class="board_btn_area">
        <a href="<?php echo $list_href; ?>" class="btn_board btn_list">목록</a>
        <?php if ($update_href) { ?>
        <a href="<?php echo $update_href; ?>" class="btn_board btn_update">수정</a>
        <?php } ?>
        <?php if ($delete_href) { ?>
        <a href="<?php echo $delete_href; ?>" class="btn_board btn_delete"
           onclick="return confirm('이 게시물을 삭제하시겠습니까?');">삭제</a>
        <?php } ?>
    </div>

</div>
```

### 주의점

- `$view['content']` 는 에디터가 만든 HTML이다. `htmlspecialchars()` 를 적용하면 태그가 그대로 보인다. 적용하지 않는다.
- 본문 영역 CSS는 `.view_content` 하위에 별도로 정의한다.
  에디터 산출물에는 `reset.css` 가 지워버린 기본 스타일(`ul` 불릿, `p` 여백)이 필요하다.
  ```css
  .view_content ul { list-style: disc; padding-left: 20px; }
  .view_content ol { list-style: decimal; padding-left: 20px; }
  .view_content p  { margin-bottom: 10px; }
  .view_content img { max-width: 100%; height: auto; }
  ```
- 본문 이미지에 `max-width: 100%` 를 반드시 넣는다. 넣지 않으면 모바일에서 레이아웃이 깨진다.
- 첨부파일 배열 순회는 그누보드 버전마다 구조가 다르다. 기본 스킨의 순회 방식을 그대로 가져온다.
- `$update_href` / `$delete_href` 는 권한이 없으면 빈 값이다. 값 존재 여부로 버튼을 조건 출력한다.
  **버튼을 CSS로 숨기는 것은 권한 제어가 아니다.**

## 6. Q&A 문의 작성 — `write.skin.php`

비회원 문의를 허용하는 폼이다. 작성자 정보를 직접 받고 캡차와 동의 체크를 반드시 포함한다.

### 제공되는 주요 변수

| 변수 | 설명 |
|---|---|
| `$w` | 빈 값이면 신규, `'u'` 면 수정 |
| `$qa_id` | 수정 시 대상 ID |
| `$qa` | 수정 시 기존 데이터 배열 |
| `$qa_config` | Q&A 설정 |
| `$is_member` / `$member` | 로그인 여부 / 회원 정보 |

### 변환 예제

```php
<?php
if (!defined('_GNUBOARD_')) exit;

// 로그인 회원이면 기본값을 채워준다. 비회원이면 빈 값.
$def_name  = $is_member ? $member['mb_name']  : '';
$def_email = $is_member ? $member['mb_email'] : '';
$def_hp    = $is_member ? $member['mb_hp']    : '';
?>
<div class="qa_wrap qa_write">

    <form name="fwrite" id="fwrite" method="post" action="<?php echo G5_BBS_URL; ?>/qawrite_update.php"
          enctype="multipart/form-data" onsubmit="return fwrite_submit(this);">
        <input type="hidden" name="token" value="">
        <input type="hidden" name="w" value="<?php echo $w; ?>">
        <input type="hidden" name="qa_id" value="<?php echo $qa_id; ?>">
        <input type="hidden" name="qa_html" value="0">

        <div class="form_row">
            <label for="qa_name" class="form_label">이름 <span class="req">*</span></label>
            <input type="text" name="qa_name" id="qa_name" required
                   value="<?php echo isset($qa['qa_name']) ? get_text($qa['qa_name']) : $def_name; ?>"
                   class="form_input" maxlength="20">
        </div>

        <div class="form_row">
            <label for="qa_hp" class="form_label">연락처 <span class="req">*</span></label>
            <input type="tel" name="qa_hp" id="qa_hp" required
                   value="<?php echo isset($qa['qa_hp']) ? get_text($qa['qa_hp']) : $def_hp; ?>"
                   class="form_input" maxlength="20" placeholder="010-0000-0000">
        </div>

        <div class="form_row">
            <label for="qa_email" class="form_label">이메일</label>
            <input type="email" name="qa_email" id="qa_email"
                   value="<?php echo isset($qa['qa_email']) ? get_text($qa['qa_email']) : $def_email; ?>"
                   class="form_input" maxlength="100">
        </div>

        <?php if (!$is_member) { ?>
        <div class="form_row">
            <label for="qa_password" class="form_label">비밀번호 <span class="req">*</span></label>
            <input type="password" name="qa_password" id="qa_password" <?php echo ($w === '' ? 'required' : ''); ?>
                   class="form_input" maxlength="20">
            <p class="form_help">문의 내용 확인 시 사용합니다.</p>
        </div>
        <?php } ?>

        <div class="form_row">
            <label for="qa_subject" class="form_label">제목 <span class="req">*</span></label>
            <input type="text" name="qa_subject" id="qa_subject" required
                   value="<?php echo isset($qa['qa_subject']) ? get_text($qa['qa_subject']) : ''; ?>"
                   class="form_input" maxlength="255">
        </div>

        <div class="form_row">
            <label for="qa_content" class="form_label">문의 내용 <span class="req">*</span></label>
            <textarea name="qa_content" id="qa_content" required class="form_textarea" rows="10"><?php echo isset($qa['qa_content']) ? get_text($qa['qa_content']) : ''; ?></textarea>
        </div>

        <div class="form_row form_agree">
            <input type="checkbox" name="agree_privacy" id="agree_privacy" value="1">
            <label for="agree_privacy">개인정보 수집·이용에 동의합니다. <span class="req">*</span></label>
            <div class="agree_box">
                수집항목: 이름, 연락처, 이메일 / 목적: 문의 접수 및 답변 / 보유기간: 문의 처리 완료 후 3년
            </div>
        </div>

        <div class="form_row form_captcha">
            <?php echo captcha_html(); ?>
        </div>

        <div class="board_btn_area">
            <button type="submit" class="btn_board btn_submit">문의 등록</button>
            <a href="<?php echo G5_BBS_URL; ?>/qalist.php" class="btn_board btn_cancel">취소</a>
        </div>
    </form>

</div>
```

### 검증 스크립트

클라이언트 검증은 편의 기능일 뿐이다. **서버 검증을 반드시 함께 둔다.**

```javascript
function fwrite_submit(f) {
    if (!f.agree_privacy.checked) {
        alert('개인정보 수집·이용에 동의해 주세요.');
        f.agree_privacy.focus();
        return false;
    }
    if (typeof chk_captcha === 'function' && !chk_captcha()) {
        return false;
    }
    return true;
}
```

### 주의점

- `qa_html` 은 `0` 으로 고정한다. 비회원 입력에 HTML을 허용하지 않는다.
- `captcha_html()` 을 넣었다고 검증이 되는 것이 아니다.
  저장 처리에서 `chk_captcha()` 를 호출해야 실제로 막힌다.
- `token` hidden 값은 그누보드 스크립트가 채운다. 필드 자체를 빠뜨리면 저장이 거부된다.
- 동의 체크박스는 서버에서도 검증한다. JS는 우회 가능하다.
- 수정(`$w === 'u'`) 일 때 비밀번호를 필수로 만들면 수정이 막힌다. 신규일 때만 `required` 를 붙인다.
- `enctype="multipart/form-data"` 는 첨부파일을 받을 때만 의미가 있다. 첨부가 없으면 생략해도 된다.

## 7. Q&A 목록과 상세

### 목록 — `list.skin.php`

상태 배지를 함께 노출한다. 라벨은 `g5-custom/inc/config.php` 의 배열을 쓴다.

```php
<?php
if (!defined('_GNUBOARD_')) exit;
?>
<div class="qa_wrap qa_list">

    <ul class="qa_items">
    <?php if (count($list) === 0) { ?>
        <li class="qa_empty">등록된 문의가 없습니다.</li>
    <?php } ?>

    <?php for ($i = 0; $i < count($list); $i++) { ?>
        <li class="qa_item">
            <span class="qa_status <?php echo $qa_status_class[(int)$list[$i]['qa_status']]; ?>">
                <?php echo get_qa_status_label($list[$i]['qa_status']); ?>
            </span>
            <a href="<?php echo $list[$i]['href']; ?>" class="qa_subject">
                <?php echo get_text($list[$i]['qa_subject']); ?>
            </a>
            <span class="qa_name"><?php echo get_text($list[$i]['qa_name']); ?></span>
            <span class="qa_date"><?php echo substr($list[$i]['qa_datetime'], 0, 10); ?></span>
        </li>
    <?php } ?>
    </ul>

    <div class="board_paging"><?php echo $write_pages; ?></div>

    <div class="board_btn_area">
        <a href="<?php echo G5_BBS_URL; ?>/qawrite.php" class="btn_board btn_write">문의하기</a>
    </div>

</div>
```

### 상세 — `view.skin.php`

문의와 답변을 한 화면에 나눠 보여준다.
답변은 `qa_parent` 가 원본 `qa_id` 인 레코드다. 진입점이 넘겨주는 변수명을 기본 스킨에서 확인한다.

```php
<?php
if (!defined('_GNUBOARD_')) exit;
?>
<div class="qa_wrap qa_view">

    <div class="qa_view_head">
        <span class="qa_status <?php echo $qa_status_class[(int)$qa['qa_status']]; ?>">
            <?php echo get_qa_status_label($qa['qa_status']); ?>
        </span>
        <h2 class="qa_view_subject"><?php echo get_text($qa['qa_subject']); ?></h2>
        <ul class="qa_view_meta">
            <li><?php echo get_text($qa['qa_name']); ?></li>
            <li><?php echo substr($qa['qa_datetime'], 0, 16); ?></li>
        </ul>
    </div>

    <div class="qa_view_content">
        <?php echo nl2br(get_text($qa['qa_content'])); ?>
    </div>

    <?php if (!empty($answer)) { ?>
    <div class="qa_answer">
        <h3 class="qa_answer_title">답변</h3>
        <div class="qa_answer_content">
            <?php echo nl2br(get_text($answer['qa_content'])); ?>
        </div>
        <p class="qa_answer_date"><?php echo substr($answer['qa_datetime'], 0, 16); ?></p>
    </div>
    <?php } else { ?>
    <p class="qa_answer_none">아직 답변이 등록되지 않았습니다.</p>
    <?php } ?>

    <div class="board_btn_area">
        <a href="<?php echo G5_BBS_URL; ?>/qalist.php" class="btn_board btn_list">목록</a>
    </div>

</div>
```

### 주의점

- 문의 본문은 `qa_html = 0` 이므로 평문이다. `nl2br(get_text())` 로 출력한다.
  `get_text()` 없이 출력하면 XSS가 된다.
- 목록에서 연락처를 노출하지 않는다. 비회원 목록은 누구나 볼 수 있다.
  이름도 마스킹이 필요한지 프로젝트 초반에 정한다.
- 비회원 문의 상세 접근은 비밀번호 확인을 거치게 한다. `qa_id` 만으로 열리면 개인정보가 노출된다.
- 상태 배지는 `qa_status` 값 4가지 모두에 대해 CSS 클래스를 정의한다.
  값이 배열에 없을 때를 대비해 `get_qa_status_label()` 이 기본값을 반환하게 해 둔다.

## 8. 변환 완료 체크리스트

- 모든 스킨 파일에 `if (!defined('_GNUBOARD_')) exit;` 가 있는가
- 단축 PHP 태그(`<?`)를 쓰지 않았는가
- 목록 0건 상태의 마크업이 있는가
- 원시 DB 값에 `get_text()` 를 적용했는가
- 가공된 값(`$view['content']`)에 이스케이프를 중복 적용하지 않았는가
- 권한이 필요한 버튼을 CSS가 아니라 조건 출력으로 처리했는가
- 문의 폼에 캡차, 동의 체크, 서버 검증이 모두 있는가
- 본문 영역 이미지에 `max-width: 100%` 가 적용됐는가
- PC / 태블릿 / 모바일 3개 구간에서 목록과 상세가 깨지지 않는가
