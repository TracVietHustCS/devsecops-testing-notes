=== pipeline-vuln-mgmt | Vận hành | Tích hợp pipeline và quản lý lỗ hổng
>> Công cụ quét chỉ có giá trị khi kết quả được xử lý. Trang này ghép các lớp quét vào một pipeline, đặt cổng chặn hợp lý và quản lý lỗ hổng theo rủi ro, thời hạn và chỉ số.

## Ghép các lớp thành một pipeline

```ascii
 Pre-commit   Pull request (nhanh, < 10 phút)                 Sau merge / staging            Theo lịch
 ----------   ----------------------------------------        ------------------------       ---------------
 gitleaks     lint + unit test + tích hợp                     build image, quét image        quét lại phụ thuộc
 (cục bộ)  -> gitleaks  | SAST (Semgrep)  | SCA (OSV/Trivy) -> SBOM + ký                    và image đang chạy
              quét IaC / hadolint                             DAST baseline + API test       DAST đầy đủ, E2E,
                                                              smoke test sau triển khai       kiểm thử tải
```

Nguyên tắc: phản hồi nhanh ở **PR**, kiểm tra sâu hơn ở các giai đoạn sau, và quét **định kỳ** vì CVE mới xuất hiện sau khi triển khai.

## Workflow mẫu: nhiều lớp quét trên mỗi PR

```
name: security
on:
  pull_request:
  push:
    branches: [main]
  schedule:
    - cron: "0 2 * * *"            # quét lại hằng đêm

permissions:
  contents: read

jobs:
  secrets:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with: {fetch-depth: 0}      # cần toàn bộ lịch sử để quét
      - uses: gitleaks/gitleaks-action@v2
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}

  sast:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      security-events: write        # tải kết quả SARIF lên code scanning
    steps:
      - uses: actions/checkout@v4
      - run: pip install semgrep
      - run: semgrep scan --config p/ci --error --sarif --output semgrep.sarif .
      - uses: github/codeql-action/upload-sarif@v3
        if: always()
        with: {sarif_file: semgrep.sarif}

  dependencies:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: aquasecurity/trivy-action@0.28.0
        with:
          scan-type: fs
          severity: HIGH,CRITICAL
          ignore-unfixed: true
          exit-code: "1"

  iac-and-container:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: aquasecurity/trivy-action@0.28.0
        with:
          scan-type: config
          severity: HIGH,CRITICAL
          exit-code: "1"
      - run: docker build -t app:${{ github.sha }} .
      - uses: aquasecurity/trivy-action@0.28.0
        with:
          image-ref: app:${{ github.sha }}
          severity: CRITICAL
          ignore-unfixed: true
          exit-code: "1"

  dast:
    if: github.event_name == 'push'
    needs: [secrets, sast, dependencies, iac-and-container]
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: docker compose up -d --build            # dựng ứng dụng thử
      - run: |
          docker run --rm --network host -v "$PWD":/zap/wrk -t ghcr.io/zaproxy/zaproxy:stable \
            zap-baseline.py -t http://localhost:8000 -r zap.html
      - uses: actions/upload-artifact@v4
        if: always()
        with: {name: zap-report, path: zap.html}
```

