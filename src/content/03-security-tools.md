=== sast | Công cụ bảo mật | SAST: phân tích mã tĩnh
>> SAST (Static Application Security Testing) đọc mã nguồn mà không chạy nó để tìm các mẫu lỗi bảo mật như injection, dùng hàm nguy hiểm, mật mã yếu. Rẻ và sớm, nhưng có cảnh báo giả và bỏ sót lỗi logic.

## Cách hoạt động

Công cụ phân tích cú pháp mã thành cây cú pháp và theo dõi luồng dữ liệu (**taint analysis**): dữ liệu từ **nguồn không tin cậy** (tham số yêu cầu, tệp tải lên) có tới **điểm nhạy cảm** (câu truy vấn SQL, lệnh shell, `eval`, render HTML) mà không qua **bước làm sạch** nào không.

| Mạnh | Yếu |
|---|---|
| Chạy sớm (IDE, PR), không cần ứng dụng chạy | Cảnh báo giả (cho rằng lỗi mà thực tế an toàn) |
| Chỉ ra đúng dòng mã | Bỏ sót lỗi logic, phân quyền, cấu hình lúc chạy |
| Phủ toàn bộ mã, kể cả đường ít được gọi | Phụ thuộc ngôn ngữ và framework được hỗ trợ |
| Tự động hoá tốt | Quy tắc cần tinh chỉnh theo dự án |

## Công cụ phổ biến

| Công cụ | Ghi chú |
|---|---|
| **Semgrep** | Nhẹ, quy tắc dạng mẫu mã dễ viết, nhiều ngôn ngữ, bộ quy tắc cộng đồng; chạy cục bộ và CI |
| **CodeQL** | Truy vấn mã như cơ sở dữ liệu; tích hợp sẵn GitHub code scanning; phân tích sâu |
| **SonarQube / SonarCloud** | Chất lượng mã kèm bảo mật, quality gate |
| **Bandit** | Chuyên Python, chạy rất nhanh |
| **gosec, Brakeman, SpotBugs + FindSecBugs, ESLint plugin security** | Chuyên theo ngôn ngữ (Go, Ruby on Rails, Java, JavaScript) |

Chọn một công cụ phủ ngôn ngữ của bạn và bắt đầu nhỏ; nhiều công cụ cùng lúc tạo nhiều nhiễu.

## Semgrep

```
pip install semgrep
semgrep scan --config p/python --config p/owasp-top-ten .        # bộ quy tắc từ registry
semgrep scan --config p/ci --error --sarif -o semgrep.sarif .    # phù hợp cho CI: lỗi thì thoát mã khác 0
```

Viết **quy tắc riêng** cho các thực hành của tổ chức, mẫu mã giống chính mã bạn tìm:

```
# rules/no-shell-true.yml
rules:
  - id: khong-dung-shell-true
    languages: [python]
    severity: ERROR
    message: "subprocess với shell=True dễ bị command injection. Truyền danh sách tham số và bỏ shell=True."
    pattern: subprocess.$FUNC(..., shell=True, ...)
  - id: sql-noi-chuoi
    languages: [python]
    severity: ERROR
    message: "Không nối chuỗi vào câu SQL; dùng tham số hoá."
    patterns:
      - pattern: $CUR.execute($Q + ...)
```

```
semgrep scan --config rules/ src/
```

## Bandit (Python)

```
pip install bandit
bandit -r src -ll                      # chỉ mức Trung bình trở lên
bandit -r src -f sarif -o bandit.sarif
```

## Ví dụ lỗi và cách sửa

```
# DỄ BỊ SQL INJECTION
def tim_user(cur, name):
    cur.execute("SELECT * FROM users WHERE name = '" + name + "'")

# AN TOÀN: truy vấn tham số hoá
def tim_user(cur, name):
    cur.execute("SELECT * FROM users WHERE name = %s", (name,))

# NGUY HIỂM: thực thi chuỗi do người dùng cung cấp
result = eval(request.args["expr"])
# thay bằng phân tích có kiểm soát (ví dụ ast.literal_eval cho dữ liệu, hoặc bộ phân tích riêng)
```

