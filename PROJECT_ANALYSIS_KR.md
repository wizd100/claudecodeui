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

**상세 동작 방식**:

#### WebSocket 실시간 통신 시스템
```javascript
// 1. WebSocket 서버 초기화
const wss = new WebSocketServer({ server });

// 2. 클라이언트 연결 관리
wss.on('connection', (ws, req) => {
    // JWT 토큰 검증 및 사용자 인증
    const token = extractTokenFromRequest(req);
    ws.userId = verifyJWT(token);
    
    // 연결된 클라이언트 추적
    connectedClients.add(ws);
});

// 3. 브로드캐스트 시스템 - 모든 클라이언트에게 업데이트 전송
function broadcastUpdate(data) {
    connectedClients.forEach(client => {
        if (client.readyState === WebSocket.OPEN) {
            client.send(JSON.stringify(data));
        }
    });
}
```

#### 파일 시스템 감시 및 자동 업데이트
```javascript
// chokidar를 사용한 실시간 파일 변경 감지
const watcher = chokidar.watch(claudeProjectsPath, {
    ignored: /node_modules|\.git/,
    persistent: true
});

watcher.on('all', (event, path) => {
    // 세션 보호: 활성 대화 중인 프로젝트는 업데이트 방지
    if (!isProjectInActiveSession(path)) {
        broadcastUpdate({
            type: 'PROJECT_UPDATED',
            projectPath: path,
            event: event
        });
    }
});
```

#### 터미널 세션 관리
```javascript
// node-pty를 사용한 실제 터미널 프로세스 생성
app.post('/api/terminal', (req, res) => {
    const ptyProcess = pty.spawn('bash', [], {
        name: 'xterm-color',
        cols: 80,
        rows: 24,
        cwd: projectDirectory
    });
    
    // 터미널 출력을 WebSocket으로 실시간 전송
    ptyProcess.on('data', (data) => {
        ws.send(JSON.stringify({
            type: 'terminal_output',
            data: data
        }));
    });
    
    // 사용자 입력을 터미널로 전달
    ws.on('message', (message) => {
        const parsed = JSON.parse(message);
        if (parsed.type === 'terminal_input') {
            ptyProcess.write(parsed.data);
        }
    });
});
```

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

**상세 동작 방식**:

#### Claude CLI 프로세스 관리 시스템
```javascript
// 1. 활성 프로세스 추적을 위한 전역 Map
const activeClaudeProcesses = new Map();

// 2. Claude 세션 시작 프로세스
async function spawnClaude(sessionId, projectPath, userMessage, images, tools) {
    // 기존 프로세스가 있으면 종료
    if (activeClaudeProcesses.has(sessionId)) {
        const existingProcess = activeClaudeProcesses.get(sessionId);
        existingProcess.kill('SIGTERM');
    }
    
    // 새로운 Claude CLI 프로세스 생성
    const claudeProcess = spawn('claude', [
        '--no-confirm',
        '--project-path', projectPath,
        ...(tools.length > 0 ? ['--tools', tools.join(',')] : [])
    ], {
        cwd: projectPath,
        stdio: ['pipe', 'pipe', 'pipe']
    });
    
    // 프로세스 추적에 등록
    activeClaudeProcesses.set(sessionId, claudeProcess);
    
    return claudeProcess;
}
```

#### 실시간 스트림 데이터 처리
```javascript
// Claude CLI 출력을 실시간으로 파싱하고 WebSocket으로 전송
claudeProcess.stdout.on('data', (data) => {
    const lines = data.toString().split('\n');
    
    lines.forEach(line => {
        if (line.trim()) {
            try {
                // JSON 형태의 스트림 데이터 파싱
                const parsed = JSON.parse(line);
                
                // 메시지 타입별 처리
                switch (parsed.type) {
                    case 'content_block_start':
                        ws.send(JSON.stringify({
                            type: 'stream_start',
                            content: parsed.content_block.text
                        }));
                        break;
                        
                    case 'content_block_delta':
                        ws.send(JSON.stringify({
                            type: 'stream_delta',
                            content: parsed.delta.text
                        }));
                        break;
                        
                    case 'tool_use':
                        ws.send(JSON.stringify({
                            type: 'tool_execution',
                            tool: parsed.name,
                            input: parsed.input
                        }));
                        break;
                }
            } catch (e) {
                // 비JSON 데이터 처리 (일반 텍스트 출력)
                ws.send(JSON.stringify({
                    type: 'raw_output',
                    content: line
                }));
            }
        }
    });
});
```

#### 이미지 업로드 처리 시스템
```javascript
// 이미지 데이터를 임시 파일로 저장하고 Claude에 전달
async function handleImageUpload(imageData, filename) {
    // Base64 데이터를 디코딩
    const buffer = Buffer.from(imageData.split(',')[1], 'base64');
    
    // 임시 파일 생성
    const tempPath = path.join(os.tmpdir(), `claude_image_${Date.now()}_${filename}`);
    await fs.writeFile(tempPath, buffer);
    
    // Claude CLI에 이미지 경로 전달
    claudeProcess.stdin.write(`IMAGE:${tempPath}\n`);
    
    // 처리 완료 후 임시 파일 정리
    setTimeout(() => {
        fs.unlink(tempPath).catch(console.error);
    }, 30000); // 30초 후 정리
}
```

