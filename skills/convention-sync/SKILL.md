---
name: convention-sync
description: 개인 코드 컨벤션(github.com/whsanha55/conventions)을 현재 프로젝트의 docs/convention/으로 가져오거나 최신화한다. README와 CLAUDE.md에 안내를 넣고, 다시 실행하면 원본 최신 커밋과 비교해 갱신한다. 사용자가 "/convention-sync"를 호출하거나 "컨벤션 가져와줘", "컨벤션 동기화", "컨벤션 최신화"처럼 명시적으로 요청할 때만 사용한다. 평소 코드 작성 중에는 호출하지 않는다 — 그때는 프로젝트의 docs/convention/을 읽는다.
---

# convention-sync

원본 저장소의 컨벤션을 현재 프로젝트에 복사하고, 이후 최신 상태로 유지한다.

- 원본: `https://github.com/whsanha55/conventions` (public, `main` 브랜치)
- 설치 위치: 프로젝트 루트의 `docs/convention/`
- 파일을 바꾸기만 하고 커밋하지 않는다.

## 설치 결과

```text
docs/convention/
├── README.md               # 원본 README (우선순위, 예외 절차)
├── common/                 # 원본 common/
├── backend/common/         # 백엔드일 때
├── backend/{language}/     # 백엔드일 때, 해당 언어
├── frontend/...            # 프론트엔드일 때
├── LOCAL.md                # 프로젝트별 예외. 동기화가 건드리지 않는다
├── .convention-lock        # 원본 주소, 커밋 SHA, 대상 경로
└── .convention-sha256      # 설치한 파일의 해시 (로컬 수정 감지용)
```

## 절차

### 1. 프로젝트 종류 판단

프로젝트 루트에서 판단한다.

| 조건 | 종류 | 대상 경로 |
|---|---|---|
| `build.gradle.kts`, `build.gradle`, `pom.xml` 중 하나가 있고 `src/main/kotlin` 또는 Kotlin 플러그인이 있음 | backend/kotlin | `README.md`, `common/`, `backend/common/`, `backend/kotlin/` |
| 위 빌드 파일이 있고 Java만 있음 | backend/java | `README.md`, `common/`, `backend/common/`, `backend/java/` |
| `package.json`이 있음 | frontend | `README.md`, `common/`, `frontend/` 아래 해당 경로 |

- 여러 조건에 해당하거나(모노레포) 판단할 수 없으면 사용자에게 묻는다.
- 원본에 해당 언어 폴더가 아직 없으면 알리고, 있는 경로만 가져온다.
- 기존 lock 파일이 있으면 그 안의 대상 경로를 우선한다.

### 2. 원본 가져오기

```bash
REPO=https://github.com/whsanha55/conventions
REMOTE_SHA=$(git ls-remote "$REPO" refs/heads/main | cut -f1)
TMP=$(mktemp -d)
git clone --quiet --filter=blob:none "$REPO" "$TMP/conventions"
```

### 3. 최신 여부 확인

`docs/convention/.convention-lock`이 있으면 `sha`를 `REMOTE_SHA`와 비교한다.

- 같으면: "최신 상태"라고 알리고, 로컬 수정 여부(4단계)만 확인한 뒤 끝낸다.
- 다르면: 바뀐 내용을 요약해 보여준다.

```bash
git -C "$TMP/conventions" log --oneline "$OLD_SHA..$REMOTE_SHA" -- <대상 경로들>
git -C "$TMP/conventions" diff --stat "$OLD_SHA" "$REMOTE_SHA" -- <대상 경로들>
```

대상 경로에 변경이 없으면 lock의 `sha`만 갱신한다. 변경이 있으면 사용자 확인 후 5단계로 간다.

### 4. 로컬 수정 감지

```bash
cd docs/convention && shasum -a 256 -c .convention-sha256 --quiet
```

해시가 다른 파일은 누군가 직접 고친 것이다. 덮어쓰기 전에 파일마다 로컬 내용과 원본 최신 내용의 diff를 보여주고 묻는다.

- 덮어쓰기: 원본으로 교체한다.
- 유지: 그 파일은 건너뛴다. 프로젝트 예외라면 `LOCAL.md`로 옮기라고 제안한다.

### 5. 파일 복사

- 대상 경로의 파일을 `docs/convention/`에 같은 구조로 복사한다.
- 이전 `.convention-sha256`에 있었지만 원본에서 삭제된 파일은 지운다.
- `LOCAL.md`는 없을 때만 아래 내용으로 만든다. 있으면 절대 건드리지 않는다.

```markdown
# Local Convention

이 프로젝트에만 적용하는 규칙과 예외. `docs/convention/`의 다른 문서보다 우선한다.

(없음)
```

### 6. 도구 설정 (백엔드 Kotlin)

`backend/kotlin/tooling/`의 설정을 프로젝트에 설치할지 묻는다. 최초 설치 때, 또는 원본의 설정 파일이 바뀌었을 때만 묻는다.

| 원본 | 설치 위치 |
|---|---|
| `tooling/.editorconfig` | 프로젝트 루트 `.editorconfig` |
| `tooling/detekt.yml` | `config/detekt/detekt.yml` |

- 설치 위치에 파일이 이미 있고 내용이 다르면, 덮어쓰지 않고 diff를 보여준 뒤 묻는다.
- Gradle 설정은 파일로 설치하지 않는다. `tooling/README.md`의 예시를 안내만 한다.

### 7. README와 CLAUDE.md 안내

마커 사이 블록을 넣는다. 블록이 이미 있으면 마커 사이만 교체한다. 파일이 없으면 만든다.

README.md:

```markdown
<!-- convention:start -->
## Convention

이 프로젝트는 [whsanha55/conventions](https://github.com/whsanha55/conventions) (`<short sha>`)를 따른다. 문서는 `docs/convention/`에 있고, 프로젝트 예외는 `docs/convention/LOCAL.md`에 적는다.
<!-- convention:end -->
```

CLAUDE.md:

```markdown
<!-- convention:start -->
## Convention

코드를 작성하거나 리뷰하기 전에 `docs/convention/`에서 관련 문서와 `LOCAL.md`를 읽고 따른다. `LOCAL.md`가 우선한다.
`docs/convention/`의 파일은 `LOCAL.md`를 제외하고 직접 수정하지 않는다. 갱신은 `/convention-sync`로만 한다.
<!-- convention:end -->
```

CLAUDE.md에서 `@docs/convention/...` import를 쓰지 않는다. 문서 전체가 매 세션 컨텍스트에 들어가기 때문이다.

### 8. lock 파일 기록

```bash
# TARGETS: 1단계에서 정한 대상 경로 (예: README.md common backend/common backend/kotlin)
cat > docs/convention/.convention-lock <<EOF
source=https://github.com/whsanha55/conventions
sha=$REMOTE_SHA
synced_at=$(date -u +%Y-%m-%dT%H:%M:%SZ)
targets=$TARGETS
EOF

# 해시는 로컬 파일이 아니라 원본 파일 기준으로 계산한다 (경로 구조는 같다)
(cd "$TMP/conventions" && find $TARGETS -type f | sort | xargs shasum -a 256) \
  > docs/convention/.convention-sha256
```

해시를 원본 기준으로 기록하므로, 4단계에서 "유지"를 고른 파일은 다음 동기화 때도 로컬 수정으로 감지되어 다시 묻는다.

### 9. 정리와 보고

- `rm -rf "$TMP"`
- 추가, 변경, 삭제, 건너뛴 파일과 새 SHA를 보고한다.
- 커밋 메시지를 제안한다: `docs: 컨벤션 동기화 (<short sha>)`
