```swift
import SwiftUI
import Combine
import AVFoundation
import AVKit
import WebKit
import PhotosUI
import FirebaseCore
import FirebaseAuth
import FirebaseFirestore
import FirebaseStorage

// MARK: - 1. 應用程式入口 (App Entry)
@main
struct LegislatureMessengerApp: App {
    init() {
        FirebaseApp.configure()
    }
    
    var body: some Scene {
        WindowGroup {
            ContentView()
        }
    }
}

// MARK: - 2. 資料模型 (Models)
struct ChatItem: Identifiable {
    var id: String
    var name: String
    var lastMessage: String
    var time: String
    var isUnread: Bool
    var isGroup: Bool
    var avatarUrl: String
}

struct DeviceSession: Identifiable {
    let id: String
    let ip: String
    let location: String
    let lastLogin: String
    let isCurrent: Bool
}

struct ReadMember: Identifiable, Equatable {
    let id: String
    let name: String
    let avatar: String
}

struct MessageItem: Identifiable {
    var id: String
    var text: String
    var senderId: String
    var timestamp: Date
    var isMine: Bool
    var isSystem: Bool = false
    var isDeleted: Bool = false
    var senderAvatar: String = ""
    var senderName: String = ""
    var fileUrl: String = ""
    var fileType: String = ""
    var cardType: String = ""
    var cardData: [String: Any]?
    var readBy: [ReadMember] = []
    var replyToText: String = ""
    var reactions: [String: [String]] = [:]
}

struct EventItem: Identifiable {
    var id: String
    var title: String
    var time: String
    var creatorUid: String
    var creatorName: String
    var rawData: [String: Any]
}

struct PollItem: Identifiable {
    var id: String
    var question: String
    var content: String
    var deadline: String
    var creatorUid: String
    var creatorName: String
    var allowAddOptions: Bool
    var options: [String]
}

struct GBItem: Identifiable {
    var id: String
    var title: String
    var deadline: String
    var creatorUid: String
    var creatorName: String
    var rawData: [String: Any]
}

struct ConfItem: Identifiable {
    var id: String
    var title: String
    var creatorUid: String
    var creatorName: String
    var rawData: [String: Any]
}

// MARK: - 3. 視圖模型與輔助引擎 (ViewModels & Helpers)
class ImageLoader: ObservableObject {
    @Published var image: UIImage?
    private let urlString: String

    init(url: String) {
        self.urlString = url
        loadImage()
    }

    private func loadImage() {
        guard let url = URL(string: urlString) else { return }
        let safeFileName = urlString.addingPercentEncoding(withAllowedCharacters: .alphanumerics) ?? UUID().uuidString
        let cachePath = FileManager.default.urls(for: .cachesDirectory, in: .userDomainMask)[0]
        let fileURL = cachePath.appendingPathComponent(safeFileName)

        if let data = try? Data(contentsOf: fileURL), let uiImage = UIImage(data: data) {
            self.image = uiImage
            return
        }

        URLSession.shared.dataTask(with: url) { data, response, error in
            guard let data = data, let uiImage = UIImage(data: data) else { return }
            try? data.write(to: fileURL)
            DispatchQueue.main.async {
                self.image = uiImage
            }
        }.resume()
    }
}

class AudioRecorder: NSObject, ObservableObject, AVAudioRecorderDelegate {
    var audioRecorder: AVAudioRecorder?
    @Published var isRecording = false
    var recordingURL: URL?

    func startRecording() {
        let session = AVAudioSession.sharedInstance()
        try? session.setCategory(.playAndRecord, mode: .default)
        try? session.setActive(true)

        let cachePath = FileManager.default.urls(for: .cachesDirectory, in: .userDomainMask)[0]
        let audioFilename = cachePath.appendingPathComponent(UUID().uuidString + ".m4a")
        recordingURL = audioFilename

        let settings = [
            AVFormatIDKey: Int(kAudioFormatMPEG4AAC),
            AVSampleRateKey: 12000,
            AVNumberOfChannelsKey: 1,
            AVEncoderAudioQualityKey: AVAudioQuality.high.rawValue
        ]

        do {
            audioRecorder = try AVAudioRecorder(url: audioFilename, settings: settings)
            audioRecorder?.delegate = self
            audioRecorder?.record()
            isRecording = true
        } catch {
            print("錄音失敗")
        }
    }

    func stopRecording() -> URL? {
        audioRecorder?.stop()
        isRecording = false
        return recordingURL
    }
}

class ChatViewModel: ObservableObject {
    @Published var chats: [ChatItem] = []
    private var db = Firestore.firestore()
    
    func fetchChats() {
        guard let currentUser = Auth.auth().currentUser else { return }
        let currentUid = currentUser.uid
        
        db.collection("chats")
            .whereField("members", arrayContains: currentUid)
            .order(by: "updatedAt", descending: true)
            .addSnapshotListener { querySnapshot, error in
                guard let documents = querySnapshot?.documents else { return }
                
                var fetchedChats = documents.compactMap { doc -> ChatItem? in
                    let data = doc.data()
                    let isGroup = data["isGroup"] as? Bool ?? false
                    let lastMessage = data["lastMessage"] as? String ?? "尚無訊息"
                    let lastSenderId = data["lastSenderId"] as? String ?? ""
                    
                    var timeStr = ""
                    var msgDate = Date()
                    if let timestamp = data["updatedAt"] as? Timestamp {
                        msgDate = timestamp.dateValue()
                        let formatter = DateFormatter()
                        formatter.dateFormat = "MM/dd"
                        timeStr = formatter.string(from: msgDate)
                    }
                    
                    var isUnread = false
                    if lastSenderId != currentUid {
                        let readDict = data["lastReadAt"] as? [String: Timestamp]
                        let myLastRead = readDict?[currentUid]?.dateValue() ?? Date(timeIntervalSince1970: 0)
                        if msgDate > myLastRead { isUnread = true }
                    }
                    
                    let name = isGroup ? (data["groupName"] as? String ?? "未命名群組") : "載入中..."
                    let avatarUrl = isGroup ? (data["groupAvatar"] as? String ?? "") : ""
                    
                    return ChatItem(id: doc.documentID, name: name, lastMessage: lastMessage, time: timeStr, isUnread: isUnread, isGroup: isGroup, avatarUrl: avatarUrl)
                }
                
                fetchedChats.sort { chat1, chat2 in
                    if chat1.isGroup && chat1.name == "立法院 Legislature" { return true }
                    if chat2.isGroup && chat2.name == "立法院 Legislature" { return false }
                    return false
                }
                
                self.chats = fetchedChats
                
                for (index, doc) in documents.enumerated() {
                    let data = doc.data()
                    let isGroup = data["isGroup"] as? Bool ?? false
                    if !isGroup {
                        let members = data["members"] as? [String] ?? []
                        let targetUid = members.first(where: { $0 != currentUid }) ?? ""
                        if !targetUid.isEmpty {
                            self.db.collection("act").document(targetUid).getDocument { userDoc, _ in
                                if let userData = userDoc?.data() {
                                    let realName = userData["displayName"] as? String ?? "未知成員"
                                    let realAvatar = userData["photoURL"] as? String ?? ""
                                    DispatchQueue.main.async {
                                        if let i = self.chats.firstIndex(where: { $0.id == doc.documentID }) {
                                            self.chats[i].name = realName
                                            self.chats[i].avatarUrl = realAvatar
                                        }
                                    }
                                }
                            }
                        }
                    }
                }
            }
    }
}

class ChatRoomViewModel: ObservableObject {
    @Published var messages: [MessageItem] = []
    @Published var isLoadingOldMessages = true
    
    @Published var groupReadTimes: [String: Date] = [:]
    @Published var groupMembersCount: Int = 0
    @Published var pinnedMessageText: String = ""
    
    @Published var myName: String = "自己"
    @Published var myAvatar: String = ""
    @Published var targetAvatar: String = ""
    @Published var isGroup: Bool = false
    
    private var db = Firestore.firestore()
    let chatId: String
    private var userCache: [String: (name: String, avatar: String)] = [:]
    
    var messageLimit = 20
    private var listener: ListenerRegistration?
    private var chatListener: ListenerRegistration?
    
    init(chatId: String) {
        self.chatId = chatId
        fetchMyProfile()
        fetchChatData()
        loadMessages()
    }
    
    deinit {
        listener?.remove()
        chatListener?.remove()
    }

    private func fetchMyProfile() {
        guard let uid = Auth.auth().currentUser?.uid else { return }
        db.collection("act").document(uid).getDocument { [weak self] doc, _ in
            if let data = doc?.data() {
                self?.myName = data["displayName"] as? String ?? "自己"
                self?.myAvatar = data["photoURL"] as? String ?? ""
            }
        }
    }

    func fetchUserInfo(uid: String) {
        if userCache[uid] != nil { return }
        db.collection("act").document(uid).getDocument { [weak self] doc, _ in
            if let data = doc?.data() {
                let name = data["displayName"] as? String ?? "未知"
                let avatar = data["photoURL"] as? String ?? ""
                self?.userCache[uid] = (name: name, avatar: avatar)
                self?.recalculateReadStatus()
                self?.updateMessagesUserInfo()
            }
        }
    }

    func fetchChatData() {
        guard let currentUid = Auth.auth().currentUser?.uid else { return }
        chatListener = db.collection("chats").document(chatId).addSnapshotListener { [weak self] doc, _ in
            guard let self = self, let data = doc?.data() else { return }
            
            self.isGroup = data["isGroup"] as? Bool ?? false
            self.pinnedMessageText = (data["pinnedMessage"] as? [String: Any])?["text"] as? String ?? ""
            
            let members = data["members"] as? [String] ?? []
            self.groupMembersCount = members.count
            
            if self.isGroup {
                self.targetAvatar = data["groupAvatar"] as? String ?? ""
            } else {
                if let otherUid = members.first(where: { $0 != currentUid }) {
                    self.fetchUserInfo(uid: otherUid)
                }
            }
            
            if let readDict = data["lastReadAt"] as? [String: Timestamp] {
                var newReadTimes: [String: Date] = [:]
                for (uid, timestamp) in readDict {
                    newReadTimes[uid] = timestamp.dateValue()
                    self.fetchUserInfo(uid: uid)
                }
                self.groupReadTimes = newReadTimes
                self.recalculateReadStatus()
            }
        }
    }

    func recalculateReadStatus() {
        var updatedMessages = self.messages
        for i in 0..<updatedMessages.count {
            let msg = updatedMessages[i]
            var readers: [ReadMember] = []
            
            for (uid, timestamp) in self.groupReadTimes {
                if uid != msg.senderId && timestamp >= msg.timestamp {
                    let name = self.userCache[uid]?.name ?? "未知"
                    let avatar = self.userCache[uid]?.avatar ?? ""
                    readers.append(ReadMember(id: uid, name: name, avatar: avatar))
                }
            }
            updatedMessages[i].readBy = readers
        }
        self.messages = updatedMessages
    }

    private func updateMessagesUserInfo() {
        var updatedMessages = self.messages
        for i in 0..<updatedMessages.count {
            let senderId = updatedMessages[i].senderId
            if let info = userCache[senderId] {
                updatedMessages[i].senderName = info.name
                updatedMessages[i].senderAvatar = info.avatar
            }
        }
        self.messages = updatedMessages
    }

    func loadMessages() {
        guard let currentUid = Auth.auth().currentUser?.uid else { return }
        listener = db.collection("chats").document(chatId).collection("messages")
            .order(by: "timestamp", descending: true)
            .limit(to: messageLimit)
            .addSnapshotListener { [weak self] snap, error in
                guard let self = self, let docs = snap?.documents else { return }
                
                var newMessages: [MessageItem] = []
                for doc in docs {
                    let data = doc.data()
                    let senderId = data["senderId"] as? String ?? ""
                    let timestamp = (data["timestamp"] as? Timestamp)?.dateValue() ?? Date()
                    let isMine = senderId == currentUid
                    
                    var senderName = ""
                    var senderAvatar = ""
                    
                    if isMine {
                        senderName = self.myName
                        senderAvatar = self.myAvatar
                    } else if let cached = self.userCache[senderId] {
                        senderName = cached.name
                        senderAvatar = cached.avatar
                    } else {
                        self.fetchUserInfo(uid: senderId)
                    }
                    
                    let msg = MessageItem(
                        id: doc.documentID,
                        text: data["text"] as? String ?? "",
                        senderId: senderId,
                        timestamp: timestamp,
                        isMine: isMine,
                        isSystem: data["isSystem"] as? Bool ?? false,
                        isDeleted: data["isDeleted"] as? Bool ?? false,
                        senderAvatar: senderAvatar,
                        senderName: senderName,
                        fileUrl: data["fileUrl"] as? String ?? "",
                        fileType: data["fileType"] as? String ?? "",
                        cardType: data["cardType"] as? String ?? "",
                        cardData: data["cardData"] as? [String: Any],
                        readBy: [],
                        replyToText: data["replyToText"] as? String ?? "",
                        reactions: data["reactions"] as? [String: [String]] ?? [:]
                    )
                    newMessages.append(msg)
                }
                
                self.messages = newMessages.reversed()
                self.recalculateReadStatus()
                self.isLoadingOldMessages = false
                self.markAsRead()
            }
    }

    func loadMore() {
        messageLimit += 20
        loadMessages()
    }

    func markAsRead() {
        guard let uid = Auth.auth().currentUser?.uid else { return }
        db.collection("chats").document(chatId).updateData([
            "lastReadAt.\(uid)": FieldValue.serverTimestamp()
        ])
    }

    func sendMessage(text: String, replyTo: String = "") {
        guard let uid = Auth.auth().currentUser?.uid else { return }
        let msgRef = db.collection("chats").document(chatId).collection("messages").document()
        
        var data: [String: Any] = [
            "text": text,
            "senderId": uid,
            "timestamp": FieldValue.serverTimestamp()
        ]
        if !replyTo.isEmpty { data["replyToText"] = replyTo }
        
        let batch = db.batch()
        batch.setData(data, forDocument: msgRef)
        batch.updateData([
            "lastMessage": text,
            "lastSenderId": uid,
            "updatedAt": FieldValue.serverTimestamp(),
            "lastReadAt.\(uid)": FieldValue.serverTimestamp()
        ], forDocument: db.collection("chats").document(chatId))
        batch.commit()
    }

    func uploadImage(_ image: UIImage) {
        guard let uid = Auth.auth().currentUser?.uid, let imageData = image.jpegData(compressionQuality: 0.5) else { return }
        let ref = Storage.storage().reference().child("chat_images/\(UUID().uuidString).jpg")
        ref.putData(imageData, metadata: nil) { [weak self] _, _ in
            ref.downloadURL { url, _ in
                if let urlString = url?.absoluteString {
                    self?.sendMediaMessage(fileUrl: urlString, fileType: "image", text: "[圖片]")
                }
            }
        }
    }

    func uploadAudio(_ url: URL) {
        guard let uid = Auth.auth().currentUser?.uid else { return }
        let ref = Storage.storage().reference().child("chat_audio/\(UUID().uuidString).m4a")
        ref.putFile(from: url, metadata: nil) { [weak self] _, _ in
            ref.downloadURL { downloadUrl, _ in
                if let urlString = downloadUrl?.absoluteString {
                    self?.sendMediaMessage(fileUrl: urlString, fileType: "audio", text: "[語音訊息]")
                }
            }
        }
    }

    private func sendMediaMessage(fileUrl: String, fileType: String, text: String) {
        guard let uid = Auth.auth().currentUser?.uid else { return }
        let msgRef = db.collection("chats").document(chatId).collection("messages").document()
        let batch = db.batch()
        batch.setData([
            "text": text,
            "senderId": uid,
            "timestamp": FieldValue.serverTimestamp(),
            "fileUrl": fileUrl,
            "fileType": fileType
        ], forDocument: msgRef)
        batch.updateData([
            "lastMessage": text,
            "lastSenderId": uid,
            "updatedAt": FieldValue.serverTimestamp(),
            "lastReadAt.\(uid)": FieldValue.serverTimestamp()
        ], forDocument: db.collection("chats").document(chatId))
        batch.commit()
    }

    func revokeMessage(messageId: String) {
        db.collection("chats").document(chatId).collection("messages").document(messageId).updateData([
            "isDeleted": true,
            "text": "此訊息已收回",
            "fileUrl": "",
            "fileType": "",
            "cardType": "",
            "cardData": FieldValue.delete()
        ])
    }

    func hideMessage(messageId: String) {
        if let index = messages.firstIndex(where: { $0.id == messageId }) {
            messages.remove(at: index)
        }
    }

    func pinMessage(text: String) {
        db.collection("chats").document(chatId).updateData([
            "pinnedMessage": ["text": text, "timestamp": FieldValue.serverTimestamp()]
        ])
    }

    func unpinMessage() {
        db.collection("chats").document(chatId).updateData([
            "pinnedMessage": FieldValue.delete()
        ])
    }

    func addReaction(messageId: String, emoji: String) {
        guard let uid = Auth.auth().currentUser?.uid else { return }
        let msgRef = db.collection("chats").document(chatId).collection("messages").document(messageId)
        
        db.runTransaction({ (transaction, errorPointer) -> Any? in
            let doc: DocumentSnapshot
            do { doc = try transaction.getDocument(msgRef) } catch let error as NSError { errorPointer?.pointee = error; return nil }
            guard let data = doc.data() else { return nil }
            
            var reactions = data["reactions"] as? [String: [String]] ?? [:]
            for (key, uids) in reactions { reactions[key] = uids.filter { $0 != uid } }
            
            if let currentUids = (data["reactions"] as? [String: [String]])?[emoji], currentUids.contains(uid) {
                // 取消反應
            } else {
                if reactions[emoji] == nil { reactions[emoji] = [] }
                reactions[emoji]?.append(uid)
            }
            
            transaction.updateData(["reactions": reactions], forDocument: msgRef)
            return nil
        }) { _, _ in }
    }

    func deleteChat(completion: @escaping (Bool) -> Void) {
        guard let uid = Auth.auth().currentUser?.uid else { return }
        let ref = db.collection("chats").document(chatId)
        
        ref.getDocument { [weak self] doc, _ in
            if let data = doc?.data(), let members = data["members"] as? [String] {
                let newMembers = members.filter { $0 != uid }
                if newMembers.isEmpty {
                    ref.delete { _ in completion(true) }
                } else {
                    ref.updateData(["members": newMembers]) { _ in completion(true) }
                }
            } else { completion(false) }
        }
    }
    
    func castVote(msgId: String, option: String) {
        guard let uid = Auth.auth().currentUser?.uid else { return }
        let msgRef = db.collection("chats").document(chatId).collection("messages").document(msgId)
        
        db.runTransaction({ (transaction, errorPointer) -> Any? in
            let doc: DocumentSnapshot
            do { doc = try transaction.getDocument(msgRef) } catch let error as NSError { errorPointer?.pointee = error; return nil }
            guard let data = doc.data(), let rawText = data["text"] as? String else { return nil }
            
            do {
                if var parsed = try JSONSerialization.jsonObject(with: Data(rawText.utf8), options: []) as? [String: Any],
                   var votes = parsed["votes"] as? [String: [String]] {
                    
                    if let deadlineStr = parsed["deadline"] as? String, !deadlineStr.isEmpty {
                        let formatter = DateFormatter(); formatter.dateFormat = "yyyy-MM-dd'T'HH:mm"
                        if let d = formatter.date(from: deadlineStr), Date() > d { return nil }
                    }
                    
                    for (key, uids) in votes { votes[key] = uids.filter { $0 != uid } }
                    
                    if let originalUids = (try? JSONSerialization.jsonObject(with: Data(rawText.utf8), options: []) as? [String: Any])?["votes"] as? [String: [String]],
                       let targetUids = originalUids[option], !targetUids.contains(uid) {
                        if votes[option] == nil { votes[option] = [] }
                        votes[option]?.append(uid)
                    }
                    
                    parsed["votes"] = votes
                    let newData = try JSONSerialization.data(withJSONObject: parsed, options: [])
                    if let newText = String(data: newData, encoding: .utf8) { transaction.updateData(["text": newText], forDocument: msgRef) }
                }
            } catch let e as NSError { errorPointer?.pointee = e; return nil }
            return nil
        }) { _, _ in }
    }
}

// 輔助函式：解析舊版 Array / String 資料
func parseOldArrayData(from value: Any?) -> [String] {
    if let arr = value as? [String] { return arr }
    if let str = value as? String {
        if str.hasPrefix("[") {
            if let data = str.data(using: .utf8), let arr = try? JSONSerialization.jsonObject(with: data) as? [String] { return arr }
        }
        return str.split(separator: ",").map { String($0).trimmingCharacters(in: .whitespacesAndNewlines) }.filter { !$0.isEmpty }
    }
    return []
}

// 輔助函式：判斷是否有權限編輯
func canEdit(creatorUid: String, myName: String) -> Bool {
    return creatorUid == Auth.auth().currentUser?.uid || myName == "班長"
}

// MARK: - 4. 共用 UI 元件與擴充 (Shared Components & Extensions)
extension View {
    func cornerRadius(_ radius: CGFloat, corners: UIRectCorner) -> some View {
        clipShape(RoundedCorner(radius: radius, corners: corners))
    }
}

struct RoundedCorner: Shape {
    var radius: CGFloat = .infinity
    var corners: UIRectCorner = .allCorners
    func path(in rect: CGRect) -> Path {
        return Path(UIBezierPath(roundedRect: rect, byRoundingCorners: corners, cornerRadii: CGSize(width: radius, height: radius)).cgPath)
    }
}

struct CameraImagePicker: UIViewControllerRepresentable {
    @Binding var image: UIImage?
    @Environment(\.presentationMode) var presentationMode
    
    func makeUIViewController(context: Context) -> UIImagePickerController {
        let picker = UIImagePickerController()
        if UIImagePickerController.isSourceTypeAvailable(.camera) {
            picker.sourceType = .camera
        } else {
            picker.sourceType = .photoLibrary
        }
        picker.delegate = context.coordinator
        return picker
    }
    
    func updateUIViewController(_ uiViewController: UIImagePickerController, context: Context) {}
    func makeCoordinator() -> Coordinator { Coordinator(self) }
    
    class Coordinator: NSObject, UINavigationControllerDelegate, UIImagePickerControllerDelegate {
        let parent: CameraImagePicker
        init(_ parent: CameraImagePicker) { self.parent = parent }
        func imagePickerController(_ picker: UIImagePickerController, didFinishPickingMediaWithInfo info: [UIImagePickerController.InfoKey : Any]) {
            if let uiImage = info[.originalImage] as? UIImage { parent.image = uiImage }
            parent.presentationMode.wrappedValue.dismiss()
        }
        func imagePickerControllerDidCancel(_ picker: UIImagePickerController) {
            parent.presentationMode.wrappedValue.dismiss()
        }
    }
}

struct GIFView: UIViewRepresentable {
    let urlString: String
    var allowZoom: Bool = false
    func makeUIView(context: Context) -> WKWebView {
        let prefs = WKWebpagePreferences()
        let config = WKWebViewConfiguration()
        config.defaultWebpagePreferences = prefs
        let webView = WKWebView(frame: .zero, configuration: config)
        webView.isOpaque = false
        webView.backgroundColor = .clear
        webView.scrollView.isScrollEnabled = allowZoom
        webView.isUserInteractionEnabled = allowZoom
        return webView
    }
    func updateUIView(_ uiView: WKWebView, context: Context) {
        let html = """
        <!DOCTYPE html><html><head><meta name="viewport" content="width=device-width, initial-scale=1, maximum-scale=5, user-scalable=yes"></head>
        <body style="margin:0; padding:0; background:transparent; display:flex; justify-content:center; align-items:center; height:100vh; overflow:hidden;">
            <img src="\(urlString)" style="width:100%; height:100%; object-fit:contain;">
        </body></html>
        """
        uiView.loadHTMLString(html, baseURL: nil)
    }
}

struct CachedAvatarView: View {
    let url: String
    let size: CGFloat
    @StateObject private var loader: ImageLoader
    
    init(url: String, size: CGFloat = 55) {
        self.url = url
        self.size = size
        _loader = StateObject(wrappedValue: ImageLoader(url: url))
    }
    
    var body: some View {
        if let image = loader.image {
            Image(uiImage: image)
                .resizable()
                .scaledToFill()
                .frame(width: size, height: size)
                .clipShape(Circle())
        } else {
            ZStack {
                Circle().fill(Color(UIColor.systemGray5))
                    .frame(width: size, height: size)
                ProgressView()
            }
        }
    }
}

struct LiveAvatarView: View {
    let uidOrName: String
    var showName: Bool = true
    var size: CGFloat = 50
    @State private var displayName: String = ""
    @State private var avatar: String = ""
    let lyGold = Color(red: 184/255, green: 134/255, blue: 11/255)

    var body: some View {
        VStack(spacing: 4) {
            if avatar.isEmpty {
                ZStack {
                    Circle().fill(Color(UIColor.systemGray5)).frame(width: size, height: size).overlay(Circle().stroke(Color.black, lineWidth: 2)).overlay(Circle().stroke(lyGold, lineWidth: 3))
                    if !displayName.isEmpty { Text(String(displayName.prefix(1))).font(.system(size: size * 0.4, weight: .bold)).foregroundColor(.gray) }
                }
            } else {
                CachedAvatarView(url: avatar, size: size).clipShape(Circle()).overlay(Circle().stroke(Color.black, lineWidth: 2)).overlay(Circle().stroke(lyGold, lineWidth: 3))
            }
            if showName { Text(displayName).font(.caption).foregroundColor(.gray).lineLimit(1) }
        }
        .onAppear {
            self.displayName = uidOrName
            if uidOrName.count > 20 { Firestore.firestore().collection("act").document(uidOrName).getDocument { doc, _ in if let data = doc?.data() { self.displayName = data["displayName"] as? String ?? "未知"; self.avatar = data["photoURL"] as? String ?? "" } } }
        }
    }
}

struct LiveUserRow: View {
    let uid: String
    @State private var name: String = "載入中..."
    @State private var avatar: String = ""
    var body: some View {
        HStack(spacing: 15) {
            if avatar.isEmpty {
                Circle().fill(Color.gray.opacity(0.3)).frame(width: 40, height: 40)
            } else {
                CachedAvatarView(url: avatar, size: 40).clipShape(Circle())
            }
            Text(name).font(.headline).foregroundColor(.primary)
            Spacer()
        }
        .onAppear {
            Firestore.firestore().collection("act").document(uid).getDocument { doc, _ in
                if let data = doc?.data() {
                    self.name = data["displayName"] as? String ?? "未知"
                    self.avatar = data["photoURL"] as? String ?? ""
                }
            }
        }
    }
}

struct AudioBubbleView: View {
    let audioUrl: String
    let isMine: Bool
    @State private var player: AVPlayer?
    @State private var isPlaying = false

    var body: some View {
        HStack {
            Button(action: togglePlay) {
                Image(systemName: isPlaying ? "pause.circle.fill" : "play.circle.fill")
                    .font(.title)
                    .foregroundColor(isMine ? .white : .primary)
            }
            Text("語音訊息")
                .font(.subheadline)
                .foregroundColor(isMine ? .white : .primary)
        }
        .padding(.horizontal, 16)
        .padding(.vertical, 12)
        .background(isMine ? Color(red: 0/255, green: 49/255, blue: 83/255) : Color(UIColor.systemGray5))
        .cornerRadius(20)
        .onDisappear {
            player?.pause()
        }
    }

    func togglePlay() {
        if player == nil {
            guard let url = URL(string: audioUrl) else { return }
            player = AVPlayer(url: url)
            NotificationCenter.default.addObserver(forName: .AVPlayerItemDidPlayToEndTime, object: player?.currentItem, queue: .main) { _ in
                isPlaying = false
                player?.seek(to: .zero)
            }
        }
        if isPlaying {
            player?.pause()
            isPlaying = false
        } else {
            player?.play()
            isPlaying = true
        }
    }
}

struct GenericListRow: View {
    let title: String
    let subtitle: String
    let creator: String
    let icon: String
    let color: Color
    let action: () -> Void
    
    var body: some View {
        Button(action: action) {
            HStack(spacing: 0) {
                Rectangle().fill(color).frame(width: 6)
                VStack(alignment: .leading, spacing: 10) {
                    HStack {
                        Image(systemName: icon)
                        Text(title).font(.title3).fontWeight(.bold)
                    }.foregroundColor(color)
                    
                    HStack(alignment: .bottom) {
                        HStack(spacing: 4) {
                            Image(systemName: "clock")
                            Text(subtitle)
                        }.font(.caption).foregroundColor(subtitle.contains("無") ? .gray : .red)
                        Spacer()
                        Text("建立人：\(creator)").font(.caption).fontWeight(.bold).foregroundColor(color).padding(.horizontal, 10).padding(.vertical, 4).background(Color.gray.opacity(0.1)).cornerRadius(12)
                    }
                }.padding(.vertical, 12).padding(.horizontal, 15)
            }.background(Color.white).cornerRadius(12).overlay(RoundedRectangle(cornerRadius: 12).stroke(Color.gray.opacity(0.2), lineWidth: 1))
        }
        .buttonStyle(PlainButtonStyle())
        .listRowInsets(EdgeInsets(top: 6, leading: 16, bottom: 6, trailing: 16))
        .listRowSeparator(.hidden)
        .listRowBackground(Color.clear)
    }
}

struct PrivilegeBanner: View {
    let myName: String
    let color: Color
    
    var body: some View {
        HStack(alignment: .top, spacing: 10) {
            Image(systemName: "lightbulb.fill").foregroundColor(color).font(.title3)
            if myName == "班長" {
                Text("班長特權：").fontWeight(.bold).foregroundColor(color) + Text("您擁有所有項目的「編輯」與「刪除」權限，向左滑動即可操作。").foregroundColor(.gray)
            } else {
                Text("小提示：").fontWeight(.bold).foregroundColor(color) + Text("自己發起的功能，向左滑動即可「編輯」或「刪除」。").foregroundColor(.gray)
            }
        }
        .padding(.vertical, 8)
        .listRowBackground(Color(red: 255/255, green: 253/255, blue: 245/255))
    }
}

// MARK: - 5. 主流程與導覽 (Main Flow)
struct ContentView: View {
    @State private var email = ""
    @State private var password = ""
    @State private var errorMessage = ""
    @State private var isLoading = false
    @State private var isLoggedIn = false
    
    let lyBlue = Color(red: 0/255, green: 49/255, blue: 83/255)
    let lyGold = Color(red: 184/255, green: 134/255, blue: 11/255)

    var body: some View {
        if isLoggedIn {
             MainTabView(isLoggedIn: $isLoggedIn)
         } else {
            loginForm
        }
    }
    
    var loginForm: some View {
        VStack(spacing: 30) {
            Spacer()
            
            VStack(spacing: 8) {
                Image(systemName: "building.columns.fill")
                    .font(.system(size: 45))
                    .foregroundColor(lyBlue)
                Text("立法院")
                    .font(.system(size: 32, weight: .bold))
                    .foregroundColor(lyBlue)
                    .tracking(2)
                Rectangle().fill(lyGold).frame(width: 40, height: 4).padding(.top, 5)
            }
            
            Text("ADMINISTRATION SYSTEM")
                .font(.caption)
                .foregroundColor(.gray)
                .tracking(1)
            
            VStack(alignment: .leading, spacing: 20) {
                VStack(alignment: .leading, spacing: 8) {
                    Text("帳號 (Email)").font(.footnote).fontWeight(.semibold).foregroundColor(.secondary)
                    HStack {
                        Image(systemName: "envelope.fill").foregroundColor(.gray)
                        TextField("請輸入公務帳號", text: $email)
                            .keyboardType(.emailAddress)
                            .autocapitalization(.none)
                    }
                    .padding().background(Color(UIColor.systemGray6)).cornerRadius(8)
                }
                
                VStack(alignment: .leading, spacing: 8) {
                    Text("密碼 (Password)").font(.footnote).fontWeight(.semibold).foregroundColor(.secondary)
                    HStack {
                        Image(systemName: "lock.fill").foregroundColor(.gray)
                        SecureField("請輸入密碼", text: $password)
                    }
                    .padding().background(Color(UIColor.systemGray6)).cornerRadius(8)
                }
            }
            .padding(.horizontal, 10)
            
            if !errorMessage.isEmpty {
                Text(errorMessage).foregroundColor(.red).font(.footnote)
            }
            
            Button(action: loginWithFirebase) {
                HStack {
                    if isLoading {
                        ProgressView().progressViewStyle(CircularProgressViewStyle(tint: .white))
                    } else {
                        Text("進入管理系統")
                    }
                }
                .font(.system(size: 18, weight: .bold))
                .foregroundColor(.white)
                .frame(maxWidth: .infinity)
                .padding()
                .background(lyBlue)
                .cornerRadius(8)
            }
            .disabled(isLoading)
            .padding(.top, 10)
            
            Spacer()
        }
        .padding(30)
        .background(Color(UIColor.systemGroupedBackground).ignoresSafeArea())
    }
    
    func loginWithFirebase() {
        guard !email.isEmpty, !password.isEmpty else {
            errorMessage = "請完整填寫帳號與密碼"
            return
        }
        isLoading = true; errorMessage = ""
        Auth.auth().signIn(withEmail: email, password: password) { result, error in
            isLoading = false
            if let error = error {
                errorMessage = "登入失敗: \(error.localizedDescription)"
            } else if let _ = result?.user {
                withAnimation { isLoggedIn = true }
            }
        }
    }
}

struct MainTabView: View {
    @Binding var isLoggedIn: Bool
    
    @State private var selectedTab = 1
    @State private var myAvatarUrl = Auth.auth().currentUser?.photoURL?.absoluteString ?? ""
    @State private var myName = "自己"
    
    @State private var isTabBarHidden = false
    let lyBlue = Color(red: 0/255, green: 49/255, blue: 83/255)
    let lyGold = Color(red: 184/255, green: 134/255, blue: 11/255)

    var body: some View {
        ZStack(alignment: .bottom) {
            Group {
                switch selectedTab {
                case 0: Text("首頁 (準備開發)").frame(maxWidth: .infinity, maxHeight: .infinity).background(Color(UIColor.systemGroupedBackground))
                case 1: ChatListView()
                case 2: ProfileView(isLoggedIn: $isLoggedIn)
                case 3: Text("日曆 (CalendarView)").font(.title).frame(maxWidth: .infinity, maxHeight: .infinity).background(Color(UIColor.systemGroupedBackground))
                case 4: Text("其他 (準備開發)").frame(maxWidth: .infinity, maxHeight: .infinity).background(Color(UIColor.systemGroupedBackground))
                default: EmptyView()
                }
            }.frame(maxWidth: .infinity, maxHeight: .infinity)
            
            if !isTabBarHidden {
                HStack(spacing: 0) {
                    tabButton(icon: "house.fill", title: "首頁", index: 0)
                    tabButton(icon: "message.fill", title: "聊天", index: 1)
                    Spacer().frame(width: 80)
                    tabButton(icon: "calendar", title: "日曆", index: 3)
                    tabButton(icon: "ellipsis", title: "其他", index: 4)
                }
                .frame(height: 55).padding(.horizontal, 10)
                .background(Color.white.shadow(color: Color.black.opacity(0.08), radius: 3, x: 0, y: -3).ignoresSafeArea(.all, edges: .bottom))
                .overlay(
                    Button(action: { selectedTab = 2 }) {
                        VStack(spacing: 4) {
                            ZStack {
                                Circle().fill(Color.white).frame(width: 65, height: 65).shadow(color: Color.black.opacity(0.15), radius: 5, x: 0, y: -2)
                                if myAvatarUrl.isEmpty { Image(systemName: "person.crop.circle.fill").resizable().frame(width: 55, height: 55).foregroundColor(lyBlue) }
                                else { CachedAvatarView(url: myAvatarUrl, size: 55).clipShape(Circle()) }
                            }
                            Text(myName).font(.system(size: 10, weight: .black)).foregroundColor(selectedTab == 2 ? lyGold : lyBlue).lineLimit(1)
                        }
                    }.offset(y: -20), alignment: .top
                ).transition(.move(edge: .bottom).combined(with: .opacity))
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
            if let data = doc?.data() { self.myName = data["displayName"] as? String ?? "未知"; self.myAvatarUrl = data["photoURL"] as? String ?? "" }
        }
    }
    
    func tabButton(icon: String, title: String, index: Int) -> some View {
        Button(action: { selectedTab = index }) {
            VStack(spacing: 5) {
                Image(systemName: icon).font(.system(size: 22))
                Text(title).font(.system(size: 10, weight: .bold))
            }.foregroundColor(selectedTab == index ? lyBlue : .gray.opacity(0.5)).frame(maxWidth: .infinity)
        }
    }
}

struct ProfileView: View {
    @Binding var isLoggedIn: Bool
    @State private var displayName: String = Auth.auth().currentUser?.displayName ?? "未設定名稱"
    @State private var avatarUrl: String = Auth.auth().currentUser?.photoURL?.absoluteString ?? ""
    @State private var isUploading = false
    @State private var selectedPhoto: PhotosPickerItem?
    
    @State private var sessions: [DeviceSession] = [
        DeviceSession(id: "1", ip: "211.75.XX.XX", location: "台灣, 台北市", lastLogin: "剛剛", isCurrent: true),
        DeviceSession(id: "2", ip: "114.32.XX.XX", location: "台灣, 新北市", lastLogin: "2026/04/10 14:20", isCurrent: false)
    ]
    
    let lyBlue = Color(red: 0/255, green: 49/255, blue: 83/255)
    let lyGold = Color(red: 184/255, green: 134/255, blue: 11/255)
    
    var body: some View {
        NavigationStack {
            ScrollView {
                VStack(spacing: 30) {
                    VStack(spacing: 15) {
                        PhotosPicker(selection: $selectedPhoto, matching: .images, photoLibrary: .shared()) {
                            ZStack(alignment: .bottomTrailing) {
                                if isUploading {
                                    Circle().fill(Color.gray.opacity(0.3)).frame(width: 110, height: 110)
                                    ProgressView().tint(.white)
                                } else if avatarUrl.isEmpty {
                                    Image(systemName: "person.crop.circle.fill").resizable().frame(width: 110, height: 110).foregroundColor(lyBlue)
                                } else {
                                    CachedAvatarView(url: avatarUrl, size: 110).clipShape(Circle())
                                }
                                Image(systemName: "camera.circle.fill").font(.title).foregroundColor(lyGold).background(Circle().fill(Color.white)).offset(x: -5, y: -5)
                            }
                        }
                        .onChange(of: selectedPhoto) { newItem in Task { if let data = try? await newItem?.loadTransferable(type: Data.self), let uiImage = UIImage(data: data) { uploadAvatar(image: uiImage) } } }
                        
                        TextField("編輯顯示名稱", text: $displayName).font(.title2).fontWeight(.bold).foregroundColor(lyBlue).multilineTextAlignment(.center).onSubmit { updateDisplayName() }
                        Text(Auth.auth().currentUser?.email ?? "信箱載入中...").font(.subheadline).foregroundColor(.gray)
                    }.padding(.top, 30)
                    
                    VStack(alignment: .leading, spacing: 15) {
                        HStack { Image(systemName: "shield.checkerboard").foregroundColor(lyGold); Text("帳號安全性與登入紀錄").font(.headline).foregroundColor(lyBlue) }
                        VStack(spacing: 12) {
                            ForEach(sessions) { session in
                                HStack {
                                    VStack(alignment: .leading, spacing: 6) {
                                        HStack {
                                            Image(systemName: session.isCurrent ? "iphone.radiowaves.left.and.right" : "desktopcomputer").foregroundColor(session.isCurrent ? .green : .gray)
                                            Text(session.location).fontWeight(.bold).foregroundColor(.primary)
                                            if session.isCurrent { Text("目前裝置").font(.caption2).padding(.horizontal, 6).padding(.vertical, 2).background(Color.green.opacity(0.2)).foregroundColor(.green).cornerRadius(4) }
                                        }
                                        Text("IP位址：\(session.ip)").font(.caption).foregroundColor(.gray)
                                        Text("最後活動：\(session.lastLogin)").font(.caption).foregroundColor(.gray)
                                    }
                                    Spacer()
                                    if !session.isCurrent {
                                        Button(action: { kickDevice(id: session.id) }) { Text("強制登出").font(.caption).fontWeight(.bold).foregroundColor(.red).padding(.horizontal, 10).padding(.vertical, 6).background(Color.red.opacity(0.1)).cornerRadius(8) }
                                    }
                                }.padding().background(Color.white).cornerRadius(10).overlay(RoundedRectangle(cornerRadius: 10).stroke(Color.gray.opacity(0.2), lineWidth: 1))
                            }
                        }
                    }.padding(.horizontal, 20)
                    
                    Spacer(minLength: 40)
                    
                    Button(action: { try? Auth.auth().signOut(); isLoggedIn = false }) {
                        HStack { Image(systemName: "signpost.right.fill"); Text("登出目前裝置") }.font(.headline).foregroundColor(.white).frame(maxWidth: .infinity).padding().background(Color.red).cornerRadius(10)
                    }.padding(.horizontal, 20).padding(.bottom, 40)
                }
            }.background(Color(UIColor.systemGroupedBackground).ignoresSafeArea()).navigationTitle("關於我").navigationBarTitleDisplayMode(.inline)
        }
    }
    
    private func kickDevice(id: String) { withAnimation { sessions.removeAll { $0.id == id } }; print("已執行遠端踢除") }
    private func updateDisplayName() { guard let user = Auth.auth().currentUser else { return }; let changeRequest = user.createProfileChangeRequest(); changeRequest.displayName = displayName; changeRequest.commitChanges { error in if error == nil { Firestore.firestore().collection("act").document(user.uid).setData(["displayName": displayName], merge: true) } } }
    private func uploadAvatar(image: UIImage) { guard let uid = Auth.auth().currentUser?.uid, let imageData = image.jpegData(compressionQuality: 0.3) else { return }; isUploading = true; let ref = Storage.storage().reference().child("avatars/\(uid).jpg"); ref.putData(imageData, metadata: nil) { _, error in if error == nil { ref.downloadURL { url, _ in if let newUrl = url?.absoluteString { let changeRequest = Auth.auth().currentUser?.createProfileChangeRequest(); changeRequest?.photoURL = url; changeRequest?.commitChanges { _ in }; Firestore.firestore().collection("act").document(uid).setData(["photoURL": newUrl], merge: true); DispatchQueue.main.async { self.avatarUrl = newUrl; self.isUploading = false } } } } else { isUploading = false } } }
}

// 👉 修正 3：還原乾淨俐落的列表設計 (截圖 5.04.27)
struct ChatListView: View {
    let lyBlue = Color(red: 0/255, green: 49/255, blue: 83/255)
    let lyGold = Color(red: 184/255, green: 134/255, blue: 11/255)
    @StateObject private var viewModel = ChatViewModel()

    var body: some View {
        NavigationStack {
            List(viewModel.chats) { chat in
                ZStack {
                    NavigationLink(destination: ChatRoomView(chatId: chat.id, chatName: chat.name)) { EmptyView() }.opacity(0)
                    
                    HStack(spacing: 15) {
                        ZStack(alignment: .topLeading) {
                            if chat.avatarUrl.isEmpty { ZStack { Circle().fill(Color(UIColor.systemGray5)).frame(width: 55, height: 55); Image(systemName: chat.isGroup ? "person.3.fill" : "person.fill").foregroundColor(.gray) } }
                            else { CachedAvatarView(url: chat.avatarUrl, size: 55).clipShape(Circle()) }
                            
                            if chat.name == "立法院 Legislature" {
                                Image(systemName: "pin.fill").foregroundColor(lyGold).rotationEffect(.degrees(-45)).offset(x: -5, y: -5)
                            }
                        }
                        VStack(alignment: .leading, spacing: 6) {
                            HStack(alignment: .top) {
                                Text(chat.name)
                                    .font(.system(size: 17, weight: .bold))
                                    .foregroundColor(.primary)
                                Spacer()
                                Text(chat.time)
                                    .font(.system(size: 13))
                                    .foregroundColor(.gray)
                            }
                            Text(chat.lastMessage)
                                .font(.system(size: 14))
                                .foregroundColor(.gray)
                                .lineLimit(1)
                        }
                    }
                    .padding(.vertical, 4)
                }
                .listRowBackground(Color.white) // 統一為乾淨的白底
            }
            .listStyle(PlainListStyle()) // 使用原生的 Plain 樣式產生分隔線
            .navigationTitle("訊息中心")
            .navigationBarTitleDisplayMode(.inline)
            .toolbar {
                ToolbarItem(placement: .navigationBarTrailing) {
                    HStack(spacing: 16) {
                        Button(action: { }) { Image(systemName: "person.2.fill") }
                        Button(action: { print("準備開啟發起對話的畫面") }) { Image(systemName: "plus") }
                    }
                    .font(.system(size: 16, weight: .bold))
                    .foregroundColor(lyBlue)
                    .padding(.horizontal, 14)
                    .padding(.vertical, 8)
                    .background(Color.white)
                    .cornerRadius(20)
                    .shadow(color: Color.black.opacity(0.08), radius: 4, x: 0, y: 2)
                }
            }
            .onAppear { viewModel.fetchChats(); NotificationCenter.default.post(name: NSNotification.Name("ShowTabBar"), object: nil) }
        }
    }
}

// MARK: - 6. 聊天室核心 (Chat Room)
struct ChatRoomView: View {
    let chatName: String
    @StateObject private var viewModel: ChatRoomViewModel
    
    @State private var inputText = ""
    @State private var isMenuExpanded = true
    
    @State private var isUIReady = false
    @State private var hasInitialScrolled = false
    @State private var scrollTrigger = UUID()
    @State private var selectedPhoto: PhotosPickerItem?
    @StateObject private var audioRecorder = AudioRecorder()
    
    @State private var showReadList = false
    @State private var currentReadMembers: [ReadMember] = []
    @State private var menuMessage: MessageItem? = nil
    @State private var showCamera = false
    @State private var cameraImage: UIImage?
    @State private var showAttachmentMenu = false
    @State private var replyingToText: String? = nil
    @State private var showPinnedModal = false
    @State private var showReplyDetailModal = false
    @State private var popupDetailText = ""
    @State private var showFullScreenMedia = false
    @State private var fullScreenMediaUrl = ""
    @State private var fullScreenMediaIsGif = false
    
    let lyBlue = Color(red: 0/255, green: 49/255, blue: 83/255)
    let lyGold = Color(red: 184/255, green: 134/255, blue: 11/255)
    let timeFormatter: DateFormatter = { let f = DateFormatter(); f.dateFormat = "HH:mm"; return f }()
    let dateFormatter: DateFormatter = { let f = DateFormatter(); f.dateFormat = "yyyy-MM-dd (E)"; f.locale = Locale(identifier: "zh_Hant_TW"); return f }()
    
    init(chatId: String, chatName: String) {
        self.chatName = chatName
        _viewModel = StateObject(wrappedValue: ChatRoomViewModel(chatId: chatId))
    }
    
    var body: some View {
        ZStack {
            VStack(spacing: 0) {
                pinnedBannerView
                messageScrollView
                replyPreviewBanner
                bottomInputBar
            }
            .navigationTitle(chatName)
            .navigationBarTitleDisplayMode(.inline)
            .toolbar(.hidden, for: .tabBar)
            .toolbar { ToolbarItem(placement: .navigationBarTrailing) { NavigationLink(destination: ChatSettingsView(chatName: chatName, viewModel: viewModel)) { Image(systemName: "line.3.horizontal").foregroundColor(.gray) } } }
            .fullScreenCover(isPresented: $showCamera) { CameraImagePicker(image: $cameraImage).ignoresSafeArea() }
            .onChange(of: cameraImage) { newImage in if let img = newImage { viewModel.uploadImage(img); cameraImage = nil } }

            attachmentMenuOverlay
            modalsAndOverlays
            
            if showFullScreenMedia { FullScreenMediaViewer(urlString: fullScreenMediaUrl, isGif: fullScreenMediaIsGif, isPresented: $showFullScreenMedia).zIndex(100).transition(.opacity) }
        }
        .onAppear { NotificationCenter.default.post(name: NSNotification.Name("HideTabBar"), object: nil) }
    }
    
    private func shouldShowDateDivider(for msg: MessageItem, in messages: [MessageItem]) -> Bool { guard let index = messages.firstIndex(where: { $0.id == msg.id }) else { return false }; if index == 0 { return true }; let previousMsg = messages[index - 1]; return !Calendar.current.isDate(msg.timestamp, inSameDayAs: previousMsg.timestamp) }
    
    @ViewBuilder private var pinnedBannerView: some View {
        if !viewModel.pinnedMessageText.isEmpty {
            HStack {
                Image(systemName: "pin.fill").foregroundColor(lyGold).rotationEffect(.degrees(45))
                if isImageUrl(viewModel.pinnedMessageText) {
                    if isGifUrl(viewModel.pinnedMessageText) { GIFView(urlString: viewModel.pinnedMessageText).frame(width: 30, height: 30).cornerRadius(4).clipped() }
                    else { AsyncImage(url: URL(string: viewModel.pinnedMessageText)) { phase in if let image = phase.image { image.resizable().scaledToFill().frame(width: 30, height: 30).cornerRadius(4).clipped() } else { Rectangle().fill(Color.gray.opacity(0.3)).frame(width: 30, height: 30).cornerRadius(4) } } }
                    Text("圖片訊息").font(.subheadline).foregroundColor(.primary).lineLimit(1)
                } else { Text(viewModel.pinnedMessageText).font(.subheadline).foregroundColor(.primary).lineLimit(1) }
                Spacer(); Button(action: { viewModel.unpinMessage() }) { Image(systemName: "xmark").foregroundColor(.gray).padding(5) }
            }.padding(.horizontal).padding(.vertical, 8).background(Color.white.opacity(0.95)).shadow(color: Color.black.opacity(0.05), radius: 3, x: 0, y: 2).onTapGesture { popupDetailText = viewModel.pinnedMessageText; showPinnedModal = true }.zIndex(10)
        }
    }
    
    private var messageScrollView: some View {
        ScrollViewReader { proxy in
            ScrollView {
                LazyVStack(spacing: 2) {
                    if viewModel.isLoadingOldMessages && viewModel.messages.count >= 10 { ProgressView().padding().onAppear { viewModel.loadMore() } }
                    ForEach(viewModel.messages) { msg in
                        VStack(spacing: 12) {
                            if shouldShowDateDivider(for: msg, in: viewModel.messages) { Text(dateFormatter.string(from: msg.timestamp)).font(.system(size: 12, weight: .bold)).foregroundColor(.gray).padding(.horizontal, 14).padding(.vertical, 6).background(Color.gray.opacity(0.15)).cornerRadius(12).padding(.vertical, 10) }
                            if msg.isSystem || msg.isDeleted { Text(msg.isDeleted ? "此訊息已收回" : msg.text).font(.system(size: 11, weight: .bold)).foregroundColor(.white).padding(.horizontal, 12).padding(.vertical, 6).background(Color.gray.opacity(0.5)).cornerRadius(12).padding(.vertical, 8) }
                            else { messageRow(msg: msg) }
                        }.id(msg.id)
                    }
                    // 👉 修正 1：加大底部留白空間，讓捲動到底部時訊息不會被輸入框擋住
                    Color.clear.frame(height: 20).id("BOTTOM_ANCHOR")
                }
                .padding(.vertical, 8)
            }.background(Color.white).opacity(isUIReady ? 1 : 0)
            .onAppear { DispatchQueue.main.asyncAfter(deadline: .now() + 0.15) { if let lastId = viewModel.messages.last?.id { proxy.scrollTo(lastId, anchor: .bottom) }; withAnimation(.easeIn(duration: 0.2)) { isUIReady = true; hasInitialScrolled = true } } }
            .onChange(of: viewModel.messages.count) { _ in if isUIReady { DispatchQueue.main.asyncAfter(deadline: .now() + 0.1) { withAnimation(.easeOut(duration: 0.25)) { proxy.scrollTo("BOTTOM_ANCHOR", anchor: .bottom) } } } }
            .onChange(of: scrollTrigger) { _ in withAnimation(.easeOut(duration: 0.25)) { proxy.scrollTo("BOTTOM_ANCHOR", anchor: .bottom) } }
        }
    }
    
    @ViewBuilder private func messageRow(msg: MessageItem) -> some View {
        HStack(alignment: .top, spacing: 10) {
            if !msg.isMine {
                VStack(spacing: 6) {
                    if msg.senderAvatar.isEmpty { Circle().fill(Color(UIColor.systemGray4)).frame(width: 42, height: 42) } else { CachedAvatarView(url: msg.senderAvatar, size: 42) }
                    if !msg.senderName.isEmpty && viewModel.isGroup { Text(msg.senderName).font(.system(size: 11)).foregroundColor(.gray).frame(width: 45).lineLimit(1) }
                }
                VStack(alignment: .leading, spacing: 4) { bubbleContent(msg: msg); readStatusView(msg: msg) }
                Spacer(minLength: 40)
            } else {
                Spacer(minLength: 40)
                VStack(alignment: .trailing, spacing: 4) { bubbleContent(msg: msg); readStatusView(msg: msg) }
            }
        }.padding(.horizontal).padding(.top, 4)
    }
    
    @ViewBuilder private var replyPreviewBanner: some View {
        if let replyText = replyingToText { HStack { Image(systemName: "arrowshape.turn.up.left.fill").foregroundColor(lyBlue); Text("回覆：\(replyText)").font(.caption).foregroundColor(.gray).lineLimit(1); Spacer(); Button(action: { withAnimation { replyingToText = nil } }) { Image(systemName: "xmark.circle.fill").foregroundColor(.gray) } }.padding(.horizontal).padding(.vertical, 8).background(Color(UIColor.systemGray5)).transition(.move(edge: .bottom).combined(with: .opacity)) }
    }
    
    private var bottomInputBar: some View {
        HStack(alignment: .bottom, spacing: 12) {
            if isMenuExpanded {
                HStack(spacing: 12) {
                    Button(action: { withAnimation(.interactiveSpring()) { showAttachmentMenu = true } }) { Image(systemName: "plus").font(.system(size: 26)).foregroundColor(.gray) }
                    PhotosPicker(selection: $selectedPhoto, matching: .images, photoLibrary: .shared()) { Image(systemName: "photo").font(.system(size: 24)).foregroundColor(.gray) }
                    .onChange(of: selectedPhoto) { newItem in Task { if let data = try? await newItem?.loadTransferable(type: Data.self), let uiImage = UIImage(data: data) { viewModel.uploadImage(uiImage) } } }
                    Button(action: { showCamera = true }) { Image(systemName: "camera").font(.system(size: 24)).foregroundColor(.gray) }
                    Button(action: { if audioRecorder.isRecording { if let audioUrl = audioRecorder.stopRecording() { viewModel.uploadAudio(audioUrl) } } else { audioRecorder.startRecording() } }) { Image(systemName: audioRecorder.isRecording ? "stop.circle.fill" : "mic").font(.system(size: 24)).foregroundColor(audioRecorder.isRecording ? .red : .gray) }
                }.transition(.move(edge: .leading).combined(with: .opacity)).padding(.bottom, 6)
            } else { Button(action: { withAnimation(.spring()) { isMenuExpanded = true } }) { Image(systemName: "chevron.right").font(.system(size: 22, weight: .bold)).foregroundColor(.white).frame(width: 32, height: 32).background(lyBlue).clipShape(Circle()) }.transition(.scale.combined(with: .opacity)).padding(.bottom, 4) }
            
            TextField(audioRecorder.isRecording ? "錄音中..." : "Aa", text: $inputText, axis: .vertical)
                .lineLimit(1...4)
                .disabled(audioRecorder.isRecording)
                .padding(.horizontal, 16)
                .padding(.vertical, 10)
                .background(Color.white)
                .cornerRadius(20)
                .overlay(RoundedRectangle(cornerRadius: 20).stroke(Color.gray.opacity(0.3), lineWidth: 1))
                .onChange(of: inputText) { newValue in
                    if !newValue.isEmpty && isMenuExpanded {
                        withAnimation(.spring()) { isMenuExpanded = false }
                    } else if newValue.isEmpty && !isMenuExpanded {
                        withAnimation(.spring()) { isMenuExpanded = true }
                    }
                }
            
            // 👉 修正 2：永遠顯示在右側的發送按鈕
            Button(action: {
                if !inputText.isEmpty || audioRecorder.isRecording {
                    viewModel.sendMessage(text: inputText, replyTo: replyingToText ?? "")
                    inputText = ""
                    withAnimation { replyingToText = nil }
                    scrollTrigger = UUID()
                }
            }) {
                Image(systemName: "paperplane.fill")
                    .font(.system(size: 22))
                    // 沒有輸入文字時變成淡灰色，有文字變成藍色
                    .foregroundColor((inputText.isEmpty && !audioRecorder.isRecording) ? .gray.opacity(0.4) : lyBlue)
                    .padding(.bottom, 6)
                    .padding(.trailing, 4)
            }
            .disabled(inputText.isEmpty && !audioRecorder.isRecording)
        }
        .padding(.horizontal, 16).padding(.vertical, 10).background(Color(UIColor.systemGray6))
    }
    
    @ViewBuilder private var attachmentMenuOverlay: some View {
        if showAttachmentMenu {
            Color.black.opacity(0.3).ignoresSafeArea().onTapGesture { withAnimation(.interactiveSpring()) { showAttachmentMenu = false } }
            VStack(spacing: 0) { Spacer(); VStack(spacing: 25) { HStack(alignment: .top, spacing: 25) {
                NavigationLink(destination: PollListView(chatId: viewModel.chatId, viewModel: viewModel)) { VStack(spacing: 8) { ZStack { Circle().fill(Color.white).frame(width: 60, height: 60).shadow(color: Color.black.opacity(0.08), radius: 5, x: 0, y: 2); Image(systemName: "chart.bar.fill").font(.system(size: 24)).foregroundColor(lyBlue) }; Text("發起投票").font(.system(size: 12, weight: .medium)).foregroundColor(.gray) } }.simultaneousGesture(TapGesture().onEnded { withAnimation { showAttachmentMenu = false } })
                NavigationLink(destination: EventListView(chatId: viewModel.chatId, viewModel: viewModel)) { VStack(spacing: 8) { ZStack { Circle().fill(Color.white).frame(width: 60, height: 60).shadow(color: Color.black.opacity(0.08), radius: 5, x: 0, y: 2); Image(systemName: "calendar").font(.system(size: 24)).foregroundColor(lyBlue) }; Text("活動行程").font(.system(size: 12, weight: .medium)).foregroundColor(.gray) } }.simultaneousGesture(TapGesture().onEnded { withAnimation { showAttachmentMenu = false } })
                NavigationLink(destination: GroupBuyListView(chatId: viewModel.chatId, viewModel: viewModel)) { VStack(spacing: 8) { ZStack { Circle().fill(Color.white).frame(width: 60, height: 60).shadow(color: Color.black.opacity(0.08), radius: 5, x: 0, y: 2); Image(systemName: "basket.fill").font(.system(size: 24)).foregroundColor(lyBlue) }; Text("團購功能").font(.system(size: 12, weight: .medium)).foregroundColor(.gray) } }.simultaneousGesture(TapGesture().onEnded { withAnimation { showAttachmentMenu = false } })
                NavigationLink(destination: ConfidentialListView(chatId: viewModel.chatId, viewModel: viewModel)) { VStack(spacing: 8) { ZStack { Circle().fill(Color.white).frame(width: 60, height: 60).shadow(color: Color.black.opacity(0.08), radius: 5, x: 0, y: 2); Image(systemName: "lock.shield.fill").font(.system(size: 24)).foregroundColor(lyBlue) }; Text("機密文件").font(.system(size: 12, weight: .medium)).foregroundColor(.gray) } }.simultaneousGesture(TapGesture().onEnded { withAnimation { showAttachmentMenu = false } })
            } }.padding(.top, 30).padding(.bottom, 35).frame(maxWidth: .infinity).background(Color.white).cornerRadius(24, corners: [.topLeft, .topRight]) }.ignoresSafeArea(.all, edges: .bottom).transition(.move(edge: .bottom)).zIndex(20)
        }
    }
    
    @ViewBuilder private var modalsAndOverlays: some View {
        if showReadList { Color.black.opacity(0.4).ignoresSafeArea().onTapGesture { showReadList = false }; VStack(spacing: 0) { HStack { Image(systemName: "eye.fill"); Text("已讀成員").font(.headline); Spacer(); Button(action: { showReadList = false }) { Image(systemName: "xmark").font(.title3) } }.padding().foregroundColor(.white).background(lyBlue).overlay(Rectangle().frame(height: 4).foregroundColor(lyGold), alignment: .bottom); ScrollView { LazyVGrid(columns: [GridItem(.adaptive(minimum: 80), spacing: 15)], spacing: 15) { ForEach(currentReadMembers) { member in VStack(spacing: 8) { CachedAvatarView(url: member.avatar, size: 45).overlay(Circle().stroke(Color.black, lineWidth: 2)).overlay(Circle().stroke(lyBlue, lineWidth: 3)); Text(member.name).font(.caption).fontWeight(.bold).foregroundColor(lyGold).lineLimit(1) }.padding(.vertical, 12).frame(maxWidth: .infinity).background(Color(red: 253/255, green: 252/255, blue: 245/255)).cornerRadius(8).overlay(RoundedRectangle(cornerRadius: 8).stroke(lyGold, lineWidth: 1)) } }.padding(20) }.frame(maxHeight: 300); Button(action: { showReadList = false }) { Text("關閉").font(.headline).foregroundColor(.gray).frame(maxWidth: .infinity).padding(.vertical, 12).background(Color.white).overlay(Rectangle().frame(height: 1).foregroundColor(Color(UIColor.systemGray5)), alignment: .top) } }.background(Color.white).cornerRadius(12).padding(.horizontal, 30) }

        if showPinnedModal || showReplyDetailModal { Color.black.opacity(0.4).ignoresSafeArea().onTapGesture { showPinnedModal = false; showReplyDetailModal = false }; VStack(spacing: 0) { HStack { Image(systemName: showPinnedModal ? "pin.fill" : "arrowshape.turn.up.left.fill").foregroundColor(lyGold).rotationEffect(showPinnedModal ? .degrees(45) : .zero); Text(showPinnedModal ? "釘選訊息內容" : "原始回覆訊息").font(.headline); Spacer(); Button(action: { showPinnedModal = false; showReplyDetailModal = false }) { Image(systemName: "xmark").foregroundColor(.gray) } }.padding().background(Color(UIColor.systemGray6)); ScrollView { if isImageUrl(popupDetailText) { if isGifUrl(popupDetailText) { GIFView(urlString: popupDetailText).frame(height: 250).cornerRadius(8).padding() } else { AsyncImage(url: URL(string: popupDetailText)) { phase in if let image = phase.image { image.resizable().scaledToFit().frame(maxHeight: 250).cornerRadius(8) } else { ProgressView() } }.padding() } } else { Text(popupDetailText).font(.body).padding().frame(maxWidth: .infinity, alignment: .leading) } }.frame(maxHeight: 300) }.background(Color.white).cornerRadius(12).padding(.horizontal, 40) }

        if let targetMsg = menuMessage {
            Color.black.opacity(0.15).ignoresSafeArea().onTapGesture { withAnimation(.spring()) { menuMessage = nil } }
            
            VStack(alignment: targetMsg.isMine ? .trailing : .leading, spacing: 12) {
                HStack(spacing: 16) {
                    ForEach(["👍", "❤️", "😂", "🙏", "👀"], id: \.self) { emoji in
                        Text(emoji).font(.system(size: 28))
                            .onTapGesture { viewModel.addReaction(messageId: targetMsg.id, emoji: emoji); withAnimation(.spring()) { menuMessage = nil } }
                    }
                }
                .padding(.horizontal, 20).padding(.vertical, 12)
                .background(Color.white)
                .cornerRadius(30)
                .shadow(color: Color.black.opacity(0.1), radius: 10, x: 0, y: 5)
                
                HStack(spacing: 24) {
                    Button(action: { UIPasteboard.general.string = targetMsg.text; withAnimation { menuMessage = nil } }) { Image(systemName: "doc.on.doc").font(.system(size: 22, weight: .semibold)) }
                    Button(action: { withAnimation { replyingToText = targetMsg.text.isEmpty ? "[圖片/影片]" : targetMsg.text; menuMessage = nil } }) { Image(systemName: "arrowshape.turn.up.left").font(.system(size: 22, weight: .semibold)) }
                    Button(action: { viewModel.pinMessage(text: targetMsg.text.isEmpty ? (targetMsg.fileType == "image" ? targetMsg.fileUrl : "[卡片/語音]") : targetMsg.text); withAnimation { menuMessage = nil } }) { Image(systemName: "pin").font(.system(size: 22, weight: .semibold)) }
                    
                    if targetMsg.isMine {
                        Button(action: {
                            if !targetMsg.cardType.isEmpty { viewModel.hideMessage(messageId: targetMsg.id) }
                            else { viewModel.revokeMessage(messageId: targetMsg.id) }
                            withAnimation { menuMessage = nil }
                        }) { Image(systemName: "trash").font(.system(size: 22, weight: .semibold)).foregroundColor(.red) }
                    }
                }
                .foregroundColor(.black)
                .padding(.horizontal, 24).padding(.vertical, 14)
                .background(Color.white)
                .cornerRadius(30)
                .shadow(color: Color.black.opacity(0.1), radius: 10, x: 0, y: 5)
            }
            .transition(.scale(scale: 0.9).combined(with: .opacity))
        }
    }
    
    @ViewBuilder func readStatusView(msg: MessageItem) -> some View {
        HStack(spacing: 6) {
            Text(timeFormatter.string(from: msg.timestamp)).font(.system(size: 11)).foregroundColor(.gray)
            
            if !msg.readBy.isEmpty {
                let displayNames = msg.readBy.map { $0.name }.joined(separator: "、")
                Text(displayNames)
                    .font(.system(size: 11, weight: .medium))
                    .foregroundColor(lyBlue)
                    .onTapGesture { currentReadMembers = msg.readBy; withAnimation { showReadList = true } }
            }
        }.padding(.horizontal, 4)
    }
    
    @ViewBuilder func bubbleContent(msg: MessageItem) -> some View {
        VStack(alignment: msg.isMine ? .trailing : .leading, spacing: 2) {
            if !msg.replyToText.isEmpty { HStack(spacing: 4) { Rectangle().fill(lyGold).frame(width: 3, height: 12); Text(msg.replyToText).font(.system(size: 11)).foregroundColor(.gray).lineLimit(1) }.padding(.horizontal, 8).padding(.bottom, 2).onTapGesture { popupDetailText = msg.replyToText; showReplyDetailModal = true } }
            Group {
                if msg.fileType == "image" || isImageUrl(msg.text) { let urlString = msg.fileType == "image" ? msg.fileUrl : msg.text.trimmingCharacters(in: .whitespacesAndNewlines); Button(action: { fullScreenMediaUrl = urlString; fullScreenMediaIsGif = isGifUrl(urlString); withAnimation { showFullScreenMedia = true } }) { if isGifUrl(urlString) { GIFView(urlString: urlString).frame(width: 220, height: 220).cornerRadius(15) } else { AsyncImage(url: URL(string: urlString)) { phase in if let image = phase.image { image.resizable().scaledToFill().frame(width: 220, height: 220).cornerRadius(15).clipped() } else { ZStack { Rectangle().fill(Color(UIColor.systemGray5)).frame(width: 220, height: 220).cornerRadius(15); ProgressView() } } } } }.buttonStyle(PlainButtonStyle()) }
                else if msg.fileType == "audio" { AudioBubbleView(audioUrl: msg.fileUrl, isMine: msg.isMine) }
                else if !msg.cardType.isEmpty, let cardData = msg.cardData {
                    if msg.cardType == "event_share" { NavigationLink(destination: EventDetailView(cardData: cardData, viewModel: viewModel)) { eventCardUI(cardData: cardData) }.buttonStyle(PlainButtonStyle()) }
                    else if msg.cardType == "group_buy" { NavigationLink(destination: GroupBuyDetailView(cardData: cardData)) { groupBuyCardUI(cardData: cardData) }.buttonStyle(PlainButtonStyle()) }
                    else if msg.cardType == "poll" { NavigationLink(destination: PollDetailView(msgId: msg.id, cardData: cardData, viewModel: viewModel)) { pollCardUI(cardData: cardData) }.buttonStyle(PlainButtonStyle()) }
                    else if msg.cardType == "confidential" { NavigationLink(destination: ConfidentialDetailView(cardData: cardData)) { confidentialCardUI(cardData: cardData) }.buttonStyle(PlainButtonStyle()) }
                } else {
                    Text(msg.text)
                        .font(.system(size: 15))
                        .padding(.horizontal, 16).padding(.vertical, 12)
                        .background(msg.isMine ? Color(red: 245/255, green: 245/255, blue: 255/255) : Color(UIColor.systemGray6))
                        .foregroundColor(.primary)
                        .cornerRadius(20)
                }
            }.simultaneousGesture(LongPressGesture(minimumDuration: 0.2).onEnded { _ in let impactMed = UIImpactFeedbackGenerator(style: .medium); impactMed.impactOccurred(); withAnimation(.spring(response: 0.3, dampingFraction: 0.7)) { menuMessage = msg } })
            
            if !msg.reactions.isEmpty {
                HStack(spacing: 4) {
                    ForEach(msg.reactions.keys.sorted(), id: \.self) { emoji in
                        if let count = msg.reactions[emoji]?.count, count > 0 {
                            Text("\(emoji)")
                                .font(.system(size: 14))
                                .padding(.horizontal, 8)
                                .padding(.vertical, 4)
                                .background(Color.white)
                                .overlay(RoundedRectangle(cornerRadius: 15).stroke(lyBlue, lineWidth: 1))
                                .cornerRadius(15)
                                .shadow(color: Color.black.opacity(0.1), radius: 2, x: 0, y: 1)
                                .onTapGesture { viewModel.addReaction(messageId: msg.id, emoji: emoji) }
                        }
                    }
                }
                .padding(msg.isMine ? .trailing : .leading, 10)
                .offset(y: -12)
                .zIndex(1)
            }
        }
    }
    
    func isImageUrl(_ text: String) -> Bool { let cleanText = text.trimmingCharacters(in: .whitespacesAndNewlines).lowercased(); guard cleanText.hasPrefix("http") else { return false }; return cleanText.contains(".jpg") || cleanText.contains(".jpeg") || cleanText.contains(".png") || cleanText.contains(".gif") }
    func isGifUrl(_ text: String) -> Bool { return text.trimmingCharacters(in: .whitespacesAndNewlines).lowercased().contains(".gif") }
    func eventCardUI(cardData: [String: Any]) -> some View { VStack(alignment: .leading, spacing: 8) { Text("活動分享").font(.caption).fontWeight(.bold).foregroundColor(lyGold); Text(cardData["title"] as? String ?? "未命名活動").font(.headline).foregroundColor(lyBlue).lineLimit(1); Text("時間: \(cardData["timeStr"] as? String ?? "")").font(.caption).foregroundColor(.gray) }.padding().frame(width: 200, alignment: .leading).background(Color.white).cornerRadius(12).overlay(RoundedRectangle(cornerRadius: 12).stroke(lyGold, lineWidth: 1)) }
    func groupBuyCardUI(cardData: [String: Any]) -> some View { VStack(alignment: .leading, spacing: 8) { Text("團購訂單").font(.caption).fontWeight(.bold).foregroundColor(.green); Text(cardData["title"] as? String ?? "未命名團購").font(.headline).foregroundColor(lyBlue).lineLimit(1); Text("發起人: \(cardData["initiator"] as? String ?? "")").font(.caption).foregroundColor(.gray) }.padding().frame(width: 200, alignment: .leading).background(Color.white).cornerRadius(12).overlay(RoundedRectangle(cornerRadius: 12).stroke(Color.green, lineWidth: 1)) }
    func pollCardUI(cardData: [String: Any]) -> some View { VStack(alignment: .leading, spacing: 8) { Text("群組投票").font(.caption).fontWeight(.bold).foregroundColor(lyBlue); Text(cardData["question"] as? String ?? "未命名投票").font(.headline).foregroundColor(lyBlue).lineLimit(1); Text("點擊前往投票...").font(.caption).foregroundColor(.gray) }.padding().frame(width: 200, alignment: .leading).background(Color.white).cornerRadius(12).overlay(RoundedRectangle(cornerRadius: 12).stroke(lyBlue, lineWidth: 1)) }
    func confidentialCardUI(cardData: [String: Any]) -> some View { VStack(alignment: .leading, spacing: 8) { Text("絕對機密").font(.caption).fontWeight(.bold).foregroundColor(.red); Text(cardData["title"] as? String ?? "機密檔案").font(.headline).foregroundColor(.white).lineLimit(1); Text("點擊輸入密碼解鎖").font(.caption).foregroundColor(.gray) }.padding().frame(width: 200, alignment: .leading).background(Color(red: 43/255, green: 43/255, blue: 43/255)).cornerRadius(12).overlay(RoundedRectangle(cornerRadius: 12).stroke(Color.red, lineWidth: 1)) }
}

struct ChatSettingsView: View {
    let chatName: String
    @ObservedObject var viewModel: ChatRoomViewModel
    @Environment(\.presentationMode) var presentationMode
    
    let lyBlue = Color(red: 0/255, green: 49/255, blue: 83/255)
    
    var body: some View {
        ScrollView {
            VStack(spacing: 25) {
                
                VStack(alignment: .leading, spacing: 8) {
                    Text("基本設定").font(.caption).foregroundColor(.gray).padding(.horizontal, 16)
                    VStack(spacing: 0) {
                        HStack { Text("群組名稱"); Spacer(); Text(chatName).foregroundColor(.gray) }.padding()
                        Divider().padding(.leading, 16)
                        HStack { Text("更換群組頭像").foregroundColor(.blue); Spacer() }.padding()
                        Divider().padding(.leading, 16)
                        NavigationLink(destination: ChatSearchView(viewModel: viewModel)) {
                            HStack {
                                Image(systemName: "magnifyingglass").foregroundColor(.blue).font(.system(size: 20))
                                Text("搜尋聊天內容").foregroundColor(.primary)
                                Spacer()
                                Image(systemName: "chevron.right").foregroundColor(.gray)
                            }.padding()
                        }
                    }
                    .background(Color.white).cornerRadius(16)
                }
                
                VStack(alignment: .leading, spacing: 8) {
                    Text("多媒體與檔案").font(.caption).foregroundColor(.gray).padding(.horizontal, 16)
                    VStack(spacing: 0) {
                        NavigationLink(destination: SharedMediaView(viewModel: viewModel)) {
                            HStack {
                                Image(systemName: "photo.on.rectangle.angled")
                                    .foregroundColor(.blue)
                                    .frame(width: 36, height: 36)
                                    .background(Color.blue.opacity(0.1))
                                    .cornerRadius(8)
                                
                                Text("歷史照片 (5)").foregroundColor(.primary)
                                Spacer()
                                Image(systemName: "chevron.right").foregroundColor(.gray)
                            }.padding()
                        }
                    }
                    .background(Color.white).cornerRadius(16)
                }
                
                VStack(alignment: .leading, spacing: 8) {
                    HStack {
                        Text("成員名單 (\(viewModel.groupMembersCount))").font(.caption).foregroundColor(.gray)
                        Spacer()
                        NavigationLink(destination: ChatMembersView(chatId: viewModel.chatId)) {
                            Text("編輯").font(.caption).foregroundColor(.blue)
                        }
                    }.padding(.horizontal, 16)
                    
                    VStack(spacing: 0) {
                        ScrollView(.horizontal, showsIndicators: false) {
                            HStack(spacing: 20) {
                                VStack(spacing: 6) { CachedAvatarView(url: "", size: 55).overlay(Circle().stroke(Color.gray.opacity(0.3), lineWidth: 1)); Text("班長").font(.caption) }
                                VStack(spacing: 6) { CachedAvatarView(url: "", size: 55).overlay(Circle().stroke(Color.gray.opacity(0.3), lineWidth: 1)); Text("俊齊").font(.caption) }
                                VStack(spacing: 6) { CachedAvatarView(url: "", size: 55).overlay(Circle().stroke(Color.gray.opacity(0.3), lineWidth: 1)); Text("博峻").font(.caption) }
                            }.padding()
                        }
                        
                        Divider().padding(.horizontal, 16)
                        
                        NavigationLink(destination: ChatMembersView(chatId: viewModel.chatId)) {
                            HStack { Spacer(); Text("展開所有成員 (\(viewModel.groupMembersCount))").foregroundColor(.blue); Spacer() }.padding()
                        }
                        
                        Divider().padding(.horizontal, 16)
                        
                        Button(action: { print("邀請新成員") }) {
                            HStack {
                                Image(systemName: "person.badge.plus").foregroundColor(lyBlue).font(.system(size: 20))
                                Text("邀請新成員").foregroundColor(lyBlue)
                                Spacer()
                            }.padding()
                        }
                    }
                    .background(Color.white).cornerRadius(16)
                }
                
                Button(action: { viewModel.deleteChat { success in if success { presentationMode.wrappedValue.dismiss() } } }) {
                    Text("退出群組")
                        .font(.headline)
                        .foregroundColor(.red)
                        .frame(maxWidth: .infinity)
                        .padding()
                        .background(Color.white)
                        .cornerRadius(16)
                }
                
            }.padding(.vertical, 20).padding(.horizontal, 16)
        }
        .background(Color(UIColor.systemGroupedBackground).ignoresSafeArea())
        .navigationBarBackButtonHidden(true)
        .toolbar {
            ToolbarItem(placement: .principal) { Text("聊天室設定").font(.headline) }
            ToolbarItem(placement: .navigationBarLeading) {
                Button("關閉") { presentationMode.wrappedValue.dismiss() }
                    .font(.system(size: 14, weight: .bold))
                    .foregroundColor(.gray)
                    .padding(.horizontal, 14).padding(.vertical, 6)
                    .background(Color.gray.opacity(0.15))
                    .cornerRadius(20)
            }
            ToolbarItem(placement: .navigationBarTrailing) {
                Button("完成") { presentationMode.wrappedValue.dismiss() }
                    .font(.system(size: 14, weight: .bold))
                    .foregroundColor(.white)
                    .padding(.horizontal, 14).padding(.vertical, 6)
                    .background(Color(white: 0.15))
                    .cornerRadius(20)
            }
        }
        .onAppear { NotificationCenter.default.post(name: NSNotification.Name("HideTabBar"), object: nil) }
    }
}

struct SharedMediaView: View {
    @ObservedObject var viewModel: ChatRoomViewModel
    var mediaMessages: [MessageItem] { return viewModel.messages.filter { $0.fileType == "image" || isImageUrl($0.text) } }
    func isImageUrl(_ text: String) -> Bool { let t = text.lowercased(); return t.hasPrefix("http") && (t.contains(".jpg") || t.contains(".png") || t.contains(".gif")) }
    
    var body: some View {
        ScrollView {
            if mediaMessages.isEmpty { Text("尚無分享的媒體...").foregroundColor(.gray).padding(30) } else { LazyVGrid(columns: [GridItem(.adaptive(minimum: 100), spacing: 2)], spacing: 2) { ForEach(mediaMessages) { msg in let urlString = msg.fileType == "image" ? msg.fileUrl : msg.text.trimmingCharacters(in: .whitespacesAndNewlines); AsyncImage(url: URL(string: urlString)) { phase in if let image = phase.image { image.resizable().scaledToFill().frame(minWidth: 0, maxWidth: .infinity, minHeight: 100, maxHeight: 100).clipped() } else { Rectangle().fill(Color(UIColor.systemGray5)).frame(height: 100) } } } } }
        }.navigationTitle("相片與影片").navigationBarTitleDisplayMode(.inline).onAppear { NotificationCenter.default.post(name: NSNotification.Name("HideTabBar"), object: nil) }
    }
}

struct ChatSearchView: View {
    @ObservedObject var viewModel: ChatRoomViewModel
    @State private var searchText = ""
    let lyBlue = Color(red: 0/255, green: 49/255, blue: 83/255)
    var searchResults: [MessageItem] { if searchText.isEmpty { return [] }; return viewModel.messages.filter { $0.text.lowercased().contains(searchText.lowercased()) }.reversed() }
    
    var body: some View {
        VStack {
            HStack { Image(systemName: "magnifyingglass").foregroundColor(.gray); TextField("輸入關鍵字尋找對話...", text: $searchText) }.padding(12).background(Color.white).cornerRadius(10).padding()
            ScrollView {
                if searchText.isEmpty { Text("輸入上方文字開始搜尋").foregroundColor(.gray).padding() } else if searchResults.isEmpty { Text("找不到符合的紀錄").foregroundColor(.red).padding() } else { VStack(spacing: 10) { ForEach(searchResults) { msg in VStack(alignment: .leading, spacing: 4) { HStack { Text(msg.isMine ? "您" : msg.senderName).font(.caption).fontWeight(.bold).foregroundColor(lyBlue); Spacer(); Text(msg.timestamp, style: .date).font(.caption2).foregroundColor(.gray) }; Text(msg.text).font(.subheadline).foregroundColor(.primary) }.padding().frame(maxWidth: .infinity, alignment: .leading).background(Color.white).cornerRadius(8) } }.padding(.horizontal) }
            }
        }.background(Color(UIColor.systemGroupedBackground).ignoresSafeArea()).navigationTitle("歷史對話搜尋").navigationBarTitleDisplayMode(.inline).onAppear { NotificationCenter.default.post(name: NSNotification.Name("HideTabBar"), object: nil) }
    }
}

struct ChatMembersView: View {
    let chatId: String
    @State private var members: [String] = []
    @State private var isLoading = true
    var body: some View {
        List { if isLoading { ProgressView("成員載入中...") } else if members.isEmpty { Text("此群組尚無成員。").foregroundColor(.gray) } else { ForEach(members, id: \.self) { uid in LiveUserRow(uid: uid).padding(.vertical, 8) } } }
        .navigationTitle("群組成員名單").navigationBarTitleDisplayMode(.inline).onAppear { fetchMembers(); NotificationCenter.default.post(name: NSNotification.Name("HideTabBar"), object: nil) }
    }
    private func fetchMembers() { Firestore.firestore().collection("chats").document(chatId).getDocument { doc, _ in if let data = doc?.data(), let membersArray = data["members"] as? [String] { self.members = membersArray }; self.isLoading = false } }
}

struct FullScreenMediaViewer: View {
    let urlString: String
    let isGif: Bool
    @Binding var isPresented: Bool
    @State private var scale: CGFloat = 1.0
    @State private var lastScale: CGFloat = 1.0
    @State private var offset: CGSize = .zero
    @State private var lastOffset: CGSize = .zero

    var body: some View {
        ZStack {
            Color.black.ignoresSafeArea()
            if isGif { GIFView(urlString: urlString, allowZoom: true).ignoresSafeArea() } else { AsyncImage(url: URL(string: urlString)) { phase in if let image = phase.image { image.resizable().scaledToFit().scaleEffect(scale).offset(offset).gesture(MagnificationGesture().onChanged { val in scale = lastScale * val }.onEnded { val in lastScale = scale; if scale < 1.0 { withAnimation { scale = 1.0; lastScale = 1.0; offset = .zero; lastOffset = .zero } } }.simultaneously(with: DragGesture().onChanged { val in if scale > 1.0 { offset = CGSize(width: lastOffset.width + val.translation.width, height: lastOffset.height + val.translation.height) } }.onEnded { val in lastOffset = offset }) ) } else { ProgressView().tint(.white) } } }
            VStack { HStack { Spacer(); Button(action: { isPresented = false }) { Image(systemName: "xmark.circle.fill").font(.system(size: 30)).foregroundColor(.white.opacity(0.8)).padding() } }; Spacer() }
        }
    }
}


// MARK: - 7. 擴充功能列表與發起 (Features Views)

// --- 投票 ---
struct PollListView: View {
    let chatId: String
    @ObservedObject var viewModel: ChatRoomViewModel
    @State private var polls: [PollItem] = []
    @State private var isLoading = true
    @State private var selectedPollToShare: PollItem? = nil
    @State private var showShareConfirmation = false
    let lyBlue = Color(red: 0/255, green: 49/255, blue: 83/255)
    let lyGold = Color(red: 184/255, green: 134/255, blue: 11/255)
    
    var body: some View {
        VStack(spacing: 0) {
            if isLoading { Spacer(); ProgressView("載入投票資料中..."); Spacer() } else if polls.isEmpty { Spacer(); Text("尚無投票，點擊右上角新增。").foregroundColor(.gray); Spacer() } else {
                List {
                    Section { PrivilegeBanner(myName: viewModel.myName, color: lyGold) }
                    ForEach(polls) { poll in
                        pollCardRow(poll: poll).swipeActions(edge: .trailing, allowsFullSwipe: false) {
                            if canEdit(creatorUid: poll.creatorUid, myName: viewModel.myName) {
                                Button(role: .destructive) { Firestore.firestore().collection("polls").document(poll.id).delete() } label: { Label("刪除", systemImage: "trash.fill") }
                                NavigationLink { CreatePollView(chatId: chatId, editingPollId: poll.id) } label: { Label("編輯", systemImage: "pencil") }.tint(lyBlue)
                            } else { Button { } label: { Label("權限不足", systemImage: "lock.fill") }.tint(.gray) }
                        }
                    }
                }.listStyle(.plain).background(Color(UIColor.systemGroupedBackground))
            }
        }.background(Color(UIColor.systemGroupedBackground).ignoresSafeArea()).navigationTitle("投票清單").navigationBarTitleDisplayMode(.inline).toolbar { ToolbarItem(placement: .navigationBarTrailing) { NavigationLink(destination: CreatePollView(chatId: chatId)) { Image(systemName: "plus").font(.title2).foregroundColor(.white) } } }
        .onAppear(perform: loadPolls).confirmationDialog("確認分享", isPresented: $showShareConfirmation, titleVisibility: .visible) { Button("分享至聊天室") { if let poll = selectedPollToShare { sendPollToChat(poll: poll) } }; Button("取消", role: .cancel) { } } message: { Text("您確定要將「\(selectedPollToShare?.question ?? "")」這項投票分享至目前的聊天室嗎？") }
    }
    
    private func pollCardRow(poll: PollItem) -> some View {
        Button(action: { selectedPollToShare = poll; showShareConfirmation = true }) {
            HStack(spacing: 0) {
                Rectangle().fill(lyBlue).frame(width: 6)
                VStack(alignment: .leading, spacing: 10) {
                    HStack { Image(systemName: "chart.bar.fill"); Text(poll.question).font(.title3).fontWeight(.bold) }.foregroundColor(lyBlue)
                    HStack(alignment: .bottom) {
                        VStack(alignment: .leading, spacing: 4) { HStack(spacing: 4) { Image(systemName: "clock"); Text("截止："); Text(poll.deadline.isEmpty ? "無期限" : poll.deadline.replacingOccurrences(of: "T", with: " ")) }.font(.caption).foregroundColor(.red) }
                        Spacer()
                        Text("建立人：\(poll.creatorName)").font(.caption).fontWeight(.bold).foregroundColor(lyBlue).padding(.horizontal, 10).padding(.vertical, 4).background(Color.gray.opacity(0.1)).cornerRadius(12)
                    }
                }.padding(.vertical, 12).padding(.horizontal, 15).background(Color.white)
            }.cornerRadius(12).overlay(RoundedRectangle(cornerRadius: 12).stroke(Color.gray.opacity(0.2), lineWidth: 1))
        }.buttonStyle(PlainButtonStyle()).listRowInsets(EdgeInsets(top: 6, leading: 16, bottom: 6, trailing: 16)).listRowSeparator(.hidden).listRowBackground(Color.clear)
    }
    
    private func loadPolls() { Firestore.firestore().collection("polls").order(by: "createdAt", descending: true).addSnapshotListener { snap, error in guard let docs = snap?.documents else { self.isLoading = false; return }; self.polls = docs.map { doc in let data = doc.data(); return PollItem(id: doc.documentID, question: data["question"] as? String ?? "未命名投票", content: data["content"] as? String ?? "", deadline: data["deadline"] as? String ?? "", creatorUid: data["creatorUid"] as? String ?? "", creatorName: data["creatorName"] as? String ?? "未知", allowAddOptions: data["allowAddOptions"] as? Bool ?? false, options: data["options"] as? [String] ?? []) }; self.isLoading = false } }
    
    private func sendPollToChat(poll: PollItem) { guard let currentUid = Auth.auth().currentUser?.uid else { return }; let db = Firestore.firestore(); let batch = db.batch(); let chatRef = db.collection("chats").document(chatId); let msgRef = chatRef.collection("messages").document(); var votesMap: [String: [String]] = [:]; var optionCreators: [String: String] = [:]; poll.options.forEach { opt in votesMap[opt] = []; optionCreators[opt] = currentUid }; let payload: [String: Any] = [ "type": "poll", "pollId": poll.id, "question": poll.question, "content": poll.content, "options": poll.options, "votes": votesMap, "deadline": poll.deadline, "allowAddOptions": poll.allowAddOptions, "optionCreators": optionCreators ]; if let jsonData = try? JSONSerialization.data(withJSONObject: payload), let jsonString = String(data: jsonData, encoding: .utf8) { batch.setData(["text": jsonString, "senderId": currentUid, "timestamp": FieldValue.serverTimestamp()], forDocument: msgRef); batch.updateData(["lastMessage": "📊 發起了投票：\(poll.question)", "lastSenderId": currentUid, "updatedAt": FieldValue.serverTimestamp(), "lastReadAt.\(currentUid)": FieldValue.serverTimestamp()], forDocument: chatRef); batch.commit() } }
}

struct CreatePollView: View {
    let chatId: String; var editingPollId: String? = nil; @Environment(\.presentationMode) var presentationMode
    @State private var question: String = ""; @State private var content: String = ""; @State private var deadline: Date = Date().addingTimeInterval(86400); @State private var useDeadline: Bool = false; @State private var allowAddOptions: Bool = false; @State private var options: [String] = ["", ""]; @State private var isSubmitting = false; @State private var isDataLoaded = false
    let lyBlue = Color(red: 0/255, green: 49/255, blue: 83/255)
    
    var body: some View {
        ScrollView { VStack(alignment: .leading, spacing: 20) {
            VStack(alignment: .leading, spacing: 8) { Text("投票主題 (必填)").font(.subheadline).foregroundColor(lyBlue).fontWeight(.bold); TextField("例如：下次要買哪間的法棍", text: $question).padding().background(Color.white).cornerRadius(8).overlay(RoundedRectangle(cornerRadius: 8).stroke(Color.gray.opacity(0.3), lineWidth: 1)) }
            VStack(alignment: .leading, spacing: 8) { Text("投票內容說明 (選填)").font(.subheadline).foregroundColor(lyBlue).fontWeight(.bold); TextEditor(text: $content).frame(minHeight: 100).padding(8).background(Color.white).cornerRadius(8).overlay(RoundedRectangle(cornerRadius: 8).stroke(Color.gray.opacity(0.3), lineWidth: 1)) }
            VStack(alignment: .leading, spacing: 8) { Toggle(isOn: $useDeadline) { Text("設定投票結束時間").font(.subheadline).foregroundColor(lyBlue).fontWeight(.bold) }.tint(lyBlue); if useDeadline { DatePicker("", selection: $deadline).datePickerStyle(.compact).labelsHidden().padding(.vertical, 8) } }
            Toggle(isOn: $allowAddOptions) { Text("允許成員新增選項").font(.subheadline).foregroundColor(lyBlue).fontWeight(.bold) }.tint(lyBlue).padding().background(Color(UIColor.systemGray6)).cornerRadius(8).overlay(RoundedRectangle(cornerRadius: 8).stroke(lyBlue.opacity(0.5), style: StrokeStyle(lineWidth: 1, dash: [5])))
            VStack(alignment: .leading, spacing: 10) { Text("選項設定").font(.subheadline).foregroundColor(lyBlue).fontWeight(.bold); ForEach(0..<options.count, id: \.self) { index in HStack { TextField("選項 \(index + 1)", text: $options[index]).padding().background(Color.white).cornerRadius(8).overlay(RoundedRectangle(cornerRadius: 8).stroke(Color.gray.opacity(0.3), lineWidth: 1)); if options.count > 2 { Button(action: { options.remove(at: index) }) { Image(systemName: "xmark.circle.fill").foregroundColor(.red).font(.title2) } } } }; Button(action: { options.append("") }) { HStack { Image(systemName: "plus"); Text("新增選項") }.font(.headline).foregroundColor(lyBlue).frame(maxWidth: .infinity).padding().background(Color(UIColor.systemGray6)).cornerRadius(8).overlay(RoundedRectangle(cornerRadius: 8).stroke(lyBlue.opacity(0.5), style: StrokeStyle(lineWidth: 1, dash: [5]))) } }
            Button(action: submitPoll) { HStack { if isSubmitting { ProgressView().tint(.white).padding(.trailing, 5) } else { Image(systemName: editingPollId == nil ? "paperplane.fill" : "save.fill") }; Text(isSubmitting ? "處理中..." : (editingPollId == nil ? "建立儲存" : "儲存修改")) }.font(.headline).foregroundColor(.white).frame(maxWidth: .infinity).padding().background(lyBlue).cornerRadius(10).shadow(color: lyBlue.opacity(0.3), radius: 5, x: 0, y: 3) }.disabled(isSubmitting || question.trimmingCharacters(in: .whitespaces).isEmpty || !isDataLoaded).padding(.top, 20)
        }.padding(20) }.background(Color(UIColor.systemGroupedBackground).ignoresSafeArea()).navigationTitle(editingPollId == nil ? "發起投票" : "編輯投票").navigationBarTitleDisplayMode(.inline).onAppear(perform: loadEditingDataIfNeeded)
    }
    
    private func loadEditingDataIfNeeded() { guard let pollId = editingPollId else { isDataLoaded = true; return }; Firestore.firestore().collection("polls").document(pollId).getDocument { doc, _ in if let data = doc?.data() { self.question = data["question"] as? String ?? ""; self.content = data["content"] as? String ?? ""; self.allowAddOptions = data["allowAddOptions"] as? Bool ?? false; self.options = data["options"] as? [String] ?? ["", ""]; let deadlineStr = data["deadline"] as? String ?? ""; if !deadlineStr.isEmpty { let f = DateFormatter(); f.dateFormat = "yyyy-MM-dd'T'HH:mm"; if let d = f.date(from: deadlineStr) { self.deadline = d; self.useDeadline = true } } }; self.isDataLoaded = true } }
    private func submitPoll() { let cleanOptions = options.map { $0.trimmingCharacters(in: .whitespaces) }.filter { !$0.isEmpty }; if cleanOptions.count < 2 { return }; isSubmitting = true; let db = Firestore.firestore(); let uid = Auth.auth().currentUser?.uid ?? ""; let userName = Auth.auth().currentUser?.displayName ?? "未知"; let deadlineStr = useDeadline ? ISO8601DateFormatter().string(from: deadline) : ""; var pollData: [String: Any] = [ "question": question, "content": content, "deadline": deadlineStr, "allowAddOptions": allowAddOptions, "options": cleanOptions, "creatorUid": uid, "creatorName": userName ]; if let pollId = editingPollId { pollData["updatedAt"] = FieldValue.serverTimestamp(); db.collection("polls").document(pollId).updateData(pollData) { _ in isSubmitting = false; presentationMode.wrappedValue.dismiss() } } else { pollData["createdAt"] = FieldValue.serverTimestamp(); db.collection("polls").addDocument(data: pollData) { error in isSubmitting = false; presentationMode.wrappedValue.dismiss() } } }
}

// --- 活動行程 ---
struct EventListView: View {
    let chatId: String
    @ObservedObject var viewModel: ChatRoomViewModel
    @State private var items: [EventItem] = []
    @State private var isLoading = true
    let lyBlue = Color(red: 0/255, green: 49/255, blue: 83/255)
    let lyGold = Color(red: 184/255, green: 134/255, blue: 11/255)
    
    var body: some View {
        VStack(spacing: 0) {
            if isLoading { Spacer(); ProgressView("載入中..."); Spacer() } else if items.isEmpty { Spacer(); Text("尚無活動。").foregroundColor(.gray); Spacer() } else {
                List { Section { PrivilegeBanner(myName: viewModel.myName, color: lyGold) }; ForEach(items) { item in GenericListRow(title: item.title, subtitle: item.time.isEmpty ? "無時間" : item.time, creator: item.creatorName, icon: "calendar", color: lyBlue) { sendToChat(item) }.swipeActions(edge: .trailing, allowsFullSwipe: false) { if canEdit(creatorUid: item.creatorUid, myName: viewModel.myName) { Button(role: .destructive) { Firestore.firestore().collection("events").document(item.id).delete() } label: { Label("刪除", systemImage: "trash.fill") } } else { Button { } label: { Label("權限不足", systemImage: "lock.fill") }.tint(.gray) } } } }.listStyle(.plain).background(Color(UIColor.systemGroupedBackground))
            }
        }.background(Color(UIColor.systemGroupedBackground).ignoresSafeArea()).navigationTitle("活動行程").navigationBarTitleDisplayMode(.inline).toolbar { ToolbarItem(placement: .navigationBarTrailing) { Button("新增(待開發)") { }.foregroundColor(.white) } }.onAppear { load(); NotificationCenter.default.post(name: NSNotification.Name("HideTabBar"), object: nil) }
    }
    
    private func load() { Firestore.firestore().collection("events").order(by: "createdAt", descending: true).addSnapshotListener { snap, _ in self.items = (snap?.documents ?? []).map { doc in let data = doc.data(); let fields = data["fields"] as? [[String: Any]] ?? []; let time = fields.first(where: { $0["type"] as? String == "time" })?["content"] as? String ?? ""; return EventItem(id: doc.documentID, title: data["title"] as? String ?? "未命名", time: time.replacingOccurrences(of: "T", with: " "), creatorUid: data["creatorUid"] as? String ?? "", creatorName: data["creatorName"] as? String ?? "未知", rawData: data) }; self.isLoading = false } }
    private func sendToChat(_ item: EventItem) { guard let uid = Auth.auth().currentUser?.uid else { return }; let payload: [String: Any] = [ "type": "event_share", "eventId": item.id, "title": item.title, "timeStr": item.time ]; if let jsonData = try? JSONSerialization.data(withJSONObject: payload), let str = String(data: jsonData, encoding: .utf8) { let db = Firestore.firestore(); let batch = db.batch(); let cRef = db.collection("chats").document(chatId); let mRef = cRef.collection("messages").document(); batch.setData(["text": str, "senderId": uid, "timestamp": FieldValue.serverTimestamp()], forDocument: mRef); batch.updateData(["lastMessage": "📅 分享了活動：\(item.title)", "lastSenderId": uid, "updatedAt": FieldValue.serverTimestamp()], forDocument: cRef); batch.commit() } }
}

// --- 團購 ---
struct GroupBuyListView: View {
    let chatId: String; @ObservedObject var viewModel: ChatRoomViewModel
    @State private var items: [GBItem] = []; @State private var isLoading = true
    
    var body: some View { VStack(spacing: 0) { if isLoading { Spacer(); ProgressView("載入中..."); Spacer() } else if items.isEmpty { Spacer(); Text("尚無團購，點擊右上角新增。").foregroundColor(.gray); Spacer() } else { List { Section { PrivilegeBanner(myName: viewModel.myName, color: .green) }; ForEach(items) { item in GenericListRow(title: item.title, subtitle: item.deadline.isEmpty ? "無期限" : item.deadline, creator: item.creatorName, icon: "basket.fill", color: .green) { sendToChat(item) }.swipeActions(edge: .trailing, allowsFullSwipe: false) { if canEdit(creatorUid: item.creatorUid, myName: viewModel.myName) { Button(role: .destructive) { Firestore.firestore().collection("group_buys").document(item.id).delete() } label: { Label("刪除", systemImage: "trash.fill") } } else { Button { } label: { Label("權限不足", systemImage: "lock.fill") }.tint(.gray) } } } }.listStyle(.plain).background(Color(UIColor.systemGroupedBackground)) } }.background(Color(UIColor.systemGroupedBackground).ignoresSafeArea()).navigationTitle("團購清單").navigationBarTitleDisplayMode(.inline).toolbar { ToolbarItem(placement: .navigationBarTrailing) { NavigationLink(destination: CreateGroupBuyView(chatId: chatId)) { Image(systemName: "plus") } } }.onAppear { load(); NotificationCenter.default.post(name: NSNotification.Name("HideTabBar"), object: nil) }
    }
    
    private func load() { Firestore.firestore().collection("group_buys").order(by: "createdAt", descending: true).addSnapshotListener { snap, _ in self.items = (snap?.documents ?? []).map { doc in let data = doc.data(); return GBItem(id: doc.documentID, title: data["title"] as? String ?? "未命名", deadline: data["deadline"] as? String ?? "", creatorUid: data["initiatorUid"] as? String ?? "", creatorName: data["initiator"] as? String ?? "未知", rawData: data) }; self.isLoading = false } }
    private func sendToChat(_ item: GBItem) { guard let uid = Auth.auth().currentUser?.uid else { return }; let payload: [String: Any] = [ "type": "group_buy", "groupBuyId": item.id, "title": item.title, "initiator": item.creatorName, "deadline": item.deadline ]; if let jsonData = try? JSONSerialization.data(withJSONObject: payload), let str = String(data: jsonData, encoding: .utf8) { let db = Firestore.firestore(); let batch = db.batch(); let cRef = db.collection("chats").document(chatId); let mRef = cRef.collection("messages").document(); batch.setData(["text": str, "senderId": uid, "timestamp": FieldValue.serverTimestamp()], forDocument: mRef); batch.updateData(["lastMessage": "🛍️ 分享了團購：\(item.title)", "lastSenderId": uid, "updatedAt": FieldValue.serverTimestamp()], forDocument: cRef); batch.commit() } }
}

struct CreateGroupBuyView: View {
    let chatId: String; @Environment(\.presentationMode) var presentationMode
    @State private var title: String = ""; @State private var itemName: String = ""; @State private var price: String = ""; @State private var deadline: Date = Date().addingTimeInterval(86400); @State private var isSubmitting = false
    let lyBlue = Color(red: 0/255, green: 49/255, blue: 83/255)
    
    var body: some View { ScrollView { VStack(alignment: .leading, spacing: 20) { inputSection(title: "團購主題", placeholder: "例如：下午茶訂購", text: $title); inputSection(title: "商品名稱", placeholder: "例如：珍珠奶茶", text: $itemName); VStack(alignment: .leading, spacing: 8) { Text("單價").font(.subheadline).foregroundColor(lyBlue).fontWeight(.bold); TextField("輸入金額...", text: $price).keyboardType(.numberPad).padding().background(Color.white).cornerRadius(8).overlay(RoundedRectangle(cornerRadius: 8).stroke(Color.gray.opacity(0.3), lineWidth: 1)) }; VStack(alignment: .leading, spacing: 8) { Text("截止時間").font(.subheadline).foregroundColor(lyBlue).fontWeight(.bold); DatePicker("", selection: $deadline).datePickerStyle(.compact).labelsHidden().padding(.vertical, 8) }; Button(action: submitGroupBuy) { HStack { if isSubmitting { ProgressView().tint(.white).padding(.trailing, 5) }; Text(isSubmitting ? "處理中..." : "發布團購") }.font(.headline).foregroundColor(.white).frame(maxWidth: .infinity).padding().background(lyBlue).cornerRadius(10).shadow(color: lyBlue.opacity(0.3), radius: 5, x: 0, y: 3) }.disabled(isSubmitting || title.isEmpty || itemName.isEmpty || price.isEmpty).padding(.top, 20) }.padding(20) }.background(Color(UIColor.systemGroupedBackground).ignoresSafeArea()).navigationTitle("發起團購").navigationBarTitleDisplayMode(.inline).onAppear { NotificationCenter.default.post(name: NSNotification.Name("HideTabBar"), object: nil) } }
    
    func inputSection(title: String, placeholder: String, text: Binding<String>) -> some View { VStack(alignment: .leading, spacing: 8) { Text(title).font(.subheadline).foregroundColor(lyBlue).fontWeight(.bold); TextField(placeholder, text: text).padding().background(Color.white).cornerRadius(8).overlay(RoundedRectangle(cornerRadius: 8).stroke(Color.gray.opacity(0.3), lineWidth: 1)) } }
    
    private func submitGroupBuy() { isSubmitting = true; let db = Firestore.firestore(); guard let uid = Auth.auth().currentUser?.uid else { return }; let userName = Auth.auth().currentUser?.displayName ?? "未知"; let deadlineStr = ISO8601DateFormatter().string(from: deadline); let payload: [String: Any] = [ "type": "group_buy", "title": title, "itemName": itemName, "price": Int(price) ?? 0, "deadline": deadlineStr, "initiatorUid": uid, "initiator": userName, "orders": [String: Any]() ]; let batch = db.batch(); let chatRef = db.collection("chats").document(chatId); let msgRef = chatRef.collection("messages").document(); if let jsonData = try? JSONSerialization.data(withJSONObject: payload), let jsonStr = String(data: jsonData, encoding: .utf8) { batch.setData(["text": jsonStr, "senderId": uid, "timestamp": FieldValue.serverTimestamp()], forDocument: msgRef); batch.updateData([ "lastMessage": "🛍️ 發起了團購：\(title)", "lastSenderId": uid, "updatedAt": FieldValue.serverTimestamp(), "lastReadAt.\(uid)": FieldValue.serverTimestamp() ], forDocument: chatRef); batch.commit { _ in DispatchQueue.main.async { isSubmitting = false; presentationMode.wrappedValue.dismiss() } } } }
}

// --- 機密文件 ---
struct ConfidentialListView: View {
    let chatId: String; @ObservedObject var viewModel: ChatRoomViewModel
    @State private var items: [ConfItem] = []; @State private var isLoading = true
    
    var body: some View { VStack(spacing: 0) { if isLoading { Spacer(); ProgressView("載入中..."); Spacer() } else if items.isEmpty { Spacer(); Text("尚無機密檔案，點擊右上角新增。").foregroundColor(.gray); Spacer() } else { List { Section { PrivilegeBanner(myName: viewModel.myName, color: .red) }; ForEach(items) { item in GenericListRow(title: item.title, subtitle: "無期限", creator: item.creatorName, icon: "lock.shield.fill", color: .red) { sendToChat(item) }.swipeActions(edge: .trailing, allowsFullSwipe: false) { if canEdit(creatorUid: item.creatorUid, myName: viewModel.myName) { Button(role: .destructive) { Firestore.firestore().collection("confidentials").document(item.id).delete() } label: { Label("刪除", systemImage: "trash.fill") } } else { Button { } label: { Label("權限不足", systemImage: "lock.fill") }.tint(.gray) } } } }.listStyle(.plain).background(Color(UIColor.systemGroupedBackground)) } }.background(Color(UIColor.systemGroupedBackground).ignoresSafeArea()).navigationTitle("機密檔案").navigationBarTitleDisplayMode(.inline).toolbar { ToolbarItem(placement: .navigationBarTrailing) { NavigationLink(destination: CreateConfidentialView(chatId: chatId)) { Image(systemName: "plus") } } }.onAppear { load(); NotificationCenter.default.post(name: NSNotification.Name("HideTabBar"), object: nil) }
    }
    
    private func load() { Firestore.firestore().collection("confidentials").order(by: "createdAt", descending: true).addSnapshotListener { snap, _ in self.items = (snap?.documents ?? []).map { doc in let data = doc.data(); return ConfItem(id: doc.documentID, title: data["title"] as? String ?? "機密檔案", creatorUid: data["creatorUid"] as? String ?? "", creatorName: data["creatorName"] as? String ?? "未知", rawData: data) }; self.isLoading = false } }
    private func sendToChat(_ item: ConfItem) { guard let uid = Auth.auth().currentUser?.uid else { return }; var payload = item.rawData; payload["type"] = "confidential"; if let jsonData = try? JSONSerialization.data(withJSONObject: payload), let str = String(data: jsonData, encoding: .utf8) { let db = Firestore.firestore(); let batch = db.batch(); let cRef = db.collection("chats").document(chatId); let mRef = cRef.collection("messages").document(); batch.setData(["text": str, "senderId": uid, "timestamp": FieldValue.serverTimestamp()], forDocument: mRef); batch.updateData(["lastMessage": "🔒 發送了一份機密文件", "lastSenderId": uid, "updatedAt": FieldValue.serverTimestamp()], forDocument: cRef); batch.commit() } }
}

struct CreateConfidentialView: View {
    let chatId: String; @Environment(\.presentationMode) var presentationMode
    @State private var title: String = ""; @State private var content: String = ""; @State private var unlockPassword: String = ""; @State private var isSubmitting = false
    let lyBlue = Color(red: 0/255, green: 49/255, blue: 83/255)
    
    var body: some View { ScrollView { VStack(alignment: .leading, spacing: 20) { VStack(alignment: .leading, spacing: 8) { Text("機密標題").font(.subheadline).foregroundColor(.red).fontWeight(.bold); TextField("例如：本月薪資單", text: $title).padding().background(Color.white).cornerRadius(8).overlay(RoundedRectangle(cornerRadius: 8).stroke(Color.red.opacity(0.3), lineWidth: 1)) }; VStack(alignment: .leading, spacing: 8) { Text("解鎖密碼").font(.subheadline).foregroundColor(.red).fontWeight(.bold); SecureField("請設定解鎖密碼...", text: $unlockPassword).padding().background(Color.white).cornerRadius(8).overlay(RoundedRectangle(cornerRadius: 8).stroke(Color.red.opacity(0.3), lineWidth: 1)) }; VStack(alignment: .leading, spacing: 8) { Text("機密內容").font(.subheadline).foregroundColor(.red).fontWeight(.bold); TextEditor(text: $content).frame(minHeight: 150).padding(8).background(Color.white).cornerRadius(8).overlay(RoundedRectangle(cornerRadius: 8).stroke(Color.red.opacity(0.3), lineWidth: 1)) }; Button(action: submitConfidential) { HStack { if isSubmitting { ProgressView().tint(.white).padding(.trailing, 5) } else { Image(systemName: "lock.shield.fill") }; Text(isSubmitting ? "加密中..." : "加密並發送") }.font(.headline).foregroundColor(.white).frame(maxWidth: .infinity).padding().background(Color.red).cornerRadius(10).shadow(color: Color.red.opacity(0.3), radius: 5, x: 0, y: 3) }.disabled(isSubmitting || title.isEmpty || content.isEmpty || unlockPassword.isEmpty).padding(.top, 20) }.padding(20) }.background(Color(UIColor.systemGroupedBackground).ignoresSafeArea()).navigationTitle("發送機密文件").navigationBarTitleDisplayMode(.inline).onAppear { NotificationCenter.default.post(name: NSNotification.Name("HideTabBar"), object: nil) } }
    
    private func submitConfidential() { isSubmitting = true; let db = Firestore.firestore(); guard let uid = Auth.auth().currentUser?.uid else { return }; let payload: [String: Any] = [ "type": "confidential", "title": title, "content": content, "password": unlockPassword ]; let batch = db.batch(); let chatRef = db.collection("chats").document(chatId); let msgRef = chatRef.collection("messages").document(); if let jsonData = try? JSONSerialization.data(withJSONObject: payload), let jsonStr = String(data: jsonData, encoding: .utf8) { batch.setData(["text": jsonStr, "senderId": uid, "timestamp": FieldValue.serverTimestamp()], forDocument: msgRef); batch.updateData([ "lastMessage": "🔒 發送了一份機密文件", "lastSenderId": uid, "updatedAt": FieldValue.serverTimestamp(), "lastReadAt.\(uid)": FieldValue.serverTimestamp() ], forDocument: chatRef); batch.commit { _ in DispatchQueue.main.async { isSubmitting = false; presentationMode.wrappedValue.dismiss() } } } }
}

// MARK: - 8. 擴充功能詳細頁 (Card Detail Views)

struct EventDetailView: View {
    var cardData: [String: Any]; @ObservedObject var viewModel: ChatRoomViewModel
    @State private var liveEventData: [String: Any]? = nil
    let lyBlue = Color(red: 0/255, green: 49/255, blue: 83/255)
    let lyGold = Color(red: 184/255, green: 134/255, blue: 11/255)
    var currentData: [String: Any] { return liveEventData ?? cardData["fullEventData"] as? [String: Any] ?? cardData }

    var body: some View {
        let ev = currentData
        let title = ev["title"] as? String ?? "未命名活動"
        let fields = ev["fields"] as? [[String: Any]] ?? []
        let timeField = fields.first(where: { $0["type"] as? String == "time" })?["content"] as? String ?? "未提供"
        let meetupField = fields.first(where: { $0["type"] as? String == "meetup" })?["content"] as? String ?? "未提供"
        let locField = fields.first(where: { $0["type"] as? String == "location" })?["content"] as? String ?? "未提供"
        let contentField = fields.first(where: { $0["type"] as? String == "content_text" })?["content"] as? String ?? "無詳細內容"

        let pUids = parseOldArrayData(from: ev["participants"])
        let pNames = parseOldArrayData(from: ev["participantNames"])
        let finalParticipants = !pNames.isEmpty ? pNames : pUids
        let hasJoined = finalParticipants.contains(viewModel.myName) || finalParticipants.contains(Auth.auth().currentUser?.uid ?? "")

        ScrollView {
            VStack(alignment: .leading, spacing: 20) {
                Text(title).font(.title2).fontWeight(.black).foregroundColor(lyBlue).multilineTextAlignment(.center).frame(maxWidth: .infinity).padding(.top, 20)

                VStack(alignment: .leading, spacing: 25) {
                    detailSection(title: "活動時間", content: timeField.replacingOccurrences(of: "T", with: " "))
                    detailSection(title: "集合時間", content: meetupField.replacingOccurrences(of: "T", with: " "))
                    detailSection(title: "地點", content: locField)
                    detailSection(title: "活動內容", content: contentField)

                    VStack(spacing: 0) {
                        Text("請確認行程").font(.headline).foregroundColor(lyGold).frame(maxWidth: .infinity).padding(.vertical, 12).background(lyBlue)
                        HStack(spacing: 15) {
                            Button(action: { toggleParticipation(isJoin: true) }) { HStack { if hasJoined { Image(systemName: "checkmark") }; Text(hasJoined ? "已參與" : "參與") }.fontWeight(.bold).frame(maxWidth: .infinity).padding(.vertical, 12).background(hasJoined ? LinearGradient(gradient: Gradient(colors: [Color(red: 249/255, green: 226/255, blue: 125/255), Color(red: 212/255, green: 175/255, blue: 55/255)]), startPoint: .topLeading, endPoint: .bottomTrailing) : LinearGradient(gradient: Gradient(colors: [Color(UIColor.systemGray5)]), startPoint: .top, endPoint: .bottom)).foregroundColor(hasJoined ? .white : .gray).cornerRadius(25) }
                            Button(action: { toggleParticipation(isJoin: false) }) { Text("下次").fontWeight(.bold).frame(maxWidth: .infinity).padding(.vertical, 12).background(Color.gray.opacity(0.6)).foregroundColor(.white).cornerRadius(25) }
                        }.padding(20).background(Color.white)
                    }.cornerRadius(12).overlay(RoundedRectangle(cornerRadius: 12).stroke(Color.gray.opacity(0.2), lineWidth: 1)).padding(.vertical, 10)

                    VStack(alignment: .leading, spacing: 15) {
                        HStack { Rectangle().fill(lyGold).frame(width: 4, height: 16); Text("參與成員").font(.headline).fontWeight(.bold).foregroundColor(lyBlue); Text("(共 \(finalParticipants.count) 人)").font(.subheadline).foregroundColor(.gray) }
                        if finalParticipants.isEmpty { Text("尚無成員參與...").foregroundColor(.gray).font(.subheadline) }
                        else { LazyVGrid(columns: [GridItem(.adaptive(minimum: 60))], spacing: 15) { ForEach(finalParticipants, id: \.self) { identifier in LiveAvatarView(uidOrName: identifier, showName: true, size: 50) } } }
                    }
                }.padding(.horizontal, 20)
            }.padding(.bottom, 40)
        }.background(Color.white.ignoresSafeArea()).navigationTitle("活動詳情").navigationBarTitleDisplayMode(.inline).onAppear { startListeningToEvent(); NotificationCenter.default.post(name: NSNotification.Name("HideTabBar"), object: nil) }
    }

    private func startListeningToEvent() { let eventId = cardData["eventId"] as? String ?? ""; if eventId.isEmpty { return }; Firestore.firestore().collection("events").document(eventId).addSnapshotListener { doc, _ in if let data = doc?.data() { self.liveEventData = data } } }
    private func toggleParticipation(isJoin: Bool) { let eventId = cardData["eventId"] as? String ?? ""; guard let uid = Auth.auth().currentUser?.uid else { return }; if eventId.isEmpty { return }; let ref = Firestore.firestore().collection("events").document(eventId); Firestore.firestore().runTransaction({ (transaction, errorPointer) -> Any? in let doc: DocumentSnapshot; do { doc = try transaction.getDocument(ref) } catch { return nil }; guard let data = doc.data() else { return nil }; var parts = parseOldArrayData(from: data["participants"]); var names = parseOldArrayData(from: data["participantNames"]); let myName = viewModel.myName; if isJoin { if !parts.contains(uid) { parts.append(uid) }; if !names.contains(myName) { names.append(myName) } } else { parts.removeAll(where: { $0 == uid }); names.removeAll(where: { $0 == myName }) }; transaction.updateData(["participants": parts, "participantNames": names], forDocument: ref); return nil }) { _, _ in } }
    func detailSection(title: String, content: String) -> some View { VStack(alignment: .leading, spacing: 8) { Text("[ \(title) ]").font(.headline).fontWeight(.black).foregroundColor(lyBlue); Text(content).font(.body).foregroundColor(.primary).lineSpacing(5) } }
}

struct PollDetailView: View {
    let msgId: String; var cardData: [String: Any]; @ObservedObject var viewModel: ChatRoomViewModel
    @State private var showVotersModal = false; let lyBlue = Color(red: 0/255, green: 49/255, blue: 83/255); let lyGold = Color(red: 184/255, green: 134/255, blue: 11/255)

    var body: some View {
        let options = cardData["options"] as? [String] ?? []; let votes = cardData["votes"] as? [String: [String]] ?? [:]; let totalVotes = votes.values.reduce(0) { $0 + $1.count }; let deadline = cardData["deadline"] as? String ?? ""; let currentUid = Auth.auth().currentUser?.uid ?? ""; let isExpired = !deadline.isEmpty && (Date() > { let formatter = DateFormatter(); formatter.dateFormat = "yyyy-MM-dd'T'HH:mm"; return formatter.date(from: deadline) ?? Date.distantFuture }())

        ScrollView { VStack(alignment: .leading, spacing: 20) {
            HStack { Rectangle().fill(lyGold).frame(width: 5, height: 25); Text(cardData["question"] as? String ?? "投票").font(.title2).fontWeight(.black).foregroundColor(lyBlue) }.padding(.horizontal, 20).padding(.top, 20); if let content = cardData["content"] as? String, !content.isEmpty { Text(content).foregroundColor(.gray).padding(.horizontal, 20) }
            VStack(spacing: 12) { ForEach(options, id: \.self) { opt in let optVotes = votes[opt] ?? []; let hasVoted = optVotes.contains(currentUid); let percentage = totalVotes > 0 ? Int((Double(optVotes.count) / Double(totalVotes)) * 100) : 0; GeometryReader { geo in ZStack(alignment: .leading) { RoundedRectangle(cornerRadius: 8).fill(hasVoted ? Color(red: 253/255, green: 247/255, blue: 213/255) : Color(UIColor.systemGray6)); if percentage > 0 { RoundedRectangle(cornerRadius: 8).fill(hasVoted ? lyGold.opacity(0.15) : Color.gray.opacity(0.1)).frame(width: geo.size.width * CGFloat(percentage) / 100) }; HStack { if hasVoted { Image(systemName: "checkmark.circle.fill").foregroundColor(lyGold) }; Text(opt).fontWeight(hasVoted ? .bold : .regular).foregroundColor(lyBlue); Spacer(); if !optVotes.isEmpty { HStack(spacing: -10) { ForEach(optVotes.prefix(3), id: \.self) { vUid in LiveAvatarView(uidOrName: vUid, showName: false, size: 24) }; if optVotes.count > 3 { Text("+\(optVotes.count - 3)").font(.system(size: 10, weight: .bold)).foregroundColor(.gray).frame(width: 24, height: 24).background(Color.white).clipShape(Circle()).overlay(Circle().stroke(Color.gray.opacity(0.3), lineWidth: 1)) } } }; Text("\(optVotes.count) 票 (\(percentage)%)").font(.subheadline).foregroundColor(.gray).fontWeight(.bold) }.padding() }.overlay(RoundedRectangle(cornerRadius: 8).stroke(hasVoted ? lyGold : Color.gray.opacity(0.2), lineWidth: 1)).contentShape(Rectangle()) .onTapGesture { if isExpired { print("投票已截止！") } else { viewModel.castVote(msgId: msgId, option: opt) } } }.frame(height: 50) }; Button(action: { showVotersModal = true }) { HStack { Image(systemName: "person.2.fill"); Text("查看投票成員") }.font(.headline).foregroundColor(.white).frame(maxWidth: .infinity).padding().background(Color(red: 25/255, green: 38/255, blue: 50/255)).cornerRadius(10).shadow(radius: 3) }.padding(.top, 10) }.padding(.horizontal, 20)
            VStack(alignment: .leading, spacing: 10) { infoRow(icon: "person.fill", title: "建立人", value: cardData["creatorName"] as? String ?? "未提供"); HStack { infoRow(icon: "hourglass", title: "結束時間", value: deadline.isEmpty ? "無期限" : deadline.replacingOccurrences(of: "T", with: " ")); Spacer(); if isExpired { Text("(已截止)").foregroundColor(.gray).fontWeight(.bold) } } }.padding().background(Color(red: 253/255, green: 252/255, blue: 245/255)).cornerRadius(12).padding(.horizontal, 20).padding(.top, 20)
        }.padding(.bottom, 40) }.background(Color(UIColor.systemGroupedBackground).ignoresSafeArea()).navigationTitle("投票詳情").navigationBarTitleDisplayMode(.inline).sheet(isPresented: $showVotersModal) { PollVotersView(options: options, votes: votes) }.onAppear { NotificationCenter.default.post(name: NSNotification.Name("HideTabBar"), object: nil) }
    }
    func infoRow(icon: String, title: String, value: String) -> some View { HStack(spacing: 8) { Image(systemName: icon).foregroundColor(lyGold).frame(width: 20); Text("\(title) :").fontWeight(.bold).foregroundColor(.gray); Text(value).foregroundColor(.gray) }.font(.subheadline) }
}

struct PollVotersView: View {
    var options: [String]; var votes: [String: [String]]; @Environment(\.presentationMode) var presentationMode
    var body: some View { NavigationView { List { ForEach(options, id: \.self) { opt in let uids = votes[opt] ?? []; Section(header: Text("\(opt) (\(uids.count)票)").font(.headline).foregroundColor(.primary)) { if uids.isEmpty { Text("尚無人投票").foregroundColor(.gray).font(.subheadline) } else { ForEach(uids, id: \.self) { uid in LiveUserRow(uid: uid) } } } } }.navigationTitle("投票成員名單").navigationBarTitleDisplayMode(.inline).toolbar { ToolbarItem(placement: .navigationBarTrailing) { Button("關閉") { presentationMode.wrappedValue.dismiss() } } } } }
}

struct GroupBuyDetailView: View {
    var cardData: [String: Any]; let lyBlue = Color(red: 0/255, green: 49/255, blue: 83/255); let lyGold = Color(red: 184/255, green: 134/255, blue: 11/255)
    var body: some View { VStack(spacing: 0) { VStack(alignment: .leading, spacing: 10) { HStack(spacing: 8) { Image(systemName: "person.fill.badge.plus").foregroundColor(lyGold); Text("代購成員 :").foregroundColor(.gray); Text(cardData["initiator"] as? String ?? "未提供").fontWeight(.bold).foregroundColor(lyBlue) }; HStack(spacing: 8) { Image(systemName: "clock.fill").foregroundColor(lyGold); Text("截止時間 :").foregroundColor(.gray); Text(cardData["deadline"] as? String ?? "未提供").fontWeight(.bold).foregroundColor(.red) } }.padding().frame(maxWidth: .infinity, alignment: .leading).background(Color(red: 253/255, green: 252/255, blue: 245/255)).border(Color.gray.opacity(0.2), width: 1); ScrollView { VStack(spacing: 20) { VStack(spacing: 15) { HStack(alignment: .top) { Image(systemName: "photo").resizable().padding().frame(width: 60, height: 60).foregroundColor(.gray).background(Color.white).cornerRadius(8).overlay(RoundedRectangle(cornerRadius: 8).stroke(Color.gray.opacity(0.3))); VStack(alignment: .leading, spacing: 5) { Text("此處將由 API 帶入").font(.headline).foregroundColor(lyBlue); Text("等待功能開發中...").font(.caption).padding(.horizontal, 8).padding(.vertical, 3).background(Color(UIColor.systemGray6)).cornerRadius(4) }; Spacer() } }.padding().background(Color.white).cornerRadius(12) }.padding() } }.background(Color(UIColor.systemGroupedBackground).ignoresSafeArea()).navigationTitle(cardData["title"] as? String ?? "團購詳情").navigationBarTitleDisplayMode(.inline).onAppear { NotificationCenter.default.post(name: NSNotification.Name("HideTabBar"), object: nil) } }
}

struct ConfidentialDetailView: View {
    var cardData: [String: Any]; @State private var inputPassword = ""; @State private var isUnlocked = false; @State private var showError = false; let lyBlue = Color(red: 0/255, green: 49/255, blue: 83/255)
    var body: some View { VStack(spacing: 25) { Image(systemName: isUnlocked ? "lock.open.fill" : "lock.fill").font(.system(size: 65)).foregroundColor(isUnlocked ? .green : .red).padding(.top, 40); Text(cardData["title"] as? String ?? "機密檔案").font(.title).fontWeight(.black).foregroundColor(lyBlue); if isUnlocked { ScrollView { Text(cardData["content"] as? String ?? "這是機密內容...").font(.body).padding(20).frame(maxWidth: .infinity, alignment: .leading).background(Color.white).cornerRadius(12).shadow(color: Color.black.opacity(0.05), radius: 5, x: 0, y: 2) } } else { Text("此為受保護的機密文件，請輸入存取密碼。").font(.subheadline).foregroundColor(.gray); SecureField("輸入密碼...", text: $inputPassword).padding().background(Color.white).cornerRadius(10).overlay(RoundedRectangle(cornerRadius: 10).stroke(showError ? Color.red : Color.gray.opacity(0.3), lineWidth: 1)).padding(.horizontal, 30); if showError { Text("密碼錯誤，請重新輸入！").foregroundColor(.red).font(.caption) }; Button(action: unlock) { Text("確認解鎖").fontWeight(.bold).foregroundColor(.white).frame(maxWidth: .infinity).padding(.vertical, 14).background(lyBlue).cornerRadius(10) }.padding(.horizontal, 30) }; Spacer() }.padding().background(Color(UIColor.systemGroupedBackground).ignoresSafeArea()).navigationTitle("機密文件").navigationBarTitleDisplayMode(.inline).onAppear { NotificationCenter.default.post(name: NSNotification.Name("HideTabBar"), object: nil) } }
    func unlock() { let correctPassword = cardData["password"] as? String ?? "1234"; if inputPassword == correctPassword { withAnimation(.spring()) { isUnlocked = true; showError = false } } else { withAnimation(.default) { showError = true; inputPassword = "" } } }
}

```
### Ver.2
```
import SwiftUI
import Combine
import AVFoundation
import AVKit
import WebKit
import PhotosUI
import FirebaseCore
import FirebaseAuth
import FirebaseFirestore
import FirebaseStorage

// MARK: - 1. 應用程式入口 (App Entry)
@main
struct LegislatureMessengerApp: App {
    init() {
        FirebaseApp.configure()
    }
    
    var body: some Scene {
        WindowGroup {
            ContentView()
        }
    }
}

// MARK: - 2. 網址判斷擴充引擎
extension String {
    var isImageURL: Bool {
        let trimmed = self.trimmingCharacters(in: .whitespacesAndNewlines).lowercased()
        guard trimmed.hasPrefix("http://") || trimmed.hasPrefix("https://") else { return false }
        return trimmed.hasSuffix(".png") || trimmed.hasSuffix(".jpg") || trimmed.hasSuffix(".jpeg") || trimmed.hasSuffix(".gif") || trimmed.hasSuffix(".webp")
    }
    
    var isGifURL: Bool {
        let trimmed = self.trimmingCharacters(in: .whitespacesAndNewlines).lowercased()
        return trimmed.hasSuffix(".gif")
    }
}

// MARK: - 3. 資料模型 (Models)
struct ChatItem: Identifiable {
    var id: String
    var name: String
    var lastMessage: String
    var time: String
    var isUnread: Bool
    var isGroup: Bool
    var avatarUrl: String
}

struct DeviceSession: Identifiable {
    let id: String
    let ip: String
    let location: String
    let lastLogin: String
    let isCurrent: Bool
}

struct ReadMember: Identifiable, Equatable {
    let id: String
    let name: String
    let avatar: String
}

struct ReactionSheetItem: Identifiable {
    let id = UUID()
    let emoji: String
    let uids: [String]
}

struct MessageItem: Identifiable {
    var id: String
    var text: String
    var senderId: String
    var timestamp: Date
    var isMine: Bool
    var isSystem: Bool = false
    var isDeleted: Bool = false
    var senderAvatar: String = ""
    var senderName: String = ""
    var fileUrl: String = ""
    var fileType: String = ""
    var cardType: String = ""
    var cardData: [String: Any]?
    var readBy: [ReadMember] = []
    var replyToText: String = ""
    var reactions: [String: [String]] = [:]
}

struct EventItem: Identifiable {
    var id: String
    var title: String
    var time: String
    var creatorUid: String
    var creatorName: String
    var rawData: [String: Any]
}

struct PollItem: Identifiable {
    var id: String
    var question: String
    var content: String
    var deadline: String
    var creatorUid: String
    var creatorName: String
    var allowAddOptions: Bool
    var options: [String]
}

struct GBItem: Identifiable {
    var id: String
    var title: String
    var deadline: String
    var creatorUid: String
    var creatorName: String
    var rawData: [String: Any]
}

struct ConfItem: Identifiable {
    var id: String
    var title: String
    var creatorUid: String
    var creatorName: String
    var rawData: [String: Any]
}

// MARK: - 4. 視圖模型與輔助引擎 (ViewModels & Helpers)

// 👉 記憶體秒開技術
class PersistentImageCache {
    static let shared = PersistentImageCache()
    private let fileManager = FileManager.default
    private let cacheDirectory: URL
    private let memoryCache = NSCache<NSString, UIImage>()

    init() {
        cacheDirectory = fileManager.urls(for: .cachesDirectory, in: .userDomainMask)[0].appendingPathComponent("UserAvatars")
        if !fileManager.fileExists(atPath: cacheDirectory.path) {
            try? fileManager.createDirectory(at: cacheDirectory, withIntermediateDirectories: true)
        }
        memoryCache.totalCostLimit = 1024 * 1024 * 50 // 50MB
    }

    func getImage(for urlString: String) -> UIImage? {
        let cacheKey = NSString(string: urlString)
        if let memImage = memoryCache.object(forKey: cacheKey) { return memImage }
        
        let fileName = urlString.addingPercentEncoding(withAllowedCharacters: .alphanumerics) ?? UUID().uuidString
        let fileURL = cacheDirectory.appendingPathComponent(fileName)
        if let data = try? Data(contentsOf: fileURL), let diskImage = UIImage(data: data) {
            memoryCache.setObject(diskImage, forKey: cacheKey)
            return diskImage
        }
        return nil
    }

    func saveImage(_ image: UIImage, for urlString: String) {
        let cacheKey = NSString(string: urlString)
        let fileName = urlString.addingPercentEncoding(withAllowedCharacters: .alphanumerics) ?? UUID().uuidString
        let fileURL = cacheDirectory.appendingPathComponent(fileName)
        
        memoryCache.setObject(image, forKey: cacheKey)
        DispatchQueue.global(qos: .background).async {
            if let data = image.jpegData(compressionQuality: 0.8) { try? data.write(to: fileURL) }
        }
    }
}

class ImageLoader: ObservableObject {
    @Published var image: UIImage?
    private let urlString: String

    init(url: String) {
        self.urlString = url
        loadImage()
    }

    private func loadImage() {
        guard let url = URL(string: urlString) else { return }
        let safeFileName = urlString.addingPercentEncoding(withAllowedCharacters: .alphanumerics) ?? UUID().uuidString
        let cachePath = FileManager.default.urls(for: .cachesDirectory, in: .userDomainMask)[0]
        let fileURL = cachePath.appendingPathComponent(safeFileName)

        if let data = try? Data(contentsOf: fileURL), let uiImage = UIImage(data: data) {
            self.image = uiImage
            return
        }

        URLSession.shared.dataTask(with: url) { data, response, error in
            guard let data = data, let uiImage = UIImage(data: data) else { return }
            try? data.write(to: fileURL)
            DispatchQueue.main.async {
                self.image = uiImage
            }
        }.resume()
    }
}

class AudioRecorder: NSObject, ObservableObject, AVAudioRecorderDelegate {
    var audioRecorder: AVAudioRecorder?
    @Published var isRecording = false
    var recordingURL: URL?

    func startRecording() {
        let session = AVAudioSession.sharedInstance()
        try? session.setCategory(.playAndRecord, mode: .default)
        try? session.setActive(true)

        let cachePath = FileManager.default.urls(for: .cachesDirectory, in: .userDomainMask)[0]
        let audioFilename = cachePath.appendingPathComponent(UUID().uuidString + ".m4a")
        recordingURL = audioFilename

        let settings = [
            AVFormatIDKey: Int(kAudioFormatMPEG4AAC),
            AVSampleRateKey: 12000,
            AVNumberOfChannelsKey: 1,
            AVEncoderAudioQualityKey: AVAudioQuality.high.rawValue
        ]

        do {
            audioRecorder = try AVAudioRecorder(url: audioFilename, settings: settings)
            audioRecorder?.delegate = self
            audioRecorder?.record()
            isRecording = true
        } catch {
            print("錄音失敗")
        }
    }

    func stopRecording() -> URL? {
        audioRecorder?.stop()
        isRecording = false
        return recordingURL
    }
}

class ChatViewModel: ObservableObject {
    @Published var chats: [ChatItem] = []
    private var db = Firestore.firestore()
    
    func fetchChats() {
        guard let currentUser = Auth.auth().currentUser else { return }
        let currentUid = currentUser.uid
        
        db.collection("chats")
            .whereField("members", arrayContains: currentUid)
            .order(by: "updatedAt", descending: true)
            .addSnapshotListener { querySnapshot, error in
                guard let documents = querySnapshot?.documents else { return }
                
                var fetchedChats = documents.compactMap { doc -> ChatItem? in
                    let data = doc.data()
                    let isGroup = data["isGroup"] as? Bool ?? false
                    let lastMessage = data["lastMessage"] as? String ?? "尚無訊息"
                    let lastSenderId = data["lastSenderId"] as? String ?? ""
                    
                    var timeStr = ""
                    var msgDate = Date()
                    if let timestamp = data["updatedAt"] as? Timestamp {
                        msgDate = timestamp.dateValue()
                        let formatter = DateFormatter()
                        formatter.dateFormat = Calendar.current.isDateInToday(msgDate) ? "HH:mm" : "MM/dd"
                        timeStr = formatter.string(from: msgDate)
                    }
                    
                    var isUnread = false
                    if lastSenderId != currentUid {
                        let readDict = data["lastReadAt"] as? [String: Timestamp]
                        let myLastRead = readDict?[currentUid]?.dateValue() ?? Date(timeIntervalSince1970: 0)
                        if msgDate > myLastRead { isUnread = true }
                    }
                    
                    let name = isGroup ? (data["groupName"] as? String ?? "未命名群組") : "載入中..."
                    let avatarUrl = isGroup ? (data["groupAvatar"] as? String ?? "") : ""
                    
                    return ChatItem(id: doc.documentID, name: name, lastMessage: lastMessage, time: timeStr, isUnread: isUnread, isGroup: isGroup, avatarUrl: avatarUrl)
                }
                
                fetchedChats.sort { chat1, chat2 in
                    if chat1.isGroup && chat1.name == "立法院 Legislature" { return true }
                    if chat2.isGroup && chat2.name == "立法院 Legislature" { return false }
                    return false
                }
                
                self.chats = fetchedChats
                
                for (index, doc) in documents.enumerated() {
                    let data = doc.data()
                    let isGroup = data["isGroup"] as? Bool ?? false
                    if !isGroup {
                        let members = data["members"] as? [String] ?? []
                        let targetUid = members.first(where: { $0 != currentUid }) ?? ""
                        if !targetUid.isEmpty {
                            self.db.collection("act").document(targetUid).getDocument { userDoc, _ in
                                if let userData = userDoc?.data() {
                                    let realName = userData["displayName"] as? String ?? "未知成員"
                                    let realAvatar = userData["photoURL"] as? String ?? ""
                                    DispatchQueue.main.async {
                                        if let i = self.chats.firstIndex(where: { $0.id == doc.documentID }) {
                                            self.chats[i].name = realName
                                            self.chats[i].avatarUrl = realAvatar
                                        }
                                    }
                                }
                            }
                        }
                    }
                }
            }
    }
}

class ChatRoomViewModel: ObservableObject {
    @Published var messages: [MessageItem] = []
    @Published var isLoadingOldMessages = true
    
    @Published var groupReadTimes: [String: Date] = [:]
    @Published var groupMembersCount: Int = 0
    @Published var pinnedMessageText: String = ""
    @Published var activeMenuMsgId: String? = nil
    
    @Published var myName: String = "自己"
    @Published var myAvatar: String = ""
    @Published var targetAvatar: String = ""
    @Published var isGroup: Bool = false
    
    private var db = Firestore.firestore()
    let chatId: String
    private var userCache: [String: (name: String, avatar: String)] = [:]
    
    var messageLimit = 20
    private var listener: ListenerRegistration?
    private var chatListener: ListenerRegistration?
    
    init(chatId: String) {
        self.chatId = chatId
        fetchMyProfile()
        fetchChatData()
        loadMessages()
    }
    
    deinit {
        listener?.remove()
        chatListener?.remove()
    }

    private func fetchMyProfile() {
        guard let uid = Auth.auth().currentUser?.uid else { return }
        db.collection("act").document(uid).getDocument { [weak self] doc, _ in
            if let data = doc?.data() {
                self?.myName = data["displayName"] as? String ?? "自己"
                self?.myAvatar = data["photoURL"] as? String ?? ""
            }
        }
    }

    func fetchUserInfo(uid: String) {
        if userCache[uid] != nil { return }
        db.collection("act").document(uid).getDocument { [weak self] doc, _ in
            if let data = doc?.data() {
                let name = data["displayName"] as? String ?? "未知"
                let avatar = data["photoURL"] as? String ?? ""
                self?.userCache[uid] = (name: name, avatar: avatar)
                self?.recalculateReadStatus()
                self?.updateMessagesUserInfo()
            }
        }
    }

    func fetchChatData() {
        guard let currentUid = Auth.auth().currentUser?.uid else { return }
        chatListener = db.collection("chats").document(chatId).addSnapshotListener { [weak self] doc, _ in
            guard let self = self, let data = doc?.data() else { return }
            
            self.isGroup = data["isGroup"] as? Bool ?? false
            self.pinnedMessageText = (data["pinnedMessage"] as? [String: Any])?["text"] as? String ?? ""
            
            let members = data["members"] as? [String] ?? []
            self.groupMembersCount = members.count
            
            if self.isGroup {
                self.targetAvatar = data["groupAvatar"] as? String ?? ""
            } else {
                if let otherUid = members.first(where: { $0 != currentUid }) {
                    self.fetchUserInfo(uid: otherUid)
                }
            }
            
            if let readDict = data["lastReadAt"] as? [String: Timestamp] {
                var newReadTimes: [String: Date] = [:]
                for (uid, timestamp) in readDict {
                    newReadTimes[uid] = timestamp.dateValue()
                    self.fetchUserInfo(uid: uid)
                }
                self.groupReadTimes = newReadTimes
                self.recalculateReadStatus()
            }
        }
    }

    func recalculateReadStatus() {
        var updatedMessages = self.messages
        for i in 0..<updatedMessages.count {
            let msg = updatedMessages[i]
            var readers: [ReadMember] = []
            
            for (uid, timestamp) in self.groupReadTimes {
                if uid != msg.senderId && timestamp >= msg.timestamp {
                    let name = self.userCache[uid]?.name ?? "未知"
                    let avatar = self.userCache[uid]?.avatar ?? ""
                    readers.append(ReadMember(id: uid, name: name, avatar: avatar))
                }
            }
            updatedMessages[i].readBy = readers
        }
        self.messages = updatedMessages
    }

    private func updateMessagesUserInfo() {
        var updatedMessages = self.messages
        for i in 0..<updatedMessages.count {
            let senderId = updatedMessages[i].senderId
            if let info = userCache[senderId] {
                updatedMessages[i].senderName = info.name
                updatedMessages[i].senderAvatar = info.avatar
            }
        }
        self.messages = updatedMessages
    }

    func loadMessages() {
        guard let currentUid = Auth.auth().currentUser?.uid else { return }
        listener = db.collection("chats").document(chatId).collection("messages")
            .order(by: "timestamp", descending: true)
            .limit(to: messageLimit)
            .addSnapshotListener { [weak self] snap, error in
                guard let self = self, let docs = snap?.documents else { return }
                
                var newMessages: [MessageItem] = []
                for doc in docs {
                    let data = doc.data()
                    let senderId = data["senderId"] as? String ?? ""
                    let timestamp = (data["timestamp"] as? Timestamp)?.dateValue() ?? Date()
                    let isMine = senderId == currentUid
                    
                    var senderName = ""
                    var senderAvatar = ""
                    
                    if isMine {
                        senderName = self.myName
                        senderAvatar = self.myAvatar
                    } else if let cached = self.userCache[senderId] {
                        senderName = cached.name
                        senderAvatar = cached.avatar
                    } else {
                        self.fetchUserInfo(uid: senderId)
                    }
                    
                    var text = data["text"] as? String ?? ""
                    var cType = data["cardType"] as? String
                    var cData = data["cardData"] as? [String: Any]
                    
                    if (cType == nil || cType!.isEmpty) && text.hasPrefix("{") && text.contains("\"type\"") {
                        if let textData = text.data(using: .utf8),
                           let parsed = try? JSONSerialization.jsonObject(with: textData) as? [String: Any],
                           let type = parsed["type"] as? String {
                            cType = type
                            cData = parsed
                            text = ""
                        }
                    }
                    
                    let msg = MessageItem(
                        id: doc.documentID,
                        text: text,
                        senderId: senderId,
                        timestamp: timestamp,
                        isMine: isMine,
                        isSystem: data["isSystem"] as? Bool ?? false,
                        isDeleted: data["isDeleted"] as? Bool ?? false,
                        senderAvatar: senderAvatar,
                        senderName: senderName,
                        fileUrl: data["fileUrl"] as? String ?? "",
                        fileType: data["fileType"] as? String ?? "",
                        cardType: cType ?? "",
                        cardData: cData,
                        readBy: [],
                        replyToText: data["replyToText"] as? String ?? "",
                        reactions: data["reactions"] as? [String: [String]] ?? [:]
                    )
                    newMessages.append(msg)
                }
                
                self.messages = newMessages.reversed()
                self.recalculateReadStatus()
                self.isLoadingOldMessages = false
                self.markAsRead()
            }
    }

    func loadMore() {
        messageLimit += 20
        loadMessages()
    }

    func markAsRead() {
        guard let uid = Auth.auth().currentUser?.uid else { return }
        db.collection("chats").document(chatId).updateData([
            "lastReadAt.\(uid)": FieldValue.serverTimestamp()
        ])
    }

    func sendMessage(text: String, replyTo: String = "") {
        guard let uid = Auth.auth().currentUser?.uid else { return }
        let msgRef = db.collection("chats").document(chatId).collection("messages").document()
        
        var data: [String: Any] = [
            "text": text,
            "senderId": uid,
            "timestamp": FieldValue.serverTimestamp()
        ]
        if !replyTo.isEmpty { data["replyToText"] = replyTo }
        
        let batch = db.batch()
        batch.setData(data, forDocument: msgRef)
        batch.updateData([
            "lastMessage": text,
            "lastSenderId": uid,
            "updatedAt": FieldValue.serverTimestamp(),
            "lastReadAt.\(uid)": FieldValue.serverTimestamp()
        ], forDocument: db.collection("chats").document(chatId))
        batch.commit()
    }

    func uploadImage(_ image: UIImage) {
        guard let uid = Auth.auth().currentUser?.uid, let imageData = image.jpegData(compressionQuality: 0.5) else { return }
        let ref = Storage.storage().reference().child("chat_images/\(UUID().uuidString).jpg")
        ref.putData(imageData, metadata: nil) { [weak self] _, _ in
            ref.downloadURL { url, _ in
                if let urlString = url?.absoluteString {
                    self?.sendMediaMessage(fileUrl: urlString, fileType: "image", text: "[圖片]")
                }
            }
        }
    }

    func uploadAudio(_ url: URL) {
        guard let uid = Auth.auth().currentUser?.uid else { return }
        let ref = Storage.storage().reference().child("chat_audio/\(UUID().uuidString).m4a")
        ref.putFile(from: url, metadata: nil) { [weak self] _, _ in
            ref.downloadURL { downloadUrl, _ in
                if let urlString = downloadUrl?.absoluteString {
                    self?.sendMediaMessage(fileUrl: urlString, fileType: "audio", text: "[語音訊息]")
                }
            }
        }
    }

    private func sendMediaMessage(fileUrl: String, fileType: String, text: String) {
        guard let uid = Auth.auth().currentUser?.uid else { return }
        let msgRef = db.collection("chats").document(chatId).collection("messages").document()
        let batch = db.batch()
        batch.setData([
            "text": text,
            "senderId": uid,
            "timestamp": FieldValue.serverTimestamp(),
            "fileUrl": fileUrl,
            "fileType": fileType
        ], forDocument: msgRef)
        batch.updateData([
            "lastMessage": text,
            "lastSenderId": uid,
            "updatedAt": FieldValue.serverTimestamp(),
            "lastReadAt.\(uid)": FieldValue.serverTimestamp()
        ], forDocument: db.collection("chats").document(chatId))
        batch.commit()
    }

    func revokeMessage(messageId: String) {
        db.collection("chats").document(chatId).collection("messages").document(messageId).updateData([
            "isDeleted": true,
            "text": "此訊息已收回",
            "fileUrl": "",
            "fileType": "",
            "cardType": "",
            "cardData": FieldValue.delete()
        ])
    }

    func hideMessage(messageId: String) {
        if let index = messages.firstIndex(where: { $0.id == messageId }) {
            messages.remove(at: index)
        }
    }

    func pinMessage(text: String) {
        db.collection("chats").document(chatId).updateData([
            "pinnedMessage": ["text": text, "timestamp": FieldValue.serverTimestamp()]
        ])
    }

    func unpinMessage() {
        db.collection("chats").document(chatId).updateData([
            "pinnedMessage": FieldValue.delete()
        ])
    }

    func addReaction(messageId: String, emoji: String) {
        guard let uid = Auth.auth().currentUser?.uid else { return }
        let msgRef = db.collection("chats").document(chatId).collection("messages").document(messageId)
        
        db.runTransaction({ (transaction, errorPointer) -> Any? in
            let doc: DocumentSnapshot
            do { doc = try transaction.getDocument(msgRef) } catch let error as NSError { errorPointer?.pointee = error; return nil }
            guard let data = doc.data() else { return nil }
            
            var reactions = data["reactions"] as? [String: [String]] ?? [:]
            for (key, uids) in reactions { reactions[key] = uids.filter { $0 != uid } }
            
            if let currentUids = (data["reactions"] as? [String: [String]])?[emoji], currentUids.contains(uid) {
                // 取消反應
            } else {
                if reactions[emoji] == nil { reactions[emoji] = [] }
                reactions[emoji]?.append(uid)
            }
            
            transaction.updateData(["reactions": reactions], forDocument: msgRef)
            return nil
        }) { _, _ in }
    }

    func deleteChat(completion: @escaping (Bool) -> Void) {
        guard let uid = Auth.auth().currentUser?.uid else { return }
        let ref = db.collection("chats").document(chatId)
        
        ref.getDocument { [weak self] doc, _ in
            if let data = doc?.data(), let members = data["members"] as? [String] {
                let newMembers = members.filter { $0 != uid }
                if newMembers.isEmpty {
                    ref.delete { _ in completion(true) }
                } else {
                    ref.updateData(["members": newMembers]) { _ in completion(true) }
                }
            } else { completion(false) }
        }
    }
    
    func castVote(msgId: String, option: String) {
        guard let uid = Auth.auth().currentUser?.uid else { return }
        let msgRef = db.collection("chats").document(chatId).collection("messages").document(msgId)
        
        db.runTransaction({ (transaction, errorPointer) -> Any? in
            let doc: DocumentSnapshot
            do { doc = try transaction.getDocument(msgRef) } catch let error as NSError { errorPointer?.pointee = error; return nil }
            guard let data = doc.data(), let rawText = data["text"] as? String else { return nil }
            
            do {
                if var parsed = try JSONSerialization.jsonObject(with: Data(rawText.utf8), options: []) as? [String: Any],
                   var votes = parsed["votes"] as? [String: [String]] {
                    
                    if let deadlineStr = parsed["deadline"] as? String, !deadlineStr.isEmpty {
                        let formatter = DateFormatter(); formatter.dateFormat = "yyyy-MM-dd'T'HH:mm"
                        if let d = formatter.date(from: deadlineStr), Date() > d { return nil }
                    }
                    
                    for (key, uids) in votes { votes[key] = uids.filter { $0 != uid } }
                    
                    if let originalUids = (try? JSONSerialization.jsonObject(with: Data(rawText.utf8), options: []) as? [String: Any])?["votes"] as? [String: [String]],
                       let targetUids = originalUids[option], !targetUids.contains(uid) {
                        if votes[option] == nil { votes[option] = [] }
                        votes[option]?.append(uid)
                    }
                    
                    parsed["votes"] = votes
                    let newData = try JSONSerialization.data(withJSONObject: parsed, options: [])
                    if let newText = String(data: newData, encoding: .utf8) { transaction.updateData(["text": newText], forDocument: msgRef) }
                }
            } catch let e as NSError { errorPointer?.pointee = e; return nil }
            return nil
        }) { _, _ in }
    }
}

// 輔助函式
func parseOldArrayData(from value: Any?) -> [String] {
    if let arr = value as? [String] { return arr }
    if let str = value as? String {
        if str.hasPrefix("[") {
            if let data = str.data(using: .utf8), let arr = try? JSONSerialization.jsonObject(with: data) as? [String] { return arr }
        }
        return str.split(separator: ",").map { String($0).trimmingCharacters(in: .whitespacesAndNewlines) }.filter { !$0.isEmpty }
    }
    return []
}

func canEdit(creatorUid: String, myName: String) -> Bool {
    return creatorUid == Auth.auth().currentUser?.uid || myName == "班長"
}

// MARK: - 5. 共用 UI 元件與擴充 (Shared Components)

// 👉 補回遺失的 CachedImage
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
                Color(UIColor.systemGray6).onAppear { loadImage() }
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

extension View {
    func cornerRadius(_ radius: CGFloat, corners: UIRectCorner) -> some View {
        clipShape(RoundedCorner(radius: radius, corners: corners))
    }
}

struct RoundedCorner: Shape {
    var radius: CGFloat = .infinity
    var corners: UIRectCorner = .allCorners
    func path(in rect: CGRect) -> Path {
        return Path(UIBezierPath(roundedRect: rect, byRoundingCorners: corners, cornerRadii: CGSize(width: radius, height: radius)).cgPath)
    }
}

struct CameraImagePicker: UIViewControllerRepresentable {
    @Binding var image: UIImage?
    @Environment(\.presentationMode) var presentationMode
    
    func makeUIViewController(context: Context) -> UIImagePickerController {
        let picker = UIImagePickerController()
        if UIImagePickerController.isSourceTypeAvailable(.camera) { picker.sourceType = .camera } else { picker.sourceType = .photoLibrary }
        picker.delegate = context.coordinator
        return picker
    }
    func updateUIViewController(_ uiViewController: UIImagePickerController, context: Context) {}
    func makeCoordinator() -> Coordinator { Coordinator(self) }
    
    class Coordinator: NSObject, UINavigationControllerDelegate, UIImagePickerControllerDelegate {
        let parent: CameraImagePicker
        init(_ parent: CameraImagePicker) { self.parent = parent }
        func imagePickerController(_ picker: UIImagePickerController, didFinishPickingMediaWithInfo info: [UIImagePickerController.InfoKey : Any]) {
            if let uiImage = info[.originalImage] as? UIImage { parent.image = uiImage }
            parent.presentationMode.wrappedValue.dismiss()
        }
        func imagePickerControllerDidCancel(_ picker: UIImagePickerController) { parent.presentationMode.wrappedValue.dismiss() }
    }
}

struct GIFView: UIViewRepresentable {
    let urlString: String
    var allowZoom: Bool = false
    func makeUIView(context: Context) -> WKWebView {
        let prefs = WKWebpagePreferences()
        let config = WKWebViewConfiguration()
        config.defaultWebpagePreferences = prefs
        let webView = WKWebView(frame: .zero, configuration: config)
        webView.isOpaque = false
        webView.backgroundColor = .clear
        webView.scrollView.isScrollEnabled = allowZoom
        webView.isUserInteractionEnabled = allowZoom
        return webView
    }
    func updateUIView(_ uiView: WKWebView, context: Context) {
        let html = """
        <!DOCTYPE html><html><head><meta name="viewport" content="width=device-width, initial-scale=1, maximum-scale=5, user-scalable=yes"></head>
        <body style="margin:0; padding:0; background:transparent; display:flex; justify-content:center; align-items:center; height:100vh; overflow:hidden;">
            <img src="\(urlString)" style="width:100%; height:100%; object-fit:contain;">
        </body></html>
        """
        uiView.loadHTMLString(html, baseURL: nil)
    }
}

// 👉 補回遺失的 FullScreenMediaViewer (修正 Cannot find 'FullScreenMediaViewer' in scope)
struct FullScreenMediaViewer: View {
    let urlString: String
    let isGif: Bool
    @Binding var isPresented: Bool
    
    @State private var scale: CGFloat = 1.0
    @State private var lastScale: CGFloat = 1.0
    @State private var offset: CGSize = .zero
    @State private var lastOffset: CGSize = .zero

    var body: some View {
        ZStack {
            Color.black.ignoresSafeArea()
            
            if isGif {
                GIFView(urlString: urlString, allowZoom: true).ignoresSafeArea()
            } else {
                CachedImage(urlString: urlString, contentMode: .fit)
                    .scaleEffect(scale)
                    .offset(offset)
                    .gesture(MagnificationGesture().onChanged { val in
                        let delta = val / lastScale; lastScale = val; scale = min(max(scale * delta, 1), 5)
                    }.onEnded { _ in
                        lastScale = 1.0; if scale < 1 { withAnimation(.spring()) { scale = 1.0; offset = .zero } }
                    })
                    .simultaneousGesture(DragGesture().onChanged { val in
                        if scale > 1 { offset = CGSize(width: lastOffset.width + val.translation.width, height: lastOffset.height + val.translation.height) }
                        else { if val.translation.height > 0 { offset = val.translation } }
                    }.onEnded { val in
                        if scale > 1 { lastOffset = offset }
                        else { if val.translation.height > 100 { isPresented = false } else { withAnimation(.spring()) { offset = .zero } } }
                    })
                    .onTapGesture(count: 2) {
                        withAnimation(.spring()) { scale = scale > 1 ? 1.0 : 2.5; offset = .zero; lastOffset = .zero }
                    }
            }
            
            VStack {
                HStack {
                    Spacer()
                    Button(action: { isPresented = false }) {
                        Image(systemName: "xmark")
                            .font(.system(size: 20, weight: .bold))
                            .foregroundColor(.white)
                            .padding(12)
                            .background(Color.black.opacity(0.5))
                            .clipShape(Circle())
                    }
                    .padding()
                }
                Spacer()
            }
        }
    }
}

struct CachedAvatarView: View {
    let url: String
    let size: CGFloat
    @StateObject private var loader: ImageLoader
    
    init(url: String, size: CGFloat = 55) {
        self.url = url
        self.size = size
        _loader = StateObject(wrappedValue: ImageLoader(url: url))
    }
    
    var body: some View {
        if let image = loader.image {
            Image(uiImage: image).resizable().scaledToFill().frame(width: size, height: size).clipShape(Circle())
        } else {
            ZStack {
                Circle().fill(Color(UIColor.systemGray5)).frame(width: size, height: size)
                ProgressView()
            }
        }
    }
}

struct LiveAvatarView: View {
    let uidOrName: String
    var showName: Bool = true
    var size: CGFloat = 50
    @State private var displayName: String = ""
    @State private var avatar: String = ""
    let lyGold = Color(red: 184/255, green: 134/255, blue: 11/255)

    var body: some View {
        VStack(spacing: 4) {
            if avatar.isEmpty {
                ZStack {
                    Circle().fill(Color(UIColor.systemGray5)).frame(width: size, height: size).overlay(Circle().stroke(Color.black, lineWidth: 2)).overlay(Circle().stroke(lyGold, lineWidth: 3))
                    if !displayName.isEmpty { Text(String(displayName.prefix(1))).font(.system(size: size * 0.4, weight: .bold)).foregroundColor(.gray) }
                }
            } else {
                CachedAvatarView(url: avatar, size: size).clipShape(Circle()).overlay(Circle().stroke(Color.black, lineWidth: 2)).overlay(Circle().stroke(lyGold, lineWidth: 3))
            }
            if showName { Text(displayName).font(.caption).foregroundColor(.gray).lineLimit(1) }
        }
        .onAppear {
            self.displayName = uidOrName
            if uidOrName.count > 20 { Firestore.firestore().collection("act").document(uidOrName).getDocument { doc, _ in if let data = doc?.data() { self.displayName = data["displayName"] as? String ?? "未知"; self.avatar = data["photoURL"] as? String ?? "" } } }
        }
    }
}

struct LiveUserRow: View {
    let uid: String
    @State private var name: String = "載入中..."
    @State private var avatar: String = ""
    var body: some View {
        HStack(spacing: 15) {
            if avatar.isEmpty { Circle().fill(Color.gray.opacity(0.3)).frame(width: 40, height: 40) } else { CachedAvatarView(url: avatar, size: 40).clipShape(Circle()) }
            Text(name).font(.headline).foregroundColor(.primary)
            Spacer()
        }
        .onAppear {
            Firestore.firestore().collection("act").document(uid).getDocument { doc, _ in
                if let data = doc?.data() {
                    self.name = data["displayName"] as? String ?? "未知"
                    self.avatar = data["photoURL"] as? String ?? ""
                }
            }
        }
    }
}

struct AudioBubbleView: View {
    let audioUrl: String
    let isMine: Bool
    @State private var player: AVPlayer?
    @State private var isPlaying = false

    var body: some View {
        HStack {
            Button(action: togglePlay) { Image(systemName: isPlaying ? "pause.circle.fill" : "play.circle.fill").font(.title).foregroundColor(isMine ? .white : .primary) }
            Text("語音訊息").font(.subheadline).foregroundColor(isMine ? .white : .primary)
        }
        .padding(.horizontal, 16).padding(.vertical, 12).background(isMine ? Color(red: 0/255, green: 49/255, blue: 83/255) : Color(UIColor.systemGray5)).cornerRadius(20)
        .onDisappear { player?.pause() }
    }

    func togglePlay() {
        if player == nil { guard let url = URL(string: audioUrl) else { return }; player = AVPlayer(url: url); NotificationCenter.default.addObserver(forName: .AVPlayerItemDidPlayToEndTime, object: player?.currentItem, queue: .main) { _ in isPlaying = false; player?.seek(to: .zero) } }
        if isPlaying { player?.pause(); isPlaying = false } else { player?.play(); isPlaying = true }
    }
}

struct GenericListRow: View {
    let title: String; let subtitle: String; let creator: String; let icon: String; let color: Color; let action: () -> Void
    var body: some View {
        Button(action: action) {
            HStack(spacing: 0) {
                Rectangle().fill(color).frame(width: 6)
                VStack(alignment: .leading, spacing: 10) {
                    HStack { Image(systemName: icon); Text(title).font(.title3).fontWeight(.bold) }.foregroundColor(color)
                    HStack(alignment: .bottom) {
                        HStack(spacing: 4) { Image(systemName: "clock"); Text(subtitle) }.font(.caption).foregroundColor(subtitle.contains("無") ? .gray : .red)
                        Spacer()
                        Text("建立人：\(creator)").font(.caption).fontWeight(.bold).foregroundColor(color).padding(.horizontal, 10).padding(.vertical, 4).background(Color.gray.opacity(0.1)).cornerRadius(12)
                    }
                }.padding(.vertical, 12).padding(.horizontal, 15)
            }.background(Color.white).cornerRadius(12).overlay(RoundedRectangle(cornerRadius: 12).stroke(Color.gray.opacity(0.2), lineWidth: 1))
        }
        .buttonStyle(PlainButtonStyle()).listRowInsets(EdgeInsets(top: 6, leading: 16, bottom: 6, trailing: 16)).listRowSeparator(.hidden).listRowBackground(Color.clear)
    }
}

struct PrivilegeBanner: View {
    let myName: String; let color: Color
    var body: some View {
        HStack(alignment: .top, spacing: 10) {
            Image(systemName: "lightbulb.fill").foregroundColor(color).font(.title3)
            if myName == "班長" { Text("班長特權：").fontWeight(.bold).foregroundColor(color) + Text("您擁有所有項目的「編輯」與「刪除」權限，向左滑動即可操作。").foregroundColor(.gray) } else { Text("小提示：").fontWeight(.bold).foregroundColor(color) + Text("自己發起的功能，向左滑動即可「編輯」或「刪除」。").foregroundColor(.gray) }
        }.padding(.vertical, 8).listRowBackground(Color(red: 255/255, green: 253/255, blue: 245/255))
    }
}

struct ChatBubbleShape: Shape {
    let isMine: Bool
    func path(in rect: CGRect) -> Path {
        let p = UIBezierPath(roundedRect: rect, byRoundingCorners: [.topLeft, .topRight, isMine ? .bottomLeft : .bottomRight], cornerRadii: CGSize(width: 18, height: 18))
        return Path(p.cgPath)
    }
}

struct MenuActionBtn: View {
    let icon: String
    let isRed: Bool
    let action: () -> Void
    init(icon: String, isRed: Bool = false, action: @escaping () -> Void) { self.icon = icon; self.isRed = isRed; self.action = action }
    var body: some View { Button(action: action) { Image(systemName: icon).font(.system(size: 22)).foregroundColor(isRed ? .red : .primary) } }
}

// MARK: - 6. 主流程與導覽 (Main Flow)
struct ContentView: View {
    @State private var email = ""
    @State private var password = ""
    @State private var errorMessage = ""
    @State private var isLoading = false
    @State private var isLoggedIn = false
    
    let lyBlue = Color(red: 0/255, green: 49/255, blue: 83/255)
    let lyGold = Color(red: 184/255, green: 134/255, blue: 11/255)

    var body: some View {
        if isLoggedIn { MainTabView(isLoggedIn: $isLoggedIn) } else { loginForm }
    }
    
    var loginForm: some View {
        VStack(spacing: 30) {
            Spacer()
            VStack(spacing: 8) {
                Image(systemName: "building.columns.fill").font(.system(size: 45)).foregroundColor(lyBlue)
                Text("立法院").font(.system(size: 32, weight: .bold)).foregroundColor(lyBlue).tracking(2)
                Rectangle().fill(lyGold).frame(width: 40, height: 4).padding(.top, 5)
            }
            Text("ADMINISTRATION SYSTEM").font(.caption).foregroundColor(.gray).tracking(1)
            
            VStack(alignment: .leading, spacing: 20) {
                VStack(alignment: .leading, spacing: 8) {
                    Text("帳號 (Email)").font(.footnote).fontWeight(.semibold).foregroundColor(.secondary)
                    HStack { Image(systemName: "envelope.fill").foregroundColor(.gray); TextField("請輸入公務帳號", text: $email).keyboardType(.emailAddress).autocapitalization(.none) }.padding().background(Color(UIColor.systemGray6)).cornerRadius(8)
                }
                VStack(alignment: .leading, spacing: 8) {
                    Text("密碼 (Password)").font(.footnote).fontWeight(.semibold).foregroundColor(.secondary)
                    HStack { Image(systemName: "lock.fill").foregroundColor(.gray); SecureField("請輸入密碼", text: $password) }.padding().background(Color(UIColor.systemGray6)).cornerRadius(8)
                }
            }.padding(.horizontal, 10)
            
            if !errorMessage.isEmpty { Text(errorMessage).foregroundColor(.red).font(.footnote) }
            
            Button(action: loginWithFirebase) { HStack { if isLoading { ProgressView().progressViewStyle(CircularProgressViewStyle(tint: .white)) } else { Text("進入管理系統") } }.font(.system(size: 18, weight: .bold)).foregroundColor(.white).frame(maxWidth: .infinity).padding().background(lyBlue).cornerRadius(8) }.disabled(isLoading).padding(.top, 10)
            Spacer()
        }.padding(30).background(Color(UIColor.systemGroupedBackground).ignoresSafeArea())
    }
    
    func loginWithFirebase() {
        guard !email.isEmpty, !password.isEmpty else { errorMessage = "請完整填寫帳號與密碼"; return }
        isLoading = true; errorMessage = ""
        Auth.auth().signIn(withEmail: email, password: password) { result, error in
            isLoading = false
            if let error = error { errorMessage = "登入失敗: \(error.localizedDescription)" } else if let _ = result?.user { withAnimation { isLoggedIn = true } }
        }
    }
}

struct MainTabView: View {
    @Binding var isLoggedIn: Bool
    @State private var selectedTab = 1
    @State private var myAvatarUrl = Auth.auth().currentUser?.photoURL?.absoluteString ?? ""
    @State private var myName = "自己"
    @State private var isTabBarHidden = false
    
    let lyBlue = Color(red: 0/255, green: 49/255, blue: 83/255)
    let lyGold = Color(red: 184/255, green: 134/255, blue: 11/255)

    var body: some View {
        ZStack(alignment: .bottom) {
            Group {
                switch selectedTab {
                case 0: Text("首頁 (準備開發)").frame(maxWidth: .infinity, maxHeight: .infinity).background(Color(UIColor.systemGroupedBackground))
                case 1: ChatListView()
                case 2: ProfileView(isLoggedIn: $isLoggedIn)
                case 3: Text("日曆 (CalendarView)").font(.title).frame(maxWidth: .infinity, maxHeight: .infinity).background(Color(UIColor.systemGroupedBackground))
                case 4: Text("其他 (準備開發)").frame(maxWidth: .infinity, maxHeight: .infinity).background(Color(UIColor.systemGroupedBackground))
                default: EmptyView()
                }
            }.frame(maxWidth: .infinity, maxHeight: .infinity)
            
            if !isTabBarHidden {
                HStack(spacing: 0) {
                    tabButton(icon: "house.fill", title: "首頁", index: 0)
                    tabButton(icon: "message.fill", title: "聊天", index: 1)
                    Spacer().frame(width: 80)
                    tabButton(icon: "calendar", title: "日曆", index: 3)
                    tabButton(icon: "ellipsis", title: "其他", index: 4)
                }
                .frame(height: 55).padding(.horizontal, 10)
                .background(Color.white.shadow(color: Color.black.opacity(0.08), radius: 3, x: 0, y: -3).ignoresSafeArea(.all, edges: .bottom))
                .overlay(
                    Button(action: { selectedTab = 2 }) {
                        VStack(spacing: 4) {
                            ZStack {
                                Circle().fill(Color.white).frame(width: 65, height: 65).shadow(color: Color.black.opacity(0.15), radius: 5, x: 0, y: -2)
                                if myAvatarUrl.isEmpty { Image(systemName: "person.crop.circle.fill").resizable().frame(width: 55, height: 55).foregroundColor(lyBlue) } else { CachedAvatarView(url: myAvatarUrl, size: 55).clipShape(Circle()) }
                            }
                            Text(myName).font(.system(size: 10, weight: .black)).foregroundColor(selectedTab == 2 ? lyGold : lyBlue).lineLimit(1)
                        }
                    }.offset(y: -20), alignment: .top
                ).transition(.move(edge: .bottom).combined(with: .opacity))
            }
        }
        .ignoresSafeArea(.keyboard, edges: .bottom)
        .onAppear { fetchMyProfile() }
        .onChange(of: selectedTab) { _ in withAnimation(.easeIn(duration: 0.2)) { isTabBarHidden = false } }
        .onReceive(NotificationCenter.default.publisher(for: NSNotification.Name("HideTabBar"))) { _ in withAnimation(.easeOut(duration: 0.2)) { isTabBarHidden = true } }
        .onReceive(NotificationCenter.default.publisher(for: NSNotification.Name("ShowTabBar"))) { _ in withAnimation(.easeIn(duration: 0.2)) { isTabBarHidden = false } }
    }
    
    private func fetchMyProfile() { guard let uid = Auth.auth().currentUser?.uid else { return }; Firestore.firestore().collection("act").document(uid).getDocument { doc, _ in if let data = doc?.data() { self.myName = data["displayName"] as? String ?? "未知"; self.myAvatarUrl = data["photoURL"] as? String ?? "" } } }
    func tabButton(icon: String, title: String, index: Int) -> some View { Button(action: { selectedTab = index }) { VStack(spacing: 5) { Image(systemName: icon).font(.system(size: 22)); Text(title).font(.system(size: 10, weight: .bold)) }.foregroundColor(selectedTab == index ? lyBlue : .gray.opacity(0.5)).frame(maxWidth: .infinity) } }
}

struct ProfileView: View {
    @Binding var isLoggedIn: Bool
    @State private var displayName: String = Auth.auth().currentUser?.displayName ?? "未設定名稱"
    @State private var avatarUrl: String = Auth.auth().currentUser?.photoURL?.absoluteString ?? ""
    @State private var isUploading = false
    @State private var selectedPhoto: PhotosPickerItem?
    
    @State private var sessions: [DeviceSession] = [
        DeviceSession(id: "1", ip: "211.75.XX.XX", location: "台灣, 台北市", lastLogin: "剛剛", isCurrent: true),
        DeviceSession(id: "2", ip: "114.32.XX.XX", location: "台灣, 新北市", lastLogin: "2026/04/10 14:20", isCurrent: false)
    ]
    
    let lyBlue = Color(red: 0/255, green: 49/255, blue: 83/255)
    let lyGold = Color(red: 184/255, green: 134/255, blue: 11/255)
    
    var body: some View {
        NavigationStack {
            ScrollView {
                VStack(spacing: 30) {
                    VStack(spacing: 15) {
                        PhotosPicker(selection: $selectedPhoto, matching: .images, photoLibrary: .shared()) {
                            ZStack(alignment: .bottomTrailing) {
                                if isUploading { Circle().fill(Color.gray.opacity(0.3)).frame(width: 110, height: 110); ProgressView().tint(.white) } else if avatarUrl.isEmpty { Image(systemName: "person.crop.circle.fill").resizable().frame(width: 110, height: 110).foregroundColor(lyBlue) } else { CachedAvatarView(url: avatarUrl, size: 110).clipShape(Circle()) }
                                Image(systemName: "camera.circle.fill").font(.title).foregroundColor(lyGold).background(Circle().fill(Color.white)).offset(x: -5, y: -5)
                            }
                        }.onChange(of: selectedPhoto) { newItem in Task { if let data = try? await newItem?.loadTransferable(type: Data.self), let uiImage = UIImage(data: data) { uploadAvatar(image: uiImage) } } }
                        
                        TextField("編輯顯示名稱", text: $displayName).font(.title2).fontWeight(.bold).foregroundColor(lyBlue).multilineTextAlignment(.center).onSubmit { updateDisplayName() }
                        Text(Auth.auth().currentUser?.email ?? "信箱載入中...").font(.subheadline).foregroundColor(.gray)
                    }.padding(.top, 30)
                    
                    VStack(alignment: .leading, spacing: 15) {
                        HStack { Image(systemName: "shield.checkerboard").foregroundColor(lyGold); Text("帳號安全性與登入紀錄").font(.headline).foregroundColor(lyBlue) }
                        VStack(spacing: 12) {
                            ForEach(sessions) { session in
                                HStack {
                                    VStack(alignment: .leading, spacing: 6) {
                                        HStack { Image(systemName: session.isCurrent ? "iphone.radiowaves.left.and.right" : "desktopcomputer").foregroundColor(session.isCurrent ? .green : .gray); Text(session.location).fontWeight(.bold).foregroundColor(.primary); if session.isCurrent { Text("目前裝置").font(.caption2).padding(.horizontal, 6).padding(.vertical, 2).background(Color.green.opacity(0.2)).foregroundColor(.green).cornerRadius(4) } }
                                        Text("IP位址：\(session.ip)").font(.caption).foregroundColor(.gray); Text("最後活動：\(session.lastLogin)").font(.caption).foregroundColor(.gray)
                                    }
                                    Spacer()
                                    if !session.isCurrent { Button(action: { kickDevice(id: session.id) }) { Text("強制登出").font(.caption).fontWeight(.bold).foregroundColor(.red).padding(.horizontal, 10).padding(.vertical, 6).background(Color.red.opacity(0.1)).cornerRadius(8) } }
                                }.padding().background(Color.white).cornerRadius(10).overlay(RoundedRectangle(cornerRadius: 10).stroke(Color.gray.opacity(0.2), lineWidth: 1))
                            }
                        }
                    }.padding(.horizontal, 20)
                    Spacer(minLength: 40)
                    Button(action: { try? Auth.auth().signOut(); isLoggedIn = false }) { HStack { Image(systemName: "signpost.right.fill"); Text("登出目前裝置") }.font(.headline).foregroundColor(.white).frame(maxWidth: .infinity).padding().background(Color.red).cornerRadius(10) }.padding(.horizontal, 20).padding(.bottom, 40)
                }
            }.background(Color(UIColor.systemGroupedBackground).ignoresSafeArea()).navigationTitle("關於我").navigationBarTitleDisplayMode(.inline)
        }
    }
    
    private func kickDevice(id: String) { withAnimation { sessions.removeAll { $0.id == id } }; print("已執行遠端踢除") }
    private func updateDisplayName() { guard let user = Auth.auth().currentUser else { return }; let changeRequest = user.createProfileChangeRequest(); changeRequest.displayName = displayName; changeRequest.commitChanges { error in if error == nil { Firestore.firestore().collection("act").document(user.uid).setData(["displayName": displayName], merge: true) } } }
    private func uploadAvatar(image: UIImage) { guard let uid = Auth.auth().currentUser?.uid, let imageData = image.jpegData(compressionQuality: 0.3) else { return }; isUploading = true; let ref = Storage.storage().reference().child("avatars/\(uid).jpg"); ref.putData(imageData, metadata: nil) { _, error in if error == nil { ref.downloadURL { url, _ in if let newUrl = url?.absoluteString { let changeRequest = Auth.auth().currentUser?.createProfileChangeRequest(); changeRequest?.photoURL = url; changeRequest?.commitChanges { _ in }; Firestore.firestore().collection("act").document(uid).setData(["photoURL": newUrl], merge: true); DispatchQueue.main.async { self.avatarUrl = newUrl; self.isUploading = false } } } } else { isUploading = false } } }
}

// MARK: - 7. 訊息中心 (ChatListView)
struct ChatListView: View {
    let lyBlue = Color(red: 0/255, green: 49/255, blue: 83/255)
    let lyGold = Color(red: 184/255, green: 134/255, blue: 11/255)
    @StateObject private var viewModel = ChatViewModel()

    var body: some View {
        NavigationStack {
            List {
                ForEach(viewModel.chats) { chat in
                    ZStack {
                        NavigationLink(destination: ChatRoomView(chatId: chat.id, chatName: chat.name)) { EmptyView() }.opacity(0)
                        ChatRowView(chat: chat)
                    }
                    .alignmentGuide(.listRowSeparatorLeading) { _ in 0 }
                    .listRowInsets(EdgeInsets(top: 12, leading: 16, bottom: 12, trailing: 16))
                    .listRowBackground(Color.white)
                }
            }
            .listStyle(PlainListStyle())
            .navigationTitle("訊息中心")
            .navigationBarTitleDisplayMode(.inline)
            .toolbar {
                ToolbarItem(placement: .navigationBarTrailing) {
                    HStack(spacing: 16) {
                        Button(action: { }) { Image(systemName: "person.2.fill") }
                        Button(action: { print("準備開啟發起對話的畫面") }) { Image(systemName: "plus") }
                    }
                    .font(.system(size: 16, weight: .bold))
                    .foregroundColor(lyBlue)
                    .padding(.horizontal, 14)
                    .padding(.vertical, 8)
                    .background(Color.white)
                    .cornerRadius(20)
                    .shadow(color: Color.black.opacity(0.08), radius: 4, x: 0, y: 2)
                }
            }
            .onAppear {
                viewModel.fetchChats()
                NotificationCenter.default.post(name: NSNotification.Name("ShowTabBar"), object: nil)
            }
        }
    }
}

struct ChatRowView: View {
    let chat: ChatItem
    let lyGold = Color(red: 184/255, green: 134/255, blue: 11/255)
    
    var body: some View {
        HStack(spacing: 15) {
            ZStack(alignment: .topLeading) {
                if chat.avatarUrl.isEmpty {
                    ZStack {
                        Circle().fill(Color(UIColor.systemGray5)).frame(width: 55, height: 55)
                        Image(systemName: chat.isGroup ? "person.3.fill" : "person.fill").foregroundColor(.gray)
                    }
                } else {
                    CachedAvatarView(url: chat.avatarUrl, size: 55).clipShape(Circle())
                }
                
                if chat.name == "立法院 Legislature" {
                    Image(systemName: "pin.fill")
                        .font(.system(size: 14))
                        .foregroundColor(lyGold)
                        .rotationEffect(.degrees(-45))
                        .shadow(color: .black.opacity(0.3), radius: 2, x: 1, y: 1)
                        .offset(x: -5, y: -2)
                }
            }
            
            VStack(alignment: .leading, spacing: 6) {
                HStack(alignment: .top) {
                    Text(chat.name).font(.system(size: 18, weight: chat.isUnread ? .heavy : .bold)).foregroundColor(.primary).lineLimit(1)
                    Spacer()
                    Text(chat.time).font(.system(size: 13)).foregroundColor(.gray)
                }
                Text(chat.lastMessage).font(.system(size: 15)).foregroundColor(chat.isUnread ? .primary : .secondary).lineLimit(1)
            }
        }
        .padding(.vertical, 4)
        .overlay(Circle().fill(chat.isUnread ? lyGold : Color.clear).frame(width: 8, height: 8).offset(x: -10), alignment: .leading)
    }
}

// MARK: - 8. 聊天室核心 (Chat Room)

struct MessageRowView: View {
    let msg: MessageItem
    let isGroup: Bool
    @ObservedObject var viewModel: ChatRoomViewModel
    let currentUid: String
    
    @Binding var replyingToText: String?
    @Binding var popupDetailText: String
    @Binding var showReplyDetailModal: Bool
    @Binding var fullScreenMediaUrl: String
    @Binding var fullScreenMediaIsGif: Bool
    @Binding var showFullScreenMedia: Bool
    @Binding var showReadList: Bool
    @Binding var currentReadMembers: [ReadMember]
    @Binding var reactionSheetItem: ReactionSheetItem?
    
    @State private var bubbleY: CGFloat = 0
    let lyBlue = Color(red: 0/255, green: 49/255, blue: 83/255)
    let lyGold = Color(red: 184/255, green: 134/255, blue: 11/255)
    
    var body: some View {
        let showMenu = (viewModel.activeMenuMsgId == msg.id)
        let showMenuAtTop = bubbleY > UIScreen.main.bounds.height - 300

        HStack(alignment: .top, spacing: 8) {
            if msg.isMine {
                Spacer(minLength: 40)
                VStack(alignment: .trailing, spacing: 4) {
                    if showMenu && showMenuAtTop { CustomMenuView(showMenuAtTop: showMenuAtTop) }
                    
                    BubbleContent()
                    
                    HStack(spacing: 6) {
                        if !msg.readBy.isEmpty { ReadStatusText() }
                        MessageTimeText()
                    }
                    ReactionsRow()
                    
                    if showMenu && !showMenuAtTop { CustomMenuView(showMenuAtTop: showMenuAtTop) }
                }
            } else {
                VStack(spacing: 4) {
                    CachedAvatarView(url: msg.senderAvatar.isEmpty ? "" : msg.senderAvatar, size: 36).clipShape(Circle())
                    if isGroup { Text(msg.senderName).font(.system(size: 10)).foregroundColor(.gray).lineLimit(1).frame(width: 45) }
                }
                
                VStack(alignment: .leading, spacing: 4) {
                    if showMenu && showMenuAtTop { CustomMenuView(showMenuAtTop: showMenuAtTop) }
                    
                    BubbleContent()
                    
                    HStack(spacing: 6) {
                        MessageTimeText()
                        if !msg.readBy.isEmpty { ReadStatusText() }
                    }
                    ReactionsRow()
                    
                    if showMenu && !showMenuAtTop { CustomMenuView(showMenuAtTop: showMenuAtTop) }
                }
                Spacer(minLength: 40)
            }
        }
        .padding(.vertical, 4)
        .background(
            GeometryReader { geo in
                Color.clear
                    .onAppear { self.bubbleY = geo.frame(in: .global).minY }
                    .onChange(of: geo.frame(in: .global).minY) { newY in self.bubbleY = newY }
            }
        )
    }
    
    @ViewBuilder
    private func BubbleContent() -> some View {
        VStack(alignment: msg.isMine ? .trailing : .leading, spacing: 6) {
            if !msg.replyToText.isEmpty {
                Text("回覆：\(msg.replyToText)").font(.caption2).foregroundColor(msg.isMine ? .white.opacity(0.8) : .gray)
                    .lineLimit(1).truncationMode(.tail).frame(maxWidth: 220, alignment: .leading)
                    .padding(.horizontal, 8).padding(.vertical, 4).background(msg.isMine ? Color.white.opacity(0.2) : Color.gray.opacity(0.2)).cornerRadius(6)
                    .onTapGesture { popupDetailText = msg.replyToText; showReplyDetailModal = true }
            }
            
            Group {
                if msg.fileType == "image" || msg.text.isImageURL {
                    let urlString = msg.fileType == "image" ? msg.fileUrl : msg.text.trimmingCharacters(in: .whitespacesAndNewlines)
                    Button(action: { fullScreenMediaUrl = urlString; fullScreenMediaIsGif = urlString.isGifURL; withAnimation { showFullScreenMedia = true } }) {
                        if urlString.isGifURL { GIFView(urlString: urlString).frame(width: 220, height: 220).cornerRadius(15) }
                        else { AsyncImage(url: URL(string: urlString)) { phase in if let image = phase.image { image.resizable().scaledToFill().frame(width: 220, height: 220).cornerRadius(15).clipped() } else { ZStack { Rectangle().fill(Color(UIColor.systemGray5)).frame(width: 220, height: 220).cornerRadius(15); ProgressView() } } } }
                    }.buttonStyle(PlainButtonStyle())
                } else if msg.fileType == "audio" { AudioBubbleView(audioUrl: msg.fileUrl, isMine: msg.isMine) }
                else if !msg.cardType.isEmpty, let cardData = msg.cardData {
                    if msg.cardType == "event_share" { NavigationLink(destination: EventDetailView(cardData: cardData, viewModel: viewModel)) { eventCardUI(cardData: cardData) }.buttonStyle(PlainButtonStyle()) }
                    else if msg.cardType == "group_buy" { NavigationLink(destination: GroupBuyDetailView(cardData: cardData)) { groupBuyCardUI(cardData: cardData) }.buttonStyle(PlainButtonStyle()) }
                    else if msg.cardType == "poll" { NavigationLink(destination: PollDetailView(msgId: msg.id, cardData: cardData, viewModel: viewModel)) { pollCardUI(cardData: cardData) }.buttonStyle(PlainButtonStyle()) }
                    else if msg.cardType == "confidential" { NavigationLink(destination: ConfidentialDetailView(cardData: cardData)) { confidentialCardUI(cardData: cardData) }.buttonStyle(PlainButtonStyle()) }
                } else {
                    Text(msg.text).padding(.horizontal, 16).padding(.vertical, 12)
                        .background(msg.isMine ? lyBlue : Color(UIColor.secondarySystemBackground))
                        .foregroundColor(msg.isMine ? .white : .primary)
                        .clipShape(ChatBubbleShape(isMine: msg.isMine))
                }
            }
            .onLongPressGesture(minimumDuration: 0.3) {
                UIApplication.shared.sendAction(#selector(UIResponder.resignFirstResponder), to: nil, from: nil, for: nil)
                withAnimation(.spring(response: 0.3, dampingFraction: 0.7)) {
                    viewModel.activeMenuMsgId = (viewModel.activeMenuMsgId == msg.id) ? nil : msg.id
                }
            }
        }
    }
    
    @ViewBuilder
    private func CustomMenuView(showMenuAtTop: Bool) -> some View {
        VStack(alignment: msg.isMine ? .trailing : .leading, spacing: 8) {
            HStack(spacing: 18) {
                ForEach(["👍", "❤️", "😂", "🙏", "👀"], id: \.self) { emoji in
                    Text(emoji).font(.system(size: 26))
                        .onTapGesture { viewModel.addReaction(messageId: msg.id, emoji: emoji); closeMenu() }
                }
            }.padding(.horizontal, 16).padding(.vertical, 10).background(Color.white).cornerRadius(25).shadow(color: Color.black.opacity(0.12), radius: 6, y: 3)
            
            HStack(spacing: 20) {
                MenuActionBtn(icon: "doc.on.doc") { UIPasteboard.general.string = msg.text; closeMenu() }
                MenuActionBtn(icon: "arrowshape.turn.up.left") { replyingToText = msg.text.isEmpty ? "[圖片/語音]" : msg.text; closeMenu() }
                MenuActionBtn(icon: "pin") { viewModel.pinMessage(text: msg.text.isEmpty ? (msg.fileType == "image" ? msg.fileUrl : "[卡片/語音]") : msg.text); closeMenu() }
                if msg.isMine {
                    MenuActionBtn(icon: "trash", isRed: true) {
                        if !msg.cardType.isEmpty { viewModel.hideMessage(messageId: msg.id) } else { viewModel.revokeMessage(messageId: msg.id) }
                        closeMenu()
                    }
                } else {
                    MenuActionBtn(icon: "eye.slash", isRed: true) { viewModel.hideMessage(messageId: msg.id); closeMenu() }
                }
            }.padding(.horizontal, 16).padding(.vertical, 12).background(Color.white).cornerRadius(18).shadow(color: Color.black.opacity(0.12), radius: 6, y: 3)
        }
        .transition(.scale(scale: 0.8, anchor: msg.isMine ? (showMenuAtTop ? .bottomTrailing : .topTrailing) : (showMenuAtTop ? .bottomLeading : .topLeading)).combined(with: .opacity))
        .zIndex(100)
    }
    
    private func closeMenu() { withAnimation(.spring()) { viewModel.activeMenuMsgId = nil } }

    @ViewBuilder private func ReactionsRow() -> some View {
        if !msg.reactions.isEmpty {
            HStack(spacing: 6) {
                ForEach(msg.reactions.keys.sorted(), id: \.self) { emoji in
                    let uids = msg.reactions[emoji] ?? []
                    let hasMe = uids.contains(currentUid)
                    
                    HStack(spacing: 4) {
                        Text(emoji).font(.system(size: 14))
                        if isGroup {
                            HStack(spacing: -6) {
                                ForEach(uids.prefix(3), id: \.self) { uid in
                                    CachedAvatarView(url: "", size: 18)
                                        .clipShape(Circle())
                                        .overlay(Circle().stroke(Color.white, lineWidth: 1.5))
                                }
                            }
                        }
                        if uids.count > 1 || !isGroup {
                            Text("\(uids.count)").font(.system(size: 11, weight: .bold)).foregroundColor(.gray)
                        }
                    }
                    .padding(.horizontal, 8)
                    .padding(.vertical, 4)
                    .background(hasMe ? lyBlue.opacity(0.1) : Color(UIColor.secondarySystemBackground))
                    .overlay(RoundedRectangle(cornerRadius: 12).stroke(hasMe ? lyBlue : Color.clear, lineWidth: 1))
                    .onTapGesture { viewModel.addReaction(messageId: msg.id, emoji: emoji) }
                    .onLongPressGesture { reactionSheetItem = ReactionSheetItem(emoji: emoji, uids: uids) }
                }
            }.padding(.top, 2)
        }
    }
    
    @ViewBuilder private func ReadStatusText() -> some View {
        if isGroup {
            let displayNames = msg.readBy.prefix(2).map { $0.name }.joined(separator: "、"); let suffix = msg.readBy.count > 2 ? "...等 \(msg.readBy.count) 人" : ""
            Text("已讀 \(displayNames)\(suffix)").font(.system(size: 10, weight: .bold)).foregroundColor(lyBlue).lineLimit(1).onTapGesture { currentReadMembers = msg.readBy; withAnimation { showReadList = true } }
        } else { Text("已讀").font(.system(size: 10, weight: .bold)).foregroundColor(lyBlue) }
    }
    @ViewBuilder private func MessageTimeText() -> some View { Text({ let f = DateFormatter(); f.dateFormat = "HH:mm"; return f.string(from: msg.timestamp) }()).font(.system(size: 11)).foregroundColor(.gray) }

    func eventCardUI(cardData: [String: Any]) -> some View { VStack(alignment: .leading, spacing: 8) { Text("活動分享").font(.caption).fontWeight(.bold).foregroundColor(lyGold); Text(cardData["title"] as? String ?? "未命名活動").font(.headline).foregroundColor(lyBlue).lineLimit(1); Text("時間: \(cardData["timeStr"] as? String ?? "")").font(.caption).foregroundColor(.gray) }.padding().frame(width: 200, alignment: .leading).background(Color.white).cornerRadius(12).overlay(RoundedRectangle(cornerRadius: 12).stroke(lyGold, lineWidth: 1)) }
    func groupBuyCardUI(cardData: [String: Any]) -> some View { VStack(alignment: .leading, spacing: 8) { Text("團購訂單").font(.caption).fontWeight(.bold).foregroundColor(.green); Text(cardData["title"] as? String ?? "未命名團購").font(.headline).foregroundColor(lyBlue).lineLimit(1); Text("發起人: \(cardData["initiator"] as? String ?? "")").font(.caption).foregroundColor(.gray) }.padding().frame(width: 200, alignment: .leading).background(Color.white).cornerRadius(12).overlay(RoundedRectangle(cornerRadius: 12).stroke(Color.green, lineWidth: 1)) }
    func pollCardUI(cardData: [String: Any]) -> some View { VStack(alignment: .leading, spacing: 8) { Text("群組投票").font(.caption).fontWeight(.bold).foregroundColor(lyBlue); Text(cardData["question"] as? String ?? "未命名投票").font(.headline).foregroundColor(lyBlue).lineLimit(1); Text("點擊前往投票...").font(.caption).foregroundColor(.gray) }.padding().frame(width: 200, alignment: .leading).background(Color.white).cornerRadius(12).overlay(RoundedRectangle(cornerRadius: 12).stroke(lyBlue, lineWidth: 1)) }
    func confidentialCardUI(cardData: [String: Any]) -> some View { VStack(alignment: .leading, spacing: 8) { Text("絕對機密").font(.caption).fontWeight(.bold).foregroundColor(.red); Text(cardData["title"] as? String ?? "機密檔案").font(.headline).foregroundColor(.white).lineLimit(1); Text("點擊輸入密碼解鎖").font(.caption).foregroundColor(.gray) }.padding().frame(width: 200, alignment: .leading).background(Color(red: 43/255, green: 43/255, blue: 43/255)).cornerRadius(12).overlay(RoundedRectangle(cornerRadius: 12).stroke(Color.red, lineWidth: 1)) }
}

struct ChatRoomView: View {
    let chatName: String
    @StateObject private var viewModel: ChatRoomViewModel
    
    @State private var inputText = ""
    @State private var isMenuExpanded = true
    @State private var isUIReady = false
    @State private var scrollTrigger = UUID()
    @State private var selectedPhoto: PhotosPickerItem?
    @StateObject private var audioRecorder = AudioRecorder()
    
    @State private var showReadList = false
    @State private var currentReadMembers: [ReadMember] = []
    @State private var reactionSheetItem: ReactionSheetItem? = nil
    
    @State private var showCamera = false
    @State private var cameraImage: UIImage?
    @State private var showAttachmentMenu = false
    @State private var replyingToText: String? = nil
    @State private var showPinnedModal = false
    @State private var showReplyDetailModal = false
    @State private var popupDetailText = ""
    @State private var showFullScreenMedia = false
    @State private var fullScreenMediaUrl = ""
    @State private var fullScreenMediaIsGif = false
    
    let lyBlue = Color(red: 0/255, green: 49/255, blue: 83/255)
    let lyGold = Color(red: 184/255, green: 134/255, blue: 11/255)
    let dateFormatter: DateFormatter = { let f = DateFormatter(); f.dateFormat = "yyyy-MM-dd (E)"; f.locale = Locale(identifier: "zh_Hant_TW"); return f }()
    @AppStorage("currentUid") private var storedUid: String = Auth.auth().currentUser?.uid ?? ""
    
    init(chatId: String, chatName: String) {
        self.chatName = chatName
        _viewModel = StateObject(wrappedValue: ChatRoomViewModel(chatId: chatId))
    }
    
    var body: some View {
        ZStack {
            VStack(spacing: 0) {
                pinnedBannerView
                messageScrollView
                replyPreviewBanner
                bottomInputBar
            }
            .navigationTitle(chatName)
            .navigationBarTitleDisplayMode(.inline)
            .toolbar(.hidden, for: .tabBar)
            .toolbar { ToolbarItem(placement: .navigationBarTrailing) { NavigationLink(destination: ChatSettingsView(chatName: chatName, viewModel: viewModel)) { Image(systemName: "line.3.horizontal").foregroundColor(.gray) } } }
            .fullScreenCover(isPresented: $showCamera) { CameraImagePicker(image: $cameraImage).ignoresSafeArea() }
            .onChange(of: cameraImage) { newImage in if let img = newImage { viewModel.uploadImage(img); cameraImage = nil } }

            attachmentMenuOverlay
            modalsAndOverlays
            
            if showFullScreenMedia {
                FullScreenMediaViewer(urlString: fullScreenMediaUrl, isGif: fullScreenMediaIsGif, isPresented: $showFullScreenMedia).zIndex(100).transition(.opacity)
            }
        }
        .onAppear { NotificationCenter.default.post(name: NSNotification.Name("HideTabBar"), object: nil) }
    }
    
    private func shouldShowDateDivider(for msg: MessageItem, in messages: [MessageItem]) -> Bool { guard let index = messages.firstIndex(where: { $0.id == msg.id }) else { return false }; if index == 0 { return true }; let previousMsg = messages[index - 1]; return !Calendar.current.isDate(msg.timestamp, inSameDayAs: previousMsg.timestamp) }
    
    @ViewBuilder private var pinnedBannerView: some View {
        if !viewModel.pinnedMessageText.isEmpty {
            HStack {
                Image(systemName: "pin.fill").foregroundColor(lyGold).rotationEffect(.degrees(45))
                if viewModel.pinnedMessageText.isImageURL {
                    if viewModel.pinnedMessageText.isGifURL { GIFView(urlString: viewModel.pinnedMessageText).frame(width: 30, height: 30).cornerRadius(4).clipped() }
                    else { AsyncImage(url: URL(string: viewModel.pinnedMessageText)) { phase in if let image = phase.image { image.resizable().scaledToFill().frame(width: 30, height: 30).cornerRadius(4).clipped() } else { Rectangle().fill(Color.gray.opacity(0.3)).frame(width: 30, height: 30).cornerRadius(4) } } }
                    Text("圖片訊息").font(.subheadline).foregroundColor(.primary).lineLimit(1)
                } else { Text(viewModel.pinnedMessageText).font(.subheadline).foregroundColor(.primary).lineLimit(1) }
                Spacer(); Button(action: { viewModel.unpinMessage() }) { Image(systemName: "xmark").foregroundColor(.gray).padding(5) }
            }.padding(.horizontal).padding(.vertical, 8).background(Color.white.opacity(0.95)).shadow(color: Color.black.opacity(0.05), radius: 3, x: 0, y: 2).onTapGesture { popupDetailText = viewModel.pinnedMessageText; showPinnedModal = true }.zIndex(10)
        }
    }
    
    private var messageScrollView: some View {
        ScrollViewReader { proxy in
            ScrollView {
                LazyVStack(spacing: 2) {
                    if viewModel.isLoadingOldMessages && viewModel.messages.count >= 10 { ProgressView().padding().onAppear { viewModel.loadMore() } }
                    
                    ForEach(viewModel.messages) { msg in
                        VStack(spacing: 12) {
                            if shouldShowDateDivider(for: msg, in: viewModel.messages) { Text(dateFormatter.string(from: msg.timestamp)).font(.system(size: 12, weight: .bold)).foregroundColor(.gray).padding(.horizontal, 14).padding(.vertical, 6).background(Color.gray.opacity(0.15)).cornerRadius(12).padding(.vertical, 10) }
                            if msg.isSystem || msg.isDeleted { Text(msg.isDeleted ? "此訊息已收回" : msg.text).font(.system(size: 11, weight: .bold)).foregroundColor(.white).padding(.horizontal, 12).padding(.vertical, 6).background(Color.gray.opacity(0.5)).cornerRadius(12).padding(.vertical, 8) }
                            else {
                                MessageRowView(
                                    msg: msg,
                                    isGroup: viewModel.isGroup,
                                    viewModel: viewModel,
                                    currentUid: storedUid,
                                    replyingToText: $replyingToText,
                                    popupDetailText: $popupDetailText,
                                    showReplyDetailModal: $showReplyDetailModal,
                                    fullScreenMediaUrl: $fullScreenMediaUrl,
                                    fullScreenMediaIsGif: $fullScreenMediaIsGif,
                                    showFullScreenMedia: $showFullScreenMedia,
                                    showReadList: $showReadList,
                                    currentReadMembers: $currentReadMembers,
                                    reactionSheetItem: $reactionSheetItem
                                )
                            }
                        }.id(msg.id)
                    }
                    Color.clear.frame(height: 20).id("BOTTOM_ANCHOR")
                }.padding(.vertical, 8)
            }
            .background(Color(UIColor.systemGroupedBackground)).opacity(isUIReady ? 1 : 0)
            .onTapGesture { UIApplication.shared.sendAction(#selector(UIResponder.resignFirstResponder), to: nil, from: nil, for: nil); withAnimation { viewModel.activeMenuMsgId = nil } }
            .onAppear { DispatchQueue.main.asyncAfter(deadline: .now() + 0.15) { if let lastId = viewModel.messages.last?.id { proxy.scrollTo(lastId, anchor: .bottom) }; withAnimation(.easeIn(duration: 0.2)) { isUIReady = true } } }
            .onChange(of: viewModel.messages.count) { _ in if isUIReady { DispatchQueue.main.asyncAfter(deadline: .now() + 0.1) { withAnimation(.easeOut(duration: 0.25)) { proxy.scrollTo("BOTTOM_ANCHOR", anchor: .bottom) } } } }
            .onChange(of: scrollTrigger) { _ in withAnimation(.easeOut(duration: 0.25)) { proxy.scrollTo("BOTTOM_ANCHOR", anchor: .bottom) } }
        }
    }
    
    @ViewBuilder private var replyPreviewBanner: some View {
        if let replyText = replyingToText { HStack { Image(systemName: "arrowshape.turn.up.left.fill").foregroundColor(lyBlue); Text("回覆：\(replyText)").font(.caption).foregroundColor(.gray).lineLimit(1); Spacer(); Button(action: { withAnimation { replyingToText = nil } }) { Image(systemName: "xmark.circle.fill").foregroundColor(.gray) } }.padding(.horizontal).padding(.vertical, 8).background(Color(UIColor.systemGray5)).transition(.move(edge: .bottom).combined(with: .opacity)) }
    }
    
    private var bottomInputBar: some View {
        HStack(alignment: .bottom, spacing: 12) {
            if isMenuExpanded {
                HStack(spacing: 12) {
                    Button(action: { withAnimation(.interactiveSpring()) { showAttachmentMenu = true } }) { Image(systemName: "plus").font(.system(size: 26)).foregroundColor(.gray) }
                    PhotosPicker(selection: $selectedPhoto, matching: .images, photoLibrary: .shared()) { Image(systemName: "photo").font(.system(size: 24)).foregroundColor(.gray) }
                    .onChange(of: selectedPhoto) { newItem in Task { if let data = try? await newItem?.loadTransferable(type: Data.self), let uiImage = UIImage(data: data) { viewModel.uploadImage(uiImage) } } }
                    Button(action: { showCamera = true }) { Image(systemName: "camera").font(.system(size: 24)).foregroundColor(.gray) }
                    Button(action: { if audioRecorder.isRecording { if let audioUrl = audioRecorder.stopRecording() { viewModel.uploadAudio(audioUrl) } } else { audioRecorder.startRecording() } }) { Image(systemName: audioRecorder.isRecording ? "stop.circle.fill" : "mic").font(.system(size: 24)).foregroundColor(audioRecorder.isRecording ? .red : .gray) }
                }.transition(.move(edge: .leading).combined(with: .opacity)).padding(.bottom, 6)
            } else { Button(action: { withAnimation(.spring()) { isMenuExpanded = true } }) { Image(systemName: "chevron.right").font(.system(size: 22, weight: .bold)).foregroundColor(.white).frame(width: 32, height: 32).background(lyBlue).clipShape(Circle()) }.transition(.scale.combined(with: .opacity)).padding(.bottom, 4) }
            
            TextField(audioRecorder.isRecording ? "錄音中..." : "Aa", text: $inputText, axis: .vertical).lineLimit(1...4).disabled(audioRecorder.isRecording).padding(.horizontal, 16).padding(.vertical, 10).background(Color.white).cornerRadius(20).overlay(RoundedRectangle(cornerRadius: 20).stroke(Color.gray.opacity(0.3), lineWidth: 1))
            .onChange(of: inputText) { newValue in if !newValue.isEmpty && isMenuExpanded { withAnimation(.spring()) { isMenuExpanded = false } } else if newValue.isEmpty && !isMenuExpanded { withAnimation(.spring()) { isMenuExpanded = true } } }
            
            Button(action: { if !inputText.isEmpty || audioRecorder.isRecording { viewModel.sendMessage(text: inputText, replyTo: replyingToText ?? ""); inputText = ""; withAnimation { replyingToText = nil }; scrollTrigger = UUID() } }) {
                Image(systemName: "paperplane.fill").font(.system(size: 22)).foregroundColor((inputText.isEmpty && !audioRecorder.isRecording) ? .gray.opacity(0.4) : lyBlue).padding(.bottom, 6).padding(.trailing, 4)
            }.disabled(inputText.isEmpty && !audioRecorder.isRecording)
        }.padding(.horizontal, 16).padding(.vertical, 10).background(Color(UIColor.systemGray6))
    }
    
    @ViewBuilder private var attachmentMenuOverlay: some View {
        if showAttachmentMenu {
            Color.black.opacity(0.3).ignoresSafeArea().onTapGesture { withAnimation(.interactiveSpring()) { showAttachmentMenu = false } }
            VStack(spacing: 0) { Spacer(); VStack(spacing: 25) { HStack(alignment: .top, spacing: 25) {
                NavigationLink(destination: PollListView(chatId: viewModel.chatId, viewModel: viewModel)) { VStack(spacing: 8) { ZStack { Circle().fill(Color.white).frame(width: 60, height: 60).shadow(color: Color.black.opacity(0.08), radius: 5, x: 0, y: 2); Image(systemName: "chart.bar.fill").font(.system(size: 24)).foregroundColor(lyBlue) }; Text("發起投票").font(.system(size: 12, weight: .medium)).foregroundColor(.gray) } }.simultaneousGesture(TapGesture().onEnded { withAnimation { showAttachmentMenu = false } })
                NavigationLink(destination: EventListView(chatId: viewModel.chatId, viewModel: viewModel)) { VStack(spacing: 8) { ZStack { Circle().fill(Color.white).frame(width: 60, height: 60).shadow(color: Color.black.opacity(0.08), radius: 5, x: 0, y: 2); Image(systemName: "calendar").font(.system(size: 24)).foregroundColor(lyBlue) }; Text("活動行程").font(.system(size: 12, weight: .medium)).foregroundColor(.gray) } }.simultaneousGesture(TapGesture().onEnded { withAnimation { showAttachmentMenu = false } })
                NavigationLink(destination: GroupBuyListView(chatId: viewModel.chatId, viewModel: viewModel)) { VStack(spacing: 8) { ZStack { Circle().fill(Color.white).frame(width: 60, height: 60).shadow(color: Color.black.opacity(0.08), radius: 5, x: 0, y: 2); Image(systemName: "basket.fill").font(.system(size: 24)).foregroundColor(lyBlue) }; Text("團購功能").font(.system(size: 12, weight: .medium)).foregroundColor(.gray) } }.simultaneousGesture(TapGesture().onEnded { withAnimation { showAttachmentMenu = false } })
                NavigationLink(destination: ConfidentialListView(chatId: viewModel.chatId, viewModel: viewModel)) { VStack(spacing: 8) { ZStack { Circle().fill(Color.white).frame(width: 60, height: 60).shadow(color: Color.black.opacity(0.08), radius: 5, x: 0, y: 2); Image(systemName: "lock.shield.fill").font(.system(size: 24)).foregroundColor(lyBlue) }; Text("機密文件").font(.system(size: 12, weight: .medium)).foregroundColor(.gray) } }.simultaneousGesture(TapGesture().onEnded { withAnimation { showAttachmentMenu = false } })
            } }.padding(.top, 30).padding(.bottom, 35).frame(maxWidth: .infinity).background(Color.white).cornerRadius(24, corners: [.topLeft, .topRight]) }.ignoresSafeArea(.all, edges: .bottom).transition(.move(edge: .bottom)).zIndex(20)
        }
    }
    
    @ViewBuilder private var modalsAndOverlays: some View {
        if showReadList {
            Color.black.opacity(0.4).ignoresSafeArea().onTapGesture { showReadList = false }
            VStack(spacing: 0) {
                HStack { Image(systemName: "eye.fill"); Text("已讀成員").font(.headline); Spacer(); Button(action: { showReadList = false }) { Image(systemName: "xmark").font(.title3) } }.padding().foregroundColor(.white).background(lyBlue).overlay(Rectangle().frame(height: 4).foregroundColor(lyGold), alignment: .bottom)
                ScrollView { LazyVGrid(columns: [GridItem(.adaptive(minimum: 80), spacing: 15)], spacing: 15) { ForEach(currentReadMembers) { member in VStack(spacing: 8) { CachedAvatarView(url: member.avatar, size: 45).overlay(Circle().stroke(Color.black, lineWidth: 2)).overlay(Circle().stroke(lyBlue, lineWidth: 3)); Text(member.name).font(.caption).fontWeight(.bold).foregroundColor(lyGold).lineLimit(1) }.padding(.vertical, 12).frame(maxWidth: .infinity).background(Color(red: 253/255, green: 252/255, blue: 245/255)).cornerRadius(8).overlay(RoundedRectangle(cornerRadius: 8).stroke(lyGold, lineWidth: 1)) } }.padding(20) }.frame(maxHeight: 300)
                Button(action: { showReadList = false }) { Text("關閉").font(.headline).foregroundColor(.gray).frame(maxWidth: .infinity).padding(.vertical, 12).background(Color.white).overlay(Rectangle().frame(height: 1).foregroundColor(Color(UIColor.systemGray5)), alignment: .top) }
            }.background(Color.white).cornerRadius(12).padding(.horizontal, 30)
        }

        if showPinnedModal || showReplyDetailModal {
            Color.black.opacity(0.4).ignoresSafeArea().onTapGesture { showPinnedModal = false; showReplyDetailModal = false }
            VStack(spacing: 0) {
                HStack { Image(systemName: showPinnedModal ? "pin.fill" : "arrowshape.turn.up.left.fill").foregroundColor(lyGold).rotationEffect(showPinnedModal ? .degrees(45) : .zero); Text(showPinnedModal ? "釘選訊息內容" : "原始回覆訊息").font(.headline); Spacer(); Button(action: { showPinnedModal = false; showReplyDetailModal = false }) { Image(systemName: "xmark").foregroundColor(.gray) } }.padding().background(Color(UIColor.systemGray6))
                ScrollView {
                    if popupDetailText.isImageURL {
                        if popupDetailText.isGifURL { GIFView(urlString: popupDetailText).frame(height: 250).cornerRadius(8).padding() }
                        else { AsyncImage(url: URL(string: popupDetailText)) { phase in if let image = phase.image { image.resizable().scaledToFit().frame(maxHeight: 250).cornerRadius(8) } else { ProgressView() } }.padding() }
                    } else { Text(popupDetailText).font(.body).padding().frame(maxWidth: .infinity, alignment: .leading) }
                }.frame(maxHeight: 300)
            }.background(Color.white).cornerRadius(12).padding(.horizontal, 40)
        }
        
        if let item = reactionSheetItem {
            Color.black.opacity(0.4).ignoresSafeArea().onTapGesture { reactionSheetItem = nil }
            VStack(spacing: 0) {
                HStack { Text("\(item.emoji) 回應成員").font(.headline); Spacer(); Button(action: { reactionSheetItem = nil }) { Image(systemName: "xmark").font(.title3) } }.padding().background(Color(UIColor.systemGray6))
                ScrollView { LazyVStack { ForEach(item.uids, id: \.self) { uid in LiveUserRow(uid: uid).padding(.horizontal) } }.padding(.vertical) }.frame(maxHeight: 300)
            }.background(Color.white).cornerRadius(12).padding(.horizontal, 30)
        }
    }
}

// MARK: - Chat Settings

struct ChatSettingsView: View {
    let chatName: String
    @ObservedObject var viewModel: ChatRoomViewModel
    @Environment(\.presentationMode) var presentationMode
    
    let lyBlue = Color(red: 0/255, green: 49/255, blue: 83/255)
    
    var body: some View {
        ScrollView {
            VStack(spacing: 25) {
                
                VStack(alignment: .leading, spacing: 8) {
                    Text("基本設定").font(.caption).foregroundColor(.gray).padding(.horizontal, 16)
                    VStack(spacing: 0) {
                        HStack { Text("群組名稱"); Spacer(); Text(chatName).foregroundColor(.gray) }.padding()
                        Divider().padding(.leading, 16)
                        HStack { Text("更換群組頭像").foregroundColor(.blue); Spacer() }.padding()
                        Divider().padding(.leading, 16)
                        NavigationLink(destination: ChatSearchView(viewModel: viewModel)) {
                            HStack {
                                Image(systemName: "magnifyingglass").foregroundColor(.blue).font(.system(size: 20))
                                Text("搜尋聊天內容").foregroundColor(.primary)
                                Spacer()
                                Image(systemName: "chevron.right").foregroundColor(.gray)
                            }.padding()
                        }
                    }
                    .background(Color.white).cornerRadius(16)
                }
                
                VStack(alignment: .leading, spacing: 8) {
                    Text("多媒體與檔案").font(.caption).foregroundColor(.gray).padding(.horizontal, 16)
                    VStack(spacing: 0) {
                        NavigationLink(destination: SharedMediaView(viewModel: viewModel)) {
                            HStack {
                                Image(systemName: "photo.on.rectangle.angled")
                                    .foregroundColor(.blue)
                                    .frame(width: 36, height: 36)
                                    .background(Color.blue.opacity(0.1))
                                    .cornerRadius(8)
                                
                                Text("歷史照片").foregroundColor(.primary)
                                Spacer()
                                Image(systemName: "chevron.right").foregroundColor(.gray)
                            }.padding()
                        }
                    }
                    .background(Color.white).cornerRadius(16)
                }
                
                VStack(alignment: .leading, spacing: 8) {
                    HStack {
                        Text("成員名單 (\(viewModel.groupMembersCount))").font(.caption).foregroundColor(.gray)
                        Spacer()
                        NavigationLink(destination: ChatMembersView(chatId: viewModel.chatId)) {
                            Text("編輯").font(.caption).foregroundColor(.blue)
                        }
                    }.padding(.horizontal, 16)
                    
                    VStack(spacing: 0) {
                        ScrollView(.horizontal, showsIndicators: false) {
                            HStack(spacing: 20) {
                                VStack(spacing: 6) { CachedAvatarView(url: "", size: 55).overlay(Circle().stroke(Color.gray.opacity(0.3), lineWidth: 1)); Text("班長").font(.caption) }
                                VStack(spacing: 6) { CachedAvatarView(url: "", size: 55).overlay(Circle().stroke(Color.gray.opacity(0.3), lineWidth: 1)); Text("俊齊").font(.caption) }
                                VStack(spacing: 6) { CachedAvatarView(url: "", size: 55).overlay(Circle().stroke(Color.gray.opacity(0.3), lineWidth: 1)); Text("博峻").font(.caption) }
                            }.padding()
                        }
                        
                        Divider().padding(.horizontal, 16)
                        
                        NavigationLink(destination: ChatMembersView(chatId: viewModel.chatId)) {
                            HStack { Spacer(); Text("展開所有成員 (\(viewModel.groupMembersCount))").foregroundColor(.blue); Spacer() }.padding()
                        }
                        
                        Divider().padding(.horizontal, 16)
                        
                        Button(action: { print("邀請新成員") }) {
                            HStack {
                                Image(systemName: "person.badge.plus").foregroundColor(lyBlue).font(.system(size: 20))
                                Text("邀請新成員").foregroundColor(lyBlue)
                                Spacer()
                            }.padding()
                        }
                    }
                    .background(Color.white).cornerRadius(16)
                }
                
                Button(action: { viewModel.deleteChat { success in if success { presentationMode.wrappedValue.dismiss() } } }) {
                    Text("退出群組")
                        .font(.headline)
                        .foregroundColor(.red)
                        .frame(maxWidth: .infinity)
                        .padding()
                        .background(Color.white)
                        .cornerRadius(16)
                }
                
            }.padding(.vertical, 20).padding(.horizontal, 16)
        }
        .background(Color(UIColor.systemGroupedBackground).ignoresSafeArea())
        .navigationBarBackButtonHidden(true)
        .toolbar {
            ToolbarItem(placement: .principal) { Text("聊天室設定").font(.headline) }
            ToolbarItem(placement: .navigationBarLeading) {
                Button("關閉") { presentationMode.wrappedValue.dismiss() }
                    .font(.system(size: 14, weight: .bold))
                    .foregroundColor(.gray)
                    .padding(.horizontal, 14).padding(.vertical, 6)
                    .background(Color.gray.opacity(0.15))
                    .cornerRadius(20)
            }
            ToolbarItem(placement: .navigationBarTrailing) {
                Button("完成") { presentationMode.wrappedValue.dismiss() }
                    .font(.system(size: 14, weight: .bold))
                    .foregroundColor(.white)
                    .padding(.horizontal, 14).padding(.vertical, 6)
                    .background(Color(white: 0.15))
                    .cornerRadius(20)
            }
        }
        .onAppear { NotificationCenter.default.post(name: NSNotification.Name("HideTabBar"), object: nil) }
    }
}

struct SharedMediaView: View {
    @ObservedObject var viewModel: ChatRoomViewModel
    var mediaMessages: [MessageItem] { return viewModel.messages.filter { $0.fileType == "image" || $0.text.isImageURL } }
    
    var body: some View {
        ScrollView {
            if mediaMessages.isEmpty { Text("尚無分享的媒體...").foregroundColor(.gray).padding(30) } else { LazyVGrid(columns: [GridItem(.adaptive(minimum: 100), spacing: 2)], spacing: 2) { ForEach(mediaMessages) { msg in let urlString = msg.fileType == "image" ? msg.fileUrl : msg.text.trimmingCharacters(in: .whitespacesAndNewlines); AsyncImage(url: URL(string: urlString)) { phase in if let image = phase.image { image.resizable().scaledToFill().frame(minWidth: 0, maxWidth: .infinity, minHeight: 100, maxHeight: 100).clipped() } else { Rectangle().fill(Color(UIColor.systemGray5)).frame(height: 100) } } } } }
        }.navigationTitle("相片與影片").navigationBarTitleDisplayMode(.inline).onAppear { NotificationCenter.default.post(name: NSNotification.Name("HideTabBar"), object: nil) }
    }
}

struct ChatSearchView: View {
    @ObservedObject var viewModel: ChatRoomViewModel
    @State private var searchText = ""
    let lyBlue = Color(red: 0/255, green: 49/255, blue: 83/255)
    var searchResults: [MessageItem] { if searchText.isEmpty { return [] }; return viewModel.messages.filter { $0.text.lowercased().contains(searchText.lowercased()) }.reversed() }
    
    var body: some View {
        VStack {
            HStack { Image(systemName: "magnifyingglass").foregroundColor(.gray); TextField("輸入關鍵字尋找對話...", text: $searchText) }.padding(12).background(Color.white).cornerRadius(10).padding()
            ScrollView {
                if searchText.isEmpty { Text("輸入上方文字開始搜尋").foregroundColor(.gray).padding() } else if searchResults.isEmpty { Text("找不到符合的紀錄").foregroundColor(.red).padding() } else { VStack(spacing: 10) { ForEach(searchResults) { msg in VStack(alignment: .leading, spacing: 4) { HStack { Text(msg.isMine ? "您" : msg.senderName).font(.caption).fontWeight(.bold).foregroundColor(lyBlue); Spacer(); Text(msg.timestamp, style: .date).font(.caption2).foregroundColor(.gray) }; Text(msg.text).font(.subheadline).foregroundColor(.primary) }.padding().frame(maxWidth: .infinity, alignment: .leading).background(Color.white).cornerRadius(8) } }.padding(.horizontal) }
            }
        }.background(Color(UIColor.systemGroupedBackground).ignoresSafeArea()).navigationTitle("歷史對話搜尋").navigationBarTitleDisplayMode(.inline).onAppear { NotificationCenter.default.post(name: NSNotification.Name("HideTabBar"), object: nil) }
    }
}

struct ChatMembersView: View {
    let chatId: String
    @State private var members: [String] = []
    @State private var isLoading = true
    var body: some View {
        List { if isLoading { ProgressView("成員載入中...") } else if members.isEmpty { Text("此群組尚無成員。").foregroundColor(.gray) } else { ForEach(members, id: \.self) { uid in LiveUserRow(uid: uid).padding(.vertical, 8) } } }
        .navigationTitle("群組成員名單").navigationBarTitleDisplayMode(.inline).onAppear { fetchMembers(); NotificationCenter.default.post(name: NSNotification.Name("HideTabBar"), object: nil) }
    }
    private func fetchMembers() { Firestore.firestore().collection("chats").document(chatId).getDocument { doc, _ in if let data = doc?.data(), let membersArray = data["members"] as? [String] { self.members = membersArray }; self.isLoading = false } }
}

// MARK: - 9. 擴充功能列表與發起 (Features Views)

struct PollListView: View {
    let chatId: String
    @ObservedObject var viewModel: ChatRoomViewModel
    @State private var polls: [PollItem] = []
    @State private var isLoading = true
    @State private var selectedPollToShare: PollItem? = nil
    @State private var showShareConfirmation = false
    let lyBlue = Color(red: 0/255, green: 49/255, blue: 83/255)
    let lyGold = Color(red: 184/255, green: 134/255, blue: 11/255)
    
    var body: some View {
        VStack(spacing: 0) {
            if isLoading { Spacer(); ProgressView("載入投票資料中..."); Spacer() } else if polls.isEmpty { Spacer(); Text("尚無投票，點擊右上角新增。").foregroundColor(.gray); Spacer() } else {
                List {
                    Section { PrivilegeBanner(myName: viewModel.myName, color: lyGold) }
                    ForEach(polls) { poll in
                        pollCardRow(poll: poll).swipeActions(edge: .trailing, allowsFullSwipe: false) {
                            if canEdit(creatorUid: poll.creatorUid, myName: viewModel.myName) {
                                Button(role: .destructive) { Firestore.firestore().collection("polls").document(poll.id).delete() } label: { Label("刪除", systemImage: "trash.fill") }
                                NavigationLink { CreatePollView(chatId: chatId, editingPollId: poll.id) } label: { Label("編輯", systemImage: "pencil") }.tint(lyBlue)
                            } else { Button { } label: { Label("權限不足", systemImage: "lock.fill") }.tint(.gray) }
                        }
                    }
                }.listStyle(.plain).background(Color(UIColor.systemGroupedBackground))
            }
        }.background(Color(UIColor.systemGroupedBackground).ignoresSafeArea()).navigationTitle("投票清單").navigationBarTitleDisplayMode(.inline).toolbar { ToolbarItem(placement: .navigationBarTrailing) { NavigationLink(destination: CreatePollView(chatId: chatId)) { Image(systemName: "plus").font(.title2).foregroundColor(.white) } } }
        .onAppear(perform: loadPolls).confirmationDialog("確認分享", isPresented: $showShareConfirmation, titleVisibility: .visible) { Button("分享至聊天室") { if let poll = selectedPollToShare { sendPollToChat(poll: poll) } }; Button("取消", role: .cancel) { } } message: { Text("您確定要將「\(selectedPollToShare?.question ?? "")」這項投票分享至目前的聊天室嗎？") }
    }
    
    private func pollCardRow(poll: PollItem) -> some View {
        Button(action: { selectedPollToShare = poll; showShareConfirmation = true }) {
            HStack(spacing: 0) {
                Rectangle().fill(lyBlue).frame(width: 6)
                VStack(alignment: .leading, spacing: 10) {
                    HStack { Image(systemName: "chart.bar.fill"); Text(poll.question).font(.title3).fontWeight(.bold) }.foregroundColor(lyBlue)
                    HStack(alignment: .bottom) {
                        VStack(alignment: .leading, spacing: 4) { HStack(spacing: 4) { Image(systemName: "clock"); Text("截止："); Text(poll.deadline.isEmpty ? "無期限" : poll.deadline.replacingOccurrences(of: "T", with: " ")) }.font(.caption).foregroundColor(.red) }
                        Spacer()
                        Text("建立人：\(poll.creatorName)").font(.caption).fontWeight(.bold).foregroundColor(lyBlue).padding(.horizontal, 10).padding(.vertical, 4).background(Color.gray.opacity(0.1)).cornerRadius(12)
                    }
                }.padding(.vertical, 12).padding(.horizontal, 15).background(Color.white)
            }.cornerRadius(12).overlay(RoundedRectangle(cornerRadius: 12).stroke(Color.gray.opacity(0.2), lineWidth: 1))
        }.buttonStyle(PlainButtonStyle()).listRowInsets(EdgeInsets(top: 6, leading: 16, bottom: 6, trailing: 16)).listRowSeparator(.hidden).listRowBackground(Color.clear)
    }
    
    private func loadPolls() { Firestore.firestore().collection("polls").order(by: "createdAt", descending: true).addSnapshotListener { snap, error in guard let docs = snap?.documents else { self.isLoading = false; return }; self.polls = docs.map { doc in let data = doc.data(); return PollItem(id: doc.documentID, question: data["question"] as? String ?? "未命名投票", content: data["content"] as? String ?? "", deadline: data["deadline"] as? String ?? "", creatorUid: data["creatorUid"] as? String ?? "", creatorName: data["creatorName"] as? String ?? "未知", allowAddOptions: data["allowAddOptions"] as? Bool ?? false, options: data["options"] as? [String] ?? []) }; self.isLoading = false } }
    
    private func sendPollToChat(poll: PollItem) { guard let currentUid = Auth.auth().currentUser?.uid else { return }; let db = Firestore.firestore(); let batch = db.batch(); let chatRef = db.collection("chats").document(chatId); let msgRef = chatRef.collection("messages").document(); var votesMap: [String: [String]] = [:]; var optionCreators: [String: String] = [:]; poll.options.forEach { opt in votesMap[opt] = []; optionCreators[opt] = currentUid }; let payload: [String: Any] = [ "type": "poll", "pollId": poll.id, "question": poll.question, "content": poll.content, "options": poll.options, "votes": votesMap, "deadline": poll.deadline, "allowAddOptions": poll.allowAddOptions, "optionCreators": optionCreators ]; if let jsonData = try? JSONSerialization.data(withJSONObject: payload), let jsonString = String(data: jsonData, encoding: .utf8) { batch.setData(["text": jsonString, "senderId": currentUid, "timestamp": FieldValue.serverTimestamp()], forDocument: msgRef); batch.updateData(["lastMessage": "📊 發起了投票：\(poll.question)", "lastSenderId": currentUid, "updatedAt": FieldValue.serverTimestamp(), "lastReadAt.\(currentUid)": FieldValue.serverTimestamp()], forDocument: chatRef); batch.commit() } }
}

struct CreatePollView: View {
    let chatId: String; var editingPollId: String? = nil; @Environment(\.presentationMode) var presentationMode
    @State private var question: String = ""; @State private var content: String = ""; @State private var deadline: Date = Date().addingTimeInterval(86400); @State private var useDeadline: Bool = false; @State private var allowAddOptions: Bool = false; @State private var options: [String] = ["", ""]; @State private var isSubmitting = false; @State private var isDataLoaded = false
    let lyBlue = Color(red: 0/255, green: 49/255, blue: 83/255)
    
    var body: some View {
        ScrollView { VStack(alignment: .leading, spacing: 20) {
            VStack(alignment: .leading, spacing: 8) { Text("投票主題 (必填)").font(.subheadline).foregroundColor(lyBlue).fontWeight(.bold); TextField("例如：下次要買哪間的法棍", text: $question).padding().background(Color.white).cornerRadius(8).overlay(RoundedRectangle(cornerRadius: 8).stroke(Color.gray.opacity(0.3), lineWidth: 1)) }
            VStack(alignment: .leading, spacing: 8) { Text("投票內容說明 (選填)").font(.subheadline).foregroundColor(lyBlue).fontWeight(.bold); TextEditor(text: $content).frame(minHeight: 100).padding(8).background(Color.white).cornerRadius(8).overlay(RoundedRectangle(cornerRadius: 8).stroke(Color.gray.opacity(0.3), lineWidth: 1)) }
            VStack(alignment: .leading, spacing: 8) { Toggle(isOn: $useDeadline) { Text("設定投票結束時間").font(.subheadline).foregroundColor(lyBlue).fontWeight(.bold) }.tint(lyBlue); if useDeadline { DatePicker("", selection: $deadline).datePickerStyle(.compact).labelsHidden().padding(.vertical, 8) } }
            Toggle(isOn: $allowAddOptions) { Text("允許成員新增選項").font(.subheadline).foregroundColor(lyBlue).fontWeight(.bold) }.tint(lyBlue).padding().background(Color(UIColor.systemGray6)).cornerRadius(8).overlay(RoundedRectangle(cornerRadius: 8).stroke(lyBlue.opacity(0.5), style: StrokeStyle(lineWidth: 1, dash: [5])))
            VStack(alignment: .leading, spacing: 10) { Text("選項設定").font(.subheadline).foregroundColor(lyBlue).fontWeight(.bold); ForEach(0..<options.count, id: \.self) { index in HStack { TextField("選項 \(index + 1)", text: $options[index]).padding().background(Color.white).cornerRadius(8).overlay(RoundedRectangle(cornerRadius: 8).stroke(Color.gray.opacity(0.3), lineWidth: 1)); if options.count > 2 { Button(action: { options.remove(at: index) }) { Image(systemName: "xmark.circle.fill").foregroundColor(.red).font(.title2) } } } }; Button(action: { options.append("") }) { HStack { Image(systemName: "plus"); Text("新增選項") }.font(.headline).foregroundColor(lyBlue).frame(maxWidth: .infinity).padding().background(Color(UIColor.systemGray6)).cornerRadius(8).overlay(RoundedRectangle(cornerRadius: 8).stroke(lyBlue.opacity(0.5), style: StrokeStyle(lineWidth: 1, dash: [5]))) } }
            Button(action: submitPoll) { HStack { if isSubmitting { ProgressView().tint(.white).padding(.trailing, 5) } else { Image(systemName: editingPollId == nil ? "paperplane.fill" : "save.fill") }; Text(isSubmitting ? "處理中..." : (editingPollId == nil ? "建立儲存" : "儲存修改")) }.font(.headline).foregroundColor(.white).frame(maxWidth: .infinity).padding().background(lyBlue).cornerRadius(10).shadow(color: lyBlue.opacity(0.3), radius: 5, x: 0, y: 3) }.disabled(isSubmitting || question.trimmingCharacters(in: .whitespaces).isEmpty || !isDataLoaded).padding(.top, 20)
        }.padding(20) }.background(Color(UIColor.systemGroupedBackground).ignoresSafeArea()).navigationTitle(editingPollId == nil ? "發起投票" : "編輯投票").navigationBarTitleDisplayMode(.inline).onAppear(perform: loadEditingDataIfNeeded)
    }
    
    private func loadEditingDataIfNeeded() { guard let pollId = editingPollId else { isDataLoaded = true; return }; Firestore.firestore().collection("polls").document(pollId).getDocument { doc, _ in if let data = doc?.data() { self.question = data["question"] as? String ?? ""; self.content = data["content"] as? String ?? ""; self.allowAddOptions = data["allowAddOptions"] as? Bool ?? false; self.options = data["options"] as? [String] ?? ["", ""]; let deadlineStr = data["deadline"] as? String ?? ""; if !deadlineStr.isEmpty { let f = DateFormatter(); f.dateFormat = "yyyy-MM-dd'T'HH:mm"; if let d = f.date(from: deadlineStr) { self.deadline = d; self.useDeadline = true } } }; self.isDataLoaded = true } }
    private func submitPoll() { let cleanOptions = options.map { $0.trimmingCharacters(in: .whitespaces) }.filter { !$0.isEmpty }; if cleanOptions.count < 2 { return }; isSubmitting = true; let db = Firestore.firestore(); let uid = Auth.auth().currentUser?.uid ?? ""; let userName = Auth.auth().currentUser?.displayName ?? "未知"; let deadlineStr = useDeadline ? ISO8601DateFormatter().string(from: deadline) : ""; var pollData: [String: Any] = [ "question": question, "content": content, "deadline": deadlineStr, "allowAddOptions": allowAddOptions, "options": cleanOptions, "creatorUid": uid, "creatorName": userName ]; if let pollId = editingPollId { pollData["updatedAt"] = FieldValue.serverTimestamp(); db.collection("polls").document(pollId).updateData(pollData) { _ in isSubmitting = false; presentationMode.wrappedValue.dismiss() } } else { pollData["createdAt"] = FieldValue.serverTimestamp(); db.collection("polls").addDocument(data: pollData) { error in isSubmitting = false; presentationMode.wrappedValue.dismiss() } } }
}

struct EventListView: View {
    let chatId: String
    @ObservedObject var viewModel: ChatRoomViewModel
    @State private var items: [EventItem] = []
    @State private var isLoading = true
    let lyBlue = Color(red: 0/255, green: 49/255, blue: 83/255)
    let lyGold = Color(red: 184/255, green: 134/255, blue: 11/255)
    
    var body: some View {
        VStack(spacing: 0) {
            if isLoading { Spacer(); ProgressView("載入中..."); Spacer() } else if items.isEmpty { Spacer(); Text("尚無活動。").foregroundColor(.gray); Spacer() } else {
                List { Section { PrivilegeBanner(myName: viewModel.myName, color: lyGold) }; ForEach(items) { item in GenericListRow(title: item.title, subtitle: item.time.isEmpty ? "無時間" : item.time, creator: item.creatorName, icon: "calendar", color: lyBlue) { sendToChat(item) }.swipeActions(edge: .trailing, allowsFullSwipe: false) { if canEdit(creatorUid: item.creatorUid, myName: viewModel.myName) { Button(role: .destructive) { Firestore.firestore().collection("events").document(item.id).delete() } label: { Label("刪除", systemImage: "trash.fill") } } else { Button { } label: { Label("權限不足", systemImage: "lock.fill") }.tint(.gray) } } } }.listStyle(.plain).background(Color(UIColor.systemGroupedBackground))
            }
        }.background(Color(UIColor.systemGroupedBackground).ignoresSafeArea()).navigationTitle("活動行程").navigationBarTitleDisplayMode(.inline).toolbar { ToolbarItem(placement: .navigationBarTrailing) { Button("新增(待開發)") { }.foregroundColor(.white) } }.onAppear { load(); NotificationCenter.default.post(name: NSNotification.Name("HideTabBar"), object: nil) }
    }
    
    private func load() { Firestore.firestore().collection("events").order(by: "createdAt", descending: true).addSnapshotListener { snap, _ in self.items = (snap?.documents ?? []).map { doc in let data = doc.data(); let fields = data["fields"] as? [[String: Any]] ?? []; let time = fields.first(where: { $0["type"] as? String == "time" })?["content"] as? String ?? ""; return EventItem(id: doc.documentID, title: data["title"] as? String ?? "未命名", time: time.replacingOccurrences(of: "T", with: " "), creatorUid: data["creatorUid"] as? String ?? "", creatorName: data["creatorName"] as? String ?? "未知", rawData: data) }; self.isLoading = false } }
    private func sendToChat(_ item: EventItem) { guard let uid = Auth.auth().currentUser?.uid else { return }; let payload: [String: Any] = [ "type": "event_share", "eventId": item.id, "title": item.title, "timeStr": item.time ]; if let jsonData = try? JSONSerialization.data(withJSONObject: payload), let str = String(data: jsonData, encoding: .utf8) { let db = Firestore.firestore(); let batch = db.batch(); let cRef = db.collection("chats").document(chatId); let mRef = cRef.collection("messages").document(); batch.setData(["text": str, "senderId": uid, "timestamp": FieldValue.serverTimestamp()], forDocument: mRef); batch.updateData(["lastMessage": "📅 分享了活動：\(item.title)", "lastSenderId": uid, "updatedAt": FieldValue.serverTimestamp()], forDocument: cRef); batch.commit() } }
}

struct GroupBuyListView: View {
    let chatId: String; @ObservedObject var viewModel: ChatRoomViewModel
    @State private var items: [GBItem] = []; @State private var isLoading = true
    
    var body: some View { VStack(spacing: 0) { if isLoading { Spacer(); ProgressView("載入中..."); Spacer() } else if items.isEmpty { Spacer(); Text("尚無團購，點擊右上角新增。").foregroundColor(.gray); Spacer() } else { List { Section { PrivilegeBanner(myName: viewModel.myName, color: .green) }; ForEach(items) { item in GenericListRow(title: item.title, subtitle: item.deadline.isEmpty ? "無期限" : item.deadline, creator: item.creatorName, icon: "basket.fill", color: .green) { sendToChat(item) }.swipeActions(edge: .trailing, allowsFullSwipe: false) { if canEdit(creatorUid: item.creatorUid, myName: viewModel.myName) { Button(role: .destructive) { Firestore.firestore().collection("group_buys").document(item.id).delete() } label: { Label("刪除", systemImage: "trash.fill") } } else { Button { } label: { Label("權限不足", systemImage: "lock.fill") }.tint(.gray) } } } }.listStyle(.plain).background(Color(UIColor.systemGroupedBackground)) } }.background(Color(UIColor.systemGroupedBackground).ignoresSafeArea()).navigationTitle("團購清單").navigationBarTitleDisplayMode(.inline).toolbar { ToolbarItem(placement: .navigationBarTrailing) { NavigationLink(destination: CreateGroupBuyView(chatId: chatId)) { Image(systemName: "plus") } } }.onAppear { load(); NotificationCenter.default.post(name: NSNotification.Name("HideTabBar"), object: nil) }
    }
    
    private func load() { Firestore.firestore().collection("group_buys").order(by: "createdAt", descending: true).addSnapshotListener { snap, _ in self.items = (snap?.documents ?? []).map { doc in let data = doc.data(); return GBItem(id: doc.documentID, title: data["title"] as? String ?? "未命名", deadline: data["deadline"] as? String ?? "", creatorUid: data["initiatorUid"] as? String ?? "", creatorName: data["initiator"] as? String ?? "未知", rawData: data) }; self.isLoading = false } }
    private func sendToChat(_ item: GBItem) { guard let uid = Auth.auth().currentUser?.uid else { return }; let payload: [String: Any] = [ "type": "group_buy", "groupBuyId": item.id, "title": item.title, "initiator": item.creatorName, "deadline": item.deadline ]; if let jsonData = try? JSONSerialization.data(withJSONObject: payload), let str = String(data: jsonData, encoding: .utf8) { let db = Firestore.firestore(); let batch = db.batch(); let cRef = db.collection("chats").document(chatId); let mRef = cRef.collection("messages").document(); batch.setData(["text": str, "senderId": uid, "timestamp": FieldValue.serverTimestamp()], forDocument: mRef); batch.updateData(["lastMessage": "🛍️ 分享了團購：\(item.title)", "lastSenderId": uid, "updatedAt": FieldValue.serverTimestamp()], forDocument: cRef); batch.commit() } }
}

struct CreateGroupBuyView: View {
    let chatId: String; @Environment(\.presentationMode) var presentationMode
    @State private var title: String = ""; @State private var itemName: String = ""; @State private var price: String = ""; @State private var deadline: Date = Date().addingTimeInterval(86400); @State private var isSubmitting = false
    let lyBlue = Color(red: 0/255, green: 49/255, blue: 83/255)
    
    var body: some View { ScrollView { VStack(alignment: .leading, spacing: 20) { inputSection(title: "團購主題", placeholder: "例如：下午茶訂購", text: $title); inputSection(title: "商品名稱", placeholder: "例如：珍珠奶茶", text: $itemName); VStack(alignment: .leading, spacing: 8) { Text("單價").font(.subheadline).foregroundColor(lyBlue).fontWeight(.bold); TextField("輸入金額...", text: $price).keyboardType(.numberPad).padding().background(Color.white).cornerRadius(8).overlay(RoundedRectangle(cornerRadius: 8).stroke(Color.gray.opacity(0.3), lineWidth: 1)) }; VStack(alignment: .leading, spacing: 8) { Text("截止時間").font(.subheadline).foregroundColor(lyBlue).fontWeight(.bold); DatePicker("", selection: $deadline).datePickerStyle(.compact).labelsHidden().padding(.vertical, 8) }; Button(action: submitGroupBuy) { HStack { if isSubmitting { ProgressView().tint(.white).padding(.trailing, 5) }; Text(isSubmitting ? "處理中..." : "發布團購") }.font(.headline).foregroundColor(.white).frame(maxWidth: .infinity).padding().background(lyBlue).cornerRadius(10).shadow(color: lyBlue.opacity(0.3), radius: 5, x: 0, y: 3) }.disabled(isSubmitting || title.isEmpty || itemName.isEmpty || price.isEmpty).padding(.top, 20) }.padding(20) }.background(Color(UIColor.systemGroupedBackground).ignoresSafeArea()).navigationTitle("發起團購").navigationBarTitleDisplayMode(.inline).onAppear { NotificationCenter.default.post(name: NSNotification.Name("HideTabBar"), object: nil) } }
    
    func inputSection(title: String, placeholder: String, text: Binding<String>) -> some View { VStack(alignment: .leading, spacing: 8) { Text(title).font(.subheadline).foregroundColor(lyBlue).fontWeight(.bold); TextField(placeholder, text: text).padding().background(Color.white).cornerRadius(8).overlay(RoundedRectangle(cornerRadius: 8).stroke(Color.gray.opacity(0.3), lineWidth: 1)) } }
    
    private func submitGroupBuy() { isSubmitting = true; let db = Firestore.firestore(); guard let uid = Auth.auth().currentUser?.uid else { return }; let userName = Auth.auth().currentUser?.displayName ?? "未知"; let deadlineStr = ISO8601DateFormatter().string(from: deadline); let payload: [String: Any] = [ "type": "group_buy", "title": title, "itemName": itemName, "price": Int(price) ?? 0, "deadline": deadlineStr, "initiatorUid": uid, "initiator": userName, "orders": [String: Any]() ]; let batch = db.batch(); let chatRef = db.collection("chats").document(chatId); let msgRef = chatRef.collection("messages").document(); if let jsonData = try? JSONSerialization.data(withJSONObject: payload), let jsonStr = String(data: jsonData, encoding: .utf8) { batch.setData(["text": jsonStr, "senderId": uid, "timestamp": FieldValue.serverTimestamp()], forDocument: msgRef); batch.updateData([ "lastMessage": "🛍️ 發起了團購：\(title)", "lastSenderId": uid, "updatedAt": FieldValue.serverTimestamp(), "lastReadAt.\(uid)": FieldValue.serverTimestamp() ], forDocument: chatRef); batch.commit { _ in DispatchQueue.main.async { isSubmitting = false; presentationMode.wrappedValue.dismiss() } } } }
}

struct ConfidentialListView: View {
    let chatId: String; @ObservedObject var viewModel: ChatRoomViewModel
    @State private var items: [ConfItem] = []; @State private var isLoading = true
    
    var body: some View { VStack(spacing: 0) { if isLoading { Spacer(); ProgressView("載入中..."); Spacer() } else if items.isEmpty { Spacer(); Text("尚無機密檔案，點擊右上角新增。").foregroundColor(.gray); Spacer() } else { List { Section { PrivilegeBanner(myName: viewModel.myName, color: .red) }; ForEach(items) { item in GenericListRow(title: item.title, subtitle: "無期限", creator: item.creatorName, icon: "lock.shield.fill", color: .red) { sendToChat(item) }.swipeActions(edge: .trailing, allowsFullSwipe: false) { if canEdit(creatorUid: item.creatorUid, myName: viewModel.myName) { Button(role: .destructive) { Firestore.firestore().collection("confidentials").document(item.id).delete() } label: { Label("刪除", systemImage: "trash.fill") } } else { Button { } label: { Label("權限不足", systemImage: "lock.fill") }.tint(.gray) } } } }.listStyle(.plain).background(Color(UIColor.systemGroupedBackground)) } }.background(Color(UIColor.systemGroupedBackground).ignoresSafeArea()).navigationTitle("機密檔案").navigationBarTitleDisplayMode(.inline).toolbar { ToolbarItem(placement: .navigationBarTrailing) { NavigationLink(destination: CreateConfidentialView(chatId: chatId)) { Image(systemName: "plus") } } }.onAppear { load(); NotificationCenter.default.post(name: NSNotification.Name("HideTabBar"), object: nil) }
    }
    
    private func load() { Firestore.firestore().collection("confidentials").order(by: "createdAt", descending: true).addSnapshotListener { snap, _ in self.items = (snap?.documents ?? []).map { doc in let data = doc.data(); return ConfItem(id: doc.documentID, title: data["title"] as? String ?? "機密檔案", creatorUid: data["creatorUid"] as? String ?? "", creatorName: data["creatorName"] as? String ?? "未知", rawData: data) }; self.isLoading = false } }
    private func sendToChat(_ item: ConfItem) { guard let uid = Auth.auth().currentUser?.uid else { return }; var payload = item.rawData; payload["type"] = "confidential"; if let jsonData = try? JSONSerialization.data(withJSONObject: payload), let str = String(data: jsonData, encoding: .utf8) { let db = Firestore.firestore(); let batch = db.batch(); let cRef = db.collection("chats").document(chatId); let mRef = cRef.collection("messages").document(); batch.setData(["text": str, "senderId": uid, "timestamp": FieldValue.serverTimestamp()], forDocument: mRef); batch.updateData(["lastMessage": "🔒 發送了一份機密文件", "lastSenderId": uid, "updatedAt": FieldValue.serverTimestamp()], forDocument: cRef); batch.commit() } }
}

struct CreateConfidentialView: View {
    let chatId: String; @Environment(\.presentationMode) var presentationMode
    @State private var title: String = ""; @State private var content: String = ""; @State private var unlockPassword: String = ""; @State private var isSubmitting = false
    let lyBlue = Color(red: 0/255, green: 49/255, blue: 83/255)
    
    var body: some View { ScrollView { VStack(alignment: .leading, spacing: 20) { VStack(alignment: .leading, spacing: 8) { Text("機密標題").font(.subheadline).foregroundColor(.red).fontWeight(.bold); TextField("例如：本月薪資單", text: $title).padding().background(Color.white).cornerRadius(8).overlay(RoundedRectangle(cornerRadius: 8).stroke(Color.red.opacity(0.3), lineWidth: 1)) }; VStack(alignment: .leading, spacing: 8) { Text("解鎖密碼").font(.subheadline).foregroundColor(.red).fontWeight(.bold); SecureField("請設定解鎖密碼...", text: $unlockPassword).padding().background(Color.white).cornerRadius(8).overlay(RoundedRectangle(cornerRadius: 8).stroke(Color.red.opacity(0.3), lineWidth: 1)) }; VStack(alignment: .leading, spacing: 8) { Text("機密內容").font(.subheadline).foregroundColor(.red).fontWeight(.bold); TextEditor(text: $content).frame(minHeight: 150).padding(8).background(Color.white).cornerRadius(8).overlay(RoundedRectangle(cornerRadius: 8).stroke(Color.red.opacity(0.3), lineWidth: 1)) }; Button(action: submitConfidential) { HStack { if isSubmitting { ProgressView().tint(.white).padding(.trailing, 5) } else { Image(systemName: "lock.shield.fill") }; Text(isSubmitting ? "加密中..." : "加密並發送") }.font(.headline).foregroundColor(.white).frame(maxWidth: .infinity).padding().background(Color.red).cornerRadius(10).shadow(color: Color.red.opacity(0.3), radius: 5, x: 0, y: 3) }.disabled(isSubmitting || title.isEmpty || content.isEmpty || unlockPassword.isEmpty).padding(.top, 20) }.padding(20) }.background(Color(UIColor.systemGroupedBackground).ignoresSafeArea()).navigationTitle("發送機密文件").navigationBarTitleDisplayMode(.inline).onAppear { NotificationCenter.default.post(name: NSNotification.Name("HideTabBar"), object: nil) } }
    
    private func submitConfidential() { isSubmitting = true; let db = Firestore.firestore(); guard let uid = Auth.auth().currentUser?.uid else { return }; let payload: [String: Any] = [ "type": "confidential", "title": title, "content": content, "password": unlockPassword ]; let batch = db.batch(); let chatRef = db.collection("chats").document(chatId); let msgRef = chatRef.collection("messages").document(); if let jsonData = try? JSONSerialization.data(withJSONObject: payload), let jsonStr = String(data: jsonData, encoding: .utf8) { batch.setData(["text": jsonStr, "senderId": uid, "timestamp": FieldValue.serverTimestamp()], forDocument: msgRef); batch.updateData([ "lastMessage": "🔒 發送了一份機密文件", "lastSenderId": uid, "updatedAt": FieldValue.serverTimestamp(), "lastReadAt.\(uid)": FieldValue.serverTimestamp() ], forDocument: chatRef); batch.commit { _ in DispatchQueue.main.async { isSubmitting = false; presentationMode.wrappedValue.dismiss() } } } }
}

// MARK: - 10. 擴充功能詳細頁 (Card Detail Views)

struct EventDetailView: View {
    var cardData: [String: Any]; @ObservedObject var viewModel: ChatRoomViewModel
    @State private var liveEventData: [String: Any]? = nil
    let lyBlue = Color(red: 0/255, green: 49/255, blue: 83/255)
    let lyGold = Color(red: 184/255, green: 134/255, blue: 11/255)
    var currentData: [String: Any] { return liveEventData ?? cardData["fullEventData"] as? [String: Any] ?? cardData }

    var body: some View {
        let ev = currentData
        let title = ev["title"] as? String ?? "未命名活動"
        let fields = ev["fields"] as? [[String: Any]] ?? []
        let timeField = fields.first(where: { $0["type"] as? String == "time" })?["content"] as? String ?? "未提供"
        let meetupField = fields.first(where: { $0["type"] as? String == "meetup" })?["content"] as? String ?? "未提供"
        let locField = fields.first(where: { $0["type"] as? String == "location" })?["content"] as? String ?? "未提供"
        let contentField = fields.first(where: { $0["type"] as? String == "content_text" })?["content"] as? String ?? "無詳細內容"

        let pUids = parseOldArrayData(from: ev["participants"])
        let pNames = parseOldArrayData(from: ev["participantNames"])
        let finalParticipants = !pNames.isEmpty ? pNames : pUids
        let hasJoined = finalParticipants.contains(viewModel.myName) || finalParticipants.contains(Auth.auth().currentUser?.uid ?? "")

        ScrollView {
            VStack(alignment: .leading, spacing: 20) {
                Text(title).font(.title2).fontWeight(.black).foregroundColor(lyBlue).multilineTextAlignment(.center).frame(maxWidth: .infinity).padding(.top, 20)

                VStack(alignment: .leading, spacing: 25) {
                    detailSection(title: "活動時間", content: timeField.replacingOccurrences(of: "T", with: " "))
                    detailSection(title: "集合時間", content: meetupField.replacingOccurrences(of: "T", with: " "))
                    detailSection(title: "地點", content: locField)
                    detailSection(title: "活動內容", content: contentField)

                    VStack(spacing: 0) {
                        Text("請確認行程").font(.headline).foregroundColor(lyGold).frame(maxWidth: .infinity).padding(.vertical, 12).background(lyBlue)
                        HStack(spacing: 15) {
                            Button(action: { toggleParticipation(isJoin: true) }) { HStack { if hasJoined { Image(systemName: "checkmark") }; Text(hasJoined ? "已參與" : "參與") }.fontWeight(.bold).frame(maxWidth: .infinity).padding(.vertical, 12).background(hasJoined ? LinearGradient(gradient: Gradient(colors: [Color(red: 249/255, green: 226/255, blue: 125/255), Color(red: 212/255, green: 175/255, blue: 55/255)]), startPoint: .topLeading, endPoint: .bottomTrailing) : LinearGradient(gradient: Gradient(colors: [Color(UIColor.systemGray5)]), startPoint: .top, endPoint: .bottom)).foregroundColor(hasJoined ? .white : .gray).cornerRadius(25) }
                            Button(action: { toggleParticipation(isJoin: false) }) { Text("下次").fontWeight(.bold).frame(maxWidth: .infinity).padding(.vertical, 12).background(Color.gray.opacity(0.6)).foregroundColor(.white).cornerRadius(25) }
                        }.padding(20).background(Color.white)
                    }.cornerRadius(12).overlay(RoundedRectangle(cornerRadius: 12).stroke(Color.gray.opacity(0.2), lineWidth: 1)).padding(.vertical, 10)

                    VStack(alignment: .leading, spacing: 15) {
                        HStack { Rectangle().fill(lyGold).frame(width: 4, height: 16); Text("參與成員").font(.headline).fontWeight(.bold).foregroundColor(lyBlue); Text("(共 \(finalParticipants.count) 人)").font(.subheadline).foregroundColor(.gray) }
                        if finalParticipants.isEmpty { Text("尚無成員參與...").foregroundColor(.gray).font(.subheadline) }
                        else { LazyVGrid(columns: [GridItem(.adaptive(minimum: 60))], spacing: 15) { ForEach(finalParticipants, id: \.self) { identifier in LiveAvatarView(uidOrName: identifier, showName: true, size: 50) } } }
                    }
                }.padding(.horizontal, 20)
            }.padding(.bottom, 40)
        }.background(Color.white.ignoresSafeArea()).navigationTitle("活動詳情").navigationBarTitleDisplayMode(.inline).onAppear { startListeningToEvent(); NotificationCenter.default.post(name: NSNotification.Name("HideTabBar"), object: nil) }
    }

    private func startListeningToEvent() { let eventId = cardData["eventId"] as? String ?? ""; if eventId.isEmpty { return }; Firestore.firestore().collection("events").document(eventId).addSnapshotListener { doc, _ in if let data = doc?.data() { self.liveEventData = data } } }
    private func toggleParticipation(isJoin: Bool) { let eventId = cardData["eventId"] as? String ?? ""; guard let uid = Auth.auth().currentUser?.uid else { return }; if eventId.isEmpty { return }; let ref = Firestore.firestore().collection("events").document(eventId); Firestore.firestore().runTransaction({ (transaction, errorPointer) -> Any? in let doc: DocumentSnapshot; do { doc = try transaction.getDocument(ref) } catch { return nil }; guard let data = doc.data() else { return nil }; var parts = parseOldArrayData(from: data["participants"]); var names = parseOldArrayData(from: data["participantNames"]); let myName = viewModel.myName; if isJoin { if !parts.contains(uid) { parts.append(uid) }; if !names.contains(myName) { names.append(myName) } } else { parts.removeAll(where: { $0 == uid }); names.removeAll(where: { $0 == myName }) }; transaction.updateData(["participants": parts, "participantNames": names], forDocument: ref); return nil }) { _, _ in } }
    func detailSection(title: String, content: String) -> some View { VStack(alignment: .leading, spacing: 8) { Text("[ \(title) ]").font(.headline).fontWeight(.black).foregroundColor(lyBlue); Text(content).font(.body).foregroundColor(.primary).lineSpacing(5) } }
}

struct PollDetailView: View {
    let msgId: String; var cardData: [String: Any]; @ObservedObject var viewModel: ChatRoomViewModel
    @State private var showVotersModal = false; let lyBlue = Color(red: 0/255, green: 49/255, blue: 83/255); let lyGold = Color(red: 184/255, green: 134/255, blue: 11/255)

    var body: some View {
        let options = cardData["options"] as? [String] ?? []; let votes = cardData["votes"] as? [String: [String]] ?? [:]; let totalVotes = votes.values.reduce(0) { $0 + $1.count }; let deadline = cardData["deadline"] as? String ?? ""; let currentUid = Auth.auth().currentUser?.uid ?? ""; let isExpired = !deadline.isEmpty && (Date() > { let formatter = DateFormatter(); formatter.dateFormat = "yyyy-MM-dd'T'HH:mm"; return formatter.date(from: deadline) ?? Date.distantFuture }())

        ScrollView { VStack(alignment: .leading, spacing: 20) {
            HStack { Rectangle().fill(lyGold).frame(width: 5, height: 25); Text(cardData["question"] as? String ?? "投票").font(.title2).fontWeight(.black).foregroundColor(lyBlue) }.padding(.horizontal, 20).padding(.top, 20); if let content = cardData["content"] as? String, !content.isEmpty { Text(content).foregroundColor(.gray).padding(.horizontal, 20) }
            VStack(spacing: 12) { ForEach(options, id: \.self) { opt in let optVotes = votes[opt] ?? []; let hasVoted = optVotes.contains(currentUid); let percentage = totalVotes > 0 ? Int((Double(optVotes.count) / Double(totalVotes)) * 100) : 0; GeometryReader { geo in ZStack(alignment: .leading) { RoundedRectangle(cornerRadius: 8).fill(hasVoted ? Color(red: 253/255, green: 247/255, blue: 213/255) : Color(UIColor.systemGray6)); if percentage > 0 { RoundedRectangle(cornerRadius: 8).fill(hasVoted ? lyGold.opacity(0.15) : Color.gray.opacity(0.1)).frame(width: geo.size.width * CGFloat(percentage) / 100) }; HStack { if hasVoted { Image(systemName: "checkmark.circle.fill").foregroundColor(lyGold) }; Text(opt).fontWeight(hasVoted ? .bold : .regular).foregroundColor(lyBlue); Spacer(); if !optVotes.isEmpty { HStack(spacing: -10) { ForEach(optVotes.prefix(3), id: \.self) { vUid in LiveAvatarView(uidOrName: vUid, showName: false, size: 24) }; if optVotes.count > 3 { Text("+\(optVotes.count - 3)").font(.system(size: 10, weight: .bold)).foregroundColor(.gray).frame(width: 24, height: 24).background(Color.white).clipShape(Circle()).overlay(Circle().stroke(Color.gray.opacity(0.3), lineWidth: 1)) } } }; Text("\(optVotes.count) 票 (\(percentage)%)").font(.subheadline).foregroundColor(.gray).fontWeight(.bold) }.padding() }.overlay(RoundedRectangle(cornerRadius: 8).stroke(hasVoted ? lyGold : Color.gray.opacity(0.2), lineWidth: 1)).contentShape(Rectangle()) .onTapGesture { if isExpired { print("投票已截止！") } else { viewModel.castVote(msgId: msgId, option: opt) } } }.frame(height: 50) }; Button(action: { showVotersModal = true }) { HStack { Image(systemName: "person.2.fill"); Text("查看投票成員") }.font(.headline).foregroundColor(.white).frame(maxWidth: .infinity).padding().background(Color(red: 25/255, green: 38/255, blue: 50/255)).cornerRadius(10).shadow(radius: 3) }.padding(.top, 10) }.padding(.horizontal, 20)
            VStack(alignment: .leading, spacing: 10) { infoRow(icon: "person.fill", title: "建立人", value: cardData["creatorName"] as? String ?? "未提供"); HStack { infoRow(icon: "hourglass", title: "結束時間", value: deadline.isEmpty ? "無期限" : deadline.replacingOccurrences(of: "T", with: " ")); Spacer(); if isExpired { Text("(已截止)").foregroundColor(.gray).fontWeight(.bold) } } }.padding().background(Color(red: 253/255, green: 252/255, blue: 245/255)).cornerRadius(12).padding(.horizontal, 20).padding(.top, 20)
        }.padding(.bottom, 40) }.background(Color(UIColor.systemGroupedBackground).ignoresSafeArea()).navigationTitle("投票詳情").navigationBarTitleDisplayMode(.inline).sheet(isPresented: $showVotersModal) { PollVotersView(options: options, votes: votes) }.onAppear { NotificationCenter.default.post(name: NSNotification.Name("HideTabBar"), object: nil) }
    }
    func infoRow(icon: String, title: String, value: String) -> some View { HStack(spacing: 8) { Image(systemName: icon).foregroundColor(lyGold).frame(width: 20); Text("\(title) :").fontWeight(.bold).foregroundColor(.gray); Text(value).foregroundColor(.gray) }.font(.subheadline) }
}

struct PollVotersView: View {
    var options: [String]; var votes: [String: [String]]; @Environment(\.presentationMode) var presentationMode
    var body: some View { NavigationView { List { ForEach(options, id: \.self) { opt in let uids = votes[opt] ?? []; Section(header: Text("\(opt) (\(uids.count)票)").font(.headline).foregroundColor(.primary)) { if uids.isEmpty { Text("尚無人投票").foregroundColor(.gray).font(.subheadline) } else { ForEach(uids, id: \.self) { uid in LiveUserRow(uid: uid) } } } } }.navigationTitle("投票成員名單").navigationBarTitleDisplayMode(.inline).toolbar { ToolbarItem(placement: .navigationBarTrailing) { Button("關閉") { presentationMode.wrappedValue.dismiss() } } } } }
}

struct GroupBuyDetailView: View {
    var cardData: [String: Any]; let lyBlue = Color(red: 0/255, green: 49/255, blue: 83/255); let lyGold = Color(red: 184/255, green: 134/255, blue: 11/255)
    var body: some View { VStack(spacing: 0) { VStack(alignment: .leading, spacing: 10) { HStack(spacing: 8) { Image(systemName: "person.fill.badge.plus").foregroundColor(lyGold); Text("代購成員 :").foregroundColor(.gray); Text(cardData["initiator"] as? String ?? "未提供").fontWeight(.bold).foregroundColor(lyBlue) }; HStack(spacing: 8) { Image(systemName: "clock.fill").foregroundColor(lyGold); Text("截止時間 :").foregroundColor(.gray); Text(cardData["deadline"] as? String ?? "未提供").fontWeight(.bold).foregroundColor(.red) } }.padding().frame(maxWidth: .infinity, alignment: .leading).background(Color(red: 253/255, green: 252/255, blue: 245/255)).border(Color.gray.opacity(0.2), width: 1); ScrollView { VStack(spacing: 20) { VStack(spacing: 15) { HStack(alignment: .top) { Image(systemName: "photo").resizable().padding().frame(width: 60, height: 60).foregroundColor(.gray).background(Color.white).cornerRadius(8).overlay(RoundedRectangle(cornerRadius: 8).stroke(Color.gray.opacity(0.3))); VStack(alignment: .leading, spacing: 5) { Text("此處將由 API 帶入").font(.headline).foregroundColor(lyBlue); Text("等待功能開發中...").font(.caption).padding(.horizontal, 8).padding(.vertical, 3).background(Color(UIColor.systemGray6)).cornerRadius(4) }; Spacer() } }.padding().background(Color.white).cornerRadius(12) }.padding() } }.background(Color(UIColor.systemGroupedBackground).ignoresSafeArea()).navigationTitle(cardData["title"] as? String ?? "團購詳情").navigationBarTitleDisplayMode(.inline).onAppear { NotificationCenter.default.post(name: NSNotification.Name("HideTabBar"), object: nil) } }
}

struct ConfidentialDetailView: View {
    var cardData: [String: Any]; @State private var inputPassword = ""; @State private var isUnlocked = false; @State private var showError = false; let lyBlue = Color(red: 0/255, green: 49/255, blue: 83/255)
    var body: some View { VStack(spacing: 25) { Image(systemName: isUnlocked ? "lock.open.fill" : "lock.fill").font(.system(size: 65)).foregroundColor(isUnlocked ? .green : .red).padding(.top, 40); Text(cardData["title"] as? String ?? "機密檔案").font(.title).fontWeight(.black).foregroundColor(lyBlue); if isUnlocked { ScrollView { Text(cardData["content"] as? String ?? "這是機密內容...").font(.body).padding(20).frame(maxWidth: .infinity, alignment: .leading).background(Color.white).cornerRadius(12).shadow(color: Color.black.opacity(0.05), radius: 5, x: 0, y: 2) } } else { Text("此為受保護的機密文件，請輸入存取密碼。").font(.subheadline).foregroundColor(.gray); SecureField("輸入密碼...", text: $inputPassword).padding().background(Color.white).cornerRadius(10).overlay(RoundedRectangle(cornerRadius: 10).stroke(showError ? Color.red : Color.gray.opacity(0.3), lineWidth: 1)).padding(.horizontal, 30); if showError { Text("密碼錯誤，請重新輸入！").foregroundColor(.red).font(.caption) }; Button(action: unlock) { Text("確認解鎖").fontWeight(.bold).foregroundColor(.white).frame(maxWidth: .infinity).padding(.vertical, 14).background(lyBlue).cornerRadius(10) }.padding(.horizontal, 30) }; Spacer() }.padding().background(Color(UIColor.systemGroupedBackground).ignoresSafeArea()).navigationTitle("機密文件").navigationBarTitleDisplayMode(.inline).onAppear { NotificationCenter.default.post(name: NSNotification.Name("HideTabBar"), object: nil) } }
    func unlock() { let correctPassword = cardData["password"] as? String ?? "1234"; if inputPassword == correctPassword { withAnimation(.spring()) { isUnlocked = true; showError = false } } else { withAnimation(.default) { showError = true; inputPassword = "" } } }
}
```
