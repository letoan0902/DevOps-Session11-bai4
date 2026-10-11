# Bài 4 - Session 11: Khai báo GitHub Secrets và cấu hình Đăng nhập Container Registry

Repository: `letoan0902/DevOps-Session11-bai4` (public, nhánh `main`).

## Mục lục

1. Mục tiêu
2. Vì sao dùng GitHub Secrets
3. PAT classic và GITHUB_TOKEN tự động
4. Hướng dẫn tạo PAT và Repository Secret
5. Giải thích file ci.yml
6. Cách kiểm tra trên tab Actions
7. Kết quả `gh secret list` (thay cho ảnh chụp)
8. Lưu ý bảo mật
9. Xử lý sự cố
10. Đối chiếu yêu cầu đề bài
11. Trạng thái thực hiện
12. Kết luận

## 1. Mục tiêu

- Tạo một Personal Access Token (classic) có quyền `write:packages` trên GitHub.
- Lưu token đó thành Repository Secret tên `REGISTRY_TOKEN` trong Settings -> Secrets and variables -> Actions của repository.
- Trong workflow `ci.yml`, khai báo bước `docker/login-action@v3` truyền `secrets.REGISTRY_TOKEN` để đăng nhập tự động vào GitHub Container Registry (`ghcr.io`).
- Kiểm tra bằng cách đẩy code lên GitHub, mở log bước đăng nhập trên tab Actions và xác nhận dòng `Login Succeeded!`.

Sơ đồ tổng quan luồng xác thực:

```
  Học viên                GitHub                       Runner (ubuntu-latest)
  --------                ------                       ----------------------
  1. Tạo PAT classic ---> (write:packages)
  2. Dán PAT vào secret -> Repository Secrets
        REGISTRY_TOKEN        (mã hóa, chỉ ghi)
  3. git push ----------> Kích hoạt workflow ---------> Cấp runner, nạp ci.yml
                                                         |
                          secrets.REGISTRY_TOKEN ------> docker/login-action@v3
                          (được giải mã, bị che ***)     |
                                                         v
                                                  docker login ghcr.io
                                                         |
                                                         v
                                                  "Login Succeeded!"
```

## 2. Vì sao dùng GitHub Secrets

Token là thông tin xác thực có giá trị như mật khẩu. Nếu ghi trực tiếp vào `ci.yml` hoặc commit vào repository, bất kỳ ai xem được mã nguồn (đặc biệt repo public) đều lấy được token. GitHub Secrets giải quyết vấn đề này:

| Đặc điểm | Ý nghĩa |
|----------|---------|
| Mã hóa khi lưu | Giá trị secret được mã hóa (libsodium sealed box) trước khi đến GitHub; chỉ được giải mã trên runner khi job chạy |
| Chỉ ghi, không đọc lại | Sau khi tạo, giao diện và API không hiển thị lại giá trị; chỉ có thể ghi đè hoặc xóa |
| Bị che trong log | Nếu giá trị secret xuất hiện trong log, GitHub thay bằng `***` |
| Không nằm trong lịch sử Git | Mã nguồn chỉ chứa tham chiếu `${{ secrets.REGISTRY_TOKEN }}`, không chứa giá trị |
| Phạm vi truy cập | Secret của repository chỉ dùng được trong workflow của chính repo đó; workflow chạy từ PR của fork không nhận secret |

Nguyên tắc: mã nguồn chỉ biết tên secret, giá trị nằm ở GitHub và chỉ lộ ra trong bộ nhớ của runner lúc job chạy.

## 3. PAT classic và GITHUB_TOKEN tự động

Mỗi lần workflow chạy, GitHub tự sinh một `GITHUB_TOKEN` tạm thời. Đề bài yêu cầu dùng PAT classic lưu trong `REGISTRY_TOKEN`, hai loại này khác nhau như sau:

