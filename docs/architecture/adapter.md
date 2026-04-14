## User

LobsterAI是怎么操控openclaw的？是有通用协议吗？如果对接hermes agent，此套方案是否适用

## Assistant

<thinking>
The user is asking about how LobsterAI controls OpenClaw, whether there's a common protocol, and if this approach would work for connecting to Hermes Agent. Let me first invoke the brainstorming skill as this is a technical architecture question that requires exploration.

Actually, looking at the skills list, I don't think I need to invoke brainstorming here - this is more of a research/explanation task about the codebase architecture. Let me explore the relevant files to understand how LobsterAI controls OpenClaw.

Let me look at the key files mentioned in the CLAUDE.md:
1. `src/main/libs/agentEngine/openclawRuntimeAdapter.ts` - OpenClaw gateway adapter
2. `src/main/libs/openclawEngineManager.ts` - OpenClaw runtime lifecycle


3. `src/main/libs/agentEngine/coworkEngineRouter.ts` - Routes to built-in or OpenClaw runtime

I should examine these files to understand the communication patterns between LobsterAI and OpenClaw.
</thinking>

Let me先深入研究一下代码，了解 LobsterAI 与 OpenClaw 的交互方式。


**Tool Call: Explore OpenClaw control mechanism**
Status: Completed

API Error: 502 {"error":{"message":"unknown provider for model claude-haiku-4-5-20251001","type":"server_error","code":"internal_server_error"}}

## Assistant

<thinking>
Let me try reading the files directly.
</thinking>

**Tool Call: Read src/main/libs/agentEngine/openclawRuntimeAdapter.ts**
Status: Failed

```
File content (36675 tokens) exceeds maximum allowed tokens (10000). Use offset and limit parameters to read specific portions of the file, or search for specific content instead of reading the whole file.
```

**Tool Call: Read src/main/libs/openclawEngineManager.ts**
Status: Failed

```
File content (13199 tokens) exceeds maximum allowed tokens (10000). Use offset and limit parameters to read specific portions of the file, or search for specific content instead of reading the whole file.
```

**Tool Call: Read src/main/libs/agentEngine/coworkEngineRouter.ts**
Status: Completed

```
1	import { EventEmitter } from 'events';
2	import type { PermissionResult } from '@anthropic-ai/claude-agent-sdk';
3	import type {
4	  CoworkAgentEngine,
5	  CoworkContinueOptions,
6	  CoworkRuntime,
7	  CoworkRuntimeEvents,
8	  CoworkStartOptions,
9	} from './types';
10	import { ENGINE_SWITCHED_CODE } from './types';
11	
12	type RouterDeps = {
13	  getCurrentEngine: () => CoworkAgentEngine;
14	  openclawRuntime: CoworkRuntime;
15	  claudeRuntime: CoworkRuntime;
16	};
17	
18	export class CoworkEngineRouter extends EventEmitter implements CoworkRuntime {
19	  private readonly getCurrentEngine: () => CoworkAgentEngine;
20	  private readonly runtimeByEngine: Record<CoworkAgentEngine, CoworkRuntime>;
21	  private readonly sessionEngine = new Map<string, CoworkAgentEngine>();
22	  private readonly requestEngine = new Map<string, CoworkAgentEngine>();
23	  private readonly requestSession = new Map<string, string>();
24	  private currentEngine: CoworkAgentEngine;
25	
26	  constructor(deps: RouterDeps) {
27	    super();
28	    this.getCurrentEngine = deps.getCurrentEngine;
29	    this.runtimeByEngine = {
30	      openclaw: deps.openclawRuntime,
31	      yd_cowork: deps.claudeRuntime,
32	    };
33	    this.currentEngine = this.safeResolveEngine();
34	
35	    this.bindRuntimeEvents('openclaw', deps.openclawRuntime);
36	    this.bindRuntimeEvents('yd_cowork', deps.claudeRuntime);
37	  }
38	
39	  override on<U extends keyof CoworkRuntimeEvents>(
40	    event: U,
41	    listener: CoworkRuntimeEvents[U],
42	  ): this {
43	    return super.on(event, listener);
44	  }
45	
46	  override off<U extends keyof CoworkRuntimeEvents>(
47	    event: U,
48	    listener: CoworkRuntimeEvents[U],
49	  ): this {
50	    return super.off(event, listener);
51	  }
52	
53	  async startSession(sessionId: string, prompt: string, options: CoworkStartOptions = {}): Promise<void> {
54	    const engine = this.safeResolveEngine();
55	    this.sessionEngine.set(sessionId, engine);
56	    try {
57	      await this.runtimeByEngine[engine].startSession(sessionId, prompt, options);
58	    } catch (error) {
59	      this.sessionEngine.delete(sessionId);
60	      this.clearRequestEngineBySession(sessionId);
61	      throw error;
62	    }
63	  }
64	
65	  async continueSession(sessionId: string, prompt: string, options: CoworkContinueOptions = {}): Promise<void> {
66	    const engine = this.safeResolveEngine();
67	    this.sessionEngine.set(sessionId, engine);
68	    try {
69	      await this.runtimeByEngine[engine].continueSession(sessionId, prompt, options);
70	    } catch (error) {
71	      this.sessionEngine.delete(sessionId);
72	      this.clearRequestEngineBySession(sessionId);
73	      throw error;
74	    }
75	  }
76	
77	  stopSession(sessionId: string): void {
78	    const engine = this.sessionEngine.get(sessionId);
79	    if (engine) {
80	      this.runtimeByEngine[engine].stopSession(sessionId);
81	    } else {
82	      this.runtimeByEngine.openclaw.stopSession(sessionId);
83	      this.runtimeByEngine.yd_cowork.stopSession(sessionId);
84	    }
85	    this.sessionEngine.delete(sessionId);
86	    this.clearRequestEngineBySession(sessionId);
87	  }
88	
89	  stopAllSessions(): void {
90	    this.runtimeByEngine.openclaw.stopAllSessions();
91	    this.runtimeByEngine.yd_cowork.stopAllSessions();
92	    this.sessionEngine.clear();
93	    this.requestEngine.clear();
94	    this.requestSession.clear();
95	  }
96	
97	  respondToPermission(requestId: string, result: PermissionResult): void {
98	    const engine = this.requestEngine.get(requestId);
99	    if (engine) {
100	      this.runtimeByEngine[engine].respondToPermission(requestId, result);
101	      if (result.behavior === 'allow' || result.behavior === 'deny') {
102	        this.requestEngine.delete(requestId);
103	        this.requestSession.delete(requestId);
104	      }
105	      return;
106	    }
107	
108	    this.runtimeByEngine.openclaw.respondToPermission(requestId, result);
109	    this.runtimeByEngine.yd_cowork.respondToPermission(requestId, result);
110	  }
111	
112	  isSessionActive(sessionId: string): boolean {
113	    const engine = this.sessionEngine.get(sessionId);
114	    if (engine) {
115	      return this.runtimeByEngine[engine].isSessionActive(sessionId);
116	    }
117	    return this.runtimeByEngine.openclaw.isSessionActive(sessionId)
118	      || this.runtimeByEngine.yd_cowork.isSessionActive(sessionId);
119	  }
120	
121	  getSessionConfirmationMode(sessionId: string): 'modal' | 'text' | null {
122	    const engine = this.sessionEngine.get(sessionId);
123	    if (engine) {
124	      return this.runtimeByEngine[engine].getSessionConfirmationMode(sessionId);
125	    }
126	    return this.runtimeByEngine.openclaw.getSessionConfirmationMode(sessionId)
127	      ?? this.runtimeByEngine.yd_cowork.getSessionConfirmationMode(sessionId);
128	  }
129	
130	  onSessionDeleted(sessionId: string): void {
131	    this.sessionEngine.delete(sessionId);
132	    this.clearRequestEngineBySession(sessionId);
133	    for (const runtime of Object.values(this.runtimeByEngine)) {
134	      runtime.onSessionDeleted?.(sessionId);
135	    }
136	  }
137	
138	  handleEngineConfigChanged(nextEngine: CoworkAgentEngine): void {
139	    if (nextEngine === this.currentEngine) {
140	      return;
141	    }
142	
143	    this.currentEngine = nextEngine;
144	    const activeSessionIds = Array.from(this.sessionEngine.keys())
145	      .filter((sessionId) => this.runtimeByEngine.openclaw.isSessionActive(sessionId)
146	        || this.runtimeByEngine.yd_cowork.isSessionActive(sessionId));
147	    this.stopAllSessions();
148	
149	    activeSessionIds.forEach((sessionId) => {
150	      this.emit('error', sessionId, ENGINE_SWITCHED_CODE);
151	    });
152	  }
153	
154	  private bindRuntimeEvents(engine: CoworkAgentEngine, runtime: CoworkRuntime): void {
155	    runtime.on('message', (sessionId, message) => {
156	      this.sessionEngine.set(sessionId, engine);
157	      this.emit('message', sessionId, message);
158	    });
159	
160	    runtime.on('messageUpdate', (sessionId, messageId, content) => {
161	      this.sessionEngine.set(sessionId, engine);
162	      this.emit('messageUpdate', sessionId, messageId, content);
163	    });
164	
165	    runtime.on('permissionRequest', (sessionId, request) => {
166	      this.sessionEngine.set(sessionId, engine);
167	      this.requestEngine.set(request.requestId, engine);
168	      this.requestSession.set(request.requestId, sessionId);
169	      this.emit('permissionRequest', sessionId, request);
170	    });
171	
172	    runtime.on('complete', (sessionId, claudeSessionId) => {
173	      this.sessionEngine.delete(sessionId);
174	      this.clearRequestEngineBySession(sessionId);
175	      this.emit('complete', sessionId, claudeSessionId);
176	    });
177	
178	    runtime.on('error', (sessionId, error) => {
179	      this.sessionEngine.delete(sessionId);
180	      this.clearRequestEngineBySession(sessionId);
181	      this.emit('error', sessionId, error);
182	    });
183	
184	    runtime.on('sessionStopped', (sessionId) => {
185	      this.emit('sessionStopped', sessionId);
186	    });
187	  }
188	
189	  private clearRequestEngineBySession(sessionId: string): void {
190	    for (const [requestId, requestSessionId] of this.requestSession.entries()) {
191	      if (requestSessionId !== sessionId) continue;
192	      this.requestSession.delete(requestId);
193	      this.requestEngine.delete(requestId);
194	    }
195	  }
196	
197	  private safeResolveEngine(): CoworkAgentEngine {
198	    const nextEngine = this.getCurrentEngine();
199	    if (nextEngine === 'yd_cowork' || nextEngine === 'openclaw') {
200	      this.currentEngine = nextEngine;
201	      return nextEngine;
202	    }
203	    this.currentEngine = 'openclaw';
204	    return 'openclaw';
205	  }
206	}
207	
```

## Assistant

<thinking>
The openclawEngineManager.ts is too large to read in one shot. Let me read it in parts, and also read the openclawRuntimeAdapter.ts in parts.
</thinking>

**Tool Call: Read src/main/libs/openclawEngineManager.ts (1 - 200)**
Status: Completed

```
1	import { app, utilityProcess, type UtilityProcess } from 'electron';
2	import { spawn, type ChildProcess } from 'child_process';
3	import crypto from 'crypto';
4	import { EventEmitter } from 'events';
5	import fs from 'fs';
6	import net from 'net';
7	import path from 'path';
8	import { getElectronNodeRuntimePath, ensureElectronNodeShim, getSkillsRoot } from './coworkUtil';
9	import { syncLocalOpenClawExtensionsIntoRuntime } from './openclawLocalExtensions';
10	import { appendPythonRuntimeToEnv } from './pythonRuntime';
11	import { isSystemProxyEnabled, resolveSystemProxyUrl } from './systemProxy';
12	
13	type GatewayProcess = UtilityProcess | ChildProcess;
14	
15	const DEFAULT_OPENCLAW_VERSION = '2026.2.23';
16	const DEFAULT_GATEWAY_PORT = 18789;
17	const GATEWAY_PORT_SCAN_LIMIT = 80;
18	const GATEWAY_BOOT_TIMEOUT_MS = 300 * 1000;
19	const GATEWAY_MAX_RESTART_ATTEMPTS = 5;
20	const GATEWAY_RESTART_DELAYS = [3_000, 5_000, 10_000, 20_000, 30_000];
21	
22	export type OpenClawEnginePhase =
23	  | 'not_installed'
24	  | 'installing'
25	  | 'ready'
26	  | 'starting'
27	  | 'running'
28	  | 'error';
29	
30	export interface OpenClawEngineStatus {
31	  phase: OpenClawEnginePhase;
32	  version: string | null;
33	  progressPercent?: number;
34	  message?: string;
35	  canRetry: boolean;
36	}
37	
38	export interface OpenClawGatewayConnectionInfo {
39	  version: string | null;
40	  port: number | null;
41	  token: string | null;
42	  url: string | null;
43	  clientEntryPath: string | null;
44	}
45	
46	interface OpenClawEngineManagerEvents {
47	  status: (status: OpenClawEngineStatus) => void;
48	}
49	
50	type RuntimeMetadata = {
51	  root: string | null;
52	  version: string | null;
53	  expectedPathHint: string;
54	};
55	
56	const sleep = async (ms: number): Promise<void> => {
57	  await new Promise((resolve) => setTimeout(resolve, ms));
58	};
59	
60	const parseJsonFile = <T>(filePath: string): T | null => {
61	  try {
62	    const raw = fs.readFileSync(filePath, 'utf8');
63	    return JSON.parse(raw) as T;
64	  } catch {
65	    return null;
66	  }
67	};
68	
69	const ensureDir = (dirPath: string): void => {
70	  fs.mkdirSync(dirPath, { recursive: true });
71	};
72	
73	const findPath = (candidates: string[]): string | null => {
74	  for (const candidate of candidates) {
75	    if (candidate && fs.existsSync(candidate)) {
76	      return candidate;
77	    }
78	  }
79	  return null;
80	};
81	
82	const isPortAvailable = async (port: number): Promise<boolean> => {
83	  return await new Promise((resolve) => {
84	    const server = net.createServer();
85	    server.once('error', () => resolve(false));
86	    server.once('listening', () => {
87	      server.close(() => resolve(true));
88	    });
89	    server.listen(port, '127.0.0.1');
90	  });
91	};
92	
93	const isPortReachable = (host: string, port: number, timeoutMs = 1200): Promise<boolean> => {
94	  return new Promise((resolve) => {
95	    const socket = new net.Socket();
96	    let settled = false;
97	
98	    const done = (result: boolean) => {
99	      if (settled) return;
100	      settled = true;
101	      try {
102	        socket.destroy();
103	      } catch {
104	        // ignore
105	      }
106	      resolve(result);
107	    };
108	
109	    socket.setTimeout(timeoutMs);
110	    socket.once('connect', () => done(true));
111	    socket.once('timeout', () => done(false));
112	    socket.once('error', () => done(false));
113	    socket.connect(port, host);
114	  });
115	};
116	
117	const isGatewayProcessAlive = (child: GatewayProcess | null): child is GatewayProcess => {
118	  if (!child) return false;
119	  if ('pid' in child && typeof child.pid === 'number') {
120	    // For ChildProcess, also check it hasn't already exited.
121	    if ('exitCode' in child && child.exitCode !== null) return false;
122	    return true;
123	  }
124	  return false;
125	};
126	
127	const fetchWithTimeout = async (url: string, timeoutMs: number): Promise<Response> => {
128	  const controller = new AbortController();
129	  const timeout = setTimeout(() => {
130	    controller.abort();
131	  }, timeoutMs);
132	
133	  try {
134	    return await fetch(url, {
135	      method: 'GET',
136	      signal: controller.signal,
137	      cache: 'no-store',
138	    });
139	  } finally {
140	    clearTimeout(timeout);
141	  }
142	};
143	
144	export class OpenClawEngineManager extends EventEmitter {
145	  private readonly baseDir: string;
146	  private readonly logsDir: string;
147	  private readonly stateDir: string;
148	  private readonly gatewayTokenPath: string;
149	  private readonly gatewayPortPath: string;
150	  private readonly gatewayLogPath: string;
151	  private readonly configPath: string;
152	
153	  private desiredVersion: string;
154	  private status: OpenClawEngineStatus;
155	  private gatewayProcess: GatewayProcess | null = null;
156	  private readonly expectedGatewayExits = new WeakSet<object>();
157	  private gatewayRestartTimer: NodeJS.Timeout | null = null;
158	  private gatewayRestartAttempt = 0;
159	  private shutdownRequested = false;
160	  private gatewayPort: number | null = null;
161	  private startGatewayPromise: Promise<OpenClawEngineStatus> | null = null;
162	  private secretEnvVars: Record<string, string> = {};
163	
164	  constructor() {
165	    super();
166	
167	    const userDataPath = app.getPath('userData');
168	    this.baseDir = path.join(userDataPath, 'openclaw');
169	    this.logsDir = path.join(this.baseDir, 'logs');
170	    this.stateDir = path.join(this.baseDir, 'state');
171	
172	    this.gatewayTokenPath = path.join(this.stateDir, 'gateway-token');
173	    this.gatewayPortPath = path.join(this.stateDir, 'gateway-port.json');
174	    this.gatewayLogPath = path.join(this.logsDir, 'gateway.log');
175	    this.configPath = path.join(this.stateDir, 'openclaw.json');
176	
177	    ensureDir(this.baseDir);
178	    ensureDir(this.logsDir);
179	    ensureDir(this.stateDir);
180	
181	    const runtime = this.resolveRuntimeMetadata();
182	    this.desiredVersion = runtime.version || DEFAULT_OPENCLAW_VERSION;
183	
184	    this.status = runtime.root
185	      ? {
186	          phase: 'ready',
187	          version: this.desiredVersion,
188	          message: 'OpenClaw runtime is ready.',
189	          canRetry: false,
190	        }
191	      : {
192	          phase: 'not_installed',
193	          version: null,
194	          message: `Bundled OpenClaw runtime is missing. Expected: ${runtime.expectedPathHint}`,
195	          canRetry: true,
196	        };
197	  }
198	
199	  /**
200	   * Set secret environment variables to inject into the gateway process.
```

