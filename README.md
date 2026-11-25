# 从微信聊天记录蒸馏一个 Claude Skill — 完整实战记录

> 把一个真实微信好友（大学同学）的 **5512 条私聊 + 819 条群聊** 蒸馏成可对话的 Claude Code Skill。
> 全程本地处理，原始聊天记录不离开本机。本文记录从零到成品的每一步命令、结果与踩坑。

## 产出

```
wujindong-skill/
├── README.md      ← 本文（流程文档）
├── SKILL.md       ← 可运行的完整 Skill（记忆 + 人格合体，触发词 /wujindong）
├── memory.md      ← Part A 关系记忆（时间线 / inside jokes / 争吵档案）
├── persona.md     ← Part B 五层人格（硬规则 → 身份 → 说话风格 → 情感 → 关系行为）
├── meta.json      ← 元数据（素材来源、标签、版本）
├── memories/      ← 追加素材存放处（聊天/照片/社交）
└── versions/      ← 版本存档（支持回滚）
```

---

## 背景：为什么不用 WeChatMsg

起点是经典开源项目 [LC044/WeChatMsg](https://github.com/LC044/WeChatMsg)（留痕），源码放在本地也成功跑起来了（Python 3.12 venv + PyQt5），但**识别不到微信**。原因：

| 微信 4.2 实际情况 | WeChatMsg 的假设 |
|---|---|
| 进程名 `Weixin.exe` | 只认 3.x 的 `WeChat.exe` |
| 数据目录 `xwechat_files` | 只认老的 `WeChat Files` |
| 4.1.11+ 密钥不再以内存明文缓存 | 老式内存扫描 0 命中 |
| GitHub Releases 已清空 | 官网下载也无现成 exe |

**结论**：微信 4.x 必须换工具链。

## 工具链

| 阶段 | 工具 | 作用 |
|---|---|---|
| 1. 探测 | [wechat-4x-decrypt](https://github.com/xscanzm/wechat-4x-decrypt) `scripts/find_wechat.py` | 找进程 / 版本 / 数据目录 / 活跃账号 |
| 2. 取密钥 | [wcdb-key-tool](https://github.com/TANGandXUE/wcdb-key-tool) | 从 `Weixin.exe` 进程内存提取 WCDB passphrase（零第三方依赖，走 bcrypt.dll） |
| 3. 解密 | 同上 `decrypt` | SQLCipher 4 变体（AES-256-CBC + HMAC-SHA512）批量解出明文 SQLite |
| 4. 导出 | wechat-4x-decrypt `scripts/query_messages.py` | 按会话/关键词检索导出 TXT |
| 5. 蒸馏 | [ex-skill](https://github.com/therealXiaomanChu/ex-skill)（create-ex） | 解析统计 → 生成 memory + persona → 写成 SKILL.md |

---

## 完整流程（实测命令）

### Step 1 — 环境探测

```bash
python scripts/find_wechat.py
```

输出关键信息（需**微信 4.x 运行且已登录**，密钥只在进程内存里）：

```
Weixin.exe  pid=21372  <== 4.x 主进程
版本: C:\Program Files\Tencent\Weixin\Weixin.exe | 4.2.0.53
账号: wxid_18kr6edwzgnl22_1e41   db=24  最近写入=41s 前
--db-dir "D:\xwechat_files\wxid_18kr6edwzgnl22_1e41\db_storage"
```

### Step 2 — 提取密钥

```bash
git clone --depth 1 https://github.com/TANGandXUE/wcdb-key-tool.git
python wcdb_key_tool_windows.py extract \
  --db-dir "D:\xwechat_files\wxid_18kr6edwzgnl22_1e41\db_storage" \
  --output all_keys.json
# 结果：23 个库全部 OK（message_*.db / contact / session / sns ...）
```

### Step 3 — 解密

```bash
python wcdb_key_tool_windows.py decrypt --keys all_keys.json --output decrypted
# 结果：24/24 成功，0 失败，产出 0.8GB 明文 SQLite
```

### Step 4 — 列会话 + 导出目标联系人

```bash
# 看有哪些会话（208 个，按消息量排序）
PYTHONUTF8=1 python scripts/query_messages.py --decrypted-dir decrypted --list

# 导出目标（注意：默认 --max-per-session 会截断到 400 条！）
PYTHONUTF8=1 python scripts/query_messages.py \
  --decrypted-dir decrypted \
  --session "wxid_zpdl43v3by1912" \
  --max-per-session 100000 \
  --out-dir out
# 结果：5512 条 → out/digest.txt（2025-11-13 ~ 2026-11-24）
```

`digest.txt` 格式：`[2025-11-13 11:55] 发送者: 内容`（每行一条）

### Step 5 — 群聊定向抽取（自写 SQL）

导出工具只能按**会话**过滤，要抽"某人在所有群里说的话"需要自己查库：

```python
# 关键：message_fts.db 的 name2id 表 rowid ↔ username
#       message_fts_v4_0~3 四张分片（只查第一张会漏 ~84% 消息！）
his = 744  # SELECT rowid FROM name2id WHERE username='wxid_zpdl43v3by1912'
for i in range(4):
    cur.execute(f"""
      SELECT session_id, acontent, create_time
      FROM message_fts_v4_{i}
      WHERE sender_id=? AND session_id!=?""", (his, his))
# 群名映射：contact.db 的 contact 表，优先 remark 再 nick_name
```

结果：**819 条群聊 / 16 个群**（大头是备份里那个「叁〇叁柒伍」392 条）→ `out/group_wjd.txt`

### Step 6 — 格式转换 + 解析统计

ex-skill 的 `wechat_parser.py` 只认 WeChatMsg 格式（`时间戳 发送者\n内容`），digest 需要一行正则转换：

```python
pat = re.compile(r'^\[(\d{4}-\d{2}-\d{2} \d{2}:\d{2})\] (.+?): (.*)$')
# [2025-11-13 11:55] 吴锦东: 在吗
#   → 2025-11-13 11:55:00 吴锦东\n在吗
```

```bash
python tools/wechat_parser.py --file converted.txt --target "吴锦东" \
  --output analysis.txt --format auto
# 结果：5512 条 / ta 2869 条 / 平均 9.7 字 / 短句连发型
# 口头禅：么×218 吗×209 啊×79 吧×74
```

### Step 7 — 生成 Skill

Claude Code 里跑 create-ex 流程：**录入 → 导入素材 → 双线分析（Memory + Persona）→ 预览确认 → 写文件**：

```
exes/wujindong/
├── SKILL.md    # frontmatter + PART A + PART B + 运行规则
├── memory.md   # 时间线 / Inside Jokes / 争吵档案 / 甜蜜档案
├── persona.md  # Layer 0~4 + 群聊人设补充
└── meta.json   # 素材来源清单、标签、版本 v1
```

分析方法：parser 统计（高频词/标点/时段/主动性）+ 分段精读（头/中/尾 + 关键词定位窗口）+ 群聊通读（戏精台词、关心模式、中二文学）。

---

## 踩坑备忘

| 坑 | 解法 |
|---|---|
| 系统 Python 3.14 装不上 WeChatMsg 依赖 | 找 3.12（uv 管理的），`python -m venv .venv` |
| `ModuleNotFoundError: pkg_resources`（docxcompose） | `pip install "setuptools<81"`（81+ 移除了 pkg_resources） |
| 中文昵称含 emoji 导致 `UnicodeEncodeError: gbk` | 所有 Python 命令前加 `PYTHONUTF8=1` |
| query 默认只导 400 条 | `--max-per-session 100000` |
| 正文表分片 4 张，只查第一张漏 84% | `message_fts_v4_0~3` 全部遍历 |
| digest 格式 parser 不认 | 正则转成 `时间 发送者\n内容` |
| 微信 4.x 备份文件（`Backup/` RMFH 格式）无开源工具 | 不走备份，直接解**活数据库**（登录态即可） |

---

## 隐私与合规

- 所有解密与分析**仅在本机**进行，原始聊天记录（digest / 解密库）**不上传**
- 本仓库只含蒸馏后的 Skill 文件，仍涉及真实人物言语风格——请勿用于骚扰、跟踪或冒充本人
- 仅适合导出**自己的**微信账号数据

## 致谢

- [xscanzm/wechat-4x-decrypt](https://github.com/xscanzm/wechat-4x-decrypt) — 微信 4.x 解密检索 skill
- [TANGandXUE/wcdb-key-tool](https://github.com/TANGandXUE/wcdb-key-tool) — 进程内存取密钥
- [therealXiaomanChu/ex-skill](https://github.com/therealXiaomanChu/ex-skill) — 蒸馏框架
