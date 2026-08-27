---
inclusion: manual
---

# 그누보드5 기업 사이트 프로젝트 가이드

그누보드5 v5.6.32 기반 기업용 웹사이트 제작 기준 문서.
`{project}` 는 프로젝트 식별자(snake_case)로 치환해서 쓴다.

## 1. 호스팅 환경 전제

국내 공유호스팅을 기준으로 한다. 아래 제약을 항상 가정한다.

| 항목 | 전제 |
|---|---|
| 셸 접근 | 없음 (SSH 불가) |
| Composer / npm | 사용 불가 |
| 빌드 단계 | 없음. 업로드한 파일이 그대로 실행된다 |
| 배포 | FTP 수동 업로드 |
| PHP | 호스팅사 지정 버전. 임의 변경 불가 |
| DB | MySQL, 단일 DB |
| cron | 호스팅사 관리도구로만 등록 가능 |

이 전제에서 파생되는 규칙:

- **Sass/TypeScript/번들러를 쓰지 않는다.** 브라우저가 바로 읽는 CSS/JS를 직접 작성한다.
- 외부 라이브러리는 CDN 또는 파일 직접 업로드로 해결한다.
- `.env` 같은 런타임 환경변수에 의존하지 않는다. 설정은 PHP 상수로 둔다.
- 파일 경로 대소문자를 정확히 맞춘다. 로컬(Windows)에서는 통과하고 서버(Linux)에서 404가 나는 대표 원인이다.
- 롤백 수단이 없다. 덮어쓰기 전 원본 백업이 필수다.

## 2. 디렉터리 구조

그누보드 코어는 건드리지 않고, 프로젝트 산출물은 `g5-custom/` 에 모은다.

```
/ (그누보드 루트)
├── common.php              # 코어 부트스트랩
├── adm/                    # 그누보드 기본 관리자
├── bbs/                    # 게시판 진입점 (board.php, qalist.php ...)
├── data/                   # 업로드/캐시/DB설정. 커밋 금지
├── lib/  plugin/  skin/    # 코어
│
└── g5-custom/              # ★ 프로젝트 전용 영역
    ├── inc/
    │   ├── config.php      # 프로젝트 상수 정의
    │   ├── head.php        # 공통 head + 헤더
    │   └── tail.php        # 공통 푸터
    ├── css/
    │   ├── reset.css
    │   ├── page.css
    │   ├── sub.css
    │   ├── board.css
    │   └── animation.css
    ├── js/
    │   ├── page_script.js
    │   ├── sub_script.js
    │   └── animation.js
    ├── img/
    ├── skin/
    │   ├── board/{project}/    # 게시판 스킨
    │   └── qa/{project}/       # Q&A 스킨
    └── adm/
        ├── qa_list.php         # 상담 목록
        ├── qa_view.php         # 상담 상세 / 답변
        ├── qa_update.php       # 상태 변경 처리
        └── board_manage.php    # 게시판 관리
```

코어를 부득이하게 수정하는 경우는 다음 정도로 제한한다.

- `bbs/qawrite.php` — 비회원 문의 허용을 위한 로그인 체크 비활성화
- `adm/admin.menu.*.php` — 커스텀 관리자 메뉴 등록

수정한 코어 파일은 목록으로 남긴다. 그누보드 업그레이드 시 되돌아가기 때문이다.

## 3. DB 테이블

기본 접두사는 `g5_` 다. 테이블명을 하드코딩하지 말고 항상 `$g5` 배열을 쓴다.
접두사는 설치 시 바뀔 수 있다.

| `$g5` 키 | 테이블 | 용도 |
|---|---|---|
| `config_table` | `g5_config` | 사이트 전역 환경설정 (1행) |
| `member_table` | `g5_member` | 회원 |
| `group_table` | `g5_group` | 게시판 그룹 |
| `board_table` | `g5_board` | 게시판 설정 (스킨, 권한, 제목) |
| `board_new_table` | `g5_board_new` | 최신글 인덱스 |
| `board_file_table` | `g5_board_file` | 게시판 첨부파일 |
| `write_prefix` | `g5_write_` | 게시글 테이블 접두사 |
| `qa_config_table` | `g5_qa_config` | Q&A 전역 설정 (1행) |
| `qa_content_table` | `g5_qa_content` | Q&A 문의/답변 본문 |
| `content_table` | `g5_content` | 내용관리 페이지 |
| `popular_table` | `g5_popular` | 인기 검색어 |

