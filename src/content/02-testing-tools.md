=== unit-integration | Kiểm thử tự động | Kiểm thử đơn vị và tích hợp
>> Kiểm thử đơn vị cho phản hồi tức thì về logic; kiểm thử tích hợp bảo đảm các phần phối hợp đúng với database và dịch vụ thật. Ví dụ dùng Python và pytest, nhưng nguyên tắc áp dụng cho mọi ngôn ngữ.

## Kiểm thử đơn vị với pytest

```
# app/pricing.py
def apply_discount(price: float, percent: float) -> float:
    if price < 0:
        raise ValueError("giá không được âm")
    if not 0 <= percent <= 100:
        raise ValueError("phần trăm phải trong 0..100")
    return round(price * (1 - percent / 100), 2)
```

```
# tests/test_pricing.py
import pytest
from app.pricing import apply_discount

def test_giam_gia_binh_thuong():
    assert apply_discount(200.0, 25) == 150.0

def test_khong_giam():
    assert apply_discount(99.99, 0) == 99.99

@pytest.mark.parametrize("price,percent", [(-1, 10), (100, -5), (100, 101)])
def test_dau_vao_sai_bi_tu_choi(price, percent):
    with pytest.raises(ValueError):
        apply_discount(price, percent)
```

```
pip install pytest pytest-cov
pytest -q
pytest --cov=app --cov-report=term-missing --cov-fail-under=80
```

Chú ý các thói quen: kiểm thử cả đường lỗi và giá trị biên (0, 100, âm), dùng `parametrize` thay vì nhân bản mã, đặt tên nói lên điều kiểm tra.

## Giả lập (mock) và khi nào dùng

Giả lập thay thế phụ thuộc chậm hoặc khó kiểm soát (mạng, thời gian, dịch vụ ngoài) bằng bản giả có hành vi định sẵn:

```
from unittest.mock import patch
from app.notify import send_welcome

def test_gui_email_chao_mung():
    with patch("app.notify.smtp_send") as fake:
        send_welcome("an@example.com")
        fake.assert_called_once_with("an@example.com", subject="Chào mừng")
```

Lưu ý: giả lập quá nhiều làm kiểm thử chỉ kiểm tra rằng bạn gọi mock đúng cách, không kiểm tra hệ thống thật chạy được. Quy tắc: **giả lập ở ranh giới hệ thống** (dịch vụ bên ngoài), **không** giả lập chính mã bạn đang kiểm thử, và dùng kiểm thử tích hợp để kiểm tra phần dính vào hạ tầng thật.

## Kiểm thử tích hợp với container thật

Dùng database thật trong container thay vì giả lập: bắt được lỗi SQL, ràng buộc, migration, hành vi thực của driver. Thư viện Testcontainers khởi động container cho mỗi lần chạy kiểm thử rồi dọn:

```
pip install testcontainers[postgres] sqlalchemy psycopg2-binary pytest
```

```
# tests/test_repo_integration.py
import pytest
from sqlalchemy import create_engine, text
from testcontainers.postgres import PostgresContainer

@pytest.fixture(scope="module")
def engine():
    with PostgresContainer("postgres:16") as pg:
        eng = create_engine(pg.get_connection_url())
        with eng.begin() as c:
            c.execute(text("create table users(id serial primary key, email text unique not null)"))
        yield eng

def test_email_khong_duoc_trung(engine):
    with engine.begin() as c:
        c.execute(text("insert into users(email) values ('an@example.com')"))
    with pytest.raises(Exception):
        with engine.begin() as c:
            c.execute(text("insert into users(email) values ('an@example.com')"))
```

