---
title: 檔案系統橋接器深入分析
summary: "Mirror 與 Remote 兩種 FS Bridge 的實作細節、路徑解析、安全機制與效能取捨"
read_when:
  - 你想了解沙箱中檔案操作的完整機制
  - 你遇到路徑解析或權限錯誤
  - 你需要理解 Mirror 與 Remote 模式的 I/O 差異
---

# 檔案系統橋接器深入分析

## 概述

OpenShell 插件根據工作模式提供兩種截然不同的檔案系統橋接器（FS Bridge）：

| | Mirror FS Bridge | Remote FS Bridge |
|---|---|---|
| **檔案** | `fs-bridge.ts` | `remote-fs-bridge.ts` |
| **實作** | 自行實作 `SandboxFsBridge` | 委派至 Plugin SDK |
| **資料位置** | 本地 + 遠端雙寫 | 僅遠端 |
| **hostPath** | 有（本地路徑） | 無（`undefined`） |
| **原子性** | 本地 temp+rename | 遠端 Python3 script |
| **行數** | ~356 行 | ~16 行 |

```mermaid
graph TB
    subgraph "backend.ts → createFsBridge()"
        DECISION{config.mode?}
    end

    DECISION -->|"mirror"| MIRROR["createOpenShellFsBridge()<br/>fs-bridge.ts"]
    DECISION -->|"remote"| REMOTE["createOpenShellRemoteFsBridge()<br/>remote-fs-bridge.ts"]

    MIRROR --> LOCAL["本地 fs 操作"]
    MIRROR --> SYNC["syncLocalPathToRemote()"]
    LOCAL --> SYNC

    REMOTE --> SDK["createRemoteShellSandboxFsBridge()<br/>(Plugin SDK)"]
    SDK --> SSH["SSH shell scripts"]

    style MIRROR fill:#4AD97A,color:#000
    style REMOTE fill:#4A90D9,color:#fff
```

## Mirror FS Bridge

### 雙掛載點架構

Mirror Bridge 支援兩個掛載點，對應到遠端沙箱中的兩個目錄：

```
本地主機                           遠端沙箱

workspaceDir ─────────────── /sandbox (containerWorkdir)
  (可讀寫)                      主要工作區

agentWorkspaceDir ────────── /agent (remoteAgentWorkspaceDir)
  (可讀寫)                      Agent 專用工作區
```

雙掛載啟用的條件：
1. `workspaceAccess !== "none"`
2. `workspaceDir !== agentWorkspaceDir`（兩者不是同一目錄）

### 路徑解析流程

`resolveTarget()` 方法將使用者提供的路徑解析為 `ResolvedMountPath`：

```mermaid
flowchart TD
    INPUT["輸入: filePath"] --> A{以 /sandbox 開頭？}
    A -->|是| B["解析為 workspace mount<br/>hostPath = workspaceDir + relative"]
    A -->|否| C{有 agent mount 且<br/>以 /agent 開頭？}
    C -->|是| D["解析為 agent mount<br/>hostPath = agentDir + relative"]
    C -->|否| E["以 cwd 或 workspaceDir<br/>為基準解析絕對路徑"]
    E --> F{hostPath 在 workspaceDir 內？}
    F -->|是| G["歸類為 workspace mount"]
    F -->|否| H{hostPath 在 agentDir 內？}
    H -->|是| I["歸類為 agent mount"]
    H -->|否| J["拋出錯誤:<br/>Path escapes sandbox root"]

    style J fill:#D94A4A,color:#fff
```

### ResolvedMountPath 型別

```typescript
type ResolvedMountPath = SandboxResolvedPath & {
  mountHostRoot: string;      // 掛載根目錄（本地）
  writable: boolean;          // workspaceAccess === "rw"
  source: "workspace" | "agent";  // 來源掛載點
};

// SandboxResolvedPath:
// {
//   hostPath?: string;       // 本地檔案路徑
//   relativePath: string;    // 相對路徑
//   containerPath: string;   // 容器內路徑（/sandbox/...）
// }
```

### 範例：路徑解析

```typescript
// 場景：workspaceDir=/home/user/project, containerWorkdir=/sandbox

// Case 1: 容器路徑
resolvePath({ filePath: "/sandbox/src/main.ts" })
// → hostPath: "/home/user/project/src/main.ts"
// → containerPath: "/sandbox/src/main.ts"
// → source: "workspace"

// Case 2: 相對路徑
resolvePath({ filePath: "src/main.ts" })
// → hostPath: "/home/user/project/src/main.ts"
// → containerPath: "/sandbox/src/main.ts"
// → source: "workspace"

// Case 3: Agent 掛載
resolvePath({ filePath: "/agent/data/model.bin" })
// → hostPath: "/home/user/agent-workspace/data/model.bin"
// → containerPath: "/agent/data/model.bin"
// → source: "agent"

// Case 4: 越界 → 錯誤
resolvePath({ filePath: "/etc/passwd" })
// → Error: Path escapes sandbox root
```

