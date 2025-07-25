# Claude Code UI 프로젝트 핵심 기능 파일 분석

## 프로젝트 개요
Claude Code UI는 Anthropic의 공식 Claude Code CLI를 위한 웹 기반 사용자 인터페이스입니다. React 프론트엔드와 Node.js 백엔드로 구성된 풀스택 애플리케이션으로, Claude AI와의 실시간 채팅, 파일 관리, Git 통합 등의 기능을 제공합니다.

## 시스템 아키텍처

```
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   Frontend      │    │   Backend       │    │  Claude CLI     │
│   (React/Vite)  │◄──►│ (Express/WS)    │◄──►│  Integration    │
└─────────────────┘    └─────────────────┘    └─────────────────┘
```

---

## 백엔드 핵심 파일 분석

### 1. `server/index.js` - 메인 서버 애플리케이션
**역할**: Express 서버 및 WebSocket 서버의 중앙 관리
**핵심 기능**:
- Express 애플리케이션 초기화 및 미들웨어 설정
- WebSocket 서버 생성 및 실시간 통신 관리
- API 라우트 등록 (Git, 인증, MCP)
- Claude 프로젝트 폴더 감시 및 자동 업데이트
- 파일 시스템 API 제공 (파일 읽기/쓰기/삭제)
- 정적 파일 서빙 및 CORS 설정
- 터미널 세션 관리 (node-pty 사용)

**주요 엔드포인트**:
- `/api/projects` - 프로젝트 목록 조회
- `/api/files/*` - 파일 시스템 접근
- `/api/terminal` - 터미널 세션 관리
- `/ws` - WebSocket 연결

### 2. `server/claude-cli.js` - Claude CLI 통합 관리자
**역할**: Claude Code CLI와의 직접적인 인터페이스 및 프로세스 관리
**핵심 기능**:
- Claude CLI 프로세스 스폰 및 관리
- 세션별 활성 프로세스 추적 (`activeClaudeProcesses` Map)
- 이미지 업로드 처리 (임시 파일 생성)
- 스트림 데이터 파싱 및 실시간 메시지 전송
- 세션 중단 및 정리 기능
- 도구 권한 설정 관리
- 오류 처리 및 프로세스 복구

**핵심 함수**:
- `spawnClaude()` - 새로운 Claude 세션 시작
- `abortClaudeSession()` - 세션 강제 중단

### 3. `server/projects.js` - 프로젝트 데이터 관리자
**역할**: Claude 프로젝트 및 세션 데이터 처리
**핵심 기능**:
- 프로젝트 목록 스캔 및 메타데이터 추출
- 세션 데이터 JSONL 파싱
- 프로젝트 디렉토리 경로 추출 및 캐싱
- 프로젝트/세션 CRUD 작업
- package.json 기반 프로젝트명 생성
- 세션 메시지 내용 로드

**핵심 함수**:
- `getProjects()` - 모든 프로젝트 조회
- `getSessions()` - 특정 프로젝트 세션 목록
- `getSessionMessages()` - 세션 메시지 내용
- `extractProjectDirectory()` - 실제 프로젝트 경로 추출

### 4. `server/routes/` - API 라우트 모듈들

#### `server/routes/git.js` - Git 통합 라우터
**역할**: Git 버전 관리 시스템과의 인터페이스
**핵심 기능**:
- Git 저장소 유효성 검증 및 상태 확인
- 파일 상태 추적 (staged, modified, untracked)
- 브랜치 생성/전환/삭제
- 커밋 생성 및 히스토리 조회
- diff 생성 및 파일 비교
- 실제 프로젝트 경로 추출 및 검증

**핵심 엔드포인트**:
- `GET /status` - Git 상태 및 변경 파일 목록
- `POST /stage` - 파일 스테이징
- `POST /commit` - 커밋 생성
- `GET /branches` - 브랜치 목록 조회

#### `server/routes/auth.js` - 인증 및 보안 관리
**역할**: 사용자 인증 및 API 보안
**핵심 기능**:
- JWT 토큰 기반 인증
- 사용자 등록/로그인 처리
- API 키 관리 및 저장
- 세션 보안 유지

#### `server/routes/mcp.js` - Model Context Protocol
**역할**: Claude AI 확장 프로토콜 지원
**핵심 기능**:
- MCP 서버 통합
- 확장된 AI 도구 제공
- 컨텍스트 관리 및 최적화

---

## 프론트엔드 핵심 파일 분석

