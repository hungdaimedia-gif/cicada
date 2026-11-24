# Phương án thêm Tiếng Việt cho Cicada (知了)

Ngày lập: 2026-11-24 · Bản clone: `H:\HDS\apps\cicada` (commit `8450cb9`, upstream `hungdaimedia-gif/cicada`)

## 1. Hiện trạng (đã kiểm tra trong mã)

- **Chưa có tiếng Việt ở đâu cả.** Ngôn ngữ nghe (nguồn): en, ja, zh-CN, ko, es, fr, de, pt. Ngôn ngữ dịch (đích): zh-CN, zh-TW, en, ja, ko, es, fr, de, pt. Giao diện: tiếng Trung (gốc) + en/ja/ko.
- Chỉ một chỗ biết chữ "vi": `linux/cicada/core/support.py:100` (bảng tên ngôn ngữ cho prompt).
- Cách dịch giao diện: **câu tiếng Trung chính là khoá**; bản dịch nằm trong
  - `Resources/{en,ja,ko}.lproj/Localizable.strings` — ~1.650 dòng, dùng chung cho Mac/iOS và các bản port.
  - `windows/src/Cicada/Strings/*.lproj/Windows.strings` — 371 dòng, chữ riêng của Windows.
  - `linux/strings/*.lproj`, `android/app/src/main/res/values-*` — chữ riêng của Linux/Android.
  - Danh sách ngôn ngữ giao diện cố định trong `scripts/strings.py` (`LANGS = ["en", "ja", "ko"]`) và script strings của từng port.
- Danh sách ngôn ngữ nằm ở: Mac `Sources/Cicada/Languages.swift`; Windows `windows/src/Cicada.Core/Support.cs` (`Languages.Sources/Targets/EnglishName`); Android/Linux có bảng tương tự.
- Máy này: **chưa có .NET SDK** (`dotnet --list-sdks` trống) — bản Windows cần .NET 10 SDK mới build được. Bản Mac/iOS không build được trên Windows.

## 2. Ba mức "có tiếng Việt" — làm theo thứ tự

### Mức A — Dịch SANG tiếng Việt (phụ đề, ghi chú họp bằng tiếng Việt) ⭐ làm trước
Lợi ích lớn nhất, ít rủi ro nhất: nghe họp tiếng Anh/Trung/Nhật… mà phụ đề và biên bản ra tiếng Việt.

| Việc | File |
|---|---|
| Thêm `new("Tiếng Việt", "vi")` vào `Targets` | `windows/src/Cicada.Core/Support.cs` |
| `EnglishName("vi") => "Vietnamese"` (tên dùng trong prompt) | cùng file |
| Mac/iOS: thêm `.init(name: "Tiếng Việt", locale: "vi", code: "vi")` vào `targetLanguages` | `Sources/Cicada/Languages.swift` |
| Tên hiển thị trong danh sách bản ghi: `"vi": L("越南语")` | `Sources/Cicada/RecordList.swift:78` (+ chỗ tương ứng ở Windows) |
| Kiểm tra `InScript`: tiếng Việt là chữ Latin → nhánh "không phải CJK" đã đúng, chỉ thêm unit test | `Support.cs:100`, `Translator.swift:393` |
| Phần dịch nháp (partial) đi qua Google — Google có `vi`, không cần sửa | — |
| Notes (`Notes.swift:675`): tiếng Việt không phải CJK → kiểm tra tóm tắt/tiêu đề có ra đúng tiếng Việt có dấu | — |

Kiểm thử: `cicada-cli listen spike/clips/en.wav --target vi --notes` → phụ đề + biên bản tiếng Việt, đủ dấu, không lẫn chữ Trung.

### Mức B — NGHE tiếng Việt (người nói tiếng Việt → dịch ra ngôn ngữ khác)
Khó hơn, phụ thuộc mô hình nhận dạng giọng nói.

- **Windows / Android / Linux** (nghe bằng AI đám mây):
  - Ming (`Ming-3.0-audio-flash`) chỉ có `language_id` cho ja/ko/de/fr/es (`Ling.swift:760`, `Speak.cs:41`) → **phải thử thực tế** xem Ming có nghe được tiếng Việt không. Nếu kém: dùng OpenRouter/ZenMux với mô hình nghe được audio (vd. dòng Gemini) — app đã hỗ trợ chọn ở Cài đặt › AI 服务.
  - Thêm `new("Tiếng Việt", "vi")` vào `Sources`; `CloudHearing.swift:90` thêm `"vietnamese": "vi"` để nhận diện ngôn ngữ trả về.