#### 세션 중단 및 정리
```javascript
// 안전한 세션 종료 처리
function abortClaudeSession(sessionId) {
    const process = activeClaudeProcesses.get(sessionId);
    if (process) {
        // 정상적인 종료 시도 (SIGTERM)
        process.kill('SIGTERM');
        
        // 5초 후에도 종료되지 않으면 강제 종료 (SIGKILL)
        setTimeout(() => {
            if (!process.killed) {
                process.kill('SIGKILL');
            }
        }, 5000);
        
        // 프로세스 추적에서 제거
        activeClaudeProcesses.delete(sessionId);
    }
}
```

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

**상세 동작 방식**:

#### 프로젝트 스캔 및 메타데이터 추출
```javascript
// Claude 프로젝트 디렉토리 스캔 시스템
async function getProjects() {
    const claudeConfigPath = path.join(os.homedir(), '.claude');
    const projectsPath = path.join(claudeConfigPath, 'projects');
    
    // 프로젝트 디렉토리 스캔
    const projectFolders = await fs.readdir(projectsPath);
    const projects = [];
    
    for (const folder of projectFolders) {
        const projectPath = path.join(projectsPath, folder);
        const configPath = path.join(projectPath, 'config.json');
        
        try {
            // 프로젝트 설정 파일 읽기
            const config = JSON.parse(await fs.readFile(configPath, 'utf8'));
            
            // 실제 프로젝트 경로 추출
            const actualPath = extractProjectDirectory(config);
            
            // package.json 기반 프로젝트명 생성
            const displayName = await generateProjectName(actualPath);
            
            projects.push({
                id: folder,
                name: displayName,
                path: actualPath,
                lastModified: await getLastModified(projectPath),
                sessionCount: await getSessionCount(projectPath)
            });
        } catch (error) {
            console.warn(`프로젝트 ${folder} 로드 실패:`, error.message);
        }
    }
    
    return projects.sort((a, b) => b.lastModified - a.lastModified);
}
```

#### JSONL 세션 데이터 파싱
```javascript
// 세션 메시지를 JSONL 형태로 저장된 파일에서 로드
async function getSessionMessages(projectId, sessionId) {
    const sessionPath = path.join(
        os.homedir(), '.claude', 'projects', projectId, 'sessions', sessionId
    );
    
    try {
        const data = await fs.readFile(sessionPath, 'utf8');
        const lines = data.trim().split('\n');
        const messages = [];
        
        for (const line of lines) {
            try {
                const parsed = JSON.parse(line);
                
                // 메시지 타입별 처리
                if (parsed.type === 'user_message') {
                    messages.push({
                        role: 'user',
                        content: parsed.content,
                        timestamp: parsed.timestamp,
                        images: parsed.images || []
                    });
                } else if (parsed.type === 'assistant_message') {
                    messages.push({
                        role: 'assistant',
                        content: parsed.content,
                        timestamp: parsed.timestamp,
                        tools_used: parsed.tools_used || []
                    });
                }
            } catch (parseError) {
                console.warn('JSONL 파싱 오류:', parseError);
            }
        }
        
        return messages;
    } catch (error) {
        console.error('세션 메시지 로드 실패:', error);
        return [];
    }
}
```

#### 프로젝트 디렉토리 경로 추출 및 캐싱
```javascript
// 캐시를 사용한 효율적인 경로 추출
const directoryCache = new Map();

function extractProjectDirectory(config) {
    const configKey = JSON.stringify(config);
    
    // 캐시에서 먼저 확인
    if (directoryCache.has(configKey)) {
        return directoryCache.get(configKey);
    }
    
    let actualPath;
    
    // 설정에서 실제 프로젝트 경로 추출
    if (config.project_root) {
        actualPath = config.project_root;
    } else if (config.include_paths && config.include_paths.length > 0) {
        // 첫 번째 include_path를 기본 경로로 사용
        actualPath = path.dirname(config.include_paths[0]);
    } else {
        actualPath = process.cwd(); // 기본값
    }
    
    // 상대 경로를 절대 경로로 변환
    if (!path.isAbsolute(actualPath)) {
        actualPath = path.resolve(actualPath);
    }
    
    // 캐시에 저장
    directoryCache.set(configKey, actualPath);
    
    return actualPath;
}
```

#### package.json 기반 스마트 프로젝트명 생성
```javascript
// 프로젝트의 package.json에서 이름을 추출하여 사용자 친화적인 이름 생성
async function generateProjectName(projectPath) {
    try {
        const packageJsonPath = path.join(projectPath, 'package.json');
        const packageData = JSON.parse(await fs.readFile(packageJsonPath, 'utf8'));
        
        if (packageData.name) {
            // npm 패키지명을 사용자 친화적으로 변환
            return packageData.name
                .replace(/[@\/]/g, '-') // @ 및 / 문자 제거
                .replace(/^-+|-+$/g, '') // 앞뒤 하이픈 제거
                .toLowerCase();
        }
    } catch (error) {
        // package.json이 없거나 읽을 수 없는 경우
    }
    
    // 폴더명을 기본값으로 사용
    return path.basename(projectPath);
}
```

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

**상세 동작 방식**:

