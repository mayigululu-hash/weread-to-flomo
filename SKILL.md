---
name: weread-to-flomo
description: |
  把微信读书的划线和想法批量导入 flomo。触发语：「把《书名》的微信读书笔记导入 flomo」「微信读书笔记同步到 flomo」。手动触发，一次导一本书，不设自动化。
  读端走微信读书官方 Skill 网关，写端走 flomo 官方 MCP。分类规则确定性执行（划线/问答/批注），写入前整书预览确认，写入后如实汇报。
version: 1.0.0
author: 白杨
---

# Weread to Flomo — 微信读书笔记导入 flomo

## 前提

- 所在 AI 工具已安装微信读书官方 Skill（安装指令见微信读书 App 内「微信读书 skill」入口，或 `npx skills add Tencent/WeChatReading`），API Key 已配置
- 已连接 flomo 的 MCP：flomo 设置里开启「MCP / AI 工具连接」，复制接入 token 配置到 AI 工具（入口细节以 flomo 官方文档 help.flomoapp.com 为准）

## 流程（五步，一次只导一本书）

**search → fetch → classify → dedup + preview → 确认后写入 + 汇报**

用户点名多本书时，逐本走完整流程，每本单独预览。

### 1. search：锁定书

用 `/user/notebooks` 拉个人笔记本，按用户给的书名做完整书名精确匹配（casefold）：

- 唯一命中 → 取 bookId 继续
- 多本同名 / 模糊命中 / 未命中 → 列出候选（书名 + 作者 + bookId）让用户选，不擅自定版本
- 笔记本里没有这本书 → 如实说明，不硬导

书名可能带副标题（如「洞见写作：打造可复制的文章方法论」）。标签和落款用短名（第一个「：」前的部分），拉数据用 bookId。

### 2. fetch：网关拉数据

POST `https://i.weread.qq.com/api/agent/gateway`，Header `Authorization: Bearer <API Key>`，JSON body 必带 `api_name` + `skill_version: "1.0.5"`。

四个接口，参数名大小写必须照抄：

| 接口 | 关键参数 | 返回 |
|---|---|---|
| `/book/info` | `bookId`（驼峰） | 书籍元信息 |
| `/book/bookmarklist` | `bookId`（驼峰） | 划线：`updated[].markText / range / chapterUid / userEditTime` |
| `/book/chapterinfo` | `bookId`（驼峰） | 章节表：`chapters[].chapterUid / title / level / chapterIdx` |
| `/review/list/mine` | **`bookid`（全小写，写 bookId 会报「缺少必填参数: bookid」）** + `synckey` + `count:100` | 想法：`reviews[].review` 嵌套 `content / abstract / chapterUid / range / reviewId` |

`/review/list/mine` 按 `hasMore` + `synckey` 翻页拉全量；游标缺失、重复或空页声明 hasMore 都要报错，不把不完整数据当成功。

响应处理：`errcode` 非 0 → 原样报错（隐藏密钥）终止；出现 `upgrade_info` → 说明网关协议要升级，如实告知，不硬拉。

### 3. classify：三类分类

划线独立分类：`bookmarklist.updated[]` 中 `markText` 非空的每条 → **划线**。

想法按 `review.content` 非空正文判断，按顺序：

1. 正则 `(问[:：].*?)(?:\n|<br>)+(答[:：；;].*)` 命中 → **问答**（问、答拆开；答案前缀容错中文分号「；」，真实数据常见手滑变体）
2. `注[:：]` 开头 → **批注**
3. `问[:：]` 开头但没有答案 → **问答**，答案留空并在预览中标注「缺答案」
4. 其他（无前缀）→ **批注**
5. 正文为空但有 `abstract` 摘录 → 按划线导；正文摘录皆空 → 跳过

时间戳取 `userEditTime`（缺失退 `createTime`），用于排序。

### 4. dedup + preview：去重，整书预览

1. 在 flomo 按 `微信读书/<短名>` 标签搜索已有笔记（单次结果上限 50 条，超出时改用关键词补搜并注明覆盖可能不全）
2. 归一化比对（去 Markdown 符号、空白折叠）→ 一致即跳过；问答卡用「问 + 答」拼接文本比对
3. 向用户展示一次完整预览：
   - 分类统计：划线 N、问答 N、批注 N、缺答案 N、跳过 N、已存在将跳过 N
   - 三类各一条完整样例（真实数据）
   - 标签体系：`#微信读书/<短名>` + `#划线/#问答/#批注`
   - 不确定项（缺答案、部分重复）单独列出

**未经用户确认不得批量写入。**确认后立即写入，不再追问。

### 5. 写入 + 汇报

经 flomo MCP 的 `memo_create` 逐条写入（标签写在正文首行）。三条模板：

```
#微信读书/{短名} #划线

{markText}

——《{短名}》
```

```
#微信读书/{短名} #问答

**问**：{question}

**答**：{answer}
```

```
#微信读书/{短名} #批注

{content}

**原文**：{abstract}

——《{短名}》
```

批注无摘录时省略「**原文**」段和落款；问答缺答案写 `**答**：（待补）`。flomo 只支持加粗/高亮/下划线/列表，不用标题、引用块、代码块。

写入后抽 3 条读回校验（截断、标签、格式）。汇报格式：

```
导入完毕：新增 N，跳过已存在 N；划线 N，问答 N，批注 N；无前缀转批注 N，缺答案 N
```

部分失败必须如实说明（写了哪些、失败哪些），不许报成全部成功。

## 边界

- 手动触发，一次一本书；同名多版本的书不猜，列候选问用户
- 只从微信读书网关读、只写 flomo，不碰其他服务
- 网关错误、密钥缺失、协议升级提示原样传达，不吞错误