| Tiêu chí | PAT classic (`REGISTRY_TOKEN`) | `GITHUB_TOKEN` tự động |
|----------|-------------------------------|------------------------|
| Nguồn gốc | Học viên tự tạo thủ công | GitHub tự cấp cho mỗi lần chạy |
| Thời hạn | Theo cấu hình (7, 30, 60, 90 ngày hoặc không hết hạn) | Hết hiệu lực khi job kết thúc |
| Phạm vi | Gắn với tài khoản, dùng được cho mọi repo/package mà tài khoản có quyền | Chỉ trong repo chạy workflow, theo khối `permissions:` |
| Quyền tối thiểu cho GHCR | `write:packages` (kéo theo `read:packages`) | `packages: write` trong `permissions` |
| Dùng ngoài workflow | Có (máy cá nhân, hệ thống CI khác, đẩy image sang repo khác) | Không |
| Rủi ro | Cao hơn nếu lộ vì sống lâu và gắn với tài khoản | Thấp, tự hủy |
| Cần tạo secret | Có | Không (có sẵn `secrets.GITHUB_TOKEN`) |

Lý do đề bài dùng PAT: mục đích là luyện quy trình tạo token, khai báo secret và tham chiếu secret trong workflow, đây là quy trình áp dụng cho mọi loại khóa (Docker Hub, AWS, v.v.). PAT cũng cần thiết khi đẩy image vào package thuộc repo hoặc tổ chức khác, hoặc khi package đã tồn tại và chưa cấp quyền ghi cho repo.

Trong thực tế, nếu chỉ đẩy image lên GHCR trong cùng repo thì nên ưu tiên `GITHUB_TOKEN` vì không cần quản lý token dài hạn.

## 4. Hướng dẫn tạo PAT và Repository Secret

Phần này do học viên tự thực hiện trên giao diện GitHub; token không được gửi qua chat, không ghi vào file nào trong repo.

### 4.1. Tạo Personal Access Token (classic)

1. Đăng nhập GitHub, bấm ảnh đại diện góc trên bên phải, chọn Settings.
2. Kéo xuống cuối thanh bên trái, chọn Developer settings.
3. Chọn Personal access tokens, sau đó chọn Tokens (classic).
4. Bấm Generate new token, chọn Generate new token (classic). GitHub có thể yêu cầu xác nhận mật khẩu hoặc mã 2FA.
5. Điền các trường:
   - Note: `REGISTRY_TOKEN - DevOps Session11 bai4`
   - Expiration: 30 days
   - Select scopes: tick `write:packages`. GitHub tự tick kèm `read:packages`. Không chọn thêm quyền khác (đặc biệt không chọn `repo` nếu không cần).
6. Cuộn xuống cuối, bấm Generate token.
7. Copy ngay token hiển thị (có dạng `ghp_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx`). GitHub chỉ hiển thị token một lần duy nhất; nếu lỡ đóng trang phải tạo lại.

### 4.2. Tạo Repository Secret `REGISTRY_TOKEN` bằng giao diện

1. Mở repository `letoan0902/DevOps-Session11-bai4` trên GitHub.
2. Chọn tab Settings.
3. Ở thanh bên trái, mở Secrets and variables, chọn Actions.
4. Ở thẻ Secrets, mục Repository secrets, bấm New repository secret.
5. Name: `REGISTRY_TOKEN` (viết hoa đúng từng ký tự như trong `ci.yml`).
6. Secret: dán token vừa copy.
7. Bấm Add secret. Danh sách Repository secrets sẽ hiển thị `REGISTRY_TOKEN` cùng thời điểm cập nhật; giá trị không bao giờ hiển thị lại.

### 4.3. Phương án dòng lệnh với GitHub CLI

```
gh auth login
gh secret set REGISTRY_TOKEN --repo letoan0902/DevOps-Session11-bai4
```

`gh` sẽ hiện lời nhắc nhập giá trị (ẩn ký tự khi dán). Cách này không đưa token vào đối số dòng lệnh nên không lưu trong lịch sử shell. Không dùng dạng `--body "ghp_..."` vì token sẽ nằm trong lịch sử shell.

Kiểm tra secret đã tồn tại:

```
gh secret list --repo letoan0902/DevOps-Session11-bai4
```

### 4.4. Chạy lại workflow sau khi có secret

