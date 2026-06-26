# Hướng dẫn public Narratium.ai cho mọi người chơi (Tiếng Việt)

Tài liệu này hướng dẫn triển khai dự án **Narratium.ai / AI-Chat-Roleplay** thành một website công khai để nhiều người có thể truy cập và chơi qua trình duyệt.

> Lưu ý quan trọng: mã nguồn hiện tại là ứng dụng **Next.js xuất static** (`output: "export"`). Phần lớn dữ liệu người chơi được lưu **trên trình duyệt của từng người** bằng `localStorage` và `IndexedDB`, không phải một cơ sở dữ liệu máy chủ dùng chung. Vì vậy, khi public cho nhiều người, mỗi người chơi sẽ có bộ nhân vật, hội thoại, cấu hình API riêng trên thiết bị/trình duyệt của họ.

---

## 1. Dự án này chạy theo mô hình nào?

Sau khi xem mã nguồn, có thể tóm tắt kiến trúc vận hành như sau:

- Frontend: **Next.js 15 + React 19**.
- Build production: `next build` và xuất ra thư mục static `out/`.
- Serve production: dùng `serve -s out` hoặc bất kỳ static hosting nào.
- PWA: có cấu hình service worker qua `next-pwa`.
- Lưu dữ liệu cục bộ:
  - `localStorage`: lưu trạng thái đăng nhập khách, cấu hình model/API, ngôn ngữ, cài đặt giao diện.
  - `IndexedDB` database `CharacterAppDB`: lưu nhân vật, hội thoại, world book, regex script, ảnh nhân vật, agent conversation, memory/RAG.
- Gọi model AI:
  - Người chơi nhập API key/base URL/model trong giao diện.
  - Hỗ trợ endpoint tương thích OpenAI và Ollama/local model.
  - API key nhập trong trình duyệt sẽ nằm ở phía client của người chơi.

---

## 2. Khi public cho nhiều người cần hiểu rõ điều gì?

### 2.1. Đây không phải server game tập trung

Ứng dụng được build thành static site. Không có backend bắt buộc để lưu nhân vật/hội thoại chung cho tất cả người chơi.

Điều này có nghĩa:

- Người A tạo nhân vật thì dữ liệu nằm trong trình duyệt của người A.
- Người B không tự động thấy dữ liệu của người A.
- Nếu người chơi xóa cache/trình duyệt hoặc đổi thiết bị, dữ liệu cục bộ có thể mất nếu chưa export/backup.
- Muốn đồng bộ dữ liệu giữa nhiều thiết bị/người dùng thì cần phát triển thêm backend riêng.

### 2.2. Không nên nhúng API key bí mật vào build public

Dự án có hỗ trợ biến môi trường dạng `NEXT_PUBLIC_*`. Tất cả biến `NEXT_PUBLIC_*` sẽ được đưa vào bundle phía trình duyệt khi build, vì vậy **không được đặt API key bí mật dùng chung cho toàn bộ website** nếu bạn không muốn người khác có thể trích xuất key.

Khuyến nghị an toàn:

- Để mỗi người chơi tự nhập API key của họ trong giao diện model.
- Nếu muốn dùng key chung, hãy tạo backend proxy riêng có giới hạn quota, rate limit, xác thực, kiểm soát nội dung và không để lộ key thật cho frontend.

### 2.3. Ollama/local model khi public

Nếu người chơi chọn Ollama với URL mặc định kiểu `http://localhost:11434`, thì `localhost` là máy của **người chơi**, không phải máy chủ public website.

- Người chơi chạy Ollama trên máy cá nhân: dùng `http://localhost:11434` trong trình duyệt của họ.
- Bạn chạy Ollama trên server public: cần cấu hình reverse proxy HTTPS và CORS rất cẩn thận; không nên mở thẳng Ollama ra internet nếu không có bảo vệ.

---

## 3. Yêu cầu hệ thống

### 3.1. Chạy bằng Node.js

Cần:

- Node.js 20 trở lên, khuyến nghị Node 20 LTS.
- pnpm, npm hoặc yarn. Repo có `pnpm-lock.yaml`, `package-lock.json`, `yarn.lock`, nhưng Dockerfile đang dùng pnpm. Khuyến nghị dùng **pnpm** để đồng nhất.

### 3.2. Chạy bằng Docker

Cần:

- Docker.
- Docker Compose.
- Cổng public, ví dụ `80/443` qua reverse proxy hoặc `3000` để test trực tiếp.

---

## 4. Chạy thử local trước khi public

### Bước 1: tải mã nguồn

