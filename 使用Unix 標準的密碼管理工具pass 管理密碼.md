---
tags: Tool Sharing, draft
---
# 使用 Unix 標準的密碼管理工具 pass (password-store) 管理密碼 

## `pass`： 一個符合 Unix 標準的密碼管理工具
### `pass` 是什麼?
為了要維持好的安全性，密碼設定時需要有足夠的長度、字母數字隨機混合、定期更換密碼、不要使用固定的密碼，然而隨著使用的軟體服務、註冊的網站越來越多，人類的腦袋難以記住這些又長又複雜還要時常更換的密碼。因此我們需要一個工具來管理這些密碼，常見的工具包含 [1Password](https://1password.com/zh-tw/)、[KeePass](https://keepass.info/) 等，另外還有這篇文章要介紹的 `pass`，這是一個符合 Unix 哲學的密碼管理工具，其包含下列特色：
- 所有的密碼會透過 GPG 加密後以檔案儲存在特定目錄下
- 密碼儲存路徑及檔名命名自由，可以依照註冊網站或資源的名稱來命名方便查找
- 因為是檔案，可以複製到不同的電腦上，方便不同設備搬遷
- 透過 CLI 操作，自動補全讓查找密碼容易、上手簡單（fish, zsh 也相容）
- 增刪修改密碼時會透過 git 進行版本控制，分散式儲存
- 多數裝置可以使用如 android、iOS 設備，基於 Chromium 的瀏覽器、Firefox 等都有擴充工具


`pass` 會將所有密碼儲存在 `~/.password-store` 這個目錄下，並且有簡單的指令介面可以操作該目錄下的所有檔案，檢索密碼時可以透過 `pass -c` 自動將密碼複製到剪貼簿上並會在特定時間後自動刪除。



### 安裝 `pass`

在 MacOS 環境中，安裝 `pass` 只需要輸入指令 `brew install pass` 即可，Homebrew 會將相依的 gnupg 、tree 等工具一起安裝好。

其他環境的安裝方式可以參考[官方文件](https://www.passwordstore.org/)。

### 設定 `pass` 及 `gnupg`
由於 `pass` 儲存密碼的檔案會經過 `gpg` 加密過，因此需要先產生 GPG keypair，因為上一步已經安裝好 gnupg ，可以直接透過指令 `gpg --full-generate-key` 來產生 keypair，求方便可以選擇預設選項（密鑰種類為 RSA 用於加密及簽名、密鑰長度為 3072 bits 、密鑰不會過期）。
```
$gpg --full-gen-key
gpg (GnuPG) 2.2.26; Copyright (C) 2020 Free Software Foundation, Inc.
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.

Please select what kind of key you want:
   (1) RSA and RSA (default)
   (2) DSA and Elgamal
   (3) DSA (sign only)
   (4) RSA (sign only)
  (14) Existing key from card
Your selection? 1
RSA keys may be between 1024 and 4096 bits long.
What keysize do you want? (3072)
Requested keysize is 3072 bits
Please specify how long the key should be valid.
         0 = key does not expire
      <n>  = key expires in n days
      <n>w = key expires in n weeks
      <n>m = key expires in n months
      <n>y = key expires in n years
Key is valid for? (0)
Key does not expire at all
Is this correct? (y/N) y
```

接者需要輸入使用者資訊如使用者姓名、信箱位置，最終需要設定 passphrase，passphrase 需設定強度高一些並且要記得該設定內容，未來僅會使用 passphrase 來取得儲存於 pass 的密碼，接者可以隨意的動作如打字、用滑鼠等，以生成隨機數：

```
Real name: testuser
Email address: testuser@example.com
Comment:
You selected this USER-ID:
    "testuser <testuser@example.com>"

Change (N)ame, (C)omment, (E)mail or (O)kay/(Q)uit? o
We need to generate a lot of random bytes. It is a good idea to perform
some other action (type on the keyboard, move the mouse, utilize the
disks) during the prime generation; this gives the random number
generator a better chance to gain enough entropy.
We need to generate a lot of random bytes. It is a good idea to perform
some other action (type on the keyboard, move the mouse, utilize the
disks) during the prime generation; this gives the random number
generator a better chance to gain enough entropy.
```

完成後會輸出下列內容：
```
gpg: key 4033683BFDD66FA1 marked as ultimately trusted
gpg: revocation certificate stored as '/Users/wilson/.gnupg/openpgp-revocs.d/8CCE91BD36D05CE31210F1154033683BFDD66FA1.rev'
public and secret key created and signed.

pub   rsa3072 2021-03-31 [SC]
uid                      testuser <testuser@example.com>
sub   rsa3072 2021-03-31 [E]
```
其中包含資訊 keyid `4033683BFDD66FA1` 為使用者 ID 經過 hash 後產生的字串，後續使用可以用此字串取代輸入使用者 ID。

可以使用 `gpg --list-secret-keys --keyid-format LONG` 列出已啟用的密鑰及其密鑰 ID，而我們會使用密鑰 ID （範例中為 `4033683BFDD66FA1`）作為 `pass` 建立 password store 時需要的參數。

```
/Users/wilson/.gnupg/pubring.kbx
--------------------------------
pub   rsa3072/4033683BFDD66FA1 2021-03-31 [SC]
uid                 [ultimate] testuser <testuser@example.com>
sub   rsa3072/4B11C286610BC41C 2021-03-31 [E]
```

#### 使用 `pass init <gpg-id>` 建立 password store
接著透過 `pass init <KEYID>` 帶入剛剛產生的 Key ID，便可以建立 password store，會在 `~/.password-store` 下看到 `.gpg-id` 檔案，其中內容即為設定的 GPG Key ID。

#### 使用 `pass git init` 對密碼進行版本控制
若要使用 git 進行版本控制可以輸入 `pass git init`，`~/.password-store` 目錄下會建立 git repository 並且在 `.gitattributes` 設定 gpg 副檔名的檔案會用 gpg 解密後再進行 git diff。

若要使用 remote repository 也可以輸入 `pass git remote add <name> <url>` 加入 remote repo，並可透過 `pass git push` 及 `pass git pull` 更新密碼。

建立 password store git repository 後，`pass` 會在每次操作自動產生 git commit 記錄每次進行的變更。

### `pass` 基本操作
#### 使用 `pass generate <pass-name>` 產生密碼
#### 使用 `pass insert <pass-name>` 新增密碼
#### 使用 `pass <pass-name>` 或 `pass show <pass-name>` 查看密碼內容
#### 使用 `pass <subfolder>` 或 `pass ls <subfolder>` 查看已紀錄的密碼


### 使用 [browserpass](https://chrome.google.com/webstore/detail/browsrpass/naepdomgkenhinolocfifgehidddafch) 在瀏覽器上讀取本地密碼資料

### 使用 [Password Store](https://play.google.com/store/apps/details?id=dev.msfjarvis.aps&hl=en_US&gl=US) 和 [OpenKeychain](https://play.google.com/store/apps/details?id=org.sufficientlysecure.keychain&hl=zh_TW&gl=US) 在 Android 手機上管理密碼


## Reference
- [The Standard Unix Password Manager: Pass](https://www.passwordstore.org/)
- [pass - ArchWiki](https://wiki.archlinux.org/index.php/Pass)
- [The Standard Unix Password Manager](https://www.youtube.com/watch?v=hlRQTj1D9LA)
- [Manage your passwords on macOS using pass, the standard Unix password manager](https://guimauve.io/articles/manage-your-passwords-on-macos-using-pass-the-standard-unix-password-manager)