### 1. `src/App.jsx` - 메인 애플리케이션 컨테이너
**역할**: 최상위 애플리케이션 상태 관리 및 라우팅
**핵심 기능**:
- **세션 보호 시스템**: 활성 대화 중 프로젝트 업데이트 일시 정지
- 프로젝트/세션 상태 관리
- 반응형 디자인 지원 (모바일/데스크톱)
- WebSocket 연결 및 실시간 업데이트 처리
- 도구 설정 및 사용자 환경설정 관리
- 버전 업데이트 알림 시스템

**특별한 기능 - 세션 보호 시스템**:
```javascript
// 활성 세션 추적으로 대화 중단 방지
const [activeSessions, setActiveSessions] = useState(new Set());
```

### 2. `src/components/ChatInterface.jsx` - 채팅 인터페이스
**역할**: Claude AI와의 실시간 대화 인터페이스
**핵심 기능**:
- 메시지 전송/수신 및 실시간 스트리밍
- 마크다운 렌더링 및 코드 하이라이팅
- 이미지 업로드 및 드래그 앤 드롭 지원
- 메시지 히스토리 로컬 저장 (압축 및 크기 제한)
- 세션 보호 시스템과 통합
- 도구 사용 결과 표시
- 자동 스크롤 및 사용자 설정

**핵심 상태**:
- `messages` - 현재 세션 메시지 목록
- `isStreaming` - 실시간 응답 수신 상태
- `currentStreamingMessage` - 스트리밍 중인 메시지

### 3. `src/components/Sidebar.jsx` - 사이드바 네비게이션
**역할**: 프로젝트 및 세션 탐색 인터페이스
**핵심 기능**:
- 프로젝트 목록 표시 및 검색
- 세션 히스토리 관리
- 세션/프로젝트 생성/삭제
- 실시간 업데이트 표시
- 사용자 설정 접근
- 반응형 모바일 지원

### 4. `src/components/MainContent.jsx` - 메인 콘텐츠 영역
**역할**: 탭 기반 인터페이스 관리 및 콘텐츠 라우팅
**핵심 기능**:
- 탭 전환 (채팅, 파일, Git, 미리보기)
- 모바일 네비게이션 통합
- 세션별 컨텍스트 유지
- 로딩 상태 관리

### 5. `src/components/CodeEditor.jsx` - 코드 에디터
**역할**: 파일 편집 및 코드 하이라이팅
**핵심 기능**:
- CodeMirror 기반 고급 코드 에디터
- 다중 언어 지원 (Python, JavaScript, HTML, CSS, JSON, Markdown)
- 다크 모드 지원
- 파일 저장 및 실시간 미리보기
- 구문 하이라이팅 및 코드 폴딩

### 6. `src/components/FileTree.jsx` - 파일 탐색기
**역할**: 프로젝트 파일 시스템 탐색 및 관리
**핵심 기능**:
- 트리 구조 파일 브라우저
- 파일/폴더 생성/삭제/이름변경
- 실시간 파일 변경 감지
- 파일 타입별 아이콘 표시
- 검색 및 필터링

### 7. `src/components/GitPanel.jsx` - Git 통합 패널
**역할**: Git 버전 관리 인터페이스
**핵심 기능**:
- Git 상태 조회 및 표시
- 파일 스테이징/언스테이징
- 커밋 생성 및 히스토리 보기
- 브랜치 전환 및 관리
- 차이점 보기 (diff)

### 8. `src/components/Shell.jsx` - 터미널 에뮬레이터
**역할**: 브라우저 내 터미널 인터페이스
**핵심 기능**:
- xterm.js 기반 완전한 터미널 에뮬레이션
- Claude CLI 직접 접근
- 다중 터미널 세션 지원
- 복사/붙여넣기 및 키보드 단축키
- 터미널 테마 및 폰트 설정

### 9. `src/components/ToolsSettings.jsx` - 도구 설정 관리자
**역할**: Claude 도구 권한 및 시스템 설정 관리
**핵심 기능**:
- Claude 도구 허용/차단 목록 관리
- 권한 검증 설정 (skipPermissions)
- MCP (Model Context Protocol) 서버 구성
- 테마 설정 (다크/라이트 모드)
- 프로젝트 정렬 순서 설정
- 설정 데이터 영구 저장

**보안 특징**:
- 기본적으로 모든 도구 비활성화
- 사용자 명시적 허용 필요
- 세분화된 권한 제어

