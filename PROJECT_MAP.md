# PROJECT_MAP.md — HealthyMonitor

> Phạm vi khảo sát: repository `quangtien3035/App`, branch `main`, thư mục source `HealthyMonitor-main/`.
> Đã kiểm tra cây GitHub đệ quy (không bị `truncated`) và đọc các file entry point, cấu hình, services, store, screens/components chính. Đây là bản đồ dựa trên source hiện có; chưa chạy app, chưa kiểm chứng Firebase Rules hoặc firmware ESP32.

## 1. Tổng quan

HealthyMonitor là ứng dụng di động theo dõi nhịp tim (BPM) và độ bão hòa oxy máu (SpO₂). Source phía ứng dụng dùng **Expo + React Native + TypeScript**, Firebase Realtime Database/Auth, Zustand quản lý state, React Navigation cho điều hướng. Dự án có màn hình tổng quan, lịch sử, đăng nhập/đăng ký và cài đặt; có logic ghi cảnh báo, chẩn đoán thiết bị và xuất báo cáo PDF.

### Công nghệ đã xác minh

- Expo SDK khai báo `~57.0.24`; React Native `0.86.3`; React `19.2.3`; TypeScript `~6.0.3`.
- Firebase JS SDK `^12.19.0`; Realtime Database dùng ở `src/config/firebase.ts`. Authentication được dùng trong `AppNavigator.tsx` và `services/authService.ts`.
- Zustand `^5.0.15` ở `src/store/useHealthStore.ts`.
- React Navigation bottom tabs/native stack.
- AsyncStorage cho ngưỡng sức khỏe, hồ sơ, cài đặt tự xóa và chế độ giao diện.
- `react-native-chart-kit` cho biểu đồ; `expo-print` + `expo-sharing` cho xuất/chia sẻ PDF; `expo-location` và React Native Linking/Vibration cho nhánh SOS.

## 2. Cây thư mục rút gọn

```text
App/
├── README.md                         # Chỉ 5 bytes; không có tài liệu hữu ích
└── HealthyMonitor-main/
    ├── App.tsx                       # Entry component; khởi động/dừng monitor
    ├── app.json                      # Cấu hình Expo, icon, plugins
    ├── package.json                  # Dependencies, scripts, entry Expo
    ├── tsconfig.json
    ├── src/
    │   ├── config/firebase.ts        # Khởi tạo Firebase app + Realtime Database
    │   ├── navigation/AppNavigator.tsx # Auth gate, stack Login/Register, tabs chính
    │   ├── screens/
    │   │   ├── LoginScreen.tsx
    │   │   ├── RegisterScreen.tsx
    │   │   ├── DashboardScreen.tsx   # Chỉ số hiện tại + trạng thái sức khỏe
    │   │   ├── HistoryScreen.tsx     # Lịch sử, biểu đồ, thống kê, PDF
    │   │   └── SettingsScreen.tsx    # Ngưỡng, hồ sơ, lưu trữ, giao diện, logout
    │   ├── components/
    │   │   ├── AlertLog.tsx           # Đăng ký lắng nghe nhật ký cảnh báo
    │   │   ├── ConnectionStatus.tsx
    │   │   ├── DeviceDiagnosticCard.tsx
    │   │   ├── MeasurementStatus.tsx
    │   │   ├── MetricCard.tsx
    │   │   └── SmartConnectionCard.tsx
    │   ├── services/
    │   │   ├── HeartMonitorController.ts # Điều phối nhận dữ liệu + cảnh báo/SOS
    │   │   ├── healthService.ts       # Subscribe health_data realtime
    │   │   ├── historyService.ts      # Đọc/lưu/xóa health_history
    │   │   ├── alertService.ts        # Lưu/subscribe health_alerts
    │   │   ├── authService.ts         # Firebase Auth actions
    │   │   ├── settingsService.ts     # AsyncStorage cho ngưỡng/hồ sơ
    │   │   ├── deviceDiagnosticService.ts # Kiểm tra kết nối/dữ liệu
    │   │   └── reportService.ts       # Tạo PDF từ dữ liệu lịch sử
    │   ├── store/useHealthStore.ts    # State dùng chung bằng Zustand
    │   ├── types/index.ts             # Kiểu dữ liệu chính
    │   ├── utils/
    │   │   ├── healthAnalysis.ts      # Phân loại chỉ số so với ngưỡng
    │   │   └── format.ts              # Định dạng ngày/giờ
    │   └── theme/
    │       ├── ThemeContext.tsx       # Theme sáng/tối + lưu lựa chọn
    │       └── colors.ts              # Palette màu
    └── assets/                        # Icon, logo, hình nền; bỏ qua khi đọc logic
```

