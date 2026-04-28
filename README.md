# Semantic Release & Commitlint Demo

這個 repo 用來示範 **commitlint** + **semantic-release** 的完整工作流程。

---

## Commitlint 與 Conventional Commits

`commitlint.config.js` 繼承 `@commitlint/config-conventional`，強制所有 commit message 必須符合 [Conventional Commits](https://www.conventionalcommits.org/) 格式：

```
<type>(<scope>): <subject>

[optional body]

[optional footer]
```

### Commit Type 一覽

| Type | 說明 |
|------|------|
| `feat` | 新增功能 |
| `fix` | 修復 bug |
| `docs` | 文件變更 |
| `style` | 格式調整（不影響邏輯） |
| `refactor` | 重構（不新增功能、不修 bug） |
| `perf` | 效能改善 |
| `test` | 新增或修改測試 |
| `build` | 建置系統或外部套件異動 |
| `ci` | CI/CD 設定變更 |
| `chore` | 其他雜務（不影響原始碼） |
| `revert` | 還原先前的 commit |

### 錯誤範例

```
# ❌ 缺少 type prefix
update button color

# ❌ type 不在清單內
change: update button color

# ❌ subject 開頭大寫
feat: Update button color
```

### 正確範例

```
# ✅
feat: add dark mode toggle
fix(button): correct hover color
docs: update installation steps
```

---

## Commitlint 與 Semantic Release 的關係

兩者共同依賴 **Conventional Commits** 格式，但職責不同：

```
commit message
     │
     ├─► commitlint   → 格式驗證（PR 階段，不符合格式直接擋下）
     │
     └─► semantic-release → 語意分析（merge 到 main 後，決定版本號）
```

- **commitlint** 是守門員：確保進入 main 的每一個 commit 格式正確
- **semantic-release** 是解析器：讀取 commit type 來判斷這次要 bump 哪個版本號

兩者使用相同的 type 詞彙，所以通過 commitlint 的 commit，semantic-release 必然能正確解析。

---

## Commit Type → 版本號規則

semantic-release 透過 `@semantic-release/commit-analyzer` 分析 commit，對應 [semver](https://semver.org/) 規則：

```
版本格式：MAJOR.MINOR.PATCH
          1    . 2    . 3
```

| Commit 內容 | 版本 Bump | 範例 |
|------------|-----------|------|
| `fix:` | PATCH `1.0.0 → 1.0.1` | 修 bug，不影響 API |
| `feat:` | MINOR `1.0.0 → 1.1.0` | 新功能，向下相容 |
| `feat:!` | MAJOR `1.0.0 → 2.0.0` | 不向下相容的變更 |
| `docs:` `chore:` `style:` 等 | 不 bump | 不影響使用者的變更 |

---

## 分支策略與預發布版本

`.releaserc.json` 設定三條分支，對應不同的發布管道：

| 分支 | 用途 | 版本格式 | 範例 |
|------|------|---------|------|
| `main` | 正式發布 | `MAJOR.MINOR.PATCH` | `1.2.0` |
| `beta` | 公開測試版 | `MAJOR.MINOR.PATCH-beta.N` | `1.2.0-beta.1` |

### 典型工作流程

```
beta ──────────────────────────────────────────► 1.2.0-beta.1
  │  feat: add new dashboard                      1.2.0-beta.2
  │  fix: resolve sidebar bug
  │
  └──► main ──────────────────────────────────► 1.2.0
           (穩定後 merge，發正式版)
```

---

## Semantic Release Plugin 說明

`.releaserc.json` 的 `plugins` 陣列依序執行，每個 plugin 負責一個階段：

### 1. `@semantic-release/commit-analyzer`

分析自上次 release 以來的所有 commit，依照 Conventional Commits 規則決定這次要 bump 哪個版本號。若沒有任何會觸發版本 bump 的 commit，則不發布。

```json
"@semantic-release/commit-analyzer"
```

### 2. `@semantic-release/release-notes-generator`

將 commit 清單整理成人類可讀的 Release Notes（Markdown 格式），作為後續 plugin 的輸入來源。

```json
"@semantic-release/release-notes-generator"
```

### 3. `@semantic-release/changelog`

將 Release Notes 寫入 `CHANGELOG.md`。若檔案不存在則自動建立；若已存在則在最上方追加新版本內容。

```json
["@semantic-release/changelog", {
  "changelogFile": "CHANGELOG.md"
}]
```

### 4. `@semantic-release/npm`

更新 `package.json` 的 `version` 欄位。`npmPublish: false` 表示只更新版本號，不實際發布到 npm registry。

```json
["@semantic-release/npm", {
  "npmPublish": false
}]
```

> 若要發布到 npm，將 `npmPublish` 改為 `true` 並在 GitHub Actions 設定 `NPM_TOKEN` secret。

### 5. `@semantic-release/git`

將 `package.json`（含新版本號）與 `CHANGELOG.md` commit 回 repo。`[skip ci]` 標記確保這個自動 commit 不會再次觸發 CI。

```json
["@semantic-release/git", {
  "assets": ["package.json", "CHANGELOG.md"],
  "message": "chore(release): ${nextRelease.version} [skip ci]"
}]
```

### 6. `@semantic-release/github`

在 GitHub 上建立 Release，並將指定檔案作為附件上傳。`dist/index.js.gz` 是在 CI 中壓縮 `src/index.js` 產生的。

```json
["@semantic-release/github", {
  "assets": [
    {
      "path": "dist/index.js.gz",
      "label": "index.js (gzip compressed)"
    }
  ]
}]
```