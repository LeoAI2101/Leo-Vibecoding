# vibe-coding — "Thợ code AI" có rào chắn an toàn

**vibe-coding** là một công cụ dòng lệnh (viết bằng TypeScript, dùng Claude API) giúp bạn **giao việc
lập trình cho AI bằng một câu tiếng thường**, ví dụ:

> "thêm endpoint /health trả về 200"

Công cụ sẽ tự đọc code, tự sửa file, tự chạy kiểm tra (lint + test), tự nhờ một AI thứ hai soi lỗi,
và chỉ **commit** vào git khi mọi thứ đều đạt. Nó **không bao giờ tự push** lên GitHub — bạn luôn là
người xem lại và quyết định cuối cùng.

Điểm khác biệt so với việc chat với AI rồi copy code: **các rào chắn an toàn do chương trình ép buộc,
không phụ thuộc vào việc AI có "ngoan" hay không.** AI có muốn sửa file cấm, chạy lệnh lạ hay bỏ qua
test cũng không được.

Tài liệu thiết kế gốc (tiếng Anh):
[`docs/specs/2026-08-26-agent-coding-engine-design.md`](docs/specs/2026-08-26-agent-coding-engine-design.md)

---

## Mục lục

1. [Công cụ này làm được gì](#1-công-cụ-này-làm-được-gì)
2. [Cách hoạt động — quy trình 4 bước](#2-cách-hoạt-động--quy-trình-4-bước)
3. [Các rào chắn an toàn](#3-các-rào-chắn-an-toàn)
4. [Cài đặt](#4-cài-đặt)
5. [Cấu hình cho dự án muốn giao việc](#5-cấu-hình-cho-dự-án-muốn-giao-việc)
6. [Cách sử dụng (chạy trên máy)](#6-cách-sử-dụng-chạy-trên-máy)
7. [Chế độ tự động trên GitHub (/vibe-code)](#7-chế-độ-tự-động-trên-github-vibe-code)
8. [Thư mục claude-skills](#8-thư-mục-claude-skills)
9. [Cấu trúc mã nguồn](#9-cấu-trúc-mã-nguồn)
10. [Lỗi thường gặp](#10-lỗi-thường-gặp)
11. [Cảnh báo quan trọng](#11-cảnh-báo-quan-trọng)

---

## 1. Công cụ này làm được gì

| Bạn làm | Công cụ làm |
|---|---|
| Viết một câu mô tả việc cần làm | Claude đọc code trong dự án, sửa/tạo file cần thiết |
| — | Tự chạy lệnh lint và test của dự án |
| — | Nếu lỗi: đưa lỗi cho Claude sửa lại (tối đa N lần) |
| — | Nhờ một lượt Claude **độc lập khác** đóng vai "người review khó tính" soi thay đổi |
| — | Nếu review thấy lỗi nghiêm trọng: quay lại sửa |
| — | Khi tất cả đạt: tạo **1 commit** trong git của dự án |
| Xem lại commit, rồi tự `git push` | — |

Có hai cách dùng:

- **Chế độ chạy trên máy (CLI)** — bạn gõ lệnh trên máy mình, trỏ vào một dự án bất kỳ.
- **Chế độ GitHub (CI)** — bạn viết bình luận `/vibe-code <việc cần làm>` trong một Issue trên GitHub,
  GitHub Actions sẽ tự chạy và mở Pull Request.

---

## 2. Cách hoạt động — quy trình 4 bước

```
  Câu mô tả việc
        │
        ▼
 ┌──────────────┐   Claude dùng 2 công cụ: sửa file + chạy lệnh bash (có giới hạn)
 │ 1. IMPLEMENT │   Tối đa 50 lượt gọi công cụ mỗi lần, tránh AI chạy vô tận
 └──────┬───────┘
        ▼
 ┌──────────────┐   Chương trình (không phải AI) tự chạy lintCommand rồi testCommand
 │ 2. VERIFY    │── lỗi ──► gửi log lỗi cho Claude ──► quay lại bước 1
 └──────┬───────┘
        ▼
 ┌──────────────┐   Một lượt Claude mới, KHÔNG biết bước 1 đã nghĩ gì, chỉ xem diff
 │ 3. REVIEW    │── lỗi "blocking" ──► gửi danh sách lỗi ──► quay lại bước 1
 └──────┬───────┘   (lỗi "minor" chỉ mang tính góp ý, không bắt sửa)
        ▼
 ┌──────────────┐   Chương trình tự chạy git add + git commit
 │ 4. COMMIT    │   Từ chối nếu có file nằm trong danh sách cấm
 └──────────────┘
```

Số lần quay lại tối đa do `maxRetries` quyết định (mặc định 3). Kết thúc sẽ có 1 trong 3 kết quả:

| Thông báo | Ý nghĩa |
|---|---|
| `done: committed.` | Thành công, đã có commit mới trong dự án |
| `done: no changes were needed.` | AI thấy không cần sửa gì |
| `gave up: <lý do>` | Bỏ cuộc (hết lượt thử, sửa file cấm, diff quá lớn…) — không có gì được commit |

---

## 3. Các rào chắn an toàn

Đây là phần quan trọng nhất của dự án. Tất cả được **chương trình ép buộc**, AI không thể tự gỡ:

1. **Danh sách file cấm (`denylist`)** — AI không được đọc/sửa các file khớp mẫu này (ví dụ `.env*`,
   `prisma/migrations/**`). Luôn tự động cấm thêm: `agent.config.json`, file quy ước (`AGENTS.md`)
   và `.git/**` — để AI không thể tự nới lỏng luật cho lần chạy sau.
2. **Chặn thoát khỏi thư mục dự án** — đường dẫn kiểu `../../` hay symlink trỏ ra ngoài đều bị chặn
   (`src/guard.ts`).
3. **Danh sách lệnh được phép (`bashAllowlist`)** — AI chỉ chạy được các lệnh bạn liệt kê (ví dụ `git`,
   `npm`, `ls`); chặn các kiểu ghép lệnh để lách luật (`;`, `&&`, `$(...)`…).
4. **Chương trình tự chạy test** — lint/test không phải công cụ của AI, nên AI không thể "báo cáo láo"
   là test đã qua.
5. **Review độc lập** — người review là một lượt gọi Claude riêng, chỉ thấy câu mô tả việc + diff.
   Diff quá lớn (trên 400.000 ký tự) bị từ chối thay vì cắt bớt, để không duyệt thứ chưa xem hết.
6. **Kiểm tra trước khi gửi diff đi** — nếu có file cấm bị thay đổi, quy trình dừng ngay, nội dung file
   đó không bao giờ được gửi lên API.
7. **Ẩn biến môi trường nhạy cảm** — các biến có tên chứa `TOKEN`, `SECRET`, `PASSWORD`, `API_KEY`
   bị xoá khỏi môi trường của các lệnh con.
8. **Không tự push** — ở chế độ trên máy, mọi thứ dừng ở commit cục bộ.

---

## 4. Cài đặt

### Yêu cầu

- **Node.js 22** trở lên (kiểm tra: `node -v`)
- **git**
- **API key của Anthropic** (lấy tại https://console.anthropic.com → API Keys)

### Các bước

```bash
git clone https://github.com/LeoAI2101/Leo-Vibecoding.git
cd Leo-Vibecoding
npm install
```

### Đặt API key

Chương trình đọc key từ **biến môi trường** `ANTHROPIC_API_KEY` (nó **không** tự đọc file `.env`).
Chọn một trong hai cách:

**Cách 1 — Đặt vĩnh viễn (khuyên dùng trên Windows):**

```bash
setx ANTHROPIC_API_KEY "sk-ant-..."
```

Sau đó **đóng terminal và mở lại** để biến có hiệu lực. Từ giờ không cần đặt lại nữa.

**Cách 2 — Đặt tạm cho cửa sổ terminal hiện tại** (mất khi đóng cửa sổ):

- PowerShell: `$env:ANTHROPIC_API_KEY = "sk-ant-..."`
- macOS/Linux (bash): `export ANTHROPIC_API_KEY=sk-ant-...`

Kiểm tra đã đặt chưa (PowerShell): `echo $env:ANTHROPIC_API_KEY`

> File `.env.example` chỉ là mẫu ghi nhớ tên biến. Nếu bạn tạo file `.env` để lưu key cho tiện thì
> file đó đã nằm trong `.gitignore`, không bị đẩy lên GitHub. **Tuyệt đối không** dán key vào bất kỳ
> file nào khác, nhất là `README.md` hay code.

Kiểm tra cài đặt thành công:

```bash
npm test
```

Thấy toàn bộ test báo xanh (passed) là được.

---

## 5. Cấu hình cho dự án muốn giao việc

"Dự án muốn giao việc" (gọi là **dự án đích**) là dự án mà bạn muốn AI sửa code — **không phải** thư
mục vibe-coding này. Dự án đích cần:

1. **Là một git repo** và đã có ít nhất 1 commit.
2. **Có file `agent.config.json` ở thư mục gốc.** Copy từ `agent.config.example.json` rồi chỉnh:

```json
{
  "denylist": ["prisma/migrations/**", ".env*", "lib/authz.ts", "node_modules/**", ".git/**"],
  "bashAllowlist": ["git", "npm", "node", "ls", "cat", "grep"],
  "lintCommand": "npm run lint",
  "testCommand": "npm test",
  "agentsFile": "AGENTS.md",
  "maxRetries": 3
}
```

| Trường | Ý nghĩa | Gợi ý |
|---|---|---|
| `denylist` | Các file/thư mục AI **không được đụng** (cú pháp glob, `**` = mọi thư mục con) | Thêm file bí mật, migration DB, file phân quyền |
| `bashAllowlist` | Các lệnh AI **được phép chạy** | Càng ít càng an toàn |
| `lintCommand` | Lệnh kiểm tra cú pháp/kiểu | `npm run lint`, `npm run build`, `npx tsc --noEmit` |
| `testCommand` | Lệnh chạy test | `npm test`, `pytest`… |
| `agentsFile` | File mô tả quy ước code của dự án, được đưa cho AI đọc trước khi làm | `AGENTS.md` hoặc `CLAUDE.md` |
| `maxRetries` | Số lần tối đa được sửa lại khi test/review fail | 3 là hợp lý |

> ⚠️ `lintCommand` và `testCommand` **phải chạy được và đang pass** trên dự án trước khi giao việc.
> Nếu test vốn đã hỏng sẵn, AI sẽ cố sửa đến hết lượt rồi bỏ cuộc.
> Nếu dự án chưa có lint/test, có thể tạm đặt một lệnh luôn thành công, ví dụ `"node -v"` — nhưng
> khi đó bạn mất đi lớp bảo vệ quan trọng nhất.

3. **(Nên có) File `AGENTS.md`** — viết bằng lời thường về dự án: dùng công nghệ gì, quy ước đặt tên,
   cấu trúc thư mục, điều gì không được làm. AI đọc file này trước mỗi lần làm việc, viết càng rõ
   kết quả càng tốt. Xem `AGENTS.md` của chính repo này làm mẫu.

---

## 6. Cách sử dụng (chạy trên máy)

Đứng trong thư mục vibe-coding, chạy:

```bash
npm run cli -- --repo <đường-dẫn-dự-án-đích> --task "<mô tả việc cần làm>"
```

Ví dụ trên Windows:

```bash
npm run cli -- --repo "C:\Users\Admin\Projects\my-shop" --task "thêm endpoint GET /health trả về JSON {status: 'ok'} và viết test cho nó"
```

> Dấu `--` sau `npm run cli` là bắt buộc, để npm chuyển các tham số phía sau cho chương trình.

### Sau khi chạy xong

Vào thư mục dự án đích để xem AI đã làm gì:

```bash
git log -1 --stat
```

```bash
git show
```

- Hài lòng → `git push` như bình thường.
- Không hài lòng → huỷ commit vừa tạo (giữ lại thay đổi để sửa tay): `git reset --soft HEAD~1`,
  hoặc bỏ hẳn: `git reset --hard HEAD~1`.

### Mẹo viết câu giao việc hiệu quả

- **Cụ thể:** nói rõ file/chức năng/kết quả mong muốn.
  ✅ "Trong `src/cart.ts`, thêm hàm `applyCoupon(code)` giảm 10% khi mã là SALE10, kèm test"
  ❌ "làm chức năng giảm giá"
- **Mỗi lần một việc nhỏ.** Việc lớn hãy chia thành nhiều lần chạy.
- **Yêu cầu viết test** — vừa giúp AI tự kiểm tra, vừa giúp bạn tin tưởng kết quả.
- Có thể viết tiếng Việt; dòng đầu tiên của câu giao việc sẽ thành tiêu đề commit (tối đa 72 ký tự).

### Chi phí

Mỗi lần chạy gọi Claude nhiều lượt (bước implement tối đa 50 lượt × số lần thử lại, cộng lượt
review), tính tiền theo token trên tài khoản Anthropic của bạn. Hãy theo dõi mức dùng tại
console.anthropic.com, nhất là khi giao việc lớn. Model đang dùng được khai báo ở hằng `MODEL`
trong `src/implement.ts` và `src/review.ts`.

---

## 7. Chế độ tự động trên GitHub (/vibe-code)

Thay vì chạy trên máy, bạn có thể giao việc ngay trên GitHub:

1. Mở một **Issue** trong repo đã cài workflow.
2. Viết bình luận: `/vibe-code thêm nút đăng xuất vào header`
3. GitHub Actions tự chạy đúng quy trình 4 bước ở trên, rồi **đẩy một nhánh mới và mở Pull Request**
   liên kết với Issue đó. Bạn review PR rồi merge.

Chỉ người có vai trò **OWNER / MEMBER / COLLABORATOR** của repo mới kích hoạt được — người lạ bình
luận trên repo công khai sẽ bị bỏ qua (kiểm tra trong `src/trigger.ts`).

### Cài đặt chế độ này

1. Vào repo trên GitHub → **Settings → Secrets and variables → Actions → New repository secret**:
   tên `ANTHROPIC_API_KEY`, giá trị là API key của bạn.
2. Repo cần có file `.github/workflows/vibe-code.yml` (repo này đã có sẵn) và `agent.config.json`.
3. Vào **Settings → Actions → General**, chọn **Read and write permissions** và tích
   **Allow GitHub Actions to create and approve pull requests** để job tạo được PR.
4. Để dùng cho repo khác: copy file workflow + tạo `agent.config.json` riêng cho repo đó, và đưa mã
   nguồn vibe-coding vào (cài như package hoặc copy thư mục `src/`) để lệnh `npm run ci` chạy được.

### Vì sao chế độ GitHub an toàn hơn

Workflow tách làm 2 job:

- **`agent`** — chạy AI và code do AI viết, nhưng chỉ có quyền **đọc** repo và **không có token
  GitHub**. API key được đưa vào dưới dạng file và bị xoá trước khi chạy bất kỳ lệnh nào. Chương trình
  từ chối chạy nếu tiến trình con có thể đọc bộ nhớ của nó hoặc leo lên quyền root (Docker bị tắt,
  `sudo` bị vô hiệu — xem `isolationProblems` trong `src/env.ts`).
- **`publish`** — giữ quyền ghi để push và mở PR, nhưng **không chạy code nào do AI viết**. Nó kiểm
  tra lại bản vá theo danh sách cấm của commit gốc (`src/check-commit.ts`) trước khi push, vì kết quả
  từ job `agent` được coi là không đáng tin.

---

## 8. Thư mục `claude-skills`

Đây là bộ **skills / agents cá nhân cho Claude Code** (hơn 80 skill: review code, bảo mật, UI/UX,
deploy, quy trình làm việc…). Nó **không bắt buộc** để chạy vibe-coding, mà là bộ quy ước làm việc
đi kèm. Hướng dẫn cài chi tiết (tiếng Việt) nằm ở [`claude-skills/README.md`](claude-skills/README.md).
Tóm tắt:

```bash
cp -r claude-skills ~/.claude/skills
bash ~/.claude/skills/_bootstrap/install.sh
```

Sau đó khởi động lại Claude Code. Kiểm tra đã đủ chưa: `bash ~/.claude/skills/_bootstrap/doctor.sh`.
(Trên Windows chạy các lệnh này trong Git Bash.)

---

## 9. Cấu trúc mã nguồn

| File | Vai trò |
|---|---|
| `src/cli.ts` | Điểm vào chế độ trên máy: đọc `--repo`, `--task`, chạy quy trình |
| `src/ci.ts` | Điểm vào chế độ GitHub Actions |
| `src/pipeline.ts` | Điều phối 4 bước implement → verify → review → commit, xử lý thử lại |
| `src/implement.ts` | Vòng lặp gọi Claude với công cụ sửa file + bash (tối đa 50 lượt) |
| `src/tools.ts` | Thực thi công cụ cho AI (đọc/sửa file, chạy lệnh) qua lớp kiểm tra |
| `src/guard.ts` | **Phần quan trọng nhất về bảo mật:** kiểm tra đường dẫn và lệnh bash |
| `src/verify.ts` | Chạy `lintCommand` và `testCommand` |
| `src/review.ts` | Lượt Claude độc lập review diff, trả về lỗi `blocking`/`minor` |
| `src/commit.ts` | `git add` + `git commit`, chặn file cấm |
| `src/check-commit.ts` | Kiểm tra lại bản vá trước khi push (chế độ GitHub) |
| `src/config.ts` | Đọc và kiểm tra `agent.config.json` |
| `src/env.ts` | Lọc biến môi trường nhạy cảm, kiểm tra cách ly tiến trình |
| `src/trigger.ts` | Phân tích bình luận `/vibe-code` và kiểm tra quyền người gửi |
| `src/truncate.ts` | Rút gọn log lỗi quá dài trước khi gửi cho AI |
| `*.test.ts` | Test tự động (vitest) cho từng module |

### Lệnh cho người phát triển

```bash
npm test         # chạy toàn bộ test
npm run build    # kiểm tra kiểu TypeScript + biên dịch
```

Quy ước đóng góp code xem trong [`AGENTS.md`](AGENTS.md). Đặc biệt: **không được nới lỏng
`src/guard.ts` mà không kèm test chứng minh vẫn an toàn** (xem `src/guard.test.ts`).

---

## 10. Lỗi thường gặp

| Thông báo | Nguyên nhân & cách xử lý |
|---|---|
| `usage: vibe-coding --repo <path> --task "<description>"` | Thiếu `--repo` hoặc `--task`, hoặc quên dấu `--` sau `npm run cli` |
| `agent.config.json not found at ...` | Dự án đích chưa có file cấu hình — xem mục 5 |
| Lỗi xác thực / `401` / `authentication_error` | Chưa đặt `ANTHROPIC_API_KEY` hoặc key sai |
| `gave up: exceeded maxRetries (3)` | AI sửa mãi không qua test/review. Kiểm tra test có pass sẵn không; chia nhỏ việc; viết câu giao việc rõ hơn |
| `gave up: implement exceeded 50 tool turns` | Việc quá lớn/mơ hồ — chia nhỏ |
| `gave up: denylisted files changed: ...` | AI đã đụng vào file cấm, quy trình dừng để bảo vệ. Không có gì được commit |
| `gave up: diff too large to review` | Thay đổi quá lớn để review an toàn — chia nhỏ việc |
| `Deletion of directory '.git/objects/..' failed` | Dropbox/OneDrive khoá file trong `.git`. Tạm dừng đồng bộ khi chạy git, hoặc chạy `git config gc.auto 0` |

---

## 11. Cảnh báo quan trọng

> ⚠️ **Ở chế độ chạy trên máy, code do AI viết chạy với quyền của chính bạn.** Nó có thể chạm tới
> file cá nhân, SSH key, thông tin đăng nhập `git`/`gh` trên máy. Danh sách lệnh cho phép và việc ẩn
> biến môi trường chỉ giảm rủi ro, **không phải hộp cát cách ly hoàn toàn**.
>
> Chỉ dùng cho dự án và yêu cầu mà bạn chấp nhận chạy code bất kỳ. Muốn an toàn hơn, dùng chế độ
> GitHub (mục 7), chạy trong máy ảo, hoặc container.

Luôn **đọc lại commit** trước khi push. AI có thể viết code chạy được, qua test, nhưng vẫn sai ý bạn.
