# ZtoApi 账号管理系统使用指南

## 功能概述

ZtoApi 现在包含一个完整的账号管理系统，支持：
- **登录鉴权**：基于 Session 的管理员认证
- **账号导入**：批量导入 TXT 格式的 Z.ai 账号
- **账号导出**：导出所有账号为 TXT 格式
- **账号查询**：支持搜索和列表展示
- **数据持久化**：使用 Deno KV 本地存储

## 环境变量配置

在 `.env.local` 或环境变量中配置以下参数：

```bash
# 管理员账号配置（可选，默认值如下）
ADMIN_USERNAME=admin          # 管理员用户名
ADMIN_PASSWORD=123456         # 管理员密码
ADMIN_ENABLED=true            # 是否启用管理面板（false 则禁用）
```

## 启动服务

```bash
# 进入目录
cd deno/zai

# 启动开发模式（带自动重载）
deno task dev

# 生产模式
deno task start
```

服务启动后会显示：
```
🔐 Admin Panel: http://localhost:9090/admin (Username: admin)
```

## 使用流程

### 1. 登录管理面板

1. 访问 `http://localhost:9090/admin/login`
2. 输入用户名和密码（默认 `admin` / `123456`）
3. 登录成功后跳转到账号管理页面

### 2. 导入账号

**TXT 文件格式要求**：
```
email----password----token
email----password----token----extrapart
```

示例：
```
test1@example.com----password123----eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.testtoken1
test2@example.com----password456----eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.testtoken2
test3@example.com----password789----eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.testtoken3----extrapart
```

**导入步骤**：
1. 在管理面板点击「导入 TXT」按钮
2. 选择账号文件（格式如上）
3. 系统自动去重并导入
4. 显示导入成功数量和跳过重复数量

### 3. 导出账号

1. 点击「导出 TXT」按钮
2. 自动下载文件 `zai_accounts_[时间戳].txt`
3. 文件格式与导入格式一致

### 4. 分页功能

**分页控件（参考 Element UI 设计）**：
- **每页显示条数**：支持 10/20/50/100 条/页切换
- **页码按钮**：
  - 首页/上一页/下一页/尾页快速导航
  - 智能页码显示（超过7页时显示省略号）
  - 点击省略号快速跳转5页
- **跳转功能**：输入页码直接跳转（支持回车键）

**分页特性**：
- 搜索后自动重置到第一页
- 切换每页条数后重置到第一页
- 序号全局连续（跨页保持连续编号）
- 边界保护（防止越界）

### 5. 搜索账号

在搜索框输入邮箱关键词，实时过滤显示结果，自动重置到第一页。

### 6. 复制账号信息

- **复制账号**：复制格式为 `email----password`
- **复制 Token**：复制完整的 token 字符串

## API 接口说明

### 登录 API

```bash
POST /admin/api/login
Content-Type: application/json

{
  "username": "admin",
  "password": "123456"
}
```

响应：
```json
{
  "success": true,
  "sessionId": "uuid"
}
```

### 账号列表 API

```bash
GET /admin/api/accounts?search=keyword
Cookie: adminSessionId=session_id
```

响应：
```json
[
  {
    "email": "test@example.com",
    "password": "password123",
    "token": "eyJhbGci...",
    "createdAt": "2025-01-01T00:00:00.000Z"
  }
]
```

### 批量导入 API

```bash
POST /admin/api/import-batch
Cookie: adminSessionId=session_id
Content-Type: application/json

{
  "accounts": [
    {
      "email": "test@example.com",
      "password": "password123",
      "token": "token_string"
    }
  ]
}
```

响应：
```json
{
  "success": true,
  "imported": 10,
  "skipped": 2
}
```

### 导出 API

```bash
GET /admin/api/export
Cookie: adminSessionId=session_id
```

返回 TXT 文件（格式：`email----password----token`）

### 登出 API

```bash
POST /admin/api/logout
Cookie: adminSessionId=session_id
```

响应：
```json
{
  "success": true
}
```

## 数据存储

- **存储方式**：Deno KV（本地数据库）
- **数据路径**：默认存储在 Deno 的 KV 目录
- **Session 过期**：24 小时自动过期
- **账号数据**：永久存储，支持完整 CRUD

## 安全说明

1. **生产环境必须修改默认密码**：
   ```bash
   export ADMIN_USERNAME=your_username
   export ADMIN_PASSWORD=your_strong_password
   ```

2. **禁用管理面板**（如不需要）：
   ```bash
   export ADMIN_ENABLED=false
   ```

3. **Session 管理**：
   - 使用 Cookie 存储 Session ID
   - Session 数据存储在 KV 中，24 小时过期
   - 登出后 Session 立即失效

4. **访问控制**：
   - 所有 `/admin/api/*` 接口（除登录外）需要鉴权
   - 未登录访问会返回 401 或重定向到登录页

## 与 zai_register.ts 的联动

如果你同时使用 `zai_register.ts` 批量注册账号：

1. 在 `zai_register.ts` 中注册账号
2. 导出账号为 TXT 文件
3. 在 `main.ts` 管理面板导入 TXT 文件
4. 账号自动存储到 KV，供 Token Pool 使用

## 故障排查

### 问题1：端口已被占用
```bash
# 查看占用端口的进程
lsof -ti:9090

# 杀掉进程
lsof -ti:9090 | xargs kill -9
```

### 问题2：Deno KV 初始化失败
确保使用了 `--unstable-kv` 标志：
```bash
deno run --allow-net --allow-env --allow-read --unstable-kv main.ts
```

### 问题3：导入失败
检查 TXT 文件格式：
- 每行一个账号
- 使用 `----` 分隔字段
- 至少包含 `email----password----token`

### 问题4：Session 失效
- Session 默认 24 小时过期
- 重新登录即可

## 开发说明

### 目录结构
```
deno/zai/
├── main.ts                    # 主服务文件（包含账号管理功能）
├── zai_register.ts           # 账号注册工具（可选）
├── test_accounts.txt         # 测试账号文件
├── ADMIN_GUIDE.md            # 本文档
└── deno.json                 # Deno 配置
```

### 核心代码位置

- **鉴权配置**：`main.ts:18-23`
- **数据结构**：`main.ts:130-136` (ZaiAccount 接口)
- **Session 管理**：`main.ts:249-282` (generateSessionId, checkAuth)
- **KV 操作**：`main.ts:284-333` (saveAccountToKV, getAllAccounts, accountExists)
- **API 路由**：`main.ts:3383-3533`
- **HTML 界面**：`main.ts:3266-3583` (adminLoginHTML, adminPanelHTML)

### 扩展功能

如需添加新功能（如删除账号、编辑账号等），参考现有 API 实现即可。

## 许可与贡献

本功能为 ZtoApi 项目的一部分，遵循项目原始许可协议。
