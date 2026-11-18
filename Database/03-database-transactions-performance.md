---
title: "Database: dữ liệu đúng, transaction đúng, truy vấn đo được"
description: "Giáo trình thực hành PostgreSQL 17, đối chiếu MySQL/InnoDB và PlanetScale, dành cho backend engineer."
shell: bash
---

# 03 — Database: từ mô hình dữ liệu đến vận hành trên VPS

> **Đích đến:** bạn giải thích được vì sao một thiết kế giữ đúng dữ liệu khi có cạnh tranh, chứng minh được điều đó bằng thí nghiệm, đọc được kế hoạch truy vấn, và tự khôi phục được database từ bản sao lưu. Khả năng này quan trọng hơn việc thuộc tên index hoặc một bộ thông số cấu hình.
>
> **Nền thực hành:** PostgreSQL 17, image `postgres:17-bookworm`; Docker Compose v2; Node.js 24 chỉ làm bộ điều phối thí nghiệm. Các phần MySQL dùng InnoDB 8.4, chạy tùy chọn. Ngày đối chiếu tài liệu chính thức: **2026-11-17**. Tag theo major có thể thay đổi bản vá; lab ghi lại phiên bản thực tế và image digest.
>
> **Trạng thái kiểm chứng khi biên soạn:** các cell Bash và script JavaScript đã qua kiểm tra cú pháp tĩnh. Môi trường biên soạn không có Docker daemon, PostgreSQL, MySQL hay PgBouncer; các kết quả SQL/runtime bên dưới là **tiêu chí cần kiểm tra trên máy học viên**, không phải số đo đã chạy trong môi trường biên soạn. Không có cam kết thời gian thực thi cố định.

## Cách học để biến kiến thức thành năng lực

Mỗi bài có một câu hỏi cần trả lời, điều kiện đầu vào, cell tạo tình huống, tiêu chí đúng, cách tìm nguyên nhân nếu kết quả khác, và giới hạn khi chuyển sang production. Đọc phần dự đoán trước khi chạy. Sau khi chạy, lưu bằng chứng và viết ba câu: tôi thấy gì, cơ chế nào giải thích được, và thử nghiệm này chưa chứng minh điều gì. Một kết quả nhanh nhưng sai dữ liệu là thất bại; một kết quả đúng nhưng chưa có kế hoạch phục hồi cũng chưa đủ để vận hành.

Bạn không cần học tất cả trong một ngày. Lượt đầu đi qua mô hình dữ liệu, transaction, index cơ bản và backup/restore. Lượt hai quay lại thống kê, contention, outbox, migration và pooling. Lượt ba dùng capstone cuối sách: bạn phải tự chọn bằng chứng để thuyết phục một người review mà không chỉ dẫn họ bấm lại từng cell.

**Quy ước chạy:**

- **LOCAL:** chạy trên máy học tập, trong `${HOME}/backend-labs/database`.
- **VPS:** chỉ chạy trên VPS học tập có SSH; database chỉ bind loopback. Không mở PostgreSQL ra Internet.
- **FAULT:** cố ý tạo lỗi, block, duplicate hoặc crash trong project `be-db`. Dừng workload khác trong project trước khi chạy.
- **RESET:** thay đổi/xóa dữ liệu được gọi tên trong lab. Đọc phạm vi ngay trước cell.
- Mỗi cell Bash tự `cd` về thư mục cố định. Không dựa vào biến hoặc hàm của cell trước; chỉ dựa vào **file và dữ liệu đã tạo ở bài tiên quyết**.
- Tất cả cell đều có `excludeFromRunAll`. Chạy từng bài theo thứ tự. Block SQL minh họa có `ignore=true`; SQL thực thi nằm trong heredoc của Bash.
- `./db.sh` là client vào database học tập, có `ON_ERROR_STOP`. Một lỗi mong đợi phải được bắt đúng SQLSTATE; không giấu lỗi rồi in “thành công”.

Dữ liệu mặc định là **100.000 đơn hàng**, phù hợp hơn với VPS học tập khoảng 2 GiB RAM. Đây là kích thước lựa chọn để thí nghiệm, không phải ngưỡng mà database mới bắt đầu cần index. Các bài lớn có thể tăng dữ liệu sau khi đo dung lượng, WAL, thời gian backup và bộ nhớ. Không chạy PostgreSQL, MySQL và nhiều workload nặng cùng lúc trên máy ít RAM.

## Bản đồ kiến thức và bài thực hành

| Chặng | Bài | Bạn phải chứng minh được |
|---|---|---|
| Nền tảng | 00–03 | Môi trường riêng biệt; tiền có đơn vị; constraint chặn dữ liệu sai |
| Bộ máy lưu trữ | 04–06 | Snapshot giữ lịch sử; vacuum khác xóa file; WAL khác backup |
| Truy vấn | 07–15 | Index phục vụ một họ truy vấn; kế hoạch có bằng chứng; phân trang xử lý khóa trùng |
| Cạnh tranh | 16–22 | Tái lập lost update, write skew, blocking, deadlock; chọn biện pháp theo invariant |
| Luồng nghiệp vụ | 23–24 | Retry không tạo giao dịch thứ hai; publish/consume có cửa sổ lỗi được xử lý |
| Thay đổi và vận hành | 25–30 | Partition có giới hạn; pool có queue; migration tương thích; backup restore được |
| Triển khai và đối chiếu | 31–33 | VPS không lộ DB; biết khác biệt InnoDB/PlanetScale; có playbook sự cố |

---

# Phần I — Xác định dữ liệu đúng trước khi tối ưu

## 1. Database phục vụ điều gì trong backend?

Một API “chuyển 100 EUR từ A sang B” không chỉ là hai lệnh `UPDATE`. Nó có các bất biến: chỉ người có quyền mới được yêu cầu, tiền cùng loại, số tiền hợp lệ, A không vượt hạn mức, B nhận đúng số A bị trừ, yêu cầu retry không thực hiện hai lần, và lịch sử đủ để giải thích trạng thái. Có bất biến đặt ở gateway hoặc service, có bất biến nên đặt ở database, có bất biến cần phối hợp cả hai. Việc gọi ORM đúng cú pháp không chứng minh được các điều đó.

Hãy phân biệt ba lớp đúng. **Đúng một dòng** gồm số lượng không âm, trạng thái thuộc tập cho phép, ngày kết thúc sau ngày bắt đầu. **Đúng một tập dòng** gồm không đặt hai chỗ cùng ghế, không có ledger lệch tổng, ít nhất một người trực. **Đúng giữa các hệ thống** gồm ngân hàng ngoài đã nhận lệnh hay chưa, email đã gửi hay chưa, broker đã nhận event hay chưa. Tăng isolation ở database có thể giúp lớp thứ hai; nó không tự biến các hệ thống bên ngoài thành một transaction duy nhất.

Trong một thay đổi backend thực tế, hãy bắt đầu bằng bảng tình huống. Với yêu cầu rút tiền, hỏi chuyện gì xảy ra nếu số dư vừa đủ, amount bằng không, amount âm, loại tiền không khớp, tài khoản bị khóa, request lặp, hai request cùng lúc, kết nối mất sau commit, hoặc provider trả kết quả muộn. Đây là đầu vào cho schema và test. Nếu chỉ kiểm tra happy path, bạn có thể tối ưu rất tốt một quy trình sai.

## 2. Mô hình quan hệ, khóa và sự phụ thuộc

Một bảng đại diện cho một loại sự kiện hoặc thực thể có danh tính rõ ràng. `customer`, `account`, `payment_request`, `ledger_entry` và `provider_attempt` không phải năm cách đặt tên cho cùng một object JSON. Chúng có vòng đời, quyền sửa và tần suất truy cập khác nhau. Gộp tất cả vào một bảng “transaction” với nhiều cột nullable khiến trạng thái hợp lệ khó nhìn thấy và migration khó kiểm soát.

Khóa chính trả lời “dòng nào”. Khóa nghiệp vụ trả lời “hai dòng nào không được phép cùng tồn tại”. Một payment có `id` nội bộ và có thể có unique `(tenant_id, provider, provider_reference)`. Unique chỉ trên reference sẽ sai nếu hai provider dùng cùng định dạng. Unique chỉ trên email cũng cần cân nhắc tenant, quy tắc so sánh hoa/thường và việc một người có được nhiều tài khoản hay không. Đừng chọn khóa dựa vào việc ORM tạo migration dễ nhất.

Chuẩn hóa giảm số nơi cần cập nhật cho cùng một sự thật. Nếu tên quốc gia được lặp trong một triệu dòng customer, sửa tên có thể tạo nhiều trạng thái không nhất quán. Nhưng một invoice đã phát hành có thể cần **snapshot địa chỉ khi phát hành**, độc lập với địa chỉ hiện tại. Việc lưu lặp có chủ đích này phục vụ một sự thật lịch sử khác. Câu hỏi đúng là “dữ liệu này đang mô tả thời điểm nào, ai sở hữu, có được sửa không”, không chỉ “có bị trùng không”.

Không dùng `CHECK` gọi hàm truy vấn bảng khác để giả vờ tạo constraint tổng quát. Tính đúng dưới concurrent write, restore và thay đổi dữ liệu liên quan cần cơ chế phù hợp: unique/exclusion, foreign key, khóa một dòng đại diện, serializable hoặc một giao thức cập nhật được kiểm soát. Một hàm kiểm tra đọc thấy đúng tại thời điểm chạy không có nghĩa là invariant còn đúng khi hai giao dịch cùng commit.

