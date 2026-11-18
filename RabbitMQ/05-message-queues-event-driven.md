---
shell: bash
---

# 05. Message Queues và Event-Driven Architecture: BullMQ, RabbitMQ, PostgreSQL

> **Giáo trình thực hành cho backend engineer.** Học cách giao việc, xác nhận, thử lại, chống tác dụng phụ trùng, giữ sự kiện khi dịch vụ hỏng và vận hành trên VPS. Mỗi bài có câu hỏi cần trả lời, mã nguồn đầy đủ, lệnh chạy, kết quả cần kiểm chứng và giải thích giới hạn.
>
> **Mốc đối chiếu tài liệu: 17/11/2026.** Các phiên bản trong bài là baseline học tập được khóa có chủ đích. Chúng không phải lời khuyên rằng mọi hệ thống đang chạy nên giữ nguyên phiên bản đó. Chạy các bài theo thứ tự lần đầu; sau đó dùng tên ô Runme để quay lại bài cần luyện.

## Cách sử dụng cuốn sách

Mở Markdown bằng Runme trong VS Code hoặc đọc và chạy từng khối Bash trong terminal. Tất cả ô chạy có tên bắt đầu bằng `mq-` và `excludeFromRunAll: "true"` để việc chạy đồng loạt không vô tình thực hiện một bài gây lỗi hay dọn dữ liệu. Mỗi ô tự xác định thư mục; không trông chờ biến môi trường hoặc lệnh `cd` từ ô trước còn tồn tại. Những file tạo ở bài trước là tiền điều kiện được ghi rõ. Quy ước này dựa trên [Runme cell-level options](https://docs.runme.dev/configuration/cell-level/) và [document-level options](https://docs.runme.dev/configuration/document-level/).

Có ba loại kết luận cần phân biệt:

- **Quan sát trong lab:** điều chương trình vừa đo được trên máy của bạn. Ví dụ worker hoàn thành 12 công việc trong 1,3 giây.
- **Cam kết của thành phần:** điều tài liệu giao thức hoặc thư viện nói trong những điều kiện cụ thể. Ví dụ publisher confirm không xác nhận rằng một email đã gửi.
- **Bất biến do ứng dụng xây dựng:** điều bạn bảo vệ bằng ràng buộc dữ liệu, transaction và quy trình phục hồi. Ví dụ mỗi đơn chỉ có một dòng cộng thưởng, dù nhận nhiều bản sao của sự kiện.

Một hàng log “success” chưa chứng minh bất biến. Sau mỗi bài, hãy nhìn số lượng bản ghi, trạng thái hàng đợi và trường hợp thất bại. Thói quen này quan trọng hơn việc thuộc tên API.

### Phạm vi môi trường

| Thành phần | Baseline | Vai trò |
|---|---|---|
| Node.js | `node:24-bookworm-slim` | Chạy toàn bộ mã JavaScript ESM trong container |
| BullMQ | `5.65.1`, khóa chính xác | Task queue, retry, delay, scheduler, flow |
| ioredis | `5.6.1`, khóa chính xác | Kết nối Redis và bài sự cố |
| amqplib | `0.10.9`, khóa chính xác | AMQP 0-9-1, publisher confirms, consumer acknowledgements |
| node-postgres | `pg@8.16.3` | Transaction PostgreSQL thật |
| Redis | `redis:7.4-alpine` | Redis riêng cho queue, AOF, `noeviction` |
| RabbitMQ | `rabbitmq:4-management` | Broker, quorum queues, management UI |
| PostgreSQL | `postgres:17-bookworm` | Orders, wallet ledger, inbox, outbox |
| Docker Compose project | `be-mq` | Phạm vi container, network và volumes của chương này |
| Thư mục trên máy học | `$HOME/backend-labs/messaging` | Mã nguồn, cấu hình, evidence và bản sao lưu |

BullMQ đã có dòng major mới hơn tại thời điểm đối chiếu. Bài này dùng một baseline v5 có `upsertJobScheduler`, vì mục tiêu là hiểu cơ chế và có môi trường ổn định để thực hành. Lịch sử API được đối chiếu với [BullMQ v5 changelog](https://docs.bullmq.io/changelogs/changelog-v5). Không dùng `npm install bullmq@latest` giữa chừng: một major mới có thể đổi dependency, hành vi hoặc backend được hỗ trợ.

Các tag Docker ở bảng vẫn có thể trỏ tới image mới. Bài khởi tạo sẽ ghi lại phiên bản thực tế và digest; production cần khóa digest đã kiểm thử. amqplib `0.10.9` đáp ứng yêu cầu client `0.10.7+` của RabbitMQ `4.1+`; xem [README chính thức của amqplib](https://github.com/amqp-node/amqplib) và [changelog](https://github.com/amqp-node/amqplib/blob/main/CHANGELOG.md).

Redis và PostgreSQL không mở cổng ra máy chủ. AMQP cũng chỉ được dùng trong network Compose. Chỉ management UI bind `127.0.0.1:15673`. Tài khoản `lab / lab_only_change_me` là dữ liệu demo, chỉ dành cho môi trường này. Trên VPS, truy cập UI bằng SSH tunnel; phần triển khai cuối sách hướng dẫn cách đổi credentials và bỏ các giả định của lab.

### Lộ trình và sản phẩm học tập

| Chặng | Bài lab | Điều bạn phải giải thích được |
|---|---:|---|
| Chuẩn bị và mô hình tư duy | 01–04 | “Đã nhận yêu cầu” khác “đã làm xong”; queue là nơi tích nợ |
| BullMQ | 05–14 | Vòng đời, concurrency, retries, priority, delay, scheduler, flows, dedupe, stalled jobs |
| RabbitMQ | 15–19 | Routing, confirms, return, ACK, bounded retry, prefetch và thứ tự |
| Tính đúng đắn bằng PostgreSQL | 20–27 | Inbox, ledger, transactional outbox, lease, crash windows, concurrent relay |
| Phục hồi và tiến hóa | 28–31 | Schema, quarantine, replay, reconciliation, saga compensation |
| Quan sát và vận hành | 32–35 | Capacity, queue age, backup/restore, VPS, shutdown |
| Tổng kết | Capstone, runbooks, tự kiểm tra | Chứng minh kết quả bằng dữ liệu, không bằng cảm giác |

Không cần dựng lại dịch vụ từ các chương Redis hoặc PostgreSQL khác. Chương này có database, network và cấu hình của riêng mình. Các bài tài chính sử dụng dữ liệu tổng hợp, đơn vị tiền nhỏ nhất và số nguyên; không thực hiện thanh toán thật.

---

# Phần I. Hiểu bài toán trước khi chọn broker

## 1. Queue chuyển thời gian chờ đi đâu?

Một endpoint tạo báo cáo có thể mất 50 mili giây kiểm tra quyền và ghi một yêu cầu, rồi mất 40 giây đọc dữ liệu, render PDF và upload. Cho endpoint chờ toàn bộ công việc khiến kết nối HTTP tồn tại lâu; client có thể timeout dù tác vụ vẫn đang chạy. Nếu client gửi lại, bạn có hai lần xuất hoặc hai lần thu phí. Đưa bước dài vào queue giúp endpoint trả trạng thái `accepted` và ID theo dõi sớm hơn. Công việc dài vẫn tốn gần như lượng CPU, I/O và thời gian như cũ.

Vì vậy cần tách ba thời điểm:

1. **Accepted:** hệ thống đã ghi bền vững nghĩa vụ phải làm việc.
2. **Started:** worker bắt đầu xử lý.
3. **Completed:** tác dụng phụ cần thiết đã được xác nhận và lưu trạng thái.

Người dùng muốn file có thể quan tâm `completed`; load balancer chỉ thấy latency của `accepted`. Tối ưu số thứ hai trên dashboard API mà bỏ qua thời gian chờ cuối cùng có thể làm sản phẩm tệ hơn trong lúc biểu đồ HTTP đẹp hơn.

HTTP `202 Accepted` phù hợp khi bạn chấp nhận yêu cầu và sẽ xử lý sau. HTTP `201 Created` vẫn hợp lý nếu tài nguyên “yêu cầu xuất báo cáo” đã được tạo ngay, có trạng thái `queued`. Mã HTTP không tự tạo độ bền. Nếu chỉ gọi một Promise không `await` rồi trả 202, process có thể chết trước khi công việc được ghi ở bất kỳ đâu.

Queue cũng không tạo thêm năng lực vô hạn. Giả sử mỗi giây có 80 công việc mới nhưng hệ thống chỉ làm xong 50, backlog tăng 30 công việc mỗi giây. Sau 10 phút có thêm khoảng 18.000 việc. Nếu payload và metadata dùng trung bình 4 KiB, phần tồn đọng đó đã có quy mô khoảng 70 MiB trước overhead khác. Quan trọng hơn, một người mới đến đang phải chờ phía sau lượng việc rất lớn. Khi đầu vào vượt năng lực ổn định, cần admission control, tăng năng lực có điều kiện hoặc giảm đầu vào.

### Chọn đơn vị công việc

Một job nên mô tả một thao tác có thể kiểm tra trạng thái và thử lại. “Xử lý mọi thứ cho khách hàng” khó phục hồi vì không rõ bước nào đã xong. “Tạo hóa đơn cho order X, phiên bản mẫu Y” dễ xây khóa idempotency, lưu kết quả và đối chiếu. Nhưng tách quá nhỏ thành từng câu SQL cũng làm số message, network roundtrip và vận hành tăng mạnh.

Hãy đặt câu hỏi: nếu process chết ngay sau từng bước, worker tiếp theo có thể xác định việc gì còn phải làm không? Nếu câu trả lời là phải đọc log bằng mắt, ranh giới job hoặc mô hình trạng thái chưa đủ rõ.

## 2. Job, command và event

| Loại thông điệp | Ví dụ | Ai quyết định thực hiện? | Kỳ vọng người nhận |
|---|---|---|---|
| Job | `render-invoice(orderId)` | Thành phần tạo job đã chọn loại việc | Một worker trong nhóm |
| Command | `ReserveInventory` | Chủ thể gửi yêu cầu; service đích có thể từ chối | Một chủ sở hữu nghiệp vụ |
| Event | `OrderPaid` | Sự kiện đã xảy ra trong service sở hữu dữ liệu | Không, một hoặc nhiều subscribers |
| Snapshot | `OrderProjectionUpdated` kèm trạng thái hiện tại | Hệ thống phát bản chụp | Consumer có thể bỏ bản cũ theo version |

Đặt tên `OrderPaid` rồi dùng nó để yêu cầu service khác “hãy thử thu tiền” làm mờ ranh giới. Một event ở thì quá khứ nên phản ánh điều đã được chấp nhận trong dữ liệu nguồn. Nếu thanh toán mới bắt đầu, event có thể là `PaymentRequested` hoặc command rõ ràng hơn.

Trong lab, `order.paid` được tạo sau một transaction mô phỏng việc đơn đã được xác nhận thanh toán. Không có adapter ngân hàng thật. Với production, bạn phải xác minh callback chữ ký, trạng thái provider và khóa chống lặp trước transaction đó.

“Event-driven” cũng không đồng nghĩa “microservices”. Một modular monolith có thể dùng outbox và vài workers với một database. Ranh giới deployment nhỏ hơn thường làm quan sát và transaction đơn giản hơn. Chia thành nhiều service chỉ có giá trị khi có nhu cầu sở hữu, mở rộng hoặc triển khai độc lập đủ rõ.

## 3. BullMQ, RabbitMQ và log sự kiện

| Nhu cầu chính | Phương án nên xem xét trước | Vì sao |
|---|---|---|
| Backend Node cần delay, retry, tiến độ và vài worker | BullMQ với Redis riêng | API task queue trực tiếp, ít mã điều phối |
| Nhiều ứng dụng cần routing và nhận bản sao độc lập | RabbitMQ | Exchange/binding tách publisher khỏi topology consumers |
| Nhiều lần đọc lại cùng lịch sử với offset và retention dài | Stream/log, gồm RabbitMQ Streams hoặc nền tảng phù hợp khác | Semantics đọc giữ dữ liệu khác queue xóa sau ACK |
| Ít tác vụ, đã có PostgreSQL và đội vận hành nhỏ | Bảng jobs/outbox có worker | Giảm một hệ thống phải vận hành; vẫn cần locking, retries, metrics |
| Cần trả lời ngay để quyết định giao dịch | HTTP/gRPC hoặc transaction trong service | Queue không tự phù hợp với đường đi cần kết quả đồng bộ |

Đừng quyết định chỉ bằng câu “Kafka nhanh hơn” hoặc “RabbitMQ enterprise”. Cần workload: kích thước message, độ bền, số nhóm consumer, thứ tự theo khóa, retention, replay và khả năng của đội trực. RabbitMQ Streams là append-only log có thể đọc nhiều lần; không nên mô tả toàn bộ RabbitMQ như một hàng đợi chỉ đọc được một lần. Xem [RabbitMQ Streams](https://www.rabbitmq.com/docs/streams).

Với BullMQ v5 trong bài, Redis là nơi lưu trạng thái công việc. Với RabbitMQ AMQP 0-9-1, routing xảy ra ở exchange và delivery/ACK thuộc về queue. Hai sản phẩm có thể cùng tồn tại: service nhận sự kiện qua RabbitMQ, ghi ý định tạo báo cáo trong DB, rồi một relay đưa job sang BullMQ. Mỗi đường nối thêm một cửa sổ lỗi cần thiết kế; dùng cả hai chỉ vì “kiến trúc đẹp” thường không đáng chi phí đó.

## 4. Bốn câu hỏi về độ tin cậy

Trước mọi đoạn producer hoặc consumer, ghi câu trả lời cho bốn câu sau:

- Yêu cầu đã được lưu bền ở đâu trước khi báo accepted?
- Nếu không biết thao tác vừa thành công hay thất bại, lần thử lại dùng cùng định danh nào?
- Nếu cùng công việc được thực hiện đồng thời hai lần, ràng buộc nào chặn tác dụng phụ trùng?
- Nếu sau nhiều giờ không có tiến triển, ai phát hiện, đối chiếu và khôi phục?

Một thiết kế thường có thể bảo đảm tốt việc “không bỏ quên nghĩa vụ” trong phạm vi lỗi đã tính đến, nhưng vẫn phát sinh bản sao. Thiết kế khác chọn bỏ thông điệp khi không chắc để tránh xử lý trùng, đổi lại có thể mất việc. Thuật ngữ `at-least-once` mô tả delivery; nó không tự chứng minh “chính xác một lần trừ tiền”. Chúng ta sẽ bảo vệ việc cộng thưởng bằng inbox, ledger và transaction, rồi cố ý tạo delivery trùng để kiểm tra.

---

# Phần II. Dựng phòng lab độc lập

## Lab 01 — Kiểm tra điều kiện và tạo project

**Mục tiêu:** biết mình đang thao tác với Docker daemon nào, lưu mọi thứ vào thư mục riêng và không cài Node/npm trên máy chủ.

**Chuẩn bị:** Linux hoặc macOS với Docker Engine/Desktop đang chạy và Compose v2. Trên Windows dùng Bash trong WSL2. Cần kết nối registry Docker/npm ở lần đầu. Dự kiến dành 4 GiB RAM cho toàn bộ lab để còn dư cho hệ điều hành; mức này là cấu hình học tập, không phải benchmark sản xuất. Nếu máy có ít RAM, chạy riêng nhóm Redis/BullMQ hoặc RabbitMQ/PostgreSQL.

**Bước 1:** kiểm tra context trước khi tạo. Nếu context trỏ tới cluster hoặc máy từ xa không dành cho thực hành, chọn lại context cục bộ bằng quy trình của bạn rồi chạy tiếp. Không tự động đổi context trong notebook.

```bash {"name":"mq-01-preflight","excludeFromRunAll":"true"}
set -euo pipefail
command -v docker
docker version
docker compose version
docker context show
docker info --format 'Server={{.Name}} CPUs={{.NCPU}} Memory={{.MemTotal}}'
LAB="$HOME/backend-labs/messaging"
if [ -e "$LAB" ] && [ ! -f "$LAB/.be-mq-lab" ]; then
  printf '%s\n' "Thư mục đã tồn tại nhưng chưa có marker lab: $LAB" >&2
  exit 1
fi
mkdir -p "$LAB/app/lib" "$LAB/app/labs" "$LAB/app/capstone" "$LAB/evidence" "$LAB/backups"
printf '%s\n' 'be-mq local training workspace' > "$LAB/.be-mq-lab"
printf '%s\n' "Lab root: $LAB"
```

**Kết quả cần thấy:** server Docker có tên phù hợp, Compose in phiên bản và marker đã được tạo. Một máy có Docker CLI nhưng không kết nối daemon sẽ dừng ở `docker version`; không cố bỏ `set -e` để lách lỗi đó.

**Vì sao dùng marker?** Đường dẫn hợp lệ không có nghĩa là thư mục thuộc bài học. Marker khiến những ô viết cấu hình hoặc dọn volumes sau này có thêm điều kiện kiểm tra. Nó không phải security boundary, nhưng giảm lỗi thao tác nhầm thư mục.

## Lab 02 — Compose, dependencies, readiness và bằng chứng phiên bản

**Mục tiêu:** khởi tạo Redis, RabbitMQ, PostgreSQL và một container Node dùng chung để chạy các scripts. Readiness phải kiểm tra dịch vụ thật, không đợi một số giây tùy ý.

**Tiền điều kiện:** Lab 01. Ô này ghi các file cấu hình thuộc thư mục lab. Nếu bạn đã sửa chúng, lưu bản sửa trước khi chạy lại.

```bash {"name":"mq-02-create-compose","excludeFromRunAll":"true"}
set -euo pipefail
LAB="$HOME/backend-labs/messaging"
test -f "$LAB/.be-mq-lab"
cd "$LAB"
printf 'MQ_UID=%s\nMQ_GID=%s\n' "$(id -u)" "$(id -g)" > .env
cat > compose.yml <<'YAML'
name: be-mq
services:
  redis:
    image: redis:7.4-alpine
    command:
      - redis-server
      - --appendonly
      - "yes"
      - --appendfsync
      - everysec
      - --maxmemory
      - 192mb
      - --maxmemory-policy
      - noeviction
    volumes:
      - redis-data:/data
    mem_limit: 384m
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 2s
      timeout: 2s
      retries: 30
  rabbit:
    image: rabbitmq:4-management
    hostname: mq-rabbit
    environment:
      RABBITMQ_DEFAULT_USER: lab
      RABBITMQ_DEFAULT_PASS: lab_only_change_me
      RABBITMQ_DEFAULT_VHOST: /lab
    ports:
      - "127.0.0.1:15673:15672"
    volumes:
      - rabbit-data:/var/lib/rabbitmq
    mem_limit: 1024m
    healthcheck:
      test: ["CMD", "rabbitmq-diagnostics", "-q", "check_running"]
      interval: 5s
      timeout: 5s
      retries: 30
  postgres:
    image: postgres:17-bookworm
    environment:
      POSTGRES_USER: lab
      POSTGRES_PASSWORD: lab_only_change_me
      POSTGRES_DB: messaging
    volumes:
      - pg-data:/var/lib/postgresql/data
    mem_limit: 512m
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U lab -d messaging"]
      interval: 2s
      timeout: 2s
      retries: 30
  app:
    image: node:24-bookworm-slim
    working_dir: /app
    user: "${MQ_UID:-1000}:${MQ_GID:-1000}"
    init: true
    command: ["sleep", "infinity"]
    environment:
      REDIS_HOST: redis
      REDIS_PORT: "6379"
      AMQP_URL: amqp://lab:lab_only_change_me@rabbit:5672/%2Flab?heartbeat=10
      DATABASE_URL: postgres://lab:lab_only_change_me@postgres:5432/messaging
      NPM_CONFIG_CACHE: /tmp/be-mq-npm-cache
      NODE_ENV: development
    volumes:
      - ./app:/app
    mem_limit: 768m
    stop_grace_period: 30s
    depends_on:
      redis:
        condition: service_healthy
      rabbit:
        condition: service_healthy
      postgres:
        condition: service_healthy
volumes:
  redis-data:
  rabbit-data:
  pg-data:
YAML
cat > app/package.json <<'JSON'
{
  "name": "backend-messaging-labs",
  "version": "1.0.0",
  "private": true,
  "type": "module",
  "engines": { "node": ">=24 <25" },
  "dependencies": {
    "amqplib": "0.10.9",
    "bullmq": "5.65.1",
    "ioredis": "5.6.1",
    "pg": "8.16.3"
  }
}
JSON
docker compose -p be-mq config --quiet
docker compose -p be-mq up -d --wait --wait-timeout 180
docker compose -p be-mq exec -T app npm install --package-lock-only --ignore-scripts
docker compose -p be-mq exec -T app npm ci --ignore-scripts
docker compose -p be-mq ps
```

Lần cài đầu tiên vẫn cần giải dependency gián tiếp theo registry ở thời điểm chạy. `package-lock.json` ghi phiên bản và integrity cụ thể sau bước đó. Những lần tái tạo tiếp theo dùng `npm ci` với lock đã lưu; không xóa lock để “sửa nhanh” một lỗi. `--ignore-scripts` phù hợp bộ dependencies thuần JavaScript của baseline này; khi thêm dependency cần build, phải xem xét riêng.

**Bước 2:** lưu bằng chứng. Lệnh inspect chỉ đọc images mà Compose đang dùng. Không có thao tác prune hoặc xóa images dùng bởi dự án khác.

```bash {"name":"mq-02-record-baseline","excludeFromRunAll":"true"}
set -euo pipefail
LAB="$HOME/backend-labs/messaging"
test -f "$LAB/.be-mq-lab"
cd "$LAB"
{
  date -u '+Recorded at %Y-%m-%dT%H:%M:%SZ'
  docker compose version
  docker compose -p be-mq exec -T app node --version
  docker compose -p be-mq exec -T app npm ls --depth=0
  docker compose -p be-mq exec -T redis redis-server --version
  docker compose -p be-mq exec -T rabbit rabbitmq-diagnostics -q server_version
  docker compose -p be-mq exec -T postgres psql -U lab -d messaging -Atc 'SELECT version();'
  for service in redis rabbit postgres app; do
    image_id="$(docker compose -p be-mq images -q "$service" | head -n 1)"
    docker image inspect "$image_id" --format '{{.Id}} {{json .RepoDigests}}'
  done
} | tee evidence/baseline.txt
docker compose -p be-mq exec -T redis redis-cli CONFIG GET maxmemory-policy
docker compose -p be-mq exec -T redis redis-cli CONFIG GET appendonly
docker compose -p be-mq exec -T rabbit rabbitmq-diagnostics -q check_local_alarms
```

**Kết quả:** Redis trả `noeviction` và `yes`; RabbitMQ không có alarm; PostgreSQL trả phiên bản. Truy cập [RabbitMQ Management cục bộ](http://127.0.0.1:15673) với tài khoản demo ở đầu sách, chọn vhost `/lab`. Khi chạy lab trên VPS, URL localhost trên laptop không tự trỏ tới VPS; hãy dùng tunnel ở phần vận hành.

### Tại sao Redis queue phải tách khỏi cache?

Một cache có thể bỏ bản sao dữ liệu vì nguồn gốc còn trong PostgreSQL. Một key của queue có thể là trạng thái duy nhất cho biết một job đang đợi hoặc giữ lock của worker. Eviction một phần trong nhiều cấu trúc liên quan có thể làm dữ liệu queue không còn nhất quán. `noeviction` khiến lệnh cần thêm bộ nhớ thất bại khi chạm giới hạn thay vì tùy ý loại key. Nó không làm RAM vô hạn: producer phải xử lý lỗi và hệ thống cần retention, capacity và cảnh báo. Cách dùng connection khác nhau giữa producer và worker được mô tả trong [BullMQ production guide](https://docs.bullmq.io/guide/going-to-production).

AOF `everysec` cũng không có nghĩa “mất điện không mất một byte”. Có cửa sổ trước fsync và phụ thuộc tầng lưu trữ. Redis persistence là một phần của thiết kế RPO, không thay thế backup hoặc outbox trong database nguồn. Xem [Redis persistence](https://redis.io/docs/latest/operate/oss_and_stack/management/persistence/). Trong capstone, nghĩa vụ chưa hoàn thành có thể được đối chiếu và phát lại từ PostgreSQL.

## Lab 03 — Tạo công cụ hỗ trợ và schema PostgreSQL

**Mục tiêu:** các lab sau dùng chung helper nhỏ, đọc được toàn bộ; không giấu phần khó trong một package tự viết không rõ nguồn.

Có ba điểm cần chú ý trong helper:

1. Các bài BullMQ ngắn dùng tên queue ngẫu nhiên; cleanup chỉ xóa queue vừa tạo.
2. Producer Redis dùng số lần retry hữu hạn; worker cho phép kết nối lại. Lỗi worker có listener.
3. `confirmSend` dùng confirm channel, `mandatory`, định danh lần publish và listener `return`. Nó không nhầm giá trị boolean của `publish` với lời hứa nhận bền vững.

Trong amqplib, `publish` và `sendToQueue` không trả Promise, kể cả trên promise API. Giá trị boolean là dấu hiệu backpressure cục bộ. Callback trên confirm channel mới nói về broker confirm; ACK của consumer là một chiều khác. Xem [amqplib channel API](https://amqp-node.github.io/amqplib/channel_api.html).

```bash {"name":"mq-03-shared-helpers","excludeFromRunAll":"true"}
set -euo pipefail
LAB="$HOME/backend-labs/messaging"
test -f "$LAB/.be-mq-lab"
cd "$LAB"
cat > app/lib/common.mjs <<'JS'
import { randomUUID, createHash } from 'node:crypto';
import { setTimeout as sleep } from 'node:timers/promises';
import { Queue, Worker, QueueEvents } from 'bullmq';
import amqp from 'amqplib';
import pg from 'pg';

export { sleep, randomUUID };
export const connection = {
  host: process.env.REDIS_HOST || 'redis',
  port: Number(process.env.REDIS_PORT || 6379),
  password: process.env.REDIS_PASSWORD || undefined,
  maxRetriesPerRequest: 1,
  connectTimeout: 3000
};
export const workerConnection = {
  ...connection,
  maxRetriesPerRequest: null
};
export const pool = new pg.Pool({
  connectionString: process.env.DATABASE_URL,
  max: 6,
  connectionTimeoutMillis: 3000,
  idleTimeoutMillis: 10000,
  statement_timeout: 8000
});
pool.on('error', error => console.error('PG_IDLE_ERROR', error.message));
export function log(kind, details = {}) {
  console.log(JSON.stringify({ at: new Date().toISOString(), kind, ...details }));
}
export function stable(value) {
  if (Array.isArray(value)) return '[' + value.map(stable).join(',') + ']';
  if (value && typeof value === 'object') {
    return '{' + Object.keys(value).sort().map(
      key => JSON.stringify(key) + ':' + stable(value[key])
    ).join(',') + '}';
  }
  return JSON.stringify(value);
}
export function hash(value) {
  return createHash('sha256').update(stable(value)).digest('hex');
}
export async function until(check, timeoutMs = 15000, label = 'condition') {
  const end = Date.now() + timeoutMs;
  while (Date.now() < end) {
    const value = await check();
    if (value) return value;
    await sleep(50);
  }
  throw new Error('TIMEOUT waiting for ' + label);
}
export async function tx(fn) {
  const client = await pool.connect();
  try {
    await client.query('BEGIN');
    const result = await fn(client);
    await client.query('COMMIT');
    return result;
  } catch (error) {
    await client.query('ROLLBACK').catch(() => {});
    throw error;
  } finally {
    client.release();
  }
}
export async function bullLab(label) {
  const name = 'mq-' + label + '-' + randomUUID();
  const queue = new Queue(name, {
    connection,
    defaultJobOptions: {
      removeOnComplete: { age: 3600, count: 200 },
      removeOnFail: { age: 86400, count: 200 }
    }
  });
  const events = new QueueEvents(name, { connection: workerConnection });
  const workers = [];
  queue.on('error', error => log('QUEUE_ERROR', { message: error.message }));
  events.on('error', error => log('EVENTS_ERROR', { message: error.message }));
  await Promise.all([queue.waitUntilReady(), events.waitUntilReady()]);
  log('LAB_QUEUE', { name });
  return {
    name, queue, events,
    spawn(processor, options = {}) {
      const worker = new Worker(name, processor, {
        connection: workerConnection,
        concurrency: 1,
        ...options
      });
      worker.on('error', error => log('WORKER_ERROR', { message: error.message }));
      workers.push(worker);
      return worker;
    },
    async close() {
      for (const worker of workers) await worker.close();
      await events.close();
      await queue.obliterate();
      await queue.close();
    }
  };
}
export async function rabbit() {
  const conn = await amqp.connect(process.env.AMQP_URL, { timeout: 5000 });
  conn.on('error', error => log('AMQP_CONNECTION_ERROR', { message: error.message }));
  conn.on('blocked', reason => log('AMQP_BLOCKED', { reason }));
  conn.on('unblocked', () => log('AMQP_UNBLOCKED'));
  return conn;
}
export async function channel(conn, confirm = false) {
  const ch = confirm ? await conn.createConfirmChannel() : await conn.createChannel();
  ch.on('error', error => log('AMQP_CHANNEL_ERROR', { message: error.message }));
  return ch;
}
// Dùng một confirm channel riêng, gửi tuần tự hoặc số lượng nhỏ có giới hạn.
// Mỗi lần gọi có token riêng, còn messageId giữ ID nghiệp vụ qua các lần retry.
export function confirmSend(ch, exchange, key, body, options = {}, timeoutMs = 8000) {
  return new Promise((resolve, reject) => {
    const attemptId = randomUUID();
    let returned;
    let confirmed = false;
    let drained = true;
    let sent = false;
    let finished = false;
    const timer = setTimeout(() => finish(new Error('PUBLISH_UNCERTAIN timeout')), timeoutMs);
    const clean = () => {
      clearTimeout(timer);
      ch.removeListener('return', onReturn);
      ch.removeListener('close', onClose);
      ch.removeListener('error', onError);
      ch.removeListener('drain', onDrain);
    };
    function finish(error) {
      if (finished) return;
      finished = true;
      clean();
      if (error) reject(error); else resolve();
    }
    function maybeFinish() {
      if (sent && confirmed && drained) finish(returned);
    }
    function onReturn(msg) {
      if (msg.properties.headers?.['x-publish-attempt'] === attemptId) {
        returned = new Error('UNROUTABLE ' + msg.fields.replyCode + ' ' + msg.fields.replyText);
      }
    }
    function onClose() { finish(new Error('PUBLISH_UNCERTAIN channel closed')); }
    function onError(error) { finish(error); }
    function onDrain() { drained = true; maybeFinish(); }
    ch.on('return', onReturn);
    ch.once('close', onClose);
    ch.once('error', onError);
    ch.once('drain', onDrain);
    try {
      const ok = ch.publish(exchange, key, body, {
        ...options,
        persistent: true,
        mandatory: true,
        contentType: options.contentType || 'application/json',
        messageId: options.messageId || randomUUID(),
        headers: { ...(options.headers || {}), 'x-publish-attempt': attemptId }
      }, error => {
        if (error) return finish(error);
        confirmed = true;
        maybeFinish();
      });
      drained = ok;
      sent = true;
      if (ok) ch.removeListener('drain', onDrain);
      maybeFinish();
    } catch (error) {
      finish(error);
    }
  });
}
export async function quorum(ch, name, args = {}) {
  return ch.assertQueue(name, {
    durable: true,
    arguments: { 'x-queue-type': 'quorum', ...args }
  });
}
export class PermanentError extends Error {}
JS
cat > app/schema.sql <<'SQL'
CREATE TABLE IF NOT EXISTS wallets (
  id text PRIMARY KEY,
  currency text NOT NULL CHECK (currency ~ '^[A-Z]{3}$'),
  balance_minor bigint NOT NULL DEFAULT 0 CHECK (balance_minor >= 0)
);
CREATE TABLE IF NOT EXISTS orders (
  id text PRIMARY KEY,
  client_key text NOT NULL UNIQUE,
  request_hash text NOT NULL,
  wallet_id text NOT NULL REFERENCES wallets(id),
  currency text NOT NULL,
  total_minor bigint NOT NULL CHECK (total_minor > 0),
  reward_minor bigint NOT NULL CHECK (reward_minor >= 0),
  status text NOT NULL CHECK (status IN ('PENDING','PAID','EXPIRED','REFUNDED')),
  expires_at timestamptz,
  created_at timestamptz NOT NULL DEFAULT clock_timestamp()
);
CREATE TABLE IF NOT EXISTS inbox (
  consumer text NOT NULL,
  event_id text NOT NULL,
  payload_hash text NOT NULL,
  processed_at timestamptz NOT NULL DEFAULT clock_timestamp(),
  PRIMARY KEY (consumer, event_id)
);
CREATE TABLE IF NOT EXISTS wallet_ledger (
  operation_key text PRIMARY KEY,
  wallet_id text NOT NULL REFERENCES wallets(id),
  currency text NOT NULL,
  delta_minor bigint NOT NULL CHECK (delta_minor <> 0),
  event_id text NOT NULL,
  created_at timestamptz NOT NULL DEFAULT clock_timestamp()
);
CREATE TABLE IF NOT EXISTS outbox (
  id text PRIMARY KEY,
  aggregate_id text NOT NULL,
  routing_key text NOT NULL,
  payload jsonb NOT NULL,
  created_at timestamptz NOT NULL DEFAULT clock_timestamp(),
  sent_at timestamptz,
  attempts integer NOT NULL DEFAULT 0,
  claim_token text,
  claimed_at timestamptz,
  next_attempt_at timestamptz NOT NULL DEFAULT clock_timestamp(),
  last_error text,
  parked_at timestamptz
);
CREATE INDEX IF NOT EXISTS outbox_pending_idx
ON outbox (next_attempt_at, created_at, id)
WHERE sent_at IS NULL AND parked_at IS NULL;
CREATE TABLE IF NOT EXISTS replay_audit (
  id bigserial PRIMARY KEY,
  event_id text NOT NULL,
  operator_name text NOT NULL,
  reason text NOT NULL,
  outcome text NOT NULL,
  created_at timestamptz NOT NULL DEFAULT clock_timestamp()
);
CREATE TABLE IF NOT EXISTS saga_steps (
  saga_id text NOT NULL,
  step text NOT NULL,
  status text NOT NULL,
  result jsonb NOT NULL DEFAULT '{}',
  updated_at timestamptz NOT NULL DEFAULT clock_timestamp(),
  PRIMARY KEY (saga_id, step)
);
SQL
docker compose -p be-mq exec -T postgres psql -v ON_ERROR_STOP=1 -U lab -d messaging < app/schema.sql
docker compose -p be-mq exec -T app node --check lib/common.mjs
```

**Vì sao transaction phải dùng cùng một client?** Một pool phân phối các lệnh độc lập tới nhiều connection. Nếu `BEGIN` chạy trên connection A nhưng `UPDATE` chạy trên B, bạn không có transaction như tưởng tượng. Helper `tx` lấy một client, giữ đến commit/rollback rồi release. [node-postgres transactions](https://node-postgres.com/features/transactions) mô tả yêu cầu này.

**Vì sao định danh publish khác định danh event?** Một sự kiện `evt-123` có thể được gửi ba lần do mất confirm. Consumer vẫn phải nhận `eventId=evt-123` để chống tác dụng phụ trùng. Trong khi đó, producer cần biết thông điệp bị `return` thuộc lần publish thứ nhất hay thứ ba. Header `x-publish-attempt` đáp ứng nhu cầu thứ hai; không dùng nó làm khóa idempotency nghiệp vụ.

Helper giới hạn timeout và gắn listener theo lần gửi, nhưng không có pool giới hạn vô hạn số lần gọi đồng thời. Các scripts dưới đây gọi tuần tự hoặc với số lượng nhỏ biết trước. Nếu đưa helper vào server thật, bổ sung hàng đợi nội bộ có dung lượng hữu hạn, số confirms outstanding tối đa, metrics và chiến lược reconnect. Không tạo hàng triệu Promise rồi mong boolean `false` tự bảo vệ RAM.

## Lab 04 — Đo “accepted” và “completed” bằng queue thật

**Mục tiêu:** nhìn thấy sự khác nhau giữa API trả nhanh và tác vụ hoàn tất. Bài dùng tác vụ I/O giả lập có thời gian cố ý; phần enqueue và worker là BullMQ thật. Không diễn giải thời gian sleep thành benchmark gửi email.

```bash {"name":"mq-04-acceptance-latency","excludeFromRunAll":"true"}
set -euo pipefail
LAB="$HOME/backend-labs/messaging"
test -f "$LAB/.be-mq-lab"
cd "$LAB"
cat > app/labs/04-latency.mjs <<'JS'
import assert from 'node:assert/strict';
import { performance } from 'node:perf_hooks';
import { bullLab, sleep } from '../lib/common.mjs';
const lab = await bullLab('latency');
try {
  lab.spawn(async job => {
    await sleep(1200);
    return { artifact: 'demo-report-' + job.data.reportId + '.pdf' };
  });
  const start = performance.now();
  const job = await lab.queue.add('render-report', { reportId: 1 });
  const acceptedMs = performance.now() - start;
  const result = await job.waitUntilFinished(lab.events, 15000);
  const completedMs = performance.now() - start;
  console.log({ acceptedMs, completedMs, result });
  assert.ok(completedMs >= 1100);
  assert.ok(completedMs > acceptedMs);
} finally {
  await lab.close();
}
JS
docker compose -p be-mq exec -T app node labs/04-latency.mjs | tee evidence/04-latency.txt
```

**Kết quả cần kiểm chứng:** `completedMs` khoảng trên 1.200 ms; `acceptedMs` thấp hơn trong môi trường bình thường. Không ép ngưỡng accepted bằng 5 ms: máy vừa khởi động, filesystem, Redis và tải cạnh tranh thay đổi kết quả.

**Phân tích:** nếu worker dừng hẳn, enqueue vẫn có thể thành công nhưng completed không xảy ra. API cần endpoint trạng thái hoặc callback và phải nói thật với người dùng. Nếu tác vụ không thể hoàn tất trước deadline, status nên cho biết thất bại, hết hạn hoặc cần thử lại, thay vì giữ `processing` mãi.

**Bài tập mở rộng:** tạo 100 jobs trước khi worker bắt đầu; đo accepted của job cuối và thời điểm completed của nó. Giải thích vì sao những số đó đi theo hai đường khác nhau. Đừng sửa thành concurrency 100 chỉ để bài chạy nhanh: trước tiên hãy tính provider downstream và pool DB chịu được bao nhiêu.

---

# Phần III. BullMQ: xây dựng background jobs có thể phục hồi

## 5. Vòng đời không phải một chiếc hộp FIFO đơn giản

Trong baseline v5, job có thể chờ ở `waiting`, `prioritized`, `delayed` hoặc `waiting-children`, được chuyển sang `active`, rồi `completed` hoặc `failed`. Retry có backoff lại đưa job về thời điểm chờ. Queue bị pause thêm một điều kiện không thể nhận việc. “Stalled” là một sự kiện phát hiện worker không giữ lock đúng cách, không phải một nơi lưu job vĩnh viễn.

Worker lấy job cùng lock có thời hạn. Trong khi processor đang chạy, worker gia hạn lock. Nếu process bị kill hoặc event loop bị chặn lâu, lock có thể mất. Một worker khác có thể xử lý lại. Lock bảo vệ quyền sở hữu tạm thời trong broker; nó không làm thao tác SQL hay HTTP nằm trong cùng transaction với Redis.

BullMQ từ phiên bản 2 đã không cần lớp `QueueScheduler` kiểu cũ để xử lý delayed/stalled jobs. Với baseline này, thêm `QueueScheduler` theo bài viết cũ gây nhầm API. `Job Scheduler` hiện đại là cơ chế tạo job theo lịch, khác lớp cũ có tên gần giống. Xem [QueueScheduler migration note](https://docs.bullmq.io/guide/queuescheduler).

## Lab 05 — Progress, kết quả và trạng thái cuối

**Mục tiêu:** không kết luận một job hoàn tất chỉ vì đã đợi một thời gian cố định.

```bash {"name":"mq-05-job-lifecycle","excludeFromRunAll":"true"}
set -euo pipefail
LAB="$HOME/backend-labs/messaging"
test -f "$LAB/.be-mq-lab"
cd "$LAB"
cat > app/labs/05-lifecycle.mjs <<'JS'
import assert from 'node:assert/strict';
import { bullLab, sleep } from '../lib/common.mjs';
const lab = await bullLab('lifecycle');
try {
  lab.events.on('progress', ({ jobId, data }) => console.log('PROGRESS', jobId, data));
  lab.spawn(async job => {
    for (const percent of [10, 40, 70, 100]) {
      await job.updateProgress({ percent, stage: percent < 100 ? 'render' : 'uploaded' });
      await sleep(80);
    }
    return { objectKey: 'reports/' + job.data.reportId + '/v1.pdf', bytes: 4096 };
  });
  const job = await lab.queue.add('export', { reportId: 'R-001' });
  const result = await job.waitUntilFinished(lab.events, 15000);
  const state = await job.getState();
  console.log({ state, result });
  assert.equal(state, 'completed');
  assert.equal(result.bytes, 4096);
} finally {
  await lab.close();
}
JS
docker compose -p be-mq exec -T app node labs/05-lifecycle.mjs | tee evidence/05-lifecycle.txt
```

**Phân tích:** progress là thông tin hỗ trợ UX và quan sát. Nó không thay thế một bản ghi trạng thái nghiệp vụ bền vững. Nếu progress báo 100 rồi upload hoàn tất bị lỗi, bạn cần xác định 100 có nghĩa “render xong” hay “toàn bộ công việc xong”. Tên stage giúp tránh mơ hồ.

Trả về `objectKey` ổn định tốt hơn URL ký có thời hạn. Khi người dùng tải, API kiểm tra quyền và tạo URL mới. Không ghi token truy cập, mật khẩu hay dữ liệu cá nhân vào progress vì chúng có thể được lưu trong Redis và stream sự kiện.

**Tự kiểm tra:** tại sao không dùng `await sleep(2000)` để kết thúc demo? Vì máy chậm hoặc retry dài hơn sẽ làm chương trình đóng kết nối trước khi đạt điều kiện. `waitUntilFinished` với deadline trả lỗi rõ ràng khi tiến trình không đạt trạng thái mong muốn.

## Lab 06 — Concurrency cho I/O và giới hạn tài nguyên chung

**Mục tiêu:** đo mức song song thực tế; hiểu tổng concurrency bằng tổng workers, không chỉ giá trị của một process.

```bash {"name":"mq-06-concurrency","excludeFromRunAll":"true"}
set -euo pipefail
LAB="$HOME/backend-labs/messaging"
test -f "$LAB/.be-mq-lab"
cd "$LAB"
cat > app/labs/06-concurrency.mjs <<'JS'
import assert from 'node:assert/strict';
import { performance } from 'node:perf_hooks';
import { bullLab, sleep } from '../lib/common.mjs';
for (const concurrency of [1, 4]) {
  const lab = await bullLab('concurrency');
  let active = 0;
  let maxActive = 0;
  try {
    lab.spawn(async () => {
      active++;
      maxActive = Math.max(maxActive, active);
      try { await sleep(150); return 'done'; }
      finally { active--; }
    }, { concurrency });
    const start = performance.now();
    const jobs = await lab.queue.addBulk(Array.from({ length: 12 }, (_, index) => ({
      name: 'io-task', data: { index }
    })));
    await Promise.all(jobs.map(job => job.waitUntilFinished(lab.events, 15000)));
    console.log({ concurrency, maxActive, totalMs: performance.now() - start });
    assert.ok(maxActive <= concurrency);
    if (concurrency === 4) assert.ok(maxActive > 1);
  } finally {
    await lab.close();
  }
}
JS
docker compose -p be-mq exec -T app node labs/06-concurrency.mjs | tee evidence/06-concurrency.txt
```

Với tác vụ chủ yếu đợi I/O, concurrency 4 thường rút thời gian tổng so với 1. Nó không tạo bốn JavaScript threads trong cùng event loop. Nếu processor thực hiện bốn phép nén CPU nặng đồng bộ, cả bốn vẫn cạnh tranh cùng event loop; lock renewal và metrics có thể bị chậm.

**Bài toán capacity:** hai replicas, mỗi replica concurrency 20, có thể tạo 40 requests đang thực hiện. Nếu mỗi job mở hai query SQL đồng thời, số query chờ pool có thể lên tới 80. Pool nhỏ giữ DB an toàn hơn nhưng kéo dài service time, tăng backlog. Tăng pool tiếp tục có thể đẩy DB vào quá tải. Mục tiêu là tối ưu throughput ở mức latency ổn định, không tối đa mọi con số.

**Chiến lược thiết kế:** đặt concurrency riêng theo loại việc, pool DB có ngân sách, timeout từng I/O, payload limit và rate limit theo provider. Nếu cần không quá một job toàn queue đang active, xem cơ chế global concurrency của baseline; một worker `concurrency: 1` vẫn không chặn replica khác chạy. [BullMQ workers concurrency](https://docs.bullmq.io/guide/workers/concurrency) phân biệt mức worker và giới hạn toàn queue.



## Lab 07 — Retry có ngân sách, backoff và lỗi không nên thử lại

**Bài toán:** provider trả 503 hai lần rồi hoạt động trở lại; một yêu cầu khác sai dữ liệu ngay từ đầu. Hai loại lỗi không nên dùng cùng chính sách.

`attempts: 3` là tối đa ba lần thực thi do failure thông thường, gồm lần đầu và hai lần thử lại. Với backoff exponential có delay 200 ms, hai khoảng chờ danh nghĩa là 200 ms và 400 ms. Không có khoảng chờ thứ ba sau khi đã dùng hết ngân sách. Job stalled, manual retry và rate-limit cần hiểu theo semantics riêng; không suy diễn mọi bộ đếm đều đồng nghĩa số lần provider nhận request. [BullMQ retrying failing jobs](https://docs.bullmq.io/guide/retrying-failing-jobs) mô tả options và công thức backoff.

```bash {"name":"mq-07-retry-budget","excludeFromRunAll":"true"}
set -euo pipefail
LAB="$HOME/backend-labs/messaging"
test -f "$LAB/.be-mq-lab"
cd "$LAB"
cat > app/labs/07-retry.mjs <<'JS'
import assert from 'node:assert/strict';
import { UnrecoverableError } from 'bullmq';
import { bullLab } from '../lib/common.mjs';
const lab = await bullLab('retry');
const calls = new Map();
try {
  lab.spawn(async job => {
    const count = (calls.get(job.name) || 0) + 1;
    calls.set(job.name, count);
    console.log('ATTEMPT', job.name, count, new Date().toISOString());
    if (job.name === 'invalid') throw new UnrecoverableError('INVALID_EMAIL_FORMAT');
    if (count < 3) throw new Error('UPSTREAM_503_SIMULATED');
    return { providerStatus: 'accepted', callCount: count };
  }, { concurrency: 2 });
  const retry = await lab.queue.add('transient', {}, {
    attempts: 3, backoff: { type: 'exponential', delay: 200, jitter: 0.25 }
  });
  const invalid = await lab.queue.add('invalid', {}, {
    attempts: 5, backoff: { type: 'exponential', delay: 200 }
  });
  const results = await Promise.allSettled([
    retry.waitUntilFinished(lab.events, 15000),
    invalid.waitUntilFinished(lab.events, 15000)
  ]);
  console.log(results);
  assert.equal(await retry.getState(), 'completed');
  assert.equal(await invalid.getState(), 'failed');
  assert.equal(calls.get('transient'), 3);
  assert.equal(calls.get('invalid'), 1);
} finally {
  await lab.close();
}
JS
docker compose -p be-mq exec -T app node labs/07-retry.mjs | tee evidence/07-retry.txt
```

**Kết quả:** `transient` chạy ba lần, `invalid` chỉ một lần. Jitter khiến thời gian thực không trùng hoàn toàn công thức danh nghĩa. Nếu 10.000 jobs lỗi cùng lúc và tất cả thử lại đúng một thời điểm, chúng có thể làm provider vừa phục hồi lại sập. Jitter rải thời điểm; rate limit giới hạn nhịp; circuit breaker giảm thử lại khi đã biết dependency không sẵn sàng. Ba cơ chế giải quyết ba vấn đề khác nhau.

### Bảng phân loại lỗi để dùng khi thiết kế worker

| Tình huống | Quyết định ban đầu | Điều cần biết thêm |
|---|---|---|
| Payload không hợp lệ | Quarantine, không retry tự động | Lưu đủ metadata để sửa producer |
| 503 hoặc mất kết nối trước khi gửi request | Retry hữu hạn và backoff | Có thể áp dụng circuit breaker |
| Timeout sau khi đã gửi request thu tiền | Trạng thái chưa biết | Hỏi provider bằng idempotency key trước khi quyết định |
| 401 do secret sai | Dừng nhóm worker và cảnh báo | Retry từng message không sửa được cấu hình |
| 429 | Tôn trọng Retry-After và rate limit | Phân biệt quota giây, phút, ngày |
| Tài khoản không đủ tiền | Kết quả nghiệp vụ | Không coi là lỗi hạ tầng để thử lại vô hạn |
| DB deadlock hoặc serialization failure | Retry transaction ngắn, hữu hạn | Giữ idempotency; quan sát tỷ lệ |
| Object nguồn đã bị xóa vĩnh viễn | Kết thúc/compensate theo hợp đồng | Đừng lặp một URL hỏng hàng nghìn lần |

Timeout là tình huống dễ bị xử lý sai nhất. “Client không nhận response” không chứng minh server chưa thực hiện. Nếu gọi provider thêm lần nữa với một key mới, retry có thể tạo tác dụng phụ mới. Một queue tốt chỉ giữ nghĩa vụ làm việc; nó không biết ngân hàng đã thu tiền hay chưa.

## Lab 08 — Priority, fairness và rate limit qua nhiều workers

**Mục tiêu:** thấy thứ tự lấy việc khác nhau và quan sát giới hạn nhịp dùng chung. Bài có hai pha độc lập, mỗi pha tự dọn queue riêng.

Trong BullMQ, số priority nhỏ hơn có ưu tiên cao hơn trong tập prioritized. Job không có priority, tương đương 0, được xử lý trước các job gán priority. Vì vậy không nên chỉ gán `priority: 1` cho VIP rồi để mọi job thường thiếu trường này. Hãy gán một quy ước nhất quán, chẳng hạn 1 cho ưu tiên và 10 cho thông thường. Xem [BullMQ prioritized jobs](https://docs.bullmq.io/guide/jobs/prioritized).

```bash {"name":"mq-08-priority-rate-limit","excludeFromRunAll":"true"}
set -euo pipefail
LAB="$HOME/backend-labs/messaging"
test -f "$LAB/.be-mq-lab"
cd "$LAB"
cat > app/labs/08-priority-rate.mjs <<'JS'
import assert from 'node:assert/strict';
import { bullLab, sleep } from '../lib/common.mjs';
const priorityLab = await bullLab('priority');
try {
  await priorityLab.queue.pause();
  const order = [];
  const jobs = [
    await priorityLab.queue.add('normal', {}, { priority: 10 }),
    await priorityLab.queue.add('urgent', {}, { priority: 1 }),
    await priorityLab.queue.add('no-priority', {})
  ];
  priorityLab.spawn(async job => { order.push(job.name); return job.name; });
  await priorityLab.queue.resume();
  await Promise.all(jobs.map(job => job.waitUntilFinished(priorityLab.events, 15000)));
  console.log('START_ORDER', order);
  assert.deepEqual(order, ['no-priority', 'urgent', 'normal']);
} finally {
  await priorityLab.close();
}
const rateLab = await bullLab('rate');
try {
  const starts = [];
  const processor = async job => {
    starts.push(Date.now());
    console.log('START', job.id, new Date().toISOString());
    await sleep(30);
    return 'ok';
  };
  rateLab.spawn(processor, { concurrency: 2, limiter: { max: 2, duration: 500 } });
  rateLab.spawn(processor, { concurrency: 2, limiter: { max: 2, duration: 500 } });
  const jobs = await rateLab.queue.addBulk(Array.from({ length: 6 }, (_, n) => ({
    name: 'provider-call', data: { n }
  })));
  await Promise.all(jobs.map(job => job.waitUntilFinished(rateLab.events, 15000)));
  const spreadMs = Math.max(...starts) - Math.min(...starts);
  console.log('START_SPREAD_MS', spreadMs);
  assert.ok(spreadMs >= 900);
} finally {
  await rateLab.close();
}
JS
docker compose -p be-mq exec -T app node labs/08-priority-rate.mjs | tee evidence/08-priority-rate.txt
```

**Kết quả:** thứ tự bắt đầu ở pha priority khớp assertion. Ở pha limiter, sáu lần bắt đầu trải trên khoảng một giây hoặc hơn dù có hai workers. Không suy ra rằng mọi cửa sổ trượt bất kỳ 500 ms đều có đúng cùng giới hạn; hãy kiểm tra semantics quota của provider. Nếu provider giới hạn theo tài khoản và có nhiều queue dùng cùng tài khoản, limiter trên riêng từng queue chưa bao phủ tổng tải.

**Vì sao priority không đủ cho SLA?** Một luồng ưu tiên đến liên tục có thể làm luồng thường chờ lâu. Với khách hàng hoặc loại công việc có hợp đồng khác nhau, queue riêng và ngân sách tài nguyên riêng thường dễ quản lý hơn một thang priority lớn. Ngược lại, tách queue quá nhiều có thể khiến worker của một queue rảnh trong khi queue khác tắc; cần biết mục đích cách ly là fairness, blast radius hay bảo vệ dependency.

**Bài tập:** dự đoán điều gì xảy ra nếu mỗi job gọi provider ba lần. Rate limit job 2/500 ms có thể dẫn đến sáu requests provider trong khoảng tương tự. Đơn vị giới hạn phải trùng đơn vị quota, hoặc processor phải tự quản lý quota cho từng request.

## Lab 09 — Hủy đơn quá hạn: DB quyết định, timer chỉ đánh thức

**Tình huống:** khách thanh toán gần lúc job hết hạn được worker nhận. Xóa job delay là một tối ưu; nó không phải bảo vệ tính đúng đắn vì job có thể đã active hoặc hai đường xử lý chạy đua.

**Bất biến:** đơn `PAID` không được đổi thành `EXPIRED`. Chuyển trạng thái phải là một câu UPDATE có điều kiện, trong cùng transaction với hoàn kho nếu có. Lab chỉ cập nhật đơn để tập trung vào cuộc đua.

```bash {"name":"mq-09-delayed-order-expiry","excludeFromRunAll":"true"}
set -euo pipefail
LAB="$HOME/backend-labs/messaging"
test -f "$LAB/.be-mq-lab"
cd "$LAB"
cat > app/labs/09-expiry.mjs <<'JS'
import assert from 'node:assert/strict';
import { bullLab, pool, tx, hash, randomUUID, sleep } from '../lib/common.mjs';
const lab = await bullLab('expiry');
const prefix = randomUUID();
const walletId = 'expiry-wallet-' + prefix;
const ids = ['paid-' + prefix, 'unpaid-' + prefix];
try {
  await tx(async client => {
    await client.query('INSERT INTO wallets(id,currency) VALUES($1,$2)', [walletId, 'VND']);
    for (const id of ids) {
      await client.query(
        "INSERT INTO orders(id,client_key,request_hash,wallet_id,currency,total_minor,reward_minor,status,expires_at) VALUES($1,$1,$2,$3,'VND',100000,1000,'PENDING',clock_timestamp()+interval '800 milliseconds')",
        [id, hash({ id }), walletId]
      );
    }
  });
  lab.spawn(async job => {
    const result = await pool.query(
      "UPDATE orders SET status='EXPIRED' WHERE id=$1 AND status='PENDING' AND expires_at<=clock_timestamp() RETURNING id",
      [job.data.orderId]
    );
    return { expired: result.rowCount === 1 };
  });
  const jobs = [];
  for (const id of ids) {
    jobs.push(await lab.queue.add('expire', { orderId: id }, {
      delay: 1000, jobId: 'expire-' + id
    }));
  }
  await sleep(100);
  await pool.query("UPDATE orders SET status='PAID' WHERE id=$1 AND status='PENDING'", [ids[0]]);
  const results = await Promise.all(jobs.map(job => job.waitUntilFinished(lab.events, 15000)));
  const states = await pool.query('SELECT id,status FROM orders WHERE id=ANY($1::text[]) ORDER BY id', [ids]);
  console.log({ results, rows: states.rows });
  assert.equal(results[0].expired, false);
  assert.equal(results[1].expired, true);
} finally {
  await lab.close();
  await pool.end();
}
JS
docker compose -p be-mq exec -T app node labs/09-expiry.mjs | tee evidence/09-expiry.txt
```

**Giải thích:** worker không dựa vào tên job để suy ra đơn chưa thanh toán. UPDATE kiểm tra trạng thái hiện tại dưới quyền quản lý concurrency của DB. Nếu logic hoàn kho đi kèm, chỉ thực hiện khi UPDATE trả một dòng và trong cùng transaction. Nếu dùng hai lệnh rời rạc, process có thể đổi status nhưng chưa hoàn kho.

Delayed job được xử lý sau khi đủ điều kiện về thời gian và có worker sẵn sàng; không phải cam kết chạy đúng từng mili giây. [BullMQ delayed jobs](https://docs.bullmq.io/guide/jobs/delayed) nêu rõ độ trễ phụ thuộc tải. Một scanner SQL có index phù hợp không phải thiết kế “sai” mặc định. Scanner có thể là tuyến reconciliation rất có ích khi Redis mất dữ liệu hoặc lịch chưa được tạo.

**Thiết kế thực tế:** tạo đơn cùng outbox “hãy lập lịch kiểm tra hết hạn”; relay thêm delayed job với ID ổn định. Một job đến sớm phải nhìn `expires_at` và lập lịch lại hoặc để scanner xử lý; một job đến muộn vẫn kiểm tra trạng thái. Với stock reservation, thêm ID reservation và trạng thái để việc release có thể thử lại mà không cộng tồn kho hai lần.

## Lab 10 — Job Scheduler, nhiều instance, timezone và overlap

**Mục tiêu:** tạo một định nghĩa lịch bằng ID ổn định, quan sát số lượt chạy, rồi xóa lịch. Không dùng API `repeat` cũ cho thiết kế mới.

Scheduler tạo job khi tiến trình xử lý lịch tiến lên; nó không cam kết rằng task chạy chính xác vào mọi mốc dù hệ thống bận hoặc ngừng hoạt động. Một lịch tồn tại duy nhất cũng không có nghĩa side effect thực hiện duy nhất, hoặc hai lượt không chồng lên nhau. [Job Schedulers](https://docs.bullmq.io/guide/job-schedulers/) giải thích quá trình tạo job mới khi job trước bắt đầu xử lý.

```bash {"name":"mq-10-job-scheduler","excludeFromRunAll":"true"}
set -euo pipefail
LAB="$HOME/backend-labs/messaging"
test -f "$LAB/.be-mq-lab"
cd "$LAB"
cat > app/labs/10-scheduler.mjs <<'JS'
import assert from 'node:assert/strict';
import { bullLab, sleep, until } from '../lib/common.mjs';
const lab = await bullLab('scheduler');
let runs = 0;
let active = 0;
let peak = 0;
try {
  await lab.queue.setGlobalConcurrency(1);
  for (let replica = 0; replica < 5; replica++) {
    await lab.queue.upsertJobScheduler('report-tick', {
      every: 300, limit: 3
    }, {
      name: 'report-tick',
      data: { report: 'daily-demo' },
      opts: { attempts: 2, backoff: { type: 'fixed', delay: 100 } }
    });
  }
  const definitions = await lab.queue.getJobSchedulers(0, -1, true);
  assert.equal(definitions.length, 1);
  lab.spawn(async job => {
    active++;
    peak = Math.max(peak, active);
    try {
      await sleep(180);
      runs++;
      console.log('SCHEDULE_RUN', job.id, runs);
      return { runs };
    } finally { active--; }
  }, { concurrency: 3 });
  await until(() => runs >= 3, 15000, 'three schedule runs');
  await lab.queue.upsertJobScheduler('weekly-vn', {
    pattern: '0 0 8 * * 0',
    tz: 'Asia/Ho_Chi_Minh'
  }, { name: 'weekly-summary', data: { timezone: 'Asia/Ho_Chi_Minh' } });
  console.log('SCHEDULES', await lab.queue.getJobSchedulers(0, -1, true));
  console.log({ runs, peak });
  assert.equal(peak, 1);
  await lab.queue.removeJobScheduler('weekly-vn');
  await lab.queue.removeJobScheduler('report-tick');
} finally {
  await lab.close();
}
JS
docker compose -p be-mq exec -T app node labs/10-scheduler.mjs | tee evidence/10-scheduler.txt
```

**Kết quả:** chỉ một định nghĩa `report-tick`, ba lượt hoàn thành và peak active bằng một do global concurrency. Lịch `weekly-vn` được inspect rồi xóa, không yêu cầu bạn đợi đến Chủ nhật. Cron có sáu trường trong ví dụ: giây, phút, giờ, ngày, tháng, thứ; chủ động đọc parser và timezone khi đổi biểu thức.

**Câu chuyện vận hành:** báo cáo “doanh thu ngày hôm qua” cần định danh business window, chẳng hạn `sales-VN-2026-11-16`. Nếu job bị chạy lại, unique key trên cửa sổ báo cáo ngăn phát hành hai bản chính. Nếu hệ thống ngừng hai ngày, lịch có thể không tự bù toàn bộ các ngày bị bỏ. Một bảng report_runs và scanner các cửa sổ chưa hoàn thành đáng tin hơn việc suy ra rằng cron đã gọi thì báo cáo tồn tại.

**Timezone:** lưu instant dưới UTC nhưng lưu cả timezone nghiệp vụ nếu đó là điều khách yêu cầu. Với khu vực có DST, “02:30 mỗi ngày” có ngày không tồn tại hoặc xuất hiện hai lần. Quyết định cần được ghi trong hợp đồng scheduler, không để thư viện chọn ngầm. Việc gửi mã giảm giá cũng cần unique key theo khách và chiến dịch, không chỉ scheduler ID.

## Lab 11 — Flows: phụ thuộc không thay thế transaction hoặc saga

**Mục tiêu:** ba bước con hoàn tất trước bước cha, kết quả được tổng hợp và kiểm tra.

```bash {"name":"mq-11-flow-dependencies","excludeFromRunAll":"true"}
set -euo pipefail
LAB="$HOME/backend-labs/messaging"
test -f "$LAB/.be-mq-lab"
cd "$LAB"
cat > app/labs/11-flow.mjs <<'JS'
import assert from 'node:assert/strict';
import { FlowProducer } from 'bullmq';
import { bullLab, connection, sleep } from '../lib/common.mjs';
const child = await bullLab('flow-child');
const parent = await bullLab('flow-parent');
const flow = new FlowProducer({ connection });
try {
  child.spawn(async job => {
    await sleep(job.data.waitMs);
    return { check: job.name, passed: true };
  }, { concurrency: 3 });
  parent.spawn(async job => {
    const values = Object.values(await job.getChildrenValues());
    if (values.length !== 3 || values.some(value => !value.passed)) {
      throw new Error('INCOMPLETE_CHECKS');
    }
    return { approved: true, checks: values.length };
  });
  const tree = await flow.add({
    name: 'assemble-report',
    queueName: parent.name,
    data: { customerId: 'synthetic-customer' },
    children: [
      { name: 'identity', queueName: child.name, data: { waitMs: 200 }, opts: { failParentOnFailure: true } },
      { name: 'documents', queueName: child.name, data: { waitMs: 100 }, opts: { failParentOnFailure: true } },
      { name: 'policy', queueName: child.name, data: { waitMs: 150 }, opts: { failParentOnFailure: true } }
    ]
  });
  const result = await tree.job.waitUntilFinished(parent.events, 15000);
  console.log(result);
  assert.equal(result.checks, 3);
} finally {
  await flow.close();
  await child.close();
  await parent.close();
}
JS
docker compose -p be-mq exec -T app node labs/11-flow.mjs | tee evidence/11-flow.txt
```

Một flow tổ chức phụ thuộc công việc; nó không rollback dịch vụ ngoài. Nếu bước con A đã gửi một email, bước B thất bại không thu hồi được email đó. Với bước có tác dụng phụ, hãy lưu trạng thái và định nghĩa hành động bù hợp lệ. Một hành động bù thường là thao tác mới, như phát hành biên bản sửa hoặc hoàn tiền, chứ không “quay ngược thời gian”.

Đừng chuyển việc phê duyệt thật sang `approved: true` chỉ vì ba jobs hoàn tất. `completed` có thể nghĩa “đã gọi được API”, trong khi dữ liệu trả về cho biết không đạt điều kiện. Processor cha phải đọc kết quả nghiệp vụ; ví dụ deliberately kiểm tra `passed`. Các options khi con thất bại có semantics khác nhau và phải chọn theo workflow. [BullMQ Flows](https://docs.bullmq.io/guide/flows/) là tài liệu tham chiếu cho ràng buộc cha–con.

**Bài tập:** cho `policy` ném lỗi và quan sát trạng thái của parent. Sau đó thử phân biệt “kiểm tra không đạt” bằng kết quả `passed: false` với “không thể thực hiện kiểm tra” bằng failure. Tại sao hai điều này cần dashboard và quy trình xử lý khác nhau?

## Lab 12 — Job ID, deduplication và thời gian lưu dữ liệu

**Mục tiêu:** chứng minh việc chặn enqueue trùng không phải idempotency lâu dài của nghiệp vụ.

```bash {"name":"mq-12-dedupe-retention","excludeFromRunAll":"true"}
set -euo pipefail
LAB="$HOME/backend-labs/messaging"
test -f "$LAB/.be-mq-lab"
cd "$LAB"
cat > app/labs/12-dedupe.mjs <<'JS'
import assert from 'node:assert/strict';
import { bullLab } from '../lib/common.mjs';
const lab = await bullLab('dedupe');
let sideEffects = 0;
try {
  const first = await lab.queue.add('invoice', { orderId: 'A' }, { jobId: 'invoice-A-v1' });
  const duplicate = await lab.queue.add('invoice', { orderId: 'A' }, { jobId: 'invoice-A-v1' });
  assert.equal(first.id, duplicate.id);
  assert.equal(await lab.queue.getWaitingCount(), 1);
  lab.spawn(async () => { sideEffects++; return { sideEffects }; });
  await first.waitUntilFinished(lab.events, 15000);
  await first.remove();
  const afterRemoval = await lab.queue.add('invoice', { orderId: 'A' }, { jobId: 'invoice-A-v1' });
  await afterRemoval.waitUntilFinished(lab.events, 15000);
  console.log({ sideEffects, explanation: 'Same business key admitted after job removal' });
  assert.equal(sideEffects, 2);
} finally {
  await lab.close();
}
const refresh = await bullLab('dedupe-simple');
try {
  const a = await refresh.queue.add('refresh-profile', { customerId: 42 }, {
    deduplication: { id: 'customer-42' }
  });
  const b = await refresh.queue.add('refresh-profile', { customerId: 42 }, {
    deduplication: { id: 'customer-42' }
  });
  assert.equal(a.id, b.id);
  refresh.spawn(async job => ({ customerId: job.data.customerId }));
  await a.waitUntilFinished(refresh.events, 15000);
  console.log('SIMPLE_DEDUPLICATION_OK', a.id, b.id);
} finally {
  await refresh.close();
}
JS
docker compose -p be-mq exec -T app node labs/12-dedupe.mjs | tee evidence/12-dedupe.txt
```

**Kết quả:** ở pha đầu biến `sideEffects` bằng hai. Queue không “bị hỏng”; job cũ đã bị xóa nên khóa dựa vào job ID không còn tồn tại. Một email hoặc khoản cộng tiền thật cần bảo vệ bằng dữ liệu nghiệp vụ hoặc idempotency key do provider hỗ trợ.

BullMQ có những chế độ deduplication dành cho các mục đích như bỏ yêu cầu khi công việc tương đương đang pending hoặc gộp một chuỗi cập nhật. Chúng giúp giảm tải. Phải chọn ID, thời gian và điều kiện hết bảo vệ phù hợp, không gọi mọi chế độ là “exactly once”. Xem [BullMQ deduplication](https://docs.bullmq.io/guide/jobs/deduplication).

Retention có đánh đổi: giữ quá lâu tăng Redis RAM; xóa quá sớm làm mất evidence chẩn đoán. Cấu hình `age/count` được dọn theo hoạt động hoàn thành/failure, không phải một daemon chạy đúng thời điểm hết tuổi của mọi job. Vì vậy queue yên lặng có thể giữ job cũ hơn age cho đến lần hoạt động tiếp theo. [BullMQ auto-removal](https://docs.bullmq.io/guide/queues/auto-removal-of-jobs) cũng nêu ảnh hưởng của job removal tới chống trùng.

**Thiết kế khóa:** `invoice-order123-template4` mô tả một kết quả có thể tái sử dụng. `Date.now()` tạo key mới mỗi lần retry, không chống trùng. Chỉ dùng `order123` lại có thể chặn một lần xuất hợp lệ với template mới. Trong v5, tránh dấu hai chấm trong custom job ID; dùng một encoding hoặc hash rõ ràng và lưu trường nghiệp vụ riêng để tra cứu.

## Lab 13 — Event loop bị chặn, lock hết hạn và sandboxed processor

**Đây là bài gây lỗi có chủ đích.** Vòng lặp CPU chỉ chạy khoảng 1,2 giây trong container app của lab. Không chạy đoạn spin này trong API thật. Các ngưỡng lock dưới đây rất ngắn để quan sát; không dùng chúng như cấu hình production.

**Mục tiêu:** phân biệt “worker đang chạy code” với “worker vẫn chứng minh được quyền sở hữu job”.

```bash {"name":"mq-13-stalled-and-sandbox","excludeFromRunAll":"true"}
set -euo pipefail
LAB="$HOME/backend-labs/messaging"
test -f "$LAB/.be-mq-lab"
cd "$LAB"
cat > app/labs/13-cpu.cjs <<'JS'
module.exports = async function (job) {
  const end = Date.now() + job.data.ms;
  while (Date.now() < end) Math.sqrt(12345);
  return { isolated: true };
};
JS
cat > app/labs/13-stall.mjs <<'JS'
import assert from 'node:assert/strict';
import { fileURLToPath } from 'node:url';
import { bullLab, sleep } from '../lib/common.mjs';
const blocked = await bullLab('event-loop-blocked');
let executions = 0;
try {
  blocked.events.on('stalled', event => console.log('STALLED', event));
  blocked.spawn(async () => {
    executions++;
    console.log('PROCESSOR_ENTER', executions);
    if (executions === 1) {
      const end = Date.now() + 1200;
      while (Date.now() < end) Math.sqrt(12345);
    } else {
      await sleep(20);
    }
    return { executions };
  }, { lockDuration: 400, stalledInterval: 500, maxStalledCount: 1 });
  const job = await blocked.queue.add('cpu', {});
  const outcome = await job.waitUntilFinished(blocked.events, 20000);
  console.log('BLOCKED_OUTCOME', outcome);
  assert.ok(executions >= 2);
} finally {
  await blocked.close();
}
const isolated = await bullLab('cpu-isolated');
try {
  const processor = fileURLToPath(new URL('./13-cpu.cjs', import.meta.url));
  isolated.spawn(processor, { lockDuration: 2000, stalledInterval: 1000 });
  const job = await isolated.queue.add('cpu', { ms: 1200 });
  const result = await job.waitUntilFinished(isolated.events, 20000);
  console.log('ISOLATED_OUTCOME', result);
  assert.equal(result.isolated, true);
} finally {
  await isolated.close();
}
JS
docker compose -p be-mq exec -T app node labs/13-stall.mjs | tee evidence/13-stall.txt
```

**Kết quả mong đợi:** pha đầu xuất hiện thông báo mất lock hoặc `stalled` và processor được vào lại; pha sau hoàn tất bằng tiến trình xử lý riêng. Đây là kiểm chứng cơ chế trên môi trường của bạn, không phải thước đo hiệu năng sandbox. Trên máy đang quá tải, deadline 20 giây có thể thất bại; đọc logs và CPU, không chỉ nới mọi timeout.

Gia hạn lock cần event loop được lên lịch và Redis có thể truy cập. Một thao tác CPU đồng bộ dài, GC pause, mạng chậm hoặc Redis ngừng đều có thể phá điều kiện đó. [BullMQ stalled jobs](https://docs.bullmq.io/guide/jobs/stalled) giải thích lý do nên tránh chặn event loop. Với xử lý ảnh, nén video hoặc parse dữ liệu lớn, dùng child process/worker thread thích hợp, hạn chế concurrency theo RAM và CPU, và lưu checkpoint nếu thời gian thực thi dài.

**Điều cần rút ra:** thời gian khóa không phải giao dịch nghiệp vụ. Dù tăng lockDuration lên một giờ, process vẫn có thể chết sau khi side effect đã xong. Bạn vẫn cần idempotency. Tăng khóa quá lớn còn kéo dài thời gian phát hiện job bị bỏ rơi khi worker thật sự chết.

## Lab 14 — Redis gián đoạn: producer phải có deadline và đường phục hồi

**Mục tiêu:** producer không treo vô hạn khi Redis không khả dụng. Bài chỉ dừng service Redis thuộc project `be-mq` rồi tự khởi động lại bằng trap. Không có thao tác lên Redis cache hay dịch vụ khác.

**Tiền điều kiện:** chạy các bài trước xong, không có worker tương tác đang xử lý. Mọi job trong chương đều là dữ liệu demo.

```bash {"name":"mq-14-redis-outage","excludeFromRunAll":"true"}
set -euo pipefail
LAB="$HOME/backend-labs/messaging"
test -f "$LAB/.be-mq-lab"
cd "$LAB"
cat > app/labs/14-producer-offline.mjs <<'JS'
import { Queue } from 'bullmq';
const watchdog = setTimeout(() => {
  console.error('PRODUCER_DEADLINE_EXCEEDED: request outcome must be treated as unknown');
  process.exit(2);
}, 6000);
const queue = new Queue('mq-offline-probe', {
  connection: {
    host: process.env.REDIS_HOST,
    port: Number(process.env.REDIS_PORT),
    connectTimeout: 1000,
    maxRetriesPerRequest: 1,
    enableOfflineQueue: false,
    retryStrategy: () => null
  }
});
queue.on('error', error => console.log('EXPECTED_CONNECTION_ERROR', error.message));
try {
  await queue.add('probe', { synthetic: true });
  console.error('Unexpected acceptance: Redis may not be stopped');
  clearTimeout(watchdog);
  process.exit(1);
} catch (error) {
  console.log('REQUEST_REJECTED', error.message);
  clearTimeout(watchdog);
  // Demo producer ngắn, chưa có công việc nghiệp vụ được chấp nhận.
  process.exit(0);
}
JS
restore_redis() {
  docker compose -p be-mq start redis >/dev/null
}
trap restore_redis EXIT INT TERM
docker compose -p be-mq stop redis
set +e
docker compose -p be-mq exec -T app node labs/14-producer-offline.mjs
probe_status=$?
set -e
restore_redis
trap - EXIT INT TERM
docker compose -p be-mq up -d --wait --wait-timeout 90 redis
if [ "$probe_status" -ne 0 ]; then
  printf '%s\n' "Probe status=$probe_status; đọc kết quả, không xem đây là enqueue thành công."
fi
docker compose -p be-mq exec -T redis redis-cli PING
docker compose -p be-mq exec -T app node labs/04-latency.mjs
```

Nếu probe bị deadline cắt, đó vẫn là kết quả cần phân tích: lớp gọi phải đưa ra trạng thái `unknown` hoặc lỗi có thể retry bằng cùng idempotency key, không trả “đã xử lý”. Lệnh sau phục hồi kiểm tra Redis thật sự trả PONG và chạy lại một job thành công.

### Fail fast không có nghĩa bỏ nghĩa vụ

Một API chỉ tạo báo cáo có thể trả 503 khi enqueue chưa được ghi ở đâu và cho client thử lại với cùng key. Một API đã commit đơn hàng không được trả 503 như thể đơn chưa tồn tại chỉ vì gửi thông báo thất bại. Trường hợp đó cần outbox: đơn hàng và nghĩa vụ phát sự kiện nằm trong một transaction, relay chạy lại sau khi queue trở lại.

Worker có nhu cầu khác: nó có thể chờ kết nối lại, nhưng cần liveness, cảnh báo mất kết nối kéo dài và đảm bảo side effect đã làm vẫn an toàn khi delivery lặp. Tách connection producer, worker và QueueEvents giúp chọn timeout phù hợp. [BullMQ connections](https://docs.bullmq.io/guide/connections) mô tả sự khác biệt giữa client phục vụ HTTP và client xử lý nền.

**Tự kiểm tra:** nếu Redis trả OOM do `noeviction`, khởi động lại Redis không tạo thêm bộ nhớ. Bạn cần biết backlog, lượng completed/failed giữ lại, payload size, AOF rewrite, maxmemory và RAM thực. Đổi sang `allkeys-lru` để request hết lỗi sẽ đánh đổi cấu trúc queue và làm lỗi khó phát hiện hơn.

---

# Phần IV. RabbitMQ: routing và hai hướng xác nhận

## 6. Exchange, queue, channel và connection

Publisher gửi bằng AMQP 0-9-1 vào một exchange. Exchange chọn tập đích bằng bindings; queue giữ message cho nhóm consumers. Một queue có nhiều consumers thường là competing consumers: mỗi delivery được giao cho một consumer đủ điều kiện. Muốn inventory, email và analytics cùng có bản sao, cần ba queues với bindings phù hợp, không chỉ ba consumers của một queue.

`sendToQueue(name, body)` là API hợp lệ. Nó publish vào **default exchange**, tên rỗng, với routing key bằng tên queue. Vì thế câu “publisher không bao giờ gửi thẳng tới queue” dễ gây hiểu sai: ở mức giao thức vẫn có exchange; ở mức API bạn hoàn toàn có thể nhắm tên queue. [RabbitMQ exchanges](https://www.rabbitmq.com/docs/exchanges) mô tả default exchange và các kiểu routing.

Connection là kết nối mạng có heartbeat, xác thực và trạng thái. Channel là phiên giao thức trên connection. Dùng connection lâu dài; không mở kết nối cho mỗi message. Tách channel của consumer khỏi publisher confirms để delivery tags và lifecycle rõ ràng. Tách connection publish/consume cũng có ích khi publisher bị broker block do tài nguyên, tránh coupling không mong muốn.

### Một event, nhiều subscribers

Khi thêm subscriber mới, queue mới thường chỉ nhận các message được route từ lúc binding tồn tại. Nó không tự nhận lại tất cả lịch sử mà publisher đã phát trước đó. Nếu cần bootstrap trạng thái, dùng snapshot, event archive, stream retention hoặc API backfill có version. Đó là yêu cầu dữ liệu, không phải checkbox “fanout”.

### Hai ACK có nghĩa khác nhau

| Tín hiệu | Người gửi → người nhận | Nó chứng minh điều gì? | Nó không chứng minh điều gì? |
|---|---|---|---|
| Publisher confirm | Broker → publisher | Broker đã xử lý publish theo semantics loại đích | Consumer đã commit DB hoặc gửi email |
| `basic.return` với mandatory | Broker → publisher | Message không route được tới queue phù hợp | Không có lỗi nghiệp vụ ở consumer |
| Consumer ACK | Consumer → broker | Consumer tuyên bố delivery đã xử lý | Mọi database/provider đã đúng nếu code ACK sai thời điểm |
| HTTP response provider | Provider → worker | Điều hợp đồng HTTP cụ thể nói | Không có response thì chưa thực hiện |
| DB COMMIT | Database → ứng dụng | Transaction được commit theo cấu hình DB | Broker đã nhận ACK |

Publisher confirm và consumer ACK độc lập. Một message không route tới queue vẫn có thể được confirm; vì vậy cần `mandatory` và xử lý `return` nếu việc route tới ít nhất một queue là yêu cầu. RabbitMQ gửi return trước confirm cho message mandatory không route được, nhưng ứng dụng vẫn phải ghép đúng lần publish. Xem [consumer acknowledgements and publisher confirms](https://www.rabbitmq.com/docs/confirms).

Độ bền cũng gồm nhiều điều kiện: queue durable, delivery mode persistent khi loại queue yêu cầu, confirms, volumes bền, replication nếu cần chịu mất node, backups và cách phục hồi nghiệp vụ. Từ “durable” không làm hỏng ổ đĩa hoặc xóa volume trở thành sự cố có thể tự chữa.

## Lab 15 — Chạy đủ direct, fanout, topic và headers

**Mục tiêu:** kiểm chứng luật routing bằng bảng kết quả và assertions, không chỉ nhìn một sơ đồ.

Các queue của bài có tên riêng cho mỗi lần chạy, durable quorum, rồi được xóa trong finally. Single-node quorum ở đây dùng để học semantics; nó không cho khả năng chịu mất chính VPS đó.

```bash {"name":"mq-15-four-exchange-types","excludeFromRunAll":"true"}
set -euo pipefail
LAB="$HOME/backend-labs/messaging"
test -f "$LAB/.be-mq-lab"
cd "$LAB"
cat > app/labs/15-exchanges.mjs <<'JS'
import assert from 'node:assert/strict';
import { rabbit, channel, quorum, confirmSend, randomUUID } from '../lib/common.mjs';
const conn = await rabbit();
const pub = await channel(conn, true);
const read = await channel(conn);
const id = randomUUID();
const queues = [];
const exchanges = [];
async function makeQueue(suffix, ex, key, args = {}) {
  const name = 'mq-route-' + id + '-' + suffix;
  await quorum(read, name);
  await read.bindQueue(name, ex, key, args);
  queues.push(name);
  return name;
}
async function takeAll(name) {
  const values = [];
  for (;;) {
    const msg = await read.get(name, { noAck: false });
    if (!msg) return values;
    values.push(msg.content.toString());
    read.ack(msg);
  }
}
try {
  const direct = 'mq-direct-' + id;
  const fanout = 'mq-fanout-' + id;
  const topic = 'mq-topic-' + id;
  const headers = 'mq-headers-' + id;
  for (const [name, type] of [[direct,'direct'],[fanout,'fanout'],[topic,'topic'],[headers,'headers']]) {
    await pub.assertExchange(name, type, { durable: true });
    exchanges.push(name);
  }
  const directQ = await makeQueue('direct', direct, 'invoice.ready');
  const fanA = await makeQueue('fan-a', fanout, '');
  const fanB = await makeQueue('fan-b', fanout, '');
  const vn = await makeQueue('vn', topic, 'order.*.vn');
  const all = await makeQueue('all', topic, 'order.#');
  const hdr = await makeQueue('headers', headers, '', { 'x-match': 'all', format: 'pdf', region: 'vn' });
  await confirmSend(pub, direct, 'invoice.ready', Buffer.from('direct-hit'));
  await confirmSend(pub, fanout, 'ignored-key', Buffer.from('fanout-hit'));
  await confirmSend(pub, topic, 'order.paid.vn', Buffer.from('vn-order'));
  await confirmSend(pub, topic, 'order.paid.us', Buffer.from('us-order'));
  await confirmSend(pub, headers, '', Buffer.from('headers-hit'), { headers: { format: 'pdf', region: 'vn' } });
  const result = {
    direct: await takeAll(directQ),
    fanA: await takeAll(fanA), fanB: await takeAll(fanB),
    vn: await takeAll(vn), all: await takeAll(all),
    headers: await takeAll(hdr)
  };
  console.log(result);
  assert.deepEqual(result.direct, ['direct-hit']);
  assert.deepEqual(result.fanA, ['fanout-hit']);
  assert.deepEqual(result.fanB, ['fanout-hit']);
  assert.deepEqual(result.vn, ['vn-order']);
  assert.deepEqual(result.all, ['vn-order','us-order']);
  assert.deepEqual(result.headers, ['headers-hit']);
} finally {
  for (const name of queues) await read.deleteQueue(name).catch(() => {});
  for (const name of exchanges) await pub.deleteExchange(name).catch(() => {});
  await conn.close().catch(() => {});
}
JS
docker compose -p be-mq exec -T app node labs/15-exchanges.mjs | tee evidence/15-exchanges.txt
```

**Giải thích:** direct khớp chuỗi routing key; fanout bỏ qua key; topic hiểu các từ phân tách bởi dấu chấm; headers xét cặp khóa/giá trị theo binding. Trong topic, `*` khớp một từ, `#` khớp không hoặc nhiều từ. `order.*.vn` không khớp `order.payment.paid.vn` vì có thêm một từ.

**Bài tập:** thêm binding `order.paid.#` và publish `order.paid`, `order.paid.vn`, `order.cancelled.vn`. Trước khi chạy, tự ghi tập queue dự kiến. Với headers, đổi `region` thành `VN` hoặc thành số để thấy hợp đồng kiểu dữ liệu quan trọng như tên trường.

Trong production, topology thường được quản lý bằng deploy/migration riêng. Nếu service có quyền tạo mọi exchange/queue, một lỗi chính tả có thể tạo queue mới rồi để message tồn đọng không ai đọc. Nếu chỉ dùng passive checks, service khởi động trước topology sẽ fail readiness rõ ràng. Chọn trách nhiệm sở hữu topology và quyền `configure/write/read` theo vai trò.

## Lab 16 — Publish thành công cục bộ nhưng không có đích

**Mục tiêu:** chứng minh rằng `persistent: true` và confirm riêng lẻ chưa trả lời “có queue nhận không”.

Bài dùng exchange có tồn tại nhưng không có binding cho key cần gửi. Gửi vào exchange không tồn tại là lỗi khác: broker có thể đóng channel. Sau lỗi channel, bạn phải tạo channel mới; không tiếp tục dùng đối tượng bị đóng như chưa có gì xảy ra.

```bash {"name":"mq-16-mandatory-return","excludeFromRunAll":"true"}
set -euo pipefail
LAB="$HOME/backend-labs/messaging"
test -f "$LAB/.be-mq-lab"
cd "$LAB"
cat > app/labs/16-mandatory.mjs <<'JS'
import assert from 'node:assert/strict';
import { rabbit, channel, confirmSend, randomUUID } from '../lib/common.mjs';
const conn = await rabbit();
const pub = await channel(conn, true);
const exchange = 'mq-unroutable-' + randomUUID();
try {
  await pub.assertExchange(exchange, 'direct', { durable: true });
  await new Promise((resolve, reject) => {
    pub.publish(exchange, 'missing', Buffer.from('no-mandatory'), {
      persistent: true
    }, error => error ? reject(error) : resolve());
  });
  console.log('BROKER_CONFIRMED_BUT_NO_QUEUE_RECEIVED');
  await assert.rejects(
    confirmSend(pub, exchange, 'missing', Buffer.from('{}'), { messageId: 'stable-event-id' }),
    /UNROUTABLE/
  );
  console.log('MANDATORY_RETURN_DETECTED');
} finally {
  await pub.deleteExchange(exchange).catch(() => {});
  await conn.close().catch(() => {});
}
JS
docker compose -p be-mq exec -T app node labs/16-mandatory.mjs | tee evidence/16-mandatory.txt
```

**Kết quả:** lần đầu vẫn confirm, lần hai helper báo `UNROUTABLE`. Không có consumer nào trong bài vì câu hỏi đang ở đoạn publisher → broker.

Một outbox relay gặp return phải giữ event chưa sent, ghi lỗi routing và cảnh báo. Tự động đánh dấu sent chỉ vì callback confirm không có error làm mất nghĩa vụ nghiệp vụ. Nhưng nếu publisher timeout chờ confirm, cũng không được kết luận rằng broker chưa nhận. Giữ event để thử lại với cùng event ID và chuẩn bị consumer chịu duplicate. [RabbitMQ reliability guide](https://www.rabbitmq.com/docs/reliability) giải thích cửa sổ mất xác nhận và khả năng gửi lại bản sao.

### Giới hạn của mandatory

Mandatory trả lời “định tuyến tới ít nhất một queue”, không trả lời “đã tới tất cả subscribers mà đội nghiệp vụ mong muốn”. Nếu audit queue còn binding nhưng rewards queue bị tháo binding, publish vẫn có thể thành công. Cần topology checks, consumer readiness, outbox-to-business reconciliation và alert theo lag của từng chức năng. Không dùng một cờ protocol để thay thế kiểm chứng hợp đồng toàn hệ thống.

## Lab 17 — Worker biến mất trước ACK và delivery được giao lại

**Mục tiêu:** kiểm tra trạng thái redelivered, ACK cùng channel và sự khác nhau giữa delivery với side effect. Bài chưa thao tác tiền; phần PostgreSQL tiếp theo sẽ chứng minh side effect.

```bash {"name":"mq-17-redelivery-before-ack","excludeFromRunAll":"true"}
set -euo pipefail
LAB="$HOME/backend-labs/messaging"
test -f "$LAB/.be-mq-lab"
cd "$LAB"
cat > app/labs/17-redelivery.mjs <<'JS'
import assert from 'node:assert/strict';
import { rabbit, channel, quorum, confirmSend, randomUUID, until } from '../lib/common.mjs';
const producer = await rabbit();
const pub = await channel(producer, true);
const name = 'mq-redelivery-' + randomUUID();
await quorum(pub, name);
let abandoned;
let replacement;
try {
  await confirmSend(pub, '', name, Buffer.from('{"operation":"synthetic"}'), { messageId: randomUUID() });
  abandoned = await rabbit();
  const a = await channel(abandoned);
  const first = await a.get(name, { noAck: false });
  assert.ok(first);
  console.log('FIRST', { redelivered: first.fields.redelivered });
  await abandoned.close();
  abandoned = undefined;
  replacement = await rabbit();
  const b = await channel(replacement);
  const second = await until(() => b.get(name, { noAck: false }), 10000, 'redelivery');
  console.log('SECOND', { redelivered: second.fields.redelivered });
  assert.equal(second.fields.redelivered, true);
  b.ack(second);
  const status = await b.checkQueue(name);
  console.log('READY_AFTER_ACK', status.messageCount);
  assert.equal(status.messageCount, 0);
} finally {
  if (abandoned) await abandoned.close().catch(() => {});
  if (replacement) await replacement.close().catch(() => {});
  await pub.deleteQueue(name).catch(() => {});
  await producer.close().catch(() => {});
}
JS
docker compose -p be-mq exec -T app node labs/17-redelivery.mjs | tee evidence/17-redelivery.txt
```

Đóng connection là một cách tạo điều kiện unacked được giao lại, không mô phỏng toàn bộ mất điện hoặc network partition. Trong sự cố thật, broker cần phát hiện connection chết qua socket/heartbeat, nên không hứa “ngay lập tức”. `redelivered: true` là tín hiệu đã được giao trước; nó không chứng minh worker trước đã commit, chưa commit hay gửi email. Consumer vẫn phải kiểm tra khóa nghiệp vụ.

Delivery tag có phạm vi channel. ACK delivery từ channel khác hoặc ACK hai lần có thể làm channel lỗi. Khi xử lý song song, tránh `ack(msg, true)` nếu các delivery trước chưa hoàn thành; nó có thể xác nhận cả công việc đang chạy dở. Lab dùng ACK một delivery để quan hệ với transaction rõ ràng.

### ACK đúng thời điểm

Thứ tự thường đúng cho thay đổi DB là: parse và validate → BEGIN → inbox insert → business mutation → outbox nếu cần → COMMIT → ACK. Parse ngoài `try` hoặc callback async không có catch có thể thành unhandled rejection và để delivery mắc kẹt. ACK trong `finally` cũng sai vì chạy cả khi rollback. Hãy coi ACK là một nhánh thành công có chủ đích, không phải thao tác dọn dẹp.

## Lab 18 — Retry TTL theo các tầng và quarantine hữu hạn

**Mục tiêu:** thiết lập topology dùng cho capstone, retry hai lần qua queue chờ 1 giây và 4 giây, rồi parking queue. Mỗi tầng có TTL cố định để tránh trộn nhiều expiration rất khác nhau trong cùng queue.

Lab dùng **quorum queues** và policy bật `dead-letter-strategy: at-least-once` cùng `overflow: reject-publish` cho nguồn có DLX. DLX mặc định không tự có mức bảo đảm này; classic queue hay cấu hình quorum không đầy đủ có thể mất message trong đoạn chuyển tiếp. Cấu hình này còn cần target hoạt động và có thể tạo duplicate khi thử lại nội bộ. Xem [RabbitMQ quorum dead-lettering](https://www.rabbitmq.com/docs/4.2/quorum-queues) và [dead letter exchanges](https://www.rabbitmq.com/docs/4.2/dlx).

```bash {"name":"mq-18-topology-and-policies","excludeFromRunAll":"true"}
set -euo pipefail
LAB="$HOME/backend-labs/messaging"
test -f "$LAB/.be-mq-lab"
cd "$LAB"
cat > app/lib/topology.mjs <<'JS'
import { quorum } from './common.mjs';
export const topology = {
  events: 'mq.events',
  retryExchange: 'mq.retry.ex',
  parkingExchange: 'mq.parking.ex',
  rewards: 'mq.rewards',
  retry1: 'mq.retry.1000',
  retry4: 'mq.retry.4000',
  parking: 'mq.parking',
  audit: 'mq.audit',
  notifications: 'mq.notifications'
};
export async function ensureTopology(ch) {
  const t = topology;
  await ch.assertExchange(t.events, 'topic', { durable: true });
  await ch.assertExchange(t.retryExchange, 'direct', { durable: true });
  await ch.assertExchange(t.parkingExchange, 'direct', { durable: true });
  for (const name of [t.rewards, t.retry1, t.retry4, t.parking, t.audit, t.notifications]) {
    await quorum(ch, name);
  }
  await ch.bindQueue(t.rewards, t.events, 'order.paid');
  await ch.bindQueue(t.audit, t.events, '#');
  await ch.bindQueue(t.notifications, t.events, 'wallet.rewarded');
  await ch.bindQueue(t.retry1, t.retryExchange, 'retry.1');
  await ch.bindQueue(t.retry4, t.retryExchange, 'retry.2');
  await ch.bindQueue(t.parking, t.parkingExchange, 'parked');
}
JS
cat > app/capstone/topology.mjs <<'JS'
import { rabbit, channel } from '../lib/common.mjs';
import { ensureTopology } from '../lib/topology.mjs';
const conn = await rabbit();
try {
  const ch = await channel(conn, true);
  await ensureTopology(ch);
  console.log('TOPOLOGY_READY');
} finally { await conn.close(); }
JS
docker compose -p be-mq exec -T rabbit rabbitmqctl set_policy -p /lab --apply-to queues --priority 10 mq-capacity '^mq\.(audit|notifications)$' '{"max-length":10000,"overflow":"reject-publish"}'
docker compose -p be-mq exec -T rabbit rabbitmqctl set_policy -p /lab --apply-to queues --priority 10 mq-parking '^mq\.parking$' '{"max-length":10000,"overflow":"reject-publish","delivery-limit":-1}'
docker compose -p be-mq exec -T rabbit rabbitmqctl set_policy -p /lab --apply-to queues --priority 20 mq-main '^mq\.rewards$' '{"dead-letter-exchange":"mq.parking.ex","dead-letter-routing-key":"parked","dead-letter-strategy":"at-least-once","overflow":"reject-publish","delivery-limit":10,"max-length":10000}'
docker compose -p be-mq exec -T rabbit rabbitmqctl set_policy -p /lab --apply-to queues --priority 20 mq-retry1 '^mq\.retry\.1000$' '{"message-ttl":1000,"dead-letter-exchange":"mq.events","dead-letter-routing-key":"order.paid","dead-letter-strategy":"at-least-once","overflow":"reject-publish","max-length":10000}'
docker compose -p be-mq exec -T rabbit rabbitmqctl set_policy -p /lab --apply-to queues --priority 20 mq-retry4 '^mq\.retry\.4000$' '{"message-ttl":4000,"dead-letter-exchange":"mq.events","dead-letter-routing-key":"order.paid","dead-letter-strategy":"at-least-once","overflow":"reject-publish","max-length":10000}'
docker compose -p be-mq exec -T app node capstone/topology.mjs
docker compose -p be-mq exec -T rabbit rabbitmqctl list_policies -p /lab
docker compose -p be-mq exec -T rabbit rabbitmqctl list_queues -p /lab name type policy messages_ready messages_unacknowledged
```

Parking queue tắt delivery-limit tự động để việc inspect/requeue có kiểm soát không làm mất thông điệp sau nhiều lần giao lại; main queue vẫn giữ giới hạn 10 và DLX. Quyết định này đi cùng giới hạn dung lượng, quyền replay và cảnh báo, không dùng để bật vòng requeue nóng.

Các policies được quản lý ngoài mã worker để có thể thay đổi mà không xóa queue chứa dữ liệu. Queue type vẫn là đặc tính cần chọn khi khai báo. Nếu queue có sẵn cùng tên nhưng khác type/arguments, không sửa bằng purge hoặc delete mù quáng: lên kế hoạch queue mới và chuyển traffic.

**Bước 2:** một script xử lý thông điệp tổng hợp có lỗi transient, rồi quan sát nó vào parking. Nó không sử dụng queue capstone để không lẫn dữ liệu; topology nhỏ của bài có tên riêng. Vì policy TTL source trong bài này được đặt bằng arguments để tự chứa, không dùng nó làm template quản lý production.

```bash {"name":"mq-18-bounded-retry-demo","excludeFromRunAll":"true"}
set -euo pipefail
LAB="$HOME/backend-labs/messaging"
test -f "$LAB/.be-mq-lab"
cd "$LAB"
cat > app/labs/18-retry.mjs <<'JS'
import assert from 'node:assert/strict';
import { rabbit, channel, quorum, confirmSend, randomUUID, until, sleep } from '../lib/common.mjs';
const conn = await rabbit();
const pub = await channel(conn, true);
const read = await channel(conn);
const root = 'mq-retry-demo-' + randomUUID();
const main = root + '-main';
const park = root + '-park';
const waits = [root + '-wait-300', root + '-wait-600'];
const queues = [main, park, ...waits];
let attempts = 0;
let parked;
try {
  await quorum(read, main);
  await quorum(read, park);
  for (let i = 0; i < waits.length; i++) {
    await quorum(read, waits[i], {
      'x-message-ttl': [300, 600][i],
      'x-dead-letter-exchange': '',
      'x-dead-letter-routing-key': main,
      'x-dead-letter-strategy': 'at-least-once',
      'x-overflow': 'reject-publish',
      'x-max-length': 100
    });
  }
  await confirmSend(pub, '', main, Buffer.from('{"simulate":"transient"}'), {
    messageId: randomUUID(), headers: { 'x-app-attempt': 0 }
  });
  await until(async () => {
    const msg = await read.get(main, { noAck: false });
    if (msg) {
      const attempt = Number(msg.properties.headers?.['x-app-attempt'] || 0);
      attempts++;
      console.log('PROCESS_ATTEMPT', attempt + 1);
      const target = attempt < 2 ? waits[attempt] : park;
      await confirmSend(pub, '', target, msg.content, {
        messageId: msg.properties.messageId,
        headers: { ...msg.properties.headers, 'x-app-attempt': attempt + 1, 'x-error-code': 'SIMULATED_503' }
      });
      read.ack(msg);
    }
    parked = await read.get(park, { noAck: false });
    return parked;
  }, 15000, 'message in parking queue');
  console.log('PARKED_HEADERS', parked.properties.headers);
  assert.equal(attempts, 3);
  read.ack(parked);
  await sleep(50);
} finally {
  await read.close().catch(() => {});
  for (const name of queues) await pub.deleteQueue(name).catch(() => {});
  await conn.close().catch(() => {});
}
JS
docker compose -p be-mq exec -T app node labs/18-retry.mjs | tee evidence/18-retry.txt
```

**Điều được bảo vệ:** source ACK chỉ xảy ra sau confirmed publish tới target có route. Nếu process chết giữa hai thao tác đó, source có thể giao lại và target đã có một bản sao. Khoảng trống này không biến mất; consumer dùng idempotency để chịu nó.

**Điều không nên làm:** gọi `nack(msg, false, true)` cho mọi exception. Requeue không có delay có thể tạo vòng nóng. Nếu broker/target retry không khả dụng, giữ delivery unacked, dừng consumer có kiểm soát và đóng connection để broker phục hồi sau. Supervisor cần backoff và giới hạn flapping. Không tạo một “retry loop” ở tầng process rồi tưởng đã có ngân sách per-message.

TTL là thời gian sống trong queue, không là đồng hồ chính xác tuyệt đối. Per-message TTL khác nhau có thể bị ảnh hưởng bởi vị trí trong queue; các tầng TTL cố định giúp reasoning dễ hơn. Không đặt consumer vào retry wait queues. [RabbitMQ TTL](https://www.rabbitmq.com/docs/ttl) phân biệt message TTL với queue expiration; hết hạn cả queue không phải cách chuyển từng message sang DLQ.

Một card decline hoặc thiếu số dư thường là kết quả nghiệp vụ, không phải ví dụ hợp lý để retry exponential ba lần. Lab dùng lỗi transient tổng hợp để không tập thói quen retry mọi vấn đề.

## Lab 19 — Prefetch, concurrency ứng dụng và thứ tự hoàn thành

**Mục tiêu:** thấy message có thể được giao theo thứ tự nhưng hoàn thành khác thứ tự. Prefetch giới hạn số delivery chưa ACK; nó không tạo một semaphore riêng cho mỗi hàm async.

```bash {"name":"mq-19-prefetch-ordering","excludeFromRunAll":"true"}
set -euo pipefail
LAB="$HOME/backend-labs/messaging"
test -f "$LAB/.be-mq-lab"
cd "$LAB"
cat > app/labs/19-prefetch.mjs <<'JS'
import assert from 'node:assert/strict';
import { rabbit, channel, quorum, confirmSend, randomUUID, sleep, until } from '../lib/common.mjs';
const conn = await rabbit();
const pub = await channel(conn, true);
const read = await channel(conn);
const name = 'mq-prefetch-' + randomUUID();
let active = 0;
let peak = 0;
const started = [];
const finished = [];
const pending = new Set();
let fatal;
let tag;
try {
  await quorum(read, name);
  for (const [index, delay] of [[1,500],[2,40],[3,60],[4,40]]) {
    await confirmSend(pub, '', name, Buffer.from(JSON.stringify({ index, delay })));
  }
  await read.prefetch(2);
  const subscription = await read.consume(name, msg => {
    if (!msg) { fatal = new Error('Consumer cancelled'); return; }
    const task = (async () => {
      const value = JSON.parse(msg.content.toString());
      active++; peak = Math.max(peak, active); started.push(value.index);
      try {
        await sleep(value.delay);
        finished.push(value.index);
        read.ack(msg);
      } finally { active--; }
    })().catch(error => { fatal = error; });
    pending.add(task);
    task.then(() => pending.delete(task));
  }, { noAck: false });
  tag = subscription.consumerTag;
  await until(() => {
    if (fatal) throw fatal;
    return finished.length === 4;
  }, 10000, 'four completed deliveries');
  console.log({ started, finished, peak });
  assert.ok(peak <= 2);
  assert.equal(started[0], 1);
  assert.notEqual(finished[0], 1);
} finally {
  if (tag) await read.cancel(tag).catch(() => {});
  await Promise.allSettled([...pending]);
  await read.close().catch(() => {});
  await pub.deleteQueue(name).catch(() => {});
  await conn.close().catch(() => {});
}
JS
docker compose -p be-mq exec -T app node labs/19-prefetch.mjs | tee evidence/19-prefetch.txt
```

RabbitMQ áp dụng prefetch theo consumer khi dùng cách gọi thông thường; đừng mặc định rằng một channel có năm consumers sẽ chỉ nhận tổng cộng đúng một prefetch. [RabbitMQ consumer prefetch](https://www.rabbitmq.com/docs/consumer-prefetch) giải thích khác biệt với AMQP gốc. Thông số tối ưu phụ thuộc service time, roundtrip, payload và RAM; `prefetch: 1` không luôn tốt nhất.

**Thứ tự nghiệp vụ:** cập nhật trạng thái `PAID` rồi `REFUNDED` có thể hoàn thành ngược nhau nếu dùng nhiều consumers. Có thể dùng một consumer có giới hạn phù hợp, phân vùng theo aggregate key, optimistic version trong DB hoặc state machine chặn transition sai. Đặt hàng đợi FIFO chỉ kiểm soát một phần; retry, requeue, ưu tiên và xử lý song song thay đổi thứ tự quan sát. Xem [RabbitMQ queues and ordering](https://www.rabbitmq.com/docs/queues).

Trong ví dụ, callback của `consume` không trực tiếp là async không được quản lý. Nó tạo task, gắn catch, theo dõi active set và đợi các task khi đóng. Mẫu này tránh Promise rejection rơi ngoài luồng điều khiển. Khi bổ sung DB, rollback nằm trong helper transaction; ACK chỉ nằm sau kết quả commit.



---

# Phần V. Bảo vệ nghiệp vụ bằng inbox, ledger và transactional outbox

## 7. Idempotency là thuộc tính của thao tác với một khóa xác định

Giả sử cùng một event cộng 5.000 VND được giao ba lần. Điều ta cần không phải “chỉ nhìn thấy callback một lần”; điều cần là số dư tăng đúng 5.000 và có một bút toán tương ứng. Callback có thể chạy nhiều lần để kiểm tra dữ liệu rồi kết thúc mà không tạo thêm tác dụng phụ.

Khóa cần bao gồm đúng phạm vi nghiệp vụ. Inbox thường dùng `(consumer, event_id)`, vì cùng một event có thể được rewards và inventory xử lý độc lập. Ledger còn có `operation_key` như `reward:<orderId>` để chặn trường hợp producer vô tình phát hai event IDs khác nhau cho cùng một nghĩa vụ cộng thưởng. Hai lớp bảo vệ giải quyết hai loại trùng khác nhau.

### Vì sao SETNX rồi xử lý có thể làm mất việc?

Hãy đặt một key `processed:event-1=DONE` bằng Redis SET NX, rồi gọi provider gửi email. Nếu process chết giữa hai bước, lần sau thấy key đã tồn tại và ACK bỏ qua. Email chưa từng được gửi. Đổi giá trị thành `PROCESSING` chưa tự giải quyết: bạn cần lease, owner token, timeout, trạng thái chưa biết và cơ chế phục hồi. Nếu key hết TTL sau khi provider đã gửi nhưng trước khi ghi DONE, lần chạy sau có thể gửi lại.

Ngay cả khi đánh dấu sau side effect, crash ở giữa side effect và dấu DONE lại tạo duplicate. Hai hệ thống độc lập không tự có transaction chung chỉ vì các lệnh được `await` theo thứ tự. Với thay đổi trong một DB, đặt inbox và business mutation trong cùng transaction. Với HTTP provider, dùng idempotency key có hợp đồng cụ thể và API hỏi trạng thái, hoặc chấp nhận semantics yếu hơn được product đồng ý.

### Độ bền và phạm vi bảo đảm

Bản ghi unique không phải phép màu nếu xóa nó quá sớm, restore DB về mốc cũ, đổi key mỗi retry hoặc ghi sang database khác. Hãy định nghĩa retention theo khoảng thời gian có thể phát lại message, bao gồm backup restore và replay thủ công. Nếu archive inbox, ledger hoặc business constraint dài hạn vẫn cần bảo vệ các tác dụng phụ không được lặp.

Các ví dụ dưới đây bảo vệ **cộng thưởng trong cùng PostgreSQL**. Chúng không tự chứng minh sự đúng đắn của một hệ thống ví thật: ledger thật có thể cần double-entry, chính sách hoàn/bù, kiểm soát quyền và quy trình đối soát riêng. Ở đây “minor unit” của VND là một đồng; với USD thường là cent. Mã không dùng số thực để tính tiền.

## Lab 20 — Inbox + ledger + số dư: transaction thật và thử đồng thời

**Mục tiêu:** chứng minh ba điều bằng SQL:

1. Lỗi trước COMMIT không để lại inbox hoặc khoản cộng dở.
2. Mười lời gọi đồng thời với cùng event chỉ tạo một tác dụng phụ.
3. Event ID khác nhưng cùng business operation vẫn không cộng lại; event ID cũ với payload khác bị phát hiện.

**Tiền điều kiện:** schema ở Lab 03. Mã dưới đây là miền nghiệp vụ dùng tiếp trong capstone.

```bash {"name":"mq-20-domain-and-idempotency","excludeFromRunAll":"true"}
set -euo pipefail
LAB="$HOME/backend-labs/messaging"
test -f "$LAB/.be-mq-lab"
cd "$LAB"
cat > app/lib/domain.mjs <<'JS'
import { tx, hash, randomUUID, PermanentError } from './common.mjs';
function textField(value, name) {
  if (typeof value !== 'string' || value.length < 1 || value.length > 180) {
    throw new PermanentError('INVALID_' + name);
  }
}
function amount(value, name, allowZero = false) {
  if (!Number.isSafeInteger(value) || value < (allowZero ? 0 : 1) || value > 1000000000000) {
    throw new PermanentError('INVALID_' + name);
  }
}
export function validateOrderPaid(event) {
  if (!event || typeof event !== 'object' || Array.isArray(event)) throw new PermanentError('INVALID_EVENT');
  textField(event.eventId, 'EVENT_ID');
  if (event.eventType !== 'order.paid') throw new PermanentError('UNSUPPORTED_EVENT_TYPE');
  if (event.schemaVersion !== 1) throw new PermanentError('UNSUPPORTED_SCHEMA_VERSION');
  if (!event.aggregate || event.aggregate.type !== 'order') throw new PermanentError('INVALID_AGGREGATE');
  textField(event.aggregate.id, 'AGGREGATE_ID');
  if (event.aggregate.version !== 1) throw new PermanentError('UNSUPPORTED_AGGREGATE_VERSION');
  if (typeof event.occurredAt !== 'string' || !Number.isFinite(Date.parse(event.occurredAt))) {
    throw new PermanentError('INVALID_OCCURRED_AT');
  }
  const data = event.data;
  if (!data || typeof data !== 'object') throw new PermanentError('INVALID_DATA');
  textField(data.orderId, 'ORDER_ID');
  textField(data.walletId, 'WALLET_ID');
  if (data.orderId !== event.aggregate.id) throw new PermanentError('AGGREGATE_MISMATCH');
  if (data.currency !== 'VND') throw new PermanentError('UNSUPPORTED_CURRENCY');
  amount(data.totalMinor, 'TOTAL_MINOR');
  amount(data.rewardMinor, 'REWARD_MINOR');
  if (data.rewardMinor > data.totalMinor) throw new PermanentError('REWARD_EXCEEDS_TOTAL');
  return event;
}
export function makeOrderPaid({ orderId, walletId, totalMinor, rewardMinor, eventId = randomUUID() }) {
  return {
    eventId,
    eventType: 'order.paid',
    schemaVersion: 1,
    occurredAt: new Date().toISOString(),
    aggregate: { type: 'order', id: orderId, version: 1 },
    traceId: randomUUID().replaceAll('-', ''),
    data: { orderId, walletId, currency: 'VND', totalMinor, rewardMinor }
  };
}
export async function createPaidOrder(input, options = {}) {
  const { clientKey, walletId, totalMinor, rewardMinor } = input;
  textField(clientKey, 'CLIENT_KEY');
  textField(walletId, 'WALLET_ID');
  amount(totalMinor, 'TOTAL_MINOR');
  amount(rewardMinor, 'REWARD_MINOR');
  if (rewardMinor > totalMinor) throw new PermanentError('REWARD_EXCEEDS_TOTAL');
  const requestHash = hash({ walletId, totalMinor, rewardMinor, currency: 'VND' });
  return tx(async client => {
    await client.query("INSERT INTO wallets(id,currency) VALUES($1,'VND') ON CONFLICT(id) DO NOTHING", [walletId]);
    const wallet = await client.query('SELECT currency FROM wallets WHERE id=$1', [walletId]);
    if (wallet.rows[0].currency !== 'VND') throw new PermanentError('WALLET_CURRENCY_MISMATCH');
    const orderId = randomUUID();
    const inserted = await client.query(
      "INSERT INTO orders(id,client_key,request_hash,wallet_id,currency,total_minor,reward_minor,status) VALUES($1,$2,$3,$4,'VND',$5,$6,'PAID') ON CONFLICT(client_key) DO NOTHING RETURNING id",
      [orderId, clientKey, requestHash, walletId, String(totalMinor), String(rewardMinor)]
    );
    if (!inserted.rowCount) {
      const old = await client.query('SELECT id,request_hash,wallet_id FROM orders WHERE client_key=$1', [clientKey]);
      if (old.rows[0].request_hash !== requestHash) throw new PermanentError('IDEMPOTENCY_KEY_REUSED_WITH_DIFFERENT_REQUEST');
      const evt = await client.query(
        "SELECT id FROM outbox WHERE aggregate_id=$1 AND routing_key='order.paid' ORDER BY created_at LIMIT 1",
        [old.rows[0].id]
      );
      return { orderId: old.rows[0].id, walletId: old.rows[0].wallet_id, eventId: evt.rows[0]?.id, created: false };
    }
    if (options.failAfterOrder) throw new Error('INJECTED_FAILURE_AFTER_ORDER');
    const event = makeOrderPaid({ orderId, walletId, totalMinor, rewardMinor });
    await client.query(
      'INSERT INTO outbox(id,aggregate_id,routing_key,payload) VALUES($1,$2,$3,$4::jsonb)',
      [event.eventId, orderId, 'order.paid', JSON.stringify(event)]
    );
    return { orderId, walletId, eventId: event.eventId, created: true };
  });
}
export async function applyReward(rawEvent, options = {}) {
  const event = validateOrderPaid(rawEvent);
  const consumer = 'wallet-rewards-v1';
  const payloadHash = hash(event);
  const data = event.data;
  const operationKey = 'reward:' + data.orderId;
  return tx(async client => {
    const inserted = await client.query(
      'INSERT INTO inbox(consumer,event_id,payload_hash) VALUES($1,$2,$3) ON CONFLICT(consumer,event_id) DO NOTHING RETURNING event_id',
      [consumer, event.eventId, payloadHash]
    );
    if (!inserted.rowCount) {
      const existing = await client.query(
        'SELECT payload_hash FROM inbox WHERE consumer=$1 AND event_id=$2',
        [consumer, event.eventId]
      );
      if (existing.rows[0].payload_hash !== payloadHash) throw new PermanentError('EVENT_ID_PAYLOAD_CONFLICT');
      return { duplicateDelivery: true, credited: false };
    }
    const ledger = await client.query(
      'INSERT INTO wallet_ledger(operation_key,wallet_id,currency,delta_minor,event_id) VALUES($1,$2,$3,$4,$5) ON CONFLICT(operation_key) DO NOTHING RETURNING operation_key',
      [operationKey, data.walletId, data.currency, String(data.rewardMinor), event.eventId]
    );
    if (!ledger.rowCount) {
      const existing = await client.query(
        'SELECT wallet_id,currency,delta_minor FROM wallet_ledger WHERE operation_key=$1',
        [operationKey]
      );
      const row = existing.rows[0];
      if (row.wallet_id !== data.walletId || row.currency !== data.currency || row.delta_minor !== String(data.rewardMinor)) {
        throw new PermanentError('BUSINESS_OPERATION_CONFLICT');
      }
      return { duplicateOperation: true, credited: false };
    }
    const updated = await client.query(
      'UPDATE wallets SET balance_minor=balance_minor+$1::bigint WHERE id=$2 AND currency=$3 RETURNING balance_minor',
      [String(data.rewardMinor), data.walletId, data.currency]
    );
    if (updated.rowCount !== 1) throw new PermanentError('WALLET_NOT_FOUND_OR_CURRENCY_MISMATCH');
    if (options.failBeforeCommit) throw new Error('INJECTED_FAILURE_BEFORE_COMMIT');
    const rewarded = {
      eventId: 'rewarded-' + data.orderId,
      eventType: 'wallet.rewarded',
      schemaVersion: 1,
      occurredAt: new Date().toISOString(),
      aggregate: { type: 'order', id: data.orderId, version: 1 },
      causationId: event.eventId,
      traceId: event.traceId,
      data: { orderId: data.orderId, walletId: data.walletId, currency: data.currency, amountMinor: data.rewardMinor }
    };
    await client.query(
      'INSERT INTO outbox(id,aggregate_id,routing_key,payload) VALUES($1,$2,$3,$4::jsonb)',
      [rewarded.eventId, data.orderId, 'wallet.rewarded', JSON.stringify(rewarded)]
    );
    return { credited: true, balanceMinor: updated.rows[0].balance_minor };
  });
}
JS
cat > app/labs/20-inbox.mjs <<'JS'
import assert from 'node:assert/strict';
import { pool, randomUUID } from '../lib/common.mjs';
import { applyReward, makeOrderPaid } from '../lib/domain.mjs';
const walletId = 'inbox-' + randomUUID();
const orderId = randomUUID();
const event = makeOrderPaid({ orderId, walletId, totalMinor: 100000, rewardMinor: 5000 });
try {
  await pool.query("INSERT INTO wallets(id,currency) VALUES($1,'VND')", [walletId]);
  await assert.rejects(applyReward(event, { failBeforeCommit: true }), /INJECTED/);
  const rolledBack = await pool.query('SELECT balance_minor FROM wallets WHERE id=$1', [walletId]);
  assert.equal(rolledBack.rows[0].balance_minor, '0');
  assert.equal((await pool.query('SELECT 1 FROM inbox WHERE event_id=$1', [event.eventId])).rowCount, 0);
  assert.equal((await pool.query('SELECT 1 FROM wallet_ledger WHERE operation_key=$1', ['reward:' + orderId])).rowCount, 0);
  const results = await Promise.all(Array.from({ length: 10 }, () => applyReward(event)));
  assert.equal(results.filter(result => result.credited).length, 1);
  const conflict = structuredClone(event);
  conflict.data.rewardMinor = 6000;
  await assert.rejects(applyReward(conflict), /EVENT_ID_PAYLOAD_CONFLICT/);
  const newEventSameOperation = { ...event, eventId: randomUUID() };
  const repeatedBusiness = await applyReward(newEventSameOperation);
  assert.equal(repeatedBusiness.credited, false);
  const balance = await pool.query('SELECT balance_minor FROM wallets WHERE id=$1', [walletId]);
  const ledger = await pool.query('SELECT * FROM wallet_ledger WHERE operation_key=$1', ['reward:' + orderId]);
  console.log({ balance: balance.rows[0], ledgerCount: ledger.rowCount, results, repeatedBusiness });
  assert.equal(balance.rows[0].balance_minor, '5000');
  assert.equal(ledger.rowCount, 1);
} finally { await pool.end(); }
JS
docker compose -p be-mq exec -T app node labs/20-inbox.mjs | tee evidence/20-inbox.txt
```

**Đọc kỹ nhánh conflict:** inbox insert với `ON CONFLICT DO NOTHING` trả zero rows khi delivery đã được xử lý. Consumer vẫn kiểm tra hash để không âm thầm bỏ một payload khác nhưng trùng ID. Nếu chỉ bắt mọi unique violation rồi ACK, một lỗi unique ở bảng nghiệp vụ khác cũng có thể bị che đi.

**Vì sao mười lời gọi đồng thời vẫn đúng?** Ràng buộc unique và transaction được DB thực thi; chúng không dựa vào kiểm tra “SELECT chưa thấy rồi INSERT”. Những transaction cạnh tranh có thể chờ nhau, thấy kết quả commit trước rồi trả duplicate. Nếu transaction đầu rollback, transaction khác vẫn có cơ hội thực hiện. [PostgreSQL isolation](https://www.postgresql.org/docs/17/transaction-iso.html) là tài liệu nền cho việc hiểu hiện tượng đồng thời; bài cần chứng minh bằng ràng buộc, không chỉ chọn một isolation level thật cao.

**Một giới hạn có chủ đích:** rewards consumer ở đây tin cậy event của order service đã được xác thực qua quyền publish. Nó không truy vấn bảng orders như điều kiện bắt buộc của mỗi message, để mô hình vẫn có thể tách DB về sau. Vì vậy quyền publish và validation payload là một phần hợp đồng. Một service bất kỳ được quyền phát `order.paid` có thể cấp thưởng trái phép; idempotency không thay thế authorization.

### Bài tập suy nghĩ

Nếu reward thay đổi sau khi đơn đã phát event, có nên sửa payload rồi giữ eventId cũ? Không. Event đã công bố là dữ kiện bất biến trong hợp đồng. Hãy phát một event điều chỉnh hoặc một nghiệp vụ reversal với ID riêng và tham chiếu nguyên nhân. Ledger giữ lịch sử thay vì ghi đè dấu vết cũ.

## 8. Dual-write và ý nghĩa thật của outbox

Nếu API commit order rồi publish, nó có thể chết giữa hai bước. Nếu publish rồi commit order, consumer có thể nhìn thấy một điều chưa tồn tại hoặc transaction cuối bị rollback. Transaction SQL không bao phủ TCP publish một cách tự động.

Outbox đưa **ý định phát sự kiện** vào cùng database transaction với dữ liệu nguồn. Sau COMMIT, nghĩa vụ đã được lưu dù API chết. Relay đọc nghĩa vụ rồi publish. Vì relay vẫn có hai thao tác khác hệ thống, cửa sổ “broker đã nhận nhưng DB chưa đánh dấu sent” còn tồn tại. Kết quả đúng là phát ít nhất một lần kết hợp consumer chịu duplicate.

Một array JavaScript không minh họa được durability hoặc rollback này. Process chết làm mất array; việc push hai phần tử liên tiếp không là ACID transaction. Bài sau dùng PostgreSQL thật, có crash injection và kiểm tra số hàng.

## Lab 21 — Tạo đơn và outbox trong cùng transaction

**Mục tiêu:** rollback giữa hai INSERT không để lại đơn mồ côi; retry HTTP cùng khóa trả lại cùng đơn.

```bash {"name":"mq-21-transactional-outbox-write","excludeFromRunAll":"true"}
set -euo pipefail
LAB="$HOME/backend-labs/messaging"
test -f "$LAB/.be-mq-lab"
cd "$LAB"
cat > app/capstone/create-order.mjs <<'JS'
import { pool, randomUUID } from '../lib/common.mjs';
import { createPaidOrder } from '../lib/domain.mjs';
const clientKey = process.argv[2] || 'checkout-' + randomUUID();
const overrides = process.argv[3] ? JSON.parse(process.argv[3]) : {};
try {
  const result = await createPaidOrder({
    clientKey,
    walletId: overrides.walletId || 'wallet-' + clientKey,
    totalMinor: overrides.totalMinor ?? 100000,
    rewardMinor: overrides.rewardMinor ?? 5000
  });
  console.log(JSON.stringify({ ...result, clientKey }));
} finally { await pool.end(); }
JS
cat > app/labs/21-outbox-atomic.mjs <<'JS'
import assert from 'node:assert/strict';
import { pool, randomUUID } from '../lib/common.mjs';
import { createPaidOrder } from '../lib/domain.mjs';
const input = {
  clientKey: 'atomic-' + randomUUID(),
  walletId: 'atomic-wallet-' + randomUUID(),
  totalMinor: 100000,
  rewardMinor: 5000
};
try {
  await assert.rejects(createPaidOrder(input, { failAfterOrder: true }), /INJECTED/);
  assert.equal((await pool.query('SELECT 1 FROM orders WHERE client_key=$1', [input.clientKey])).rowCount, 0);
  const first = await createPaidOrder(input);
  const retried = await createPaidOrder(input);
  assert.equal(first.orderId, retried.orderId);
  assert.equal(first.eventId, retried.eventId);
  await assert.rejects(createPaidOrder({ ...input, totalMinor: 200000 }), /IDEMPOTENCY_KEY_REUSED/);
  const rows = await pool.query('SELECT id,routing_key,sent_at FROM outbox WHERE aggregate_id=$1', [first.orderId]);
  assert.equal(rows.rowCount, 1);
  assert.equal(rows.rows[0].sent_at, null);
  console.log({ first, retried, outbox: rows.rows });
} finally { await pool.end(); }
JS
docker compose -p be-mq exec -T app node labs/21-outbox-atomic.mjs | tee evidence/21-outbox-atomic.txt
```

**Kết quả:** không có order sau failure trước outbox; một order và một outbox sau thành công; cùng client key trả cùng IDs; đổi nội dung với cùng key bị từ chối.

**Tại sao request idempotency khác event idempotency?** HTTP client có thể retry trước khi nhận orderId. API cần khóa do client giữ ổn định để không tạo hai orders. Event ID được hệ thống tạo sau khi biết thao tác nguồn; consumer dùng nó ở đoạn broker. Mỗi lớp xử lý một biên retry khác nhau. Chỉ bảo vệ consumer vẫn có thể tạo hai orders hoàn toàn hợp lệ từ hai lần POST không có khóa.

Ở production, lưu đủ response hoặc kết quả có thể dựng lại để lần retry trả cùng nghĩa. Chọn TTL request key theo thời gian client được phép retry. Một request key bị dùng lại sau TTL có thể tạo nghĩa vụ mới; product và API contract phải nói rõ điều đó.

## Lab 22 — Relay có lease, SKIP LOCKED và publisher confirms

**Mục tiêu:** triển khai relay thực, không giữ transaction SQL mở trong lúc chờ broker; nhiều relay có thể nhận các dòng khác nhau.

Bài dùng một dòng mỗi claim. Claim được commit nhanh, có owner token và thời điểm hết lease. Timeout publish là 8 giây, lease mặc định 30 giây. Nếu worker relay chết, dòng được nhận lại sau lease. Nếu process bị treo lâu hơn lease nhưng chưa chết, relay khác có thể publish cùng event; token chỉ bảo vệ việc cập nhật ownership, consumer vẫn phải chống duplicate.

`SKIP LOCKED` hợp với hàng đợi công việc vì bỏ qua dòng đang được instance khác khóa. Nó không tạo một snapshot nhất quán cho báo cáo tài chính. [PostgreSQL SELECT locking](https://www.postgresql.org/docs/17/sql-select.html) nêu rõ mục đích phù hợp của cơ chế này.

```bash {"name":"mq-22-outbox-relay","excludeFromRunAll":"true"}
set -euo pipefail
LAB="$HOME/backend-labs/messaging"
test -f "$LAB/.be-mq-lab"
cd "$LAB"
cat > app/capstone/relay.mjs <<'JS'
import { pool, tx, randomUUID, rabbit, channel, confirmSend, log } from '../lib/common.mjs';
import { ensureTopology, topology } from '../lib/topology.mjs';
const limit = Number(process.env.MAX_EVENTS || 100);
const leaseSeconds = Number(process.env.CLAIM_LEASE_SECONDS || 30);
const onlyAggregate = process.env.ONLY_AGGREGATE_ID || null;
if (!Number.isInteger(limit) || limit < 1 || limit > 1000) throw new Error('INVALID_MAX_EVENTS');
if (!Number.isInteger(leaseSeconds) || leaseSeconds < 15 || leaseSeconds > 3600) throw new Error('INVALID_LEASE');
let conn;
let failed = false;
let stopped = false;
const stopRelay = () => { stopped = true; };
process.on('SIGTERM', stopRelay);
process.on('SIGINT', stopRelay);
async function claim() {
  const token = randomUUID();
  return tx(async client => {
    const result = await client.query(
      "WITH candidate AS (SELECT id FROM outbox WHERE sent_at IS NULL AND parked_at IS NULL AND next_attempt_at<=clock_timestamp() AND (claimed_at IS NULL OR claimed_at<clock_timestamp()-make_interval(secs=>$1::int)) AND ($2::text IS NULL OR aggregate_id=$2) ORDER BY created_at,id FOR UPDATE SKIP LOCKED LIMIT 1) UPDATE outbox o SET claim_token=$3,claimed_at=clock_timestamp(),attempts=o.attempts+1 FROM candidate c WHERE o.id=c.id RETURNING o.*",
      [leaseSeconds, onlyAggregate, token]
    );
    return result.rows[0];
  });
}
try {
  conn = await rabbit();
  const pub = await channel(conn, true);
  await ensureTopology(pub);
  for (let i = 0; i < limit && !stopped; i++) {
    const row = await claim();
    if (!row) break;
    try {
      await confirmSend(pub, topology.events, row.routing_key, Buffer.from(JSON.stringify(row.payload)), {
        messageId: row.id,
        type: row.payload.eventType,
        timestamp: Math.floor(Date.now() / 1000),
        headers: { 'schema-version': row.payload.schemaVersion, 'trace-id': row.payload.traceId }
      });
      log('OUTBOX_CONFIRMED', { eventId: row.id, attempt: row.attempts });
      if (process.env.CRASH_AFTER_CONFIRM === '1' && row.routing_key === 'order.paid') {
        log('INJECTED_CRASH_AFTER_CONFIRM', { eventId: row.id });
        process.exit(86);
      }
      const marked = await pool.query(
        'UPDATE outbox SET sent_at=clock_timestamp(),claim_token=NULL,claimed_at=NULL,last_error=NULL WHERE id=$1 AND claim_token=$2 AND sent_at IS NULL',
        [row.id, row.claim_token]
      );
      log(marked.rowCount ? 'OUTBOX_SENT' : 'OUTBOX_STALE_OWNER', { eventId: row.id });
    } catch (error) {
      await pool.query(
        "UPDATE outbox SET claim_token=NULL,claimed_at=NULL,next_attempt_at=clock_timestamp()+interval '2 seconds',last_error=$3,parked_at=CASE WHEN attempts>=8 THEN clock_timestamp() ELSE NULL END WHERE id=$1 AND claim_token=$2 AND sent_at IS NULL",
        [row.id, row.claim_token, error.message.slice(0, 500)]
      );
      log('OUTBOX_PUBLISH_FAILED', { eventId: row.id, message: error.message });
      failed = true;
      break;
    }
  }
} finally {
  if (conn) await conn.close().catch(() => {});
  await pool.end();
  process.removeListener('SIGTERM', stopRelay);
  process.removeListener('SIGINT', stopRelay);
}
if (failed) process.exitCode = 1;
JS
docker compose -p be-mq exec -T app node --check capstone/relay.mjs
docker compose -p be-mq exec -T app node capstone/relay.mjs | tee evidence/22-relay.txt
```

**Kết quả:** các outbox từ Lab 20/21 được gửi vào queues đúng routing key và marked sent. Message `wallet.rewarded` đi vào notification/audit; message `order.paid` vào rewards/audit. Chưa chạy rewards worker nên một số message có thể đang ready; đó là trạng thái mong đợi.

### Phân tích các cửa sổ lỗi của relay

| Thời điểm chết | Trạng thái nguồn | Điều gì xảy ra khi khôi phục? |
|---|---|---|
| Trước claim COMMIT | Dòng pending, chưa có owner bền | Relay khác nhận được |
| Sau claim COMMIT, trước publish | Dòng claimed, chưa có message | Hết lease rồi publish |
| Broker nhận, confirm chưa tới | Kết quả publish chưa biết | Thử lại có thể tạo bản sao |
| Confirm đã tới, trước sent UPDATE | Broker đã có message, dòng vẫn chưa sent | Thử lại chắc có khả năng bản sao |
| Sau sent UPDATE COMMIT | Nguồn biết publish đã được xác nhận | Consumer xử lý hoặc vẫn đang đợi |
| Lease hết khi relay cũ tiếp tục chạy | Có thể hai publishers | Owner token chặn ghi trạng thái thay owner; inbox/ledger chặn effect trùng |

**Vì sao không đánh dấu sent trước?** Làm vậy đổi duplicate thành mất việc: process có thể chết sau sent nhưng trước publish, và không relay nào nhận nữa. Thứ tự publish-confirm rồi mark sent ưu tiên không bỏ nghĩa vụ và chấp nhận bản sao.

**Tại sao claim limit bằng một?** Với batch 100, gửi tuần tự mỗi message mất tối đa 8 giây thì nhiều dòng hết lease trước khi đến lượt. Có thể claim batch và gia hạn lease, hoặc dùng limited parallelism; phải đo và thiết kế ngân sách. Bản đầu chọn đơn giản, quan sát được. Khi tối ưu, giữ nguyên phép chứng minh crash windows.

**Vì sao có parked_at trong outbox?** Binding sai vĩnh viễn không cần spam broker liên tục. Sau ngân sách lỗi, dòng vẫn nằm ở DB nhưng cần vận hành sửa nguyên nhân và mở lại có audit. Đây là parking ở producer, khác DLQ của consumer. Dashboard phải nhìn cả hai.

## Lab 23 — Consumer an toàn với async, rollback, retry và graceful shutdown

**Mục tiêu:** xây worker cho `mq.rewards`. ACK sau commit; nếu xử lý thất bại, chỉ ACK source sau confirmed publish sang retry/quarantine. Nếu transfer thất bại, worker dừng và source vẫn unacked.

Worker có prefetch 4 nên số handlers và confirmed publishes đồng thời có giới hạn nhỏ. Signal chỉ yêu cầu ngừng nhận; finally cancel consumer, chờ pending, rồi đóng connection/DB. Khi process bị SIGKILL, finally không chạy; inbox và ledger vẫn là tuyến bảo vệ.

```bash {"name":"mq-23-rewards-worker","excludeFromRunAll":"true"}
set -euo pipefail
LAB="$HOME/backend-labs/messaging"
test -f "$LAB/.be-mq-lab"
cd "$LAB"
cat > app/capstone/worker.mjs <<'JS'
import { pool, rabbit, channel, confirmSend, PermanentError, sleep, log } from '../lib/common.mjs';
import { applyReward } from '../lib/domain.mjs';
import { ensureTopology, topology } from '../lib/topology.mjs';
const runMs = Number(process.env.RUN_MS ?? 12000);
if (!Number.isInteger(runMs) || runMs < 0 || runMs > 3600000) throw new Error('INVALID_RUN_MS');
let stopping = false;
let fatal;
let tag;
let conn;
let read;
const pending = new Set();
const stop = () => { stopping = true; };
process.on('SIGTERM', stop);
process.on('SIGINT', stop);
function parse(msg) {
  if (msg.content.length > 65536) throw new PermanentError('PAYLOAD_TOO_LARGE');
  if (msg.properties.contentType !== 'application/json') throw new PermanentError('UNSUPPORTED_CONTENT_TYPE');
  let event;
  try { event = JSON.parse(msg.content.toString('utf8')); }
  catch { throw new PermanentError('INVALID_JSON'); }
  if (!msg.properties.messageId || msg.properties.messageId !== event?.eventId) {
    throw new PermanentError('MESSAGE_ID_MISMATCH');
  }
  return event;
}
try {
  conn = await rabbit();
  read = await channel(conn);
  const disconnected = () => {
    if (!stopping) { fatal = new Error('BROKER_CONNECTION_OR_CHANNEL_CLOSED'); stopping = true; }
  };
  conn.on('close', disconnected);
  read.on('close', disconnected);
  const pub = await channel(conn, true);
  await ensureTopology(pub);
  await read.prefetch(4);
  const handle = async msg => {
    const started = Date.now();
    try {
      const event = parse(msg);
      if (process.env.SIMULATE_TRANSIENT_EVENT_ID === event.eventId) {
        throw new Error('SIMULATED_DEPENDENCY_UNAVAILABLE');
      }
      const result = await applyReward(event);
      log('REWARD_COMMITTED', { eventId: event.eventId, orderId: event.data.orderId, traceId: event.traceId, ...result });
      if (process.env.CRASH_AFTER_COMMIT === '1' && process.env.CRASH_ON_EVENT_ID === event.eventId && result.credited) {
        log('INJECTED_CRASH_AFTER_COMMIT', { eventId: event.eventId });
        process.exit(87);
      }
      read.ack(msg);
      log('DELIVERY_ACKED', { eventId: event.eventId, durationMs: Date.now() - started });
    } catch (error) {
      const rawAttempt = msg.properties.headers?.['x-app-attempt'] ?? 0;
      const validAttempt = Number.isInteger(rawAttempt) && rawAttempt >= 0 && rawAttempt <= 2;
      const attempt = validAttempt ? rawAttempt : 2;
      const permanent = error instanceof PermanentError || !validAttempt;
      const park = permanent || attempt >= 2;
      const exchange = park ? topology.parkingExchange : topology.retryExchange;
      const key = park ? 'parked' : 'retry.' + (attempt + 1);
      await confirmSend(pub, exchange, key, msg.content, {
        messageId: msg.properties.messageId,
        contentType: msg.properties.contentType || 'application/octet-stream',
        type: msg.properties.type,
        headers: {
          ...(msg.properties.headers || {}),
          'x-app-attempt': attempt + 1,
          'x-error-code': permanent ? error.message.slice(0, 120) : 'TRANSIENT_HANDLER_ERROR',
          'x-quarantined-at': park ? new Date().toISOString() : ''
        }
      });
      read.ack(msg);
      log(park ? 'DELIVERY_PARKED' : 'DELIVERY_RETRY', {
        eventId: msg.properties.messageId,
        attempt: attempt + 1,
        reason: permanent ? error.message : error.name
      });
    }
  };
  const subscription = await read.consume(topology.rewards, msg => {
    if (!msg) { fatal = new Error('BROKER_CANCELLED_CONSUMER'); stopping = true; return; }
    const task = handle(msg).catch(error => {
      fatal = error;
      stopping = true;
      log('TRANSFER_FAILED_SOURCE_UNACKED', { message: error.message });
    });
    pending.add(task);
    task.then(() => pending.delete(task));
  }, { noAck: false });
  tag = subscription.consumerTag;
  log('WORKER_READY', { queue: topology.rewards, runMs });
  const deadline = runMs === 0 ? Infinity : Date.now() + runMs;
  while (!stopping && Date.now() < deadline) await sleep(100);
} finally {
  stopping = true;
  if (read && tag) await read.cancel(tag).catch(() => {});
  await Promise.allSettled([...pending]);
  if (conn) await conn.close().catch(() => {});
  await pool.end();
  process.removeListener('SIGTERM', stop);
  process.removeListener('SIGINT', stop);
}
if (fatal) {
  console.error('WORKER_STOPPED_WITH_ERROR', fatal.message);
  process.exitCode = 1;
}
JS
docker compose -p be-mq exec -T app node --check capstone/worker.mjs
docker compose -p be-mq exec -T -e RUN_MS=8000 app node capstone/worker.mjs | tee evidence/23-worker.txt
```

**Kết quả:** event từ Lab 21 được cộng thưởng, COMMIT rồi ACK. Các bản sao nếu có trả `duplicateDelivery` hoặc `duplicateOperation`. Worker tự dừng sau tám giây trong ô trên. `RUN_MS=0` dùng cho một process chạy đến signal; không bấm Run All để vô tình giữ worker mãi.

### Điểm cần review trong mọi consumer

Parser nằm trong try, message null được xử lý, mọi Promise của handler có catch, một delivery chỉ được ACK sau nhánh thành công hoặc chuyển giao bền vững. Dữ liệu DB dùng placeholders. Không đưa số tiền qua `parseFloat`, không dùng nhiều connection cho cùng transaction và không gọi provider ngoài trong đoạn giữ row lock.

Mã chưa có hệ thống reconnect/supervision production đầy đủ. Khi connection lỗi hoặc transfer thất bại, worker kết thúc rõ ràng để supervisor khởi động lại có backoff; schema và inbox bảo vệ redelivery. Trên VPS, bổ sung readiness riêng, restart policy có tốc độ hợp lý, alert restart liên tục và kiểm tra topology trước nhận traffic. Bật shutdown hooks trong framework chỉ cung cấp hook; bạn vẫn phải gọi và chờ các hàm đóng đúng thứ tự.

### Consumer ACK không có roundtrip confirm riêng

`read.ack(msg)` ghi lệnh vào channel; nó không trả một Promise xác nhận broker đã nhận ACK. Nếu connection đứt sau COMMIT và trước khi ACK tới nơi, delivery có thể được giao lại. Đó là lý do ta kiểm tra inbox trước business mutation mỗi lần, thay vì chỉ khi `redelivered=true`.

## Lab 24 — Chạy toàn tuyến và đối chiếu trạng thái

**Mục tiêu:** từ yêu cầu tạo đơn đến outbox, broker, inbox, ledger và event tiếp theo. Tạo file chỉ dẫn `current-order.json` để các lab sau biết chính xác order đang kiểm tra.

```bash {"name":"mq-24-end-to-end","excludeFromRunAll":"true"}
set -euo pipefail
LAB="$HOME/backend-labs/messaging"
test -f "$LAB/.be-mq-lab"
cd "$LAB"
cat > app/capstone/inspect.mjs <<'JS'
import assert from 'node:assert/strict';
import { readFile } from 'node:fs/promises';
import { pool } from '../lib/common.mjs';
const path = process.argv[2] || 'capstone/current-order.json';
const target = JSON.parse(await readFile(path, 'utf8'));
try {
  const order = await pool.query('SELECT id,status,total_minor,reward_minor FROM orders WHERE id=$1', [target.orderId]);
  const wallet = await pool.query('SELECT id,currency,balance_minor FROM wallets WHERE id=$1', [target.walletId]);
  const ledger = await pool.query('SELECT operation_key,delta_minor,event_id FROM wallet_ledger WHERE operation_key=$1', ['reward:' + target.orderId]);
  const inbox = await pool.query('SELECT consumer,event_id,processed_at FROM inbox WHERE event_id=$1', [target.eventId]);
  const outbox = await pool.query('SELECT id,routing_key,sent_at,attempts,claimed_at,parked_at,last_error FROM outbox WHERE aggregate_id=$1 ORDER BY created_at', [target.orderId]);
  console.log(JSON.stringify({ target, order: order.rows, wallet: wallet.rows, ledger: ledger.rows, inbox: inbox.rows, outbox: outbox.rows }, null, 2));
  if (process.env.ASSERT_REWARDED === '1') {
    assert.equal(ledger.rowCount, 1);
    assert.equal(inbox.rowCount, 1);
    assert.equal(ledger.rows[0].delta_minor, order.rows[0].reward_minor);
    const summed = await pool.query('SELECT COALESCE(sum(delta_minor),0)::text AS balance FROM wallet_ledger WHERE wallet_id=$1', [target.walletId]);
    assert.equal(wallet.rows[0].balance_minor, summed.rows[0].balance);
  }
} finally { await pool.end(); }
JS
docker compose -p be-mq exec -T app node capstone/create-order.mjs > app/capstone/current-order.json
order_id="$(docker compose -p be-mq exec -T app node -p "JSON.parse(require('node:fs').readFileSync('capstone/current-order.json')).orderId")"
docker compose -p be-mq exec -T -e "ONLY_AGGREGATE_ID=$order_id" app node capstone/relay.mjs
docker compose -p be-mq exec -T -e RUN_MS=8000 app node capstone/worker.mjs
docker compose -p be-mq exec -T -e "ONLY_AGGREGATE_ID=$order_id" app node capstone/relay.mjs
docker compose -p be-mq exec -T -e ASSERT_REWARDED=1 app node capstone/inspect.mjs | tee evidence/24-end-to-end.json
docker compose -p be-mq exec -T rabbit rabbitmqctl list_queues -p /lab name messages_ready messages_unacknowledged consumers
```

**Kết quả:** order PAID; một ledger cộng 5.000; một inbox cho event nguồn; outbox `order.paid` và `wallet.rewarded` đã sent sau relay thứ hai. Notification queue còn message vì chưa xây adapter SMTP; nghĩa vụ gửi thông báo đang được giữ ở queue. Không dùng việc queue rỗng làm điều kiện “toàn hệ thống tốt” khi có consumer chưa triển khai.

**Vì sao relay chạy hai lần?** Lần đầu phát event từ order. Rewards worker commit số dư và tạo event mới trong transaction của nó. Lần thứ hai phát event đó. Trong production, relay thường chạy liên tục có backoff khi không có việc, hoặc CDC theo log database. Kiến trúc causal chain phải hiện rõ để đội vận hành hiểu backlog nằm ở bước nào.

**Tự kiểm tra:** nếu trace log ghi `OUTBOX_SENT` nhưng wallet vẫn 0, các khả năng gồm worker chưa chạy, binding sai, payload vào parking, DB lỗi hoặc backlog. Đọc traceId/eventId và trạng thái ở từng biên; không sửa bằng cách cộng tiền thủ công trước khi đối chiếu ledger.

## Lab 25 — Crash sau broker confirm nhưng trước sent UPDATE

**Đây là bài fault injection cục bộ.** Process relay tự thoát mã 86 sau một publish đã confirm. PostgreSQL và RabbitMQ vẫn chạy. Bài chờ lease 15 giây hết rồi publish lại cùng event.

**Mục tiêu:** nhìn thấy duplicate delivery là hệ quả có thể dự đoán, trong khi ledger vẫn chỉ có một effect.

```bash {"name":"mq-25-crash-after-confirm","excludeFromRunAll":"true"}
set -euo pipefail
LAB="$HOME/backend-labs/messaging"
test -f "$LAB/.be-mq-lab"
cd "$LAB"
docker compose -p be-mq exec -T app node capstone/create-order.mjs > app/capstone/current-order.json
order_id="$(docker compose -p be-mq exec -T app node -p "JSON.parse(require('node:fs').readFileSync('capstone/current-order.json')).orderId")"
set +e
docker compose -p be-mq exec -T -e "ONLY_AGGREGATE_ID=$order_id" -e CLAIM_LEASE_SECONDS=15 -e CRASH_AFTER_CONFIRM=1 app node capstone/relay.mjs
crash_status=$?
set -e
test "$crash_status" -eq 86
docker compose -p be-mq exec -T app node capstone/inspect.mjs
docker compose -p be-mq exec -T app node --input-type=module - <<'JS'
import { pool, until } from './lib/common.mjs';
import { readFile } from 'node:fs/promises';
const target = JSON.parse(await readFile('capstone/current-order.json','utf8'));
try {
  await until(async () => {
    const rows = await pool.query(
      "SELECT 1 FROM outbox WHERE id=$1 AND sent_at IS NULL AND claimed_at<clock_timestamp()-interval '15 seconds'",
      [target.eventId]
    );
    return rows.rowCount === 1;
  }, 25000, 'claim lease expiration');
  console.log('LEASE_EXPIRED');
} finally { await pool.end(); }
JS
docker compose -p be-mq exec -T -e "ONLY_AGGREGATE_ID=$order_id" -e CLAIM_LEASE_SECONDS=15 app node capstone/relay.mjs
docker compose -p be-mq exec -T -e RUN_MS=8000 app node capstone/worker.mjs
docker compose -p be-mq exec -T -e ASSERT_REWARDED=1 app node capstone/inspect.mjs | tee evidence/25-crash-confirm.json
```

**Kết quả:** trước khi lease hết, outbox sent_at vẫn null nhưng broker có message. Sau phục hồi, worker có thể log một lần credited và một lần duplicate. SQL assertion vẫn cho một ledger và số dư phù hợp.

**Không nên “sửa” bằng cách tăng lease vô hạn.** Lease dài giảm xác suất overlap nhưng kéo dài hồi phục khi owner thật sự chết. Lease ngắn phục hồi nhanh hơn nhưng nhạy với pause và network. Chọn dựa trên publish timeout, load, GC, clock và SLO; đo tuổi claim, stale owner và tỷ lệ duplicate.

Clock trong lease dùng PostgreSQL để các relay không so sánh đồng hồ máy khác nhau. Điều đó không loại bỏ mọi vấn đề thời gian, nhưng tránh dùng Date.now của từng VPS cho quyền sở hữu hàng outbox.

## Lab 26 — Crash sau COMMIT nhưng trước consumer ACK

**Mục tiêu:** lần đầu worker đã cộng tiền nhưng chưa ACK; lần sau dùng inbox để ACK mà không cộng tiếp.

```bash {"name":"mq-26-crash-after-db-commit","excludeFromRunAll":"true"}
set -euo pipefail
LAB="$HOME/backend-labs/messaging"
test -f "$LAB/.be-mq-lab"
cd "$LAB"
docker compose -p be-mq exec -T app node capstone/create-order.mjs > app/capstone/current-order.json
order_id="$(docker compose -p be-mq exec -T app node -p "JSON.parse(require('node:fs').readFileSync('capstone/current-order.json')).orderId")"
event_id="$(docker compose -p be-mq exec -T app node -p "JSON.parse(require('node:fs').readFileSync('capstone/current-order.json')).eventId")"
docker compose -p be-mq exec -T -e "ONLY_AGGREGATE_ID=$order_id" app node capstone/relay.mjs
set +e
docker compose -p be-mq exec -T -e CRASH_AFTER_COMMIT=1 -e "CRASH_ON_EVENT_ID=$event_id" -e RUN_MS=12000 app node capstone/worker.mjs
crash_status=$?
set -e
test "$crash_status" -eq 87
docker compose -p be-mq exec -T -e ASSERT_REWARDED=1 app node capstone/inspect.mjs
docker compose -p be-mq exec -T -e RUN_MS=8000 app node capstone/worker.mjs
docker compose -p be-mq exec -T -e ASSERT_REWARDED=1 app node capstone/inspect.mjs | tee evidence/26-crash-commit.json
```

**Điều phải thấy:** ngay sau crash, DB đã có ledger/inbox; sau lần chạy lại, balance không tăng thêm. Có thể có một khoảng broker cần phát hiện connection đóng trước redelivery; deadline không thay thế quan sát ready/unacked.

**Mở rộng phân tích:** nếu COMMIT đã tới PostgreSQL nhưng response bị mất, helper transaction có thể trả lỗi chưa biết. Retry của consumer vẫn an toàn khi transaction cũ thực tế commit, vì inbox đã cùng commit. Nếu transaction cũ rollback, inbox không tồn tại và lần sau được phép làm. Đó là ý nghĩa thiết thực của cùng transaction.

Với email provider ngoài DB, không thể dùng kết luận tương tự nếu provider không có idempotency. Một bảng “email_sent” cùng DB chỉ ghi điều bạn biết, không có quyền rollback SMTP. Thiết kế tốt có status requested/sending/unknown/sent, provider key, webhook kết quả nếu có và quy trình reconciliation phù hợp.

## Lab 27 — Broker ngừng và nhiều relay nhận việc đồng thời

**Pha A:** dừng broker, vẫn commit order/outbox, xác minh relay chưa thể phát, rồi khôi phục. Bài chỉ dừng `rabbit` thuộc project cục bộ. Trap khôi phục service ngay cả khi một bước lỗi.

```bash {"name":"mq-27-broker-outage","excludeFromRunAll":"true"}
set -euo pipefail
LAB="$HOME/backend-labs/messaging"
test -f "$LAB/.be-mq-lab"
cd "$LAB"
restore_broker() {
  docker compose -p be-mq start rabbit >/dev/null
}
trap restore_broker EXIT INT TERM
docker compose -p be-mq stop rabbit
docker compose -p be-mq exec -T app node capstone/create-order.mjs > app/capstone/current-order.json
order_id="$(docker compose -p be-mq exec -T app node -p "JSON.parse(require('node:fs').readFileSync('capstone/current-order.json')).orderId")"
set +e
docker compose -p be-mq exec -T -e "ONLY_AGGREGATE_ID=$order_id" app node capstone/relay.mjs
relay_status=$?
set -e
test "$relay_status" -ne 0
docker compose -p be-mq exec -T app node capstone/inspect.mjs
restore_broker
trap - EXIT INT TERM
docker compose -p be-mq up -d --wait --wait-timeout 120 rabbit
docker compose -p be-mq exec -T -e "ONLY_AGGREGATE_ID=$order_id" app node capstone/relay.mjs
docker compose -p be-mq exec -T -e RUN_MS=8000 app node capstone/worker.mjs
docker compose -p be-mq exec -T -e ASSERT_REWARDED=1 app node capstone/inspect.mjs | tee evidence/27-broker-outage.json
```

**Kết quả:** create-order hoạt động vì chỉ phụ thuộc DB; relay lỗi trong lúc broker dừng; event vẫn có thể phát sau phục hồi. Đây không phải lý do để API chấp nhận vô hạn: backlog outbox vẫn chiếm DB/disk và người dùng vẫn chờ. Khi queue age vượt SLO hoặc storage gần ngưỡng, API cần admission control hoặc công khai độ trễ dự kiến.

**Pha B:** tạo 20 đơn độc lập, chạy hai relay processes cùng lúc, xử lý rồi kiểm tra. Một process parent trong Node sinh hai processes relay; không dùng PID file chung hay kill theo tên process.

```bash {"name":"mq-27-parallel-relays","excludeFromRunAll":"true"}
set -euo pipefail
LAB="$HOME/backend-labs/messaging"
test -f "$LAB/.be-mq-lab"
cd "$LAB"
cat > app/labs/27-parallel-relay.mjs <<'JS'
import assert from 'node:assert/strict';
import { spawn } from 'node:child_process';
import { pool, randomUUID } from '../lib/common.mjs';
import { createPaidOrder } from '../lib/domain.mjs';
function run(script, extra = {}) {
  return new Promise((resolve, reject) => {
    const child = spawn(process.execPath, [script], {
      stdio: 'inherit', env: { ...process.env, ...extra }
    });
    child.once('error', reject);
    child.once('exit', code => code === 0 ? resolve() : reject(new Error(script + ' exited ' + code)));
  });
}
const prefix = 'parallel-' + randomUUID();
const orders = [];
try {
  for (let i = 0; i < 20; i++) {
    orders.push(await createPaidOrder({
      clientKey: prefix + '-' + i, walletId: prefix + '-wallet-' + i,
      totalMinor: 100000, rewardMinor: 5000
    }));
  }
  await Promise.all([
    run('capstone/relay.mjs', { MAX_EVENTS: '200', ONLY_AGGREGATE_ID: '' }),
    run('capstone/relay.mjs', { MAX_EVENTS: '200', ONLY_AGGREGATE_ID: '' })
  ]);
  await run('capstone/worker.mjs', { RUN_MS: '10000' });
  const ids = orders.map(order => order.orderId);
  const result = await pool.query(
    "SELECT count(*)::int AS count,sum(delta_minor)::text AS total FROM wallet_ledger WHERE operation_key=ANY($1::text[])",
    [ids.map(id => 'reward:' + id)]
  );
  console.log('PARALLEL_RELAY_RESULT', result.rows[0]);
  assert.equal(result.rows[0].count, 20);
  assert.equal(result.rows[0].total, '100000');
} finally { await pool.end(); }
JS
docker compose -p be-mq exec -T app node labs/27-parallel-relay.mjs | tee evidence/27-parallel-relays.txt
```

**Giới hạn:** hai relay có thể publish các events khác nhau ngoài thứ tự created_at vì lock scheduling, network và confirms. Bài có một order.paid mỗi aggregate nên không phụ thuộc thứ tự giữa aggregates. Khi cùng order có version 1,2,3 cần consumer kiểm tra version hoặc relay partition theo aggregate. SKIP LOCKED không tự giữ thứ tự nghiệp vụ.

Hãy quan sát outbox attempts và số event thực nhận để biết mức duplicate. Không assert rằng “mỗi event chỉ publish một lần” làm điều kiện test toàn hệ thống, vì đó không phải cam kết thiết kế. Assertion đáng giá nằm ở ledger, nghĩa vụ còn pending và trạng thái nghiệp vụ.

---

# Phần VI. Poison message, schema evolution, replay và reconciliation

## 9. Một DLQ hữu ích phải có người sở hữu và quy trình

DLQ hoặc parking queue giữ message không thể xử lý tự động. Nó không phải thùng rác để hệ thống trông “healthy”. Mỗi nhóm cần biết:

- Message đến từ nghiệp vụ nào, tenant nào, eventId và schemaVersion nào.
- Lỗi là dữ liệu, code, cấu hình, dependency hay kết quả chưa biết.
- Ai được replay, với code/schema nào và số lượng tối đa bao nhiêu.
- Replay có giữ nguyên eventId và idempotency key không.
- Nếu sửa payload, đó là event gốc được hiệu chỉnh có kiểm soát hay một event correction mới.
- Làm sao chứng minh không còn nghĩa vụ bị bỏ sau replay.

Đừng log toàn bộ payload chỉ vì đang điều tra. Payload có thể chứa thông tin cá nhân, access token hoặc dữ liệu lớn. Tách metadata chẩn đoán khỏi nội dung; giới hạn quyền đọc DLQ và retention. Một thông điệp độc còn có thể là JSON khổng lồ làm worker tốn CPU/RAM dù parse thành công.

## Lab 28 — Gửi schema không hỗ trợ và chứng minh quarantine

**Mục tiêu:** poison message không bị ném lại main queue vô hạn. Bài gửi schemaVersion 99 vào routing `order.paid`; worker từ chối ngay, không tạo inbox hay ledger.

```bash {"name":"mq-28-schema-quarantine","excludeFromRunAll":"true"}
set -euo pipefail
LAB="$HOME/backend-labs/messaging"
test -f "$LAB/.be-mq-lab"
cd "$LAB"
cat > app/labs/28-poison.mjs <<'JS'
import { writeFile } from 'node:fs/promises';
import { rabbit, channel, confirmSend, randomUUID } from '../lib/common.mjs';
import { makeOrderPaid } from '../lib/domain.mjs';
import { ensureTopology, topology } from '../lib/topology.mjs';
const conn = await rabbit();
const event = makeOrderPaid({
  orderId: randomUUID(), walletId: 'poison-synthetic-wallet',
  totalMinor: 100000, rewardMinor: 5000
});
event.schemaVersion = 99;
try {
  const pub = await channel(conn, true);
  await ensureTopology(pub);
  await confirmSend(pub, topology.events, 'order.paid', Buffer.from(JSON.stringify(event)), {
    messageId: event.eventId, type: event.eventType
  });
  await writeFile('capstone/poison-event.json', JSON.stringify(event, null, 2));
  console.log('POISON_PUBLISHED', event.eventId);
} finally { await conn.close(); }
JS
docker compose -p be-mq exec -T app node labs/28-poison.mjs
docker compose -p be-mq exec -T -e RUN_MS=5000 app node capstone/worker.mjs | tee evidence/28-quarantine.txt
docker compose -p be-mq exec -T rabbit rabbitmqctl list_queues -p /lab name messages_ready messages_unacknowledged
docker compose -p be-mq exec -T app node --input-type=module - <<'JS'
import assert from 'node:assert/strict';
import { readFile } from 'node:fs/promises';
import { pool } from './lib/common.mjs';
const event = JSON.parse(await readFile('capstone/poison-event.json','utf8'));
try {
  const count = await pool.query('SELECT 1 FROM inbox WHERE event_id=$1', [event.eventId]);
  assert.equal(count.rowCount, 0);
  console.log('NO_BUSINESS_MUTATION_FOR_UNSUPPORTED_SCHEMA', event.eventId);
} finally { await pool.end(); }
JS
```

**Quan sát:** log `DELIVERY_PARKED` với lý do unsupported schema; `mq.parking` tăng. Không dùng `get(... noAck:true)` để “xem nhanh” vì thao tác đó tiêu thụ message. Management UI cũng có tùy chọn ACK/requeue; đọc đúng mode trước khi bấm.

### Schema evolution theo hướng tương thích

Nếu thêm trường tùy chọn như `campaignId`, consumer cũ thường có thể bỏ qua. Nếu đổi `amount` từ đơn vị đồng sang nghìn đồng mà không đổi tên/version, JSON vẫn parse thành công nhưng ý nghĩa sai. Đây là lỗi nguy hiểm hơn syntax error vì dễ đi qua mọi healthcheck.

Một quy trình triển khai có thể là: consumer hỗ trợ cả v1/v2 và đo tỷ lệ → producer bắt đầu phát v2 → đợi backlog/replay window của v1 hoặc giữ upcaster → loại bỏ v1 có kế hoạch. Không xóa khả năng đọc v1 chỉ vì deploy mới đã xong: backup hoặc DLQ có thể chứa payload cũ.

Upcaster là hàm thuần biến đổi dạng dữ liệu cũ sang mô hình nội bộ, với kiểm thử ví dụ thật đã ẩn dữ liệu nhạy cảm. Nó không nên gọi network hoặc suy đoán thông tin không có trong event. Nếu dữ liệu bắt buộc không tồn tại, quarantine với lý do cụ thể tốt hơn tạo giá trị giả.

## Lab 29 — Replay có audit, giữ ID và hạn chế số lượng

**Mục tiêu:** xem metadata một message parking mà không mất nó, sau đó replay có chủ đích chỉ một message đã tương thích. Poison v99 ở Lab 28 sẽ bị chặn; để demo replay hợp lệ, bài tạo một event v1 của một order mới và chuyển nó thẳng vào parking với lý do synthetic.

Không sửa `eventId` để né inbox. Replay là cơ hội thực hiện nghĩa vụ còn thiếu; nếu nghĩa vụ đã xong, duplicate response là kết quả đúng.

```bash {"name":"mq-29-controlled-replay","excludeFromRunAll":"true"}
set -euo pipefail
LAB="$HOME/backend-labs/messaging"
test -f "$LAB/.be-mq-lab"
cd "$LAB"
cat > app/capstone/replay-one.mjs <<'JS'
import { pool, rabbit, channel, confirmSend, PermanentError, log } from '../lib/common.mjs';
import { validateOrderPaid } from '../lib/domain.mjs';
import { topology } from '../lib/topology.mjs';
const expectedEventId = process.env.REPLAY_EVENT_ID;
const operator = process.env.REPLAY_OPERATOR || 'local-student';
const reason = process.env.REPLAY_REASON || 'local training recovery drill';
const maxScan = 50;
const conn = await rabbit();
const read = await channel(conn);
const pub = await channel(conn, true);
const held = [];
let found = false;
try {
  if (!expectedEventId) throw new Error('REPLAY_EVENT_ID is required');
  for (let i = 0; i < maxScan; i++) {
    const msg = await read.get(topology.parking, { noAck: false });
    if (!msg) break;
    if (msg.properties.messageId !== expectedEventId) {
      held.push(msg);
      continue;
    }
    found = true;
    let event;
    try { event = JSON.parse(msg.content.toString()); validateOrderPaid(event); }
    catch (error) {
      held.push(msg);
      await pool.query('INSERT INTO replay_audit(event_id,operator_name,reason,outcome) VALUES($1,$2,$3,$4)',
        [expectedEventId, operator, reason, 'blocked: ' + error.message.slice(0,120)]);
      throw new PermanentError('REPLAY_BLOCKED_UNSUPPORTED_PAYLOAD');
    }
    await pool.query('INSERT INTO replay_audit(event_id,operator_name,reason,outcome) VALUES($1,$2,$3,$4)',
      [expectedEventId, operator, reason, 'requested']);
    await confirmSend(pub, topology.events, 'order.paid', msg.content, {
      messageId: event.eventId,
      type: event.eventType,
      headers: {
        ...(msg.properties.headers || {}),
        'x-app-attempt': 0,
        'x-replay-reason': reason.slice(0,120),
        'x-replayed-by': operator.slice(0,120)
      }
    });
    read.ack(msg);
    await pool.query('INSERT INTO replay_audit(event_id,operator_name,reason,outcome) VALUES($1,$2,$3,$4)',
      [expectedEventId, operator, reason, 'publish-confirmed']);
    log('REPLAY_CONFIRMED', { eventId: event.eventId });
    break;
  }
  if (!found) throw new Error('Target not found within scan limit; do not drain whole queue blindly');
} finally {
  // Các message không chọn chưa ACK; đóng channel để trả lại queue.
  await read.close().catch(() => {});
  await conn.close().catch(() => {});
  await pool.end();
}
JS
cat > app/labs/29-seed-replay.mjs <<'JS'
import { writeFile } from 'node:fs/promises';
import { pool, randomUUID, rabbit, channel, confirmSend } from '../lib/common.mjs';
import { createPaidOrder } from '../lib/domain.mjs';
import { ensureTopology, topology } from '../lib/topology.mjs';
let conn;
try {
  const key = 'replay-' + randomUUID();
  const target = await createPaidOrder({ clientKey: key, walletId: 'wallet-' + key, totalMinor: 100000, rewardMinor: 5000 });
  const row = await pool.query('SELECT payload FROM outbox WHERE id=$1', [target.eventId]);
  conn = await rabbit();
  const pub = await channel(conn, true);
  await ensureTopology(pub);
  await confirmSend(pub, topology.parkingExchange, 'parked', Buffer.from(JSON.stringify(row.rows[0].payload)), {
    messageId: target.eventId, type: 'order.paid', headers: { 'x-error-code': 'SYNTHETIC_REPLAY_DRILL' }
  });
  await writeFile('capstone/current-order.json', JSON.stringify(target, null, 2));
  console.log('REPLAY_TARGET', target.eventId);
} finally {
  if (conn) await conn.close();
  await pool.end();
}
JS
docker compose -p be-mq exec -T app node labs/29-seed-replay.mjs
event_id="$(docker compose -p be-mq exec -T app node -p "JSON.parse(require('node:fs').readFileSync('capstone/current-order.json')).eventId")"
docker compose -p be-mq exec -T -e "REPLAY_EVENT_ID=$event_id" -e REPLAY_OPERATOR=local-student -e REPLAY_REASON=training-approved-one-event app node capstone/replay-one.mjs
docker compose -p be-mq exec -T -e RUN_MS=6000 app node capstone/worker.mjs
docker compose -p be-mq exec -T -e ASSERT_REWARDED=1 app node capstone/inspect.mjs | tee evidence/29-replay.json
```

**Kết quả:** event tương thích được republish và cộng thưởng một lần; poison v99 khác được giữ lại. Việc giữ tạm tối đa 50 deliveries rồi đóng channel sẽ thay đổi thứ tự/requeue metadata của những message không chọn. Đây là công cụ học quy mô nhỏ. Với DLQ lớn, xây công cụ export/index an toàn hoặc quarantine table để chọn theo ID, không scan hàng triệu message bằng get.

**Một cửa sổ lỗi vẫn còn:** replay đã confirm rồi process chết trước source ACK hoặc trước ghi audit “confirmed”. Message có thể còn ở parking dù đã quay lại main. Audit “requested” nhưng chưa “confirmed” là trạng thái cần đối chiếu, không tự suy ra chưa gửi. Inbox/ledger tiếp tục bảo vệ.

Outbox nguồn của seed vẫn pending có chủ đích. Relay sau này có thể phát thêm cùng event. Đó là một bài kiểm tra nữa: replay không được đòi xóa outbox hoặc thay ID để trông như không có duplicate. Vận hành thật cần xem nguồn nghĩa vụ trước khi replay và ghi lại lý do đường phục hồi được chọn.

## Lab 30 — Reconciliation: tìm nghĩa vụ thiếu và phát lại an toàn

**Mục tiêu:** tìm orders PAID chưa có ledger cộng thưởng, kiểm tra outbox gốc còn tồn tại và đưa ý định chưa hoàn thành trở lại relay. Bài tách report và repair để luôn đọc được danh sách trước khi mutation.

Reconciliation không đơn giản là “queue rỗng chưa”. Queue có thể rỗng vì consumer đã ACK quá sớm hoặc message bị xóa. Nguồn cần đối chiếu là bất biến nghiệp vụ: mỗi order PAID đủ điều kiện cần có một operation ledger. Trong lab này tất cả order có reward_minor > 0 đều thuộc phạm vi; hệ thống thật cần thêm trạng thái hoàn tiền, chiến dịch, thời hạn và ngoại lệ được duyệt.

```bash {"name":"mq-30-reconciliation-report","excludeFromRunAll":"true"}
set -euo pipefail
LAB="$HOME/backend-labs/messaging"
test -f "$LAB/.be-mq-lab"
cd "$LAB"
cat > app/capstone/reconcile.mjs <<'JS'
import { readFile } from 'node:fs/promises';
import { pool, tx } from '../lib/common.mjs';
const target = JSON.parse(await readFile('capstone/current-order.json','utf8'));
const repair = process.env.REPAIR_ONE === '1';
try {
  const result = await pool.query(
    "SELECT o.id,o.status,o.reward_minor,b.id AS event_id,b.sent_at,b.claimed_at,b.parked_at FROM orders o LEFT JOIN wallet_ledger l ON l.operation_key='reward:'||o.id LEFT JOIN outbox b ON b.aggregate_id=o.id AND b.routing_key='order.paid' WHERE o.id=$1 AND o.status='PAID' AND o.reward_minor>0 AND l.operation_key IS NULL",
    [target.orderId]
  );
  console.log('MISSING_REWARD', JSON.stringify(result.rows, null, 2));
  if (repair && result.rowCount !== 0) {
  if (result.rowCount !== 1 || !result.rows[0].event_id) throw new Error('SOURCE_EVENT_MISSING_REQUIRES_MANUAL_REVIEW');
  await tx(async client => {
    const row = await client.query(
      'SELECT id,claimed_at FROM outbox WHERE id=$1 FOR UPDATE',
      [result.rows[0].event_id]
    );
    if (row.rows[0].claimed_at) throw new Error('ACTIVE_OR_STALE_CLAIM_REQUIRES_LEASE_REVIEW');
    await client.query(
      "UPDATE outbox SET sent_at=NULL,parked_at=NULL,next_attempt_at=clock_timestamp(),last_error='reconciliation requested' WHERE id=$1",
      [result.rows[0].event_id]
    );
    await client.query(
      'INSERT INTO replay_audit(event_id,operator_name,reason,outcome) VALUES($1,$2,$3,$4)',
      [result.rows[0].event_id, 'local-student', 'business reconciliation of one order', 'outbox reset for same eventId']
    );
  });
  console.log('RECONCILIATION_REPAIR_QUEUED', target.orderId);
  }
} finally { await pool.end(); }
JS
docker compose -p be-mq exec -T app node capstone/create-order.mjs > app/capstone/current-order.json
docker compose -p be-mq exec -T app node capstone/reconcile.mjs | tee evidence/30-reconciliation-report.txt
```

Ô tạo report phía trên cố ý có một order mới chưa relay, nên phải thấy một nghĩa vụ thiếu. Nếu một hệ thống thật vừa tạo order 10 ms trước, đó chưa là sự cố. Báo cáo vận hành cần khoảng grace hợp với SLO trước khi cảnh báo hoặc sửa.

```bash {"name":"mq-30-reconciliation-repair-one","excludeFromRunAll":"true"}
set -euo pipefail
LAB="$HOME/backend-labs/messaging"
test -f "$LAB/.be-mq-lab"
cd "$LAB"
docker compose -p be-mq exec -T -e REPAIR_ONE=1 app node capstone/reconcile.mjs
order_id="$(docker compose -p be-mq exec -T app node -p "JSON.parse(require('node:fs').readFileSync('capstone/current-order.json')).orderId")"
docker compose -p be-mq exec -T -e "ONLY_AGGREGATE_ID=$order_id" app node capstone/relay.mjs
docker compose -p be-mq exec -T -e RUN_MS=6000 app node capstone/worker.mjs
docker compose -p be-mq exec -T app node capstone/reconcile.mjs
docker compose -p be-mq exec -T -e ASSERT_REWARDED=1 app node capstone/inspect.mjs | tee evidence/30-reconciliation-repaired.json
```

**Kết quả:** report sau xử lý không còn missing reward cho order đó. Không cộng thẳng vào wallet để sửa số dư, vì làm vậy bỏ qua ledger, event nguồn, audit và khả năng retry. Một lệnh repair tốt thường tạo lại nghĩa vụ qua đường idempotent đã có.

### Khi restore từ backup

Restore DB về một mốc có thể làm inbox quên các message đã xử lý sau mốc đó. Broker có thể vẫn giữ message mới hơn, hoặc provider đã thực hiện side effect mà DB chưa nhớ. Vì thế backup/restore của từng thành phần không tự tạo một snapshot toàn hệ thống. Cần xác định điểm phục hồi, giữ event archive/ledger đủ lâu, đối chiếu provider và kiểm soát replay.

Đôi khi phục hồi một hệ thống đúng cần để queue dừng nhận việc, khôi phục dữ liệu nguồn, chạy reconciliation và chỉ sau đó mở workers. Quy trình này phải diễn tập; một tệp backup tồn tại nhưng chưa từng restore không chứng minh RTO.

## Lab 31 — Saga: compensation là nghiệp vụ mới có idempotency

**Bài toán:** giữ tồn kho, xác nhận thanh toán rồi yêu cầu giao hàng. Nếu giao hàng thất bại cuối cùng, có thể phải hoàn tiền và nhả tồn. Đó không là rollback một transaction SQL xuyên mọi service. Mỗi bước đã commit ở hệ thống riêng.

Lab dưới đây mô hình hóa state machine bền trong PostgreSQL cho một saga tổng hợp. Nó không gọi provider thật. Mục tiêu là thấy trạng thái mỗi bước được ghi, compensation dùng key riêng và việc chạy lại không làm compensation hai lần.

```bash {"name":"mq-31-saga-compensation","excludeFromRunAll":"true"}
set -euo pipefail
LAB="$HOME/backend-labs/messaging"
test -f "$LAB/.be-mq-lab"
cd "$LAB"
cat > app/labs/31-saga.mjs <<'JS'
import assert from 'node:assert/strict';
import { pool, tx, randomUUID } from '../lib/common.mjs';
const sagaId = 'saga-' + randomUUID();
async function record(step, result) {
  return tx(async client => {
    const inserted = await client.query(
      "INSERT INTO saga_steps(saga_id,step,status,result) VALUES($1,$2,'DONE',$3::jsonb) ON CONFLICT(saga_id,step) DO NOTHING RETURNING step",
      [sagaId, step, JSON.stringify(result)]
    );
    return inserted.rowCount === 1;
  });
}
async function compensate() {
  await record('refund-payment', { providerIdempotencyKey: sagaId + '-refund', simulated: true });
  await record('release-stock', { reservationId: sagaId + '-stock', simulated: true });
}
try {
  await record('reserve-stock', { reservationId: sagaId + '-stock' });
  await record('capture-payment', { providerIdempotencyKey: sagaId + '-capture' });
  await record('shipment-failed', { reason: 'SYNTHETIC_ADDRESS_UNSERVICEABLE' });
  await compensate();
  await compensate();
  const rows = await pool.query('SELECT step,status,result FROM saga_steps WHERE saga_id=$1 ORDER BY step', [sagaId]);
  console.table(rows.rows);
  assert.equal(rows.rowCount, 5);
  assert.equal(rows.rows.filter(row => row.step === 'refund-payment').length, 1);
} finally { await pool.end(); }
JS
docker compose -p be-mq exec -T app node labs/31-saga.mjs | tee evidence/31-saga.txt
```

**Đừng copy sai ý nghĩa của record:** bài chỉ ghi trạng thái giả lập trong DB, nên unique insert đủ để chứng minh record không lặp. Nếu thay phần result bằng gọi API hoàn tiền trước/sau INSERT, bạn lại có dual-write. Saga thật cần trạng thái requested/in-flight/unknown/completed, outbox gửi command, provider key ổn định và reconciliation; kết quả hoàn tiền chỉ chuyển completed sau bằng chứng phù hợp.

### Quyết định compensation

Không phải bước nào cũng bù được. Email đã đọc không thể thu hồi. Một hàng đã giao phải xử lý trả hàng thay vì “undo ship”. Hoàn tiền cũng có thể thất bại hoặc bị treo ở provider. Saga cần trạng thái `COMPENSATION_PENDING`, retry budget riêng, alert và quyền can thiệp. “Catch rồi gọi refund” chưa là một hệ thống saga.

Khi timeout giữ tồn đến trước callback thanh toán muộn, state machine phải có quy tắc. Có thể từ chối callback mới, tạo quy trình hoàn tiền tự động, hoặc tái giữ tồn nếu còn hàng. Quyết định là nghiệp vụ; broker chỉ vận chuyển các thông điệp phản ánh quyết định đó.

### Ordering, version và causation

Một event nên chứa aggregate ID và version nếu consumer cần phát hiện khoảng trống hoặc dữ liệu cũ. `eventId` nói về danh tính thông điệp; `correlationId` nối một hành trình; `causationId` trỏ tới nguyên nhân trực tiếp; `traceId` phục vụ quan sát. Dùng một trường cho tất cả mục đích làm việc debug và chống trùng khó hơn.

Nếu snapshot version 9 đến trước 8, projection có thể bỏ 8 bằng UPDATE WHERE version < incoming. Nếu event là delta “cộng 5” rồi “trừ 2”, bỏ event version cũ một cách mù quáng có thể mất tác dụng phụ. Thiết kế xử lý out-of-order phụ thuộc loại dữ kiện, không có một công thức version dùng cho mọi event.



---

# Phần VII. Capacity, quan sát và triển khai VPS

## 10. Đo công việc mà người dùng đang chờ

Một dashboard chỉ có messages ready và CPU chưa đủ. Công việc có thể chuyển từ ready sang unacked rồi treo ở worker; queue nhìn nhỏ hơn nhưng người dùng chưa nhận kết quả. Ngược lại, một queue có 10.000 delayed jobs hẹn tuần sau không nhất thiết là sự cố hôm nay.

Tách thời gian:

- **Enqueue latency:** từ yêu cầu tới khi broker/DB chấp nhận nghĩa vụ.
- **Queue wait:** từ khi đủ điều kiện chạy tới khi worker bắt đầu.
- **Service time:** từ bắt đầu tới khi xử lý xong.
- **End-to-end latency:** từ yêu cầu nghiệp vụ tới kết quả người dùng cần.
- **Age of oldest due work:** tuổi công việc đã đến hạn lâu nhất chưa hoàn thành.
- **Outbox lag:** tuổi hoặc khoảng cách của nghĩa vụ chưa publish.
- **Retry age:** tổng thời gian một nghĩa vụ đã trải qua, gồm chờ thử lại.

Với delayed jobs, dùng timestamp đến hạn để phân biệt chờ chủ đích với trễ. Với message từ máy khác, `occurredAt` có thể chịu clock skew; đồng hồ hệ thống cần được đồng bộ và số đo latency cần hiểu phạm vi. Trace giúp điều tra một trường hợp; metrics giúp nhìn toàn bộ phân phối.

### Ước tính capacity trước khi load test

Nếu một job cần trung bình `S` giây làm việc và có `C` slots thực sự hữu ích, công suất lý tưởng gần `C/S` jobs/giây. Đây là mô hình sơ bộ, chưa tính bottleneck DB, provider quota, GC, CPU, disk hoặc serialization. Khi sử dụng gần 100%, biến động service time khiến thời gian chờ tăng mạnh; cần headroom.

Ví dụ giả định: đầu vào 80 jobs/giây, service time trung bình 0,12 giây, mục tiêu utilization khoảng 70%. Ước tính ban đầu `C ≥ 80 × 0,12 / 0,7 ≈ 13,7`, tức khoảng 14 slots. Nếu provider chỉ cho 50 requests/giây và mỗi job gọi provider một lần, thêm slots không làm throughput bền vững vượt 50. Đầu vào 80 cần admission control, quota cao hơn hoặc thiết kế ít request hơn.

Nếu backlog là `B`, tốc độ xử lý `μ` và đầu vào hiện tại `λ`, thời gian drain xấp xỉ `B/(μ−λ)` khi `μ>λ` và các tốc độ tương đối ổn định. Với 24.000 jobs, xử lý 120/s và nhận mới 80/s, mất khoảng 600 giây để hết nợ. Nếu μ bằng λ, backlog không giảm. Tắt cảnh báo chỉ vì consumer đã chạy lại bỏ qua khoảng thời gian phục hồi này.

Little’s Law `L=λW` liên hệ số công việc trung bình trong hệ thống và thời gian lưu trung bình khi hệ thống ổn định trong khoảng quan sát. Không áp nó máy móc vào một burst đang tăng vô hạn hoặc thay p99 vào công thức của trung bình. Các phép tính trên là giả định minh họa; số đo lab của bạn mới cho thấy baseline thực tế.

## Lab 32 — Load nhỏ, latency phân vị và ảnh chụp sức khỏe

**Mục tiêu:** tạo burst hữu hạn, đo p50/p95 queue wait và service time, thấy backlog tăng trong khi worker hoạt động. Không bắn lưu lượng ra provider ngoài.

```bash {"name":"mq-32-bounded-load-metrics","excludeFromRunAll":"true"}
set -euo pipefail
LAB="$HOME/backend-labs/messaging"
test -f "$LAB/.be-mq-lab"
cd "$LAB"
cat > app/labs/32-load.mjs <<'JS'
import assert from 'node:assert/strict';
import { performance } from 'node:perf_hooks';
import { bullLab, sleep } from '../lib/common.mjs';
const lab = await bullLab('load');
const waitMs = [];
const serviceMs = [];
let completed = 0;
let active = 0;
let peak = 0;
function quantile(values, q) {
  const sorted = [...values].sort((a,b) => a-b);
  return sorted[Math.max(0, Math.ceil(q * sorted.length) - 1)];
}
try {
  await lab.queue.setGlobalConcurrency(4);
  const processor = async job => {
    const start = performance.now();
    waitMs.push(Date.now() - job.timestamp);
    active++; peak = Math.max(peak, active);
    try {
      await sleep(25 + (job.data.index % 6) * 15);
      completed++;
      return { ok: true };
    } finally {
      active--;
      serviceMs.push(performance.now() - start);
    }
  };
  lab.spawn(processor, { concurrency: 4, metrics: { maxDataPoints: 60 } });
  lab.spawn(processor, { concurrency: 4, metrics: { maxDataPoints: 60 } });
  const start = performance.now();
  const jobs = await lab.queue.addBulk(Array.from({ length: 120 }, (_, index) => ({
    name: 'synthetic-io', data: { index }
  })));
  console.log('BACKLOG_SNAPSHOT', await lab.queue.getJobCounts('waiting','active','completed','failed'));
  await Promise.all(jobs.map(job => job.waitUntilFinished(lab.events, 30000)));
  const elapsed = performance.now() - start;
  const result = {
    completed, peak, elapsedMs: elapsed, throughputPerSecond: completed * 1000 / elapsed,
    queueWaitP50: quantile(waitMs, 0.5), queueWaitP95: quantile(waitMs, 0.95),
    serviceP50: quantile(serviceMs, 0.5), serviceP95: quantile(serviceMs, 0.95)
  };
  console.log(JSON.stringify(result, null, 2));
  assert.equal(completed, 120);
  assert.ok(peak <= 4);
  assert.ok(result.queueWaitP95 > result.serviceP50);
  console.log('BULLMQ_METRICS', await lab.queue.getMetrics('completed', 0, 10));
} finally { await lab.close(); }
JS
docker compose -p be-mq exec -T app node labs/32-load.mjs | tee evidence/32-load.txt
docker compose -p be-mq exec -T rabbit rabbitmqctl list_queues -p /lab name type messages_ready messages_unacknowledged consumers
docker compose -p be-mq exec -T rabbit rabbitmq-diagnostics -q check_local_alarms
docker compose -p be-mq exec -T redis redis-cli INFO memory
docker compose -p be-mq exec -T redis redis-cli INFO persistence
docker compose -p be-mq exec -T postgres psql -U lab -d messaging -c "SELECT count(*) AS pending, min(created_at) AS oldest_pending, count(*) FILTER(WHERE parked_at IS NOT NULL) AS parked FROM outbox WHERE sent_at IS NULL;"
docker compose -p be-mq stats --no-stream
```

**Kết quả:** p95 queue wait lớn hơn service time trong một burst; peak không vượt bốn dù hai workers đều khai báo concurrency bốn. Built-in metrics có độ phân giải riêng, không nhất thiết lập tức có nhiều bucket sau bài vài giây. [BullMQ metrics](https://docs.bullmq.io/guide/metrics/) mô tả dữ liệu hoàn thành/thất bại và cách cấu hình retention.

Một load test đáng dùng phải chạy lâu hơn thời gian warmup, có payload gần thực, giới hạn tài nguyên giống môi trường dự kiến và có tiêu chí dừng. Khi p95 tăng, tìm tầng bão hòa trước khi scale: CPU app, pool DB, IOPS broker, fsync, Redis latency hoặc provider quota. Chạy benchmark chỉ với sleep chứng minh concurrency của harness, không ước tính được báo cáo PDF 500 trang.

### Bộ cảnh báo khởi đầu

| Tín hiệu | Câu hỏi vận hành | Ví dụ ngưỡng khởi đầu cần hiệu chỉnh |
|---|---|---|
| Oldest due age | Người dùng chờ quá lâu chưa? | Vượt SLA trong vài phút |
| Arrival rate − completion rate | Nợ công việc đang tăng? | Dương liên tục qua nhiều khoảng đo |
| Unacked cao, completion thấp | Workers có treo/chậm không? | Gần prefetch tổng, không tiến triển |
| Outbox oldest age | Producer có giữ nghĩa vụ chưa phát không? | Tăng dù relay vẫn “running” |
| Outbox parked | Relay gặp lỗi không tự chữa? | Bất kỳ dòng mới cần phân loại |
| Parking rate theo reason | Code hay dữ liệu đang lỗi? | Tăng đột biến so với baseline |
| Duplicate rate | Có mất confirms, restart hay replay? | Tăng khác thường, không nhất thiết lỗi dữ liệu |
| Lock renewal/stalled | Event loop, CPU hay Redis bất ổn? | Có spike cùng deploy hoặc latency |
| Disk/memory alarm | Broker có chặn publishers không? | Alert theo alarm và headroom |
| Business reconciliation gap | Có nghĩa vụ đã bỏ quên không? | Quá grace window và chưa có ngoại lệ |

Không gắn eventId, orderId hoặc email vào label metrics: cardinality có thể tăng theo số giao dịch. Giữ chúng trong log/trace có truy vấn theo ID và kiểm soát dữ liệu. [RabbitMQ monitoring](https://www.rabbitmq.com/docs/monitoring) khuyến nghị theo dõi cả broker và ứng dụng; [memory/disk alarms](https://www.rabbitmq.com/docs/alarms) giải thích lý do publisher có thể bị block trong khi broker vẫn còn chạy.

### Trace xuyên nhiều lần retry

Log nên có service, instance, eventId, business operation key, queue, attempt, outcome, duration và trace/correlation khi có. Khi retry tạo một execution mới, đừng ghi đè lịch sử của execution cũ. Với OpenTelemetry, truyền context hợp lệ, tạo consumer span thích hợp và cân nhắc span links cho batch hoặc fanout; một chuỗi hex tự đặt tên `traceId` trong lab chưa phải triển khai tracing đầy đủ.

Một trace có thể bị sampling bỏ; vì vậy audit ledger và nghĩa vụ outbox không được phụ thuộc vào trace tồn tại. Khi sự cố tiền hoặc dữ liệu xảy ra, DB constraints và reconciliation cung cấp bằng chứng bền vững hơn một giao diện tracing.

## Lab 33 — Backup PostgreSQL và restore vào database mới

**Mục tiêu:** tạo dump, restore sang đích mới và kiểm tra ledger. Không ghi đè database `messaging` đang chạy.

**Tiền điều kiện:** đĩa còn đủ dung lượng cho dump và database phục hồi. Không chạy đồng thời capstone writers khi bạn muốn so sánh số lượng chính xác tại cùng thời điểm. `pg_dump` tạo snapshot nhất quán cho database nhưng không snapshot cùng RabbitMQ/Redis. Xem [PostgreSQL pg_dump](https://www.postgresql.org/docs/17/app-pgdump.html).

```bash {"name":"mq-33-backup-and-restore","excludeFromRunAll":"true"}
set -euo pipefail
LAB="$HOME/backend-labs/messaging"
test -f "$LAB/.be-mq-lab"
cd "$LAB"
stamp="$(date -u +%Y%m%dT%H%M%SZ)"
backup_file="backups/messaging-$stamp.dump"
restore_db="mq_restore_$(date -u +%Y%m%d%H%M%S)"
umask 077
docker compose -p be-mq exec -T postgres pg_dump -U lab -d messaging -Fc > "$backup_file"
test -s "$backup_file"
docker compose -p be-mq exec -T postgres pg_restore --list < "$backup_file" > "evidence/restore-list-$stamp.txt"
docker compose -p be-mq exec -T postgres createdb -U lab "$restore_db"
docker compose -p be-mq exec -T postgres pg_restore -U lab -d "$restore_db" --no-owner --exit-on-error < "$backup_file"
docker compose -p be-mq exec -T postgres psql -v ON_ERROR_STOP=1 -U lab -d "$restore_db" -c "SELECT count(*) AS orders FROM orders;"
docker compose -p be-mq exec -T postgres psql -v ON_ERROR_STOP=1 -U lab -d "$restore_db" -c "SELECT w.id,w.balance_minor,COALESCE(sum(l.delta_minor),0) AS ledger_sum FROM wallets w LEFT JOIN wallet_ledger l ON l.wallet_id=w.id GROUP BY w.id,w.balance_minor HAVING w.balance_minor<>COALESCE(sum(l.delta_minor),0);"
printf 'backup=%s\nrestored_database=%s\n' "$backup_file" "$restore_db" | tee "evidence/restore-manifest-$stamp.txt"
```

**Kết quả:** restore hoàn tất và query lệch wallet/ledger không có dòng cho dữ liệu lab hợp lệ. Database đích vẫn được giữ để bạn kiểm tra thêm. Lần chạy lại trong cùng một giây có thể trùng tên và dừng ở createdb; chờ giây kế tiếp hoặc chọn suffix khác, không drop database hiện có để lách lỗi.

**Bước tùy chọn:** xuất definitions RabbitMQ và snapshot RDB Redis để biết chúng chứa gì. Những file này không thay thế việc restore toàn hệ thống.

```bash {"name":"mq-33-broker-and-redis-evidence","excludeFromRunAll":"true"}
set -euo pipefail
LAB="$HOME/backend-labs/messaging"
test -f "$LAB/.be-mq-lab"
cd "$LAB"
stamp="$(date -u +%Y%m%dT%H%M%SZ)"
umask 077
docker compose -p be-mq exec -T rabbit rabbitmqctl export_definitions /var/lib/rabbitmq/be-mq-definitions.json
docker compose -p be-mq cp rabbit:/var/lib/rabbitmq/be-mq-definitions.json "backups/rabbit-definitions-$stamp.json"
docker compose -p be-mq exec -T redis redis-cli BGSAVE
finished=0
for attempt in $(seq 1 40); do
  persistence="$(docker compose -p be-mq exec -T redis redis-cli INFO persistence)"
  if printf '%s\n' "$persistence" | tr -d '\r' | grep -q '^rdb_bgsave_in_progress:0$'; then
    printf '%s\n' "$persistence" | tr -d '\r' | grep -q '^rdb_last_bgsave_status:ok$'
    finished=1
    break
  fi
  sleep 1
done
test "$finished" -eq 1
docker compose -p be-mq cp redis:/data/dump.rdb "backups/redis-$stamp.rdb"
chmod 600 "backups/rabbit-definitions-$stamp.json" "backups/redis-$stamp.rdb"
printf '%s\n' "Saved definitions and RDB for $stamp"
```

Definitions là topology, users, permissions và cấu hình liên quan, không phải toàn bộ message backlog. Có password hashes hoặc cấu hình nhạy cảm nên file được hạn chế quyền. RabbitMQ data-directory backup có ràng buộc tên node, phiên bản và trạng thái; không chép sống tùy ý rồi coi đó là phương án phục hồi đã thử. Xem [RabbitMQ backup/restore](https://www.rabbitmq.com/docs/backup).

Snapshot RDB và AOF là hai dạng persistence khác nhau. Redis đang ưu tiên AOF khi khởi động theo cấu hình của lab; copy một RDB vào thư mục còn AOF cũ không có nghĩa Redis sẽ đọc RDB bạn muốn. Bài chỉ tạo evidence RDB, chưa tuyên bố đã diễn tập restore Redis đầy đủ. Một lịch backup production phải có bản ngoài VPS, mã hóa, retention, kiểm tra tính toàn vẹn và diễn tập RTO.

### RPO và RTO phải gắn với nghiệp vụ

RPO là lượng thay đổi có thể phải chấp nhận mất hoặc phục hồi bằng nguồn khác; RTO là thời gian cần để cung cấp dịch vụ trở lại. “Backup mỗi ngày” chưa trả lời nếu đơn hàng mới nhất trong ngày có thể mất. Nếu provider bên ngoài còn lịch sử giao dịch, bạn có thể đối soát phần thiếu nhưng thời gian và quy trình đó phải được tính vào RTO.

Queue có thể được tái tạo từ outbox và trạng thái nghiệp vụ cho một số loại tác vụ. Với message chứa dữ liệu duy nhất không có nguồn khác, mất broker volume có thể nghiêm trọng hơn nhiều. Xác định nguồn sự thật và retention trước khi chọn backup technology.

## Lab 34 — Chuẩn bị một VPS theo từng bước

**Mục tiêu:** chuyển lab sang một VPS riêng để luyện vận hành, với credentials ngẫu nhiên, cổng quản trị chỉ loopback, image ứng dụng có lock file và process worker/relay riêng. Đây là cấu hình staging để học; trước dữ liệu thật cần hoàn thành các yêu cầu quyền, backup, SLO và HA bên dưới.

**Không có lệnh nào ở đây tự kết nối một VPS chưa được bạn chỉ định.** Các ô tạo file chạy trên máy lab. Ô SSH có placeholder và dừng nếu chưa sửa rõ host. Không dùng VPS đang chứa production của bạn để thử kill, stop hoặc dọn volumes.

### Bước 1 — Chọn cấu hình theo workload

| Profile học tập | CPU/RAM gợi ý ban đầu | Phù hợp để luyện | Điều phải đo |
|---|---|---|---|
| Chỉ BullMQ + Redis | 2 vCPU / 2–4 GiB | Email giả lập, jobs nhỏ, retries | RSS, event loop lag, Redis maxmemory, latency |
| Toàn bộ chương | 2–4 vCPU / 4–8 GiB | Broker, DB, outbox và fault drills | Disk/fsync, DB pool, broker alarms, backlog |
| Công việc CPU/file lớn | Tách worker có RAM/CPU phù hợp | PDF, video, image, import | Peak RSS mỗi job, throughput thực, OOM |
| Cần chịu mất một node | Nhiều fault domains hoặc managed service | HA và rolling maintenance | Quorum, failover time, RPO/RTO và network |

Đây là điểm bắt đầu cho lab, không phải sizing bảo đảm số requests/giây. Shared vCPU có thể biến động; SSD có dung lượng lớn chưa chắc có fsync latency tốt. Một VPS chạy ba containers RabbitMQ vẫn có cùng nguồn điện, kernel và ổ đĩa; không được gọi đó là HA.

RabbitMQ 4 không dùng classic mirrored queues kiểu cũ. Khi cần replication, đánh giá quorum queues hoặc streams theo mục tiêu. Với quorum ba replicas ở các node độc lập, mất một node có thể còn majority; mất majority làm queue không thể tiếp tục phục vụ theo cam kết an toàn. Đừng dùng bài hướng dẫn `ha-mode: all` của dòng cũ làm blueprint RabbitMQ 4. Baseline này có một node nên chỉ luyện semantics, không kiểm chứng chịu hỏng node.

### Bước 2 — Tạo image ứng dụng và vòng chạy relay

**Tiền điều kiện:** toàn bộ file tới Lab 30 và `app/package-lock.json` đã được sinh từ Lab 02. Không build bằng source thiếu lock rồi cài @latest trên VPS.

```bash {"name":"mq-34-prepare-vps-files","excludeFromRunAll":"true"}
set -euo pipefail
LAB="$HOME/backend-labs/messaging"
test -f "$LAB/.be-mq-lab"
cd "$LAB"
test -f app/package-lock.json
mkdir -p vps
cat > app/Dockerfile <<'DOCKER'
FROM node:24-bookworm-slim
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci --omit=dev --ignore-scripts && npm cache clean --force
COPY --chown=node:node . .
USER node
CMD ["node", "capstone/worker.mjs"]
DOCKER
cat > app/.dockerignore <<'TEXT'
node_modules
*.log
.env*
capstone/current-order.json
capstone/poison-event.json
TEXT
cat > app/capstone/relay-loop.mjs <<'JS'
import { spawn } from 'node:child_process';
import { setTimeout as sleep } from 'node:timers/promises';
let stopping = false;
let current;
let failures = 0;
const stop = () => {
  stopping = true;
  if (current) current.kill('SIGTERM');
};
process.on('SIGTERM', stop);
process.on('SIGINT', stop);
while (!stopping) {
  const code = await new Promise(resolve => {
    current = spawn(process.execPath, ['capstone/relay.mjs'], {
      stdio: 'inherit',
      env: { ...process.env, MAX_EVENTS: '100' }
    });
    current.once('error', () => resolve(1));
    current.once('exit', value => resolve(value ?? 1));
  });
  current = undefined;
  if (stopping) break;
  failures = code === 0 ? 0 : Math.min(failures + 1, 5);
  const delay = code === 0 ? 1000 : Math.min(30000, 1000 * 2 ** failures) + Math.floor(Math.random() * 500);
  console.log('RELAY_LOOP_WAIT_MS', delay);
  // Ngủ từng phần ngắn để signal không phải chờ hết backoff.
  const until = Date.now() + delay;
  while (!stopping && Date.now() < until) await sleep(Math.min(200, until - Date.now()));
}
process.removeListener('SIGTERM', stop);
process.removeListener('SIGINT', stop);
JS
cat > vps/compose.yml <<'YAML'
name: be-mq
x-app: &app
  image: be-mq-course:locked
  build:
    context: ../app
  init: true
  restart: unless-stopped
  stop_grace_period: 35s
  mem_limit: 512m
  environment: &app-env
    AMQP_URL: amqp://lab:${RABBIT_PASSWORD:?Set RABBIT_PASSWORD}@rabbit:5672/%2Flab?heartbeat=10
    DATABASE_URL: postgres://lab:${POSTGRES_PASSWORD:?Set POSTGRES_PASSWORD}@postgres:5432/messaging
    REDIS_HOST: redis
    REDIS_PORT: "6379"
    REDIS_PASSWORD: "${REDIS_PASSWORD:?Set REDIS_PASSWORD}"
    RUN_MS: "0"
    NODE_ENV: production
  logging:
    driver: json-file
    options:
      max-size: "10m"
      max-file: "3"
services:
  redis:
    image: redis:7.4-alpine
    command: ["sh", "-c", "exec redis-server --appendonly yes --appendfsync everysec --maxmemory 256mb --maxmemory-policy noeviction --requirepass \"$$REDIS_PASSWORD\""]
    environment:
      REDIS_PASSWORD: "${REDIS_PASSWORD}"
      REDISCLI_AUTH: "${REDIS_PASSWORD}"
    volumes:
      - redis-data:/data
    restart: unless-stopped
    mem_limit: 512m
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      timeout: 3s
      retries: 20
  rabbit:
    image: rabbitmq:4-management
    hostname: mq-rabbit
    environment:
      RABBITMQ_DEFAULT_USER: lab
      RABBITMQ_DEFAULT_PASS: "${RABBIT_PASSWORD}"
      RABBITMQ_DEFAULT_VHOST: /lab
    ports:
      - "127.0.0.1:15673:15672"
    volumes:
      - rabbit-data:/var/lib/rabbitmq
    restart: unless-stopped
    mem_limit: 1536m
    healthcheck:
      test: ["CMD", "rabbitmq-diagnostics", "-q", "check_running"]
      interval: 5s
      timeout: 5s
      retries: 30
  postgres:
    image: postgres:17-bookworm
    environment:
      POSTGRES_USER: lab
      POSTGRES_PASSWORD: "${POSTGRES_PASSWORD}"
      POSTGRES_DB: messaging
    volumes:
      - pg-data:/var/lib/postgresql/data
    restart: unless-stopped
    mem_limit: 768m
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U lab -d messaging"]
      interval: 5s
      timeout: 3s
      retries: 30
  worker:
    <<: *app
    command: ["node", "capstone/worker.mjs"]
    depends_on:
      rabbit:
        condition: service_healthy
      postgres:
        condition: service_healthy
  relay:
    <<: *app
    command: ["node", "capstone/relay-loop.mjs"]
    depends_on:
      rabbit:
        condition: service_healthy
      postgres:
        condition: service_healthy
  tools:
    <<: *app
    profiles: ["tools"]
    restart: "no"
    command: ["node", "--version"]
volumes:
  redis-data:
  rabbit-data:
  pg-data:
YAML
umask 077
if [ ! -e vps/.env ]; then
  docker compose -p be-mq exec -T app node --input-type=module - <<'JS' > vps/.env
import { randomBytes } from 'node:crypto';
for (const name of ['RABBIT_PASSWORD','POSTGRES_PASSWORD','REDIS_PASSWORD']) {
  console.log(name + '=' + randomBytes(32).toString('hex'));
}
JS
fi
chmod 600 vps/.env
docker compose --env-file vps/.env -f vps/compose.yml -p be-mq config --quiet
printf '%s\n' 'VPS files prepared. Passwords are in vps/.env; do not print or commit that file.'
```

**Giải thích:** random hex tránh phải URL-encode các ký tự đặc biệt trong connection URL mẫu. File dotenv vẫn có thể bị đọc bởi người có quyền trên host; nó chưa là secret manager. Không commit nó, không đưa vào log và không chạy `docker compose config` không có `--quiet` rồi paste output. Khi triển khai thật, dùng phương thức cấp secrets của tổ chức và rotate theo quy trình.

Template này dùng một tài khoản lab để tự tạo topology, thích hợp staging riêng. Production cần user riêng cho provisioning, relay, consumer và management; quyền vhost và resource regex chỉ cho phép thao tác cần thiết. Consumer không cần quyền publish mọi business event, relay không cần đọc mọi queue. [RabbitMQ access control](https://www.rabbitmq.com/docs/access-control) mô tả ba loại permission configure/write/read.

Compose tạo network riêng và service names ổn định để container tìm nhau; container không dùng `127.0.0.1` để tìm PostgreSQL của container khác. Khi recreate container, IP có thể thay đổi, còn service name vẫn được resolve. Xem [Docker Compose networking](https://docs.docker.com/compose/how-tos/networking/).

### Bước 3 — Chuẩn bị VPS và chuyển bản phát hành

Cài Docker Engine và Compose plugin theo tài liệu chính thức cho đúng distribution; ví dụ [Docker Engine on Ubuntu](https://docs.docker.com/engine/install/ubuntu/). Không áp dụng lệnh apt cho một VPS Alpine hay CentOS. Tạo user vận hành, dùng SSH key và xác minh fingerprint host qua kênh tin cậy. Quyền Docker daemon rất mạnh; chỉ cấp cho người cần vận hành.

Các ô sau có `VPS_HOST="CHANGE_ME"`. Sửa thành hostname/IP VPS luyện tập và `VPS_USER` của bạn ngay trong từng ô trước khi chạy. Mỗi ô độc lập; không trông chờ biến từ ô trước.

```bash {"name":"mq-34-package-release","excludeFromRunAll":"true"}
set -euo pipefail
LAB="$HOME/backend-labs/messaging"
test -f "$LAB/.be-mq-lab"
cd "$LAB"
umask 077
tar --exclude='app/node_modules' --exclude='app/*.log' -czf backups/be-mq-vps-release.tar.gz app vps .be-mq-lab
chmod 600 backups/be-mq-vps-release.tar.gz
tar -tzf backups/be-mq-vps-release.tar.gz
```

Archive chứa secrets staging, nên không đăng công khai hoặc gửi qua kênh không được phép. Nếu dùng CI, build/push image đã khóa và cấp secrets trên VPS riêng sẽ phù hợp hơn đóng gói dotenv.

```bash {"name":"mq-34-upload-to-explicit-vps","excludeFromRunAll":"true"}
set -euo pipefail
LAB="$HOME/backend-labs/messaging"
test -f "$LAB/.be-mq-lab"
cd "$LAB"
VPS_HOST="CHANGE_ME"
VPS_USER="deploy"
test "$VPS_HOST" != "CHANGE_ME"
test -s backups/be-mq-vps-release.tar.gz
ssh "$VPS_USER@$VPS_HOST" 'mkdir -p "$HOME/backend-labs/messaging" && chmod 700 "$HOME/backend-labs/messaging"'
scp backups/be-mq-vps-release.tar.gz "$VPS_USER@$VPS_HOST:backend-labs/messaging/be-mq-vps-release.tar.gz"
```

**Bước 4:** kết nối SSH bằng terminal riêng, đọc hostname và chạy ô dưới **trên VPS luyện tập** sau khi chuyển archive. Ô từ chối môi trường chưa có archive. Mọi thao tác vẫn ở thư mục riêng; không sửa Docker context toàn máy.

```bash {"name":"mq-34-bootstrap-on-vps","excludeFromRunAll":"true"}
set -euo pipefail
LAB="$HOME/backend-labs/messaging"
cd "$LAB"
test -f be-mq-vps-release.tar.gz
hostname
docker context show
tar -xzf be-mq-vps-release.tar.gz
test -f .be-mq-lab
chmod 600 vps/.env
docker compose --env-file vps/.env -f vps/compose.yml -p be-mq config --quiet
docker compose --env-file vps/.env -f vps/compose.yml -p be-mq build worker relay tools
docker compose --env-file vps/.env -f vps/compose.yml -p be-mq up -d --wait --wait-timeout 180 redis rabbit postgres
docker compose --env-file vps/.env -f vps/compose.yml -p be-mq exec -T postgres psql -v ON_ERROR_STOP=1 -U lab -d messaging < app/schema.sql
docker compose --env-file vps/.env -f vps/compose.yml -p be-mq run --rm --no-deps tools node capstone/topology.mjs
docker compose --env-file vps/.env -f vps/compose.yml -p be-mq exec -T rabbit rabbitmqctl set_policy -p /lab --apply-to queues --priority 20 mq-main '^mq\.rewards$' '{"dead-letter-exchange":"mq.parking.ex","dead-letter-routing-key":"parked","dead-letter-strategy":"at-least-once","overflow":"reject-publish","delivery-limit":10,"max-length":10000}'
docker compose --env-file vps/.env -f vps/compose.yml -p be-mq exec -T rabbit rabbitmqctl set_policy -p /lab --apply-to queues --priority 20 mq-retry1 '^mq\.retry\.1000$' '{"message-ttl":1000,"dead-letter-exchange":"mq.events","dead-letter-routing-key":"order.paid","dead-letter-strategy":"at-least-once","overflow":"reject-publish","max-length":10000}'
docker compose --env-file vps/.env -f vps/compose.yml -p be-mq exec -T rabbit rabbitmqctl set_policy -p /lab --apply-to queues --priority 20 mq-retry4 '^mq\.retry\.4000$' '{"message-ttl":4000,"dead-letter-exchange":"mq.events","dead-letter-routing-key":"order.paid","dead-letter-strategy":"at-least-once","overflow":"reject-publish","max-length":10000}'
docker compose --env-file vps/.env -f vps/compose.yml -p be-mq exec -T rabbit rabbitmqctl set_policy -p /lab --apply-to queues --priority 10 mq-capacity '^mq\.(audit|notifications)$' '{"max-length":10000,"overflow":"reject-publish"}'
docker compose --env-file vps/.env -f vps/compose.yml -p be-mq exec -T rabbit rabbitmqctl set_policy -p /lab --apply-to queues --priority 10 mq-parking '^mq\.parking$' '{"max-length":10000,"overflow":"reject-publish","delivery-limit":-1}'
docker compose --env-file vps/.env -f vps/compose.yml -p be-mq up -d worker relay
docker compose --env-file vps/.env -f vps/compose.yml -p be-mq ps
docker compose --env-file vps/.env -f vps/compose.yml -p be-mq logs --tail 80 worker relay
```

**Điểm cần kiểm tra:** worker log READY; relay không lỗi quyền hoặc schema; PostgreSQL có bảng; policies đúng vhost; không có cổng Redis/PG/AMQP publish ra host. Audit/notification queues tích message vì chưa có downstream consumers đầy đủ; cần triển khai hoặc đặt retention có chủ đích trước khi có lưu lượng dài hạn.

Tài khoản bootstrap trong image RabbitMQ/PostgreSQL chỉ được áp dụng khi volume khởi tạo trống. Đổi dotenv rồi restart một volume cũ không đảm bảo password trong DB/broker đổi theo. Rotation cần lệnh quản trị đúng, rollout credentials mới và xác nhận client dùng key mới; không xóa volume để “cập nhật password”.

### Bước 5 — Tunnel UI và smoke test

Chạy tunnel từ laptop, trong terminal giữ mở. Cổng `15674` phía laptop tránh xung đột management local `15673` của lab khác. Kết thúc tunnel bằng Ctrl+C.

```bash {"name":"mq-34-ssh-management-tunnel","excludeFromRunAll":"true"}
set -euo pipefail
LAB="$HOME/backend-labs/messaging"
test -f "$LAB/.be-mq-lab"
cd "$LAB"
VPS_HOST="CHANGE_ME"
VPS_USER="deploy"
test "$VPS_HOST" != "CHANGE_ME"
ssh -N -o ExitOnForwardFailure=yes -L 127.0.0.1:15674:127.0.0.1:15673 "$VPS_USER@$VPS_HOST"
```

Mở [management qua tunnel](http://127.0.0.1:15674). Dùng credentials staging đã cấp; không đổi bind thành `0.0.0.0:15673` để tiện truy cập. Nếu nhiều VPS/services cần giao tiếp, sử dụng mạng riêng và TLS phù hợp; một SSH tunnel quản trị không thay thế TLS giữa các services trên mạng không tin cậy.

Chạy smoke test **trên VPS**:

```bash {"name":"mq-34-vps-smoke-test","excludeFromRunAll":"true"}
set -euo pipefail
LAB="$HOME/backend-labs/messaging"
test -f "$LAB/.be-mq-lab"
cd "$LAB"
docker compose --env-file vps/.env -f vps/compose.yml -p be-mq run --rm --no-deps tools node capstone/create-order.mjs
docker compose --env-file vps/.env -f vps/compose.yml -p be-mq exec -T rabbit rabbitmq-diagnostics -q check_local_alarms
docker compose --env-file vps/.env -f vps/compose.yml -p be-mq exec -T postgres psql -U lab -d messaging -c "SELECT count(*) AS pending,min(created_at) AS oldest FROM outbox WHERE sent_at IS NULL;"
docker compose --env-file vps/.env -f vps/compose.yml -p be-mq logs --tail 100 worker relay
```

Đối chiếu eventId từ create-order với log COMMITTED và ledger. Vì xử lý bất đồng bộ, một query chạy ngay sau producer có thể vẫn thấy pending; hãy quan sát đến điều kiện có deadline thay vì coi một ảnh chụp duy nhất là thất bại. Trước khi mở API thật, thêm readiness phản ánh khả năng phục vụ và instrumentation end-to-end.

### Trước dữ liệu thật: những việc template staging chưa làm thay bạn

Tách quyền RabbitMQ và PostgreSQL theo service; kiểm soát tenant và xác thực nguồn event; quản lý secrets và TLS; khóa images bằng digest đã kiểm thử; cấu hình backup ngoài host và phục hồi có diễn tập; đặt tài nguyên cùng cảnh báo; có dashboard queue age/business gaps; kiểm tra provider idempotency; đặt lịch nâng cấp và tương thích schema.

Một process relay-loop sinh child mỗi batch giúp bài dễ kiểm tra và backoff rõ ràng. Với tải cao, có thể dùng vòng lặp lâu dài tái sử dụng connection, bounded confirms và batch claim có lease renewal. Đừng tối ưu trước khi có số đo. Bảo toàn các invariants của Lab 20–27 quan trọng hơn giảm một lần tạo process.

## Lab 35 — Graceful shutdown: ngừng lấy mới, chờ việc đang chạy

**Mục tiêu:** gửi SIGTERM tới đúng child worker của bài; job đang active hoàn tất, job thêm sau shutdown vẫn waiting. Không dùng `pkill node` hoặc kill theo tên container chung.

[Node.js v24 signal events](https://nodejs.org/docs/latest-v24.x/api/process.html#signal-events) cho biết khi bạn cài handler, ứng dụng phải tự hoàn tất logic thoát. [BullMQ graceful shutdown](https://docs.bullmq.io/guide/workers/graceful-shutdown) mô tả việc `worker.close()` ngừng nhận và chờ active jobs; hàm đó không biến tác vụ vô hạn thành hữu hạn.

```bash {"name":"mq-35-graceful-shutdown","excludeFromRunAll":"true"}
set -euo pipefail
LAB="$HOME/backend-labs/messaging"
test -f "$LAB/.be-mq-lab"
cd "$LAB"
cat > app/labs/35-worker.mjs <<'JS'
import { Worker } from 'bullmq';
import { workerConnection, sleep } from '../lib/common.mjs';
const worker = new Worker(process.env.LAB_QUEUE, async job => {
  console.log('ACTIVE ' + job.id);
  await sleep(700);
  return { done: true };
}, { connection: workerConnection, concurrency: 1 });
worker.on('error', error => console.error('WORKER_ERROR', error.message));
let stopping = false;
async function shutdown() {
  if (stopping) return;
  stopping = true;
  console.log('STOPPING');
  try {
    await worker.close();
    console.log('DRAINED');
  } catch (error) {
    console.error(error.message);
    process.exitCode = 1;
  }
}
process.on('SIGTERM', () => { void shutdown(); });
await worker.waitUntilReady();
console.log('READY');
JS
cat > app/labs/35-shutdown.mjs <<'JS'
import assert from 'node:assert/strict';
import { spawn } from 'node:child_process';
import { bullLab, until } from '../lib/common.mjs';
const lab = await bullLab('shutdown');
let child;
let output = '';
let signalled = false;
let resolveExit;
const exited = new Promise(resolve => { resolveExit = resolve; });
try {
  child = spawn(process.execPath, ['labs/35-worker.mjs'], {
    env: { ...process.env, LAB_QUEUE: lab.name },
    stdio: ['ignore','pipe','inherit']
  });
  child.on('error', error => { output += 'SPAWN_ERROR ' + error.message; resolveExit(1); });
  child.on('exit', code => resolveExit(code));
  child.stdout.on('data', data => {
    const text = data.toString();
    output += text;
    process.stdout.write(text);
    if (!signalled && output.includes('ACTIVE ')) {
      signalled = true;
      child.kill('SIGTERM');
    }
  });
  await until(() => output.includes('READY'), 10000, 'worker READY');
  const first = await lab.queue.add('slow-job', {});
  await until(() => output.includes('STOPPING'), 10000, 'worker STOPPING');
  const second = await lab.queue.add('after-stop', {});
  await first.waitUntilFinished(lab.events, 15000);
  await until(() => child.exitCode !== null, 10000, 'worker exit');
  const code = await exited;
  assert.equal(code, 0);
  assert.ok(output.includes('DRAINED'));
  assert.equal(await second.getState(), 'waiting');
  console.log('SHUTDOWN_VERIFIED: first completed, second still waiting');
} finally {
  if (child && child.exitCode === null) child.kill('SIGKILL');
  await lab.close();
}
JS
docker compose -p be-mq exec -T app node labs/35-shutdown.mjs | tee evidence/35-shutdown.txt
```

**Kết quả:** READY → ACTIVE → STOPPING → DRAINED; job thứ nhất completed, thứ hai waiting. Docker stop_grace_period phải lớn hơn thời gian drain cho workload đã đặt deadline. Sau hạn, SIGKILL có thể xảy ra; consumer idempotency vẫn là tuyến bảo vệ khi drain không hoàn thành.

Với RabbitMQ, cancel consumer ngăn delivery mới nhưng không hủy tác vụ đang chạy. Chờ pending handlers, commit rồi ACK, sau đó đóng channel/connection. Nếu tác vụ HTTP ngoài không trả, cần timeout và trạng thái unknown để shutdown có thể kết thúc. Trong NestJS, dùng shutdown hooks để gọi các bước này và kiểm tra adapter thực tế; không giả định chỉ bật `enableShutdownHooks` là mọi thư viện tự drain đúng.

### Rolling deploy và schema

Deploy worker mới khi producer cũ vẫn chạy là tình huống bình thường. Consumer phải xử lý các schemas trong retention window. Nếu sửa processor khiến cùng jobId có nghĩa khác, những jobs cũ trong queue cần được xem xét. Giữ version nghiệp vụ hoặc processor version khi kết quả phụ thuộc định nghĩa công việc.

Khi đổi queue type hay immutable arguments, tạo queue mới, bind song song có chủ đích, theo dõi traffic, drain queue cũ và chuyển producer/consumer theo kế hoạch. Không xóa queue còn messages vì một lỗi PRECONDITION_FAILED trong startup.

---

# Phần VIII. Runbooks dùng khi có sự cố

## Runbook A — Ready tăng liên tục, API vẫn nhanh

**Mục tiêu:** xác định nghĩa vụ nào trễ và nút thắt trước khi tăng replicas.

1. Xem arrival/completion rate và tuổi công việc đã đến hạn, tách delayed có chủ đích.
2. Xác định consumer count, queue paused/rate limited/global concurrency, process restarts và cấu hình cùng queue name/vhost.
3. Kiểm tra service time p95, DB pool wait, provider quota và CPU/event loop.
4. Ước tính drain time với tốc độ ròng; báo tác động cho đội sản phẩm.
5. Giảm hoặc giới hạn đầu vào nếu backlog vượt dung lượng/thời hạn. Chọn việc có thể trì hoãn hoặc từ chối theo nghiệp vụ.
6. Tăng workers có kiểm soát chỉ khi dependency còn headroom; theo dõi p95 và error rate sau mỗi thay đổi.
7. Đối chiếu các công việc quá hạn sau khi backlog giảm, vì một số có thể không còn giá trị thực hiện.

Không tăng concurrency và maxmemory cùng lúc nếu bạn muốn biết thay đổi nào giúp hoặc hại. Mỗi bước nên có số đo trước/sau và điều kiện rollback.

## Runbook B — Unacked cao nhưng ready thấp

Có thể worker đang làm việc nặng đúng dự kiến, đang chờ DB, deadlock, provider timeout hoặc Promise bị bỏ rơi. Xem handler duration và in-flight count. So sánh unacked với prefetch × consumers; nếu bằng trần và completion gần zero, dependency hoặc code cần điều tra.

Không ACK toàn bộ bằng công cụ quản trị để “giải phóng queue”. ACK chỉ thay đổi trách nhiệm giữ message; nó không hoàn tất nghiệp vụ. Nếu buộc restart worker, chuẩn bị redelivery và kiểm tra idempotency trước khi tăng concurrency phục hồi.

## Runbook C — Publisher confirm timeout

Kiểm tra broker memory/disk alarms, network, channel errors, exchange tồn tại, route và queue overflow reject-publish. Một broker còn ping được có thể đang block publishers. Đọc cả return events và confirm errors; hai loại kết quả khác nhau.

Giữ event chưa sent và thử lại cùng ID sau backoff hoặc đưa producer outbox vào trạng thái cần vận hành. Nếu confirm cuối cùng đã tới nhưng process mất trạng thái, duplicate là dự kiến. Tìm ledger/inbox để biết effect đã xảy ra, không suy đoán từ publish log.

## Runbook D — Parking queue tăng

Phân nhóm theo error code/schema/producer version/deploy time, chọn mẫu có kiểm soát, bảo vệ payload. Unsupported schema thường cần consumer hỗ trợ hoặc producer sửa; lỗi quyền cần sửa credentials; dữ liệu không hợp lệ có thể cần correction nghiệp vụ. Không replay tất cả ngay sau deploy chỉ vì code mới đã lên.

Replay một mẫu nhỏ, kiểm tra ledger/đối chiếu, rồi tăng batch theo quota. Giữ event IDs và audit operator/reason. Nếu retry lại vẫn permanent, dừng mở rộng. Với lỗi không xác định kết quả provider, hỏi provider hoặc đối soát trước.

## Runbook E — Redis noeviction OOM

Xem used_memory, maxmemory, completed/failed retention, payload size và queue counts. Phân biệt memory do live jobs với lịch sử, connections, fragmentation hoặc rewrite buffer. Không dùng FLUSHDB nếu Redis chứa queue đang chạy. Dọn các job lịch sử theo phạm vi đã xác nhận hoặc tăng dung lượng có giới hạn; giữ evidence cần điều tra.

Nếu queue đã mất key do cấu hình eviction trước đó, sửa policy không tái tạo dữ liệu bị mất. Dừng traffic phù hợp, xác định phạm vi thiếu, đối chiếu DB nguồn và phát lại qua đường idempotent. Việc reconnect thành công chưa chứng minh queue lành.

## Runbook F — PRECONDITION_FAILED hoặc unknown delivery tag

PRECONDITION_FAILED thường liên quan khai báo topology khác với đặc tính queue/exchange đang tồn tại. So sánh type, durable, auto-delete và arguments; kiểm tra đúng vhost trước. Không delete queue chứa dữ liệu như một bước startup tự động.

Unknown delivery tag có thể do ACK sai channel, ACK hai lần hoặc multiple ACK quá rộng. Trace handler lifecycle và delivery tag gắn channel. Khi channel đã lỗi, tạo channel mới theo chính sách recovery; đừng gửi ACK qua channel khác cho delivery cũ.

## Runbook G — Ledger đúng nhưng người dùng thấy sai

Có thể read model/cache chưa cập nhật hoặc frontend đang đọc replica lag. Theo dõi event wallet.rewarded, notification/projection queue và version của read model. Không cộng thêm tiền chỉ để UI khớp. Nguồn dữ liệu đúng cần được xác định trước.

Đối với dashboard, trạng thái “đang cập nhật” có thể hợp lý hơn hiển thị số dư cũ như thể đã xác nhận. Eventual consistency cần biểu hiện trong API và UX, không chỉ ở bản vẽ backend.

## Runbook H — Sau restore, cùng event có thể xuất hiện lại

Giữ consumers dừng khi cần đối chiếu dữ liệu nguồn, inbox, ledger và trạng thái provider. Xác định recovery point của từng thành phần. Replay cùng ID với cùng payload, theo batch nhỏ, và dựa vào ràng buộc business operation tồn tại dài hạn. Nếu restore đã làm mất ledger mà provider còn effect, phải tái lập dữ kiện từ nguồn tin cậy theo quy trình nghiệp vụ.

Giữ bằng chứng incident, cửa sổ dữ liệu bị ảnh hưởng và các lệnh repair. Không “đánh dấu toàn bộ sent” chỉ vì cần giảm pending count.

---

# Phần IX. Các bài toán production để tự thiết kế

## 11. Webhook ra ngoài

Producer lưu event cùng outbox; một job mô tả destination ID và event ID, không nhét secret vào payload. Worker đọc cấu hình hiện tại, ký body theo hợp đồng, đặt timeout và gửi với delivery ID ổn định. Endpoint khách có thể timeout sau khi đã nhận nên delivery phải chịu duplicate.

Retry 429/5xx theo ngân sách, phân biệt 401 cấu hình với 404 endpoint bị bỏ. Giữ trạng thái và lần thử để khách tự xem, có replay có kiểm soát và hạn chế mỗi tenant. Thêm quota toàn hệ thống để một khách lỗi không chiếm mọi workers. Nếu destination là URL do người dùng nhập, cần kiểm soát SSRF và quyền truy cập mạng; không để worker truy cập địa chỉ nội bộ tùy ý.

## 12. Email và thông báo

Một campaign có thể tạo hàng triệu recipients. Không đưa toàn bộ danh sách vào một job; lưu chiến dịch và phân trang bằng cursor, tạo các đơn vị có khóa `campaign-recipient-channel`. Tách transactional email như OTP/hóa đơn khỏi marketing để fairness và SLA rõ ràng.

Không dùng queue như chỗ lưu raw token OTP lâu hơn cần thiết. Expiration nghiệp vụ phải được kiểm tra khi thực thi, vì email tới trễ có thể không còn giá trị. Unsubscribe và quyền nhận thông báo có thể thay đổi sau enqueue; consumer cần hợp đồng rõ về đọc snapshot hay trạng thái hiện tại.

Nếu provider có idempotency, dùng đúng phạm vi và TTL họ cam kết. Nếu không có, chấp nhận rằng có cửa sổ gửi trùng khi kết quả chưa biết, rồi giảm tác hại bằng nội dung, theo dõi và chính sách. Không quảng cáo “chống gửi email trùng 100%” bằng một Redis key đặt trước send.

## 13. Xuất báo cáo và xử lý file

Queue chứa object key, content hash, version template và quyền truy cập theo tenant. Worker tải file từ nguồn được phép, giới hạn kích thước và thời gian; output dùng key có version để retry không ghi đè một tài liệu khác. Upload multipart hoặc bước render dài cần checkpoint và cleanup tệp tạm theo job, không thư mục dùng chung.

Nếu người dùng hủy, cancellation thường là cooperative. Processor cần kiểm tra trạng thái ở checkpoint; job đã active không biến mất chỉ vì xóa entry waiting. Phân biệt hủy nghĩa vụ, dừng CPU và xóa output. Một phép xóa file có thể là job riêng có idempotency và retention theo chính sách.

Với file lớn, queue RAM và broker disk sẽ bị chiếm nếu nhét bytes vào payload. Nhưng chỉ gửi URL ký ngắn hạn cũng gây failure khi job chờ lâu hơn TTL URL. Lưu object key ổn định và cấp quyền tạm thời lúc worker chạy thường dễ phục hồi hơn.

## 14. Import dữ liệu nhiều dòng

Tạo import session, lưu checksum file, parse/chunk có giới hạn và ghi checkpoint theo vị trí đã commit. Đừng ACK một job 500.000 dòng sau khi mới ghi 100.000 mà không có record tiến độ; cũng đừng giữ một transaction khổng lồ suốt nhiều giờ.

Mỗi dòng có business key hoặc import-row key để retry không nhân bản. Partial success cần báo số dòng thành công/lỗi và file lỗi cho người dùng. Poison row không nên làm cả file retry từ đầu vô hạn. Cân nhắc staging table rồi validate/merge theo batch để vừa có evidence vừa giữ business constraints.

## 15. Cache invalidation và projection

Event chỉ chứa ID khiến consumer phải đọc dữ liệu hiện tại; tốt khi bạn muốn projection mới nhất, nhưng mất khả năng tái hiện chính xác trạng thái lịch sử nếu dữ liệu nguồn đã đổi. Event chứa snapshot đủ trường giúp replay độc lập hơn, đổi lại payload lớn và có dữ liệu nhạy cảm. Chọn theo yêu cầu.

Một projection có thể dùng version conditional update để bỏ snapshot cũ. Cache invalidation có thể gộp nhiều events cùng key, nhưng audit hoặc ledger thường không được gộp bỏ các delta. Dedupe giảm công việc chỉ an toàn khi semantics cho phép.

## 16. Thanh toán và đối soát

Worker timeout sau khi gửi request capture cần trạng thái unknown. Gọi capture lần nữa với key mới có thể thu hai lần; ghi failed ngay có thể khiến hệ thống giao hàng sai. Provider transaction ID, merchant reference và idempotency key là những định danh khác nhau, cần giữ đầy đủ.

Hãy lưu requested amount/currency bằng số nguyên chính xác, response có xác thực, trạng thái và nhật ký chuyển trạng thái. Callback có thể tới trước response hoặc lặp nhiều lần. State machine cần nhận ra callback cũ, version hoặc terminal states. Reconciliation định kỳ so sánh ledger nội bộ và provider để phát hiện lệch không xuất hiện trong queue metrics.

## 17. Đặt chỗ và timeout

Giữ ghế hoặc kho có deadline thuộc dữ liệu nghiệp vụ. Delayed job đánh thức quá trình giải phóng; DB conditional transition quyết định có được release không. Nếu paid event tới sau timeout, cần quy tắc rõ về tái giữ, hủy hoặc hoàn tiền.

Đừng dùng Redis lock của worker như reservation duy nhất. Lock hết hạn cho phép worker khác xử lý; reservation xác định quyền giữ tài nguyên cho khách. Hai loại thời hạn có mục đích và chủ sở hữu khác nhau.

---

# Capstone — Chứng minh một pipeline cộng thưởng phục hồi được

## Yêu cầu đầu ra

Bạn phải nộp một thư mục evidence gồm baseline phiên bản/digest, lock file, kết quả happy path, hai bài crash, một bài broker outage, một poison case, một controlled replay, một reconciliation report và kết quả restore. Mỗi evidence gắn ngày, eventId/orderId và kết luận từ SQL. Không chứa secrets thật hoặc dữ liệu khách hàng.

Sản phẩm kỹ thuật là pipeline: yêu cầu tạo đơn có client key → order và outbox cùng transaction → relay có confirm/mandatory → rewards consumer với inbox/ledger cùng transaction → event wallet.rewarded → notification queue. Phần notification vẫn là một nghĩa vụ chưa có adapter thật; bạn phải nói rõ điều đó trong bản mô tả hệ thống.

## Kịch bản nghiệm thu

1. **Retry cùng API request:** gọi createPaidOrder năm lần cùng key và dữ liệu; chỉ một order/outbox nguồn. Đổi nội dung cùng key phải lỗi rõ ràng.
2. **Broker không hoạt động:** order vẫn được chấp nhận theo hợp đồng DB; outbox chưa sent; sau khôi phục event đi tiếp.
3. **Crash sau confirm:** relay chết trước sent; lease hết; event có thể được phát lại; ledger chỉ một operation.
4. **Crash sau COMMIT:** consumer chưa ACK nhưng DB đã cập nhật; lần tiếp theo không cộng thêm.
5. **Payload sai hoặc version mới:** message vào parking, không tạo mutation; có metadata đủ điều tra.
6. **Replay đúng mục tiêu:** giữ event ID, có operator/reason, số lượng giới hạn; không drain queue khác.
7. **Concurrent relay/consumer:** kết quả business không đổi; chấp nhận duplicate delivery như thiết kế.
8. **Backlog:** đo oldest due age và p95; giải thích được thời gian drain dự kiến và dependency limiting.
9. **Shutdown:** active job được drain khi có thời gian; forced crash vẫn phục hồi qua idempotency.
10. **Restore:** DB phục hồi ở đích mới, constraints còn, reconciliation được chạy trước khi bật lại toàn bộ flows.

## Rubric đánh giá

| Mức | Đạt khi nào? | Điểm còn thiếu thường gặp |
|---|---|---|
| Biết dùng API | Tạo queue, producer, worker, routing và retry chạy được | Chỉ kiểm tra happy path |
| Làm việc độc lập | Có timeout, error handling, graceful shutdown, policies, metrics | Chưa chứng minh cửa sổ crash |
| Sẵn sàng review production | Inbox/outbox thật, constraints, replay/reconcile, quyền và backup | Chưa kiểm tra capacity/HA/provider |
| Có thể dẫn dắt thiết kế | Nêu được invariants, tradeoffs, failure matrix, migration và vận hành | Hứa bảo đảm ngoài phạm vi bằng chứng |

Không đạt mức cao chỉ bằng nhiều công nghệ hơn. Một thiết kế dùng PostgreSQL jobs đơn giản nhưng có invariants, observability và recovery rõ ràng có thể tốt hơn một hệ thống gồm ba brokers mà không biết message nào đã mất.

---

# Tự kiểm tra: câu hỏi và lời giải

### 1. Durable queue và persistent message đã đủ để producer báo thành công chưa?

Chưa. Producer cần biết broker đã chấp nhận publish theo semantics mong muốn bằng confirms, và cần mandatory/return nếu route tới queue là điều kiện. Đồng thời volumes, replication và lỗi ngoài phạm vi vẫn ảnh hưởng độ bền. Không có một cờ nào chứng minh nghiệp vụ phía consumer hoàn tất.

### 2. Vì sao publish confirm có thể thành công dù không có queue?

Confirm xác nhận broker đã xử lý publish; với exchange không có route, broker có thể xác định message không có đích. Mandatory yêu cầu trả message không route được. Helper phải ghép return với đúng publish, không chỉ dùng một biến lỗi toàn cục khi nhiều messages outstanding.

### 3. Vì sao ACK sau COMMIT vẫn có thể delivery trùng?

COMMIT và ACK đi tới hai hệ thống khác nhau. Process hoặc mạng có thể lỗi giữa chúng. Inbox cùng transaction với mutation làm lần delivery sau nhận ra effect đã hoàn tất và ACK an toàn.

### 4. Vì sao inbox INSERT trước rồi COMMIT riêng vẫn sai?

Nếu commit inbox rồi process chết trước business mutation, lần sau thấy inbox và bỏ việc chưa làm. Nếu commit mutation trước inbox, crash có thể tạo effect trùng. Chúng phải có cùng transaction hoặc một cơ chế khác có bằng chứng tương đương.

### 5. Job ID trong BullMQ có phải idempotency vĩnh viễn?

Không. Khi job bị xóa hoặc hết retention, key không còn chặn enqueue mới. Dedupe ở queue chủ yếu giảm công việc trong một phạm vi thời gian/trạng thái. Business operation cần constraint hoặc provider contract riêng.

### 6. Vì sao không retry mọi lỗi 20 lần?

Một schema sai hoặc credential hết hiệu lực không tự đúng sau 20 lần. Retry tăng tải, làm trì hoãn việc hợp lệ và che nguyên nhân. Phân loại lỗi, ngân sách, backoff/jitter và quarantine giúp hệ thống tiến triển có kiểm soát.

### 7. Prefetch 10 có nghĩa worker chạy 10 threads không?

Không. Nó giới hạn delivery chưa ACK theo phạm vi được cấu hình. Cách app thực thi callbacks mới quyết định concurrency/parallelism. Node CPU đồng bộ vẫn có thể chặn event loop dù prefetch lớn.

### 8. FIFO có giữ thứ tự cập nhật DB không?

Không tự động. Hai jobs được lấy theo thứ tự có thể hoàn thành ngược, retry có thể đưa việc cũ quay lại sau. Muốn thứ tự theo aggregate phải thiết kế partition, consumer scheduling, version hoặc state transitions phù hợp.

### 9. Tại sao outbox vẫn cần inbox?

Outbox giữ nghĩa vụ nguồn bền vững nhưng relay có cửa sổ publish thành công rồi chưa mark sent. Retry tạo bản sao. Inbox/ledger giúp xử lý bản sao mà không lặp effect. Hai pattern đứng ở hai phía khác nhau của biên giao tiếp.

### 10. Nếu message đã nằm parking, có nên đổi eventId rồi replay?

Thông thường không. Đổi ID làm mất khả năng nhận diện cùng event. Nếu payload thật sự cần correction có ý nghĩa mới, đó là nghiệp vụ mới với event mới và causation, cần audit rõ. Replay cùng dữ kiện phải giữ định danh.

### 11. DLX có luôn bảo đảm at-least-once không?

Không. Cần hiểu loại queue, strategy, overflow và target. Trong bài, quorum source có at-least-once dead-lettering và reject-publish. Target không khả dụng có thể giữ message tại nguồn và tạo duplicate khi phục hồi. Đây không là cam kết exactly-once.

### 12. Một VPS ba containers có là cluster chịu lỗi không?

Nó có thể luyện giao thức cluster nhưng không chịu mất host chung. HA cần fault domains độc lập, majority phù hợp, mạng và storage được thiết kế. Backup giải quyết phục hồi dữ liệu; HA giải quyết tính sẵn sàng; hai mục tiêu liên quan nhưng không đồng nhất.

### 13. Queue rỗng có nghĩa mọi yêu cầu đã thành công?

Không. Consumer có thể ACK sai, messages có thể bị discard, hoặc còn trong retry/parking/outbox/ngoài broker. Kiểm tra bất biến nghiệp vụ và end-to-end completion mới trả lời yêu cầu đã đạt hay chưa.

### 14. Tại sao không giữ transaction SQL trong suốt HTTP provider call?

Thời gian mạng biến động kéo dài locks, chiếm pool và làm DB dễ nghẽn. Ngay cả giữ transaction cũng không rollback được provider ngoài. Nên ghi ý định, thực hiện bên ngoài transaction ngắn, rồi ghi bằng chứng kết quả với idempotency/reconciliation.

### 15. Khi nào nên tách queue?

Khi SLA, quota, retry policy, tài nguyên, quyền hoặc blast radius khác nhau rõ ràng. Tách transactional email khỏi batch marketing là ví dụ. Không tách chỉ vì mỗi function có tên khác; số queue và consumer tăng chi phí quản lý.

### 16. Tại sao global concurrency một không bảo đảm scheduler “chỉ làm một lần”?

Nó hạn chế số jobs active đồng thời, không loại bỏ retry hoặc redelivery theo thời gian. Một lượt có thể chạy lại sau crash. Business window key và dữ liệu trạng thái vẫn cần thiết.

### 17. Khi event mới tới trước event cũ, chỉ bỏ version thấp có luôn đúng không?

Phù hợp với một số snapshots có thể thay thế, nhưng nguy hiểm với events delta phải áp dụng đủ. Cần hiểu event có biểu diễn trạng thái đầy đủ hay thay đổi từng phần, rồi chọn buffer gaps, replay hoặc transition rules.

### 18. Vì sao old message sau restore có thể cộng lại dù trước đây inbox đúng?

Bản restore có thể ở trước thời điểm inbox/ledger được tạo. Tác dụng phụ ở provider hoặc hệ thống khác có thể không rollback cùng DB. Cần recovery plan và đối soát xuyên hệ thống, không chỉ restore một component.

### 19. Nếu Redis bị đầy, đổi eviction policy có phải cách chữa?

Đổi sang eviction có thể làm mất một phần cấu trúc queue. Cần xử lý dung lượng, retention, payload và đầu vào. Nếu queue hỏng, phục hồi từ nguồn nghĩa vụ và đối chiếu dữ liệu; không coi restart là kiểm chứng.

### 20. Một developer “master queue” cần chứng minh gì?

Biết chỉ ra nơi lưu nghĩa vụ, khóa idempotency ở từng biên, điều kiện ACK, cửa sổ lỗi, ngân sách retry, trạng thái unknown và quy trình phục hồi. Biết đo capacity và tác động người dùng. Biết nói “bằng chứng hiện có chưa bao phủ lỗi này” thay vì dùng từ tuyệt đối.

---

# Dọn môi trường có phạm vi

## Dừng dịch vụ nhưng giữ dữ liệu

Ô này chỉ áp dụng Compose local ở thư mục lab. Trên VPS dùng file `vps/compose.yml` và dotenv tương ứng, sau khi chắc chắn muốn dừng dịch vụ staging đó.

```bash {"name":"mq-stop-local-keep-data","excludeFromRunAll":"true"}
set -euo pipefail
LAB="$HOME/backend-labs/messaging"
test -f "$LAB/.be-mq-lab"
cd "$LAB"
docker compose -p be-mq stop
docker compose -p be-mq ps -a
```

## Xóa containers và volumes của lab local

**Mất dữ liệu lab:** Redis jobs, RabbitMQ messages, PostgreSQL database và các database restore trong volume sẽ bị xóa. Source, evidence và backups trên host vẫn giữ. Đọc lại project name và compose file trước khi chạy. Không dùng `docker system prune` để dọn chương này.

```bash {"name":"mq-reset-local-volumes","excludeFromRunAll":"true"}
set -euo pipefail
LAB="$HOME/backend-labs/messaging"
test -f "$LAB/.be-mq-lab"
cd "$LAB"
test "$(pwd -P)" = "$(cd "$HOME/backend-labs/messaging" && pwd -P)"
docker compose -p be-mq config --services
printf '%s\n' 'Removing only project be-mq containers and declared volumes from this compose.yml.'
docker compose -p be-mq down --volumes --remove-orphans
```

---

# Trạng thái kiểm chứng và cách tự nghiệm thu

Bản giáo trình đã được kiểm tra tĩnh bằng Bash parser cho các ô lệnh và Node.js v24 cho các file JavaScript tạo bằng heredoc. Kiểm tra tĩnh bắt lỗi cú pháp và một số sai lệch cấu trúc; nó không kiểm tra được Docker image có sẵn, npm registry, RabbitMQ policies thực tế, hành vi clock/timing, mạng hay kết quả SQL.

**Môi trường biên soạn không có Docker daemon, Runme, PostgreSQL server, Redis server hoặc RabbitMQ server. Vì vậy toàn bộ Compose và các kịch bản end-to-end chưa được chạy trong môi trường biên soạn.** Mọi dòng “kết quả mong đợi” là tiêu chí để bạn chạy và kiểm chứng, không phải log được bịa ra. Khi bắt đầu, chạy Lab 01–03, lưu baseline và lock; sau đó dùng assertions của từng lab để biết phiên bản/máy của bạn có đáp ứng giả định hay không.

Các faults dùng thời gian ngắn để học cơ chế. Không lấy lock 400 ms, retry 300 ms, queue limit 10.000 hoặc RAM limits của lab làm con số production mặc định. Đo workload thật và giữ điều kiện dừng rõ ràng.

## Tài liệu gốc để tra cứu tiếp

Các liên kết được đối chiếu ngày **17/11/2026**. Tài liệu “latest” có thể đổi; baseline của bạn nằm trong evidence/baseline.txt, package-lock.json và image digests.

| Nhóm | Nguồn chính thức |
|---|---|
| Runme | [Cell options](https://docs.runme.dev/configuration/cell-level/), [Document options](https://docs.runme.dev/configuration/document-level/) |
| BullMQ phiên bản | [v5 changelog](https://docs.bullmq.io/changelogs/changelog-v5) |
| BullMQ vận hành | [Production](https://docs.bullmq.io/guide/going-to-production), [Connections](https://docs.bullmq.io/guide/connections), [Metrics](https://docs.bullmq.io/guide/metrics/) |
| BullMQ vòng đời | [Retry](https://docs.bullmq.io/guide/retrying-failing-jobs), [Stalled](https://docs.bullmq.io/guide/jobs/stalled), [Shutdown](https://docs.bullmq.io/guide/workers/graceful-shutdown) |
| BullMQ điều phối | [Job Schedulers](https://docs.bullmq.io/guide/job-schedulers/), [Flows](https://docs.bullmq.io/guide/flows/), [Priority](https://docs.bullmq.io/guide/jobs/prioritized), [Deduplication](https://docs.bullmq.io/guide/jobs/deduplication) |
| BullMQ lưu giữ | [Auto-removal](https://docs.bullmq.io/guide/queues/auto-removal-of-jobs), [Delayed](https://docs.bullmq.io/guide/jobs/delayed) |
| amqplib | [Project/compatibility](https://github.com/amqp-node/amqplib), [API](https://amqp-node.github.io/amqplib/channel_api.html) |
| RabbitMQ giao thức | [Exchanges](https://www.rabbitmq.com/docs/exchanges), [Confirms/ACK](https://www.rabbitmq.com/docs/confirms), [Reliability](https://www.rabbitmq.com/docs/reliability) |
| RabbitMQ delivery | [Prefetch](https://www.rabbitmq.com/docs/consumer-prefetch), [Queues](https://www.rabbitmq.com/docs/queues), [TTL](https://www.rabbitmq.com/docs/ttl), [DLX](https://www.rabbitmq.com/docs/4.2/dlx) |
| RabbitMQ kiến trúc | [Quorum queues 4.2](https://www.rabbitmq.com/docs/4.2/quorum-queues), [Streams](https://www.rabbitmq.com/docs/streams) |
| RabbitMQ vận hành | [Monitoring](https://www.rabbitmq.com/docs/monitoring), [Alarms](https://www.rabbitmq.com/docs/alarms), [Access control](https://www.rabbitmq.com/docs/access-control), [Backup](https://www.rabbitmq.com/docs/backup) |
| Redis | [Persistence](https://redis.io/docs/latest/operate/oss_and_stack/management/persistence/) |
| PostgreSQL | [SELECT/locking](https://www.postgresql.org/docs/17/sql-select.html), [Isolation](https://www.postgresql.org/docs/17/transaction-iso.html), [pg_dump](https://www.postgresql.org/docs/17/app-pgdump.html) |
| Node và pg | [Node v24 signals](https://nodejs.org/docs/latest-v24.x/api/process.html#signal-events), [node-postgres transactions](https://node-postgres.com/features/transactions) |
| Docker | [Compose networking](https://docs.docker.com/compose/how-tos/networking/), [Ubuntu installation](https://docs.docker.com/engine/install/ubuntu/) |

