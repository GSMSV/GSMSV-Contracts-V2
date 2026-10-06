# GSMSV v2 GitHub 협업 규칙

> 읽는 사람: GSMSV v2를 개발하는 학생. 다른 사람과 GitHub로 협업하는 게 처음이어도 이 문서만 보고 따라 할 수 있게 썼다.

## 1. 한눈에 보기

GSMSV v2는 분야별로 저장소를 나눈다.

| 저장소 | 내용 | Discord 알림 채널 |
|---|---|---|
| `GSMSV-Server-V2` | Kotlin API 백엔드 | `backend` |
| `GSMSV-Client-V2` | React 웹 콘솔 | `frontend` |
| `GSMSV-Infra-V2` | Go로 Proxmox·k3s를 다루는 실행 엔진 | `cloud` |
| `GSMSV-Contracts-V2` | 저장소끼리 함께 쓰는 약속 (API 명세, 자원 이름 규칙) | `backend`·`frontend`·`cloud` 모두 |

웹 API 명세는 `GSMSV-Contracts-V2`에 **OpenAPI** 형식으로 쓴다. API를 새로 만들거나 바꿀 때는 구현보다 **명세 PR을 먼저** 올린다 (계약 먼저).

작업은 항상 이 순서로 흐른다.

```text
이슈 만들기 → develop에서 작업 브랜치 → 커밋·push → develop으로 PR → 직접 합치기 (승인 없음)
                                                         │
                                         배포할 때: develop → main PR → 1명 승인 → 합치기
```

- `develop`: 다음 배포에 들어갈 코드를 모으는 브랜치. **기본 브랜치**다
- `main`: 실제로 배포되는 코드
- 작업 브랜치: 각자 작업하는 곳. 마음대로 push해도 된다

## 2. 처음 왔을 때

1. 조직 owner에게 GitHub 아이디를 알려 주고 v2 저장소의 **Write** 권한을 받는다
2. 저장소를 받는다

   ```bash
   git clone https://github.com/GSMSV/GSMSV-Server-V2.git
   cd GSMSV-Server-V2
   git switch develop
   ```

3. 아래 3~7장을 한 번 읽는다

## 3. 작업 순서

### 3.1 이슈 만들기

- **기능 추가·버그 수정은 이슈를 먼저 만든다.** 저장소 Issues 탭 → New issue
- 오타 수정, 설정 파일 정리 같은 작은 작업은 이슈 없이 PR만 열어도 된다
- 이유: 나중에 다음 기수가 "이 코드는 왜 이렇게 됐지?"를 이슈에서 찾는다

### 3.2 작업 브랜치 만들기

develop을 최신으로 받은 뒤 거기서 만든다.

```bash
git switch develop
git pull
git switch -c feat/12-vm-create-api
```

브랜치 이름은 `타입/이슈번호-설명`이다.

| 규칙 | 예 |
|---|---|
| 타입은 4장의 목록에서 고른다 | `feat`, `fix`, `docs` … |
| 이슈 번호를 넣는다. 이슈가 없는 작은 작업은 번호를 뺀다 | `fix/31-login-redirect`, `docs/readme-typo` |
| 설명은 영어 소문자, 단어는 하이픈(`-`)으로 잇는다 | `vm-create-api` (O) / `VM_Create` (X) |

### 3.3 커밋하고 push하기

```bash
git add 바꾼파일
git commit -m "feat: VM 생성 API 추가"
git push -u origin feat/12-vm-create-api   # 처음 한 번만 -u, 그다음부터는 git push
```

- 작업 브랜치 안의 커밋은 여러 개여도 된다. develop에 합칠 때 하나로 뭉쳐지기 때문이다 (5장)
- 그래도 커밋 메시지는 4장 형식을 따르는 게 좋다

작업하는 동안 develop이 바뀌었으면 내 브랜치에 가져온다.

```bash
git fetch origin
git merge origin/develop
```

### 3.4 PR 열기

1. push한 뒤 저장소 화면에 뜨는 **Compare & pull request**를 누른다. 또는 Pull requests 탭 → New pull request
2. **base가 `develop`인지 확인한다** (기본값이 develop이다)
3. 제목과 본문을 쓴다 (5장)
4. 아직 작업 중이면 **Create draft pull request**로 연다. 리뷰받을 준비가 안 됐다는 표시다. 준비되면 PR 화면에서 **Ready for review**를 누른다