## 3. Trách nhiệm module

| Thành phần | Trách nhiệm / điểm cần tra cứu |
|---|---|
| `App.tsx` | Gọi `heartMonitorController.start()` khi app mount, gọi `stop()` khi unmount; bọc giao diện trong `ThemeProvider`. |
| `AppNavigator.tsx` | Theo dõi `onAuthStateChanged`; khi đang kiểm tra thì hiển thị spinner; có user thì `MainTabs`, chưa có user thì `AuthStack`. |
| `DashboardScreen.tsx` | Đọc dữ liệu realtime từ Zustand; dùng controller để reload ngưỡng; trình bày chỉ số và phân tích. Cần đọc sâu để xác minh chính xác các component được render. |
| `HistoryScreen.tsx` | Gọi `subscribeHistory(startTimestamp, ...)`; chọn phạm vi ngày (hôm nay/7 ngày/30 ngày), tính thống kê, biểu đồ và xuất PDF. |
| `SettingsScreen.tsx` | Đọc/lưu ngưỡng, hồ sơ và thời gian tự xóa; gọi xóa lịch sử; reload ngưỡng controller; logout. |
| `HeartMonitorController.ts` | Điều phối luồng sức khỏe chính: tải cài đặt, subscribe dữ liệu, cập nhật Zustand, so ngưỡng, lưu cảnh báo và khởi chạy chuỗi SOS. |
| `healthService.ts` | Nghe Firebase path `health_data`; chuyển snapshot thành `HealthData`; dùng thời gian nhận ở điện thoại. |
| `historyService.ts` | Firebase path `health_history`: `saveHealthHistory`, `subscribeHistory`, `clearAllHistory`, `deleteOldHistory`. |
| `alertService.ts` | Firebase path `health_alerts`: lưu một cảnh báo và subscribe danh sách cảnh báo. |
| `authService.ts` | Firebase email/password, guest, logout; Google đang bị vô hiệu hóa; Apple auth trả về credential nhưng chưa thấy flow Firebase credential trong file. |
| `settingsService.ts` | Lưu ngưỡng BPM/SpO₂, hồ sơ và cấu hình tự xóa qua AsyncStorage. |
| `useHealthStore.ts` | Shared state cho chỉ số, ngưỡng, kết nối, hồ sơ, tự xóa và user auth. |
| `healthAnalysis.ts` | Hàm thuần `analyzeHealth(data, thresholds)` trả mức `unconfigured/idle/critical/warning/normal`. |
| `deviceDiagnosticService.ts` | Đọc `.info/connected` và `health_data`; báo tình trạng Firebase, ESP32 và giá trị BPM/SpO₂. |
| `reportService.ts` | Tính thống kê, dựng HTML, in thành PDF và mở chia sẻ file. |
| `ThemeContext.tsx` | Cung cấp theme sáng/tối; đọc và lưu mode qua AsyncStorage. |
| `types/index.ts` | Định nghĩa `HealthData`, `HistoryRecord`, `HealthThresholds`, `AlertRecord`, `UserProfile`, `HealthAnalysis`. |

## 4. Luồng xử lý chính

### A. Khởi động app và điều hướng

```text
Expo entry (package.json: expo/AppEntry.js)
  └─ App.tsx
      ├─ useEffect → heartMonitorController.start()
      └─ ThemeProvider → AppNavigator
          ├─ onAuthStateChanged(Firebase Auth)
          ├─ isAuthLoading → spinner
          ├─ authUser có giá trị → MainTabs
          │   ├─ Home → DashboardScreen
          │   ├─ History → HistoryScreen
          │   └─ Settings → SettingsScreen
          └─ chưa đăng nhập → AuthStack
              ├─ LoginScreen
              └─ RegisterScreen
```