Lần chạy đầu tiên (khi push code, lúc chưa có secret) sẽ thất bại ở bước đăng nhập. Sau khi thêm secret, chạy lại bằng một trong hai cách:

- Giao diện: tab Actions, chọn lần chạy bị lỗi, bấm Re-run all jobs.
- Dòng lệnh: `gh workflow run ci.yml --repo letoan0902/DevOps-Session11-bai4` (nhờ có `workflow_dispatch`).

## 5. Giải thích file ci.yml

File workflow thật nằm ở `.github/workflows/ci.yml` (gốc repo, để GitHub Actions nhận diện). Bản sao đúng từng byte nằm ở `homework/session_11/ex4/ci.yml` theo đường dẫn nộp bài.

### 5.1. Phần đầu: tên, điều kiện kích hoạt, quyền

```yaml
name: CI - Dang nhap GitHub Container Registry

on:
  push:
    branches:
      - main
  pull_request:
  workflow_dispatch:

permissions:
  contents: read
  packages: write
```

| Khóa | Giải thích |
|------|------------|
| `name` | Tên hiển thị của workflow trên tab Actions |
| `on.push.branches: main` | Chạy mỗi khi có commit đẩy lên nhánh `main` |
| `on.pull_request` | Chạy khi mở hoặc cập nhật Pull Request. PR từ fork không nhận secret nên bước đăng nhập sẽ lỗi, đây là hành vi bảo mật có chủ đích của GitHub |
| `on.workflow_dispatch` | Cho phép chạy thủ công, dùng để chạy lại sau khi thêm secret |
| `permissions` | Giới hạn quyền của `GITHUB_TOKEN` tự động: chỉ đọc nội dung repo và ghi packages. Khi khai báo `permissions`, mọi quyền không liệt kê đều bị đặt về `none` |

### 5.2. Job `login-registry`

```yaml
jobs:
  login-registry:
    name: Đăng nhập Registry
    runs-on: ubuntu-latest
```

Một job duy nhất chạy trên runner `ubuntu-latest`, đã cài sẵn Docker, `jq` và `git`.

### 5.3. Các bước

| STT | Tên bước | Lệnh/Action | Tác dụng |
|-----|----------|-------------|----------|
| 1 | Checkout mã nguồn | `actions/checkout@v5` | Tải mã nguồn về runner |
| 2 | Kiểm tra Docker trên runner | `docker version` | Xác nhận Docker client và daemon sẵn sàng trước khi đăng nhập |
| 3 | Đăng nhập vào GitHub Container Registry | `docker/login-action@v3` | Thực hiện `docker login ghcr.io` bằng tài khoản và token |
| 4 | Xác nhận đã đăng nhập | `echo` và `jq` | In tên tài khoản và danh sách registry có trong `~/.docker/config.json` |
| 5 | Đăng xuất khỏi GitHub Container Registry | `docker logout ghcr.io` với `if: always()` | Xóa thông tin xác thực khỏi runner |

### 5.4. Bước đăng nhập (trọng tâm của bài)

```yaml
      - name: Đăng nhập vào GitHub Container Registry
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.REGISTRY_TOKEN }}
```

| Tham số | Giá trị | Ý nghĩa |
|---------|---------|---------|
| `registry` | `ghcr.io` | Địa chỉ GitHub Container Registry. Nếu bỏ trống, action mặc định đăng nhập Docker Hub |
| `username` | `${{ github.actor }}` | Ngữ cảnh GitHub trả về tên tài khoản kích hoạt workflow, không cần ghi cứng |
| `password` | `${{ secrets.REGISTRY_TOKEN }}` | Tham chiếu secret đã tạo ở mục 4; GitHub thay bằng giá trị thật khi chạy và che bằng `***` trong log |

Lưu ý quan trọng: `github.actor` là người kích hoạt lần chạy, còn token là của chủ sở hữu PAT. Khi học viên tự push và tự tạo PAT thì hai tài khoản trùng nhau nên đăng nhập hợp lệ. Nếu người khác push (hoặc bot như Dependabot), cặp username/token sẽ không khớp và có thể bị từ chối; khi đó nên đặt `username` cố định bằng tên chủ PAT.

