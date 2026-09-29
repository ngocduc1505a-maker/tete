# Telegram Gift Code Browser Bot — Tài liệu toàn bộ project

> **Phiên bản tài liệu:** 2026-09-30
**Mục đích:** Tài liệu vận hành, cấu hình, kiến trúc và xử lý sự cố cho bot nhận gift code từ Telegram, trích xuất code từ text/spoiler/caption/OCR và nhập code qua trình duyệt Edge.

---

## 1. Tổng quan

Project là một bot Telegram chạy theo mô hình:

```
Telegram channel
    ↓
Telethon event handler
    ↓
DurableInbox — SQLite chống mất tin / xử lý lại
    ↓
Message workers
    ↓
Trích xuất code:
  spoiler → marker/caption/plain text → OCR media/video nếu được phép
    ↓
Domain queue + account selection + rate limiter
    ↓
Browser adapter
    ↓
Edge CDP + TabPool + browser profile theo từng site
    ↓
Điền username/code → submit → đọc kết quả
    ↓
Database / CSV history / dashboard / log
```

Bot **không gửi code trực tiếp qua HTTP API của các site**. Bot điều khiển giao diện bằng Playwright kết nối vào Microsoft Edge đang chạy qua CDP port `9222`.

### Chức năng chính

- Nhận tin nhắn từ các channel Telegram đã khai báo trong `Config.CHANNEL_CONFIG`.

- Lọc đúng channel ở nguồn và xử lý durable inbox.

- Đọc code trong:
  - Tin nhắn text thông thường.
  - Caption ảnh/video.
  - Telegram spoiler.
  - Marker như `NHẬN CODE`, `CODE`, `MM88`, `RR88`, `XX88`, `GG88`.
  - Ảnh hoặc video được whitelist OCR.

- Phát code theo đúng domain/site.

- Hỗ trợ nhiều tài khoản cho mỗi site.

- Submit song song có giới hạn theo domain.

- Giữ thứ tự code trong queue của từng domain.

- Tự nhận diện thành công, thất bại, rate limit, không có kết quả và kết quả mơ hồ.

- Ghi database, history CSV, log theo kênh và dashboard.

- Có cơ chế retry, circuit breaker, watchdog Edge/CDP và dọn tab.

---

## 2. Cấu trúc thư mục và file

### 2.1. File khởi động và cài đặt

| File | Vai trò |
| --- | --- |
| `run.bat` | Script chạy production trên Windows. Kích hoạt virtualenv, kiểm tra `.env`, kiểm tra Edge CDP và chạy `main_script.py`. |
| `setup.bat` | Tạo `.venv312`, nâng pip và cài `requirements.txt`. |
| `requirements.txt` | Danh sách thư viện Python. |
| `.env` | Cấu hình production và thông tin nhạy cảm. Không chia sẻ file này. |
| `.env.example` | Mẫu biến môi trường, không chứa secret production. |
| `Dockerfile` | Cấu hình Docker Linux tùy chọn. Container vẫn cần kết nối tới Edge CDP bên ngoài. |
| `.dockerignore` | Loại secret, session, database, log, cache và test khỏi Docker build context. |
| `session_babao_bot.session` | Session Telethon. Đây là dữ liệu nhạy cảm, không gửi công khai. |

### 2.2. File Python chính