**Điểm cần chú ý:** monitor được khởi động ở `App.tsx` trước khi logic điều hướng xác định user đã đăng nhập hay chưa. Không nên mặc định việc đăng xuất sẽ dừng monitor; trong source, `stop()` được gọi trong cleanup của App component, còn logout được xử lý riêng trong SettingsScreen.

### B. Nhận số đo realtime

```text
ESP32 / nguồn ghi dữ liệu (firmware ngoài repository)
   → Firebase Realtime Database: health_data
   → healthService.subscribeHealthData()
   → HeartMonitorController callback
       ├─ useHealthStore.setHealthData(data)
       ├─ useHealthStore.setConnected(true)
       └─ checkThresholdsAndAlert(data)
           ├─ trong ngưỡng: reset cờ alert theo chỉ số
           └─ vượt ngưỡng lần đầu:
               ├─ alertService.saveHealthAlert(...)
               └─ executeAutomatedSOSSequence(...)
                   ├─ rung điện thoại
                   ├─ xin quyền/vị trí GPS
                   ├─ mở ứng dụng SMS đã điền nội dung
                   └─ mở trình quay số
   → DashboardScreen / component đọc Zustand để render
```

`healthService.ts` chuẩn hóa BPM/SpO₂ thành số; timestamp gán tại thời điểm điện thoại nhận snapshot, không lấy timestamp do nguồn gửi. Controller có import hàm lưu lịch sử nhưng đoạn gọi lưu định kỳ đang bị comment; vì vậy chưa có bằng chứng rằng lịch sử được ghi tự động trong luồng đang chạy.

### C. Lịch sử và báo cáo

```text
HistoryScreen
  → subscribeHistory(mốc thời gian)
  → historyService.ts
  → Firebase health_history (lọc timestamp bằng query)
  → chuẩn hóa thành HistoryRecord[]
  → sắp xếp mới nhất trước
  → biểu đồ + thống kê
  → export PDF (trong HistoryScreen hoặc reportService)
  → expo-print + expo-sharing
```

Hai nơi liên quan PDF cần phân biệt: `HistoryScreen.tsx` có hàm `exportPdf` riêng; `services/reportService.ts` cũng export `exportHealthReportPdf`. Chưa có bằng chứng trong phần import đã khảo sát rằng màn hình lịch sử đang gọi reportService, nên không được coi hai đường này là một.

### D. Cài đặt

```text
SettingsScreen
  ├─ settingsService ↔ AsyncStorage
  │   ├─ ngưỡng maxBpm / minSpo2
  │   ├─ hồ sơ người dùng / số điện thoại khẩn cấp
  │   └─ số ngày giữ lịch sử tự động
  ├─ saveThresholds → heartMonitorController.reloadThresholds()
  ├─ historyService.clearAllHistory / deleteOldHistory
  ├─ ThemeContext.setMode → AsyncStorage
  └─ authService.logoutUser → Firebase signOut
      → AppNavigator nhận onAuthStateChanged và đổi stack
```

## 5. Dữ liệu và các phụ thuộc

- Firebase path đã thấy trong code: `health_data`, `health_history`, `health_alerts`, `.info/connected`.
- `HeartMonitorController` phụ thuộc `healthService`, `historyService`, `alertService`, `settingsService`, Zustand store, kiểu `HealthData`, Expo Location và API Vibration/Linking.
- `DashboardScreen` phụ thuộc Zustand, controller, ThemeContext và biểu đồ.
- `HistoryScreen` phụ thuộc `historyService`, `HistoryRecord`, ThemeContext và thư viện chart/print/share.
- `SettingsScreen` phụ thuộc settings/history/auth services, Zustand, controller và ThemeContext.
- `AppNavigator` phụ thuộc screens, Zustand, ThemeContext, Firebase Auth và config.
- `README.md` ở root chỉ có 5 bytes; `HealthyMonitor-main/CLAUDE.md` rất ngắn; `AGENTS.md` chỉ hướng dẫn tham khảo tài liệu Expo SDK v57 trước khi sửa code. Không có tài liệu giải thích kiến trúc hữu ích đi kèm.