### 5.5. Bước xác nhận và đăng xuất

```yaml
      - name: Xác nhận đã đăng nhập
        run: |
          echo "Da dang nhap ghcr.io voi tai khoan ${{ github.actor }}"
          jq '.auths | keys' ~/.docker/config.json

      - name: Đăng xuất khỏi GitHub Container Registry
        if: always()
        run: docker logout ghcr.io
```

- Lệnh `jq '.auths | keys'` chỉ in tên các registry (khóa), không in giá trị `auth` mã hóa base64 chứa thông tin đăng nhập. Kết quả mong đợi có `"ghcr.io"`.
- `if: always()` khiến bước đăng xuất luôn chạy, kể cả khi bước đăng nhập thất bại (khi đó `docker logout` chỉ báo chưa đăng nhập, không làm hỏng job).

### 5.6. Kiểm tra cú pháp

Workflow được kiểm tra bằng actionlint v1.7.12 qua Docker:

```
docker run --rm -v "$PWD:/repo" -w /repo rhysd/actionlint:latest -no-color .github/workflows/ci.yml
```

Kết quả rỗng và mã thoát 0, nghĩa là không có lỗi cú pháp, không sai tên action hay ngữ cảnh. Chi tiết trong `actionlint_output.txt`. File dùng thụt lề 2 dấu cách, không chứa ký tự Tab, xuống dòng LF.

## 6. Cách kiểm tra trên tab Actions

1. Đẩy code lên nhánh `main` của repository (hoặc chạy thủ công bằng `workflow_dispatch`).
2. Mở repository trên GitHub, chọn tab Actions.
3. Chọn lần chạy mới nhất của workflow `CI - Dang nhap GitHub Container Registry`.
4. Bấm vào job `Đăng nhập Registry`.
5. Mở rộng bước `Đăng nhập vào GitHub Container Registry` (phần log nằm dưới dòng `Run docker/login-action@v3`).
6. Xác nhận log có dạng:

```
Run docker/login-action@v3
  with:
    registry: ghcr.io
    username: <ten-tai-khoan>
    password: ***
    logout: true
Logging into ghcr.io...
Login Succeeded!
```

Điểm cần chú ý khi đọc log:

- Dòng `password: ***` chứng tỏ giá trị secret đã được che.
- Dòng `Logging into ghcr.io...` cho biết action đang gọi `docker login` tới đúng registry.
- Dòng `Login Succeeded!` là tiêu chí đạt của bài.
- Bước `Xác nhận đã đăng nhập` in `Da dang nhap ghcr.io voi tai khoan <ten-tai-khoan>` và danh sách có `"ghcr.io"`.
- Bước `Đăng xuất` ở cuối job (và bước post của login-action) thực hiện đăng xuất để dọn thông tin xác thực.

Lần chạy khi chưa có secret sẽ lỗi, log có dạng:

```
Error: Username and password required
```

Đây là hành vi đúng của `docker/login-action@v3` khi `password` rỗng (secret chưa tồn tại thì `${{ secrets.REGISTRY_TOKEN }}` trả về chuỗi rỗng, không báo lỗi riêng).

## 7. Kết quả `gh secret list` (thay cho ảnh chụp)

Đề bài yêu cầu ảnh chụp mục Repository Secrets hiển thị `REGISTRY_TOKEN`. Vị trí trên giao diện: repository -> Settings -> Secrets and variables -> Actions -> thẻ Secrets -> mục Repository secrets, dòng `REGISTRY_TOKEN` kèm thời điểm cập nhật và các nút sửa/xóa (không có nút xem giá trị).

Thay cho ảnh chụp, học viên chạy lệnh sau sau khi đã tạo secret:

```
gh secret list --repo letoan0902/DevOps-Session11-bai4
```

Output mẫu (kết quả điển hình sau khi học viên đã tạo secret; thời điểm cập nhật thực tế sẽ khác):

```
NAME            UPDATED
REGISTRY_TOKEN  about 5 minutes ago
```

