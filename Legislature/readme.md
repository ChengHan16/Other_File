### update
```
PS D:\Firebase\Projects\Legislature> firebase deploy
```
```
firebase deploy
```
### cmd
```
cd my-app
```
### \my-app\ios\App\App
```Info.plist``` Version change
```plist
	<key>CFBundleShortVersionString</key>
	<string>1.0.530</string>
	<key>CFBundleVersion</key>
	<string>53</string>
```
### 執行 Capacitor 同步
```
npx cap sync ios
```
### (Git) Push檔案：
```
git add .
git commit -m "chore: bump build version to 5"
git push
```
### 上傳縮圖：
`npx @capacitor/assets generate --ios`

### 注意！
#### 版本更新修改 ios/App/App/Info.plist
##### > CFBundleVersion:
```
<key>CFBundleVersion</key>
<string>5</string>
```
#### > CFBundleShortVersionString:
```
<key>CFBundleShortVersionString</key>
<string>1.0.2</string>
```
https://appstoreconnect.apple.com/login <br>
https://developer.apple.com/ <br>
https://ionic.io/login <br>

---
### 1. 標準 Firebase 登出指令
``` console
auth.signOut().then(() => {
  console.log("已成功登出");
  window.location.replace("index.html");
}).catch((error) => {
  console.error("登出失敗:", error);
});
```
### 2. 強制清除本地暫存（離線登出）
```
// 清除自訂的 Token 紀錄
localStorage.removeItem("token");

// 清除離線快取的個人資料（如大頭貼、名稱）
localStorage.removeItem("localUserName");
localStorage.removeItem("localUserAvatar");
localStorage.removeItem("localUserTheme");

// 強制跳回登入頁面
window.location.replace("index.html");
```
### 3. 進階：清除所有本機資料（最徹底）
```
localStorage.clear();
window.location.reload();
```