| File | Vai trò |
| --- | --- |
| `main_script.py` | Entry point và orchestration chính: Telegram client, event handler, durable inbox, message workers, domain workers, trích xuất code, submit và shutdown. |
| `config.py` | Nạp `.env`, định nghĩa tất cả cấu hình, channel, filter group, account override và danh sách OCR. |
| `browser_adapter.py` | Lớp adapter chuẩn hóa kết quả browser engine cho phần còn lại của bot. |
| `browser_engine.py` | Kết nối Edge CDP, quản lý TabPool, tìm input, điền form, CAPTCHA/Turnstile, click submit, đọc kết quả và dọn trang. |
| `browser_site_profiles.py` | Selector và timeout riêng cho từng site. Đây là nơi cần cập nhật khi frontend site đổi. |
| `submission_outcomes.py` | Phân loại kết quả thành công, thất bại, mơ hồ, không có kết quả hoặc rate limit. |
| `code_validator.py` | Kiểm tra độ dài, ký tự, entropy, filter group và tính hợp lệ theo domain. |
| `durable_inbox.py` | SQLite inbox có lease, claim, retry, retry-or-fail và trạng thái xử lý. |
| `queue_manager.py` | Theo dõi queue, giới hạn mềm/cứng và callback khi queue đầy. |
| `database.py` | Database lịch sử submit và chống trùng code. |
| `media_download_manager.py` | Tải ảnh/video Telegram, multipart download và metrics tải media. |
| `media_helpers.py` | Tiện ích media, normalize domain và chụp screenshot debug. |
| `image_code_extractor.py` | RapidOCR/ONNXRuntime, crop ảnh, resize, trích xuất frame video và nhận diện code. |
| `monitoring.py` | Health monitor, performance monitor và task metrics. |
| `timing.py` | Request timer và stage timing theo context. |
| `logger_setup.py` | Console log gọn, file log xoay vòng, log theo channel và context tag. |
| `dashboard.py` | Dashboard runtime và tổng hợp trạng thái submit/download. |
| `features.py` | Graceful shutdown, command registry và lệnh admin/status. |
| `task_tracker.py` | Tiện ích theo dõi task; phần logic tương thích cũng được dùng trong `main_script.py`. |

### 2.3. Thư mục runtime được tạo khi chạy

Các thư mục sau có thể xuất hiện sau khi bot khởi động:

```
logs/                 Log chính và log theo channel
logs/channels/        Log riêng từng channel/domain
data/                 Database code history và durable inbox
backups/sessions/     Backup session Telegram
ocr_tmp/              File media/frame tạm cho OCR
edge_bot_profile/     Edge profile riêng khi bot tự mở Edge
screenshots/          Screenshot/HTML khi kết quả UNKNOWN hoặc debug
```

---

## 3. Cài đặt trên Windows

### 3.1. Yêu cầu

- Windows 10/11.

- Python 3.10 trở lên; khuyến nghị Python 3.12.

- Microsoft Edge.

- FFmpeg trong `PATH` nếu cần OCR video.

- Tài khoản Telegram đã đăng nhập hoặc session Telethon hợp lệ.

- API ID/API Hash Telegram hợp lệ.

### 3.2. Cài đặt

1. Giải nén project vào:

   ```
   D:\AutoBot
   ```

1. Mở Command Prompt tại thư mục project.

1. Chạy:

   ```
   setup.bat
   ```

1. Điền các biến bắt buộc trong `.env`:

   ```
   API_ID=...
   API_HASH=...
   SESSION_NAME=session_babao_bot
   ```

1. Chạy:

   ```
   run.bat
   ```

`run.bat` sẽ:

- Kích hoạt `.venv312` hoặc `venv`.

- Kiểm tra `.env`.

- Kiểm tra Microsoft Edge.

- Mở Edge với CDP port `9222` nếu chưa có.

- Chạy `python main_script.py`.

### 3.3. Edge CDP

Các tham số quan trọng:

```
CDP host:       127.0.0.1
CDP port:       9222
Profile:        D:\AutoBot\edge_bot_profile
Profile name:   Default
```

Bot kết nối vào Edge qua CDP, không cần `playwright install` và không tự launch Chromium riêng.

Nếu Edge không kết nối được:

1. Đóng toàn bộ Edge.

1. Kiểm tra port `9222` không bị process khác dùng.

1. Chạy lại `run.bat`.

1. Không mở Edge thủ công bằng profile khác trong lúc bot đang khởi động.

---

## 4. Docker và Linux

`Dockerfile` có cài:

- Python 3.11.

- FFmpeg.

- Compiler cần cho một số package.

- Dependencies trong `requirements.txt`.

Tuy nhiên bot **không chạy browser bên trong container**. Container cần kết nối tới một Edge/Chromium đang chạy CDP bên ngoài.

Các dữ liệu nên mount:

```
/app/data
/app/logs
/app/backups
/app/edge_bot_profile
```

Không copy các file sau vào image production:

```
.env
*.session
*.db
*.sqlite*
logs/
data/
backups/
edge_bot_profile/
```

