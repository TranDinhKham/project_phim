# Ý tưởng: Web xem phim có AI gợi ý

> Trạng thái: bản nháp v1 (2026-11-15). Các mục đánh dấu **[CHƯA CHỐT]** cần quyết định thêm, xem cuối file.

## 1. Tóm tắt

Một web xem phim cá nhân, trong đó:

- Phim được **tự động thu thập** từ các trang web phim mà chủ web gắn link vào (trang admin nhập link nguồn → hệ thống vào đọc và lấy dữ liệu về).
- Người dùng **mô tả tự do** điều mình muốn xem ("tối nay mệt, muốn xem gì nhẹ nhàng", "phim giống Inception nhưng dễ hiểu hơn") và AI luôn trả về phim phù hợp **có trong kho**, không bị bó trong danh sách thể loại cố định.
- AI **cá nhân hoá theo lịch sử xem** của từng tài khoản.

## 2. Các quyết định đã chốt

| Hạng mục | Quyết định |
|---|---|
| Mục đích | Dự án cá nhân, quy mô nhỏ (bản thân + bạn bè) |
| Nguồn phim | Web tự truy cập các link trang phim đã gắn và lấy dữ liệu ra |
| Phạm vi AI | Hỏi gì cũng gợi ý được (mô tả tự do) |
| Tín hiệu cá nhân hoá | Lịch sử xem |
| Công nghệ | Next.js fullstack (TypeScript, Tailwind, Prisma) |
| Model AI | Claude API |
| Tài khoản | Có đăng nhập |

## 3. Người dùng và vai trò

- **Khách**: duyệt phim, tìm kiếm, xem phim. Hỏi AI được nhưng không có cá nhân hoá.
- **Thành viên** (đã đăng nhập): thêm lịch sử xem, xem tiếp phim dở, AI gợi ý theo gu riêng.
- **Admin** (chủ web): gắn link nguồn phim, chạy đồng bộ, ẩn/sửa phim lỗi.

## 4. Tính năng

### 4.1. Bản đầu tiên (MVP)

1. **Kho phim**: trang chủ (mới cập nhật, theo thể loại), trang danh sách có lọc (thể loại, quốc gia, năm, phim lẻ/bộ), tìm theo tên.
2. **Trang chi tiết phim**: poster, mô tả, thể loại, diễn viên, danh sách tập.
3. **Trình phát**: phát link embed (iframe) hoặc m3u8 (hls.js), chọn tập, chọn server nếu phim có nhiều nguồn.
4. **Đăng nhập**: đăng ký/đăng nhập bằng email + mật khẩu.
5. **Lịch sử xem**: tự ghi phim/tập đã xem và vị trí đang xem dở; mục "Xem tiếp" ở trang chủ.
6. **AI gợi ý**:
   - Ô nhập mô tả tự do → danh sách phim kèm một câu lý do cho từng phim.
   - Mục "Dành cho bạn" ở trang chủ, sinh từ lịch sử xem.
7. **Quản lý nguồn** (admin): thêm/xoá link nguồn, bấm đồng bộ, xem log lần đồng bộ gần nhất.

### 4.2. Để sau

- Yêu thích / xem sau, chấm điểm (thêm tín hiệu tốt cho AI ngoài lịch sử xem).
- Chat nhiều lượt với AI ("ít bạo lực hơn", "cái nào mới hơn").
- Đồng bộ nguồn tự động theo lịch.
- Bình luận, thông báo tập mới.

## 5. AI gợi ý hoạt động thế nào

Nguyên tắc: **AI chỉ được gợi ý phim có thật trong kho**, nên Claude không tự "nghĩ ra" tên phim mà phải tìm trong database thông qua công cụ (tool use).