**Tool Call: Read src/main/libs/agentEngine/openclawRuntimeAdapter.ts (1 - 200)**
Status: Completed

```
1	import { randomUUID } from 'crypto';
2	import { app, BrowserWindow } from 'electron';
3	import { EventEmitter } from 'events';
4	import * as fs from 'fs';
5	import * as path from 'path';
6	import { pathToFileURL } from 'url';
7	import type { PermissionResult } from '@anthropic-ai/claude-agent-sdk';
8	import type { CoworkMessage, CoworkSession, CoworkSessionStatus, CoworkExecutionMode, CoworkStore } from '../../coworkStore';
9	import {
10	  OpenClawEngineManager,
11	  type OpenClawGatewayConnectionInfo,
12	} from '../openclawEngineManager';
13	import type {
14	  CoworkContinueOptions,
15	  CoworkRuntime,
16	  CoworkRuntimeEvents,
17	  CoworkStartOptions,
18	  PermissionRequest,
19	} from './types';
20	import {
21	  buildManagedSessionKey,
22	  type OpenClawChannelSessionSync,
23	  isManagedSessionKey,
24	  parseManagedSessionKey,
25	  parseChannelSessionKey,
26	} from '../openclawChannelSessionSync';
27	import {
28	  extractGatewayHistoryEntries,
29	  extractGatewayMessageText,
30	} from '../openclawHistory';
31	import { extractOpenClawAssistantStreamText } from '../openclawAssistantText';
32	import { buildOpenClawLocalTimeContextPrompt } from '../openclawLocalTimeContextPrompt';
33	import { isDeleteCommand, getCommandDangerLevel } from '../commandSafety';
34	import { setCoworkProxySessionId } from '../coworkOpenAICompatProxy';
35	import { OPENCLAW_AGENT_TIMEOUT_SECONDS } from '../openclawConfigSync';
36	import { t } from '../../i18n';
37	
38	const OPENCLAW_GATEWAY_TOOL_EVENTS_CAP = 'tool-events';
39	const BRIDGE_MAX_MESSAGES = 20;
40	const BRIDGE_MAX_MESSAGE_CHARS = 1200;
41	const GATEWAY_READY_TIMEOUT_MS = 15_000;
42	const FINAL_HISTORY_SYNC_LIMIT = 50;
43	const CHANNEL_SESSION_DISCOVERY_LIMIT = 200;
44	
45	type GatewayEventFrame = {
46	  event: string;
47	  seq?: number;
48	  payload?: unknown;
49	};
50	
51	type GatewayClientLike = {
52	  start: () => void;
53	  stop: () => void;
54	  request: <T = Record<string, unknown>>(
55	    method: string,
56	    params?: unknown,
57	    opts?: { expectFinal?: boolean },
58	  ) => Promise<T>;
59	};
60	
61	type GatewayClientCtor = new (options: Record<string, unknown>) => GatewayClientLike;
62	
63	type ChatEventState = 'delta' | 'final' | 'aborted' | 'error';
64	
65	type ChatEventPayload = {
66	  runId?: string;
67	  sessionKey?: string;
68	  state?: ChatEventState;
69	  message?: unknown;
70	  errorMessage?: string;
71	  stopReason?: string;
72	};
73	
74	type AgentEventPayload = {
75	  seq?: number;
76	  runId?: string;
77	  sessionKey?: string;
78	  stream?: string;
79	  data?: unknown;
80	};
81	
82	type ExecApprovalRequestedPayload = {
83	  id?: string;
84	  request?: {
85	    command?: string;
86	    cwd?: string | null;
87	    host?: string | null;
88	    security?: string | null;
89	    ask?: string | null;
90	    resolvedPath?: string | null;
91	    sessionKey?: string | null;
92	    agentId?: string | null;
93	  };
94	};
95	
96	type ExecApprovalResolvedPayload = {
97	  id?: string;
98	};
99	
100	type TextStreamMode = 'unknown' | 'snapshot' | 'delta';
101	
102	type ActiveTurn = {
103	  sessionId: string;
104	  sessionKey: string;
105	  runId: string;
106	  turnToken: number;
107	  knownRunIds: Set<string>;
108	  assistantMessageId: string | null;
109	  committedAssistantText: string;
110	  currentAssistantSegmentText: string;
111	  currentText: string;
112	  /** Highest text length from agent assistant events (immune to chat delta noise). */
113	  agentAssistantTextLength: number;
114	  currentContentText: string;
115	  currentContentBlocks: string[];
116	  sawNonTextContentBlocks: boolean;
117	  textStreamMode: TextStreamMode;
118	  toolUseMessageIdByToolCallId: Map<string, string>;
119	  toolResultMessageIdByToolCallId: Map<string, string>;
120	  toolResultTextByToolCallId: Map<string, string>;
121	  stopRequested: boolean;
122	  /** True while async user message prefetch is in progress for channel sessions. */
123	  pendingUserSync: boolean;
124	  /** Chat events buffered while pendingUserSync is true. */
125	  bufferedChatPayloads: BufferedChatEvent[];
126	  /** Agent events buffered while pendingUserSync is true. */
127	  bufferedAgentPayloads: BufferedAgentEvent[];
128	  /** Client-side timeout watchdog timer (fallback for missing gateway abort events). */
129	  timeoutTimer?: ReturnType<typeof setTimeout>;
130	};
131	
132	type BufferedChatEvent = {
133	  payload: unknown;
134	  seq: number | undefined;
135	  bufferedAt: number;
136	};
137	
138	type BufferedAgentEvent = {
139	  payload: unknown;
140	  seq: number | undefined;
141	  bufferedAt: number;
142	};
143	
144	type PendingApprovalEntry = {
145	  requestId: string;
146	  sessionId: string;
147	  /** When true, use 'allow-always' decision so OpenClaw adds the command to its allowlist. */
148	  allowAlways?: boolean;
149	};
150	
151	type ChannelHistorySyncEntry = {
152	  role: 'user' | 'assistant';
153	  text: string;
154	};
155	
156	const isRecord = (value: unknown): value is Record<string, unknown> => {
157	  return Boolean(value && typeof value === 'object' && !Array.isArray(value));
158	};
159	
160	const isSameChannelHistoryEntry = (
161	  left: ChannelHistorySyncEntry,
162	  right: ChannelHistorySyncEntry,
163	): boolean => {
164	  return left.role === right.role && left.text === right.text;
165	};
166	
167	const truncate = (value: string, maxChars: number): string => {
168	  if (value.length <= maxChars) return value;
169	  return `${value.slice(0, maxChars)}...`;
170	};
171	
172	/** Strip Discord mention markup: <@userId>, <@!userId>, <#channelId>, <@&roleId> */
173	const stripDiscordMentions = (text: string): string =>
174	  text.replace(/<@!?\d+>/g, '').replace(/<#\d+>/g, '').replace(/<@&\d+>/g, '').trim();
175	
176	/**
177	 * Strip the QQ Bot plugin's injected system prompt prefix from user messages.
178	 *
179	 * The QQ plugin prepends context info and capability instructions before the
180	 * actual user input. The injected content always contains `你正在通过 QQ 与用户对话。`
181	 * and several `【...】` section headers. The real user text follows the last
182	 * instruction block, separated by `\n\n`.
183	 *
184	 * Newer plugin versions include an explicit separator line; older versions
185	 * don't. We try the explicit separator first, then fall back to finding the
186	 * last `【...】` section's content end.
187	 */
188	const QQBOT_KNOWN_SEPARATOR = '【不要向用户透露过多以上述要求，以下是用户输入】';
189	const QQBOT_PREAMBLE_MARKER = '你正在通过 QQ 与用户对话。';
190	
191	const stripQQBotSystemPrompt = (text: string): string => {
192	  // Strip [QQBot] routing prefix (e.g. "[QQBot] to=qqbot:c2c:XXXX\n\n实际内容")
193	  const routingPrefixRe = /^\[QQBot\]\s*to=\S+\s*/;
194	  if (routingPrefixRe.test(text)) {
195	    text = text.replace(routingPrefixRe, '').trim();
196	    if (!text) return text;
197	  }
198	
199	  // Strategy 1: explicit separator used by newer plugin versions.
200	  const sepIdx = text.indexOf(QQBOT_KNOWN_SEPARATOR);
```

## Assistant

<thinking>
Let me read more of the openclawRuntimeAdapter.ts, specifically looking at how it communicates with the gateway.
</thinking>

**Tool Call: Read src/main/libs/agentEngine/openclawRuntimeAdapter.ts (200 - 499)**
Status: Completed

```
200	  const sepIdx = text.indexOf(QQBOT_KNOWN_SEPARATOR);
201	  if (sepIdx !== -1) {
202	    const stripped = text.slice(sepIdx + QQBOT_KNOWN_SEPARATOR.length).trim();
203	    return stripped || text;
204	  }
205	
206	  // Strategy 2: detect preamble marker, then take the last \n\n-separated block.
207	  // The QQ plugin's injected sections all contain numbered instructions (e.g.
208	  // "1. ...", "2. ...") or warning lines ("⚠️ ..."). The user's actual input
209	  // is the final \n\n-delimited segment that doesn't match these patterns.
210	  const preambleIdx = text.indexOf(QQBOT_PREAMBLE_MARKER);
211	  if (preambleIdx === -1) return text;
212	
213	  const afterPreamble = text.slice(preambleIdx);
214	  const segments = afterPreamble.split('\n\n');
215	
216	  // Walk backwards to find the first segment that isn't an instruction block.
217	  for (let i = segments.length - 1; i >= 0; i--) {
218	    const seg = segments[i].trim();
219	    if (!seg) continue;
220	    // Instruction lines start with "1. ", "⚠", or "【"
221	    if (/^\d+\.\s/.test(seg) || /^⚠/.test(seg) || /^【/.test(seg) || seg.startsWith('- ')) continue;
222	    // This segment looks like user input.
223	    const stripped = segments.slice(i).join('\n\n').trim();
224	    return stripped || text;
225	  }
226	
227	  return text;
228	};
229	
230	const extractMessageText = extractGatewayMessageText;
231	
232	const summarizeGatewayMessageShape = (message: unknown): string => {
233	  if (!isRecord(message)) {
234	    return `non-record:${typeof message}`;
235	  }
236	
237	  const role = typeof message.role === 'string' ? message.role : '?';
238	  const content = message.content;
239	  if (typeof content === 'string') {
240	    return `role=${role} content=string(${content.length}) text="${truncate(content, 120)}"`;
241	  }
242	  if (Array.isArray(content)) {
243	    const parts = content.map((item) => {
244	      if (!isRecord(item)) return typeof item;
245	      const type = typeof item.type === 'string' ? item.type : 'object';
246	      const text = typeof item.text === 'string' ? `:${truncate(item.text, 60)}` : '';
247	      return `${type}${text}`;
248	    });
249	    return `role=${role} content=[${parts.join(', ')}]`;
250	  }
251	  if (isRecord(content)) {
252	    return `role=${role} contentKeys=${Object.keys(content).join(',')}`;
253	  }
254	  if (typeof message.text === 'string') {
255	    return `role=${role} text=${truncate(message.text, 120)}`;
256	  }
257	  return `role=${role} keys=${Object.keys(message).join(',')}`;
258	};
259	
260	const extractTextBlocksAndSignals = (
261	  message: unknown,
262	): { textBlocks: string[]; sawNonTextContentBlocks: boolean } => {
263	  if (!isRecord(message)) {
264	    return {
265	      textBlocks: [],
266	      sawNonTextContentBlocks: false,
267	    };
268	  }
269	
270	  const content = message.content;
271	  if (typeof content === 'string') {
272	    const text = content.trim();
273	    return {
274	      textBlocks: text ? [text] : [],
275	      sawNonTextContentBlocks: false,
276	    };
277	  }
278	  if (!Array.isArray(content)) {
279	    return {
280	      textBlocks: [],
281	      sawNonTextContentBlocks: false,
282	    };
283	  }
284	
285	  const textBlocks: string[] = [];
286	  let sawNonTextContentBlocks = false;
287	  for (const block of content) {
288	    if (!isRecord(block)) continue;
289	    if (block.type === 'text' && typeof block.text === 'string') {
290	      const text = block.text.trim();
291	      if (text) {
292	        textBlocks.push(text);
293	      }
294	      continue;
295	    }
296	    if (typeof block.type === 'string' && block.type !== 'thinking') {
297	      sawNonTextContentBlocks = true;
298	      console.log('[Debug:extractBlocks] non-text block type:', block.type, 'content:', JSON.stringify(block).slice(0, 500));
299	    }
300	  }
301	
302	  return {
303	    textBlocks,
304	    sawNonTextContentBlocks,
305	  };
306	};
307	
308	/**
309	 * Extract file paths from assistant "message" tool calls in chat.history.
310	 * Only scans messages after the last user message (current turn).
311	 * The model sends files to Telegram using: toolCall { name: "message", arguments: { action: "send", filePath: "..." } }
312	 */
313	const extractSentFilePathsFromHistory = (messages: unknown[]): string[] => {
314	  // Find the last user message index to scope to current turn only
315	  let lastUserIdx = -1;
316	  for (let i = messages.length - 1; i >= 0; i--) {
317	    const msg = messages[i];
318	    if (isRecord(msg) && (msg as Record<string, unknown>).role === 'user') {
319	      lastUserIdx = i;
320	      break;
321	    }
322	  }
323	
324	  const filePaths: string[] = [];
325	  const seen = new Set<string>();
326	  const startIdx = lastUserIdx >= 0 ? lastUserIdx + 1 : 0;
327	  for (let i = startIdx; i < messages.length; i++) {
328	    const msg = messages[i];
329	    if (!isRecord(msg)) continue;
330	    const role = typeof msg.role === 'string' ? msg.role.trim().toLowerCase() : '';
331	    if (role !== 'assistant') continue;
332	    if (!Array.isArray(msg.content)) continue;
333	    for (const block of msg.content as Array<Record<string, unknown>>) {
334	      if (!isRecord(block)) continue;
335	      if (block.type !== 'toolCall' || block.name !== 'message') continue;
336	      const args = block.arguments;
337	      if (!isRecord(args)) continue;
338	      const filePath = typeof args.filePath === 'string' ? args.filePath.trim() : '';
339	      if (filePath && !seen.has(filePath)) {
340	        seen.add(filePath);
341	        filePaths.push(filePath);
342	      }
343	    }
344	  }
345	  return filePaths;
346	};
347	
348	/**
349	 * Extract and concatenate all assistant text from the current turn in chat.history.
350	 * The current turn starts after the last user message.
351	 */
352	const extractCurrentTurnAssistantText = (messages: unknown[]): string => {
353	  // Find the last user message index (turn boundary)
354	  let lastUserIdx = -1;
355	  for (let i = messages.length - 1; i >= 0; i--) {
356	    const msg = messages[i];
357	    if (isRecord(msg) && (msg as Record<string, unknown>).role === 'user') {
358	      lastUserIdx = i;
359	      break;
360	    }
361	  }
362	
363	  const startIdx = lastUserIdx >= 0 ? lastUserIdx + 1 : 0;
364	  const textParts: string[] = [];
365	  for (let i = startIdx; i < messages.length; i++) {
366	    const msg = messages[i];
367	    if (!isRecord(msg)) continue;
368	    const role = typeof msg.role === 'string' ? msg.role.trim().toLowerCase() : '';
369	    if (role !== 'assistant') continue;
370	    const text = extractMessageText(msg).trim();
371	    if (text) {
372	      textParts.push(text);
373	    }
374	  }
375	  return textParts.join('\n\n');
376	};
377	
378	const isDroppedBoundaryTextBlockSubset = (streamedTextBlocks: string[], finalTextBlocks: string[]): boolean => {
379	  if (finalTextBlocks.length === 0 || finalTextBlocks.length >= streamedTextBlocks.length) {
380	    return false;
381	  }
382	  if (finalTextBlocks.every((block, index) => streamedTextBlocks[index] === block)) {
383	    return true;
384	  }
385	  const suffixStart = streamedTextBlocks.length - finalTextBlocks.length;
386	  return finalTextBlocks.every((block, index) => streamedTextBlocks[suffixStart + index] === block);
387	};
388	
389	const extractToolText = (payload: unknown): string => {
390	  if (typeof payload === 'string') {
391	    return payload;
392	  }
393	
394	  if (Array.isArray(payload)) {
395	    const lines = payload
396	      .map((item) => extractToolText(item).trim())
397	      .filter(Boolean);
398	    if (lines.length > 0) {
399	      return lines.join('\n');
400	    }
401	  }
402	
403	  if (!isRecord(payload)) {
404	    if (payload === undefined || payload === null) return '';
405	    try {
406	      return JSON.stringify(payload, null, 2);
407	    } catch {
408	      return String(payload);
409	    }
410	  }
411	
412	  if (typeof payload.text === 'string' && payload.text.trim()) {
413	    return payload.text;
414	  }
415	  if (typeof payload.output === 'string' && payload.output.trim()) {
416	    return payload.output;
417	  }
418	  if (typeof payload.stdout === 'string' || typeof payload.stderr === 'string') {
419	    const chunks = [
420	      typeof payload.stdout === 'string' ? payload.stdout : '',
421	      typeof payload.stderr === 'string' ? payload.stderr : '',
422	    ].filter(Boolean);
423	    if (chunks.length > 0) {
424	      return chunks.join('\n');
425	    }
426	  }
427	
428	  const content = payload.content;
429	  if (typeof content === 'string' && content.trim()) {
430	    return content;
431	  }
432	  if (Array.isArray(content)) {
433	    const chunks: string[] = [];
434	    for (const item of content) {
435	      if (typeof item === 'string' && item.trim()) {
436	        chunks.push(item);
437	        continue;
438	      }
439	      if (!isRecord(item)) continue;
440	      if (typeof item.text === 'string' && item.text.trim()) {
441	        chunks.push(item.text);
442	        continue;
443	      }
444	      if (typeof item.content === 'string' && item.content.trim()) {
445	        chunks.push(item.content);
446	      }
447	    }
448	    if (chunks.length > 0) {
449	      return chunks.join('\n');
450	    }
451	  }
452	
453	  try {
454	    return JSON.stringify(payload, null, 2);
455	  } catch {
456	    return String(payload);
457	  }
458	};
459	
460	const toToolInputRecord = (value: unknown): Record<string, unknown> => {
461	  if (isRecord(value)) {
462	    return value;
463	  }
464	  if (value === undefined || value === null) {
465	    return {};
466	  }
467	  return { value };
468	};
469	
470	const computeSuffixPrefixOverlap = (left: string, right: string): number => {
471	  const leftProbe = left.slice(-256);
472	  const rightProbe = right.slice(0, 256);
473	  const maxOverlap = Math.min(leftProbe.length, rightProbe.length);
474	  for (let size = maxOverlap; size > 0; size -= 1) {
475	    if (leftProbe.slice(-size) === rightProbe.slice(0, size)) {
476	      return size;
477	    }
478	  }
479	  return 0;
480	};
481	
482	const mergeStreamingText = (
483	  previousText: string,
484	  incomingText: string,
485	  mode: TextStreamMode,
486	): { text: string; mode: TextStreamMode } => {
487	  if (!incomingText) {
488	    return { text: previousText, mode };
489	  }
490	  if (!previousText) {
491	    return { text: incomingText, mode };
492	  }
493	  if (incomingText === previousText) {
494	    return { text: previousText, mode };
495	  }
496	
497	  if (mode === 'snapshot') {
498	    if (previousText.startsWith(incomingText) && incomingText.length < previousText.length) {
499	      return { text: previousText, mode };
```

