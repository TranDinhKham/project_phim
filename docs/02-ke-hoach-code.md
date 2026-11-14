# Kế hoạch code

> Đi kèm [01-y-tuong.md](01-y-tuong.md). Bản nháp v1 (2026-11-15).

## 1. Công nghệ

| Lớp | Lựa chọn | Ghi chú |
|---|---|---|
| Framework | Next.js (App Router) + TypeScript | Frontend và API chung một project |
| Giao diện | Tailwind CSS + shadcn/ui | |
| Database | SQLite qua Prisma | Đổi sang PostgreSQL khi deploy |
| Đăng nhập | Auth.js (credentials, mật khẩu băm bcrypt) | |
| Crawl | fetch + cheerio | Adapter riêng cho từng nguồn |
| Trình phát | iframe cho link embed, hls.js cho m3u8 | |
| AI | Claude API (`@anthropic-ai/sdk`), tool use + output JSON | Gọi từ server, không lộ API key ra trình duyệt |
| Kiểm tra dữ liệu | zod | Validate input API và output của adapter |

## 2. Cấu trúc thư mục

```
project_phim/
├─ docs/
├─ prisma/
│  └─ schema.prisma
├─ src/
│  ├─ app/
│  │  ├─ (site)/
│  │  │  ├─ page.tsx                  # Trang chủ
│  │  │  ├─ phim/page.tsx             # Danh sách + lọc
│  │  │  ├─ phim/[slug]/page.tsx      # Chi tiết
│  │  │  ├─ xem/[slug]/[tap]/page.tsx # Trình phát
│  │  │  ├─ goi-y/page.tsx            # AI gợi ý
│  │  │  └─ lich-su/page.tsx
│  │  ├─ (auth)/dang-nhap, dang-ky
│  │  ├─ admin/nguon/page.tsx         # Quản lý nguồn
│  │  └─ api/
│  │     ├─ auth/[...nextauth]/
│  │     ├─ recommend/                # POST: mô tả → phim gợi ý
│  │     ├─ history/                  # Ghi tiến độ xem
│  │     └─ admin/sources/, admin/sync/
│  ├─ components/
│  ├─ lib/
│  │  ├─ db.ts
│  │  ├─ auth.ts
│  │  ├─ ai/
│  │  │  ├─ client.ts                 # Khởi tạo Claude
│  │  │  ├─ tools.ts                  # Định nghĩa search_movies
│  │  │  ├─ taste.ts                  # Tóm tắt gu xem từ lịch sử
│  │  │  └─ recommend.ts              # Vòng lặp tool use
│  │  └─ crawler/
│  │     ├─ types.ts                  # Interface SourceAdapter
│  │     ├─ adapters/                 # Mỗi nguồn một file
│  │     ├─ normalize.ts              # Chuẩn hoá tên, thể loại, gộp trùng
│  │     └─ sync.ts                   # Chạy đồng bộ một nguồn
│  └─ scripts/sync.ts                 # Chạy đồng bộ từ dòng lệnh
└─ .env                               # ANTHROPIC_API_KEY, DATABASE_URL, AUTH_SECRET
```

## 3. Mô hình dữ liệu

```
User           id, email, passwordHash, name, role (USER | ADMIN), createdAt
Movie          id, slug, title, originalTitle, description, posterUrl, year,
               type (SINGLE | SERIES), country, duration, status, hidden, updatedAt
Genre          id, slug, name
MovieGenre     movieId, genreId
Episode        id, movieId, name, order
StreamLink     id, episodeId, sourceId, serverName, kind (EMBED | M3U8), url, broken
Source         id, name, baseUrl, adapter, config (JSON), enabled, lastSyncAt
SyncLog        id, sourceId, startedAt, finishedAt, added, updated, errors
WatchHistory   id, userId, movieId, episodeId, positionSec, durationSec,
               completed, updatedAt          (unique: userId + episodeId)
RecCache       userId, payload (JSON), historyVersion, createdAt
```