---

## 5. Cấu hình Telegram

### Biến quan trọng

| Biến | Giá trị hiện tại | Ý nghĩa |
| --- | --- | --- |
| `API_ID` | Secret | Telegram API ID. |
| `API_HASH` | Secret | Telegram API Hash. Không phải Bot API token. |
| `SESSION_NAME` | `session_babao_bot` | Tên session Telethon. |
| `TELEGRAM_CATCH_UP` | `false` | Không tự xử lý toàn bộ tin cũ khi khởi động. |
| `TELEGRAM_FILTER_AT_SOURCE` | `true` | Lọc channel ngay ở nguồn. |
| `TELEGRAM_INBOX_RECOVERY_MODE` | `at_least_once` | Khôi phục row chưa hoàn tất sau restart. |
| `TELEGRAM_INBOX_DRAIN_BATCH` | `250` | Số row recovery quét mỗi vòng. |
| `TELEGRAM_INBOX_DRAIN_INTERVAL` | `0.5` | Khoảng quét recovery. Tin mới vẫn đánh thức bằng Event. |
| `MESSAGE_WORKERS` | `6` | Số worker xử lý message. |
| `MAX_CONCURRENT_PROCESSING` | `50` | Giới hạn xử lý message đồng thời. |
| `MAX_CONCURRENT_INGRESS_TASKS` | `256` | Giới hạn task ingress. |

### Bot token cảnh báo

Các biến sau là tùy chọn và khác với `API_HASH`:

```
ALERT_BOT_TOKEN=...
ALERT_CHAT_ID=...
SUBMIT_SUCCESS_NOTIFY=false
SUBMIT_SUCCESS_CHAT_ID=...
```

Chúng chỉ dùng cho cảnh báo hoặc thông báo submit thành công. Không dùng Bot API token thay cho `API_HASH`.

---

## 6. Channel đang cấu hình

Tất cả channel dưới đây hiện đang `enabled=True` trong `Config.CHANNEL_CONFIG`.

