# Monorepo Manager

모노레포 워크플로우를 관리하는 Claude Code 플러그인입니다. Turborepo, Nx, pnpm, Yarn, Lerna를 지원합니다.

## 개요

`monorepo-manager`는 모노레포 프로젝트의 워크스페이스 분석, 패키지 추가, 변경 영향 분석, 의존성 그래프 시각화 등 일상적인 모노레포 워크플로우를 자동화하는 Claude Code CLI 플러그인입니다.

## 주요 기능

| 기능 | 설명 |
|------|------|
| 워크스페이스 분석 | 패키지 구조, 내부 의존성, 공유 설정 전체 분석 |
| 패키지 추가 | 기존 컨벤션에 맞춰 새 패키지/앱 자동 스캐폴딩 |
| 변경 영향 분석 | Git diff 기반 변경된 패키지 및 전이적 영향 분석 |
| 의존성 그래프 | 순환 의존성 탐지, 메트릭 계산, Mermaid 시각화 |
| 빌드 최적화 | 토폴로지 빌드 순서, 병렬화, 캐시 설정 제안 |

## 지원 모노레포 도구

| 도구 | 역할 | 감지 파일 |
|------|------|-----------|
| Turborepo | 오케스트레이터 | `turbo.json` |
| Nx | 오케스트레이터 | `nx.json` |
| Lerna | 오케스트레이터 | `lerna.json` |
| pnpm | 워크스페이스 매니저 | `pnpm-workspace.yaml` |
| Yarn | 워크스페이스 매니저 | `yarn.lock` + `workspaces` |
| npm | 워크스페이스 매니저 | `package-lock.json` + `workspaces` |

## 설치

프로젝트의 `.claude/plugins/` 디렉토리에 clone합니다:

```bash
git clone https://github.com/wowoyong/claude-plugin-monorepo-manager.git .claude/plugins/monorepo-manager
```

### 업데이트

```bash
cd .claude/plugins/monorepo-manager && git pull
```


## 사용법

### 패키지 추가

```bash
# 대화형 패키지 추가
/workspace-add

# 이름 지정
/workspace-add my-utils

# 앱 추가
/workspace-add my-app --type app

# 스코프 지정
/workspace-add auth --scope @myorg
```

7단계 스캐폴딩 워크플로우:
1. 모노레포 타입 감지
2. 정보 수집 (이름, 타입, 설명, 의존성)
3. 기존 패키지에서 컨벤션 파악
4. 디렉토리 구조 및 설정 파일 생성
5. 워크스페이스 등록 및 설치
6. 내부 의존성 추가 (workspace protocol)
7. 빌드 검증

### 변경 영향 분석

```bash
# 기본 분석 (main 브랜치 대비)
/affected

# 특정 브랜치 대비
/affected --base develop

# 영향받는 패키지만 빌드
/affected --action build

# JSON 출력
/affected --json
```

### 의존성 그래프 분석

의존성 그래프 에이전트가 제공하는 분석:
- 순환 의존성 탐지 (DFS 기반)
- 패키지별 Fan-in/Fan-out 메트릭
- 토폴로지 빌드 순서
- Mermaid 다이어그램 시각화
- ASCII 트리 (20개 미만 패키지)
- 의존성 매트릭스 (20개 이상 패키지)

## 플러그인 구조

```
monorepo-manager/
├── .claude-plugin/
│   └── plugin.json              # 플러그인 메타데이터 및 설정
├── skills/
│   └── monorepo/
│       └── SKILL.md             # 전체 모노레포 관리 워크플로우
├── commands/
│   ├── workspace-add.md         # /workspace-add 명령어 정의
│   └── affected.md              # /affected 명령어 정의
├── agents/
│   ├── workspace-analyzer.md    # 워크스페이스 구조 분석 에이전트
│   └── dependency-graph.md      # 의존성 그래프 분석 에이전트
└── README.md                    # 이 파일
```

## CI/CD 연동

```bash
# 영향받는 패키지만 빌드/테스트 (Turborepo)
turbo run build test --filter=...[origin/main]

# 영향받는 패키지만 빌드/테스트 (Nx)
nx affected --target=build test

# /affected 명령으로 동적 필터 생성
/affected --json | jq -r '.commands.build'
```

## 라이선스

MIT License
