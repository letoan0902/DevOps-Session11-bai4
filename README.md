# DevOps Session 11 - Bài 4: GitHub Secrets và đăng nhập GitHub Container Registry

Repository này chứa bài tập số 4 của Session 11 môn DevOps: khai báo Repository Secret `REGISTRY_TOKEN` (Personal Access Token classic, quyền `write:packages`) và cấu hình workflow GitHub Actions đăng nhập tự động vào GitHub Container Registry (`ghcr.io`) bằng `docker/login-action@v3`.

Trạng thái CI: run đầu tiên (ID 38175497465, push lên `main`) thất bại ở bước "Đăng nhập vào GitHub Container Registry" với thông báo `Error: Password required`, vì repository chưa có secret `REGISTRY_TOKEN`. Các bước checkout và kiểm tra Docker đều xanh. Sau khi học viên tạo secret theo hướng dẫn trong báo cáo và Re-run jobs, bước đăng nhập sẽ in `Login Succeeded!`. Xem tab Actions: https://github.com/letoan0902/DevOps-Session11-bai4/actions

## Tài liệu

- Báo cáo chi tiết, hướng dẫn tạo PAT và secret: [homework/session_11/ex4/README.md](homework/session_11/ex4/README.md)
- Workflow thật (GitHub Actions chạy file này): [.github/workflows/ci.yml](.github/workflows/ci.yml)

## Danh sách file

| Đường dẫn | Mô tả |
|-----------|-------|
| `.github/workflows/ci.yml` | Workflow CI đăng nhập GHCR, nằm ở gốc repo để GitHub Actions nhận diện |
| `homework/session_11/ex4/ci.yml` | Bản sao đúng từng byte của workflow theo đường dẫn nộp bài |
| `homework/session_11/ex4/README.md` | Báo cáo bài làm |
| `homework/session_11/ex4/actionlint_output.txt` | Kết quả kiểm tra cú pháp bằng actionlint |
| `README.md` | Tệp giới thiệu này |

## Lưu ý

Token không được lưu trong repository. Secret `REGISTRY_TOKEN` do chủ repository tự tạo theo hướng dẫn trong báo cáo; khi chưa có secret, bước đăng nhập sẽ báo `Error: Username and password required`.