- `Movie` là phim đã gộp; `StreamLink` giữ link phát từ từng nguồn, nên một tập có thể có nhiều server.
- `RecCache.historyVersion` dùng để biết khi nào phải tính lại mục "Dành cho bạn".

## 4. Interface adapter nguồn

```ts
interface SourceAdapter {
  // Liệt kê phim theo trang (mới cập nhật trước)
  listMovies(page: number): Promise<{ items: MovieStub[]; hasMore: boolean }>;
  // Lấy đầy đủ thông tin + danh sách tập + link phát của một phim
  getMovie(stub: MovieStub): Promise<CrawledMovie>;
}
```

`sync.ts` gọi `listMovies` từng trang, dừng khi gặp phim không đổi kể từ lần đồng bộ trước, có giãn cách giữa các request. Output của adapter đi qua zod trước khi ghi DB, để trang nguồn đổi cấu trúc thì báo lỗi rõ ràng thay vì ghi dữ liệu rác.

## 5. Các giai đoạn

### Giai đoạn 0 — Khởi tạo
- Tạo project Next.js + TypeScript + Tailwind, khởi tạo git.
- Cài Prisma, viết schema, migrate, seed vài phim giả để làm giao diện.

**Xong khi:** `npm run dev` chạy, DB có dữ liệu mẫu.

### Giai đoạn 1 — Thu thập phim
- Viết `SourceAdapter`, `normalize.ts`, `sync.ts`.
- Viết adapter cho **một** nguồn đầu tiên.
- Script `npm run sync` + trang admin thêm nguồn và bấm đồng bộ.

**Xong khi:** chạy sync là DB có phim thật kèm tập và link phát; chạy lần hai không tạo bản trùng.

### Giai đoạn 2 — Duyệt và xem phim
- Trang chủ, danh sách có lọc + phân trang, tìm theo tên, trang chi tiết.
- Trình phát (iframe / hls.js), chọn tập, chọn server.

**Xong khi:** từ trang chủ bấm vào phim và xem được một tập.

### Giai đoạn 3 — Tài khoản và lịch sử
- Đăng ký, đăng nhập, phân quyền admin.
- Ghi tiến độ xem định kỳ khi đang phát; mục "Xem tiếp"; trang lịch sử.

**Xong khi:** đăng nhập ở trình duyệt khác vẫn thấy phim đang xem dở đúng vị trí.

### Giai đoạn 4 — AI gợi ý
- Tool `search_movies` truy vấn DB theo bộ lọc.
- `recommend.ts`: vòng lặp tool use, output JSON `{ movieId, reason }[]`, kiểm tra mọi `movieId` có tồn tại.
- `taste.ts`: tóm tắt lịch sử xem đưa vào prompt.
- Trang `/goi-y` và mục "Dành cho bạn" (có cache).
- Giới hạn số lượt gọi AI theo tài khoản.

**Xong khi:** nhập mô tả bất kỳ đều ra phim có trong kho kèm lý do; hai tài khoản có lịch sử khác nhau nhận gợi ý khác nhau.

### Giai đoạn 5 — Hoàn thiện
- Thêm adapter cho các nguồn còn lại, gộp trùng giữa các nguồn.
- Đánh dấu link chết, trạng thái loading/lỗi, responsive cho điện thoại.
- (Tuỳ chọn) deploy, đồng bộ theo lịch.

## 6. Rủi ro chính

| Rủi ro | Cách xử lý |
|---|---|
| Trang nguồn đổi cấu trúc hoặc chặn crawl | Adapter tách riêng từng nguồn, validate bằng zod, log lỗi trong `SyncLog`; ưu tiên nguồn có API JSON |
| Link phát chết | Nhiều server cho một tập, cờ `broken`, đồng bộ lại |
| AI gợi ý phim không có trong kho | Bắt buộc đi qua tool `search_movies`, kiểm tra `movieId` trước khi trả về |
| Chi phí API | Cache "Dành cho bạn", giới hạn lượt hỏi, chỉ gửi mô tả ngắn của phim ứng viên |
| Bản quyền | Giữ ở mức dùng cá nhân, không public, không kiếm tiền |
