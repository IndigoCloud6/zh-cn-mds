\# Agent SDK reference - TypeScript



Complete API reference for the TypeScript Agent SDK, including all functions, types, and interfaces.



\---



<script src="/components/typescript-sdk-type-links.js" defer />



<Note>

\*\*Try the new V2 interface (preview):\*\* A simplified interface with `send()` and `stream()` patterns is now available, making multi-turn conversations easier. \[Learn more about the TypeScript V2 preview](/docs/en/agent-sdk/typescript-v2-preview)

</Note>



\## Installation



```bash

npm install @anthropic-ai/claude-agent-sdk

```



\## Functions



\### `query()`



The primary function for interacting with Claude Code. Creates an async generator that streams messages as they arrive.



```typescript

function query({

&#x20; prompt,

&#x20; options

}: {

&#x20; prompt: string | AsyncIterable<SDKUserMessage>;

&#x20; options?: Options;

}): Query;

```



\#### Parameters



| Parameter | Type | Description |

| :-------- | :--- | :---------- |

| `prompt` | `string \\| AsyncIterable<`\[`SDKUserMessage`](#sdkuser-message)`>` | The input prompt as a string or async iterable for streaming mode |

| `options` | \[`Options`](#options) | Optional configuration object (see Options type below) |



\#### Returns



Returns a \[`Query`](#query-object) object that extends `AsyncGenerator<`\[`SDKMessage`](#sdk-message)`, void>` with additional methods.



\### `tool()`



Creates a type-safe MCP tool definition for use with SDK MCP servers.



```typescript

function tool<Schema extends AnyZodRawShape>(

&#x20; name: string,

&#x20; description: string,

&#x20; inputSchema: Schema,

&#x20; handler: (args: InferShape<Schema>, extra: unknown) => Promise<CallToolResult>,

&#x20; extras?: { annotations?: ToolAnnotations }

): SdkMcpToolDefinition<Schema>;

```



\#### Parameters



| Parameter | Type | Description |

| :-------- | :--- | :---------- |

| `name` | `string` | The name of the tool |

| `description` | `string` | A description of what the tool does |

| `inputSchema` | `Schema extends AnyZodRawShape` | Zod schema defining the tool's input parameters (supports both Zod 3 and Zod 4) |

| `handler` | `(args, extra) => Promise<`\[`CallToolResult`](#call-tool-result)`>` | Async function that executes the tool logic |

| `extras` | `{ annotations?: `\[`ToolAnnotations`](#tool-annotations)` }` | Optional MCP tool annotations providing behavioral hints to clients |



\#### `ToolAnnotations`



Re-exported from `@modelcontextprotocol/sdk/types.js`. All fields are optional hints; clients should not rely on them for security decisions.



| Field | Type | Default | Description |

| :---- | :--- | :------ | :---------- |

| `title` | `string` | `undefined` | Human-readable title for the tool |

| `readOnlyHint` | `boolean` | `false` | If `true`, the tool does not modify its environment |

| `destructiveHint` | `boolean` | `true` | If `true`, the tool may perform destructive updates (only meaningful when `readOnlyHint` is `false`) |

| `idempotentHint` | `boolean` | `false` | If `true`, repeated calls with the same arguments have no additional effect (only meaningful when `readOnlyHint` is `false`) |

| `openWorldHint` | `boolean` | `true` | If `true`, the tool interacts with external entities (for example, web search). If `false`, the tool's domain is closed (for example, a memory tool) |



```typescript

import { tool } from "@anthropic-ai/claude-agent-sdk";

import { z } from "zod";



const searchTool = tool(

&#x20; "search",

&#x20; "Search the web",

&#x20; { query: z.string() },

&#x20; async ({ query }) => {

&#x20;   return { content: \[{ type: "text", text: `Results for: ${query}` }] };

&#x20; },

&#x20; { annotations: { readOnlyHint: true, openWorldHint: true } }

);

```



\### `createSdkMcpServer()`



Creates an MCP server instance that runs in the same process as your application.



```typescript

function createSdkMcpServer(options: {

&#x20; name: string;

&#x20; version?: string;

&#x20; tools?: Array<SdkMcpToolDefinition<any>>;

}): McpSdkServerConfigWithInstance;

```



\#### Parameters



| Parameter | Type | Description |

| :-------- | :--- | :---------- |

| `options.name` | `string` | The name of the MCP server |

| `options.version` | `string` | Optional version string |

| `options.tools` | `Array<SdkMcpToolDefinition>` | Array of tool definitions created with \[`tool()`](#tool) |



\### `listSessions()`



Discovers and lists past sessions with light metadata. Filter by project directory or list sessions across all projects.



```typescript

function listSessions(options?: ListSessionsOptions): Promise<SDKSessionInfo\[]>;

```



\#### Parameters



| Parameter | Type | Default | Description |

| :-------- | :--- | :------ | :---------- |

| `options.dir` | `string` | `undefined` | Directory to list sessions for. When omitted, returns sessions across all projects |

| `options.limit` | `number` | `undefined` | Maximum number of sessions to return |

| `options.includeWorktrees` | `boolean` | `true` | When `dir` is inside a git repository, include sessions from all worktree paths |



\#### Return type: `SDKSessionInfo`



| Property | Type | Description |

| :------- | :--- | :---------- |

| `sessionId` | `string` | Unique session identifier (UUID) |

| `summary` | `string` | Display title: custom title, auto-generated summary, or first prompt |

| `lastModified` | `number` | Last modified time in milliseconds since epoch |

| `fileSize` | `number \\| undefined` | Session file size in bytes. Only populated for local JSONL storage |

| `customTitle` | `string \\| undefined` | User-set session title (via `/rename`) |

| `firstPrompt` | `string \\| undefined` | First meaningful user prompt in the session |

| `gitBranch` | `string \\| undefined` | Git branch at the end of the session |

| `cwd` | `string \\| undefined` | Working directory for the session |

| `tag` | `string \\| undefined` | User-set session tag (see \[`tagSession()`](#tag-session)) |

| `createdAt` | `number \\| undefined` | Creation time in milliseconds since epoch, from the first entry's timestamp |



\#### Example



Print the 10 most recent sessions for a project. Results are sorted by `lastModified` descending, so the first item is the newest. Omit `dir` to search across all projects.



```typescript

import { listSessions } from "@anthropic-ai/claude-agent-sdk";



const sessions = await listSessions({ dir: "/path/to/project", limit: 10 });



for (const session of sessions) {

&#x20; console.log(`${session.summary} (${session.sessionId})`);

}

```



\### `getSessionMessages()`



Reads user and assistant messages from a past session transcript.



```typescript

function getSessionMessages(

&#x20; sessionId: string,

&#x20; options?: GetSessionMessagesOptions

): Promise<SessionMessage\[]>;

```



\#### Parameters



| Parameter | Type | Default | Description |

| :-------- | :--- | :------ | :---------- |

| `sessionId` | `string` | required | Session UUID to read (see `listSessions()`) |

| `options.dir` | `string` | `undefined` | Project directory to find the session in. When omitted, searches all projects |

| `options.limit` | `number` | `undefined` | Maximum number of messages to return |

| `options.offset` | `number` | `undefined` | Number of messages to skip from the start |



\#### Return type: `SessionMessage`



| Property | Type | Description |

| :------- | :--- | :---------- |

| `type` | `"user" \\| "assistant"` | Message role |

| `uuid` | `string` | Unique message identifier |

| `session\_id` | `string` | Session this message belongs to |

| `message` | `unknown` | Raw message payload from the transcript |

| `parent\_tool\_use\_id` | `null` | Reserved |



\#### Example



```typescript

import { listSessions, getSessionMessages } from "@anthropic-ai/claude-agent-sdk";



const \[latest] = await listSessions({ dir: "/path/to/project", limit: 1 });



if (latest) {

&#x20; const messages = await getSessionMessages(latest.sessionId, {

&#x20;   dir: "/path/to/project",

&#x20;   limit: 20

&#x20; });



&#x20; for (const msg of messages) {

&#x20;   console.log(`\[${msg.type}] ${msg.uuid}`);

&#x20; }

}

```



\### `getSessionInfo()`



Reads metadata for a single session by ID without scanning the full project directory.



```typescript

function getSessionInfo(

&#x20; sessionId: string,

&#x20; options?: GetSessionInfoOptions

): Promise<SDKSessionInfo | undefined>;

```



\#### Parameters



| Parameter | Type | Default | Description |

| :-------- | :--- | :------ | :---------- |

| `sessionId` | `string` | required | UUID of the session to look up |

| `options.dir` | `string` | `undefined` | Project directory path. When omitted, searches all project directories |



Returns \[`SDKSessionInfo`](#return-type-sdk-session-info), or `undefined` if the session is not found.



\### `renameSession()`



Renames a session by appending a custom-title entry. Repeated calls are safe; the most recent title wins.



```typescript

function renameSession(

&#x20; sessionId: string,

&#x20; title: string,

&#x20; options?: SessionMutationOptions

): Promise<void>;

```



\#### Parameters



| Parameter | Type | Default | Description |

| :-------- | :--- | :------ | :---------- |

| `sessionId` | `string` | required | UUID of the session to rename |

| `title` | `string` | required | New title. Must be non-empty after trimming whitespace |

| `options.dir` | `string` | `undefined` | Project directory path. When omitted, searches all project directories |



\### `tagSession()`



Tags a session. Pass `null` to clear the tag. Repeated calls are safe; the most recent tag wins.



```typescript

function tagSession(

&#x20; sessionId: string,

&#x20; tag: string | null,

&#x20; options?: SessionMutationOptions

): Promise<void>;

```



\#### Parameters



| Parameter | Type | Default | Description |

| :-------- | :--- | :------ | :---------- |

| `sessionId` | `string` | required | UUID of the session to tag |

| `tag` | `string \\| null` | required | Tag string, or `null` to clear |

| `options.dir` | `string` | `undefined` | Project directory path. When omitted, searches all project directories |



\## Types



\### `Options`



Configuration object for the `query()` function.



| Property | Type | Default | Description |

| :------- | :--- | :------ | :---------- |

| `abortController` | `AbortController` | `new AbortController()` | Controller for cancelling operations |

| `additionalDirectories` | `string\[]` | `\[]` | Additional directories Claude can access |

| `agent` | `string` | `undefined` | Agent name for the main thread. The agent must be defined in the `agents` option or in settings |

| `agents` | `Record<string, \[`AgentDefinition`](#agent-definition)>` | `undefined` | Programmatically define subagents |

| `allowDangerouslySkipPermissions` | `boolean` | `false` | Enable bypassing permissions. Required when using `permissionMode: 'bypassPermissions'` |

| `allowedTools` | `string\[]` | `\[]` | Tools to auto-approve without prompting. This does not restrict Claude to only these tools; unlisted tools fall through to `permissionMode` and `canUseTool`. Use `disallowedTools` to block tools. See \[Permissions](/docs/en/agent-sdk/permissions#allow-and-deny-rules) |

| `betas` | \[`SdkBeta`](#sdk-beta)`\[]` | `\[]` | Enable beta features (e.g., `\['context-1m-2025-08-07']`) |

| `canUseTool` | \[`CanUseTool`](#can-use-tool) | `undefined` | Custom permission function for tool usage |

| `continue` | `boolean` | `false` | Continue the most recent conversation |

| `cwd` | `string` | `process.cwd()` | Current working directory |

| `debug` | `boolean` | `false` | Enable debug mode for the Claude Code process |

| `debugFile` | `string` | `undefined` | Write debug logs to a specific file path. Implicitly enables debug mode |

| `disallowedTools` | `string\[]` | `\[]` | Tools to always deny. Deny rules are checked first and override `allowedTools` and `permissionMode` (including `bypassPermissions`) |

| `effort` | `'low' \\| 'medium' \\| 'high' \\| 'max'` | `'high'` | Controls how much effort Claude puts into its response. Works with adaptive thinking to guide thinking depth |

| `enableFileCheckpointing` | `boolean` | `false` | Enable file change tracking for rewinding. See \[File checkpointing](/docs/en/agent-sdk/file-checkpointing) |

| `env` | `Record<string, string \\| undefined>` | `process.env` | Environment variables. Set `CLAUDE\_AGENT\_SDK\_CLIENT\_APP` to identify your app in the User-Agent header |

| `executable` | `'bun' \\| 'deno' \\| 'node'` | Auto-detected | JavaScript runtime to use |

| `executableArgs` | `string\[]` | `\[]` | Arguments to pass to the executable |

| `extraArgs` | `Record<string, string \\| null>` | `{}` | Additional arguments |

| `fallbackModel` | `string` | `undefined` | Model to use if primary fails |

| `forkSession` | `boolean` | `false` | When resuming with `resume`, fork to a new session ID instead of continuing the original session |

| `hooks` | `Partial<Record<`\[`HookEvent`](#hook-event)`, `\[`HookCallbackMatcher`](#hook-callback-matcher)`\[]>>` | `{}` | Hook callbacks for events |

| `includePartialMessages` | `boolean` | `false` | Include partial message events |

| `maxBudgetUsd` | `number` | `undefined` | Maximum budget in USD for the query |

| `maxThinkingTokens` | `number` | `undefined` | \_Deprecated:\_ Use `thinking` instead. Maximum tokens for thinking process |

| `maxTurns` | `number` | `undefined` | Maximum agentic turns (tool-use round trips) |

| `mcpServers` | `Record<string, \[`McpServerConfig`](#mcp-server-config)>` | `{}` | MCP server configurations |

| `model` | `string` | Default from CLI | Claude model to use |

| `outputFormat` | `{ type: 'json\_schema', schema: JSONSchema }` | `undefined` | Define output format for agent results. See \[Structured outputs](/docs/en/agent-sdk/structured-outputs) for details |

| `pathToClaudeCodeExecutable` | `string` | Uses built-in executable | Path to Claude Code executable |

| `permissionMode` | \[`PermissionMode`](#permission-mode) | `'default'` | Permission mode for the session |

| `permissionPromptToolName` | `string` | `undefined` | MCP tool name for permission prompts |

| `persistSession` | `boolean` | `true` | When `false`, disables session persistence to disk. Sessions cannot be resumed later |

| `plugins` | \[`SdkPluginConfig`](#sdk-plugin-config)`\[]` | `\[]` | Load custom plugins from local paths. See \[Plugins](/docs/en/agent-sdk/plugins) for details |

| `promptSuggestions` | `boolean` | `false` | Enable prompt suggestions. Emits a `prompt\_suggestion` message after each turn with a predicted next user prompt |

| `resume` | `string` | `undefined` | Session ID to resume |

| `resumeSessionAt` | `string` | `undefined` | Resume session at a specific message UUID |

| `sandbox` | \[`SandboxSettings`](#sandbox-settings) | `undefined` | Configure sandbox behavior programmatically. See \[Sandbox settings](#sandbox-settings) for details |

| `sessionId` | `string` | Auto-generated | Use a specific UUID for the session instead of auto-generating one |

| `settingSources` | \[`SettingSource`](#setting-source)`\[]` | `\[]` (no settings) | Control which filesystem settings to load. When omitted, no settings are loaded. \*\*Note:\*\* Must include `'project'` to load CLAUDE.md files |

| `spawnClaudeCodeProcess` | `(options: SpawnOptions) => SpawnedProcess` | `undefined` | Custom function to spawn the Claude Code process. Use to run Claude Code in VMs, containers, or remote environments |

| `stderr` | `(data: string) => void` | `undefined` | Callback for stderr output |

| `strictMcpConfig` | `boolean` | `false` | Enforce strict MCP validation |

| `systemPrompt` | `string \\| { type: 'preset'; preset: 'claude\_code'; append?: string }` | `undefined` (minimal prompt) | System prompt configuration. Pass a string for custom prompt, or `{ type: 'preset', preset: 'claude\_code' }` to use Claude Code's system prompt. When using the preset object form, add `append` to extend the system prompt with additional instructions |

| `thinking` | \[`ThinkingConfig`](#thinking-config) | `{ type: 'adaptive' }` for supported models | Controls Claude's thinking/reasoning behavior. See \[`ThinkingConfig`](#thinking-config) for options |

| `toolConfig` | \[`ToolConfig`](#tool-config) | `undefined` | Configuration for built-in tool behavior. See \[`ToolConfig`](#tool-config) for details |

| `tools` | `string\[] \\| { type: 'preset'; preset: 'claude\_code' }` | `undefined` | Tool configuration. Pass an array of tool names or use the preset to get Claude Code's default tools |



\### `Query` object



Interface returned by the `query()` function.



```typescript

interface Query extends AsyncGenerator<SDKMessage, void> {

&#x20; interrupt(): Promise<void>;

&#x20; rewindFiles(

&#x20;   userMessageId: string,

&#x20;   options?: { dryRun?: boolean }

&#x20; ): Promise<RewindFilesResult>;

&#x20; setPermissionMode(mode: PermissionMode): Promise<void>;

&#x20; setModel(model?: string): Promise<void>;

&#x20; setMaxThinkingTokens(maxThinkingTokens: number | null): Promise<void>;

&#x20; initializationResult(): Promise<SDKControlInitializeResponse>;

&#x20; supportedCommands(): Promise<SlashCommand\[]>;

&#x20; supportedModels(): Promise<ModelInfo\[]>;

&#x20; supportedAgents(): Promise<AgentInfo\[]>;

&#x20; mcpServerStatus(): Promise<McpServerStatus\[]>;

&#x20; accountInfo(): Promise<AccountInfo>;

&#x20; reconnectMcpServer(serverName: string): Promise<void>;

&#x20; toggleMcpServer(serverName: string, enabled: boolean): Promise<void>;

&#x20; setMcpServers(servers: Record<string, McpServerConfig>): Promise<McpSetServersResult>;

&#x20; streamInput(stream: AsyncIterable<SDKUserMessage>): Promise<void>;

&#x20; stopTask(taskId: string): Promise<void>;

&#x20; close(): void;

}

```



\#### Methods



| Method | Description |

| :----- | :---------- |

| `interrupt()` | Interrupts the query (only available in streaming input mode) |

| `rewindFiles(userMessageId, options?)` | Restores files to their state at the specified user message. Pass `{ dryRun: true }` to preview changes. Requires `enableFileCheckpointing: true`. See \[File checkpointing](/docs/en/agent-sdk/file-checkpointing) |

| `setPermissionMode()` | Changes the permission mode (only available in streaming input mode) |

| `setModel()` | Changes the model (only available in streaming input mode) |

| `setMaxThinkingTokens()` | \_Deprecated:\_ Use the `thinking` option instead. Changes the maximum thinking tokens |

| `initializationResult()` | Returns the full initialization result including supported commands, models, account info, and output style configuration |

| `supportedCommands()` | Returns available slash commands |

| `supportedModels()` | Returns available models with display info |

| `supportedAgents()` | Returns available subagents as \[`AgentInfo`](#agent-info)`\[]` |

| `mcpServerStatus()` | Returns status of connected MCP servers |

| `accountInfo()` | Returns account information |

| `reconnectMcpServer(serverName)` | Reconnect an MCP server by name |

| `toggleMcpServer(serverName, enabled)` | Enable or disable an MCP server by name |

| `setMcpServers(servers)` | Dynamically replace the set of MCP servers for this session. Returns info about which servers were added, removed, and any errors |

| `streamInput(stream)` | Stream input messages to the query for multi-turn conversations |

| `stopTask(taskId)` | Stop a running background task by ID |

| `close()` | Close the query and terminate the underlying process. Forcefully ends the query and cleans up all resources |



\### `SDKControlInitializeResponse`



Return type of `initializationResult()`. Contains session initialization data.



```typescript

type SDKControlInitializeResponse = {

&#x20; commands: SlashCommand\[];

&#x20; agents: AgentInfo\[];

&#x20; output\_style: string;

&#x20; available\_output\_styles: string\[];

&#x20; models: ModelInfo\[];

&#x20; account: AccountInfo;

&#x20; fast\_mode\_state?: "off" | "cooldown" | "on";

};

```



\### `AgentDefinition`



Configuration for a subagent defined programmatically.



```typescript

type AgentDefinition = {

&#x20; description: string;

&#x20; tools?: string\[];

&#x20; disallowedTools?: string\[];

&#x20; prompt: string;

&#x20; model?: "sonnet" | "opus" | "haiku" | "inherit";

&#x20; mcpServers?: AgentMcpServerSpec\[];

&#x20; skills?: string\[];

&#x20; maxTurns?: number;

&#x20; criticalSystemReminder\_EXPERIMENTAL?: string;

};

```



| Field | Required | Description |

|:------|:---------|:------------|

| `description` | Yes | Natural language description of when to use this agent |

| `tools` | No | Array of allowed tool names. If omitted, inherits all tools from parent |

| `disallowedTools` | No | Array of tool names to explicitly disallow for this agent |

| `prompt` | Yes | The agent's system prompt |

| `model` | No | Model override for this agent. If omitted or `'inherit'`, uses the main model |

| `mcpServers` | No | MCP server specifications for this agent |

| `skills` | No | Array of skill names to preload into the agent context |

| `maxTurns` | No | Maximum number of agentic turns (API round-trips) before stopping |

| `criticalSystemReminder\_EXPERIMENTAL` | No | Experimental: Critical reminder added to the system prompt |



\### `AgentMcpServerSpec`



Specifies MCP servers available to a subagent. Can be a server name (string referencing a server from the parent's `mcpServers` config) or an inline server configuration record mapping server names to configs.



```typescript

type AgentMcpServerSpec = string | Record<string, McpServerConfigForProcessTransport>;

```



Where `McpServerConfigForProcessTransport` is `McpStdioServerConfig | McpSSEServerConfig | McpHttpServerConfig | McpSdkServerConfig`.



\### `SettingSource`



Controls which filesystem-based configuration sources the SDK loads settings from.



```typescript

type SettingSource = "user" | "project" | "local";

```



| Value | Description | Location |

|:------|:------------|:---------|

| `'user'` | Global user settings | `\~/.claude/settings.json` |

| `'project'` | Shared project settings (version controlled) | `.claude/settings.json` |

| `'local'` | Local project settings (gitignored) | `.claude/settings.local.json` |



\#### Default behavior



When `settingSources` is \*\*omitted\*\* or \*\*undefined\*\*, the SDK does \*\*not\*\* load any filesystem settings. This provides isolation for SDK applications.



\#### Why use settingSources



\*\*Load all filesystem settings (legacy behavior):\*\*

```typescript

// Load all settings like SDK v0.0.x did

const result = query({

&#x20; prompt: "Analyze this code",

&#x20; options: {

&#x20;   settingSources: \["user", "project", "local"] // Load all settings

&#x20; }

});

```



\*\*Load only specific setting sources:\*\*

```typescript

// Load only project settings, ignore user and local

const result = query({

&#x20; prompt: "Run CI checks",

&#x20; options: {

&#x20;   settingSources: \["project"] // Only .claude/settings.json

&#x20; }

});

```



\*\*Testing and CI environments:\*\*

```typescript

// Ensure consistent behavior in CI by excluding local settings

const result = query({

&#x20; prompt: "Run tests",

&#x20; options: {

&#x20;   settingSources: \["project"], // Only team-shared settings

&#x20;   permissionMode: "bypassPermissions"

&#x20; }

});

```



\*\*SDK-only applications:\*\*

```typescript

// Define everything programmatically (default behavior)

// No filesystem dependencies - settingSources defaults to \[]

const result = query({

&#x20; prompt: "Review this PR",

&#x20; options: {

&#x20;   // settingSources: \[] is the default, no need to specify

&#x20;   agents: {

&#x20;     /\* ... \*/

&#x20;   },

&#x20;   mcpServers: {

&#x20;     /\* ... \*/

&#x20;   },

&#x20;   allowedTools: \["Read", "Grep", "Glob"]

&#x20; }

});

```



\*\*Loading CLAUDE.md project instructions:\*\*

```typescript

// Load project settings to include CLAUDE.md files

const result = query({

&#x20; prompt: "Add a new feature following project conventions",

&#x20; options: {

&#x20;   systemPrompt: {

&#x20;     type: "preset",

&#x20;     preset: "claude\_code" // Required to use CLAUDE.md

&#x20;   },

&#x20;   settingSources: \["project"], // Loads CLAUDE.md from project directory

&#x20;   allowedTools: \["Read", "Write", "Edit"]

&#x20; }

});

```



\#### Settings precedence



When multiple sources are loaded, settings are merged with this precedence (highest to lowest):

1\. Local settings (`.claude/settings.local.json`)

2\. Project settings (`.claude/settings.json`)

3\. User settings (`\~/.claude/settings.json`)



Programmatic options (like `agents`, `allowedTools`) always override filesystem settings.



\### `PermissionMode`



```typescript

type PermissionMode =

&#x20; | "default" // Standard permission behavior

&#x20; | "acceptEdits" // Auto-accept file edits

&#x20; | "bypassPermissions" // Bypass all permission checks

&#x20; | "plan" // Planning mode - no execution

&#x20; | "dontAsk"; // Don't prompt for permissions, deny if not pre-approved

```



\### `CanUseTool`



Custom permission function type for controlling tool usage.



```typescript

type CanUseTool = (

&#x20; toolName: string,

&#x20; input: Record<string, unknown>,

&#x20; options: {

&#x20;   signal: AbortSignal;

&#x20;   suggestions?: PermissionUpdate\[];

&#x20;   blockedPath?: string;

&#x20;   decisionReason?: string;

&#x20;   toolUseID: string;

&#x20;   agentID?: string;

&#x20; }

) => Promise<PermissionResult>;

```



| Option | Type | Description |

| :----- | :--- | :---------- |

| `signal` | `AbortSignal` | Signaled if the operation should be aborted |

| `suggestions` | \[`PermissionUpdate`](#permission-update)`\[]` | Suggested permission updates so the user is not prompted again for this tool |

| `blockedPath` | `string` | The file path that triggered the permission request, if applicable |

| `decisionReason` | `string` | Explains why this permission request was triggered |

| `toolUseID` | `string` | Unique identifier for this specific tool call within the assistant message |

| `agentID` | `string` | If running within a sub-agent, the sub-agent's ID |



\### `PermissionResult`



Result of a permission check.



```typescript

type PermissionResult =

&#x20; | {

&#x20;     behavior: "allow";

&#x20;     updatedInput?: Record<string, unknown>;

&#x20;     updatedPermissions?: PermissionUpdate\[];

&#x20;     toolUseID?: string;

&#x20;   }

&#x20; | {

&#x20;     behavior: "deny";

&#x20;     message: string;

&#x20;     interrupt?: boolean;

&#x20;     toolUseID?: string;

&#x20;   };

```



\### `ToolConfig`



Configuration for built-in tool behavior.



```typescript

type ToolConfig = {

&#x20; askUserQuestion?: {

&#x20;   previewFormat?: "markdown" | "html";

&#x20; };

};

```



| Field | Type | Description |

|:------|:-----|:------------|

| `askUserQuestion.previewFormat` | `'markdown' \\| 'html'` | Opts into the `preview` field on \[`AskUserQuestion`](/docs/en/agent-sdk/user-input#question-format) options and sets its content format. When unset, Claude does not emit previews |



\### `McpServerConfig`



Configuration for MCP servers.



```typescript

type McpServerConfig =

&#x20; | McpStdioServerConfig

&#x20; | McpSSEServerConfig

&#x20; | McpHttpServerConfig

&#x20; | McpSdkServerConfigWithInstance;

```



\#### `McpStdioServerConfig`



```typescript

type McpStdioServerConfig = {

&#x20; type?: "stdio";

&#x20; command: string;

&#x20; args?: string\[];

&#x20; env?: Record<string, string>;

};

```



\#### `McpSSEServerConfig`



```typescript

type McpSSEServerConfig = {

&#x20; type: "sse";

&#x20; url: string;

&#x20; headers?: Record<string, string>;

};

```



\#### `McpHttpServerConfig`



```typescript

type McpHttpServerConfig = {

&#x20; type: "http";

&#x20; url: string;

&#x20; headers?: Record<string, string>;

};

```



\#### `McpSdkServerConfigWithInstance`



```typescript

type McpSdkServerConfigWithInstance = {

&#x20; type: "sdk";

&#x20; name: string;

&#x20; instance: McpServer;

};

```



\#### `McpClaudeAIProxyServerConfig`



```typescript

type McpClaudeAIProxyServerConfig = {

&#x20; type: "claudeai-proxy";

&#x20; url: string;

&#x20; id: string;

};

```



\### `SdkPluginConfig`



Configuration for loading plugins in the SDK.



```typescript

type SdkPluginConfig = {

&#x20; type: "local";

&#x20; path: string;

};

```



| Field | Type | Description |

|:------|:-----|:------------|

| `type` | `'local'` | Must be `'local'` (only local plugins currently supported) |

| `path` | `string` | Absolute or relative path to the plugin directory |



\*\*Example:\*\*

```typescript

plugins: \[

&#x20; { type: "local", path: "./my-plugin" },

&#x20; { type: "local", path: "/absolute/path/to/plugin" }

];

```



For complete information on creating and using plugins, see \[Plugins](/docs/en/agent-sdk/plugins).



\## Message Types



\### `SDKMessage`



Union type of all possible messages returned by the query.



```typescript

type SDKMessage =

&#x20; | SDKAssistantMessage

&#x20; | SDKUserMessage

&#x20; | SDKUserMessageReplay

&#x20; | SDKResultMessage

&#x20; | SDKSystemMessage

&#x20; | SDKPartialAssistantMessage

&#x20; | SDKCompactBoundaryMessage

&#x20; | SDKStatusMessage

&#x20; | SDKLocalCommandOutputMessage

&#x20; | SDKHookStartedMessage

&#x20; | SDKHookProgressMessage

&#x20; | SDKHookResponseMessage

&#x20; | SDKToolProgressMessage

&#x20; | SDKAuthStatusMessage

&#x20; | SDKTaskNotificationMessage

&#x20; | SDKTaskStartedMessage

&#x20; | SDKTaskProgressMessage

&#x20; | SDKFilesPersistedEvent

&#x20; | SDKToolUseSummaryMessage

&#x20; | SDKRateLimitEvent

&#x20; | SDKPromptSuggestionMessage;

```



\### `SDKAssistantMessage`



Assistant response message.



```typescript

type SDKAssistantMessage = {

&#x20; type: "assistant";

&#x20; uuid: UUID;

&#x20; session\_id: string;

&#x20; message: BetaMessage; // From Anthropic SDK

&#x20; parent\_tool\_use\_id: string | null;

&#x20; error?: SDKAssistantMessageError;

};

```



The `message` field is a \[`BetaMessage`](/docs/en/api/messages) from the Anthropic SDK. It includes fields like `id`, `content`, `model`, `stop\_reason`, and `usage`.



`SDKAssistantMessageError` is one of: `'authentication\_failed'`, `'billing\_error'`, `'rate\_limit'`, `'invalid\_request'`, `'server\_error'`, `'max\_output\_tokens'`, or `'unknown'`.



\### `SDKUserMessage`



User input message.



```typescript

type SDKUserMessage = {

&#x20; type: "user";

&#x20; uuid?: UUID;

&#x20; session\_id: string;

&#x20; message: MessageParam; // From Anthropic SDK

&#x20; parent\_tool\_use\_id: string | null;

&#x20; isSynthetic?: boolean;

&#x20; tool\_use\_result?: unknown;

};

```



\### `SDKUserMessageReplay`



Replayed user message with required UUID.



```typescript

type SDKUserMessageReplay = {

&#x20; type: "user";

&#x20; uuid: UUID;

&#x20; session\_id: string;

&#x20; message: MessageParam;

&#x20; parent\_tool\_use\_id: string | null;

&#x20; isSynthetic?: boolean;

&#x20; tool\_use\_result?: unknown;

&#x20; isReplay: true;

};

```



\### `SDKResultMessage`



Final result message.



```typescript

type SDKResultMessage =

&#x20; | {

&#x20;     type: "result";

&#x20;     subtype: "success";

&#x20;     uuid: UUID;

&#x20;     session\_id: string;

&#x20;     duration\_ms: number;

&#x20;     duration\_api\_ms: number;

&#x20;     is\_error: boolean;

&#x20;     num\_turns: number;

&#x20;     result: string;

&#x20;     stop\_reason: string | null;

&#x20;     total\_cost\_usd: number;

&#x20;     usage: NonNullableUsage;

&#x20;     modelUsage: { \[modelName: string]: ModelUsage };

&#x20;     permission\_denials: SDKPermissionDenial\[];

&#x20;     structured\_output?: unknown;

&#x20;   }

&#x20; | {

&#x20;     type: "result";

&#x20;     subtype:

&#x20;       | "error\_max\_turns"

&#x20;       | "error\_during\_execution"

&#x20;       | "error\_max\_budget\_usd"

&#x20;       | "error\_max\_structured\_output\_retries";

&#x20;     uuid: UUID;

&#x20;     session\_id: string;

&#x20;     duration\_ms: number;

&#x20;     duration\_api\_ms: number;

&#x20;     is\_error: boolean;

&#x20;     num\_turns: number;

&#x20;     stop\_reason: string | null;

&#x20;     total\_cost\_usd: number;

&#x20;     usage: NonNullableUsage;

&#x20;     modelUsage: { \[modelName: string]: ModelUsage };

&#x20;     permission\_denials: SDKPermissionDenial\[];

&#x20;     errors: string\[];

&#x20;   };

```



\### `SDKSystemMessage`



System initialization message.



```typescript

type SDKSystemMessage = {

&#x20; type: "system";

&#x20; subtype: "init";

&#x20; uuid: UUID;

&#x20; session\_id: string;

&#x20; agents?: string\[];

&#x20; apiKeySource: ApiKeySource;

&#x20; betas?: string\[];

&#x20; claude\_code\_version: string;

&#x20; cwd: string;

&#x20; tools: string\[];

&#x20; mcp\_servers: {

&#x20;   name: string;

&#x20;   status: string;

&#x20; }\[];

&#x20; model: string;

&#x20; permissionMode: PermissionMode;

&#x20; slash\_commands: string\[];

&#x20; output\_style: string;

&#x20; skills: string\[];

&#x20; plugins: { name: string; path: string }\[];

};

```



\### `SDKPartialAssistantMessage`



Streaming partial message (only when `includePartialMessages` is true).



```typescript

type SDKPartialAssistantMessage = {

&#x20; type: "stream\_event";

&#x20; event: BetaRawMessageStreamEvent; // From Anthropic SDK

&#x20; parent\_tool\_use\_id: string | null;

&#x20; uuid: UUID;

&#x20; session\_id: string;

};

```



\### `SDKCompactBoundaryMessage`



Message indicating a conversation compaction boundary.



```typescript

type SDKCompactBoundaryMessage = {

&#x20; type: "system";

&#x20; subtype: "compact\_boundary";

&#x20; uuid: UUID;

&#x20; session\_id: string;

&#x20; compact\_metadata: {

&#x20;   trigger: "manual" | "auto";

&#x20;   pre\_tokens: number;

&#x20; };

};

```



\### `SDKPermissionDenial`



Information about a denied tool use.



```typescript

type SDKPermissionDenial = {

&#x20; tool\_name: string;

&#x20; tool\_use\_id: string;

&#x20; tool\_input: Record<string, unknown>;

};

```



\## Hook Types



For a comprehensive guide on using hooks with examples and common patterns, see the \[Hooks guide](/docs/en/agent-sdk/hooks).



\### `HookEvent`



Available hook events.



```typescript

type HookEvent =

&#x20; | "PreToolUse"

&#x20; | "PostToolUse"

&#x20; | "PostToolUseFailure"

&#x20; | "Notification"

&#x20; | "UserPromptSubmit"

&#x20; | "SessionStart"

&#x20; | "SessionEnd"

&#x20; | "Stop"

&#x20; | "SubagentStart"

&#x20; | "SubagentStop"

&#x20; | "PreCompact"

&#x20; | "PermissionRequest"

&#x20; | "Setup"

&#x20; | "TeammateIdle"

&#x20; | "TaskCompleted"

&#x20; | "ConfigChange"

&#x20; | "WorktreeCreate"

&#x20; | "WorktreeRemove";

```



\### `HookCallback`



Hook callback function type.



```typescript

type HookCallback = (

&#x20; input: HookInput, // Union of all hook input types

&#x20; toolUseID: string | undefined,

&#x20; options: { signal: AbortSignal }

) => Promise<HookJSONOutput>;

```



\### `HookCallbackMatcher`



Hook configuration with optional matcher.



```typescript

interface HookCallbackMatcher {

&#x20; matcher?: string;

&#x20; hooks: HookCallback\[];

&#x20; timeout?: number; // Timeout in seconds for all hooks in this matcher

}

```



\### `HookInput`



Union type of all hook input types.



```typescript

type HookInput =

&#x20; | PreToolUseHookInput

&#x20; | PostToolUseHookInput

&#x20; | PostToolUseFailureHookInput

&#x20; | NotificationHookInput

&#x20; | UserPromptSubmitHookInput

&#x20; | SessionStartHookInput

&#x20; | SessionEndHookInput

&#x20; | StopHookInput

&#x20; | SubagentStartHookInput

&#x20; | SubagentStopHookInput

&#x20; | PreCompactHookInput

&#x20; | PermissionRequestHookInput

&#x20; | SetupHookInput

&#x20; | TeammateIdleHookInput

&#x20; | TaskCompletedHookInput

&#x20; | ConfigChangeHookInput

&#x20; | WorktreeCreateHookInput

&#x20; | WorktreeRemoveHookInput;

```



\### `BaseHookInput`



Base interface that all hook input types extend.



```typescript

type BaseHookInput = {

&#x20; session\_id: string;

&#x20; transcript\_path: string;

&#x20; cwd: string;

&#x20; permission\_mode?: string;

&#x20; agent\_id?: string;

&#x20; agent\_type?: string;

};

```



\#### `PreToolUseHookInput`



```typescript

type PreToolUseHookInput = BaseHookInput \& {

&#x20; hook\_event\_name: "PreToolUse";

&#x20; tool\_name: string;

&#x20; tool\_input: unknown;

&#x20; tool\_use\_id: string;

};

```



\#### `PostToolUseHookInput`



```typescript

type PostToolUseHookInput = BaseHookInput \& {

&#x20; hook\_event\_name: "PostToolUse";

&#x20; tool\_name: string;

&#x20; tool\_input: unknown;

&#x20; tool\_response: unknown;

&#x20; tool\_use\_id: string;

};

```



\#### `PostToolUseFailureHookInput`



```typescript

type PostToolUseFailureHookInput = BaseHookInput \& {

&#x20; hook\_event\_name: "PostToolUseFailure";

&#x20; tool\_name: string;

&#x20; tool\_input: unknown;

&#x20; tool\_use\_id: string;

&#x20; error: string;

&#x20; is\_interrupt?: boolean;

};

```



\#### `NotificationHookInput`



```typescript

type NotificationHookInput = BaseHookInput \& {

&#x20; hook\_event\_name: "Notification";

&#x20; message: string;

&#x20; title?: string;

&#x20; notification\_type: string;

};

```



\#### `UserPromptSubmitHookInput`



```typescript

type UserPromptSubmitHookInput = BaseHookInput \& {

&#x20; hook\_event\_name: "UserPromptSubmit";

&#x20; prompt: string;

};

```



\#### `SessionStartHookInput`



```typescript

type SessionStartHookInput = BaseHookInput \& {

&#x20; hook\_event\_name: "SessionStart";

&#x20; source: "startup" | "resume" | "clear" | "compact";

&#x20; agent\_type?: string;

&#x20; model?: string;

};

```



\#### `SessionEndHookInput`



```typescript

type SessionEndHookInput = BaseHookInput \& {

&#x20; hook\_event\_name: "SessionEnd";

&#x20; reason: ExitReason; // String from EXIT\_REASONS array

};

```



\#### `StopHookInput`



```typescript

type StopHookInput = BaseHookInput \& {

&#x20; hook\_event\_name: "Stop";

&#x20; stop\_hook\_active: boolean;

&#x20; last\_assistant\_message?: string;

};

```



\#### `SubagentStartHookInput`



```typescript

type SubagentStartHookInput = BaseHookInput \& {

&#x20; hook\_event\_name: "SubagentStart";

&#x20; agent\_id: string;

&#x20; agent\_type: string;

};

```



\#### `SubagentStopHookInput`



```typescript

type SubagentStopHookInput = BaseHookInput \& {

&#x20; hook\_event\_name: "SubagentStop";

&#x20; stop\_hook\_active: boolean;

&#x20; agent\_id: string;

&#x20; agent\_transcript\_path: string;

&#x20; agent\_type: string;

&#x20; last\_assistant\_message?: string;

};

```



\#### `PreCompactHookInput`



```typescript

type PreCompactHookInput = BaseHookInput \& {

&#x20; hook\_event\_name: "PreCompact";

&#x20; trigger: "manual" | "auto";

&#x20; custom\_instructions: string | null;

};

```



\#### `PermissionRequestHookInput`



```typescript

type PermissionRequestHookInput = BaseHookInput \& {

&#x20; hook\_event\_name: "PermissionRequest";

&#x20; tool\_name: string;

&#x20; tool\_input: unknown;

&#x20; permission\_suggestions?: PermissionUpdate\[];

};

```



\#### `SetupHookInput`



```typescript

type SetupHookInput = BaseHookInput \& {

&#x20; hook\_event\_name: "Setup";

&#x20; trigger: "init" | "maintenance";

};

```



\#### `TeammateIdleHookInput`



```typescript

type TeammateIdleHookInput = BaseHookInput \& {

&#x20; hook\_event\_name: "TeammateIdle";

&#x20; teammate\_name: string;

&#x20; team\_name: string;

};

```



\#### `TaskCompletedHookInput`



```typescript

type TaskCompletedHookInput = BaseHookInput \& {

&#x20; hook\_event\_name: "TaskCompleted";

&#x20; task\_id: string;

&#x20; task\_subject: string;

&#x20; task\_description?: string;

&#x20; teammate\_name?: string;

&#x20; team\_name?: string;

};

```



\#### `ConfigChangeHookInput`



```typescript

type ConfigChangeHookInput = BaseHookInput \& {

&#x20; hook\_event\_name: "ConfigChange";

&#x20; source:

&#x20;   | "user\_settings"

&#x20;   | "project\_settings"

&#x20;   | "local\_settings"

&#x20;   | "policy\_settings"

&#x20;   | "skills";

&#x20; file\_path?: string;

};

```



\#### `WorktreeCreateHookInput`



```typescript

type WorktreeCreateHookInput = BaseHookInput \& {

&#x20; hook\_event\_name: "WorktreeCreate";

&#x20; name: string;

};

```



\#### `WorktreeRemoveHookInput`



```typescript

type WorktreeRemoveHookInput = BaseHookInput \& {

&#x20; hook\_event\_name: "WorktreeRemove";

&#x20; worktree\_path: string;

};

```



\### `HookJSONOutput`



Hook return value.



```typescript

type HookJSONOutput = AsyncHookJSONOutput | SyncHookJSONOutput;

```



\#### `AsyncHookJSONOutput`



```typescript

type AsyncHookJSONOutput = {

&#x20; async: true;

&#x20; asyncTimeout?: number;

};

```



\#### `SyncHookJSONOutput`



```typescript

type SyncHookJSONOutput = {

&#x20; continue?: boolean;

&#x20; suppressOutput?: boolean;

&#x20; stopReason?: string;

&#x20; decision?: "approve" | "block";

&#x20; systemMessage?: string;

&#x20; reason?: string;

&#x20; hookSpecificOutput?:

&#x20;   | {

&#x20;       hookEventName: "PreToolUse";

&#x20;       permissionDecision?: "allow" | "deny" | "ask";

&#x20;       permissionDecisionReason?: string;

&#x20;       updatedInput?: Record<string, unknown>;

&#x20;       additionalContext?: string;

&#x20;     }

&#x20;   | {

&#x20;       hookEventName: "UserPromptSubmit";

&#x20;       additionalContext?: string;

&#x20;     }

&#x20;   | {

&#x20;       hookEventName: "SessionStart";

&#x20;       additionalContext?: string;

&#x20;     }

&#x20;   | {

&#x20;       hookEventName: "Setup";

&#x20;       additionalContext?: string;

&#x20;     }

&#x20;   | {

&#x20;       hookEventName: "SubagentStart";

&#x20;       additionalContext?: string;

&#x20;     }

&#x20;   | {

&#x20;       hookEventName: "PostToolUse";

&#x20;       additionalContext?: string;

&#x20;       updatedMCPToolOutput?: unknown;

&#x20;     }

&#x20;   | {

&#x20;       hookEventName: "PostToolUseFailure";

&#x20;       additionalContext?: string;

&#x20;     }

&#x20;   | {

&#x20;       hookEventName: "Notification";

&#x20;       additionalContext?: string;

&#x20;     }

&#x20;   | {

&#x20;       hookEventName: "PermissionRequest";

&#x20;       decision:

&#x20;         | {

&#x20;             behavior: "allow";

&#x20;             updatedInput?: Record<string, unknown>;

&#x20;             updatedPermissions?: PermissionUpdate\[];

&#x20;           }

&#x20;         | {

&#x20;             behavior: "deny";

&#x20;             message?: string;

&#x20;             interrupt?: boolean;

&#x20;           };

&#x20;     };

};

```



\## Tool Input Types



Documentation of input schemas for all built-in Claude Code tools. These types are exported from `@anthropic-ai/claude-agent-sdk` and can be used for type-safe tool interactions.



\### `ToolInputSchemas`



Union of all tool input types, exported from `@anthropic-ai/claude-agent-sdk`.



```typescript

type ToolInputSchemas =

&#x20; | AgentInput

&#x20; | AskUserQuestionInput

&#x20; | BashInput

&#x20; | TaskOutputInput

&#x20; | ConfigInput

&#x20; | EnterWorktreeInput

&#x20; | ExitPlanModeInput

&#x20; | FileEditInput

&#x20; | FileReadInput

&#x20; | FileWriteInput

&#x20; | GlobInput

&#x20; | GrepInput

&#x20; | ListMcpResourcesInput

&#x20; | McpInput

&#x20; | NotebookEditInput

&#x20; | ReadMcpResourceInput

&#x20; | SubscribeMcpResourceInput

&#x20; | SubscribePollingInput

&#x20; | TaskStopInput

&#x20; | TodoWriteInput

&#x20; | UnsubscribeMcpResourceInput

&#x20; | UnsubscribePollingInput

&#x20; | WebFetchInput

&#x20; | WebSearchInput;

```



\### Agent



\*\*Tool name:\*\* `Agent` (previously `Task`, which is still accepted as an alias)



```typescript

type AgentInput = {

&#x20; description: string;

&#x20; prompt: string;

&#x20; subagent\_type: string;

&#x20; model?: "sonnet" | "opus" | "haiku";

&#x20; resume?: string;

&#x20; run\_in\_background?: boolean;

&#x20; max\_turns?: number;

&#x20; name?: string;

&#x20; team\_name?: string;

&#x20; mode?: "acceptEdits" | "bypassPermissions" | "default" | "dontAsk" | "plan";

&#x20; isolation?: "worktree";

};

```



Launches a new agent to handle complex, multi-step tasks autonomously.



\### AskUserQuestion



\*\*Tool name:\*\* `AskUserQuestion`



```typescript

type AskUserQuestionInput = {

&#x20; questions: Array<{

&#x20;   question: string;

&#x20;   header: string;

&#x20;   options: Array<{ label: string; description: string; preview?: string }>;

&#x20;   multiSelect: boolean;

&#x20; }>;

};

```



Asks the user clarifying questions during execution. See \[Handle approvals and user input](/docs/en/agent-sdk/user-input#handle-clarifying-questions) for usage details.



\### Bash



\*\*Tool name:\*\* `Bash`



```typescript

type BashInput = {

&#x20; command: string;

&#x20; timeout?: number;

&#x20; description?: string;

&#x20; run\_in\_background?: boolean;

&#x20; dangerouslyDisableSandbox?: boolean;

};

```



Executes bash commands in a persistent shell session with optional timeout and background execution.



\### TaskOutput



\*\*Tool name:\*\* `TaskOutput`



```typescript

type TaskOutputInput = {

&#x20; task\_id: string;

&#x20; block: boolean;

&#x20; timeout: number;

};

```



Retrieves output from a running or completed background task.



\### Edit



\*\*Tool name:\*\* `Edit`



```typescript

type FileEditInput = {

&#x20; file\_path: string;

&#x20; old\_string: string;

&#x20; new\_string: string;

&#x20; replace\_all?: boolean;

};

```



Performs exact string replacements in files.



\### Read



\*\*Tool name:\*\* `Read`



```typescript

type FileReadInput = {

&#x20; file\_path: string;

&#x20; offset?: number;

&#x20; limit?: number;

&#x20; pages?: string;

};

```



Reads files from the local filesystem, including text, images, PDFs, and Jupyter notebooks. Use `pages` for PDF page ranges (for example, `"1-5"`).



\### Write



\*\*Tool name:\*\* `Write`



```typescript

type FileWriteInput = {

&#x20; file\_path: string;

&#x20; content: string;

};

```



Writes a file to the local filesystem, overwriting if it exists.



\### Glob



\*\*Tool name:\*\* `Glob`



```typescript

type GlobInput = {

&#x20; pattern: string;

&#x20; path?: string;

};

```



Fast file pattern matching that works with any codebase size.



\### Grep



\*\*Tool name:\*\* `Grep`



```typescript

type GrepInput = {

&#x20; pattern: string;

&#x20; path?: string;

&#x20; glob?: string;

&#x20; type?: string;

&#x20; output\_mode?: "content" | "files\_with\_matches" | "count";

&#x20; "-i"?: boolean;

&#x20; "-n"?: boolean;

&#x20; "-B"?: number;

&#x20; "-A"?: number;

&#x20; "-C"?: number;

&#x20; context?: number;

&#x20; head\_limit?: number;

&#x20; offset?: number;

&#x20; multiline?: boolean;

};

```



Powerful search tool built on ripgrep with regex support.



\### TaskStop



\*\*Tool name:\*\* `TaskStop`



```typescript

type TaskStopInput = {

&#x20; task\_id?: string;

&#x20; shell\_id?: string; // Deprecated: use task\_id

};

```



Stops a running background task or shell by ID.



\### NotebookEdit



\*\*Tool name:\*\* `NotebookEdit`



```typescript

type NotebookEditInput = {

&#x20; notebook\_path: string;

&#x20; cell\_id?: string;

&#x20; new\_source: string;

&#x20; cell\_type?: "code" | "markdown";

&#x20; edit\_mode?: "replace" | "insert" | "delete";

};

```



Edits cells in Jupyter notebook files.



\### WebFetch



\*\*Tool name:\*\* `WebFetch`



```typescript

type WebFetchInput = {

&#x20; url: string;

&#x20; prompt: string;

};

```



Fetches content from a URL and processes it with an AI model.



\### WebSearch



\*\*Tool name:\*\* `WebSearch`



```typescript

type WebSearchInput = {

&#x20; query: string;

&#x20; allowed\_domains?: string\[];

&#x20; blocked\_domains?: string\[];

};

```



Searches the web and returns formatted results.



\### TodoWrite



\*\*Tool name:\*\* `TodoWrite`



```typescript

type TodoWriteInput = {

&#x20; todos: Array<{

&#x20;   content: string;

&#x20;   status: "pending" | "in\_progress" | "completed";

&#x20;   activeForm: string;

&#x20; }>;

};

```



Creates and manages a structured task list for tracking progress.



\### ExitPlanMode



\*\*Tool name:\*\* `ExitPlanMode`



```typescript

type ExitPlanModeInput = {

&#x20; allowedPrompts?: Array<{

&#x20;   tool: "Bash";

&#x20;   prompt: string;

&#x20; }>;

};

```



Exits planning mode. Optionally specifies prompt-based permissions needed to implement the plan.



\### ListMcpResources



\*\*Tool name:\*\* `ListMcpResources`



```typescript

type ListMcpResourcesInput = {

&#x20; server?: string;

};

```



Lists available MCP resources from connected servers.



\### ReadMcpResource



\*\*Tool name:\*\* `ReadMcpResource`



```typescript

type ReadMcpResourceInput = {

&#x20; server: string;

&#x20; uri: string;

};

```



Reads a specific MCP resource from a server.



\### Config



\*\*Tool name:\*\* `Config`



```typescript

type ConfigInput = {

&#x20; setting: string;

&#x20; value?: string | boolean | number;

};

```



Gets or sets a configuration value.



\### EnterWorktree



\*\*Tool name:\*\* `EnterWorktree`



```typescript

type EnterWorktreeInput = {

&#x20; name?: string;

};

```



Creates and enters a temporary git worktree for isolated work.



\## Tool Output Types



Documentation of output schemas for all built-in Claude Code tools. These types are exported from `@anthropic-ai/claude-agent-sdk` and represent the actual response data returned by each tool.



\### `ToolOutputSchemas`



Union of all tool output types.



```typescript

type ToolOutputSchemas =

&#x20; | AgentOutput

&#x20; | AskUserQuestionOutput

&#x20; | BashOutput

&#x20; | ConfigOutput

&#x20; | EnterWorktreeOutput

&#x20; | ExitPlanModeOutput

&#x20; | FileEditOutput

&#x20; | FileReadOutput

&#x20; | FileWriteOutput

&#x20; | GlobOutput

&#x20; | GrepOutput

&#x20; | ListMcpResourcesOutput

&#x20; | NotebookEditOutput

&#x20; | ReadMcpResourceOutput

&#x20; | TaskStopOutput

&#x20; | TodoWriteOutput

&#x20; | WebFetchOutput

&#x20; | WebSearchOutput;

```



\### Agent



\*\*Tool name:\*\* `Agent` (previously `Task`, which is still accepted as an alias)



```typescript

type AgentOutput =

&#x20; | {

&#x20;     status: "completed";

&#x20;     agentId: string;

&#x20;     content: Array<{ type: "text"; text: string }>;

&#x20;     totalToolUseCount: number;

&#x20;     totalDurationMs: number;

&#x20;     totalTokens: number;

&#x20;     usage: {

&#x20;       input\_tokens: number;

&#x20;       output\_tokens: number;

&#x20;       cache\_creation\_input\_tokens: number | null;

&#x20;       cache\_read\_input\_tokens: number | null;

&#x20;       server\_tool\_use: {

&#x20;         web\_search\_requests: number;

&#x20;         web\_fetch\_requests: number;

&#x20;       } | null;

&#x20;       service\_tier: ("standard" | "priority" | "batch") | null;

&#x20;       cache\_creation: {

&#x20;         ephemeral\_1h\_input\_tokens: number;

&#x20;         ephemeral\_5m\_input\_tokens: number;

&#x20;       } | null;

&#x20;     };

&#x20;     prompt: string;

&#x20;   }

&#x20; | {

&#x20;     status: "async\_launched";

&#x20;     agentId: string;

&#x20;     description: string;

&#x20;     prompt: string;

&#x20;     outputFile: string;

&#x20;     canReadOutputFile?: boolean;

&#x20;   }

&#x20; | {

&#x20;     status: "sub\_agent\_entered";

&#x20;     description: string;

&#x20;     message: string;

&#x20;   };

```



Returns the result from the subagent. Discriminated on the `status` field: `"completed"` for finished tasks, `"async\_launched"` for background tasks, and `"sub\_agent\_entered"` for interactive subagents.



\### AskUserQuestion



\*\*Tool name:\*\* `AskUserQuestion`



```typescript

type AskUserQuestionOutput = {

&#x20; questions: Array<{

&#x20;   question: string;

&#x20;   header: string;

&#x20;   options: Array<{ label: string; description: string; preview?: string }>;

&#x20;   multiSelect: boolean;

&#x20; }>;

&#x20; answers: Record<string, string>;

};

```



Returns the questions asked and the user's answers.



\### Bash



\*\*Tool name:\*\* `Bash`



```typescript

type BashOutput = {

&#x20; stdout: string;

&#x20; stderr: string;

&#x20; rawOutputPath?: string;

&#x20; interrupted: boolean;

&#x20; isImage?: boolean;

&#x20; backgroundTaskId?: string;

&#x20; backgroundedByUser?: boolean;

&#x20; dangerouslyDisableSandbox?: boolean;

&#x20; returnCodeInterpretation?: string;

&#x20; structuredContent?: unknown\[];

&#x20; persistedOutputPath?: string;

&#x20; persistedOutputSize?: number;

};

```



Returns command output with stdout/stderr split. Background commands include a `backgroundTaskId`.



\### Edit



\*\*Tool name:\*\* `Edit`



```typescript

type FileEditOutput = {

&#x20; filePath: string;

&#x20; oldString: string;

&#x20; newString: string;

&#x20; originalFile: string;

&#x20; structuredPatch: Array<{

&#x20;   oldStart: number;

&#x20;   oldLines: number;

&#x20;   newStart: number;

&#x20;   newLines: number;

&#x20;   lines: string\[];

&#x20; }>;

&#x20; userModified: boolean;

&#x20; replaceAll: boolean;

&#x20; gitDiff?: {

&#x20;   filename: string;

&#x20;   status: "modified" | "added";

&#x20;   additions: number;

&#x20;   deletions: number;

&#x20;   changes: number;

&#x20;   patch: string;

&#x20; };

};

```



Returns the structured diff of the edit operation.



\### Read



\*\*Tool name:\*\* `Read`



```typescript

type FileReadOutput =

&#x20; | {

&#x20;     type: "text";

&#x20;     file: {

&#x20;       filePath: string;

&#x20;       content: string;

&#x20;       numLines: number;

&#x20;       startLine: number;

&#x20;       totalLines: number;

&#x20;     };

&#x20;   }

&#x20; | {

&#x20;     type: "image";

&#x20;     file: {

&#x20;       base64: string;

&#x20;       type: "image/jpeg" | "image/png" | "image/gif" | "image/webp";

&#x20;       originalSize: number;

&#x20;       dimensions?: {

&#x20;         originalWidth?: number;

&#x20;         originalHeight?: number;

&#x20;         displayWidth?: number;

&#x20;         displayHeight?: number;

&#x20;       };

&#x20;     };

&#x20;   }

&#x20; | {

&#x20;     type: "notebook";

&#x20;     file: {

&#x20;       filePath: string;

&#x20;       cells: unknown\[];

&#x20;     };

&#x20;   }

&#x20; | {

&#x20;     type: "pdf";

&#x20;     file: {

&#x20;       filePath: string;

&#x20;       base64: string;

&#x20;       originalSize: number;

&#x20;     };

&#x20;   }

&#x20; | {

&#x20;     type: "parts";

&#x20;     file: {

&#x20;       filePath: string;

&#x20;       originalSize: number;

&#x20;       count: number;

&#x20;       outputDir: string;

&#x20;     };

&#x20;   };

```



Returns file contents in a format appropriate to the file type. Discriminated on the `type` field.



\### Write



\*\*Tool name:\*\* `Write`



```typescript

type FileWriteOutput = {

&#x20; type: "create" | "update";

&#x20; filePath: string;

&#x20; content: string;

&#x20; structuredPatch: Array<{

&#x20;   oldStart: number;

&#x20;   oldLines: number;

&#x20;   newStart: number;

&#x20;   newLines: number;

&#x20;   lines: string\[];

&#x20; }>;

&#x20; originalFile: string | null;

&#x20; gitDiff?: {

&#x20;   filename: string;

&#x20;   status: "modified" | "added";

&#x20;   additions: number;

&#x20;   deletions: number;

&#x20;   changes: number;

&#x20;   patch: string;

&#x20; };

};

```



Returns the write result with structured diff information.



\### Glob



\*\*Tool name:\*\* `Glob`



```typescript

type GlobOutput = {

&#x20; durationMs: number;

&#x20; numFiles: number;

&#x20; filenames: string\[];

&#x20; truncated: boolean;

};

```



Returns file paths matching the glob pattern, sorted by modification time.



\### Grep



\*\*Tool name:\*\* `Grep`



```typescript

type GrepOutput = {

&#x20; mode?: "content" | "files\_with\_matches" | "count";

&#x20; numFiles: number;

&#x20; filenames: string\[];

&#x20; content?: string;

&#x20; numLines?: number;

&#x20; numMatches?: number;

&#x20; appliedLimit?: number;

&#x20; appliedOffset?: number;

};

```



Returns search results. The shape varies by `mode`: file list, content with matches, or match counts.



\### TaskStop



\*\*Tool name:\*\* `TaskStop`



```typescript

type TaskStopOutput = {

&#x20; message: string;

&#x20; task\_id: string;

&#x20; task\_type: string;

&#x20; command?: string;

};

```



Returns confirmation after stopping the background task.



\### NotebookEdit



\*\*Tool name:\*\* `NotebookEdit`



```typescript

type NotebookEditOutput = {

&#x20; new\_source: string;

&#x20; cell\_id?: string;

&#x20; cell\_type: "code" | "markdown";

&#x20; language: string;

&#x20; edit\_mode: string;

&#x20; error?: string;

&#x20; notebook\_path: string;

&#x20; original\_file: string;

&#x20; updated\_file: string;

};

```



Returns the result of the notebook edit with original and updated file contents.



\### WebFetch



\*\*Tool name:\*\* `WebFetch`



```typescript

type WebFetchOutput = {

&#x20; bytes: number;

&#x20; code: number;

&#x20; codeText: string;

&#x20; result: string;

&#x20; durationMs: number;

&#x20; url: string;

};

```



Returns the fetched content with HTTP status and metadata.



\### WebSearch



\*\*Tool name:\*\* `WebSearch`



```typescript

type WebSearchOutput = {

&#x20; query: string;

&#x20; results: Array<

&#x20;   | {

&#x20;       tool\_use\_id: string;

&#x20;       content: Array<{ title: string; url: string }>;

&#x20;     }

&#x20;   | string

&#x20; >;

&#x20; durationSeconds: number;

};

```



Returns search results from the web.



\### TodoWrite



\*\*Tool name:\*\* `TodoWrite`



```typescript

type TodoWriteOutput = {

&#x20; oldTodos: Array<{

&#x20;   content: string;

&#x20;   status: "pending" | "in\_progress" | "completed";

&#x20;   activeForm: string;

&#x20; }>;

&#x20; newTodos: Array<{

&#x20;   content: string;

&#x20;   status: "pending" | "in\_progress" | "completed";

&#x20;   activeForm: string;

&#x20; }>;

};

```



Returns the previous and updated task lists.



\### ExitPlanMode



\*\*Tool name:\*\* `ExitPlanMode`



```typescript

type ExitPlanModeOutput = {

&#x20; plan: string | null;

&#x20; isAgent: boolean;

&#x20; filePath?: string;

&#x20; hasTaskTool?: boolean;

&#x20; awaitingLeaderApproval?: boolean;

&#x20; requestId?: string;

};

```



Returns the plan state after exiting plan mode.



\### ListMcpResources



\*\*Tool name:\*\* `ListMcpResources`



```typescript

type ListMcpResourcesOutput = Array<{

&#x20; uri: string;

&#x20; name: string;

&#x20; mimeType?: string;

&#x20; description?: string;

&#x20; server: string;

}>;

```



Returns an array of available MCP resources.



\### ReadMcpResource



\*\*Tool name:\*\* `ReadMcpResource`



```typescript

type ReadMcpResourceOutput = {

&#x20; contents: Array<{

&#x20;   uri: string;

&#x20;   mimeType?: string;

&#x20;   text?: string;

&#x20; }>;

};

```



Returns the contents of the requested MCP resource.



\### Config



\*\*Tool name:\*\* `Config`



```typescript

type ConfigOutput = {

&#x20; success: boolean;

&#x20; operation?: "get" | "set";

&#x20; setting?: string;

&#x20; value?: unknown;

&#x20; previousValue?: unknown;

&#x20; newValue?: unknown;

&#x20; error?: string;

};

```



Returns the result of a configuration get or set operation.



\### EnterWorktree



\*\*Tool name:\*\* `EnterWorktree`



```typescript

type EnterWorktreeOutput = {

&#x20; worktreePath: string;

&#x20; worktreeBranch?: string;

&#x20; message: string;

};

```



Returns information about the created git worktree.



\## Permission Types



\### `PermissionUpdate`



Operations for updating permissions.



```typescript

type PermissionUpdate =

&#x20; | {

&#x20;     type: "addRules";

&#x20;     rules: PermissionRuleValue\[];

&#x20;     behavior: PermissionBehavior;

&#x20;     destination: PermissionUpdateDestination;

&#x20;   }

&#x20; | {

&#x20;     type: "replaceRules";

&#x20;     rules: PermissionRuleValue\[];

&#x20;     behavior: PermissionBehavior;

&#x20;     destination: PermissionUpdateDestination;

&#x20;   }

&#x20; | {

&#x20;     type: "removeRules";

&#x20;     rules: PermissionRuleValue\[];

&#x20;     behavior: PermissionBehavior;

&#x20;     destination: PermissionUpdateDestination;

&#x20;   }

&#x20; | {

&#x20;     type: "setMode";

&#x20;     mode: PermissionMode;

&#x20;     destination: PermissionUpdateDestination;

&#x20;   }

&#x20; | {

&#x20;     type: "addDirectories";

&#x20;     directories: string\[];

&#x20;     destination: PermissionUpdateDestination;

&#x20;   }

&#x20; | {

&#x20;     type: "removeDirectories";

&#x20;     directories: string\[];

&#x20;     destination: PermissionUpdateDestination;

&#x20;   };

```



\### `PermissionBehavior`



```typescript

type PermissionBehavior = "allow" | "deny" | "ask";

```



\### `PermissionUpdateDestination`



```typescript

type PermissionUpdateDestination =

&#x20; | "userSettings" // Global user settings

&#x20; | "projectSettings" // Per-directory project settings

&#x20; | "localSettings" // Gitignored local settings

&#x20; | "session" // Current session only

&#x20; | "cliArg"; // CLI argument

```



\### `PermissionRuleValue`



```typescript

type PermissionRuleValue = {

&#x20; toolName: string;

&#x20; ruleContent?: string;

};

```



\## Other Types



\### `ApiKeySource`



```typescript

type ApiKeySource = "user" | "project" | "org" | "temporary" | "oauth";

```



\### `SdkBeta`



Available beta features that can be enabled via the `betas` option. See \[Beta headers](/docs/en/api/beta-headers) for more information.



```typescript

type SdkBeta = "context-1m-2025-08-07";

```



| Value | Description | Compatible Models |

|:------|:------------|:------------------|

| `'context-1m-2025-08-07'` | Enables the 1 million token \[context window](/docs/en/build-with-claude/context-windows). | Claude Sonnet 4.5, Claude Sonnet 4 |



<Note>

Claude Opus 4.6 and Sonnet 4.6 have a 1M token context window. Including `context-1m-2025-08-07` has no effect on those models.

</Note>



\### `SlashCommand`



Information about an available slash command.



```typescript

type SlashCommand = {

&#x20; name: string;

&#x20; description: string;

&#x20; argumentHint: string;

};

```



\### `ModelInfo`



Information about an available model.



```typescript

type ModelInfo = {

&#x20; value: string;

&#x20; displayName: string;

&#x20; description: string;

&#x20; supportsEffort?: boolean;

&#x20; supportedEffortLevels?: ("low" | "medium" | "high" | "max")\[];

&#x20; supportsAdaptiveThinking?: boolean;

&#x20; supportsFastMode?: boolean;

};

```



\### `AgentInfo`



Information about an available subagent that can be invoked via the Agent tool.



```typescript

type AgentInfo = {

&#x20; name: string;

&#x20; description: string;

&#x20; model?: string;

};

```



| Field | Type | Description |

|:------|:-----|:------------|

| `name` | `string` | Agent type identifier (e.g., `"Explore"`, `"general-purpose"`) |

| `description` | `string` | Description of when to use this agent |

| `model` | `string \\| undefined` | Model alias this agent uses. If omitted, inherits the parent's model |



\### `McpServerStatus`



Status of a connected MCP server.



```typescript

type McpServerStatus = {

&#x20; name: string;

&#x20; status: "connected" | "failed" | "needs-auth" | "pending" | "disabled";

&#x20; serverInfo?: {

&#x20;   name: string;

&#x20;   version: string;

&#x20; };

&#x20; error?: string;

&#x20; config?: McpServerStatusConfig;

&#x20; scope?: string;

&#x20; tools?: {

&#x20;   name: string;

&#x20;   description?: string;

&#x20;   annotations?: {

&#x20;     readOnly?: boolean;

&#x20;     destructive?: boolean;

&#x20;     openWorld?: boolean;

&#x20;   };

&#x20; }\[];

};

```



\### `McpServerStatusConfig`



The configuration of an MCP server as reported by `mcpServerStatus()`. This is the union of all MCP server transport types.



```typescript

type McpServerStatusConfig =

&#x20; | McpStdioServerConfig

&#x20; | McpSSEServerConfig

&#x20; | McpHttpServerConfig

&#x20; | McpSdkServerConfig

&#x20; | McpClaudeAIProxyServerConfig;

```



See \[`McpServerConfig`](#mcp-server-config) for details on each transport type.



\### `AccountInfo`



Account information for the authenticated user.



```typescript

type AccountInfo = {

&#x20; email?: string;

&#x20; organization?: string;

&#x20; subscriptionType?: string;

&#x20; tokenSource?: string;

&#x20; apiKeySource?: string;

};

```



\### `ModelUsage`



Per-model usage statistics returned in result messages.



```typescript

type ModelUsage = {

&#x20; inputTokens: number;

&#x20; outputTokens: number;

&#x20; cacheReadInputTokens: number;

&#x20; cacheCreationInputTokens: number;

&#x20; webSearchRequests: number;

&#x20; costUSD: number;

&#x20; contextWindow: number;

&#x20; maxOutputTokens: number;

};

```



\### `ConfigScope`



```typescript

type ConfigScope = "local" | "user" | "project";

```



\### `NonNullableUsage`



A version of \[`Usage`](#usage) with all nullable fields made non-nullable.



```typescript

type NonNullableUsage = {

&#x20; \[K in keyof Usage]: NonNullable<Usage\[K]>;

};

```



\### `Usage`



Token usage statistics (from `@anthropic-ai/sdk`).



```typescript

type Usage = {

&#x20; input\_tokens: number | null;

&#x20; output\_tokens: number | null;

&#x20; cache\_creation\_input\_tokens?: number | null;

&#x20; cache\_read\_input\_tokens?: number | null;

};

```



\### `CallToolResult`



MCP tool result type (from `@modelcontextprotocol/sdk/types.js`).



```typescript

type CallToolResult = {

&#x20; content: Array<{

&#x20;   type: "text" | "image" | "resource";

&#x20;   // Additional fields vary by type

&#x20; }>;

&#x20; isError?: boolean;

};

```



\### `ThinkingConfig`



Controls Claude's thinking/reasoning behavior. Takes precedence over the deprecated `maxThinkingTokens`.



```typescript

type ThinkingConfig =

&#x20; | { type: "adaptive" } // The model determines when and how much to reason (Opus 4.6+)

&#x20; | { type: "enabled"; budgetTokens?: number } // Fixed thinking token budget

&#x20; | { type: "disabled" }; // No extended thinking

```



\### `SpawnedProcess`



Interface for custom process spawning (used with `spawnClaudeCodeProcess` option). `ChildProcess` already satisfies this interface.



```typescript

interface SpawnedProcess {

&#x20; stdin: Writable;

&#x20; stdout: Readable;

&#x20; readonly killed: boolean;

&#x20; readonly exitCode: number | null;

&#x20; kill(signal: NodeJS.Signals): boolean;

&#x20; on(

&#x20;   event: "exit",

&#x20;   listener: (code: number | null, signal: NodeJS.Signals | null) => void

&#x20; ): void;

&#x20; on(event: "error", listener: (error: Error) => void): void;

&#x20; once(

&#x20;   event: "exit",

&#x20;   listener: (code: number | null, signal: NodeJS.Signals | null) => void

&#x20; ): void;

&#x20; once(event: "error", listener: (error: Error) => void): void;

&#x20; off(

&#x20;   event: "exit",

&#x20;   listener: (code: number | null, signal: NodeJS.Signals | null) => void

&#x20; ): void;

&#x20; off(event: "error", listener: (error: Error) => void): void;

}

```



\### `SpawnOptions`



Options passed to the custom spawn function.



```typescript

interface SpawnOptions {

&#x20; command: string;

&#x20; args: string\[];

&#x20; cwd?: string;

&#x20; env: Record<string, string | undefined>;

&#x20; signal: AbortSignal;

}

```



\### `McpSetServersResult`



Result of a `setMcpServers()` operation.



```typescript

type McpSetServersResult = {

&#x20; added: string\[];

&#x20; removed: string\[];

&#x20; errors: Record<string, string>;

};

```



\### `RewindFilesResult`



Result of a `rewindFiles()` operation.



```typescript

type RewindFilesResult = {

&#x20; canRewind: boolean;

&#x20; error?: string;

&#x20; filesChanged?: string\[];

&#x20; insertions?: number;

&#x20; deletions?: number;

};

```



\### `SDKStatusMessage`



Status update message (e.g., compacting).



```typescript

type SDKStatusMessage = {

&#x20; type: "system";

&#x20; subtype: "status";

&#x20; status: "compacting" | null;

&#x20; permissionMode?: PermissionMode;

&#x20; uuid: UUID;

&#x20; session\_id: string;

};

```



\### `SDKTaskNotificationMessage`



Notification when a background task completes, fails, or is stopped.



```typescript

type SDKTaskNotificationMessage = {

&#x20; type: "system";

&#x20; subtype: "task\_notification";

&#x20; task\_id: string;

&#x20; tool\_use\_id?: string;

&#x20; status: "completed" | "failed" | "stopped";

&#x20; output\_file: string;

&#x20; summary: string;

&#x20; usage?: {

&#x20;   total\_tokens: number;

&#x20;   tool\_uses: number;

&#x20;   duration\_ms: number;

&#x20; };

&#x20; uuid: UUID;

&#x20; session\_id: string;

};

```



\### `SDKToolUseSummaryMessage`



Summary of tool usage in a conversation.



```typescript

type SDKToolUseSummaryMessage = {

&#x20; type: "tool\_use\_summary";

&#x20; summary: string;

&#x20; preceding\_tool\_use\_ids: string\[];

&#x20; uuid: UUID;

&#x20; session\_id: string;

};

```



\### `SDKHookStartedMessage`



Emitted when a hook begins executing.



```typescript

type SDKHookStartedMessage = {

&#x20; type: "system";

&#x20; subtype: "hook\_started";

&#x20; hook\_id: string;

&#x20; hook\_name: string;

&#x20; hook\_event: string;

&#x20; uuid: UUID;

&#x20; session\_id: string;

};

```



\### `SDKHookProgressMessage`



Emitted while a hook is running, with stdout/stderr output.



```typescript

type SDKHookProgressMessage = {

&#x20; type: "system";

&#x20; subtype: "hook\_progress";

&#x20; hook\_id: string;

&#x20; hook\_name: string;

&#x20; hook\_event: string;

&#x20; stdout: string;

&#x20; stderr: string;

&#x20; output: string;

&#x20; uuid: UUID;

&#x20; session\_id: string;

};

```



\### `SDKHookResponseMessage`



Emitted when a hook finishes executing.



```typescript

type SDKHookResponseMessage = {

&#x20; type: "system";

&#x20; subtype: "hook\_response";

&#x20; hook\_id: string;

&#x20; hook\_name: string;

&#x20; hook\_event: string;

&#x20; output: string;

&#x20; stdout: string;

&#x20; stderr: string;

&#x20; exit\_code?: number;

&#x20; outcome: "success" | "error" | "cancelled";

&#x20; uuid: UUID;

&#x20; session\_id: string;

};

```



\### `SDKToolProgressMessage`



Emitted periodically while a tool is executing to indicate progress.



```typescript

type SDKToolProgressMessage = {

&#x20; type: "tool\_progress";

&#x20; tool\_use\_id: string;

&#x20; tool\_name: string;

&#x20; parent\_tool\_use\_id: string | null;

&#x20; elapsed\_time\_seconds: number;

&#x20; task\_id?: string;

&#x20; uuid: UUID;

&#x20; session\_id: string;

};

```



\### `SDKAuthStatusMessage`



Emitted during authentication flows.



```typescript

type SDKAuthStatusMessage = {

&#x20; type: "auth\_status";

&#x20; isAuthenticating: boolean;

&#x20; output: string\[];

&#x20; error?: string;

&#x20; uuid: UUID;

&#x20; session\_id: string;

};

```



\### `SDKTaskStartedMessage`



Emitted when a background task begins.



```typescript

type SDKTaskStartedMessage = {

&#x20; type: "system";

&#x20; subtype: "task\_started";

&#x20; task\_id: string;

&#x20; tool\_use\_id?: string;

&#x20; description: string;

&#x20; task\_type?: string;

&#x20; uuid: UUID;

&#x20; session\_id: string;

};

```



\### `SDKTaskProgressMessage`



Emitted periodically while a background task is running.



```typescript

type SDKTaskProgressMessage = {

&#x20; type: "system";

&#x20; subtype: "task\_progress";

&#x20; task\_id: string;

&#x20; tool\_use\_id?: string;

&#x20; description: string;

&#x20; usage: {

&#x20;   total\_tokens: number;

&#x20;   tool\_uses: number;

&#x20;   duration\_ms: number;

&#x20; };

&#x20; last\_tool\_name?: string;

&#x20; uuid: UUID;

&#x20; session\_id: string;

};

```



\### `SDKFilesPersistedEvent`



Emitted when file checkpoints are persisted to disk.



```typescript

type SDKFilesPersistedEvent = {

&#x20; type: "system";

&#x20; subtype: "files\_persisted";

&#x20; files: { filename: string; file\_id: string }\[];

&#x20; failed: { filename: string; error: string }\[];

&#x20; processed\_at: string;

&#x20; uuid: UUID;

&#x20; session\_id: string;

};

```



\### `SDKRateLimitEvent`



Emitted when the session encounters a rate limit.



```typescript

type SDKRateLimitEvent = {

&#x20; type: "rate\_limit\_event";

&#x20; rate\_limit\_info: {

&#x20;   status: "allowed" | "allowed\_warning" | "rejected";

&#x20;   resetsAt?: number;

&#x20;   utilization?: number;

&#x20; };

&#x20; uuid: UUID;

&#x20; session\_id: string;

};

```



\### `SDKLocalCommandOutputMessage`



Output from a local slash command (for example, `/voice` or `/cost`). Displayed as assistant-style text in the transcript.



```typescript

type SDKLocalCommandOutputMessage = {

&#x20; type: "system";

&#x20; subtype: "local\_command\_output";

&#x20; content: string;

&#x20; uuid: UUID;

&#x20; session\_id: string;

};

```



\### `SDKPromptSuggestionMessage`



Emitted after each turn when `promptSuggestions` is enabled. Contains a predicted next user prompt.



```typescript

type SDKPromptSuggestionMessage = {

&#x20; type: "prompt\_suggestion";

&#x20; suggestion: string;

&#x20; uuid: UUID;

&#x20; session\_id: string;

};

```



\### `AbortError`



Custom error class for abort operations.



```typescript

class AbortError extends Error {}

```



\## Sandbox Configuration



\### `SandboxSettings`



Configuration for sandbox behavior. Use this to enable command sandboxing and configure network restrictions programmatically.



```typescript

type SandboxSettings = {

&#x20; enabled?: boolean;

&#x20; autoAllowBashIfSandboxed?: boolean;

&#x20; excludedCommands?: string\[];

&#x20; allowUnsandboxedCommands?: boolean;

&#x20; network?: SandboxNetworkConfig;

&#x20; filesystem?: SandboxFilesystemConfig;

&#x20; ignoreViolations?: Record<string, string\[]>;

&#x20; enableWeakerNestedSandbox?: boolean;

&#x20; ripgrep?: { command: string; args?: string\[] };

};

```



| Property | Type | Default | Description |

| :------- | :--- | :------ | :---------- |

| `enabled` | `boolean` | `false` | Enable sandbox mode for command execution |

| `autoAllowBashIfSandboxed` | `boolean` | `true` | Auto-approve bash commands when sandbox is enabled |

| `excludedCommands` | `string\[]` | `\[]` | Commands that always bypass sandbox restrictions (e.g., `\['docker']`). These run unsandboxed automatically without model involvement |

| `allowUnsandboxedCommands` | `boolean` | `true` | Allow the model to request running commands outside the sandbox. When `true`, the model can set `dangerouslyDisableSandbox` in tool input, which falls back to the \[permissions system](#permissions-fallback-for-unsandboxed-commands) |

| `network` | \[`SandboxNetworkConfig`](#sandbox-network-config) | `undefined` | Network-specific sandbox configuration |

| `filesystem` | \[`SandboxFilesystemConfig`](#sandbox-filesystem-config) | `undefined` | Filesystem-specific sandbox configuration for read/write restrictions |

| `ignoreViolations` | `Record<string, string\[]>` | `undefined` | Map of violation categories to patterns to ignore (e.g., `{ file: \['/tmp/\*'], network: \['localhost'] }`) |

| `enableWeakerNestedSandbox` | `boolean` | `false` | Enable a weaker nested sandbox for compatibility |

| `ripgrep` | `{ command: string; args?: string\[] }` | `undefined` | Custom ripgrep binary configuration for sandbox environments |



\#### Example usage



```typescript

import { query } from "@anthropic-ai/claude-agent-sdk";



for await (const message of query({

&#x20; prompt: "Build and test my project",

&#x20; options: {

&#x20;   sandbox: {

&#x20;     enabled: true,

&#x20;     autoAllowBashIfSandboxed: true,

&#x20;     network: {

&#x20;       allowLocalBinding: true

&#x20;     }

&#x20;   }

&#x20; }

})) {

&#x20; if ("result" in message) console.log(message.result);

}

```



<Warning>

\*\*Unix socket security:\*\* The `allowUnixSockets` option can grant access to powerful system services. For example, allowing `/var/run/docker.sock` effectively grants full host system access through the Docker API, bypassing sandbox isolation. Only allow Unix sockets that are strictly necessary and understand the security implications of each.

</Warning>



\### `SandboxNetworkConfig`



Network-specific configuration for sandbox mode.



```typescript

type SandboxNetworkConfig = {

&#x20; allowedDomains?: string\[];

&#x20; allowManagedDomainsOnly?: boolean;

&#x20; allowLocalBinding?: boolean;

&#x20; allowUnixSockets?: string\[];

&#x20; allowAllUnixSockets?: boolean;

&#x20; httpProxyPort?: number;

&#x20; socksProxyPort?: number;

};

```



| Property | Type | Default | Description |

| :------- | :--- | :------ | :---------- |

| `allowedDomains` | `string\[]` | `\[]` | Domain names that sandboxed processes can access |

| `allowManagedDomainsOnly` | `boolean` | `false` | Restrict network access to only the domains in `allowedDomains` |

| `allowLocalBinding` | `boolean` | `false` | Allow processes to bind to local ports (e.g., for dev servers) |

| `allowUnixSockets` | `string\[]` | `\[]` | Unix socket paths that processes can access (e.g., Docker socket) |

| `allowAllUnixSockets` | `boolean` | `false` | Allow access to all Unix sockets |

| `httpProxyPort` | `number` | `undefined` | HTTP proxy port for network requests |

| `socksProxyPort` | `number` | `undefined` | SOCKS proxy port for network requests |



\### `SandboxFilesystemConfig`



Filesystem-specific configuration for sandbox mode.



```typescript

type SandboxFilesystemConfig = {

&#x20; allowWrite?: string\[];

&#x20; denyWrite?: string\[];

&#x20; denyRead?: string\[];

};

```



| Property | Type | Default | Description |

| :------- | :--- | :------ | :---------- |

| `allowWrite` | `string\[]` | `\[]` | File path patterns to allow write access to |

| `denyWrite` | `string\[]` | `\[]` | File path patterns to deny write access to |

| `denyRead` | `string\[]` | `\[]` | File path patterns to deny read access to |



\### Permissions Fallback for Unsandboxed Commands



When `allowUnsandboxedCommands` is enabled, the model can request to run commands outside the sandbox by setting `dangerouslyDisableSandbox: true` in the tool input. These requests fall back to the existing permissions system, meaning your `canUseTool` handler is invoked, allowing you to implement custom authorization logic.



<Note>

\*\*`excludedCommands` vs `allowUnsandboxedCommands`:\*\*

\- `excludedCommands`: A static list of commands that always bypass the sandbox automatically (e.g., `\['docker']`). The model has no control over this.

\- `allowUnsandboxedCommands`: Lets the model decide at runtime whether to request unsandboxed execution by setting `dangerouslyDisableSandbox: true` in the tool input.

</Note>



```typescript

import { query } from "@anthropic-ai/claude-agent-sdk";



for await (const message of query({

&#x20; prompt: "Deploy my application",

&#x20; options: {

&#x20;   sandbox: {

&#x20;     enabled: true,

&#x20;     allowUnsandboxedCommands: true // Model can request unsandboxed execution

&#x20;   },

&#x20;   permissionMode: "default",

&#x20;   canUseTool: async (tool, input) => {

&#x20;     // Check if the model is requesting to bypass the sandbox

&#x20;     if (tool === "Bash" \&\& input.dangerouslyDisableSandbox) {

&#x20;       // The model is requesting to run this command outside the sandbox

&#x20;       console.log(`Unsandboxed command requested: ${input.command}`);



&#x20;       if (isCommandAuthorized(input.command)) {

&#x20;         return { behavior: "allow" as const, updatedInput: input };

&#x20;       }

&#x20;       return {

&#x20;         behavior: "deny" as const,

&#x20;         message: "Command not authorized for unsandboxed execution"

&#x20;       };

&#x20;     }

&#x20;     return { behavior: "allow" as const, updatedInput: input };

&#x20;   }

&#x20; }

})) {

&#x20; if ("result" in message) console.log(message.result);

}

```



This pattern enables you to:



\- \*\*Audit model requests:\*\* Log when the model requests unsandboxed execution

\- \*\*Implement allowlists:\*\* Only permit specific commands to run unsandboxed

\- \*\*Add approval workflows:\*\* Require explicit authorization for privileged operations



<Warning>

Commands running with `dangerouslyDisableSandbox: true` have full system access. Ensure your `canUseTool` handler validates these requests carefully.



If `permissionMode` is set to `bypassPermissions` and `allowUnsandboxedCommands` is enabled, the model can autonomously execute commands outside the sandbox without any approval prompts. This combination effectively allows the model to escape sandbox isolation silently.

</Warning>



\## See also



\- \[SDK overview](/docs/en/agent-sdk/overview) - General SDK concepts

\- \[Python SDK reference](/docs/en/agent-sdk/python) - Python SDK documentation

\- \[CLI reference](https://code.claude.com/docs/en/cli-reference) - Command-line interface

\- \[Common workflows](https://code.claude.com/docs/en/common-workflows) - Step-by-step guides