| Chat ID | Tên channel | Site | Filter group |
| --- | --- | --- | --- |
| `-1002272716520` | QQ88 DỄ CHƠI, DỄ PHÁT TÀI | `tangquaqq88.com` | `qq88` |
| `-1002528908352` | KJC GÁI XINH | `xx88code.com` | `multi_site_strict` |
| `-1003503954906` | KJC - ĐỒNG HÀNH THỂ THAO | `xx88code.com` | `multi_site_strict` |
| `-1002817093108` | PHÁT CODE XX88 | `xx88code.com` | `multi_site_strict` |
| `-1003734537786` | XX88 SĂN CODE MỖI NGÀY | `xx88code.com` | `multi_site_strict` |
| `-1002768264448` | XX88 THỂ THAO ESPORT | `xx88code.com` | `multi_site_strict` |
| `-1002730903277` | XX88 DỊCH VỤ GIAI NHÂN | `xx88code.com` | `multi_site_strict` |
| `-1003731231345` | G88 DỊCH VỤ GIAI NHÂN | `gg88live.tv/nhap-code` | `multi_site_strict` |
| `-1002421765170` | QQ88 - KHO GIF | `tangquaqq88.com` | `qq88` |
| `-1003134541072` | MM88VIP Dịch Vụ Giai Nhân | `livemm88.net/nhap-code` | `mm88` |
| `-1002278162941` | QQ88 - TIN HOT 24/7 | `tangquaqq88.com` | `qq88` |
| `-1002324210129` | QQ88 - GIẢI TRÍ | `tangquaqq88.com` | `qq88` |
| `-1002377579866` | QQ88 - TIN TỨC MỖI NGÀY | `tangquaqq88.com` | `qq88` |
| `-1002325212717` | QQ88 - REVIEW PHIM HAY | `tangquaqq88.com` | `qq88` |
| `-1003802387209` | o8 TIN HOT 24H | `o8code.com` | `o8` |
| `-1003574944644` | o8 - TROLL BÓNG ĐÁ | `o8code.com` | `o8` |
| `-1002386905514` | RR88 DỊCH VỤ GIAI NHÂN | `rr88code.com` | `multi_site_strict` |
| `-1002446066378` | QQ88 - PHÁT CODE MIỄN PHÍ | `tangquaqq88.com` | `qq88` |
| `-1004435825431` | Hi88 PHÁT CODE MIỄN PHÍ NỖ HŨ-BẮN CÁ | `hi88-freecode.pages.dev` | `hi88` |
| `-1003933844700` | Hi88 - CƯỢC GIẢI TRÍ, KIẾM TIỀN TỶ | `hi88-freecode.pages.dev` | `hi88` |
| `-1002695720902` | Hi88 - KÊNH GIẢI TRÍ HOT | `hi88-freecode.pages.dev` | `hi88` |
| `-1002657420328` | Hi88 - TIN HOT MỖI NGÀY | `hi88-freecode.pages.dev` | `hi88` |
| `-1002018121888` | Hi88 - KHO GIF | `hi88-freecode.pages.dev` | `hi88` |
| `-1002662584621` | Hi88 - REVIEW PHIM HAY MỖI NGÀY | `hi88-freecode.pages.dev` | `hi88` |
| `-1002625548636` | Hi88 TUYỂN ĐẠI LÝ HOA HỒNG 60% | `hi88-freecode.pages.dev` | `hi88` |
| `-1003936595246` | GÁI 18+ | `livemm88.net/nhap-code` | `mm88` |
| `-1003939163957` | MM88 GIRL DANCE | `livemm88.net/nhap-code` | `mm88` |
| `-1002519029952` | MM88 ĐỘNG BÀN TƠ | `livemm88.net/nhap-code` | `mm88` |
| `-1003396129975` | o8 DỊCH VỤ GIAI NHÂN | `o8code.com` | `o8` |
| `-1003904150684` | o8 SOI KÈO 24/7 | `o8code.com` | `o8` |
| `-1004411105242` | KÈO BÓNG GG88 | `gg88live.tv/nhap-code` | `multi_site_strict` |
| `-1003912975699` | SOI KÈO MM88 | `livemm88.net/nhap-code` | `mm88` |
| `-1004406362195` | RR88 SOI KÈO | `rr88code.com` | `multi_site_strict` |
| `-1004352437280` | SOI KÈO CÙNG XX88 | `xx88code.com` | `multi_site_strict` |

### Xóa hoặc tắt channel

Ưu tiên đặt:

```python
"enabled": False
```

Nếu channel đã mất hoàn toàn, xóa entry khỏi `CHANNEL_CONFIG` và kiểm tra thêm các danh sách OCR/override liên quan. Không chỉ xóa tên channel mà để lại ID cũ trong allowlist.

---

## 7. Browser site profiles

Các profile nằm trong `browser_site_profiles.py`.

| Domain | URL nhập code | Ô tài khoản | Ô code | Nút submit |
| --- | --- | --- | --- | --- |
| QQ88 | `https://tangquaqq88.com` | `#account-code` | `#promo-code` | `button[type='submit']`, form button |
| HI88 | `https://hi88-freecode.pages.dev` | placeholder `Nhập tài khoản` | placeholder `Nhập mã` | `button[aria-label='Kiểm tra ngay']` hoặc submit |
| MM88 | `https://livemm88.net/nhap-code` | `#enter-code-username` | `#enter-code-code` | `img[alt='KIỂM TRA']` hoặc submit |
| RR88 | `https://rr88code.com` | `#username` | `#code` | `.code-submit-btn` hoặc submit |
| XX88 | `https://xx88code.com` | placeholder `Nhập tài khoản` | placeholder `Mã code` | submit |
| GG88 | `https://gg88live.tv/nhap-code` | `#enter-code-username` | `#enter-code-code` | `button[aria-label='Kiểm tra']` hoặc submit |
| O8 | `https://o8code.com` | placeholder `Nhập tên người dùng` | placeholder `Nhập mã code` | `.modal-submit-wrap button` hoặc submit |

### Quy tắc khi site đổi giao diện

1. Kiểm tra URL mới bằng `curl` hoặc mở bằng Edge.

1. Xác định selector thực tế bằng DevTools hoặc HTML.

1. Chỉ sửa `browser_site_profiles.py` trước.