#### 세션 보호 시스템 (핵심 특징)
```javascript
// 활성 세션 추적을 통한 데이터 일관성 보장
const [activeSessions, setActiveSessions] = useState(new Set());

// 세션 시작 시 보호 활성화
function startSessionProtection(sessionId) {
    setActiveSessions(prev => new Set([...prev, sessionId]));
    
    // 서버에 세션 보호 상태 전송
    ws.send(JSON.stringify({
        type: 'SESSION_PROTECTION_START',
        sessionId: sessionId
    }));
}

// 세션 종료 시 보호 해제
function endSessionProtection(sessionId) {
    setActiveSessions(prev => {
        const newSet = new Set(prev);
        newSet.delete(sessionId);
        return newSet;
    });
    
    ws.send(JSON.stringify({
        type: 'SESSION_PROTECTION_END',
        sessionId: sessionId
    }));
}

// 프로젝트 업데이트 필터링
useEffect(() => {
    if (wsMessage?.type === 'PROJECT_UPDATED') {
        // 활성 세션의 프로젝트는 업데이트 무시
        const isProtected = Array.from(activeSessions).some(sessionId => 
            wsMessage.projectPath.includes(currentProject?.id)
        );
        
        if (!isProtected) {
            // 안전한 경우에만 프로젝트 데이터 갱신
            refreshProjects();
        }
    }
}, [wsMessage, activeSessions]);
```

#### 전역 상태 관리 시스템
```javascript
// 계층적 상태 관리 구조
const [globalState, setGlobalState] = useState({
    // 프로젝트 관련 상태
    projects: [],
    currentProject: null,
    currentSession: null,
    
    // UI 상태
    sidebarOpen: true,
    currentTab: 'chat',
    isMobile: false,
    
    // 시스템 상태
    isConnected: false,
    lastUpdate: null,
    
    // 사용자 설정
    theme: 'dark',
    toolsSettings: {},
    sortOrder: 'lastModified'
});

// 상태 업데이트 최적화
const updateGlobalState = useCallback((updates) => {
    setGlobalState(prev => ({
        ...prev,
        ...updates,
        lastUpdate: Date.now()
    }));
}, []);
```

#### 실시간 WebSocket 통신 관리
```javascript
// WebSocket 연결 및 메시지 처리
useEffect(() => {
    const initializeWebSocket = () => {
        const token = localStorage.getItem('auth_token');
        ws = new WebSocket(`ws://localhost:3001/ws?token=${token}`);
        
        ws.onopen = () => {
            setIsConnected(true);
            console.log('WebSocket 연결 성공');
        };
        
        ws.onmessage = (event) => {
            const message = JSON.parse(event.data);
            handleWebSocketMessage(message);
        };
        
        ws.onclose = () => {
            setIsConnected(false);
            // 자동 재연결 시도
            setTimeout(initializeWebSocket, 5000);
        };
        
        ws.onerror = (error) => {
            console.error('WebSocket 오류:', error);
        };
    };
    
    initializeWebSocket();
    
    return () => {
        if (ws) ws.close();
    };
}, []);

// 메시지 타입별 처리
const handleWebSocketMessage = useCallback((message) => {
    switch (message.type) {
        case 'PROJECT_UPDATED':
            handleProjectUpdate(message);
            break;
        case 'SESSION_CREATED':
            handleSessionCreated(message);
            break;
        case 'STREAM_DATA':
            handleStreamData(message);
            break;
        case 'TOOL_EXECUTION':
            handleToolExecution(message);
            break;
    }
}, [activeSessions]);
```

#### 반응형 디자인 및 모바일 지원
```javascript
// 화면 크기 감지 및 UI 적응
useEffect(() => {
    const handleResize = () => {
        const isMobileView = window.innerWidth < 768;
        setIsMobile(isMobileView);
        
        // 모바일에서는 사이드바 자동 닫기
        if (isMobileView && sidebarOpen) {
            setSidebarOpen(false);
        }
    };
    
    window.addEventListener('resize', handleResize);
    handleResize(); // 초기 실행
    
    return () => window.removeEventListener('resize', handleResize);
}, [sidebarOpen]);

// 모바일 터치 제스처 지원
const handleTouchGesture = useCallback((direction) => {
    if (isMobile) {
        if (direction === 'right' && !sidebarOpen) {
            setSidebarOpen(true);
        } else if (direction === 'left' && sidebarOpen) {
            setSidebarOpen(false);
        }
    }
}, [isMobile, sidebarOpen]);
```

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

**상세 동작 방식**:

#### 실시간 스트리밍 메시지 처리
```javascript
// 스트리밍 상태 관리
const [isStreaming, setIsStreaming] = useState(false);
const [currentStreamingMessage, setCurrentStreamingMessage] = useState('');
const [streamBuffer, setStreamBuffer] = useState('');

// WebSocket 메시지 처리
useEffect(() => {
    if (wsMessage?.type === 'stream_start') {
        setIsStreaming(true);
        setCurrentStreamingMessage('');
        setStreamBuffer('');
        
        // 세션 보호 활성화
        onSessionStart?.(currentSession?.id);
    }
    
    if (wsMessage?.type === 'stream_delta') {
        setStreamBuffer(prev => prev + wsMessage.content);
        
        // 부분 렌더링을 위한 디바운싱
        debouncedUpdateStreaming(streamBuffer + wsMessage.content);
    }
    
    if (wsMessage?.type === 'stream_end') {
        setIsStreaming(false);
        
        // 최종 메시지를 히스토리에 추가
        const finalMessage = {
            role: 'assistant',
            content: streamBuffer,
            timestamp: Date.now(),
            tools_used: wsMessage.tools_used || []
        };
        
        addMessageToHistory(finalMessage);
        setCurrentStreamingMessage('');
        setStreamBuffer('');
        
        // 세션 보호 해제
        onSessionEnd?.(currentSession?.id);
    }
}, [wsMessage]);