## Xử lý kết quả: giảm nhiễu

SAST mà ai cũng bỏ qua thì vô dụng. Một chương trình tốt:

1. **Bắt đầu nhỏ**: chỉ bật các quy tắc độ tin cậy cao và mức nghiêm trọng cao; mở rộng dần.
2. **Đường cơ sở (baseline)**: ghi nhận cảnh báo cũ, chỉ chặn **cảnh báo mới** do PR đưa vào, rồi xử lý nợ cũ theo kế hoạch.
3. **Chú thích bỏ qua có lý do và hạn**: ví dụ `# nosec B602 - đầu vào là hằng số, xem ticket SEC-123` (Bandit) hoặc `# nosemgrep: rule-id`. Rà soát định kỳ các ngoại lệ.
4. **Phân loại (triage)**: người xem cảnh báo quyết định: lỗi thật, giả (false positive), chấp nhận rủi ro. Ghi lại để quy tắc được tinh chỉnh.
5. **Đưa kết quả vào nơi làm việc**: bình luận trên PR, SARIF tải lên code scanning của GitHub/GitLab.
6. **Đo**: số cảnh báo mới mỗi tuần, thời gian xử lý, tỷ lệ giả.

::: warn Công cụ sạch không có nghĩa là mã an toàn
SAST chỉ thấy những mẫu mà quy tắc biết. Nó không thay thế review mã, threat modeling hay kiểm thử phân quyền. Dùng nó như lưới lọc đầu tiên cho lỗi phổ biến, không phải bằng chứng an toàn.
:::

::: lab Quét một đoạn mã có lỗi cố ý
```
mkdir -p ~/sast && cd ~/sast
cat > app.py <<'EOF'
import subprocess, sqlite3, hashlib, yaml

def ping(host):
    return subprocess.check_output("ping -c1 " + host, shell=True)      # command injection

def find(conn, name):
    return conn.execute("SELECT * FROM users WHERE name = '" + name + "'").fetchall()   # SQL injection

def fingerprint(pw):
    return hashlib.md5(pw.encode()).hexdigest()                          # băm yếu

def load(text):
    return yaml.load(text)                                               # nạp YAML không an toàn

TOKEN = "ghp_" + "exampleexampleexampleexample12345"                    # bí mật cố định (giả)
EOF
pip install -q bandit semgrep
bandit -r . -ll
semgrep scan --config p/python app.py 2>&1 | tail -25
```
Với mỗi cảnh báo: đúng hay giả, mức độ, và bạn sửa thế nào (truyền danh sách tham số, truy vấn tham số hoá, `hashlib.sha256` hoặc `bcrypt/argon2` cho mật khẩu, `yaml.safe_load`, đọc token từ môi trường)? Viết một quy tắc Semgrep riêng bắt `eval(` trong mã của bạn.
:::

=== sca-sbom | Công cụ bảo mật | Phụ thuộc, SCA và SBOM
>> Phần lớn mã trong ứng dụng hiện đại là thư viện của người khác. SCA tìm lỗ hổng đã biết trong chúng, và SBOM cho biết chính xác bạn đang dùng gì để trả lời nhanh khi có CVE mới.

## Vấn đề

Ứng dụng của bạn kéo theo hàng trăm đến hàng nghìn **phụ thuộc gián tiếp**. Một lỗ hổng trong một thư viện sâu bên trong (như sự cố Log4Shell năm 2021) ảnh hưởng tới mọi ứng dụng dùng nó, kể cả khi bạn không biết mình dùng. Hai câu hỏi cần trả lời nhanh: **mình có dùng thành phần X không?** và **bản nào, ở dịch vụ nào?**

## SCA (Software Composition Analysis)

SCA đối chiếu danh sách thư viện (từ tệp khoá `package-lock.json`, `requirements.txt`, `go.sum`, `pom.xml`...) với cơ sở dữ liệu lỗ hổng (NVD, GitHub Advisory, OSV...).