### 檔案操作與安全機制

#### writeFile -- 原子寫入 + 遠端同步

```mermaid
sequenceDiagram
    participant Caller
    participant Bridge as Mirror FS Bridge
    participant FS as 本地檔案系統
    participant Backend as Backend.syncLocalPathToRemote

    Caller->>Bridge: writeFile({filePath, data, mkdir})
    Bridge->>Bridge: resolveTarget() → hostPath
    Bridge->>Bridge: ensureWritable() 檢查權限
    Bridge->>Bridge: assertLocalPathSafety() 安全驗證

    alt mkdir !== false
        Bridge->>FS: mkdir(parentDir, {recursive: true})
    end

    Bridge->>FS: writeFile(tempPath, data)
    Note over FS: 暫存檔名：.openclaw-openshell-write-{basename}-{pid}-{timestamp}
    Bridge->>FS: rename(tempPath, hostPath)
    Note over FS: 原子性 rename 確保寫入完整性
    Bridge->>Backend: syncLocalPathToRemote(hostPath, containerPath)
    Note over Backend: 透過 openshell sandbox upload 上傳
```

為什麼使用 temp+rename 策略？
- **原子性**：如果寫入過程中斷，原檔案不受影響
- **一致性**：讀取端不會看到部分寫入的內容
- **跨平台**：POSIX `rename()` 在同一檔案系統上是原子操作

#### readFile -- 安全讀取

```typescript
async readFile(params): Promise<Buffer> {
  const target = this.resolveTarget(params);
  const hostPath = this.requireHostPath(target);
  // 安全驗證：防止 symlink 越界
  await assertLocalPathSafety({
    target,
    root: target.mountHostRoot,
    allowMissingLeaf: false,
    allowFinalSymlinkForUnlink: false,
  });
  return await fsPromises.readFile(hostPath);
}
```

#### remove -- 本地刪除 + 遠端清理

```typescript
// 遠端刪除策略根據是否遞迴而不同：
const remoteScript = params.recursive
  ? 'rm -rf -- "$1"'                    // 遞迴：直接 rm -rf
  : 'if [ -d "$1" ] && [ ! -L "$1" ]; then rmdir -- "$1"; ' +
    'elif [ -e "$1" ] || [ -L "$1" ]; then rm -f -- "$1"; fi';
    // 非遞迴：目錄用 rmdir（必須為空），檔案用 rm -f
```

#### rename -- 跨檔案系統支援

```typescript
// 本地 rename 使用 movePathWithCopyFallback：
async function movePathWithCopyFallback(params) {
  try {
    await fs.rename(params.from, params.to);  // 嘗試原子 rename
  } catch (error) {
    if (error.code !== "EXDEV") throw error;  // 非跨裝置錯誤 → 重新拋出
    // EXDEV = 跨檔案系統 → 用 copy + delete 替代
    await fs.cp(params.from, params.to, {
      recursive: true, force: true, dereference: false
    });
    await fs.rm(params.from, { recursive: true, force: true });
  }
}
```

### 安全機制：assertLocalPathSafety

此函式防止透過 symlink 進行路徑逃逸攻擊：

```mermaid
flowchart TD
    A["assertLocalPathSafety(target, root)"] --> B["解析 canonicalRoot<br/>= realpath(root)"]
    B --> C["解析 candidate<br/>= resolveCanonicalCandidate(hostPath)"]
    C --> D{candidate 在 canonicalRoot 內？}
    D -->|否| E["拋出錯誤:<br/>Sandbox path escapes allowed mounts"]
    D -->|是| F["逐段檢查路徑"]
    F --> G{當前段是 symlink？}
    G -->|是| H{是最後一段？}
    H -->|是且允許| I["通過（unlink 情境）"]
    H -->|否或不允許| J["拋出錯誤:<br/>Sandbox boundary checks failed"]
    G -->|否| K{段不存在？}
    K -->|是且為最後一段且允許| L["通過（建立新檔情境）"]
    K -->|否| F

    style E fill:#D94A4A,color:#fff
    style J fill:#D94A4A,color:#fff
```

#### 攻擊情境範例

```
攻擊：在沙箱內建立 symlink 指向外部
/sandbox/evil → /etc/passwd

防禦流程：
1. resolveTarget("/sandbox/evil") → hostPath="/home/user/project/evil"
2. assertLocalPathSafety 開始逐段檢查
3. 發現 "evil" 是 symlink
4. allowFinalSymlinkForUnlink=false → 拋出錯誤
```

#### resolveCanonicalCandidate

處理部分路徑不存在的情況（如寫入新檔案）：

```typescript
// 從路徑末端向上走，找到第一個存在的祖先
// 對該祖先做 realpath()，再接回剩餘的路徑段
async function resolveCanonicalCandidate(targetPath: string): Promise<string> {
  const missing: string[] = [];
  let cursor = path.resolve(targetPath);
  while (true) {
    const exists = await fs.lstat(cursor).then(() => true).catch(() => false);
    if (exists) {
      const canonical = await fs.realpath(cursor).catch(() => cursor);
      return path.resolve(canonical, ...missing);
    }
    const parent = path.dirname(cursor);
    if (parent === cursor) break;
    missing.unshift(path.basename(cursor));
    cursor = parent;
  }
}
```

