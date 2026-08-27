# gnuboard-steering-kiro

그누보드5 기반 **기업용 웹사이트를 반복 제작**할 때 Kiro가 참고하는 steering 파일 모음이다.
`yesfa.co.kr` 프로젝트에서 정리된 패턴을 재사용 가능한 형태로 문서화했다.

## 이 레포는 무엇인가

기업 사이트 제작은 매번 비슷한 흐름을 반복한다.
정적 HTML을 퍼블리싱하고, 그누보드5 스킨으로 변환하고, 상담/문의를 처리할 커스텀 관리자를 붙이고, FTP로 올린다.
매번 같은 판단을 다시 하지 않도록 그 흐름과 규칙을 steering 문서로 고정한 것이 이 레포다.

## 대상 환경

| 항목 | 값 |
|---|---|
| CMS | 그누보드5 v5.6.32 |
| 런타임 | PHP + MySQL |
| 호스팅 | 국내 공유호스팅 (SSH/Composer/빌드도구 없음 가정) |
| 배포 | FTP 수동 업로드 |
| 게시판 스킨 | `g5-custom/skin/board/{project}/` |
| Q&A 스킨 | `g5-custom/skin/qa/{project}/` |
| 상담 상태 | 접수중 / 접수완료 / 답변중 / 답변완료 (`qa_status`) |
| 비회원 문의 | 허용 (`qawrite.php` 로그인 체크 비활성화) |

## 수록 문서

| 파일 | 내용 |
|---|---|
| `steering/gnuboard-project-guide.md` | 프로젝트 구조, 호스팅 환경, DB 테이블, 커스텀 관리자, 진입점 연결, 그누보드 상수/변수 |
| `steering/publishing-standards.md` | HTML/CSS/JS 퍼블리싱 표준, 네이밍 규칙, 반응형, 애니메이션 규칙 |
| `steering/skin-conversion-rules.md` | HTML → PHP 스킨 변환 규칙 (공지 목록/상세, Q&A 문의) 코드 예제 |

세 문서 모두 frontmatter에 `inclusion: manual` 이 설정되어 있다.
항상 로딩되지 않으며, 채팅에서 `#` 컨텍스트로 직접 불러서 쓴다.

## 새 프로젝트 시작하기

1. 새 레포 생성 + 클론
2. 이 레포의 `steering/` 파일들을 `.kiro/steering/` 에 복사
3. 그누보드5 다운로드 → FTP 업로드 → 설치 마법사 실행 → `install/` 삭제
4. 디자이너 시안 → HTML/CSS 퍼블리싱 (`publishing-standards.md` 참고)
5. 퍼블리싱 → 그누보드 스킨 변환 (`skin-conversion-rules.md` 참고)
6. 커스텀 관리자 이식 (이전 프로젝트에서 `g5-custom/adm/` 복사 후 수정)
7. 프론트 진입점 연결 — HTML → PHP (`gnuboard-project-guide.md` 참고)
8. FTP 배포 + 동작 확인
