```swift
import SwiftUI
import Combine
import FirebaseAuth
import FirebaseFirestore

// 👉 全域成員排序表
let calMemberOrderList = [
    "もも", "班長", "俊齊", "博峻", "佳偉", "書銘", "江旭", "尊玄", "俊諺",
    "成一", "宗航", "崇憲", "光庭", "威丞", "韋仁", "祐瑜", "至軒", "奕賢",
    "正德", "鈞儒", "凱瑞", "承遠", "皓鈞", "鎮遠", "智皓", "哲遠", "隆勳",
    "柏鑫", "承佑", "凱駿", "冠叡", "祐祥", "可愛柴"
]

func getCalMemberSortIndex(_ name: String) -> Int {
    return calMemberOrderList.firstIndex(of: name) ?? 999
}

// MARK: - 📅 資料模型
enum CalendarItemType {
    case event
    case plan
}

struct CalendarEvent: Identifiable {
    var id: String
    var title: String
    var start: Date
    var end: Date
    var isAllDay: Bool
    var color: String
    var type: CalendarItemType
    var creatorUid: String
    var creatorName: String
    var sharedWith: [String]
    var isEncrypted: Bool
    var isQuickTag: Bool
    var createdAt: Date?
    
    var planItems: [CalPlanSectionModel]?
    
    var startDateStr: String {
        let f = DateFormatter(); f.timeZone = TimeZone.current; f.dateFormat = "yyyy-MM-dd"; return f.string(from: start)
    }
    var endDateStr: String {
        let f = DateFormatter(); f.timeZone = TimeZone.current; f.dateFormat = "yyyy-MM-dd"; return f.string(from: end)
    }
}

struct DayModel: Identifiable, Hashable {
    let id = UUID(); let date: Date; let isToday: Bool; let isOtherMonth: Bool
    let dayString: String; let lunarString: String; let dateStr: String
}

enum CalPlanSectionType { case heading, content, warning, flight }
struct CalPlanSectionModel: Identifiable {
    let id = UUID()
    var type: CalPlanSectionType
    var text: String = ""
    var flightNo: String = ""
    var date: Date = Date()
    var time: Date = Date()
    var arrivalTime: Date = Date().addingTimeInterval(3600 * 3)
    var from: String = ""
    var to: String = ""
    var termFrom: String = ""
    var termTo: String = ""
    var gateFrom: String = ""
    var gateTo: String = ""
    var member: String = ""
    var buddy: String = ""
    var airline: String = ""
}

struct CalCachedProfile: Codable {
    let name: String
    let avatar: String
}

// 👉 專屬隔離版快取圖片元件，保證不再報錯
struct CalCachedAvatarView: View {
    let urlString: String
    let size: CGFloat
    var body: some View {
        AsyncImage(url: URL(string: urlString)) { phase in
            if let image = phase.image {
                image.resizable().scaledToFill()
            } else if phase.error != nil {
                Image(systemName: "person.circle.fill").resizable().foregroundColor(.gray)
            } else {
                ProgressView().scaleEffect(0.5)
            }
        }
        .frame(width: size, height: size)
        .background(Color.white)
        .clipShape(Circle())
    }
}

// 👉 精簡版計畫歷史卡片
struct PlanHistoryCompactCard: View {
    var publisher: String
    var dateStr: String
    var title: String
    var timeRange: String
    var body: some View {
        VStack(alignment: .leading, spacing: 10) {
            HStack { Label("發布人：\(publisher)", systemImage: "person.fill").font(.caption).foregroundColor(.gray); Spacer(); Label(dateStr, systemImage: "clock").font(.caption).foregroundColor(.gray) }
            Text(title).font(.title3).fontWeight(.bold).foregroundColor(safeParseHexColor("#003153")).padding(.leading, 10).overlay(Rectangle().frame(width: 4).foregroundColor(safeParseHexColor("#003153")), alignment: .leading)
            Label(timeRange, systemImage: "calendar").font(.caption).foregroundColor(.gray)
        }.padding(15).background(Color.white).cornerRadius(8).overlay(RoundedRectangle(cornerRadius: 8).stroke(Color.gray.opacity(0.2), lineWidth: 1))
    }
}

// 👉 時間格式擴充
extension DateFormatter {
    static let localISO: DateFormatter = {
        let f = DateFormatter()
        f.timeZone = TimeZone.current
        f.dateFormat = "yyyy-MM-dd'T'HH:mm"
        return f
    }()
    static let displayTimeRange: DateFormatter = {
        let f = DateFormatter()
        f.locale = Locale(identifier: "en_GB")
        f.dateFormat = "HH:mm"
        return f
    }()
    static let displayDateRange: DateFormatter = {
        let f = DateFormatter()
        f.locale = Locale(identifier: "zh_TW")
        f.dateFormat = "yyyy/MM/dd HH:mm"
        return f
    }()
}

// 👉 顏色與圓角擴充
func safeParseHexColor(_ hexString: String) -> Color {
    let hex = hexString.trimmingCharacters(in: CharacterSet.alphanumerics.inverted)
    var int: UInt64 = 0
    Scanner(string: hex).scanHexInt64(&int)
    let a, r, g, b: UInt64
    switch hex.count {
    case 3: (a, r, g, b) = (255, (int >> 8) * 17, (int >> 4 & 0xF) * 17, (int & 0xF) * 17)
    case 6: (a, r, g, b) = (255, int >> 16, int >> 8 & 0xFF, int & 0xFF)
    case 8: (a, r, g, b) = (int >> 24, int >> 16 & 0xFF, int >> 8 & 0xFF, int & 0xFF)
    default: (a, r, g, b) = (255, 0, 49, 83)
    }
    return Color(red: Double(r) / 255.0, green: Double(g) / 255.0, blue: Double(b) / 255.0, opacity: Double(a) / 255.0)
}

struct CalRoundedCorner: Shape {
    var radius: CGFloat = .infinity
    var corners: UIRectCorner = .allCorners
    func path(in rect: CGRect) -> Path { Path(UIBezierPath(roundedRect: rect, byRoundingCorners: corners, cornerRadii: CGSize(width: radius, height: radius)).cgPath) }
}
extension Date { func component(_ component: Calendar.Component) -> Int { return Calendar.current.component(component, from: self) } }

extension Color {
    func toSafeHex() -> String? {
        let uic = UIColor(self)
        guard let components = uic.cgColor.components, components.count >= 3 else { return nil }
        let r = Float(components[0]); let g = Float(components[1]); let b = Float(components[2]); var a = Float(1.0); if components.count >= 4 { a = Float(components[3]) }
        if a != Float(1.0) { return String(format: "%02lX%02lX%02lX%02lX", lroundf(r * 255), lroundf(g * 255), lroundf(b * 255), lroundf(a * 255)) } else { return String(format: "#%02lX%02lX%02lX", lroundf(r * 255), lroundf(g * 255), lroundf(b * 255)) }
    }
}
// MARK: - ⚙️ 行事曆核心引擎
class CalendarViewModel: ObservableObject {
    @Published var currentMonthDate: Date = Date()
    @Published var events: [CalendarEvent] = []
    @Published var isLoading = false
    @Published var eventSlots: [String: [Int: CalendarEvent]] = [:]
    @Published var planSlots: [String: [Int: CalendarEvent]] = [:]
    @Published var calMode: CalDisplayMode = .days
    @Published var selectorYear: Int = Calendar.current.component(.year, from: Date())
    
    let topPadding: CGFloat = 0.0
    let headerGap: CGFloat = 0.0
    let rowHeight: CGFloat = 150.4545497894287
    let bottomPadding: CGFloat = -100.0
    let isBottomFill: Bool = false
    
    @Published var isSharedMode = false
    @Published var isQuickTagMode = false
    @Published var selectedQuickDates: Set<String> = []
    @Published var selectedQuickTag: QuickTagOption = .off
    
    @Published var showDeleteTagModal: Bool = false
    @Published var tagsToDeleteList: [CalendarEvent] = []
    @Published var selectedTagsToDelete: Set<String> = []
    
    @Published var myName: String = "自己"
    @Published var isMonitor: Bool = false
    @Published var highlightTodayFlag: Bool = false
    
    @Published var userCache: [String: CalCachedProfile] = [:]
    @Published var allUsers: [(id: String, name: String, avatar: String)] = []
    
    enum CalDisplayMode { case days, months, years }
    struct QuickTagOption: Hashable {
        let title: String; let hexColor: String
        static let off = QuickTagOption(title: "休假 (紅)", hexColor: "#d93025")
        static let work = QuickTagOption(title: "上班 (藍)", hexColor: "#5ba4d6")
        static let trip = QuickTagOption(title: "出差 (金)", hexColor: "#cfa900")
    }
    
    private var db = Firestore.firestore()
    private let dateFormatter: DateFormatter = {
        let f = DateFormatter(); f.timeZone = TimeZone.current; f.dateFormat = "yyyy-MM-dd"; return f
    }()
    
    init() {
        loadUserCache()
        checkIdentityAndFetch()
    }
    
    func loadUserCache() {
        if let data = UserDefaults.standard.data(forKey: "UserCache_Global"),
           let decoded = try? JSONDecoder().decode([String: CalCachedProfile].self, from: data) {
            self.userCache = decoded
        }
    }
    
    func saveUserCache() {
        if let data = try? JSONEncoder().encode(userCache) {
            UserDefaults.standard.set(data, forKey: "UserCache_Global")
        }
    }
    
    func fetchAllUsers() {
        db.collection("act").getDocuments { [weak self] snap, _ in
            guard let docs = snap?.documents else { return }
            var users: [(id: String, name: String, avatar: String)] = []
            
            let hiddenEmails = ["shiba@ll.com", "meiju@ll.com", "momo@ll.com"]
            let currentEmail = Auth.auth().currentUser?.email ?? ""
            let canViewHidden = (self?.myName == "班長") || hiddenEmails.contains(currentEmail)
            
            for doc in docs {
                let data = doc.data()
                let email = data["email"] as? String ?? ""
                if !canViewHidden && hiddenEmails.contains(email) { continue }
                
                let id = doc.documentID
                let name = data["displayName"] as? String ?? "未知"
                let avatar = data["photoURL"] as? String ?? ""
                
                users.append((id: id, name: name, avatar: avatar))
                DispatchQueue.main.async {
                    self?.userCache[id] = CalCachedProfile(name: name, avatar: avatar)
                    self?.saveUserCache()
                }
            }
            users.sort { getCalMemberSortIndex($0.name) < getCalMemberSortIndex($1.name) }
            DispatchQueue.main.async { self?.allUsers = users }
        }
    }
    
    func fetchMissingUsers(for uids: [String], completion: @escaping () -> Void) {
        let needed = uids.filter { userCache[$0] == nil }
        if needed.isEmpty { completion(); return }
        
        let group = DispatchGroup()
        for uid in needed {
            group.enter()
            db.collection("act").document(uid).getDocument { [weak self] doc, _ in
                if let data = doc?.data() {
                    let name = data["displayName"] as? String ?? "未知"
                    let avatar = data["photoURL"] as? String ?? ""
                    DispatchQueue.main.async {
                        self?.userCache[uid] = CalCachedProfile(name: name, avatar: avatar)
                        self?.saveUserCache()
                    }
                }
                group.leave()
            }
        }
        group.notify(queue: .main) { completion() }
    }
    
    func triggerTodayHighlight() {
        withAnimation(.easeIn(duration: 0.1)) { self.highlightTodayFlag = true }
        DispatchQueue.main.asyncAfter(deadline: .now() + 0.5) {
            withAnimation(.easeOut(duration: 0.3)) { self.highlightTodayFlag = false }
        }
    }
    
    func checkIdentityAndFetch() {
        guard let uid = Auth.auth().currentUser?.uid else { return }
        db.collection("act").document(uid).getDocument { [weak self] doc, _ in
            if let data = doc?.data() {
                DispatchQueue.main.async {
                    self?.myName = data["displayName"] as? String ?? "未知"
                    self?.isMonitor = (self?.myName == "班長")
                    self?.fetchEvents()
                    self?.fetchAllUsers()
                }
            }
        }
    }
    
    func fetchEvents() {
        guard let uid = Auth.auth().currentUser?.uid else { return }
        isLoading = true
        let dispatchGroup = DispatchGroup()
        var tempEvents: [CalendarEvent] = []
        
        dispatchGroup.enter()
        db.collection("Calendar_page").document("calendar").collection(myName)
            .whereField("creatorUid", isEqualTo: uid).getDocuments { snap, _ in
                if let docs = snap?.documents { for doc in docs { if let ev = self.parseEvent(doc: doc, type: .event) { tempEvents.append(ev) } } }
                dispatchGroup.leave()
            }
        
        dispatchGroup.enter()
        db.collection("post").getDocuments { snap, _ in
            if let docs = snap?.documents {
                for doc in docs {
                    let data = doc.data()
                    let sharedWith = data["sharedWith"] as? [String] ?? []
                    if (data["creatorUid"] as? String ?? "") == uid || sharedWith.contains(uid) || self.isMonitor {
                        if let ev = self.parseEvent(doc: doc, type: .plan) { tempEvents.append(ev) }
                    }
                }
            }
            dispatchGroup.leave()
        }
        
        dispatchGroup.notify(queue: .main) {
            self.events = tempEvents
            self.computeSlots(for: tempEvents)
            self.isLoading = false
        }
    }
    
    private func parseEvent(doc: QueryDocumentSnapshot, type: CalendarItemType) -> CalendarEvent? {
        let data = doc.data()
        guard let startStr = data["start"] as? String, let endStr = data["end"] as? String else { return nil }
        let isAllDay = data["isAllDay"] as? Bool ?? false
        let isQuickTag = data["isQuickTag"] as? Bool ?? false
        
        let createdAt: Date
        if let ts = data["timestamp"] as? Timestamp { createdAt = ts.dateValue() }
        else if let ts = data["createdAt"] as? Timestamp { createdAt = ts.dateValue() }
        else { createdAt = Date() }
        
        var sDate = Date()
        var eDate = Date()
        
        if isAllDay || isQuickTag {
            let cleanStartStr = String(startStr.prefix(10))
            let cleanEndStr = String(endStr.prefix(10))
            let localFormatter = DateFormatter(); localFormatter.timeZone = TimeZone.current; localFormatter.dateFormat = "yyyy-MM-dd"
            if let parsedStart = localFormatter.date(from: cleanStartStr) { sDate = parsedStart }
            if let parsedEnd = localFormatter.date(from: cleanEndStr) { eDate = parsedEnd }
        } else {
            let formatter = ISO8601DateFormatter(); formatter.formatOptions = [.withInternetDateTime]
            sDate = formatter.date(from: startStr + (startStr.count == 16 ? ":00Z" : "Z")) ?? Date()
            eDate = formatter.date(from: endStr + (endStr.count == 16 ? ":00Z" : "Z")) ?? Date()
        }
        
        var planItems: [CalPlanSectionModel] = []
        if type == .plan {
            if let itemsData = data["items"] as? [[String: Any]] {
                for itemData in itemsData {
                    var sectionType = CalPlanSectionType.content
                    if let typeStr = itemData["type"] as? String {
                        switch typeStr {
                        case "heading": sectionType = .heading
                        case "warning": sectionType = .warning
                        case "flight": sectionType = .flight
                        default: sectionType = .content
                        }
                    }
                    var model = CalPlanSectionModel(type: sectionType)
                    model.text = itemData["text"] as? String ?? ""
                    if sectionType == .flight {
                        model.flightNo = itemData["flightNo"] as? String ?? ""
                        model.from = itemData["from"] as? String ?? ""
                        model.to = itemData["to"] as? String ?? ""
                        model.termFrom = itemData["termFrom"] as? String ?? ""
                        model.termTo = itemData["termTo"] as? String ?? ""
                        model.gateFrom = itemData["gateFrom"] as? String ?? ""
                        model.gateTo = itemData["gateTo"] as? String ?? ""
                        model.member = itemData["member"] as? String ?? ""
                        model.buddy = itemData["buddy"] as? String ?? ""
                        model.airline = itemData["airline"] as? String ?? ""
                        if let dateStr = itemData["date"] as? String, let parsedDate = ISO8601DateFormatter().date(from: dateStr) { model.date = parsedDate }
                        if let timeStr = itemData["time"] as? String, let parsedTime = DateFormatter.localISO.date(from: timeStr) { model.time = parsedTime }
                        if let arrTimeStr = itemData["arrivalTime"] as? String, let parsedArr = DateFormatter.localISO.date(from: arrTimeStr) { model.arrivalTime = parsedArr } else { model.arrivalTime = model.time.addingTimeInterval(3600 * 3) }
                    }
                    planItems.append(model)
                }
            }
        }
        return CalendarEvent(
            id: doc.documentID, title: data["title"] as? String ?? "未命名",
            start: sDate, end: eDate, isAllDay: isAllDay, color: data["color"] as? String ?? "#5ba4d6",
            type: type, creatorUid: data["creatorUid"] as? String ?? "", creatorName: data["creatorName"] as? String ?? "",
            sharedWith: data["sharedWith"] as? [String] ?? [], isEncrypted: data["isEncrypted"] as? Bool ?? false,
            isQuickTag: isQuickTag, createdAt: createdAt, planItems: planItems.isEmpty ? nil : planItems
        )
    }

    private func computeSlots(for events: [CalendarEvent]) {
        var eSlots: [String: [Int: CalendarEvent]] = [:]
        var pSlots: [String: [Int: CalendarEvent]] = [:]
        let sorted = events.sorted { a, b in if a.start != b.start { return a.start < b.start }; return (b.end.timeIntervalSince(b.start)) > (a.end.timeIntervalSince(a.start)) }
        for ev in sorted {
            let dates = getDates(from: ev.start, to: ev.end)
            if ev.type == .plan { let slot = findFreeSlot(dates: dates, slots: pSlots); for d in dates { pSlots[d, default: [:]][slot] = ev } }
            else { let slot = findFreeSlot(dates: dates, slots: eSlots); for d in dates { eSlots[d, default: [:]][slot] = ev } }
        }
        self.eventSlots = eSlots; self.planSlots = pSlots
    }
    
    private func getDates(from start: Date, to end: Date) -> [String] {
        var dates: [String] = []
        var current = Calendar.current.startOfDay(for: start)
        let endDate = Calendar.current.startOfDay(for: end)
        while current <= endDate {
            dates.append(dateFormatter.string(from: current))
            current = Calendar.current.date(byAdding: .day, value: 1, to: current) ?? current.addingTimeInterval(86400)
        }
        if Calendar.current.isDate(start, inSameDayAs: end) { return [dateFormatter.string(from: start)] }
        return dates
    }
    
    private func findFreeSlot(dates: [String], slots: [String: [Int: CalendarEvent]]) -> Int {
        var s = 0; while true { var ok = true; for d in dates { if slots[d]?[s] != nil { ok = false; break } }; if ok { return s }; s += 1 }
    }
    
    func generateDays(for date: Date) -> [DayModel] {
        var days: [DayModel] = []; let cal = Calendar.current; let comps = cal.dateComponents([.year, .month], from: date)
        guard let start = cal.date(from: comps) else { return [] }
        let weekday = cal.component(.weekday, from: start); let prevCount = weekday - 1
        if prevCount > 0 { for i in (1...prevCount).reversed() { let d = cal.date(byAdding: .day, value: -i, to: start)!; days.append(makeDay(d, true)) } }
        let range = cal.range(of: .day, in: .month, for: start)!
        for i in 0..<range.count { let d = cal.date(byAdding: .day, value: i, to: start)!; days.append(makeDay(d, false)) }
        let rows = Int(ceil(Double(days.count) / 7.0)); let target = rows * 7
        while days.count < target { let d = cal.date(byAdding: .day, value: 1, to: days.last!.date)!; days.append(makeDay(d, true)) }
        return days
    }
    
    private func makeDay(_ date: Date, _ other: Bool) -> DayModel {
        let cal = Calendar.current; let chinese = Calendar(identifier: .chinese)
        let day = chinese.component(.day, from: date); let lunarDays = ["初一","初二","初三","初四","初五","初六","初七","初八","初九","初十","十一","十二","十三","十四","十五","十六","十七","十八","十九","二十","廿一","廿二","廿三","廿四","廿五","廿六","廿七","廿八","廿九","三十"]
        return DayModel(date: date, isToday: cal.isDateInToday(date), isOtherMonth: other, dayString: "\(cal.component(.day, from: date))", lunarString: lunarDays[day-1], dateStr: dateFormatter.string(from: date))
    }
}

// MARK: - 🎨 主要畫面渲染
struct CalendarView: View {
    var onClose: () -> Void
    @StateObject private var vm = CalendarViewModel()
    @State private var monthOffsets: [Int] = Array(-240...240)
    @State private var currentOffset: Int = 0
    @State private var dragOffset: CGFloat = 0
    
    @State private var showTagAlert = false
    @State private var alertAction: (() -> Void)? = nil
    @State private var alertMessage = ""
    
    @State private var showTagDropdown = false
    @State private var customTitle = ""
    @State private var customColor = Color.purple
    @State private var showCustomPicker = false
    let tagOptions: [CalendarViewModel.QuickTagOption] = [.off, .work, .trip]
    
    // 👉 獨立視窗控制狀態
    @State private var showDaySchedule = false
    @State private var selectedDate: Date = Date()
    @State private var isShowEventView = false
    @State private var isShowPlanView = false
    @State private var isShowEditPlanView = false
    @State private var planToEdit: CalendarEvent? = nil
    @State private var selectedPlan: CalendarEvent? = nil
    
    let lyBlue = Color(red: 0.0/255.0, green: 49.0/255.0, blue: 83.0/255.0)
    let lyGold = Color(red: 184.0/255.0, green: 134.0/255.0, blue: 11.0/255.0)
    
    var body: some View {
        GeometryReader { fullGeo in
            // 👉 ZStack 原生堆疊，滑動時背景會跟著卡片一起移動，透出底下畫面！
            ZStack {
                Color.white.ignoresSafeArea(.all)
                mainLayout
                floatingModals
            }
        }
        .onReceive(NotificationCenter.default.publisher(for: NSNotification.Name("TriggerEditPlan"))) { notif in
            if let targetPlan = notif.object as? CalendarEvent {
                self.planToEdit = targetPlan
                withAnimation(.spring()) {
                    self.isShowEditPlanView = true
                }
            }
        }
        .onReceive(NotificationCenter.default.publisher(for: NSNotification.Name("PlanDidUpdate"))) { notif in
            if let updatedPlan = notif.object as? CalendarEvent {
                self.selectedPlan = updatedPlan
            }
        }
        .onReceive(NotificationCenter.default.publisher(for: NSNotification.Name("TriggerDetailPlan"))) { notif in
            if let targetPlan = notif.object as? CalendarEvent {
                withAnimation(.spring()) {
                    self.selectedPlan = targetPlan
                }
            }
        }
    }
    
    @ViewBuilder
    private var mainLayout: some View {
        ZStack {
            VStack(spacing: 0) {
                calendarHeaderArea
                calendarGridArea
            }
            .offset(x: dragOffset)
            // 👉 防滑保護：如果上方有浮動視窗，主層日曆就靜止不動，避免跟著被滑走！
            .gesture(DragGesture().onChanged { v in
                let isModalOpen = showDaySchedule || isShowEventView || isShowPlanView || selectedPlan != nil || isShowEditPlanView
                if !isModalOpen {
                    if v.startLocation.x < 40 && v.translation.width > 0 { dragOffset = v.translation.width }
                }
            }.onEnded { v in
                let isModalOpen = showDaySchedule || isShowEventView || isShowPlanView || selectedPlan != nil || isShowEditPlanView
                if !isModalOpen {
                    if v.startLocation.x < 40 {
                        if v.translation.width > 100 { onClose() } else { withAnimation(.spring()) { dragOffset = 0 } }
                    }
                }
            })
            
            alertModals
        }
    }

    @ViewBuilder
    private var calendarHeaderArea: some View {
        VStack(spacing: 0) {
            HStack {
                if vm.isQuickTagMode {
                    Button(action: { withAnimation(.spring()) { vm.isQuickTagMode = false; vm.selectedQuickDates.removeAll() } }) { Image(systemName: "xmark").font(.title3).foregroundColor(.primary).padding(.trailing, 10) }
                    HStack(spacing: 10) {
                        Button(action: { withAnimation { showTagDropdown.toggle() } }) {
                            HStack { Text(vm.selectedQuickTag.title).fontWeight(.bold).foregroundColor(safeParseHexColor(vm.selectedQuickTag.hexColor)).lineLimit(1); Spacer(); Image(systemName: showTagDropdown ? "chevron.up" : "chevron.down").foregroundColor(.gray) }
                            .padding(.horizontal, 10).padding(.vertical, 8).background(Color.white).cornerRadius(8).overlay(Rectangle().stroke(Color.gray.opacity(0.3), lineWidth: 1))
                        }.frame(maxWidth: 160)
                        .overlay(
                            Group {
                                if showTagDropdown {
                                    VStack(spacing: 0) {
                                        ForEach(tagOptions, id: \.title) { opt in Button(action: { vm.selectedQuickTag = opt; withAnimation { showTagDropdown = false } }) { HStack { Text(opt.title).foregroundColor(safeParseHexColor(opt.hexColor)); Spacer(); if vm.selectedQuickTag == opt { Image(systemName: "checkmark").foregroundColor(.blue) } }.padding(12).background(Color.white) }; Divider() }
                                        Button(action: { showCustomPicker = true; showTagDropdown = false }) { HStack { Text("自訂標籤...").foregroundColor(.purple); Spacer(); Image(systemName: "plus").foregroundColor(.purple) }.padding(12).background(Color.white) }
                                    }.background(Color.white).cornerRadius(8).shadow(color: Color.black.opacity(0.15), radius: 5, y: 5).overlay(RoundedRectangle(cornerRadius: 8).stroke(Color.gray.opacity(0.2), lineWidth: 1)).frame(width: 160).offset(y: 42)
                                }
                            }, alignment: .top
                        ).zIndex(100)
                        Spacer()
                        Button(action: { handleTagAdd() }) { Text("新增").font(.system(size: 14, weight: .bold)).foregroundColor(.white).padding(.horizontal, 12).padding(.vertical, 8).background(Color(red: 184/255, green: 134/255, blue: 11/255)).cornerRadius(8) }
                        Button(action: { handleTagDelete() }) { Text("刪除").font(.system(size: 14, weight: .bold)).foregroundColor(.white).padding(.horizontal, 12).padding(.vertical, 8).background(Color.red).cornerRadius(8) }
                    }.zIndex(100)
                } else {
                    Button(action: onClose) { Image(systemName: "chevron.left").font(.title3).foregroundColor(.primary).padding(.trailing, 10) }
                    Button(action: { withAnimation { vm.calMode = (vm.calMode == .days ? .months : .years) } }) { HStack(spacing: 4) { Text(headerTitle).font(.title3).fontWeight(.bold).foregroundColor(.primary); Image(systemName: "chevron.down").font(.caption).foregroundColor(.gray) } }
                    Spacer()
                    HStack(spacing: 16) {
                        Button(action: { withAnimation(.spring()) { currentOffset = 0 }; vm.triggerTodayHighlight() }) { Text("今天").font(.system(size: 14, weight: .black)).foregroundColor(lyBlue).fixedSize(horizontal: true, vertical: false) }
                        Button(action: { vm.isSharedMode.toggle() }) { Image(systemName: "person.2.fill").foregroundColor(vm.isSharedMode ? .red : lyBlue) }
                        Button(action: { }) { Image(systemName: "square.and.arrow.up").foregroundColor(lyBlue) }
                        Button(action: { withAnimation(.spring()) { vm.isQuickTagMode = true; vm.selectedQuickDates.removeAll() } }) { Image(systemName: "tag.fill").foregroundColor(lyBlue) }
                        Menu {
                            Button(action: { withAnimation(.spring()) { isShowEventView = true } }) { Label("新增一般行程", systemImage: "calendar.badge.plus") }
                            Button(action: { withAnimation(.spring()) { isShowPlanView = true } }) { Label("新增行政計畫", systemImage: "doc.text") }
                        } label: { Image(systemName: "plus").fontWeight(.bold).foregroundColor(lyGold).font(.title2) }
                    }
                }
            }
            .frame(height: 35).padding(.horizontal, 20).padding(.bottom, 8).padding(.top, vm.topPadding).zIndex(100)
            
            HStack(spacing: 0) {
                ForEach(["日","一","二","三","四","五","六"], id: \.self) { day in Text(day).font(.system(size: 12, weight: .bold)).foregroundColor(day == "日" ? .red : (day == "六" ? .blue : .gray)).frame(maxWidth: .infinity).padding(.vertical, 8) }
            }.padding(.top, vm.headerGap).overlay(Rectangle().frame(height: 1).foregroundColor(Color.gray.opacity(0.2)), alignment: .bottom)
        }.background(Color.white).zIndex(100)
    }

    @ViewBuilder
    private var calendarGridArea: some View {
        ZStack {
            GeometryReader { geo in
                if vm.calMode == .days {
                    TabView(selection: $currentOffset) {
                        ForEach(monthOffsets, id: \.self) { offset in
                            let date = Calendar.current.date(byAdding: .month, value: offset, to: Date())!
                            MonthGridView(date: date, vm: vm, availableHeight: geo.size.height) { d in self.selectedDate = d; withAnimation(.spring()) { self.showDaySchedule = true } }.tag(offset)
                        }
                    }.tabViewStyle(.page(indexDisplayMode: .never)).onChange(of: currentOffset) { _, newOffset in vm.currentMonthDate = Calendar.current.date(byAdding: .month, value: newOffset, to: Date())! }
                } else if vm.calMode == .months { MonthSelectorView(vm: vm, currentOffset: $currentOffset, lyBlue: lyBlue) }
                else { YearSelectorView(vm: vm, lyBlue: lyBlue) }
            }
            .padding(.bottom, vm.bottomPadding).ignoresSafeArea(.all, edges: vm.isBottomFill ? .bottom : [])
            if showTagDropdown { Color.clear.contentShape(Rectangle()).onTapGesture { withAnimation { showTagDropdown = false } }.zIndex(99) }
        }
    }
    
    // 👉 所有浮動視窗都經過原生 transition 處理，完全解除手勢限制！
    @ViewBuilder
    private var floatingModals: some View {
        if showDaySchedule {
            CalDayScheduleView(date: selectedDate, events: vm.events, lyBlue: lyBlue, lyGold: lyGold, topPadding: vm.topPadding) { withAnimation(.spring()) { showDaySchedule = false } }
                .transition(.move(edge: .trailing))
                .zIndex(200)
        }
        
        if isShowEventView {
            CalCreateEventView(vm: vm, isPresented: $isShowEventView)
                .transition(.move(edge: .trailing))
                .zIndex(300)
        }
        
        if isShowPlanView {
            CalCreatePlanView(vm: vm, isPresented: $isShowPlanView)
                .transition(.move(edge: .trailing))
                .zIndex(300)
        }
        
        if selectedPlan != nil {
            CalPlanDetailView(plan: selectedPlan!, vm: vm, isPresented: Binding(get: { selectedPlan != nil }, set: { if !$0 { selectedPlan = nil } }))
                .transition(.move(edge: .trailing))
                .zIndex(400)
        }
        
        if isShowEditPlanView, let editPlan = planToEdit {
            CalEditPlanView(plan: editPlan, vm: vm, isPresented: $isShowEditPlanView)
                .id("edit-\(editPlan.id)")
                .transition(.move(edge: .trailing))
                .zIndex(500)
        }
    }
    
    @ViewBuilder
    private var alertModals: some View {
        if showTagAlert {
            Color.black.opacity(0.4).ignoresSafeArea().zIndex(600)
            VStack(spacing: 20) {
                Text("提示").font(.headline); Text(alertMessage).font(.body)
                HStack {
                    if alertAction != nil { Button("取消") { withAnimation { showTagAlert = false; alertAction = nil } }.frame(maxWidth: .infinity).foregroundColor(.gray) }
                    Button("確定") { alertAction?(); withAnimation { showTagAlert = false; alertAction = nil } }.frame(maxWidth: .infinity).padding().foregroundColor(.white).background(lyGold).cornerRadius(8)
                }
            }.padding(20).background(Color.white).cornerRadius(15).padding(40).zIndex(601)
        }
        
        if vm.showDeleteTagModal {
            Color.black.opacity(0.4).ignoresSafeArea().zIndex(600)
            DeleteTagModalView(vm: vm).zIndex(601)
        }
        
        if showCustomPicker {
            Color.black.opacity(0.4).ignoresSafeArea().zIndex(600)
            VStack(spacing: 15) {
                Text("自訂快速標籤").font(.headline)
                TextField("輸入標籤名稱", text: $customTitle).textFieldStyle(RoundedBorderTextFieldStyle())
                ColorPicker("選擇標籤顏色", selection: $customColor)
                HStack {
                    Button("取消") { withAnimation { showCustomPicker = false; showTagDropdown = false } }.frame(maxWidth: .infinity).foregroundColor(.gray)
                    Button("確定") {
                        if !customTitle.isEmpty { vm.selectedQuickTag = CalendarViewModel.QuickTagOption(title: "\(customTitle) (自訂)", hexColor: "#8e44ad") }
                        withAnimation { showCustomPicker = false; showTagDropdown = false }
                    }.frame(maxWidth: .infinity).padding(8).background(Color.blue).foregroundColor(.white).cornerRadius(8)
                }
            }.padding(20).background(Color.white).cornerRadius(15).padding(40).zIndex(601)
        }
    }
    
    var headerTitle: String {
        if vm.calMode == .days { return "\(Calendar.current.component(.year, from: vm.currentMonthDate))年 \(Calendar.current.component(.month, from: vm.currentMonthDate))月" }
        else if vm.calMode == .months { return String(format: "%d年", vm.selectorYear) }
        else { return "選擇年份" }
    }
    
    private func handleTagAdd() {
        if vm.selectedQuickDates.isEmpty { alertMessage = "請先點擊月曆選取日期！"; showTagAlert = true; return }
        alertMessage = "確定要將選取的 \(vm.selectedQuickDates.count) 天加入「\(vm.selectedQuickTag.title)」標籤嗎？"
        alertAction = {
            guard let uid = Auth.auth().currentUser?.uid else { return }
            let db = Firestore.firestore(); let batch = db.batch()
            for dateStr in vm.selectedQuickDates {
                let docRef = db.collection("Calendar_page").document("calendar").collection(vm.myName).document()
                let timeStr = "\(dateStr)T00:00"
                batch.setData(["title": vm.selectedQuickTag.title, "color": vm.selectedQuickTag.hexColor, "isAllDay": false, "isQuickTag": true, "start": timeStr, "end": timeStr, "creatorUid": uid, "creatorName": vm.myName, "isEncrypted": false], forDocument: docRef)
            }
            batch.commit { _ in vm.fetchEvents(); withAnimation { vm.isQuickTagMode = false; vm.selectedQuickDates.removeAll() } }
        }
        showTagAlert = true
    }
    
    private func handleTagDelete() {
        if vm.selectedQuickDates.isEmpty { alertMessage = "請先選取要刪除標籤的日期！"; showTagAlert = true; return }
        guard let uid = Auth.auth().currentUser?.uid else { return }
        let allTargetEvents = vm.events.filter { ev in ev.creatorUid == uid && vm.selectedQuickDates.contains(ev.startDateStr) && (ev.isQuickTag || ev.isAllDay) }
        if allTargetEvents.isEmpty { alertMessage = "您選取的日期上沒有可刪除的「快速標籤」。"; alertAction = nil; showTagAlert = true; return; }
        if allTargetEvents.count == 1 {
            alertMessage = "確定要刪除這筆「\(allTargetEvents[0].title)」標籤嗎？"
            alertAction = { Firestore.firestore().collection("Calendar_page").document("calendar").collection(vm.myName).document(allTargetEvents[0].id).delete { _ in vm.fetchEvents(); withAnimation { vm.isQuickTagMode = false; vm.selectedQuickDates.removeAll() } } }
            showTagAlert = true; return;
        }
        vm.tagsToDeleteList = allTargetEvents.sorted { $0.startDateStr < $1.startDateStr }
        vm.selectedTagsToDelete = Set(vm.tagsToDeleteList.map { $0.id })
        withAnimation { vm.showDeleteTagModal = true }
    }
}
// MARK: - 🗑️ 精細刪除清單視圖 (Delete Modal)
struct DeleteTagModalView: View {
    @ObservedObject var vm: CalendarViewModel
    private func formatMonthDay(_ dateStr: String) -> String { let parts = dateStr.split(separator: "-"); if parts.count == 3 { return "\(parts[1])/\(parts[2])" }; return dateStr }
    var body: some View {
        VStack(spacing: 0) {
            HStack { Text("選擇要刪除的標籤").font(.headline).foregroundColor(.black); Spacer(); Button(action: { withAnimation { vm.showDeleteTagModal = false } }) { Image(systemName: "xmark").foregroundColor(.gray).padding(5) } }.padding(20)
            HStack {
                let isAllSelected = vm.selectedTagsToDelete.count == vm.tagsToDeleteList.count
                Button(action: { if isAllSelected { vm.selectedTagsToDelete.removeAll() } else { vm.selectedTagsToDelete = Set(vm.tagsToDeleteList.map { $0.id }) } }) {
                    HStack { Image(systemName: isAllSelected ? "checkmark.square.fill" : "square").foregroundColor(isAllSelected ? .blue : .gray); Text(isAllSelected ? "取消全選" : "全選").font(.subheadline).foregroundColor(.gray) }
                }
                Spacer(); Text("共 \(vm.tagsToDeleteList.count) 筆").font(.caption).foregroundColor(.gray)
            }.padding(.horizontal, 20).padding(.bottom, 10)
            ScrollView {
                VStack(spacing: 12) {
                    ForEach(vm.tagsToDeleteList) { ev in
                        let isSelected = vm.selectedTagsToDelete.contains(ev.id)
                        Button(action: { if isSelected { vm.selectedTagsToDelete.remove(ev.id) } else { vm.selectedTagsToDelete.insert(ev.id) } }) {
                            HStack(spacing: 12) {
                                Image(systemName: isSelected ? "checkmark.circle.fill" : "circle").foregroundColor(isSelected ? .red : .gray).font(.title3)
                                Text(formatMonthDay(ev.startDateStr)).font(.system(size: 14, weight: .bold)).foregroundColor(.gray).frame(width: 50, alignment: .leading)
                                Text(ev.title).font(.system(size: 15, weight: .bold)).foregroundColor(.white).padding(.horizontal, 10).padding(.vertical, 6).background(safeParseHexColor(ev.color)).cornerRadius(6)
                                Spacer()
                            }
                            .padding(12).background(Color.gray.opacity(0.05)).cornerRadius(8)
                            .overlay(RoundedRectangle(cornerRadius: 8).stroke(isSelected ? Color.red.opacity(0.5) : Color.clear, lineWidth: 2))
                        }
                    }
                }
                .padding(.horizontal, 20).padding(.bottom, 20)
            }.frame(maxHeight: 300)
            HStack {
                Button("取消") { withAnimation { vm.showDeleteTagModal = false } }.frame(maxWidth: .infinity).padding(12).background(Color.gray.opacity(0.1)).foregroundColor(.black).cornerRadius(8)
                Button("刪除選定 (\(vm.selectedTagsToDelete.count))") { executeDeleteList() }.frame(maxWidth: .infinity).padding(12).background(vm.selectedTagsToDelete.isEmpty ? Color.gray : Color.red).foregroundColor(.white).cornerRadius(8).disabled(vm.selectedTagsToDelete.isEmpty)
            }.padding(20)
        }.background(Color.white).cornerRadius(16).padding(30)
    }
    private func executeDeleteList() {
        if vm.selectedTagsToDelete.isEmpty { return }
        let db = Firestore.firestore(); let batch = db.batch()
        for evId in vm.selectedTagsToDelete { let docRef = db.collection("Calendar_page").document("calendar").collection(vm.myName).document(evId); batch.deleteDocument(docRef) }
        batch.commit { _ in vm.fetchEvents(); withAnimation { vm.showDeleteTagModal = false; vm.isQuickTagMode = false; vm.selectedQuickDates.removeAll() } }
    }
}

// MARK: - 🗓️ 年月選擇器
struct MonthSelectorView: View {
    @ObservedObject var vm: CalendarViewModel; @Binding var currentOffset: Int; let lyBlue: Color
    var body: some View {
        ScrollView {
            LazyVGrid(columns: Array(repeating: GridItem(.flexible(), spacing: 15), count: 4), spacing: 20) {
                ForEach(1...12, id: \.self) { month in
                    let isCurrent = (month == Calendar.current.component(.month, from: vm.currentMonthDate) && vm.selectorYear == Calendar.current.component(.year, from: vm.currentMonthDate))
                    Button(action: {
                        let today = Date(); let todayYear = Calendar.current.component(.year, from: today); let todayMonth = Calendar.current.component(.month, from: today)
                        let targetOffset = ((vm.selectorYear - todayYear) * 12) + (month - todayMonth)
                        withAnimation { currentOffset = targetOffset; vm.calMode = .days }
                    }) {
                        Text("\(month)月").font(.system(size: 16, weight: .bold)).foregroundColor(isCurrent ? .white : .primary).frame(width: 65, height: 65).background(isCurrent ? lyBlue : Color.clear).clipShape(Circle())
                    }
                }
            }.padding(30)
        }.background(Color.white)
    }
}

struct YearSelectorView: View {
    @ObservedObject var vm: CalendarViewModel; let lyBlue: Color; let years = Array(1970...2050)
    var body: some View {
        ScrollViewReader { proxy in
            ScrollView {
                LazyVGrid(columns: Array(repeating: GridItem(.flexible(), spacing: 15), count: 4), spacing: 20) {
                    ForEach(years, id: \.self) { year in
                        let isCurrent = (year == vm.selectorYear)
                        Button(action: { vm.selectorYear = year; withAnimation { vm.calMode = .months } }) { Text(String(format: "%d", year)).font(.system(size: 16, weight: .bold)).foregroundColor(isCurrent ? .white : .primary).frame(width: 65, height: 65).background(isCurrent ? lyBlue : Color.clear).clipShape(Circle()) }.id(year)
                    }
                }.padding(30)
            }.background(Color.white).onAppear { proxy.scrollTo(vm.selectorYear, anchor: .center) }
        }
    }
}

// MARK: - 🗓️ 網格視圖
struct MonthGridView: View {
    let date: Date; @ObservedObject var vm: CalendarViewModel; let availableHeight: CGFloat; var onDayClick: (Date) -> Void
    var body: some View {
        let days = vm.generateDays(for: date); let rows = days.count / 7; let exactRowHeight = vm.rowHeight
        VStack(spacing: 0) {
            ForEach(0..<rows, id: \.self) { r in
                HStack(spacing: 0) {
                    ForEach(0..<7, id: \.self) { c in
                        let index = r * 7 + c
                        if index < days.count {
                            let d = days[index]
                            let isSelected = vm.selectedQuickDates.contains(d.dateStr)
                            let isHighlightedToday = d.isToday && vm.highlightTodayFlag
                            DayCellView(day: d, vm: vm, isSelected: isSelected)
                                .frame(height: exactRowHeight).frame(maxWidth: .infinity).zIndex(isHighlightedToday ? 100 : (isSelected ? 10 : 0))
                                .onTapGesture {
                                    UIApplication.shared.sendAction(#selector(UIResponder.resignFirstResponder), to: nil, from: nil, for: nil)
                                    if vm.isQuickTagMode && !d.isOtherMonth {
                                        withAnimation(.spring(response: 0.3, dampingFraction: 0.6)) { if vm.selectedQuickDates.contains(d.dateStr) { vm.selectedQuickDates.remove(d.dateStr) } else { vm.selectedQuickDates.insert(d.dateStr) } }
                                    } else { onDayClick(d.date) }
                                }
                        }
                    }
                }.zIndex(Double(rows - r))
            }
        }.background(Color.white).frame(maxHeight: .infinity, alignment: .top)
    }
}

// MARK: - 🧊 單個日期格子
struct DayCellView: View {
    let day: DayModel; @ObservedObject var vm: CalendarViewModel; let isSelected: Bool
    let lyGold = Color(red: 184.0/255.0, green: 134.0/255.0, blue: 11.0/255.0)
    var body: some View {
        let isHighlightedToday = day.isToday && vm.highlightTodayFlag
        ZStack(alignment: .top) {
            if isHighlightedToday { Color.yellow.opacity(0.4) } else if day.isToday { lyGold.opacity(0.05) } else if day.isOtherMonth { Color.gray.opacity(0.05) } else { Color.white }
            if let pSlots = vm.planSlots[day.dateStr] {
                VStack(spacing: 2) {
                    Spacer().frame(height: 25)
                    ForEach(0..<3, id: \.self) { i in
                        if let plan = pSlots[i] {
                            let isStart = plan.startDateStr == day.dateStr; let isEnd = plan.endDateStr == day.dateStr
                            Rectangle().fill(safeParseHexColor(plan.color)).frame(height: 5).clipShape(CalRoundedCorner(radius: 2.5, corners: cornerMask(isStart: isStart, isEnd: isEnd))).padding(.leading, isStart ? 4 : 0).padding(.trailing, isEnd ? 4 : 0).opacity(day.isOtherMonth ? 0.4 : 0.85)
                        } else { Color.clear.frame(height: 5) }
                    }
                }
            }
            VStack(spacing: 0) {
                VStack(spacing: 0) { Text(day.dayString).font(.system(size: 14, weight: day.isToday ? .heavy : .medium)).foregroundColor(day.isToday ? .white : (day.isOtherMonth ? .gray.opacity(0.4) : .primary)).background(Circle().fill(day.isToday ? lyGold : Color.clear).frame(width: 24, height: 24)).padding(.top, 8); Text(day.lunarString).font(.system(size: 9)).foregroundColor(day.isToday ? lyGold : .gray.opacity(0.8)).padding(.top, 2) }
                .frame(height: 48).padding(.bottom, 6)
                let eSlots = vm.eventSlots[day.dateStr] ?? [:]
                VStack(spacing: 2) {
                    ForEach(0..<3, id: \.self) { i in
                        if let ev = eSlots[i] {
                            let isStart = ev.startDateStr == day.dateStr; let isEnd = ev.endDateStr == day.dateStr; let showText = isStart || day.date.component(.weekday) == 1
                            Text(showText ? (ev.isEncrypted ? "加密" : ev.title) : " ")
                                .font(.system(size: 9, weight: .bold)).lineLimit(1).foregroundColor(.white).frame(height: 14).frame(maxWidth: .infinity, alignment: .leading).padding(.leading, showText ? 4 : 0).background(safeParseHexColor(ev.color)).clipShape(CalRoundedCorner(radius: 3, corners: cornerMask(isStart: isStart, isEnd: isEnd))).padding(.leading, isStart ? 2 : 0).padding(.trailing, isEnd ? 2 : 0).opacity(day.isOtherMonth ? 0.5 : 1.0)
                        } else { Color.clear.frame(height: 14) }
                    }
                    if eSlots.keys.count > 3 { Text("...").font(.system(size: 8, weight: .black)).foregroundColor(.gray).padding(.top, -2) }
                }
                Spacer(minLength: 0)
            }
            if vm.highlightTodayFlag && !day.isToday { Color.black.opacity(0.3) }
        }
        .overlay(Rectangle().stroke(Color.gray.opacity(0.15), lineWidth: 0.5))
        .overlay(Rectangle().stroke(isSelected ? lyGold : (vm.isQuickTagMode && !day.isOtherMonth ? Color.gray.opacity(0.5) : Color.clear), style: StrokeStyle(lineWidth: isSelected ? 2 : 1, dash: isSelected ? [] : [4])))
        .scaleEffect(isHighlightedToday ? 1.08 : (isSelected ? 1.05 : 1.0))
        .shadow(color: isHighlightedToday ? Color.yellow.opacity(0.5) : (isSelected ? Color.black.opacity(0.12) : .clear), radius: isHighlightedToday ? 8 : 5, x: 0, y: isHighlightedToday ? 0 : 3)
        .contentShape(Rectangle()).animation(.easeInOut(duration: 0.2), value: vm.highlightTodayFlag)
    }
    private func cornerMask(isStart: Bool, isEnd: Bool) -> UIRectCorner {
        if isStart && isEnd { return .allCorners }
        if isStart { return [.topLeft, .bottomLeft] }
        if isEnd { return [.topRight, .bottomRight] }
        return []
    }
}

// 👉 完美補回的單日行程視窗 (背景包裹於 offset 內，搭載 highPriorityGesture)
struct CalDayScheduleView: View {
    let date: Date
    let events: [CalendarEvent]
    let lyBlue: Color
    let lyGold: Color
    let topPadding: CGFloat
    var onClose: () -> Void
    
    @State private var dragOffset: CGFloat = 0
    
    private var todaysEvents: [CalendarEvent] {
        let startOfDay = Calendar.current.startOfDay(for: date)
        let endOfDay = Calendar.current.date(byAdding: .day, value: 1, to: startOfDay)!
        return events.filter { $0.start < endOfDay && $0.end >= startOfDay }.sorted { $0.start < $1.start }
    }
    
    var body: some View {
        VStack(spacing: 0) {
            HStack {
                Button(action: onClose) { Image(systemName: "chevron.left").font(.title3).foregroundColor(.primary) }
                VStack(alignment: .leading) { Text("行程表").font(.headline); Text(date, style: .date).font(.caption).foregroundColor(.gray) }
                Spacer()
                Button(action: {}) { Image(systemName: "plus").foregroundColor(lyGold) }
            }.padding(.horizontal, 20).padding(.bottom, 12).padding(.top, 15).background(Color.white.ignoresSafeArea(edges: .top)).shadow(color: Color.black.opacity(0.05), radius: 2, y: 2)
            
            ScrollView {
                if todaysEvents.isEmpty { Text("本日無行程").foregroundColor(.gray).padding(.top, 100) }
                else { ForEach(todaysEvents) { ev in Text(ev.title).padding().frame(maxWidth: .infinity).background(Color.white).cornerRadius(10).shadow(radius: 2).padding(.horizontal) } }
            }.background(Color(UIColor.systemGroupedBackground))
        }
        .frame(maxWidth: .infinity, maxHeight: .infinity)
        .background(Color(UIColor.systemGroupedBackground).ignoresSafeArea())
        .shadow(color: Color.black.opacity(0.15), radius: 8, x: -3, y: 0)
        .offset(x: dragOffset)
        .highPriorityGesture(
            DragGesture().onChanged { v in
                if v.startLocation.x < 40 && v.translation.width > 0 { dragOffset = v.translation.width }
            }.onEnded { v in
                if v.startLocation.x < 40 {
                    let pred = v.translation.width + v.predictedEndTranslation.width
                    if v.translation.width > 100 || pred > 150 {
                        withAnimation(.easeOut(duration: 0.25)) { dragOffset = UIScreen.main.bounds.width }
                        DispatchQueue.main.asyncAfter(deadline: .now() + 0.25) { onClose(); dragOffset = 0 }
                    } else {
                        withAnimation(.spring()) { dragOffset = 0 }
                    }
                }
            }
        )
    }
}
// MARK: - 📝 建立/編輯 一般行程
struct CalCreateEventView: View {
    @ObservedObject var vm: CalendarViewModel
    @Binding var isPresented: Bool
    
    @State private var dragOffset: CGFloat = 0
    @State private var title: String = ""
    @State private var isAllDay: Bool = false
    @State private var startDate: Date = Date()
    @State private var endDate: Date = Date().addingTimeInterval(3600)
    @State private var selectedColor: String = "#5ba4d6"
    @State private var notifyOption: String = "none"
    @State private var location: String = ""
    @State private var note: String = ""
    @State private var url: String = ""
    @State private var isEncrypted: Bool = false
    
    let colorPresets = ["#dc3838", "#dab511", "#5ba4d6", "#003153", "#44ad62", "#8e44ad"]
    let notifyOptions = [ ("none", "無"), ("0", "開始時"), ("5", "5 分鐘前"), ("30", "30 分鐘前"), ("1440", "1 天前") ]

    var body: some View {
        VStack(spacing: 0) {
            // 👉 完美貼齊狀態列的標題區塊
            HStack {
                Button(action: { withAnimation(.spring()) { isPresented = false } }) { Image(systemName: "chevron.left").font(.title3).foregroundColor(safeParseHexColor("#003153")) }
                Text("編輯一般行程").font(.headline).foregroundColor(safeParseHexColor("#003153")).padding(.leading, 10)
                Spacer()
                Button("儲存") { saveEvent() }.font(.headline).foregroundColor(safeParseHexColor("#cfa900"))
            }
            .padding(.horizontal, 20).padding(.bottom, 12).padding(.top, 15)
            .background(Color.white.ignoresSafeArea(edges: .top))
            .shadow(color: Color.black.opacity(0.05), radius: 2, y: 2)
            
            Form {
                Section(header: Text("基本資訊")) {
                    TextField("標題 (可選填)", text: $title)
                    Toggle("全天行程", isOn: $isAllDay)
                    if isAllDay {
                        DatePicker("開始日期", selection: $startDate, displayedComponents: .date).environment(\.locale, Locale(identifier: "zh_TW"))
                        DatePicker("結束日期", selection: $endDate, displayedComponents: .date).environment(\.locale, Locale(identifier: "zh_TW"))
                    } else {
                        DatePicker("開始時間", selection: $startDate, displayedComponents: [.date, .hourAndMinute]).environment(\.locale, Locale(identifier: "zh_TW"))
                        DatePicker("結束時間", selection: $endDate, displayedComponents: [.date, .hourAndMinute]).environment(\.locale, Locale(identifier: "zh_TW"))
                    }
                }
                Section(header: Text("外觀與設定")) {
                    VStack(alignment: .leading, spacing: 10) {
                        ColorPicker("自訂標籤顏色", selection: Binding( get: { safeParseHexColor(selectedColor) }, set: { selectedColor = $0.toSafeHex() ?? "#5ba4d6" } ))
                        ScrollView(.horizontal, showsIndicators: false) {
                            HStack(spacing: 12) {
                                ForEach(colorPresets, id: \.self) { hex in Circle().fill(safeParseHexColor(hex)).frame(width: 30, height: 30).overlay(Circle().stroke(Color.gray.opacity(0.3), lineWidth: 1)).overlay(Image(systemName: "checkmark").foregroundColor(.white).opacity(selectedColor == hex ? 1 : 0)).onTapGesture { selectedColor = hex } }
                            }
                        }
                    }
                    Picker("通知提醒", selection: $notifyOption) { ForEach(notifyOptions, id: \.0) { option in Text(option.1).tag(option.0) } }
                }
                Section(header: Text("詳細資訊 (選填)")) {
                    TextField("地點", text: $location)
                    TextField("網址 (URL)", text: $url).keyboardType(.URL).autocapitalization(.none)
                    ZStack(alignment: .topLeading) { if note.isEmpty { Text("備註...").foregroundColor(Color(UIColor.placeholderText)).padding(.top, 8) }; TextEditor(text: $note).frame(minHeight: 100) }
                }
                Section(header: Text("安全設定")) {
                    Toggle(isOn: $isEncrypted) { Label("加密行程", systemImage: "lock.fill").foregroundColor(.red) }
                    if isEncrypted { SecureField("設定密碼...", text: .constant("")); SecureField("確認密碼...", text: .constant("")); Text("※ 密碼將受 +3 偏移保護").font(.caption).foregroundColor(.red) }
                }
            }
        }
        .frame(maxWidth: .infinity, maxHeight: .infinity)
        .background(Color(UIColor.systemGroupedBackground).ignoresSafeArea())
        .shadow(color: Color.black.opacity(0.15), radius: 8, x: -3, y: 0)
        .offset(x: dragOffset)
        .highPriorityGesture(
            DragGesture().onChanged { v in
                if v.startLocation.x < 40 && v.translation.width > 0 { dragOffset = v.translation.width }
            }.onEnded { v in
                if v.startLocation.x < 40 {
                    let pred = v.translation.width + v.predictedEndTranslation.width
                    if v.translation.width > 100 || pred > 150 {
                        withAnimation(.easeOut(duration: 0.25)) { dragOffset = UIScreen.main.bounds.width }
                        DispatchQueue.main.asyncAfter(deadline: .now() + 0.25) {
                            isPresented = false
                            dragOffset = 0
                        }
                    } else {
                        withAnimation(.spring()) { dragOffset = 0 }
                    }
                }
            }
        )
    }
    private func saveEvent() { print("儲存行程: \(title)"); withAnimation(.spring()) { isPresented = false } }
}

// MARK: - 📝 建立/發布 行政計畫
struct CalCreatePlanView: View {
    @ObservedObject var vm: CalendarViewModel
    @Binding var isPresented: Bool
    @State private var dragOffset: CGFloat = 0
    @State private var isFormExpanded: Bool = false
    @State private var mainTitle: String = ""
    @State private var startDate: Date = Date()
    @State private var endDate: Date = Date().addingTimeInterval(86400)
    @State private var selectedColor: String = "#003153"
    @State private var sections: [CalPlanSectionModel] = []
    @State private var showMemberPicker: Bool = false
    @State private var authorizedUIDs: [String] = []
    
    var filteredHistoryPlans: [CalendarEvent] {
        vm.events.filter { ev in
            ev.type == .plan && (ev.creatorUid == Auth.auth().currentUser?.uid || ev.sharedWith.contains(Auth.auth().currentUser?.uid ?? "") || vm.myName == "班長")
        }.sorted { $0.start > $1.start }
    }
    
    var body: some View {
        VStack(spacing: 0) {
            // 👉 完美貼齊狀態列的標題區塊
            HStack { Button(action: { withAnimation(.spring()) { isPresented = false } }) { Image(systemName: "chevron.left").font(.title3).foregroundColor(safeParseHexColor("#003153")) }; Text("行政計畫管理").font(.headline).foregroundColor(safeParseHexColor("#003153")).padding(.leading, 10); Spacer() }
            .padding(.horizontal, 20).padding(.bottom, 12).padding(.top, 15)
            .background(Color.white.ignoresSafeArea(edges: .top)).shadow(color: Color.black.opacity(0.05), radius: 2, y: 2)
            
            // 👉 移除 List 的 editMode 環境變數，這樣系統預設的右側把手就不會出現，也解決重複圖示的問題
            List {
                Section {
                    Button(action: { withAnimation(.easeInOut) { isFormExpanded.toggle() } }) { HStack { VStack(alignment: .leading, spacing: 4) { Text("發布新計畫").font(.headline).foregroundColor(safeParseHexColor("#003153")); Text("點擊展開以建立新的內容").font(.caption).foregroundColor(.gray) }; Spacer(); Image(systemName: isFormExpanded ? "chevron.up" : "chevron.down").foregroundColor(.gray) }.padding(.vertical, 8) }.listRowBackground(Color.white)
                }
                
                if isFormExpanded {
                    Section {
                        TextField("計畫主標題", text: $mainTitle).padding(.vertical, 4)
                        VStack(alignment: .leading, spacing: 12) {
                            VStack(alignment: .leading, spacing: 4) { Text("開始時間 *").font(.caption).foregroundColor(.gray); HStack { DatePicker("", selection: $startDate, displayedComponents: .date).labelsHidden().environment(\.locale, Locale(identifier: "zh_TW")); Spacer(); DatePicker("", selection: $startDate, displayedComponents: .hourAndMinute).labelsHidden().environment(\.locale, Locale(identifier: "en_GB")).datePickerStyle(.compact) } }
                            VStack(alignment: .leading, spacing: 4) { Text("結束時間 *").font(.caption).foregroundColor(.gray); HStack { DatePicker("", selection: $endDate, displayedComponents: .date).labelsHidden().environment(\.locale, Locale(identifier: "zh_TW")); Spacer(); DatePicker("", selection: $endDate, displayedComponents: .hourAndMinute).labelsHidden().environment(\.locale, Locale(identifier: "en_GB")).datePickerStyle(.compact) } }
                        }.padding(.vertical, 4)
                        VStack(alignment: .leading, spacing: 8) {
                            Text("計畫標籤顏色").font(.subheadline).foregroundColor(safeParseHexColor("#003153"))
                            HStack { ColorPicker("", selection: Binding(get: { safeParseHexColor(selectedColor) }, set: { selectedColor = $0.toSafeHex() ?? "#003153" })).labelsHidden().frame(width: 40); ScrollView(.horizontal, showsIndicators: false) { HStack(spacing: 12) { ForEach(["#003153", "#dc3838", "#dab511", "#5ba4d6", "#44ad62", "#8e44ad"], id: \.self) { hex in Circle().fill(safeParseHexColor(hex)).frame(width: 30, height: 30).overlay(Circle().stroke(Color.gray.opacity(0.3), lineWidth: 1)).overlay(Image(systemName: "checkmark").foregroundColor(.white).opacity(selectedColor == hex ? 1 : 0)).onTapGesture { selectedColor = hex } } } } }
                        }.padding(.vertical, 4)
                        VStack(alignment: .leading, spacing: 8) {
                            Text("授權共享對象 (預設僅自己可見)").font(.subheadline).foregroundColor(safeParseHexColor("#003153"))
                            if authorizedUIDs.isEmpty { Text("僅自己可見").font(.caption).foregroundColor(.gray).padding(10).frame(maxWidth: .infinity, alignment: .leading).background(Color.white).cornerRadius(8).overlay(RoundedRectangle(cornerRadius: 8).stroke(Color.gray.opacity(0.3), style: StrokeStyle(lineWidth: 1, dash: [4]))) } else { Text("已選擇 \(authorizedUIDs.count) 個對象").font(.caption).foregroundColor(.blue).padding(10).frame(maxWidth: .infinity, alignment: .leading).background(Color.white).cornerRadius(8).overlay(RoundedRectangle(cornerRadius: 8).stroke(Color.gray.opacity(0.3), style: StrokeStyle(lineWidth: 1))) }
                            Button(action: { showMemberPicker = true }) { Text("+ 選擇授權對象...").font(.caption).foregroundColor(.gray).frame(maxWidth: .infinity).padding(8).overlay(RoundedRectangle(cornerRadius: 8).stroke(Color.gray.opacity(0.3), style: StrokeStyle(lineWidth: 1, dash: [4]))) }
                        }.padding(.vertical, 4)
                    }
                    
                    Section { ForEach($sections) { $section in CalPlanSectionRow(section: $section, sections: $sections) } }
                    
                    Section {
                        VStack(spacing: 8) {
                            HStack { Button("+ 大標題") { sections.append(CalPlanSectionModel(type: .heading)) }.buttonStyle(CalPlanAddButtonStyle(cornerRadius: 5)); Button("+ 內容資訊") { sections.append(CalPlanSectionModel(type: .content)) }.buttonStyle(CalPlanAddButtonStyle(cornerRadius: 5)) }
                            HStack { Button("+ 航班資訊") { sections.append(CalPlanSectionModel(type: .flight)) }.buttonStyle(CalPlanAddButtonStyle(cornerRadius: 5)); Button("+ 注意事項") { sections.append(CalPlanSectionModel(type: .warning)) }.buttonStyle(CalPlanAddButtonStyle(color: .red, cornerRadius: 5)) }
                        }.padding(.horizontal, 8).listRowBackground(Color.clear).listRowInsets(EdgeInsets(top: 5, leading: 0, bottom: 5, trailing: 0))
                        Button(action: savePlan) { Text("發佈內容").font(.headline).foregroundColor(.white).frame(maxWidth: .infinity).padding().background(safeParseHexColor("#003153")).cornerRadius(5) }.padding(.horizontal, 8).listRowBackground(Color.clear).listRowInsets(EdgeInsets(top: 5, leading: 0, bottom: 10, trailing: 0))
                    }
                }
                
                // 👉 完美解決 8.35.05 截圖中的空白問題：使用原生的 Section Header！
                Section(header: Text("歷史紀錄").font(.headline).foregroundColor(.gray)) {
                    if filteredHistoryPlans.isEmpty {
                        Text("尚無行政計畫").foregroundColor(.gray).font(.subheadline).frame(maxWidth: .infinity, alignment: .center).padding()
                    } else {
                        ForEach(filteredHistoryPlans) { plan in
                            // 👉 將點擊事件包裝進 Button，保證 List 內絕對能點擊觸發
                            Button(action: {
                                NotificationCenter.default.post(name: NSNotification.Name("TriggerDetailPlan"), object: plan)
                            }) {
                                PlanHistoryCompactCard(
                                    publisher: vm.userCache[plan.creatorUid]?.name ?? plan.creatorName,
                                    dateStr: DateFormatter.displayDateRange.string(from: plan.createdAt ?? plan.start),
                                    title: plan.title,
                                    timeRange: "\(plan.startDateStr) \(DateFormatter.displayTimeRange.string(from: plan.start)) ~ \(plan.endDateStr) \(DateFormatter.displayTimeRange.string(from: plan.end))"
                                )
                            }
                            .listRowInsets(EdgeInsets(top: 8, leading: 0, bottom: 8, trailing: 0))
                            .listRowBackground(Color.clear)
                            .listRowSeparator(.hidden)
                            .buttonStyle(PlainButtonStyle())
                        }
                    }
                }
            }
            .listStyle(.insetGrouped)
        }
        .frame(maxWidth: .infinity, maxHeight: .infinity)
        .background(Color(UIColor.systemGroupedBackground).ignoresSafeArea())
        .shadow(color: Color.black.opacity(0.15), radius: 8, x: -3, y: 0)
        .offset(x: dragOffset)
        .highPriorityGesture(
            DragGesture().onChanged { v in
                if v.startLocation.x < 40 && v.translation.width > 0 { dragOffset = v.translation.width }
            }.onEnded { v in
                if v.startLocation.x < 40 {
                    let pred = v.translation.width + v.predictedEndTranslation.width
                    if v.translation.width > 100 || pred > 150 {
                        withAnimation(.easeOut(duration: 0.25)) { dragOffset = UIScreen.main.bounds.width }
                        DispatchQueue.main.asyncAfter(deadline: .now() + 0.25) {
                            isPresented = false
                            dragOffset = 0
                        }
                    } else {
                        withAnimation(.spring()) { dragOffset = 0 }
                    }
                }
            }
        )
        .sheet(isPresented: $showMemberPicker) { CalMemberPickerView(vm: vm, selectedUIDs: $authorizedUIDs) }
    }
    
    private func savePlan() {
        guard let currentUser = Auth.auth().currentUser else { return }
        if mainTitle.isEmpty || sections.isEmpty { return }
        let db = Firestore.firestore(); var itemsData: [[String: Any]] = []
        for sec in sections {
            var dict: [String: Any] = ["type": "\(sec.type)", "text": sec.text]
            if sec.type == .flight {
                dict["flightNo"] = sec.flightNo; dict["date"] = DateFormatter.localISO.string(from: sec.date); dict["time"] = DateFormatter.localISO.string(from: sec.time); dict["arrivalTime"] = DateFormatter.localISO.string(from: sec.arrivalTime); dict["from"] = sec.from; dict["to"] = sec.to; dict["termFrom"] = sec.termFrom; dict["termTo"] = sec.termTo; dict["gateFrom"] = sec.gateFrom; dict["gateTo"] = sec.gateTo; dict["member"] = sec.member; dict["buddy"] = sec.buddy; dict["airline"] = sec.airline
            }
            itemsData.append(dict)
        }
        let planData: [String: Any] = [ "user": currentUser.email ?? "", "userName": vm.myName, "creatorUid": currentUser.uid, "title": mainTitle, "start": DateFormatter.localISO.string(from: startDate), "end": DateFormatter.localISO.string(from: endDate), "color": selectedColor, "sharedWith": authorizedUIDs, "items": itemsData, "orderIndex": Date().timeIntervalSince1970 * 1000, "timestamp": FieldValue.serverTimestamp(), "createdAt": FieldValue.serverTimestamp() ]
        db.collection("post").addDocument(data: planData) { err in if err == nil { DispatchQueue.main.async { vm.fetchEvents(); withAnimation(.spring()) { isPresented = false } } } }
    }
}

// MARK: - 📝 獨立編輯畫面
struct CalEditPlanView: View {
    let plan: CalendarEvent
    @ObservedObject var vm: CalendarViewModel
    @Binding var isPresented: Bool
    @State private var dragOffset: CGFloat = 0
    @State private var mainTitle: String = ""
    @State private var startDate: Date = Date()
    @State private var endDate: Date = Date()
    @State private var selectedColor: String = ""
    @State private var authorizedUIDs: [String] = []
    @State private var sections: [CalPlanSectionModel] = []
    @State private var showMemberPicker: Bool = false
    
    var body: some View {
        VStack(spacing: 0) {
            // 👉 完美貼齊狀態列的標題區塊
            HStack { Button(action: { withAnimation(.spring()) { isPresented = false } }) { Image(systemName: "chevron.left").font(.title3).foregroundColor(safeParseHexColor("#003153")) }; Text("編輯計畫").font(.headline).foregroundColor(safeParseHexColor("#003153")).padding(.leading, 10); Spacer() }
            .padding(.horizontal, 20).padding(.bottom, 12).padding(.top, 15)
            .background(Color.white.ignoresSafeArea(edges: .top)).shadow(color: Color.black.opacity(0.05), radius: 2, y: 2)
            
            List {
                Section {
                    TextField("計畫主標題", text: $mainTitle).padding(.vertical, 4)
                    VStack(alignment: .leading, spacing: 12) {
                        VStack(alignment: .leading, spacing: 4) { Text("開始時間 *").font(.caption).foregroundColor(.gray); HStack { DatePicker("", selection: $startDate, displayedComponents: .date).labelsHidden().environment(\.locale, Locale(identifier: "zh_TW")); Spacer(); DatePicker("", selection: $startDate, displayedComponents: .hourAndMinute).labelsHidden().environment(\.locale, Locale(identifier: "en_GB")).datePickerStyle(.compact) } }
                        VStack(alignment: .leading, spacing: 4) { Text("結束時間 *").font(.caption).foregroundColor(.gray); HStack { DatePicker("", selection: $endDate, displayedComponents: .date).labelsHidden().environment(\.locale, Locale(identifier: "zh_TW")); Spacer(); DatePicker("", selection: $endDate, displayedComponents: .hourAndMinute).labelsHidden().environment(\.locale, Locale(identifier: "en_GB")).datePickerStyle(.compact) } }
                    }.padding(.vertical, 4)
                    VStack(alignment: .leading, spacing: 8) { Text("計畫標籤顏色").font(.subheadline).foregroundColor(safeParseHexColor("#003153")); HStack { ColorPicker("", selection: Binding(get: { safeParseHexColor(selectedColor) }, set: { selectedColor = $0.toSafeHex() ?? "#003153" })).labelsHidden().frame(width: 40); ScrollView(.horizontal, showsIndicators: false) { HStack(spacing: 12) { ForEach(["#003153", "#dc3838", "#dab511", "#5ba4d6", "#44ad62", "#8e44ad"], id: \.self) { hex in Circle().fill(safeParseHexColor(hex)).frame(width: 30, height: 30).overlay(Circle().stroke(Color.gray.opacity(0.3), lineWidth: 1)).overlay(Image(systemName: "checkmark").foregroundColor(.white).opacity(selectedColor == hex ? 1 : 0)).onTapGesture { selectedColor = hex } } } } } }
                    VStack(alignment: .leading, spacing: 8) { Text("授權共享對象 (預設僅自己可見)").font(.subheadline).foregroundColor(safeParseHexColor("#003153")); if authorizedUIDs.isEmpty { Text("僅自己可見").font(.caption).foregroundColor(.gray).padding(10).frame(maxWidth: .infinity, alignment: .leading).background(Color.white).cornerRadius(8).overlay(RoundedRectangle(cornerRadius: 8).stroke(Color.gray.opacity(0.3), style: StrokeStyle(lineWidth: 1, dash: [4]))) } else { Text("已選擇 \(authorizedUIDs.count) 個對象").font(.caption).foregroundColor(.blue).padding(10).frame(maxWidth: .infinity, alignment: .leading).background(Color.white).cornerRadius(8).overlay(RoundedRectangle(cornerRadius: 8).stroke(Color.gray.opacity(0.3), style: StrokeStyle(lineWidth: 1))) }; Button(action: { showMemberPicker = true }) { Text("+ 編輯授權對象...").font(.caption).foregroundColor(.gray).frame(maxWidth: .infinity).padding(8).overlay(RoundedRectangle(cornerRadius: 8).stroke(Color.gray.opacity(0.3), style: StrokeStyle(lineWidth: 1, dash: [4]))) } }
                }
                
                Section {
                    ForEach($sections) { $section in
                        CalPlanSectionRow(section: $section, sections: $sections)
                    }
                }
                
                Section {
                    VStack(spacing: 8) {
                        HStack { Button("+ 大標題") { sections.append(CalPlanSectionModel(type: .heading)) }.buttonStyle(CalPlanAddButtonStyle(cornerRadius: 5)); Button("+ 內容資訊") { sections.append(CalPlanSectionModel(type: .content)) }.buttonStyle(CalPlanAddButtonStyle(cornerRadius: 5)) }
                        HStack { Button("+ 航班資訊") { sections.append(CalPlanSectionModel(type: .flight)) }.buttonStyle(CalPlanAddButtonStyle(cornerRadius: 5)); Button("+ 注意事項") { sections.append(CalPlanSectionModel(type: .warning)) }.buttonStyle(CalPlanAddButtonStyle(color: .red, cornerRadius: 5)) }
                    }.padding(.horizontal, 8).listRowBackground(Color.clear).listRowInsets(EdgeInsets(top: 5, leading: 0, bottom: 5, trailing: 0))
                    Button(action: updatePlan) { Text("儲存修改").font(.headline).foregroundColor(.white).frame(maxWidth: .infinity).padding(.vertical, 14).background(safeParseHexColor("#003153")).clipShape(RoundedRectangle(cornerRadius: 5)) }.buttonStyle(PlainButtonStyle()).padding(.top, 10).listRowBackground(Color.clear)
                }
            }
            .listStyle(.insetGrouped)
        }
        .frame(maxWidth: .infinity, maxHeight: .infinity)
        .background(Color(UIColor.systemGroupedBackground).ignoresSafeArea())
        .shadow(color: Color.black.opacity(0.15), radius: 8, x: -3, y: 0)
        .offset(x: dragOffset)
        .highPriorityGesture(
            DragGesture().onChanged { v in
                if v.startLocation.x < 40 && v.translation.width > 0 { dragOffset = v.translation.width }
            }.onEnded { v in
                if v.startLocation.x < 40 {
                    let pred = v.translation.width + v.predictedEndTranslation.width
                    if v.translation.width > 100 || pred > 150 {
                        withAnimation(.easeOut(duration: 0.25)) { dragOffset = UIScreen.main.bounds.width }
                        DispatchQueue.main.asyncAfter(deadline: .now() + 0.25) {
                            isPresented = false
                            dragOffset = 0
                        }
                    } else {
                        withAnimation(.spring()) { dragOffset = 0 }
                    }
                }
            }
        )
        .onAppear { mainTitle = plan.title; startDate = plan.start; endDate = plan.end; selectedColor = plan.color; authorizedUIDs = plan.sharedWith; sections = plan.planItems ?? [] }
        .sheet(isPresented: $showMemberPicker) { CalMemberPickerView(vm: vm, selectedUIDs: $authorizedUIDs) }
    }
    
    private func updatePlan() {
        if mainTitle.isEmpty || sections.isEmpty { return }
        let db = Firestore.firestore(); var itemsData: [[String: Any]] = []
        for sec in sections {
            var dict: [String: Any] = ["type": "\(sec.type)", "text": sec.text]
            if sec.type == .flight { dict["flightNo"] = sec.flightNo; dict["date"] = DateFormatter.localISO.string(from: sec.date); dict["time"] = DateFormatter.localISO.string(from: sec.time); dict["arrivalTime"] = DateFormatter.localISO.string(from: sec.arrivalTime); dict["from"] = sec.from; dict["to"] = sec.to; dict["termFrom"] = sec.termFrom; dict["termTo"] = sec.termTo; dict["gateFrom"] = sec.gateFrom; dict["gateTo"] = sec.gateTo; dict["member"] = sec.member; dict["buddy"] = sec.buddy; dict["airline"] = sec.airline }
            itemsData.append(dict)
        }
        db.collection("post").document(plan.id).updateData([ "title": mainTitle, "start": DateFormatter.localISO.string(from: startDate), "end": DateFormatter.localISO.string(from: endDate), "color": selectedColor, "sharedWith": authorizedUIDs, "items": itemsData, "updatedAt": FieldValue.serverTimestamp() ]) { err in
            if err == nil {
                DispatchQueue.main.async {
                    vm.fetchEvents()
                    var updatedPlan = plan
                    updatedPlan.title = mainTitle; updatedPlan.start = startDate; updatedPlan.end = endDate; updatedPlan.color = selectedColor; updatedPlan.sharedWith = authorizedUIDs; updatedPlan.planItems = sections
                    NotificationCenter.default.post(name: NSNotification.Name("PlanDidUpdate"), object: updatedPlan)
                    withAnimation { isPresented = false }
                }
            }
        }
    }
}

// 👉 授權成員選擇器
struct CalMemberPickerView: View {
    @Environment(\.presentationMode) var presentationMode
    @ObservedObject var vm: CalendarViewModel
    @Binding var selectedUIDs: [String]
    
    @State private var searchText = ""
    @State private var tempSelectedUIDs: [String] = []
    
    let lyBlue = safeParseHexColor("#003153")
    
    var filteredUsers: [(id: String, name: String, avatar: String)] {
        if searchText.isEmpty { return vm.allUsers }
        return vm.allUsers.filter { $0.name.lowercased().contains(searchText.lowercased()) }
    }
    
    var body: some View {
        NavigationView {
            VStack(spacing: 0) {
                HStack { Image(systemName: "magnifyingglass").foregroundColor(.gray); TextField("搜尋成員名稱...", text: $searchText) }
                .padding(10).background(Color(.systemGray6)).cornerRadius(10).padding()
                
                ScrollView {
                    LazyVGrid(columns: [GridItem(.adaptive(minimum: 80), spacing: 15)], spacing: 15) {
                        ForEach(filteredUsers, id: \.id) { user in
                            let isSelected = tempSelectedUIDs.contains(user.id)
                            Button(action: {
                                if isSelected { tempSelectedUIDs.removeAll { $0 == user.id } }
                                else { tempSelectedUIDs.append(user.id) }
                            }) {
                                VStack(spacing: 6) {
                                    ZStack {
                                        if !user.avatar.isEmpty {
                                            CalCachedAvatarView(urlString: user.avatar, size: 55).overlay(Circle().stroke(isSelected ? lyBlue : Color.gray.opacity(0.3), lineWidth: isSelected ? 3 : 1))
                                        } else {
                                            Image(systemName: "person.circle.fill").resizable().frame(width: 55, height: 55).foregroundColor(.gray).background(Color.white).clipShape(Circle()).overlay(Circle().stroke(isSelected ? lyBlue : Color.gray.opacity(0.3), lineWidth: isSelected ? 3 : 1))
                                        }
                                        
                                        if isSelected {
                                            Circle().fill(Color.white.opacity(0.6)).frame(width: 55, height: 55)
                                            Image(systemName: "checkmark").font(.title2).foregroundColor(lyBlue).fontWeight(.heavy)
                                        }
                                    }
                                    Text(user.name).font(.caption).fontWeight(isSelected ? .heavy : .medium).foregroundColor(isSelected ? lyBlue : .primary).lineLimit(1).frame(maxWidth: 70)
                                }
                            }
                        }
                    }
                    .padding(.horizontal)
                    .padding(.top, 10)
                }
                
                if !tempSelectedUIDs.isEmpty {
                    VStack(alignment: .leading, spacing: 5) {
                        Text("已選擇 \(tempSelectedUIDs.count) 位成員").font(.caption).foregroundColor(.gray).padding(.horizontal)
                        ScrollView(.horizontal, showsIndicators: false) {
                            HStack(spacing: 12) {
                                let sortedPreviews = tempSelectedUIDs.sorted { uid1, uid2 in
                                    let name1 = vm.userCache[uid1]?.name ?? "未知"
                                    let name2 = vm.userCache[uid2]?.name ?? "未知"
                                    return getCalMemberSortIndex(name1) < getCalMemberSortIndex(name2)
                                }
                                
                                ForEach(sortedPreviews, id: \.self) { uid in
                                    if let cache = vm.userCache[uid] {
                                        VStack {
                                            if !cache.avatar.isEmpty {
                                                CalCachedAvatarView(urlString: cache.avatar, size: 40)
                                            } else {
                                                Image(systemName: "person.circle.fill").resizable().frame(width: 40, height: 40).foregroundColor(.gray)
                                            }
                                            Text(cache.name).font(.system(size: 10)).lineLimit(1).frame(maxWidth: 50)
                                        }
                                    }
                                }
                            }
                            .padding(.horizontal)
                            .padding(.vertical, 5)
                        }
                    }
                    .padding(.vertical, 10)
                    .background(Color.white)
                    .shadow(color: Color.black.opacity(0.05), radius: 5, y: -2)
                }
            }
            .navigationTitle("選擇授權對象")
            .navigationBarTitleDisplayMode(.inline)
            .toolbar {
                ToolbarItem(placement: .navigationBarTrailing) {
                    Button("完成") {
                        selectedUIDs = tempSelectedUIDs
                        presentationMode.wrappedValue.dismiss()
                    }.fontWeight(.bold)
                }
                ToolbarItem(placement: .navigationBarLeading) {
                    Button("取消") { presentationMode.wrappedValue.dismiss() }.foregroundColor(.gray)
                }
            }
            .background(Color(UIColor.systemGroupedBackground))
            .onAppear { tempSelectedUIDs = selectedUIDs }
        }
    }
}

// 👉 計畫詳細資訊頁面 (無多餘空白，支援完美滑動透視返回)
struct CalPlanDetailView: View {
    let plan: CalendarEvent
    @ObservedObject var vm: CalendarViewModel
    @Binding var isPresented: Bool
    
    @State private var dragOffset: CGFloat = 0
    @State private var showDeleteConfirm = false
    @State private var refreshTrigger = UUID()
    
    var body: some View {
        VStack(spacing: 0) {
            // 👉 完美貼齊狀態列的標題區塊
            HStack {
                Button(action: { withAnimation(.spring()) { isPresented = false } }) { Image(systemName: "chevron.left").font(.title3).foregroundColor(safeParseHexColor("#003153")) }
                Text("計畫詳情").font(.headline).foregroundColor(safeParseHexColor("#003153")).padding(.leading, 10)
                Spacer()
            }.padding(.horizontal, 20).padding(.bottom, 12).padding(.top, 15).background(Color.white.ignoresSafeArea(edges: .top)).shadow(color: Color.black.opacity(0.05), radius: 2, y: 2)
            
            ScrollView {
                VStack(alignment: .leading, spacing: 20) {
                    
                    VStack(alignment: .center, spacing: 12) {
                        Text(plan.title)
                            .font(.system(size: 24, weight: .bold))
                            .foregroundColor(safeParseHexColor("#003153"))
                            .frame(maxWidth: .infinity, alignment: .center)
                            .multilineTextAlignment(.center)
                        
                        VStack(alignment: .center, spacing: 4) {
                            Text("開始：\(plan.startDateStr) \(format24Time(plan.start))")
                                .font(.subheadline).foregroundColor(.gray)
                                .fixedSize(horizontal: true, vertical: false)
                            Text("結束：\(plan.endDateStr) \(format24Time(plan.end))")
                                .font(.subheadline).foregroundColor(.gray)
                                .fixedSize(horizontal: true, vertical: false)
                        }
                        .frame(maxWidth: .infinity, alignment: .center)
                    }
                    .padding(.horizontal)
                    .padding(.top, 20)
                    .padding(.bottom, 10)
                    
                    if !plan.sharedWith.isEmpty {
                        VStack(spacing: 0) {
                            VStack(spacing: 8) {
                                if let creatorCache = vm.userCache[plan.creatorUid], !creatorCache.avatar.isEmpty {
                                    CalCachedAvatarView(urlString: creatorCache.avatar, size: 50).overlay(Circle().stroke(safeParseHexColor("#cfa900"), lineWidth: 2))
                                } else {
                                    Image(systemName: "person.circle.fill").resizable().frame(width: 50, height: 50).foregroundColor(safeParseHexColor("#cfa900")).background(Color.white).clipShape(Circle()).overlay(Circle().stroke(safeParseHexColor("#cfa900"), lineWidth: 2))
                                }
                                let cName = vm.userCache[plan.creatorUid]?.name ?? plan.creatorName
                                Text(cName.isEmpty ? "建立人" : cName).font(.subheadline).fontWeight(.bold).foregroundColor(safeParseHexColor("#003153"))
                            }
                            
                            VStack(spacing: 0) {
                                Rectangle().fill(Color.gray.opacity(0.3)).frame(width: 2, height: 15)
                                ZStack {
                                    Circle().fill(Color.white).frame(width: 24, height: 24).shadow(color: Color.black.opacity(0.1), radius: 2, x: 0, y: 1)
                                    Image(systemName: "link").font(.system(size: 10)).foregroundColor(.gray)
                                }
                                Rectangle().fill(Color.gray.opacity(0.3)).frame(width: 2, height: 15)
                            }
                            
                            GeometryReader { geo in
                                ScrollView(.horizontal, showsIndicators: false) {
                                    HStack(spacing: 15) {
                                        let sortedSharedWith = plan.sharedWith.sorted { uid1, uid2 in
                                            let name1 = vm.userCache[uid1]?.name ?? "未知"
                                            let name2 = vm.userCache[uid2]?.name ?? "未知"
                                            return getCalMemberSortIndex(name1) < getCalMemberSortIndex(name2)
                                        }
                                        
                                        ForEach(sortedSharedWith, id: \.self) { uid in
                                            let name = vm.userCache[uid]?.name ?? "未知"
                                            let avatarUrl = vm.userCache[uid]?.avatar ?? ""
                                            
                                            VStack(spacing: 6) {
                                                if !avatarUrl.isEmpty {
                                                    CalCachedAvatarView(urlString: avatarUrl, size: 45).overlay(Circle().stroke(safeParseHexColor("#5ba4d6"), lineWidth: 2))
                                                } else {
                                                    Image(systemName: "person.crop.circle.fill").resizable().frame(width: 45, height: 45).foregroundColor(safeParseHexColor("#5ba4d6")).background(Color.white).clipShape(Circle()).overlay(Circle().stroke(safeParseHexColor("#5ba4d6"), lineWidth: 2))
                                                }
                                                Text(name).font(.caption).fontWeight(.bold).foregroundColor(safeParseHexColor("#003153")).lineLimit(1).frame(maxWidth: 60)
                                            }
                                        }
                                    }
                                    .padding(.vertical, 5)
                                    .frame(minWidth: geo.size.width)
                                }
                            }
                            .frame(height: 85)
                        }
                        .padding(.vertical, 15)
                        .frame(maxWidth: .infinity)
                        .background(Color.white)
                        .cornerRadius(12)
                        .overlay(RoundedRectangle(cornerRadius: 12).stroke(Color.gray.opacity(0.15), lineWidth: 1))
                        .padding(.horizontal)
                        .padding(.bottom, 10)
                        .id(refreshTrigger)
                    }
                    
                    if let items = plan.planItems {
                        ForEach(items) { item in
                            if item.type == .flight {
                                FlightTicketPreview(item: item)
                                    .padding(.horizontal)
                            } else if item.type == .heading {
                                Text(item.text).font(.title3).fontWeight(.bold).foregroundColor(safeParseHexColor("#003153")).padding(.top, 10).padding(.horizontal, 24)
                            } else if item.type == .content {
                                Text(item.text).font(.body).foregroundColor(.primary).frame(maxWidth: .infinity, alignment: .leading).padding(.horizontal, 24)
                            } else if item.type == .warning {
                                HStack(alignment: .top) {
                                    Image(systemName: "exclamationmark.triangle.fill").foregroundColor(.red)
                                    Text(item.text).font(.subheadline).fontWeight(.bold).foregroundColor(.red)
                                }
                                .padding().frame(maxWidth: .infinity, alignment: .leading).background(Color.yellow.opacity(0.1)).border(Color.red.opacity(0.5), width: 1).cornerRadius(5).padding(.horizontal)
                            }
                        }
                    }
                    
                    let currentUid = Auth.auth().currentUser?.uid ?? ""
                    if plan.creatorUid == currentUid || vm.myName == "班長" {
                        VStack(spacing: 12) {
                            Divider().padding(.vertical, 10)
                            HStack(spacing: 15) {
                                Button(action: { NotificationCenter.default.post(name: NSNotification.Name("TriggerEditPlan"), object: plan) }) {
                                    Text("編輯內容").font(.headline).foregroundColor(safeParseHexColor("#003153")).frame(maxWidth: .infinity).padding(12).background(Color.white).cornerRadius(8).overlay(RoundedRectangle(cornerRadius: 8).stroke(safeParseHexColor("#003153"), lineWidth: 1))
                                }
                                Button(action: { showDeleteConfirm = true }) {
                                    Text("刪除計畫").font(.headline).foregroundColor(.white).frame(maxWidth: .infinity).padding(12).background(Color.red).cornerRadius(8)
                                }
                            }
                            .padding(.horizontal)
                            .padding(.bottom, 30)
                        }
                    }
                }
                .padding(.vertical)
            }
        }
        .frame(maxWidth: .infinity, maxHeight: .infinity)
        .background(Color(UIColor.systemGroupedBackground).ignoresSafeArea())
        .shadow(color: Color.black.opacity(0.15), radius: 8, x: -3, y: 0)
        .offset(x: dragOffset)
        .highPriorityGesture(
            DragGesture().onChanged { v in
                if v.startLocation.x < 40 && v.translation.width > 0 { dragOffset = v.translation.width }
            }.onEnded { v in
                if v.startLocation.x < 40 {
                    let pred = v.translation.width + v.predictedEndTranslation.width
                    if v.translation.width > 100 || pred > 150 {
                        withAnimation(.easeOut(duration: 0.25)) { dragOffset = UIScreen.main.bounds.width }
                        DispatchQueue.main.asyncAfter(deadline: .now() + 0.25) {
                            isPresented = false
                            dragOffset = 0
                        }
                    } else {
                        withAnimation(.spring()) { dragOffset = 0 }
                    }
                }
            }
        )
        .onAppear {
            var uids = plan.sharedWith
            uids.append(plan.creatorUid)
            vm.fetchMissingUsers(for: uids) { self.refreshTrigger = UUID() }
        }
        .alert(isPresented: $showDeleteConfirm) {
            Alert(
                title: Text("確認刪除計畫"), message: Text("確定要永久刪除此行政計畫嗎？此動作無法復原。"),
                primaryButton: .destructive(Text("刪除")) {
                    Firestore.firestore().collection("post").document(plan.id).delete()
                    vm.fetchEvents()
                    withAnimation { isPresented = false }
                },
                secondaryButton: .cancel(Text("取消"))
            )
        }
    }
    
    private func format24Time(_ d: Date) -> String {
        let f = DateFormatter(); f.locale = Locale(identifier: "en_GB"); f.dateFormat = "HH:mm"; return f.string(from: d)
    }
}

// 👉 機票預覽元件
struct FlightTicketPreview: View {
    var item: CalPlanSectionModel
    @State private var isExpanded = false
    let lyBlue = safeParseHexColor("#003153")
    
    var durationStr: String {
        var diff = item.arrivalTime.timeIntervalSince(item.time)
        if diff < 0 { diff += 86400 }
        if diff <= 0 { return "-- 小時 -- 分鐘" }
        let h = Int(diff) / 3600
        let m = (Int(diff) % 3600) / 60
        return "\(h) 小時 \(m) 分鐘"
    }
    
    private func format24Time(_ d: Date) -> String {
        let f = DateFormatter()
        f.locale = Locale(identifier: "en_GB")
        f.dateFormat = "HH:mm"
        return f.string(from: d)
    }

    private func getAirportCode(from name: String) -> String {
        if name.contains("羽田") || name.contains("HND") { return "HND" }
        if name.contains("成田") || name.contains("NRT") { return "NRT" }
        if name.contains("松山") || name.contains("TSA") { return "TSA" }
        if name.contains("桃園") || name.contains("TPE") { return "TPE" }
        if name.contains("福岡") || name.contains("FUK") { return "FUK" }
        if name.contains("札幌") || name.contains("千歲") || name.contains("CTS") { return "CTS" }
        if name.contains("高雄") || name.contains("小港") || name.contains("KHH") { return "KHH" }
        return "N/A"
    }
    
    var body: some View {
        VStack(spacing: 0) {
            HStack(spacing: 0) {
                VStack {
                    Image(systemName: "airplane").font(.title2).foregroundColor(.white).padding(.bottom, 4)
                    Text("FLIGHT").font(.system(size: 10, weight: .black)).foregroundColor(.white)
                }
                .frame(width: 75)
                .frame(maxHeight: .infinity)
                .background(lyBlue)
                
                Rectangle()
                    .stroke(style: StrokeStyle(lineWidth: 1, dash: [4]))
                    .foregroundColor(.white.opacity(0.5))
                    .frame(width: 1)
                
                VStack(alignment: .leading, spacing: 8) {
                    HStack(spacing: 12) {
                        Text(item.from.isEmpty ? "出發地" : item.from).font(.system(size: 20, weight: .black)).foregroundColor(lyBlue)
                        Image(systemName: "airplane").font(.system(size: 16)).foregroundColor(Color.gray.opacity(0.5))
                        Text(item.to.isEmpty ? "目的地" : item.to).font(.system(size: 20, weight: .black)).foregroundColor(lyBlue)
                    }
                    VStack(alignment: .leading, spacing: 2) {
                        Text("DATE & TIME").font(.system(size: 10, weight: .bold)).foregroundColor(.gray)
                        HStack(alignment: .bottom) {
                            Text("\(formattedDate(item.date)) \(format24Time(item.time))")
                                .font(.system(size: 16, weight: .bold)).foregroundColor(Color.black.opacity(0.8))
                                .fixedSize(horizontal: true, vertical: false)
                            Spacer()
                            Text(item.flightNo.isEmpty ? "航班號" : item.flightNo)
                                .font(.system(size: 18, weight: .black)).foregroundColor(safeParseHexColor("#cfa900"))
                        }
                    }
                }
                .padding(15)
                .background(Color.white)
            }
            .contentShape(Rectangle())
            .onTapGesture {
                withAnimation(.easeInOut(duration: 0.3)) { isExpanded.toggle() }
            }
            
            if isExpanded {
                VStack(spacing: 0) {
                    HStack {
                        Text(getAirportCode(from: item.from)).font(.system(size: 32, weight: .black)).foregroundColor(lyBlue)
                        Spacer()
                        VStack(spacing: 4) {
                            Text(durationStr).font(.caption).foregroundColor(.gray)
                            ZStack {
                                Rectangle().fill(Color.gray.opacity(0.3)).frame(height: 2)
                                Image(systemName: "airplane").foregroundColor(safeParseHexColor("#4caf50"))
                            }
                        }.frame(maxWidth: .infinity).padding(.horizontal, 20)
                        Spacer()
                        Text(getAirportCode(from: item.to)).font(.system(size: 32, weight: .black)).foregroundColor(lyBlue)
                    }
                    .padding(.bottom, 20)
                    
                    HStack(alignment: .top) {
                        VStack(alignment: .leading, spacing: 8) {
                            Text(item.from).font(.subheadline).fontWeight(.bold).foregroundColor(.primary)
                            
                            VStack(alignment: .leading, spacing: 4) {
                                Text("預定起飛時間").font(.caption).foregroundColor(.gray)
                                Text(format24Time(item.time))
                                    .font(.title2)
                                    .foregroundColor(safeParseHexColor("#4caf50"))
                                    .fixedSize(horizontal: true, vertical: false)
                            }
                            
                            HStack(spacing: 12) {
                                VStack(alignment: .leading, spacing: 4) {
                                    Text("航廈").font(.caption).foregroundColor(.gray)
                                    Text(item.termFrom.isEmpty ? "-" : item.termFrom).font(.headline).foregroundColor(.primary)
                                }
                                VStack(alignment: .leading, spacing: 4) {
                                    Text("登機門").font(.caption).foregroundColor(.gray)
                                    Text(item.gateFrom.isEmpty ? "-" : item.gateFrom).font(.headline).foregroundColor(.primary)
                                }
                            }
                        }
                        .frame(maxWidth: .infinity, alignment: .leading)
                        
                        Rectangle().fill(Color.gray.opacity(0.2)).frame(width: 1)
                        
                        VStack(alignment: .leading, spacing: 8) {
                            Text(item.to).font(.subheadline).fontWeight(.bold).foregroundColor(.primary)
                            
                            VStack(alignment: .leading, spacing: 4) {
                                Text("預定抵達時間").font(.caption).foregroundColor(.gray)
                                Text(format24Time(item.arrivalTime))
                                    .font(.title2)
                                    .foregroundColor(safeParseHexColor("#4caf50"))
                                    .fixedSize(horizontal: true, vertical: false)
                            }
                            
                            HStack(spacing: 12) {
                                VStack(alignment: .leading, spacing: 4) {
                                    Text("航廈").font(.caption).foregroundColor(.gray)
                                    Text(item.termTo.isEmpty ? "-" : item.termTo).font(.headline).foregroundColor(.primary)
                                }
                                VStack(alignment: .leading, spacing: 4) {
                                    Text("登機門").font(.caption).foregroundColor(.gray)
                                    Text(item.gateTo.isEmpty ? "-" : item.gateTo).font(.headline).foregroundColor(.primary)
                                }
                            }
                        }
                        .frame(maxWidth: .infinity, alignment: .leading)
                        .padding(.leading, 15)
                    }
                    
                    Divider().background(Color.gray.opacity(0.2)).padding(.vertical, 15)
                    
                    HStack {
                        Text("資料來源：系統模擬庫 (\(item.airline.isEmpty ? "未指定" : item.airline))").font(.caption).foregroundColor(.gray)
                        Spacer()
                        Image(systemName: "square.and.arrow.up").foregroundColor(.gray)
                    }
                }
                .padding(20)
                .background(Color(UIColor.systemGray6))
            }
        }
        .cornerRadius(12)
        .overlay(RoundedRectangle(cornerRadius: 12).stroke(Color.gray.opacity(0.3), lineWidth: 1))
        .padding(.vertical, 5)
    }
    
    private func formattedDate(_ d: Date) -> String {
        let f = DateFormatter(); f.locale = Locale(identifier: "zh_TW"); f.dateFormat = "yyyy-MM-dd"; return f.string(from: d)
    }
}

// 👉 動態輸入區塊 (縮小把手 padding，完美排版)
struct CalPlanSectionRow: View {
    @Binding var section: CalPlanSectionModel
    @Binding var sections: [CalPlanSectionModel]
    
    @State private var showDeleteConfirm = false
    
    struct FlightData {
        let from: String; let to: String; let termFrom: String; let termTo: String
        let gateFrom: String; let gateTo: String; let depTime: String; let arrTime: String; let airline: String
    }
    
    let mockFlightDB: [String: FlightData] = [
        "CI126": FlightData(from: "高雄", to: "東京成田", termFrom: "I", termTo: "2", gateFrom: "31", gateTo: "-", depTime: "12:55", arrTime: "17:35", airline: "中華航空"),
        "CI223": FlightData(from: "東京羽田", to: "台北松山", termFrom: "3", termTo: "1", gateFrom: "143", gateTo: "-", depTime: "07:55", arrTime: "10:55", airline: "中華航空"),
        "CI107": FlightData(from: "東京成田", to: "台北桃園", termFrom: "2", termTo: "2", gateFrom: "-", gateTo: "D5", depTime: "09:20", arrTime: "12:30", airline: "中華航空"),
        "CI222": FlightData(from: "台北松山", to: "東京羽田", termFrom: "1", termTo: "3", gateFrom: "-", gateTo: "-", depTime: "18:25", arrTime: "22:15", airline: "中華航空"),
        "CI104": FlightData(from: "台北桃園", to: "東京成田", termFrom: "2", termTo: "2", gateFrom: "D1", gateTo: "67A", depTime: "12:35", arrTime: "16:35", airline: "中華航空"),
        "BR191": FlightData(from: "東京羽田", to: "台北松山", termFrom: "3", termTo: "1", gateFrom: "111", gateTo: "-", depTime: "12:40", arrTime: "15:05", airline: "長榮航空"),
        "JL307": FlightData(from: "東京羽田", to: "福岡", termFrom: "1", termTo: "D", gateFrom: "12", gateTo: "-", depTime: "08:00", arrTime: "10:00", airline: "日本航空"),
        "JL309": FlightData(from: "東京羽田", to: "福岡", termFrom: "1", termTo: "D", gateFrom: "10", gateTo: "-", depTime: "09:05", arrTime: "11:05", airline: "日本航空"),
        "JL511": FlightData(from: "東京羽田", to: "札幌新千歲", termFrom: "1", termTo: "D", gateFrom: "-", gateTo: "-", depTime: "10:20", arrTime: "11:55", airline: "日本航空"),
        "JL503": FlightData(from: "東京羽田", to: "札幌新千歲", termFrom: "1", termTo: "D", gateFrom: "-", gateTo: "-", depTime: "07:30", arrTime: "09:00", airline: "日本航空"),
        "JL530": FlightData(from: "札幌新千歲", to: "東京羽田", termFrom: "D", termTo: "1", gateFrom: "-", gateTo: "-", depTime: "21:15", arrTime: "23:00", airline: "日本航空"),
        "JL528": FlightData(from: "札幌新千歲", to: "東京羽田", termFrom: "D", termTo: "1", gateFrom: "-", gateTo: "-", depTime: "21:05", arrTime: "22:50", airline: "日本航空")
    ]
    
    private func updateTime(for date: Date, timeString: String) -> Date {
        let parts = timeString.split(separator: ":")
        guard parts.count == 2, let h = Int(parts[0]), let m = Int(parts[1]) else { return date }
        return Calendar.current.date(bySettingHour: h, minute: m, second: 0, of: date) ?? date
    }
    
    var body: some View {
        VStack(alignment: .leading, spacing: 4) {
            
            HStack {
                Image(systemName: "line.3.horizontal")
                    .foregroundColor(.gray)
                    .font(.system(size: 16))
                    .padding(.vertical, 2)
                    .padding(.horizontal, 8)
                
                Spacer()
                
                Button(action: { showDeleteConfirm = true }) {
                    Image(systemName: "xmark.circle.fill")
                        .foregroundColor(.red)
                        .font(.title3)
                        .padding(.vertical, 2)
                        .padding(.horizontal, 8)
                }
            }
            .background(Color.clear)
            
            switch section.type {
            case .heading: TextField("大標題...", text: $section.text).font(.headline).foregroundColor(safeParseHexColor("#003153")).padding(10).background(Color.white).cornerRadius(5).overlay(RoundedRectangle(cornerRadius: 5).stroke(Color.gray.opacity(0.3), lineWidth: 1))
            case .content: ZStack(alignment: .topLeading) { if section.text.isEmpty { Text("內容資訊...").foregroundColor(Color(UIColor.placeholderText)).padding(.top, 12).padding(.leading, 8) }; TextEditor(text: $section.text).frame(minHeight: 60).padding(4).background(Color.white).cornerRadius(5).overlay(RoundedRectangle(cornerRadius: 5).stroke(Color.gray.opacity(0.3), lineWidth: 1)) }
            case .warning: ZStack(alignment: .topLeading) { if section.text.isEmpty { Text("注意事項...").foregroundColor(Color(UIColor.placeholderText)).padding(.top, 12).padding(.leading, 8) }; TextEditor(text: $section.text).frame(minHeight: 60).padding(4).background(Color.yellow.opacity(0.2)).border(Color.black, width: 1.5) }.cornerRadius(5)
            case .flight:
                VStack(spacing: 8) {
                    HStack {
                        TextField("航班號 (如 CI222)", text: $section.flightNo).padding(10).background(Color.white).cornerRadius(5)
                        Button(action: {
                            let fNo = section.flightNo.uppercased().replacingOccurrences(of: " ", with: "")
                            if let flight = mockFlightDB[fNo] {
                                section.from = flight.from; section.to = flight.to
                                section.termFrom = flight.termFrom; section.termTo = flight.termTo
                                section.gateFrom = flight.gateFrom; section.gateTo = flight.gateTo
                                section.airline = flight.airline
                                section.time = updateTime(for: section.date, timeString: flight.depTime)
                                section.arrivalTime = updateTime(for: section.date, timeString: flight.arrTime)
                            }
                        }) { Text("帶入").padding(.horizontal, 16).padding(.vertical, 10).background(safeParseHexColor("#003153")).foregroundColor(.white).cornerRadius(5) }
                    }
                    
                    LazyVGrid(columns: [GridItem(.flexible(), spacing: 10), GridItem(.flexible())], spacing: 10) {
                        DatePicker("日期", selection: $section.date, displayedComponents: .date).labelsHidden().environment(\.locale, Locale(identifier: "zh_TW")).padding(8).frame(maxWidth: .infinity, alignment: .center).background(Color.white).cornerRadius(5)
                        DatePicker("日期", selection: $section.date, displayedComponents: .date).labelsHidden().environment(\.locale, Locale(identifier: "zh_TW")).padding(8).frame(maxWidth: .infinity, alignment: .center).background(Color.white).cornerRadius(5).disabled(true).opacity(0.6)
                        
                        VStack(alignment: .leading, spacing: 4) {
                            Text("起飛").font(.caption).foregroundColor(.gray)
                            DatePicker("起飛", selection: $section.time, displayedComponents: .hourAndMinute).labelsHidden().environment(\.locale, Locale(identifier: "en_GB"))
                        }.padding(8).frame(maxWidth: .infinity, alignment: .leading).background(Color.white).cornerRadius(5)
                        
                        VStack(alignment: .leading, spacing: 4) {
                            Text("降落").font(.caption).foregroundColor(.gray)
                            DatePicker("降落", selection: $section.arrivalTime, displayedComponents: .hourAndMinute).labelsHidden().environment(\.locale, Locale(identifier: "en_GB"))
                        }.padding(8).frame(maxWidth: .infinity, alignment: .leading).background(Color.white).cornerRadius(5)
                        
                        TextField("起飛機場", text: $section.from).padding(10).background(Color.white).cornerRadius(5)
                        TextField("降落機場", text: $section.to).padding(10).background(Color.white).cornerRadius(5)
                        
                        TextField("起飛航廈", text: $section.termFrom).padding(10).background(Color.white).cornerRadius(5)
                        TextField("起飛登機門", text: $section.gateFrom).padding(10).background(Color.white).cornerRadius(5)
                        
                        TextField("降落航廈", text: $section.termTo).padding(10).background(Color.white).cornerRadius(5)
                        TextField("降落登機門", text: $section.gateTo).padding(10).background(Color.white).cornerRadius(5)
                        
                        TextField("成員", text: $section.member).padding(10).background(Color.white).cornerRadius(5)
                        TextField("同行", text: $section.buddy).padding(10).background(Color.white).cornerRadius(5)
                    }
                    
                    TextField("備註資訊", text: $section.text).padding(10).background(Color.white).cornerRadius(5)
                }.padding(10).background(Color.blue.opacity(0.05)).cornerRadius(5).overlay(RoundedRectangle(cornerRadius: 5).stroke(safeParseHexColor("#5ba4d6"), style: StrokeStyle(lineWidth: 1, dash: [4])))
            }
        }.padding(.vertical, 4).background(Color.clear).alert(isPresented: $showDeleteConfirm) { Alert(title: Text("確認刪除"), message: Text("確定要移除此內容區塊嗎？"), primaryButton: .destructive(Text("刪除")) { withAnimation { if let index = sections.firstIndex(where: { $0.id == section.id }) { sections.remove(at: index) } } }, secondaryButton: .cancel(Text("取消"))) }
    }
}

struct CalPlanAddButtonStyle: ButtonStyle {
    var color: Color = Color(red: 0, green: 49/255, blue: 83/255)
    var cornerRadius: CGFloat = 5
    func makeBody(configuration: Configuration) -> some View { configuration.label.font(.caption).fontWeight(.bold).foregroundColor(color).frame(maxWidth: .infinity).padding(.vertical, 10).background(Color.white).cornerRadius(cornerRadius).overlay(RoundedRectangle(cornerRadius: cornerRadius).stroke(color, lineWidth: 1)).scaleEffect(configuration.isPressed ? 0.95 : 1) }
}
```
