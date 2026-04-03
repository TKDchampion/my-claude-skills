# Step 05 - Deep Code Review, Improvement Suggestions, and Security Review

## Goal

對所有已完成的 plan-step 與整體變更進行深入 code review，
檢查 correctness、maintainability、readability、consistency、performance、architecture、security。
同時執行 `commands/security-review.md` 產出深度安全檢查報告。
若 review 結果不滿意，需持續優化並重複 review，直到品質達標。

---

## Core Responsibilities

1. Review 所有 plan-step 的程式碼與整體 diff。
2. 詳細檢查是否存在：
   - 邏輯錯誤
   - 命名不一致
   - 結構不清晰
   - 重複程式碼
   - 過度耦合
   - 型別不完整
   - 不良錯誤處理
   - 不必要複雜度
   - 可讀性差
   - 維護成本高
   - 潛在效能問題
   - 架構不一致
   - 安全風險

3. 執行：
   - `commands/security-review.md`

4. 產出：
   - 詳細 code review 報告
   - 改善建議
   - 安全報告
   - 是否建議立即修正 / 可後續優化分類

5. 若 review 後有要修正的內容：
   - 實作修正
   - 重新測試
   - 再次 review
   - 重複直到滿意

6. 當使用者確認目前版本 OK 後，再執行 commit，進入 Step 06。

---

## Review Dimensions

至少要檢查：

### Code Quality

- 命名是否清楚
- 模組邊界是否合理
- 函式是否過長
- 邏輯是否過於複雜
- 是否符合 clean code

### Architecture

- 是否符合專案既有架構
- 分層是否清楚
- interface / type / abstraction 是否合理
- 是否有不必要依賴

### Maintainability

- 是否容易擴充
- 是否容易測試
- 是否容易閱讀
- 是否容易 debug

### Performance

- 是否有重複計算 / 多餘請求 / 不必要 rerender / inefficient query
- 是否存在明顯瓶頸

### Security

- 權限檢查
- 輸入驗證
- 資料暴露
- 敏感資訊處理
- injection / XSS / CSRF / auth / token / role-based risks
- 錯誤訊息是否過度暴露
- 第三方依賴風險
- 檔案上傳 / API 保護 / DB 操作安全性

---

## Security Review Command

必須執行：

- `commands/security-review.md`

若專案也有：

- `commands/owasp-security.md`
  可作為補充檢查規範一起參照

---

## Required Output Format

請用以下格式輸出 review 結果：

### Review Summary

- 本次 review 範圍：
- 整體評價：

### Issues Found

- Critical:
- Major:
- Minor:

### Improvement Suggestions

- 可立即優化：
- 可後續優化：
- 非必要但建議改善：

### Security Review

- 安全風險：
- 風險等級：
- 修正建議：

### Recommendation

- 是否建議先修正後再進入下一步
- 或目前可接受進入 Step 06

### Confirmation

- 若使用者不滿意，繼續優化並重跑 review
- 若使用者確認 OK，執行 commit 後進入 Step 06

---

## Exit Criteria

只有在以下條件都成立時才能進入 Step 06：

- 已完成完整 code review
- 已完成 security review
- 重大問題已修正
- 使用者確認目前版本可接受
- 已完成對應 commit
