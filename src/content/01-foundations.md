=== overview | Bắt đầu | Tổng quan và cách học
>> Bộ ghi chú về DevSecOps và kiểm thử tự động: đưa bảo mật và chất lượng vào pipeline thay vì kiểm tra ở cuối. Đi từ nguyên tắc và threat modeling, qua kiểm thử đơn vị, tích hợp, E2E, hiệu năng, đến các công cụ quét bí mật, SAST, SCA, container, IaC, DAST và quản lý lỗ hổng.

::: warn Đọc trước khi dùng
Đây là ghi chú **tự học, tự soạn**, không phải tài liệu chính thức của OWASP hay bất kỳ nhà cung cấp công cụ nào. Tên công cụ, cờ dòng lệnh và bộ quy tắc thay đổi nhanh; ghi chú không tra cứu trực tuyến nên có thể lỗi thời, hãy đối chiếu tài liệu hiện hành. **Chỉ quét và kiểm thử hệ thống bạn sở hữu hoặc được cho phép rõ ràng bằng văn bản.** Quét hay tấn công thử hệ thống của người khác là vi phạm pháp luật, dù mục đích là "kiểm tra giúp".
:::

## Bạn sẽ làm được gì sau khi học xong

- Giải thích DevSecOps (dịch chuyển bảo mật sang trái) và vẽ được mô hình mối đe doạ cho một tính năng.
- Thiết kế chiến lược kiểm thử tự động theo tầng, và viết kiểm thử đơn vị, tích hợp, API, E2E và tải.
- Đưa vào pipeline các lớp quét: bí mật, SAST, phụ thuộc và SBOM, container và IaC, DAST.
- Đặt cổng chặn hợp lý, xử lý cảnh báo giả, và quản lý lỗ hổng theo mức rủi ro thực.
- Biết bảo vệ ở thời điểm chạy và đo hiệu quả chương trình bảo mật.

## Cấu trúc