```bash
git clone <URL_REPO_CUA_BAN>
cd AI-Chat-Roleplay
```

Nếu bạn đang ở sẵn thư mục repo thì bỏ qua bước clone.

### Bước 2: cài dependencies

Khuyến nghị:

```bash
corepack enable
corepack prepare pnpm@latest --activate
pnpm install --frozen-lockfile
```

Nếu môi trường không dùng pnpm, có thể dùng npm:

```bash
npm install
```

### Bước 3: chạy dev server

```bash
pnpm dev
```

Mở trình duyệt:

```text
http://localhost:3000
```

Nếu cổng 3000 bận, Next.js có thể báo cổng khác hoặc bạn có thể chạy:

```bash
pnpm dev -- -p 3001
```

---

## 5. Build production static

Chạy:

```bash
pnpm build
```

Sau khi build thành công, thư mục `out/` sẽ chứa website static.

Chạy thử bản production static:

```bash
npx serve -s out -l 3000
```

Mở:

```text
http://localhost:3000
```

---

## 6. Public nhanh bằng Docker Compose

Repo đã có `Dockerfile` và `docker-compose.yml`. Cách này phù hợp nếu bạn muốn đưa lên VPS.

### Bước 1: chuẩn bị file môi trường nếu cần

Bạn có thể tạo `.env.production` hoặc truyền biến môi trường trong Docker Compose. Với cấu hình hiện tại, biến quan trọng nhất để metadata/canonical đúng domain là:

```env
NEXT_PUBLIC_BASE_URL=https://ten-mien-cua-ban.com
```

Nếu dùng Google OAuth hoặc backend auth riêng, xem thêm mục biến môi trường bên dưới.

### Bước 2: sửa `docker-compose.yml` cho domain thật

Trong `docker-compose.yml`, đổi:

```yaml
NEXT_PUBLIC_BASE_URL=http://localhost:3000
```

thành:

```yaml
NEXT_PUBLIC_BASE_URL=https://ten-mien-cua-ban.com
```

### Bước 3: build và chạy container

```bash
docker compose up -d --build
```

Kiểm tra log:

```bash
docker compose logs -f web
```

Truy cập thử:

```text
http://IP_SERVER:3000
```

Nếu chạy ổn, cấu hình reverse proxy HTTPS ở mục tiếp theo.

---

## 7. Public bằng Nginx reverse proxy + HTTPS

Giả sử container đang chạy ở `127.0.0.1:3000` và domain là `ten-mien-cua-ban.com`.

### 7.1. Cấu hình Nginx mẫu

Tạo file:

```bash
sudo nano /etc/nginx/sites-available/narratium
```

Nội dung mẫu:

