---
title: 組態系統與部署指南
summary: "OpenShell 插件的完整組態結構、驗證邏輯、部署情境範例與疑難排解"
read_when:
  - 你正在配置 OpenShell 插件
  - 你需要了解各組態選項的作用與預設值
  - 你正在將 OpenShell 部署到生產環境
---

# 組態系統與部署指南

## 組態結構

OpenShell 的組態分為兩層：

```mermaid
graph TB
    subgraph "OpenClaw Config"
        AGENTS["agents.defaults.sandbox<br/>(沙箱行為設定)"]
        PLUGINS["plugins.entries.openshell<br/>(插件設定)"]
    end

    subgraph "agents.defaults.sandbox"
        S_MODE["mode: 'all' | 'exec' | 'off'"]
        S_BACKEND["backend: 'openshell'"]
        S_SCOPE["scope: 'session' | 'agent'"]
        S_ACCESS["workspaceAccess: 'rw' | 'ro' | 'none'"]
    end

    subgraph "plugins.entries.openshell.config"
        P_MODE["mode: 'mirror' | 'remote'"]
        P_CMD["command: string"]
        P_FROM["from: string"]
        P_GW["gateway / gatewayEndpoint"]
        P_PROVIDERS["providers / gpu / autoProviders"]
        P_DIRS["remoteWorkspaceDir / remoteAgentWorkspaceDir"]
        P_TIMEOUT["timeoutSeconds"]
    end

    AGENTS --> S_MODE
    AGENTS --> S_BACKEND
    AGENTS --> S_SCOPE
    AGENTS --> S_ACCESS
    PLUGINS --> P_MODE
    PLUGINS --> P_CMD
    PLUGINS --> P_FROM
    PLUGINS --> P_GW
    PLUGINS --> P_PROVIDERS
    PLUGINS --> P_DIRS
    PLUGINS --> P_TIMEOUT
```

## 完整組態參考

### 型別定義

```typescript
// 原始組態（所有欄位皆為可選）
type OpenShellPluginConfig = {
  mode?: string;
  command?: string;
  gateway?: string;
  gatewayEndpoint?: string;
  from?: string;
  policy?: string;
  providers?: string[];
  gpu?: boolean;
  autoProviders?: boolean;
  remoteWorkspaceDir?: string;
  remoteAgentWorkspaceDir?: string;
  timeoutSeconds?: number;
};

// 解析後組態（所有必要欄位皆有值）
type ResolvedOpenShellPluginConfig = {
  mode: "mirror" | "remote";    // 已驗證的工作模式
  command: string;               // CLI 指令或路徑
  gateway?: string;              // 可選 gateway 名稱
  gatewayEndpoint?: string;      // 可選 gateway URL
  from: string;                  // 沙箱來源
  policy?: string;               // 可選政策檔案
  providers: string[];           // 已去重的 provider 列表
  gpu: boolean;                  // GPU 請求
  autoProviders: boolean;        // 自動建立 providers
  remoteWorkspaceDir: string;    // 遠端工作區路徑（已正規化）
  remoteAgentWorkspaceDir: string; // 遠端 agent 路徑（已正規化）
  timeoutMs: number;             // 逾時（毫秒，由 seconds 轉換）
};
```

### 欄位詳細說明

| 欄位 | 型別 | 預設值 | 說明 |
|------|------|--------|------|
| `mode` | `"mirror"` \| `"remote"` | `"mirror"` | 工作區同步模式。Mirror 為雙向同步，Remote 為單次種子後直接遠端操作 |
| `command` | `string` | `"openshell"` | `openshell` CLI 的路徑或指令名稱。設為 `"openshell"` 時觸發 bundled fallback |
| `gateway` | `string` | -- | OpenShell gateway 名稱，對應 `--gateway` 旗標 |
| `gatewayEndpoint` | `string` | -- | OpenShell gateway endpoint URL，對應 `--gateway-endpoint` 旗標 |
| `from` | `string` | `"openclaw"` | 沙箱映像來源，用於 `openshell sandbox create --from` |
| `policy` | `string` | -- | 自訂沙箱政策 YAML 檔案路徑 |
| `providers` | `string[]` | `[]` | 建立沙箱時附加的 provider 名稱。重複項會自動去除 |
| `gpu` | `boolean` | `false` | 建立沙箱時請求 GPU 資源 |
| `autoProviders` | `boolean` | `true` | 是否傳遞 `--auto-providers`（否則傳 `--no-auto-providers`） |
| `remoteWorkspaceDir` | `string` | `"/sandbox"` | 沙箱內的主要可寫工作區路徑。必須為絕對路徑 |
| `remoteAgentWorkspaceDir` | `string` | `"/agent"` | Agent 工作區掛載路徑（用於 read-only 存取模式） |
| `timeoutSeconds` | `number` | `120` | CLI 操作逾時（秒）。最小值 1。內部轉換為毫秒 |