| Công cụ | Dùng cho |
|---|---|
| `pip-audit` | Python |
| `npm audit`, `yarn audit` | JavaScript |
| **OSV-Scanner** | Đa ngôn ngữ, dùng dữ liệu OSV |
| **Trivy** (`trivy fs`) | Quét thư mục dự án, cả image và IaC |
| **Grype** | Quét thư mục và image |
| **Dependabot, Renovate** | Tự mở PR cập nhật phụ thuộc có lỗ hổng |
| OWASP Dependency-Check, Snyk, các dịch vụ thương mại | Tương tự với tính năng bổ sung |

```
pip-audit -r requirements.txt
npm audit --audit-level=high
osv-scanner scan -r .                      # (tên lệnh con thay đổi theo phiên bản)
trivy fs --severity HIGH,CRITICAL --ignore-unfixed .
grype dir:. --fail-on high
```

Các công cụ khác nhau cho kết quả khác nhau do nguồn dữ liệu và cách khớp. Chọn một công cụ chính và hiểu giới hạn của nó.

## Đọc kết quả: không phải mọi CVE đều đáng lo như nhau

| Câu hỏi | Vì sao quan trọng |
|---|---|
| Mức nghiêm trọng (CVSS) | Điểm cơ sở, **không** phải rủi ro thực của bạn |
| Có bản vá chưa? (`--ignore-unfixed`) | Không vá được thì xử lý khác (giảm nhẹ, theo dõi) |
| Mã dễ bị tấn công có **được gọi** trong ứng dụng không? | Phụ thuộc có lỗ hổng nhưng đường mã không bao giờ chạm tới thì rủi ro thấp hơn nhiều (phân tích khả năng tiếp cận) |
| Có khai thác trên thực tế không? | EPSS (xác suất khai thác) và danh mục **CISA KEV** (lỗ hổng đã bị khai thác) giúp ưu tiên |
| Phụ thuộc trực tiếp hay gián tiếp? | Gián tiếp thì cần nâng phiên bản của thư viện bao ngoài |
| Có tiếp xúc với Internet không? | Dịch vụ công khai ưu tiên hơn công cụ nội bộ |