1. Không sửa selector chung nếu chỉ một site đổi.

1. Kiểm tra `python -m py_compile`.

1. Chạy thử một code test an toàn hoặc chờ code thật.

1. Nếu cần debug, bật screenshot cho UNKNOWN/AMBIGUOUS; không bật screenshot FAILED lâu dài.

---

## 8. Luồng nhận và trích xuất code

### 8.1. Thứ tự ưu tiên

```
1. Telegram spoiler
2. Marker gần code
3. Caption/text sau spoiler
4. Plain text có validator
5. OCR ảnh/video nếu channel được phép
```

QQ88 và HI88 được cấu hình để không trả sớm sau khi tìm thấy spoiler; bot tiếp tục quét caption/text để không bỏ sót code thứ hai.

### 8.2. Filter groups

| Group | Site | Đặc điểm |
| --- | --- | --- |
| `qq88` | QQ88 | Cho phép random mix, giữ nguyên chữ/khoảng trắng theo cấu hình. |
| `hi88` | HI88 | Ưu tiên spoiler/text, OCR fallback có kiểm soát. |
| `multi_site_strict` | XX88, RR88, GG88 | Strict hơn, thường yêu cầu uppercase, giới hạn ký tự và entropy. |
| `mm88` | MM88 | Strict, uppercase và ưu tiên spoiler. |
| `o8` | O8 | Strict, uppercase và giới hạn code. |

### 8.3. OCR

Cấu hình hiện tại:

```
OCR_FAST_PATH=true
OCR_FAST_MIN_CHARS=8
OCR_FAST_VARIANTS=1
OCR_PREPROCESS_FALLBACK=true
OCR_MAX_IMAGE_SIDE=1024
OCR_CONFIDENCE_THRESHOLD=0.70
OCR_CROP_FALLBACK=false
OCR_FALLBACK_ALL_MEDIA=false
MAX_CONCURRENT_OCR=2
MAX_CONCURRENT_MEDIA_DOWNLOADS=4
VIDEO_OCR_MAX_FRAMES=3
```

Channel OCR chính hiện tại là channel XX88:

```
OCR_ALLOWED_CHANNEL_IDS=-1002817093108
OCR_MEDIA_ONLY_CHANNEL_IDS=-1002817093108
OCR_MEDIA_FALLBACK_CHANNEL_IDS=-1002817093108
```

QQ88/HI88 ưu tiên text/spoiler và không tự động OCR nếu không nằm trong allowlist phù hợp.

### 8.4. Video OCR

Video OCR phụ thuộc vào:

- FFmpeg.

- `VIDEO_OCR_MAX_FRAMES`.

- Danh sách thời điểm/crop trong cấu hình channel.

- Tốc độ tải media Telegram.

Nếu code xuất hiện gần đầu video nhưng cấu hình chỉ chụp frame cuối, thời gian xử lý sẽ tăng. Khi thay đổi frame time, cần cân bằng giữa tốc độ và khả năng bắt đúng code.

---

## 9. Luồng submit browser

### 9.1. TabPool

Mỗi domain có các tab nóng được preload. Cấu hình hiện tại:

```
TAB_POOL_SIZE=3
MAX_TAB_PER_DOMAIN_CAP=3
TAB_POOL_MIN_TABS_PER_DOMAIN=3
MAX_CONCURRENT_SUBMITS_PER_DOMAIN=3
TAB_ACQUIRE_WAIT_SECONDS=15.0
```

Mỗi `(domain, account )` có lock riêng để tránh hai task ghi đè username/code trong cùng một page.

### 9.2. Các bước submit

```
Acquire tab
  ↓
Kiểm tra page còn mở
  ↓
Goto nếu cần
  ↓
Tìm input theo site profile
  ↓
Điền username/code bằng React-compatible setter
  ↓
Kiểm tra lại giá trị input
  ↓
Xử lý CAPTCHA/Turnstile nếu có
  ↓
Click nút submit
  ↓
Poll selector kết quả site
  ↓
Phân loại kết quả
  ↓
Dọn form hoặc reload định kỳ
```

### 9.3. RR88

RR88 dùng Cloudflare Turnstile. Bot:

- Không bypass CAPTCHA.

