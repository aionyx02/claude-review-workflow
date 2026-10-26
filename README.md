# claude-review-workflow

一個 Claude Code skill：需要時在專案裡打 `/setup-review-pipeline`，就搭出一整套分層的 code review 流程。

依據 claude.dev 的〈How the Claude Code team does code review〉（2026-10-15），並針對**個人 GitHub 帳號**調整：

| 層 | 內容 |
|---|---|
| 本機 | 專案 `CLAUDE.md` 列出驗證指令；開 PR 前跑 `/code-review`、`/security-review` |
| CI | 既有 CI 的 job 列為必要檢查 |
| AI review | `claude-code-action` 審查每個 PR：驗證後才留言、Important / Nit / Pre-existing 分級、三輪後收斂 |
| Security review | 獨立的第二個 job，只報有攻擊路徑的漏洞 |
| Ruleset | 不要求 approve（不能 approve 自己的 PR）；所有 review 對話必須解決、分支必須最新才能 merge |

專案分兩級：**完整**（有在維護的專案，套用上面全部）與**輕量**（作業、練習，只做本機那層，不碰 GitHub）。

## 安裝

把 `setup-review-pipeline/` 放進 Claude Code 的 skills 目錄：

```bash
cp -r setup-review-pipeline ~/.claude/skills/
```

（若 `CLAUDE_CONFIG_DIR` 指向別處，就放到該目錄下的 `skills/`。）

skill 設了 `disable-model-invocation: true`，只有手動打 `/setup-review-pipeline` 才會載入，平常不佔 context。

## 使用

在要套用的 repo 裡打 `/setup-review-pipeline`，它會依序：

1. 檢查 repo（組織 repo 會停下來；私有 repo 在 Free 方案下 ruleset 不會強制執行，會先提醒）
2. 決定分級
3. 寫入檔案並給你確認：workflow、`REVIEW.md`、PR 模板
4. 請你自己設定 secret（token 不會經過對話）：
   ```bash
   claude setup-token
   gh secret set CLAUDE_CODE_OAUTH_TOKEN --repo <owner>/<repo>
   ```
5. 開一個測試 PR，確認兩個審查都有跑
6. 經你同意後套用 ruleset

## 檔案

```
setup-review-pipeline/
├── SKILL.md                         # 決策與步驟
└── templates/
    ├── claude-review.yml            # AI review + security review workflow
    ├── REVIEW.md                    # 給審查者的 repo 專屬指示（搭建時依專案填寫）
    ├── pull_request_template.md     # What & why / Testing / Risk / How to revert
    └── ruleset.json                 # merge 規則
```