## Remote FS Bridge

### 架構

Remote Bridge 本身只有 16 行，是一個薄包裝：

```typescript
export function createOpenShellRemoteFsBridge(params: {
  sandbox: SandboxContext;
  backend: RemoteShellSandboxHandle;
}): SandboxFsBridge {
  return createRemoteShellSandboxFsBridge({
    sandbox: params.sandbox,
    runtime: params.backend,
  });
}
```

所有實際邏輯委派給 Plugin SDK 的 `createRemoteShellSandboxFsBridge()`。

### 運作方式

Remote Bridge 透過 `runRemoteShellScript()` 在遠端執行 shell script 來操作檔案：

```mermaid
graph LR
    subgraph "Remote FS Bridge (Plugin SDK)"
        READ["readFile()"]
        WRITE["writeFile()"]
        STAT["stat()"]
        REMOVE["remove()"]
        RENAME["rename()"]
    end

    READ --> SH1["set -eu\ncat -- '$1'"]
    WRITE --> PY["python3 /dev/fd/3 write ..."]
    STAT --> SH3["stat -c '%F|%s|%Y' -- '$1'"]
    REMOVE --> PY
    RENAME --> PY

    SH1 --> SSH["SSH 執行"]
    PY --> SSH
    SH3 --> SSH
```

### 遠端 Shell Script 對照表

| 操作 | Shell Script | 說明 |
|------|-------------|------|
| **讀取檔案** | `set -eu; cat -- "$1"` | 直接 cat 輸出至 stdout |
| **檔案存在** | `if [ -e "$1" ] \|\| [ -L "$1" ]; then printf "1\n"; else printf "0\n"; fi` | 含 symlink 檢查 |
| **canonicalize** | `canonical=$(readlink -f -- "$cursor")` | 解析真實路徑 |
| **型別+硬連結** | `stat -c "%F\|%h" -- "$1"` | 判斷檔案類型 |
| **完整 stat** | `stat -c "%F\|%s\|%Y" -- "$1"` | 類型、大小、修改時間 |
| **寫入/刪除/重命名** | `python3 /dev/fd/3 "$@" 3<<'PY'` | Python3 原子操作腳本 |

### Python3 原子操作腳本

Remote 模式使用 Python3 進行需要原子性的操作：

```python
# 寫入操作 (概念)
# args: write <root> <relativeParent> <basename> <mkdir>
# stdin: 檔案內容

import os, sys
operation = sys.argv[1]
if operation == "write":
    root, parent, basename, mkdir = sys.argv[2:6]
    full_parent = os.path.join(root, parent)
    if mkdir == "1":
        os.makedirs(full_parent, exist_ok=True)
    with open(os.path.join(full_parent, basename), 'wb') as f:
        f.write(sys.stdin.buffer.read())
```

支援的操作：
- `write` -- 寫入檔案（可自動建立父目錄）
- `mkdirp` -- 遞迴建立目錄
- `remove` -- 刪除檔案/目錄（支援 recursive/force 旗標）
- `rename` -- 重命名/搬移（可自動建立目標父目錄）

## 效能比較

```mermaid
graph TB
    subgraph "Mirror 模式 writeFile"
        M1["resolveTarget()"] --> M2["assertLocalPathSafety()"]
        M2 --> M3["寫入暫存檔 (本地)"]
        M3 --> M4["rename (本地)"]
        M4 --> M5["syncLocalPathToRemote()"]
        M5 --> M6["openshell sandbox upload"]
    end

    subgraph "Remote 模式 writeFile"
        R1["resolveTarget()"] --> R2["建立 SSH session"]
        R2 --> R3["python3 mutation script"]
        R3 --> R4["stdin 傳送檔案內容"]
    end
```

| 指標 | Mirror | Remote |
|------|--------|--------|
| **readFile 延遲** | 低（本地 fs） | 高（SSH 往返） |
| **writeFile 延遲** | 中（本地 + 上傳） | 中（SSH + script） |
| **exec 前開銷** | 高（全量上傳） | 低（僅首次種子） |
| **exec 後開銷** | 高（全量下載） | 無 |
| **大檔案讀取** | 快（本地 I/O） | 慢（SSH 傳輸） |
| **頻繁小寫入** | 尚可（每次觸發 upload） | 較好（直接遠端操作） |

### 選擇建議

```
你的工作流偏向？

├── 頻繁在本地編輯、需要雙向同步
│   → Mirror 模式
│
├── 長時間運行的 Agent、CI/CD
│   → Remote 模式
│
├── 大型工作區（>1GB）
│   → Remote 模式（避免每次 exec 的全量同步）
│
└── 需要本地 IDE 同步
    → Mirror 模式
```