게시글은 게시판마다 별도 테이블이다. `notice` 게시판이면 `g5_write_notice` 다.
스킨 안에서는 `$write_table` 변수로 이미 주어지므로 직접 조립하지 않는다.

```php
// 권장
$sql = " select * from {$write_table} where wr_is_comment = 0 ";

// 지양 — 접두사/게시판명 하드코딩
$sql = " select * from g5_write_notice ";
```

### g5_qa_content 주요 컬럼

| 컬럼 | 설명 |
|---|---|
| `qa_id` | PK |
| `qa_num` | 정렬용 음수 번호 |
| `qa_parent` | 원본 문의 `qa_id` (답변인 경우) |
| `qa_related` | 연결된 문의 |
| `qa_category` | 문의 분류 |
| `qa_subject` / `qa_content` | 제목 / 본문 |
| `qa_name` / `qa_email` / `qa_hp` | 작성자 / 이메일 / 연락처 |
| `qa_password` | 비회원 문의 확인용 비밀번호 |
| `qa_status` | 처리 상태 |
| `qa_1` ~ `qa_5` | 여분 필드 (회사명, 지역 등 프로젝트별 활용) |
| `qa_file1` / `qa_file2` | 첨부 |
| `qa_html` | 본문 HTML 사용 여부 |
| `qa_ip` / `qa_datetime` | 작성 IP / 작성 시각 |
| `mb_id` | 회원 아이디. 비회원 문의는 빈 값 |

> 문의 폼에 항목을 추가할 때 컬럼을 새로 만들기 전에 `qa_1`~`qa_5` 를 먼저 검토한다.
> 코어 스키마를 덜 건드리는 쪽이 업그레이드에 안전하다.

## 4. 그누보드 상수와 전역 변수

### 상수 (경로/URL)

| 상수 | 의미 |
|---|---|
| `G5_PATH` | 그누보드 루트 절대 경로 (파일 include용) |
| `G5_URL` | 그누보드 루트 URL (브라우저 출력용) |
| `G5_BBS_PATH` / `G5_BBS_URL` | `bbs` 디렉터리 |
| `G5_SKIN_PATH` / `G5_SKIN_URL` | `skin` 디렉터리 |
| `G5_ADMIN_PATH` / `G5_ADMIN_URL` | `adm` 디렉터리 |
| `G5_DATA_PATH` / `G5_DATA_URL` | `data` 디렉터리 (업로드) |
| `G5_LIB_PATH` | `lib` 디렉터리 |
| `G5_EXTEND_PATH` | `extend` 디렉터리. 여기 둔 `*.extend.php` 는 자동 로드된다 |
| `G5_THEME_PATH` / `G5_THEME_URL` | 테마 사용 시 |
| `G5_IS_MOBILE` | 모바일 접속 여부 (bool) |
| `G5_TIME_YMDHIS` | 현재 시각 `Y-m-d H:i:s` |
| `G5_TIME_YMD` | 현재 날짜 `Y-m-d` |

**PATH 와 URL 을 혼동하지 않는다.** `include` 에는 `PATH`, `href`/`src` 에는 `URL` 이다.
서버 경로가 HTML에 노출되는 사고 대부분이 여기서 나온다.

### 전역 변수

| 변수 | 설명 |
|---|---|
| `$g5` | 테이블명 배열 |
| `$config` | `g5_config` 1행. 사이트 전역 설정 |
| `$member` | 로그인 회원 정보 배열 |
| `$is_member` | 로그인 여부 |
| `$is_guest` | 비로그인 여부 |
| `$is_admin` | `'super'` / `'group'` / `'board'` / `''`. **빈 문자열이면 관리자 아님** |
| `$board` | 현재 게시판 설정 |
| `$bo_table` | 현재 게시판 테이블명 |
| `$write_table` | 현재 게시글 테이블명 |
| `$qa_config` | Q&A 설정 |

