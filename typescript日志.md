# 日志机制说明（可复用版）

## 1. 目标

这套日志机制的目标是：

- 控制台输出不变（开发体验不受影响）
- 同步落盘，便于线上排障
- 按天切分，文件可控
- 自动清理旧日志，避免无限增长

---

## 2. 核心工作机制

### 2.1 初始化时机

在管理端每次获取 `manager` 时，先调用日志初始化函数：

- `getManager()` -> `initDailyLogger()`
- 代码位置：`lib/manager.ts`

### 2.2 全局 console patch

`initDailyLogger()` 会 patch 这些方法：

- `console.log`
- `console.info`
- `console.warn`
- `console.error`
- `console.debug`

patch 后每次输出会做两件事：

1. 先调用原始 console（终端正常显示）
2. 再把同一条内容写入日志文件

对应实现：`src/logger.ts`

### 2.3 日志行格式

每条日志统一格式：

```text
[2026-04-21T03:12:34.567Z] [INFO] your message
```

即：

- 时间：ISO 字符串
- 级别：`INFO/WARN/ERROR/DEBUG`
- 内容：`util.format(...)` 后的文本

### 2.4 写盘方式

使用 `appendFileSync(...)` 同步追加写文件。

优点：

- 简单稳定
- 崩溃前不易丢日志

代价：

- 高频日志场景下会有阻塞成本

---

## 3. 切分与保留策略

### 3.1 按天切分

按本地日期生成文件名：

- `YYYY-MM-DD.log`
- 例如：`2026-04-21.log`

### 3.2 清理策略

清理触发时机：

- 初始化时
- 检测到跨天时

清理规则：

- 仅保留“当天的 `.log` 文件”
- 其他日期 `.log` 删除
- 非 `.log` 文件（如 `random.txt`）不处理

对应函数：`cleanupDailyLogs(logDir, keepDay)`。

---

## 4. 入站消息摘要日志

项目还定义了 Telegram 入站消息摘要日志格式（`[incoming]`）：

```text
[incoming] acc=... chat=... chatId=... mid=... sender=... text=...
```

特点：

- 文本会折叠换行为空格
- 超长文本会截断（默认 80 字符）
- 可通过环境变量开关

实现文件：`src/message-log.ts`，使用位置：`src/account-runtime.ts`。

---

## 5. 依赖与边界

### 5.1 依赖

日志模块本身只依赖 Node 内置模块：

- `fs`
- `path`
- `util`

即：不依赖 `winston/pino` 等第三方日志库。

### 5.2 生效边界

由于是 patch 全局 `console`，所以只要代码路径里用了 `console.*`，理论上都会进入同一日志文件。

---

## 6. 配置项

| 环境变量 | 默认值 | 作用 |
|---|---|---|
| `TG_LOG_DIR` | `./logs` | 日志目录 |
| `TG_LOG_INCOMING` | `1` | 是否记录入站消息摘要（`0/false/off/no` 表示关闭） |
| `NODE_ENV` | - | 当 `NODE_ENV=test` 时不启用文件日志 patch |

---

## 7. 迁移到其他项目（最小步骤）

1. 复制 `src/logger.ts`（或同等实现）到新项目。
2. 在服务启动最早阶段调用 `initDailyLogger()`。
3. 配置日志目录环境变量（如 `TG_LOG_DIR`）。
4. 若有消息系统，参考 `message-log.ts` 加入摘要格式化与开关。
5. 加一个日志清理单测，确保“只保留当天 `.log`”。

---

## 8. 可直接复用的初始化示例

```ts
import { initDailyLogger } from "./logger";

function bootstrap() {
  initDailyLogger(process.env.TG_LOG_DIR); // 可不传，走默认 ./logs
  console.log("service started");
}

bootstrap();
```

---

## 9. 可选优化建议（按需）

- 将 `appendFileSync` 替换为异步队列写入（降低阻塞）
- 增加多级保留策略（如保留近 7 天）
- 增加结构化 JSON 日志模式（便于日志平台检索）
- 为日志增加请求 ID / 账号 ID 关联字段