```
Người dùng nhập mô tả
        │
        ▼
Server ghép: mô tả + tóm tắt gu xem (từ lịch sử) + danh sách thể loại/quốc gia hiện có
        │
        ▼
Claude gọi tool search_movies(thể loại, từ khoá, năm, quốc gia, loại phim...)
        │  (có thể gọi vài lần với bộ lọc khác nhau)
        ▼
Server truy vấn DB, trả danh sách ứng viên (tên, năm, thể loại, mô tả ngắn)
        │
        ▼
Claude chọn 5–10 phim hợp nhất, loại phim đã xem, viết lý do ngắn cho từng phim
        │
        ▼
Trả JSON có cấu trúc → giao diện hiển thị thẻ phim
```

- **Tóm tắt gu xem**: lấy khoảng 20–30 phim xem gần nhất (tên, thể loại, xem hết hay bỏ dở) đưa vào ngữ cảnh. Phim bỏ dở sớm được coi là tín hiệu "không hợp".
- **"Dành cho bạn"**: cùng luồng trên nhưng không có mô tả của người dùng; kết quả được lưu cache theo tài khoản và chỉ tính lại khi lịch sử xem thay đổi, để tiết kiệm chi phí API.
- **Người mới chưa có lịch sử**: hiện phim mới/phổ biến và mời nhập mô tả.
- **Câu hỏi không liên quan phim**: AI lịch sự kéo về chủ đề phim. Không có phim nào khớp → gợi ý phim gần nhất và nói rõ là gần đúng.

## 6. Thu thập phim từ các link nguồn

Admin gắn link trang phim → hệ thống đọc và lưu về DB của mình. Web **chỉ lưu thông tin và link phát**, không tải video về.

- Mỗi nguồn gắn với một **adapter** (đoạn code biết cách đọc trang đó):
  - **Adapter JSON**: dùng khi trang nguồn có sẵn API trả JSON. Ổn định nhất, nên ưu tiên.
  - **Adapter HTML**: đọc HTML bằng cheerio theo bộ selector cấu hình cho từng trang. Dễ gãy khi trang đổi giao diện.
- Mỗi trang phim có cấu trúc khác nhau, nên **thêm một trang mới thường phải viết/cấu hình thêm adapter**, không thể dán link bất kỳ là chạy ngay. **[CHƯA CHỐT]** danh sách trang nguồn cụ thể.
- **Gộp trùng**: cùng một phim ở nhiều nguồn được gộp thành một phim với nhiều "server" phát (so theo tên gốc + năm).
- **Link chết**: link phát của nguồn ngoài hay thay đổi, nên có nút đồng bộ lại và đánh dấu tập lỗi.

### Lưu ý bản quyền

Phần lớn trang phim miễn phí không có bản quyền nội dung. Dự án này nên giữ ở mức **dùng cá nhân**: không đưa lên public rộng rãi, không gắn quảng cáo hay thu tiền. Khi crawl thì giới hạn tốc độ để không làm ảnh hưởng trang nguồn. Nếu sau này muốn công khai, cần chuyển sang nguồn hợp pháp (metadata TMDB, nội dung có giấy phép).

## 7. Câu hỏi còn mở

1. **Trang nguồn cụ thể**: bạn định gắn những trang nào? (quyết định viết adapter JSON hay HTML, và dữ liệu có những trường gì)
2. **Database**: SQLite (đơn giản, chạy local) hay PostgreSQL (nếu deploy lên mạng)? Đề xuất: SQLite trước, Prisma cho phép đổi sau.
3. **Chạy ở đâu**: chỉ máy cá nhân, hay deploy (Vercel/VPS)? Ảnh hưởng tới cách chạy crawler.
4. **Kiểu đăng nhập**: email + mật khẩu là đủ, hay muốn thêm đăng nhập Google?
5. **Giao diện**: tối kiểu Netflix, hay có phong cách riêng? Chỉ tiếng Việt hay song ngữ?
6. **Giới hạn AI**: có cần giới hạn số lượt hỏi AI mỗi ngày cho mỗi tài khoản để kiểm soát chi phí không?