`$is_admin` 은 boolean 이 아니라 문자열이다. 권한 단계를 구분해야 할 때 값을 비교한다.

```php
if ($is_admin !== 'super') {
    alert('최고관리자만 접근할 수 있습니다.', G5_URL);
}
```

### 프로젝트 상수 정의

경로를 매번 조립하지 않도록 `g5-custom/inc/config.php` 에 상수를 둔다.

```php
<?php
if (!defined('_GNUBOARD_')) exit;

define('PRJ_NAME', '{project}');
define('PRJ_PATH', G5_PATH.'/g5-custom');   // include 용
define('PRJ_URL',  G5_URL.'/g5-custom');    // 브라우저 출력용
define('PRJ_CSS',  PRJ_URL.'/css');
define('PRJ_JS',   PRJ_URL.'/js');
define('PRJ_IMG',  PRJ_URL.'/img');
```

`extend/{project}.extend.php` 에서 위 파일을 불러오면 모든 페이지에서 상수를 쓸 수 있다.
`extend/` 디렉터리의 `*.extend.php` 는 `common.php` 가 자동으로 읽는다.

```php
<?php
// extend/{project}.extend.php
if (!defined('_GNUBOARD_')) exit;
include_once(G5_PATH.'/g5-custom/inc/config.php');
```

모든 PHP 파일 최상단에는 직접 접근 차단 코드를 넣는다.

```php
if (!defined('_GNUBOARD_')) exit;
```

## 5. 비회원 Q&A 허용

기업 사이트 문의는 회원가입 없이 받는 것이 일반적이다. 그누보드 기본 Q&A는 회원만 쓸 수 있어 이를 푼다.

`bbs/qawrite.php` 상단의 로그인 체크를 주석 처리한다.
버전에 따라 문구가 다르므로 `$is_guest` 또는 `$member['mb_id']` 를 검사하는 줄을 찾아 처리한다.

```php
// bbs/qawrite.php
// [{project}] 비회원 문의 허용을 위해 로그인 체크 비활성화 (원본 보존)
// if ($is_guest) alert('회원만 이용하실 수 있습니다.', G5_URL);
```

로그인 체크를 풀면 다음이 **필수**가 된다. 하나라도 빠지면 스팸에 그대로 노출된다.

1. **캡차** — `chk_captcha()` 로 검증한다. 선택이 아니다.
2. **작성자 정보 직접 입력** — `qa_name`, `qa_hp`, `qa_email` 을 폼에서 받고 서버에서 필수 검증한다.
3. **개인정보 수집·이용 동의** — 체크박스 필수. 국내 사이트는 법적 요건이다.
4. **비밀번호** — 비회원이 자기 문의를 확인할 수 있도록 `qa_password` 를 받는다.
5. **본문 HTML 차단** — `qa_html` 을 0으로 고정한다. 비회원 입력에 HTML을 허용하지 않는다.

```php
// 스킨 write 폼
<?php echo captcha_html(); ?>

// 저장 처리
if (!chk_captcha()) {
    alert('자동등록방지 숫자가 틀렸습니다.');
}
```

출력 시에는 반드시 이스케이프한다.

```php
<?php echo get_text($qa['qa_name']); ?>
<?php echo nl2br(get_text($qa['qa_content'])); ?>
```

> 비회원 문의는 개인정보(이름/연락처)가 DB에 쌓인다.
> 관리자 화면 권한 검사와 목록에서의 연락처 마스킹 여부를 프로젝트 초반에 정한다.

## 6. qa_status 4단계 확장

그누보드 기본 `qa_status` 는 미답변(0) / 답변완료(1) 2단계다. 상담 운영을 위해 4단계로 넓힌다.

