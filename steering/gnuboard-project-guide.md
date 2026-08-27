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