Theo [tài liệu constraint PostgreSQL 17](https://www.postgresql.org/docs/17/ddl-constraints.html), `CHECK` chấp nhận kết quả true hoặc null; vì vậy `CHECK(amount > 0)` không thay thế `NOT NULL`. Foreign key giữ quan hệ tồn tại, nhưng PostgreSQL không tự tạo index phía tham chiếu. Đây là hai chi tiết cần kiểm tra trực tiếp trong thiết kế.

## 3. Tiền, đơn vị và số chính xác

Có hai mô hình thường hữu ích. Một là lưu số nguyên theo đơn vị nhỏ nhất: `12345` nghĩa là 123,45 EUR nếu quy tắc tiền tệ đó có hai chữ số thập phân. Hai là lưu `NUMERIC(p,s)` hoặc `DECIMAL(p,s)` khi cần độ chính xác thập phân với scale được quy định. Cả hai cần currency và quy tắc làm tròn. Không thể cộng 100 EUR với 100 VND chỉ vì cả hai cột đều là `BIGINT`.

Trong mô hình số nguyên, phép chia phí hoặc phân bổ cần quyết định xử lý phần dư. Ví dụ chia 10 cent cho ba bên không thể cho mỗi bên 3,333 cent trong ledger cent. Bạn cần thuật toán phân phối phần dư có thứ tự ổn định, tổng cuối bằng 10, và audit lý do. Trong mô hình decimal, việc ghi vào `NUMERIC(18,2)` có thể làm tròn đầu vào nhiều chữ số; đừng dùng sự làm tròn ngầm của cột như chính sách sản phẩm. Validate scale trước, tính với kiểu decimal, rồi áp dụng chính sách tại ranh giới được đặt tên.

[Numeric types của PostgreSQL](https://www.postgresql.org/docs/17/datatype-numeric.html) phân biệt số chính xác với floating point. Trong Node.js, còn phải bảo vệ ranh giới chuyển đổi: giá trị `BIGINT` có thể vượt phạm vi số nguyên an toàn của `Number`. Driver thường trả số lớn và numeric dưới dạng chuỗi. Chuyển bằng `Number(value)` cho tiện có thể phá độ chính xác trước khi query chạy. Dùng chuỗi thập phân hoặc `BigInt` theo loại phép toán; JSON API cần quy ước chuỗi khi không thể biểu diễn chính xác bằng number.

Thêm một chiều thường bị bỏ quên: tỷ giá là dữ liệu có nguồn, thời điểm hiệu lực và chiều quy đổi. Không chỉ lưu “rate = 25.000” rồi để người sau đoán đó là VND/EUR hay EUR/VND. Khoản gốc, khoản đích, phí và tỷ giá có thể có các scale khác nhau. Khi reconcile, bạn cần tái dựng được quyết định cũ, kể cả khi bảng tỷ giá hiện tại đã thay đổi.

## 4. ACID và ranh giới transaction

Atomicity có nghĩa các thay đổi nằm trong transaction hoặc cùng tồn tại hoặc cùng bị hủy. Consistency không tự biết quy định nghiệp vụ; database chỉ thực thi những ràng buộc và giao thức bạn thực sự cài. Isolation điều khiển những tương tác concurrent nào được phép. Durability còn gắn với cấu hình WAL, storage và phạm vi lỗi bạn xét. Một VPS còn đĩa sau process crash khác hoàn toàn một VPS bị mất cả ổ.

Một transaction phải có **một connection** trong suốt `BEGIN … COMMIT`. Dùng `pool.query('BEGIN')` rồi `pool.query('UPDATE …')` không đảm bảo hai câu chạy trên cùng backend connection. [node-postgres ghi rõ yêu cầu dùng cùng client](https://node-postgres.com/features/transactions). Trong ORM, truyền transaction client xuống đúng các repository tham gia; một repository dùng nhầm client toàn cục có thể commit độc lập dù service phía ngoài rollback.

Đừng giữ transaction mở trong khi chờ người dùng, gọi HTTP chậm hoặc đợi broker. Khóa được giữ lâu hơn, pool mất một connection lâu hơn, snapshot giữ lịch sử lâu hơn, và xác suất timeout tăng. Một thiết kế tốt tách quyết định bền vững trong database khỏi việc giao tiếp bên ngoài, kèm trạng thái trung gian và khả năng retry/reconcile.

---

# Bài 00 — LOCAL: dựng môi trường có tên và phạm vi rõ ràng

**Mục tiêu.** Mọi file thuộc thư mục học tập, mọi container thuộc project `be-db`, mọi volume có nhãn project đó. Khởi động lại container không xóa dữ liệu; reset dữ liệu là thao tác riêng.

**Tiên quyết.** Bash, Docker Engine hoặc Docker Desktop đang chạy, Compose v2 có `up --wait`, Internet để pull image và cài hai driver npm. Máy học tập còn dung lượng trống. Không cần cài PostgreSQL hay Node lên host. Nếu bạn dùng Windows, chạy trong WSL2 hoặc terminal Bash có Docker integration.

**Chuẩn bị và hành động.** Cell đầu tạo cấu hình. Nó không xóa thư mục hiện có; nếu chạy lại, giữ password cũ để không làm password trong volume lệch với môi trường.

```bash {"name":"db-00-bootstrap","excludeFromRunAll":"true"}
set -Eeuo pipefail
mkdir -p "${HOME}/backend-labs/database"
cd "${HOME}/backend-labs/database"
umask 077
chmod 700 .
mkdir -p scripts evidence backups state pgbouncer
docker version
docker compose version
if [ ! -f .env ]; then
  db_lab_password="$(od -An -N24 -tx1 /dev/urandom | tr -d ' \n')"
  printf 'DB_LAB_PASSWORD=%s\nLAB_UID=%s\nLAB_GID=%s\n' \
    "$db_lab_password" "$(id -u)" "$(id -g)" > .env
fi
cat > compose.yaml <<'YAML'
name: be-db
services:
  postgres:
    image: postgres:17-bookworm
    environment:
      POSTGRES_USER: lab
      POSTGRES_PASSWORD: ${DB_LAB_PASSWORD:?Missing .env}
      POSTGRES_DB: banklab
      TZ: UTC
    ports:
      - "127.0.0.1:15432:5432"
    volumes:
      - pgdata:/var/lib/postgresql/data
    command:
      - postgres
      - -c
      - shared_preload_libraries=pg_stat_statements
      - -c
      - track_io_timing=on
      - -c
      - max_connections=40
      - -c
      - shared_buffers=128MB
      - -c
      - work_mem=4MB
      - -c
      - log_lock_waits=on
      - -c
      - log_min_duration_statement=500
      - -c
      - log_line_prefix=%m [%p] %a %u@%d
    shm_size: 128m
    mem_limit: 768m
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U lab -d banklab"]
      interval: 2s
      timeout: 3s
      retries: 30
    restart: unless-stopped
  toolbox:
    image: node:24-bookworm-slim
    profiles: ["tools"]
    working_dir: /lab
    user: "${LAB_UID}:${LAB_GID}"
    entrypoint: ["node"]
    environment:
      PGHOST: postgres
      PGPORT: "5432"
      PGUSER: lab
      PGDATABASE: banklab
      PGPASSWORD: ${DB_LAB_PASSWORD}
      MYSQL_HOST: mysql
      MYSQL_PASSWORD: ${DB_LAB_PASSWORD}
      npm_config_cache: /tmp/npm-cache
    volumes:
      - .:/lab
    mem_limit: 256m
  mysql:
    image: mysql:8.4
    profiles: ["mysql"]
    environment:
      MYSQL_ROOT_PASSWORD: ${DB_LAB_PASSWORD}
      MYSQL_DATABASE: mysqllab
      MYSQL_USER: lab
      MYSQL_PASSWORD: ${DB_LAB_PASSWORD}
    ports:
      - "127.0.0.1:13306:3306"
    volumes:
      - mysqldata:/var/lib/mysql
    command: ["--innodb-buffer-pool-size=128M", "--max-connections=30"]
    mem_limit: 640m
    healthcheck:
      test: ["CMD-SHELL", "MYSQL_PWD=\"$${MYSQL_ROOT_PASSWORD}\" mysql -h127.0.0.1 -uroot -Nse 'SELECT 1'"]
      interval: 3s
      timeout: 5s
      retries: 40
volumes:
  pgdata:
  mysqldata:
YAML
cat > db.sh <<'BASH'
#!/usr/bin/env bash
set -Eeuo pipefail
cd "${HOME}/backend-labs/database"
exec docker compose exec -T postgres \
  psql -X -U lab -d banklab -v ON_ERROR_STOP=1 -P pager=off "$@"
BASH
chmod 700 db.sh
cat > package.json <<'JSON'
{
  "name": "backend-database-labs",
  "private": true,
  "type": "module",
  "dependencies": {
    "pg": "8.16.3",
    "mysql2": "3.11.5"
  }
}
JSON
docker compose config --quiet
docker compose up -d --wait --wait-timeout 120 postgres
docker compose run --rm --entrypoint npm toolbox \
  install --ignore-scripts --no-audit --no-fund
./db.sh -c "SELECT version(), current_database(), current_user;"
docker compose images
```

**Kết quả cần thấy.** Database `banklab`, user `lab`, server major 17, cổng publish chỉ `127.0.0.1:15432`. `package-lock.json` được tạo; giữ nó cùng kết quả học để lần sau dùng `npm ci`. Image tag baseline đã đối chiếu với [Docker Official Image PostgreSQL](https://hub.docker.com/_/postgres), nhưng bản vá mà bạn pull mới là phiên bản của kết quả đo.

**Debug.** Lỗi port occupied: tìm dịch vụ đang chiếm `15432`; không đổi sang bind `0.0.0.0`. Lỗi password sau khi sửa `.env` thường do volume đã được init bằng password cũ: khôi phục password cũ hoặc thực hiện quy trình đổi password trong DB; không xóa volume để “sửa nhanh” nếu muốn giữ bài. Nếu OOMKilled, dừng MySQL và giảm kích thước data trước khi chỉnh nhiều tham số. Dùng `docker compose logs --tail=80 postgres`, không in cấu hình đã nội suy secret ra nơi công cộng.

**Phục hồi.** `docker compose up -d --wait postgres` khởi động lại cùng volume. `docker compose stop postgres` dừng mà giữ dữ liệu. Cell RESET cuối sách mới xóa volume. Giới hạn memory trong lab là lựa chọn cho bài học, không phải cấu hình production mặc định.

---

# Bài 01 — LOCAL: schema có đơn vị và constraint kiểm tra được

**Câu hỏi.** Sai dữ liệu ở tầng application có bị chặn ở database không? Ta tạo các bảng mẫu trong schema `lab`, cùng hàm assertion làm bài thất bại nếu một tính chất không đúng.

**Tiên quyết.** Bài 00. Cell sau reset các bảng lab nền và lab.ledger_demo nếu bạn học lại từ đầu; chưa dùng cho dữ liệu thật.

```bash {"name":"db-01-schema","excludeFromRunAll":"true"}
set -Eeuo pipefail
cd "${HOME}/backend-labs/database"
./db.sh <<'SQL'
CREATE EXTENSION IF NOT EXISTS pg_stat_statements;
CREATE SCHEMA IF NOT EXISTS lab;
CREATE OR REPLACE FUNCTION lab.assert(ok boolean, message text)
RETURNS void LANGUAGE plpgsql AS $$
BEGIN
  IF ok IS DISTINCT FROM true THEN
    RAISE EXCEPTION 'ASSERTION FAILED: %', message;
  END IF;
END $$;

DROP TABLE IF EXISTS lab.orders;
DROP TABLE IF EXISTS lab.customers;
CREATE TABLE lab.customers (
  id integer PRIMARY KEY,
  tenant_id integer NOT NULL,
  email text NOT NULL,
  display_name text NOT NULL,
  UNIQUE (tenant_id, email)
);
CREATE TABLE lab.orders (
  id bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  customer_id integer NOT NULL REFERENCES lab.customers(id),
  status text NOT NULL CHECK (status IN ('PENDING','COMPLETED','CANCELLED')),
  amount_minor bigint NOT NULL CHECK (amount_minor >= 0),
  currency text NOT NULL CHECK (currency IN ('EUR','VND')),
  created_at timestamptz NOT NULL,
  metadata jsonb NOT NULL DEFAULT '{}'::jsonb
);
CREATE TABLE IF NOT EXISTS lab.wallets (
  id integer PRIMARY KEY,
  balance_minor bigint NOT NULL CHECK (balance_minor >= 0),
  version bigint NOT NULL DEFAULT 1 CHECK (version > 0)
);
DROP TABLE IF EXISTS lab.ledger_demo;
TRUNCATE lab.wallets;
INSERT INTO lab.wallets VALUES (1,100000,1),(2,100000,1);
CREATE TABLE IF NOT EXISTS lab.debit_audit (
  request_id text PRIMARY KEY,
  amount_minor bigint NOT NULL CHECK (amount_minor > 0)
);
TRUNCATE lab.debit_audit;

DO $$
BEGIN
  BEGIN
    INSERT INTO lab.wallets VALUES (9,-1,1);
    RAISE EXCEPTION 'negative amount unexpectedly accepted';
  EXCEPTION WHEN check_violation THEN
    RAISE NOTICE 'PASS: negative balance rejected';
  END;
  BEGIN
    INSERT INTO lab.wallets VALUES (9,NULL,1);
    RAISE EXCEPTION 'NULL unexpectedly accepted';
  EXCEPTION WHEN not_null_violation THEN
    RAISE NOTICE 'PASS: NULL rejected separately from CHECK';
  END;
END $$;
SELECT lab.assert((SELECT count(*)=2 FROM lab.wallets),'two fixture wallets');
SELECT 0.1::numeric + 0.2::numeric = 0.3::numeric AS exact_decimal;
SELECT 1::numeric / 3 AS fraction, round(1::numeric / 3,2) AS rounded;
SQL
```

**Giải thích kết quả.** Hai NOTICE là lỗi mong đợi được bắt đúng loại. Nếu bạn bỏ `NOT NULL` và chỉ giữ `CHECK(balance_minor >= 0)`, NULL có thể lọt qua do logic ba giá trị. Nếu đổi `amount_minor` thành số thực, dữ liệu có thể biểu diễn gần đúng; loại kiểu dữ liệu là một phần hợp đồng API chứ không chỉ lựa chọn tiết kiệm dung lượng.

**Debug và phục hồi.** Nếu thấy “relation does not exist”, kiểm tra schema prefix `lab.`. Không sửa `search_path` của database toàn cục chỉ để một query hết lỗi. Chạy lại cell để tạo fixture. Tại production, không chạy DROP như lab; dùng migration tương thích. Các role vận hành và role ứng dụng phải tách nhau: user `lab` ở đây có quyền quản trị để tiện học, không phải mẫu quyền cho ứng dụng ngân hàng.

**Bài mở rộng.** Thêm unique `(tenant_id, lower(email))` rồi quyết định chính sách Unicode, tenant và xóa mềm trước khi triển khai. Đổi account thành khóa gồm tenant và account id, thử cố tạo một ledger tham chiếu account của tenant khác. Mục tiêu là để database ngăn một lớp lỗi cách ly tenant, thay vì chỉ tin mọi đường code đều nhớ thêm WHERE.

---

# Bài 02 — LOCAL: dữ liệu có phân phối, không chỉ có số dòng

**Ý định.** Sinh dữ liệu ổn định để so sánh index. `random()` tiện nhưng có thể khiến mỗi lượt học khác nhau; bài này dùng công thức theo id. Status có phân bố 1% PENDING, 9% CANCELLED, 90% COMPLETED. Timestamp được cố ý trùng theo nhóm để phân trang phải có khóa phụ.

**Tiên quyết.** Bài 01. Biến `DB_LAB_ROWS` có mặc định ngay trong cell, không dựa vào cell trước. Với VPS 2 GiB hãy giữ 100.000; muốn một triệu thì sửa giá trị hoặc đặt biến khi chạy riêng cell. Reseed làm mất orders/customers mẫu hiện có.

```bash {"name":"db-02-seed","excludeFromRunAll":"true"}
set -Eeuo pipefail
cd "${HOME}/backend-labs/database"
db_lab_rows="${DB_LAB_ROWS:-100000}"
case "$db_lab_rows" in ''|*[!0-9]*) echo "Rows must be an integer" >&2; exit 2;; esac
if [ "$db_lab_rows" -lt 10000 ] || [ "$db_lab_rows" -gt 1000000 ]; then
  echo "This cell accepts 10,000..1,000,000 rows; default 100,000." >&2
  exit 2
fi
./db.sh -v n="$db_lab_rows" <<'SQL'
TRUNCATE lab.orders, lab.customers RESTART IDENTITY;
INSERT INTO lab.customers
SELECT i, 1 + i % 10, 'customer-' || i || '@example.invalid', 'Customer ' || i
FROM generate_series(1,5000) AS i;

INSERT INTO lab.orders(customer_id,status,amount_minor,currency,created_at,metadata)
SELECT
  1 + ((i * 37) % 5000),
  CASE WHEN i % 100=0 THEN 'PENDING'
       WHEN i % 10=0 THEN 'CANCELLED' ELSE 'COMPLETED' END,
  100 + ((i * 97) % 100000),
  CASE WHEN i%2=0 THEN 'EUR' ELSE 'VND' END,
  timestamptz '2026-01-01 00:00:00+00' + ((i/5) || ' seconds')::interval,
  jsonb_build_object('brand',CASE WHEN i%5=0 THEN 'A' ELSE 'B' END,
                     'priority',i%500=0, 'channel', i%4)
FROM generate_series(1,:n) AS i;
ANALYZE lab.customers;
ANALYZE lab.orders;
SELECT lab.assert((SELECT count(*)=:n FROM lab.orders),'requested rows loaded');
SELECT status,count(*) FROM lab.orders GROUP BY status ORDER BY status;
SELECT min(created_at),max(created_at),count(DISTINCT customer_id) FROM lab.orders;
SELECT pg_size_pretty(pg_relation_size('lab.orders')) AS heap,
       pg_size_pretty(pg_indexes_size('lab.orders')) AS indexes,
       pg_size_pretty(pg_total_relation_size('lab.orders')) AS total;
SQL
./db.sh -Atc "SELECT version();" > evidence/server-version.txt
docker image inspect postgres:17-bookworm --format '{{json .RepoDigests}}' \
  > evidence/postgres-image-digests.json
```

**Semantic assertion.** Count phải bằng n. Khi n chia hết cho 100, ba nhóm đúng tỷ lệ thiết kế. Thời gian nạp và kích thước thay đổi theo máy; không coi “5 giây” là điều kiện đạt. Dữ liệu này có tương quan nhân tạo giữa customer, status và metadata; đó là chủ đích có lợi cho bài thống kê, nhưng cũng là lý do không suy ra benchmark production từ nó.

**Debug.** Kiểm tra query có chọn thời gian nằm trong min/max vừa sinh hay không. Khi giảm n, khoảng thời gian chỉ kéo dài vài giờ; lấy một ngày ngoài khoảng sẽ tạo query “rất nhanh” vì không có dữ liệu. Nếu count khác, cell phải dừng; tìm lỗi trước khi tạo thêm index.

**Biên production.** Một bảng 100 dòng vẫn có thể cần unique index để giữ đúng hoặc phục vụ join. Một bảng 100 triệu dòng vẫn có truy vấn seq scan hợp lý để tổng hợp phần lớn bảng. Kích thước đơn thuần không xác định giải pháp; cần phân phối, selectivity, độ rộng dòng, tần suất gọi và khả năng giữ working set trong cache.

---

# Bài 03 — LOCAL: kiểm tra invariant kế toán bằng truy vấn đối soát

**Ý định.** Phân biệt “số dư không âm” với “hệ thống không tạo hoặc làm mất tiền”. Hai ví đều không âm vẫn có thể có tổng tiền sai nếu chỉ ghi một phía chuyển khoản.

**Thiết lập.** Dùng bảng ledger nhỏ riêng. Đây chưa phải một core banking ledger hoàn chỉnh: chưa có lịch mở kỳ, loại tài khoản, reversal, FX, phí hay quyền duyệt. Ta học cách viết invariant dưới dạng truy vấn mà một job đối soát có thể chạy độc lập.

```bash {"name":"db-03-ledger-invariant","excludeFromRunAll":"true"}
set -Eeuo pipefail
cd "${HOME}/backend-labs/database"
./db.sh <<'SQL'
DROP TABLE IF EXISTS lab.ledger_demo;
CREATE TABLE lab.ledger_demo (
  entry_id bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  transfer_id text NOT NULL,
  account_id integer NOT NULL REFERENCES lab.wallets(id),
  currency text NOT NULL,
  delta_minor bigint NOT NULL CHECK (delta_minor <> 0)
);
BEGIN;
INSERT INTO lab.ledger_demo(transfer_id,account_id,currency,delta_minor)
VALUES ('good-1',1,'EUR',-1200),('good-1',2,'EUR',1200);
COMMIT;
SELECT lab.assert(
  NOT EXISTS (
    SELECT 1 FROM lab.ledger_demo GROUP BY transfer_id,currency
    HAVING sum(delta_minor) <> 0 OR count(*) < 2
  ), 'every demonstration transfer balances per currency');

BEGIN;
INSERT INTO lab.ledger_demo(transfer_id,account_id,currency,delta_minor)
VALUES ('broken-1',1,'EUR',-700);
SELECT transfer_id,currency,sum(delta_minor) AS imbalance
FROM lab.ledger_demo GROUP BY transfer_id,currency HAVING sum(delta_minor)<>0;
ROLLBACK;
SELECT lab.assert(
  (SELECT count(*)=2 FROM lab.ledger_demo),'bad demonstration rolled back');
SQL
```

**Mong đợi.** Query ở giữa chỉ ra `broken-1` lệch -700. Sau rollback còn hai entries của good-1. Bạn đã viết được công cụ phát hiện sai, nhưng chưa buộc mọi writer phải tuân thủ; một user có quyền INSERT trực tiếp vẫn có thể commit entry đơn lẻ.

**Bài học thiết kế.** Cần quản lý đường ghi: role chỉ được gọi API/statement được kiểm soát, transaction bao phủ cả hai phía, invariant kiểm tra trước commit hoặc thiết kế constraint/trigger phù hợp, và job reconcile độc lập để phát hiện bug hoặc thao tác vận hành ngoài đường chuẩn. Đối với ledger bất biến, sửa sai thường tạo reversal cùng tham chiếu đến giao dịch gốc; sửa trực tiếp lịch sử phá khả năng giải thích.

**Debug/phục hồi.** Query tổng balance trên toàn bộ currency là sai nếu trộn EUR và VND. Hãy nhóm currency trước. Chạy lại cell chỉ reset `lab.ledger_demo`. Trong production, đối soát cần snapshot hoặc watermark có định nghĩa; không so hai query chạy ở hai thời điểm khác nhau rồi kết luận có mất tiền.

---

# Phần II — Bộ máy lưu trữ và lịch sử của dữ liệu

## Từ dòng logic đến page, index, WAL và snapshot

Trong PostgreSQL, bảng thông thường lưu các tuple trong heap; index chứa đường đi tìm tuple. Page thường có kích thước 8 KiB, nhưng giá trị compile-time phải kiểm tra bằng `SHOW block_size`. `ctid` chỉ là địa chỉ vật lý của một phiên bản, có thể thay đổi; không dùng làm khóa public hoặc cursor bền vững. `xmin` và `xmax` phục vụ MVCC, không phải cột audit nghiệp vụ. Đặc biệt `xmax` không đơn giản là cờ “dòng chết”: khóa dòng, transaction abort và MultiXact làm diễn giải phức tạp hơn. Xem [page layout](https://www.postgresql.org/docs/17/storage-page-layout.html).

Một UPDATE có thể tạo phiên bản tuple mới. Phiên bản cũ còn cần thiết nếu một snapshot hợp lệ vẫn có thể nhìn thấy nó. Vì thế “UPDATE đã commit” không đồng nghĩa “phiên bản cũ có thể xóa ngay”. Bạn cũng không thể dự đoán chính xác `ctid` sẽ thành `(0,5)`: không gian trống, HOT, vacuum và lịch sử bảng ảnh hưởng nơi đặt tuple. Hãy dùng chúng để quan sát cơ chế, không biến giá trị vật lý thành assertion nghiệp vụ.

Đọc snapshot thông thường thường không chờ row lock của writer; nhưng không phải mọi read đều không chặn mọi write. SELECT còn lấy table lock, DDL có thể xung đột, và locking read chủ động đợi row lock. Câu “MVCC khiến đọc không bao giờ chặn ghi” che mất những sự cố migration thường gặp. Học bảng lock và chuỗi blocker giúp giải thích chính xác hơn.

WAL phục vụ redo và durability của thay đổi được log. Heap page không cần được flush đầy đủ sau từng commit nếu WAL cần thiết đã bền vững theo cấu hình. Checkpoint giới hạn lượng công việc cần redo sau crash; nó không tạo một backup ở máy khác. Hệ điều hành, đĩa và cấu hình durability là những phần của giả định. Giải thích nền tảng này có trong [WAL introduction](https://www.postgresql.org/docs/17/wal-intro.html).

## Bộ điều phối hai session có rào chắn quan sát được

Các bài concurrency cần ép được thứ tự: A đọc xong, B đọc xong, A ghi, B ghi. `sleep 0.2` không chứng minh B đã đọc trước khi A commit; trên máy chậm nó có thể cho một lịch chạy khác. Bộ điều phối dùng `await` làm rào chắn cho lệnh đã hoàn thành, và `pg_blocking_pids` để xác nhận một connection đang thực sự bị connection khác chặn. Khoảng đợi 25 ms chỉ giảm tần suất polling; điều kiện database mới quyết định bước tiếp theo.

Mỗi bài dùng A, B để thực hiện transaction và O để quan sát. Observer không tham gia vào invariant nghiệp vụ. Timeout là dây an toàn cho thí nghiệm hỏng, không phải dụng cụ để đoán thứ tự. Nếu timeout xảy ra, bài phải thất bại và rollback, không tự biến thành kết quả đạt.

```bash {"name":"db-04-create-session-harness","excludeFromRunAll":"true"}
set -Eeuo pipefail
cd "${HOME}/backend-labs/database"
cat > scripts/harness.mjs <<'JS'
import pg from 'pg';
import assert from 'node:assert/strict';
import { setTimeout as delay } from 'node:timers/promises';
export { assert };
export const capture = promise => promise.then(
  value => ({ok:true,value}),
  error => ({ok:false,error})
);
export async function waitBlocked(observer, waiter, holder) {
  const deadline = Date.now() + 5000;
  while (Date.now() < deadline) {
    const result = await observer.query(
      'SELECT $2::int = ANY(pg_blocking_pids($1::int)) AS blocked',
      [waiter,holder]
    );
    if (result.rows[0].blocked) {
      console.log('Observed blocking edge:', waiter, 'waits for', holder);
      return;
    }
    await delay(25);
  }
  throw new Error('No expected blocking edge within 5 seconds');
}
export async function run(work) {
  const clients = ['A','B','O'].map(label => new pg.Client({
    application_name:'db-lab-' + label,
    connectionTimeoutMillis:5000
  }));
  try {
    for (const client of clients) {
      await client.connect();
      await client.query("SET search_path=lab,public");
      await client.query("SET statement_timeout='15s'");
      await client.query("SET lock_timeout='10s'");
      await client.query("SET idle_in_transaction_session_timeout='30s'");
    }
    const pids = [];
    for (const client of clients) {
      pids.push((await client.query('SELECT pg_backend_pid() AS pid')).rows[0].pid);
    }
    console.log('Session PIDs A,B,O:', pids);
    await work(...clients,pids);
  } finally {
    await Promise.allSettled(clients.map(client => client.query('ROLLBACK')));
    await Promise.allSettled(clients.map(client => client.end()));
  }
}
JS
docker compose run --rm toolbox --check scripts/harness.mjs
```

---

# Bài 04 — LOCAL: snapshot Read Committed và Repeatable Read

**Câu hỏi.** Hai SELECT trong cùng transaction có thấy cùng giá trị không? Câu trả lời phụ thuộc isolation và thời điểm lấy snapshot, không chỉ có chữ BEGIN.

**Tiên quyết.** Bài 01, harness. Bài này chỉ reset `lab.wallets`. A luôn đọc trước khi B update và commit, nên không cần đua với scheduler.

```bash {"name":"db-04-snapshot","excludeFromRunAll":"true"}
set -Eeuo pipefail
cd "${HOME}/backend-labs/database"
cat > scripts/snapshot.mjs <<'JS'
import {run,assert} from './harness.mjs';
await run(async (a,b) => {
  for (const isolation of ['READ COMMITTED','REPEATABLE READ']) {
    await b.query('UPDATE wallets SET balance_minor=100000 WHERE id=1');
    await a.query('BEGIN ISOLATION LEVEL ' + isolation);
    const before = await a.query(
      'SELECT balance_minor::text AS balance,ctid::text AS physical FROM wallets WHERE id=1'
    );
    await b.query('UPDATE wallets SET balance_minor=90000 WHERE id=1');
    const after = await a.query(
      'SELECT balance_minor::text AS balance,ctid::text AS physical FROM wallets WHERE id=1'
    );
    assert.equal(before.rows[0].balance,'100000');
    assert.equal(after.rows[0].balance,
      isolation==='READ COMMITTED' ? '90000' : '100000');
    console.log(isolation,{before:before.rows[0],after:after.rows[0]});
    await a.query('COMMIT');
  }
});
JS
docker compose run --rm toolbox scripts/snapshot.mjs
```

**Mong đợi.** RC đọc 100000 rồi 90000; RR đọc 100000 hai lần. Sau transaction, connection khác thấy 90000. Không assertion ctid vì đó là chi tiết vật lý. [PostgreSQL 17 transaction isolation](https://www.postgresql.org/docs/17/transaction-iso.html) giải thích snapshot theo statement ở RC và snapshot giữ trong transaction ở RR; bài này là mô hình nhỏ do ta xây để kiểm tra một hệ quả.

**Debug.** Nếu RR thấy mới, kiểm tra có vô tình dùng hai connection cho A hay không. Nếu có lỗi transaction idle, thí nghiệm đã bị dừng quá lâu. Chạy lại toàn bộ cell, không chạy tiếp từ câu SQL giữa transaction lỗi.

**Biên production.** Snapshot nhất quán hữu ích cho báo cáo, nhưng giữ transaction trong nhiều phút/hàng giờ làm tăng chi phí giữ phiên bản. Một dashboard thường không cần một snapshot cho toàn bộ ngày làm việc. Ngược lại, báo cáo tài chính nhiều query có thể cần snapshot hoặc một snapshot dữ liệu đã materialize với thời điểm chốt rõ ràng. Chọn theo yêu cầu dữ liệu, không theo mong muốn “mức cao nhất chắc tốt nhất”.

---

# Bài 05 — LOCAL: vì sao VACUUM chưa dọn được lịch sử

**Ý định.** Cho một transaction RR giữ snapshot cũ, update từ connection khác, vacuum khi snapshot còn mở, rồi kết thúc snapshot và vacuum lần nữa. Ta kiểm chứng **phiên bản cũ vẫn đọc được khi cần**, không yêu cầu `n_dead_tup` đổi thành một con số chính xác ngay lập tức.

```bash {"name":"db-05-vacuum-snapshot","excludeFromRunAll":"true"}
set -Eeuo pipefail
cd "${HOME}/backend-labs/database"
cat > scripts/vacuum.mjs <<'JS'
import {run,assert} from './harness.mjs';
await run(async (a,b,o) => {
  await o.query('DROP TABLE IF EXISTS lab.mvcc_demo');
  await o.query(
    'CREATE TABLE lab.mvcc_demo(id int PRIMARY KEY,value int NOT NULL,padding text) WITH (fillfactor=70)'
  );
  await o.query(
    "INSERT INTO lab.mvcc_demo SELECT i,1,repeat('x',100) FROM generate_series(1,5000) i"
  );
  await o.query('VACUUM (ANALYZE) lab.mvcc_demo');
  await a.query('BEGIN ISOLATION LEVEL REPEATABLE READ');
  assert.equal((await a.query('SELECT sum(value)::text AS s FROM lab.mvcc_demo')).rows[0].s,'5000');
  await b.query('UPDATE lab.mvcc_demo SET value=2');
  await o.query('VACUUM (VERBOSE,ANALYZE) lab.mvcc_demo');
  assert.equal((await a.query('SELECT sum(value)::text AS s FROM lab.mvcc_demo')).rows[0].s,'5000');
  assert.equal((await b.query('SELECT sum(value)::text AS s FROM lab.mvcc_demo')).rows[0].s,'10000');
  console.log('Old snapshot survived UPDATE and VACUUM as required');
  console.table((await o.query(
    "SELECT pid,state,backend_xmin,xact_start FROM pg_stat_activity WHERE application_name LIKE 'db-lab-%'"
  )).rows);
  await a.query('COMMIT');
  await o.query('VACUUM (VERBOSE,ANALYZE) lab.mvcc_demo');
  console.table((await o.query(
    "SELECT n_live_tup,n_dead_tup,n_tup_upd,n_tup_hot_upd,last_vacuum FROM pg_stat_user_tables WHERE schemaname='lab' AND relname='mvcc_demo'"
  )).rows);
});
JS
docker compose run --rm toolbox scripts/vacuum.mjs
```

**Kết quả.** A tiếp tục thấy tổng 5000, B thấy 10000. Snapshot cũ không bị vacuum phá hỏng. Sau khi A commit, vacuum có thêm khả năng thu hồi không gian dùng lại; độ giảm file và số thống kê không phải assertion của bài.

**Cơ chế.** Vacuum thường thu hồi không gian bên trong relation để dùng lại; nó có thể truncate các page trống cuối file khi điều kiện cho phép, nhưng không phải công cụ mặc định để ép mọi file nhỏ lại. `VACUUM FULL` rewrite và lấy lock mạnh, cần thời gian và dung lượng. Autovacuum còn phục vụ freeze để bảo vệ khỏi transaction ID wraparound, không chỉ “dọn rác”. Tóm tắt cơ chế trong [routine vacuuming](https://www.postgresql.org/docs/17/routine-vacuuming.html).

**HOT.** Một số update tránh thêm entry vào các index thông thường khi không đổi cột cần index và có chỗ thích hợp trên page. `fillfactor` thấp có thể dành chỗ, đổi lại bảng ban đầu lớn hơn. Điều kiện và ngoại lệ cho summarizing index có trong [HOT documentation](https://www.postgresql.org/docs/17/storage-hot.html). Không đặt `fillfactor=70` cho mọi bảng: append-only có nhu cầu khác bảng cập nhật liên tục.

**Debug và phục hồi.** Statistic là ước lượng và có độ trễ; đọc lại từ session mới, kiểm tra snapshot thống kê trước khi kết luận vacuum không chạy. Nếu bài treo, xem `backend_xmin` và transaction age. Chạy lại cell chỉ reset `mvcc_demo`. Production cần tìm transaction dài, replication slot, tải I/O và mức thay đổi trên từng bảng trước khi chỉnh autovacuum.

---

# Bài 06 — LOCAL: WAL, rollback và checkpoint là những thứ khác nhau

**Câu hỏi.** Rollback có làm mọi chi phí ghi biến mất không? Không: bạn có thể hủy kết quả logic nhưng vẫn đã tạo WAL, dùng CPU, giữ lock và tạo phiên bản cần xử lý.

```bash {"name":"db-06-wal-observation","excludeFromRunAll":"true"}
set -Eeuo pipefail
cd "${HOME}/backend-labs/database"
./db.sh <<'SQL'
SHOW block_size;
SHOW fsync;
SHOW synchronous_commit;
SHOW full_page_writes;
SELECT pg_current_wal_insert_lsn() AS before_lsn \gset
BEGIN;
UPDATE lab.wallets SET balance_minor=balance_minor+10;
ROLLBACK;
SELECT pg_wal_lsn_diff(pg_current_wal_insert_lsn(), :'before_lsn') AS wal_bytes_since_start;
SELECT wal_records,wal_fpi,wal_bytes,stats_reset FROM pg_stat_wal;
SELECT num_timed,num_requested,write_time,sync_time,stats_reset
FROM pg_stat_checkpointer;
SELECT id,balance_minor FROM lab.wallets ORDER BY id;
SQL
```

**Kỳ vọng.** Số dư không tăng bởi transaction đã rollback. WAL LSN thường tiến lên; không assert một byte cụ thể vì hoạt động hệ thống khác cùng ghi WAL. `pg_stat_checkpointer` là view của baseline PG17; không sao chép nguyên query sang mọi major cũ. [Cumulative statistics PostgreSQL 17](https://www.postgresql.org/docs/17/monitoring-stats.html) là nguồn xác định tên view và ý nghĩa cột theo phiên bản.

**Đọc sâu.** Một commit bền vững cục bộ chưa phải một commit đã được máy khác lưu. Nếu ứng dụng timeout sau khi gửi COMMIT, trạng thái có thể là đã commit nhưng response bị mất. Retry cần idempotency; rollback phía client không thể đảo ngược commit mà server đã thực hiện. Đây là lý do “request failed” và “payment failed” phải là hai khái niệm riêng.

**Checkpoint.** Làm checkpoint quá thường xuyên có thể tăng I/O và full page images; quá thưa có thể tăng công việc recovery. Không đặt checkpoint thủ công trong vòng lặp để “lưu dữ liệu cho chắc”. Theo dõi tốc độ sinh WAL, checkpoint được yêu cầu, latency storage và thời gian crash recovery rồi mới thay đổi. Giữ fsync/durability trong lab bình thường; tắt để tăng điểm benchmark sẽ thay đổi bài toán.

**Phục hồi.** Bài không thay cấu hình. Nếu cần đọc lại số dư chuẩn, dùng bài reset của concurrency sau này. Để học crash recovery thật trong phạm vi project, dùng Bài 30; đừng kill một database đang phục vụ công việc khác.

---

# Phần III — Tối ưu bằng giả thuyết và bằng chứng

## 1. Một query chậm có thể đang chờ thứ gì?

Tổng latency của một API gồm chờ pool, truyền mạng, planning, execution, chờ lock, đọc/ghi storage, serialize result và công việc application. Nếu query riêng trong psql mất 2 ms nhưng API mất 800 ms, thêm index chưa chắc giúp. Bạn cần trace một request xuyên các tầng: lúc xin connection, lúc có connection, lúc gửi query, lúc nhận đủ rows, lúc trả response. Một báo cáo chỉ có “SQL time trung bình” có thể che hàng đợi ở phía trước SQL.

Bắt đầu với một query và một nhóm tham số có ý nghĩa. Dữ liệu lệch có thể làm tenant lớn chậm trong khi tenant nhỏ nhanh. Query normalized giống nhau không đồng nghĩa chi phí tương đương. Hãy ghi n dòng, distribution, range truy vấn, số rows trả về, phiên bản server và cấu hình ảnh hưởng plan. Sau đó đặt giả thuyết cụ thể, chẳng hạn “query phải đọc nhiều heap page vì index thiếu cột equality”, hoặc “ước lượng số rows của hai cột tương quan quá thấp”.

Một thay đổi index tác động đến cả read và write. Mỗi INSERT/DELETE có thêm công việc bảo trì, UPDATE có thể mất cơ hội HOT, WAL nhiều hơn, backup dài hơn và working set lớn hơn. Index lớn ít dùng có thể vẫn là unique constraint quan trọng; index có `idx_scan=0` sau restart chưa chứng minh vô dụng. Quyết định xóa cần hiểu workload trong khoảng đại diện và các ràng buộc mà index đang thực thi.

## 2. Cách đọc kế hoạch từ dưới lên

Bạn có thể đọc một plan theo ba lượt. Lượt một tìm lượng dữ liệu: ở đâu rows tăng mạnh, ở đâu bị lọc bỏ, ở đâu có LIMIT khiến việc đọc dừng sớm. Lượt hai so estimated rows với actual rows để tìm vị trí ước lượng bắt đầu lệch. Lượt ba xem buffers, sort method, temporary I/O, batches, heap fetches và loops để biết công việc vật lý. Tổng thời gian không phải bằng cách cộng mọi node, vì thời gian node cha thường bao gồm công việc con.

Trong [hướng dẫn EXPLAIN PostgreSQL 17](https://www.postgresql.org/docs/17/using-explain.html), cost là đơn vị chi phí của planner, không phải millisecond; actual rows/time của node có nhiều loops cần đọc theo mỗi lần thực hiện. `EXPLAIN ANALYZE` thực sự chạy câu lệnh. `BUFFERS shared read` là block được nạp vào shared buffers, có thể được phục vụ từ OS cache; không tự chứng minh từng block đều là physical disk read.

Đừng lấy `Rows Removed by Filter` cao làm verdict tự động. Một báo cáo cần toàn bộ tháng và chỉ bỏ ít dòng có thể dùng seq scan tốt. Cũng đừng ca ngợi Index Only Scan bất kể hoàn cảnh: index lớn hơn có thể đổi lấy write amplification mà endpoint không cần. Kết quả cần đánh giá là workload với SLO, chi phí và tính đúng, không phải tên node đẹp.

**Về EXPLAIN trên câu ghi.** Bọc BEGIN/ROLLBACK giúp tránh commit thay đổi dữ liệu trong transaction, nhưng không xóa CPU/I/O, WAL, thời gian giữ lock, tác động sequence hoặc side effect ngoài DB do extension/hàm thực hiện. Với production, ưu tiên dữ liệu sao chép hoặc kế hoạch không ANALYZE nếu chưa đánh giá tải; không coi rollback như vùng thử vô hại.

## 3. Chọn index theo truy vấn và thứ tự

Với B-tree nhiều cột, thứ tự sắp xếp là thứ tự từ điển: cùng tenant mới xét customer, cùng customer mới xét time. Equality đầu index thường giúp thu hẹp vùng cần đọc; range đầu tiên thường quyết định phần tiếp theo phải đi qua. Nhưng một index “không đúng thứ tự lý tưởng” vẫn có thể hữu ích cho query khác hoặc giảm heap access. Viết “không dùng được” thường là diễn đạt quá mạnh.

[PG17 multicolumn indexes](https://www.postgresql.org/docs/17/indexes-multicolumn.html) cho phép planner dùng index với một tập con cột, kể cả thiếu cột đầu, nhưng có thể phải đi qua nhiều index. [PG18 multicolumn indexes](https://www.postgresql.org/docs/18/indexes-multicolumn.html) bổ sung skip scan: planner có thể lặp các tìm kiếm theo các giá trị tiền tố để khai thác điều kiện ở cột sau khi có lợi. Ít distinct ở tiền tố thường thuận lợi, không có ngưỡng cardinality tuyệt đối bảo đảm hoặc cấm nó.

**Baseline sách là PG17.** Không có cell nào gọi `uuidv7()` hoặc dựa vào skip scan. Nếu muốn so PG18, tạo **project và volume mới**, restore logical dump, chạy cùng truy vấn và so kế hoạch; không đổi image major trên volume PG17. UUIDv7 có thể cải thiện locality trong một số workload nhưng không biến khóa thành phép tuần tự hoàn hảo hay loại bỏ mọi page split. UUIDv4 có lợi ích khác; quyết định phải gồm threat model, sinh id phân tán, kích thước index và pattern insert.

---

# Bài 07 — LOCAL: giữ nguyên kết quả, giảm công việc đọc

**Mục tiêu.** Đo cùng query trước/sau index và lưu plan. Trước hết xác nhận query trả dữ liệu, tránh “tối ưu” một điều kiện sai.

**Tiên quyết.** Bài 02. Các bài query dùng bảng sao chép `lab.q_orders` để độc lập với index trên bảng orders gốc. Chạy lại một bài reset bảng bản sao, không ảnh hưởng ví/ledger.

```bash {"name":"db-07-plan-baseline","excludeFromRunAll":"true"}
set -Eeuo pipefail
cd "${HOME}/backend-labs/database"
./db.sh <<'SQL' | tee evidence/07-plans.txt
DROP TABLE IF EXISTS lab.q_orders;
CREATE TABLE lab.q_orders AS TABLE lab.orders;
ANALYZE lab.q_orders;
SELECT lab.assert(
  (SELECT count(*)>0 FROM lab.q_orders WHERE customer_id=1234),
  'chosen customer exists');
EXPLAIN (ANALYZE,BUFFERS,SETTINGS,TIMING OFF)
SELECT id,status,amount_minor,created_at FROM lab.q_orders WHERE customer_id=1234;
CREATE INDEX q_customer_idx ON lab.q_orders(customer_id);
ANALYZE lab.q_orders;
EXPLAIN (ANALYZE,BUFFERS,SETTINGS,TIMING OFF)
SELECT id,status,amount_minor,created_at FROM lab.q_orders WHERE customer_id=1234;
SELECT count(*),sum(amount_minor) FROM lab.q_orders WHERE customer_id=1234;
SQL
./db.sh -Atc "EXPLAIN (ANALYZE,BUFFERS,FORMAT JSON,TIMING OFF)
SELECT id,status,amount_minor,created_at FROM lab.q_orders WHERE customer_id=1234;" \
  > evidence/07-after.json
```

**Kỳ vọng về semantics.** Count và tổng amount không thay đổi. Trước index thường có seq scan; sau có thể index scan hoặc bitmap heap scan. Không assert loại scan vì planner có quyền chọn khác theo kích thước/cấu hình. Điều cần ghi là số dòng đọc/lọc, buffers và thời gian nhiều lượt cùng điều kiện.

**Giải thích.** Index phụ biết customer và địa chỉ tuple nhưng chưa chứa đủ cột trả về, nên còn cần heap. Bitmap scan gom vị trí để đọc heap theo page; nó không bảo toàn thứ tự index và có thể cần Sort. Index scan thường hợp query selective; seq scan thường hợp đọc phần lớn relation. Không đặt một ngưỡng phần trăm cố định cho mọi hệ thống.

**Debug/phục hồi.** Nếu hai plan giống nhau, kiểm tra số rows thực, pg_indexes và ANALYZE trước. Đừng lập tức `SET enable_seqscan=off` rồi coi plan ép buộc là tối ưu. Có thể dùng nó như một thí nghiệm chẩn đoán trong session riêng, nhưng cần quay lại phép đo tự nhiên. Bài tiếp theo tự tạo lại q_orders.

**Bài tập.** Chạy query select status COMPLETED rồi giải thích tại sao index customer không giải quyết bài đó. Thêm LIMIT nhưng không ORDER BY, hỏi vì sao tập kết quả không phải hợp đồng ổn định. Chuyển từ trả 20 dòng sang trả 20.000 dòng và đo cả thời gian client nhận/serialize để thấy database execution không phải toàn bộ latency.

---

# Bài 08 — LOCAL: composite index, range và điều kiện có thể tìm trong index

**Ý định.** So `(created_at,status)` với `(status,created_at)` cho cùng query. Từ đó hiểu vùng tìm kiếm, không ghi nhớ câu khẩu quyết như một định luật duy nhất.

```bash {"name":"db-08-composite-index","excludeFromRunAll":"true"}
set -Eeuo pipefail
cd "${HOME}/backend-labs/database"
./db.sh <<'SQL' | tee evidence/08-composite.txt
DROP TABLE IF EXISTS lab.q_orders;
CREATE TABLE lab.q_orders AS TABLE lab.orders;
ANALYZE lab.q_orders;
CREATE INDEX q_time_status_idx ON lab.q_orders(created_at,status);
EXPLAIN (ANALYZE,BUFFERS,TIMING OFF)
SELECT id,status,created_at FROM lab.q_orders
WHERE status='PENDING'
  AND created_at >= timestamptz '2026-01-01 00:01:00+00'
  AND created_at < timestamptz '2026-01-01 00:10:00+00';

DROP INDEX lab.q_time_status_idx;
CREATE INDEX q_status_time_idx ON lab.q_orders(status,created_at);
EXPLAIN (ANALYZE,BUFFERS,TIMING OFF)
SELECT id,status,created_at FROM lab.q_orders
WHERE status='PENDING'
  AND created_at >= timestamptz '2026-01-01 00:01:00+00'
  AND created_at < timestamptz '2026-01-01 00:10:00+00';

EXPLAIN (ANALYZE,BUFFERS,TIMING OFF)
SELECT id FROM lab.q_orders
WHERE created_at >= timestamptz '2026-01-01 00:01:00+00'
  AND created_at < timestamptz '2026-01-01 00:02:00+00';

CREATE INDEX q_time_idx ON lab.q_orders(created_at);
EXPLAIN (ANALYZE,BUFFERS,TIMING OFF)
SELECT id FROM lab.q_orders
WHERE date_trunc('minute',created_at)=timestamptz '2026-01-01 00:01:00+00';
EXPLAIN (ANALYZE,BUFFERS,TIMING OFF)
SELECT id FROM lab.q_orders
WHERE created_at >= timestamptz '2026-01-01 00:01:00+00'
  AND created_at < timestamptz '2026-01-01 00:02:00+00';
SQL
```

**Mong đợi.** Hai query theo phút phải cùng tập dòng. Với query PENDING, equality trước range thường tạo khoảng index hẹp hơn. Với query chỉ time, có thể planner dùng scan rộng, seq scan hoặc index khác; nó không “bị cấm” dùng q_status_time_idx. Trong PG18, skip scan có thể làm kết quả khác baseline, hãy gắn phiên bản vào bằng chứng.

**Điều kiện và kiểu dữ liệu.** Hàm đặt quanh cột có thể khiến index trên giá trị gốc không trực tiếp phục vụ tìm kiếm. Viết range nửa mở `>= start AND < end` thường rõ hơn so với ép ngày. Tuy nhiên phải xác định start/end theo múi giờ nghiệp vụ; một ngày tại địa phương có chuyển giờ mùa hè không luôn dài đúng 24 giờ. Expression index là lựa chọn khi query thực sự cần biểu thức ổn định, ví dụ lower(email), nhưng phải xét collation và quy tắc so sánh.

**Debug/phục hồi.** Nếu thấy không có rows, đối chiếu Bài 02 min/max. Nếu query biểu thức dùng index khác, xem Index Cond và Filter để biết phần nào làm lookup. Không so một lần cold với một lần warm rồi gán toàn bộ chênh lệch cho thứ tự cột. Chạy lại bài để reset q_orders; trong production, build/drop index phải theo quy trình migration.

**Biên production.** Một endpoint lọc tenant, status và order theo created_at cần index khác một endpoint tìm exact provider_reference. Viết ra top query shapes và tần suất trước khi tạo ba index tương tự nhau. Bất kỳ đề xuất index nào cũng nên kèm query đại diện, dữ liệu đại diện, tác động write và kế hoạch rollback.

---

# Bài 09 — LOCAL: covering index và visibility map

**Câu hỏi.** Có đủ cột trong index thì đã chắc chắn không cần đọc heap chưa? Bài này cho thấy tên Index Only Scan và Heap Fetches phải được đọc cùng nhau.

```bash {"name":"db-09-covering-visibility","excludeFromRunAll":"true"}
set -Eeuo pipefail
cd "${HOME}/backend-labs/database"
./db.sh <<'SQL' | tee evidence/09-covering.txt
DROP TABLE IF EXISTS lab.q_orders;
CREATE TABLE lab.q_orders AS TABLE lab.orders;
CREATE INDEX q_cover_idx ON lab.q_orders(customer_id,created_at DESC,id DESC)
INCLUDE(status,amount_minor);
VACUUM (ANALYZE) lab.q_orders;

EXPLAIN (ANALYZE,BUFFERS,TIMING OFF)
SELECT id,status,amount_minor,created_at FROM lab.q_orders
WHERE customer_id=1234 ORDER BY created_at DESC,id DESC LIMIT 20;

UPDATE lab.q_orders SET amount_minor=amount_minor+1 WHERE customer_id=1234;
EXPLAIN (ANALYZE,BUFFERS,TIMING OFF)
SELECT id,status,amount_minor,created_at FROM lab.q_orders
WHERE customer_id=1234 ORDER BY created_at DESC,id DESC LIMIT 20;

VACUUM (ANALYZE) lab.q_orders;
EXPLAIN (ANALYZE,BUFFERS,TIMING OFF)
SELECT id,status,amount_minor,created_at FROM lab.q_orders
WHERE customer_id=1234 ORDER BY created_at DESC,id DESC LIMIT 20;

SELECT pg_size_pretty(pg_relation_size('lab.q_cover_idx')) AS index_size;
SQL
```

**Mong đợi.** Truy vấn có thể dùng Index Only Scan vì các cột đều có trong index. Sau UPDATE, heap fetches có thể tăng do page chưa được xác nhận all-visible; sau VACUUM có thể giảm. Không yêu cầu luôn bằng 0: concurrent activity và snapshot cũng ảnh hưởng. Cơ chế này được mô tả trong [index-only scans](https://www.postgresql.org/docs/17/indexes-index-only-scans.html).

**Suy luận thiết kế.** Cột INCLUDE là dữ liệu mang theo để trả kết quả, không phải một key bổ sung dùng tùy ý để tìm kiếm hoặc sắp thứ tự. Đính cả JSON lớn vào index có thể làm index rất nặng, tăng chi phí ghi hoặc gặp giới hạn kích thước tuple index. Hãy cân nhắc endpoint có thực sự cần mọi cột. Nhiều lần bỏ cột dư trong SELECT có lợi hơn thêm cột rộng vào index.

**Debug.** Nếu plan không index-only, kiểm tra SELECT có một cột chưa include, hoặc planner chọn giải pháp khác vì query trả quá nhiều rows. Nếu heap fetches còn cao, kiểm tra update liên tục và vacuum, không gọi VACUUM FULL để xử lý một triệu chứng chưa chẩn đoán.

**Phục hồi/production.** Chạy lại cell tạo lại q_orders. Cột amount trong INCLUDE làm update amount cũng phải bảo trì index; đây là đánh đổi cụ thể cần đo ở workload có ghi nhiều. Khi review một covering index, yêu cầu cả evidence read và write, không chỉ ảnh chụp một query nhanh.

---

# Bài 10 — LOCAL: partial index và query có tham số

**Mục tiêu.** Index một tập nhỏ dữ liệu đang hoạt động, đồng thời thấy rằng planner phải chứng minh query thỏa predicate. “ORM có parameter” không tự làm partial index vô ích, nhưng generic plan có hạn chế cần quan sát.

```bash {"name":"db-10-partial-generic-plan","excludeFromRunAll":"true"}
set -Eeuo pipefail
cd "${HOME}/backend-labs/database"
./db.sh <<'SQL' | tee evidence/10-partial.txt
DROP TABLE IF EXISTS lab.q_orders;
CREATE TABLE lab.q_orders AS TABLE lab.orders;
CREATE INDEX q_all_time_idx ON lab.q_orders(created_at,id);
CREATE INDEX q_pending_time_idx ON lab.q_orders(created_at,id)
WHERE status='PENDING';
ANALYZE lab.q_orders;
SELECT indexrelname,pg_size_pretty(pg_relation_size(indexrelid)) AS size
FROM pg_stat_user_indexes WHERE schemaname='lab' AND relname='q_orders';

EXPLAIN (ANALYZE,BUFFERS,TIMING OFF)
SELECT id FROM lab.q_orders
WHERE status='PENDING' ORDER BY created_at,id LIMIT 20;

SET plan_cache_mode=force_generic_plan;
PREPARE pending_search(text) AS
SELECT id FROM lab.q_orders
WHERE status=$1 ORDER BY created_at,id LIMIT 20;
EXPLAIN (ANALYZE,BUFFERS,TIMING OFF) EXECUTE pending_search('PENDING');
DEALLOCATE pending_search;
RESET plan_cache_mode;

SET plan_cache_mode=force_custom_plan;
PREPARE pending_search(text) AS
SELECT id FROM lab.q_orders
WHERE status=$1 ORDER BY created_at,id LIMIT 20;
EXPLAIN (ANALYZE,BUFFERS,TIMING OFF) EXECUTE pending_search('PENDING');
DEALLOCATE pending_search;
RESET plan_cache_mode;
SQL
```

**Kết quả.** Partial index thường nhỏ hơn full index trong phân phối mẫu. Generic plan phải đúng cho mọi value của `$1`, nên không được giả định status=PENDING. Custom plan có giá trị cụ thể để cân nhắc. Xem [partial indexes](https://www.postgresql.org/docs/17/indexes-partial.html); đừng lấy một ví dụ forced generic rồi kết luận mọi prepared query luôn bỏ partial index.

**Chi phí ít được thấy.** Khi status chuyển PENDING sang COMPLETED, hệ thống vẫn phải bảo trì index và phiên bản tuple liên quan. Partial không có nghĩa “UPDATE trạng thái miễn phí”. Predicate nằm trên một cột nóng có thể làm lợi ích read và chi phí write thay đổi mạnh theo tỷ lệ luân chuyển, dù số rows PENDING tại một thời điểm nhỏ.

**Debug/phục hồi.** Nếu custom plan cũng không chọn partial, kiểm tra selectivity, size và ORDER BY; tên index không phải đích cuối. Các SET chỉ áp dụng connection psql trong cell. Chạy lại reset bảng bản sao. Production phải kiểm tra loại prepared statement mà driver và pool thực sự sử dụng, cùng các tham số phổ biến/hiếm.

**Bài tập.** Thiết kế unique email cho user chưa xóa mềm, rồi trả lời: khi restore một user có email đang được user khác sử dụng, API nên phản hồi gì? Partial unique có thể giữ constraint, nhưng sản phẩm vẫn cần chính sách xử lý xung đột. Database nên từ chối một trạng thái sai rõ ràng, còn dịch vụ chuyển lỗi đó thành thông điệp phù hợp.

---

# Bài 11 — LOCAL: JSONB, GIN và dữ liệu đáng ra nên là cột

**Ý định.** Tìm thuộc tính động bằng toán tử tương thích index. So sánh điều đó với một thuộc tính thường xuyên được lọc và có schema đủ ổn định để thành cột riêng.

```bash {"name":"db-11-json-index","excludeFromRunAll":"true"}
set -Eeuo pipefail
cd "${HOME}/backend-labs/database"
./db.sh <<'SQL' | tee evidence/11-json.txt
DROP TABLE IF EXISTS lab.q_orders;
CREATE TABLE lab.q_orders AS TABLE lab.orders;
ANALYZE lab.q_orders;
EXPLAIN (ANALYZE,BUFFERS,TIMING OFF)
SELECT id FROM lab.q_orders WHERE metadata @> '{"priority":true,"brand":"A"}'::jsonb;
CREATE INDEX q_json_gin_idx ON lab.q_orders USING gin(metadata jsonb_path_ops);
EXPLAIN (ANALYZE,BUFFERS,TIMING OFF)
SELECT id FROM lab.q_orders WHERE metadata @> '{"priority":true,"brand":"A"}'::jsonb;

CREATE INDEX q_brand_expression_idx ON lab.q_orders((metadata->>'brand'));
ANALYZE lab.q_orders;
EXPLAIN (ANALYZE,BUFFERS,TIMING OFF)
SELECT id FROM lab.q_orders WHERE metadata->>'brand'='A';

SELECT '{}'::jsonb->>'x' IS NULL AS missing_is_sql_null,
       '{"x":null}'::jsonb->>'x' IS NULL AS json_null_as_text_is_sql_null,
       '{}'::jsonb ? 'x' AS missing_key,
       '{"x":null}'::jsonb ? 'x' AS present_key;
SQL
```

**Mong đợi.** Hai truy vấn containment trả cùng tập id. Sau GIN, planner có thêm đường truy cập phù hợp; nó có thể chọn bitmap và recheck. Query brand biểu thức dùng một biểu thức khác, nên ta tạo index phù hợp riêng. Truy vấn cuối phân biệt thiếu key với key có giá trị JSON null.

**Cơ chế cần nhớ.** `jsonb_ops` và `jsonb_path_ops` không hỗ trợ cùng tập toán tử. Path ops phù hợp một số containment/jsonpath, nhưng không phải lựa chọn thay thế tổng quát cho toán tử existence `?`. [JSON types and indexing](https://www.postgresql.org/docs/17/datatype-json.html) là nguồn để kiểm tra operator class trước khi tạo index.

**Giới hạn mô hình.** Dùng JSON cho metadata biến thiên là hợp lý; nhét amount, currency, tenant_id và status cốt lõi vào JSON khiến kiểu dữ liệu, constraint và query khó quản lý hơn. Khi một field trở thành điều kiện lọc chính, cần xem lại schema. Không phải lúc nào một GIN toàn cột cũng là cách nhỏ nhất: expression index cho một thuộc tính có thể phù hợp hơn.

**Debug/phục hồi.** Nếu query không dùng GIN, kiểm tra operator thay vì chỉ kiểm tra tên cột. Xem Rows Removed by Index Recheck và buffers để hiểu false positives. Chạy lại reset q_orders. Production còn phải đo write amplification và GIN pending-list behavior với workload ghi; một benchmark đọc tĩnh không phản ánh điều đó.

---

# Bài 12 — LOCAL: BRIN phụ thuộc tương quan vật lý

**Câu hỏi.** Hai bảng có cùng dữ liệu và cùng BRIN có cùng hiệu năng không? Một bảng được lưu gần thứ tự time, bảng kia được trộn; truy vấn logic giống nhau nhưng phạm vi page cần kiểm tra khác nhau.

```bash {"name":"db-12-brin-correlation","excludeFromRunAll":"true"}
set -Eeuo pipefail
cd "${HOME}/backend-labs/database"
./db.sh <<'SQL' | tee evidence/12-brin.txt
DROP TABLE IF EXISTS lab.q_orders;
DROP TABLE IF EXISTS lab.q_shuffled;
CREATE TABLE lab.q_orders AS SELECT * FROM lab.orders ORDER BY created_at,id;
CREATE TABLE lab.q_shuffled AS SELECT * FROM lab.orders ORDER BY md5(id::text);
CREATE INDEX q_brin_idx ON lab.q_orders USING brin(created_at) WITH (pages_per_range=16);
CREATE INDEX q_shuffled_brin_idx ON lab.q_shuffled USING brin(created_at) WITH (pages_per_range=16);
ANALYZE lab.q_orders;
ANALYZE lab.q_shuffled;
SELECT tablename,attname,correlation FROM pg_stats
WHERE schemaname='lab' AND tablename IN ('q_orders','q_shuffled')
AND attname='created_at';
EXPLAIN (ANALYZE,BUFFERS,TIMING OFF)
SELECT count(*) FROM lab.q_orders
WHERE created_at>=timestamptz '2026-01-01 00:02:00+00'
AND created_at<timestamptz '2026-01-01 00:03:00+00';
EXPLAIN (ANALYZE,BUFFERS,TIMING OFF)
SELECT count(*) FROM lab.q_shuffled
WHERE created_at>=timestamptz '2026-01-01 00:02:00+00'
AND created_at<timestamptz '2026-01-01 00:03:00+00';
SELECT lab.assert(
  (SELECT count(*) FROM lab.q_orders)=(SELECT count(*) FROM lab.q_shuffled),
  'same logical row count');
SELECT pg_size_pretty(pg_relation_size('lab.q_brin_idx')) AS brin_size;
SQL
```

**Mong đợi.** Row count bằng nhau. Bảng xếp time thường có correlation lớn về trị tuyệt đối hơn. BRIN lưu tóm tắt block range nên có thể bỏ qua nhiều range ở bảng có tương quan, nhưng khó bỏ qua range chứa thời gian trộn. Planner chọn seq scan trên bảng trộn có thể là quyết định hợp lý. [BRIN documentation](https://www.postgresql.org/docs/17/brin.html) mô tả tính lossy và cơ chế summarization.

**Debug.** Với bảng quá nhỏ, cả hai query đều nhanh và khác biệt không rõ. Đọc công việc thay vì một con số millisecond. Nếu bảng trộn vẫn có tương quan bất ngờ, kiểm tra dữ liệu, plan và số page. Không thay cost để ép hình ảnh mong muốn rồi coi đó là kết luận.

**Biên production.** Log append-only có backfill lịch sử, clock skew hoặc batch import có thể mất tương quan theo thời gian. Chọn pages_per_range đổi mức chi tiết, kích thước index và lượng recheck. BRIN không thay thế unique index; nó không tự bảo đảm id duy nhất. Thực hành mở rộng: thêm một batch dữ liệu cũ vào cuối bảng rồi đo lại.

**Phục hồi.** Hai bản sao có thể giữ lại để quan sát; trước bài kế tiếp q_orders được tạo lại. Nếu muốn giảm dung lượng sau khi học, chỉ drop q_shuffled trong schema lab, không dùng công cụ prune toàn host.

---

# Bài 13 — LOCAL: thống kê và tương quan giữa cột

**Ý định.** Cho planner dữ liệu mà hai cột phụ thuộc nhau, quan sát ước lượng trước/sau extended statistics. Thêm index vào một ước lượng sai có thể không chữa đúng gốc vấn đề.

```bash {"name":"db-13-statistics","excludeFromRunAll":"true"}
set -Eeuo pipefail
cd "${HOME}/backend-labs/database"
./db.sh <<'SQL' | tee evidence/13-statistics.txt
DROP TABLE IF EXISTS lab.q_stats;
CREATE TABLE lab.q_stats(a integer NOT NULL,b integer NOT NULL,padding text);
INSERT INTO lab.q_stats
SELECT i%100,i%100,repeat('s',40) FROM generate_series(1,20000) i;
ANALYZE lab.q_stats;
EXPLAIN (ANALYZE,BUFFERS,TIMING OFF)
SELECT * FROM lab.q_stats WHERE a=7 AND b=7;
CREATE STATISTICS lab.q_stats_ab (dependencies,mcv,ndistinct)
ON a,b FROM lab.q_stats;
ANALYZE lab.q_stats;
EXPLAIN (ANALYZE,BUFFERS,TIMING OFF)
SELECT * FROM lab.q_stats WHERE a=7 AND b=7;
SELECT lab.assert(
  (SELECT count(*)=200 FROM lab.q_stats WHERE a=7 AND b=7),
  'known correlated result');
SELECT attname,n_distinct,most_common_vals FROM pg_stats
WHERE schemaname='lab' AND tablename='q_stats';
SQL
```

**Mong đợi.** Actual rows là 200. Trước khi có thống kê đa cột, planner có thể ước lượng thấp nếu coi điều kiện đủ độc lập; sau có thêm bằng chứng về quan hệ giữa a,b. Kiểm tra độ lệch, không assert một estimated number duy nhất vì thống kê là mẫu và planner thay đổi giữa phiên bản.

**Lý thuyết gắn với quyết định.** ANALYZE cập nhật thông tin phân phối; tăng statistics target có thể cải thiện thông tin cho cột lệch nhưng tốn thời gian/thống kê lớn hơn. Extended statistics giải quyết một số quan hệ trong cùng bảng, không phải phương thuốc cho mọi join estimate. [Planner statistics](https://www.postgresql.org/docs/17/planner-stats.html) và [CREATE STATISTICS](https://www.postgresql.org/docs/17/sql-createstatistics.html) xác định khả năng từng loại.

**Debug/phục hồi.** Nếu ước lượng đã gần đúng trước, đó không phải bài thất bại: ghi nhận baseline và kiểm tra version/statistics hiện có. Nếu sau vẫn sai, hỏi condition có nằm trong khả năng loại statistics đó không. Chạy lại cell reset q_stats, không cần restart server.

**Biên production.** Sau bulk load hoặc thay đổi phân phối mạnh, query plan có thể đổi dù code chưa deploy. Điều tra cần đối chiếu thời điểm data change, ANALYZE và plan, không chỉ commit application. Nếu áp dụng tăng target, làm trên cột cụ thể và đo overhead; tránh tăng global để xử lý một query mà chưa biết nguyên nhân.

---

# Bài 14 — LOCAL: JOIN và N+1 là bài toán round trip

**Câu hỏi.** Database có xử lý một query nhanh thì endpoint có hết chậm không? Ta đếm số câu SQL cho cùng kết quả, không lấy latency mạng localhost làm đại diện môi trường production.

**Ba cách ghép.** Nested loop có thể tốt khi phía ngoài nhỏ và phía trong có lookup hiệu quả. Hash join xây cấu trúc tra cứu và có thể chia batches khi thiếu bộ nhớ. Merge join khai thác thứ tự đầu vào, nhưng việc tạo thứ tự cũng có chi phí. Tên thuật toán không có thứ bậc tốt/xấu cố định. Hãy nhìn rows, loops và dữ liệu trung gian; với một join inner, 10 rows ở ngoài và 10 triệu rows ở ngoài là hai bài toán khác nhau.

```bash {"name":"db-14-n-plus-one","excludeFromRunAll":"true"}
set -Eeuo pipefail
cd "${HOME}/backend-labs/database"
./db.sh -c "CREATE INDEX IF NOT EXISTS orders_customer_id_idx ON lab.orders(customer_id);"
cat > scripts/nplusone.mjs <<'JS'
import {run,assert} from './harness.mjs';
await run(async a => {
  let calls=0;
  const query=async (sql,args=[]) => { calls++; return a.query(sql,args); };
  const customers=(await query('SELECT id FROM customers ORDER BY id LIMIT 30')).rows;
  const slow=[];
  for (const customer of customers) {
    const row=(await query(
      'SELECT count(*)::int AS n FROM orders WHERE customer_id=$1',[customer.id]
    )).rows[0];
    slow.push([customer.id,row.n]);
  }
  const naiveCalls=calls;
  calls=0;
  const ids=(await query('SELECT id FROM customers ORDER BY id LIMIT 30')).rows.map(r=>r.id);
  const grouped=(await query(
    'SELECT customer_id,count(*)::int AS n FROM orders WHERE customer_id=ANY($1::int[]) GROUP BY customer_id',
    [ids]
  )).rows;
  const byId=new Map(grouped.map(r=>[r.customer_id,r.n]));
  const batched=ids.map(id=>[id,byId.get(id)??0]);
  assert.deepEqual(batched,slow);
  assert.equal(naiveCalls,31);
  assert.equal(calls,2);
  console.log({naiveCalls,batchedCalls:calls,sameResult:true});
});
JS
docker compose run --rm toolbox scripts/nplusone.mjs
./db.sh <<'SQL' | tee evidence/14-joins.txt
EXPLAIN (ANALYZE,BUFFERS,TIMING OFF)
SELECT c.id,count(o.id)
FROM lab.customers c LEFT JOIN lab.orders o ON o.customer_id=c.id
WHERE c.id IN (1,2,3) GROUP BY c.id;
EXPLAIN (ANALYZE,BUFFERS,TIMING OFF)
SELECT c.tenant_id,count(o.id)
FROM lab.customers c JOIN lab.orders o ON o.customer_id=c.id
GROUP BY c.tenant_id;
SQL
```

**Assertion.** Cùng kết quả, 31 query thành 2. Không chứng minh 15,5 lần nhanh hơn: execution, network và materialization khác nhau. Với join một-nhiều, nhớ rằng số row SQL có thể phình theo số con; LIMIT trực tiếp trên tập join không nhất thiết nghĩa “20 customer”. Hãy chọn parent page trước hoặc dùng cấu trúc query phù hợp.

**Debug.** Nếu customer không có order, batch map phải đưa về 0 chứ không làm mất customer. Nếu ORM có relation batching, đo SQL thực tế thay vì đoán từ số lời gọi method. Log query nên có correlation id và tránh lộ dữ liệu nhạy cảm trong parameters.

**Production.** Batch quá lớn có thể thành query nặng hoặc trả một payload khổng lồ. Chọn giới hạn page, chọn cột cần, và quan sát memory ở application. Dataloader cần phạm vi request hoặc tenant phù hợp, tránh cache kết quả sai quyền giữa users. Schema và query giữ đúng rồi mới xem có cần cache bên ngoài không.

---

# Bài 15 — LOCAL: keyset pagination đúng khi timestamp trùng

**Ý định.** Phân trang cần một thứ tự toàn phần. `ORDER BY created_at` chưa đủ nếu nhiều dòng trùng thời gian. Dùng cặp `(created_at,id)` và cursor mang đủ giá trị; cùng hướng sort và phép so sánh.

```bash {"name":"db-15-pagination-correctness","excludeFromRunAll":"true"}
set -Eeuo pipefail
cd "${HOME}/backend-labs/database"
cat > scripts/pagination.mjs <<'JS'
import {run,assert} from './harness.mjs';
await run(async a => {
  await a.query('DROP TABLE IF EXISTS lab.page_demo');
  await a.query('CREATE TABLE lab.page_demo(id bigint PRIMARY KEY,created_at timestamptz NOT NULL)');
  await a.query(
    "INSERT INTO lab.page_demo SELECT i,timestamptz '2026-01-01 00:00:00+00'+(i/3)*interval '1 second' FROM generate_series(1,8) i"
  );
  await a.query('CREATE INDEX page_time_id ON lab.page_demo(created_at DESC,id DESC)');
  const first=(await a.query(
    'SELECT id,created_at FROM lab.page_demo ORDER BY created_at DESC,id DESC LIMIT 2'
  )).rows;
  assert.deepEqual(first.map(r=>r.id),['8','7']);
  const cursor=first.at(-1);
  const wrong=(await a.query(
    'SELECT id FROM lab.page_demo WHERE created_at<$1 ORDER BY created_at DESC,id DESC LIMIT 2',
    [cursor.created_at]
  )).rows.map(r=>r.id);
  assert.deepEqual(wrong,['5','4']);
  await a.query("INSERT INTO lab.page_demo VALUES(9,timestamptz '2026-01-01 00:00:03+00')");
  const offset=(await a.query(
    'SELECT id FROM lab.page_demo ORDER BY created_at DESC,id DESC OFFSET 2 LIMIT 2'
  )).rows.map(r=>r.id);
  const keyset=(await a.query(
    'SELECT id FROM lab.page_demo WHERE (created_at,id)<($1::timestamptz,$2::bigint) ORDER BY created_at DESC,id DESC LIMIT 2',
    [cursor.created_at,cursor.id]
  )).rows.map(r=>r.id);
  assert.deepEqual(offset,['7','6']);
  assert.deepEqual(keyset,['6','5']);
  console.log({first:first.map(r=>r.id),wrong,offset,keyset});
});
JS
docker compose run --rm toolbox scripts/pagination.mjs
./db.sh <<'SQL' | tee evidence/15-pagination-plan.txt
CREATE INDEX IF NOT EXISTS orders_time_id_idx ON lab.orders(created_at DESC,id DESC);
EXPLAIN (ANALYZE,BUFFERS,TIMING OFF)
SELECT id,created_at FROM lab.orders ORDER BY created_at DESC,id DESC OFFSET 5000 LIMIT 20;
EXPLAIN (ANALYZE,BUFFERS,TIMING OFF)
SELECT id,created_at FROM lab.orders
WHERE (created_at,id)<(timestamptz '2026-01-01 00:15:00+00',4500::bigint)
ORDER BY created_at DESC,id DESC LIMIT 20;
SQL
```

**Kết quả đọc được.** Cursor chỉ time làm mất id6. OFFSET sau khi có insert phía trước lặp id7. Keyset theo cặp tiếp tục id6,5. Hai query plan trên bảng lớn chỉ minh họa công việc; chúng không cố trả cùng một trang vì mốc của query keyset là mốc độc lập. Muốn benchmark cùng trang, lấy đúng dòng biên rồi dùng chính cặp đó.

**Điều chưa giải quyết.** Keyset không tự tạo snapshot xuyên nhiều HTTP request. Nếu trường sắp xếp của dòng thay đổi giữa hai trang, dòng có thể di chuyển qua cursor và bị bỏ/lặp. Có thể cần khóa sort bất biến, watermark, snapshot export/report job hoặc đặc tả chấp nhận live feed. NULL cần quy tắc rõ vì row comparison với NULL không giống một giá trị bình thường.

**Cursor API.** Gói đủ các giá trị sort, version schema cursor và bộ lọc liên quan. Base64 chỉ là encoding; nếu cần chống sửa thì ký hoặc xác thực server-side. Cursor không thay thế authorization: luôn lọc tenant và quyền từ session hiện tại, không tin tenant trong cursor. Đồng bộ độ chính xác timestamp giữa DB/driver/API; JavaScript Date có precision khác timestamp microsecond, nên production cần giữ chuỗi chính xác hoặc cursor biểu diễn không làm mất phần thời gian.

**Debug/phục hồi.** Assertion kiểm tra id cụ thể nên nếu sai, đọc thứ tự insert và toán tử tuple trước khi sửa index. Bảng nhỏ page_demo được reset khi chạy lại. Trên production, dùng keyset cho feed/sequential navigation; nếu sản phẩm cần nhảy thẳng trang 10.000, hãy bàn lại UX, giới hạn offset hoặc tạo bảng thứ tự/materialization phù hợp thay vì hứa một cursor làm mọi loại phân trang.

---

# Phần IV — Cạnh tranh: bảo vệ một invariant đã được gọi tên

## 1. Đọc trước rồi ghi sau tạo một khoảng trống

Một transaction có thể hoàn toàn hợp lệ theo từng câu SQL mà vẫn sai về nghiệp vụ. Ví dụ hai request đều đọc balance=100000, mỗi request tính 100000−80000=20000, rồi cùng ghi 20000. Constraint balance>=0 không phản đối. Cả hai có thể tạo audit “đã chấp nhận 80000”, trong khi số dư cuối chỉ phản ánh một lần trừ. Đây là lost update của phép đọc–tính–ghi dùng giá trị cũ.

Nó khác trường hợp viết `balance=balance-80000`. Một phép cập nhật tương đối đánh giá trong database có tương tác khác với concurrent writer; thêm điều kiện `balance>=80000` biến quyết định đủ tiền thành một phần của statement. Từ khóa “atomic” phải gắn với một thao tác và invariant cụ thể: một câu UPDATE đúng không tự bảo vệ việc ghi ledger ở câu sau nếu câu sau chạy ngoài transaction.

Một lỗi khác là write skew. Hai bác sĩ cùng thấy còn hai người trực; mỗi người cập nhật dòng của chính mình thành nghỉ. Không có xung đột ghi cùng dòng, nhưng cả hai commit làm không còn người trực. Khóa mỗi dòng mình sắp cập nhật không bảo vệ điều kiện “tổng số người trực ít nhất một”. Bạn cần khóa tập quyết định đúng, khóa một dòng đại diện cho ca trực, hoặc isolation/giao thức phát hiện được bất thường đó.

## 2. Bảng lựa chọn theo bài toán

| Nhu cầu | Công cụ có thể dùng | Điều phải kiểm tra |
|---|---|---|
| Trừ một lượng khi còn đủ | Conditional UPDATE | rowCount=1; amount>0; ledger cùng transaction |
| Form mở lâu, tránh ghi đè chỉnh sửa khác | Version column | Mọi writer tăng version; xử lý conflict có chủ đích |
| Quyết định dựa trên một vài dòng đã biết | SELECT FOR UPDATE | Khóa trước khi đọc quyết định; thứ tự khóa nhất quán |
| Invariant dựa trên tập/predicate | Unique, exclusion, dòng đại diện hoặc Serializable | Không bỏ writer ngoài giao thức; retry toàn transaction |
| Chia job cho worker | SKIP LOCKED và lease | Không suy ra FIFO tuyệt đối; có reclaim và dedup side effect |
| HTTP retry sau timeout | Idempotency key trong DB | Key có phạm vi, fingerprint và response bền vững |
| Tác động ngoài DB | Outbox/inbox và trạng thái | Cửa sổ crash có đường reconcile |

Không có hàng nào là “tốt nhất cho ngân hàng”. Một payment thường dùng nhiều hàng: unique request key, atomic hoặc locked account update, ledger transaction, outbox event, inbox consumer và provider idempotency. Lựa chọn công cụ riêng lẻ mà thiếu đường kết nối giữa chúng vẫn để lại lỗ hổng.

## 3. Isolation không sửa dữ liệu cũ ở ngoài transaction

RR của PostgreSQL có thể từ chối ghi một row đã bị transaction concurrent cập nhật sau snapshot của mình. Nhưng nếu frontend đọc balance lúc 9 giờ, người khác sửa lúc 10 giờ, rồi 11 giờ backend mở một RR transaction mới và ghi giá trị từ frontend, không có lý do để RR nhận ra đó là dữ liệu cũ. Bạn cần version hoặc điều kiện nghiệp vụ phù hợp.

Serializable mô hình hóa một thứ tự tương đương cho các transaction tham gia trong database. Nó không chứng minh tính đúng của logic tuần tự, không bắt provider hoàn lại tiền, và không làm callback chỉ chạy một lần. Nếu callback có thể retry, những side effect trong callback phải được thiết kế riêng. Trả error 40001 lên trên mà không xử lý có thể giữ dữ liệu đúng nhưng làm API có tỷ lệ lỗi không chấp nhận được.

Trong code, phân loại lỗi theo SQLSTATE/ORM error chính xác. `40001` là serialization failure, `40P01` deadlock, `55P03` lock unavailable/lock timeout, `23505` unique violation. Không retry mù tất cả: amount invalid hoặc cùng key khác payload là lỗi nghiệp vụ, không phải lỗi transient. Với lỗi mạng lúc COMMIT, trạng thái là **chưa biết** cho đến khi tra theo idempotency key, không mặc định thất bại.

---

# Bài 16 — FAULT: tái lập lost update mà không dựa vào sleep

**Ý định.** Chứng minh hai lần chấp nhận debit có thể chỉ làm số dư giảm một lần. Đây là dữ liệu giả; bài không thực hiện rút tiền hay gọi provider thật.

**Tiên quyết.** Bài 01 và harness Bài 04. Không chạy cùng bài nào khác đang sửa wallets/debit_audit. Bộ điều phối chủ động đợi cả A và B đọc xong rồi mới cho A ghi.

```bash {"name":"db-16-lost-update","excludeFromRunAll":"true"}
set -Eeuo pipefail
cd "${HOME}/backend-labs/database"
cat > scripts/lost-update.mjs <<'JS'
import {run,assert,capture} from './harness.mjs';
const mode=process.argv[2]??'rc';
assert.ok(['rc','rr'].includes(mode));
const isolation=mode==='rc'?'READ COMMITTED':'REPEATABLE READ';
await run(async (a,b,o) => {
  await o.query('TRUNCATE lab.debit_audit');
  await o.query('UPDATE wallets SET balance_minor=100000,version=1');
  await a.query('BEGIN ISOLATION LEVEL '+isolation);
  await b.query('BEGIN ISOLATION LEVEL '+isolation);
  const oldA=(await a.query('SELECT balance_minor FROM wallets WHERE id=1')).rows[0].balance_minor;
  const oldB=(await b.query('SELECT balance_minor FROM wallets WHERE id=1')).rows[0].balance_minor;
  assert.equal(oldA,'100000');
  assert.equal(oldB,'100000');
  console.log('Barrier reached: A and B both read 100000');
  await a.query('UPDATE wallets SET balance_minor=$1 WHERE id=1',[(BigInt(oldA)-80000n).toString()]);
  await a.query("INSERT INTO debit_audit VALUES('A',80000)");
  await a.query('COMMIT');
  const updateB=await capture(b.query(
    'UPDATE wallets SET balance_minor=$1 WHERE id=1',[(BigInt(oldB)-80000n).toString()]
  ));
  if (updateB.ok) {
    await b.query("INSERT INTO debit_audit VALUES('B',80000)");
    await b.query('COMMIT');
  } else {
    await b.query('ROLLBACK');
  }
  const balance=(await o.query('SELECT balance_minor FROM wallets WHERE id=1')).rows[0].balance_minor;
  const audit=(await o.query('SELECT count(*)::int AS n,sum(amount_minor)::text AS amount FROM debit_audit')).rows[0];
  assert.equal(balance,'20000');
  if (mode==='rc') {
    assert.equal(updateB.ok,true);
    assert.equal(audit.n,2);
    assert.equal(BigInt(balance)+BigInt(audit.amount),180000n);
    console.log('Expected bug reproduced: balance + accepted debits != initial balance');
  } else {
    assert.equal(updateB.ok,false);
    assert.equal(updateB.error.code,'40001');
    assert.equal(audit.n,1);
    assert.equal(BigInt(balance)+BigInt(audit.amount),100000n);
    console.log('Expected 40001; second DB transaction rolled back');
  }
  console.log({isolation,balance,audit});
});
JS
docker compose run --rm toolbox scripts/lost-update.mjs rc
```

**Mong đợi.** Hai audit, mỗi audit 80000; balance cuối 20000. Bài **thành công khi tái hiện bug**, không phải khi dữ liệu đạt invariant. Phép kiểm tổng `balance + accepted debits` cho 180000 thay vì 100000 làm sai lệch dễ thấy hơn một câu “race condition nguy hiểm”.

**Debug.** Nếu chỉ có một audit ở RC, xem có vô tình chạy RR, constraint khác hoặc lỗi bị nuốt không. Nếu initial balance khác, có workload ngoài bài đang ghi; dừng nó rồi chạy lại. Các khoản tiền được đọc thành string và tính bằng BigInt, tránh đưa lỗi floating point vào một thí nghiệm concurrency.

**Phục hồi.** Cell chạy lại tự reset hai bảng fixture. Không giải quyết bằng cách “thêm một transaction” vì bài đã có transaction. Hãy chọn điều kiện atomic, version, khóa đúng hoặc isolation phù hợp, rồi thử lại cùng lịch chạy này.

---

# Bài 17 — LOCAL: Repeatable Read chặn đúng xung đột nào?

**Mục tiêu.** Chạy cùng thí nghiệm với RR, bắt chính xác 40001, và kiểm tra B không ghi audit.

```bash {"name":"db-17-repeatable-read-conflict","excludeFromRunAll":"true"}
set -Eeuo pipefail
cd "${HOME}/backend-labs/database"
docker compose run --rm toolbox scripts/lost-update.mjs rr
```

**Kỳ vọng.** Chỉ A commit. Balance=20000, accepted debit=80000. Driver nhận 40001 ở B, B rollback. Bài không thực hiện retry tự động vì muốn bạn quan sát conflict. Trong application thực, retry phải đọc lại balance trong transaction mới; khi đó không đủ để trừ 80000 lần nữa.

**Suy nghĩ tiếp.** Giả sử code đã gọi provider trước khi B gặp 40001. Database rollback không hoàn lại side effect đó. Hoặc giả sử catch error rồi chỉ thử lại UPDATE với balance cũ 20000: retry như vậy có thể khôi phục bug thay vì sửa. Giao thức cần retry toàn bộ quyết định trong phạm vi an toàn, cùng idempotency cho hiệu ứng ngoài database.

**Debug/phục hồi.** Nếu không có 40001, kiểm tra cả hai SELECT đã lấy snapshot trước A commit. Harness in rào chắn để hỗ trợ giải thích. Script tự rollback và đóng client; nếu dừng container đột ngột, server cũng kết thúc session/transaction khi connection mất. Đừng giữ tab psql ở trạng thái failed transaction rồi tiếp tục test.

**Biên production.** Đây là hành vi PostgreSQL, không phải bảo đảm có thể sao sang MySQL bằng việc chọn cùng tên isolation. Bài 32 sẽ chạy thí nghiệm tương ứng trên InnoDB. Dùng test theo invariant và driver thực tế trước khi đổi database backend.

---

# Bài 18 — FAULT/LOCAL: write skew và Serializable

**Câu hỏi.** Hai transaction không cập nhật cùng dòng có thể cùng phá một điều kiện chéo không? Ta dùng quy tắc “luôn có ít nhất một người trực”.

```bash {"name":"db-18-write-skew","excludeFromRunAll":"true"}
set -Eeuo pipefail
cd "${HOME}/backend-labs/database"
cat > scripts/write-skew.mjs <<'JS'
import {run,assert,capture} from './harness.mjs';
await run(async (a,b,o) => {
  await o.query('DROP TABLE IF EXISTS lab.on_call');
  await o.query('CREATE TABLE lab.on_call(id int PRIMARY KEY,working boolean NOT NULL)');
  for (const isolation of ['REPEATABLE READ','SERIALIZABLE']) {
    await o.query('TRUNCATE lab.on_call');
    await o.query('INSERT INTO lab.on_call VALUES(1,true),(2,true)');
    await a.query('BEGIN ISOLATION LEVEL '+isolation);
    await b.query('BEGIN ISOLATION LEVEL '+isolation);
    const read='SELECT count(*)::int AS n FROM lab.on_call WHERE working';
    assert.equal((await a.query(read)).rows[0].n,2);
    assert.equal((await b.query(read)).rows[0].n,2);
    console.log('Barrier: both transactions decided from count=2',isolation);
    const ua=await capture(a.query('UPDATE lab.on_call SET working=false WHERE id=1'));
    const ub=await capture(b.query('UPDATE lab.on_call SET working=false WHERE id=2'));
    const ca=ua.ok?await capture(a.query('COMMIT')):ua;
    if (!ca.ok) await a.query('ROLLBACK');
    const cb=ub.ok?await capture(b.query('COMMIT')):ub;
    if (!cb.ok) await b.query('ROLLBACK');
    const errors=[ca,cb].filter(r=>!r.ok).map(r=>r.error.code);
    const remaining=(await o.query(read)).rows[0].n;
    if (isolation==='REPEATABLE READ') {
      assert.deepEqual(errors,[]);
      assert.equal(remaining,0);
    } else {
      assert.deepEqual(errors,['40001']);
      assert.equal(remaining,1);
    }
    console.log({isolation,remaining,errors});
  }
});
JS
docker compose run --rm toolbox scripts/write-skew.mjs
```

**Mong đợi.** RR cho cả hai nghỉ, remaining=0. Serializable loại một transaction bằng 40001; không phụ thuộc phải là A hay B. Error có thể xuất hiện lúc UPDATE hoặc COMMIT, nên code phải xử lý cả hai. Các transaction commit còn lại giữ một người trực.

**Thiết kế thay thế.** Có thể tạo một dòng `shift` đại diện ca trực và buộc mọi thay đổi khóa dòng đó trước khi đếm. Cách này đơn giản hóa chứng minh nhưng tuần tự hóa thay đổi của cùng ca. Có thể materialize counter với conditional update, nhưng phải giữ counter đúng với từng thay đổi chi tiết. Không tự chọn isolation cao hay counter trước khi hiểu quy mô contention và mọi đường ghi.

**Debug.** Không thêm FOR UPDATE vào query count một cách máy móc: aggregate không trực tiếp cho bạn khóa một “dòng tổng” tồn tại. Nếu muốn khóa các bác sĩ hiện tại, còn phải xét insert/delete và phạm vi predicate. Hãy vẽ những transaction được phép chạy và chứng minh đường nào ngăn cả hai quyết định nghỉ.

**Phục hồi/production.** Bài tự reset on_call. Retry serializable cần backoff, jitter, giới hạn số lần và ngân sách latency; đừng retry vô hạn dưới contention cao. Logic retry không được tự động gọi lại provider không có idempotency. Khi tỷ lệ serialization failure cao, xem lại hot key và thiết kế workflow trước khi chỉ tăng số lần retry.

---

# Bài 19 — LOCAL: conditional UPDATE và kiểm tra affected rows

**Ý định.** Đưa kiểm tra đủ tiền vào cùng statement thay đổi balance. Ta còn chứng minh B thực sự chờ A rồi đánh giá lại điều kiện trên kết quả mới.

```bash {"name":"db-19-atomic-conditional-debit","excludeFromRunAll":"true"}
set -Eeuo pipefail
cd "${HOME}/backend-labs/database"
cat > scripts/atomic-debit.mjs <<'JS'
import {run,assert,capture,waitBlocked} from './harness.mjs';
await run(async (a,b,o,[pidA,pidB]) => {
  await o.query('UPDATE wallets SET balance_minor=100000 WHERE id=1');
  await a.query('BEGIN');
  await b.query('BEGIN');
  const sql='UPDATE wallets SET balance_minor=balance_minor-$1 WHERE id=$2 AND balance_minor >= $1 RETURNING balance_minor';
  const ra=await a.query(sql,['80000',1]);
  assert.equal(ra.rowCount,1);
  const pending=capture(b.query(sql,['80000',1]));
  await waitBlocked(o,pidB,pidA);
  await a.query('COMMIT');
  const rb=await pending;
  assert.equal(rb.ok,true);
  assert.equal(rb.value.rowCount,0);
  await b.query('COMMIT');
  const end=(await o.query('SELECT balance_minor FROM wallets WHERE id=1')).rows[0].balance_minor;
  assert.equal(end,'20000');
  console.log({acceptedA:ra.rowCount,acceptedB:rb.value.rowCount,balance:end});
});
JS
docker compose run --rm toolbox scripts/atomic-debit.mjs
```

**Assertion.** A thay đổi một dòng, B thay đổi không dòng, balance=20000. Query “không lỗi” chưa có nghĩa debit thành công; `rowCount=0` là một kết quả nghiệp vụ cần xử lý. Trong API thật, phân biệt account không tồn tại, bị khóa, sai currency hay thiếu tiền có thể cần thiết kế response riêng mà không tiết lộ thông tin ngoài quyền.

**Điều kiện đầu vào.** Amount phải dương, đúng currency, trong giới hạn và biểu diễn chính xác. Nếu amount âm, câu `balance=balance-amount` thành cộng tiền; constraint balance>=0 không ngăn được. Điều kiện state và tenant cũng thuộc predicate hoặc bước authorization phù hợp. Ghi ledger và outbox trong cùng transaction sau khi rowCount=1.

**Debug/phục hồi.** Nếu waitBlocked timeout, xem statement của A có đang autocommit vì quên BEGIN không. Nếu B còn pending, nhìn lock_timeout và session PID. Script finally rollback để không để khóa sót. Chạy lại reset wallet1.

**Biên production.** Atomic update một dòng không bảo vệ tổng hạn mức của nhiều account, tổng quota ngày hoặc ledger hai phía nếu phần còn lại nằm ngoài transaction. Khi cần kết hợp nhiều điều kiện, hãy xác định điểm tuần tự hóa và kiểm tra invariant sau mọi nhánh failure.

---

# Bài 20 — LOCAL: pessimistic và optimistic cho hai kiểu tương tác

## 20A. Khóa trước khi ra quyết định

**Ý định.** B đọc balance sau khi A hoàn tất thay đổi, nhờ locking read chờ đúng holder. Cách này hữu ích khi logic cần nhiều bước trên các dòng đã biết, miễn transaction ngắn.

```bash {"name":"db-20-pessimistic","excludeFromRunAll":"true"}
set -Eeuo pipefail
cd "${HOME}/backend-labs/database"
cat > scripts/pessimistic.mjs <<'JS'
import {run,assert,capture,waitBlocked} from './harness.mjs';
await run(async (a,b,o,[pidA,pidB]) => {
  await o.query('UPDATE wallets SET balance_minor=100000 WHERE id=1');
  await a.query('BEGIN');
  await b.query('BEGIN');
  const ra=await a.query('SELECT balance_minor FROM wallets WHERE id=1 FOR UPDATE');
  assert.equal(ra.rows[0].balance_minor,'100000');
  const pending=capture(b.query('SELECT balance_minor FROM wallets WHERE id=1 FOR UPDATE'));
  await waitBlocked(o,pidB,pidA);
  await a.query('UPDATE wallets SET balance_minor=20000 WHERE id=1');
  await a.query('COMMIT');
  const rb=await pending;
  assert.equal(rb.ok,true);
  assert.equal(rb.value.rows[0].balance_minor,'20000');
  await b.query('COMMIT');
  console.log('B acquired lock and saw committed balance=20000');
});
JS
docker compose run --rm toolbox scripts/pessimistic.mjs
```

**Mong đợi.** Observer thấy cạnh B→A; B nhận balance=20000 sau A commit. Đó là bằng chứng khóa đã áp dụng trước bước quyết định. Nếu bạn SELECT thường trước, xử lý logic rồi mới FOR UPDATE nhưng không đọc/kiểm tra lại, khoảng trống vẫn còn.

**Giới hạn.** Khóa row không tự khóa “một row chưa tồn tại” theo cùng cách trên mọi engine. Với yêu cầu không được có booking trùng, unique/exclusion thường là invariant mạnh hơn kiểm tra SELECT rồi INSERT. [Explicit locking PostgreSQL](https://www.postgresql.org/docs/17/explicit-locking.html) cũng mô tả lock modes và deadlock; chọn FOR UPDATE cho mọi query có thể giữ khóa mạnh hơn cần thiết.

## 20B. Version cho form hoặc quyết định dựa trên dữ liệu cũ

**Ý định.** Hai người đọc version1. A lưu version2; B cố lưu bằng version1 phải nhận conflict để quyết định reload/merge, không ghi đè im lặng.

```bash {"name":"db-20-optimistic-version","excludeFromRunAll":"true"}
set -Eeuo pipefail
cd "${HOME}/backend-labs/database"
cat > scripts/optimistic.mjs <<'JS'
import {run,assert} from './harness.mjs';
await run(async (a,b,o) => {
  await o.query('UPDATE wallets SET balance_minor=100000,version=1 WHERE id=1');
  const oldA=(await a.query('SELECT version FROM wallets WHERE id=1')).rows[0].version;
  const oldB=(await b.query('SELECT version FROM wallets WHERE id=1')).rows[0].version;
  const sql='UPDATE wallets SET balance_minor=$1,version=version+1 WHERE id=$2 AND version=$3 RETURNING version';
  const ra=await a.query(sql,['90000',1,oldA]);
  const rb=await b.query(sql,['80000',1,oldB]);
  assert.equal(ra.rowCount,1);
  assert.equal(rb.rowCount,0);
  const row=(await o.query('SELECT balance_minor,version FROM wallets WHERE id=1')).rows[0];
  assert.deepEqual(row,{balance_minor:'90000',version:'2'});
  console.log('Second writer must handle a version conflict:',row);
});
JS
docker compose run --rm toolbox scripts/optimistic.mjs
```

**Kết quả.** A ghi được, B không ghi. Mọi writer của tài nguyên phải tuân theo version; một batch job không tăng version sẽ làm giao thức thiếu tin cậy. Version nên gắn với phạm vi aggregate mà quyết định dựa vào, không tự mặc định một cột version của một row đại diện cho mọi bảng con.

**Debug/phục hồi.** Cả hai bài tự reset wallet1. Nếu xung đột được bỏ qua và API vẫn trả thành công, lỗi ở việc diễn giải affected rows. Có thể dùng HTTP 409/412 theo contract, nhưng cần cung cấp khả năng người dùng xem thay đổi mới; retry tự động một bản chỉnh sửa cũ có thể làm mất ý định của người sửa.

**Production.** Pessimistic thích hợp công việc ngắn dưới kiểm soát server; optimistic thích hợp lúc suy nghĩ/chỉnh sửa kéo dài ngoài transaction. Chọn dựa trên tần suất xung đột, chi phí retry và ý nghĩa merge, không dựa vào tên gọi “lạc quan” hay “bi quan”.

---

# Bài 21 — FAULT: deadlock và thứ tự khóa

**Câu hỏi.** Vì sao mỗi transaction chỉ giữ hai row lock mà vẫn có thể kẹt? Ta buộc A giữ ví1, B giữ ví2, rồi A chờ ví2 trước khi B xin ví1. Đây là một vòng chờ thật, không phải hy vọng hai process va nhau.

```bash {"name":"db-21-deadlock","excludeFromRunAll":"true"}
set -Eeuo pipefail
cd "${HOME}/backend-labs/database"
cat > scripts/deadlock.mjs <<'JS'
import {run,assert,capture,waitBlocked} from './harness.mjs';
await run(async (a,b,o,[pidA,pidB]) => {
  await o.query('UPDATE wallets SET balance_minor=100000');
  await a.query('BEGIN');
  await b.query('BEGIN');
  await a.query('UPDATE wallets SET balance_minor=balance_minor-100 WHERE id=1');
  await b.query('UPDATE wallets SET balance_minor=balance_minor-100 WHERE id=2');
  const pa=capture(a.query('UPDATE wallets SET balance_minor=balance_minor+100 WHERE id=2'));
  await waitBlocked(o,pidA,pidB);
  const pb=capture(b.query('UPDATE wallets SET balance_minor=balance_minor+100 WHERE id=1'));
  const outcomes=await Promise.all([pa,pb]);
  for (const [index,client] of [a,b].entries()) {
    await client.query(outcomes[index].ok?'COMMIT':'ROLLBACK');
  }
  const errors=outcomes.filter(r=>!r.ok).map(r=>r.error.code);
  assert.deepEqual(errors,['40P01']);
  const total=(await o.query('SELECT sum(balance_minor)::text AS total FROM wallets')).rows[0].total;
  assert.equal(total,'200000');
  console.log({errors,total,committed:outcomes.filter(r=>r.ok).length});
});
JS
docker compose run --rm toolbox scripts/deadlock.mjs
```

**Mong đợi.** Chính xác một transaction bị 40P01 trong lịch chạy mẫu, transaction còn lại commit, tổng balance không đổi. Không mặc định A hay B là nạn nhân; chọn nạn nhân không phải contract của application. Nếu lỗi là 55P03, có thể môi trường quá chậm hoặc tham số timeout đã đổi; đó không phải bằng chứng deadlock detector đã chạy.

**Biện pháp giảm.** Khi khóa nhiều tài khoản đã biết, khóa theo thứ tự khóa chuẩn thống nhất trước khi thay đổi. Mọi code path phải cùng quy tắc: chuyển tiền, refund, reconcile, batch và sửa dữ liệu. Nhưng thứ tự id trong một bảng không loại bỏ mọi deadlock có thể đến từ unique index, foreign key, DDL, trigger hoặc tài nguyên khác. Vẫn cần retry đúng loại lỗi và quan sát tỷ lệ.

**Thử thiết kế lành mạnh.** Thay hai câu UPDATE đầu bằng SELECT cả hai ví theo ORDER BY id FOR UPDATE trong mỗi transaction; giao dịch thứ hai sẽ chờ ở cùng điểm. Không giữ khóa trong khi gửi HTTP. Giữ tổng số row khóa và thời gian transaction nhỏ. Ghi lại đồ thị chờ từ log để xác định toàn bộ tài nguyên tham gia thay vì chỉ sửa câu SQL xuất hiện cuối.

**Debug/phục hồi.** Script đóng cả ba connection. Nếu bạn dừng cell ở giữa, xác minh session db-lab không còn trong pg_stat_activity trước bài tiếp. Không terminate tất cả backend của database như cách “gỡ deadlock”; server đã hủy một nạn nhân, và hủy hàng loạt có thể biến một xung đột nhỏ thành gián đoạn lớn.

---

# Bài 22 — LOCAL/FAULT: SKIP LOCKED, lease và worker cũ

**Ý định.** Chia việc không chờ một row đang bị giữ, sau đó chứng minh reclaim cần token để worker cũ không ghi completion sai. Bài dùng database như queue nhỏ để học concurrency; khi chọn broker hay DB queue, còn phải xét throughput, retention, scheduling và vận hành.

```bash {"name":"db-22-skip-locked-and-lease","excludeFromRunAll":"true"}
set -Eeuo pipefail
cd "${HOME}/backend-labs/database"
cat > scripts/jobs.mjs <<'JS'
import {run,assert} from './harness.mjs';
await run(async (a,b,o) => {
  await o.query('DROP TABLE IF EXISTS lab.jobs');
  await o.query("CREATE TABLE lab.jobs(id int PRIMARY KEY,status text NOT NULL DEFAULT 'ready',lease_token bigint NOT NULL DEFAULT 0,lease_until timestamptz)");
  await o.query('INSERT INTO lab.jobs(id) SELECT generate_series(1,5)');
  await a.query('BEGIN');
  const held=(await a.query(
    "SELECT id FROM lab.jobs WHERE status='ready' ORDER BY id LIMIT 1 FOR UPDATE"
  )).rows[0].id;
  assert.equal(held,1);
  await b.query('BEGIN');
  const claimed=(await b.query(
    "WITH pick AS (SELECT id FROM lab.jobs WHERE status='ready' ORDER BY id LIMIT 1 FOR UPDATE SKIP LOCKED) UPDATE lab.jobs j SET status='processing',lease_token=lease_token+1,lease_until=clock_timestamp()+interval '1 minute' FROM pick WHERE j.id=pick.id RETURNING j.id,j.lease_token"
  )).rows[0];
  assert.equal(claimed.id,2);
  assert.equal(claimed.lease_token,'1');
  await b.query('COMMIT');
  await a.query('ROLLBACK');
  console.log('Locked job 1 skipped; job 2 claimed and committed');
  await o.query("UPDATE lab.jobs SET lease_until=clock_timestamp()-interval '1 second' WHERE id=2");
  const reclaimed=(await o.query(
    "UPDATE lab.jobs SET lease_token=lease_token+1,lease_until=clock_timestamp()+interval '1 minute' WHERE id=2 AND status='processing' AND lease_until<clock_timestamp() RETURNING lease_token"
  )).rows[0].lease_token;
  assert.equal(reclaimed,'2');
  const stale=await a.query(
    "UPDATE lab.jobs SET status='done' WHERE id=2 AND status='processing' AND lease_token=$1",
    [claimed.lease_token]
  );
  assert.equal(stale.rowCount,0);
  const current=await b.query(
    "UPDATE lab.jobs SET status='done' WHERE id=2 AND status='processing' AND lease_token=$1",
    [reclaimed]
  );
  assert.equal(current.rowCount,1);
  console.log('Stale completion rejected; current owner completed once');
});
JS
docker compose run --rm toolbox scripts/jobs.mjs
```

**Kết quả.** Khi A đang giữ id1, B lấy id2; không cần sleep. A rollback làm id1 vẫn ready. Ta chủ động đặt expiry vào quá khứ để mô phỏng worker không heartbeat; token2 thay token1. Completion từ token1 bị từ chối.

**Điều SKIP LOCKED không hứa.** Nó cho góc nhìn bỏ qua một số row đang khóa, thích hợp cho kiểu phân phối công việc nhất định. Không dùng nó như bảo đảm generic consistency, không dùng để kết luận “không còn hàng tồn”, và không coi ORDER BY là FIFO tuyệt đối khi các row đầu bị skip. Row có thể bị bỏ qua lâu nếu liên tục bị giữ; cần theo dõi tuổi job. [SELECT locking clauses](https://www.postgresql.org/docs/17/sql-select.html) nêu giới hạn của góc nhìn này.

**Lease không bằng exactly once.** Worker cũ có thể đã gửi email trước khi phát hiện mất lease. Conditional completion bảo vệ trạng thái DB, không ngăn side effect tại một hệ thống không kiểm tra token. Dùng idempotency ở đích, inbox hoặc reconcile cho hiệu ứng. Token phải tăng đơn điệu trong phạm vi tài nguyên nếu dùng như fencing; UUID ngẫu nhiên nhận diện owner nhưng không cung cấp thứ tự cũ/mới.

**Debug/phục hồi.** Nếu B lấy id1, A chưa giữ transaction/lock đúng; kiểm tra BEGIN và await. Nếu cả hai completion thành công, WHERE thiếu token hoặc code bỏ qua rowCount. Chạy lại cell reset jobs. Trong production, claim batch nhỏ, commit nhanh, xử lý bên ngoài transaction, heartbeat/reclaim bằng clock database và theo dõi retry count/dead-letter theo chính sách.

---

# Phần V — Một request nghiệp vụ đi qua database và hệ thống ngoài

## 1. Idempotency là một hợp đồng có dữ liệu bền vững

Client timeout có thể gửi lại cùng yêu cầu. Worker crash có thể xử lý lại event. Provider webhook có thể đến nhiều lần hoặc ngoài thứ tự. Thiết kế đáng tin cậy không giả định các trường hợp ấy sẽ hiếm đến mức bỏ qua; nó phải phân biệt được “yêu cầu mới” và “lặp lại yêu cầu cũ”.

Một idempotency key cần phạm vi. `request-123` của tenant A không phải cùng request với `request-123` của tenant B. Một key cho endpoint chuyển tiền không nhất thiết dùng chung với endpoint refund. Khóa dữ liệu thường gồm tenant/principal hoặc scope hợp lý, operation và key. Kiểm tra quyền phải xảy ra theo identity xác thực hiện tại; không cho client tự chọn một scope để đọc response của người khác.

Fingerprint gắn key với nội dung có ý nghĩa. Nếu lần đầu chuyển 10 EUR, lần sau dùng cùng key để chuyển 100 EUR, không nên im lặng trả “thành công” như thể request mới đã được thực hiện. Phải canonicalize dữ liệu được hash: amount theo biểu diễn chuẩn, currency, source, destination, API version và các yếu tố ảnh hưởng hành vi. Một JSON stringify thay đổi thứ tự key không phải canonicalization đủ tin cậy nếu dùng tùy tiện giữa nhiều ngôn ngữ.

Kết quả lưu lại cũng là hợp đồng. Khi replay, API có trả cùng transfer id, status và amount không? Nếu transaction chưa commit, request thứ hai chờ, nhận in-progress hay retry-after? Nếu request đầu bị lỗi nghiệp vụ, có lưu lỗi để replay hay cho phép thử lại sau khi số dư đổi? Không có một chính sách đúng cho mọi API; sách chọn rollback cả marker nếu transaction thất bại, và ghi rõ sự lựa chọn.

TTL cần khớp với thời gian có thể retry, retention nghiệp vụ và khả năng reconcile. Xóa key quá sớm có thể khiến một request muộn thành giao dịch mới. Nếu ledger giữ vĩnh viễn business reference thì unique ở ledger còn giúp ngăn lặp, nhưng response cũ và hành vi replay vẫn cần thiết kế. Cache Redis có thể tăng tốc lookup; tính đúng của lần chấp nhận đầu tiên cần điểm kiểm soát bền vững phù hợp.

## 2. Outbox/inbox xử lý khoảng trống giữa hai commit

Giả sử ghi transaction vào DB xong mới publish event. Process chết giữa hai bước làm payment tồn tại mà event chưa được gửi. Publish trước rồi ghi DB có khoảng trống ngược lại: consumer đã thấy event của transaction chưa commit hoặc cuối cùng rollback. Không có thứ tự hai lệnh đơn giản nào loại bỏ cả hai cửa sổ lỗi.

Outbox ghi “cần gửi event này” trong cùng transaction với thay đổi nghiệp vụ. Dispatcher sau đó gửi và đánh dấu. Nếu chết sau broker ack nhưng trước đánh dấu, event có thể được gửi lại. Vì vậy outbox thường tạo nền cho **at-least-once delivery**, và consumer phải chịu duplicate. Inbox lưu event đã xử lý, cùng transaction với hiệu ứng trong DB consumer, để một delivery lặp không áp dụng hiệu ứng lần nữa.

Đây là lập luận về cửa sổ lỗi, không phải một tuyên bố exactly-once toàn hệ thống. Nếu consumer gọi dịch vụ ngoài, inbox và database của consumer không tự commit cùng dịch vụ ngoài đó. Bạn phải tiếp tục dùng provider idempotency, state machine, reconciliation hoặc workflow khác. Mỗi ranh giới durable cần một câu trả lời riêng cho “đã xảy ra nhưng response thất lạc”.

---

# Bài 23 — LOCAL: chuyển tiền có idempotency, ledger và outbox trong một transaction

**Mục tiêu.** Hai session dùng cùng key chỉ tạo một transfer, hai ledger entries và một outbox event. Cùng key nhưng khác amount bị từ chối. Thiếu tiền không để lại debit hoặc marker hoàn tất nửa chừng.

**Phạm vi.** Đây là mô hình học tập trong schema `pay`, không phải implementation có thể copy nguyên vào ECB. Chưa có authorization hoàn chỉnh, compliance, FX, provider attempt, holds, reversal hay hạn mức tổng. Function đặt invariant cục bộ dễ quan sát; production có thể thực hiện cùng giao thức trong service/repository với transaction client và quyền DB thích hợp.

**Chuẩn bị.** Cell reset toàn bộ schema nghiệp vụ mẫu bằng danh sách table cụ thể. Tài khoản mẫu thuộc scope t1, cùng EUR, mỗi tài khoản 100000 đơn vị nhỏ.

```bash {"name":"db-23-payment-schema","excludeFromRunAll":"true"}
set -Eeuo pipefail
cd "${HOME}/backend-labs/database"
./db.sh <<'SQL'
CREATE EXTENSION IF NOT EXISTS pgcrypto;
CREATE SCHEMA IF NOT EXISTS pay;
DROP FUNCTION IF EXISTS pay.post_transfer(text,text,integer,integer,bigint);
DROP TABLE IF EXISTS pay.consumer_effects;
DROP TABLE IF EXISTS pay.inbox;
DROP TABLE IF EXISTS pay.outbox;
DROP TABLE IF EXISTS pay.entries;
DROP TABLE IF EXISTS pay.transfers;
DROP TABLE IF EXISTS pay.requests;
DROP TABLE IF EXISTS pay.accounts;

CREATE TABLE pay.accounts(
  scope text NOT NULL,
  id integer NOT NULL,
  currency text NOT NULL,
  balance_minor bigint NOT NULL CHECK(balance_minor>=0),
  PRIMARY KEY(scope,id)
);
INSERT INTO pay.accounts VALUES('t1',1,'EUR',100000),('t1',2,'EUR',100000);

CREATE TABLE pay.requests(
  scope text NOT NULL,
  request_key text NOT NULL,
  fingerprint text NOT NULL,
  response jsonb,
  created_at timestamptz NOT NULL DEFAULT clock_timestamp(),
  PRIMARY KEY(scope,request_key)
);
CREATE TABLE pay.transfers(
  id bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  scope text NOT NULL,
  request_key text NOT NULL,
  source_id integer NOT NULL,
  destination_id integer NOT NULL,
  amount_minor bigint NOT NULL CHECK(amount_minor>0),
  currency text NOT NULL,
  UNIQUE(scope,request_key),
  FOREIGN KEY(scope,request_key) REFERENCES pay.requests(scope,request_key),
  FOREIGN KEY(scope,source_id) REFERENCES pay.accounts(scope,id),
  FOREIGN KEY(scope,destination_id) REFERENCES pay.accounts(scope,id),
  CHECK(source_id<>destination_id)
);
CREATE TABLE pay.entries(
  id bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  transfer_id bigint NOT NULL REFERENCES pay.transfers(id),
  scope text NOT NULL,
  account_id integer NOT NULL,
  currency text NOT NULL,
  delta_minor bigint NOT NULL CHECK(delta_minor<>0),
  FOREIGN KEY(scope,account_id) REFERENCES pay.accounts(scope,id)
);
CREATE INDEX pay_entries_account_idx ON pay.entries(scope,account_id,id);
CREATE INDEX pay_entries_transfer_idx ON pay.entries(transfer_id);
CREATE TABLE pay.outbox(
  event_id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  transfer_id bigint NOT NULL UNIQUE REFERENCES pay.transfers(id),
  payload jsonb NOT NULL,
  created_at timestamptz NOT NULL DEFAULT clock_timestamp(),
  published_at timestamptz
);
CREATE INDEX pay_outbox_pending_idx ON pay.outbox(created_at,event_id)
WHERE published_at IS NULL;

CREATE FUNCTION pay.post_transfer(
  p_scope text,p_key text,p_source integer,p_destination integer,p_amount bigint
) RETURNS jsonb LANGUAGE plpgsql AS $$
DECLARE
  v_fingerprint text;
  v_existing pay.requests%ROWTYPE;
  v_inserted integer;
  v_locked integer;
  v_source_currency text;
  v_destination_currency text;
  v_balance bigint;
  v_transfer bigint;
  v_response jsonb;
BEGIN
  IF p_scope IS NULL OR btrim(p_scope)='' OR p_key IS NULL
     OR btrim(p_key)='' OR length(p_key)>200 THEN
    RAISE EXCEPTION 'invalid request scope or key';
  END IF;
  IF p_amount IS NULL OR p_amount<=0 OR p_source IS NULL
     OR p_destination IS NULL OR p_source=p_destination THEN
    RAISE EXCEPTION 'invalid transfer arguments';
  END IF;
  v_fingerprint := encode(digest(
    jsonb_build_array('transfer-v1',p_scope,p_source,p_destination,p_amount::text)::text,
    'sha256'),'hex');
  INSERT INTO pay.requests(scope,request_key,fingerprint)
  VALUES(p_scope,p_key,v_fingerprint)
  ON CONFLICT(scope,request_key) DO NOTHING;
  GET DIAGNOSTICS v_inserted=ROW_COUNT;
  IF v_inserted=0 THEN
    SELECT * INTO STRICT v_existing FROM pay.requests
    WHERE scope=p_scope AND request_key=p_key;
    IF v_existing.fingerprint<>v_fingerprint THEN
      RAISE EXCEPTION 'idempotency key reused with different payload';
    END IF;
    IF v_existing.response IS NULL THEN
      RAISE EXCEPTION 'incomplete request state requires investigation';
    END IF;
    RETURN v_existing.response;
  END IF;

  PERFORM id FROM pay.accounts
  WHERE scope=p_scope AND id IN (p_source,p_destination)
  ORDER BY id FOR UPDATE;
  GET DIAGNOSTICS v_locked=ROW_COUNT;
  IF v_locked<>2 THEN RAISE EXCEPTION 'account not found in scope'; END IF;
  SELECT currency INTO v_source_currency FROM pay.accounts
  WHERE scope=p_scope AND id=p_source;
  SELECT currency INTO v_destination_currency FROM pay.accounts
  WHERE scope=p_scope AND id=p_destination;
  IF v_source_currency<>v_destination_currency THEN
    RAISE EXCEPTION 'currency mismatch';
  END IF;
  UPDATE pay.accounts SET balance_minor=balance_minor-p_amount
  WHERE scope=p_scope AND id=p_source AND balance_minor>=p_amount
  RETURNING balance_minor INTO v_balance;
  IF NOT FOUND THEN RAISE EXCEPTION 'insufficient funds'; END IF;
  UPDATE pay.accounts SET balance_minor=balance_minor+p_amount
  WHERE scope=p_scope AND id=p_destination;

  INSERT INTO pay.transfers(scope,request_key,source_id,destination_id,amount_minor,currency)
  VALUES(p_scope,p_key,p_source,p_destination,p_amount,v_source_currency)
  RETURNING id INTO v_transfer;
  INSERT INTO pay.entries(transfer_id,scope,account_id,currency,delta_minor)
  VALUES(v_transfer,p_scope,p_source,v_source_currency,-p_amount),
        (v_transfer,p_scope,p_destination,v_source_currency,p_amount);
  INSERT INTO pay.outbox(transfer_id,payload)
  VALUES(v_transfer,jsonb_build_object(
    'type','TransferPosted','schema_version',1,'scope',p_scope,
    'transfer_id',v_transfer::text,'amount_minor',p_amount::text,
    'currency',v_source_currency));
  v_response := jsonb_build_object(
    'transfer_id',v_transfer::text,'balance_minor',v_balance::text,'status','POSTED');
  UPDATE pay.requests SET response=v_response
  WHERE scope=p_scope AND request_key=p_key;
  RETURN v_response;
END $$;
SQL
```

Hàm digest đến từ [pgcrypto](https://www.postgresql.org/docs/17/pgcrypto.html). Fingerprint trong bài dùng array có thứ tự với dữ liệu đã chuẩn hóa, không nhận một chuỗi hash tùy ý từ client làm bằng chứng tin cậy. Database key là scope/key; authorization vẫn phải xác định scope mà người gọi được phép sử dụng.

**Hành động concurrent.** A thực thi function nhưng chưa COMMIT. B gọi cùng key và bị unique conflict chặn đến khi A kết thúc. Ở RC, các statement bên trong function có thể đọc kết quả đã commit khi tiếp tục. Nếu dùng isolation khác, phải kiểm tra lại conflict/retry; bài không hứa mọi isolation có cùng đường đi.

```bash {"name":"db-23-payment-concurrency","excludeFromRunAll":"true"}
set -Eeuo pipefail
cd "${HOME}/backend-labs/database"
cat > scripts/payment.mjs <<'JS'
import {run,assert,capture,waitBlocked} from './harness.mjs';
await run(async (a,b,o,[pidA,pidB]) => {
  const sql='SELECT pay.post_transfer($1,$2,$3,$4,$5) AS result';
  const args=['t1','payment-001',1,2,'10000'];
  await a.query('BEGIN ISOLATION LEVEL READ COMMITTED');
  await b.query('BEGIN ISOLATION LEVEL READ COMMITTED');
  const first=(await a.query(sql,args)).rows[0].result;
  const pending=capture(b.query(sql,args));
  await waitBlocked(o,pidB,pidA);
  await a.query('COMMIT');
  const replay=await pending;
  assert.equal(replay.ok,true);
  assert.deepEqual(replay.value.rows[0].result,first);
  await b.query('COMMIT');
  const bad=await capture(o.query(sql,['t1','payment-001',1,2,'10001']));
  assert.equal(bad.ok,false);
  assert.equal(bad.error.code,'P0001');
  assert.match(bad.error.message,/different payload/);
  const poor=await capture(o.query(sql,['t1','too-large',1,2,'999999']));
  assert.equal(poor.ok,false);
  assert.match(poor.error.message,/insufficient funds/);
  const audit=(await o.query(
    "SELECT (SELECT count(*)::int FROM pay.transfers) AS transfers,(SELECT count(*)::int FROM pay.entries) AS entries,(SELECT count(*)::int FROM pay.outbox) AS events,(SELECT count(*)::int FROM pay.requests WHERE request_key='too-large') AS failed_markers"
  )).rows[0];
  assert.deepEqual(audit,{transfers:1,entries:2,events:1,failed_markers:0});
  const accounts=(await o.query('SELECT id,balance_minor FROM pay.accounts ORDER BY id')).rows;
  assert.deepEqual(accounts,[{id:1,balance_minor:'90000'},{id:2,balance_minor:'110000'}]);
  const imbalanced=await o.query(
    'SELECT transfer_id,currency FROM pay.entries GROUP BY transfer_id,currency HAVING sum(delta_minor)<>0 OR count(*)<>2'
  );
  assert.equal(imbalanced.rowCount,0);
  console.log({sameResponse:true,first,audit,accounts});
});
JS
docker compose run --rm toolbox scripts/payment.mjs
```

**Mong đợi.** Một transfer, hai entries cân bằng, một event, balance 90000/110000 và response replay giống nhau. Không có marker của request thiếu tiền, vì sách chọn rollback toàn request thất bại. Nếu chạy lại test mà không reset schema, replay đầu tiên không còn block; hãy chạy cell schema trước khi tái lập lịch cạnh tranh.

**Phân tích lỗi.** Mọi INSERT, hai UPDATE và response ở trong transaction của caller. Exception làm câu function thất bại và transaction không được commit bình thường. Khi có lỗi mạng sau COMMIT, tra pay.requests theo scope/key để lấy trạng thái; không tạo key mới cho retry của cùng ý định. Nếu application coi việc gọi function thành công là đã commit trước khi thực sự COMMIT, vẫn có thể trả thành công sớm sai.

**Giới hạn quyền.** Schema mẫu cho phép user lab trực tiếp sửa table. Vì vậy invariant ledger dựa vào giao thức function, chưa được bảo vệ khỏi user quản trị cố ý hoặc code bypass. Khi chuyển sang sản phẩm, xác định role nào được DML trực tiếp, role nào chỉ gọi routine, ai có quyền migration, và audit thao tác khẩn cấp. Không bổ sung SECURITY DEFINER một cách tùy tiện; đó là ranh giới quyền cần review search_path, ownership và quyền EXECUTE.

**Ánh xạ sang ECB/Prisma.** Service nhận intent đã được authorize, domain xác định amount/currency/state hợp lệ, repository chạy conditional update/locking và ghi ledger bằng transaction client được truyền xuống, provider adapter chỉ làm nhiệm vụ ngoài DB theo workflow bền vững. Không gọi provider trong một interactive transaction kéo dài. Tên API và lỗi Prisma phụ thuộc major; [tài liệu Prisma v6 về transaction](https://www.prisma.io/docs/orm/v6/prisma-client/queries/transactions) là nguồn cho ứng dụng đang dùng v6, không phải bằng chứng rằng dự án hiện tại có đúng version hay cấu hình đó.

---

# Bài 24 — LOCAL/FAULT: duplicate delivery và inbox cùng transaction

**Mục tiêu.** Mô phỏng dispatcher gửi thành công rồi crash trước đánh dấu published. Delivery thứ hai có cùng event_id; hiệu ứng của consumer chỉ tăng một lần. Không cần broker để nhìn thấy invariant DB, và bài không khẳng định đã kiểm chứng cấu hình broker.

**Tiên quyết.** Bài 23 có một event. Cell reset inbox/effect mẫu nhưng giữ transfer/outbox. Trong hệ thống thật, consumer thường có database riêng; vì thế inbox ở đây cố ý không foreign key sang outbox.

```bash {"name":"db-24-outbox-inbox-duplicate","excludeFromRunAll":"true"}
set -Eeuo pipefail
cd "${HOME}/backend-labs/database"
./db.sh <<'SQL'
CREATE TABLE IF NOT EXISTS pay.inbox(
  consumer text NOT NULL,event_id uuid NOT NULL,
  processed_at timestamptz NOT NULL DEFAULT clock_timestamp(),
  PRIMARY KEY(consumer,event_id)
);
CREATE TABLE IF NOT EXISTS pay.consumer_effects(
  consumer text PRIMARY KEY,applied_count integer NOT NULL CHECK(applied_count>=0)
);
TRUNCATE pay.inbox,pay.consumer_effects;
INSERT INTO pay.consumer_effects VALUES('statement-projection',0);
SELECT event_id::text AS event FROM pay.outbox ORDER BY created_at,event_id LIMIT 1 \gset
UPDATE pay.outbox SET published_at=NULL WHERE event_id=:'event'::uuid;

BEGIN;
WITH accepted AS (
  INSERT INTO pay.inbox(consumer,event_id)
  VALUES('statement-projection',:'event'::uuid)
  ON CONFLICT DO NOTHING RETURNING event_id
)
UPDATE pay.consumer_effects SET applied_count=applied_count+1
WHERE consumer='statement-projection' AND EXISTS(SELECT 1 FROM accepted);
COMMIT;

-- Mô phỏng: broker đã chuyển event nhưng dispatcher chưa ghi published_at.
BEGIN;
WITH accepted AS (
  INSERT INTO pay.inbox(consumer,event_id)
  VALUES('statement-projection',:'event'::uuid)
  ON CONFLICT DO NOTHING RETURNING event_id
)
UPDATE pay.consumer_effects SET applied_count=applied_count+1
WHERE consumer='statement-projection' AND EXISTS(SELECT 1 FROM accepted);
COMMIT;
UPDATE pay.outbox SET published_at=clock_timestamp() WHERE event_id=:'event'::uuid;

SELECT lab.assert(
  (SELECT applied_count=1 FROM pay.consumer_effects WHERE consumer='statement-projection'),
  'duplicate event does not apply consumer effect twice');
SELECT lab.assert((SELECT count(*)=1 FROM pay.inbox),'one processed marker');
TABLE pay.consumer_effects;
SQL
```

**Assertion.** Dù hai delivery, applied_count=1. CTE nối việc được nhận mới với hiệu ứng. Marker và hiệu ứng commit cùng nhau: nếu consumer update lỗi, transaction phải rollback marker, để lần sau còn xử lý lại được. Đánh dấu processed trước trong transaction riêng có thể làm mất hiệu ứng vĩnh viễn khi crash sau marker.

**Bảng phân tích crash để dùng khi review:**

| Điểm crash | Trạng thái durable | Hành động sau restart |
|---|---|---|
| Trước commit nghiệp vụ | Chưa có transfer/outbox | Retry cùng key; kiểm tra kết quả đã biết/chưa biết |
| Sau commit, trước publish | Transfer và outbox pending | Dispatcher gửi event pending |
| Sau broker ack, trước published_at | Broker có event, outbox còn pending | Có thể gửi lại cùng event_id |
| Consumer nhận, trước commit hiệu ứng | Chưa có inbox commit | Delivery lại được xử lý |
| Sau commit consumer, trước ack | Inbox và effect đã có | Delivery lại trở thành no-op có kiểm chứng |
| Sau gọi hệ thống ngoài, trước ghi trạng thái | Hiệu ứng ngoài có thể đã xảy ra | Tra theo provider key/reconcile; không đoán từ local timeout |

**Ordering.** Hai dispatcher có thể publish event cùng aggregate ngoài thứ tự nếu không có giao thức bổ sung. Inbox dedup theo event_id chỉ loại lặp; nó không ngăn version3 được áp dụng trước version2. Với projection quan trọng, đưa aggregate id và version vào event, xác định cách xử lý gap/stale event, và có quy trình rebuild. Một event schema thay đổi cần version và tương thích consumer.

**Debug/phục hồi.** Nếu applied_count=2, kiểm tra hiệu ứng có thực sự phụ thuộc row được INSERT mới không. Nếu 0, kiểm tra inbox chưa reset hoặc consumer name khác. Chạy lại cell tái lập chỉ phạm vi inbox/effect. Trên production, retention inbox phải khớp thời gian có thể replay; xóa marker sớm làm duplicate cũ áp dụng lại.

**Bài mở rộng.** Thay counter bằng projection balance theo account và dùng event version. Gửi version2 trước version1, quyết định buffer, retry hay rebuild. Viết một truy vấn đối chiếu tổng transfer đã post với projection, kèm watermark để không coi độ trễ bình thường là mất dữ liệu. Bạn sẽ thấy “consumer idempotent” chỉ là một trong nhiều tính chất cần chứng minh.

---

# Phần VI — Thay đổi cấu trúc và giới hạn tài nguyên

## 1. Partition không phải phép thay thế index hoặc sharding

Partition chia một bảng logic thành nhiều phần vật lý trong cùng hệ quản trị. Sharding phân phối dữ liệu qua nhiều nơi xử lý/lưu trữ với một quy tắc định tuyến. Hai việc có thể cùng tồn tại nhưng khác ranh giới lỗi, query và uniqueness. Ba partition trên cùng đĩa VPS không tạo ba máy độc lập, cũng không tăng băng thông đĩa gấp ba.

Với log có retention theo tháng, partition giúp thao tác theo vòng đời dữ liệu: tạo trước partition mới, detach/archive phần cũ, đo đúng phạm vi query. Nó có thể hỗ trợ pruning nếu predicate khớp key. Nếu đa số query chỉ tìm event_id và không có time, phân theo time có thể khiến query chạm nhiều partition. Vì thế hãy xuất phát từ vòng đời dữ liệu và access pattern, không từ một ngưỡng “50 triệu dòng thì index hết tác dụng”.

Khóa unique/primary của partitioned table trong baseline cần bao gồm các cột partition key theo các điều kiện của PostgreSQL. Unique `(event_id,created_at)` không có nghĩa event_id duy nhất toàn bảng. Nếu idempotency cần global uniqueness, có thể dùng một registry không partition hoặc chọn lại mô hình khóa. Nhiều partition cũng làm catalog/planning/maintenance phức tạp hơn. Các giới hạn được xác định trong [table partitioning PostgreSQL 17](https://www.postgresql.org/docs/17/ddl-partitioning.html).

## 2. Tài nguyên là ngân sách có cạnh tranh

`shared_buffers` là bộ nhớ cache của PostgreSQL. `effective_cache_size` là giả định để planner ước tính cache hữu dụng, không phải vùng RAM được cấp phát. `work_mem` có thể được dùng bởi nhiều operator, nhiều query, nhiều worker cùng lúc; không thể lấy giá trị đó nhân đúng một lần với số connections để tìm upper bound chắc chắn. Hash operation còn có hệ số/bối cảnh riêng. Xem [resource consumption](https://www.postgresql.org/docs/17/runtime-config-resource.html) và [planner cost constants](https://www.postgresql.org/docs/17/runtime-config-query.html).

Một VPS còn chạy Node, Nginx, Redis và agent giám sát cần ngân sách khác database host chuyên dụng. “25% RAM” hoặc “random_page_cost=1.1 trên mọi SSD” chỉ có thể là điểm xuất phát thảo luận, không phải công thức đúng. Bộ nhớ container, OS cache, workload, concurrency và storage latency thực tế mới quyết định cấu hình có hợp lý không.

Hãy thay một nhóm yếu tố nhỏ, đặt cửa sổ đo, ghi cấu hình trước/sau và rollback condition. Nếu tăng work_mem để một báo cáo hết spill, hãy thử `SET LOCAL` cho transaction/report đó trước khi tăng global cho toàn API. Nếu tăng max_connections để hết từ chối, bạn có thể chuyển lỗi rõ thành latency dài và OOM. Admission control và pool queue là công cụ quản lý tải, không chỉ là tối ưu kết nối.

---

# Bài 25 — LOCAL: partition pruning, uniqueness và detach có phục hồi

**Mục tiêu.** Quan sát dữ liệu định tuyến vào tháng, chứng minh khóa ghép không bảo đảm global event id, và detach/attach một phần dữ liệu mà không xóa file của nó.

```bash {"name":"db-25-partitions","excludeFromRunAll":"true"}
set -Eeuo pipefail
cd "${HOME}/backend-labs/database"
./db.sh <<'SQL' | tee evidence/25-partitions.txt
DROP TABLE IF EXISTS lab.audit_events;
DROP TABLE IF EXISTS lab.audit_2026_01;
CREATE TABLE lab.audit_events(
  event_id bigint NOT NULL,
  created_at timestamptz NOT NULL,
  account_id integer NOT NULL,
  payload jsonb NOT NULL,
  PRIMARY KEY(event_id,created_at)
) PARTITION BY RANGE(created_at);
CREATE TABLE lab.audit_2026_01 PARTITION OF lab.audit_events
FOR VALUES FROM ('2026-01-01 00:00:00+00') TO ('2026-02-01 00:00:00+00');
CREATE TABLE lab.audit_2026_02 PARTITION OF lab.audit_events
FOR VALUES FROM ('2026-02-01 00:00:00+00') TO ('2026-03-01 00:00:00+00');
CREATE TABLE lab.audit_2026_03 PARTITION OF lab.audit_events
FOR VALUES FROM ('2026-03-01 00:00:00+00') TO ('2026-04-01 00:00:00+00');
CREATE INDEX audit_account_time_idx ON lab.audit_events(account_id,created_at);
INSERT INTO lab.audit_events VALUES
(1,'2026-01-15 00:00:00+00',1,'{"type":"sample"}'),
(1,'2026-02-15 00:00:00+00',1,'{"type":"duplicate-id-different-time"}'),
(3,'2026-03-15 00:00:00+00',1,'{"type":"sample"}');
SELECT tableoid::regclass,event_id,created_at FROM lab.audit_events ORDER BY created_at;
SELECT lab.assert((SELECT count(*)=2 FROM lab.audit_events WHERE event_id=1),
                  'composite key does not enforce global event_id');
EXPLAIN (ANALYZE,BUFFERS,TIMING OFF)
SELECT * FROM lab.audit_events WHERE account_id=1
AND created_at>='2026-02-01 00:00:00+00'
AND created_at<'2026-03-01 00:00:00+00';
DO $$
BEGIN
  BEGIN
    INSERT INTO lab.audit_events VALUES(9,'2026-04-01 00:00:00+00',1,'{}');
    RAISE EXCEPTION 'unexpected: missing partition accepted';
  EXCEPTION WHEN check_violation THEN
    RAISE NOTICE 'PASS: future partition must be created';
  END;
END $$;
SET lock_timeout='2s';
ALTER TABLE lab.audit_events DETACH PARTITION lab.audit_2026_01;
SELECT lab.assert((SELECT count(*)=2 FROM lab.audit_events),'parent no longer includes January');
SELECT lab.assert((SELECT count(*)=1 FROM lab.audit_2026_01),'detached table still has data');
ALTER TABLE lab.audit_events ATTACH PARTITION lab.audit_2026_01
FOR VALUES FROM ('2026-01-01 00:00:00+00') TO ('2026-02-01 00:00:00+00');
RESET lock_timeout;
SELECT lab.assert((SELECT count(*)=3 FROM lab.audit_events),'January attached back');
SQL
```

**Mong đợi.** Query tháng hai chỉ cần partition liên quan trong plan. Missing future partition gây lỗi được bắt; việc vận hành phải tạo trước, không chờ đến midnight mới phát hiện. Detach làm dữ liệu biến khỏi parent query nhưng còn trong bảng detached; attach khôi phục quan hệ.

**Debug/phục hồi.** Nếu detach timeout, có transaction giữ lock; kiểm tra blocker rồi chạy lại khi kết thúc. Script dừng giữa detach và attach có thể để tháng một tách rời; cell chạy lại chủ động drop bảng detached có tên mẫu trước khi tạo lại. Đây là reset dữ liệu lab, không phải quy trình retention production.

**Biên production.** Detach/drop có chi phí lock, catalog, filesystem và replication; không hứa hoàn tất 5 ms. Trước retention, xác nhận archive restore được, thời gian giữ theo yêu cầu nghiệp vụ và downstream có còn tham chiếu không. `DETACH ... CONCURRENTLY` có giới hạn riêng, không chạy bên trong mọi transaction wrapper. Với default partition, cần giám sát dữ liệu rơi vào đó và khả năng attach partition mới phải validate.

**Bài tập.** Đưa một unique request registry vào bảng không partition rồi cho audit history tham chiếu id đó. Hãy giải thích tại sao registry có vòng đời retention khác audit. Đo query tìm một event theo id với và không có range thời gian trên nhiều partition; đưa số partition chạm vào vào evidence thay vì chỉ ghi execution time.

---

# Bài 26 — LOCAL: pool có hàng đợi, PgBouncer có giới hạn session

## 26A. Hiểu ngân sách connection

Một ứng dụng có 8 API instance và 4 worker, mỗi instance pool tối đa 10, có thể xin tới 120 connection ứng dụng. Thêm migration, monitoring, admin và connection trực tiếp khác. Con số này là upper bound cấu hình, không phải throughput; nếu mỗi transaction giữ connection 2 giây, pool 10 khác hẳn transaction giữ 10 ms.

Pool ở application tái sử dụng connection và giới hạn concurrency của process đó. PgBouncer đứng giữa nhiều process và backend, giúp gom nhu cầu, nhưng database vẫn phải làm cùng công việc SQL. Client nhiều hơn server connection nghĩa là có chờ. Cần đo wait time, queue depth, transaction duration và timeout; tăng max_client_conn không tự tạo thêm năng lực xử lý.

Transaction pooling chỉ gắn một backend trong thời gian transaction. Session state không nên được giả định còn nguyên giữa hai transaction: LISTEN, session advisory lock, temporary table sống qua commit và SQL PREPARE có giới hạn. Protocol prepared statements được hỗ trợ ở các phiên bản/cấu hình thích hợp, khác với SQL PREPARE. [PgBouncer feature map](https://www.pgbouncer.org/features) và [FAQ prepared statements](https://www.pgbouncer.org/faq.html) là nơi đối chiếu chính xác.

## 26B. Dựng PgBouncer từ package hệ điều hành

**Tiên quyết.** Bài 00, Docker build có mạng. Bài tùy chọn dùng thêm RAM và thời gian tải package; VPS nhỏ nên không chạy MySQL đồng thời. Image được build từ Debian trixie với package PgBouncer; kiểm tra version thực tế. Cấu hình cần PgBouncer từ 1.21 trở lên vì bật `max_prepared_statements`. Không giả định đây là image official của dự án PgBouncer.

```bash {"name":"db-26-pgbouncer-setup","excludeFromRunAll":"true"}
set -Eeuo pipefail
cd "${HOME}/backend-labs/database"
chmod 700 .
mkdir -p pgbouncer/runtime
chmod 755 pgbouncer pgbouncer/runtime
cat > pgbouncer/Dockerfile <<'DOCKERFILE'
FROM debian:trixie-slim
RUN apt-get update \
 && apt-get install -y --no-install-recommends pgbouncer postgresql-client ca-certificates \
 && rm -rf /var/lib/apt/lists/*
USER 10001:10001
CMD ["pgbouncer","/etc/pgbouncer/pgbouncer.ini"]
DOCKERFILE
cat > pgbouncer/runtime/pgbouncer.ini <<'INI'
[databases]
banklab = host=postgres port=5432 dbname=banklab pool_size=2

[pgbouncer]
listen_addr = 0.0.0.0
listen_port = 6432
unix_socket_dir = /tmp
auth_type = scram-sha-256
auth_file = /etc/pgbouncer/userlist.txt
admin_users = lab
pool_mode = transaction
max_client_conn = 30
default_pool_size = 2
reserve_pool_size = 0
max_db_connections = 4
max_prepared_statements = 100
server_connect_timeout = 5
query_wait_timeout = 10
pidfile = /tmp/pgbouncer.pid
INI
set -a
source ./.env
set +a
printf '"lab" "%s"\n' "$DB_LAB_PASSWORD" > pgbouncer/runtime/userlist.txt
chmod 644 pgbouncer/runtime/pgbouncer.ini pgbouncer/runtime/userlist.txt
cat > compose.pool.yaml <<'YAML'
services:
  pgbouncer:
    build: ./pgbouncer
    image: be-db-pgbouncer:lab
    ports:
      - "127.0.0.1:16432:6432"
    volumes:
      - ./pgbouncer/runtime:/etc/pgbouncer:ro
    depends_on:
      postgres:
        condition: service_healthy
    mem_limit: 128m
    healthcheck:
      test: ["CMD","pg_isready","-h","127.0.0.1","-p","6432","-U","lab","-d","banklab"]
      interval: 2s
      timeout: 3s
      retries: 30
YAML
docker compose -f compose.yaml -f compose.pool.yaml config --quiet
docker compose -f compose.yaml -f compose.pool.yaml up -d --build --wait pgbouncer
docker compose -f compose.yaml -f compose.pool.yaml exec -T pgbouncer pgbouncer --version
```

Thư mục gốc lab mode 700 bảo vệ các file cấu hình khỏi user host khác; file bên trong mount đọc được bởi uid không đặc quyền của container. Đây vẫn là secret plaintext cho lab. Production cần cơ chế secret phù hợp, xoay vòng credential, TLS theo ranh giới mạng và role tách biệt; không chuyển nguyên user admin lab sang ứng dụng.

## 26C. Quan sát client thứ ba chờ hai backend

**Ý định.** A và B giữ hai transaction. C gửi query và phải chờ. Admin connection đọc SHOW POOLS, chỉ thả A sau khi đã thấy `cl_waiting` tăng.

```bash {"name":"db-26-pgbouncer-queue","excludeFromRunAll":"true"}
set -Eeuo pipefail
cd "${HOME}/backend-labs/database"
cat > scripts/pool.mjs <<'JS'
import pg from 'pg';
import assert from 'node:assert/strict';
import {setTimeout as delay} from 'node:timers/promises';
const config={host:'pgbouncer',port:6432,connectionTimeoutMillis:5000};
const a=new pg.Client(config),b=new pg.Client(config),c=new pg.Client(config);
const admin=new pg.Client({...config,database:'pgbouncer'});
const clients=[a,b,c,admin];
try {
  for (const client of clients) await client.connect();
  await a.query('BEGIN');
  const pa=(await a.query('SELECT pg_backend_pid() AS pid')).rows[0].pid;
  await b.query('BEGIN');
  const pb=(await b.query('SELECT pg_backend_pid() AS pid')).rows[0].pid;
  assert.notEqual(pa,pb);
  const pending=c.query('SELECT 42 AS answer').then(
    value=>({ok:true,value}),error=>({ok:false,error}));
  let queued=false;
  const deadline=Date.now()+5000;
  while(Date.now()<deadline) {
    const pools=(await admin.query('SHOW POOLS')).rows;
    const relevant=pools.filter(r=>r.database==='banklab');
    if(relevant.some(r=>Number(r.cl_waiting)>=1)) {
      queued=true;
      console.table(relevant.map(({database,user,cl_active,cl_waiting,sv_active,sv_idle})=>
        ({database,user,cl_active,cl_waiting,sv_active,sv_idle})));
      break;
    }
    await delay(25);
  }
  assert.equal(queued,true,'third client must be observed queued');
  await a.query('COMMIT');
  const result=await pending;
  assert.equal(result.ok,true);
  assert.equal(result.value.rows[0].answer,42);
  await b.query('COMMIT');
  for(const client of [a,b,c]) {
    const prepared=await client.query({
      name:'db_lab_increment',
      text:'SELECT $1::int+1 AS n',
      values:[41]
    });
    assert.equal(prepared.rows[0].n,42);
  }
  console.log('Observed queue and successful protocol prepared statements');
} finally {
  await Promise.allSettled([a,b,c].map(client=>client.query('ROLLBACK')));
  await Promise.allSettled(clients.map(client=>client.end()));
}
JS
docker compose -f compose.yaml -f compose.pool.yaml run --rm toolbox scripts/pool.mjs
docker compose -f compose.yaml -f compose.pool.yaml stop pgbouncer
```

**Mong đợi.** Hai backend khác nhau, C xuất hiện trong cl_waiting, sau A commit C có kết quả. Đó là admission control có hàng đợi, không phải database xử lý ba transaction đồng thời bằng hai backend. Named protocol statements chạy được với cấu hình hiện tại; không suy ra SQL PREPARE hay mọi feature session cũng chạy.

**Debug.** Nếu không queue, kiểm tra pool_size thực, reserve pool và có dùng đúng endpoint pgbouncer không. Nếu auth lỗi, so password nguồn với volume PostgreSQL đã khởi tạo, không đổi auth_type=trust để bỏ qua. Nếu max_prepared_statements không được nhận, version package không đạt điều kiện; ghi lại phiên bản và cập nhật image hoặc dùng lab không prepared theo phiên bản phù hợp.

**Biên production.** Pool mặc định thường tính theo cặp database/user, không phải một trần toàn cụm duy nhất; giới hạn tổng cần đọc cấu hình [PgBouncer](https://www.pgbouncer.org/config.html). Transaction pooling không thích hợp cho mọi migration tool và session feature; route migration/admin trực tiếp có kiểm soát khi cần. SET LOCAL trong transaction phù hợp hơn việc giữ session setting rồi giả định backend sau vẫn là backend trước.

---

# Bài 27 — LOCAL/FAULT: expand–contract, backfill theo batch và lock DDL

## Vì sao không đổi schema và application trong một cú nhảy?

Trong rolling deployment, code cũ và mới cùng tồn tại. Nếu đổi tên cột rồi mới deploy, code cũ lỗi. Nếu deploy trước mà cột mới chưa có, code mới lỗi. Expand tạo khả năng tương thích trước; migrate data và chuyển read/write có kiểm chứng; contract bỏ phần cũ sau khi không còn phụ thuộc.

Một chuỗi khả thi: thêm cột mới nullable; deploy writer ghi cả cũ/mới; backfill có điều kiện; kiểm tra không còn NULL và sự tương đương; chuyển reader sang mới; quan sát và xác nhận không còn code cũ; thêm constraint cần thiết; cuối cùng bỏ cột cũ. Nếu các writer cũ còn tồn tại, backfill một lần chưa bảo đảm sau đó không xuất hiện row mới thiếu cột mới.

Backfill cần tránh ghi đè cập nhật mới. Với mapping một-một từ trạng thái cũ trong lab, UPDATE chỉ chọn row status_code IS NULL, và lấy giá trị legacy_status tại lúc row được update. Đối với biến đổi phức tạp, cần version/watermark hoặc quy tắc conflict. Giảm batch giúp giới hạn lock, WAL và thời gian mỗi commit, nhưng không thay thế quan sát tải.

## 27A. Tạo schema cũ, mở rộng và backfill

```bash {"name":"db-27-expand-backfill","excludeFromRunAll":"true"}
set -Eeuo pipefail
cd "${HOME}/backend-labs/database"
./db.sh <<'SQL'
DROP TABLE IF EXISTS lab.migrate_orders;
CREATE TABLE lab.migrate_orders(id bigint PRIMARY KEY,legacy_status text NOT NULL);
INSERT INTO lab.migrate_orders
SELECT i,CASE WHEN i%10=0 THEN 'PENDING' ELSE 'COMPLETED' END
FROM generate_series(1,5000) i;
SET lock_timeout='2s';
ALTER TABLE lab.migrate_orders ADD COLUMN status_code text;
RESET lock_timeout;
-- Mô phỏng writer mới ghi cả hai cột.
INSERT INTO lab.migrate_orders VALUES(5001,'COMPLETED','COMPLETED');
SQL
cat > scripts/backfill.mjs <<'JS'
import {run,assert} from './harness.mjs';
await run(async a=>{
  let batches=0,updated=0;
  while(batches<100) {
    const result=await a.query(
      'WITH batch AS (SELECT id FROM lab.migrate_orders WHERE status_code IS NULL ORDER BY id LIMIT 500 FOR UPDATE SKIP LOCKED) UPDATE lab.migrate_orders m SET status_code=m.legacy_status FROM batch WHERE m.id=batch.id AND m.status_code IS NULL'
    );
    if(result.rowCount===0) break;
    updated+=result.rowCount;
    batches++;
  }
  const check=(await a.query(
    'SELECT count(*)::int AS n FROM lab.migrate_orders WHERE status_code IS NULL OR status_code<>legacy_status'
  )).rows[0].n;
  assert.equal(check,0,'no missing or inconsistent migration values');
  console.log({batches,updated});
});
JS
docker compose run --rm toolbox scripts/backfill.mjs
./db.sh <<'SQL'
SET lock_timeout='2s';
ALTER TABLE lab.migrate_orders
ADD CONSTRAINT migrate_status_present CHECK(status_code IS NOT NULL) NOT VALID;
ALTER TABLE lab.migrate_orders VALIDATE CONSTRAINT migrate_status_present;
ALTER TABLE lab.migrate_orders ALTER COLUMN status_code SET NOT NULL;
RESET lock_timeout;
CREATE INDEX CONCURRENTLY migrate_status_idx ON lab.migrate_orders(status_code,id);
SELECT lab.assert(
  (SELECT indisvalid AND indisready FROM pg_index WHERE indexrelid='lab.migrate_status_idx'::regclass),
  'new index is valid and ready');
SQL
```

**Mong đợi.** Mỗi batch UPDATE là một transaction autocommit riêng; không bọc cả vòng lặp trong BEGIN hoặc DO để rồi vô tình giữ một transaction lớn. 5000 rows cần backfill, row5001 đã được writer mới điền. Final assertion kiểm tra còn thiếu, vì batch trả 0 có thể chỉ là toàn bộ row đang bị worker khác khóa nếu chạy nhiều worker.

**Đọc constraint.** NOT VALID không có nghĩa bỏ qua mọi write mới; nó tách việc xác nhận dữ liệu cũ khỏi việc áp constraint lên thay đổi tiếp theo. Lệnh ALTER có lock khác nhau theo loại và phiên bản. [ALTER TABLE](https://www.postgresql.org/docs/17/sql-altertable.html) là nguồn xác định chính xác, không suy ra từ tên “online”.

`CREATE INDEX CONCURRENTLY` không được đặt trong transaction block và có thể thất bại để lại index invalid. Dùng [CREATE INDEX](https://www.postgresql.org/docs/17/sql-createindex.html) để hiểu các bước và giới hạn. IF NOT EXISTS chỉ kiểm tra tên tồn tại, không chứng minh index cùng định nghĩa hoặc còn valid.

## 27B. DDL bị một transaction đọc chặn

```bash {"name":"db-27-ddl-lock-timeout","excludeFromRunAll":"true"}
set -Eeuo pipefail
cd "${HOME}/backend-labs/database"
cat > scripts/ddl-lock.mjs <<'JS'
import {run,assert,capture,waitBlocked} from './harness.mjs';
await run(async(a,b,o,[pidA,pidB])=>{
  await a.query('BEGIN');
  await a.query('SELECT count(*) FROM lab.migrate_orders');
  await b.query("SET lock_timeout='1500ms'");
  const pending=capture(b.query('ALTER TABLE lab.migrate_orders ADD COLUMN migration_note text'));
  await waitBlocked(o,pidB,pidA);
  const outcome=await pending;
  assert.equal(outcome.ok,false);
  assert.equal(outcome.error.code,'55P03');
  await a.query('ROLLBACK');
  await b.query('ALTER TABLE lab.migrate_orders ADD COLUMN migration_note text');
  console.log('DDL timed out with evidence, then succeeded after blocker ended');
});
JS
docker compose run --rm toolbox scripts/ddl-lock.mjs
```

**Kết quả.** Một SELECT trong transaction còn mở có thể chặn DDL. Lock timeout giúp migration thất bại có giới hạn thay vì giữ hàng đợi lâu. Khi retry bài riêng, nếu migration_note đã tồn tại, chạy lại 27A trước; đó là trạng thái dữ liệu, không phải lỗi driver.

**Contract và rollback.** Chưa DROP legacy_status trong cell tự động vì mục tiêu là chứng minh khả năng cùng tồn tại. Trong sản phẩm, chỉ contract sau khi mọi reader/writer/job/export đã chuyển. Rollback application khi cột cũ còn được duy trì dễ hơn rollback sau khi đã bỏ. Nếu một migration làm mất thông tin, “rollback migration” không tái tạo được thông tin ấy; cần backup/restore hoặc forward fix có dữ liệu hỗ trợ.

**Biên production.** Theo dõi replication lag, WAL, độ tuổi transaction và error rate trong backfill. Tạm dừng batch nên là hành vi bình thường, không phải thất bại. Với PlanetScale, dùng quy trình deploy request/safe migration phù hợp service; không copy lệnh PostgreSQL hoặc giả định migration ORM chạy thẳng DDL trên nhánh bảo vệ.

---

# Bài 28 — LOCAL/VPS: bộ chẩn đoán trước khi thay cấu hình

**Mục tiêu.** Thu bằng chứng về SQL, lock, transaction dài, connection, WAL và không gian. Bài chỉ đọc; không tự cancel query, xóa index, vacuum full hay tăng bộ nhớ.

**Tiên quyết.** Bài 00–02; role lab có quyền xem các view. Trong production, cấp quyền monitoring phù hợp thay vì dùng superuser cho dashboard. Query text/log có thể chứa dữ liệu nhạy cảm; kiểm tra và làm sạch trước khi chia sẻ evidence.

```bash {"name":"db-28-diagnostics","excludeFromRunAll":"true"}
set -Eeuo pipefail
cd "${HOME}/backend-labs/database"
cat > scripts/diagnose.sql <<'SQL'
SELECT clock_timestamp() AS collected_at,version();
SELECT datname,numbackends,xact_commit,xact_rollback,deadlocks,temp_files,temp_bytes,
       blk_read_time,blk_write_time,stats_reset
FROM pg_stat_database WHERE datname=current_database();

SELECT pid,application_name,state,wait_event_type,wait_event,
       clock_timestamp()-xact_start AS transaction_age,
       clock_timestamp()-query_start AS query_age,
       backend_xmin,pg_blocking_pids(pid) AS blockers,left(query,160) AS query
FROM pg_stat_activity
WHERE datname=current_database() AND pid<>pg_backend_pid()
ORDER BY xact_start NULLS LAST;

SELECT state,count(*) FROM pg_stat_activity
WHERE datname=current_database() GROUP BY state ORDER BY state;

SELECT queryid,calls,round(total_exec_time::numeric,2) AS total_ms,
       round(mean_exec_time::numeric,2) AS mean_ms,rows,
       shared_blks_hit,shared_blks_read,temp_blks_written,
       left(query,160) AS normalized_query
FROM pg_stat_statements
WHERE dbid=(SELECT oid FROM pg_database WHERE datname=current_database())
ORDER BY total_exec_time DESC LIMIT 10;

SELECT schemaname,relname,n_live_tup,n_dead_tup,n_mod_since_analyze,
       last_autovacuum,last_autoanalyze,last_vacuum,last_analyze
FROM pg_stat_user_tables ORDER BY n_dead_tup DESC LIMIT 15;

SELECT s.schemaname,s.relname,s.indexrelname,s.idx_scan,
       pg_size_pretty(pg_relation_size(s.indexrelid)) AS size,
       i.indisunique,i.indisprimary,i.indisvalid
FROM pg_stat_user_indexes s JOIN pg_index i ON i.indexrelid=s.indexrelid
ORDER BY pg_relation_size(s.indexrelid) DESC LIMIT 15;

SELECT sum(heap_blks_hit) AS shared_buffer_hits,sum(heap_blks_read) AS shared_buffer_reads,
       round(100.0*sum(heap_blks_hit)/
         nullif(sum(heap_blks_hit)+sum(heap_blks_read),0),2) AS heap_hit_percent
FROM pg_statio_user_tables;

SELECT n.nspname,c.relname,age(c.relfrozenxid) AS xid_age,
       pg_size_pretty(pg_total_relation_size(c.oid)) AS total_size
FROM pg_class c JOIN pg_namespace n ON n.oid=c.relnamespace
WHERE c.relkind='r' AND n.nspname IN ('lab','pay')
ORDER BY age(c.relfrozenxid) DESC LIMIT 15;

SELECT wal_records,wal_fpi,wal_bytes,wal_buffers_full,stats_reset FROM pg_stat_wal;
SELECT num_timed,num_requested,write_time,sync_time,stats_reset FROM pg_stat_checkpointer;
SELECT slot_name,slot_type,active,restart_lsn,
       pg_wal_lsn_diff(pg_current_wal_lsn(),restart_lsn) AS retained_wal_bytes
FROM pg_replication_slots;
SQL
./db.sh < scripts/diagnose.sql | tee evidence/28-diagnostics.txt
docker stats --no-stream "$(docker compose ps -q postgres)"
docker compose exec -T postgres df -h /var/lib/postgresql/data
docker compose logs --since 10m --tail 100 postgres > evidence/28-postgres.log
```

**Cách đọc có trách nhiệm.** total_exec_time là thời gian execution tích lũy, không phải riêng CPU. Một query nhanh nhưng gọi hàng triệu lần có thể đứng đầu tổng. Mean che được tail; cần histogram từ application hoặc quan sát query cụ thể để đánh giá p95/p99. [pg_stat_statements](https://www.postgresql.org/docs/17/pgstatstatements.html) nêu cách module thu thập và chuẩn hóa; thống kê này không thay cho trace một request.

**So delta.** Counter tích lũy từ stats_reset không nói riêng sự cố 5 phút vừa rồi. Chụp trước/sau, so chênh lệch và khoảng thời gian; đừng reset statistics production để dễ đọc nếu dashboard/điều tra khác còn cần dữ liệu. Với query plan regression, giữ queryid, parameters đã che thông tin nhạy cảm, plan, deployment/data-change time và server version.

**Bloat.** n_dead_tup là ước lượng tuple chết, không phải số byte bloat chính xác. Một bảng có nhiều free space tái sử dụng không nhất thiết cần rewrite. So kích thước với lịch sử, tốc độ thay đổi, vacuum và long snapshot; cân nhắc công cụ đo chuyên dụng trên cửa sổ phù hợp nếu cần ước lượng sâu hơn. Chỉ “thấy bảng lớn” chưa đủ lý do VACUUM FULL.

**Connection saturation.** Nhiều connection idle có thể phản ánh pool cấu hình lớn, nhưng idle in transaction nguy hiểm hơn vì giữ transaction/snapshot/lock. Active nhiều và CPU thấp có thể là lock/I/O/network wait. Pool wait cao nhưng backend ít active có thể do connection bị giữ ở code trong khi chờ HTTP. Tách các trường hợp trước khi tăng max_connections.

**Cache ratio.** Không đặt ngưỡng 99% cho mọi workload. Một scan phân tích lớn có hành vi khác OLTP key lookup. Buffer read còn có thể hit OS cache. Hãy kết hợp với latency, I/O, query plan và SLO. Tỷ lệ đẹp không chứng minh người dùng đang được phục vụ nhanh.

**Phục hồi.** Bài không thay đổi trạng thái. Muốn thực hành blocker, chạy Bài 19/21 trong terminal khác và quan sát bằng file diagnose; đừng giữ workload lỗi vô hạn. Trong incident thật, ghi PID và xact_start trước khi can thiệp vì PID có thể được tái sử dụng theo thời gian.

---

# Bài 29 — LOCAL/VPS: backup chỉ được tính sau khi restore thành công

## 29A. Chọn đúng loại bản sao

Logical dump mô tả schema/data ở mức logic; phù hợp nhiều bài restore có chọn lọc và chuyển môi trường. Physical base backup cùng chuỗi WAL phục vụ khôi phục cluster và PITR. Snapshot disk cần một quy trình nhất quán phù hợp engine. Replication giúp duy trì một bản sao theo tiến trình thay đổi, nhưng lệnh xóa sai thường cũng được replicate; nó không thay retention backup.

[pg_dump](https://www.postgresql.org/docs/17/app-pgdump.html) tạo snapshot logic nhất quán của một database, kể cả khi có writer, nhưng không tự gồm mọi role/global object. [pg_restore](https://www.postgresql.org/docs/17/app-pgrestore.html) phục hồi archive và có tùy chọn dừng khi lỗi. Trong bài, dùng tool cùng major với server từ chính container để tránh lệch tool/server, và restore vào database khác tên.

**Tiên quyết.** Bài 02 và Bài 23. Dừng các bài ghi, fault và background workload trong project trong thời gian chụp manifest + dump. Dump tự nhất quán, nhưng manifest được query riêng; yêu cầu tạm dừng writer làm hai ảnh tương ứng để đối chiếu. Production có thể cần exported snapshot hoặc chiến lược kiểm chứng khác thay vì giả định hai lệnh riêng cùng thời điểm.

```bash {"name":"db-29-backup","excludeFromRunAll":"true"}
set -Eeuo pipefail
cd "${HOME}/backend-labs/database"
umask 077
mkdir -p backups
cat > scripts/manifest.sql <<'SQL'
SELECT jsonb_build_object(
  'orders_count',(SELECT count(*)::text FROM lab.orders),
  'orders_id_sum',(SELECT sum(id)::text FROM lab.orders),
  'orders_amount_sum',(SELECT sum(amount_minor)::text FROM lab.orders),
  'accounts',(SELECT jsonb_agg(to_jsonb(a) ORDER BY scope,id) FROM pay.accounts a),
  'transfers_count',(SELECT count(*)::text FROM pay.transfers),
  'entries_count',(SELECT count(*)::text FROM pay.entries),
  'entries_sum',(SELECT sum(delta_minor)::text FROM pay.entries),
  'requests_count',(SELECT count(*)::text FROM pay.requests),
  'outbox_count',(SELECT count(*)::text FROM pay.outbox)
);
SQL
db_backup_prefix="backups/banklab-$(date -u +%Y%m%dT%H%M%SZ)-$$"
./db.sh -At < scripts/manifest.sql > "$db_backup_prefix.manifest.json"
docker compose exec -T postgres \
  pg_dump -U lab -d banklab --format=custom \
  > "$db_backup_prefix.dump"
docker compose exec -T postgres \
  pg_dumpall -U lab --globals-only --no-role-passwords \
  > "$db_backup_prefix.globals.sql"
test -s "$db_backup_prefix.dump"
docker compose exec -T postgres pg_restore --list \
  < "$db_backup_prefix.dump" > "$db_backup_prefix.toc.txt"
docker compose run --rm --entrypoint sha256sum toolbox \
  "$db_backup_prefix.dump" "$db_backup_prefix.manifest.json" \
  "$db_backup_prefix.globals.sql" > "$db_backup_prefix.sha256"
printf '%s\n' "$db_backup_prefix" > backups/latest.path
printf 'Backup artifact prefix: %s\n' "$db_backup_prefix"
wc -c "$db_backup_prefix.dump"
```

**Kết quả cần có.** Dump không rỗng, TOC đọc được, manifest có accounts và counts, checksum được lưu. Đây mới là kiểm tra archive có cấu trúc; chưa chứng minh restore được. Checksum chạy trong toolbox để không phụ thuộc sha256sum có sẵn trên host macOS. File globals bỏ password hash trong bài và không tự replay vào cluster hiện tại để tránh đụng role sẵn có. Production phải có quy trình khôi phục credential/quyền/tablespace/extensions/cấu hình thích hợp, không chỉ một file data.

## 29B. RESET một database đích riêng, restore và kiểm tra

**Phạm vi RESET:** chỉ database `lab_restore` trong container thuộc project `be-db`. Không DROP banklab. Nếu bạn đã dùng lab_restore cho công việc khác, đổi tên đích và review mọi chỗ sử dụng trước khi chạy.

```bash {"name":"db-29-restore-and-verify","excludeFromRunAll":"true"}
set -Eeuo pipefail
cd "${HOME}/backend-labs/database"
db_backup_prefix="$(cat backups/latest.path)"
if [[ ! "$db_backup_prefix" =~ ^backups/banklab-[0-9]{8}T[0-9]{6}Z-[0-9]+$ ]]; then
  echo "Unexpected backup path; inspect backups/latest.path." >&2
  exit 2
fi
docker compose run --rm --entrypoint sha256sum toolbox -c "$db_backup_prefix.sha256"
docker compose exec -T postgres dropdb -U lab --if-exists --force lab_restore
docker compose exec -T postgres createdb -U lab --template=template0 lab_restore
db_restore_started="$(date +%s)"
docker compose exec -T postgres \
  pg_restore -U lab --dbname=lab_restore --exit-on-error --no-owner --no-privileges \
  < "$db_backup_prefix.dump"
docker compose exec -T postgres \
  psql -X -U lab -d lab_restore -v ON_ERROR_STOP=1 -At \
  < scripts/manifest.sql > "$db_backup_prefix.restored-manifest.json"
diff -u "$db_backup_prefix.manifest.json" "$db_backup_prefix.restored-manifest.json"
docker compose exec -T postgres \
  psql -X -U lab -d lab_restore -v ON_ERROR_STOP=1 -P pager=off <<'SQL'
ANALYZE;
SELECT lab.assert(
  NOT EXISTS (
    SELECT transfer_id,currency FROM pay.entries
    GROUP BY transfer_id,currency HAVING sum(delta_minor)<>0
  ),'restored ledger balances');
SELECT max(id) AS before_id FROM lab.orders \gset
BEGIN;
INSERT INTO lab.orders(customer_id,status,amount_minor,currency,created_at,metadata)
SELECT customer_id,status,amount_minor,currency,created_at,metadata
FROM lab.orders ORDER BY id LIMIT 1
RETURNING id AS restored_next_id \gset
SELECT lab.assert(:restored_next_id::bigint > :before_id::bigint,
                  'identity sequence restored beyond existing maximum');
ROLLBACK;
SELECT current_database(),count(*) AS transfer_count FROM pay.transfers;
SQL
db_restore_finished="$(date +%s)"
printf 'Restore plus checks: %s seconds\n' "$((db_restore_finished-db_restore_started))" \
  | tee evidence/29-restore-duration.txt
```

**Semantic assertions.** Manifest trước/sau giống, ledger không lệch, sequence tạo id vượt max trước. Test INSERT nằm trong rollback trên database đích; sequence có thể tiến dù rollback, và đó không phải lỗi. Kiểm tra count/sum không chứng minh byte-for-byte mọi row; với dữ liệu quan trọng, bổ sung đối chiếu chi tiết/checksum logic theo phạm vi, ràng buộc và smoke test application.

**Debug.** Nếu extension thiếu, đọc lỗi đầu tiên; không bỏ qua rồi coi restore “gần xong”. Nếu checksum sai, kiểm tra truyền file và storage; không đổi checksum để cho qua. Nếu role/owner lỗi ở môi trường khác, quyết định mapping quyền rõ ràng; --no-owner/--no-privileges trong lab cố ý tránh trộn quyền nguồn với cluster thử nghiệm, không phải cấu hình recovery production mặc định.

**Đưa backup khỏi miền lỗi.** File trong cùng VPS mất cùng VPS. Sau khi kiểm chứng, sao chép tới storage độc lập với quyền và retention phù hợp, mã hóa theo chính sách và thử restore từ **bản đã tải về từ storage đó**. Không đủ nếu chỉ restore file local rồi cho rằng đường upload/download cũng tốt. Kiểm tra tuổi bản backup mới nhất, dung lượng bất thường, checksum, khả năng giải mã và thời gian khôi phục định kỳ.

**RPO/RTO.** RPO là mức mất dữ liệu được chấp nhận khi xảy ra lỗi theo scenario đã chọn; RTO là thời gian khôi phục dịch vụ theo tiêu chí đã định. “Backup mỗi đêm” không tự bảo đảm RPO 24 giờ nếu backup thất bại hai ngày mà không ai biết. RTO gồm lấy file, chuẩn bị máy, restore, apply WAL nếu có, kiểm chứng, chuyển traffic và xử lý các hệ thống liên quan; số giây của cell chỉ là một thành phần.

---

# Chuyên đề — PITR và replication: kiến trúc đã giải thích, chưa tự triển khai

**Phân loại rõ:** phần này là hướng dẫn thiết kế và kiểm chứng, **không phải lab PITR đã chạy hoàn chỉnh**. Notebook không cấu hình archive_command, kho archive hoặc standby và không tuyên bố đã kiểm chứng recovery tới một timestamp. Bài 29 là lab logical backup/restore có thể thực thi; không lấy file pg_dump ghép với WAL để làm PITR.

Theo [continuous archiving PostgreSQL 17](https://www.postgresql.org/docs/17/continuous-archiving.html), PITR cần physical base backup và các WAL cần thiết liên tục đến mục tiêu phục hồi. Phải có cả điểm bắt đầu hợp lệ, chuỗi WAL không đứt và cấu hình recovery đúng. Dump logic không mang thông tin cần thiết cho WAL replay. Đổi tên một file dump thành “base backup” không thay đổi điều đó.

## Các tiên quyết để biến thành lab PITR thật

1. **Môi trường đích độc lập.** Một cluster thử nghiệm riêng với data directory/volume riêng, cùng major và tổ hợp extension/tablespace thích hợp. Không thử restore đè lên volume đang hoạt động.
2. **Archiving hoạt động trước sự cố.** Cấu hình WAL level và archive mode đúng; archive destination bền vững; command phải chỉ báo thành công khi file được lưu an toàn và phải chịu retry. Theo dõi archive failures và tuổi WAL archive gần nhất.
3. **Physical base backup được kiểm chứng.** Dùng công cụ phù hợp và ghi manifest/thời điểm/LSN. Backup phải giữ đủ WAL theo chế độ/công cụ; file copy tùy ý khi server đang ghi không tự thành base backup hợp lệ.
4. **Retention phối hợp.** Không xóa WAL mà base backup còn cần. Không giữ một replication slot bỏ quên đến mức lấp đĩa. Theo dõi khoảng LSN giữ lại và dung lượng thực.
5. **Mục tiêu phục hồi không mơ hồ.** UTC timestamp, restore point hoặc transaction/LSN theo thiết kế. Phân biệt inclusive/exclusive và timezone. Với sai sót thao tác, cần một điểm trước giao dịch gây hại.
6. **Recovery config và timeline.** Khôi phục vào volume trống, cấu hình restore command và recovery target, dùng tín hiệu recovery đúng phiên bản, đọc log xem đã tới mục tiêu hay dừng do thiếu WAL.
7. **Kiểm tra nghiệp vụ.** Giao dịch trước mục tiêu có, sau mục tiêu chưa có; invariant ledger còn đúng; ứng dụng khởi động; credential và quyền đúng.
8. **Kế hoạch đưa trở lại phục vụ.** Sau promote sẽ có timeline mới; quản lý archive, standby và tránh hai primary cùng nhận ghi. Reconcile với broker/provider vì khôi phục DB về quá khứ không làm các hệ thống ngoài quay ngược.

Các bước trên có thể dùng để viết một lab nâng cao riêng khi bạn có storage/host đích và công cụ backup lựa chọn rõ ràng. Không cung cấp một dòng shell “PITR xong” nếu những tiên quyết đó chưa tồn tại. Trong capstone, sản phẩm đầu ra của phần này là một thiết kế recovery và kế hoạch diễn tập có tiêu chí pass/fail, không phải một dấu tick vận hành.

## Replication không tự giải quyết failover

[Warm standby documentation](https://www.postgresql.org/docs/17/warm-standby.html) phân biệt vận chuyển/replay WAL và các lựa chọn synchronous/asynchronous. Async có thể mất các commit chưa đến standby khi primary mất; sync thay đổi điểm chờ và đánh đổi availability/latency theo cấu hình. Không chỉ bật “sync” rồi kết luận mọi hình thức mất dữ liệu đã được loại bỏ.

Read replica có thể trễ. Sau một API ghi thành công, truy vấn ngay replica có thể chưa thấy; sản phẩm cần read-your-writes strategy, routing hoặc chấp nhận eventual consistency rõ ràng. Một kiểm tra idempotency từ replica trễ cũng có thể trả “chưa có” dù primary đã commit. Các quyết định tài chính yêu cầu trạng thái mới nhất cần đi đến nguồn có bảo đảm phù hợp.

Hai container primary/replica trên cùng VPS giúp học giao thức nhưng vẫn chung nguồn điện, host, kernel và thường chung storage. Chúng không tạo high availability cho lỗi mất VPS. Failover còn cần phát hiện lỗi chính xác, fencing primary cũ, chọn node đủ mới, chuyển endpoint, reconnect client và reconcile trạng thái chưa biết. Split brain có thể nguy hiểm hơn một khoảng downtime có kiểm soát.

---

# Bài 30 — FAULT: crash process với transaction chưa commit

**Mục tiêu.** Kill PostgreSQL trong project học tập khi có một transaction đang ghi, rồi kiểm tra dữ liệu commit trước còn tồn tại và phần chưa commit không xuất hiện. Đây là crash process trên storage còn nguyên, không mô phỏng mất ổ đĩa hoặc backup hỏng.

**Tiên quyết.** Bài 00–01. Dừng PgBouncer/MySQL/workload khác nếu không cần; bài làm mất mọi connection tới postgres trong project. Không dùng trên production. Signal SIGKILL chỉ nhắm service postgres của `be-db`.

```bash {"name":"db-30-crash-recovery","excludeFromRunAll":"true"}
set -Eeuo pipefail
cd "${HOME}/backend-labs/database"
./db.sh <<'SQL'
DROP TABLE IF EXISTS lab.crash_marker;
CREATE TABLE lab.crash_marker(id integer PRIMARY KEY,value integer NOT NULL);
INSERT INTO lab.crash_marker VALUES(1,10);
SQL
cat > scripts/crash-open.mjs <<'JS'
import pg from 'pg';
import {writeFile} from 'node:fs/promises';
const client=new pg.Client({application_name:'db-crash-lab',connectionTimeoutMillis:5000});
let timer;
let disconnected;
const lost=new Promise(resolve=>{disconnected=resolve;});
client.on('error',error=>disconnected(error));
client.on('end',()=>disconnected(new Error('connection ended')));
try {
  await client.connect();
  await client.query('BEGIN');
  await client.query('UPDATE lab.crash_marker SET value=99 WHERE id=1');
  await client.query('INSERT INTO lab.crash_marker VALUES(2,99)');
  await writeFile('/lab/state/crash.ready','transaction-written-not-committed\n');
  await Promise.race([
    lost,
    new Promise((_,reject)=>{timer=setTimeout(()=>reject(new Error('No crash observed')),30000);})
  ]);
  console.log('Connection lost after ready barrier; transaction was not committed');
} finally {
  clearTimeout(timer);
  await client.end().catch(()=>{});
}
JS
rm -f state/crash.ready
docker compose run --rm toolbox scripts/crash-open.mjs > evidence/30-client.log 2>&1 &
db_crash_client_pid=$!
db_crash_ready=false
for ((db_crash_attempt=1; db_crash_attempt<=150; db_crash_attempt++)); do
  if [ -f state/crash.ready ]; then db_crash_ready=true; break; fi
  if ! kill -0 "$db_crash_client_pid" 2>/dev/null; then
    cat evidence/30-client.log
    exit 1
  fi
  sleep 0.1
done
if [ "$db_crash_ready" != true ]; then
  echo "No ready barrier; database has not been killed." >&2
  wait "$db_crash_client_pid"
  exit 1
fi
db_crash_restore_on_exit() {
  db_crash_exit_status=$?
  trap - EXIT INT TERM
  if ! docker compose up -d --wait --wait-timeout 120 postgres; then
    echo "Automatic recovery failed; inspect postgres logs without deleting the volume." >&2
    exit 1
  fi
  exit "$db_crash_exit_status"
}
trap db_crash_restore_on_exit EXIT
trap 'exit 130' INT
trap 'exit 143' TERM
docker compose kill -s SIGKILL postgres
wait "$db_crash_client_pid"
docker compose up -d --wait --wait-timeout 120 postgres
trap - EXIT INT TERM
./db.sh <<'SQL'
SELECT lab.assert((SELECT count(*)=1 FROM lab.crash_marker),'uncommitted insert absent');
SELECT lab.assert((SELECT value=10 FROM lab.crash_marker WHERE id=1),'committed value recovered');
TABLE lab.crash_marker;
SQL
docker compose logs --since 5m --tail 120 postgres > evidence/30-recovery.log
cat evidence/30-client.log
```

**Mong đợi.** Ready file chỉ được ghi sau hai statement chưa commit hoàn tất. Sau crash/restart, chỉ id1/value10 còn lại. Đọc log recovery để liên hệ WAL/checkpoint với hành vi. Ready polling không quyết định transaction order bằng thời gian; nó chờ bằng chứng từ client.

**Debug/phục hồi.** Nếu ready không xuất hiện, không kill database; đọc client log. Nếu healthcheck không lên, đọc PostgreSQL log và dung lượng disk; không xóa volume để làm test xanh. EXIT/INT/TERM trap cố khởi động lại PostgreSQL nếu lỗi sau khi kill làm cell dừng; SIGKILL toàn shell/host mất vẫn cần chạy lại lệnh up thủ công. Nếu Docker tự restart quá nhanh, lệnh up vẫn sử dụng cùng service/volume và wait readiness. Tránh bấm lại giữa lúc bài còn chạy.

**Giới hạn chứng minh.** Đây không thử torn write, mất host, mất volume, nguồn điện của storage, lỗi filesystem hay split brain. Durability còn phụ thuộc cấu hình và thiết bị. Kết quả đúng ở đây bổ sung cho backup restore; nó không thay backup restore. Một process sống lại được chưa có nghĩa một doanh nghiệp có thể chịu được lỗi toàn vùng.

---

# Bài 31 — VPS: đưa lab lên máy thật và dùng SSH tunnel

## 31A. Mô hình triển khai cần hiểu

Trên VPS, Docker bridge là mạng nội bộ cho service. Publish `127.0.0.1:15432:5432` làm PostgreSQL truy cập được qua loopback của host; nó không mở cổng trên mọi interface. Laptop truy cập bằng SSH tunnel đến loopback đó. API container trong cùng project dùng hostname `postgres` và cổng `5432`, không dùng `localhost:15432` bên trong container để tìm database khác.

SSH bảo vệ đoạn kết nối qua tunnel; nó không tự giải quyết quyền DB, credential bị lộ, backup, patch hay compromised host. Không public Adminer/pgAdmin chỉ để dễ xem bảng. Với lab, psql qua SSH đủ cho đa số bài. Nếu dùng GUI, đặt endpoint vào tunnel và giữ credential đúng scope học tập.

**Tiên quyết.** VPS Linux riêng cho học tập, có user SSH được phép dùng Docker, Compose v2 và SSH key hoạt động. Bạn đã xác minh host key qua kênh tin cậy; không tắt StrictHostKeyChecking cho tiện. Laptop có ssh/scp/tar. Chi phí VPS và dung lượng nằm trong ngân sách bạn lựa chọn; notebook không tự thuê máy.

## 31B. Chuyển file cấu hình và khởi động PostgreSQL riêng

Sửa đúng một target trong cell. Cell dừng khi còn placeholder. `state/vps-target` là file đầu vào cho các bước sau; đây là dữ liệu lưu trên đĩa có chủ đích, không phải biến shell mang từ cell trước. Hãy kiểm tra target trước mỗi lần chạy để tránh thao tác nhầm VPS.

```bash {"name":"db-31-vps-deploy","excludeFromRunAll":"true"}
set -Eeuo pipefail
cd "${HOME}/backend-labs/database"
db_vps_target="your-user@your-vps.example"
if [ "$db_vps_target" = "your-user@your-vps.example" ]; then
  echo "Edit db_vps_target to your learning VPS before running." >&2
  exit 2
fi
printf '%s\n' "$db_vps_target" > state/vps-target
ssh "$db_vps_target" 'umask 077; mkdir -p "$HOME/backend-labs/database"; chmod 700 "$HOME/backend-labs/database"; docker version; docker compose version'
tar -cf - compose.yaml db.sh package.json package-lock.json scripts \
  | ssh "$db_vps_target" 'tar -xf - -C "$HOME/backend-labs/database"'
scp .env "$db_vps_target:backend-labs/database/.env"
ssh "$db_vps_target" 'bash -s' <<'REMOTE'
set -Eeuo pipefail
cd "${HOME}/backend-labs/database"
umask 077
set -a
source ./.env
set +a
printf 'DB_LAB_PASSWORD=%s\nLAB_UID=%s\nLAB_GID=%s\n' \
  "$DB_LAB_PASSWORD" "$(id -u)" "$(id -g)" > .env.next
mv .env.next .env
mkdir -p evidence backups state pgbouncer
chmod 700 db.sh
docker compose config --quiet
docker compose up -d --wait --wait-timeout 120 postgres
docker compose run --rm --entrypoint npm toolbox ci --ignore-scripts --no-audit --no-fund
./db.sh -c "SELECT version(),current_database();"
docker compose ps
REMOTE
```

**Mong đợi.** Remote tạo project be-db riêng trên host đó. User ids của toolbox được điều chỉnh theo user remote để file bind mount ghi được; password lab giữ nguyên để tunnel client có thể dùng credential đã biết. Nếu VPS đã có project be-db, dừng để kiểm tra phạm vi trước; tên project là namespace, không phải cơ chế chống thao tác nhầm host.

**Nạp dữ liệu.** Mở notebook bằng VS Code Remote SSH trên VPS rồi chạy Bài 01,02 và các bài cần học. Hoặc dùng quy trình restore Bài 29 với archive đã chuyển vào backups và database đích được gọi tên rõ. Không chạy Bài 29B nếu mục tiêu là thay banklab; cell ấy cố ý restore sang lab_restore. Một clone production có dữ liệu thật cần chính sách che dữ liệu và quyền riêng; lab không cần dữ liệu khách hàng.

## 31C. Tạo tunnel và kiểm tra cổng qua laptop

Tunnel dùng cổng laptop **25432** để không đụng PostgreSQL local đang publish 15432. Control socket trong state giúp đóng đúng tunnel của bài. Nếu tunnel đã tồn tại, kiểm tra bằng cell status trước khi chạy lại, không kill ssh toàn hệ thống.

```bash {"name":"db-31-ssh-tunnel","excludeFromRunAll":"true"}
set -Eeuo pipefail
cd "${HOME}/backend-labs/database"
db_vps_target="$(cat state/vps-target)"
ssh -M -S "$PWD/state/db-vps.ctl" -fNT \
  -o ExitOnForwardFailure=yes \
  -o ServerAliveInterval=30 -o ServerAliveCountMax=3 \
  -L 127.0.0.1:25432:127.0.0.1:15432 \
  "$db_vps_target"
ssh -S "$PWD/state/db-vps.ctl" -O check "$db_vps_target"
ssh "$db_vps_target" 'cd "$HOME/backend-labs/database"; docker compose port postgres 5432'
```

**Kiểm tra chức năng.** GUI hoặc psql trên laptop dùng host127.0.0.1, port25432, databasebanklab, userlab và password trong .env. Không cần cài GUI trên VPS. Query `SELECT inet_server_addr(),current_database(),version()` kiểm tra bạn đang đến database nào; tên database giống local nên cần đối chiếu cả endpoint/tunnel và phiên bản/image evidence.

Nếu laptop có psql, cell sau kiểm tra query thực qua tunnel. Nếu không, dùng GUI với các giá trị trên hoặc terminal SSH chạy db.sh; không coi riêng SSH control check là bằng chứng PostgreSQL authentication thành công.

```bash {"name":"db-31-tunnel-query","excludeFromRunAll":"true"}
set -Eeuo pipefail
cd "${HOME}/backend-labs/database"
command -v psql >/dev/null || { echo "Install a PostgreSQL client or use your GUI for this optional check."; exit 2; }
set -a
source ./.env
set +a
PGPASSWORD="$DB_LAB_PASSWORD" psql -X -h 127.0.0.1 -p 25432 -U lab -d banklab \
  -v ON_ERROR_STOP=1 -c "SELECT inet_server_addr(),current_database(),version();"
```

**Debug.** Local port conflict: xác định chương trình đang nghe25432 hoặc chọn cổng local khác nhất quán, không đổi remote bind ra public. Connection refused: kiểm tra compose health trên VPS và mapping loopback. Authentication failed: kiểm tra credential/volume init, không thử tắt xác thực. SSH target sai: dừng và kiểm tra state/vps-target trước mọi thao tác.

**Giới hạn VPS một máy.** Docker restart phục hồi process, không phục hồi host mất. Volume cục bộ không chịu được mất disk. Backup cùng máy không chịu được mất máy. SLO của một dịch vụ trên một VPS phải phản ánh những điều đó; muốn chịu lỗi host cần kiến trúc, backup offsite và diễn tập tương ứng.

```bash {"name":"db-31-close-tunnel","excludeFromRunAll":"true"}
set -Eeuo pipefail
cd "${HOME}/backend-labs/database"
db_vps_target="$(cat state/vps-target)"
ssh -S "$PWD/state/db-vps.ctl" -O exit "$db_vps_target"
```

---

# Bài 32 — LOCAL tùy chọn: PostgreSQL khác MySQL/InnoDB ở những điểm nào?

## 32A. Không dùng tên isolation như một hợp đồng đa engine

| Khía cạnh | PostgreSQL baseline | MySQL/InnoDB 8.4 | Hệ quả khi làm backend |
|---|---|---|---|
| Isolation mặc định | Read Committed | Repeatable Read | Test phải ghi isolation thực của connection |
| Dữ liệu vật lý | Heap với các phiên bản tuple | Clustered index lưu row; undo hỗ trợ phiên bản cũ | Tác động khóa chính và index phụ khác nhau |
| Nonlocking snapshot và locking write | Quy tắc snapshot/conflict của PG | Consistent read và current locking read có thể nhìn trạng thái khác | RR không ngăn cùng một bug theo cùng cách |
| Index phụ | Trỏ đến heap tuple | Mang primary key để tìm clustered row | Khóa chính rộng làm nhiều secondary index rộng theo |
| Thu hồi phiên bản | Vacuum/freeze và các cơ chế liên quan | Undo/purge cùng cơ chế InnoDB | Không dùng câu “chạy VACUUM” cho MySQL |
| Partial/covering index | Partial predicate và INCLUDE | Thiết kế index/generated expression theo khả năng MySQL | Migration raw SQL cần dialect riêng |
| Lock range | Row/predicate mechanisms theo PG | Record, gap và next-key locks tùy query/isolation | Query range có thể chặn INSERT chưa có row |
| Quan sát | EXPLAIN BUFFERS, pg_stat_* | EXPLAIN ANALYZE và Performance Schema/InnoDB views | Tên metric và đơn vị không sao chép nguyên |
| DDL | Nhiều DDL transactional, có ngoại lệ | Nhiều DDL gây implicit commit | “Atomic DDL” không đồng nghĩa rollback cùng transaction ứng dụng |

Các cơ chế storage được đối chiếu từ [InnoDB clustered/secondary indexes](https://dev.mysql.com/doc/refman/8.4/en/innodb-index-types.html) và [InnoDB multi-versioning](https://dev.mysql.com/doc/refman/8.4/en/innodb-multi-versioning.html). Quy tắc snapshot/locking read cần đọc đúng [isolation levels](https://dev.mysql.com/doc/refman/8.4/en/innodb-transaction-isolation-levels.html); không suy ra bằng cách đổi tên lệnh PG.

**Điều quan trọng cho ECB.** Schema Prisma có cùng model không làm SQL, constraint, default isolation, affected-row behavior, error code và migration workflow giống nhau. Hãy kiểm tra provider và version thực trong repository/deployment. Notebook không đã đọc cấu hình ECB hiện hành, nên không gán các giá trị mặc định cho môi trường đó.

## 32B. Dựng MySQL trong profile riêng

**Phạm vi.** Dừng PostgreSQL trong lab để tiết kiệm RAM, không xóa volume. MySQL chỉ publish loopback13306. Quyền monitoring cấp dưới đây chỉ dành cho user lab để quan sát lock waits; không đưa nguyên quyền đó cho application production.

```bash {"name":"db-32-mysql-setup","excludeFromRunAll":"true"}
set -Eeuo pipefail
cd "${HOME}/backend-labs/database"
if [ -f compose.pool.yaml ]; then
  docker compose -f compose.yaml -f compose.pool.yaml stop pgbouncer
fi
docker compose stop postgres
docker compose --profile mysql up -d --wait --wait-timeout 180 mysql
docker compose exec -T mysql sh -c \
  'MYSQL_PWD="$MYSQL_ROOT_PASSWORD" mysql -uroot --protocol=socket' <<'SQL'
GRANT SELECT ON performance_schema.* TO 'lab'@'%';
GRANT PROCESS ON *.* TO 'lab'@'%';
SELECT VERSION(),@@transaction_isolation,@@sql_mode;
SQL
```

## 32C. RR đọc snapshot cũ nhưng UPDATE không tự chặn lost update

**Ý định.** Chạy lại lịch A/B đã dùng ở PostgreSQL RR. Sau đó dùng conditional update và quan sát lock wait bằng Performance Schema. Bài còn tạo range lock để xem INSERT vào khoảng trống bị chặn.

```bash {"name":"db-32-mysql-concurrency","excludeFromRunAll":"true"}
set -Eeuo pipefail
cd "${HOME}/backend-labs/database"
cat > scripts/mysql-labs.mjs <<'JS'
import mysql from 'mysql2/promise';
import assert from 'node:assert/strict';
import {setTimeout as delay} from 'node:timers/promises';
const config={
  host:process.env.MYSQL_HOST,user:'lab',password:process.env.MYSQL_PASSWORD,
  database:'mysqllab',supportBigNumbers:true,bigNumberStrings:true,
  connectTimeout:5000
};
const clients=[];
const captured=p=>p.then(value=>({ok:true,value}),error=>({ok:false,error}));
async function blocked(o,waiter,holder) {
  const deadline=Date.now()+5000;
  while(Date.now()<deadline) {
    const [rows]=await o.execute(
      'SELECT count(*) AS n FROM performance_schema.data_lock_waits w JOIN performance_schema.threads r ON r.THREAD_ID=w.REQUESTING_THREAD_ID JOIN performance_schema.threads h ON h.THREAD_ID=w.BLOCKING_THREAD_ID WHERE r.PROCESSLIST_ID=? AND h.PROCESSLIST_ID=?',
      [waiter,holder]
    );
    if(Number(rows[0].n)>0) { console.log('Observed InnoDB lock wait'); return; }
    await delay(25);
  }
  throw new Error('Expected MySQL lock wait not observed');
}
try {
  for(let i=0;i<3;i++) clients.push(await mysql.createConnection(config));
  const [a,b,o]=clients;
  for(const c of clients) {
    await c.query('SET SESSION TRANSACTION ISOLATION LEVEL REPEATABLE READ');
    await c.query('SET SESSION innodb_lock_wait_timeout=10');
  }
  const [[{id:pidA}]]=await a.query('SELECT CONNECTION_ID() AS id');
  const [[{id:pidB}]]=await b.query('SELECT CONNECTION_ID() AS id');
  await o.query('DROP TABLE IF EXISTS mysql_wallet');
  await o.query('CREATE TABLE mysql_wallet(id int PRIMARY KEY,balance_minor bigint NOT NULL CHECK(balance_minor>=0)) ENGINE=InnoDB');
  await o.query('INSERT INTO mysql_wallet VALUES(1,100000)');
  await a.beginTransaction(); await b.beginTransaction();
  const [[oldA]]=await a.query('SELECT balance_minor FROM mysql_wallet WHERE id=1');
  const [[oldB]]=await b.query('SELECT balance_minor FROM mysql_wallet WHERE id=1');
  assert.equal(oldA.balance_minor,'100000');
  assert.equal(oldB.balance_minor,'100000');
  await a.query('UPDATE mysql_wallet SET balance_minor=20000 WHERE id=1');
  await a.commit();
  const [[stillOld]]=await b.query('SELECT balance_minor FROM mysql_wallet WHERE id=1');
  assert.equal(stillOld.balance_minor,'100000');
  await b.query('UPDATE mysql_wallet SET balance_minor=20000 WHERE id=1');
  await b.commit();
  const [[end]]=await o.query('SELECT balance_minor FROM mysql_wallet WHERE id=1');
  assert.equal(end.balance_minor,'20000');
  console.log('MySQL RR: both stale absolute writes committed; snapshot read alone did not prevent lost update');

  await o.query('UPDATE mysql_wallet SET balance_minor=100000 WHERE id=1');
  await a.beginTransaction(); await b.beginTransaction();
  const sql='UPDATE mysql_wallet SET balance_minor=balance_minor-80000 WHERE id=1 AND balance_minor>=80000';
  const [first]=await a.query(sql);
  const pending=captured(b.query(sql));
  await blocked(o,pidB,pidA);
  await a.commit();
  const second=await pending;
  assert.equal(second.ok,true);
  assert.equal(first.affectedRows,1);
  assert.equal(second.value[0].affectedRows,0);
  await b.commit();
  console.log('Conditional update accepted once');

  await o.query('DROP TABLE IF EXISTS mysql_ranges');
  await o.query('CREATE TABLE mysql_ranges(id int PRIMARY KEY,slot int NOT NULL,KEY slot_idx(slot)) ENGINE=InnoDB');
  await o.query('INSERT INTO mysql_ranges VALUES(1,10),(2,20)');
  await a.beginTransaction(); await b.beginTransaction();
  await a.query('SELECT * FROM mysql_ranges WHERE slot>=10 AND slot<20 FOR UPDATE');
  const insert=captured(b.query('INSERT INTO mysql_ranges VALUES(3,15)'));
  await blocked(o,pidB,pidA);
  await a.commit();
  const inserted=await insert;
  assert.equal(inserted.ok,true);
  await b.commit();
  console.log('Range locking blocked an insert into the scanned gap');
} finally {
  await Promise.allSettled(clients.map(c=>c.rollback()));
  await Promise.allSettled(clients.map(c=>c.end()));
}
JS
docker compose run --rm toolbox scripts/mysql-labs.mjs
docker compose --profile mysql stop mysql
docker compose up -d --wait --wait-timeout 120 postgres
```

**Kết quả.** RR plain SELECT của B vẫn thấy100000, nhưng B có thể ghi giá trị tuyệt đối20000 sau A và commit. Conditional UPDATE chỉ chấp nhận một lần. Range query giữ khoảng thích hợp nên INSERT15 chờ. Cơ chế record/gap/next-key phụ thuộc loại index, predicate và isolation; [InnoDB locking](https://dev.mysql.com/doc/refman/8.4/en/innodb-locking.html) là nguồn để phân tích, không khái quát “mọi SELECT khóa mọi gap”.

**Debug/phục hồi.** Nếu driver trả bigint không phải string, kiểm tra supportBigNumbers/bigNumberStrings và version. Nếu không xem được data_lock_waits, kiểm tra GRANT của cell setup; đừng bỏ assertion wait rồi dùng sleep thay thế. Nếu test fail, chạy cell dừng MySQL/khởi động PostgreSQL ở cuối sau khi đọc lỗi. Dữ liệu PG vẫn ở volume cũ.

**Biên production.** MySQL local không mô phỏng Vitess router, sharding, service transaction pool, online DDL hay giới hạn nhà cung cấp. Nó giúp tách khác biệt engine khỏi khác biệt dịch vụ. Trước khi đổi isolation hoặc SQL trong ECB, chạy cùng invariant qua driver/ORM và endpoint thực của môi trường staging.

---

# Chuyên đề — PlanetScale và Prisma: xác nhận đúng hệ đang vận hành

PlanetScale có các sản phẩm database khác nhau; [trang Postgres chính thức](https://planetscale.com/postgres) xác nhận có dịch vụ Postgres. Không dùng nhãn “PlanetScale” như đồng nghĩa duy nhất với một phiên bản MySQL. Phần dưới nói riêng **PlanetScale Vitess/MySQL**, còn dịch vụ Postgres cần tài liệu và giới hạn tương ứng.

## Những thông tin phải lấy từ dự án trước khi thiết kế

| Câu hỏi | Bằng chứng cần lấy | Tác động |
|---|---|---|
| Engine/product nào? | Provider cấu hình, connection endpoint, server version | Dialect, isolation, vận hành |
| Prisma major và driver/adapter nào? | Lockfile, generated client, adapter initialization | Transaction API, pool, kiểu trả về |
| Isolation thực tế? | Query trên connection và option transaction | Lịch concurrency cần kiểm chứng |
| Có FK thật hay relation mode ở ORM? | Schema DB, Prisma relationMode, provider setting | Ai giữ referential integrity |
| Nhánh có safe migrations? | Branch setting/deploy workflow | DDL chạy ở đâu và qua bước nào |
| Transaction timeout/pool limit? | Cấu hình dịch vụ/tài liệu hiện hành | Batch size, retry budget, giữ connection |
| Replica routing? | URL/driver và topology | Read-your-writes, idempotency lookup |
| Backup/restore được thử lúc nào? | Log diễn tập, điểm phục hồi, SLO | Độ tin cậy phục hồi thực tế |

Bảng này là checklist lấy bằng chứng của người làm backend, không phải giả định về cấu hình ECB. Có thể ORM khai báo relation nhưng DB chưa có foreign key, hoặc DB có constraint mà application không xử lý lỗi rõ ràng. Chỉ nhìn type TypeScript không đủ biết dữ liệu nào được bảo vệ.

## Ba giới hạn không nên nhớ bằng khẩu truyền

**Foreign key.** Tài liệu [PlanetScale Vitess foreign keys](https://planetscale.com/docs/vitess/foreign-key-constraints) cho biết có hỗ trợ khi bật cấu hình, với giới hạn môi trường và deploy. Không còn đúng nếu nói chung “PlanetScale không có foreign key”. Tài liệu cũng lưu ý deploy request thêm FK không xác nhận mọi quan hệ cũ như cách một thao tác MySQL thông thường có thể làm; cần audit orphan trước và sau khi thay đổi.

**DDL và routine.** [MySQL compatibility](https://planetscale.com/docs/vitess/troubleshooting/mysql-compatibility) liệt kê giới hạn stored routines/triggers và direct DDL trên nhánh bật safe migrations. Vì vậy function PL/pgSQL của Bài23 là phương tiện học giao thức, không phải code deploy lên Vitess. Khi chuyển giao thức vào service transaction, bảo toàn thứ tự thao tác và invariant; không chỉ đổi cú pháp.

**Giới hạn tài nguyên.** [Vitess system limits](https://planetscale.com/docs/vitess/troubleshooting/planetscale-system-limits) tại ngày đối chiếu ghi per-transaction timeout20s và giới hạn transaction pool tùy cấu hình. Xác nhận lại cho database của bạn trước thiết kế batch. Tăng timeout ở ORM không vượt qua một giới hạn phía server. Giảm batch, giữ transaction ngắn và đo queue thường là hướng cần đánh giá trước.

## Workflow migration và transaction ở tầng ứng dụng

[Safe migrations](https://planetscale.com/docs/vitess/schema-changes/safe-migrations) dùng branch/deploy request để kiểm soát thay đổi schema; application vẫn cần tương thích cũ/mới. Một schema change “online” không tự đảm bảo một field đổi nghĩa không phá code đang chạy. Backfill, dual write, validation và contract vẫn cần kế hoạch cùng chủ sở hữu.

Trong service, bắt đầu transaction ở ranh giới nghiệp vụ, truyền transaction client xuống mọi repository cần atomicity. Tránh repository âm thầm mở transaction riêng hoặc dùng global client. Integration test nên cố tình làm bước cuối lỗi, rồi kiểm tra bước trước không còn dữ liệu; đây là cách phát hiện leak ra ngoài transaction có giá trị hơn chỉ mock rằng method $transaction đã được gọi.

Với retry, map lỗi đúng major/adapter của Prisma rồi giới hạn theo SLA. Không ghi một vòng retry quanh toàn handler có gọi provider/email mà chưa xác định side effects. Nếu code đọc một entity ngoài transaction rồi truyền vào transaction để save, cần version hoặc đọc lại; isolation bên trong không làm entity cũ thành dữ liệu mới.

Bài tập áp dụng cho ECB: chọn một use case tạo/chuyển payment, vẽ danh sách repository query và provider call theo thứ tự. Đánh dấu connection/transaction của từng query. Với mỗi điểm có thể crash, mô tả row nào tồn tại và request retry sẽ làm gì. Khi làm được, bạn có thể review tính đúng mà không phụ thuộc tên ORM.

---

# Bài 33 — LOCAL: quyền tối thiểu và parameter binding

**Ý định.** Một role chỉ đọc không được tự sửa số dư, và giá trị input có dấu nháy vẫn là dữ liệu trong câu parameterized. Bài không cung cấp một chính sách quyền hoàn chỉnh cho production; nó tạo thói quen kiểm tra quyền thực, không chỉ đặt tên user “readonly”.

```bash {"name":"db-33-privileges-and-parameters","excludeFromRunAll":"true"}
set -Eeuo pipefail
cd "${HOME}/backend-labs/database"
./db.sh <<'SQL'
DO $$
BEGIN
  IF NOT EXISTS(SELECT 1 FROM pg_roles WHERE rolname='db_lab_reader') THEN
    CREATE ROLE db_lab_reader NOLOGIN;
  END IF;
END $$;
GRANT USAGE ON SCHEMA lab TO db_lab_reader;
GRANT SELECT ON lab.wallets TO db_lab_reader;
BEGIN;
SET LOCAL ROLE db_lab_reader;
SELECT id,balance_minor FROM lab.wallets ORDER BY id;
DO $$
BEGIN
  BEGIN
    UPDATE lab.wallets SET balance_minor=balance_minor+1 WHERE id=1;
    RAISE EXCEPTION 'reader unexpectedly wrote data';
  EXCEPTION WHEN insufficient_privilege THEN
    RAISE NOTICE 'PASS: write rejected for reader role';
  END;
END $$;
ROLLBACK;
SQL
cat > scripts/parameters.mjs <<'JS'
import {run,assert} from './harness.mjs';
await run(async a=>{
  const input="nobody' OR 1=1 --";
  const result=await a.query(
    'SELECT id FROM lab.customers WHERE email=$1',[input]
  );
  assert.equal(result.rowCount,0);
  console.log('Input treated as one value; no SQL structure changed');
});
JS
docker compose run --rm toolbox scripts/parameters.mjs
```

**Mong đợi.** SELECT được phép; UPDATE bị42501 và được bắt. Parameter query không trả toàn bộ customer. Không cần thực thi một query nối chuỗi nguy hiểm để chứng minh đường an toàn có đúng semantics.

**Điều chưa được bảo vệ.** Parameter binding bảo vệ giá trị, không biến tên bảng/cột/ORDER BY tùy ý thành an toàn. Identifier và sort direction thường cần allowlist hoặc cách quoting đúng driver. Câu query vẫn có thể sai authorization nếu thiếu tenant filter. Role chỉ đọc cũng có thể đọc dữ liệu nhạy cảm vượt nhu cầu nếu được GRANT quá rộng.

**Quyền cho ứng dụng.** Tách migration owner, application role, read-only support và backup/monitoring theo nhu cầu. Privilege của object mới không tự giống object cũ nếu default privileges chưa được cấu hình đúng owner. Khi dùng routine để giới hạn ghi, xem kỹ invoker/definer và search_path; việc thêm SECURITY DEFINER thiếu review có thể mở rộng quyền ngoài ý định.

**Debug/phục hồi.** Nếu reader update được, kiểm tra membership, superuser flag và grant qua PUBLIC/role khác. Bài dùng SET LOCAL ROLE trong transaction rồi rollback để trở về quyền ban đầu. Role lab_reader còn tồn tại; đó là global object nên backup logical một database không tự mang nó theo. Bài29 đã minh họa file globals riêng để nhắc ranh giới đó.

---

# Phần VII — Playbook production: từ triệu chứng đến thay đổi có bằng chứng

Các playbook dưới đây là trình tự suy luận để luyện trên lab và áp dụng có điều chỉnh. Không có query “chẩn đoán xong trong 30 giây” cho mọi incident. Mục tiêu của những phút đầu là xác định tác động, giữ bằng chứng và giảm thêm tác động; sau đó mới sửa gốc. Một thao tác có thể hợp lý khi database học tập bị kẹt nhưng không phù hợp khi đang phục vụ thanh toán.

## Sự cố 1 — API timeout nhưng CPU database thấp

**Triệu chứng.** Endpoint ghi bị timeout, throughput giảm, CPU không cao. **Giả thuyết đầu:** nhiều request đang chờ một blocker, pool hoặc I/O, không nhất thiết thiếu CPU.

Thu request trace để tách pool wait và SQL execution. Trong pg_stat_activity, xem wait_event_type, pg_blocking_pids, xact_start và application_name. Tìm một transaction dài có nhiều waiter; đọc query đầu và lịch sử application của nó nếu có trace. Kiểm tra có migration vừa bắt đầu, session idle in transaction hoặc code chờ HTTP trong transaction không.

**Giảm tác động.** Dừng nguồn tạo thêm yêu cầu có thể retry, giới hạn concurrency hoặc tạm dừng batch đang giữ khóa nếu xác định được chủ sở hữu. Cancel query khác terminate session: cancel một query đang chạy có thể không kết thúc transaction idle của blocker. Trước khi terminate một session, đánh giá rollback, workload và trạng thái ứng dụng; ghi evidence để giải thích quyết định sau.

**Sửa bền vững.** Rút ngắn transaction, sửa đường release connection trong finally, thêm timeout có ngân sách, bỏ HTTP ra khỏi transaction, và có lock order nhất quán. Kiểm tra lại với lịch determinism của Bài19/21 và đo cả queue. Không tăng max_connections để đẩy thêm người vào cùng một cửa khóa.

## Sự cố 2 — Một query đột nhiên chậm sau khi dữ liệu tăng

**Triệu chứng.** Code không đổi nhưng p99 tăng, có thể chỉ xảy ra ở tenant lớn. **Bằng chứng:** query parameters, plan trước/sau, actual/estimated rows, stats update time, số rows và distribution của tenant.

Kiểm tra có thống kê cũ, data skew, correlated predicates, generic/custom plan hoặc một index vừa bị drop/invalid. Query bị đổi plan sau ANALYZE không nhất thiết do ANALYZE “làm hỏng”; nó có thể phản ánh phân phối mới hoặc ước lượng còn thiếu. Đo cùng tham số, cùng phiên bản, cùng trạng thái dữ liệu và workload gần tương đương.

**Giảm tác động.** Giới hạn endpoint/query tốn kém, giảm page size hoặc hoãn report nếu phù hợp sản phẩm. ANALYZE table/cột đúng có thể giúp nhưng vẫn tạo tải; xem trước quy mô. Nếu rollback một deploy query shape mới có bằng chứng rõ, ưu tiên đường ít thay đổi hơn việc sửa hàng loạt cost constants.

**Sửa bền vững.** Index theo query shape, statistics theo cột cần, rewrite để giảm dữ liệu trung gian, hoặc phân tách workload báo cáo. Đặt test/monitoring ở tenant đại diện lớn và nhỏ. Bài13 dạy một loại nguyên nhân, không bao trùm mọi plan regression.

## Sự cố 3 — WAL hoặc đĩa tăng nhanh, database sắp hết chỗ

**Triệu chứng.** Disk usage tăng, write latency tăng hoặc hệ thống từ chối ghi. **Bằng chứng:** vị trí tăng là data, WAL, temporary files, logs hay backup? Xem tốc độ tăng và thời gian còn lại, không chỉ phần trăm hiện tại.

Nếu WAL được giữ, kiểm tra replication slots, standby lag, archiving lỗi và backup đang chạy. Nếu temp files tăng, tìm sort/hash/report tương ứng. Nếu heap/index tăng sau bulk update, xem long transactions và vacuum. Kiểm tra backup mới có lưu cùng volume và retention có thực hiện không.

**Giảm tác động.** Tạm dừng nguồn ghi không thiết yếu hoặc batch lớn; tăng dung lượng theo quy trình nếu còn khả năng. Không xóa file trong pg_wal hoặc data directory bằng tay để “cứu disk”. Xóa slot cũng có thể làm consumer/replica mất khả năng tiếp tục; cần biết chủ sở hữu và kế hoạch rebuild. Không VACUUM FULL khi disk gần hết mà chưa tính dung lượng rewrite.

**Sửa bền vững.** Alert theo thời gian tới khi đầy, phân biệt các lớp dữ liệu, kiểm thử retention, xác định ownership slot và giới hạn transaction/batch. Đo dung lượng cần cho migration/restore, không chỉ trạng thái bình thường.

## Sự cố 4 — Duplicate payment sau một đợt lỗi mạng

**Triệu chứng.** Cùng intent có nhiều provider references hoặc nhiều ledger transfers. **Bằng chứng:** request id, idempotency key, tenant/scope, fingerprint, retry attempts, thời điểm commit, provider correlation id và webhook events.

Tránh “sửa balance cho đúng” ngay vì có thể làm mất quan hệ với ledger/provider. Phân biệt duplicate request, duplicate provider execution, duplicate webhook và duplicate consumer projection; chúng có cơ chế khác nhau. Một webhook lặp nhưng ledger đúng chưa phải duplicate payment, còn hai provider charge dù local chỉ có một row là sự cố liên hệ thống.

**Giảm tác động.** Ngừng nguồn retry có nguy cơ tạo key mới, chuyển request chưa biết trạng thái sang reconcile, giữ audit. Nếu cần refund/reversal, dùng quy trình nghiệp vụ có bằng chứng từ provider và tham chiếu gốc, không xóa transaction để che lỗi.

**Sửa bền vững.** Unique key/fingerprint/response trong DB, provider idempotency, inbox cho webhook, trạng thái UNKNOWN/PENDING cho kết quả chưa biết, và reconciliation có chủ sở hữu. Reproduce window bằng cách ngắt sau commit/ack trong môi trường thử nghiệm; unit test happy path không bắt được lớp lỗi này.

## Sự cố 5 — Autovacuum chạy nhưng bảng vẫn tăng

**Triệu chứng.** Có last_autovacuum gần đây nhưng dead tuples hoặc relation size vẫn cao. **Bằng chứng:** workload update, transaction age, backend_xmin, slot xmin/catalog_xmin nếu liên quan, vacuum logs và I/O.

Một vacuum chạy không có nghĩa mọi phiên bản có thể được thu hồi. Snapshot dài có thể giữ horizon; update tiếp diễn có thể tạo dead tuples mới nhanh hơn nhịp dọn. File giữ khoảng trống để tái sử dụng không đồng nghĩa rò rỉ. Cũng có thể index bloat chiếm phần lớn thay vì heap.

**Giảm tác động.** Kết thúc transaction/report không còn cần, xử lý consumer/slot bỏ quên với kế hoạch phù hợp, điều chỉnh batch để giảm churn. Đừng disable autovacuum vì nó đang dùng I/O rồi quên bật lại; vấn đề có thể nặng hơn và còn liên quan freeze.

**Sửa bền vững.** Tối ưu update không cần thiết, tránh đánh index mọi cột nóng, điều chỉnh theo từng bảng dựa trên churn, và giám sát horizon/tuổi XID. Nếu cần rewrite/reindex, lập ngân sách lock/disk và kế hoạch thực hiện; không đặt một job VACUUM FULL chung cho mọi đêm.

## Sự cố 6 — Migration làm phần lớn API bị block

**Triệu chứng.** DDL chờ lock, request mới cũng bắt đầu xếp hàng. **Bằng chứng:** lock queue, transaction lâu trước migration, loại ALTER, thời điểm deploy và số session waiting.

**Giảm tác động.** Cancel migration chờ khi chưa áp dụng nếu đó là nguồn gây tắc và có đường chạy lại rõ ràng. Kiểm tra trạng thái thực sau cancel: index concurrently có thể invalid, migration tracker có thể chưa hoàn tất, hoặc một phần DDL đã commit nếu tool chạy nhiều transaction. Không tự đánh dấu migration applied chỉ vì muốn deploy tiếp.

**Sửa bền vững.** Dùng lock_timeout phù hợp, preflight long transaction, expand–contract, backfill resumable và validation riêng. Thử migration ở kích thước đại diện và có code cũ/mới cùng chạy. Khi rollback schema không thể bảo toàn dữ liệu mới ghi, forward fix có thể là lựa chọn ít mất thông tin hơn; cần review cụ thể.

## Sự cố 7 — Restore thành công nhưng ứng dụng vẫn lỗi

**Triệu chứng.** pg_restore exit0 nhưng app không khởi động hoặc dữ liệu mới INSERT conflict. **Bằng chứng:** sequence, grants, extensions, collation/version, connection settings, migration state và invariants nghiệp vụ.

Count giống chưa đủ. Kiểm tra role có quyền gọi sequence/routine, default privileges của owner mới, config mà pg_dump không bao gồm, và những dependency ngoài database như object storage, key mã hóa, broker offset và provider state. Nếu restore về quá khứ, event ở broker có thể đã đi trước database.

**Giảm tác động.** Giữ traffic ở chế độ kiểm soát, chạy smoke test có rollback hoặc trên clone, tránh cho worker replay hàng loạt trước khi hiểu checkpoint. Reconcile các request có kết quả bên ngoài trước khi tiếp tục retry.

**Sửa bền vững.** Định nghĩa “restore đạt” bằng ứng dụng và invariant, tự động hóa manifest/sequence/permission checks, giữ configuration và runbook đi cùng backup. Đo RTO từ thời điểm tuyên bố sự cố đến khi dịch vụ được kiểm chứng, không chỉ thời gian chạy pg_restore.

## Sự cố 8 — Pool timeout sau khi tăng số replica ứng dụng

**Triệu chứng.** Thêm API instances để chịu tải nhưng tỷ lệ timeout tăng. **Bằng chứng:** số instance × pool limit, active transactions, transaction duration, queue tại application và PgBouncer, memory/CPU database.

Mỗi instance mới có pool riêng. Tổng upper bound tăng có thể vượt backend budget dù từng instance còn nhẹ. Retry đồng loạt làm queue tăng thêm. Nếu transaction giữ connection trong lúc chờ provider, thêm instance tăng số connection bị giữ mà không tăng tốc provider.

**Giảm tác động.** Giới hạn concurrency và retry, cân đối pool theo tổng deployment, tạm giảm worker/batch không thiết yếu. Ưu tiên graceful degradation đã thống nhất với sản phẩm hơn tăng kết nối vô hạn. Theo dõi lại sau mỗi thay đổi.

**Sửa bền vững.** Capacity test có queue và failure, pool sizing dựa trên workload và ngân sách connection, transaction ngắn, backpressure và retry jitter. Test cả scale-up, rolling deployment có hai thế hệ instance cùng tồn tại, và failover khiến client reconnect đồng thời.

---

# Bài 34 — LOCAL: kiểm tra tổng hợp các invariant của luồng mẫu

**Ý định.** Sau khi học nhiều cơ chế, quay lại dữ liệu nghiệp vụ để xác nhận toàn bộ mẫu vẫn nhất quán. Bài chỉ đọc schema pay của Bài23, không thực hiện thêm chuyển khoản. Các phép kiểm này cũng là gợi ý cho job reconciliation, nhưng production cần phạm vi thời gian/snapshot và đặc tả ledger đầy đủ hơn.

```bash {"name":"db-34-capstone-invariants","excludeFromRunAll":"true"}
set -Eeuo pipefail
cd "${HOME}/backend-labs/database"
./db.sh <<'SQL' | tee evidence/34-integrity-report.txt
SELECT clock_timestamp() AS checked_at;
SELECT lab.assert((SELECT count(*)>0 FROM pay.transfers),'fixture has transfers');
SELECT lab.assert(
  NOT EXISTS(
    SELECT t.id FROM pay.transfers t LEFT JOIN pay.entries e ON e.transfer_id=t.id
    GROUP BY t.id,t.amount_minor
    HAVING count(e.id)<>2 OR sum(e.delta_minor)<>0
       OR max(e.delta_minor)<>t.amount_minor OR min(e.delta_minor)<>-t.amount_minor
  ),'two balanced entries of the exact transfer amount');
SELECT lab.assert(
  NOT EXISTS(
    SELECT 1 FROM pay.entries e JOIN pay.transfers t ON t.id=e.transfer_id
    WHERE e.scope<>t.scope OR e.currency<>t.currency
       OR (e.delta_minor<0 AND e.account_id<>t.source_id)
       OR (e.delta_minor>0 AND e.account_id<>t.destination_id)
  ),'ledger scope currency and account direction agree');
SELECT lab.assert(
  NOT EXISTS(
    SELECT 1 FROM pay.transfers t LEFT JOIN pay.outbox o ON o.transfer_id=t.id
    WHERE o.event_id IS NULL
  ),'every posted transfer has an outbox event');
SELECT lab.assert(
  NOT EXISTS(SELECT 1 FROM pay.requests WHERE response IS NULL),
  'no committed incomplete request marker');
SELECT lab.assert(
  (SELECT sum(balance_minor)=200000 FROM pay.accounts WHERE scope='t1' AND currency='EUR'),
  'closed two-account fixture preserves total EUR units');
SELECT (SELECT count(*) FROM pay.transfers) AS transfers,
       (SELECT count(*) FROM pay.entries) AS entries,
       (SELECT count(*) FROM pay.outbox) AS events;
SQL
```

**Kỳ vọng.** Mọi assertion qua với fixture chưa được mở rộng bằng deposit/withdrawal ngoài hai account. Nếu bạn đã thêm nghiệp vụ nạp tiền, invariant tổng200000 phải được thay bằng phương trình gồm opening balance và các dòng external flow có ý nghĩa, không sửa số cho vừa kết quả.

**Debug/phục hồi.** Query lỗi phải được xem như bằng chứng cần giải thích. Không chạy UPDATE sửa hàng loạt để test qua. Muốn trở về fixture, chạy Bài23 schema rồi concurrency; đây là reset schema pay đã mô tả. Bài34 không chứng minh mọi lịch concurrent hoặc lỗi provider; nó bổ sung kiểm tra trạng thái cuối cho các thí nghiệm failure trước đó.

---

# Phần VIII — Bài tập, lời giải định hướng và kiểm tra năng lực

## 1. Mười hai câu hỏi phải tự giải thích được

**1. Có CHECK(balance>=0) thì có thể hết tiền nhưng vẫn “rút thành công” hai lần không?** Có. Bài16 cho hai transaction cùng ghi một số dư không âm cũ và ghi hai audit. CHECK chỉ thấy từng row. Cần bảo vệ quyết định debit bằng conditional update/locking/version phù hợp và giữ ledger cùng transaction.

**2. RR có bảo vệ một giá trị đọc từ màn hình cách đây một giờ không?** Không tự bảo vệ. Snapshot của transaction mới không biết lúc màn hình lấy dữ liệu. Version/ETag hoặc đọc lại và kiểm tra là cách thể hiện việc “không ghi đè trạng thái đã đổi”. Phải xác định conflict là lỗi người dùng xử lý hay tình huống có thể merge.

**3. Tại sao Serializable vẫn cần idempotency?** Vì transaction có thể commit nhưng response mất; request mới có thể là một transaction tuần tự hoàn toàn hợp lệ nhưng trùng ý định. Serializable quản lý tương tác transaction trong database, còn idempotency định danh ý định lặp. Nó cũng không đảo ngược side effect đã gửi ra ngoài.

**4. Có Index Only Scan thì chắc chắn không đọc heap?** Không. Visibility có thể cần kiểm tra heap, thể hiện qua Heap Fetches. Cần đọc plan và workload cập nhật; một index covering rất rộng có thể khiến hệ thống ghi chậm hơn dù query đó ít heap fetch.

**5. Vì sao thiếu cột đầu composite index không luôn làm index vô dụng?** Planner vẫn có thể scan một phần rộng hoặc toàn index để tiết kiệm heap/filter, và PG18 còn có skip scan khi có lợi. Câu hỏi đúng là lượng công việc và thứ tự kết quả, không chỉ index “dùng được” hay không.

**6. PgBouncer transaction pooling cho phép giữ temp table để dùng ở request sau không?** Không nên giả định. Backend có thể đổi khi transaction kết thúc. Feature tương thích phải kiểm tra theo version/config; transaction-local state và session state có vòng đời khác nhau. Dùng endpoint/session mode thích hợp khi feature thực sự cần.

**7. OFFSET chậm theo cấp số nhân khi page tăng?** Không phải mô tả tổng quát đúng. Nó thường phải đi qua/bỏ nhiều row hơn theo offset và plan; chi phí có thể tăng theo lượng công việc, nhưng không mặc định exponential. Ngoài hiệu năng, concurrent insert/delete làm thay đổi vị trí row và gây duplicate/missing.

**8. Vì sao partial index không miễn phí khi row rời predicate?** Thay đổi membership phải được phản ánh trong index và MVCC/WAL. Kích thước nhỏ giúp một số query nhưng không bỏ chi phí bảo trì. Đo tỷ lệ chuyển trạng thái và số index khác trước khi kết luận.

**9. Một replica luôn có bản sao nên có thể bỏ backup?** Không. Sai lệnh xóa/cập nhật có thể lan tới replica; mất cùng miền lỗi có thể mất cả hai; replica không tạo retention lịch sử tùy ý. Backup cần độc lập và restore được, còn replica giải một phần bài toán availability/read scale.

**10. Vì sao phải restore sequence và quyền, dù rows đã đúng?** Ứng dụng còn phải tạo dữ liệu mới và được phép chạy. Sequence sai có thể gây conflict; role thiếu quyền có thể làm endpoint lỗi; owner/extension/cấu hình sai có thể chỉ lộ khi chạy code. Restore không kết thúc ở bảng count.

**11. Xem idx_scan=0 rồi drop index được không?** Chưa đủ. Nó có thể giữ unique/primary constraint, phục vụ query hiếm quan trọng, hoặc thống kê mới reset. Cần workload window đại diện, dependency và kế hoạch rollback. Tăng tốc write bằng cách bỏ constraint nghiệp vụ là thay đổi tính đúng, không đơn thuần tối ưu.

**12. Hai worker dùng SKIP LOCKED có bảo đảm một email gửi đúng một lần không?** Không. Lease có thể hết sau khi email gửi nhưng trước khi completion commit; worker khác xử lý lại. Cần idempotency ở hệ thống gửi hoặc cơ chế theo dõi/reconcile phù hợp. DB claim chỉ bảo vệ quyền xử lý trong một phạm vi thời gian và giao thức.

## 2. Bài tập thiết kế có lời giải định hướng

### Bài tập A — Hạn mức tổng theo ngày

Yêu cầu: một khách không được chuyển quá500000 đơn vị mỗi ngày UTC, dù dùng nhiều account. Chỉ conditional UPDATE trên từng account không bảo vệ tổng ngày. Một hướng là tạo row quota theo(customer,day,currency), unique key và atomic tăng used khi used+amount<=limit trong cùng transaction với transfer. Cần tạo row quota bằng upsert có xử lý cạnh tranh, chuẩn hóa ngày và định nghĩa khoản đã giữ/posted/refunded ảnh hưởng quota thế nào.

Hướng khác là serializable trên truy vấn tổng phù hợp, kèm retry và index; chi phí predicate/serialization cần đo. Điều chưa đủ là “SELECT sum hôm nay rồi INSERT” ở RC. Lời giải phải nêu mọi writer, trường hợp rollback, timezone và event đến muộn, không chỉ đưa SQL happy path.

### Bài tập B — Một request đang xử lý có thể bị hủy

Yêu cầu: customer cancel payment trong khi worker đang gọi provider. Không thể chỉ UPDATE status=CANCELLED rồi cho rằng tiền ngoài chưa đi. Thiết kế state machine có transition hợp lệ, version/lock bảo vệ transition, idempotency provider, và trạng thái chờ xác nhận/reversal nếu provider đã thực hiện.

Lời giải cần xét cancel đến trước dispatch, sau dispatch nhưng trước response, sau provider success và khi callback đến muộn. Một cột status không thay bản đồ transition. Giao dịch DB chốt intent và outbox, còn kết quả ngoài được đưa về qua một bước có thể reconcile.

### Bài tập C — Thêm currency cho bảng amount cũ

Yêu cầu: bảng cũ ngầm hiểu amount là EUR cent; nay hỗ trợ VND. Thêm currency DEFAULT EUR không giải quyết việc writer cũ có thể gửi VND mà không biết cột mới. Cần contract API version, mapping cũ thànhEUR, validate scale theo currency, dual write/reader tương thích và backfill có chứng cứ.

Lời giải phải tách migration cấu trúc khỏi thay đổi nghĩa dữ liệu. Không có cách an toàn để suy currency từ giá trị amount nếu dữ liệu cũ không lưu thông tin đó. Khi thiếu dữ liệu, phải nêu cách lấy lại từ nguồn có thẩm quyền hoặc giữ unknown; không đoán theo độ lớn số tiền.

### Bài tập D — Báo cáo ngày có nhiều query

Yêu cầu: count, sum và danh sách chi tiết phải cùng một thời điểm. Với RC nhiều statement có thể nhìn các snapshot khác nhau. Có thể dùng một query tổng hợp, một RR read-only transaction đủ ngắn, hoặc export/report job chốt snapshot rồi đọc kết quả. Phải cân nhắc thời gian chạy và ảnh hưởng vacuum, không giữ transaction suốt thời gian người dùng tải từng trang.

Lời giải phải định nghĩa “ngày” và watermark, cách xử lý late-arriving records và liệu báo cáo có được sửa lại sau chốt. Tính nhất quán vật lý của snapshot chưa tự quyết định chính sách kế toán đối với event đến muộn.

### Bài tập E — Tối ưu endpoint danh sách payment

Yêu cầu: lọc tenant/status, order created_at desc, trả20 dòng. Hãy đề xuất index bắt đầu từ query shape, quyết định khóa phụ id cho timestamp trùng và chỉ include các cột cần. So full index với partial pending nếu endpoint riêng cho work queue. Kiểm tra tenant lớn và nhỏ, status phổ biến/hiếm, generic/custom plan nếu liên quan.

Lời giải phải đo chi phí ghi và cursor correctness. Không mặc định count(*) toàn tập nên chạy cùng mọi request. Nếu UX cần tổng số trang chính xác dưới concurrent write, hãy chỉ ra chi phí và semantics; có thể cần count snapshot/asynchronous estimate với label rõ.

## 3. Checkpoint năng lực theo mức

| Mức | Có thể làm độc lập | Bằng chứng tối thiểu |
|---|---|---|
| Nền | Viết schema có khóa, constraint, amount/currency đúng | Dữ liệu xấu bị từ chối theo loại lỗi mong đợi |
| Truy vấn | Đọc plan và đề xuất index có tradeoff | Plan trước/sau, cùng result, distribution và version |
| Cạnh tranh | Tái lập và sửa anomaly | Lịch chạy xác định, error/rowCount đúng, invariant cuối |
| Workflow | Thiết kế retry/idempotency/outbox/inbox | Bảng crash windows và test duplicate/unknown state |
| Vận hành | Chẩn đoán lock/pool/disk, restore backup | Evidence incident, restore log, invariant và RTO thành phần |
| Production | Kết nối tất cả với SLO và quyền | Capstone review, rollback/recovery và giới hạn còn lại |

Bạn chưa đạt mức tiếp theo chỉ vì đã chạy xong tất cả cell. Hãy thử giải thích một kết quả khác dự đoán. Ví dụ planner dùng seq scan sau khi có index, hoặc RR của MySQL không trả40001 như PG. Nếu phải ép output về đúng ảnh minh họa mới tự tin, bạn còn cần luyện cách phân biệt semantics với implementation detail.

## 4. Capstone — Mini payment service có thể review

Xây một service nhỏ trên schema tương đương pay, dùng Node/NestJS/stack quen thuộc của bạn. API chấp nhận amount dạng chuỗi, currency, source, destination và idempotency key. Response có transfer id và trạng thái; auth/tenant lấy từ context đã xác thực. Repository nhận transaction client, mọi statement cần atomicity chạy trên client đó.

**Yêu cầu chức năng và lỗi:**

- Hai request cùng key/cùng payload trả cùng transfer, chỉ debit một lần.
- Cùng key/khác payload bị từ chối rõ.
- Hai request khác key nhưng thiếu tổng balance chỉ chấp nhận số phù hợp.
- Lỗi ở bước ghi ledger/outbox rollback cả debit và credit.
- Timeout sau commit có đường tra trạng thái và retry cùng key.
- Dispatcher có thể gửi trùng; inbox ngăn áp dụng projection hai lần.
- Event ngoài thứ tự được xử lý theo version/policy đã mô tả.
- Một migration thêm field mới chạy khi code cũ/mới cùng tồn tại.
- Dump từ môi trường capstone được restore sang DB khác và chạy smoke test.
- Connection budget và SLO có phép đo; không dùng cùng admin role cho app.

**Cách kiểm chứng.** Dùng barriers hoặc fault point rõ ràng để ngắt sau database commit, sau broker ack và trước consumer ack. Ghi event/request ids để nối bằng chứng giữa các tiến trình. Assert dữ liệu cuối bằng query độc lập, không chỉ assert response HTTP. Chạy một workload nhỏ có concurrency tăng dần, dừng khi thấy queue/latency/error vượt ngưỡng bạn đặt; không cần một triệu request để chứng minh một invariant.

**Hồ sơ review phải có.** Sơ đồ ranh giới transaction; schema và lý do các constraint; bảng error/retry; query plans quan trọng; ngân sách connection/memory; kế hoạch migration; backup/restore evidence; incident playbooks gắn với metric thực; danh sách những gì chưa được mô phỏng. Không chấm điểm bằng throughput đơn lẻ hoặc lời tuyên bố “exactly once”.

**Câu hỏi reviewer cuối cùng.** Nếu mất VPS ngay sau trả success, dữ liệu nào có thể mất và tại sao? Nếu provider thực hiện nhưng API timeout, ai xác định trạng thái cuối? Nếu retry sau ba tháng khi inbox/idempotency bị dọn, điều gì ngăn làm lại? Nếu một writer bypass service, invariant nào còn được DB giữ? Một thiết kế có câu trả lời rõ về giới hạn đáng tin hơn một thiết kế gọi mọi trường hợp là an toàn tuyệt đối.

---

# Dừng và reset có phạm vi

## Dừng service, giữ dữ liệu

```bash {"name":"db-90-stop-keep-data","excludeFromRunAll":"true"}
set -Eeuo pipefail
cd "${HOME}/backend-labs/database"
db_compose_files=(-f compose.yaml)
if [ -f compose.pool.yaml ]; then db_compose_files+=(-f compose.pool.yaml); fi
docker compose "${db_compose_files[@]}" --profile mysql --profile tools stop
```

Khởi động PostgreSQL lại bằng cell00 hoặc `docker compose up -d --wait postgres` trong cùng thư mục. File evidence và backup vẫn còn. Nếu có SSH tunnel, đóng bằng cell31 trên laptop; dừng container không tự dừng tunnel.

## RESET: xóa container/network/volume của project be-db

Cell sau xóa **toàn bộ dữ liệu PostgreSQL/MySQL trong named volumes của project be-db trên host đang chạy**. File backup/evidence trong thư mục host vẫn được giữ. Chỉ chạy khi muốn dựng lại lab từ đầu; không dùng sau một lỗi chỉ vì chưa biết nguyên nhân.

```bash {"name":"db-99-reset-project-volumes","excludeFromRunAll":"true"}
set -Eeuo pipefail
cd "${HOME}/backend-labs/database"
db_compose_files=(-f compose.yaml)
if [ -f compose.pool.yaml ]; then db_compose_files+=(-f compose.pool.yaml); fi
docker compose "${db_compose_files[@]}" --profile mysql --profile tools down --volumes
printf 'Removed be-db containers and named volumes; host backups/evidence retained.\n'
```

Không có docker system prune, volume prune hoặc lệnh xóa thư mục host chung. Nếu bạn chạy nhiều lab khác nhau, chúng phải dùng project name riêng như các chương tương ứng. Bản backup chưa restore kiểm chứng không nên là lý do duy nhất để tin có thể reset dữ liệu quan trọng.

---

# Tài liệu gốc chính thức và phạm vi sử dụng

Đối chiếu ngày **2026-11-17**. Những đoạn giải thích và thí nghiệm của sách được xây riêng để học các hệ quả; thời gian, kích thước, plan và giới hạn tài khoản dịch vụ phải kiểm tra lại trong môi trường thực. Các nguồn dưới đây giúp xác nhận chi tiết theo engine/phiên bản, không thay evidence runtime.

| Nhóm | Tài liệu chính thức |
|---|---|
| PostgreSQL baseline image | [Docker Official Image PostgreSQL](https://hub.docker.com/_/postgres) |
| Kiểu số và constraint | [Numeric types](https://www.postgresql.org/docs/17/datatype-numeric.html), [Constraints](https://www.postgresql.org/docs/17/ddl-constraints.html) |
| Lưu trữ | [Page layout](https://www.postgresql.org/docs/17/storage-page-layout.html), [HOT](https://www.postgresql.org/docs/17/storage-hot.html) |
| Lịch sử và durability | [Vacuum](https://www.postgresql.org/docs/17/routine-vacuuming.html), [WAL](https://www.postgresql.org/docs/17/wal-intro.html) |
| Isolation và khóa | [PG17 isolation](https://www.postgresql.org/docs/17/transaction-iso.html), [SET TRANSACTION](https://www.postgresql.org/docs/17/sql-set-transaction.html), [Explicit locking](https://www.postgresql.org/docs/17/explicit-locking.html), [SELECT](https://www.postgresql.org/docs/17/sql-select.html) |
| Kế hoạch | [Using EXPLAIN](https://www.postgresql.org/docs/17/using-explain.html), [Planner statistics](https://www.postgresql.org/docs/17/planner-stats.html), [CREATE STATISTICS](https://www.postgresql.org/docs/17/sql-createstatistics.html) |
| B-tree theo phiên bản | [PG17 multicolumn](https://www.postgresql.org/docs/17/indexes-multicolumn.html), [PG18 skip scan](https://www.postgresql.org/docs/18/indexes-multicolumn.html), [PG18 release](https://www.postgresql.org/docs/18/release-18.html) |
| Index chuyên biệt | [Index-only](https://www.postgresql.org/docs/17/indexes-index-only-scans.html), [Partial](https://www.postgresql.org/docs/17/indexes-partial.html), [JSONB](https://www.postgresql.org/docs/17/datatype-json.html), [BRIN](https://www.postgresql.org/docs/17/brin.html) |
| DDL | [CREATE INDEX](https://www.postgresql.org/docs/17/sql-createindex.html), [ALTER TABLE](https://www.postgresql.org/docs/17/sql-altertable.html), [Partitioning](https://www.postgresql.org/docs/17/ddl-partitioning.html) |
| Resource và diagnostics | [Memory/resources](https://www.postgresql.org/docs/17/runtime-config-resource.html), [Planning costs](https://www.postgresql.org/docs/17/runtime-config-query.html), [Statistics views](https://www.postgresql.org/docs/17/monitoring-stats.html), [pg_stat_statements](https://www.postgresql.org/docs/17/pgstatstatements.html) |
| Backup/recovery | [pg_dump](https://www.postgresql.org/docs/17/app-pgdump.html), [pg_restore](https://www.postgresql.org/docs/17/app-pgrestore.html), [PITR](https://www.postgresql.org/docs/17/continuous-archiving.html), [Standby](https://www.postgresql.org/docs/17/warm-standby.html) |
| Hashing | [pgcrypto](https://www.postgresql.org/docs/17/pgcrypto.html) |
| PostgreSQL driver | [node-postgres transactions](https://node-postgres.com/features/transactions) |
| PgBouncer | [Features](https://www.pgbouncer.org/features), [FAQ](https://www.pgbouncer.org/faq.html), [Configuration](https://www.pgbouncer.org/config.html) |
| MySQL | [Isolation](https://dev.mysql.com/doc/refman/8.4/en/innodb-transaction-isolation-levels.html), [Locking](https://dev.mysql.com/doc/refman/8.4/en/innodb-locking.html), [Locking reads](https://dev.mysql.com/doc/refman/8.4/en/innodb-locking-reads.html), [Clustered indexes](https://dev.mysql.com/doc/refman/8.4/en/innodb-index-types.html), [MVCC](https://dev.mysql.com/doc/refman/8.4/en/innodb-multi-versioning.html) |
| PlanetScale | [Postgres product](https://planetscale.com/postgres), [Vitess compatibility](https://planetscale.com/docs/vitess/troubleshooting/mysql-compatibility), [Foreign keys](https://planetscale.com/docs/vitess/foreign-key-constraints), [Safe migrations](https://planetscale.com/docs/vitess/schema-changes/safe-migrations), [System limits](https://planetscale.com/docs/vitess/troubleshooting/planetscale-system-limits) |
| Prisma đúng major | [Prisma v6 transactions](https://www.prisma.io/docs/orm/v6/prisma-client/queries/transactions), [Prisma v7 transactions](https://www.prisma.io/docs/orm/v7/prisma-client/queries/transactions) |