- Không gửi request khi thiếu `captchaToken`.

- Giữ tab hiện tại để xác minh thủ công.

- Không coi widget luôn hiện là tab hỏng.

- Không reload tab khi đang chờ xác minh.

Nếu RR88 không submit:

1. Nhìn vào tab RR88.

1. Hoàn tất Turnstile nếu đang yêu cầu.

1. Không đóng tab chứa form.

1. Kiểm tra code đã được điền đúng chưa.

1. Nếu trang bị trắng sau thay đổi frontend, nhấn `Ctrl + F5`.

### 9.4. GG88

GG88 hiện dùng:

```
https://gg88live.tv/nhap-code
```

Không dùng URL cũ `gg88code.com`. Frontend hiện tại là Next.js và dùng:

```
#enter-code-username
#enter-code-code
button[aria-label="Kiểm tra"]
```

---

## 10. Rate limit, retry và thứ tự submit

Cấu hình hiện tại:

```
REQUESTS_PER_MINUTE=30
MAX_BURST=5
RATE_LIMIT_BACKOFF_SECONDS=5.0
ACCOUNTS_PER_CODE=2
MAX_RETRIES_PER_ACCOUNT=2
RETRY_ON_TIMEOUT=true
```

Không nên tăng `REQUESTS_PER_MINUTE` quá cao khi chưa có log thực tế. Nếu site trả `429`, `Too Many Requests` hoặc `RATE_LIMITED`:

1. Giảm concurrency theo domain.

1. Tăng backoff.

1. Không retry vô hạn.

1. Theo dõi tài khoản có bị giới hạn không.

Thứ tự cơ bản:

- Code vào domain queue theo thứ tự nhận.

- Worker lấy code FIFO.

- Mỗi code có thể fanout cho tối đa `ACCOUNTS_PER_CODE` tài khoản.

- Nếu tài khoản đạt giới hạn, worker đánh dấu tài khoản và chuyển tài khoản tiếp theo.

- Retry hạ tầng không được coi là code sai.

---

## 11. Database và history

### Database

```
DATABASE_PATH=data/code_history.db
TELEGRAM_INBOX_DB_PATH=data/telegram_inbox.db
```

Database lưu các thông tin như:

- Code.

- Domain/site.

- Account.

- Trạng thái submit.

- Thời gian.

- Kết quả.

- Chống trùng code.

- Trạng thái durable inbox.

### CSV history

Bot có thể ghi trong thư mục history:

```
code_history_YYYY-MM-DD.csv
daily_summary_YYYY-MM-DD.csv
```

Không xóa database khi bot đang chạy. Nếu cần reset dữ liệu, dừng bot trước và sao lưu toàn bộ `data/`.

---

## 12. Log và dashboard

### Log chính

```
logs/bot_activity.log
logs/channels/
```

Cấu hình hiện tại:

```
LOG_LEVEL=INFO
CONSOLE_LOG_LEVEL=INFO
COMPACT_CONSOLE_LOG=true
FILE_LOG_LEVEL=INFO
LOG_ROTATION_MAX_BYTES=10485760
LOG_ROTATION_BACKUP_COUNT=5
```

### Các log quan trọng

| Log | Ý nghĩa |
| --- | --- |
| `Nhận tin` | Telegram event đã vào pipeline. |
| `Đã nhận code` | Validator đã chấp nhận code. |
| `Đang nhập` | Domain worker bắt đầu submit. |
| `SUCCESS` | Site trả kết quả thành công. |
| `FAILED` | Site trả kết quả sai/hết hạn/lỗi. |
| `NO_RESULT` | Không thấy popup/kết quả trong timeout. |
| `AMBIGUOUS` | Có text nhưng chưa đủ chắc chắn để phân loại. |
| `RATE_LIMITED` | Site yêu cầu chờ. |
| `OCR-TIMING` | Timing tải media/OCR. |
| `Browser infrastructure retry` | Tab/browser hạ tầng gặp lỗi, không phải code sai. |
| `Turnstile verification pending` | RR88/Cloudflare cần xác minh. |

### Screenshot

```
SCREENSHOT_ON_UNKNOWN=true
SCREENSHOT_ON_FAILED=false
```