```nginx
server {
    listen 80;
    server_name ten-mien-cua-ban.com www.ten-mien-cua-ban.com;

    location / {
        proxy_pass http://127.0.0.1:3000;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

Kích hoạt site:

```bash
sudo ln -s /etc/nginx/sites-available/narratium /etc/nginx/sites-enabled/narratium
sudo nginx -t
sudo systemctl reload nginx
```

### 7.2. Cài HTTPS bằng Certbot

```bash
sudo apt update
sudo apt install -y certbot python3-certbot-nginx
sudo certbot --nginx -d ten-mien-cua-ban.com -d www.ten-mien-cua-ban.com
```

Sau đó truy cập:

```text
https://ten-mien-cua-ban.com
```

---

## 8. Public bằng static hosting (Vercel, Netlify, Cloudflare Pages)

Vì dự án xuất static, bạn có thể deploy lên các dịch vụ static hosting.

### 8.1. Cấu hình chung

- Install command: `pnpm install --frozen-lockfile`
- Build command: `pnpm build`
- Output directory: `out`
- Node version: 20.x

### 8.2. Vercel

Vercel thường nhận Next.js tự động, nhưng vì dự án dùng `output: "export"`, hãy đảm bảo output directory là `out` nếu cấu hình thủ công.

Biến môi trường nên đặt:

```env
NEXT_PUBLIC_BASE_URL=https://ten-du-an.vercel.app
```

### 8.3. Netlify

Cấu hình:

```text
Build command: pnpm build
Publish directory: out
```

Nếu dùng SPA fallback không cần thiết vì Next static export tạo từng route HTML, nhưng nếu gặp lỗi refresh route, có thể thêm file `_redirects` vào `public/`:

```text
/* /index.html 200
```

Chỉ thêm khi bạn kiểm tra thấy cần, vì fallback kiểu này có thể che lỗi route tĩnh.

### 8.4. Cloudflare Pages

Cấu hình:

```text
Framework preset: Next.js hoặc None
Build command: pnpm build
Build output directory: out
Node.js version: 20
```

---

## 9. Biến môi trường tham khảo

Tùy nhu cầu, bạn có thể dùng các biến sau:

| Biến | Dùng để làm gì | Có nên public? |
| --- | --- | --- |
| `NEXT_PUBLIC_BASE_URL` | URL public của website, dùng cho metadata | Có |
| `NEXT_PUBLIC_API_BASE_URL` | Backend auth/API riêng nếu bạn có triển khai | Có, nhưng endpoint phải an toàn |
| `NEXT_PUBLIC_API_URL` | Base URL mặc định cho model OpenAI-compatible | Có, nếu không chứa bí mật |
| `NEXT_PUBLIC_API_KEY` | API key mặc định phía client | **Không khuyến nghị** cho website public |
| `NEXT_PUBLIC_GOOGLE_OAUTH_CLIENT_ID` | Google OAuth client ID | Có |
| `NEXT_PUBLIC_GOOGLE_OAUTH_CLIENT_SECRET` | Google OAuth client secret | **Không nên đưa vào frontend public** |
| `NEXT_PUBLIC_GOOGLE_OAUTH_REDIRECT_URI` | Redirect URI Google OAuth | Có |

> Quy tắc nhớ nhanh: biến bắt đầu bằng `NEXT_PUBLIC_` là dữ liệu phía trình duyệt có thể đọc được sau khi build. Đừng đặt secret thật vào đó.

---

## 10. Hướng dẫn người chơi sử dụng sau khi vào website

### 10.1. Đăng nhập

Nếu bạn không triển khai backend auth riêng, hãy hướng dẫn người chơi dùng chế độ khách/local deployment.

- Chọn đăng nhập khách nếu có hộp thoại đăng nhập.
- Dữ liệu sẽ lưu trong trình duyệt hiện tại.

### 10.2. Chọn ngôn ngữ

Ứng dụng hiện có locale tiếng Trung và tiếng Anh. Nếu người chơi Việt Nam chưa có bản dịch tiếng Việt trong UI, có thể dùng tiếng Anh.

### 10.3. Cấu hình model AI

Trong phần cấu hình model/API:

#### Dùng OpenAI hoặc dịch vụ tương thích OpenAI

Ví dụ:

```text
Type: OpenAI
Base URL: https://api.openai.com/v1
Model: gpt-4o-mini hoặc model bạn muốn dùng
API Key: API key của người chơi
```

Nếu dùng OpenRouter hoặc gateway tương thích OpenAI, nhập base URL/model/API key theo nhà cung cấp đó.

#### Dùng Ollama trên máy người chơi

Người chơi cần cài và chạy Ollama trước:

```bash
ollama serve
ollama pull qwen2.5:7b
```

Sau đó trong website nhập:

```text
Type: Ollama
Base URL: http://localhost:11434
Model: qwen2.5:7b
API Key: để trống
```

Lưu ý: nếu website chạy HTTPS còn Ollama là HTTP localhost, một số trình duyệt/chính sách mixed content/CORS có thể gây lỗi. Người chơi cần kiểm tra console trình duyệt hoặc dùng cấu hình proxy local nếu cần.

### 10.4. Nhập/tạo nhân vật

Người chơi có thể:

- Tạo nhân vật trong giao diện.
- Import character card nếu tính năng import hỗ trợ định dạng tương ứng.
- Tải character card từ nguồn tích hợp nếu website truy cập được GitHub/raw file.

### 10.5. Backup dữ liệu

Vì dữ liệu lưu ở trình duyệt, nên hướng dẫn người chơi export/backup dữ liệu định kỳ nếu giao diện có nút export/import data.

Khuyến nghị trước khi:

- Xóa cache trình duyệt.
- Đổi máy.
- Đổi browser profile.
- Chạy dọn rác hệ thống.

---

## 11. Checklist trước khi mở public

- [ ] Chạy local `pnpm dev` không lỗi nghiêm trọng.
- [ ] Chạy `pnpm build` thành công.
- [ ] Chạy thử `npx serve -s out -l 3000` và chat được với model.
- [ ] `NEXT_PUBLIC_BASE_URL` đã đổi sang domain thật.
- [ ] Không nhúng API key bí mật vào `NEXT_PUBLIC_API_KEY`.
- [ ] Nếu có backend auth riêng, `NEXT_PUBLIC_API_BASE_URL` trỏ đúng HTTPS endpoint.
- [ ] Nếu dùng Google OAuth, redirect URI khớp domain public.
- [ ] Website chạy HTTPS.
- [ ] Có trang/đoạn cảnh báo người dùng về việc API key và dữ liệu lưu cục bộ.
- [ ] Có hướng dẫn backup/export dữ liệu cho người chơi.
- [ ] Tuân thủ license AGPL-3.0 và yêu cầu attribution/link repo khi triển khai web public.

---

## 12. Lệnh vận hành thường dùng

### Cài dependency

```bash
pnpm install --frozen-lockfile
```

### Chạy dev

```bash
pnpm dev
```

### Build production

```bash
pnpm build
```

### Serve static production

```bash
npx serve -s out -l 3000
```

### Docker build + run

```bash
docker compose up -d --build
```

### Xem log Docker

```bash
docker compose logs -f web
```

### Dừng Docker

```bash
docker compose down
```

---

## 13. Xử lý lỗi thường gặp

### 13.1. Build lỗi do dependency

Thử xóa dependency cũ và cài lại:

```bash
rm -rf node_modules .next out
pnpm install --frozen-lockfile
pnpm build
```

### 13.2. Website trắng trang sau deploy

Kiểm tra:

- Có đúng publish directory là `out/` không.
- Console trình duyệt có lỗi JavaScript không.
- Static hosting có phục vụ đúng file route không.
- Service worker/PWA cache có đang giữ bản cũ không. Thử hard reload hoặc unregister service worker.

### 13.3. Không lấy được danh sách model

Kiểm tra:

- Base URL đúng chưa.
- API key đúng chưa.
- Nhà cung cấp có endpoint `/models` tương thích OpenAI không.
- CORS có cho phép gọi trực tiếp từ trình duyệt không.

### 13.4. Chat lỗi 401/403

Thường do:

- API key sai/hết hạn.
- Model không được cấp quyền.
- Base URL sai nhà cung cấp.
- Nhà cung cấp chặn gọi trực tiếp từ browser.

### 13.5. Chat lỗi CORS

Vì ứng dụng static gọi API model trực tiếp từ trình duyệt, nhà cung cấp API phải cho phép CORS. Nếu không, cần:

- Dùng nhà cung cấp cho phép browser CORS; hoặc
- Tạo backend proxy riêng; hoặc
- Dùng local model/Ollama phù hợp.

### 13.6. Mất dữ liệu nhân vật/hội thoại

Nguyên nhân thường gặp:

- Người chơi xóa site data/cache.
- Dùng trình duyệt khác hoặc profile khác.
- Chế độ ẩn danh.
- Dữ liệu IndexedDB bị trình duyệt dọn.

Cách giảm rủi ro:

- Export/backup định kỳ.
- Không chơi lâu dài trong incognito/private mode.
- Không xóa site data nếu chưa backup.

---

## 14. Gợi ý mô hình public an toàn

### Mô hình đơn giản nhất

- Deploy static site lên Vercel/Netlify/Cloudflare Pages/VPS.
- Mỗi người chơi tự nhập API key hoặc dùng Ollama local.
- Không có tài khoản server, không lưu dữ liệu server.

Ưu điểm:

- Dễ triển khai.
- Ít chi phí server.
- Không giữ API key/dữ liệu riêng tư của người chơi trên máy chủ của bạn.

Nhược điểm:

- Dữ liệu không đồng bộ đa thiết bị.
- Người chơi phải tự quản lý API key/backup.

### Mô hình nâng cao

- Static frontend public.
- Backend proxy riêng cho model API.
- Database riêng để đồng bộ user/character/dialogue.
- Auth, rate limit, quota, moderation, logging, backup.

Ưu điểm:

- Trải nghiệm giống dịch vụ online hoàn chỉnh.
- Có thể cấp quota/key chung an toàn hơn.
- Đồng bộ dữ liệu đa thiết bị.

Nhược điểm:

- Cần phát triển thêm backend.
- Cần vận hành bảo mật, chi phí, pháp lý, backup.

---

## 15. Kết luận

Để public nhanh cho mọi người chơi, hướng đi khuyến nghị là:

1. Build static bằng `pnpm build`.
2. Deploy thư mục `out/` lên static hosting hoặc chạy Docker Compose trên VPS.
3. Đặt `NEXT_PUBLIC_BASE_URL` đúng domain thật.
4. Không nhúng API key bí mật vào frontend.
5. Hướng dẫn mỗi người chơi nhập API key/model riêng và backup dữ liệu cục bộ.
6. Nếu muốn vận hành như dịch vụ nhiều người dùng thật sự, cần bổ sung backend/proxy/database riêng.