| 값 | 상태 | 의미 |
|---|---|---|
| `0` | 접수중 | 문의가 등록됨. 담당자 확인 전 |
| `1` | 접수완료 | 담당자가 내용을 확인함 |
| `2` | 답변중 | 처리 진행 중 |
| `3` | 답변완료 | 처리 종료 |

컬럼이 값 3까지 담을 수 있는지 확인하고, 좁으면 넓힌다.

```sql
ALTER TABLE g5_qa_content
  MODIFY qa_status TINYINT(4) NOT NULL DEFAULT 0;
```

상태 라벨은 한 곳에서만 정의한다. 화면마다 문자열을 반복하면 반드시 어긋난다.
`g5-custom/inc/config.php` 에 배열로 둔다.

```php
$qa_status_label = array(
    0 => '접수중',
    1 => '접수완료',
    2 => '답변중',
    3 => '답변완료',
);

// 상태값 → CSS 클래스 (색상 구분용)
$qa_status_class = array(
    0 => 'status_wait',
    1 => 'status_recv',
    2 => 'status_ing',
    3 => 'status_done',
);

function get_qa_status_label($status) {
    global $qa_status_label;
    $status = (int)$status;
    return isset($qa_status_label[$status]) ? $qa_status_label[$status] : '알수없음';
}
```

기본 그누보드 코드가 `qa_status = 1` 을 "답변완료"로 간주하는 곳이 있다.
확장 후에는 목록 카운트나 메일 발송 조건이 의도와 달라질 수 있으므로,
상태를 쓰는 지점을 검색해 `qa_status = 3` 기준으로 맞추는지 확인한다.

## 7. 커스텀 관리자

그누보드 기본 관리자는 개발자 기준이라 운영자가 쓰기 어렵다.
상담 처리와 게시판 관리에 필요한 화면만 따로 만든다.

```
g5-custom/adm/
├── _common.php        # 공통 부트스트랩 + 권한 검사
├── qa_list.php        # 상담 목록 (검색/상태 필터/페이징)
├── qa_view.php        # 상담 상세 + 답변 작성
├── qa_update.php      # 상태 변경 / 답변 저장 처리
└── board_manage.php   # 게시판 글 관리 (일괄 삭제/공지 지정)
```

### 공통 부트스트랩

권한 검사를 파일마다 반복하지 않고 한 곳에 모은다.

```php
<?php
// g5-custom/adm/_common.php
include_once(dirname(dirname(dirname(__FILE__))).'/common.php');

// 관리자 외 접근 차단. $is_admin 은 문자열이므로 빈 값 검사로 판단한다.
if (!$is_admin) {
    alert('관리자만 접근할 수 있습니다.', G5_URL);
}

include_once(G5_PATH.'/g5-custom/inc/config.php');
```

각 관리자 파일 첫 줄에서 이것만 불러온다.

```php
<?php
include_once('./_common.php');
```

> 권한 검사를 빠뜨리면 상담 개인정보(이름/연락처/이메일)가 URL만 알면 열린다.
> 신규 관리자 파일을 추가할 때 `_common.php` 를 불러왔는지 반드시 확인한다.

### 상담 목록

상태 필터, 검색, 페이징이 기본이다. 쿼리는 반드시 바인딩 또는 이스케이프한다.

```php
<?php
include_once('./_common.php');

$qa_status = isset($_GET['qa_status']) && $_GET['qa_status'] !== '' ? (int)$_GET['qa_status'] : -1;
$stx       = isset($_GET['stx']) ? trim($_GET['stx']) : '';
$page      = max(1, (int)(isset($_GET['page']) ? $_GET['page'] : 1));
$rows      = 20;
$from      = ($page - 1) * $rows;

$where = array(" qa_parent = 0 ");   // 원본 문의만
if ($qa_status >= 0) {
    $where[] = " qa_status = '".$qa_status."' ";
}
if ($stx !== '') {
    $s = sql_real_escape_string($stx);
    $where[] = " (qa_subject like '%$s%' or qa_name like '%$s%' or qa_hp like '%$s%') ";
}
$sql_where = implode(' and ', $where);

$row  = sql_fetch(" select count(*) as cnt from {$g5['qa_content_table']} where {$sql_where} ");
$total = (int)$row['cnt'];

$result = sql_query("
    select qa_id, qa_subject, qa_name, qa_hp, qa_status, qa_datetime
      from {$g5['qa_content_table']}
     where {$sql_where}
     order by qa_id desc
     limit {$from}, {$rows}
");
?>
```