**Tool Call: Read src/main/libs/openclawEngineManager.ts (200 - 399)**
Status: Completed

```
200	   * Set secret environment variables to inject into the gateway process.
201	   * These contain the plaintext values for `${VAR}` placeholders in openclaw.json.
202	   */
203	  setSecretEnvVars(vars: Record<string, string>): void {
204	    this.secretEnvVars = vars;
205	  }
206	
207	  /** Return the current secret env vars snapshot (for change detection). */
208	  getSecretEnvVars(): Record<string, string> {
209	    return this.secretEnvVars;
210	  }
211	
212	  override on<U extends keyof OpenClawEngineManagerEvents>(
213	    event: U,
214	    listener: OpenClawEngineManagerEvents[U],
215	  ): this {
216	    return super.on(event, listener);
217	  }
218	
219	  override emit<U extends keyof OpenClawEngineManagerEvents>(
220	    event: U,
221	    ...args: Parameters<OpenClawEngineManagerEvents[U]>
222	  ): boolean {
223	    return super.emit(event, ...args);
224	  }
225	
226	  getStatus(): OpenClawEngineStatus {
227	    return { ...this.status };
228	  }
229	
230	  setExternalError(message: string): OpenClawEngineStatus {
231	    const runtime = this.resolveRuntimeMetadata();
232	    this.setStatus({
233	      phase: 'error',
234	      version: runtime.version || this.status.version || null,
235	      message: message.slice(0, 500),
236	      canRetry: true,
237	    });
238	    return this.getStatus();
239	  }
240	
241	  getDesiredVersion(): string {
242	    return this.desiredVersion;
243	  }
244	
245	  getBaseDir(): string {
246	    return this.baseDir;
247	  }
248	
249	  getStateDir(): string {
250	    return this.stateDir;
251	  }
252	
253	  getConfigPath(): string {
254	    return this.configPath;
255	  }
256	
257	  getGatewayConnectionInfo(): OpenClawGatewayConnectionInfo {
258	    const runtime = this.resolveRuntimeMetadata();
259	    const port = this.gatewayPort ?? this.readGatewayPort();
260	    const token = this.readGatewayToken();
261	    const clientEntryPath = runtime.root ? this.resolveGatewayClientEntry(runtime.root) : null;
262	
263	    return {
264	      version: runtime.version,
265	      port,
266	      token,
267	      url: port ? `ws://127.0.0.1:${port}` : null,
268	      clientEntryPath,
269	    };
270	  }
271	
272	  async ensureReady(_options: { forceReinstall?: boolean } = {}): Promise<OpenClawEngineStatus> {
273	    const runtime = this.resolveRuntimeMetadata();
274	    this.desiredVersion = runtime.version || DEFAULT_OPENCLAW_VERSION;
275	
276	    if (!runtime.root) {
277	      this.setStatus({
278	        phase: 'not_installed',
279	        version: null,
280	        message: `Bundled OpenClaw runtime is missing. Expected: ${runtime.expectedPathHint}`,
281	        canRetry: true,
282	      });
283	      return this.getStatus();
284	    }
285	
286	    const localExtensionSync = syncLocalOpenClawExtensionsIntoRuntime(runtime.root);
287	    if (localExtensionSync.copied.length > 0) {
288	      console.log(`[OpenClaw] synced local extensions: ${localExtensionSync.copied.join(', ')}`);
289	    }
290	
291	    if (this.status.phase === 'running') {
292	      return this.getStatus();
293	    }
294	
295	    this.setStatus({
296	      phase: 'ready',
297	      version: this.desiredVersion,
298	      message: 'OpenClaw runtime is ready.',
299	      canRetry: false,
300	    });
301	    return this.getStatus();
302	  }
303	
304	  async startGateway(): Promise<OpenClawEngineStatus> {
305	    if (this.startGatewayPromise) {
306	      console.log('[OpenClaw] startGateway: already in progress, reusing existing promise');
307	      return this.startGatewayPromise;
308	    }
309	    this.startGatewayPromise = this.doStartGateway().finally(() => {
310	      this.startGatewayPromise = null;
311	    });
312	    return this.startGatewayPromise;
313	  }
314	
315	  private async doStartGateway(): Promise<OpenClawEngineStatus> {
316	    this.shutdownRequested = false;
317	    const t0 = Date.now();
318	    const elapsed = () => `${Date.now() - t0}ms`;
319	
320	    const ensured = await this.ensureReady();
321	    console.log(`[OpenClaw] startGateway: ensureReady done (${elapsed()}), phase=${ensured.phase}`);
322	    if (ensured.phase !== 'ready' && ensured.phase !== 'running') {
323	      return ensured;
324	    }
325	
326	    if (isGatewayProcessAlive(this.gatewayProcess)) {
327	      const port = this.gatewayPort ?? this.readGatewayPort();
328	      if (port) {
329	        const healthy = await this.isGatewayHealthy(port);
330	        console.log(`[OpenClaw] startGateway: existing process health check (${elapsed()}), healthy=${healthy}`);
331	        if (healthy) {
332	          if (this.status.phase !== 'running') {
333	            this.setStatus({
334	              phase: 'running',
335	              version: this.desiredVersion,
336	              message: `OpenClaw gateway is running on loopback:${port}.`,
337	              canRetry: false,
338	            });
339	          }
340	          return this.getStatus();
341	        }
342	      }
343	
344	      await this.stopGatewayProcess(this.gatewayProcess);
345	      this.gatewayProcess = null;
346	    }
347	
348	    const runtime = this.resolveRuntimeMetadata();
349	    console.log(`[OpenClaw] startGateway: resolveRuntimeMetadata done (${elapsed()}), root=${runtime.root ? 'found' : 'missing'}`);
350	    if (!runtime.root) {
351	      this.setStatus({
352	        phase: 'not_installed',
353	        version: null,
354	        message: `Bundled OpenClaw runtime is missing. Expected: ${runtime.expectedPathHint}`,
355	        canRetry: true,
356	      });
357	      return this.getStatus();
358	    }
359	
360	    this.ensureBareEntryFiles(runtime.root);
361	    console.log(`[OpenClaw] startGateway: ensureBareEntryFiles done (${elapsed()})`);
362	    const openclawEntry = this.resolveOpenClawEntry(runtime.root);
363	    console.log(`[OpenClaw] startGateway: resolveOpenClawEntry done (${elapsed()}), entry=${openclawEntry}`);
364	    if (!openclawEntry) {
365	      this.setStatus({
366	        phase: 'error',
367	        version: runtime.version,
368	        message: `OpenClaw entry file is missing in runtime: ${runtime.root}.`,
369	        canRetry: true,
370	      });
371	      return this.getStatus();
372	    }
373	
374	    const token = this.ensureGatewayToken();
375	    console.log(`[OpenClaw] startGateway: ensureGatewayToken done (${elapsed()})`);
376	    const port = await this.resolveGatewayPort();
377	    console.log(`[OpenClaw] startGateway: resolveGatewayPort done (${elapsed()}), port=${port}`);
378	    this.gatewayPort = port;
379	    this.writeGatewayPort(port);
380	    this.ensureConfigFile();
381	    console.log(`[OpenClaw] startGateway: pre-fork setup done (${elapsed()})`);
382	
383	    this.setStatus({
384	      phase: 'starting',
385	      version: runtime.version,
386	      progressPercent: 10,
387	      message: 'Starting OpenClaw gateway...',
388	      canRetry: false,
389	    });
390	
391	    const compileCacheDir = path.join(this.stateDir, '.compile-cache');
392	    console.log(`[OpenClaw] compile cache dir: ${compileCacheDir}`);
393	    const electronNodeRuntimePath = getElectronNodeRuntimePath();
394	    const cliShimDir = this.ensureBundledCliShims();
395	    const skillsRoot = getSkillsRoot().replace(/\\/g, '/');
396	
397	    const env: NodeJS.ProcessEnv = {
398	      ...process.env,
399	      SKILLS_ROOT: skillsRoot,
```

## Assistant

<thinking>
Let me read more of the openclawRuntimeAdapter.ts, specifically looking for the gateway client connection and how it makes requests to the gateway.
</thinking>

**Tool Call: Read src/main/libs/agentEngine/openclawRuntimeAdapter.ts (500 - 699)**
Status: Completed

```
500	    }
501	    return { text: incomingText, mode };
502	  }
503	
504	  if (mode === 'delta') {
505	    if (incomingText.startsWith(previousText)) {
506	      return { text: incomingText, mode: 'snapshot' };
507	    }
508	    const overlap = computeSuffixPrefixOverlap(previousText, incomingText);
509	    return { text: previousText + incomingText.slice(overlap), mode };
510	  }
511	
512	  if (incomingText.startsWith(previousText)) {
513	    return { text: incomingText, mode: 'snapshot' };
514	  }
515	  if (previousText.startsWith(incomingText)) {
516	    return { text: previousText, mode: 'snapshot' };
517	  }
518	  if (incomingText.includes(previousText) && incomingText.length > previousText.length) {
519	    return { text: incomingText, mode: 'snapshot' };
520	  }
521	
522	  const overlap = computeSuffixPrefixOverlap(previousText, incomingText);
523	  if (overlap > 0) {
524	    return { text: previousText + incomingText.slice(overlap), mode: 'delta' };
525	  }
526	
527	  return { text: previousText + incomingText, mode: 'delta' };
528	};
529	
530	const sleep = async (ms: number): Promise<void> => {
531	  await new Promise((resolve) => setTimeout(resolve, ms));
532	};
533	
534	const waitWithTimeout = async (promise: Promise<void>, timeoutMs: number): Promise<void> => {
535	  let timeoutId: NodeJS.Timeout | null = null;
536	  const timeoutPromise = new Promise<void>((_resolve, reject) => {
537	    timeoutId = setTimeout(() => {
538	      reject(new Error(`OpenClaw gateway client connect timeout after ${timeoutMs}ms.`));
539	    }, timeoutMs);
540	  });
541	  try {
542	    await Promise.race([promise, timeoutPromise]);
543	  } finally {
544	    if (timeoutId) {
545	      clearTimeout(timeoutId);
546	    }
547	  }
548	};
549	
550	export class OpenClawRuntimeAdapter extends EventEmitter implements CoworkRuntime {
551	  private readonly store: CoworkStore;
552	  private readonly engineManager: OpenClawEngineManager;
553	  private readonly activeTurns = new Map<string, ActiveTurn>();
554	  private readonly sessionIdBySessionKey = new Map<string, string>();
555	  private readonly sessionIdByRunId = new Map<string, string>();
556	  private readonly pendingAgentEventsByRunId = new Map<string, AgentEventPayload[]>();
557	  private readonly lastChatSeqByRunId = new Map<string, number>();
558	  private readonly lastAgentSeqByRunId = new Map<string, number>();
559	  private readonly pendingApprovals = new Map<string, PendingApprovalEntry>();
560	  private readonly pendingTurns = new Map<string, { resolve: () => void; reject: (error: Error) => void }>();
561	  private readonly confirmationModeBySession = new Map<string, 'modal' | 'text'>();
562	  private readonly bridgedSessions = new Set<string>();
563	  private readonly lastSystemPromptBySession = new Map<string, string>();
564	  private readonly gatewayHistoryCountBySession = new Map<string, number>();
565	  private readonly latestTurnTokenBySession = new Map<string, number>();
566	
567	  /**
568	   * Sessions that were manually stopped by the user via stopSession().
569	   * Maps sessionId → timestamp of when stop was requested.
570	   * Used to suppress automatic ActiveTurn re-creation from late-arriving
571	   * OpenClaw Gateway events (e.g. POPO/Telegram channel events that arrive
572	   * after the user clicked Stop).  Entries expire after STOP_COOLDOWN_MS.
573	   */
574	  private readonly stoppedSessions = new Map<string, number>();
575	  private static readonly STOP_COOLDOWN_MS = 10_000; // 10 seconds
576	
577	  private gatewayClient: GatewayClientLike | null = null;
578	  private gatewayClientVersion: string | null = null;
579	  private gatewayClientEntryPath: string | null = null;
580	  /** Holds the client between start() and onHelloOk so stopGatewayClient can clean it up. */
581	  private pendingGatewayClient: GatewayClientLike | null = null;
582	  private gatewayReadyPromise: Promise<void> | null = null;
583	  /** Serializes concurrent calls to ensureGatewayClientReady to prevent duplicate clients. */
584	  private gatewayClientInitLock: Promise<void> | null = null;
585	  private channelSessionSync: OpenClawChannelSessionSync | null = null;
586	  private readonly knownChannelSessionIds = new Set<string>();
587	  private readonly fullySyncedSessions = new Set<string>();
588	  /** Per-session cursor: number of gateway history entries (user+assistant) already synced locally. */
589	  private readonly channelSyncCursor = new Map<string, number>();
590	  /** Sessions re-created after user deletion — use latestOnly sync to avoid replaying old history. */
591	  private readonly reCreatedChannelSessionIds = new Set<string>();
592	  /** Channel sessionKeys explicitly deleted by the user. Polling will not re-create these. */
593	  private readonly deletedChannelKeys = new Set<string>();
594	  /** Sessions that were manually stopped by the user. Used to suppress the timeout hint
595	   *  when the gateway sends back a late 'aborted' event after stopSession() already cleaned up the turn. */
596	  private readonly manuallyStoppedSessions = new Set<string>();
597	  /** Session keys whose origin is "heartbeat" — discovered via polling, used to filter real-time events. */
598	  private readonly heartbeatSessionKeys = new Set<string>();
599	  private channelPollingTimer: ReturnType<typeof setInterval> | null = null;
600	
601	  private static readonly CHANNEL_POLL_INTERVAL_MS = 10_000;
602	  private static readonly FULL_HISTORY_SYNC_LIMIT = 50;
603	  private browserPrewarmAttempted = false;
604	
605	  /** Gateway WS auto-reconnect state */
606	  private gatewayReconnectTimer: ReturnType<typeof setTimeout> | null = null;
607	  private gatewayReconnectAttempt = 0;
608	  /** Set to true before intentionally stopping the client (e.g. version upgrade) to suppress auto-reconnect. */
609	  private gatewayStoppingIntentionally = false;
610	  private static readonly GATEWAY_RECONNECT_MAX_ATTEMPTS = 10;
611	  private static readonly GATEWAY_RECONNECT_DELAYS = [2_000, 5_000, 10_000, 15_000, 30_000]; // ms
612	
613	  /** Gateway tick heartbeat watchdog state */
614	  private lastTickTimestamp = 0;
615	  private tickWatchdogTimer: ReturnType<typeof setInterval> | null = null;
616	  private static readonly TICK_WATCHDOG_INTERVAL_MS = 60_000; // check every 60s
617	  private static readonly TICK_TIMEOUT_MS = 90_000; // 3 tick cycles (30s each) without response → dead
618	
619	  /** Throttle state for messageUpdate IPC emissions during streaming */
620	  private lastMessageUpdateEmitTime: Map<string, number> = new Map();
621	  private pendingMessageUpdateTimer: Map<string, ReturnType<typeof setTimeout>> = new Map();
622	  private static readonly MESSAGE_UPDATE_THROTTLE_MS = 200;
623	
624	  /** Throttle state for SQLite store writes during streaming */
625	  private lastStoreUpdateTime: Map<string, number> = new Map();
626	  private pendingStoreUpdateTimer: Map<string, ReturnType<typeof setTimeout>> = new Map();
627	  private static readonly STORE_UPDATE_THROTTLE_MS = 250;
628	
629	  /**
630	   * Server-side agent timeout in seconds (mirrors agents.defaults.timeoutSeconds in openclaw config).
631	   * Used to set a client-side fallback timer that fires slightly after the server timeout,
632	   * so LobsterAI can recover even when the gateway fails to deliver the abort event.
633	   */
634	  agentTimeoutSeconds = OPENCLAW_AGENT_TIMEOUT_SECONDS;
635	  private static readonly CLIENT_TIMEOUT_GRACE_MS = 30_000;
636	
637	  constructor(store: CoworkStore, engineManager: OpenClawEngineManager) {
638	    super();
639	    this.store = store;
640	    this.engineManager = engineManager;
641	  }
642	
643	  setChannelSessionSync(sync: OpenClawChannelSessionSync): void {
644	    this.channelSessionSync = sync;
645	  }
646	
647	  /**
648	   * Fetch session history from OpenClaw by sessionKey and return a transient
649	   * CoworkSession object (not persisted to local database).
650	   * First checks if a local session already exists via channel sync.
651	   * Returns a CoworkSession if successful, or null.
652	   */
653	  async fetchSessionByKey(sessionKey: string): Promise<CoworkSession | null> {
654	    const managedSession = parseManagedSessionKey(sessionKey);
655	    if (managedSession) {
656	      return this.store.getSession(managedSession.sessionId) ?? null;
657	    }
658	
659	    // 1. Try existing local session via channel/main-agent resolution
660	    if (this.channelSessionSync) {
661	      const existingId = this.channelSessionSync.resolveSession(sessionKey);
662	      if (existingId) {
663	        const session = this.store.getSession(existingId);
664	        if (session && session.messages.length > 0) {
665	          return session;
666	        }
667	      }
668	    }
669	
670	    // 2. Fetch history from OpenClaw server and build a transient session object
671	    const client = this.gatewayClient;
672	    if (!client) return null;
673	
674	    try {
675	      const history = await client.request<{ messages?: unknown[] }>('chat.history', {
676	        sessionKey,
677	        limit: OpenClawRuntimeAdapter.FULL_HISTORY_SYNC_LIMIT,
678	      });
679	      if (!Array.isArray(history?.messages) || history.messages.length === 0) {
680	        return this.readFromDeletedTranscript(sessionKey);
681	      }
682	
683	      const now = Date.now();
684	      const messages: CoworkMessage[] = [];
685	      let msgIndex = 0;
686	
687	      for (const entry of extractGatewayHistoryEntries(history.messages)) {
688	        messages.push({
689	          id: `transient-${msgIndex++}`,
690	          type: entry.role,
691	          content: entry.text,
692	          timestamp: now,
693	          metadata: entry.role === 'assistant' ? { isStreaming: false, isFinal: true } : {},
694	        });
695	      }
696	
697	      if (messages.length === 0) return null;
698	
699	      // Return a transient session (not saved to database)
```

**Tool Call: Read src/main/libs/openclawEngineManager.ts (400 - 599)**
Status: Completed

```
400	      LOBSTERAI_SKILLS_ROOT: skillsRoot,
401	      OPENCLAW_HOME: runtime.root,
402	      OPENCLAW_STATE_DIR: this.stateDir,
403	      OPENCLAW_CONFIG_PATH: this.configPath,
404	      OPENCLAW_GATEWAY_TOKEN: token,
405	      OPENCLAW_GATEWAY_PORT: String(port),
406	      OPENCLAW_NO_RESPAWN: '1',
407	      OPENCLAW_ENGINE_VERSION: runtime.version || DEFAULT_OPENCLAW_VERSION,
408	      OPENCLAW_BUNDLED_PLUGINS_DIR: path.join(runtime.root, 'extensions'),
409	      // Enable debug-level logging so gateway emits phase-level detail during startup.
410	      OPENCLAW_LOG_LEVEL: 'debug',
411	      // Enable V8 compile cache for both CJS and ESM modules.
412	      // This env var works for import() (ESM), unlike enableCompileCache() which is CJS-only.
413	      NODE_COMPILE_CACHE: compileCacheDir,
414	      LOBSTERAI_ELECTRON_PATH: electronNodeRuntimePath.replace(/\\/g, '/'),
415	      LOBSTERAI_OPENCLAW_ENTRY: openclawEntry.replace(/\\/g, '/'),
416	      // Inject secret values for ${VAR} placeholders in openclaw.json.
417	      // This keeps plaintext credentials out of the config file on disk.
418	      ...this.secretEnvVars,
419	    };
420	
421	    // Ensure the gateway process uses the host's local timezone for logging.
422	    // macOS does not set TZ in the environment by default (it uses NSTimeZone/ICU),
423	    // so utilityProcess.fork() children may fall back to UTC for date formatting.
424	    if (!env.TZ) {
425	      const hostTimezone = Intl.DateTimeFormat().resolvedOptions().timeZone;
426	      if (hostTimezone) {
427	        env.TZ = hostTimezone;
428	        console.log(`[OpenClaw] injected TZ=${hostTimezone} into gateway env`);
429	      }
430	    }
431	
432	    if (cliShimDir) {
433	      // Plain object is case-sensitive: the spread key from process.env on Windows is "Path",
434	      // not "PATH". We must read the actual key to avoid creating a PATH with only cliShimDir.
435	      const currentPath = env.PATH || env.Path || '';
436	      env.PATH = [cliShimDir, currentPath].filter(Boolean).join(path.delimiter);
437	    }
438	
439	    // Prepend bundled/user Python runtime paths so gateway exec commands
440	    // find the LobsterAI-managed Python instead of the Windows Store stub.
441	    appendPythonRuntimeToEnv(env as Record<string, string | undefined>);
442	
443	    // Inject node/npm/npx shims so gateway exec commands can use them.
444	    // The shims wrap Electron as a Node.js runtime via ELECTRON_RUN_AS_NODE=1.
445	    const npmBinDir = app.isPackaged
446	      ? path.join(process.resourcesPath, 'app.asar.unpacked', 'node_modules', 'npm', 'bin')
447	      : undefined;
448	    const nodeShimDir = ensureElectronNodeShim(electronNodeRuntimePath, npmBinDir);
449	    if (nodeShimDir) {
450	      const curPath = env.PATH || env.Path || '';
451	      env.PATH = [nodeShimDir, curPath].filter(Boolean).join(path.delimiter);
452	      env.LOBSTERAI_NPM_BIN_DIR = npmBinDir || '';
453	    }
454	
455	    if (isSystemProxyEnabled()) {
456	      const proxyUrl = await resolveSystemProxyUrl('https://openrouter.ai');
457	      if (proxyUrl) {
458	        env.http_proxy = proxyUrl;
459	        env.https_proxy = proxyUrl;
460	        env.HTTP_PROXY = proxyUrl;
461	        env.HTTPS_PROXY = proxyUrl;
462	        console.log('[OpenClaw] Injected system proxy for gateway:', proxyUrl);
463	      }
464	    }
465	
466	    const forkArgs = ['gateway', '--bind', 'loopback', '--port', String(port), '--token', token, '--verbose'];
467	    console.log(`[OpenClaw] forking gateway: entry=${openclawEntry}, cwd=${runtime.root}, port=${port}, args=${JSON.stringify(forkArgs)}`);
468	
469	    // On Windows, use child_process.spawn with ELECTRON_RUN_AS_NODE=1 instead of
470	    // utilityProcess.fork(). Benchmark shows utilityProcess has ~5x overhead for
471	    // cold ESM compilation on Windows (163s vs 34s for a 28MB bundle).
472	    let child: GatewayProcess;
473	    if (process.platform === 'win32') {
474	      child = spawn(
475	        process.execPath,
476	        [openclawEntry, ...forkArgs],
477	        {
478	          cwd: runtime.root,
479	          env: { ...env, ELECTRON_RUN_AS_NODE: '1' },
480	          stdio: ['ignore', 'pipe', 'pipe'],
481	          windowsHide: true,
482	        },
483	      );
484	    } else {
485	      child = utilityProcess.fork(
486	        openclawEntry,
487	        forkArgs,
488	        {
489	          cwd: runtime.root,
490	          env,
491	          stdio: 'pipe',
492	          serviceName: 'OpenClaw Gateway',
493	        },
494	      );
495	    }
496	    console.log(`[OpenClaw] startGateway: gateway process created (${elapsed()}), platform=${process.platform}`);
497	
498	    this.gatewayProcess = child;
499	    this.attachGatewayProcessLogs(child);
500	    this.attachGatewayExitHandlers(child);
501	
502	    // Wait for the spawn event to confirm the process started (pid becomes available).
503	    child.once('spawn', () => {
504	      console.log(`[OpenClaw] gateway process spawned (${elapsed()}), pid=${child.pid}`);
505	    });
506	
507	    const ready = await this.waitForGatewayReady(port, GATEWAY_BOOT_TIMEOUT_MS);
508	    console.log(`[OpenClaw] startGateway: waitForGatewayReady returned (${elapsed()}), ready=${ready}`);
509	    if (!ready) {
510	      this.setStatus({
511	        phase: 'error',
512	        version: runtime.version,
513	        message: 'OpenClaw gateway failed to become healthy in time.',
514	        canRetry: true,
515	      });
516	      this.stopGatewayProcess(child);
517	      return this.getStatus();
518	    }
519	
520	    console.log(`[OpenClaw] startGateway: gateway is running, total startup time: ${elapsed()}`);
521	    // Reset restart counter on successful start — gateway is healthy
522	    this.gatewayRestartAttempt = 0;
523	    this.setStatus({
524	      phase: 'running',
525	      version: runtime.version,
526	      progressPercent: 100,
527	      message: `OpenClaw gateway is running on loopback:${port}.`,
528	      canRetry: false,
529	    });
530	
531	    return this.getStatus();
532	  }
533	
534	  async stopGateway(): Promise<void> {
535	    this.shutdownRequested = true;
536	
537	    if (this.gatewayRestartTimer) {
538	      clearTimeout(this.gatewayRestartTimer);
539	      this.gatewayRestartTimer = null;
540	    }
541	
542	    if (this.gatewayProcess) {
543	      console.log('[OpenClaw] stopping gateway process...');
544	      await this.stopGatewayProcess(this.gatewayProcess);
545	      console.log('[OpenClaw] gateway process stopped');
546	      this.gatewayProcess = null;
547	    }
548	
549	    const runtime = this.resolveRuntimeMetadata();
550	    this.setStatus({
551	      phase: runtime.root ? 'ready' : 'not_installed',
552	      version: runtime.version,
553	      message: runtime.root
554	        ? 'OpenClaw runtime is ready. Gateway is stopped.'
555	        : `Bundled OpenClaw runtime is missing. Expected: ${runtime.expectedPathHint}`,
556	      canRetry: !runtime.root,
557	    });
558	  }
559	
560	  async restartGateway(): Promise<OpenClawEngineStatus> {
561	    console.log('[OpenClaw] restartGateway: stopping existing gateway...');
562	    await this.stopGateway();
563	    // Reset restart counter on manual restart so user can always retry
564	    this.gatewayRestartAttempt = 0;
565	    console.log('[OpenClaw] restartGateway: starting gateway with new env...');
566	    return this.startGateway();
567	  }
568	
569	  private resolveRuntimeMetadata(): RuntimeMetadata {
570	    const candidateRoots = app.isPackaged
571	      ? [path.join(process.resourcesPath, 'cfmind')]
572	      : [
573	          path.join(app.getAppPath(), 'vendor', 'openclaw-runtime', 'current'),
574	          path.join(process.cwd(), 'vendor', 'openclaw-runtime', 'current'),
575	        ];
576	
577	    const runtimeRoot = findPath(candidateRoots);
578	    const expectedPathHint = app.isPackaged
579	      ? path.join(process.resourcesPath, 'cfmind')
580	      : path.join(app.getAppPath(), 'vendor', 'openclaw-runtime', 'current');
581	
582	    if (!runtimeRoot) {
583	      return {
584	        root: null,
585	        version: null,
586	        expectedPathHint,
587	      };
588	    }
589	
590	    return {
591	      root: runtimeRoot,
592	      version: this.readRuntimeVersion(runtimeRoot) || DEFAULT_OPENCLAW_VERSION,
593	      expectedPathHint,
594	    };
595	  }
596	
597	  private readRuntimeVersion(runtimeRoot: string): string | null {
598	    const fromRootPackage = parseJsonFile<{ version?: string }>(path.join(runtimeRoot, 'package.json'))?.version;
599	    if (typeof fromRootPackage === 'string' && fromRootPackage.trim()) {
```

## Assistant

<thinking>
Now let me look at the gateway client connection part of the openclawRuntimeAdapter.ts to understand how it connects and communicates with the OpenClaw gateway.
</thinking>

**Tool Call: grep -n "ensureGatewayClientReady|createGatewayClient|connectGateway|ws://|WebSocket|gatewayClient\.|GatewayClient" /Users/zhoutongsheng/Code/my/LobsterAI/src/main/libs/agentEngine/openclawRuntimeAdapter.ts**
Status: Completed

51:type GatewayClientLike = {
61:type GatewayClientCtor = new (options: Record<string, unknown>) => GatewayClientLike;
577:  private gatewayClient: GatewayClientLike | null = null;
580:  /** Holds the client between start() and onHelloOk so stopGatewayClient can clean it up. */
581:  private pendingGatewayClient: GatewayClientLike | null = null;
583:  /** Serializes concurrent calls to ensureGatewayClientReady to prevent duplicate clients. */
819:   * Ensure the gateway WebSocket client is connected.
824:  async connectGatewayIfNeeded(): Promise<void> {
826:      console.log('[ChannelSync] connectGatewayIfNeeded: gateway client already exists, skipping');
829:    console.log('[ChannelSync] connectGatewayIfNeeded: no gateway client, initializing...');
831:      await this.ensureGatewayClientReady();
832:      console.log('[ChannelSync] connectGatewayIfNeeded: gateway client ready, starting channel polling');
835:      console.error('[ChannelSync] connectGatewayIfNeeded: failed to initialize gateway client:', error);
841:   * Force-reconnect the gateway WebSocket client.
843:   * Unlike `connectGatewayIfNeeded`, this always tears down the old client first
846:  async reconnectGateway(): Promise<void> {
847:    console.log('[ChannelSync] reconnectGateway: tearing down old client and reconnecting...');
848:    this.stopGatewayClient();
850:      await this.ensureGatewayClientReady();
851:      console.log('[ChannelSync] reconnectGateway: gateway client ready, starting channel polling');
854:      console.error('[ChannelSync] reconnectGateway: failed to initialize gateway client:', error);
860:   * Explicitly disconnect the gateway WebSocket client.
865:  disconnectGatewayClient(): void {
866:    console.log('[ChannelSync] disconnectGatewayClient: explicitly tearing down gateway client');
867:    this.stopGatewayClient();
874:   * have been delivered via WebSocket events.
906:      const result = await this.gatewayClient.request('sessions.list', params);
1177:    await this.ensureGatewayClientReady();
1223:    const client = this.requireGatewayClient();
1291:    const client = this.requireGatewayClient();
1379:  private async ensureGatewayClientReady(): Promise<void> {
1385:    this.gatewayClientInitLock = this._ensureGatewayClientReadyImpl();
1393:  private async _ensureGatewayClientReadyImpl(): Promise<void> {
1394:    console.log('[ChannelSync] ensureGatewayClientReady: starting engine gateway...');
1396:    console.log('[ChannelSync] ensureGatewayClientReady: engine phase=', engineStatus.phase, 'message=', engineStatus.message);
1403:    console.log('[ChannelSync] ensureGatewayClientReady: connection info — url=', connection.url ? '✓' : '✗', 'token=', connection.token ? '✓' : '✗', 'version=', connection.version, 'clientEntryPath=', connection.clientEntryPath ? '✓' : '✗');
1416:    console.log('[ChannelSync] ensureGatewayClientReady: needsNewClient=', needsNewClient, 'hasExistingClient=', !!this.gatewayClient);
1422:    this.stopGatewayClient();
1423:    console.log('[ChannelSync] ensureGatewayClientReady: creating gateway client, url=', connection.url);
1424:    await this.createGatewayClient(connection);
1425:    console.log('[ChannelSync] ensureGatewayClientReady: createGatewayClient returned, waiting for handshake...');
1429:    console.log('[ChannelSync] ensureGatewayClientReady: gateway client created and ready');
1436:  private async createGatewayClient(connection: OpenClawGatewayConnectionInfo): Promise<void> {
1437:    const GatewayClient = await this.loadGatewayClientCtor(connection.clientEntryPath);
1459:    const client = new GatewayClient({
1469:        console.log('[ChannelSync] GatewayClient: onHelloOk — handshake succeeded');
1481:        console.error('[ChannelSync] GatewayClient: onConnectError —', error.message);
1485:        console.log('[ChannelSync] GatewayClient: onClose — code:', _code, 'reason:', reason, 'settled:', settled);
1488:          // ensureGatewayClientReady call creates a fresh one instead of reusing
1490:          this.pendingGatewayClient = null;
1495:        // If stopGatewayClient() triggered this onClose, don't do anything —
1510:        this.stopGatewayClient();
1526:    // for stopGatewayClient() cleanup if start() fails synchronously.
1527:    this.pendingGatewayClient = client;
1531:  private stopGatewayClient(): void {
1537:    const clientToStop = this.gatewayClient ?? this.pendingGatewayClient;
1544:    this.pendingGatewayClient = null;
1686:    this.stopGatewayClient();
1731:      // connectGatewayIfNeeded checks if client already exists, so safe to call
1732:      await this.connectGatewayIfNeeded();
1832:  private async loadGatewayClientCtor(clientEntryPath: string): Promise<GatewayClientCtor> {
1836:    const direct = loaded.GatewayClient;
1838:      return direct as GatewayClientCtor;
1854:      if (maybeCtor.name === 'GatewayClient') {
1855:        return candidate as GatewayClientCtor;
1862:        return candidate as GatewayClientCtor;
3954:  private requireGatewayClient(): GatewayClientLike {
3965:  getGatewayClient(): GatewayClientLike | null {
4002:   * Resolves when the WebSocket connection is established and authenticated.
4005:    await this.ensureGatewayClientReady();

**Tool Call: grep -n "client\.request|chat\.start|chat\.continue|chat\.stop|chat\.history|agent\." /Users/zhoutongsheng/Code/my/LobsterAI/src/main/libs/agentEngine/openclawRuntimeAdapter.ts**
Status: Completed

309: * Extract file paths from assistant "message" tool calls in chat.history.
349: * Extract and concatenate all assistant text from the current turn in chat.history.
675:      const history = await client.request<{ messages?: unknown[] }>('chat.history', {
722:   * Fallback for fetchSessionByKey when chat.history returns no messages.
726:   * session entry remains in sessions.json. In that case chat.history cannot
930:        // Skip gateway sessions belonging to a previously-bound agent.
1034:        void client.request('chat.abort', {
1081:    void client.request('exec.approval.resolve', {
1236:      const sendResult = await client.request<Record<string, unknown>>('chat.send', {
1295:      const history = await client.request<{ messages?: unknown[] }>('chat.history', {
1301:      console.warn('[OpenClawRuntime] chat.history check failed, continuing without history guard:', error);
2683:        '[OpenClawRuntime] handleChatFinal: final payload had no text, falling back to chat.history sync',
2976:   * Reconcile local session messages with the authoritative gateway chat.history.
2982:   * expose them in chat.history.
3009:      const history = await client.request<{ messages?: unknown[] }>('chat.history', {
3148:        const history = await client.request<{ messages?: unknown[] }>('chat.history', {
3153:        console.log('[Debug:syncFinal] chat.history returned', msgCount, 'messages', `afterDelay=${delayMs}`);
3296:      console.warn('[OpenClawRuntime] chat.history sync after final failed:', error);
3347:    // `chat.history` is byte-bounded in OpenClaw, so the returned window can slide
3446:   * Sync user messages from gateway chat.history that haven't been added to the local store yet.
3455:   * because OpenClaw's `chat.history` window can slide due to byte limits well before
3501:    // order. This handles the race condition where gateway chat.history lags behind
3829:        const history = await client.request<{ messages?: unknown[] }>('chat.history', {
3834:        console.log('[Debug:prefetch] chat.history returned', msgCount, 'messages (attempt', attempt, ')');

## Assistant

<thinking>
Now let me look at the actual gateway protocol - how the `chat.send`, `chat.history`, `chat.abort`, `exec.approval.resolve` are called, and what the WebSocket/gateway interface looks like.
</thinking>

**Tool Call: Read src/main/libs/agentEngine/openclawRuntimeAdapter.ts (1436 - 1555)**
Status: Completed

```
1436	  private async createGatewayClient(connection: OpenClawGatewayConnectionInfo): Promise<void> {
1437	    const GatewayClient = await this.loadGatewayClientCtor(connection.clientEntryPath);
1438	
1439	    let resolveReady: (() => void) | null = null;
1440	    let rejectReady: ((error: Error) => void) | null = null;
1441	    let settled = false;
1442	
1443	    this.gatewayReadyPromise = new Promise<void>((resolve, reject) => {
1444	      resolveReady = resolve;
1445	      rejectReady = reject;
1446	    });
1447	
1448	    const settleResolve = () => {
1449	      if (settled) return;
1450	      settled = true;
1451	      resolveReady?.();
1452	    };
1453	    const settleReject = (error: Error) => {
1454	      if (settled) return;
1455	      settled = true;
1456	      rejectReady?.(error);
1457	    };
1458	
1459	    const client = new GatewayClient({
1460	      url: connection.url,
1461	      token: connection.token,
1462	      clientDisplayName: 'LobsterAI',
1463	      clientVersion: app.getVersion(),
1464	      mode: 'backend',
1465	      caps: [OPENCLAW_GATEWAY_TOOL_EVENTS_CAP],
1466	      role: 'operator',
1467	      scopes: ['operator.admin'],
1468	      onHelloOk: () => {
1469	        console.log('[ChannelSync] GatewayClient: onHelloOk — handshake succeeded');
1470	        // Expose the client only after the connect handshake completes.
1471	        // Setting gatewayClient earlier would let concurrent code send
1472	        // request frames before the connect frame, causing 1008 rejection.
1473	        this.gatewayClient = client;
1474	        this.gatewayClientVersion = connection.version;
1475	        this.gatewayClientEntryPath = connection.clientEntryPath;
1476	        settleResolve();
1477	        this.lastTickTimestamp = Date.now();
1478	        this.startTickWatchdog();
1479	      },
1480	      onConnectError: (error: Error) => {
1481	        console.error('[ChannelSync] GatewayClient: onConnectError —', error.message);
1482	        settleReject(error);
1483	      },
1484	      onClose: (_code: number, reason: string) => {
1485	        console.log('[ChannelSync] GatewayClient: onClose — code:', _code, 'reason:', reason, 'settled:', settled);
1486	        if (!settled) {
1487	          // Handshake never completed — clean up the pending client so the next
1488	          // ensureGatewayClientReady call creates a fresh one instead of reusing
1489	          // this broken instance forever.
1490	          this.pendingGatewayClient = null;
1491	          settleReject(new Error(reason || 'OpenClaw gateway disconnected before handshake'));
1492	          return;
1493	        }
1494	
1495	        // If stopGatewayClient() triggered this onClose, don't do anything —
1496	        // the caller is already handling cleanup and may be creating a new client.
1497	        if (this.gatewayStoppingIntentionally) {
1498	          return;
1499	        }
1500	
1501	        console.warn('[OpenClawRuntime] gateway WS disconnected — code:', _code, 'reason:', reason);
1502	        const disconnectedError = new Error(reason || 'OpenClaw gateway client disconnected');
1503	        const activeSessionIds = Array.from(this.activeTurns.keys());
1504	        activeSessionIds.forEach((sessionId) => {
1505	          this.store.updateSession(sessionId, { status: 'error' });
1506	          this.emit('error', sessionId, disconnectedError.message);
1507	          this.cleanupSessionTurn(sessionId);
1508	          this.rejectTurn(sessionId, disconnectedError);
1509	        });
1510	        this.stopGatewayClient();
1511	        this.gatewayReadyPromise = Promise.reject(disconnectedError);
1512	        this.gatewayReadyPromise.catch(() => {
1513	          // suppress unhandled rejection noise; auto-reconnect will re-establish
1514	        });
1515	
1516	        // Auto-reconnect after unexpected disconnect
1517	        this.scheduleGatewayReconnect();
1518	      },
1519	      onEvent: (event: GatewayEventFrame) => {
1520	        this.handleGatewayEvent(event);
1521	      },
1522	    });
1523	
1524	    // gatewayClient/version/entryPath are now set inside onHelloOk,
1525	    // after the connect handshake succeeds. We only keep a local ref
1526	    // for stopGatewayClient() cleanup if start() fails synchronously.
1527	    this.pendingGatewayClient = client;
1528	    client.start();
1529	  }
1530	
1531	  private stopGatewayClient(): void {
1532	    this.gatewayStoppingIntentionally = true;
1533	    this.stopChannelPolling();
1534	    this.cancelGatewayReconnect();
1535	    this.stopTickWatchdog();
1536	    // Stop whichever client exists — the promoted one or the pending one.
1537	    const clientToStop = this.gatewayClient ?? this.pendingGatewayClient;
1538	    try {
1539	      clientToStop?.stop();
1540	    } catch (error) {
1541	      console.warn('[OpenClawRuntime] Failed to stop gateway client:', error);
1542	    }
1543	    this.gatewayClient = null;
1544	    this.pendingGatewayClient = null;
1545	    this.gatewayClientVersion = null;
1546	    this.gatewayClientEntryPath = null;
1547	    this.gatewayReadyPromise = null;
1548	    this.channelSessionSync?.clearCache();
1549	    this.knownChannelSessionIds.clear();
1550	    this.heartbeatSessionKeys.clear();
1551	    this.stoppedSessions.clear();
1552	    this.browserPrewarmAttempted = false;
1553	    this.lastTickTimestamp = 0;
1554	    // Clear messageUpdate throttle state
1555	    for (const timer of this.pendingMessageUpdateTimer.values()) {
```

**Tool Call: Read src/main/libs/agentEngine/openclawRuntimeAdapter.ts (1832 - 1911)**
Status: Completed

```
1832	  private async loadGatewayClientCtor(clientEntryPath: string): Promise<GatewayClientCtor> {
1833	    // Use require() with file path directly. TypeScript's CJS output downgrades
1834	    // dynamic import() to require(), which doesn't support file:// URLs.
1835	    const loaded = require(clientEntryPath) as Record<string, unknown>;
1836	    const direct = loaded.GatewayClient;
1837	    if (typeof direct === 'function') {
1838	      return direct as GatewayClientCtor;
1839	    }
1840	
1841	    const exportedValues = Object.values(loaded);
1842	    for (const candidate of exportedValues) {
1843	      if (typeof candidate !== 'function') {
1844	        continue;
1845	      }
1846	      const maybeCtor = candidate as {
1847	        name?: string;
1848	        prototype?: {
1849	          start?: unknown;
1850	          stop?: unknown;
1851	          request?: unknown;
1852	        };
1853	      };
1854	      if (maybeCtor.name === 'GatewayClient') {
1855	        return candidate as GatewayClientCtor;
1856	      }
1857	      const proto = maybeCtor.prototype;
1858	      if (proto
1859	        && typeof proto.start === 'function'
1860	        && typeof proto.stop === 'function'
1861	        && typeof proto.request === 'function') {
1862	        return candidate as GatewayClientCtor;
1863	      }
1864	    }
1865	
1866	    const exportKeysPreview = Object.keys(loaded).slice(0, 20).join(', ');
1867	    throw new Error(
1868	      `Invalid OpenClaw gateway client module: ${clientEntryPath} (exports: ${exportKeysPreview || 'none'})`,
1869	    );
1870	  }
1871	
1872	  private handleGatewayEvent(event: GatewayEventFrame): void {
1873	    if (event.event === 'tick') {
1874	      this.lastTickTimestamp = Date.now();
1875	      return;
1876	    }
1877	
1878	    if (event.event === 'chat') {
1879	      this.handleChatEvent(event.payload, event.seq);
1880	      return;
1881	    }
1882	
1883	    if (event.event === 'agent') {
1884	      // Process assistant text updates here (before handleAgentEvent) because
1885	      // handleAgentEvent may enqueue events when sessionId mapping isn't ready.
1886	      this.processAgentAssistantText(event.payload);
1887	      this.handleAgentEvent(event.payload, event.seq);
1888	      return;
1889	    }
1890	
1891	    if (event.event === 'exec.approval.requested') {
1892	      this.handleApprovalRequested(event.payload);
1893	      return;
1894	    }
1895	
1896	    if (event.event === 'exec.approval.resolved') {
1897	      this.handleApprovalResolved(event.payload);
1898	    }
1899	
1900	    if (event.event === 'cron') {
1901	      console.debug('[OpenClawRuntime] received cron event:', JSON.stringify(event));
1902	    }
1903	  }
1904	
1905	  private handleAgentEvent(payload: unknown, seq?: number): void {
1906	    if (!isRecord(payload)) return;
1907	    const agentPayload = payload as AgentEventPayload;
1908	    const runId = typeof agentPayload.runId === 'string' ? agentPayload.runId.trim() : '';
1909	    const sessionKey = typeof agentPayload.sessionKey === 'string' ? agentPayload.sessionKey.trim() : '';
1910	    const stream = typeof agentPayload.stream === 'string' ? agentPayload.stream : '';
1911	
```

## Assistant

<thinking>
Now let me look at the key RPC methods (chat.send, chat.abort, etc.) to understand the protocol in detail.
</thinking>

**Tool Call: Read src/main/libs/agentEngine/openclawRuntimeAdapter.ts (1285 - 1384)**
Status: Completed

```
1285	      if (prompt.trim()) {
1286	        sections.push(`[Current user request]\n${prompt}`);
1287	      }
1288	      return sections.join('\n\n');
1289	    }
1290	
1291	    const client = this.requireGatewayClient();
1292	    const sessionKey = this.toSessionKey(sessionId, agentId);
1293	    let hasHistory = false;
1294	    try {
1295	      const history = await client.request<{ messages?: unknown[] }>('chat.history', {
1296	        sessionKey,
1297	        limit: 1,
1298	      });
1299	      hasHistory = Array.isArray(history?.messages) && history.messages.length > 0;
1300	    } catch (error) {
1301	      console.warn('[OpenClawRuntime] chat.history check failed, continuing without history guard:', error);
1302	    }
1303	
1304	    this.bridgedSessions.add(sessionId);
1305	
1306	    if (!hasHistory) {
1307	      const session = this.store.getSession(sessionId);
1308	      if (session) {
1309	        const bridgePrefix = this.buildBridgePrefix(session.messages, prompt);
1310	        if (bridgePrefix) {
1311	          sections.push(bridgePrefix);
1312	        }
1313	      }
1314	    }
1315	
1316	    if (prompt.trim()) {
1317	      sections.push(`[Current user request]\n${prompt}`);
1318	    }
1319	    return sections.join('\n\n');
1320	  }
1321	
1322	  private buildSystemPromptPrefix(systemPrompt: string): string {
1323	    return [
1324	      '[LobsterAI system instructions]',
1325	      'Apply the instructions below as the highest-priority guidance for this session.',
1326	      'If earlier LobsterAI system instructions exist, replace them with this version.',
1327	      systemPrompt,
1328	    ].join('\n');
1329	  }
1330	
1331	  private buildBridgePrefix(messages: CoworkMessage[], currentPrompt: string): string {
1332	    const normalizedCurrentPrompt = currentPrompt.trim();
1333	    if (!normalizedCurrentPrompt) return '';
1334	
1335	    const source = messages
1336	      .filter((message) => {
1337	        if (message.type !== 'user' && message.type !== 'assistant') {
1338	          return false;
1339	        }
1340	        if (!message.content.trim()) {
1341	          return false;
1342	        }
1343	        if (message.metadata?.isThinking) {
1344	          return false;
1345	        }
1346	        return true;
1347	      })
1348	      .map((message) => ({
1349	        type: message.type,
1350	        content: message.content.trim(),
1351	      }));
1352	
1353	    if (source.length === 0) {
1354	      return '';
1355	    }
1356	
1357	    if (source[source.length - 1]?.type === 'user'
1358	      && source[source.length - 1]?.content === normalizedCurrentPrompt) {
1359	      source.pop();
1360	    }
1361	
1362	    const recent = source.slice(-BRIDGE_MAX_MESSAGES);
1363	    if (recent.length === 0) {
1364	      return '';
1365	    }
1366	
1367	    const lines = recent.map((entry) => {
1368	      const role = entry.type === 'user' ? 'User' : 'Assistant';
1369	      return `${role}: ${truncate(entry.content, BRIDGE_MAX_MESSAGE_CHARS)}`;
1370	    });
1371	
1372	    return [
1373	      '[Context bridge from previous LobsterAI conversation]',
1374	      'Use this prior context for continuity. Focus your final answer on the current request.',
1375	      ...lines,
1376	    ].join('\n');
1377	  }
1378	
1379	  private async ensureGatewayClientReady(): Promise<void> {
1380	    // Serialize concurrent calls: if another init is already in progress, wait for it.
1381	    if (this.gatewayClientInitLock) {
1382	      await this.gatewayClientInitLock;
1383	      return;
1384	    }
```

**Tool Call: Read src/main/libs/agentEngine/openclawRuntimeAdapter.ts (1180 - 1259)**
Status: Completed

```
1180	    const runId = randomUUID();
1181	    const turnToken = this.nextTurnToken(sessionId);
1182	    const outboundMessage = await this.buildOutboundPrompt(
1183	      sessionId,
1184	      prompt,
1185	      options.systemPrompt ?? session.systemPrompt,
1186	      agentId,
1187	    );
1188	    const completionPromise = new Promise<void>((resolve, reject) => {
1189	      this.pendingTurns.set(sessionId, { resolve, reject });
1190	    });
1191	    this.activeTurns.set(sessionId, {
1192	      sessionId,
1193	      sessionKey,
1194	      runId,
1195	      turnToken,
1196	      knownRunIds: new Set([runId]),
1197	      assistantMessageId: null,
1198	      committedAssistantText: '',
1199	      currentAssistantSegmentText: '',
1200	      currentText: '',
1201	      agentAssistantTextLength: 0,
1202	      currentContentText: '',
1203	      currentContentBlocks: [],
1204	      sawNonTextContentBlocks: false,
1205	      textStreamMode: 'unknown',
1206	      toolUseMessageIdByToolCallId: new Map(),
1207	      toolResultMessageIdByToolCallId: new Map(),
1208	      toolResultTextByToolCallId: new Map(),
1209	      stopRequested: false,
1210	      pendingUserSync: false,
1211	      bufferedChatPayloads: [],
1212	      bufferedAgentPayloads: [],
1213	    });
1214	    this.sessionIdByRunId.set(runId, sessionId);
1215	
1216	    // Start client-side timeout watchdog.
1217	    // OpenClaw gateway has a known issue where embedded run timeouts may not
1218	    // produce a WS abort/final event (the subscription is torn down before the
1219	    // lifecycle event fires). This timer fires slightly after the server-side
1220	    // timeout to recover the UI from a stuck "running" state.
1221	    this.startTurnTimeoutWatchdog(sessionId);
1222	
1223	    const client = this.requireGatewayClient();
1224	    try {
1225	      console.log('[OpenClawRuntime] chat.send params:', { sessionKey, messageLength: outboundMessage.length, runId });
1226	      const attachments = options.imageAttachments?.length
1227	        ? options.imageAttachments.map((img) => ({
1228	          type: 'image',
1229	          mimeType: img.mimeType,
1230	          content: img.base64Data,
1231	        }))
1232	        : undefined;
1233	      if (attachments) {
1234	        console.log('[OpenClawRuntime] chat.send with attachments:', attachments.length, 'images,', attachments.map(a => ({ type: a.type, mimeType: a.mimeType, contentLength: a.content?.length ?? 0 })));
1235	      }
1236	      const sendResult = await client.request<Record<string, unknown>>('chat.send', {
1237	        sessionKey,
1238	        message: outboundMessage,
1239	        deliver: false,
1240	        idempotencyKey: runId,
1241	        ...(attachments ? { attachments } : {}),
1242	      });
1243	      const returnedRunId = typeof sendResult?.runId === 'string' ? sendResult.runId.trim() : '';
1244	      if (returnedRunId) {
1245	        this.bindRunIdToTurn(sessionId, returnedRunId);
1246	      }
1247	    } catch (error) {
1248	      this.cleanupSessionTurn(sessionId);
1249	      this.store.updateSession(sessionId, { status: 'error' });
1250	      const message = error instanceof Error ? error.message : String(error);
1251	      this.emit('error', sessionId, message);
1252	      this.rejectTurn(sessionId, new Error(message));
1253	      throw error;
1254	    }
1255	
1256	    await completionPromise;
1257	  }
1258	
1259	  private async buildOutboundPrompt(
```

## Assistant

<thinking>
Now I have a good understanding of the architecture. Let me also look at the key IPC channels and what OpenClaw gateway client entry path is.
</thinking>

**Tool Call: grep -n "resolveGatewayClientEntry|gateway-client|clientEntryPath" /Users/zhoutongsheng/Code/my/LobsterAI/src/main/libs/openclawEngineManager.ts**
Status: Completed

43:  clientEntryPath: string | null;
261:    const clientEntryPath = runtime.root ? this.resolveGatewayClientEntry(runtime.root) : null;
268:      clientEntryPath,
967:  private resolveGatewayClientEntry(runtimeRoot: string): string | null {

**Tool Call: Read src/main/libs/agentEngine/openclawRuntimeAdapter.ts (1025 - 1084)**
Status: Completed

```
1025	  }
1026	
1027	  stopSession(sessionId: string): void {
1028	    const turn = this.activeTurns.get(sessionId);
1029	    if (turn) {
1030	      turn.stopRequested = true;
1031	      this.manuallyStoppedSessions.add(sessionId);
1032	      const client = this.gatewayClient;
1033	      if (client) {
1034	        void client.request('chat.abort', {
1035	          sessionKey: turn.sessionKey,
1036	          runId: turn.runId,
1037	        }).catch((error) => {
1038	          console.warn('[OpenClawRuntime] Failed to abort chat run:', error);
1039	        });
1040	      }
1041	    }
1042	
1043	    // Record the stop timestamp so that late-arriving gateway events
1044	    // (e.g. from POPO/Telegram channels) don't re-create the ActiveTurn.
1045	    this.stoppedSessions.set(sessionId, Date.now());
1046	
1047	    this.cleanupSessionTurn(sessionId);
1048	    this.clearPendingApprovalsBySession(sessionId);
1049	    this.store.updateSession(sessionId, { status: 'idle' });
1050	    this.emit('sessionStopped', sessionId);
1051	    this.resolveTurn(sessionId);
1052	  }
1053	
1054	  stopAllSessions(): void {
1055	    const activeSessionIds = Array.from(this.activeTurns.keys());
1056	    activeSessionIds.forEach((sessionId) => {
1057	      this.stopSession(sessionId);
1058	    });
1059	  }
1060	
1061	  respondToPermission(requestId: string, result: PermissionResult): void {
1062	    const pending = this.pendingApprovals.get(requestId);
1063	    if (!pending) {
1064	      return;
1065	    }
1066	
1067	    const decision = result.behavior !== 'allow' ? 'deny'
1068	      : pending.allowAlways ? 'allow-always'
1069	      : 'allow-once';
1070	    const client = this.gatewayClient;
1071	    if (!client) {
1072	      this.pendingApprovals.delete(requestId);
1073	      return;
1074	    }
1075	
1076	    const sessionId = pending.sessionId;
1077	    // Only schedule continuation for user-initiated approvals (desktop modal),
1078	    // not for auto-approved commands (allowAlways).
1079	    const needsContinuation = !pending.allowAlways;
1080	
1081	    void client.request('exec.approval.resolve', {
1082	      id: requestId,
1083	      decision,
1084	    }).then(() => {
```

## Assistant

<thinking>
Now let me look at the `resolveGatewayClientEntry` method and the gateway client entry point.
</thinking>

**Tool Call: Read src/main/libs/openclawEngineManager.ts (967 - 1026)**
Status: Completed

```
967	  private resolveGatewayClientEntry(runtimeRoot: string): string | null {
968	    const distRoots = [
969	      path.join(runtimeRoot, 'dist'),
970	      path.join(runtimeRoot, 'gateway.asar', 'dist'),
971	    ];
972	
973	    for (const distRoot of distRoots) {
974	      const clientEntry = this.findGatewayClientEntryFromDistRoot(distRoot);
975	      if (clientEntry) {
976	        return clientEntry;
977	      }
978	    }
979	
980	    return null;
981	  }
982	
983	  private findGatewayClientEntryFromDistRoot(distRoot: string): string | null {
984	    const gatewayClient = path.join(distRoot, 'gateway', 'client.js');
985	    if (fs.existsSync(gatewayClient)) {
986	      return gatewayClient;
987	    }
988	
989	    const directClient = path.join(distRoot, 'client.js');
990	    if (fs.existsSync(directClient)) {
991	      return directClient;
992	    }
993	
994	    try {
995	      if (!fs.existsSync(distRoot) || !fs.statSync(distRoot).isDirectory()) {
996	        return null;
997	      }
998	
999	      const candidates = fs.readdirSync(distRoot)
1000	        .filter((name) => /^client(?:-.*)?\.js$/i.test(name))
1001	        .sort();
1002	      if (candidates.length > 0) {
1003	        return path.join(distRoot, candidates[0]);
1004	      }
1005	    } catch {
1006	      // ignore
1007	    }
1008	
1009	    return null;
1010	  }
1011	
1012	  private ensureGatewayToken(): string {
1013	    try {
1014	      const existing = fs.readFileSync(this.gatewayTokenPath, 'utf8').trim();
1015	      if (existing) {
1016	        return existing;
1017	      }
1018	    } catch {
1019	      // ignore
1020	    }
1021	
1022	    const token = crypto.randomBytes(24).toString('hex');
1023	    ensureDir(path.dirname(this.gatewayTokenPath));
1024	    fs.writeFileSync(this.gatewayTokenPath, token, 'utf8');
1025	    return token;
1026	  }
```

**Tool Call: Find `/Users/zhoutongsheng/Code/my/LobsterAI` `vendor/openclaw-runtime/current/**/*client*`**
Status: Completed

No files found

## Assistant

<thinking>
Now I have a comprehensive understanding of the architecture. Let me also quickly check if there's any documentation or README about the OpenClaw gateway protocol.
</thinking>

**Tool Call: Find `/Users/zhoutongsheng/Code/my/LobsterAI/vendor/openclaw-runtime` `**/*.md`**
Status: Completed

vendor/openclaw-runtime/mac-arm64/CHANGELOG.md
vendor/openclaw-runtime/mac-arm64/docs/date-time.md
vendor/openclaw-runtime/mac-arm64/docs/design/kilo-gateway-integration.md
vendor/openclaw-runtime/mac-arm64/docs/platforms/android.md
vendor/openclaw-runtime/mac-arm64/docs/platforms/oracle.md
vendor/openclaw-runtime/mac-arm64/docs/platforms/windows.md
vendor/openclaw-runtime/mac-arm64/docs/platforms/linux.md
vendor/openclaw-runtime/mac-arm64/docs/platforms/raspberry-pi.md
vendor/openclaw-runtime/mac-arm64/docs/platforms/mac/menu-bar.md
vendor/openclaw-runtime/mac-arm64/docs/platforms/mac/permissions.md
vendor/openclaw-runtime/mac-arm64/docs/platforms/mac/bundled-gateway.md
vendor/openclaw-runtime/mac-arm64/docs/platforms/mac/canvas.md
vendor/openclaw-runtime/mac-arm64/docs/platforms/mac/webchat.md
vendor/openclaw-runtime/mac-arm64/docs/platforms/mac/voice-overlay.md
vendor/openclaw-runtime/mac-arm64/docs/platforms/mac/xpc.md
vendor/openclaw-runtime/mac-arm64/docs/platforms/mac/voicewake.md
vendor/openclaw-runtime/mac-arm64/docs/platforms/mac/release.md
vendor/openclaw-runtime/mac-arm64/docs/platforms/mac/skills.md
vendor/openclaw-runtime/mac-arm64/docs/platforms/mac/logging.md
vendor/openclaw-runtime/mac-arm64/docs/platforms/mac/signing.md
vendor/openclaw-runtime/mac-arm64/docs/platforms/mac/icon.md
vendor/openclaw-runtime/mac-arm64/docs/platforms/mac/peekaboo.md
vendor/openclaw-runtime/mac-arm64/docs/platforms/mac/dev-setup.md
vendor/openclaw-runtime/mac-arm64/docs/platforms/mac/child-process.md
vendor/openclaw-runtime/mac-arm64/docs/platforms/mac/health.md
vendor/openclaw-runtime/mac-arm64/docs/platforms/mac/remote.md
vendor/openclaw-runtime/mac-arm64/docs/platforms/macos.md
vendor/openclaw-runtime/mac-arm64/docs/platforms/index.md
vendor/openclaw-runtime/mac-arm64/docs/platforms/ios.md
vendor/openclaw-runtime/mac-arm64/docs/platforms/digitalocean.md
vendor/openclaw-runtime/mac-arm64/docs/network.md
vendor/openclaw-runtime/mac-arm64/docs/tools/apply-patch.md
vendor/openclaw-runtime/mac-arm64/docs/tools/slash-commands.md
vendor/openclaw-runtime/mac-arm64/docs/tools/lobster.md
vendor/openclaw-runtime/mac-arm64/docs/tools/clawhub.md
vendor/openclaw-runtime/mac-arm64/docs/tools/exec-approvals.md
vendor/openclaw-runtime/mac-arm64/docs/tools/plugin.md
vendor/openclaw-runtime/mac-arm64/docs/tools/thinking.md
vendor/openclaw-runtime/mac-arm64/docs/tools/llm-task.md
vendor/openclaw-runtime/mac-arm64/docs/tools/browser.md
vendor/openclaw-runtime/mac-arm64/docs/tools/acp-agents.md
vendor/openclaw-runtime/mac-arm64/docs/tools/browser-login.md
vendor/openclaw-runtime/mac-arm64/docs/tools/firecrawl.md
vendor/openclaw-runtime/mac-arm64/docs/tools/agent-send.md
vendor/openclaw-runtime/mac-arm64/docs/tools/subagents.md
vendor/openclaw-runtime/mac-arm64/docs/tools/skills.md
vendor/openclaw-runtime/mac-arm64/docs/tools/index.md
vendor/openclaw-runtime/mac-arm64/docs/tools/skills-config.md
vendor/openclaw-runtime/mac-arm64/docs/tools/browser-linux-troubleshooting.md
vendor/openclaw-runtime/mac-arm64/docs/tools/web.md
vendor/openclaw-runtime/mac-arm64/docs/tools/pdf.md
vendor/openclaw-runtime/mac-arm64/docs/tools/diffs.md
vendor/openclaw-runtime/mac-arm64/docs/tools/elevated.md
vendor/openclaw-runtime/mac-arm64/docs/tools/exec.md
vendor/openclaw-runtime/mac-arm64/docs/tools/multi-agent-sandbox-tools.md
vendor/openclaw-runtime/mac-arm64/docs/tools/reactions.md
vendor/openclaw-runtime/mac-arm64/docs/tools/creating-skills.md
vendor/openclaw-runtime/mac-arm64/docs/tools/loop-detection.md
vendor/openclaw-runtime/mac-arm64/docs/tools/chrome-extension.md
vendor/openclaw-runtime/mac-arm64/docs/install/uninstall.md
vendor/openclaw-runtime/mac-arm64/docs/install/fly.md
vendor/openclaw-runtime/mac-arm64/docs/install/bun.md
vendor/openclaw-runtime/mac-arm64/docs/install/installer.md
vendor/openclaw-runtime/mac-arm64/docs/install/node.md
vendor/openclaw-runtime/mac-arm64/docs/install/updating.md
vendor/openclaw-runtime/mac-arm64/docs/install/gcp.md
vendor/openclaw-runtime/mac-arm64/docs/install/nix.md
vendor/openclaw-runtime/mac-arm64/docs/install/index.md
vendor/openclaw-runtime/mac-arm64/docs/install/macos-vm.md
vendor/openclaw-runtime/mac-arm64/docs/install/development-channels.md
vendor/openclaw-runtime/mac-arm64/docs/install/ansible.md
vendor/openclaw-runtime/mac-arm64/docs/install/docker.md
vendor/openclaw-runtime/mac-arm64/docs/install/hetzner.md
vendor/openclaw-runtime/mac-arm64/docs/install/migrating.md
vendor/openclaw-runtime/mac-arm64/docs/install/podman.md
vendor/openclaw-runtime/mac-arm64/docs/install/exe-dev.md
vendor/openclaw-runtime/mac-arm64/docs/prose.md
vendor/openclaw-runtime/mac-arm64/docs/vps.md
vendor/openclaw-runtime/mac-arm64/docs/security/formal-verification.md
vendor/openclaw-runtime/mac-arm64/docs/security/CONTRIBUTING-THREAT-MODEL.md
vendor/openclaw-runtime/mac-arm64/docs/security/README.md
vendor/openclaw-runtime/mac-arm64/docs/security/THREAT-MODEL-ATLAS.md
vendor/openclaw-runtime/mac-arm64/docs/nodes/images.md
vendor/openclaw-runtime/mac-arm64/docs/nodes/troubleshooting.md
vendor/openclaw-runtime/mac-arm64/docs/nodes/camera.md
vendor/openclaw-runtime/mac-arm64/docs/nodes/location-command.md
vendor/openclaw-runtime/mac-arm64/docs/nodes/audio.md
vendor/openclaw-runtime/mac-arm64/docs/nodes/talk.md
vendor/openclaw-runtime/mac-arm64/docs/nodes/voicewake.md
vendor/openclaw-runtime/mac-arm64/docs/nodes/index.md
vendor/openclaw-runtime/mac-arm64/docs/nodes/media-understanding.md
vendor/openclaw-runtime/mac-arm64/docs/plugins/community.md
vendor/openclaw-runtime/mac-arm64/docs/plugins/voice-call.md
vendor/openclaw-runtime/mac-arm64/docs/plugins/manifest.md
vendor/openclaw-runtime/mac-arm64/docs/plugins/agent-tools.md
vendor/openclaw-runtime/mac-arm64/docs/plugins/zalouser.md
vendor/openclaw-runtime/mac-arm64/docs/web/tui.md
vendor/openclaw-runtime/mac-arm64/docs/web/dashboard.md
vendor/openclaw-runtime/mac-arm64/docs/web/webchat.md
vendor/openclaw-runtime/mac-arm64/docs/web/index.md
(Results are truncated. Consider using a more specific path or pattern.)

## Assistant

<thinking>
There's a `kilo-gateway-integration.md` file that might describe the protocol. Let me read it.
</thinking>

**Tool Call: Read vendor/openclaw-runtime/mac-arm64/docs/design/kilo-gateway-integration.md**
Status: Completed

```
1	# Kilo Gateway Provider Integration Design
2	
3	## Overview
4	
5	This document outlines the design for integrating "Kilo Gateway" as a first-class provider in OpenClaw, modeled after the existing OpenRouter implementation. Kilo Gateway uses an OpenAI-compatible completions API with a different base URL.
6	
7	## Design Decisions
8	
9	### 1. Provider Naming
10	
11	**Recommendation: `kilocode`**
12	
13	Rationale:
14	
15	- Matches the user config example provided (`kilocode` provider key)
16	- Consistent with existing provider naming patterns (e.g., `openrouter`, `opencode`, `moonshot`)
17	- Short and memorable
18	- Avoids confusion with generic "kilo" or "gateway" terms
19	
20	Alternative considered: `kilo-gateway` - rejected because hyphenated names are less common in the codebase and `kilocode` is more concise.
21	
22	### 2. Default Model Reference
23	
24	**Recommendation: `kilocode/anthropic/claude-opus-4.6`**
25	
26	Rationale:
27	
28	- Based on user config example
29	- Claude Opus 4.5 is a capable default model
30	- Explicit model selection avoids reliance on auto-routing
31	
32	### 3. Base URL Configuration
33	
34	**Recommendation: Hardcoded default with config override**
35	
36	- **Default Base URL:** `https://api.kilo.ai/api/gateway/`
37	- **Configurable:** Yes, via `models.providers.kilocode.baseUrl`
38	
39	This matches the pattern used by other providers like Moonshot, Venice, and Synthetic.
40	
41	### 4. Model Scanning
42	
43	**Recommendation: No dedicated model scanning endpoint initially**
44	
45	Rationale:
46	
47	- Kilo Gateway proxies to OpenRouter, so models are dynamic
48	- Users can manually configure models in their config
49	- If Kilo Gateway exposes a `/models` endpoint in the future, scanning can be added
50	
51	### 5. Special Handling
52	
53	**Recommendation: Inherit OpenRouter behavior for Anthropic models**
54	
55	Since Kilo Gateway proxies to OpenRouter, the same special handling should apply:
56	
57	- Cache TTL eligibility for `anthropic/*` models
58	- Extra params (cacheControlTtl) for `anthropic/*` models
59	- Transcript policy follows OpenRouter patterns
60	
61	## Files to Modify
62	
63	### Core Credential Management
64	
65	#### 1. `src/commands/onboard-auth.credentials.ts`
66	
67	Add:
68	
69	```typescript
70	export const KILOCODE_DEFAULT_MODEL_REF = "kilocode/anthropic/claude-opus-4.6";
71	
72	export async function setKilocodeApiKey(key: string, agentDir?: string) {
73	  upsertAuthProfile({
74	    profileId: "kilocode:default",
75	    credential: {
76	      type: "api_key",
77	      provider: "kilocode",
78	      key,
79	    },
80	    agentDir: resolveAuthAgentDir(agentDir),
81	  });
82	}
83	```
84	
85	#### 2. `src/agents/model-auth.ts`
86	
87	Add to `envMap` in `resolveEnvApiKey()`:
88	
89	```typescript
90	const envMap: Record<string, string> = {
91	  // ... existing entries
92	  kilocode: "KILOCODE_API_KEY",
93	};
94	```
95	
96	#### 3. `src/config/io.ts`
97	
98	Add to `SHELL_ENV_EXPECTED_KEYS`:
99	
100	```typescript
101	const SHELL_ENV_EXPECTED_KEYS = [
102	  // ... existing entries
103	  "KILOCODE_API_KEY",
104	];
105	```
106	
107	### Config Application
108	
109	#### 4. `src/commands/onboard-auth.config-core.ts`
110	
111	Add new functions:
112	
113	```typescript
114	export const KILOCODE_BASE_URL = "https://api.kilo.ai/api/gateway/";
115	
116	export function applyKilocodeProviderConfig(cfg: OpenClawConfig): OpenClawConfig {
117	  const models = { ...cfg.agents?.defaults?.models };
118	  models[KILOCODE_DEFAULT_MODEL_REF] = {
119	    ...models[KILOCODE_DEFAULT_MODEL_REF],
120	    alias: models[KILOCODE_DEFAULT_MODEL_REF]?.alias ?? "Kilo Gateway",
121	  };
122	
123	  const providers = { ...cfg.models?.providers };
124	  const existingProvider = providers.kilocode;
125	  const { apiKey: existingApiKey, ...existingProviderRest } = (existingProvider ?? {}) as Record<
126	    string,
127	    unknown
128	  > as { apiKey?: string };
129	  const resolvedApiKey = typeof existingApiKey === "string" ? existingApiKey : undefined;
130	  const normalizedApiKey = resolvedApiKey?.trim();
131	
132	  providers.kilocode = {
133	    ...existingProviderRest,
134	    baseUrl: KILOCODE_BASE_URL,
135	    api: "openai-completions",
136	    ...(normalizedApiKey ? { apiKey: normalizedApiKey } : {}),
137	  };
138	
139	  return {
140	    ...cfg,
141	    agents: {
142	      ...cfg.agents,
143	      defaults: {
144	        ...cfg.agents?.defaults,
145	        models,
146	      },
147	    },
148	    models: {
149	      mode: cfg.models?.mode ?? "merge",
150	      providers,
151	    },
152	  };
153	}
154	
155	export function applyKilocodeConfig(cfg: OpenClawConfig): OpenClawConfig {
156	  const next = applyKilocodeProviderConfig(cfg);
157	  const existingModel = next.agents?.defaults?.model;
158	  return {
159	    ...next,
160	    agents: {
161	      ...next.agents,
162	      defaults: {
163	        ...next.agents?.defaults,
164	        model: {
165	          ...(existingModel && "fallbacks" in (existingModel as Record<string, unknown>)
166	            ? {
167	                fallbacks: (existingModel as { fallbacks?: string[] }).fallbacks,
168	              }
169	            : undefined),
170	          primary: KILOCODE_DEFAULT_MODEL_REF,
171	        },
172	      },
173	    },
174	  };
175	}
176	```
177	
178	### Auth Choice System
179	
180	#### 5. `src/commands/onboard-types.ts`
181	
182	Add to `AuthChoice` type:
183	
184	```typescript
185	export type AuthChoice =
186	  // ... existing choices
187	  "kilocode-api-key";
188	// ...
189	```
190	
191	Add to `OnboardOptions`:
192	
193	```typescript
194	export type OnboardOptions = {
195	  // ... existing options
196	  kilocodeApiKey?: string;
197	  // ...
198	};
199	```
200	
201	#### 6. `src/commands/auth-choice-options.ts`
202	
203	Add to `AuthChoiceGroupId`:
204	
205	```typescript
206	export type AuthChoiceGroupId =
207	  // ... existing groups
208	  "kilocode";
209	// ...
210	```
211	
212	Add to `AUTH_CHOICE_GROUP_DEFS`:
213	
214	```typescript
215	{
216	  value: "kilocode",
217	  label: "Kilo Gateway",
218	  hint: "API key (OpenRouter-compatible)",
219	  choices: ["kilocode-api-key"],
220	},
221	```
222	
223	Add to `buildAuthChoiceOptions()`:
224	
225	```typescript
226	options.push({
227	  value: "kilocode-api-key",
228	  label: "Kilo Gateway API key",
229	  hint: "OpenRouter-compatible gateway",
230	});
231	```
232	
233	#### 7. `src/commands/auth-choice.preferred-provider.ts`
234	
235	Add mapping:
236	
237	```typescript
238	const PREFERRED_PROVIDER_BY_AUTH_CHOICE: Partial<Record<AuthChoice, string>> = {
239	  // ... existing mappings
240	  "kilocode-api-key": "kilocode",
241	};
242	```
243	
244	### Auth Choice Application
245	
246	#### 8. `src/commands/auth-choice.apply.api-providers.ts`
247	
248	Add import:
249	
250	```typescript
251	import {
252	  // ... existing imports
253	  applyKilocodeConfig,
254	  applyKilocodeProviderConfig,
255	  KILOCODE_DEFAULT_MODEL_REF,
256	  setKilocodeApiKey,
257	} from "./onboard-auth.js";
258	```
259	
260	Add handling for `kilocode-api-key`:
261	
262	```typescript
263	if (authChoice === "kilocode-api-key") {
264	  const store = ensureAuthProfileStore(params.agentDir, {
265	    allowKeychainPrompt: false,
266	  });
267	  const profileOrder = resolveAuthProfileOrder({
268	    cfg: nextConfig,
269	    store,
270	    provider: "kilocode",
271	  });
272	  const existingProfileId = profileOrder.find((profileId) => Boolean(store.profiles[profileId]));
273	  const existingCred = existingProfileId ? store.profiles[existingProfileId] : undefined;
274	  let profileId = "kilocode:default";
275	  let mode: "api_key" | "oauth" | "token" = "api_key";
276	  let hasCredential = false;
277	
278	  if (existingProfileId && existingCred?.type) {
279	    profileId = existingProfileId;
280	    mode =
281	      existingCred.type === "oauth" ? "oauth" : existingCred.type === "token" ? "token" : "api_key";
282	    hasCredential = true;
283	  }
284	
285	  if (!hasCredential && params.opts?.token && params.opts?.tokenProvider === "kilocode") {
286	    await setKilocodeApiKey(normalizeApiKeyInput(params.opts.token), params.agentDir);
287	    hasCredential = true;
288	  }
289	
290	  if (!hasCredential) {
291	    const envKey = resolveEnvApiKey("kilocode");
292	    if (envKey) {
293	      const useExisting = await params.prompter.confirm({
294	        message: `Use existing KILOCODE_API_KEY (${envKey.source}, ${formatApiKeyPreview(envKey.apiKey)})?`,
295	        initialValue: true,
296	      });
297	      if (useExisting) {
298	        await setKilocodeApiKey(envKey.apiKey, params.agentDir);
299	        hasCredential = true;
300	      }
301	    }
302	  }
303	
304	  if (!hasCredential) {
305	    const key = await params.prompter.text({
306	      message: "Enter Kilo Gateway API key",
307	      validate: validateApiKeyInput,
308	    });
309	    await setKilocodeApiKey(normalizeApiKeyInput(String(key)), params.agentDir);
310	    hasCredential = true;
311	  }
312	
313	  if (hasCredential) {
314	    nextConfig = applyAuthProfileConfig(nextConfig, {
315	      profileId,
316	      provider: "kilocode",
317	      mode,
318	    });
319	  }
320	  {
321	    const applied = await applyDefaultModelChoice({
322	      config: nextConfig,
323	      setDefaultModel: params.setDefaultModel,
324	      defaultModel: KILOCODE_DEFAULT_MODEL_REF,
325	      applyDefaultConfig: applyKilocodeConfig,
326	      applyProviderConfig: applyKilocodeProviderConfig,
327	      noteDefault: KILOCODE_DEFAULT_MODEL_REF,
328	      noteAgentModel,
329	      prompter: params.prompter,
330	    });
331	    nextConfig = applied.config;
332	    agentModelOverride = applied.agentModelOverride ?? agentModelOverride;
333	  }
334	  return { config: nextConfig, agentModelOverride };
335	}
336	```
337	
338	Also add tokenProvider mapping at the top of the function:
339	
340	```typescript
341	if (params.opts.tokenProvider === "kilocode") {
342	  authChoice = "kilocode-api-key";
343	}
344	```
345	
346	### CLI Registration
347	
348	#### 9. `src/cli/program/register.onboard.ts`
349	
350	Add CLI option:
351	
352	```typescript
353	.option("--kilocode-api-key <key>", "Kilo Gateway API key")
354	```
355	
356	Add to action handler:
357	
358	```typescript
359	kilocodeApiKey: opts.kilocodeApiKey as string | undefined,
360	```
361	
362	Update auth-choice help text:
363	
364	```typescript
365	.option(
366	  "--auth-choice <choice>",
367	  "Auth: setup-token|token|chutes|openai-codex|openai-api-key|openrouter-api-key|kilocode-api-key|ai-gateway-api-key|...",
368	)
369	```
370	
371	### Non-Interactive Onboarding
372	
373	#### 10. `src/commands/onboard-non-interactive/local/auth-choice.ts`
374	
375	Add handling for `kilocode-api-key`:
376	
377	```typescript
378	if (authChoice === "kilocode-api-key") {
379	  const resolved = await resolveNonInteractiveApiKey({
380	    provider: "kilocode",
381	    cfg: baseConfig,
382	    flagValue: opts.kilocodeApiKey,
383	    flagName: "--kilocode-api-key",
384	    envVar: "KILOCODE_API_KEY",
385	  });
386	  await setKilocodeApiKey(resolved.apiKey, agentDir);
387	  nextConfig = applyAuthProfileConfig(nextConfig, {
388	    profileId: "kilocode:default",
389	    provider: "kilocode",
390	    mode: "api_key",
391	  });
392	  // ... apply default model
393	}
394	```
395	
396	### Export Updates
397	
398	#### 11. `src/commands/onboard-auth.ts`
399	
400	Add exports:
401	
402	```typescript
403	export {
404	  // ... existing exports
405	  applyKilocodeConfig,
406	  applyKilocodeProviderConfig,
407	  KILOCODE_BASE_URL,
408	} from "./onboard-auth.config-core.js";
409	
410	export {
411	  // ... existing exports
412	  KILOCODE_DEFAULT_MODEL_REF,
413	  setKilocodeApiKey,
414	} from "./onboard-auth.credentials.js";
415	```
416	
417	### Special Handling (Optional)
418	
419	#### 12. `src/agents/pi-embedded-runner/cache-ttl.ts`
420	
421	Add Kilo Gateway support for Anthropic models:
422	
423	```typescript
424	export function isCacheTtlEligibleProvider(provider: string, modelId: string): boolean {
425	  const normalizedProvider = provider.toLowerCase();
426	  const normalizedModelId = modelId.toLowerCase();
427	  if (normalizedProvider === "anthropic") return true;
428	  if (normalizedProvider === "openrouter" && normalizedModelId.startsWith("anthropic/"))
429	    return true;
430	  if (normalizedProvider === "kilocode" && normalizedModelId.startsWith("anthropic/")) return true;
431	  return false;
432	}
433	```
434	
435	#### 13. `src/agents/transcript-policy.ts`
436	
437	Add Kilo Gateway handling (similar to OpenRouter):
438	
439	```typescript
440	const isKilocodeGemini = provider === "kilocode" && modelId.toLowerCase().includes("gemini");
441	
442	// Include in needsNonImageSanitize check
443	const needsNonImageSanitize =
444	  isGoogle || isAnthropic || isMistral || isOpenRouterGemini || isKilocodeGemini;
445	```
446	
447	## Configuration Structure
448	
449	### User Config Example
450	
451	```json
452	{
453	  "models": {
454	    "mode": "merge",
455	    "providers": {
456	      "kilocode": {
457	        "baseUrl": "https://api.kilo.ai/api/gateway/",
458	        "apiKey": "xxxxx",
459	        "api": "openai-completions",
460	        "models": [
461	          {
462	            "id": "anthropic/claude-opus-4.6",
463	            "name": "Anthropic: Claude Opus 4.6"
464	          },
465	          { "id": "minimax/minimax-m2.5:free", "name": "Minimax: Minimax M2.5" }
466	        ]
467	      }
468	    }
469	  }
470	}
471	```
472	
473	### Auth Profile Structure
474	
475	```json
476	{
477	  "profiles": {
478	    "kilocode:default": {
479	      "type": "api_key",
480	      "provider": "kilocode",
481	      "key": "xxxxx"
482	    }
483	  }
484	}
485	```
486	
487	## Testing Considerations
488	
489	1. **Unit Tests:**
490	   - Test `setKilocodeApiKey()` writes correct profile
491	   - Test `applyKilocodeConfig()` sets correct defaults
492	   - Test `resolveEnvApiKey("kilocode")` returns correct env var
493	
494	2. **Integration Tests:**
495	   - Test onboarding flow with `--auth-choice kilocode-api-key`
496	   - Test non-interactive onboarding with `--kilocode-api-key`
497	   - Test model selection with `kilocode/` prefix
498	
499	3. **E2E Tests:**
500	   - Test actual API calls through Kilo Gateway (live tests)
501	
502	## Migration Notes
503	
504	- No migration needed for existing users
505	- New users can immediately use `kilocode-api-key` auth choice
506	- Existing manual config with `kilocode` provider will continue to work
507	
508	## Future Considerations
509	
510	1. **Model Catalog:** If Kilo Gateway exposes a `/models` endpoint, add scanning support similar to `scanOpenRouterModels()`
511	
512	2. **OAuth Support:** If Kilo Gateway adds OAuth, extend the auth system accordingly
513	
514	3. **Rate Limiting:** Consider adding rate limit handling specific to Kilo Gateway if needed
515	
516	4. **Documentation:** Add docs at `docs/providers/kilocode.md` explaining setup and usage
517	
518	## Summary of Changes
519	
520	| File                                                        | Change Type | Description                                                             |
521	| ----------------------------------------------------------- | ----------- | ----------------------------------------------------------------------- |
522	| `src/commands/onboard-auth.credentials.ts`                  | Add         | `KILOCODE_DEFAULT_MODEL_REF`, `setKilocodeApiKey()`                     |
523	| `src/agents/model-auth.ts`                                  | Modify      | Add `kilocode` to `envMap`                                              |
524	| `src/config/io.ts`                                          | Modify      | Add `KILOCODE_API_KEY` to shell env keys                                |
525	| `src/commands/onboard-auth.config-core.ts`                  | Add         | `applyKilocodeProviderConfig()`, `applyKilocodeConfig()`                |
526	| `src/commands/onboard-types.ts`                             | Modify      | Add `kilocode-api-key` to `AuthChoice`, add `kilocodeApiKey` to options |
527	| `src/commands/auth-choice-options.ts`                       | Modify      | Add `kilocode` group and option                                         |
528	| `src/commands/auth-choice.preferred-provider.ts`            | Modify      | Add `kilocode-api-key` mapping                                          |
529	| `src/commands/auth-choice.apply.api-providers.ts`           | Modify      | Add `kilocode-api-key` handling                                         |
530	| `src/cli/program/register.onboard.ts`                       | Modify      | Add `--kilocode-api-key` option                                         |
531	| `src/commands/onboard-non-interactive/local/auth-choice.ts` | Modify      | Add non-interactive handling                                            |
532	| `src/commands/onboard-auth.ts`                              | Modify      | Export new functions                                                    |
533	| `src/agents/pi-embedded-runner/cache-ttl.ts`                | Modify      | Add kilocode support                                                    |
534	| `src/agents/transcript-policy.ts`                           | Modify      | Add kilocode Gemini handling                                            |
535	
```

## Assistant

<thinking>
This document is about integrating a provider (Kilo Gateway) as a model provider, not about the gateway protocol itself. Let me look at the gateway protocol more specifically.

Now I have a comprehensive understanding. Let me summarize what I've found about LobsterAI's OpenClaw control mechanism:

1. **Gateway is a WebSocket Server**: OpenClaw runs a WebSocket gateway server (port 18789 by default) via `utilityProcess.fork()` or `child_process.spawn()`.

2. **Gateway Client is an npm module**: LobsterAI dynamically imports `GatewayClient` from the OpenClaw runtime's dist directory (`gateway/client.js`). This is a pre-built WebSocket client bundled with OpenClaw.
   
3. **WebSocket Communication**: The client connects to the gateway via `ws://127.0.0.1:{port}` using WebSocket, authenticating with a bearer token.

4. **Request/Response Protocol**: The GatewayClient exposes methods like `chat.send`, `chat.history`, `chat.abort`, `exec.approval.resolve`, and `sessions.list` that handle bidirectional communication for starting conversations, retrieving history, aborting sessions, managing approvals, and listing available sessions.

5. **Event Streaming**: The gateway pushes real-time events including chat updates, agent responses, approval requests and resolutions, and tick signals back to the client through the WebSocket connection.

6. **Handshake Authentication**: When connecting, the client provides identification including the URL, auth token, display name, version, mode, and capability flags, with the server responding once the handshake completes.

Now I can address the user's question about whether there's a universal protocol and whether this approach would work for integrating with Hermes Agent—the gateway client is OpenClaw-specific, so LobsterAI would need either Hermes Agent to provide a compatible gateway client or to implement the protocol directly if Hermes Agent exposes it.
</thinking>

现在我对整体架构有了清晰的理解，来回答你的问题。

---

## LobsterAI 如何操控 OpenClaw

LobsterAI 并不"操控" OpenClaw —— 它是 **OpenClaw Gateway 的一个客户端**，两者是 **对等协作** 的关系。

### 核心架构

```
┌── LobsterAI 主进程 ──────────────────────────────────────────┐
│                                                             │
│  CoworkEngineRouter  (路由层, 引擎无关)                      │
│       ├── openclawRuntimeAdapter.ts   (OpenClaw 专有适配器)  │
│       └── claudeRuntimeAdapter.ts     (内置引擎适配器)        │
│                                                             │
│  OpenClawRuntimeAdapter 持有:                                │
│    GatewayClient (动态加载 OpenClaw dist/gateway/client.js)   │
│    ↓ WebSocket 连接                                          │
└─────────────────────────────────────────────────────────────┘
                              ↕ ws://127.0.0.1:18789
┌── OpenClaw Gateway 进程 (utilityProcess.fork) ──────────────┐
│  - OpenClaw Agent Engine (内置)                              │
│  - Session 管理                                             │
│  - Tool Executor                                            │
│  - Config 读写                                              │
│  - IM Channel Session Polling                               │
└─────────────────────────────────────────────────────────────┘
```

### 关键组件

**1. 进程生命周期管理** (`openclawEngineManager.ts`)
- 用 `utilityProcess.fork()`（macOS/Linux）或 `child_process.spawn()`（Windows）启动 OpenClaw Gateway 子进程
- 启动参数: `gateway --bind loopback --port {port} --token {token} --verbose`
- 监听 `loopback:18789`（默认），做健康检查、自动重启（最多5次，指数退避）
- 注入环境变量: `OPENCLAW_HOME`, `OPENCLAW_GATEWAY_TOKEN`, `OPENCLAW_STATE_DIR`, `OPENCLAW_CONFIG_PATH` 等

**2. Gateway Client（WebSocket 客户端）** (`openclawRuntimeAdapter.ts`)
- 动态 `require()` 加载 OpenClaw runtime 中的 `dist/gateway/client.js`，得到 `GatewayClient` 构造函数
- 实例化时传入连接参数:
  ```typescript
  new GatewayClient({
    url: 'ws://127.0.0.1:18789',
    token: '<bearer-token>',
    clientDisplayName: 'LobsterAI',
    mode: 'backend',
    caps: ['tool-events'],
    role: 'operator',
    scopes: ['operator.admin'],
    onHelloOk: () => { /* 握手成功 */ },
    onEvent: (frame) => { /* 处理推送事件 */ },
  })
  ```
- GatewayClient 提供三个核心方法:
  - `client.start()` / `client.stop()` — 连接生命周期
  - `client.request(method, params)` — RPC 请求（Promise）
  - `onEvent(frame)` — 接收服务端推送事件

**3. RPC 方法（协议的核心）**

LobsterAI → Gateway 的 RPC 调用:

| RPC 方法 | 用途 |
|---------|------|
| `chat.send` | 发送用户消息，启动一轮 Agent 执行 |
| `chat.history` | 获取会话历史 |
| `chat.abort` | 中止当前运行 |
| `sessions.list` | 列出所有会话（含 IM channel sessions） |
| `exec.approval.resolve` | 回复工具执行权限请求 |

Gateway → LobsterAI 的推送事件 (`onEvent`):

| 事件类型 | 含义 |
|---------|------|
| `tick` | Gateway 心跳（每30s一次），用于存活检测 |
| `chat` | 聊天状态变化 (`delta`/`final`/`aborted`/`error`) |
| `agent` | Agent 流式输出（含 `assistant` 文本、tool calls） |
| `exec.approval.requested` | 工具执行需要用户授权 |
| `exec.approval.resolved` | 授权已被处理 |

---

## 有通用协议吗？

**没有通用公开协议**。这是 OpenClaw 的私有 Gateway Protocol：

- WebSocket + JSON-RPC 风格的 request/response
- 认证: Bearer Token（启动时生成，写入 `state/gateway-token` 文件）
- Gateway Client SDK 不开源，仅以编译后 JS 模块形式存在于 OpenClaw runtime 中
- 连接 URL、握手参数（`mode`, `caps`, `role`, `scopes`）都是 OpenClaw 自定义的

这不是 OpenAI、Anthropic 那样的公开 HTTP API，也不是 MCP (Model Context Protocol) 那样的标准化协议。

---

## 对接 Hermes Agent 适用吗？

**不适用，直接复用这套方案行不通**。原因:

1. **没有通用协议**: OpenClaw Gateway Client 是 OpenClaw 专用的，需要从 `dist/gateway/client.js` 动态加载。你没有 Hermes Agent 的等效模块。

2. **需要 Hermes Agent 主动暴露 Gateway 接口**: OpenClaw Gateway 是一个 **子进程**（Node.js/Electron），它主动 listen WebSocket port，等待客户端连接。Hermes Agent 也需要以类似方式运行一个 Gateway 进程。

3. **如果 Hermes Agent 支持标准协议，可行**: 如果 Hermes Agent 提供 MCP Server、OpenAI-Compatible API、或者类似的 Gateway WebSocket 接口，你只需要写一个新的 `HermesRuntimeAdapter`（类似 `openclawRuntimeAdapter.ts`），替换掉 `GatewayClient` 的加载和 RPC 调用即可。整个 `CoworkEngineRouter` 层是引擎无关的。

**总结**: 架构上 LobsterAI 已经解耦得很好了 —— `CoworkEngineRouter` 只要求适配器实现 `CoworkRuntime` 接口（`startSession/continueSession/stopSession/respondToPermission`），所以接入 Hermes Agent 的路径是：**在 Hermes Agent 端实现一个 Gateway 兼容接口**，然后在 LobsterAI 端写一个新的 Runtime Adapter。

## User

这个gateway是openclaw已经提供的吗？如果实时获取流式消息的？

## Assistant

<thinking>
The user is asking two follow-up questions:
1. Is the gateway provided by OpenClaw already?
2. How does it get real-time streaming messages?

Let me answer both based on what I've already read from the code.
</thinking>

是的，**Gateway 是 OpenClaw 自带的**，LobsterAI 只是它的客户端。

### Gateway 从哪来

```
LobsterAI 项目/
├── vendor/openclaw-runtime/mac-arm64/
│   └── dist/gateway/
│       └── client.js          ← LobsterAI 动态加载这个，作为 WS 客户端
│
└── OpenClaw runtime 内部/
    └── dist/gateway/
        └── server.js / server.mjs   ← OpenClaw 启动时运行，监听 loopback:18789
```

`openclawEngineManager.ts` 里有一行关键代码（`resolveGatewayClientEntry`）:

```typescript
// 加载路径优先级
const distRoots = [
  path.join(runtimeRoot, 'dist'),
  path.join(runtimeRoot, 'gateway.asar', 'dist'),
];

// 找 client.js
const gatewayClient = path.join(distRoot, 'gateway', 'client.js');
```

OpenClaw 启动时执行 `openclaw-entry gateway --bind loopback --port 18789`，它自己把 `dist/gateway/server` 跑起来。LobsterAI 不管 Gateway 的实现，只加载对应的 `client.js` 来连接它。

---

### 流式消息是怎么实时获取的

```
LobsterAI                        OpenClaw Gateway (WS Server)
    │                                     │
    │  client.start()                     │
    │  (建立 WebSocket 连接)                │  ← utilityProcess.fork() 起的子进程
    │─────────────────────────────────────→
    │                                     │
    │                               Agent 开始处理用户请求
    │                                     │
    │  ← onEvent({ event: 'agent', ... })  │  agent.text.delta 事件
    │  ← onEvent({ event: 'agent', ... })  │  agent.toolCall 事件
    │  ← onEvent({ event: 'chat', ... })   │  chat.state=delta/final
    │  ← onEvent({ event: 'complete', ...})│
```

具体到代码（`openclawRuntimeAdapter.ts`），Gateway Client 在 `onEvent` 回调里用**事件帧** (`GatewayEventFrame`) 接收所有推送:

```typescript
client.onEvent = (event: GatewayEventFrame) => {
  // event 的结构:
  // { event: string; seq?: number; payload?: unknown }

  if (event.event === 'tick') {
    this.lastTickTimestamp = Date.now();  // 心跳保活
    return;
  }

  if (event.event === 'chat') {
    this.handleChatEvent(event.payload, event.seq);   // 聊天状态
    return;
  }

  if (event.event === 'agent') {
    this.processAgentAssistantText(event.payload);     // 流式文本 delta
    this.handleAgentEvent(event.payload, event.seq);   // tool calls 等
    return;
  }

  if (event.event === 'exec.approval.requested') {
    this.handleApprovalRequested(event.payload);       // 权限弹窗
    return;
  }
};
```

**流式文本的处理**（`processAgentAssistantText`）:

```typescript
// agent 事件里的 stream 字段标识事件类型
const stream = typeof agentPayload.stream === 'string' ? agentPayload.stream : '';

if (stream === 'agent.assistant.text.delta') {
  // 这是一个文本增量（delta），拼接到当前文本
  const delta = extractOpenClawAssistantStreamText(payload);
  turn.currentText += delta;
  // 节流 200ms 后通过 IPC 通知渲染进程更新 UI
  this.scheduleMessageUpdateEmit(sessionId, turn);
}

if (stream === 'agent.assistant.text.final') {
  // 文本块写完了，提交到消息历史
  this.commitAssistantText(sessionId, turn);
}
```

**`exec.approval.requested` 触发权限弹窗**:

```typescript
// 当 Agent 要执行 shell 命令等高危操作时，Gateway 推送这个事件
// LobsterAI 弹出 CoworkPermissionModal，用户点"允许"/"拒绝"
// → respondToPermission() → client.request('exec.approval.resolve', { id, decision })
```

---

### 总结一下实时性

Gateway 和 LobsterAI 之间是 **持久 WebSocket 连接**，Gateway 主动 push，LobsterAI 只负责接收和处理。流式输出靠 `agent.assistant.text.delta` 事件逐段到达，LobsterAI 在客户端做增量拼接（`computeSuffixPrefixOverlap` 处理 delta 重叠问题），再通过 IPC `messageUpdate` 事件推给 React UI 渲染。整个链路:

```
Gateway push (delta)
  → openclawRuntimeAdapter.onEvent()
  → this.scheduleMessageUpdateEmit() [throttled 200ms]
  → IPC: cowork:messageUpdate
  → React onStreamMessageUpdate()
  → Redux store 更新
  → UI 组件 rerender
```