## 6. Những điểm chưa xác minh / cần kiểm tra thêm

1. **Firmware ESP32 và MAX30102 không nằm trong cây repository đã truy xuất.** Cần firmware để xác minh cách đo, lọc, định dạng và ghi dữ liệu lên `health_data`.
2. **Quy tắc bảo mật Firebase chưa được cung cấp.** Chưa thể xác minh ai đọc/ghi được các path dữ liệu; cũng chưa thấy dữ liệu lịch sử/cảnh báo được phân vùng theo UID người dùng.
3. **Nguồn sinh lịch sử:** `saveHealthHistory` hiện diện, nhưng lệnh gọi lưu định kỳ trong `HeartMonitorController.start()` đã bị comment. Cần xác minh có chỗ khác gọi hàm này không.
4. **Vòng đời kết nối monitor:** `App.tsx` bật monitor toàn app; cần kiểm tra kỳ vọng khi logout, khi Firebase auth đổi user và khi app trở về foreground.
5. **SOS:** code rung và mở SMS/điện thoại sau khi yêu cầu quyền vị trí; đây không đồng nghĩa tin nhắn đã được gửi hoặc cuộc gọi đã thực hiện. Cần test thực tế trên thiết bị/OS được hỗ trợ.
6. **Đăng nhập Apple:** `loginWithApple()` lấy credential từ Expo Apple Authentication, nhưng trong file không thấy trao đổi credential đó với Firebase Auth. Cần kiểm tra luồng giao diện và yêu cầu triển khai.
7. **Đăng nhập khách:** `loginAsGuest()` dùng Firebase anonymous auth, nhưng `AppNavigator` dựa trên `onAuthStateChanged`; cần xác minh cài đặt Firebase cho phép anonymous sign-in.
8. **Cài đặt SDK mới:** package khai báo Expo SDK 57 và `AGENTS.md` yêu cầu đọc docs đúng phiên bản. Chưa chạy cài dependency/build/test, nên chưa xác nhận app chạy được trên máy hoặc Expo Go.

## 7. Thứ tự học đề xuất

1. `package.json` + `App.tsx`: hiểu app chạy từ đâu và khởi tạo gì.
2. `src/navigation/AppNavigator.tsx`: hiểu auth gate và điều hướng.
3. `src/types/index.ts` + `src/store/useHealthStore.ts`: hiểu cấu trúc dữ liệu và state dùng chung.
4. `src/config/firebase.ts` + `src/services/healthService.ts`: hiểu kết nối Firebase và luồng dữ liệu realtime.
5. `src/services/HeartMonitorController.ts`: hiểu bộ điều phối, ngưỡng, cảnh báo và SOS.
6. `src/utils/healthAnalysis.ts` + `src/screens/DashboardScreen.tsx`: nối logic phân tích với giao diện.
7. `src/services/historyService.ts` + `src/screens/HistoryScreen.tsx`: lịch sử, query, biểu đồ và PDF.
8. `src/services/settingsService.ts` + `src/screens/SettingsScreen.tsx`: lưu cài đặt, ngưỡng, hồ sơ, dọn lịch sử.
9. `src/services/authService.ts` + Login/Register screens: các kiểu auth và vòng đời đăng nhập.
10. `alertService.ts`, `deviceDiagnosticService.ts`, components, `reportService.ts`, theme/utils: các nhánh hỗ trợ.

Lý do: đi từ điểm vào → điều hướng → kiểu dữ liệu/state → dịch vụ I/O → logic nghiệp vụ → UI → tính năng phụ. Mỗi lớp sẽ có ngữ cảnh cần thiết trước khi đọc lớp kế tiếp.

## 8. Cách chạy (dựa trên scripts khai báo)

Trong thư mục `HealthyMonitor-main/`:

```bash
npm install
npm start
# hoặc
npm run android
npm run ios
npm run web
```

Đây là scripts được khai báo trong `package.json`, không phải kết quả test đã chạy. Các tính năng native như đăng nhập Apple/Google, location, SMS/call và chia sẻ PDF cần kiểm tra riêng trên nền tảng/thiết bị tương ứng.
