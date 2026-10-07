# Hướng dẫn Bước 2 trên Oracle Cloud (OCI)

Thay thế các lệnh GCP (`gsutil`, `gcloud`) trong `tasks/buoc-2.md`. Object Storage của OCI có API
tương thích S3, nên DVC dùng remote `s3` và mã Python dùng `boto3` với `endpoint_url`.

Các lệnh chạy trên máy bạn viết cho **terminal PowerShell của VS Code** (venv `.venv` đã bật, prompt có
`(.venv)`). Các lệnh trong phiên SSH vào VM là Bash của Ubuntu.

Giá trị của bạn (không bí mật):

| Ký hiệu | Giá trị |
|---|---|
| `<BUCKET>` | `lab21-cicd-for-ai` |
| `<REGION>` | `ap-singapore-1` |
| `<NS>` | `axwfixabzvp5` |
| Endpoint S3 | `https://axwfixabzvp5.compat.objectstorage.ap-singapore-1.oraclecloud.com` |
| `<VM_IP>` | IP công khai của VM (có sau bước 4) |

**Quy tắc khi chạy lệnh:** `ACCESS_KEY` và `SECRET` là giá trị bạn thay vào, đặt trong dấu nháy đơn `' '`,
không gõ dấu `< >`. Không dán khóa vào chat, không ghi vào file nằm trong repo.

---

## 1. Tạo bucket (OCI Console)

1. Menu → **Storage → Buckets** → chọn compartment → **Create Bucket**.
2. Tên chỉ gồm chữ thường, số và dấu `-` (không dùng `/` hoặc `_`), Standard, giữ chế độ Private.
3. Ghi lại **Namespace** (trang chi tiết bucket) và **Region identifier**.

## 2. Tạo Customer Secret Key (thay cho sa-key.json)

1. Góc trên phải: biểu tượng người dùng → **My profile** → tab **Tokens and keys** →
   **Customer secret keys** → **Generate secret key**.
2. Đặt tên `income-lab`, bấm Generate, **copy ngay Secret** (chỉ hiện một lần).
3. Cột **Access key** của dòng vừa tạo là `access_key_id`.
4. Lưu cả hai vào một file ghi chú **ngoài thư mục repo**.

## 3. Cấu hình DVC (PowerShell, tại thư mục gốc repo)

```powershell
dvc init
dvc remote add -d labstore s3://lab21-cicd-for-ai/dvc
dvc remote modify labstore endpointurl https://axwfixabzvp5.compat.objectstorage.ap-singapore-1.oraclecloud.com
dvc remote modify labstore region ap-singapore-1

# Khóa bí mật: lưu vào .dvc/config.local (đã nằm trong .gitignore, KHÔNG commit)
dvc remote modify --local labstore access_key_id 'ACCESS_KEY'
dvc remote modify --local labstore secret_access_key 'SECRET'

dvc add data/train_batch1.csv data/holdout.csv data/train_batch2.csv
dvc push

git add data/train_batch1.csv.dvc data/holdout.csv.dvc data/train_batch2.csv.dvc .dvc/config .dvcignore
git commit -m "feat: track datasets with DVC"
```

Kiểm tra: `dvc push` báo đã đẩy 3 file, tab **Objects** của bucket có thư mục `dvc/`, và
`git status` không hiện `.dvc/config.local`.

Lưu ý: `.gitignore` gốc đã ignore 3 file CSV nên DVC không tạo `data/.gitignore`. Điều này bình thường.

## 4. Tạo VM (OCI Console)

1. **Compute → Instances → Create instance**, chọn compartment `ai-lab-compartment`.
2. Tên `income-api`; Image: **Canonical Ubuntu 22.04**; Shape: `VM.Standard.E2.1.Micro` (Always Free)
   hoặc Ampere A1 nếu còn chỗ.
3. Networking: dùng VCN/subnet công khai, bật **Assign a public IPv4 address**.
4. SSH keys: **Generate a key pair** → tải private key về (ví dụ `oci-vm.key`) và đặt vào một thư mục
   **ngoài repo**, ví dụ `C:\Users\AdminMH\.ssh\oci-vm.key`.
5. Tạo xong, ghi lại **Public IP** (`<VM_IP>`). User mặc định là `ubuntu`.

### Mở cổng 8080 (cần cả hai chỗ)

- Console: instance → **Subnet** → **Security Lists** → Default → **Add Ingress Rules**:
  Source CIDR `0.0.0.0/0`, IP Protocol TCP, Destination Port `8080`.
- Trên VM (Ubuntu của OCI có iptables chặn sẵn): xem mục 5.

## 5. Cài đặt trên VM

Trên máy bạn (PowerShell). Windows có sẵn `ssh` và `scp`; bước `icacls` thay cho `chmod 600`:

