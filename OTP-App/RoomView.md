```swift
//
//  RootView.swift
//  PasswordVault
//

import SwiftUI

struct RootView: View {
    @EnvironmentObject var store: VaultStore
    @EnvironmentObject var auth: AuthManager

    var body: some View {
        ZStack {
            ZStack {
                MainTabView()
                    .blur(radius: shouldBlur ? 20 : 0)
                    .disabled(shouldBlur)

                if !store.settings.hasCompletedOnboarding {
                    OnboardingView().transition(.opacity)
                }

                // 💡 關鍵修復：把 RootView 的鎖定畫面拔除，全部交給 GlobalAuthOverlay 管理
                GlobalAuthOverlay()
            }
            .id(store.settings.language)

            if store.isSwitchingLanguage {
                ZStack {
                    Color(UIColor.systemBackground).ignoresSafeArea()
                    VStack(spacing: 24) {
                        ProgressView()
                            .scaleEffect(1.5)
                        Text("切換語言中...")
                            .font(.callout)
                            .foregroundColor(.secondary)
                    }
                }
                .zIndex(9999)
                .transition(.opacity)
            }
        }
        .onTapGesture {
            hideKeyboard()
        }
        .environment(\.locale, currentLocale)
    }

    private var shouldBlur: Bool {
        !store.settings.hasCompletedOnboarding
            || (store.settings.faceID && !auth.faceIDPassed)
            || (store.settings.appLockEnabled && !auth.appPasswordPassed)
    }
    
    private var currentLocale: Locale {
        if store.settings.language == "system" {
            return Locale.current
        } else {
            return Locale(identifier: store.settings.language)
        }
    }
}

// MARK: - 首次啟動導覽頁
struct OnboardingView: View {
    @EnvironmentObject var store: VaultStore
    @EnvironmentObject var auth: AuthManager
    
    @State private var currentTab = 0
    
    // 身份資料
    @State private var birthday: Date = Date()
    @State private var idnumber: String = ""
    @State private var phone: String = ""
    
    // 主金鑰
    @State private var keyName: String = "主金鑰"
    @State private var keyValue: String = ""
    @State private var keyConfirm: String = ""
    @State private var showKey = false
    
    // 密碼鎖
    @State private var pwd1 = ""
    @State private var pwd2 = ""
    
    @State private var errorMsg = ""

    var body: some View {
        ZStack {
            Color(UIColor.systemBackground).ignoresSafeArea()
            
            TabView(selection: $currentTab) {
                // 第 1 頁：歡迎
                VStack(spacing: 24) {
                    Spacer()
                    Image(systemName: "shield.lefthalf.filled")
                        .font(.system(size: 80))
                        .foregroundColor(.accentColor)
                        .padding()
                        .background(Color.accentColor.opacity(0.1))
                        .clipShape(Circle())
                    
                    Text("歡迎使用密碼本")
                        .font(.title)
                        .bold()
                    
                    Text("我們將透過幾個簡單的步驟，協助您打造最高安全級別的個人離線金鑰庫。")
                        .font(.callout)
                        .foregroundColor(.secondary)
                        .multilineTextAlignment(.center)
                        .padding(.horizontal, 32)
                    
                    Spacer()
                    
                    Button {
                        withAnimation { currentTab = 1 }
                    } label: {
                        Text("開始設定")
                            .font(.headline)
                            .foregroundColor(.white)
                            .frame(maxWidth: .infinity)
                            .padding(.vertical, 16)
                            .background(Color.accentColor)
                            .cornerRadius(14)
                    }
                    .padding(.horizontal, 32)
                    .padding(.bottom, 40)
                }
                .tag(0)

                // 第 2 頁：身份驗證資料
                VStack(spacing: 16) {
                    Spacer()
                    Text("第 1 步：設定身份驗證")
                        .font(.title2)
                        .bold()
                    
                    Text("這三項資料將作為日後最高權限操作的憑證，一經設定無法修改，請務必牢記。")
                        .font(.callout)
                        .foregroundColor(.secondary)
                        .multilineTextAlignment(.center)
                        .padding(.horizontal, 32)
                    
                    VStack(alignment: .leading, spacing: 8) {
                        Text("出生年月日").font(.caption).foregroundColor(.secondary)
                        DatePicker("", selection: $birthday, displayedComponents: .date)
                            .labelsHidden()
                            .environment(\.locale, Locale(identifier: "zh_Hant_TW"))
                        
                        Text("證件編號").font(.caption).foregroundColor(.secondary)
                        TextField("例：A123456789", text: $idnumber)
                            .textFieldStyle(.roundedBorder)
                            .textInputAutocapitalization(.characters)
                            .autocorrectionDisabled()
                        
                        Text("電話號碼").font(.caption).foregroundColor(.secondary)
                        TextField("例：0912345678", text: $phone)
                            .textFieldStyle(.roundedBorder)
                            .keyboardType(.phonePad)
                        
                        if !errorMsg.isEmpty {
                            Text(LocalizedStringKey(errorMsg))
                                .font(.caption)
                                .foregroundColor(.red)
                                .frame(maxWidth: .infinity)
                        }
                    }
                    .padding(.horizontal, 32)
                    
                    Spacer()
                    
                    Button {
                        let trimID = idnumber.trimmingCharacters(in: .whitespaces)
                        let trimPhone = phone.trimmingCharacters(in: .whitespaces)
                        guard !trimID.isEmpty else { errorMsg = store.loc("請輸入證件編號"); return }
                        guard !trimPhone.isEmpty else { errorMsg = store.loc("請輸入電話號碼"); return }
                        errorMsg = ""
                        withAnimation { currentTab = 2 }
                    } label: {
                        Text("下一步")
                            .font(.headline)
                            .foregroundColor(.white)
                            .frame(maxWidth: .infinity)
                            .padding(.vertical, 16)
                            .background(Color.accentColor)
                            .cornerRadius(14)
                    }
                    .padding(.horizontal, 32)
                    .padding(.bottom, 40)
                }
                .tag(1)

                // 第 3 頁：建立金鑰
                VStack(spacing: 16) {
                    Spacer()
                    Text("第 2 步：建立安全金鑰")
                        .font(.title2)
                        .bold()
                    
                    Text("此金鑰將用於加密您的重要密碼與記事。為確保安全，建議設定盡可能複雜的內容。")
                        .font(.callout)
                        .foregroundColor(.secondary)
                        .multilineTextAlignment(.center)
                        .padding(.horizontal, 32)
                    
                    VStack(alignment: .leading, spacing: 8) {
                        TextField("金鑰名稱", text: $keyName)
                            .textFieldStyle(.roundedBorder)
                        
                        HStack {
                            if showKey {
                                TextField("輸入金鑰", text: $keyValue)
                                    .textInputAutocapitalization(.never)
                                    .autocorrectionDisabled()
                            } else {
                                SecureField("輸入金鑰", text: $keyValue)
                                    .textInputAutocapitalization(.never)
                                    .autocorrectionDisabled()
                            }
                            Button(action: { showKey.toggle() }) {
                                Image(systemName: showKey ? "eye.slash" : "eye")
                                    .foregroundColor(showKey ? .accentColor : .secondary)
                            }
                            .buttonStyle(.plain)
                        }
                        .padding(8)
                        .overlay(RoundedRectangle(cornerRadius: 8).stroke(Color.gray.opacity(0.3), lineWidth: 0.5))
                        
                        if showKey {
                            TextField("再次確認金鑰", text: $keyConfirm)
                                .padding(8)
                                .overlay(RoundedRectangle(cornerRadius: 8).stroke(Color.gray.opacity(0.3), lineWidth: 0.5))
                                .textInputAutocapitalization(.never)
                                .autocorrectionDisabled()
                        } else {
                            SecureField("再次確認金鑰", text: $keyConfirm)
                                .padding(8)
                                .overlay(RoundedRectangle(cornerRadius: 8).stroke(Color.gray.opacity(0.3), lineWidth: 0.5))
                                .textInputAutocapitalization(.never)
                                .autocorrectionDisabled()
                        }
                        
                        if !errorMsg.isEmpty {
                            Text(LocalizedStringKey(errorMsg))
                                .font(.caption)
                                .foregroundColor(.red)
                                .frame(maxWidth: .infinity)
                        }
                    }
                    .padding(.horizontal, 32)
                    
                    Spacer()
                    
                    Button {
                        let tName = keyName.trimmingCharacters(in: .whitespaces)
                        guard !tName.isEmpty else { errorMsg = store.loc("請輸入金鑰名稱"); return }
                        guard !keyValue.isEmpty else { errorMsg = store.loc("請輸入金鑰內容"); return }
                        guard keyValue == keyConfirm else { errorMsg = store.loc("兩次金鑰內容不一致"); return }
                        errorMsg = ""
                        withAnimation { currentTab = 3 }
                    } label: {
                        Text("下一步")
                            .font(.headline)
                            .foregroundColor(.white)
                            .frame(maxWidth: .infinity)
                            .padding(.vertical, 16)
                            .background(Color.accentColor)
                            .cornerRadius(14)
                    }
                    .padding(.horizontal, 32)
                    .padding(.bottom, 40)
                }
                .tag(2)

                // 第 4 頁：App 密碼鎖
                VStack(spacing: 20) {
                    Spacer()
                    Text("第 3 步：App 密碼鎖")
                        .font(.title2)
                        .bold()
                    
                    Text("設定後，每次開啟 App 都需要輸入此密碼。若不想現在設定，可點選下方「稍後再說」。")
                        .font(.callout)
                        .foregroundColor(.secondary)
                        .multilineTextAlignment(.center)
                        .padding(.horizontal, 32)
                    
                    VStack(alignment: .leading, spacing: 8) {
                        SecretField(text: $pwd1, placeholder: "輸入密碼鎖 (至少 4 碼)", isProtected: false)
                        SecretField(text: $pwd2, placeholder: "再次確認密碼鎖", isProtected: false)
                        
                        if !errorMsg.isEmpty {
                            Text(LocalizedStringKey(errorMsg))
                                .font(.caption)
                                .foregroundColor(.red)
                                .frame(maxWidth: .infinity)
                        }
                    }
                    .padding(.horizontal, 32)
                    
                    Spacer()
                    
                    VStack(spacing: 12) {
                        Button {
                            guard pwd1.count >= 4 else { errorMsg = store.loc("密碼鎖至少 4 個字元"); return }
                            guard pwd1 == pwd2 else { errorMsg = store.loc("兩次密碼不一致"); return }
                            errorMsg = ""
                            withAnimation { currentTab = 4 }
                        } label: {
                            Text("設定並繼續")
                                .font(.headline)
                                .foregroundColor(.white)
                                .frame(maxWidth: .infinity)
                                .padding(.vertical, 16)
                                .background(Color.accentColor)
                                .cornerRadius(14)
                        }
                        
                        Button("稍後再說") {
                            pwd1 = ""
                            pwd2 = ""
                            errorMsg = ""
                            withAnimation { currentTab = 4 }
                        }
                        .font(.callout)
                        .foregroundColor(.secondary)
                        .padding()
                    }
                    .padding(.horizontal, 32)
                    .padding(.bottom, 30)
                }
                .tag(3)

                // 第 5 頁：Face ID
                VStack(spacing: 20) {
                    Spacer()
                    Image(systemName: "faceid")
                        .font(.system(size: 80))
                        .foregroundColor(.accentColor)
                    
                    Text("第 4 步：生物辨識")
                        .font(.title2)
                        .bold()
                    
                    Text("搭配 Face ID，讓您在開啟 App 時能享受安全又快速的解鎖體驗。")
                        .font(.callout)
                        .foregroundColor(.secondary)
                        .multilineTextAlignment(.center)
                        .padding(.horizontal, 32)
                    
                    Spacer()
                    
                    VStack(spacing: 12) {
                        Button {
                            store.settings.faceID = true
                            completeOnboarding()
                        } label: {
                            Text("啟用 Face ID 並完成")
                                .font(.headline)
                                .foregroundColor(.white)
                                .frame(maxWidth: .infinity)
                                .padding(.vertical, 16)
                                .background(Color.accentColor)
                                .cornerRadius(14)
                        }
                        
                        Button("不啟用") {
                            completeOnboarding()
                        }
                        .font(.callout)
                        .foregroundColor(.secondary)
                        .padding()
                    }
                    .padding(.horizontal, 32)
                    .padding(.bottom, 30)
                }
                .tag(4)
            }
            .tabViewStyle(.page(indexDisplayMode: .never))
            .onTapGesture {
                hideKeyboard()
            }
        }
    }
    
    private func completeOnboarding() {
        let f = DateFormatter()
        f.dateFormat = "yyyy-MM-dd"
        
        store.setupIdentity(
            birthday: f.string(from: birthday),
            idnumber: idnumber.trimmingCharacters(in: .whitespaces),
            phone: phone.trimmingCharacters(in: .whitespaces)
        )
        
        store.addCustomKey(
            name: keyName.trimmingCharacters(in: .whitespaces),
            value: keyValue
        )
        
        if !pwd1.isEmpty {
            store.setAppLock(pwd1)
            auth.appPasswordPassed = true
        }
        
        store.settings.hasCompletedOnboarding = true
        store.saveSettings()
    }
}

extension View {
    func hideKeyboard() {
        UIApplication.shared.sendAction(#selector(UIResponder.resignFirstResponder), to: nil, from: nil, for: nil)
    }
}

```