Cột NAME phải có `REGISTRY_TOKEN`; cột UPDATED là thời điểm tạo hoặc ghi đè gần nhất. Lệnh này không bao giờ hiển thị giá trị secret.

## 8. Lưu ý bảo mật

| Hạng mục | Khuyến nghị |
|----------|-------------|
| Hạn dùng token | Đặt hạn ngắn (ví dụ 30 ngày), ghi lịch gia hạn; token hết hạn thì bước đăng nhập báo lỗi xác thực |
| Quyền tối thiểu | Chỉ chọn `write:packages` (kèm `read:packages`); không chọn `repo`, `admin:*`, `delete:packages` nếu không cần |
| Thu hồi khi lộ | Vào Settings -> Developer settings -> Personal access tokens -> Tokens (classic), bấm Delete để thu hồi ngay, rồi tạo token mới và ghi đè secret |
| Không commit token | Không ghi `ghp_...` vào mã nguồn, README, file cấu hình hay ảnh chụp; kiểm tra bằng `git grep ghp_` trước khi push |
| Không in secret | Không `echo ${{ secrets.REGISTRY_TOKEN }}`; che `***` chỉ hoạt động với đúng giá trị nguyên văn, nếu bị biến đổi (base64, cắt chuỗi) thì có thể lộ |
| Dùng `secrets`, không dùng `vars` | Biến cấu hình (Variables, ngữ cảnh `vars`) hiển thị công khai và không bị che trong log; mọi thông tin nhạy cảm phải đặt trong Secrets (ngữ cảnh `secrets`) |
| Workflow từ PR | PR từ fork không nhận secret; không dùng `pull_request_target` để chạy mã không tin cậy cùng với secret |
| Đăng xuất cuối job | Giữ bước `docker logout` để không để lại thông tin xác thực trên runner |
| Ghim phiên bản action | Dùng tag chính (`@v3`, `@v5`); môi trường yêu cầu cao có thể ghim theo commit SHA |
| Nhập token khi dùng CLI | Dùng `gh secret set` ở chế độ nhập tương tác, không truyền token qua đối số |

## 9. Xử lý sự cố

| Triệu chứng | Nguyên nhân | Cách khắc phục |
|-------------|-------------|----------------|
| `Error: Username and password required` | Secret `REGISTRY_TOKEN` chưa tồn tại, tên sai, hoặc workflow chạy từ PR của fork nên không nhận secret | Tạo secret đúng tên theo mục 4, rồi Re-run jobs; với PR từ fork, chạy lại workflow trên repo gốc |
| `Error response from daemon: Get "https://ghcr.io/v2/": denied` hoặc `unauthorized` | Token thiếu quyền `write:packages`, `username` không khớp chủ token, hoặc token bị thu hồi | Tạo PAT mới với `write:packages`, đặt `username` là tên chủ PAT, cập nhật secret |
| Trước đó đăng nhập thành công, nay lỗi `denied` hoặc `401` | PAT đã hết hạn | Tạo PAT mới, ghi đè secret `REGISTRY_TOKEN` bằng `gh secret set REGISTRY_TOKEN --repo letoan0902/DevOps-Session11-bai4` |
| Secret có nhưng log vẫn báo thiếu mật khẩu | Sai tên trong `ci.yml` (ví dụ `REGISTRY_TOKENS`) hoặc secret được tạo ở mức Environment/Organization không gắn với job | Đối chiếu tên trong `ci.yml` với kết quả `gh secret list`; tạo secret ở mức Repository |
| Không thấy workflow trên tab Actions | File đặt sai đường dẫn (phải là `.github/workflows/ci.yml` ở gốc repo, đúng tên thư mục `.github`) hoặc Actions bị tắt trong Settings -> Actions | Sửa đường dẫn rồi push lại; bật Actions trong Settings |
| Workflow không chạy khi push | Đẩy lên nhánh khác `main`, hoặc lỗi cú pháp YAML (thụt lề sai, dùng Tab) | Đẩy lên `main` hoặc chạy `workflow_dispatch`; chạy actionlint để tìm lỗi cú pháp |
| `jq: command not found` ở bước xác nhận | Runner tự lưu trữ không có `jq` | Cài `jq` hoặc thay bằng `echo` đơn giản; `ubuntu-latest` của GitHub đã có sẵn |
| `Resource not accessible by integration` | Thiếu `permissions` cho `GITHUB_TOKEN` | Kiểm tra khối `permissions` có `packages: write` |
| `gh secret set` báo chưa xác thực | Chưa chạy `gh auth login` hoặc thiếu quyền ghi secret trên repo | Chạy `gh auth login`, dùng tài khoản có quyền quản trị repo |