```powershell
$key = "$env:USERPROFILE\.ssh\oci-vm.key"
icacls $key /inheritance:r /grant:r "$($env:USERNAME):(R)"
ssh -i $key ubuntu@<VM_IP>
```

Từ đây là phiên **Bash trên VM**. Dán từng khối:

```bash
sudo iptables -I INPUT 6 -m state --state NEW -p tcp --dport 8080 -j ACCEPT
sudo netfilter-persistent save

sudo apt update && sudo apt install -y python3-pip
pip3 install --no-cache-dir fastapi==0.111.0 uvicorn==0.29.0 scikit-learn==1.4.2 joblib==1.4.2 boto3
mkdir -p ~/models ~/src
```

Tạo file biến môi trường (thay giá trị thật, không dùng dấu nháy):

```bash
cat > ~/income-api.env <<'EOF'
ARTIFACT_BUCKET=lab21-cicd-for-ai
STORAGE_ENDPOINT=https://axwfixabzvp5.compat.objectstorage.ap-singapore-1.oraclecloud.com
AWS_ACCESS_KEY_ID=ACCESS_KEY
AWS_SECRET_ACCESS_KEY=SECRET
AWS_DEFAULT_REGION=ap-singapore-1
EOF
chmod 600 ~/income-api.env
```

Tạo service systemd:

```bash
sudo tee /etc/systemd/system/income-api.service > /dev/null <<'EOF'
[Unit]
Description=Income Model Inference Server
After=network.target

[Service]
User=ubuntu
WorkingDirectory=/home/ubuntu
EnvironmentFile=/home/ubuntu/income-api.env
ExecStart=/usr/bin/python3 /home/ubuntu/src/serve.py
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable income-api
exit
```

Quay lại PowerShell, chép mã nguồn lên VM (tại thư mục gốc repo):

```powershell
scp -i $key src/serve.py ubuntu@<VM_IP>:~/src/serve.py
```

## 6. Tạo SSH key cho GitHub Actions (PowerShell)

```powershell
ssh-keygen -t ed25519 -f "$env:USERPROFILE\.ssh\income_deploy" -N '""' -C "github-actions-deploy"

$pub = (Get-Content "$env:USERPROFILE\.ssh\income_deploy.pub").Trim()
ssh -i $key ubuntu@<VM_IP> "echo '$pub' >> ~/.ssh/authorized_keys"
```

## 7. Năm GitHub Secrets

Settings → Secrets and variables → Actions:

| Secret | Giá trị |
|---|---|
| `STORAGE_CREDENTIALS` | Chuỗi JSON (xem dưới) |
| `ARTIFACT_BUCKET` | `lab21-cicd-for-ai` |
| `SERVER_HOST` | `<VM_IP>` |
| `SERVER_USER` | `ubuntu` |
| `SERVER_SSH_KEY` | Toàn bộ nội dung file private key `income_deploy` (gồm cả dòng BEGIN/END) |

Lấy nội dung private key ra clipboard để dán vào GitHub:

```powershell
Get-Content "$env:USERPROFILE\.ssh\income_deploy" -Raw | Set-Clipboard
```

Giá trị `STORAGE_CREDENTIALS`:

```json
{"access_key_id": "ACCESS_KEY", "secret_access_key": "SECRET",
 "endpoint": "https://axwfixabzvp5.compat.objectstorage.ap-singapore-1.oraclecloud.com", "region": "ap-singapore-1"}
```

## 8. Chạy pipeline và kiểm tra (PowerShell)

```powershell
git push -u origin main
```

Xem tab **Actions**. Job Release sẽ đỏ ở lần chạy đầu tiên nếu service chưa có model hoặc chưa
chạy; sau khi job Train đã upload model, khởi động service rồi chạy lại workflow (Re-run hoặc
`workflow_dispatch`):

```powershell
ssh -i $key ubuntu@<VM_IP> "sudo systemctl start income-api"
curl.exe http://<VM_IP>:8080/healthz
curl.exe -X POST http://<VM_IP>:8080/score -H "Content-Type: application/json" -d '{\"features\": [28, 2, 14, 2, 11, 0, 1, 0, 0, 45]}'
```

Trong PowerShell phải gõ `curl.exe` (không phải `curl`, vì `curl` là bí danh của `Invoke-WebRequest`)
và thoát dấu `"` trong JSON bằng `\"`.

Xem log khi service lỗi: `ssh -i $key ubuntu@<VM_IP> "sudo journalctl -u income-api -n 50"`.

## 9. Ảnh cần chụp

- `02-actions-buoc-2.png`: 4 jobs xanh trong tab Actions.
- `04-curl-api.png`: hai lệnh curl có `<VM_IP>` và kết quả.
- `05-cloud-storage.png`: OCI Console → bucket, thấy thư mục `dvc/` và `artifacts/current/model.joblib`.
