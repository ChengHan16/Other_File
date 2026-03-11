$ cd my-app
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