// 스트리밍 업데이트 디바운싱 (성능 최적화)
const debouncedUpdateStreaming = useMemo(
    () => debounce((content) => {
        setCurrentStreamingMessage(content);
    }, 50),
    []
);
```

#### 이미지 업로드 시스템
```javascript
// 드래그 앤 드롭 이미지 처리
const [dragOver, setDragOver] = useState(false);
const [uploadedImages, setUploadedImages] = useState([]);

const handleDrop = useCallback(async (e) => {
    e.preventDefault();
    setDragOver(false);
    
    const files = Array.from(e.dataTransfer.files);
    const imageFiles = files.filter(file => file.type.startsWith('image/'));
    
    for (const file of imageFiles) {
        try {
            // 이미지 크기 제한 및 압축
            const compressedImage = await compressImage(file, {
                maxWidth: 1024,
                maxHeight: 1024,
                quality: 0.8
            });
            
            // Base64 변환
            const base64 = await convertToBase64(compressedImage);
            
            setUploadedImages(prev => [...prev, {
                id: Date.now() + Math.random(),
                name: file.name,
                data: base64,
                size: compressedImage.size
            }]);
        } catch (error) {
            console.error('이미지 업로드 실패:', error);
            showError(`이미지 업로드 실패: ${file.name}`);
        }
    }
}, []);

// 이미지 압축 함수
async function compressImage(file, options) {
    return new Promise((resolve) => {
        const canvas = document.createElement('canvas');
        const ctx = canvas.getContext('2d');
        const img = new Image();
        
        img.onload = () => {
            // 비율 유지하며 크기 조정
            const { width, height } = calculateDimensions(
                img.width, img.height, options.maxWidth, options.maxHeight
            );
            
            canvas.width = width;
            canvas.height = height;
            
            // 고품질 리샘플링
            ctx.imageSmoothingEnabled = true;
            ctx.imageSmoothingQuality = 'high';
            ctx.drawImage(img, 0, 0, width, height);
            
            canvas.toBlob(resolve, 'image/jpeg', options.quality);
        };
        
        img.src = URL.createObjectURL(file);
    });
}
```

#### 메시지 히스토리 관리 시스템
```javascript
// 로컬 스토리지 기반 메시지 저장 (압축 포함)
const [messageHistory, setMessageHistory] = useState([]);
const MAX_HISTORY_SIZE = 5000; // 최대 메시지 수
const COMPRESSION_THRESHOLD = 1000; // 압축 시작 임계값

const saveMessageHistory = useCallback(async (messages) => {
    try {
        let dataToStore = messages;
        
        // 크기가 임계값을 초과하면 압축
        if (messages.length > COMPRESSION_THRESHOLD) {
            dataToStore = await compressMessages(messages);
        }
        
        // 최대 크기 제한
        if (dataToStore.length > MAX_HISTORY_SIZE) {
            dataToStore = dataToStore.slice(-MAX_HISTORY_SIZE);
        }
        
        const storageKey = `chat_history_${currentProject?.id}_${currentSession?.id}`;
        localStorage.setItem(storageKey, JSON.stringify({
            messages: dataToStore,
            compressed: dataToStore !== messages,
            timestamp: Date.now()
        }));
    } catch (error) {
        console.error('메시지 저장 실패:', error);
    }
}, [currentProject, currentSession]);