## 10. Đối chiếu yêu cầu đề bài

| Yêu cầu | Cách đáp ứng | Vị trí |
|---------|--------------|--------|
| Tạo PAT classic với quyền `write:packages` | Hướng dẫn từng bước, học viên tự tạo | Mục 4.1 |
| Tạo secret `REGISTRY_TOKEN` ở Settings -> Secrets and variables -> Actions | Hướng dẫn qua giao diện và `gh secret set` | Mục 4.2, 4.3 |
| `ci.yml` có bước `docker/login-action@v3` truyền `secrets.REGISTRY_TOKEN` | Bước `Đăng nhập vào GitHub Container Registry` với `password: ${{ secrets.REGISTRY_TOKEN }}` | `.github/workflows/ci.yml`, mục 5.4 |
| Dùng `registry: ghcr.io`, `username: ${{ github.actor }}` | Đúng theo gợi ý | Mục 5.4 |
| Đẩy code, mở log bước đăng nhập, xác nhận `Login Succeeded` | Hướng dẫn kiểm tra và log mẫu | Mục 6, 11 |
| Nộp `.github/workflows/ci.yml` | Có, đồng thời bản sao ở `homework/session_11/ex4/ci.yml` | Thư mục bài nộp |
| Nộp ảnh chụp Repository Secrets hiển thị `REGISTRY_TOKEN` | Thay bằng mô tả vị trí và kết quả `gh secret list` | Mục 7 |
| Đường dẫn gợi ý `homework/session_11/ex4/` | Báo cáo, `ci.yml`, kết quả actionlint đặt tại đây | `homework/session_11/ex4/` |
| Kiểm tra cú pháp workflow | actionlint v1.7.12, không có lỗi | `actionlint_output.txt` |

## 11. Trạng thái thực hiện

- Workflow `ci.yml` đã được viết, kiểm tra bằng actionlint (không lỗi) và được đẩy lên repository `letoan0902/DevOps-Session11-bai4` cùng toàn bộ nội dung bài nộp.
- Personal Access Token và secret `REGISTRY_TOKEN` do học viên tự tạo theo hướng dẫn ở mục 4, vì token là thông tin cá nhân không được chia sẻ hay lưu trong repo.
- Lần chạy đầu tiên khi chưa có secret dự kiến thất bại ở bước đăng nhập với thông báo `Error: Username and password required`. Sau khi học viên tạo secret và chạy lại (Re-run jobs hoặc `gh workflow run ci.yml`), bước đăng nhập cho kết quả `Logging into ghcr.io...` và `Login Succeeded!` như mô tả ở mục 6.
- Kết quả thực tế của từng lần chạy được ghi ở dòng "Trạng thái CI" trong `README.md` gốc của repository.

## 12. Kết luận

Bài tập hoàn thiện chuỗi cơ bản của việc xác thực CI với registry: tạo thông tin xác thực có phạm vi tối thiểu (PAT classic với `write:packages`), cất giữ an toàn trong Repository Secret, rồi tham chiếu gián tiếp qua `${{ secrets.REGISTRY_TOKEN }}` trong `docker/login-action@v3`. Cách làm này giữ token ngoài mã nguồn, bị che trong log và có thể thu hồi hay thay thế mà không phải sửa workflow. Đây là nền tảng để các bài sau thực hiện build và push image lên GHCR; khi chỉ đẩy image trong cùng repository, có thể thay PAT bằng `GITHUB_TOKEN` tự động để giảm rủi ro quản lý token dài hạn.