목록 출력에서 상태 라벨과 클래스는 `config.php` 의 배열을 쓴다.

```php
<?php while ($qa = sql_fetch_array($result)) { ?>
<tr>
    <td><?php echo $qa['qa_id']; ?></td>
    <td><a href="./qa_view.php?qa_id=<?php echo $qa['qa_id']; ?>"><?php echo get_text($qa['qa_subject']); ?></a></td>
    <td><?php echo get_text($qa['qa_name']); ?></td>
    <td><?php echo get_text($qa['qa_hp']); ?></td>
    <td><span class="qa_status <?php echo $qa_status_class[(int)$qa['qa_status']]; ?>"><?php echo get_qa_status_label($qa['qa_status']); ?></span></td>
    <td><?php echo substr($qa['qa_datetime'], 0, 10); ?></td>
</tr>
<?php } ?>
```

### 상태 변경 처리

상태 변경은 POST로만 받고, 토큰으로 위조를 막고, 값 범위를 검증한다.

```php
<?php
// g5-custom/adm/qa_update.php
include_once('./_common.php');
check_admin_token();                       // CSRF 방어

$qa_id     = (int)(isset($_POST['qa_id']) ? $_POST['qa_id'] : 0);
$qa_status = (int)(isset($_POST['qa_status']) ? $_POST['qa_status'] : -1);

if ($qa_id < 1) {
    alert('잘못된 요청입니다.');
}
// 허용 범위 밖의 값을 그대로 저장하지 않는다.
if (!array_key_exists($qa_status, $qa_status_label)) {
    alert('잘못된 상태값입니다.');
}

sql_query(" update {$g5['qa_content_table']}
               set qa_status = '{$qa_status}'
             where qa_id = '{$qa_id}' ");

goto_url('./qa_view.php?qa_id='.$qa_id);
```

폼에는 토큰을 함께 넣는다.

```php
<form method="post" action="./qa_update.php">
    <input type="hidden" name="token" value="<?php echo get_admin_token(); ?>">
    <input type="hidden" name="qa_id" value="<?php echo $qa_id; ?>">
    <select name="qa_status">
        <?php foreach ($qa_status_label as $k => $v) { ?>
        <option value="<?php echo $k; ?>"<?php echo ((int)$qa['qa_status'] === $k ? ' selected' : ''); ?>><?php echo $v; ?></option>
        <?php } ?>
    </select>
    <button type="submit">상태 변경</button>
</form>
```

> `check_admin_token()` / `get_admin_token()` 은 그누보드 버전에 따라 제공 여부가 다르다.
> 없으면 `$token = get_token();` 과 `check_token()` 계열 함수로 대체하고, 반드시 POST + 토큰 검증을 유지한다.

## 8. 진입점 연결

스킨과 커스텀 관리자를 그누보드가 실제로 불러오도록 연결하는 단계다.

### 게시판 스킨 연결

그누보드는 게시판 스킨을 `G5_SKIN_PATH/board/{bo_skin}` 에서 찾는다.
스킨을 `g5-custom/` 에 두었으므로 두 가지 방법이 있다.

**방법 A — 프록시 스킨 (권장)**

`skin/board/{project}/` 에 얇은 로더만 두고 실제 구현은 `g5-custom` 에서 불러온다.
관리자 화면 스킨 선택 목록에 정상적으로 노출되는 것이 장점이다.

```php
<?php
// skin/board/{project}/list.skin.php
if (!defined('_GNUBOARD_')) exit;
include_once(G5_PATH.'/g5-custom/skin/board/{project}/list.skin.php');
```

`view.skin.php`, `write.skin.php` 도 같은 방식으로 만든다.