// 메시지 압축 함수
async function compressMessages(messages) {
    // 오래된 메시지의 내용을 요약하거나 제거
    const recentMessages = messages.slice(-500); // 최근 500개 메시지 유지
    const oldMessages = messages.slice(0, -500);
    
    // 중요한 메시지만 유지 (파일 변경, 도구 사용 등)
    const importantOldMessages = oldMessages.filter(msg => 
        msg.tools_used?.length > 0 || 
        msg.content.includes('파일') ||
        msg.role === 'user'
    );
    
    return [...importantOldMessages, ...recentMessages];
}
```

#### 마크다운 렌더링 및 코드 하이라이팅
```javascript
// React Markdown과 Prism.js 통합
const MarkdownRenderer = memo(({ content }) => {
    return (
        <ReactMarkdown
            components={{
                code: ({ node, inline, className, children, ...props }) => {
                    const match = /language-(\w+)/.exec(className || '');
                    const language = match ? match[1] : '';
                    
                    if (!inline && language) {
                        return (
                            <SyntaxHighlighter
                                style={isDarkMode ? vscDarkPlus : oneLight}
                                language={language}
                                PreTag="div"
                                customStyle={{
                                    margin: 0,
                                    borderRadius: '0.375rem'
                                }}
                            >
                                {String(children).replace(/\n$/, '')}
                            </SyntaxHighlighter>
                        );
                    }
                    
                    return (
                        <code className={className} {...props}>
                            {children}
                        </code>
                    );
                }
            }}
        >
            {content}
        </ReactMarkdown>
    );
});
```

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

**상세 동작 방식**:

#### xterm.js 터미널 초기화 및 설정
```javascript
// 터미널 인스턴스 생성 및 설정
useEffect(() => {
    if (terminalRef.current && !terminal) {
        // xterm.js 터미널 생성
        const term = new Terminal({
            cursorBlink: true,
            cursorStyle: 'block',
            fontSize: 14,
            fontFamily: 'Fira Code, Monaco, "Lucida Console", monospace',
            theme: {
                background: isDarkMode ? '#1a1a1a' : '#ffffff',
                foreground: isDarkMode ? '#ffffff' : '#000000',
                cursor: isDarkMode ? '#ffffff' : '#000000',
                selection: isDarkMode ? '#404040' : '#b4d5fe'
            },
            cols: 80,
            rows: 24,
            scrollback: 1000
        });
        
        // WebGL 애드온으로 성능 향상
        const webglAddon = new WebglAddon();
        term.loadAddon(webglAddon);
        
        // 검색 기능 애드온
        const searchAddon = new SearchAddon();
        term.loadAddon(searchAddon);
        
        // 웹 링크 클릭 지원
        const webLinksAddon = new WebLinksAddon();
        term.loadAddon(webLinksAddon);
        
        // DOM 요소에 연결
        term.open(terminalRef.current);
        setTerminal(term);
        
        // 서버에 터미널 세션 요청
        requestNewTerminalSession(projectPath);
    }
}, [terminalRef.current, isDarkMode, projectPath]);
```

#### 실시간 양방향 통신 시스템
```javascript
// WebSocket을 통한 터미널 입출력 처리
useEffect(() => {
    if (terminal && ws) {
        // 터미널 입력을 서버로 전송
        const handleTerminalInput = (data) => {
            ws.send(JSON.stringify({
                type: 'terminal_input',
                sessionId: terminalSessionId,
                data: data
            }));
        };
        
        // 사용자 입력 이벤트 리스너 등록
        terminal.onData(handleTerminalInput);
        
        // 키 입력 처리 (단축키 등)
        terminal.onKey(({ key, domEvent }) => {
            // Ctrl+C 처리
            if (domEvent.ctrlKey && domEvent.key === 'c') {
                ws.send(JSON.stringify({
                    type: 'terminal_signal',
                    sessionId: terminalSessionId,
                    signal: 'SIGINT'
                }));
            }
            
            // Ctrl+V 붙여넣기 처리
            if (domEvent.ctrlKey && domEvent.key === 'v') {
                domEvent.preventDefault();
                handlePaste();
            }
        });
        
        return () => {
            terminal.dispose();
        };
    }
}, [terminal, ws, terminalSessionId]);

// 서버에서 오는 터미널 출력 처리
useEffect(() => {
    if (wsMessage?.type === 'terminal_output' && 
        wsMessage.sessionId === terminalSessionId) {
        
        // ANSI 이스케이프 시퀀스 처리
        const processedOutput = processAnsiSequences(wsMessage.data);
        terminal?.write(processedOutput);
    }
    
    if (wsMessage?.type === 'terminal_resize') {
        terminal?.resize(wsMessage.cols, wsMessage.rows);
    }
}, [wsMessage, terminal, terminalSessionId]);
```

#### 클립보드 및 선택 텍스트 처리
```javascript
// 터미널 내 텍스트 선택 및 복사 기능
const handleCopy = useCallback(() => {
    if (terminal?.hasSelection()) {
        const selection = terminal.getSelection();
        navigator.clipboard.writeText(selection);
        
        // 복사 완료 표시
        showNotification('클립보드에 복사됨');
    }
}, [terminal]);

const handlePaste = useCallback(async () => {
    try {
        const text = await navigator.clipboard.readText();
        
        // 여러 줄 텍스트 처리
        if (text.includes('\n')) {
            const lines = text.split('\n');
            for (let i = 0; i < lines.length; i++) {
                terminal?.write(lines[i]);
                if (i < lines.length - 1) {
                    terminal?.write('\r\n');
                }
            }
        } else {
            terminal?.write(text);
        }
    } catch (error) {
        console.error('붙여넣기 실패:', error);
    }
}, [terminal]);
```

#### 터미널 테마 및 크기 조정
```javascript
// 동적 테마 변경
useEffect(() => {
    if (terminal) {
        terminal.options.theme = {
            background: isDarkMode ? '#1a1a1a' : '#ffffff',
            foreground: isDarkMode ? '#ffffff' : '#000000',
            cursor: isDarkMode ? '#ffffff' : '#000000',
            selection: isDarkMode ? '#404040' : '#b4d5fe',
            black: isDarkMode ? '#2e3436' : '#000000',
            red: '#cc0000',
            green: '#4e9a06',
            yellow: '#c4a000',
            blue: '#3465a4',
            magenta: '#75507b',
            cyan: '#06989a',
            white: isDarkMode ? '#d3d7cf' : '#ffffff'
        };
    }
}, [terminal, isDarkMode]);

