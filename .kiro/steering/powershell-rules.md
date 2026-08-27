# PowerShell 실행 규칙

이 워크스페이스의 셸은 PowerShell 7 / 코드페이지 949(`ks_c_5601-1987`)다.

## 금지

- `&`, `&&` 를 명령 구분자로 쓰지 않는다. PowerShell에서 `&`는 백그라운드 잡 실행이며,
  잡이 `Running` 으로 남아 응답 대기가 걸리면서 턴이 정지한다.
- **셸 인자에 한글 등 비ASCII 문자를 넘기지 않는다.** 출력이 깨지고 세션이 불안정해진다.
- `echo` / `Set-Content` / 히어독으로 문서 본문을 쓰지 않는다.
- 개발 서버, watch, 대화형 명령을 일반 셸로 실행하지 않는다.

## 필수

- 명령 구분자는 `;` 를 쓴다.
- 한 호출에 명령 하나. 꼭 묶어야 할 때만 `;` 로 최소한만 묶는다.
- `log` / `diff` / `show` 는 `--no-pager` 를 붙인다. 예: `git --no-pager log --oneline -5`
- 커밋 메시지는 ASCII로 쓴다. 한글이 필요하면 파일로 만들어 `git commit -F <file>` 로 전달한다.
- 한글 본문은 `fs_write` / `fs_append` 로 파일에 직접 쓴다. 셸을 경유시키지 않는다.

## 정지 의심 시 점검

```powershell
Get-Job
Get-Job | Remove-Job -Force
```

매달린 잡이 있으면 정리한다.

## 새 저장소 주의

커밋이 없는 저장소에서 `HEAD` 참조 명령(`git rev-parse HEAD`, `git log`, `git diff HEAD`)은 실패한다.
첫 커밋 전 상태 확인은 `git status --short` 로 한다.

## 참고

경로에 한글이 포함된 것 자체는 문제가 아니다(`...\큐비욘드\...` 에서 git 정상 동작 확인).
문제가 되는 것은 명령 **인자로 넘기는** 비ASCII 텍스트다.