## 驗證邏輯

`resolveOpenShellPluginConfig()` 執行以下驗證：

```mermaid
flowchart TD
    INPUT["輸入: unknown value"] --> A{是 object？}
    A -->|否| ERR1["錯誤: expected config object"]
    A -->|是| B["檢查所有 key 是否在允許列表中"]
    B --> C{有未知 key？}
    C -->|是| ERR2["錯誤: unknown config key: xxx"]
    C -->|否| D["normalizeProviders()"]
    D --> E{providers 合法？}
    E -->|否| ERR3["錯誤: providers must be array of strings"]
    E -->|是| F["驗證 timeoutSeconds"]
    F --> G{>= 1 的有限數字？}
    G -->|否| ERR4["錯誤: timeoutSeconds must be >= 1"]
    G -->|是| H["驗證 gpu, autoProviders"]
    H --> I{都是 boolean？}
    I -->|否| ERR5["錯誤: xxx must be a boolean"]
    I -->|是| J["safeParse 成功"]
    J --> K["resolveOpenShellPluginConfig()"]
    K --> L["驗證 mode"]
    L --> M{mirror 或 remote？}
    M -->|否| ERR6["錯誤: mode must be mirror or remote"]
    M -->|是| N["normalizeRemotePath()"]
    N --> O{絕對路徑？}
    O -->|否| ERR7["錯誤: path must be absolute"]
    O -->|是| P["套用預設值，回傳 ResolvedConfig"]

    style ERR1 fill:#D94A4A,color:#fff
    style ERR2 fill:#D94A4A,color:#fff
    style ERR3 fill:#D94A4A,color:#fff
    style ERR4 fill:#D94A4A,color:#fff
    style ERR5 fill:#D94A4A,color:#fff
    style ERR6 fill:#D94A4A,color:#fff
    style ERR7 fill:#D94A4A,color:#fff
    style P fill:#4AD97A,color:#000
```

### Provider 正規化

```typescript
function normalizeProviders(value: unknown): string[] | null {
  if (value === undefined) return [];     // 未提供 → 空陣列
  if (!Array.isArray(value)) return null; // 非陣列 → 錯誤

  const seen = new Set<string>();
  const providers: string[] = [];
  for (const entry of value) {
    if (typeof entry !== "string" || !entry.trim()) return null;
    const normalized = entry.trim();
    if (seen.has(normalized)) continue;  // 自動去重
    seen.add(normalized);
    providers.push(normalized);
  }
  return providers;
}
```

### 路徑正規化

```typescript
function normalizeRemotePath(value: string | undefined, fallback: string): string {
  const candidate = value ?? fallback;
  // 使用 POSIX 格式正規化
  const normalized = path.posix.normalize(candidate.trim() || fallback);
  if (!normalized.startsWith("/")) {
    throw new Error(`OpenShell remote path must be absolute: ${candidate}`);
  }
  return normalized;
}
```

## 部署情境範例

### 情境一：開發環境（Mirror 模式）

適合需要在本地 IDE 編輯並即時看到沙箱變更的開發者：

```json5
{
  agents: {
    defaults: {
      sandbox: {
        mode: "all",
        backend: "openshell",
        scope: "session",       // 每個 session 一個沙箱
        workspaceAccess: "rw",  // 可讀寫
      },
    },
  },
  plugins: {
    entries: {
      openshell: {
        enabled: true,
        config: {
          from: "openclaw",
          mode: "mirror",        // 雙向同步
          timeoutSeconds: 180,   // 較長逾時以應對大工作區
        },
      },
    },
  },
}
```

### 情境二：CI/CD（Remote 模式）

適合自動化流程，不需本地同步：

```json5
{
  agents: {
    defaults: {
      sandbox: {
        mode: "all",
        backend: "openshell",
        scope: "agent",          // 每個 agent 一個沙箱
        workspaceAccess: "rw",
      },
    },
  },
  plugins: {
    entries: {
      openshell: {
        enabled: true,
        config: {
          from: "openclaw",
          mode: "remote",        // 單次種子
          autoProviders: true,
          timeoutSeconds: 300,   // CI 可能需要更長逾時
        },
      },
    },
  },
}
```