Chi tiết ưu tiên ở [Quản lý lỗ hổng](#pipeline-vuln-mgmt).

## Cập nhật phụ thuộc tự động

```
# .github/dependabot.yml
version: 2
updates:
  - package-ecosystem: "pip"
    directory: "/"
    schedule: {interval: "weekly"}
    open-pull-requests-limit: 5
  - package-ecosystem: "github-actions"
    directory: "/"
    schedule: {interval: "weekly"}
```

- Gom thay đổi nhỏ, có kiểm thử tự động bảo vệ: PR nâng phiên bản chỉ nên hợp nhất khi test xanh.
- Cập nhật **thường xuyên** nhỏ giọt dễ hơn cập nhật dồn sau nhiều năm.
- **Tệp khoá (lockfile)** và ghim phiên bản cho kết quả lặp lại; ghim cả image nền và action của CI.
- Cẩn thận chuỗi cung ứng: gói độc hại nhái tên (typosquatting), tài khoản người duy trì bị chiếm, thư viện bị tác giả cố ý phá hoại. Dùng registry nội bộ làm trung gian và xem xét kỹ phụ thuộc mới.

## SBOM: danh mục thành phần

**SBOM (Software Bill of Materials)** liệt kê mọi thành phần trong một sản phẩm: tên, phiên bản, giấy phép, nguồn gốc. Hai định dạng phổ biến: **SPDX** và **CycloneDX**.

```
syft packages dir:. -o cyclonedx-json > sbom.cdx.json
syft registry.example.com/web:1.4.2 -o spdx-json > sbom.spdx.json
trivy image --format cyclonedx --output sbom.cdx.json registry.example.com/web:1.4.2
grype sbom:sbom.cdx.json                       # quét lại SBOM đã lưu mà không cần image
```

Giá trị của SBOM:

- Khi có CVE mới, bạn **tra SBOM đã lưu** để biết dịch vụ nào bị ảnh hưởng, **không cần quét lại** mọi thứ.
- Đáp ứng yêu cầu pháp lý và hợp đồng ngày càng phổ biến.
- Kiểm tra giấy phép (tránh dùng giấy phép không tương thích).

Thực hành: **sinh SBOM trong mỗi lần build**, gắn vào artifact, và lưu cùng phiên bản phát hành. Có thể kèm **VEX** (Vulnerability Exploitability eXchange) để nói rõ "CVE này có trong thành phần nhưng không ảnh hưởng tới sản phẩm vì ...", giảm nhiễu cho người nhận.

Chi tiết liên quan: [Chuỗi cung ứng image](https://tracviethustcs.github.io/k8s-study-notes/#adv-supply-chain), [Artifact và phiên bản](https://tracviethustcs.github.io/cicd-release-notes/#artifacts).

::: lab Quét phụ thuộc và sinh SBOM
```
mkdir -p ~/sca && cd ~/sca
printf 'flask==1.0\nrequests==2.19.0\njinja2==2.10\n' > requirements.txt     # phiên bản cũ cố ý
python3 -m venv .venv && . .venv/bin/activate && pip install -q pip-audit
pip-audit -r requirements.txt 2>&1 | head -25
# sinh SBOM (nếu đã cài syft)
syft dir:. -o cyclonedx-json > sbom.json 2>/dev/null && python3 -c "import json;d=json.load(open('sbom.json'));print(len(d['components']),'thành phần')"
```
Chọn một CVE trong kết quả: mức nghiêm trọng, phiên bản được vá, và liệu mã của bạn có gọi đường mã bị ảnh hưởng không? Bạn sẽ nâng cấp thế nào và dựa vào đâu để biết nâng cấp không phá vỡ ứng dụng (bộ kiểm thử)?
:::

=== container-iac-scan | Công cụ bảo mật | Quét container, IaC và chính sách
>> Image và cấu hình hạ tầng là nơi sai sót bảo mật phổ biến nhất: image cũ, chạy bằng root, bucket công khai, security group mở toàn bộ. Quét chúng như quét mã, trước khi triển khai.

## Quét image container

Image có hai nguồn lỗ hổng: **gói hệ điều hành** của image nền và **thư viện ứng dụng** bên trong.

```
trivy image --severity HIGH,CRITICAL --ignore-unfixed registry.example.com/web:1.4.2
trivy image --exit-code 1 --severity CRITICAL registry.example.com/web:1.4.2   # chặn pipeline nếu có CRITICAL
grype registry.example.com/web:1.4.2 --fail-on high
```

Cách giảm lỗ hổng hiệu quả nhất không phải là quét mà là **giảm bề mặt**:

- Dùng image nền **tối giản** (distroless, alpine, slim) và multi-stage ([Dockerfile tốt](https://tracviethustcs.github.io/docker-container-notes/#build-best)).
- **Xây lại định kỳ** để lấy bản vá của image nền, thay vì chỉ khi mã đổi.
- Quét **lại image đang chạy** theo lịch: CVE mới xuất hiện sau khi build.
- Chạy bằng người dùng không phải root, bỏ capability (xem [Bảo mật container](https://tracviethustcs.github.io/docker-container-notes/#security)).

## Lint Dockerfile

```
hadolint Dockerfile
docker run --rm -i hadolint/hadolint < Dockerfile
```

Hadolint bắt các thực hành xấu: dùng `latest`, thiếu `USER`, `apt-get install` không có `--no-install-recommends`, không dọn cache...

## Quét IaC (Terraform, CloudFormation, Kubernetes, Helm)

Quét cấu hình hạ tầng **trước khi apply**:

```
trivy config .                              # Terraform, Kubernetes, Dockerfile, CloudFormation...
checkov -d . --framework terraform
kube-linter lint k8s/
kubescape scan k8s/ --frameworks nsa,cis    # theo khung NSA, CIS
```

Các lỗi thường gặp mà công cụ bắt:

| Nhóm | Ví dụ |
|---|---|
| Truy cập công khai | Bucket S3 công khai, security group mở `0.0.0.0/0` cho SSH/database |
| Thiếu mã hoá | Đĩa, database, bucket không mã hoá |
| Quyền quá rộng | Chính sách IAM `Action: "*"`, `Resource: "*"` |
| Kubernetes | Pod chạy root, `privileged: true`, thiếu giới hạn tài nguyên, `hostNetwork` |
| Thiếu ghi log | Không bật CloudTrail, log truy cập |

Ví dụ lỗi và cách sửa:

```
# CÓ LỖI: SSH mở cho cả Internet
resource "aws_security_group_rule" "ssh" {
  type              = "ingress"
  from_port         = 22
  to_port           = 22
  protocol          = "tcp"
  cidr_blocks       = ["0.0.0.0/0"]
  security_group_id = aws_security_group.web.id
}

# AN TOÀN HƠN: chỉ từ dải quản trị (hoặc dùng Session Manager, bỏ cổng 22)
resource "aws_security_group_rule" "ssh" {
  type              = "ingress"
  from_port         = 22
  to_port           = 22
  protocol          = "tcp"
  cidr_blocks       = ["203.0.113.0/24"]
  security_group_id = aws_security_group.web.id
}
```

## Chính sách dưới dạng mã (policy as code)

Quy tắc riêng của tổ chức viết thành mã và thực thi tự động:

| Công cụ | Ghi chú |
|---|---|
| **OPA / Conftest** | Ngôn ngữ Rego, kiểm tra tệp cấu hình (JSON, YAML, plan Terraform) trong CI |
| **Kyverno, OPA Gatekeeper** | Chặn tài nguyên vi phạm **khi tạo trong Kubernetes** (admission) |
| **Sentinel** | Chính sách cho Terraform Cloud/Enterprise |

Ví dụ Conftest kiểm tra Kubernetes:

```
# policy/deny_root.rego
package main

deny[msg] {
  input.kind == "Deployment"
  not input.spec.template.spec.securityContext.runAsNonRoot
  msg := sprintf("Deployment %s phải đặt runAsNonRoot: true", [input.metadata.name])
}
```
```
conftest test deployment.yaml --policy policy/
```

Hai điểm kiểm tra bổ trợ nhau: **quét trong CI** cho phản hồi sớm cho lập trình viên; **admission trong cụm** là rào chắn cuối cùng không thể bỏ qua ([Kyverno và policy](https://tracviethustcs.github.io/k8s-study-notes/#psa)).

::: lab Quét Dockerfile và manifest có lỗi cố ý
```
mkdir -p ~/ci-scan && cd ~/ci-scan
cat > Dockerfile <<'EOF'
FROM ubuntu:latest
RUN apt-get update && apt-get install -y curl python3
COPY . /app
CMD python3 /app/main.py
EOF
cat > deploy.yaml <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata: {name: web}
spec:
  replicas: 1
  selector: {matchLabels: {app: web}}
  template:
    metadata: {labels: {app: web}}
    spec:
      containers:
        - name: web
          image: web:latest
          securityContext: {privileged: true}
EOF
docker run --rm -i hadolint/hadolint < Dockerfile
trivy config . 2>&1 | head -30
```
Liệt kê các lỗi tìm thấy và sửa từng cái: ghim phiên bản image nền, thêm `USER`, bỏ `privileged`, đặt `runAsNonRoot`, thêm giới hạn tài nguyên, ghim tag image. Quét lại để xác nhận đã sạch.
:::

=== dast | Công cụ bảo mật | DAST: kiểm thử bảo mật ứng dụng đang chạy
>> DAST tấn công thử ứng dụng đang chạy từ bên ngoài như kẻ tấn công thật, nên bắt được lỗi cấu hình và lỗi chỉ lộ lúc chạy mà SAST không thấy. Đổi lại phải có môi trường chạy và có thể gây tác động.

::: warn Chỉ quét hệ thống bạn được phép
DAST gửi các yêu cầu tấn công thử thật sự. Chỉ chạy với hệ thống **bạn sở hữu hoặc có văn bản cho phép**, trên **môi trường thử** (staging) với dữ liệu giả, không bao giờ trên production hay hệ thống của người khác. Quét trái phép là vi phạm pháp luật. Báo trước cho nhóm vận hành.
:::

## SAST và DAST bổ trợ nhau

| | SAST | DAST |
|---|---|---|
| Nhìn vào | Mã nguồn (hộp trắng) | Ứng dụng đang chạy qua HTTP (hộp đen) |
| Cần chạy ứng dụng | Không | **Có** |
| Giai đoạn | Sớm (PR) | Muộn hơn (staging) |
| Tìm tốt | Lỗi trong mã, mẫu nguy hiểm | Sai cấu hình, header thiếu, lỗi xác thực phiên, lỗi phát sinh từ tích hợp |
| Hạn chế | Cảnh báo giả, không thấy cấu hình runtime | Chậm hơn, khó phủ hết đường đi, cần xác thực để quét khu vực đã đăng nhập |

## OWASP ZAP

ZAP là công cụ DAST mã nguồn mở phổ biến. Ba chế độ thường dùng, mỗi chế độ một mức xâm lấn:

| Chế độ | Làm gì | Khi nào |
|---|---|---|
| **Baseline scan** | Duyệt (spider) và phân tích **bị động**, không tấn công chủ động | Phù hợp CI, an toàn |
| **API scan** | Quét API theo tệp đặc tả (OpenAPI, GraphQL) | API dịch vụ |
| **Full scan** | Quét **chủ động** với nhiều tấn công thử | Chỉ trên môi trường thử, theo lịch; có thể làm thay đổi dữ liệu |

```
docker run --rm -t ghcr.io/zaproxy/zaproxy:stable \
  zap-baseline.py -t https://staging.example.com -r zap-baseline.html

docker run --rm -v "$PWD":/zap/wrk -t ghcr.io/zaproxy/zaproxy:stable \
  zap-api-scan.py -t https://staging.example.com/openapi.json -f openapi -r zap-api.html

docker run --rm -v "$PWD":/zap/wrk -t ghcr.io/zaproxy/zaproxy:stable \
  zap-full-scan.py -t https://staging.example.com -r zap-full.html        # chủ động: cẩn thận
```

Dùng tệp cấu hình quy tắc (`-c rules.conf`) để chuyển từng quy tắc giữa `FAIL`, `WARN`, `IGNORE`, giúp đặt ngưỡng chặn.

## Quét khu vực cần đăng nhập

Phần lớn bề mặt của ứng dụng nằm sau đăng nhập. Cần cấu hình xác thực (tài khoản kiểm thử, script đăng nhập hoặc token trong header) để DAST thấy được vùng đó. Dùng **tài khoản kiểm thử riêng với quyền hạn chế**; kiểm tra thêm với hai người dùng khác quyền để bắt lỗi phân quyền (kẻ dùng ID của người khác).

## Nuclei: quét theo mẫu

Nuclei dùng các **template** do cộng đồng viết để dò lỗ hổng và sai cấu hình đã biết:

```
nuclei -u https://staging.example.com -severity high,critical
nuclei -l hosts.txt -t http/exposures/ -severity medium,high,critical
```

Hữu ích để rà nhanh các cấu hình lộ (bảng điều khiển công khai, tệp `.env`, phiên bản dễ tổn thương).

## Kiểm tra cấu hình HTTP và TLS đơn giản

Nhiều lỗi cấu hình có thể kiểm tra ngay bằng vài lệnh:

```
curl -sI https://staging.example.com | grep -i -E "strict-transport|content-security|x-content-type|x-frame|referrer"
openssl s_client -connect staging.example.com:443 -servername staging.example.com </dev/null 2>/dev/null | openssl x509 -noout -dates
```

Header nên có: `Strict-Transport-Security`, `Content-Security-Policy`, `X-Content-Type-Options: nosniff`, chính sách chống nhúng khung (`frame-ancestors` hoặc `X-Frame-Options`), và cookie có `Secure`, `HttpOnly`, `SameSite`.

## Đưa DAST vào pipeline

- **Mỗi PR/merge**: baseline scan (bị động, vài phút) trên môi trường thử ngắn hạn.
- **Theo lịch (đêm hoặc tuần)**: quét chủ động đầy đủ trên staging.
- Dựng ứng dụng bằng container (Docker Compose) cho job quét, rồi huỷ sau khi xong ([Môi trường tạm cho PR](https://tracviethustcs.github.io/cicd-release-notes/#environments)).
- Chặn khi có phát hiện **mức cao và đã xác minh**; còn lại báo cáo kèm theo dõi.
- Lưu báo cáo (HTML, SARIF) làm artifact.

::: lab Quét baseline một ứng dụng chạy cục bộ
```
# ứng dụng thử cố ý thiếu header an ninh (chạy cục bộ, chỉ của bạn)
mkdir -p ~/dast && cd ~/dast && echo '<html><body><form method=post action=/x><input name=q></form></body></html>' > index.html
docker run -d --name target -p 127.0.0.1:8088:80 -v "$PWD":/usr/share/nginx/html:ro nginx:alpine
docker run --rm --network host -v "$PWD":/zap/wrk -t ghcr.io/zaproxy/zaproxy:stable \
  zap-baseline.py -t http://127.0.0.1:8088 -r zap-report.html -I
ls -l zap-report.html
docker rm -f target
```
Mở `zap-report.html`: ZAP báo những thiếu sót nào (header, cookie, thông tin phiên bản)? Cấu hình nginx thêm các header cần thiết rồi quét lại để xác nhận cảnh báo biến mất.
:::

=== runtime-security | Công cụ bảo mật | Bảo mật lúc chạy và kiểm tra cấu hình
>> Phòng ngừa không bao giờ hoàn hảo. Bảo vệ lúc chạy phát hiện và hạn chế những gì lọt qua: hành vi bất thường trong container, cấu hình máy chủ lệch chuẩn, và lưu lượng đáng ngờ.

## Vì sao cần lớp này

Quét trước triển khai chỉ biết những gì đã biết. Lỗ hổng chưa có CVE (zero-day), tài khoản bị chiếm, hoặc cấu hình bị đổi sau triển khai chỉ lộ ra khi hệ thống chạy. Nguyên tắc "**giả định đã bị xâm nhập**": cần cách phát hiện, giới hạn tác hại và ứng phó nhanh.

## Phát hiện hành vi bất thường lúc chạy

**Falco** (dự án CNCF) theo dõi system call và sự kiện Kubernetes, cảnh báo khi hành vi vi phạm quy tắc, ví dụ: có shell mới mở trong container, ghi vào thư mục hệ thống, tiến trình lạ kết nối ra ngoài, đọc tệp bí mật.

```
# ví dụ quy tắc kiểu Falco: cảnh báo khi mở shell trong container
- rule: Shell trong container
  desc: Phát hiện shell tương tác được mở trong container
  condition: spawned_process and container and proc.name in (bash, sh, zsh) and proc.tty != 0
  output: "Shell trong container (user=%user.name container=%container.name cmd=%proc.cmdline)"
  priority: WARNING
```

Công cụ cùng nhóm: Tetragon, Tracee (dựa trên eBPF), hay các nền tảng bảo vệ workload thương mại. Các công cụ eBPF quan sát hệ thống với chi phí thấp ([eBPF và quan sát kernel](https://tracviethustcs.github.io/linux-network-notes/#os-modules)).

## Giới hạn tác hại (containment)

| Biện pháp | Tác dụng |
|---|---|
| Container non-root, bỏ capability, filesystem chỉ đọc, seccomp | Kẻ tấn công vào được cũng làm được rất ít ([Bảo mật container](https://tracviethustcs.github.io/docker-container-notes/#security)) |
| **NetworkPolicy** mặc định từ chối | Ngăn di chuyển ngang và gọi ra ngoài tuỳ tiện ([Mạng Kubernetes](https://tracviethustcs.github.io/k8s-study-notes/#ingress)) |
| Quyền tối thiểu cho tài khoản dịch vụ, RBAC chặt | Giảm phạm vi của một danh tính bị chiếm |
| Cô lập mạnh cho mã không tin cậy (gVisor, microVM) | Xem [OpenSandbox và gVisor](https://tracviethustcs.github.io/k8s-study-notes/#tool-sandbox) |
| Giới hạn tài nguyên | Chống tấn công cạn tài nguyên |
| Bí mật ngắn hạn, xoay vòng | Khoá bị đánh cắp mau hết hiệu lực |

## WAF và bảo vệ biên

**WAF (Web Application Firewall)** lọc lưu lượng HTTP độc hại (SQL injection, XSS, bot) trước khi tới ứng dụng, bằng quy tắc (ví dụ bộ OWASP Core Rule Set với ModSecurity/Coraza, hoặc WAF của cloud/CDN). WAF là lớp **bổ sung**, không thay thế sửa lỗi trong mã: nó có thể bị vượt qua và gây chặn nhầm. Dùng chế độ chỉ ghi log trước khi chuyển sang chặn.

Các biện pháp biên khác: giới hạn tốc độ, bảo vệ DDoS, TLS đúng chuẩn, mTLS giữa các dịch vụ.

## Kiểm tra cấu hình theo chuẩn

| Đối tượng | Công cụ | Chuẩn |
|---|---|---|
| Máy chủ Linux | **Lynis** | Tự kiểm tra theo thực hành tốt |
| Docker host | **docker-bench-security** | CIS Docker Benchmark |
| Cụm Kubernetes | **kube-bench**, **kubescape** | CIS Kubernetes Benchmark, NSA |
| Cloud | Công cụ CSPM (ScoutSuite, Prowler, các dịch vụ của nhà cung cấp) | CIS cloud benchmarks |

```
sudo lynis audit system
docker run --rm --net host --pid host --userns host --cap-add audit_control \
  -v /var/lib:/var/lib:ro -v /var/run/docker.sock:/var/run/docker.sock:ro \
  docker/docker-bench-security
kubescape scan framework nsa
```

Chạy **định kỳ** và theo dõi xu hướng; đừng áp dụng mọi khuyến nghị máy móc, kiểm thử trên môi trường thử vì một số biện pháp có thể làm hỏng ứng dụng. Xem [Hardening Linux](https://tracviethustcs.github.io/linux-network-notes/#lx-harden).

## Ghi log và giám sát phục vụ bảo mật

- Ghi **nhật ký kiểm toán** cho hành động nhạy cảm (đăng nhập, đổi quyền, truy cập dữ liệu), gửi về nơi tập trung mà kẻ tấn công không xoá được.
- Cảnh báo theo **hành vi** (đăng nhập thất bại hàng loạt, truy cập từ vị trí lạ, đổi quyền bất thường), không chỉ theo ngưỡng kỹ thuật.
- Có **kế hoạch ứng phó sự cố**: ai làm gì, cô lập thế nào, khôi phục ra sao, và đã diễn tập.
- Theo dõi **CloudTrail/audit log** của cloud và Kubernetes API.

::: lab Rà soát một máy Linux thử bằng Lynis
```
sudo apt install -y lynis        # hoặc: sudo dnf install lynis
sudo lynis audit system --quick 2>&1 | tail -40
```
Đọc phần "Suggestions" và "Warnings": chọn ba mục có tác động lớn nhất, nêu cách khắc phục và rủi ro khi áp dụng (ví dụ cấu hình SSH, mật khẩu, cập nhật, firewall). Làm trong VM thử.
:::
