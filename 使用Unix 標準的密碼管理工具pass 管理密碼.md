---
tags: Tool Sharing, draft
---
# 使用 Unix 標準的密碼管理工具 pass 管理密碼

## `pass`： 一個符合 Unix 標準的密碼管理工具
### `pass` 是什麼?
為了要維持好的安全性，密碼設定時需要有足夠的長度、字母數字隨機混合、定期更換密碼、不要使用固定的密碼，然而隨著使用的軟體服務、註冊的網站越來越多，人類的腦袋難以記住這些又長又複雜還要時常更換的密碼。因此我們需要一個工具來管理這些密碼，常見的工具包含 [1Password](https://1password.com/zh-tw/)、[KeePass](https://keepass.info/) 等，另外還有這篇文章要介紹的 `pass`，這是一個符合 Unix 哲學的密碼管理工具，其包含下列特色：
- 所有的密碼會透過 GPG 加密後以檔案儲存在特定目錄下
- 密碼儲存路徑及檔名命名自由，可以依照註冊網站或資源的名稱來命名方便查找
- 因為是檔案，可以複製到不同的電腦上，方便不同設備搬遷
- 透過 CLI 操作，自動補全查找密碼容易、上手簡單（fish, zsh 也相容）
- 增刪修改密碼時會透過 git 進行版本控制
- 多數裝置可以使用如 android、iOS 設備，基於 Chromium 的瀏覽器、Firefox 等都有擴充工具


`pass` 會將所有密碼儲存在 `~/.password-store` 這個目錄下，並且有簡單的指令介面可以操作該目錄下的所有檔案，檢索密碼時可以透過 `pass -c` 自動將密碼複製到剪貼簿上並會在特定時間後自動刪除。

### 使用 `pass` 管理密碼


#### Browser
#### Mobile
## What is GnuPG
### Other words in git message signing
## Reference
- [The Standard Unix Password Manager: Pass](https://www.passwordstore.org/)
- [pass - ArchWiki](https://wiki.archlinux.org/index.php/Pass)
- [The Standard Unix Password Manager](https://www.youtube.com/watch?v=hlRQTj1D9LA)