- `UNKNOWN`: nên giữ bật trong giai đoạn debug.

- `FAILED`: nên tắt trong production vì sai code/hết hạn thường xuyên và screenshot giữ tab lock lâu hơn.

---

## 13. Các lỗi thường gặp

### 13.1. `ApiIdInvalidError`

Thông báo:

```
The api_id/api_hash combination is invalid
```

Nguyên nhân thường gặp:

- `API_ID` sai.

- `API_HASH` sai hoặc đã bị rotate.

- File `.env` không được nạp từ đúng thư mục.

- Đang chạy nhầm project/virtualenv.

Kiểm tra hình thức, không in secret:

```
.venv312\Scripts\python.exe -c "from dotenv import load_dotenv; import os; load_dotenv('.env' ); print(os.getenv('API_ID','').isdigit()); print(len(os.getenv('API_HASH','')))"
```

Không gửi `API_HASH` hoặc session trong chat.

### 13.2. Không nhận được tin channel

Kiểm tra:

1. Channel có trong `Config.CHANNEL_CONFIG` không.

1. `enabled` có phải `True` không.

1. Bot/session có quyền đọc channel không.

1. `TELEGRAM_FILTER_AT_SOURCE=true` có đang loại channel không.

1. Chat ID trong Telegram có đúng dấu `-100...` không.

1. Log có dòng `accepted`, `ignored` hoặc `media_not_ocr_allowed` không.

### 13.3. Không nhận code trong spoiler/caption

Kiểm tra:

- Entity spoiler có tồn tại không.

- Text nằm trong `.message` hay `.text`.

- Filter group của channel.

- Validator có loại code do độ dài/ký tự/entropy không.

- Với QQ88/HI88: kiểm tra cả caption và plain-text fallback.

### 13.4. OCR chậm hoặc không ra code

Kiểm tra:

- Channel ID có nằm trong OCR allowlist không.

- File media có tải thành công không.

- Log `OCR-TIMING` để tách `download_ms` và `ocr_ms`.

- FFmpeg có trong PATH nếu là video.

- `OCR_MAX_IMAGE_SIDE` và crop có quá nhỏ không.

- CPU/RAM có vượt ngưỡng pause không.

### 13.5. RR88 HTTP 400

Nguyên nhân chính là thiếu Turnstile token. Không tăng retry. Hãy:

1. Xác minh Turnstile trên tab RR88.

1. Kiểm tra URL là `https://rr88code.com`.

1. Không dùng API trực tiếp.

1. Refresh tab nếu frontend bị lỗi React sau khi site đổi giao diện.

### 13.6. GG88 không nhập được

Kiểm tra URL phải là:

```
https://gg88live.tv/nhap-code
```

Không dùng:

```
https://gg88code.com
https://gg88live.tv
```

Selector hiện tại:

```
#enter-code-username
#enter-code-code
button[aria-label="Kiểm tra"]
```

### 13.7. Giao diện site bị trắng hoặc console có `ERR_FAILED`

Nguyên nhân trước đây có thể do bot chặn font/media. Mã hiện tại chỉ chặn một số host telemetry/quảng cáo, không chặn font/media/API submit.

Cách xử lý:

1. Cập nhật bản code mới nhất.

1. Đóng tab site cũ.

1. Chạy lại bot.

1. Nhấn `Ctrl + F5`.

1. Kiểm tra không có route chặn nhầm API/site asset.

---

## 14. Kiểm tra nhanh trước khi chạy production

### Syntax check

```
.venv312\Scripts\python.exe -m py_compile ^
  main_script.py ^
  config.py ^
  browser_engine.py ^
  browser_site_profiles.py ^
  submission_outcomes.py ^
  image_code_extractor.py
```

### Kiểm tra profile

```
.venv312\Scripts\python.exe -c "from browser_site_profiles import validate_profiles; print(validate_profiles( ))"
```

Kết quả mong muốn:

```
{}
```

### Kiểm tra GG88 URL

```
.venv312\Scripts\python.exe -c "from config import Config; print([x['url'] for x in Config.CHANNEL_CONFIG.values() if 'gg88' in x.get('url','').lower()])"
```

Kết quả phải chứa:

```
https://gg88live.tv/nhap-code
```

### Kiểm tra process

```
tasklist | findstr /I "python msedge"
```

### Kiểm tra port CDP

```
Test-NetConnection 127.0.0.1 -Port 9222
```

---

## 15. Quy trình cập nhật code an toàn

1. Dừng bot.

1. Sao lưu:
  - `.env`
  - `*.session`
  - `data/`
  - `logs/`
  - `backups/`

1. Cập nhật source code.

1. Không ghi đè `.env` bằng `.env.example`.

1. Chạy syntax check.

1. Kiểm tra selector/profile.

1. Chạy bot với một khoảng thời gian ngắn.

1. Kiểm tra log nhận tin và submit.

1. Chỉ sau khi ổn định mới thay toàn bộ bản production.

### Khi site đổi frontend

Không sửa nhiều module cùng lúc. Thứ tự nên là:

```
1. URL trong Config.CHANNEL_CONFIG
2. browser_site_profiles.py
3. Kết quả selector/result marker
4. browser_engine.py chỉ khi logic challenge thay đổi
5. main_script.py chỉ khi routing/filter thay đổi
```

---

## 16. Bảo mật

Không chia sẻ công khai:

```
.env
*.session
*.session-journal
*.db
*.sqlite*
edge_bot_profile/
backups/sessions/
```

Nếu nghi ngờ lộ thông tin:

1. Rotate `API_HASH` nếu cần.

1. Thu hồi/đổi Bot API token cảnh báo.

1. Đăng xuất/revoke session Telegram.

1. Xóa bản ZIP đã gửi nhầm.

1. Tạo session mới.

Không đưa các giá trị sau vào log hoặc chat:

- `API_HASH`.

- Bot token.

- Session file.

- Cookie Edge.

- Access token.

- Thông tin đăng nhập site.

---

## 17. Checklist vận hành hàng ngày

- [ ] Edge đang chạy với CDP port `9222`.

- [ ] Bot đã kết nối Telegram thành công.

- [ ] Không có `ApiIdInvalidError`.

- [ ] Có log `Handler ready`.

- [ ] Các channel cần thiết có trong config.

- [ ] GG88 dùng `/nhap-code`.

- [ ] RR88 đã xác minh Turnstile nếu site yêu cầu.

- [ ] Không có queue tăng liên tục.

- [ ] Không có nhiều `RATE_LIMITED`.

- [ ] `OCR-TIMING` không tăng bất thường.

- [ ] Log không phình quá nhanh.

- [ ] Database/history vẫn được ghi.

---

## 18. Tóm tắt cấu hình production hiện tại

```
# Telegram
TELEGRAM_CATCH_UP=false
TELEGRAM_FILTER_AT_SOURCE=true
TELEGRAM_INBOX_RECOVERY_MODE=at_least_once
TELEGRAM_INBOX_DRAIN_INTERVAL=0.5
MESSAGE_WORKERS=6

# Browser
EDGE_CDP_PORT=9222
MAX_CONCURRENT_SUBMITS=8
MAX_CONCURRENT_SUBMITS_PER_DOMAIN=3
TAB_POOL_SIZE=3
MAX_TAB_PER_DOMAIN_CAP=3
TAB_POOL_MIN_TABS_PER_DOMAIN=3

# Submit safety
ACCOUNTS_PER_CODE=2
MAX_RETRIES_PER_ACCOUNT=2
REQUESTS_PER_MINUTE=30
MAX_BURST=5

# OCR
MAX_CONCURRENT_OCR=2
MAX_CONCURRENT_MEDIA_DOWNLOADS=4
VIDEO_OCR_MAX_FRAMES=3
OCR_FAST_PATH=true
OCR_MAX_IMAGE_SIDE=1024
OCR_CROP_FALLBACK=false

# Logging/debug
COMPACT_CONSOLE_LOG=true
SCREENSHOT_ON_UNKNOWN=true
SCREENSHOT_ON_FAILED=false
```

> Các secret như `API_HASH`, bot token, chat ID nhạy cảm và session không được ghi trong tài liệu này. Hãy lấy giá trị production từ file `.env` cục bộ và không commit lên Git.
