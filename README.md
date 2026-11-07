# 謝小明的 Claude Skill 推薦清單

給同學的 Claude skill 推薦清單。先依任務挑工作流套餐，或依功能分類瀏覽全部 skill；每個 skill 都附 1–5 星實用指數、來源連結和安裝方式。

整個網站只有一個檔案 `index.html`，不需要建置，放上 GitHub Pages 就能用。

## 網址

- GitHub Pages：`https://surfcorn-web.github.io/<repo 名稱>/`
- 可以直接分享的網址：
  - `#all`：完整清單
  - `#cat-social`、`#cat-slides`……：只看某一類
  - `#pack-web`、`#pack-social`……：只看某個工作流套餐

## 怎麼更新清單

1. 打開 `index.html`，找到 `const SKILLS = [`，照現有格式新增或修改一行：
   - `n` skill 名稱、`t` 中文名稱、`c` 分類、`s` 來源、`st` 星等（1–5）
   - `d` 說明、`w` 使用前要注意的事（可省略）、`u` 來源網址
2. 工作流套餐在 `const PACKS = [`，`skills` 裡填 skill 名稱。名稱打錯不會讓網頁壞掉，只會略過並在主控台警告。
3. 頁面上的數字（總數、各分類、5 星數量）都會自動重算。
4. 修改 `const DATA_UPDATED` 的日期，頁尾就會顯示新的更新日期。

## 收錄範圍

不含 Claude 內建的 skill（Word、Excel、PPT、PDF 等），因為那些不用安裝。