// 터미널 크기 자동 조정
useEffect(() => {
    const resizeTerminal = () => {
        if (terminal && terminalRef.current) {
            const rect = terminalRef.current.getBoundingClientRect();
            const cols = Math.floor(rect.width / 9); // 문자 폭 추정
            const rows = Math.floor(rect.height / 17); // 문자 높이 추정
            
            terminal.resize(cols, rows);
            
            // 서버에 크기 변경 알림
            ws?.send(JSON.stringify({
                type: 'terminal_resize',
                sessionId: terminalSessionId,
                cols: cols,
                rows: rows
            }));
        }
    };
    
    window.addEventListener('resize', resizeTerminal);
    resizeTerminal(); // 초기 크기 설정
    
    return () => window.removeEventListener('resize', resizeTerminal);
}, [terminal, ws, terminalSessionId]);
```

#### 다중 터미널 세션 관리
```javascript
// 여러 터미널 탭 지원
const [terminalSessions, setTerminalSessions] = useState([]);
const [activeSessionId, setActiveSessionId] = useState(null);

const createNewTerminalSession = useCallback(() => {
    const sessionId = `terminal_${Date.now()}`;
    
    setTerminalSessions(prev => [...prev, {
        id: sessionId,
        name: `터미널 ${prev.length + 1}`,
        created: Date.now(),
        active: true
    }]);
    
    setActiveSessionId(sessionId);
    
    // 서버에 새 세션 요청
    ws?.send(JSON.stringify({
        type: 'create_terminal_session',
        sessionId: sessionId,
        workingDirectory: projectPath
    }));
}, [ws, projectPath]);

const closeTerminalSession = useCallback((sessionId) => {
    // 서버에 세션 종료 요청
    ws?.send(JSON.stringify({
        type: 'close_terminal_session',
        sessionId: sessionId
    }));
    
    setTerminalSessions(prev => 
        prev.filter(session => session.id !== sessionId)
    );
    
    // 활성 세션이 닫힌 경우 다른 세션으로 전환
    if (activeSessionId === sessionId) {
        const remainingSessions = terminalSessions.filter(s => s.id !== sessionId);
        setActiveSessionId(remainingSessions[0]?.id || null);
    }
}, [ws, activeSessionId, terminalSessions]);
```

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

**상세 데이터 흐름**:
```
1. 사용자 메시지 입력
   ↓
2. ChatInterface에서 입력 검증 및 이미지 처리
   ↓
3. WebSocket을 통해 서버로 전송
   ↓
4. server/claude-cli.js에서 Claude CLI 프로세스 스폰
   ↓
5. Claude CLI 실행 및 스트림 데이터 생성
   ↓
6. 실시간 스트림 파싱 및 WebSocket으로 클라이언트 전송
   ↓
7. ChatInterface에서 실시간 메시지 업데이트
   ↓
8. 완료 시 세션 보호 해제 및 히스토리 저장
```

### 2. 실시간 업데이트 시스템

#### 파일 시스템 감시 메커니즘
```javascript
// chokidar를 통한 파일 변경 감지
const watcher = chokidar.watch(claudeProjectsPath, {
    ignored: [
        /node_modules/,
        /\.git/,
        /\.DS_Store/,
        /\.vscode/
    ],
    persistent: true,
    ignoreInitial: true,
    followSymlinks: false
});

// 이벤트 타입별 처리
watcher.on('all', (event, path) => {
    const eventData = {
        type: 'FILE_SYSTEM_EVENT',
        event: event, // add, change, unlink, addDir, unlinkDir
        path: path,
        timestamp: Date.now(),
        projectId: extractProjectIdFromPath(path)
    };
    
    // 세션 보호 확인
    if (!isProtectedProject(eventData.projectId)) {
        broadcastToClients(eventData);
    }
});
```

#### 세션 보호 메커니즘 (핵심 특징)
```javascript
// 서버 측 세션 보호 상태 관리
const protectedSessions = new Map(); // sessionId -> projectId

function activateSessionProtection(sessionId, projectId) {
    protectedSessions.set(sessionId, projectId);
    
    // 보호된 프로젝트 목록 업데이트
    updateProtectedProjects();
    
    console.log(`세션 보호 활성화: ${sessionId} -> ${projectId}`);
}

function deactivateSessionProtection(sessionId) {
    const projectId = protectedSessions.get(sessionId);
    protectedSessions.delete(sessionId);
    
    updateProtectedProjects();
    
    console.log(`세션 보호 해제: ${sessionId} -> ${projectId}`);
}

// 프로젝트 보호 상태 확인
function isProtectedProject(projectId) {
    return Array.from(protectedSessions.values()).includes(projectId);
}
```

### 3. 상태 관리 계층

#### 계층별 상태 관리 구조
```javascript
// 1. 로컬 컴포넌트 상태 (useState, useReducer)
const ChatInterface = () => {
    const [messages, setMessages] = useState([]);
    const [isStreaming, setIsStreaming] = useState(false);
    // 컴포넌트별 UI 상태
};

// 2. 글로벌 애플리케이션 상태 (App.jsx)
const App = () => {
    const [globalState, setGlobalState] = useState({
        projects: [],
        currentProject: null,
        currentSession: null,
        activeSessions: new Set()
    });
};

// 3. 영구 저장 상태 (localStorage, IndexedDB)
const persistentStorage = {
    // 사용자 설정
    settings: localStorage.getItem('user_settings'),
    
    // 메시지 히스토리 (압축)
    messageHistory: localStorage.getItem('chat_history'),
    
    // 인증 토큰
    authToken: localStorage.getItem('auth_token')
};

// 4. 서버 상태 (SQLite, 파일 시스템)
const serverState = {
    // 사용자 인증 정보
    users: 'SQLite 데이터베이스',
    
    // 프로젝트 메타데이터
    projects: 'Claude 설정 파일',
    
    // 세션 데이터
    sessions: 'JSONL 파일'
};
```

#### 상태 동기화 전략
```javascript
// 상태 변경 전파 시스템
class StateManager {
    constructor() {
        this.subscribers = new Map();
        this.state = {};
    }
    