Trong CI, cách khác là dùng `services` của runner (GitHub Actions) hoặc Docker Compose; xem [GitHub Actions](https://tracviethustcs.github.io/cicd-release-notes/#gha).

## Độ phủ và kiểm thử đột biến

**Độ phủ mã** cho biết dòng nào **chưa** được chạy qua, nhưng không cho biết kiểm thử có kiểm tra gì. Ví dụ kiểm thử gọi hàm mà không có `assert` vẫn tăng độ phủ.

**Kiểm thử đột biến (mutation testing)** đo chất lượng kiểm thử bằng cách cố ý sửa mã (đổi `>` thành `>=`, `+` thành `-`) rồi chạy lại kiểm thử: nếu kiểm thử vẫn xanh thì "đột biến sống sót", nghĩa là kiểm thử không bắt được thay đổi đó.

```
pip install mutmut
mutmut run --paths-to-mutate=app/
mutmut results
mutmut show 3          # xem đột biến số 3 đã sống sót thế nào
```

Công cụ tương tự: Stryker (JavaScript, .NET), PIT (Java). Chạy chậm nên dùng theo lịch hoặc trên phần mã quan trọng, không phải mọi PR.

## Kiểm thử bảo mật ở mức đơn vị

Viết kiểm thử cho **hành vi bảo mật** như kiểm thử bình thường:

```
def test_nguoi_dung_a_khong_xem_duoc_don_cua_b(client, user_a, order_of_b):
    r = client.get(f"/orders/{order_of_b.id}", headers=auth(user_a))
    assert r.status_code in (403, 404)

def test_dau_vao_co_ky_tu_dac_biet_khong_gay_loi_sql(client, user_a):
    r = client.get("/search", params={"q": "' OR 1=1 --"}, headers=auth(user_a))
    assert r.status_code == 200
    assert r.json()["results"] == []
```

Đây là cách biến kết quả threat modeling ([Threat modeling](#threat-modeling)) thành kiểm thử hồi quy tự động: lỗi phân quyền không thể quay lại mà không làm kiểm thử đỏ.

::: lab Viết kiểm thử và đo độ phủ, đột biến
```
mkdir -p ~/ut/app ~/ut/tests && cd ~/ut && touch app/__init__.py
cat > app/pricing.py <<'EOF'
def apply_discount(price: float, percent: float) -> float:
    if price < 0:
        raise ValueError("giá không được âm")
    if not 0 <= percent <= 100:
        raise ValueError("phần trăm phải trong 0..100")
    return round(price * (1 - percent / 100), 2)
EOF
cat > tests/test_pricing.py <<'EOF'
import pytest
from app.pricing import apply_discount
def test_giam_gia(): assert apply_discount(200.0, 25) == 150.0
@pytest.mark.parametrize("p,c", [(-1, 10), (100, -5), (100, 101)])
def test_sai(p, c):
    with pytest.raises(ValueError): apply_discount(p, c)
EOF
pip install -q pytest pytest-cov mutmut && pytest -q --cov=app --cov-report=term-missing
# thử cố ý làm yếu: bỏ test biên 100 rồi chạy mutmut để xem đột biến sống sót
```
Quan sát: độ phủ có thể 100% nhưng đột biến nào sống sót (ví dụ đổi `<= 100` thành `< 100`)? Kiểm thử nào cần thêm để bắt nó?
:::

=== e2e-api-contract | Kiểm thử tự động | Kiểm thử API, E2E và hợp đồng
>> Ba loại kiểm thử ở tầng trên của kim tự tháp: API kiểm tra dịch vụ qua HTTP, hợp đồng kiểm tra giao diện giữa các dịch vụ, E2E kiểm tra luồng người dùng qua giao diện. Mỗi loại có điểm mạnh và chi phí riêng.

## Kiểm thử API

Kiểm thử dịch vụ đang chạy qua HTTP: nhanh hơn và ổn định hơn E2E giao diện, và gần với cách client thật dùng.

```
# tests/test_api.py
import httpx, os

BASE = os.getenv("BASE_URL", "http://localhost:8000")

def test_tao_va_doc_don_hang():
    with httpx.Client(base_url=BASE) as c:
        r = c.post("/orders", json={"sku": "A1", "qty": 2})
        assert r.status_code == 201
        oid = r.json()["id"]
        r = c.get(f"/orders/{oid}")
        assert r.status_code == 200 and r.json()["qty"] == 2

def test_thieu_truong_bat_buoc_tra_422():
    with httpx.Client(base_url=BASE) as c:
        assert c.post("/orders", json={"sku": "A1"}).status_code == 422
```

- Kiểm tra **mã trạng thái, cấu trúc phản hồi, lỗi và biên**, không chỉ đường vui.
- Postman với Newman (chạy bộ sưu tập trong CI) là lựa chọn không cần viết mã nhiều: `newman run collection.json -e env.json`.

### Kiểm thử dựa trên đặc tả (property-based từ OpenAPI)

Nếu dịch vụ có tài liệu OpenAPI, công cụ như **Schemathesis** tự sinh hàng loạt yêu cầu từ đặc tả (kể cả giá trị biên và bất thường) để tìm lỗi 500, vi phạm đặc tả và lỗi xử lý đầu vào:

```
pip install schemathesis
schemathesis run http://localhost:8000/openapi.json --checks all
```

Rất hiệu quả để bắt lỗi không ai nghĩ tới ở API; chạy trên môi trường thử, không phải production.

## Kiểm thử hợp đồng (contract testing)

Trong vi dịch vụ, một dịch vụ sửa API có thể làm vỡ dịch vụ khác mà kiểm thử của riêng nó không biết. **Hợp đồng hướng người tiêu dùng (consumer-driven contract)**, ví dụ với Pact, đảo lại: người tiêu dùng (consumer) ghi lại điều nó mong đợi từ nhà cung cấp (provider), nhà cung cấp chạy kiểm thử để xác minh vẫn đáp ứng.

```ascii
   Consumer (dịch vụ gọi)                       Provider (dịch vụ được gọi)
   -----------------------                       ---------------------------
   kiểm thử viết mong đợi  --> file hợp đồng --> CI của provider chạy xác minh
   "GET /users/1 -> {id,name}"    (pact)        "API thật có trả đúng như vậy không?"
                                    |
                           lưu ở Pact Broker; can-i-deploy kiểm tra
                           trước khi triển khai một phiên bản
```

Ưu điểm: bắt thay đổi phá vỡ **trước khi triển khai**, không cần dựng cả hệ thống để kiểm thử E2E. Chi phí: cần kỷ luật giữa các đội và công cụ trung gian.

## Kiểm thử E2E với Playwright

E2E mô phỏng người dùng thật qua trình duyệt. Chậm và dễ vỡ hơn, nên dành cho **một số luồng trọng yếu** (đăng nhập, thanh toán).

```
pip install pytest-playwright
playwright install chromium
```

```
# tests/test_e2e.py
from playwright.sync_api import Page, expect

def test_dang_nhap_va_thay_trang_tong_quan(page: Page):
    page.goto("https://app.example.com/login")
    page.get_by_label("Email").fill("qa@example.com")
    page.get_by_label("Mật khẩu").fill("matkhau-thu")
    page.get_by_role("button", name="Đăng nhập").click()
    expect(page.get_by_role("heading", name="Tổng quan")).to_be_visible()
```

```
pytest tests/test_e2e.py --headed          # xem trình duyệt chạy
pytest tests/test_e2e.py --tracing on      # lưu trace để gỡ lỗi (xem bằng: playwright show-trace)
```

Thói quen để E2E ổn định:

- **Chọn phần tử theo vai trò và nhãn** (`get_by_role`, `get_by_label`), không theo CSS mong manh.
- **Dùng chờ tự động** của công cụ (`expect(...).to_be_visible()`), **không** `sleep` cố định.
- **Mỗi kiểm thử tự tạo dữ liệu của mình** (qua API) và dọn, không dựa dữ liệu chung.
- Dùng **tài khoản và dữ liệu kiểm thử riêng**, không dùng tài khoản thật.
- Lưu **video, ảnh chụp, trace** khi lỗi để gỡ lỗi nhanh.
- Giữ số lượng nhỏ; đẩy logic xuống tầng thấp hơn.

## Chọn tầng nào cho loại lỗi nào

| Loại lỗi | Tầng kiểm thử hiệu quả nhất |
|---|---|
| Logic tính toán sai | Đơn vị |
| Truy vấn SQL sai, ràng buộc database | Tích hợp (database thật) |
| API trả sai mã, sai cấu trúc | API |
| Thay đổi API làm vỡ dịch vụ khác | Hợp đồng |
| Luồng người dùng bị hỏng giữa các trang | E2E |
| Lỗi 500 với đầu vào lạ | Schemathesis, fuzzing |
| Chậm khi tải cao | Hiệu năng |

::: lab Kiểm thử API và sinh yêu cầu từ OpenAPI
```
pip install -q fastapi uvicorn httpx pytest schemathesis
mkdir -p ~/api && cd ~/api
cat > main.py <<'EOF'
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel, Field
app = FastAPI()
DB = {}
class Order(BaseModel):
    sku: str = Field(min_length=1)
    qty: int = Field(gt=0)
@app.post("/orders", status_code=201)
def create(o: Order):
    oid = len(DB) + 1; DB[oid] = o.model_dump(); return {"id": oid, **DB[oid]}
@app.get("/orders/{oid}")
def read(oid: int):
    if oid not in DB: raise HTTPException(404)
    return {"id": oid, **DB[oid]}
EOF
uvicorn main:app --port 8000 &
sleep 2
curl -s -X POST localhost:8000/orders -H 'content-type: application/json' -d '{"sku":"A1","qty":2}'
curl -s -o /dev/null -w "%{http_code}\n" -X POST localhost:8000/orders -H 'content-type: application/json' -d '{"sku":"A1"}'   # 422
schemathesis run http://localhost:8000/openapi.json --checks all 2>&1 | tail -15
kill %1
```
Ghi lại: Schemathesis báo gì? Thử sửa API cố ý để trả 500 với `qty` rất lớn và xem công cụ có tìm ra không.
:::

=== performance-testing | Kiểm thử tự động | Kiểm thử hiệu năng và tải
>> Hiệu năng là một tính năng. Kiểm thử tải tìm ra giới hạn, nút thắt và hồi quy hiệu năng trước khi người dùng gặp, và có thể là một cổng trong pipeline.

## Các loại kiểm thử hiệu năng

| Loại | Mục tiêu | Cách |
|---|---|---|
| **Load (tải)** | Hệ thống chịu được tải kỳ vọng | Tải ổn định ở mức dự kiến trong một thời gian |
| **Stress (căng)** | Tìm điểm gãy và cách hỏng | Tăng tải vượt mức cho tới khi hỏng |
| **Spike (đột biến)** | Phản ứng với tăng vọt đột ngột | Tải tăng vọt trong thời gian ngắn |
| **Soak / endurance (bền)** | Rò rỉ tài nguyên, suy giảm theo thời gian | Tải vừa phải trong nhiều giờ |
| **Smoke** | Kịch bản chạy được không | Vài người dùng, vài giây |
| **Breakpoint** | Công suất tối đa | Tăng dần tới khi vượt ngưỡng chấp nhận |

## Chỉ số cần quan tâm

- **Độ trễ**: dùng **phân vị** (p50, p95, p99), không dùng trung bình (trung bình che đuôi chậm).
- **Thông lượng**: số yêu cầu mỗi giây (RPS).
- **Tỷ lệ lỗi**: phần trăm yêu cầu thất bại.
- **Bão hoà tài nguyên**: CPU, bộ nhớ, kết nối database, hàng đợi (xem [Phân tích hiệu năng](https://tracviethustcs.github.io/linux-network-notes/#lx-perf)).

Nguyên tắc: định **SLO** trước ("p95 dưới 300 ms ở 200 RPS, lỗi dưới 0,1%") rồi kiểm tra so với nó, thay vì chỉ "chạy rồi xem".

## k6: kiểm thử tải bằng mã

k6 viết kịch bản bằng JavaScript, có **ngưỡng (thresholds)** để biến kết quả thành đạt/trượt (hữu ích làm cổng CI).

```
// load.js
import http from 'k6/http';
import { check, sleep } from 'k6';

export const options = {
  stages: [
    { duration: '30s', target: 20 },   // tăng dần lên 20 người dùng ảo
    { duration: '1m',  target: 20 },   // giữ ổn định
    { duration: '10s', target: 0 },    // giảm xuống
  ],
  thresholds: {
    http_req_failed:   ['rate<0.01'],                  // lỗi dưới 1%
    http_req_duration: ['p(95)<500', 'p(99)<1000'],    // p95 dưới 500 ms
  },
};

export default function () {
  const r = http.get(`${__ENV.BASE_URL || 'http://localhost:8000'}/orders/1`);
  check(r, { 'trạng thái 200 hoặc 404': (res) => res.status === 200 || res.status === 404 });
  sleep(1);
}
```

```
k6 run load.js
k6 run -e BASE_URL=https://staging.example.com load.js
docker run --rm -i grafana/k6 run - < load.js
```

Nếu ngưỡng bị vi phạm, k6 thoát với mã khác 0: pipeline đỏ. Locust (Python), Gatling (Scala/Java) và JMeter là các lựa chọn khác.

## Thiết kế kiểm thử tải có ý nghĩa

- **Môi trường giống production** (cấu hình, kích thước dữ liệu, số bản sao): kết quả trên máy yếu không nói gì về production.
- **Dữ liệu thực tế**: bảng rỗng cho kết quả lạc quan quá.
- **Kịch bản giống người dùng thật**: pha trộn các thao tác theo tỷ lệ thật, có thời gian suy nghĩ (`sleep`).
- **Tránh nút thắt ở máy tạo tải**: máy sinh tải cũng cần đủ tài nguyên, và nên nằm gần mục tiêu.
- **Coordinated omission**: công cụ đo sai nếu chờ phản hồi chậm rồi mới gửi tiếp; dùng mô hình tải theo tốc độ đến (arrival-rate executors của k6) để mô phỏng đúng.
- **Chạy lặp lại** và so với đường cơ sở để phát hiện hồi quy.

## Trong pipeline

| Cấp | Cách |
|---|---|
| Mỗi PR/merge | **Smoke** rất nhẹ (vài người dùng, vài giây) cho các endpoint chính, để bắt hồi quy tồi tệ |
| Theo lịch (đêm) hoặc trước phát hành | Kiểm thử tải đầy đủ trên môi trường staging, so với kết quả lần trước |
| Sự kiện lớn (đợt khuyến mãi) | Kiểm thử stress và spike có chuẩn bị |

Gắn kết quả với giám sát ([DORA và đo lường](https://tracviethustcs.github.io/cicd-release-notes/#metrics-dora)); khi thấy hồi quy, tìm commit gây ra bằng cách chạy lại trên các phiên bản trước.

::: warn Chỉ kiểm thử tải hệ thống của bạn
Kiểm thử tải vào hệ thống của người khác (hoặc dịch vụ bên thứ ba, API công khai) có thể bị coi là tấn công từ chối dịch vụ. Chỉ chạy trên môi trường bạn sở hữu hoặc được phép, và báo trước cho bộ phận vận hành.
:::

::: lab Kiểm thử tải nhỏ với ngưỡng
```
# giả sử API ở lab trước đang chạy ở cổng 8000
cat > load.js <<'EOF'
import http from 'k6/http';
import { check, sleep } from 'k6';
export const options = {
  vus: 10, duration: '20s',
  thresholds: { http_req_failed: ['rate<0.01'], http_req_duration: ['p(95)<300'] },
};
export default function () {
  const r = http.get('http://localhost:8000/orders/1');
  check(r, { 'có phản hồi': (res) => res.status === 200 || res.status === 404 });
  sleep(0.5);
}
EOF
docker run --rm -i --network host grafana/k6 run - < load.js; echo "mã thoát: $?"
```
Đọc phần tóm tắt: p95 là bao nhiêu? Ngưỡng có đạt không? Thử đặt ngưỡng `p(95)<1` (rất chặt) để thấy pipeline "đỏ" thế nào (mã thoát khác 0).
:::

=== secret-scanning | Công cụ bảo mật | Quét bí mật
>> Khoá API, mật khẩu và token lỡ commit vào Git là một trong những cách kẻ tấn công vào hệ thống dễ nhất. Quét bí mật là lớp rẻ nhất, hiệu quả nhất, và nên là lớp đầu tiên bạn bật.

## Vì sao nguy hiểm

- Bí mật trong repo tồn tại **vĩnh viễn trong lịch sử Git** kể cả khi bạn xoá khỏi nhánh hiện tại.
- Kho mã bị sao chép (fork, clone, backup, bộ nhớ đệm). Bot tự động quét GitHub công khai và khai thác khoá bị lộ chỉ trong vài phút.
- Một khoá cloud bị lộ có thể dẫn tới đào tiền ảo trên tài khoản của bạn, lộ dữ liệu hay mất toàn bộ hạ tầng.

## Phòng ngừa nhiều lớp

| Lớp | Cách | Ghi chú |
|---|---|---|
| **Không có bí mật trong mã** | Dùng kho bí mật, biến môi trường, OIDC ([Bí mật và OIDC](https://tracviethustcs.github.io/cicd-release-notes/#secrets-oidc)) | Phòng ngừa tốt nhất |
| **Pre-commit** | Quét tệp sắp commit và chặn trước khi vào lịch sử | Phản hồi tức thì, nhưng có thể bị bỏ qua (`--no-verify`) |
| **Trên PR/CI** | Quét thay đổi của PR | Không thể bỏ qua nếu bắt buộc |
| **Push protection của nền tảng** | GitHub, GitLab chặn push chứa mẫu bí mật đã biết | Bật ở cấp tổ chức |
| **Quét toàn bộ lịch sử và định kỳ** | Tìm bí mật cũ chưa ai phát hiện | Chạy theo lịch |
| **Xác minh còn hiệu lực** | Công cụ thử xác thực bí mật tìm được để biết còn dùng được không | Giúp ưu tiên xử lý |

## gitleaks

```
gitleaks detect --source . -v                                  # quét toàn bộ lịch sử Git của repo
gitleaks detect --source . --report-format sarif --report-path gitleaks.sarif
gitleaks protect --staged -v                                    # quét thay đổi đã `git add` (tên lệnh con thay đổi theo phiên bản)
```

Cấu hình `.gitleaks.toml` cho phép thêm quy tắc riêng và **bỏ qua** (allowlist) các kết quả giả có chủ đích:

```
title = "Quy tắc của dự án"
[extend]
useDefault = true

[[rules]]
id = "noi-bo-api-key"
description = "Khoá API nội bộ"
regex = '''acme_(live|test)_[A-Za-z0-9]{24}'''

[allowlist]
description = "Bỏ qua dữ liệu mẫu"
paths = ['''tests/fixtures/.*''']
regexes = ['''EXAMPLE_KEY_DO_NOT_USE''']
```

Dùng cùng pre-commit:

```
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/gitleaks/gitleaks
    rev: v8.21.2          # ghim phiên bản
    hooks:
      - id: gitleaks
```
```
pip install pre-commit && pre-commit install
```

**trufflehog** là công cụ tương tự, nổi bật ở việc **xác minh** bí mật còn hiệu lực: `trufflehog git file://. --only-verified`.

## Khi phát hiện bí mật đã lộ

Thứ tự quan trọng:

1. **Thu hồi và xoay vòng ngay** (vô hiệu hoá khoá cũ, cấp khoá mới). Đây là bước **bắt buộc**, làm trước mọi thứ.
2. Kiểm tra nhật ký xem khoá có bị dùng trái phép không (CloudTrail, nhật ký của dịch vụ).
3. Xoá khỏi mã và đưa bí mật mới vào kho bí mật.
4. (Tuỳ chọn) Làm sạch lịch sử Git bằng `git filter-repo` hoặc BFG. Việc này **không** thay thế bước 1, vì bản sao đã tồn tại ở nơi khác.
5. Rút bài học: vì sao nó lọt qua, lớp nào cần bổ sung.

::: warn Xoá khỏi Git không làm bí mật an toàn trở lại
Sau khi bí mật bị push lên nơi người khác truy cập được, hãy coi như **đã bị lộ**. Làm sạch lịch sử chỉ dọn dấu vết; chỉ việc xoay vòng mới đóng cửa.
:::

## Giảm cảnh báo giả

- Tinh chỉnh quy tắc và allowlist cho dữ liệu mẫu, tệp kiểm thử; giữ allowlist trong Git để review.
- Ưu tiên kết quả **đã xác minh** hoặc có entropy và mẫu rõ ràng.
- Đặt ngưỡng chặn ở mức **bí mật có khả năng thật**; cảnh báo còn lại chỉ hiển thị.

::: lab Quét thử một repo có bí mật giả
```
rm -rf ~/sec-lab && mkdir -p ~/sec-lab && cd ~/sec-lab && git init -q .
python3 - <<'EOF'
# Ghép chuỗi giả lúc chạy: ghi chú này cố ý không chứa chuỗi giống bí mật thật
# (nếu có, GitHub push protection sẽ chặn việc đẩy repo lên, đúng như thiết kế).
secrets = {
    "GITHUB_TOKEN": "ghp_" + "aB3dE6gH9jK2mN5pQ8sT1vW4yZ7bC0eF3hJ6",
    "SLACK_TOKEN": "xox" + "b-263594206564-1234567890123-aB3dE6gH9jK2mN5pQ8sT1vW4",
    "STRIPE_KEY": "sk_" + "live_4eC39HqLyjWDarjtT1zdp7dc4eC39Hq",
}
with open("config.py", "w") as f:
    for k, v in secrets.items():
        f.write(f'{k} = "{v}"\n')
    f.write('DB_URL = "postgres://admin:SuperSecret123@db.internal:5432/app"\n')
EOF
git add . && git -c user.name=t -c user.email=t@t commit -qm "thêm cấu hình"
gitleaks detect --source . -v 2>&1 | head -30
echo 'KEY = os.environ["KEY"]' > config.py && git -c user.name=t -c user.email=t@t commit -qam "bỏ bí mật khỏi mã"
gitleaks detect --source . 2>&1 | tail -3      # vẫn thấy vì còn trong lịch sử
```
Lưu ý: chính việc tài liệu này không chứa chuỗi giống bí mật thật cho thấy push protection hoạt động. Quan sát: gitleaks tìm ra mấy bí mật (các giá trị trong lab là chuỗi giả, không có hiệu lực)? Sau khi xoá bí mật ở commit mới, `gitleaks detect` vẫn báo gì, và điều đó nói gì về việc xoay vòng khoá? Chuỗi kết nối database có mật khẩu dạng văn bản rõ có bị báo không (quy tắc mặc định có thể không bắt, hãy thêm quy tắc riêng nếu cần)? Lưu ý: gitleaks cố ý bỏ qua các khoá ví dụ công khai như `AKIAIOSFODNN7EXAMPLE` của tài liệu AWS.
:::
