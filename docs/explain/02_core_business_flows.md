# 02. 核心业务流程 (Core Business Flows)

本文档通过 Mermaid 图表详细展示了 PromptMinder 系统的核心业务路径，包括 AI 提示词生成、团队协作以及社区贡献流程。旨在帮助产品经理理解系统背后的流转逻辑与关键节点。

## 1. AI 智能优化提示词流程

用户输入原始想法，通过调用第三方 LLM（智谱 AI）进行结构化重写，并实时流式返回给用户。

### 1.1 业务流程图 (Flowchart)

```mermaid
graph TD
    User[用户]
    Input[输入原始需求]
    System[系统后端]
    ZhipuAI{智谱 AI 服务}
    Stream[流式响应]
    Display[前端展示]
    Save{保存提示词}
    DB[数据库]

    User --> Input
    Input --> System
    System -->|调用 Generate API| ZhipuAI
    ZhipuAI -->|返回流式数据| Stream
    Stream --> Display
    Display -->|满意| Save
    Save -->|写入 Prompts 表| DB
    Display -->|不满意| Input
```

### 1.2 时序交互图 (Sequence Diagram)

```mermaid
sequenceDiagram
    participant U as 用户 - User
    participant FE as 前端界面 - Frontend
    participant API as 后端 API - /api/generate
    participant AI as 智谱 AI - External LLM
    participant DB as 数据库 - Supabase

    Note over U, AI: AI 优化提示词场景

    U->>FE: 输入原始提示词想法
    FE->>API: POST /api/generate {text: "..."}
    API->>AI: 调用 Chat Completion [System Prompt + User Input]
    activate AI
    AI-->>API: Stream Response - SSE
    API-->>FE: Stream Chunks - 结构化数据
    deactivate AI

    FE->>U: 实时展示优化后的结构化 Prompt

    opt 用户保存
        U->>FE: 点击保存
        FE->>API: POST /api/prompts
        API->>DB: INSERT into prompts (title, content, version...)
        DB-->>API: Success
        API-->>FE: Success
        FE-->>U: 提示保存成功
    end
```

---

## 2. 团队协作与权限流转

展示用户如何创建团队、邀请成员以及在团队内共享提示词。

### 2.1 团队创建与邀请流程

```mermaid
graph TD
    Start[开始]
    CreateTeam[创建团队]
    InputInfo[填写团队名称/描述]
    DB_Team[插入 Teams 表]
    InviteUser[邀请成员]
    InputEmail[输入成员邮箱]
    CheckUser{用户是否存在?}
    CreatePending[创建待加入记录 - Status: Pending]
    SendEmail[发送邀请邮件]
    UserAction{受邀者操作}
    Accept[接受邀请]
    Reject[拒绝邀请]
    UpdateStatus[更新 Member Status: Active]
    End[流程结束]

    Start --> CreateTeam
    CreateTeam --> InputInfo
    InputInfo --> DB_Team
    DB_Team --> InviteUser
    InviteUser --> InputEmail
    InputEmail --> CheckUser

    CheckUser -- 是 --> CreatePending
    CheckUser -- 否 --> End

    CreatePending --> SendEmail
    SendEmail --> UserAction

    UserAction -- 同意 --> Accept
    Accept --> UpdateStatus
    UpdateStatus --> End

    UserAction -- 拒绝 --> Reject
    Reject --> End
```

---

## 3. 社区贡献与审核闭环

展示用户将私有提示词贡献给公共社区的全过程，包含管理员审核环节。

### 3.1 贡献审核状态流转

```mermaid
stateDiagram-v2
    [*] --> Draft: 用户编写
    Draft --> Pending: 提交审核 (Submit)

    state Pending {
        [*] --> Reviewing: 管理员介入
        Reviewing --> Approved: 审核通过
        Reviewing --> Rejected: 审核驳回
    }

    Approved --> Published: 系统发布 (写入公共库)
    Rejected --> Draft: 用户修改 (可重新提交)
    Published --> [*]
```

### 3.2 详细审核交互图

```mermaid
sequenceDiagram
    participant Contributor as 贡献者
    participant System as 系统后台
    participant Admin as 平台管理员
    participant PublicDB as 公共提示词库

    Contributor->>System: 提交提示词 (title, content, category)
    System->>System: 创建 Contribution 记录 (Status: Pending)

    loop 定期审核
        Admin->>System: 获取待审核列表 (pending_contributions)
        System-->>Admin: 返回列表

        alt 审核通过
            Admin->>System: 批准 (Approve)
            System->>PublicDB: 复制数据到 Prompts 表 (is_public=true)
            System->>System: 更新 Contribution 状态 (Approved)
            System-->>Contributor: 通知审核通过
        else 审核驳回
            Admin->>System: 拒绝 (Reject + Reason)
            System->>System: 更新 Contribution 状态 (Rejected)
            System-->>Contributor: 通知驳回原因
        end
    end
```
