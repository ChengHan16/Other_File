### Ver0.0.2
```swift
import SwiftUI
import Combine
import FirebaseCore
import FirebaseAuth
import FirebaseFirestore
import PhotosUI
import FirebaseStorage

// MARK: - 1. 核心持久化快取引擎
class PersistentImageCache {
    static let shared = PersistentImageCache()
    private let fileManager = FileManager.default
    private let cacheDirectory: URL

    init() {
        cacheDirectory = fileManager.urls(for: .cachesDirectory, in: .userDomainMask)[0].appendingPathComponent("UserAvatars")
        if !fileManager.fileExists(atPath: cacheDirectory.path) {
            try? fileManager.createDirectory(at: cacheDirectory, withIntermediateDirectories: true)
        }
    }

    func getImage(for urlString: String) -> UIImage? {
        let fileName = urlString.addingPercentEncoding(withAllowedCharacters: .alphanumerics) ?? UUID().uuidString
        let fileURL = cacheDirectory.appendingPathComponent(fileName)
        if let data = try? Data(contentsOf: fileURL) { return UIImage(data: data) }
        return nil
    }

    func saveImage(_ image: UIImage, for urlString: String) {
        let fileName = urlString.addingPercentEncoding(withAllowedCharacters: .alphanumerics) ?? UUID().uuidString
        let fileURL = cacheDirectory.appendingPathComponent(fileName)
        if let data = image.jpegData(compressionQuality: 0.8) { try? data.write(to: fileURL) }
    }
}

// MARK: - 2. 萬用智能快取圖片元件
struct CachedImage: View {
    let urlString: String
    var contentMode: ContentMode = .fill
    @State private var image: UIImage? = nil

    var body: some View {
        Group {
            if let img = image {
                Image(uiImage: img).resizable().aspectRatio(contentMode: contentMode)
            } else if urlString.isEmpty {
                Image(systemName: "person.crop.circle.fill").resizable().foregroundColor(Color(UIColor.systemGray4))
            } else {
                Color(UIColor.systemGray6).overlay(ProgressView()).onAppear { loadImage() }
            }
        }
    }

    private func loadImage() {
        if let cached = PersistentImageCache.shared.getImage(for: urlString) { self.image = cached; return }
        guard let url = URL(string: urlString) else { return }
        URLSession.shared.dataTask(with: url) { data, _, _ in
            if let data = data, let downloadedImage = UIImage(data: data) {
                PersistentImageCache.shared.saveImage(downloadedImage, for: urlString)
                DispatchQueue.main.async { self.image = downloadedImage }
            }
        }.resume()
    }
}

struct PhotoItem: Identifiable { let id = UUID(); let url: String }
struct ReactionSheetItem: Identifiable { let id = UUID(); let emoji: String; let uids: [String] }

// MARK: - 3. 全螢幕專業圖片瀏覽器
struct FullScreenImageView: View {
    let urlString: String
    @Environment(\.dismiss) var dismiss
    @State private var scale: CGFloat = 1.0; @State private var lastScale: CGFloat = 1.0
    @State private var offset: CGSize = .zero; @State private var lastOffset: CGSize = .zero

    var body: some View {
        ZStack {
            Color.black.ignoresSafeArea()
            CachedImage(urlString: urlString, contentMode: .fit)
                .scaleEffect(scale).offset(offset)
                .gesture(MagnificationGesture().onChanged { val in
                    let delta = val / lastScale; lastScale = val; scale = min(max(scale * delta, 1), 5)
                }.onEnded { _ in lastScale = 1.0; if scale < 1 { withAnimation(.spring()) { scale = 1.0; offset = .zero } } })
                .simultaneousGesture(DragGesture().onChanged { val in
                    if scale > 1 { offset = CGSize(width: lastOffset.width + val.translation.width, height: lastOffset.height + val.translation.height) }
                    else { if val.translation.height > 0 { offset = val.translation } }
                }.onEnded { val in
                    if scale > 1 { lastOffset = offset }
                    else { if val.translation.height > 100 { dismiss() } else { withAnimation(.spring()) { offset = .zero } } }
                })
                .onTapGesture(count: 2) { withAnimation(.spring()) { scale = scale > 1 ? 1.0 : 2.5; offset = .zero; lastOffset = .zero } }
            VStack {
                HStack { Spacer(); Button(action: { dismiss() }) { Image(systemName: "xmark").font(.system(size: 20, weight: .bold)).foregroundColor(.white).padding(12).background(Color.black.opacity(0.5)).clipShape(Circle()) }.padding() }
                Spacer()
            }
        }
    }
}

// MARK: - 4. 成員卡片 UI
struct MemberCardView: View {
    let profile: UserProfile; let isSelected: Bool; let isEditing: Bool; let isMe: Bool
    let goldColor = Color(red: 207/255, green: 169/255, blue: 0)
    var body: some View {
        VStack(spacing: 6) {
            CachedImage(urlString: profile.photoURL, contentMode: .fill).frame(width: 50, height: 50).clipShape(Circle()).overlay(Circle().stroke(isSelected ? goldColor : Color.clear, lineWidth: 2))
            Text(profile.displayName).font(.system(size: 12, weight: .semibold)).foregroundColor(isSelected ? goldColor : .primary).lineLimit(1).truncationMode(.tail)
        }
        .padding(.vertical, 10).padding(.horizontal, 4).frame(maxWidth: .infinity, minHeight: 100)
        .background(isSelected ? goldColor.opacity(0.1) : Color(UIColor.systemBackground)).cornerRadius(12)
        .overlay(RoundedRectangle(cornerRadius: 12).stroke(isSelected ? goldColor : Color(UIColor.systemGray5), lineWidth: 1.5))
        .overlay(Group { if isEditing && !isMe { Image(systemName: isSelected ? "checkmark.circle.fill" : "circle").foregroundColor(isSelected ? goldColor : Color.gray.opacity(0.5)).font(.system(size: 20)).padding(4) } }, alignment: .topTrailing)
        .opacity(isMe && isEditing ? 0.5 : 1.0)
    }
}

// MARK: - App Entry Point
@main
struct LegislatureApp: App {
    init() {
        FirebaseApp.configure()
        let settings = FirestoreSettings()
        settings.cacheSettings = PersistentCacheSettings(sizeBytes: NSNumber(value: FirestoreCacheSizeUnlimited))
        Firestore.firestore().settings = settings
    }
    @AppStorage("isDarkMode") private var isDarkMode: Bool = false
    var body: some Scene { WindowGroup { RootContentView().preferredColorScheme(isDarkMode ? .dark : .light) } }
}

class AppState: ObservableObject {
    @Published var isAuthenticated: Bool = false
    init() { _ = Auth.auth().addStateDidChangeListener { [weak self] _, user in DispatchQueue.main.async { self?.isAuthenticated = (user != nil) } } }
}

struct UserProfile: Identifiable { let id: String; var displayName: String; var photoURL: String }
struct ChatModel: Identifiable, Hashable { let id: String; var title: String; var avatarURL: String; var lastMessage: String; var updatedAt: Date; var isGroup: Bool; var isUnread: Bool; var isPinned: Bool }
struct MessageModel: Identifiable, Equatable {
    let id: String; let text: String; let senderId: String; let timestamp: Date; let isMine: Bool
    var replyToText: String?; var fileUrl: String?; var fileType: String?; var reactions: [String: String]?
}

// MARK: - ChatList ViewModel
class ChatListViewModel: ObservableObject {
    @Published var chats: [ChatModel] = []; @Published var isLoading: Bool = true
    private var db = Firestore.firestore(); private var listener: ListenerRegistration?; private var userCache: [String: UserProfile] = [:]
    private let defaultAvatar = "https://github.com/ChengHan16/Other_File/blob/master/Legislature/photo/avatar/avatar-default.png?raw=true"
    
    func fetchChats() {
        guard let currentUid = Auth.auth().currentUser?.uid else { return }
        listener = db.collection("chats").whereField("members", arrayContains: currentUid).order(by: "updatedAt", descending: true)
            .addSnapshotListener { [weak self] querySnapshot, error in
                guard let self = self, let documents = querySnapshot?.documents else { self?.isLoading = false; return }
                var loadedChats: [ChatModel] = []; let dispatchGroup = DispatchGroup()
                for doc in documents {
                    let data = doc.data(); let isGroup = data["isGroup"] as? Bool ?? false
                    let lastMessage = data["lastMessage"] as? String ?? "尚無訊息"
                    let members = data["members"] as? [String] ?? []; let updatedAt = (data["updatedAt"] as? Timestamp)?.dateValue() ?? Date()
                    let lastSenderId = data["lastSenderId"] as? String ?? ""
                    var isUnread = false
                    if let lastReadDict = data["lastReadAt"] as? [String: Timestamp], let myLastRead = lastReadDict[currentUid] {
                        if lastSenderId != currentUid && updatedAt > myLastRead.dateValue() { isUnread = true }
                    } else if lastSenderId != currentUid { isUnread = true }
                    
                    if isGroup {
                        let groupName = data["groupName"] as? String ?? "未命名群組"
                        let groupAvatar = (data["groupAvatar"] as? String ?? "").isEmpty ? "https://ui-avatars.com/api/?name=\(groupName.addingPercentEncoding(withAllowedCharacters: .urlQueryAllowed) ?? "")&background=003153&color=fff" : data["groupAvatar"] as! String
                        loadedChats.append(ChatModel(id: doc.documentID, title: groupName, avatarURL: groupAvatar, lastMessage: lastMessage, updatedAt: updatedAt, isGroup: true, isUnread: isUnread, isPinned: (groupName == "立法院 Legislature")))
                    } else {
                        if let targetUid = members.first(where: { $0 != currentUid }) {
                            dispatchGroup.enter()
                            self.fetchUserProfile(uid: targetUid) { profile in
                                loadedChats.append(ChatModel(id: doc.documentID, title: profile.displayName, avatarURL: profile.photoURL, lastMessage: lastMessage, updatedAt: updatedAt, isGroup: false, isUnread: isUnread, isPinned: false))
                                dispatchGroup.leave()
                            }
                        }
                    }
                }
                dispatchGroup.notify(queue: .main) {
                    self.chats = loadedChats.sorted { if $0.isPinned == $1.isPinned { return $0.updatedAt > $1.updatedAt }; return $0.isPinned && !$1.isPinned }
                    self.isLoading = false
                }
            }
    }
    private func fetchUserProfile(uid: String, completion: @escaping (UserProfile) -> Void) {
        if let cached = userCache[uid] { completion(cached); return }
        db.collection("act").document(uid).getDocument(source: .cache) { [weak self] cacheDoc, _ in
            if let doc = cacheDoc, doc.exists { self?.handleProfileDoc(uid: uid, doc: doc, completion: completion) }
            else { self?.db.collection("act").document(uid).getDocument { doc, _ in self?.handleProfileDoc(uid: uid, doc: doc, completion: completion) } }
        }
    }
    private func handleProfileDoc(uid: String, doc: DocumentSnapshot?, completion: @escaping (UserProfile) -> Void) {
        let name = doc?.data()?["displayName"] as? String ?? "未知成員"
        let photoURL = doc?.data()?["photoURL"] as? String ?? ""
        let profile = UserProfile(id: uid, displayName: name, photoURL: photoURL.isEmpty ? defaultAvatar : photoURL)
        self.userCache[uid] = profile; completion(profile)
    }
}

// MARK: - ChatRoom ViewModel
class ChatRoomViewModel: ObservableObject {
    @Published var messages: [MessageModel] = []; @Published var isLoadingInitial: Bool = true
    @Published var isFetchingOlder: Bool = false; @Published var lastReadAt: [String: Date] = [:]
    @Published var membersCache: [String: UserProfile] = [:]
    @Published var pinnedMessageText: String? = nil; @Published var replyingToMessage: MessageModel? = nil
    @Published var searchText: String = ""; @Published var groupMembers: [UserProfile] = []
    
    @Published var activeMenuMsgId: String? = nil
    @Published var myDisplayName: String = "" // 儲存自己的身份，判定「班長」特權
    
    let chatId: String
    private var db = Firestore.firestore(); private var messagesListener: ListenerRegistration?; private var chatDocListener: ListenerRegistration?
    private var currentMessageLimit: Int = 20; private let limitStep: Int = 20; private var hasMoreMessages: Bool = true
    private let defaultAvatar = "https://github.com/ChengHan16/Other_File/blob/master/Legislature/photo/avatar/avatar-default.png?raw=true"
    
    init(chatId: String) { self.chatId = chatId }
    
    func startListening() {
        guard let currentUid = Auth.auth().currentUser?.uid else { return }
        
        db.collection("act").document(currentUid).getDocument(source: .cache) { doc, _ in
            if let name = doc?.data()?["displayName"] as? String { DispatchQueue.main.async { self.myDisplayName = name } }
            else { self.db.collection("act").document(currentUid).getDocument { doc, _ in DispatchQueue.main.async { self.myDisplayName = doc?.data()?["displayName"] as? String ?? "未知" } } }
        }
        
        messagesListener?.remove(); chatDocListener?.remove()
        chatDocListener = db.collection("chats").document(chatId).addSnapshotListener { [weak self] doc, _ in
            guard let self = self, let data = doc?.data() else { return }
            if let pinnedDict = data["pinnedMessage"] as? [String: Any], let text = pinnedDict["text"] as? String { DispatchQueue.main.async { self.pinnedMessageText = text } } else { DispatchQueue.main.async { self.pinnedMessageText = nil } }
            if let readDict = data["lastReadAt"] as? [String: Timestamp] {
                var newReadAt: [String: Date] = [:]
                for (k, v) in readDict { newReadAt[k] = v.dateValue() }
                DispatchQueue.main.async { self.lastReadAt = newReadAt }
            }
            let members = data["members"] as? [String] ?? []
            self.fetchMembersCache(uids: members)
        }
        
        messagesListener = db.collection("chats").document(chatId).collection("messages").order(by: "timestamp", descending: true).limit(to: currentMessageLimit)
            .addSnapshotListener { [weak self] snapshot, error in
                guard let self = self, let documents = snapshot?.documents else { self?.isFetchingOlder = false; return }
                if documents.count < self.currentMessageLimit { self.hasMoreMessages = false }
                var loadedMessages: [MessageModel] = []
                for doc in documents {
                    let data = doc.data(); let text = data["text"] as? String ?? ""; let senderId = data["senderId"] as? String ?? ""
                    loadedMessages.append(MessageModel(id: doc.documentID, text: text, senderId: senderId, timestamp: (data["timestamp"] as? Timestamp)?.dateValue() ?? Date(), isMine: senderId == currentUid, replyToText: data["replyToText"] as? String, fileUrl: data["fileUrl"] as? String, fileType: data["fileType"] as? String, reactions: data["reactions"] as? [String: String]))
                }
                DispatchQueue.main.async { self.messages = loadedMessages.reversed(); self.isFetchingOlder = false; self.isLoadingInitial = false; self.updateMyReadStatus() }
            }
    }
    
    func fetchMembersCache(uids: [String]) {
        var newMembersList: [UserProfile] = []; let group = DispatchGroup()
        for uid in uids {
            group.enter()
            if let cached = membersCache[uid] { newMembersList.append(cached); group.leave() } else {
                db.collection("act").document(uid).getDocument(source: .cache) { [weak self] cacheDoc, _ in
                    if let doc = cacheDoc, doc.exists { let p = self?.createProfile(uid: uid, doc: doc); if let p = p { newMembersList.append(p) }; group.leave() } else {
                        self?.db.collection("act").document(uid).getDocument { doc, _ in let p = self?.createProfile(uid: uid, doc: doc); if let p = p { newMembersList.append(p) }; group.leave() }
                    }
                }
            }
        }
        group.notify(queue: .main) { self.groupMembers = newMembersList }
    }
    private func createProfile(uid: String, doc: DocumentSnapshot?) -> UserProfile {
        let name = doc?.data()?["displayName"] as? String ?? "未知成員"
        let photoURL = doc?.data()?["photoURL"] as? String ?? ""
        let profile = UserProfile(id: uid, displayName: name, photoURL: photoURL.isEmpty ? defaultAvatar : photoURL)
        DispatchQueue.main.async { self.membersCache[uid] = profile }; return profile
    }
    
    func toggleReaction(messageId: String, emoji: String) {
        guard let currentUid = Auth.auth().currentUser?.uid else { return }
        let msgRef = db.collection("chats").document(chatId).collection("messages").document(messageId)
        if let msg = messages.first(where: { $0.id == messageId }) {
            let currentReaction = msg.reactions?[currentUid]
            if currentReaction == emoji { msgRef.updateData(["reactions.\(currentUid)": FieldValue.delete()]) }
            else { msgRef.updateData(["reactions.\(currentUid)": emoji]) }
        }
    }
    
    func removeMembers(uids: [String]) { db.collection("chats").document(chatId).updateData(["members": FieldValue.arrayRemove(uids)]) }
    func updateGroupInfo(name: String) { db.collection("chats").document(chatId).updateData(["groupName": name]) }
    func uploadGroupAvatar(data: Data) {
        let ref = Storage.storage().reference().child("groupAvatars/\(chatId)/\(Date().timeIntervalSince1970).jpg")
        ref.putData(data, metadata: nil) { _, _ in ref.downloadURL { url, _ in if let urlStr = url?.absoluteString { self.db.collection("chats").document(self.chatId).updateData(["groupAvatar": urlStr]) } } }
    }
    
    var filteredMessages: [MessageModel] { if searchText.isEmpty { return messages }; return messages.filter { $0.text.localizedCaseInsensitiveContains(searchText) } }
    var historyPhotos: [String] { messages.compactMap { $0.fileType == "image" ? $0.fileUrl : nil } }
    func loadMoreMessages() { guard hasMoreMessages, !isFetchingOlder else { return }; isFetchingOlder = true; currentMessageLimit += limitStep; startListening() }
    func sendMessage(text: String) {
        guard let currentUid = Auth.auth().currentUser?.uid, !text.isEmpty else { return }
        let chatRef = db.collection("chats").document(chatId); let msgRef = chatRef.collection("messages").document(); let batch = db.batch()
        var msgData: [String: Any] = ["text": text, "senderId": currentUid, "timestamp": FieldValue.serverTimestamp()]
        if let reply = replyingToMessage { msgData["replyToId"] = reply.id; msgData["replyToText"] = reply.text.trimmingCharacters(in: .whitespacesAndNewlines) }
        batch.setData(msgData, forDocument: msgRef)
        batch.updateData(["lastMessage": text, "lastSenderId": currentUid, "updatedAt": FieldValue.serverTimestamp(), "lastReadAt.\(currentUid)": FieldValue.serverTimestamp()], forDocument: chatRef)
        batch.commit(); self.replyingToMessage = nil
    }
    func deleteMessage(_ messageId: String) { db.collection("chats").document(chatId).collection("messages").document(messageId).delete() }
    func pinMessage(_ message: MessageModel) {
        let senderName = membersCache[message.senderId]?.displayName ?? "成員"
        let pinData: [String: Any] = ["id": message.id, "text": message.text, "senderName": senderName]
        db.collection("chats").document(chatId).updateData(["pinnedMessage": pinData])
    }
    func unpinMessage() { db.collection("chats").document(chatId).updateData(["pinnedMessage": FieldValue.delete()]) }
    private func updateMyReadStatus() { guard let currentUid = Auth.auth().currentUser?.uid else { return }; db.collection("chats").document(chatId).updateData(["lastReadAt.\(currentUid)": FieldValue.serverTimestamp()]) }
    func stopListening() { messagesListener?.remove(); chatDocListener?.remove() }
}

class InviteViewModel: ObservableObject {
    @Published var availableUsers: [UserProfile] = []; @Published var isLoading = true
    func fetchUsers(excludeUIDs: [String]) {
        Firestore.firestore().collection("act").getDocuments { [weak self] snap, err in
            guard let docs = snap?.documents else { self?.isLoading = false; return }
            var users: [UserProfile] = []
            for doc in docs {
                if !excludeUIDs.contains(doc.documentID) {
                    let data = doc.data(); users.append(UserProfile(id: doc.documentID, displayName: data["displayName"] as? String ?? "未知", photoURL: data["photoURL"] as? String ?? ""))
                }
            }
            DispatchQueue.main.async { self?.availableUsers = users.sorted { $0.displayName < $1.displayName }; self?.isLoading = false }
        }
    }
}

// MARK: - Root & Tab View
struct RootContentView: View {
    @StateObject private var appState = AppState()
    var body: some View { Group { if appState.isAuthenticated { MainTabView() } else { LoginView(appState: appState) } } }
}

struct MainTabView: View {
    @State private var selectedTab = 1
    @State private var myAvatarUrl = Auth.auth().currentUser?.photoURL?.absoluteString ?? ""
    @State private var myName = "自己"
    
    @State private var isTabBarHidden = false
    let lyBlue = Color(red: 0/255, green: 49/255, blue: 83/255)
    let lyGold = Color(red: 184/255, green: 134/255, blue: 11/255)

    var body: some View {
        ZStack(alignment: .bottom) {
            // 第 1 層：主要畫面切換
            Group {
                switch selectedTab {
                case 0: Text("首頁 (準備開發)").frame(maxWidth: .infinity, maxHeight: .infinity).background(Color(UIColor.systemGroupedBackground))
                case 1: ChatListView()
                case 2: Text("關於我 (準備開發)").frame(maxWidth: .infinity, maxHeight: .infinity).background(Color(UIColor.systemGroupedBackground))
                case 3: Text("日曆 (CalendarView)").font(.title).frame(maxWidth: .infinity, maxHeight: .infinity).background(Color(UIColor.systemGroupedBackground))
                case 4: SettingsView()
                default: EmptyView()
                }
            }.frame(maxWidth: .infinity, maxHeight: .infinity)
            
            // 第 2 層：凸起式自訂底部導覽列
            if !isTabBarHidden {
                HStack(spacing: 0) {
                    tabButton(icon: "house.fill", title: "首頁", index: 0)
                    tabButton(icon: "message.fill", title: "聊天", index: 1)
                    
                    Spacer().frame(width: 80) // 留空給中間大按鈕
                    
                    tabButton(icon: "calendar", title: "日曆", index: 3)
                    tabButton(icon: "ellipsis", title: "其他", index: 4)
                }
                .frame(height: 55)
                .padding(.horizontal, 10)
                .background(Color.white.shadow(color: Color.black.opacity(0.08), radius: 3, x: 0, y: -3).ignoresSafeArea(.all, edges: .bottom))
                
                .overlay(
                    Button(action: { selectedTab = 2 }) {
                        VStack(spacing: 4) {
                            ZStack {
                                Circle()
                                    .fill(Color.white)
                                    .frame(width: 65, height: 65)
                                    .shadow(color: Color.black.opacity(0.15), radius: 5, x: 0, y: -2)
                                
                                if myAvatarUrl.isEmpty {
                                    Image(systemName: "person.crop.circle.fill").resizable().frame(width: 55, height: 55).foregroundColor(lyBlue)
                                } else {
                                    CachedImage(urlString: myAvatarUrl, contentMode: .fill).frame(width: 55, height: 55).clipShape(Circle())
                                }
                            }
                            
                            Text(myName)
                                .font(.system(size: 10, weight: .black))
                                .foregroundColor(selectedTab == 2 ? lyGold : lyBlue)
                                .lineLimit(1)
                        }
                    }
                    .offset(y: -20), alignment: .top
                )
                .transition(.move(edge: .bottom).combined(with: .opacity))
            }
        }
        .ignoresSafeArea(.keyboard, edges: .bottom)
        .onAppear { fetchMyProfile() }
        
        .onChange(of: selectedTab) { _ in withAnimation(.easeIn(duration: 0.2)) { isTabBarHidden = false } }
        .onReceive(NotificationCenter.default.publisher(for: NSNotification.Name("HideTabBar"))) { _ in withAnimation(.easeOut(duration: 0.2)) { isTabBarHidden = true } }
        .onReceive(NotificationCenter.default.publisher(for: NSNotification.Name("ShowTabBar"))) { _ in withAnimation(.easeIn(duration: 0.2)) { isTabBarHidden = false } }
    }
    
    private func fetchMyProfile() {
        guard let uid = Auth.auth().currentUser?.uid else { return }
        Firestore.firestore().collection("act").document(uid).getDocument { doc, _ in
            if let data = doc?.data() {
                self.myName = data["displayName"] as? String ?? "未知"
                self.myAvatarUrl = data["photoURL"] as? String ?? ""
            }
        }
    }
    
    func tabButton(icon: String, title: String, index: Int) -> some View {
        Button(action: { selectedTab = index }) {
            VStack(spacing: 5) {
                Image(systemName: icon).font(.system(size: 22))
                Text(title).font(.system(size: 10, weight: .bold))
            }
            .foregroundColor(selectedTab == index ? lyBlue : .gray.opacity(0.5))
            .frame(maxWidth: .infinity)
        }
    }
}

// MARK: - Chat List View
struct ChatListView: View {
    @StateObject private var viewModel = ChatListViewModel()
    @State private var showNewChatSheet = false; @State private var showCreateGroupSheet = false
    var body: some View {
        NavigationStack {
            List {
                if viewModel.isLoading { HStack { Spacer(); ProgressView("載入對話中..."); Spacer() }.listRowSeparator(.hidden) } else {
                    ForEach(viewModel.chats) { chat in
                        ZStack { NavigationLink(destination: ChatRoomView(chat: chat)) { EmptyView() }.opacity(0); ChatRowView(chat: chat) }
                        .alignmentGuide(.listRowSeparatorLeading) { _ in 0 }.listRowInsets(EdgeInsets(top: 12, leading: 16, bottom: 12, trailing: 16))
                    }
                }
            }.listStyle(.plain).navigationTitle("訊息中心").navigationBarTitleDisplayMode(.inline)
            .toolbar {
                ToolbarItem(placement: .navigationBarTrailing) {
                    HStack(spacing: 16) {
                        Button(action: { showCreateGroupSheet = true }) { Image(systemName: "person.2.fill").foregroundColor(Color(red: 0, green: 49/255, blue: 83/255)) }
                        Button(action: { showNewChatSheet = true }) { Image(systemName: "plus").font(.system(size: 18, weight: .bold)).foregroundColor(Color(red: 0, green: 49/255, blue: 83/255)) }
                    }
                }
            }.onAppear { viewModel.fetchChats() }
        }
    }
}

struct ChatRowView: View {
    let chat: ChatModel
    var body: some View {
        HStack(spacing: 15) {
            ZStack(alignment: .topLeading) {
                CachedImage(urlString: chat.avatarURL, contentMode: .fill).frame(width: 55, height: 55).clipShape(Circle())
                if chat.isPinned { Image(systemName: "pin.fill").font(.system(size: 14)).foregroundColor(.yellow).rotationEffect(.degrees(-45)).shadow(color: .black.opacity(0.3), radius: 2, x: 1, y: 1).offset(x: -5, y: -2) }
            }
            VStack(alignment: .leading, spacing: 6) {
                Text(chat.title).font(.system(size: 18, weight: chat.isUnread ? .heavy : .bold)).foregroundColor(.primary).lineLimit(1)
                Text(chat.lastMessage).font(.system(size: 15)).foregroundColor(chat.isUnread ? .primary : .secondary).lineLimit(1)
            }
            Spacer()
            Text(formatTime(date: chat.updatedAt)).font(.system(size: 14)).foregroundColor(.secondary)
        }.overlay(Circle().fill(chat.isUnread ? Color.yellow : Color.clear).frame(width: 8, height: 8).offset(x: -10), alignment: .leading)
    }
    private func formatTime(date: Date) -> String { let f = DateFormatter(); f.dateFormat = Calendar.current.isDateInToday(date) ? "HH:mm" : "MM/dd"; return f.string(from: date) }
}

// MARK: - Chat Room View
struct ChatRoomView: View {
    let chat: ChatModel
    @StateObject private var viewModel: ChatRoomViewModel
    @State private var messageText = ""
    @AppStorage("currentUid") private var storedUid: String = Auth.auth().currentUser?.uid ?? ""
    @State private var isMenuExpanded = false; @State private var showSettings = false
    @State private var viewingPhoto: PhotoItem? = nil
    @State private var topMessageId: String? = nil; @State private var isViewReady = false
    
    init(chat: ChatModel) { self.chat = chat; _viewModel = StateObject(wrappedValue: ChatRoomViewModel(chatId: chat.id)) }
    
    var body: some View {
        VStack(spacing: 0) {
            if let pinnedText = viewModel.pinnedMessageText {
                HStack {
                    Image(systemName: "pin.fill").foregroundColor(.yellow).rotationEffect(.degrees(-45))
                    Text(pinnedText).font(.subheadline).foregroundColor(.primary).lineLimit(1)
                    Spacer()
                    Button(action: { viewModel.unpinMessage() }) { Image(systemName: "xmark").foregroundColor(.gray) }
                }.padding(.horizontal, 16).padding(.vertical, 10).background(Color(UIColor.secondarySystemBackground).opacity(0.95))
            }
            
            ScrollViewReader { proxy in
                ScrollView {
                    LazyVStack(spacing: 12) {
                        if viewModel.isFetchingOlder { ProgressView().padding() }
                        if let firstId = viewModel.messages.first?.id {
                            Color.clear.frame(height: 1).id("top_anchor").onAppear {
                                if !viewModel.isFetchingOlder && viewModel.messages.count >= 20 { topMessageId = firstId; viewModel.loadMoreMessages() }
                            }
                        }
                        
                        ForEach(viewModel.messages) { message in
                            MessageBubbleView(
                                message: message, isGroup: chat.isGroup, viewModel: viewModel, currentUid: storedUid,
                                onPhotoTap: { url in viewingPhoto = PhotoItem(url: url) }
                            ).id(message.id)
                        }
                        Color.clear.frame(height: 1).id("bottom_anchor")
                    }.padding(.vertical).padding(.horizontal, 10)
                }
                .id("ChatScroll_\(chat.id)")
                .opacity(isViewReady ? 1 : 0)
                .onTapGesture {
                    UIApplication.shared.sendAction(#selector(UIResponder.resignFirstResponder), to: nil, from: nil, for: nil)
                    withAnimation(.spring(response: 0.3, dampingFraction: 0.7)) { viewModel.activeMenuMsgId = nil }
                }
                .onChange(of: viewModel.isLoadingInitial) { loading in
                    if !loading {
                        proxy.scrollTo("bottom_anchor", anchor: .bottom)
                        DispatchQueue.main.asyncAfter(deadline: .now() + 0.05) { proxy.scrollTo("bottom_anchor", anchor: .bottom); isViewReady = true }
                    }
                }
                .onChange(of: viewModel.messages) { newMessages in
                    let isPagination = newMessages.first?.id != topMessageId && viewModel.isFetchingOlder
                    if isPagination, let topId = topMessageId { proxy.scrollTo(topId, anchor: .top) }
                    else if !viewModel.isLoadingInitial { if isViewReady { withAnimation { proxy.scrollTo("bottom_anchor", anchor: .bottom) } } else { proxy.scrollTo("bottom_anchor", anchor: .bottom) } }
                }
                // 👉 修正 2：展開選單時，將目標訊息滾動至畫面正中央，避免被上方或下方擠出畫面
                .onChange(of: viewModel.activeMenuMsgId) { msgId in
                    if let targetId = msgId {
                        DispatchQueue.main.asyncAfter(deadline: .now() + 0.1) {
                            withAnimation(.spring(response: 0.3, dampingFraction: 0.7)) {
                                proxy.scrollTo(targetId, anchor: .center)
                            }
                        }
                    }
                }
            }
            
            if let reply = viewModel.replyingToMessage {
                HStack {
                    Text("回覆：\(reply.text)").font(.caption).foregroundColor(.gray).lineLimit(1)
                    Spacer()
                    Button(action: { viewModel.replyingToMessage = nil }) { Image(systemName: "xmark.circle.fill").foregroundColor(.gray) }
                }.padding(.horizontal, 16).padding(.vertical, 8).background(Color(UIColor.systemBackground)); Divider()
            }
            
            VStack(spacing: 0) {
                Divider()
                HStack(spacing: 12) {
                    if !messageText.isEmpty && !isMenuExpanded {
                        Button(action: { withAnimation(.spring()) { isMenuExpanded = true } }) { Image(systemName: "chevron.right").font(.title2).foregroundColor(.gray).padding(.horizontal, 4) }.transition(.move(edge: .leading).combined(with: .opacity))
                    } else {
                        HStack(spacing: 16) {
                            Button(action: { if !messageText.isEmpty { withAnimation(.spring()) { isMenuExpanded = false } } }) { Image(systemName: "plus").font(.title2).foregroundColor(.gray) }
                            Button(action: { }) { Image(systemName: "camera").font(.title2).foregroundColor(.gray) }
                            Button(action: { }) { Image(systemName: "photo").font(.title2).foregroundColor(.gray) }
                        }.transition(.move(edge: .leading).combined(with: .opacity))
                    }
                    TextField("請輸入訊息...", text: $messageText).padding(10).background(Color(UIColor.secondarySystemBackground)).cornerRadius(20)
                        .onChange(of: messageText) { newValue in withAnimation(.spring()) { if newValue.isEmpty { isMenuExpanded = true } else if !newValue.isEmpty && isMenuExpanded { isMenuExpanded = false } } }
                    Button(action: { viewModel.sendMessage(text: messageText); messageText = "" }) { Image(systemName: "paperplane.fill").padding(10).background(Color(red: 0, green: 49/255, blue: 83/255)).foregroundColor(.white).clipShape(Circle()) }.disabled(messageText.trimmingCharacters(in: .whitespaces).isEmpty)
                }.padding().background(Color(UIColor.systemBackground))
            }
        }.navigationTitle(chat.title).navigationBarTitleDisplayMode(.inline).toolbar(.hidden, for: .tabBar)
        .toolbar { ToolbarItem(placement: .navigationBarTrailing) { Button(action: { showSettings = true }) { Image(systemName: "ellipsis").rotationEffect(.degrees(90)).foregroundColor(Color(red: 0, green: 49/255, blue: 83/255)) } } }
        .sheet(isPresented: $showSettings) { ChatSettingsView(viewModel: viewModel, originalTitle: chat.title, isGroup: chat.isGroup) }
        .fullScreenCover(item: $viewingPhoto) { photo in FullScreenImageView(urlString: photo.url) }
        .onAppear {
            viewModel.startListening()
            NotificationCenter.default.post(name: NSNotification.Name("HideTabBar"), object: nil)
        }
        // 👉 修正 1：退出聊天室時，發送 ShowTabBar 廣播叫出導覽列
        .onDisappear {
            viewModel.stopListening()
            isViewReady = false
            NotificationCenter.default.post(name: NSNotification.Name("ShowTabBar"), object: nil)
        }
    }
}

// MARK: - Message Bubble View (內嵌 0.3s 極速選單與底部反應膠囊)
struct MessageBubbleView: View {
    let message: MessageModel
    let isGroup: Bool
    @ObservedObject var viewModel: ChatRoomViewModel
    let currentUid: String
    var onPhotoTap: (String) -> Void
    
    @State private var showReadersSheet = false
    @State private var reactionSheetItem: ReactionSheetItem? = nil
    
    var readers: [UserProfile] {
        var list: [UserProfile] = []
        for (uid, date) in viewModel.lastReadAt where uid != message.senderId {
            if date >= message.timestamp {
                if let profile = viewModel.membersCache[uid] { list.append(profile) }
                else { list.append(UserProfile(id: uid, displayName: "成員", photoURL: "")) }
            }
        }
        return list
    }
    
    var readStatusText: String {
        let r = readers; if r.isEmpty { return "" }
        if !isGroup && !message.isMine { return "" }
        if !isGroup { return "已讀" }
        if r.count <= 2 { return r.map { $0.displayName }.joined(separator: "、") }
        else { let firstTwo = r.prefix(2).map { $0.displayName }.joined(separator: "、"); return "\(firstTwo)...等 \(r.count) 人" }
    }
    
    var body: some View {
        HStack(alignment: .bottom, spacing: 8) {
            if message.isMine {
                Spacer()
                if isGroup {
                    VStack(alignment: .trailing, spacing: 4) {
                        BubbleContent()
                        HStack(spacing: 8) { if !readStatusText.isEmpty { ReadStatusText() }; MessageTimeText() }
                    }
                } else {
                    VStack(alignment: .trailing, spacing: 2) { if !readStatusText.isEmpty { ReadStatusText() }; MessageTimeText() }.padding(.bottom, 2)
                    BubbleContent()
                }
            } else {
                let senderProfile = viewModel.membersCache[message.senderId]
                VStack(alignment: .center, spacing: 4) {
                    CachedImage(urlString: senderProfile?.photoURL ?? "", contentMode: .fill).frame(width: 35, height: 35).clipShape(Circle())
                    if isGroup { Text(senderProfile?.displayName ?? "成員").font(.system(size: 10)).foregroundColor(.gray).lineLimit(1).frame(width: 45) }
                }
                if isGroup {
                    VStack(alignment: .leading, spacing: 4) {
                        BubbleContent()
                        HStack(spacing: 8) { MessageTimeText(); if !readStatusText.isEmpty { ReadStatusText() } }
                    }
                } else {
                    BubbleContent(); VStack(alignment: .leading, spacing: 2) { MessageTimeText() }.padding(.bottom, 2)
                }
                Spacer()
            }
        }
        .sheet(isPresented: $showReadersSheet) {
            NavigationStack {
                List(readers) { reader in HStack(spacing: 12) { CachedImage(urlString: reader.photoURL, contentMode: .fill).frame(width: 40, height: 40).clipShape(Circle()); Text(reader.displayName).font(.system(size: 16, weight: .semibold)) }.padding(.vertical, 4) }
                .navigationTitle("已讀成員").navigationBarTitleDisplayMode(.inline).toolbar { ToolbarItem(placement: .navigationBarTrailing) { Button("關閉") { showReadersSheet = false } } }
            }.presentationDetents([.medium, .large])
        }
        .sheet(item: $reactionSheetItem) { item in
            NavigationStack {
                List(item.uids, id: \.self) { uid in let member = viewModel.membersCache[uid]; HStack(spacing: 12) { CachedImage(urlString: member?.photoURL ?? "", contentMode: .fill).frame(width: 40, height: 40).clipShape(Circle()); Text(member?.displayName ?? "成員").font(.system(size: 16, weight: .semibold)) }.padding(.vertical, 4) }
                .navigationTitle("\(item.emoji) 回應成員").navigationBarTitleDisplayMode(.inline).toolbar { ToolbarItem(placement: .navigationBarTrailing) { Button("關閉") { reactionSheetItem = nil } } }
            }.presentationDetents([.medium, .large])
        }
    }
    
    @ViewBuilder
    private func BubbleContent() -> some View {
        let showCustomMenu = (viewModel.activeMenuMsgId == message.id)

        VStack(alignment: message.isMine ? .trailing : .leading, spacing: 6) {
            // 🚀 0.3秒極速顯示的氣泡內嵌選單
            if showCustomMenu {
                VStack(alignment: message.isMine ? .trailing : .leading, spacing: 8) {
                    // 水平排列表情符號列
                    HStack(spacing: 15) {
                        ForEach(["👍", "❤️", "😂", "🙏", "👀"], id: \.self) { emoji in
                            Button(emoji) {
                                viewModel.toggleReaction(messageId: message.id, emoji: emoji)
                                withAnimation(.spring()) { viewModel.activeMenuMsgId = nil }
                            }.font(.system(size: 26))
                        }
                    }
                    .padding(.horizontal, 16).padding(.vertical, 10).background(Color(UIColor.systemBackground)).cornerRadius(25).shadow(color: Color.black.opacity(0.12), radius: 6, y: 3)
                    
                    // 操作功能列
                    HStack(spacing: 20) {
                        MenuActionBtn(icon: "doc.on.doc") { UIPasteboard.general.string = message.text; withAnimation { viewModel.activeMenuMsgId = nil } }
                        MenuActionBtn(icon: "arrowshape.turn.up.left") { viewModel.replyingToMessage = message; withAnimation { viewModel.activeMenuMsgId = nil } }
                        MenuActionBtn(icon: "pin") { viewModel.pinMessage(message); withAnimation { viewModel.activeMenuMsgId = nil } }
                        if message.isMine { MenuActionBtn(icon: "trash", isRed: true) { viewModel.deleteMessage(message.id); withAnimation { viewModel.activeMenuMsgId = nil } } }
                    }
                    .padding(.horizontal, 16).padding(.vertical, 12).background(Color(UIColor.systemBackground)).cornerRadius(18).shadow(color: Color.black.opacity(0.12), radius: 6, y: 3)
                }
                .transition(.scale(scale: 0.8, anchor: message.isMine ? .bottomTrailing : .bottomLeading).combined(with: .opacity)).zIndex(100)
            }

            // 🚀 回覆破版修復：加入 lineLimit(1) 與 frame(maxWidth)
            if let replyText = message.replyToText, !replyText.isEmpty {
                Text("回覆：\(replyText)")
                    .font(.caption2).foregroundColor(message.isMine ? .white.opacity(0.8) : .gray)
                    .lineLimit(1).truncationMode(.tail)
                    .frame(maxWidth: 220, alignment: .leading) // 強制限制最大寬度
                    .padding(.horizontal, 8).padding(.vertical, 4)
                    .background(message.isMine ? Color.white.opacity(0.2) : Color.gray.opacity(0.2)).cornerRadius(6)
            }
            
            Group {
                if let imgUrl = message.fileUrl, message.fileType == "image" {
                    CachedImage(urlString: imgUrl, contentMode: .fill)
                        .frame(width: 220, height: 280)
                        .background(Color(UIColor.systemGray5))
                        .cornerRadius(12)
                        .clipped()
                        .onTapGesture {
                            onPhotoTap(imgUrl)
                        }
                } else {
                    Text(message.text).padding(.horizontal, 16).padding(.vertical, 12).background(message.isMine ? Color(red: 0, green: 49/255, blue: 83/255) : Color(UIColor.secondarySystemBackground)).foregroundColor(message.isMine ? .white : .primary).clipShape(ChatBubbleShape(isMine: message.isMine))
                }
            }
            .onLongPressGesture(minimumDuration: 0.3) {
                UIApplication.shared.sendAction(#selector(UIResponder.resignFirstResponder), to: nil, from: nil, for: nil)
                withAnimation(.spring(response: 0.3, dampingFraction: 0.7)) { viewModel.activeMenuMsgId = (viewModel.activeMenuMsgId == message.id) ? nil : message.id }
            }

            // 🚀 底部表情符號膠囊區塊與頭像疊加
            if let reactions = message.reactions, !reactions.isEmpty {
                let grouped = Dictionary(grouping: reactions.keys, by: { reactions[$0]! })
                HStack(spacing: 6) {
                    ForEach(grouped.keys.sorted(), id: \.self) { emoji in
                        let uids = grouped[emoji]!
                        let hasMe = uids.contains(currentUid)
                        
                        HStack(spacing: 4) {
                            Text(emoji).font(.system(size: 14))
                            if isGroup {
                                HStack(spacing: -6) {
                                    ForEach(uids.prefix(3), id: \.self) { uid in
                                        let profile = viewModel.membersCache[uid]
                                        CachedImage(urlString: profile?.photoURL ?? "", contentMode: .fill).frame(width: 18, height: 18).clipShape(Circle()).overlay(Circle().stroke(Color(UIColor.systemBackground), lineWidth: 1.5))
                                    }
                                }
                            }
                            if uids.count > 1 || !isGroup { Text("\(uids.count)").font(.system(size: 11, weight: .bold)).foregroundColor(.gray) }
                        }
                        .padding(.horizontal, 8).padding(.vertical, 4).background(hasMe ? Color(red: 0, green: 49/255, blue: 83/255).opacity(0.1) : Color(UIColor.secondarySystemBackground)).overlay(RoundedRectangle(cornerRadius: 12).stroke(hasMe ? Color(red: 0, green: 49/255, blue: 83/255) : Color.clear, lineWidth: 1))
                        .onTapGesture { viewModel.toggleReaction(messageId: message.id, emoji: emoji) }
                        .onLongPressGesture { reactionSheetItem = ReactionSheetItem(emoji: emoji, uids: uids) }
                    }
                }.padding(.top, 2)
            }
        }
    }
    
    @ViewBuilder private func ReadStatusText() -> some View { Text(readStatusText).font(.system(size: 11, weight: isGroup ? .semibold : .regular)).foregroundColor(isGroup ? Color(red: 0, green: 49/255, blue: 83/255) : .gray).onTapGesture { if isGroup { showReadersSheet = true } } }
    @ViewBuilder private func MessageTimeText() -> some View { Text(formatMessageTime(date: message.timestamp)).font(.system(size: 11)).foregroundColor(.gray) }
    private func formatMessageTime(date: Date) -> String { let f = DateFormatter(); f.dateFormat = "HH:mm"; return f.string(from: date) }
}

struct MenuActionBtn: View {
    let icon: String; let isRed: Bool; let action: () -> Void
    init(icon: String, isRed: Bool = false, action: @escaping () -> Void) { self.icon = icon; self.isRed = isRed; self.action = action }
    var body: some View { Button(action: action) { Image(systemName: icon).font(.system(size: 22)).foregroundColor(isRed ? .red : .primary) } }
}

// MARK: - 🚀 Chat Settings 班長特權與相簿上傳
struct ChatSettingsView: View {
    @ObservedObject var viewModel: ChatRoomViewModel
    let originalTitle: String
    @State private var chatTitleEdit: String
    let isGroup: Bool
    @Environment(\.dismiss) var dismiss

    @State private var isExpandedMembers = false; @State private var isEditingMembers = false; @State private var memberSearchText = ""
    @State private var selectedToRemove = Set<String>(); @State private var selectedPhotoItem: PhotosPickerItem? = nil; @State private var showInviteSheet = false

    init(viewModel: ChatRoomViewModel, originalTitle: String, isGroup: Bool) {
        self.viewModel = viewModel; self.originalTitle = originalTitle; self._chatTitleEdit = State(initialValue: originalTitle); self.isGroup = isGroup
    }

    // 🚀 權限判斷引擎：立法院群組只有班長能改，其他群組大家都能改
    var canEditGroup: Bool {
        guard isGroup else { return false }
        if originalTitle == "立法院 Legislature" { return viewModel.myDisplayName == "班長" }
        return true
    }

    var body: some View {
        NavigationStack {
            List {
                Section(header: Text("基本設定")) {
                    if isGroup {
                        HStack {
                            Text("群組名稱")
                            TextField("輸入名稱", text: $chatTitleEdit).multilineTextAlignment(.trailing).foregroundColor(canEditGroup ? .primary : .gray).disabled(!canEditGroup)
                        }
                        if canEditGroup {
                            PhotosPicker(selection: $selectedPhotoItem, matching: .images, photoLibrary: .shared()) { Text("更換群組頭像").foregroundColor(.blue) }
                            .onChange(of: selectedPhotoItem) { newItem in Task { if let data = try? await newItem?.loadTransferable(type: Data.self) { viewModel.uploadGroupAvatar(data: data) } } }
                        }
                    }
                    NavigationLink(destination: ChatSearchView(viewModel: viewModel)) { Label("搜尋聊天內容", systemImage: "magnifyingglass") }
                }

                Section(header: Text("多媒體與檔案")) { NavigationLink(destination: PhotoGalleryView(photos: viewModel.historyPhotos)) { Label("歷史照片 (\(viewModel.historyPhotos.count))", systemImage: "photo.on.rectangle") } }

                Section(header: HStack {
                    Text("成員名單 (\(viewModel.groupMembers.count))"); Spacer()
                    if isGroup && canEditGroup {
                        Button(isEditingMembers ? "完成" : "編輯") { withAnimation { isEditingMembers.toggle(); if !isEditingMembers { selectedToRemove.removeAll() }; if isEditingMembers { isExpandedMembers = true } } }.font(.subheadline).foregroundColor(Color(red: 0, green: 49/255, blue: 83/255))
                    }
                }) {
                    if isExpandedMembers || isEditingMembers { HStack { Image(systemName: "magnifyingglass").foregroundColor(.gray); TextField("搜尋成員...", text: $memberSearchText) }.padding(8).background(Color(UIColor.secondarySystemBackground)).cornerRadius(8).padding(.vertical, 4) }
                    let filteredMembers = viewModel.groupMembers.filter { memberSearchText.isEmpty || $0.displayName.localizedCaseInsensitiveContains(memberSearchText) }
                    let displayedMembers = (isExpandedMembers || isEditingMembers) ? filteredMembers : Array(filteredMembers.prefix(3))

                    LazyVGrid(columns: [GridItem(.adaptive(minimum: 80), spacing: 10)], spacing: 10) {
                        ForEach(displayedMembers) { member in
                            MemberCardView(profile: member, isSelected: selectedToRemove.contains(member.id), isEditing: isEditingMembers, isMe: member.id == Auth.auth().currentUser?.uid)
                            .onTapGesture { if isEditingMembers { if member.id == Auth.auth().currentUser?.uid { return }; if selectedToRemove.contains(member.id) { selectedToRemove.remove(member.id) } else { selectedToRemove.insert(member.id) } } }
                        }
                    }.padding(.vertical, 8)

                    if !isEditingMembers && !isExpandedMembers && filteredMembers.count > 3 { Button(action: { withAnimation { isExpandedMembers = true } }) { HStack { Spacer(); Text("展開所有成員 (\(filteredMembers.count))").font(.subheadline).foregroundColor(.blue); Spacer() } } }
                    if isEditingMembers && !selectedToRemove.isEmpty { Button(role: .destructive, action: { viewModel.removeMembers(uids: Array(selectedToRemove)); isEditingMembers = false; selectedToRemove.removeAll() }) { HStack { Spacer(); Text("移除選取的成員 (\(selectedToRemove.count))").fontWeight(.bold); Spacer() } } }
                    if isGroup && canEditGroup && !isEditingMembers { Button(action: { showInviteSheet = true }) { Label("邀請新成員", systemImage: "person.badge.plus").foregroundColor(Color(red: 0, green: 49/255, blue: 83/255)) } }
                }
            }
            .navigationTitle("聊天室設定").navigationBarTitleDisplayMode(.inline)
            .toolbar {
                ToolbarItem(placement: .navigationBarTrailing) { if canEditGroup { Button("完成") { if isGroup && chatTitleEdit != originalTitle { viewModel.updateGroupInfo(name: chatTitleEdit) }; dismiss() }.bold() } }
                ToolbarItem(placement: .navigationBarLeading) { Button("關閉") { dismiss() } }
            }
            .sheet(isPresented: $showInviteSheet) { InviteMemberView(chatId: viewModel.chatId, currentMembers: viewModel.groupMembers.map { $0.id }) }
        }
    }
}

// MARK: - Search View & Gallery
struct ChatSearchView: View {
    @ObservedObject var viewModel: ChatRoomViewModel
    var body: some View { VStack { HStack { Image(systemName: "magnifyingglass").foregroundColor(.gray); TextField("搜尋關鍵字...", text: $viewModel.searchText) }.padding(10).background(Color(UIColor.secondarySystemBackground)).cornerRadius(10).padding(); List(viewModel.filteredMessages) { msg in VStack(alignment: .leading, spacing: 6) { Text(viewModel.membersCache[msg.senderId]?.displayName ?? "成員").font(.caption).fontWeight(.bold).foregroundColor(Color(red: 0, green: 49/255, blue: 83/255)); Text(msg.text).font(.body); Text(msg.timestamp, style: .date).font(.caption2).foregroundColor(.gray) } }.listStyle(.plain) }.navigationTitle("搜尋對話") }
}

struct PhotoGalleryView: View {
    let photos: [String]; let columns = [GridItem(.flexible(), spacing: 2), GridItem(.flexible(), spacing: 2), GridItem(.flexible(), spacing: 2)]; @State private var viewingPhoto: PhotoItem?
    var body: some View { ScrollView { if photos.isEmpty { Text("尚無歷史照片").foregroundColor(.gray).padding(.top, 50) } else { LazyVGrid(columns: columns, spacing: 2) { ForEach(photos, id: \.self) { url in Button(action: { viewingPhoto = PhotoItem(url: url) }) { CachedImage(urlString: url, contentMode: .fill).frame(width: UIScreen.main.bounds.width / 3, height: UIScreen.main.bounds.width / 3).clipped() } } } } }.navigationTitle("歷史照片").fullScreenCover(item: $viewingPhoto) { photo in FullScreenImageView(urlString: photo.url) } }
}

struct InviteMemberView: View {
    let chatId: String; let currentMembers: [String]; @Environment(\.dismiss) var dismiss; @StateObject private var inviteVM = InviteViewModel(); @State private var searchText = ""; @State private var selectedUIDs = Set<String>()
    var body: some View {
        NavigationStack {
            VStack(spacing: 0) {
                HStack { Image(systemName: "magnifyingglass").foregroundColor(.gray); TextField("搜尋成員名稱...", text: $searchText) }.padding(10).background(Color(UIColor.secondarySystemBackground)).cornerRadius(10).padding()
                if inviteVM.isLoading { Spacer(); ProgressView("載入名單中..."); Spacer() } else {
                    let filtered = inviteVM.availableUsers.filter { searchText.isEmpty || $0.displayName.localizedCaseInsensitiveContains(searchText) }
                    if filtered.isEmpty { Spacer(); Text("沒有找到其他可邀請的成員").foregroundColor(.gray); Spacer() } else {
                        ScrollView { LazyVGrid(columns: [GridItem(.adaptive(minimum: 80), spacing: 10)], spacing: 10) { ForEach(filtered) { user in MemberCardView(profile: user, isSelected: selectedUIDs.contains(user.id), isEditing: true, isMe: false).onTapGesture { if selectedUIDs.contains(user.id) { selectedUIDs.remove(user.id) } else { selectedUIDs.insert(user.id) } } } }.padding(.horizontal).padding(.bottom, 20) }
                    }
                }
            }.navigationTitle("邀請新成員").navigationBarTitleDisplayMode(.inline)
            .toolbar { ToolbarItem(placement: .navigationBarLeading) { Button("取消") { dismiss() } }; ToolbarItem(placement: .navigationBarTrailing) { Button("加入 (\(selectedUIDs.count))") { if !selectedUIDs.isEmpty { Firestore.firestore().collection("chats").document(chatId).updateData(["members": FieldValue.arrayUnion(Array(selectedUIDs))]) }; dismiss() }.bold().disabled(selectedUIDs.isEmpty) } }
            .onAppear { inviteVM.fetchUsers(excludeUIDs: currentMembers) }
        }
    }
}

struct ChatBubbleShape: Shape { let isMine: Bool; func path(in rect: CGRect) -> Path { let p = UIBezierPath(roundedRect: rect, byRoundingCorners: [.topLeft, .topRight, isMine ? .bottomLeft : .bottomRight], cornerRadii: CGSize(width: 18, height: 18)); return Path(p.cgPath) } }
struct SettingsView: View { @AppStorage("isDarkMode") private var isDarkMode: Bool = false; var body: some View { NavigationStack { List { Section { Toggle(isOn: $isDarkMode) { Label("深色模式", systemImage: isDarkMode ? "moon.fill" : "sun.max.fill").foregroundColor(isDarkMode ? .yellow : .orange) } }; Section { Button("登出帳號") { try? Auth.auth().signOut() }.foregroundColor(.red) } }.navigationTitle("設定與其他") } } }
struct LoginView: View {
    @ObservedObject var appState: AppState; @State private var email = ""; @State private var password = ""; @State private var errorMessage = ""
    var body: some View { VStack(spacing: 25) { Text("🏛️ 立法院").font(.system(size: 32, weight: .bold)).foregroundColor(Color(red: 0, green: 49/255, blue: 83/255)); VStack(alignment: .leading, spacing: 15) { TextField("帳號 (Email)", text: $email).textFieldStyle(RoundedBorderTextFieldStyle()).autocapitalization(.none); SecureField("密碼 (Password)", text: $password).textFieldStyle(RoundedBorderTextFieldStyle()) }.padding(.horizontal, 30); if !errorMessage.isEmpty { Text(errorMessage).foregroundColor(.red).font(.footnote) }; Button("進入") { Auth.auth().signIn(withEmail: email, password: password) { _, e in if let e = e { errorMessage = e.localizedDescription } } }.padding().frame(maxWidth: .infinity).background(Color(red: 0, green: 49/255, blue: 83/255)).foregroundColor(.white).cornerRadius(8).padding(.horizontal, 30) } }
}

````