| Nhóm | Trang | Nội dung |
|---|---|---|
| Nền tảng | [Nguyên tắc DevSecOps](#devsecops-principles), [Threat modeling](#threat-modeling) | Tư duy và cách phân tích rủi ro |
| Kiểm thử tự động | [Chiến lược](#test-strategy), [Đơn vị và tích hợp](#unit-integration), [API, E2E, hợp đồng](#e2e-api-contract), [Hiệu năng](#performance-testing) | Chất lượng trong pipeline |
| Công cụ bảo mật | [Quét bí mật](#secret-scanning), [SAST](#sast), [Phụ thuộc và SBOM](#sca-sbom), [Container và IaC](#container-iac-scan), [DAST](#dast), [Runtime](#runtime-security) | Từng lớp quét |
| Vận hành | [Tích hợp pipeline và quản lý lỗ hổng](#pipeline-vuln-mgmt) | Cổng, ngoại lệ, SLA, chỉ số |
| Tham chiếu | [Cheat sheet](#cheatsheet), [Luyện tập: bảo mật](#quiz-sec), [Luyện tập: kiểm thử](#quiz-test) | Tra nhanh và tự kiểm tra |

## Môi trường thực hành

Bạn cần Docker và Python (hoặc Node) trên máy, một repo thử nghiệm. Các lab dùng **mã cố ý có lỗ hổng** do chính bạn tạo cục bộ để quét; không đưa vào hệ thống thật.

```
docker --version
python3 --version
git --version
```

Liên quan: pipeline và phát hành ở [CI/CD](https://tracviethustcs.github.io/cicd-release-notes/), chuỗi cung ứng image ở [Docker](https://tracviethustcs.github.io/docker-container-notes/#security) và [Kubernetes](https://tracviethustcs.github.io/k8s-study-notes/#adv-supply-chain), hạ tầng an toàn ở [IaC](https://tracviethustcs.github.io/iac-terraform-ansible-notes/#iac-security).

=== devsecops-principles | Nền tảng | Nguyên tắc DevSecOps
>> DevSecOps là đưa bảo mật vào từng bước của vòng đời phát triển và biến nó thành trách nhiệm chung, tự động và liên tục, thay vì một cổng kiểm tra cuối cùng do đội bảo mật đứng gác.

## Vì sao cần

Mô hình cũ: phát triển xong, đến cuối mới có một đợt kiểm tra bảo mật. Lỗi tìm thấy muộn thì sửa đắt, trễ hạn, và hai đội đối đầu nhau. DevSecOps đảo lại: kiểm tra **sớm, nhỏ, tự động**, ngay trong luồng làm việc của lập trình viên, và coi bảo mật là một thuộc tính chất lượng như hiệu năng hay độ tin cậy.

## Dịch chuyển trái (shift-left)

```ascii
  Thiết kế -> Viết mã -> Commit/PR -> Build -> Test -> Triển khai -> Vận hành
     |           |           |          |       |         |            |
  threat     IDE lint,    secret scan  SCA,   DAST,   policy,     giám sát,
  modeling   pre-commit   SAST        SBOM,   API     ký/xác      runtime
                                      image   test    minh        detection
  <---- càng sớm càng rẻ để sửa ------------------------------ vẫn cần kiểm tra ở cả bên phải ---->
```

"Dịch chuyển trái" **không** có nghĩa bỏ kiểm tra ở bên phải: có lỗi chỉ lộ khi chạy thật (DAST, cấu hình triển khai, lỗi logic nghiệp vụ). Cần phòng thủ nhiều lớp.

## Các nguyên tắc

1. **Bảo mật là trách nhiệm chung.** Lập trình viên sở hữu bảo mật của mã họ viết; đội bảo mật cung cấp công cụ, hướng dẫn và rà soát chuyên sâu.
2. **Tự động hoá kiểm tra lặp lại** trong pipeline, với phản hồi nhanh ngay trên pull request.
3. **Chia nhỏ và liên tục**: nhiều kiểm tra nhỏ thường xuyên thay vì một đợt kiểm tra lớn.
4. **Đo bằng rủi ro**: không phải mọi cảnh báo đều như nhau; ưu tiên theo khả năng bị khai thác và mức ảnh hưởng.
5. **Bảo mật dưới dạng mã**: chính sách, cấu hình hạ tầng, quy tắc quét đều nằm trong Git, có review.
6. **Phòng thủ nhiều lớp (defense in depth)**: không tin vào một lớp duy nhất.
7. **Quyền tối thiểu và mặc định an toàn.**
8. **Phản hồi có thể hành động**: cảnh báo kèm cách sửa, ít nhiễu, đúng người nhận.
9. **Giả định đã bị xâm nhập**: có phát hiện, ứng phó và khôi phục, không chỉ phòng ngừa.

## OWASP Top 10: danh mục rủi ro ứng dụng web

Tài liệu nhận thức phổ biến liệt kê các nhóm rủi ro hàng đầu. Bản 2021 gồm: Broken Access Control, Cryptographic Failures, Injection, Insecure Design, Security Misconfiguration, Vulnerable and Outdated Components, Identification and Authentication Failures, Software and Data Integrity Failures, Security Logging and Monitoring Failures, Server-Side Request Forgery. (Danh sách được cập nhật định kỳ: kiểm tra bản mới nhất.) Mỗi công cụ trong bộ ghi chú này phủ một vài nhóm trong đó:

| Nhóm rủi ro | Công cụ phát hiện chính |
|---|---|
| Injection, lỗi mã không an toàn | SAST, DAST, kiểm thử |
| Thành phần có lỗ hổng, lỗi phụ thuộc | SCA, SBOM |
| Sai cấu hình bảo mật | Quét container và IaC, DAST, CIS benchmark |
| Lỗi kiểm soát truy cập, thiết kế không an toàn | **Threat modeling, review, kiểm thử logic**: công cụ tự động phát hiện kém |
| Lỗi toàn vẹn phần mềm và chuỗi cung ứng | Ký và xác minh artifact, SBOM, pipeline an toàn |
| Thiếu ghi log và giám sát | Rà soát thiết kế, phát hiện runtime |

Điểm quan trọng: công cụ tự động bắt tốt các lỗi **có mẫu rõ ràng**, nhưng lỗi **logic và thiết kế** (kiểm soát truy cập sai, quy trình nghiệp vụ bị lợi dụng) cần con người, threat modeling và kiểm thử có chủ đích.

## Khung và chuẩn tham khảo

| Khung | Dùng để |
|---|---|
| **OWASP SAMM** | Đánh giá và nâng cấp độ trưởng thành của chương trình bảo mật phần mềm |
| **OWASP DSOMM** | Mô hình trưởng thành riêng cho DevSecOps |
| **NIST SSDF (SP 800-218)** | Bộ thực hành phát triển phần mềm an toàn |
| **SLSA** | Mức bảo đảm cho chuỗi cung ứng build |
| **OWASP ASVS** | Danh sách yêu cầu kiểm tra bảo mật ứng dụng |
| **CIS Benchmarks** | Cấu hình an toàn cho hệ điều hành, Docker, Kubernetes, cloud |

Không cần áp dụng tất cả: chọn một khung làm mốc đánh giá, xác định điểm yếu lớn nhất rồi cải thiện dần.

## Văn hoá và cách làm việc

- **Security champions**: mỗi nhóm có một người quan tâm bảo mật làm cầu nối với đội bảo mật.
- **Đào tạo theo ngữ cảnh**: ví dụ lỗi thật trong chính mã của nhóm, không chỉ bài giảng chung.
- **Ngưỡng chặn thực tế**: chặn những gì nghiêm trọng và có cách sửa; cảnh báo còn lại hiển thị nhưng không chặn. Quá nhiều chặn giả sẽ khiến mọi người tìm cách tắt công cụ.
- **Không đổ lỗi**: lỗ hổng được tìm thấy là thành công của quy trình, không phải lỗi của một người.
- **Đo lường** để cải thiện ([Tích hợp và quản lý lỗ hổng](#pipeline-vuln-mgmt)).

::: lab Đánh giá nhanh hiện trạng của một dự án
Trả lời bằng có/không cho một repo thật, rồi đếm số "không":

1. Có quét bí mật trước khi commit và trong CI không?
2. Có quét phụ thuộc (có CVE đã biết) trong CI không, và có ai theo dõi kết quả?
3. Có kiểm thử tự động chạy mỗi PR không, và chúng đáng tin cậy chứ?
4. Image container và cấu hình hạ tầng có được quét không?
5. Có bước kiểm thử bảo mật khi ứng dụng đang chạy (DAST, kiểm thử API) không?
6. Có quy trình và thời hạn xử lý lỗ hổng không?
7. Đã từng làm threat modeling cho tính năng quan trọng nào chưa?
8. Khi phát hiện bí mật bị lộ, cả nhóm có biết phải làm gì trong 15 phút đầu không?

Mỗi "không" là một mục có thể cải thiện; chọn hai mục rủi ro cao nhất, ít công nhất để làm trước.
:::

=== threat-modeling | Nền tảng | Threat modeling: nghĩ như kẻ tấn công trước khi viết mã
>> Threat modeling là phân tích có hệ thống: hệ thống gồm những gì, có thể bị tấn công thế nào, và ta sẽ làm gì. Làm ở giai đoạn thiết kế là cách rẻ nhất để tìm lỗi thiết kế mà công cụ quét không thấy.

## Bốn câu hỏi

Theo cách đặt vấn đề phổ biến:

1. **Chúng ta đang xây gì?** Vẽ sơ đồ hệ thống.
2. **Có thể sai ở đâu?** Liệt kê mối đe doạ.
3. **Chúng ta sẽ làm gì với chúng?** Giảm thiểu, chấp nhận, chuyển giao hay loại bỏ.
4. **Chúng ta đã làm đủ tốt chưa?** Kiểm tra lại, kiểm thử các biện pháp.

## Bước 1: Sơ đồ luồng dữ liệu (DFD)

Vẽ các thành phần và dữ liệu chảy qua chúng, **đánh dấu ranh giới tin cậy** (nơi quyền tin cậy thay đổi).

```ascii
   [Người dùng]  --HTTPS-->  (Web app)  --SQL-->  [[Database]]
        |                       |
   ranh giới: Internet / DMZ    +--HTTPS--> (Dịch vụ thanh toán bên ngoài)
                                |
                          ranh giới: mạng nội bộ

   ( ) = tiến trình    [ ] = thực thể ngoài    [[ ]] = kho dữ liệu    --> = luồng dữ liệu
```

Mỗi **luồng dữ liệu cắt qua ranh giới tin cậy** là nơi cần xem kỹ nhất: dữ liệu từ phía kém tin cậy phải được kiểm tra.

## Bước 2: STRIDE

STRIDE là danh sách kiểm tra sáu loại mối đe doạ; với mỗi thành phần và luồng, hỏi từng loại:

| Chữ | Mối đe doạ | Vi phạm | Câu hỏi | Biện pháp điển hình |
|---|---|---|---|---|
| **S** | Spoofing (giả mạo danh tính) | Xác thực | Ai đó có thể giả làm người dùng hay dịch vụ khác không? | Xác thực mạnh, MFA, mTLS |
| **T** | Tampering (sửa đổi dữ liệu) | Toàn vẹn | Dữ liệu có thể bị sửa khi truyền hoặc lưu? | Chữ ký, mã xác thực, TLS, kiểm soát ghi |
| **R** | Repudiation (chối bỏ hành vi) | Không thể chối bỏ | Có thể làm gì đó mà không để lại dấu vết? | Nhật ký kiểm toán không sửa được |
| **I** | Information disclosure (lộ thông tin) | Bảo mật | Dữ liệu nhạy cảm có thể lộ ra? | Mã hoá, kiểm soát truy cập, che dữ liệu |
| **D** | Denial of service | Sẵn sàng | Có thể làm hệ thống ngừng phục vụ? | Giới hạn tốc độ, quota, co giãn |
| **E** | Elevation of privilege (nâng quyền) | Phân quyền | Người dùng thường có thể làm việc của quản trị? | Quyền tối thiểu, kiểm tra phân quyền phía máy chủ |

## Bước 3: Xếp hạng rủi ro và quyết định

Ghi mỗi mối đe doạ vào bảng, ước lượng **khả năng xảy ra** và **mức ảnh hưởng**, rồi chọn cách xử lý:

| Mối đe doạ | Loại | Khả năng | Ảnh hưởng | Xử lý | Người chịu trách nhiệm |
|---|---|---|---|---|---|
| Truy cập đơn hàng của người khác bằng cách đổi ID trên URL | E, I | Cao | Cao | Giảm: kiểm tra quyền sở hữu phía máy chủ + test | Nhóm Backend |
| Thông tin thẻ nằm trong log | I | Trung bình | Cao | Giảm: che dữ liệu trong log + quét log | Nhóm Nền tảng |
| Tấn công làm cạn tài nguyên endpoint tìm kiếm | D | Trung bình | Trung bình | Giảm: giới hạn tốc độ | Nhóm Backend |

Bốn cách xử lý: **giảm thiểu** (thêm biện pháp), **chấp nhận** (có ghi nhận và người duyệt), **chuyển giao** (bảo hiểm, nhà cung cấp), **loại bỏ** (bỏ tính năng).

## Các kỹ thuật bổ trợ

- **Abuse case / misuse case**: viết thêm "câu chuyện người dùng xấu": "với tư cách kẻ tấn công, tôi muốn liệt kê các tài khoản hợp lệ để thử mật khẩu".
- **Attack tree**: cây mục tiêu tấn công chia nhỏ thành các bước.
- **Công cụ**: OWASP Threat Dragon, Microsoft Threat Modeling Tool, hoặc chỉ cần bảng trắng và bảng tính. Có thể mô hình hoá dưới dạng mã (threat modeling as code) để lưu trong repo.
- **Khung khác**: PASTA, LINDDUN (cho riêng quyền riêng tư), MITRE ATT&CK (chiến thuật kẻ tấn công), CAPEC.

## Khi nào làm

- Khi **thiết kế** một hệ thống mới hoặc tính năng đáng kể (đặc biệt có xác thực, thanh toán, dữ liệu cá nhân, tích hợp bên ngoài).
- Khi **thay đổi kiến trúc** (thêm dịch vụ, đổi nhà cung cấp, mở API công khai).
- Sau một sự cố bảo mật: cập nhật mô hình theo bài học.

Giữ nó **nhẹ**: một buổi 60 đến 90 phút với người viết mã, người thiết kế và một người có tư duy bảo mật thường đủ. Mô hình hoàn hảo nhưng không ai làm thì vô ích hơn mô hình đơn giản được làm thường xuyên. Kết quả cần chuyển thành **việc cụ thể trong backlog** (và kiểm thử tự động cho các biện pháp quan trọng).

::: lab Threat modeling cho một tính năng "đặt lại mật khẩu"
Vẽ DFD (người dùng, ứng dụng web, dịch vụ email, cơ sở dữ liệu, ranh giới tin cậy), rồi đi qua STRIDE cho từng luồng. Gợi ý các câu hỏi:

- (S) Có thể yêu cầu đặt lại mật khẩu của tài khoản người khác rồi nhận liên kết không?
- (T) Mã đặt lại có thể đoán hoặc dùng lại được không? Có hết hạn không?
- (I) Thông báo lỗi có tiết lộ email nào đã đăng ký không?
- (D) Có thể gửi hàng nghìn email đặt lại làm ngập một người dùng không?
- (R) Việc đặt lại có được ghi vào nhật ký kiểm toán không?
- (E) Quản trị viên có thể đặt lại mật khẩu mà không để lại dấu vết?

Điền kết quả vào bảng rủi ro ở trên, rồi viết **hai kiểm thử tự động** cho hai mối đe doạ rủi ro nhất.
:::

=== test-strategy | Kiểm thử tự động | Chiến lược kiểm thử tự động
>> Kiểm thử tự động đáng giá khi nó nhanh, đáng tin cậy và bắt đúng loại lỗi. Chiến lược quyết định viết loại kiểm thử nào, ở đâu, chạy khi nào, để pipeline vừa nhanh vừa có ý nghĩa.

## Mục tiêu của kiểm thử

- **Phát hiện hồi quy** khi thay đổi mã: lưới an toàn cho việc tái cấu trúc và triển khai thường xuyên.
- **Tài liệu sống** về hành vi mong đợi.
- **Cho phép tự tin triển khai**: nếu không tin vào kiểm thử, người ta sẽ không dám phát hành thường xuyên.

Kiểm thử **không** chứng minh phần mềm không có lỗi; nó giảm rủi ro ở những chỗ bạn đã nghĩ tới.

## Các tầng kiểm thử

| Tầng | Kiểm tra gì | Tốc độ | Độ ổn định | Số lượng |
|---|---|---|---|---|
| **Đơn vị (unit)** | Logic của một hàm/lớp, cô lập phụ thuộc | Mili giây | Rất cao | Nhiều nhất |
| **Tích hợp (integration)** | Các phần phối hợp: mã với database thật, hàng đợi, API nội bộ | Giây | Cao | Vừa |
| **Hợp đồng (contract)** | Giao diện giữa hai dịch vụ không bị phá vỡ | Nhanh | Cao | Theo số giao diện |
| **API / chức năng** | Hành vi qua HTTP của dịch vụ đang chạy | Giây | Trung bình đến cao | Vừa |
| **End-to-end (E2E)** | Luồng người dùng qua giao diện thật | Chậm | Thấp hơn, dễ vỡ | Ít nhất |
| **Hiệu năng/tải** | Độ trễ, thông lượng, giới hạn | Phút | Phụ thuộc môi trường | Theo kịch bản trọng yếu |
| **Bảo mật** | SAST, DAST, kiểm thử logic phân quyền | Khác nhau | Khác nhau | Theo rủi ro |

Quy tắc hình kim tự tháp: **nhiều kiểm thử nhanh ở dưới, ít kiểm thử chậm ở trên**. Ngược lại (chủ yếu E2E) làm pipeline chậm, vỡ ngẫu nhiên và khó tìm nguyên nhân. Một số đội dùng "hình huy chương" hoặc "tổ ong" (nhiều kiểm thử tích hợp hơn) cho kiến trúc vi dịch vụ; điều cốt lõi vẫn là cân bằng chi phí, tốc độ và độ tin cậy.

## Bản đồ công cụ

| Tầng | Python | JavaScript/TypeScript | Java | Đa ngôn ngữ |
|---|---|---|---|---|
| Đơn vị | pytest, unittest | Jest, Vitest | JUnit | |
| Tích hợp | pytest + Testcontainers | Jest + Testcontainers | JUnit + Testcontainers | Docker Compose |
| API | pytest + httpx/requests, Schemathesis | supertest | REST-assured | Postman/Newman, Karate |
| Hợp đồng | Pact | Pact | Pact | Pact, Spring Cloud Contract |
| E2E | Playwright, Selenium | Playwright, Cypress | Selenium | Playwright |
| Tải | Locust | k6 | Gatling | k6, JMeter |
| Đột biến | mutmut | Stryker | PIT | |

Tên công cụ để tham khảo: chọn theo ngôn ngữ và đội ngũ của bạn; quan trọng là thói quen hơn công cụ.

## Nguyên tắc viết kiểm thử tốt

1. **Độc lập và lặp lại được**: chạy theo thứ tự nào cũng được, chạy song song được, cho cùng kết quả.
2. **Nhanh**: kiểm thử chậm sẽ bị bỏ qua.
3. **Kiểm tra hành vi, không phải cài đặt**: kiểm thử gắn chặt vào chi tiết nội bộ sẽ vỡ mỗi lần tái cấu trúc.
4. **Một ý chính mỗi kiểm thử**, tên mô tả được điều đang kiểm tra (`test_dat_hang_khi_het_hang_tra_loi_409`).
5. **Có assertion ý nghĩa**: kiểm thử chạy qua mã mà không khẳng định gì thì vô giá trị.
6. **Dữ liệu kiểm thử tạo trong chính kiểm thử** và dọn sau; không dựa vào dữ liệu có sẵn trên môi trường dùng chung.
7. **Không phụ thuộc thời gian thực và dịch vụ ngoài** nếu có thể: dùng giả lập (mock, fake), đồng hồ cố định, container thay thế.
8. **Kiểm thử cả đường lỗi và biên**, không chỉ đường "vui".

## Test không ổn định (flaky)

Test lúc đỏ lúc xanh làm hỏng niềm tin vào cả pipeline. Nguyên nhân thường gặp: phụ thuộc thứ tự chạy, `sleep` cố định, dữ liệu dùng chung, thời gian thực, dịch vụ ngoài, tranh chấp tài nguyên khi chạy song song. Cách xử lý: đo, cách ly (quarantine) khỏi đường chặn, có người chịu trách nhiệm sửa gốc, không chỉ thêm retry. Xem thêm [Chiến lược kiểm thử trong pipeline](https://tracviethustcs.github.io/cicd-release-notes/#testing-strategy).

## Chạy khi nào trong pipeline

| Sự kiện | Chạy |
|---|---|
| Mỗi commit/PR | Lint, kiểm thử đơn vị, tích hợp nhanh, hợp đồng, quét bí mật, SAST, SCA |
| Sau merge, trên môi trường thử | API/chức năng, một số E2E trọng yếu (smoke), DAST cơ bản |
| Theo lịch (đêm) | E2E đầy đủ, kiểm thử tải, DAST chủ động, quét lại phụ thuộc |
| Sau triển khai | Smoke test và kiểm tra sức khoẻ |
| Trước phát hành lớn | Kiểm thử tải, rà soát bảo mật sâu |

## Chỉ số của bộ kiểm thử

- **Thời gian phản hồi** của pipeline (mục tiêu thường là vài phút cho vòng đầu).
- **Tỷ lệ flaky** và số kiểm thử bị cách ly.
- **Độ phủ mã** (coverage): hữu ích để tìm chỗ **chưa có** kiểm thử, nhưng không chứng minh chất lượng; đừng biến thành mục tiêu tuyệt đối.
- **Điểm đột biến (mutation score)**: tỷ lệ "đột biến" mà bộ kiểm thử phát hiện: đo khả năng bắt lỗi thật.
- **Tỷ lệ lỗi thoát (escape rate)**: lỗi lọt ra production dù đã qua kiểm thử, cho biết khe hở.

::: lab Lập bản đồ kiểm thử cho một dịch vụ
Chọn một dịch vụ bạn biết. Điền bảng: với mỗi tầng ở trên, hiện có bao nhiêu kiểm thử, chạy mất bao lâu, chạy khi nào, và chúng đáng tin cậy ra sao? Rồi trả lời:

1. Lỗi gần nhất lọt ra production là gì, và tầng kiểm thử nào lẽ ra bắt được nó?
2. Có kiểm thử nào bạn không dám sửa mã vì sợ làm vỡ? Đó là dấu hiệu kiểm thử gắn vào cài đặt.
3. Kiểm thử chậm nhất chiếm bao nhiêu phần thời gian pipeline và có thể xuống tầng thấp hơn không?
:::