### 情境三：GPU 工作負載

需要 GPU 的機器學習或推理任務：

```json5
{
  agents: {
    defaults: {
      sandbox: {
        mode: "all",
        backend: "openshell",
        scope: "agent",
        workspaceAccess: "rw",
      },
    },
  },
  plugins: {
    entries: {
      openshell: {
        enabled: true,
        config: {
          from: "openclaw",
          mode: "remote",
          gpu: true,             // 請求 GPU
          providers: ["openai"], // 附加 OpenAI provider
          timeoutSeconds: 600,   // ML 任務可能較久
        },
      },
    },
  },
}
```

### 情境四：自訂 Gateway + 嚴格政策

企業環境中使用私有 gateway 與安全政策：

```json5
{
  agents: {
    defaults: {
      sandbox: { mode: "off" },      // 預設關閉
    },
    list: [
      {
        id: "secure-researcher",
        sandbox: {
          mode: "all",
          backend: "openshell",
          scope: "agent",
          workspaceAccess: "rw",
        },
      },
    ],
  },
  plugins: {
    entries: {
      openshell: {
        enabled: true,
        config: {
          from: "enterprise-base",
          mode: "remote",
          gateway: "internal-lab",
          gatewayEndpoint: "https://openshell.internal.company.com",
          policy: "/etc/openclaw/strict-policy.yaml",
          autoProviders: false,      // 不自動建立
          providers: ["approved-llm"], // 僅允許核准的 provider
        },
      },
    },
  },
}
```

### 情境五：Per-Agent 覆寫

不同 Agent 使用不同的沙箱設定：

```json5
{
  agents: {
    defaults: {
      sandbox: {
        mode: "all",
        backend: "openshell",
        scope: "agent",
      },
    },
    list: [
      {
        id: "coder",
        // 使用全域預設的 openshell 設定
      },
      {
        id: "researcher",
        sandbox: {
          mode: "all",
          backend: "openshell",
          scope: "session",        // 覆寫 scope
          workspaceAccess: "ro",   // 唯讀
        },
      },
    ],
  },
  plugins: {
    entries: {
      openshell: {
        enabled: true,
        config: {
          from: "openclaw",
          mode: "remote",
        },
      },
    },
  },
}
```

## 組態覆寫機制

Manager 的 `describeRuntime()` 支援從 OpenClaw config 動態覆寫插件組態：

```typescript
function resolveOpenShellPluginConfigFromConfig(
  config: OpenClawConfig,                      // 當前 OpenClaw 組態
  fallback: ResolvedOpenShellPluginConfig,      // 插件初始化時的組態
): ResolvedOpenShellPluginConfig {
  const pluginConfig = config.plugins?.entries?.openshell?.config;
  if (!pluginConfig) return fallback;           // 無覆寫 → 使用預設
  return resolveOpenShellPluginConfig(pluginConfig); // 重新解析覆寫值
}
```

## 何時需要 recreate

修改以下設定後，必須重新建立沙箱：

| 變更的設定 | 為什麼需要 recreate |
|-----------|-------------------|
| `backend` | 切換後端類型 |
| `from` | 沙箱映像已不同 |
| `mode` | 同步策略根本改變 |
| `policy` | 安全政策需要重新套用 |

```bash
openclaw sandbox recreate --all
```

## 疑難排解

### 沙箱建立失敗

```
Error: openshell sandbox create failed
```

1. 確認 `openshell` CLI 可正常執行
2. 檢查帳號權限：`openshell auth status`
3. 確認 gateway 可達（如有設定）
4. 增加 `timeoutSeconds`（預設 120 秒可能不夠）

### 上傳/下載超時

```
Error: openshell sandbox upload failed
```

1. 檢查工作區大小，考慮切換至 Remote 模式
2. 增加 `timeoutSeconds`
3. 確認網路連線穩定

### 路徑錯誤

```
Error: OpenShell remote path must be absolute: relative/path
```

`remoteWorkspaceDir` 和 `remoteAgentWorkspaceDir` 必須為絕對路徑（以 `/` 開頭）。

### Provider 設定錯誤

```
Error: providers must be an array of strings
```

確認 `providers` 是字串陣列格式：
```json5
// 正確
providers: ["openai", "anthropic"]
// 錯誤
providers: "openai"
providers: [123]
```
