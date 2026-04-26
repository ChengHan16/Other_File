# Password Vault iOS

iOS 原生密碼管理 App，對應 HTML 原型版本的所有功能，並升級為 Swift + SwiftUI + CryptoKit + Keychain 實作。

## 系統需求

- Xcode 15+
- iOS 17+
- Swift 5.9+

## 建立 Xcode 專案

由於這只是 Swift 原始碼，你需要在 Xcode 建立專案後將檔案匯入：

### 步驟

1. 開啟 Xcode，選 **Create New Project** → **iOS App**
2. 設定：
   - **Product Name**: `PasswordVault`
   - **Bundle Identifier**: `tw.passwordvault.app`（或自訂）
   - **Interface**: SwiftUI
   - **Language**: Swift
   - **Storage**: None
3. 把 `PasswordVault/` 資料夾內所有 `.swift` 檔案拖入專案
4. **替換**自動產生的 `ContentView.swift` 與 `xxxApp.swift`，使用本專案的 `RootView.swift` 與 `PasswordVaultApp.swift`
5. 在 **Signing & Capabilities** 設定簽署團隊
6. 在 **Info.plist** 加入：

```xml
<key>NSFaceIDUsageDescription</key>
<string>使用 Face ID 解鎖密碼本</string>
```

7. 在 **Info.plist** 加入：

```xml
<key>UIFileSharingEnabled</key>
<true/>
<key>LSSupportsOpeningDocumentsInPlace</key>
<true/>
```

確保使用者可以匯入/匯出 `.pvbak` 檔案。

## 檔案結構

```
PasswordVault/
├── PasswordVaultApp.swift        # App 入口
├── Models.swift                  # 資料模型
├── CryptoService.swift           # 加密服務（含 CommonCrypto PBKDF2）
├── KeychainService.swift         # Keychain 存取
├── VaultStore.swift              # 資料管理 (ObservableObject)
├── AuthManager.swift             # 驗證流程管理
├── SecretField.swift             # 三段顯示混合元件
├── RootView.swift                # 根視圖
├── MainTabView.swift             # 底部導航
├── RegisterView.swift            # 首次設定
├── FaceIDLockView.swift          # Face ID 鎖
├── AppLockView.swift             # App 密碼鎖驗證
├── HomeView.swift                # 首頁
├── VaultView.swift               # 密碼列表
├── EntryEditView.swift           # 編輯密碼
├── EntryDetailView.swift         # 查看密碼
├── NotesView.swift               # 記事本
├── SettingsView.swift            # 設定
├── AuthDialogs.swift             # 各種驗證對話框
└── DocViews.swift                # 操作說明 + 技術白皮書
```

## 安全層級對照

| 機制 | HTML 版 | iOS 版 |
|------|---------|--------|
| 資料儲存 | localStorage | iOS Keychain（kSecAttrAccessibleWhenUnlockedThisDeviceOnly）|
| 三重加密 | JS 凱薩+Base64 | Swift 凱薩+Base64（示範用，正式版改 CryptoKit AES-GCM）|
| 匯出加密 | WebCrypto PBKDF2 + AES-GCM | CommonCrypto PBKDF2 + CryptoKit AES.GCM |
| TOTP | JS HMAC-SHA1 | CryptoKit HMAC<Insecure.SHA1> |
| Face ID | 模擬畫面 | LocalAuthentication LAContext |
| 隨機數 | crypto.getRandomValues | SecRandomCopyBytes |

## 功能對應 HTML 版本

✅ 首次身份註冊（生日 / 證件 / 電話三重加密儲存）
✅ Face ID 解鎖
✅ App 密碼鎖（可選啟用）
✅ TOTP 動態驗證（可選啟用，30 秒輪替）
✅ 三項全對 / 任一驗證
✅ 受保護欄位三段顯示（圓圈 → 凱薩 → 明文）
✅ 密碼產生器（長度、複雜度、各種旗標）
✅ 加密金鑰、資料鎖、密文加密金鑰
✅ 平台自訂（含長按抖動動作模式）
✅ 自訂平台圖示（自動壓縮到 64×64）
✅ 信箱尾綴自動建議
✅ 匯出（PBKDF2 + AES-256-GCM）/ 匯入
✅ 記事本（一般 / 機密兩種類型）
✅ 操作說明 + 技術白皮書內建頁面

## 後續強化建議

1. **CryptoKit 全面替換三重加密**：將 `CryptoService.tripleEncrypt/Decrypt` 改為 `AES.GCM.seal/open`，使用 Secure Enclave 派生金鑰
2. **Keychain 存取群組**：設定 Keychain Access Group 以支援 App Extension（如未來增加自動填入）
3. **App Group**：若要支援 Widget 或 ShareExtension，需設定 App Group
4. **iCloud 備份排除**：可選擇將 Keychain 設為 `kSecAttrSynchronizable: false` 以避免跨裝置同步
5. **檢視鎖定逾時**：背景超過 N 分鐘自動鎖定（已在 `PasswordVaultApp` 監聽 `scenePhase == .background`）
6. **密碼學審計**：上架前進行第三方安全稽核