### 3.5 develop에 합치기

- develop으로 가는 PR은 **승인이 필요 없다.** 확인이 끝나면 PR을 연 사람이 직접 합친다
- 합치는 버튼은 **Squash and merge** 하나만 있다
- 합치기 전에 **PR 제목이 4장 형식인지 한 번 더 본다.** PR 제목이 그대로 develop의 커밋 메시지가 된다
- 저장소에 CI(자동 검사)가 생기면, CI를 통과해야 합칠 수 있게 바뀐다

합친 뒤 GitHub의 작업 브랜치는 자동으로 지워진다. 내 컴퓨터의 브랜치는 직접 지운다.

```bash
git switch develop
git pull
git branch -D feat/12-vm-create-api
```

`-d` 대신 `-D`를 쓰는 이유: squash로 합치면 git은 이 브랜치가 합쳐진 줄 모른다. GitHub에서 합친 걸 확인했다면 `-D`로 지워도 된다.

## 4. 커밋·PR 제목 규칙

형식은 `타입: 한국어 설명`이다. [Conventional Commits](https://www.conventionalcommits.org/ko/v1.0.0/)의 표준 타입을 쓴다.

| 타입 | 언제 | 예 |
|---|---|---|
| `feat` | 새 기능 | `feat: VM 생성 API 추가` |
| `fix` | 버그 수정 | `fix: 만료된 토큰으로 로그인되던 문제 수정` |
| `docs` | 문서만 바꿈 | `docs: README에 실행 방법 추가` |
| `style` | 동작은 그대로, 포맷·줄바꿈만 | `style: ktlint 포맷 적용` |
| `refactor` | 동작은 그대로, 구조만 개선 | `refactor: 권한 확인 코드를 자원 로더로 옮김` |
| `perf` | 성능 개선 | `perf: 노드 상태 조회 캐시 추가` |
| `test` | 테스트 추가·수정 | `test: GRN 파서 실패 케이스 추가` |
| `build` | 빌드 설정, 의존성 | `build: Spring Boot 버전 올림` |
| `ci` | CI·GitHub Actions 설정 | `ci: 백엔드 빌드 워크플로 추가` |
| `chore` | 그 밖의 잡무 | `chore: .gitignore 정리` |
| `revert` | 이전 커밋 되돌리기 | `revert: VM 생성 API 추가 되돌림` |

- 콜론(`:`) 뒤에 한 칸 띄운다. 콜론 앞에는 띄우지 않는다
- 설명은 한국어, 끝에 마침표를 찍지 않는다
- 한 PR에는 한 가지 일만 담는다. 제목에 "그리고"가 들어가면 PR을 나눌 때다

## 5. PR 본문

```markdown
## Summary
- **키워드**: 무엇을 바꿨는지

## Test plan
- [ ] 확인한 방법

## 관련 이슈
Closes #12

## 확인
- [ ] 권한 판단을 자원 로더 밖에서 하지 않았다 (22번 권한 판단 Layer)
- [ ] GRN 문자열을 직접 자르지 않고 GRN 파서만 썼다. GRN의 소유자 칸 값을 어떤 판단에도 쓰지 않았다 (23번 자원 이름 규칙)
- [ ] 비밀값·내부 IP·포트·서버 주소 같은 운영 정보를 넣지 않았다 (공개 저장소)
```

- **`Closes #번호`**: PR이 develop에 합쳐지면 그 이슈가 자동으로 닫힌다. 이슈가 없으면 이 줄을 지운다
- **확인 항목**: 해당하는 것만 체크한다. 22번(권한 판단 Layer)·23번(자원 이름 규칙)의 자세한 내용은 팀 Notion 기능명세서에 있다. 두 규칙을 어기면 보안 사고로 이어질 수 있어서 PR마다 확인한다

## 6. 배포: develop → main

1. Pull requests 탭 → New pull request → **base: `main`, compare: `develop`**
2. 제목 예: `chore: develop → main 배포`
3. **작성자가 아닌 사람 1명이 승인**해야 합칠 수 있다. 저장소에 코드 오너(담당자 목록, `CODEOWNERS` 파일)가 정해져 있으면 그중 1명이 승인해야 한다
4. 승인되면 **Create a merge commit**으로 합친다 (main으로 갈 때는 이 버튼만 있다)

승인하는 방법: PR 화면 → **Files changed** 탭 → 오른쪽 위 **Review changes** → **Approve** → **Submit review**

## 7. 하면 안 되는 것

| 하면 안 되는 것 | 이유 / 대신 할 것 |
|---|---|
| 비밀값(토큰·비밀번호·키), 내부 IP, 포트, 서버 주소를 코드·커밋·이슈·PR·댓글에 쓰기 | **저장소가 공개다.** 비밀값이 필요하면 owner에게 요청해서 GitHub Secrets나 Environment에 넣는다 |
| 비밀값을 실수로 push했을 때 커밋만 지우고 넘어가기 | 한 번 올라간 비밀값은 이미 퍼졌다고 본다. **바로 owner에게 알리고 그 비밀값을 새로 발급(교체)**한다 |
| 보안 문제를 이슈·PR·Discord에 공개로 쓰기 | 저장소 **Security** 탭 → **Report a vulnerability**로 비공개로 알린다 |
| develop·main에 직접 push | 막혀 있다. 항상 작업 브랜치 → PR |

## 8. 막혔을 때

| 증상 | 원인 | 해결 |
|---|---|---|
| push하는데 "secret detected" 같은 메시지로 막힘 | 비밀값처럼 보이는 문자열이 커밋에 있다 (Push protection) | 진짜 비밀값이면 커밋에서 빼고 7장대로 처리한다. 테스트용 가짜 값이면 안내된 링크에서 이유를 고르고 통과시킨다 |
| develop·main에 push가 거부됨 | 직접 push 금지 규칙 | 작업 브랜치를 만들어 PR로 올린다 |
| 워크플로에서 "action is not allowed" | GitHub 공식 액션(`actions/*`)만 허용돼 있다 | 필요한 액션을 owner에게 말해 허용 목록에 추가한다 |
| 워크플로가 커밋·라벨·릴리스를 못 만듦 ("Resource not accessible") | 자동화 기본 권한이 읽기 전용이다 | 워크플로 파일에 필요한 권한을 `permissions:`로 적는다 |
| 저장소에 push 권한이 없음 | Write 권한을 아직 못 받았다 | owner에게 요청한다 |

## 9. 알림

- PR이 열리거나 닫히면, 그리고 이슈가 생기면 1장 표의 Discord 채널로 알림이 간다
- 리뷰 요청이 오면 Discord 알림을 보고 바로 확인해 준다. 리뷰가 늦어지면 배포가 늦어진다

## 10. 용어

| 용어 | 뜻 |
|---|---|
| 브랜치 | 작업용 사본 줄기 |
| PR (Pull Request) | "내 브랜치를 합쳐 주세요" 요청. base = 합쳐질 곳, compare(head) = 내 브랜치 |
| Squash and merge | PR 안의 커밋 여러 개를 하나로 뭉쳐 합친다 |
| Merge commit | 커밋을 그대로 두고 "합쳤다"는 기록을 남겨 합친다 |
| Draft PR | 아직 작업 중이라는 표시가 붙은 PR |
| 코드 오너 (CODEOWNERS) | 저장소·폴더별 담당자 목록. main으로 가는 PR은 이 중 1명이 승인한다 |
| 룰셋 (ruleset) | "main은 승인 1명 필요" 같은 규칙을 GitHub가 강제하는 기능 |
| CI | PR마다 빌드·테스트를 자동으로 돌리는 것 |
| GitHub Secrets / Environment | 비밀값을 GitHub에 암호화해 저장하는 곳. Environment는 지정한 사람이 승인해야 쓸 수 있다 |

## 11. 아직 정하지 않은 것

- 저장소별 코드 오너 명단 (`CODEOWNERS`)
- 저장소별 CI와 필수 검사
- main은 develop에서 온 PR만 받는 검사 (CI에 넣을 예정)
- 배포 방식과 `production` Environment
- 배포 PR 제목 형식 (6장의 예는 제안)
- 라이선스