### 10. `src/components/MicButton.jsx` - 음성 입력 컨트롤
**역할**: 음성 인식 및 Whisper API 통합
**핵심 기능**:
- 실시간 음성 녹음
- Whisper API를 통한 음성-텍스트 변환
- 녹음 상태 시각화
- 다국어 음성 인식 지원

---

## 유틸리티 및 라이브러리 분석

### 1. `src/utils/websocket.js` - WebSocket 관리자
**역할**: 실시간 통신 연결 관리
**핵심 기능**:
- WebSocket 연결 및 재연결 로직
- 인증 토큰 기반 보안 연결
- 메시지 큐 및 상태 관리
- 자동 재연결 메커니즘

### 2. `src/utils/api.js` - API 클라이언트 래퍼
**역할**: 백엔드 API와의 통신 추상화
**핵심 기능**:
- 인증 토큰 자동 관리 (`authenticatedFetch`)
- REST API 엔드포인트 일관된 인터페이스 제공
- 오류 처리 및 HTTP 헤더 관리
- 프로젝트, 세션, 파일 관련 API 호출
- 음성 전사 (Whisper) API 통합

**주요 API 그룹**:
- `auth` - 인증 관련 (로그인/등록/로그아웃)
- `projects` - 프로젝트 관리
- `sessions` - 세션 데이터 처리
- `files` - 파일 시스템 작업

### 3. `src/contexts/` - React Context 제공자들
**구성 요소**:
- `ThemeContext.js` - 다크/라이트 모드 관리
- `AuthContext.js` - 사용자 인증 상태 관리

### 4. `src/hooks/` - 커스텀 React Hook들
**구성 요소**:
- `useVersionCheck.js` - GitHub 버전 업데이트 확인
- `useAudioRecorder.js` - 음성 녹음 및 Whisper 변환

#### `useVersionCheck.js` - 버전 관리 Hook
**역할**: 자동 업데이트 확인 시스템
**핵심 기능**:
- GitHub API를 통한 최신 릴리스 확인
- 현재 버전과 비교
- 5분마다 자동 확인
- 업데이트 알림 상태 관리

---

## 설정 파일 분석

### 1. `package.json` - 프로젝트 의존성 및 스크립트
**주요 의존성**:
- **Frontend**: React 18, Vite, Tailwind CSS, CodeMirror
- **Backend**: Express, WebSocket, node-pty, better-sqlite3
- **UI Libraries**: Lucide React (아이콘), React Markdown
- **Development**: Concurrently (동시 실행)

### 2. `vite.config.js` - 빌드 도구 설정
**역할**: Vite 개발 서버 및 빌드 설정

### 3. `tailwind.config.js` - CSS 프레임워크 설정
**역할**: Tailwind CSS 커스터마이징 및 다크 모드 설정

---

## 데이터 흐름 및 아키텍처

### 1. 사용자 인터랙션 플로우
```
사용자 입력 → ChatInterface → WebSocket → 서버 → Claude CLI → 응답 스트림 → 프론트엔드 업데이트
```

### 2. 실시간 업데이트 시스템
- **프로젝트 감시**: `chokidar`로 파일 시스템 변경 감지
- **WebSocket 브로드캐스트**: 연결된 모든 클라이언트에 업데이트 전송
- **세션 보호**: 활성 대화 중 업데이트 일시 정지

### 3. 상태 관리 계층
- **로컬 상태**: 컴포넌트별 UI 상태
- **글로벌 상태**: App.jsx에서 프로젝트/세션 관리
- **영구 저장**: localStorage (설정, 메시지 히스토리)
- **서버 상태**: SQLite 데이터베이스 (인증)

---

## 보안 및 성능 최적화

### 1. 보안 기능
- JWT 토큰 기반 인증
- API 키 보안 저장
- 도구 권한 세분화 제어
- CORS 및 요청 검증

### 2. 성능 최적화
- React.memo를 통한 불필요한 리렌더링 방지
- 메시지 히스토리 압축 및 크기 제한
- 프로젝트 디렉토리 캐싱
- 코드 스플리팅 및 동적 임포트

---

## 결론

Claude Code UI는 복잡한 AI 도구를 직관적인 웹 인터페이스로 추상화한 정교한 시스템입니다. 특히 세션 보호 시스템과 실시간 통신, 그리고 완전한 터미널 통합을 통해 개발자에게 원활한 AI 지원 코딩 환경을 제공합니다. 각 파일은 명확한 책임을 가지고 있으며, 모듈화된 아키텍처를 통해 유지보수성과 확장성을 보장합니다.