    // 상태 변경 구독
    subscribe(key, callback) {
        if (!this.subscribers.has(key)) {
            this.subscribers.set(key, new Set());
        }
        this.subscribers.get(key).add(callback);
    }
    
    // 상태 업데이트 및 구독자 알림
    updateState(key, value) {
        const oldValue = this.state[key];
        this.state[key] = value;
        
        // 값이 실제로 변경된 경우에만 알림
        if (oldValue !== value) {
            this.notifySubscribers(key, value, oldValue);
        }
    }
    
    // 구독자들에게 변경 알림
    notifySubscribers(key, newValue, oldValue) {
        const callbacks = this.subscribers.get(key);
        if (callbacks) {
            callbacks.forEach(callback => {
                callback(newValue, oldValue);
            });
        }
    }
}
```

---

## 보안 및 성능 최적화

### 1. 보안 기능

#### JWT 기반 인증 시스템
```javascript
// 토큰 생성 및 검증 (server/routes/auth.js)
const jwt = require('jsonwebtoken');

// 로그인 시 토큰 생성
app.post('/api/auth/login', async (req, res) => {
    const { username, password } = req.body;
    
    // 사용자 인증 확인
    const user = await authenticateUser(username, password);
    if (!user) {
        return res.status(401).json({ error: '인증 실패' });
    }
    
    // JWT 토큰 생성
    const token = jwt.sign(
        { 
            userId: user.id, 
            username: user.username,
            exp: Math.floor(Date.now() / 1000) + (24 * 60 * 60) // 24시간
        },
        process.env.JWT_SECRET,
        { algorithm: 'HS256' }
    );
    
    res.json({ token, user: { id: user.id, username: user.username } });
});

// 토큰 검증 미들웨어
function verifyToken(req, res, next) {
    const token = req.headers.authorization?.replace('Bearer ', '');
    
    if (!token) {
        return res.status(401).json({ error: '토큰이 필요합니다' });
    }
    
    try {
        const decoded = jwt.verify(token, process.env.JWT_SECRET);
        req.user = decoded;
        next();
    } catch (error) {
        return res.status(401).json({ error: '유효하지 않은 토큰' });
    }
}
```

#### 도구 권한 세분화 제어
```javascript
// 도구별 권한 관리 시스템
const ToolsPermissionManager = {
    // 기본적으로 모든 도구 비활성화
    defaultPermissions: {
        file_operations: false,
        shell_commands: false,
        git_operations: false,
        network_access: false,
        system_info: false
    },
    
    // 사용자별 권한 설정
    userPermissions: new Map(),
    
    // 권한 확인
    checkPermission(userId, toolName) {
        const userPerms = this.userPermissions.get(userId) || this.defaultPermissions;
        return userPerms[toolName] || false;
    },
    
    // 권한 업데이트
    updatePermissions(userId, permissions) {
        // 보안 검증: 위험한 권한 조합 차단
        if (permissions.shell_commands && permissions.network_access) {
            throw new Error('보안상 위험한 권한 조합입니다');
        }
        
        this.userPermissions.set(userId, {
            ...this.defaultPermissions,
            ...permissions
        });
    }
};

// Claude CLI 실행 시 권한 검증
function spawnClaudeWithPermissions(userId, tools) {
    const allowedTools = tools.filter(tool => 
        ToolsPermissionManager.checkPermission(userId, tool)
    );
    
    if (allowedTools.length !== tools.length) {
        console.warn(`일부 도구가 권한 없음으로 차단됨: ${userId}`);
    }
    
    return spawnClaude(allowedTools);
}
```

#### API 보안 및 요청 검증
```javascript
// 요청 검증 및 보안 헤더
app.use((req, res, next) => {
    // CORS 설정
    res.header('Access-Control-Allow-Origin', process.env.FRONTEND_URL || 'http://localhost:5173');
    res.header('Access-Control-Allow-Credentials', 'true');
    res.header('Access-Control-Allow-Headers', 'Origin, X-Requested-With, Content-Type, Accept, Authorization');
    
    // 보안 헤더
    res.header('X-Content-Type-Options', 'nosniff');
    res.header('X-Frame-Options', 'DENY');
    res.header('X-XSS-Protection', '1; mode=block');
    
    next();
});

// 요청 크기 제한
app.use(express.json({ 
    limit: '10mb',
    verify: (req, res, buf) => {
        // 악성 페이로드 검증
        if (buf.length > 10 * 1024 * 1024) {
            throw new Error('요청 크기가 너무 큽니다');
        }
    }
}));

// 파일 업로드 보안
const upload = multer({
    limits: {
        fileSize: 5 * 1024 * 1024, // 5MB 제한
        files: 5 // 최대 5개 파일
    },
    fileFilter: (req, file, cb) => {
        // 허용된 MIME 타입만 업로드 가능
        const allowedTypes = ['image/jpeg', 'image/png', 'image/gif', 'image/webp'];
        if (allowedTypes.includes(file.mimetype)) {
            cb(null, true);
        } else {
            cb(new Error('허용되지 않는 파일 형식입니다'));
        }
    }
});
```

### 2. 성능 최적화

#### React 렌더링 최적화
```javascript
// React.memo를 사용한 불필요한 리렌더링 방지
const ChatMessage = memo(({ message, isStreaming }) => {
    return (
        <div className="message">
            <MarkdownRenderer content={message.content} />
        </div>
    );
}, (prevProps, nextProps) => {
    // 커스텀 비교 함수로 정확한 변경 감지
    return prevProps.message.content === nextProps.message.content &&
           prevProps.isStreaming === nextProps.isStreaming;
});