이때 `$board_skin_url` 은 프록시 폴더를 가리키므로, 스킨 내부 자산 경로는
`$board_skin_url` 대신 프로젝트 상수를 쓴다.

```php
<!-- 지양: 프록시 폴더를 가리켜 자산을 찾지 못한다 -->
<img src="<?php echo $board_skin_url; ?>/img/icon.png">

<!-- 권장 -->
<img src="<?php echo PRJ_URL; ?>/skin/board/{project}/img/icon.png">
```

**방법 B — 상대 경로 스킨명**

`g5_board.bo_skin` 값에 상대 경로를 넣는다.

```sql
UPDATE g5_board
   SET bo_skin        = '../g5-custom/skin/board/{project}',
       bo_mobile_skin = '../g5-custom/skin/board/{project}'
 WHERE bo_table = 'notice';
```

간단하지만 관리자 스킨 선택 목록에 나타나지 않고, 그누보드 버전에 따라 경로 처리가 다를 수 있다.
**적용 후 목록/상세/쓰기 화면이 모두 정상 동작하는지 직접 확인한 뒤 채택한다.**

### Q&A 스킨 연결

Q&A 스킨은 `g5_qa_config` 에서 지정한다. 컬럼명은 버전에 따라 다르므로 실제 스키마를 확인한다.

```sql
UPDATE g5_qa_config
   SET qa_skin        = '{project}',
       qa_mobile_skin = '{project}';
```

프록시 방식이면 `skin/qa/{project}/` 에 로더를 둔다.

```php
<?php
// skin/qa/{project}/list.skin.php
if (!defined('_GNUBOARD_')) exit;
include_once(G5_PATH.'/g5-custom/skin/qa/{project}/list.skin.php');
```

### 프론트에서 게시판 호출

정적 페이지 안에 게시판을 끼워 넣을 때는 `outlogin` 방식이 아니라 링크로 연결한다.

```php
<a href="<?php echo G5_BBS_URL; ?>/board.php?bo_table=notice">공지사항</a>
<a href="<?php echo G5_BBS_URL; ?>/qalist.php">문의하기</a>
<a href="<?php echo G5_BBS_URL; ?>/qawrite.php">문의 작성</a>
```

최신글을 메인에 뿌릴 때는 그누보드 내장 함수를 쓴다.

```php
<?php echo latest('{project}_main', 'notice', 5, 40); ?>
```

첫 인자는 최신글 스킨명이다. `skin/latest/{project}_main/latest.skin.php` 를 만들어 둔다.

### 관리자 메뉴 등록

`adm/admin.menu.*.php` 에 파일을 추가해 커스텀 관리자를 메뉴에 노출한다.
기존 파일의 배열 형식을 먼저 열어보고 **같은 형식에 맞춘다.** 버전에 따라 항목 수가 다르다.

```php
<?php
// adm/admin.menu.900.php
if (!defined('_GNUBOARD_')) exit;

$menu['menu900'] = array(
    array('900000', '{project} 관리', '', ''),
    array('900100', '상담 목록',   G5_URL.'/g5-custom/adm/qa_list.php',      'super'),
    array('900200', '게시판 관리', G5_URL.'/g5-custom/adm/board_manage.php', 'super'),
);
```

메뉴에 노출되는 것과 접근이 차단되는 것은 별개다.
**메뉴를 숨기는 것으로 권한 제어를 대신하지 않는다.** 파일 상단의 `_common.php` 권한 검사가 실제 방어선이다.

### 연결 확인 체크리스트

- 게시판 목록 / 상세 / 쓰기 3개 화면이 커스텀 스킨으로 렌더링되는가
- 모바일 접속 시에도 같은 스킨이 적용되는가 (`bo_mobile_skin` 누락 여부)
- Q&A 목록 / 작성 / 상세가 커스텀 스킨으로 나오는가
- 비로그인 상태에서 커스텀 관리자 URL 직접 입력 시 차단되는가
- CSS/JS/이미지가 404 없이 로드되는가 (경로 대소문자 확인)