Phiên bản action, ngưỡng và lệnh là ví dụ: ghim theo mã băm commit, thay bằng giá trị của bạn và thử trên repo thử nghiệm. Bảo vệ chính pipeline (quyền token, ghim action, chống chèn script) xem [Bảo mật pipeline](https://tracviethustcs.github.io/cicd-release-notes/#pipeline-security).

## Cổng chặn và ngoại lệ

| Quy tắc | Gợi ý |
|---|---|
| **Chặn** | Bí mật đã xác minh; SAST mức cao độ tin cậy cao; SCA mức CRITICAL (và HIGH) **đã có bản vá**; lỗi IaC mức cao (công khai, không mã hoá); image CRITICAL có bản vá |
| **Cảnh báo, không chặn** | Mức thấp/trung bình, lỗ hổng chưa có bản vá, kết quả độ tin cậy thấp |
| **Chỉ chặn phát hiện mới** | Dùng đường cơ sở để PR không bị chặn bởi nợ cũ không phải của nó |
| **Ngoại lệ (waiver)** | Có **lý do, người duyệt và ngày hết hạn**; lưu trong Git; rà soát định kỳ |

Quá chặt thì mọi người tìm cách tắt công cụ; quá lỏng thì vô nghĩa. Bắt đầu ở mức cảnh báo, theo dõi nhiễu vài tuần, rồi bật chặn ở nhóm kết quả đáng tin.

## Quản lý lỗ hổng: ưu tiên theo rủi ro

Tìm ra lỗ hổng mới chỉ là một nửa; nửa còn lại là quyết định xử lý cái nào trước.

| Yếu tố | Ý nghĩa |
|---|---|
| **CVSS** | Điểm 0 đến 10 về mức nghiêm trọng kỹ thuật (điểm cơ sở). Không xét môi trường của bạn |
| **EPSS** | Xác suất lỗ hổng bị khai thác trong thời gian tới theo dữ liệu thực tế |
| **CISA KEV** | Danh mục lỗ hổng **đã bị khai thác ngoài thực tế**: ưu tiên cao nhất |
| **Khả năng tiếp cận** | Đường mã dễ bị tổn thương có thật sự được gọi không |
| **Mức tiếp xúc** | Dịch vụ công khai hay nội bộ; có dữ liệu nhạy cảm không |
| **Có bản vá hoặc cách giảm nhẹ không** | Quyết định việc xử lý |

Ví dụ SLA theo rủi ro (điều chỉnh theo tổ chức):

| Mức | Thời hạn xử lý gợi ý |
|---|---|
| Đã bị khai thác (KEV) trên hệ thống công khai | Ngay lập tức (giờ) |
| Critical, có bản vá | Vài ngày |
| High | 2 đến 4 tuần |
| Medium | 1 đến 3 tháng |
| Low | Theo kế hoạch |

## Vòng đời của một lỗ hổng

1. **Phát hiện** (tự động, báo cáo bên ngoài, bug bounty).
2. **Phân loại**: đúng hay giả, ảnh hưởng thực tế, giao cho nhóm sở hữu dịch vụ.
3. **Quyết định**: sửa, giảm nhẹ, chấp nhận rủi ro (có người duyệt và hạn), hoặc loại bỏ thành phần.
4. **Sửa và kiểm thử** (có kiểm thử hồi quy cho lỗi).
5. **Xác minh** đã đóng và triển khai.
6. **Rút bài học**: lớp nào lẽ ra bắt sớm hơn.

Mỗi lỗ hổng cần **một người sở hữu** và **một hạn**; danh sách chung không ai chịu trách nhiệm sẽ phình ra vô tận.

## Chỉ số của chương trình bảo mật

| Chỉ số | Cho biết |
|---|---|
| Thời gian trung vị để sửa (MTTR) theo mức nghiêm trọng | Quy trình có hiệu quả không |
| Tỷ lệ lỗ hổng sửa trong SLA | Có giữ được cam kết không |
| Số lượng lỗ hổng mở và tuổi | Nợ bảo mật đang tăng hay giảm |
| Tỷ lệ cảnh báo giả | Mức nhiễu, độ tin cậy của công cụ |
| Phủ quét (% repo/dịch vụ có đủ các lớp quét) | Có điểm mù nào không |
| Số lỗ hổng lọt ra production (escape rate) | Pipeline bắt trước được tới đâu |
| Thời gian từ khi lộ bí mật đến khi xoay vòng | Mức sẵn sàng ứng phó |

::: warn Chỉ số không phải để xếp hạng người
Dùng chỉ số để thấy nút thắt và cải thiện quy trình, đừng chấm điểm lập trình viên theo số lỗ hổng: họ sẽ ngừng báo cáo và tránh làm mã quan trọng. Xem lưu ý tương tự ở [DORA và đo lường](https://tracviethustcs.github.io/cicd-release-notes/#metrics-dora).
:::

::: lab Thiết kế cổng cho một repo của bạn
Cho một repo thật hoặc giả định, điền bảng:

| Lớp quét | Công cụ | Chạy khi nào | Chặn khi nào | Cách xử lý ngoại lệ |
|---|---|---|---|---|
| Bí mật | | | | |
| SAST | | | | |
| SCA | | | | |
| Container | | | | |
| IaC | | | | |
| DAST | | | | |

Rồi trả lời: nếu tuần đầu bật chặn mà có 120 cảnh báo, bạn sẽ dùng đường cơ sở, ngưỡng và phân loại thế nào để PR của đồng nghiệp không bị tắc? Bạn đo thành công của chương trình sau ba tháng bằng ba chỉ số nào?
:::

=== cheatsheet | Tham chiếu | Cheat sheet DevSecOps và kiểm thử
>> Tra nhanh công cụ, lệnh và quy tắc theo nhu cầu.

## Lớp quét, công cụ và lệnh

| Lớp | Công cụ | Lệnh mẫu |
|---|---|---|
| Bí mật | gitleaks, trufflehog | `gitleaks detect --source . -v` |
| SAST | Semgrep, Bandit, CodeQL | `semgrep scan --config p/ci --error .`, `bandit -r src -ll` |
| SCA | pip-audit, npm audit, OSV-Scanner, Trivy, Grype | `pip-audit -r requirements.txt`, `trivy fs --severity HIGH,CRITICAL .` |
| SBOM | Syft, Trivy | `syft dir:. -o cyclonedx-json`, `trivy image --format cyclonedx IMG` |
| Image | Trivy, Grype | `trivy image --severity HIGH,CRITICAL --ignore-unfixed IMG` |
| Dockerfile | Hadolint | `hadolint Dockerfile` |
| IaC | Trivy config, Checkov, kube-linter, kubescape | `trivy config .`, `checkov -d .` |
| Chính sách | Conftest (OPA), Kyverno | `conftest test file.yaml --policy policy/` |
| DAST | OWASP ZAP, Nuclei | `zap-baseline.py -t URL`, `nuclei -u URL -severity high,critical` |
| Runtime | Falco, Tetragon | Quy tắc theo system call |
| Cấu hình | Lynis, docker-bench, kube-bench | `lynis audit system` |

## Kiểm thử và lệnh

| Loại | Công cụ | Lệnh mẫu |
|---|---|---|
| Đơn vị | pytest | `pytest -q --cov=app --cov-fail-under=80` |
| Đột biến | mutmut, Stryker, PIT | `mutmut run --paths-to-mutate=app/` |
| Tích hợp | Testcontainers | Container thật trong fixture |
| API | httpx/pytest, Newman, Schemathesis | `schemathesis run URL/openapi.json` |
| Hợp đồng | Pact | Consumer sinh hợp đồng, provider xác minh |
| E2E | Playwright | `pytest --tracing on` |
| Tải | k6, Locust, Gatling | `k6 run load.js` |

## Quy tắc nhớ nhanh

| Chủ đề | Quy tắc |
|---|---|
| Thứ tự phản hồi | Rẻ và nhanh ở PR; sâu và chậm ở giai đoạn sau và theo lịch |
| Bí mật bị lộ | Xoay vòng **trước**, làm sạch lịch sử sau |
| Ngưỡng chặn | Chỉ chặn kết quả đáng tin và có thể sửa; còn lại cảnh báo |
| Nợ cũ | Đường cơ sở: chặn phát hiện **mới**, xử lý nợ cũ theo kế hoạch |
| Ngoại lệ | Lý do, người duyệt, ngày hết hạn, lưu trong Git |
| Ưu tiên | CVSS là điểm đầu; xét EPSS, KEV, khả năng tiếp cận, mức tiếp xúc |
| Kiểm thử | Nhiều test nhanh ở dưới, ít test chậm ở trên |
| Độ phủ | Cho biết cái chưa kiểm thử, không phải chất lượng kiểm thử |
| DAST | Chỉ trên môi trường thử bạn sở hữu; baseline an toàn hơn full scan |
| Thiết kế | Threat modeling bắt lỗi logic mà công cụ không thấy |

## Mức nghiêm trọng và STRIDE

| STRIDE | Biện pháp điển hình |
|---|---|
| Spoofing | Xác thực mạnh, MFA, mTLS |
| Tampering | Chữ ký, TLS, kiểm soát ghi |
| Repudiation | Nhật ký kiểm toán không sửa được |
| Information disclosure | Mã hoá, kiểm soát truy cập, che dữ liệu |
| Denial of service | Giới hạn tốc độ, quota, co giãn |
| Elevation of privilege | Quyền tối thiểu, kiểm tra phân quyền phía máy chủ |

=== quiz-sec | Luyện tập | Luyện tập 1: DevSecOps và các lớp quét
>> 6 câu tình huống về nguyên tắc, quét bí mật, SAST, SCA, container và DAST. Chọn đáp án rồi bấm "Kiểm tra" để xem giải thích. Kết quả được lưu trong trình duyệt.

## Câu hỏi

@quiz
Q: Một lập trình viên lỡ commit khoá truy cập cloud vào repo, vài phút sau phát hiện và xoá khỏi mã bằng một commit mới. Hành động nào bắt buộc nhất?
- Không cần làm gì thêm vì khoá đã được xoá khỏi mã
+ Thu hồi và xoay vòng khoá ngay lập tức, kiểm tra nhật ký xem có bị dùng trái phép không; làm sạch lịch sử Git chỉ là bước dọn dẹp sau
- Chỉ làm sạch lịch sử bằng `git filter-repo` là đủ
- Đổi tên repo thành riêng tư
E: Bí mật đã push lên nơi người khác truy cập được phải coi là đã bị lộ, và lịch sử Git (cùng các bản sao, fork, bot quét) vẫn giữ nó. Chỉ việc thu hồi khoá mới đóng cửa. Kiểm tra nhật ký để biết có bị lạm dụng không.
Q: Đội của bạn bật SAST nghiêm ngặt, chặn mọi cảnh báo trên một codebase cũ và lập tức có hàng trăm PR bị đỏ; mọi người bắt đầu tắt công cụ. Điều chỉnh hợp lý nhất?
- Tắt hẳn SAST vì quá nhiễu
+ Dùng đường cơ sở: chỉ chặn cảnh báo **mới** mức cao độ tin cậy cao, ghi nhận nợ cũ để xử lý theo kế hoạch, và tinh chỉnh quy tắc giảm cảnh báo giả
- Yêu cầu cả đội sửa hết cảnh báo trước khi merge bất kỳ thứ gì
- Chuyển mọi cảnh báo sang chế độ bỏ qua vĩnh viễn
E: Công cụ nhiễu hoặc chặn vô lý bị bỏ qua. Chặn phát hiện mới đáng tin cậy bảo vệ chất lượng từ hôm nay mà không làm tắc công việc; nợ cũ xử lý dần. Ngoại lệ cần có lý do và hạn.
Q: Chọn TẤT CẢ phát biểu đúng về SAST và DAST.
+ SAST phân tích mã nguồn mà không cần chạy ứng dụng, nên chạy sớm trên PR
+ DAST kiểm thử ứng dụng đang chạy từ bên ngoài, bắt được sai cấu hình và lỗi chỉ lộ lúc chạy
- DAST nên chạy trực tiếp trên production với quét chủ động để bảo đảm kết quả chính xác nhất
+ Hai kỹ thuật bổ trợ nhau và đều không thay thế threat modeling cho lỗi logic và thiết kế
E: Quét chủ động DAST gửi tấn công thật và có thể làm thay đổi dữ liệu hay gây gián đoạn; chỉ chạy trên môi trường thử bạn sở hữu. SAST và DAST nhìn ở hai góc khác nhau, và cả hai đều bỏ sót lỗi thiết kế mà threat modeling và review bắt được.
Q: Công cụ SCA báo một thư viện có CVE mức Critical (CVSS 9.8) nhưng chưa có bản vá, và đường mã bị ảnh hưởng không được ứng dụng của bạn gọi. Cách xử lý hợp lý?
- Bỏ qua vì chưa có bản vá
+ Ghi nhận và phân loại: đánh giá khả năng tiếp cận (không gọi tới thì rủi ro thấp hơn), cân nhắc biện pháp giảm nhẹ hoặc thay thế, theo dõi bản vá, ghi VEX/ngoại lệ có hạn và người duyệt
- Chặn mọi PR vô thời hạn cho tới khi có bản vá
- Xoá cảnh báo khỏi báo cáo cho khỏi thấy
E: CVSS là điểm kỹ thuật, không phải rủi ro thực của bạn. Khả năng tiếp cận, mức tiếp xúc, và việc có khai thác ngoài thực tế (EPSS, KEV) quyết định ưu tiên. Không có bản vá thì cần giảm nhẹ và theo dõi, không phải bỏ qua hay chặn vô thời hạn.
Q: Khi có một CVE mới nổi tiếng trong một thư viện, bạn cần biết ngay dịch vụ nào đang dùng nó mà không quét lại mọi thứ. Thực hành nào giúp được?
- Hỏi từng nhóm bằng email
+ Sinh SBOM trong mỗi lần build, lưu cùng phiên bản phát hành và tra cứu trong kho SBOM (hoặc quét SBOM đã lưu bằng công cụ như Grype)
- Xoá thư viện khỏi tất cả dịch vụ ngay lập tức
- Chờ nhà cung cấp thông báo
E: SBOM là danh mục thành phần của từng bản phát hành, cho phép trả lời "bản nào dùng thành phần X" trong vài phút. Cần sinh tự động mỗi lần build và lưu trữ có thể tra cứu.
Q: Một đồng nghiệp muốn chạy ZAP "full scan" (chủ động) vào website production của công ty vào buổi chiều cho chắc chắn. Phản hồi đúng nhất?
- Đồng ý vì production là nơi quan trọng nhất cần kiểm tra
+ Không: quét chủ động gửi tấn công thật và có thể thay đổi dữ liệu hoặc gây gián đoạn; chạy trên staging với dữ liệu giả, có phép và báo trước cho vận hành, còn production chỉ dùng kiểm tra bị động an toàn nếu được phép
- Đồng ý nếu quét vào ban đêm
- Chỉ cần tắt WAF trước khi quét
E: DAST chủ động có tác động thật. Quy tắc: chỉ quét hệ thống mình sở hữu hoặc được phép, trên môi trường thử với dữ liệu giả, và phối hợp với vận hành. Tắt WAF còn làm giảm bảo vệ thực.

=== quiz-test | Luyện tập | Luyện tập 2: Kiểm thử tự động
>> 6 câu về chiến lược kiểm thử, độ phủ, đột biến, hợp đồng, E2E và tải.

## Câu hỏi

@quiz
Q: Bộ kiểm thử của dịch vụ chủ yếu là E2E giao diện, chạy 50 phút và đỏ ngẫu nhiên khoảng 15% số lần. Điều chỉnh chiến lược hợp lý nhất?
- Thêm retry vô hạn cho các test E2E
+ Đẩy logic xuống tầng thấp hơn (đơn vị, tích hợp, API), giữ ít E2E cho luồng trọng yếu, cách ly và sửa test flaky, song song hoá
- Chuyển hẳn sang kiểm thử thủ công
- Xoá các test chậm nhất
E: Kim tự tháp ngược (nhiều E2E) chậm, vỡ ngẫu nhiên và khó tìm nguyên nhân. Kiểm thử ở tầng thấp nhanh và ổn định hơn nhiều; E2E chỉ nên phủ các luồng quan trọng. Retry che flaky chứ không sửa.
Q: Dự án có độ phủ mã 95% nhưng vẫn lọt nhiều lỗi vào production. Giải thích hợp lý và cách kiểm tra chất lượng bộ test?
- Độ phủ cao nghĩa là không thể có lỗi
+ Độ phủ chỉ đo dòng mã **được chạy qua**, không đo kiểm thử có khẳng định gì; dùng kiểm thử đột biến (mutation testing) và rà soát assertion để đo khả năng bắt lỗi thật
- Cần tăng độ phủ lên 100%
- Độ phủ không liên quan tới chất lượng và nên bỏ đo
E: Test chạy qua mã mà không `assert` vẫn tăng độ phủ. Kiểm thử đột biến cố ý sửa mã rồi xem test có đỏ không: đột biến sống sót cho thấy chỗ test yếu. Độ phủ hữu ích để tìm chỗ chưa có test, không phải bằng chứng chất lượng.
Q: Hai dịch vụ thường xuyên bị vỡ giao diện khi một bên đổi API mà bên kia không biết, và E2E phát hiện quá muộn. Kỹ thuật phù hợp nhất?
- Thêm nhiều test E2E hơn
+ Kiểm thử hợp đồng hướng người tiêu dùng (như Pact): consumer ghi mong đợi, provider xác minh trong CI trước khi triển khai
- Cấm mọi thay đổi API
- Chỉ dùng tài liệu Swagger không kiểm tra
E: Hợp đồng được xác minh tự động bắt thay đổi phá vỡ sớm, trước triển khai, mà không cần dựng cả hệ thống. Tài liệu không kiểm tra sẽ lỗi thời; E2E bắt muộn và chậm.
Q: Chọn TẤT CẢ thói quen giúp test E2E ổn định hơn.
+ Chọn phần tử theo vai trò và nhãn thay vì CSS mong manh
+ Dùng chờ tự động của công cụ (assertion có chờ) thay vì `sleep` cố định
- Dùng chung một tài khoản và dữ liệu giữa mọi test để tiết kiệm thời gian
+ Mỗi test tự tạo dữ liệu của mình (qua API) và dọn dẹp sau
E: Dữ liệu dùng chung làm test phụ thuộc lẫn nhau và vỡ khi chạy song song. Chọn phần tử bền, chờ thông minh và dữ liệu riêng cho từng test là các biện pháp chính chống flaky.
Q: Bạn đặt ngưỡng k6 là `http_req_duration: ['avg<300']` và thấy luôn đạt, nhưng người dùng vẫn phàn nàn thỉnh thoảng rất chậm. Vấn đề và cách sửa?
- Ngưỡng đúng, người dùng phàn nàn vô cớ
+ Trung bình che đuôi chậm; dùng phân vị (p95, p99) làm ngưỡng, kèm tỷ lệ lỗi, và định SLO trước khi kiểm thử
- Cần giảm số người dùng ảo
- Đo bằng thời gian nhỏ nhất thay vì trung bình
E: Vài yêu cầu rất chậm hầu như không làm trung bình tăng. Phân vị cao (p95/p99) phản ánh trải nghiệm của những người gặp trường hợp xấu nhất.
Q: Bạn muốn chạy kiểm thử tải 500 người dùng ảo vào một API công khai của bên thứ ba mà đội bạn tích hợp, để "xem họ chịu được không". Đánh giá đúng nhất?
- Hợp lý vì chỉ là kiểm thử
+ Không nên: tải vào hệ thống của người khác không có phép có thể bị coi là tấn công từ chối dịch vụ; chỉ kiểm thử tải môi trường bạn sở hữu hoặc được cho phép, dùng giả lập (mock/sandbox) của nhà cung cấp cho phần tích hợp
- Hợp lý nếu chạy ngoài giờ
- Hợp lý nếu dùng nhiều địa chỉ IP
E: Kiểm thử tải hệ thống không thuộc quyền kiểm soát của bạn là rủi ro pháp lý và vận hành. Dùng môi trường thử của nhà cung cấp hoặc giả lập bên thứ ba trong kiểm thử, và đo hành vi chịu lỗi của chính ứng dụng bạn.