// useMemo를 사용한 무거운 계산 결과 캐싱
const ProcessedMessages = ({ messages }) => {
    const processedMessages = useMemo(() => {
        return messages.map(msg => ({
            ...msg,
            processed: processMessageContent(msg.content),
            wordCount: msg.content.split(' ').length
        }));
    }, [messages]);
    
    return processedMessages.map(msg => 
        <ChatMessage key={msg.id} message={msg} />
    );
};

// useCallback을 사용한 함수 참조 안정화
const ChatInterface = () => {
    const handleSendMessage = useCallback((content) => {
        // 메시지 전송 로직
    }, [currentSession, ws]);
    
    const handleImageUpload = useCallback((files) => {
        // 이미지 업로드 로직
    }, []);
};
```

#### 메시지 히스토리 압축 및 가상화
```javascript
// 메시지 가상화 (react-window 사용)
import { FixedSizeList as List } from 'react-window';

const VirtualizedMessageList = ({ messages }) => {
    const Row = ({ index, style }) => (
        <div style={style}>
            <ChatMessage message={messages[index]} />
        </div>
    );
    
    return (
        <List
            height={600}
            itemCount={messages.length}
            itemSize={100}
            overscanCount={5}
        >
            {Row}
        </List>
    );
};

// 메시지 압축 알고리즘
class MessageCompressor {
    static compress(messages) {
        return {
            recent: messages.slice(-100), // 최근 100개 메시지는 원본 유지
            compressed: this.compressOldMessages(messages.slice(0, -100))
        };
    }
    
    static compressOldMessages(messages) {
        return messages.reduce((acc, msg, index) => {
            // 중요한 메시지만 유지
            if (this.isImportantMessage(msg)) {
                acc.push(msg);
            } else if (index % 10 === 0) {
                // 10개마다 1개씩 샘플링
                acc.push(this.summarizeMessage(msg));
            }
            return acc;
        }, []);
    }
    
    static isImportantMessage(message) {
        return message.tools_used?.length > 0 ||
               message.content.includes('오류') ||
               message.content.includes('완료') ||
               message.role === 'user';
    }
}
```

#### 네트워크 및 캐싱 최적화
```javascript
// API 응답 캐싱
const apiCache = new Map();
const CACHE_EXPIRY = 5 * 60 * 1000; // 5분

async function cachedFetch(url, options = {}) {
    const cacheKey = `${url}:${JSON.stringify(options)}`;
    const cached = apiCache.get(cacheKey);
    
    if (cached && Date.now() - cached.timestamp < CACHE_EXPIRY) {
        return cached.data;
    }
    
    const response = await fetch(url, options);
    const data = await response.json();
    
    apiCache.set(cacheKey, {
        data,
        timestamp: Date.now()
    });
    
    return data;
}

// WebSocket 메시지 디바운싱
const messageQueue = [];
let processingTimeout = null;

function queueMessage(message) {
    messageQueue.push(message);
    
    if (processingTimeout) {
        clearTimeout(processingTimeout);
    }
    
    processingTimeout = setTimeout(() => {
        processBatchedMessages(messageQueue.splice(0));
    }, 16); // 60fps 타이밍
}

function processBatchedMessages(messages) {
    // 중복 메시지 제거
    const uniqueMessages = messages.filter((msg, index, arr) => 
        arr.findIndex(m => m.id === msg.id) === index
    );
    
    // 일괄 UI 업데이트
    updateUI(uniqueMessages);
}
```

#### 프로젝트 데이터 캐싱
```javascript
// 프로젝트 디렉토리 캐싱 시스템
class ProjectCache {
    constructor() {
        this.cache = new Map();
        this.lastScan = 0;
        this.SCAN_INTERVAL = 30000; // 30초
    }
    
    async getProjects(forceRefresh = false) {
        const now = Date.now();
        
        if (!forceRefresh && 
            this.cache.has('projects') && 
            now - this.lastScan < this.SCAN_INTERVAL) {
            return this.cache.get('projects');
        }
        
        const projects = await this.scanProjects();
        this.cache.set('projects', projects);
        this.lastScan = now;
        
        return projects;
    }
    
    invalidateProject(projectId) {
        // 특정 프로젝트 캐시만 무효화
        const projects = this.cache.get('projects') || [];
        const updatedProjects = projects.filter(p => p.id !== projectId);
        this.cache.set('projects', updatedProjects);
    }
}
```

---

## 결론

Claude Code UI는 복잡한 AI 도구를 직관적인 웹 인터페이스로 추상화한 정교한 시스템입니다. 특히 세션 보호 시스템과 실시간 통신, 그리고 완전한 터미널 통합을 통해 개발자에게 원활한 AI 지원 코딩 환경을 제공합니다. 각 파일은 명확한 책임을 가지고 있으며, 모듈화된 아키텍처를 통해 유지보수성과 확장성을 보장합니다.