- **Mac**: nhận dạng trên máy dùng `SpeechAnalyzer/SpeechTranscriber` của Apple — **cần xác minh có locale `vi-VN` không** (nếu không có, Mac phải dùng đường Ling/đám mây cho tiếng Việt). Kèm `LanguageDetector.swift` (ngưỡng tự nhận diện).
- **Cắt câu** (`HeardText.swift`): tiếng Việt viết cách chữ theo âm tiết, nên "5 từ" của tiếng Việt ngắn hơn tiếng Anh → nâng ngưỡng cắt câu cho `vi` (khoảng ×1,5) và thử với clip thật.
- **Sửa từ nghe nhầm**: luật hiện tại dựa vào pinyin (chỉ tiếng Trung). Với tiếng Việt cần so khớp bỏ dấu (`Nguyễn` ≈ `Nguyen`) — làm sau, không chặn mức B.
- **Nói lại (同声传译)**: giọng Ming không có tiếng Việt → dùng giọng Windows `vi-VN` (Microsoft An; cần cài gói ngôn ngữ Vietnamese trong Windows) hoặc mô hình giọng của dịch vụ AI. Thêm câu mẫu tiếng Việt vào `Speak.cs:81` và `Voices.swift`.

### Mức C — Giao diện tiếng Việt (menu, cài đặt, hướng dẫn)
Khối lượng lớn nhất nhưng là việc dịch, không đụng logic.

1. Thêm `vi` vào `LANGS` của `scripts/strings.py` và script strings của từng port.
2. Tạo `Resources/vi.lproj/Localizable.strings` (~1.650 câu) + `InfoPlist.strings` (11 dòng). Tiếng Việt không có số nhiều → không cần `.stringsdict` (hoặc bản đơn giản).
3. Tạo `windows/src/Cicada/Strings/vi.lproj/Windows.strings` (371 câu); Linux `linux/strings/vi.lproj`; Android `values-vi`.
4. Cách dịch: dịch từ **bản tiếng Anh** (`en.lproj`) sang Việt bằng LLM, giữ nguyên khoá tiếng Trung và các ký hiệu `%@ %lld %d`; `strings.py` kiểm tra ký hiệu khớp. Sau đó người đọc soát lại các câu dài.
5. Kiểm tra bố cục: chữ Việt dài hơn chữ Trung nhiều (gần như tiếng Anh) → mở từng màn hình Windows xem có bị cắt chữ.
6. Phụ (không bắt buộc): `README.vi.md`, trang web `site/i18n/vi.json`.

## 3. Lộ trình đề xuất (bạn dùng Windows)

| Bước | Nội dung | Ước lượng |
|---|---|---|
| 0 | Cài .NET 10 SDK, build bản Windows gốc, chạy thử với key AI | 0,5 ngày |
| 1 | **Mức A trên Windows** (đích = Tiếng Việt) + unit test | 0,5 ngày |
| 2 | Thử Ming / OpenRouter nghe clip tiếng Việt → quyết định đường nghe; làm **Mức B** trên Windows | 1–2 ngày |
| 3 | **Mức C**: dịch 1.650 + 371 câu, soát, kiểm tra bố cục | 1–2 ngày |
| 4 | (Nếu muốn gửi lên upstream) chuyển thay đổi sang Mac trước — luật của repo: Mac là bản gốc, cần máy Mac để build/kiểm tra | tuỳ |

## 4. Lưu ý / rủi ro

- Luật repo (`AGENTS.md`): Mac là bản tham chiếu, chữ giao diện và prompt đổi ở Mac trước. Nếu chỉ dùng cho mình trên Windows thì sửa thẳng bản Windows cũng được, nhưng muốn PR lên upstream thì phải theo luật này.
- Bản Windows **bắt buộc cần key dịch vụ AI** (Ling / OpenRouter / ZenMux…) — âm thanh được gửi lên đám mây để nhận dạng.
- Không thêm thư viện bên thứ ba (luật repo).
- Commit theo kiểu `feat(windows): …` bằng tiếng Anh.
