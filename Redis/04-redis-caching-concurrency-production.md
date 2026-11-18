---
shell: bash
---

# 04 — Redis: cấu trúc dữ liệu, caching, concurrency và vận hành

> Học Redis để biết **khi nào dùng, đảm bảo được gì, lúc nào không được tin vào nó**, rồi mới tối ưu tốc độ.
>
> Baseline tuyến chính: **Redis 7.4**, Node **24**, node-redis **5.8.2**, PostgreSQL **17** cho bài cache với database thật. Redis **8** là nhánh mở rộng có kiểm tra capability. Các phiên bản là baseline học tập, không phải tuyên bố latest.
>
> Biên soạn và đối chiếu tài liệu: **17/11/2026**. Học sau Docker và phần transaction trong Database. Dự kiến 30–50 giờ gồm làm bài, điều tra lỗi và viết lại lời giải bằng ngôn ngữ của bạn.

## Học như notebook

Dùng Runme + Bash, WSL2 nếu ở Windows. Chạy từng cell; `excludeFromRunAll` cố ý tránh chạy cả fault/cleanup cùng lúc. Mỗi cell `cd` vào `${HOME}/backend-labs/redis`. Helper là file có thật trong thư mục, không phụ thuộc shell state cell trước. Fence `text`/`ignore` là phần đọc.

**Phạm vi lab:** Compose `be-redis`, port `127.0.0.1:16379`; không publish DB. HA và Cluster dùng project riêng, không publish service ra Internet. Một VPS nhiều container chỉ mô phỏng protocol; không chống được mất host.

**Kiểm chứng:** kết quả ghi là điều bạn cần đo. Môi trường biên soạn không có Docker/Redis daemon nên chưa chạy end-to-end; xem file `00` về kiểm tra tĩnh đã thực hiện. Không có số throughput, tỷ lệ tăng tốc hay “an toàn 100%” được giả làm kết quả thực nghiệm.

## Bản đồ năng lực

| Nhóm | Câu hỏi cốt lõi | Evidence cần có |
|---|---|---|
| Cấu trúc dữ liệu | Lệnh nào, độ lớn nào, TTL nào? | schema key, memory thực, assertions |
| Atomicity | Một command khác transaction như thế nào? | race, MULTI lỗi, Lua partial write |
| Cache | Ai là source of truth? Stale chấp nhận bao lâu? | stampede, invalidation race, fallback |
| Coordination | Lock hết hạn mà worker vẫn chạy thì sao? | stale holder, fencing, bounded lease |
| Messaging | ACK, pending, replay, retention là gì? | claim lại pending, duplicate-safe effect |
| Vận hành | Restart, đầy RAM, failover mất gì? | restore drill, OOM/eviction, topology |
| Mở rộng | JSON/Search/vector có cần không? | feature probe, query thực, memory budget |

---

## 1. Bản chất: Redis là một data service có state

Redis lưu và xử lý cấu trúc dữ liệu trong memory. Nhiều thao tác cốt lõi được thực hiện tuần tự trên luồng thực thi chính, trong khi I/O, persistence và các tính năng khác có thể dùng thread/process riêng. Điều này giúp nhiều command đơn lẻ có semantics nguyên tử, **không** biến chuỗi GET→tính toán ở app→SET thành một giao dịch.

Latency quan sát từ backend gồm network RTT, chờ connection/client queue, chờ server, thực thi command, serialize và trả dữ liệu. “RAM nhanh” không loại bỏ các phần còn lại. Một lệnh quét/băm/sắp xếp lớn hoặc script chạy lâu có thể cản các client khác. CPU có thể là bottleneck; payload lớn có thể nghẽn network; fork/RDB có thể thêm memory pressure.

### 1.1. Bốn loại vai trò không nên trộn chính sách

| Vai trò | Ví dụ | Nếu mất dữ liệu |
|---|---|---|
| Cache có thể tái tạo | catalog, derived counts | query source, tăng tải, stale |
| State tạm nhưng quan trọng | session, OTP, rate counter | ảnh hưởng UX/security, phải có policy |
| Coordination | lease chống refresh trùng | có thể có nhiều worker, phải chịu được |
| Queue/durable workflow state | BullMQ jobs, Streams | mất việc hoặc xử lý lại, cần persistence/idempotency |

Cùng một process Redis có `maxmemory-policy` chung; tách logical DB 0/1 **không** tách memory, CPU, eviction hay failure domain. Đừng để cache `allkeys-lru` xóa keys quản lý job của BullMQ. Tách instance hoặc chọn managed isolation phù hợp.

**Bài suy nghĩ:** Redis chết 10 phút, endpoint xem sản phẩm có thể fallback vào DB; endpoint verify OTP không được “fallback cho đăng nhập luôn”. Availability policy phải theo nghiệp vụ. Fail-open/fail-closed cần quyết định cụ thể, không dùng chung một catch-return-default.

Nguồn: [Redis overview](https://redis.io/docs/latest/develop/get-started/), [latency diagnosis](https://redis.io/docs/latest/operate/oss_and_stack/management/optimization/latency/).

---

## 2. LOCAL — setup tuyến chính có auth và giới hạn tài nguyên

### 2.1. Preflight

```bash {"name":"redis-00-preflight","excludeFromRunAll":"true"}
set -euo pipefail
mkdir -p "${HOME}/backend-labs/redis"
cd "${HOME}/backend-labs/redis"
command -v docker
command -v python3
docker version
docker compose version
docker context show
df -h .
```

Đảm bảo context là máy lab. Mặc định chạy Redis khoảng 192MiB limit, Node tool 256MiB; DB chỉ khởi động cho bài tích hợp, limit 384MiB. Chừa RAM cho Docker/OS. Không chạy core+HA+Cluster đồng thời trên VPS 1–2GiB.

### 2.2. Generate config, helper và Node client

Password bên dưới **chỉ là credential minh họa trong môi trường private lab**. Production cần secret riêng, ACL tối thiểu, TLS/private network và cơ chế rotation. Không dùng credential lab trên service thật.

```bash {"name":"redis-01-generate","excludeFromRunAll":"true"}
set -euo pipefail
cd "${HOME}/backend-labs/redis"
if [ -e compose.yaml ]; then
  echo "Lab đã tồn tại; không ghi đè config/source."; exit 1
fi
mkdir -p app config scripts evidence backup
printf 'be-redis-course-v1\n' > .course-id
cat > config/redis.conf <<'CONF'
bind 0.0.0.0
protected-mode yes
port 6379
requirepass course-only-password
dir /data
appendonly yes
appendfsync everysec
save 60 100
maxmemory 64mb
maxmemory-policy allkeys-lru
slowlog-log-slower-than 10000
slowlog-max-len 128
latency-monitor-threshold 20
CONF
cat > compose.yaml <<'YAML'
name: be-redis
services:
  redis:
    image: redis:7.4-alpine
    command: ["redis-server", "/usr/local/etc/redis/redis.conf"]
    volumes:
      - ./config/redis.conf:/usr/local/etc/redis/redis.conf:ro
      - redis_data:/data
    ports: ["127.0.0.1:16379:6379"]
    mem_limit: 192m
    cpus: 0.50
    healthcheck:
      test: ["CMD-SHELL", "REDISCLI_AUTH=course-only-password redis-cli --no-auth-warning ping | grep -q PONG"]
      interval: 2s
      timeout: 2s
      retries: 20
    logging:
      driver: json-file
      options: { max-size: "5m", max-file: "2" }
  tools:
    build: ./app
    profiles: ["tools"]
    working_dir: /app
    volumes:
      - ./scripts:/scripts:ro
      - ./evidence:/evidence
    environment:
      REDIS_URL: redis://default:course-only-password@redis:6379
      DATABASE_URL: postgres://lab:course-db-password@db:5432/lab
    mem_limit: 256m
    cpus: 0.50
  db:
    image: postgres:17-bookworm
    profiles: ["database"]
    environment:
      POSTGRES_USER: lab
      POSTGRES_PASSWORD: course-db-password
      POSTGRES_DB: lab
    volumes: ["pg_data:/var/lib/postgresql/data"]
    mem_limit: 384m
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U lab -d lab"]
      interval: 2s
      timeout: 2s
      retries: 20
volumes:
  redis_data:
  pg_data:
YAML
cat > app/package.json <<'JSON'
{
  "name": "redis-backend-course",
  "private": true,
  "type": "module",
  "dependencies": { "redis": "5.8.2", "pg": "8.16.3" }
}
JSON
cat > app/Dockerfile <<'DOCKER'
FROM node:24-bookworm-slim
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci --omit=dev
COPY client.mjs ./
USER node
CMD ["node", "--version"]
DOCKER
cat > app/client.mjs <<'JS'
import { createClient } from 'redis';
export async function connect(url = process.env.REDIS_URL) {
  const client = createClient({
    url,
    disableOfflineQueue: true,
    commandsQueueMaxLength: 128,
    socket: { connectTimeout: 1000, reconnectStrategy: false }
  });
  client.on('error', err => console.error('redis-error', err.code ?? err.name));
  await client.connect();
  return client;
}
export const sleep = ms => new Promise(resolve => setTimeout(resolve, ms));
export const command = (client, ...args) => client.sendCommand(args.map(String));
JS
cat > rcli.sh <<'SH'
#!/usr/bin/env bash
set -euo pipefail
cd "$(dirname "$0")"
test "$(cat .course-id)" = be-redis-course-v1
docker compose -p be-redis exec -T -e REDISCLI_AUTH=course-only-password \
  redis redis-cli --no-auth-warning --raw -e "$@"
SH
cat > run-node.sh <<'SH'
#!/usr/bin/env bash
set -euo pipefail
cd "$(dirname "$0")"
test "$(cat .course-id)" = be-redis-course-v1
docker compose -p be-redis run --rm --no-deps tools node "$@"
SH
chmod +x rcli.sh run-node.sh
```

**Lưu ý import:** script trong `/scripts` nhập client ở `/app/client.mjs`; `pg` của script nên nhập qua `/app/node_modules/pg/lib/index.js` hoặc dùng `createRequire('/app/package.json')`. Node resolve package theo vị trí file, không theo `cwd` đơn thuần. Các bài dưới đã xử lý điều này.

```bash {"name":"redis-02-start","excludeFromRunAll":"true"}
set -euo pipefail
cd "${HOME}/backend-labs/redis"
docker run --rm --user "$(id -u):$(id -g)" \
  -e npm_config_cache=/tmp/npm -v "$PWD/app:/app" -w /app \
  node:24-bookworm-slim npm install --package-lock-only --ignore-scripts --no-audit --no-fund
docker compose -p be-redis build tools
docker compose -p be-redis up -d redis
ready=0
for i in $(seq 1 60); do
  if [ "$(./rcli.sh PING 2>/dev/null || true)" = PONG ]; then ready=1; break; fi
  sleep .5
done
test "$ready" = 1
./rcli.sh INFO server | tee evidence/server-info.txt
docker image inspect redis:7.4-alpine --format '{{json .RepoDigests}}' | tee evidence/redis-digest.txt
./rcli.sh CONFIG GET maxmemory maxmemory-policy appendonly appendfsync
```

**Đạt:** PONG, `redis_version` đúng dòng 7.4, config phản ánh AOF/maxmemory/policy. Tag major/minor là mutable, digest là bằng chứng image thật. Không suy Redis Stack image đồng nghĩa Redis 8 hoặc mọi lệnh đều có sẵn.

---

## 3. Keyspace, TTL, schema và SCAN

Key nên mang namespace, version và identity cần thiết, ví dụ `cache:catalog:v1:product:42`. Tên key giúp quản lý nhưng không tạo authorization. Tránh dùng email/phone/token nguyên văn làm key khi logs, metrics hay admin tools có thể lộ chúng.

Đặt TTL là policy về độ cũ/lifetime, không phải lịch chính xác gọi callback vào đúng giây. Key có thể bị expire/evict hoặc bị xóa. TTL `-1` nghĩa có key nhưng chưa có expiry; `-2` nghĩa không tồn tại. SET mới thường thay TTL trừ khi bạn chủ động giữ/đặt lại theo option; HSET field có semantics khác với SET cả key.

### Lab — TTL bị mất khi overwrite

```bash {"name":"redis-03-ttl-overwrite","excludeFromRunAll":"true"}
set -euo pipefail
cd "${HOME}/backend-labs/redis"
./rcli.sh SET lab:ttl:profile v1 EX 30
test "$(./rcli.sh TTL lab:ttl:profile)" -gt 0
./rcli.sh SET lab:ttl:profile v2
test "$(./rcli.sh TTL lab:ttl:profile)" = -1
./rcli.sh SET lab:ttl:profile v3 EX 30
./rcli.sh SET lab:ttl:profile v4 KEEPTTL
test "$(./rcli.sh TTL lab:ttl:profile)" -gt 0
./rcli.sh DEL lab:ttl:profile
test "$(./rcli.sh TTL lab:ttl:profile)" = -2
```

**Sự cố production:** code refresh session overwrite SET nhưng quên TTL, memory tăng dần và session tồn tại quá lâu. Test phải kiểm tra cả value lẫn TTL. Không coi key tồn tại là đủ.

### Lab — scan hết cursor, chấp nhận duplicate

`SCAN COUNT 20` là gợi ý lượng công việc, không phải đúng 20 keys. Có thể nhận batch rỗng nhưng cursor chưa 0. Quét khi keyspace thay đổi không phải snapshot; có thể thấy key lặp. Cleanup/migration cần idempotent, có rate limit và namespace rõ. Không dùng `KEYS *` để “tiện” trên dataset lớn.

```bash {"name":"redis-04-scan-cursor","excludeFromRunAll":"true"}
set -euo pipefail
cd "${HOME}/backend-labs/redis"
cat > scripts/scan.mjs <<'JS'
import assert from 'node:assert/strict';
import { connect, command } from '/app/client.mjs';
const r = await connect();
try {
  for (let i=0;i<50;i++) await command(r,'SET',`lab:scan:item:${i}`,i,'EX',600);
  let cursor='0';
  const seen=new Set();
  let pages=0;
  do {
    const reply=await command(r,'SCAN',cursor,'MATCH','lab:scan:item:*','COUNT',10);
    cursor=String(reply[0]);
    for (const key of reply[1]) seen.add(key);
    pages++;
    assert(pages<10000,'Unexpected cursor loop');
  } while (cursor!=='0');
  assert.equal(seen.size,50);
  console.log({pages,unique_keys:seen.size});
} finally { await r.close(); }
JS
./run-node.sh /scripts/scan.mjs
```

Nguồn: [SET](https://redis.io/docs/latest/commands/set/), [TTL](https://redis.io/docs/latest/commands/ttl/), [SCAN guarantees](https://redis.io/docs/latest/commands/scan/).

---

## 4. String, Hash, List, Set, Sorted Set — chọn theo thao tác

Một object JSON trong String thuận tiện khi đọc toàn bộ, nhưng update một field bằng read-modify-write có race. Hash cho phép HINCRBY/HGET field riêng. Không kết luận Hash luôn ít RAM hơn JSON: encoding, kích thước field/value và overhead cần đo. Dùng `OBJECT ENCODING`/`MEMORY USAGE` thay khẩu quyết.

List giữ thứ tự và cho trùng; Set phù hợp membership/unique; Sorted Set gắn score để rank/range. “Redis nhanh” không làm `HGETALL` hash triệu fields, `SMEMBERS` tập khổng lồ hay `LRANGE 0 -1` trở thành thao tác nhỏ. Thiết kế bounded results.

### 4.1. Lab — object và hash-field TTL

```bash {"name":"redis-05-hash-field-expiry","excludeFromRunAll":"true"}
set -euo pipefail
cd "${HOME}/backend-labs/redis"
./rcli.sh DEL lab:cart:101 lab:sessions:101
./rcli.sh HSET lab:cart:101 sku-a 2 sku-b 1
./rcli.sh HINCRBY lab:cart:101 sku-a 1
test "$(./rcli.sh HGET lab:cart:101 sku-a)" = 3
./rcli.sh HSET lab:sessions:101 mobile session-a browser session-b
./rcli.sh HEXPIRE lab:sessions:101 2 FIELDS 1 browser
expired=0
for i in $(seq 1 30); do
  if [ "$(./rcli.sh HEXISTS lab:sessions:101 browser)" = 0 ]; then expired=1; break; fi
  sleep .2
done
test "$expired" = 1
test "$(./rcli.sh HGET lab:sessions:101 mobile)" = session-a
./rcli.sh OBJECT ENCODING lab:cart:101
./rcli.sh MEMORY USAGE lab:cart:101
```

`HEXPIRE` có từ 7.4, không dùng mù trên 7.2. Ghi đè field có thể ảnh hưởng field expiry; đọc [HEXPIRE](https://redis.io/docs/latest/commands/hexpire/) khi thiết kế session refresh. Session trong cache vẫn cần revoke/auth policy và bảo vệ token, không lưu JWT nhạy cảm tùy tiện trong output notebook.

### 4.2. Lab — history có giới hạn và unique membership

```bash {"name":"redis-06-lists-sets","excludeFromRunAll":"true"}
set -euo pipefail
cd "${HOME}/backend-labs/redis"
./rcli.sh DEL lab:history:101 lab:team:a lab:team:b
for item in search-1 search-2 search-3 search-4 search-5 search-6; do
  ./rcli.sh LPUSH lab:history:101 "$item" >/dev/null
  ./rcli.sh LTRIM lab:history:101 0 4 >/dev/null
done
test "$(./rcli.sh LLEN lab:history:101)" = 5
./rcli.sh LRANGE lab:history:101 0 4
./rcli.sh SADD lab:team:a tuan may khang
./rcli.sh SADD lab:team:b tuan linh
./rcli.sh SINTER lab:team:a lab:team:b
test "$(./rcli.sh SISMEMBER lab:team:a tuan)" = 1
```

LPUSH và LTRIM riêng lẻ có cửa sổ list tạm vượt 5; nếu bound nghiêm ngặt hoặc hành vi xen kẽ quan trọng, gộp MULTI/Lua sau khi học atomicity. Với history không quan trọng, chi phí có thể chấp nhận; ghi rõ assumption.

### 4.3. Lab — ranking, tie và range

```bash {"name":"redis-07-leaderboard","excludeFromRunAll":"true"}
set -euo pipefail
cd "${HOME}/backend-labs/redis"
./rcli.sh DEL lab:rank:weekly
./rcli.sh ZADD lab:rank:weekly 100 user-a 120 user-b 120 user-c 50 user-d
./rcli.sh ZINCRBY lab:rank:weekly 30 user-a
./rcli.sh ZRANGE lab:rank:weekly 0 2 REV WITHSCORES
test "$(./rcli.sh ZREVRANK lab:rank:weekly user-a)" = 0
./rcli.sh ZRANGE lab:rank:weekly 100 130 BYSCORE WITHSCORES
```

Score là floating-point; đừng dùng nó làm ledger tiền với số lớn/độ chính xác yêu cầu strict. Bằng điểm có quy tắc tie theo member; nếu nghiệp vụ muốn “ai đạt trước đứng trước”, thiết kế tie-breaker riêng và chứng minh precision. Rank 0 là vị trí đầu, không phải user có thứ hạng 0.

### 4.4. Lab — bitmap, HLL, GEO và chi phí thực

```bash {"name":"redis-08-compact-types","excludeFromRunAll":"true"}
set -euo pipefail
cd "${HOME}/backend-labs/redis"
./rcli.sh DEL lab:attendance:2026:101 lab:uv:today lab:drivers:hanoi
./rcli.sh SETBIT lab:attendance:2026:101 0 1
./rcli.sh SETBIT lab:attendance:2026:101 364 1
test "$(./rcli.sh BITCOUNT lab:attendance:2026:101)" = 2
./rcli.sh STRLEN lab:attendance:2026:101
./rcli.sh MEMORY USAGE lab:attendance:2026:101
./rcli.sh PFADD lab:uv:today visitor-a visitor-b visitor-a visitor-c
./rcli.sh PFCOUNT lab:uv:today
./rcli.sh MEMORY USAGE lab:uv:today
./rcli.sh GEOADD lab:drivers:hanoi 105.8525 21.0285 driver-a 105.8412 21.0368 driver-b
./rcli.sh GEOSEARCH lab:drivers:hanoi FROMLONLAT 105.8525 21.0285 BYRADIUS 3 km ASC WITHDIST
```

**Đọc kết quả:** 365 bits cần 46 bytes payload, nhưng key/object/allocator tốn thêm. HLL đếm gần đúng, không dùng cho billing số exact hoặc membership lookup. GEO là khoảng cách địa lý xấp xỉ theo mô hình, không phải thời gian đi đường; cần độ tươi GPS và lọc tài xế offline. Không gán user ID rất lớn trực tiếp làm offset bitmap rồi ngạc nhiên vì String phình to.

Nguồn: [data types](https://redis.io/docs/latest/develop/data-types/), [HyperLogLog](https://redis.io/docs/latest/develop/data-types/probabilistic/hyperloglogs/), [GEOSEARCH](https://redis.io/docs/latest/commands/geosearch/).

---

## 5. Atomicity, pipeline, MULTI/EXEC và WATCH

| Công cụ | Giải quyết | Không giải quyết |
|---|---|---|
| Một command atomic | Không xen giữa nội bộ command | Nhiều command của app |
| Pipeline | Giảm round trips, gộp transport | Transaction isolation/rollback |
| MULTI/EXEC | Chạy nhóm command không xen kẽ | Đọc kết quả rồi branch trong nhóm; rollback runtime error |
| WATCH | Optimistic concurrency: abort khi watched key đổi | Retry policy tự động, transaction SQL |
| Lua/Function | Đọc, kiểm tra, ghi phía server cùng lượt | Rollback mọi lỗi; atomic cùng DB/provider |

### 5.1. Lab — read-modify-write mất cập nhật

```bash {"name":"redis-09-lost-update","excludeFromRunAll":"true"}
set -euo pipefail
cd "${HOME}/backend-labs/redis"
cat > scripts/race.mjs <<'JS'
import assert from 'node:assert/strict';
import { connect, command } from '/app/client.mjs';
const a=await connect(), b=await connect();
try {
  await command(a,'SET','lab:race:counter',0);
  const [oldA,oldB]=await Promise.all([
    command(a,'GET','lab:race:counter'),command(b,'GET','lab:race:counter')
  ]);
  await Promise.all([
    command(a,'SET','lab:race:counter',Number(oldA)+1),
    command(b,'SET','lab:race:counter',Number(oldB)+1)
  ]);
  assert.equal(await command(a,'GET','lab:race:counter'),'1');
  await command(a,'SET','lab:race:counter',0);
  await Promise.all([command(a,'INCR','lab:race:counter'),command(b,'INCR','lab:race:counter')]);
  assert.equal(await command(a,'GET','lab:race:counter'),'2');
  console.log('PASS: barrier ensured both read 0; GET+SET lost one, INCR preserved both');
} finally { await Promise.all([a.close(),b.close()]); }
JS
./run-node.sh /scripts/race.mjs
```

Đây là interleaving có chủ đích bằng await barrier, không phụ thuộc hai lệnh “may mắn” chạy cùng mili-giây. Single-thread Redis không cứu logic nằm giữa hai network calls.

### 5.2. Lab — MULTI và Lua không có rollback như SQL

```bash {"name":"redis-10-no-rollback","excludeFromRunAll":"true"}
set -euo pipefail
cd "${HOME}/backend-labs/redis"
./rcli.sh DEL lab:tx:bad lab:tx:after lab:lua:before
if printf 'MULTI\nSET lab:tx:bad text\nINCR lab:tx:bad\nSET lab:tx:after yes\nEXEC\n' | ./rcli.sh; then
  echo "Xem từng reply trong EXEC: redis-cli có thể không báo lỗi nested reply bằng exit status."
fi
test "$(./rcli.sh GET lab:tx:bad)" = text
test "$(./rcli.sh GET lab:tx:after)" = yes
if ./rcli.sh EVAL 'redis.call("SET",KEYS[1],"written"); return redis.call("INCR",KEYS[2])' \
  2 lab:lua:before lab:tx:bad; then
  echo "LỖI: mong đợi runtime error"; exit 1
fi
test "$(./rcli.sh GET lab:lua:before)" = written
```

**Bản chất:** nhóm không bị command client khác chen vào, nhưng lỗi runtime không tự undo writes trước nó. Validate argument, type và điều kiện **trước** ghi. Runtime script dài cũng chặn server; không biến Lua thành worker xử lý video hay vòng lặp hàng triệu records.

### 5.3. Lab — WATCH bảo vệ optimistic update

```bash {"name":"redis-11-watch-conflict","excludeFromRunAll":"true"}
set -euo pipefail
cd "${HOME}/backend-labs/redis"
cat > scripts/watch.mjs <<'JS'
import assert from 'node:assert/strict';
import { connect, command } from '/app/client.mjs';
const a=await connect(), b=await connect();
try {
  await command(a,'SET','lab:watch:version',1);
  await a.watch('lab:watch:version');
  const old=await command(a,'GET','lab:watch:version');
  await command(b,'INCR','lab:watch:version');
  let aborted=false;
  try { await a.multi().set('lab:watch:version',String(Number(old)+1)).exec(); }
  catch(error) {
    if (error.constructor.name==='WatchError') aborted=true;
    else throw error;
  }
  assert(aborted,'Expected optimistic conflict');
  assert.equal(await command(b,'GET','lab:watch:version'),'2');
  console.log('PASS: conflict explicit; caller must reread/retry within budget');
} finally { await Promise.all([a.close(),b.close()]); }
JS
./run-node.sh /scripts/watch.mjs
```

WATCH gắn **connection**, nên không chia cùng client cho transaction WATCH của các request concurrent mà không isolation/pool phù hợp. Retry phải reread và giới hạn; request nào cũng retry vô hạn có thể tạo livelock.

### 5.4. Lab — pipeline là tối ưu transport

```bash {"name":"redis-12-pipeline","excludeFromRunAll":"true"}
set -euo pipefail
cd "${HOME}/backend-labs/redis"
cat > scripts/pipeline.mjs <<'JS'
import assert from 'node:assert/strict';
import { connect, command } from '/app/client.mjs';
const r=await connect();
try {
  const n=100;
  await command(r,'SET','lab:pipe:counter',0);
  let t=performance.now();
  for(let i=0;i<n;i++) await command(r,'INCR','lab:pipe:counter');
  const sequential=performance.now()-t;
  t=performance.now();
  await Promise.all(Array.from({length:n},()=>command(r,'INCR','lab:pipe:counter')));
  const batched=performance.now()-t;
  assert.equal(await command(r,'GET','lab:pipe:counter'),String(2*n));
  console.log({sequential_ms:sequential,batched_ms:batched});
} finally { await r.close(); }
JS
./run-node.sh /scripts/pipeline.mjs
```

node-redis có auto-pipelining cho requests cùng event-loop tick. Đo trên máy bạn, không assert phải nhanh hơn một tỷ lệ cố định. Pipeline rất lớn giữ nhiều buffers/replies, làm latency tail xấu; chunk có giới hạn. Bài dùng 100 lệnh không được nâng lên hàng triệu bằng cách xóa budget.

Nguồn: [Redis transactions](https://redis.io/docs/latest/develop/using-commands/transactions/), [Lua execution](https://redis.io/docs/latest/develop/programmability/eval-intro/), [pipelining](https://redis.io/docs/latest/develop/using-commands/pipelining/), [node-redis](https://github.com/redis/node-redis).


---

## 6. Lua thực hành — invariant rõ trước khi viết script

“Invariant” là điều phải luôn đúng trong phạm vi bạn kiểm soát. Với stock demo: stock là integer không âm; quantity phải integer dương; một buyer chỉ được reserve một lần; thao tác kiểm tra và ghi trong cùng Redis instance. Điều này chưa tạo order trong SQL, chưa charge provider và chưa bảo đảm mất node không mất reservation.

### 6.1. Lab — stock có validate, nhiều request đồng thời

```bash {"name":"redis-13-stock-lua","excludeFromRunAll":"true"}
set -euo pipefail
cd "${HOME}/backend-labs/redis"
cat > scripts/reserve.lua <<'LUA'
local qty = tonumber(ARGV[1])
if not qty or qty <= 0 or qty ~= math.floor(qty) or qty > 10000 then
  return redis.error_reply("INVALID_QUANTITY")
end
if not ARGV[2] or string.len(ARGV[2]) < 1 or string.len(ARGV[2]) > 100 then
  return redis.error_reply("INVALID_BUYER")
end
local st = redis.call("TYPE", KEYS[1]).ok
local bt = redis.call("TYPE", KEYS[2]).ok
if st ~= "string" or (bt ~= "none" and bt ~= "set") then
  return redis.error_reply("INVALID_KEY_TYPE")
end
local stock = tonumber(redis.call("GET", KEYS[1]))
if not stock or stock < 0 or stock ~= math.floor(stock) then
  return redis.error_reply("INVALID_STOCK")
end
if redis.call("SISMEMBER", KEYS[2], ARGV[2]) == 1 then return -1 end
if stock < qty then return 0 end
redis.call("DECRBY", KEYS[1], qty)
redis.call("SADD", KEYS[2], ARGV[2])
return 1
LUA
cat > scripts/stock.mjs <<'JS'
import assert from 'node:assert/strict';
import { readFile } from 'node:fs/promises';
import { connect,command } from '/app/client.mjs';
const r=await connect();
const stock='lab:stock:{sku42}:qty', buyers='lab:stock:{sku42}:buyers';
try {
  const lua=await readFile('/scripts/reserve.lua','utf8');
  await command(r,'SET',stock,3);
  await command(r,'DEL',buyers);
  const results=await Promise.all(['a','b','a','c','d','e'].map(user=>
    command(r,'EVAL',lua,2,stock,buyers,1,user)));
  assert.equal(results.filter(x=>x===1).length,3);
  assert.equal(await command(r,'GET',stock),'0');
  assert.equal(await command(r,'SCARD',buyers),3);
  await assert.rejects(command(r,'EVAL',lua,2,stock,buyers,-1,'attacker'),/INVALID_QUANTITY/);
  assert.equal(await command(r,'GET',stock),'0');
  console.log({results,remaining:0});
} finally { await r.close(); }
JS
./run-node.sh /scripts/stock.mjs
```

Hai key dùng `{sku42}` để colocate khi chuyển Cluster. Nếu đưa toàn bộ hệ thống vào `{all}`, bạn vừa gom mọi key về một shard và mất lợi ích chia tải.

**Từ lab sang order thật:** reservation có operation ID, expiry/compensation, durable record và reconcile stock. Nếu DB commit fail sau Redis reserve, phải quyết định trả stock hay retry; nếu response mất sau thành công, cùng operation ID phải nhận lại kết quả. Không gán “không bao giờ oversell” cho toàn hệ thống chỉ vì Redis Lua đạt 3 accepted trong test này.

### 6.2. Fixed window, sliding window và token bucket

| Thuật toán | Đặc điểm | Tradeoff |
|---|---|---|
| Counter + TTL | Dễ, ít state | Window từ request đầu hoặc epoch tùy thiết kế; burst ở ranh giới |
| Sliding log ZSET | Đếm từng request trong khoảng gần nhất | Memory theo lượng request, cần trim và member unique |
| Sliding counter | Xấp xỉ bằng hai window | Nhẹ hơn, không exact |
| Token bucket | Cho burst đến capacity, refill theo thời gian | Cần giải thích burst/rate và clock |
| Concurrency limiter | Giới hạn tác vụ đang chạy | Khác request/giây, cần release/lease đúng |

### 6.3. Lab — fixed window atomic và sliding log thật

```bash {"name":"redis-14-rate-limiters","excludeFromRunAll":"true"}
set -euo pipefail
cd "${HOME}/backend-labs/redis"
cat > scripts/fixed-window.lua <<'LUA'
local limit, ttl = tonumber(ARGV[1]), tonumber(ARGV[2])
if not limit or limit<1 or limit>1000000 or limit~=math.floor(limit) or
   not ttl or ttl<1 or ttl>86400000 or ttl~=math.floor(ttl) then
  return redis.error_reply("INVALID_LIMIT")
end
local t=redis.call("TYPE",KEYS[1]).ok
if t~="none" and t~="string" then return redis.error_reply("WRONG_KEY_TYPE") end
local value=redis.call("GET",KEYS[1])
if value and not tonumber(value) then return redis.error_reply("INVALID_COUNTER") end
local count=redis.call("INCR",KEYS[1])
if count==1 then redis.call("PEXPIRE",KEYS[1],ttl) end
return {count<=limit and 1 or 0, count, redis.call("PTTL",KEYS[1])}
LUA
cat > scripts/sliding-window.lua <<'LUA'
local limit, window = tonumber(ARGV[1]), tonumber(ARGV[2])
if not limit or limit<1 or limit>1000000 or limit~=math.floor(limit) or
   not window or window<1 or window>86400000 or window~=math.floor(window) or not ARGV[3] then
  return redis.error_reply("INVALID_LIMIT")
end
local t=redis.call("TYPE",KEYS[1]).ok
if t~="none" and t~="zset" then return redis.error_reply("WRONG_KEY_TYPE") end
local clock=redis.call("TIME")
local now=tonumber(clock[1])*1000+math.floor(tonumber(clock[2])/1000)
redis.call("ZREMRANGEBYSCORE",KEYS[1],"-inf",now-window)
local count=redis.call("ZCARD",KEYS[1])
if count<limit then
  redis.call("ZADD",KEYS[1],now,ARGV[3])
  redis.call("PEXPIRE",KEYS[1],window)
  return {1,count+1,0}
end
local first=redis.call("ZRANGE",KEYS[1],0,0,"WITHSCORES")
local retry=math.max(1,tonumber(first[2])+window-now)
return {0,count,retry}
LUA
cat > scripts/limiter.mjs <<'JS'
import assert from 'node:assert/strict';
import { randomUUID } from 'node:crypto';
import { readFile } from 'node:fs/promises';
import { connect,command } from '/app/client.mjs';
const r=await connect();
try {
  for(const kind of ['fixed','sliding']) {
    const lua=await readFile(`/scripts/${kind}-window.lua`,'utf8');
    const key=`lab:limit:${kind}:user101`;
    await command(r,'DEL',key);
    const replies=[];
    for(let i=0;i<5;i++) replies.push(
      await command(r,'EVAL',lua,1,key,3,10000,randomUUID()));
    assert.equal(replies.filter(v=>v[0]===1).length,3);
    assert.equal(replies.filter(v=>v[0]===0).length,2);
    console.log(kind,replies);
  }
  const fixed=await readFile('/scripts/fixed-window.lua','utf8');
  const sliding=await readFile('/scripts/sliding-window.lua','utf8');
  for(const [lua,key] of [[fixed,'lab:limit:invalid-fixed'],[sliding,'lab:limit:invalid-sliding']]) {
    await command(r,'DEL',key);
    await assert.rejects(command(r,'EVAL',lua,1,key,3,'1e30',randomUUID()),/INVALID_LIMIT/);
    assert.equal(await command(r,'EXISTS',key),0);
  }
} finally { await r.close(); }
JS
./run-node.sh /scripts/limiter.mjs
```

**Đạt:** mỗi thuật toán cho 3, chặn 2 khi chạy trong 10 giây. Lab giới hạn window/TTL tối đa 24 giờ và limit tối đa 1.000.000; kiểm tra các bound trước mọi write để giá trị quá lớn không làm PEXPIRE lỗi sau khi INCR/trim đã chạy. Hai assertion invalid-window xác nhận key chưa được tạo. Để so semantics ranh giới, dựng trace timestamp và phát request ở gần cuối window; không gọi counter+TTL là sliding window. Production dùng identity đã auth, tính `Retry-After`, chọn clock policy và xử lý Redis unavailable. Các lệnh EVAL/TIME hoạt động ở server, nhưng clock wall-time vẫn có thể thay đổi; cần NTP/clock monitoring.

**Bài mở rộng:** OTP limiter có budget theo account + IP + thiết bị; nếu không kiểm soát key cardinality, người tấn công tạo vô số key khiến Redis đầy. Sau auth thất bại không ghi raw password/OTP vào key hoặc log.

---

## 7. Lease, lock và fencing — hiểu giới hạn trước khi dùng cho tiền

`SET lock token NX PX ttl` là **lease có hạn**, không phải quyền sở hữu vĩnh viễn. Token ngẫu nhiên giúp safe unlock không xóa khóa của người khác. Nó không ngăn worker A bị pause, lease hết, B bắt đầu, rồi A sống lại và ghi đè B.

```mermaid {"ignore":"true"}
sequenceDiagram
    participant A as Worker A
    participant R as Redis
    participant B as Worker B
    participant D as Resource
    A->>R: Acquire lease A
    Note over A: Pause lâu hơn TTL
    B->>R: Acquire lease B
    B->>D: Write với fence 12
    A->>D: Write cũ với fence 11
    D-->>A: Reject stale fence
```

Fencing chỉ hữu ích nếu **resource nhận write tự kiểm tra token monotonic**. Gắn token vào log mà DB không kiểm tra không bảo vệ gì. Counter trong Redis replication bất đồng bộ cũng có thể lùi khi failover/loss; nguồn token cần được thiết kế phù hợp mức correctness. Với tiền, ưu tiên DB transaction/conditional update, durable idempotency và provider key/reconciliation, không lấy Redis lease làm duy nhất.

### Lab — safe unlock không đồng nghĩa hết stale worker

```bash {"name":"redis-15-lease-expiry","excludeFromRunAll":"true"}
set -euo pipefail
cd "${HOME}/backend-labs/redis"
cat > scripts/unlock.lua <<'LUA'
if redis.call("GET",KEYS[1]) == ARGV[1] then
  return redis.call("DEL",KEYS[1])
end
return 0
LUA
cat > scripts/lease.mjs <<'JS'
import assert from 'node:assert/strict';
import { randomUUID } from 'node:crypto';
import { readFile } from 'node:fs/promises';
import { connect,command,sleep } from '/app/client.mjs';
const r=await connect();
try {
  const key='lab:lease:refresh', a=randomUUID(), b=randomUUID();
  const unlock=await readFile('/scripts/unlock.lua','utf8');
  await command(r,'DEL',key);
  assert.equal(await command(r,'SET',key,a,'NX','PX',500),'OK');
  assert.equal(await command(r,'SET',key,b,'NX','PX',500),null);
  const deadline=Date.now()+5000;
  while(await command(r,'EXISTS',key)) {
    assert(Date.now()<deadline,'TTL did not expire');
    await sleep(50);
  }
  assert.equal(await command(r,'SET',key,b,'NX','PX',5000),'OK');
  assert.equal(await command(r,'EVAL',unlock,1,key,a),0);
  assert.equal(await command(r,'GET',key),b);
  let lastFence=0, state='';
  function resourceWrite(fence,value) {
    if(fence<=lastFence) return false;
    lastFence=fence; state=value; return true;
  }
  assert(resourceWrite(12,'new'));
  assert.equal(resourceWrite(11,'stale'),false);
  assert.equal(state,'new');
  console.log('Safe unlock verified; in-memory resource illustrates fencing only');
  await command(r,'EVAL',unlock,1,key,b);
} finally { await r.close(); }
JS
./run-node.sh /scripts/lease.mjs
```

Phần `resourceWrite` chỉ minh họa thuật toán, không là durable implementation. Bài Database có `UPDATE ... WHERE version=...`; khi tích hợp, điều kiện phải nằm **trong cùng atomic write ở resource**. Side effect bên ngoài Redis (email, charge) cần idempotency tại nơi thực hiện hoặc đối soát, không có Lua nào làm nó thành transaction chung.

Nguồn: [Distributed locks](https://redis.io/docs/latest/develop/use/patterns/distributed-locks/). Đọc cả phần consistency/fencing, không chỉ copy đoạn SET/DEL.

---

## 8. Cache-aside với PostgreSQL thật

### 8.1. Thiết kế trước code

Read path: cache hit trả snapshot; miss đọc DB rồi populate. Write path: DB là source of truth, commit trước, sau đó invalidation/update cache theo policy. Điều gì xảy ra giữa hai bước? Phải có câu trả lời, nhất là nếu cache lưu thông tin nhạy với tính nhất quán.

- TTL giới hạn tuổi entry kể từ lúc ghi, nhưng entry có thể chứa một snapshot đã cũ khi ghi.
- Jitter dàn thời điểm expiry, không sửa stale data do race.
- Negative cache tránh query lặp cho “không tồn tại”, nhưng cần TTL ngắn và invalidation khi tạo mới.
- Single-flight/lease hạn chế refresh trùng, không tạo consistency với DB.
- Stale-while-revalidate giúp availability cho dữ liệu được phép cũ; phải có hard maximum age.
- Cache outage fallback không được vượt sức chịu tải DB: pool/concurrency/admission/circuit breaker.

### 8.2. Lab — tạo source of truth

```bash {"name":"redis-16-cache-database-setup","excludeFromRunAll":"true"}
set -euo pipefail
cd "${HOME}/backend-labs/redis"
docker compose -p be-redis --profile database up -d db
ready=0
for i in $(seq 1 60); do
  if docker compose -p be-redis exec -T db pg_isready -U lab -d lab >/dev/null 2>&1; then ready=1; break; fi
  sleep .5
done
test "$ready" = 1
docker compose -p be-redis exec -T db psql -U lab -d lab -v ON_ERROR_STOP=1 <<'SQL'
CREATE SCHEMA IF NOT EXISTS redis_lab;
CREATE TABLE IF NOT EXISTS redis_lab.products (
  id integer PRIMARY KEY,
  name text NOT NULL,
  price_minor bigint NOT NULL CHECK(price_minor >= 0),
  version bigint NOT NULL DEFAULT 1
);
CREATE TABLE IF NOT EXISTS redis_lab.stats (
  id integer PRIMARY KEY,
  reads integer NOT NULL DEFAULT 0
);
INSERT INTO redis_lab.products(id,name,price_minor,version)
VALUES (42,'Training product',10000,1)
ON CONFLICT(id) DO NOTHING;
INSERT INTO redis_lab.stats(id,reads) VALUES(1,0) ON CONFLICT(id) DO NOTHING;
SQL
```

Data schema là riêng `redis_lab`, không sửa bảng công việc. Giá tiền demo dùng integer minor units, không float. Không dùng cache này để quyết định số dư có đủ chuyển tiền; quyết định ấy phải theo transaction ở DB.

### 8.3. Lab — tái hiện stampede rồi giảm bằng lease

```bash {"name":"redis-17-stampede-real-db","excludeFromRunAll":"true"}
set -euo pipefail
cd "${HOME}/backend-labs/redis"
cat > scripts/cache.mjs <<'JS'
import assert from 'node:assert/strict';
import { createRequire } from 'node:module';
import { randomUUID } from 'node:crypto';
import { connect,command,sleep } from '/app/client.mjs';
const require=createRequire('/app/package.json');
const { Pool }=require('pg');
const pool=new Pool({connectionString:process.env.DATABASE_URL,max:4,connectionTimeoutMillis:1000});
const r=await connect();
const cache='lab:cache:{product42}:value', lock='lab:cache:{product42}:lock';
const publish=`
if redis.call("GET",KEYS[1])~=ARGV[1] then return 0 end
redis.call("SET",KEYS[2],ARGV[2],"EX",ARGV[3])
redis.call("DEL",KEYS[1])
return 1`;
const release=`
if redis.call("GET",KEYS[1])==ARGV[1] then return redis.call("DEL",KEYS[1]) end
return 0`;
async function load() {
  await pool.query('UPDATE redis_lab.stats SET reads=reads+1 WHERE id=1');
  await pool.query('SELECT pg_sleep(0.08)');
  return (await pool.query('SELECT * FROM redis_lab.products WHERE id=42')).rows[0];
}
async function get(guarded) {
  const deadline=Date.now()+2500;
  while(Date.now()<deadline) {
    const hit=await command(r,'GET',cache);
    if(hit!==null) return JSON.parse(hit);
    if(!guarded) {
      const row=await load();
      await command(r,'SET',cache,JSON.stringify(row),'EX',30);
      return row;
    }
    const token=randomUUID();
    if(await command(r,'SET',lock,token,'NX','PX',2000)) {
      try {
        const recheck=await command(r,'GET',cache);
        if(recheck!==null) return JSON.parse(recheck);
        const row=await load();
        const ttl=30+Math.floor(Math.random()*10);
        await command(r,'EVAL',publish,2,lock,cache,token,JSON.stringify(row),ttl);
        return row;
      } finally { await command(r,'EVAL',release,1,lock,token); }
    }
    await sleep(20+Math.floor(Math.random()*20));
  }
  throw new Error('refresh budget exceeded; choose bounded fallback or shed load');
}
try {
  for(const guarded of [false,true]) {
    await command(r,'DEL',cache,lock);
    await pool.query('UPDATE redis_lab.stats SET reads=0 WHERE id=1');
    const values=await Promise.all(Array.from({length:20},()=>get(guarded)));
    assert(values.every(v=>v.id===42));
    const reads=(await pool.query('SELECT reads FROM redis_lab.stats WHERE id=1')).rows[0].reads;
    if(guarded) assert.equal(reads,1); else assert(reads>1);
    console.log({guarded,db_reads:reads,ttl:await command(r,'TTL',cache)});
  }
} finally { await Promise.all([r.close(),pool.end()]); }
JS
./run-node.sh /scripts/cache.mjs
```

**Đạt:** naive có nhiều DB reads, guarded có một trong phép thử này. Stats update và `pg_sleep` là instrumentation dành riêng lab, không đưa vào repository thật. Lease có timeout; nếu load vượt 2 giây, owner mất lease và không được publish cache dưới token cũ. Nó vẫn có thể trả snapshot đã load cho request hiện tại: phải biết data freshness contract của endpoint.

**Bài tập:** đổi `pg_sleep` thành 3 giây trong lab để quan sát timeout/lease loss. Không sửa bằng chờ vô hạn; chọn hard deadline, stale fallback nếu được phép hoặc trả lỗi có chủ đích. Khôi phục 0,08 giây trước bài tiếp.

### 8.4. Lab — invalidate sau DB commit vẫn có race

```bash {"name":"redis-18-stale-cache-race","excludeFromRunAll":"true"}
set -euo pipefail
cd "${HOME}/backend-labs/redis"
cat > scripts/stale.mjs <<'JS'
import assert from 'node:assert/strict';
import { createRequire } from 'node:module';
import { connect,command } from '/app/client.mjs';
const { Pool }=createRequire('/app/package.json')('pg');
const pool=new Pool({connectionString:process.env.DATABASE_URL,max:2});
const r=await connect();
const key='lab:cache:{product42}:value', version='lab:cache:{product42}:watermark';
const publish=`
local current=redis.call("GET",KEYS[2]) or "0"
local incoming=ARGV[1]
local function valid(v)
  return type(v)=="string" and #v<=19 and
    (v=="0" or string.match(v,"^[1-9][0-9]*$")~=nil)
end
if not valid(incoming) or not valid(current) then
  return redis.error_reply("BAD_VERSION")
end
local older=#incoming<#current or (#incoming==#current and incoming<current)
if older then return 0 end
redis.call("SET",KEYS[1],ARGV[2],"EX",30)
redis.call("SET",KEYS[2],ARGV[1])
return 1`;
const invalidate=`
local current=redis.call("GET",KEYS[2]) or "0"
local incoming=ARGV[1]
local function valid(v)
  return type(v)=="string" and #v<=19 and
    (v=="0" or string.match(v,"^[1-9][0-9]*$")~=nil)
end
if not valid(incoming) or not valid(current) then
  return redis.error_reply("BAD_VERSION")
end
local older=#incoming<#current or (#incoming==#current and incoming<current)
if not older then
  redis.call("SET",KEYS[2],ARGV[1])
  redis.call("DEL",KEYS[1])
end
return 1`;
try {
  await command(r,'DEL',key,version);
  await pool.query('UPDATE redis_lab.products SET price_minor=10000,version=1 WHERE id=42');
  const old=(await pool.query('SELECT * FROM redis_lab.products WHERE id=42')).rows[0];
  await pool.query('UPDATE redis_lab.products SET price_minor=20000,version=2 WHERE id=42');
  await command(r,'DEL',key);
  await command(r,'SET',key,JSON.stringify(old),'EX',30);
  assert.equal(JSON.parse(await command(r,'GET',key)).version,'1');
  console.log('REPRODUCED: reader loaded v1, writer committed v2+DEL, reader repopulated v1');
  await command(r,'EVAL',invalidate,2,key,version,2);
  assert.equal(await command(r,'EVAL',publish,2,key,version,old.version,JSON.stringify(old)),0);
  assert.equal(await command(r,'GET',key),null);
  const fresh=(await pool.query('SELECT * FROM redis_lab.products WHERE id=42')).rows[0];
  assert.equal(await command(r,'EVAL',publish,2,key,version,fresh.version,JSON.stringify(fresh)),1);
  assert.equal(JSON.parse(await command(r,'GET',key)).version,'2');
  console.log('PASS: watermark rejects stale publisher while watermark exists and invalidation arrived');
  const bigKey=key+':bigint-test', bigVersion=version+':bigint-test';
  await command(r,'DEL',bigKey,bigVersion);
  await command(r,'EVAL',invalidate,2,bigKey,bigVersion,'9007199254740993');
  assert.equal(await command(r,'EVAL',publish,2,bigKey,bigVersion,
    '9007199254740992','{"stale":true}'),0);
  assert.equal(await command(r,'GET',bigKey),null);
  await command(r,'DEL',bigKey,bigVersion);
  console.log('PASS: decimal-string version comparison also handles values above 2^53');

} finally { await Promise.all([r.close(),pool.end()]); }
JS
./run-node.sh /scripts/stale.mjs
```

Version PostgreSQL BIGINT được giữ dưới dạng chuỗi thập phân chuẩn; Lua so độ dài rồi thứ tự từ điển, không dùng tonumber vì số nguyên lớn hơn 2^53 có thể mất độ chính xác. Assertion với hai version liên tiếp trên ngưỡng đó kiểm tra reader cũ không được publish. Đây là version theo aggregate, không phải timestamp wall-clock.

**Giới hạn rất quan trọng:** watermark trong Redis có thể mất do eviction/restart/data loss; DB commit→Redis invalidate vẫn có gap khi Redis hỏng. Đây là kỹ thuật giảm stale writes có điều kiện, không biến hai hệ thống thành transaction. Cần outbox invalidation/retry, retention hợp lý, version trong DB, TTL và contract eventual consistency. Với invariant bắt buộc current, đọc/kiểm tra ở authoritative DB.

**Case double-delete:** xóa cache rồi delay rồi xóa lại có thể giảm một số cửa sổ race nhưng không chứng minh correctness cho mọi schedule. Nếu snapshot cũ bị pause lâu hơn delay, race quay lại. Hãy vẽ timeline và dùng version hoặc thay đổi contract, không coi một `sleep(500)` là bằng chứng.

### 8.5. FAULT — Redis unavailable, catalog fallback có giới hạn

```bash {"name":"redis-19-cache-outage-fallback","excludeFromRunAll":"true"}
set -euo pipefail
cd "${HOME}/backend-labs/redis"
cat > scripts/fallback.mjs <<'JS'
import assert from 'node:assert/strict';
import { createRequire } from 'node:module';
import { connect,command } from '/app/client.mjs';
const { Pool }=createRequire('/app/package.json')('pg');
const pool=new Pool({connectionString:process.env.DATABASE_URL,max:2,connectionTimeoutMillis:1000,
  statement_timeout:1000,query_timeout:1500});
let r;
try {
  let cached=null;
  try {
    r=await connect();
    cached=await command(r,'GET','lab:cache:{product42}:value');
  } catch(error) { console.log('Cache unavailable, bounded read-only DB fallback'); }
  const product=cached ? JSON.parse(cached) :
    (await pool.query({text:'SELECT * FROM redis_lab.products WHERE id=$1',values:[42]})).rows[0];
  assert.equal(product.id,42);
  console.log({source:cached?'cache':'db',product});
} finally {
  if(r?.isOpen) await r.close();
  await pool.end();
}
JS
trap 'docker compose -p be-redis start redis >/dev/null' EXIT
docker compose -p be-redis stop redis
./run-node.sh /scripts/fallback.mjs
docker compose -p be-redis start redis
trap - EXIT
```

Đây là **một request catalog** để học fallback. PostgreSQL có statement_timeout 1 giây và query_timeout 1,5 giây ngoài connect/pool timeout; đó là giới hạn từng giai đoạn, chưa phải một deadline end-to-end cho mọi kiểu network stall Redis. Production cần giới hạn tổng số request fallback, circuit breaker và admission; pool nhỏ chỉ giới hạn connection, không tự giới hạn số Promise đang chờ. Không copy fallback cho authorization/OTP bằng cách trả “valid”.

Nguồn: [cache-aside](https://redis.io/docs/latest/develop/use/), [node-redis production connections](https://redis.io/docs/latest/develop/clients/nodejs/produsage/). Đọc API theo version đã pin; docs latest có thể thuộc node-redis 6 trong tương lai, khác baseline 5 của notebook.


---

## 9. Messaging primitives — List, Pub/Sub và Streams

### 9.1. List queue: atomic move chưa phải queue framework

RPOP lấy job ra rồi worker chết → job biến mất khỏi ready list. LMOVE/BLMOVE chuyển job sang processing list trong cùng thao tác giúp giữ dấu vết, nhưng bạn còn cần worker ownership, visibility timeout, retry count, quarantine và recovery. Nếu bạn tự xây hết các phần ấy, hãy cân nhắc BullMQ hoặc broker trước.

```bash {"name":"redis-20-list-queue-recovery","excludeFromRunAll":"true"}
set -euo pipefail
cd "${HOME}/backend-labs/redis"
./rcli.sh DEL 'lab:list:{jobs}:ready' 'lab:list:{jobs}:processing'
./rcli.sh LPUSH 'lab:list:{jobs}:ready' job-1 job-2
job=$(./rcli.sh LMOVE 'lab:list:{jobs}:ready' 'lab:list:{jobs}:processing' RIGHT LEFT)
test "$job" = job-1
test "$(./rcli.sh LLEN 'lab:list:{jobs}:processing')" = 1
echo "Giả lập worker chết: job còn ở processing, không ACK."
./rcli.sh LMOVE 'lab:list:{jobs}:processing' 'lab:list:{jobs}:ready' LEFT RIGHT
test "$(./rcli.sh LLEN 'lab:list:{jobs}:processing')" = 0
test "$(./rcli.sh LLEN 'lab:list:{jobs}:ready')" = 2
```

Recovery ở đây **thủ công và biết worker đã chết**. Trong production không được kéo mọi processing job về ready khi worker vẫn đang làm. Cần lease/ownership và durable idempotent consumer. Job ID phải unique; payload giống nhau không đồng nghĩa cùng business operation.

### 9.2. Lab — Pub/Sub mất message lúc không có subscriber

```bash {"name":"redis-21-pubsub-gap","excludeFromRunAll":"true"}
set -euo pipefail
cd "${HOME}/backend-labs/redis"
cat > scripts/pubsub.mjs <<'JS'
import assert from 'node:assert/strict';
import { randomUUID } from 'node:crypto';
import { connect,sleep } from '/app/client.mjs';
const pub=await connect(), sub=await connect();
const channel='lab:pubsub:'+randomUUID(), seen=[];
try {
  assert.equal(await pub.publish(channel,'before-subscribe'),0);
  await sub.subscribe(channel,message=>seen.push(message));
  assert.equal(await pub.publish(channel,'after-subscribe'),1);
  const deadline=Date.now()+2000;
  while(seen.length<1 && Date.now()<deadline) await sleep(10);
  assert.deepEqual(seen,['after-subscribe']);
  console.log({seen,missing:'before-subscribe was never retained'});
  await sub.unsubscribe(channel);
} finally { await Promise.all([pub.close(),sub.close()]); }
JS
./run-node.sh /scripts/pubsub.mjs
```

Pub/Sub hữu ích cho hint/invalidation/notification khi bạn chấp nhận mất và có nguồn khôi phục. Nếu cache invalidation chỉ qua Pub/Sub, subscriber offline có thể giữ cache stale; phải có TTL/version/rebuild strategy. Đừng dùng Pub/Sub làm bằng chứng bền vững rằng nghiệp vụ đã được thực hiện.

### 9.3. Lab — Streams pending, reclaim và ACK

`XREADGROUP ... >` lấy message chưa từng giao cho group; nó không tự lấy toàn bộ pending cũ. `XACK` bỏ entry khỏi PEL của group, không tự xóa stream entry. Retention cần thiết kế riêng; nhiều group có tiến độ khác nhau.

```bash {"name":"redis-22-stream-pending-reclaim","excludeFromRunAll":"true"}
set -euo pipefail
cd "${HOME}/backend-labs/redis"
cat > scripts/stream.mjs <<'JS'
import assert from 'node:assert/strict';
import { randomUUID } from 'node:crypto';
import { connect,command,sleep } from '/app/client.mjs';
const r=await connect();
const stream='lab:stream:'+randomUUID(), group='workers';
try {
  await command(r,'XGROUP','CREATE',stream,group,'0','MKSTREAM');
  const id=await command(r,'XADD',stream,'*','operationId','op-1','amount_minor','500');
  const batch=await command(r,'XREADGROUP','GROUP',group,'worker-a','COUNT',1,'STREAMS',stream,'>');
  assert.equal(batch[0][1][0][0],id);
  assert.equal((await command(r,'XPENDING',stream,group))[0],1);
  console.log('worker-a died before ACK; pending entry retained');
  await sleep(1100);
  let cursor='0-0', claimed=[];
  do {
    const result=await command(r,'XAUTOCLAIM',stream,group,'worker-b',1000,cursor,'COUNT',10);
    cursor=result[0];
    claimed.push(...result[1]);
  } while(cursor!=='0-0');
  assert.equal(claimed[0][0],id);
  console.log('Claimed:',claimed);
  console.log('Demo boundary: production must commit idempotent business effect BEFORE XACK');
  assert.equal(await command(r,'XACK',stream,group,id),1);
  assert.equal((await command(r,'XPENDING',stream,group))[0],0);
  assert.equal(await command(r,'XLEN',stream),1);
  await command(r,'DEL',stream);
} finally { await r.close(); }
JS
./run-node.sh /scripts/stream.mjs
```

**Đạt:** pending 1→claim→0, XLEN vẫn 1. Bài chỉ kiểm tra lifecycle stream; consumer SQL inbox/outbox thật nằm ở notebook 05. `min-idle-time` nhỏ hơn thời gian xử lý bình thường có thể tạo duplicate workers. Claim không giết worker cũ; business effect vẫn phải idempotent.

### 9.4. Lab — trim có thể xóa payload còn pending

```bash {"name":"redis-23-stream-retention-hazard","excludeFromRunAll":"true"}
set -euo pipefail
cd "${HOME}/backend-labs/redis"
cat > scripts/trim.mjs <<'JS'
import assert from 'node:assert/strict';
import { randomUUID } from 'node:crypto';
import { connect,command } from '/app/client.mjs';
const r=await connect(), key='lab:trim:'+randomUUID();
try {
  await command(r,'XGROUP','CREATE',key,'g','0','MKSTREAM');
  const old=await command(r,'XADD',key,'*','operationId','old');
  await command(r,'XREADGROUP','GROUP','g','w','COUNT',1,'STREAMS',key,'>');
  await command(r,'XADD',key,'*','operationId','new');
  await command(r,'XTRIM',key,'MAXLEN',1);
  assert.deepEqual(await command(r,'XRANGE',key,old,old),[]);
  console.log('Old payload removed even though application had not ACKed it');
  console.log('Pending before cleanup:',await command(r,'XPENDING',key,'g'));
  console.log('XAUTOCLAIM cleanup:',await command(r,'XAUTOCLAIM',key,'g','rescuer',0,'0-0'));
  await command(r,'DEL',key);
} finally { await r.close(); }
JS
./run-node.sh /scripts/trim.mjs
```

Trong baseline 7.4, không giả định MAXLEN tự bảo toàn mọi unacked event. Version mới có thêm retention/ack options thì phải đọc đúng version. Retention cần lớn hơn recovery window/lag bạn cam kết, hoặc dùng nguồn replay bền vững khác.

Nguồn: [Pub/Sub](https://redis.io/docs/latest/develop/pubsub/), [Streams](https://redis.io/docs/latest/develop/data-types/streams/), [XACK](https://redis.io/docs/latest/commands/xack/), [XAUTOCLAIM](https://redis.io/docs/latest/commands/xautoclaim/).

---

## 10. Persistence — restart không đồng nghĩa dữ liệu chắc chắn còn

RDB chụp snapshot; AOF lưu các thay đổi theo chính sách fsync và cơ chế rewrite. AOF hiện đại có thể là nhiều file + manifest, không luôn một `appendonly.aof`. Backup AOF phải nhất quán cả bộ và theo version, không copy một file đang ghi rồi gọi đó là backup.

`appendfsync everysec` thường nhắm cửa sổ mất dữ liệu khoảng giây khi crash, nhưng không phải cam kết mọi loại lỗi luôn mất tối đa đúng 1 giây: storage, OS, fsync stalls, config và failure model quan trọng. Async replication cũng không thay backup. Snapshot chứa lỗi logic/xóa dữ liệu nếu lỗi ấy đã xảy ra.

`BGSAVE` có child process, nhưng fork pause, CoW memory và disk I/O vẫn có tác động. `maxmemory` không phải hard limit RSS toàn process; cần headroom cho replication/AOF buffers, allocator và fork. Container limit đặt sát `maxmemory` có thể OOM khi persistence chạy.

### 10.1. Lab — restart với volume

```bash {"name":"redis-24-restart-persistence","excludeFromRunAll":"true"}
set -euo pipefail
cd "${HOME}/backend-labs/redis"
marker=$(python3 -c 'import uuid;print(uuid.uuid4())')
printf '%s\n' "$marker" > evidence/persistence-marker.txt
./rcli.sh SET lab:persist:marker "$marker"
docker compose -p be-redis restart redis
ready=0
for i in $(seq 1 60); do
  if [ "$(./rcli.sh PING 2>/dev/null || true)" = PONG ]; then ready=1; break; fi
  sleep .5
done
test "$ready" = 1
test "$(./rcli.sh GET lab:persist:marker)" = "$(cat evidence/persistence-marker.txt)"
./rcli.sh INFO persistence | tee evidence/persistence-info.txt
```

**Giới hạn:** restart graceful khác kill -9/power loss/đĩa mất. Test này chứng minh volume/config/load bình thường, chưa chứng minh RPO khi host chết. Khi cần crash experiment, dùng dataset riêng, ghi sequence IDs + acknowledged timestamps, kill container trong lab rồi so thiếu record; đừng mô tả nó tương đương mọi loại mất điện vì host page cache vẫn còn.

### 10.2. Lab — snapshot rồi restore sang instance tách biệt

```bash {"name":"redis-25-backup-restore-rehearsal","excludeFromRunAll":"true"}
set -euo pipefail
cd "${HOME}/backend-labs/redis"
backup_marker="$(python3 -c 'import uuid;print(uuid.uuid4())')"
printf '%s\n' "$backup_marker" > evidence/backup-marker.txt
./rcli.sh SET lab:backup:marker "$backup_marker"
./rcli.sh BGSAVE
done_save=0
for i in $(seq 1 60); do
  ./rcli.sh INFO persistence > evidence/backup-state.txt
  if python3 - <<'PY'
from pathlib import Path
s=Path('evidence/backup-state.txt').read_text()
assert 'rdb_bgsave_in_progress:0' in s
assert 'rdb_last_bgsave_status:ok' in s
PY
  then done_save=1; break; fi
  sleep .5
done
test "$done_save" = 1
stamp=$(python3 -c 'import uuid;print(uuid.uuid4().hex[:12])')
mkdir "backup/restore-$stamp"
docker compose -p be-redis cp redis:/data/dump.rdb "backup/restore-$stamp/dump.rdb"
restore_name="be-redis-restore-$stamp"
trap 'docker rm -f "$restore_name" >/dev/null 2>&1 || true' EXIT
docker run -d --rm --name "$restore_name" --label course=be-redis \
  --network none --memory 192m \
  -v "$PWD/backup/restore-$stamp:/data" redis:7.4-alpine \
  redis-server --appendonly no --save "" --protected-mode yes >/dev/null
ready=0
for i in $(seq 1 60); do
  if [ "$(docker exec "$restore_name" redis-cli --raw GET lab:backup:marker 2>/dev/null || true)" = "$backup_marker" ]; then
    ready=1; break
  fi
  sleep .5
done
test "$ready" = 1
docker exec "$restore_name" redis-cli INFO keyspace
docker rm -f "$restore_name" >/dev/null
trap - EXIT
printf 'Restore evidence: backup/restore-%s\n' "$stamp"
```

Nếu BGSAVE báo đang có background save, đợi tác vụ hiện tại xong rồi chạy lại cell. Nếu RDB corrupt hoặc version không hỗ trợ, restore phải fail; không sửa bằng bỏ kiểm tra marker.

**Production:** backup ngoài cùng VPS, encryption/access/retention phù hợp, kiểm tra checksum, đo thời gian restore thật và kiểm tra business invariants. TTL trong snapshot có thể đã trôi qua lúc restore; key biến mất sau restore chưa chắc là file hỏng. Với queues, restore có thể đưa lại jobs đã được xử lý sau thời điểm backup → consumer idempotency vẫn cần.

Nguồn: [Redis persistence](https://redis.io/docs/latest/operate/oss_and_stack/management/persistence/), [BGSAVE](https://redis.io/docs/latest/commands/bgsave/).

---

## 11. FAULT — eviction khác Redis OOM và khác kernel OOM

- **Eviction:** Redis chủ động bỏ keys theo policy để nhường chỗ.
- **Redis OOM reply:** command bị từ chối do maxmemory/policy; process có thể vẫn sống và GET vẫn được.
- **Kernel/container OOM:** process có thể bị kill khi RSS vượt cgroup/host budget.
- **Disk full:** persistence/logging có thể fail dù memory còn; xem error và `INFO persistence`.

Không dùng cả ba từ OOM như một hiện tượng. `noeviction` không tự làm dữ liệu durable; chỉ tránh việc eviction theo policy, khi đầy vẫn cần producer backpressure.

### Lab — hai instance riêng, data khoảng 10MB

```bash {"name":"redis-26-memory-policies","excludeFromRunAll":"true"}
set -euo pipefail
cd "${HOME}/backend-labs/redis"
cat > scripts/memory.mjs <<'JS'
import assert from 'node:assert/strict';
import { connect,command,sleep } from '/app/client.mjs';
async function scenario(url,evict) {
  let r;
  for(let i=0;i<30;i++) {
    try { r=await connect(url); break; }
    catch { await sleep(100); }
  }
  assert(r,'Redis sandbox did not become ready');
  try {
    await command(r,'SET','probe','alive');
    let oom=0;
    for(let i=0;i<300;i++) {
      try { await command(r,'SET',`payload:${i}`,'x'.repeat(32768)); }
      catch(error) { if(/OOM/.test(error.message)) { oom++; break; } else throw error; }
    }
    const info=await command(r,'INFO','stats');
    const evicted=Number(info.match(/evicted_keys:(\d+)/)[1]);
    if(evict) { assert.equal(oom,0); assert(evicted>0); }
    else { assert(oom>0); assert.equal(evicted,0); assert.equal(await command(r,'GET','probe'),'alive'); }
    console.log({url,evict,oom,evicted,keys:await command(r,'DBSIZE')});
  } finally { await r.close(); }
}
await scenario(process.argv[2],true);
await scenario(process.argv[3],false);
JS
suffix=$(python3 -c 'import uuid;print(uuid.uuid4().hex[:8])')
evict="be-redis-evict-$suffix"
reject="be-redis-reject-$suffix"
trap 'docker rm -f "$evict" "$reject" >/dev/null 2>&1 || true' EXIT
docker run -d --rm --name "$evict" --label course=be-redis \
  --network be-redis_default --memory 64m redis:7.4-alpine \
  redis-server --maxmemory 4mb --maxmemory-policy allkeys-lru \
  --save "" --appendonly no --protected-mode no >/dev/null
docker run -d --rm --name "$reject" --label course=be-redis \
  --network be-redis_default --memory 64m redis:7.4-alpine \
  redis-server --maxmemory 4mb --maxmemory-policy noeviction \
  --save "" --appendonly no --protected-mode no >/dev/null
./run-node.sh /scripts/memory.mjs "redis://$evict:6379" "redis://$reject:6379"
docker rm -f "$evict" "$reject" >/dev/null
trap - EXIT
```

Hai instance fault không có port host, không persistence, tên ngẫu nhiên có label riêng. `protected-mode no` chỉ phục vụ network riêng của lab; không copy vào public Redis. Nếu bạn đổi 4MB quá thấp so overhead startup, OOM ngay từ probe sẽ làm bài fail có lý do; đo used_memory trước khi điều chỉnh.

### Sizing dựa vào bằng chứng

Đo key count × distribution payload + object/index overhead, độ nóng, output buffers, fragmentation, replication backlog, AOF rewrite/CoW headroom, metrics agent và OS. “VPS 2GB → Redis maxmemory 1.8GB” không là quy tắc đúng. Dataset vừa RAM vẫn có thể thất bại khi full resync tạo peak. Chạy load bằng shape gần thật: nhiều key nhỏ khác một key 50MB.

Hot key không tự chia nhỏ khi thêm Cluster shards, vì một key ở một slot. Cần thay data model, local caching có invalidation hoặc read replicas khi consistency cho phép. Big key làm transfer/delete/replication nặng; `UNLINK` tách reclaim khỏi main thread trong các trường hợp phù hợp nhưng vẫn phải theo dõi memory/background work.

Nguồn: [eviction policies](https://redis.io/docs/latest/develop/reference/eviction/), [memory optimization](https://redis.io/docs/latest/operate/oss_and_stack/management/optimization/memory-optimization/), [UNLINK](https://redis.io/docs/latest/commands/unlink/).

---

## 12. ACL, client behavior và observability

### 12.1. Lab — ACL quyền nhỏ, quote đúng password

Ký tự `>` trong ACL **phải được quote** khi gõ trong shell. Không quote sẽ redirect output ra file thay vì đặt password. Quyền key pattern và command đều phải đạt; authentication thành công chưa đồng nghĩa mọi thao tác được phép.

```bash {"name":"redis-27-acl-least-privilege","excludeFromRunAll":"true"}
set -euo pipefail
cd "${HOME}/backend-labs/redis"
./rcli.sh ACL SETUSER course_app reset on '>app-demo-password' \
  '~cache:catalog:*' '+ping' '+get' '+set' '+del'
docker compose -p be-redis exec -T -e REDISCLI_AUTH=app-demo-password redis \
  redis-cli --user course_app --no-auth-warning -e SET cache:catalog:42 demo
docker compose -p be-redis exec -T -e REDISCLI_AUTH=app-demo-password redis \
  redis-cli --user course_app --no-auth-warning -e GET cache:catalog:42
if docker compose -p be-redis exec -T -e REDISCLI_AUTH=app-demo-password redis \
  redis-cli --user course_app --no-auth-warning -e GET private:wallet:42; then
  echo "LỖI: ACL cho đọc ngoài namespace"; exit 1
fi
if ./rcli.sh ACL DRYRUN course_app FLUSHALL; then
  echo "LỖI: app có quyền FLUSHALL"; exit 1
fi
./rcli.sh ACL LOG 5
```

ACL user runtime trong bài chưa được lưu vào ACL file; restart có thể làm mất. Production quản lý `aclfile`/configuration lifecycle và rollout/rotation, không dựa vào “tôi đã gõ SETUSER tháng trước”. Không tắt default admin trước khi có admin credential khác đã test. Không đưa secret vào screenshot hoặc commit notebook output.

### 12.2. Lab — slowlog đo server execution, không phải toàn bộ request

```bash {"name":"redis-28-observability","excludeFromRunAll":"true"}
set -euo pipefail
cd "${HOME}/backend-labs/redis"
old=$(./rcli.sh CONFIG GET slowlog-log-slower-than | tail -n 1)
trap './rcli.sh CONFIG SET slowlog-log-slower-than "$old" >/dev/null' EXIT
./rcli.sh CONFIG SET slowlog-log-slower-than 0
./rcli.sh SET lab:observe:sample demo EX 60
./rcli.sh GET lab:observe:sample
./rcli.sh SLOWLOG GET 10 > evidence/slowlog.txt
./rcli.sh CONFIG SET slowlog-log-slower-than "$old"
trap - EXIT
./rcli.sh INFO memory > evidence/memory.txt
./rcli.sh INFO stats > evidence/stats.txt
./rcli.sh INFO clients > evidence/clients.txt
./rcli.sh INFO persistence > evidence/persistence.txt
./rcli.sh LATENCY LATEST
./rcli.sh MEMORY USAGE lab:observe:sample
```

Threshold 0 chỉ trong lab ngắn để tạo evidence; để nguyên production làm noisy và có thể lưu arguments nhạy cảm. SLOWLOG GET là entries gần đây vượt threshold, không phải danh sách sắp giảm dần “10 lệnh chậm nhất”. App timeout mà slowlog rỗng có thể do network, client queue, pool hoặc event-loop lag.

**Bộ chỉ số nên có:**

| Nhóm | Quan sát | Ý nghĩa vận hành |
|---|---|---|
| Latency | client p95/p99, server latency events | phân biệt app/network/server |
| Memory | used_memory/RSS, maxmemory, fragmentation | budget và peak, không chỉ dataset |
| Cache | hits/misses theo endpoint, evictions, TTL distribution | tỷ lệ global có thể che endpoint lỗi |
| Clients | connections, blocked clients, output buffers | leaks, blocking reads, slow consumers |
| Persistence | last save/rewrite status, pending fsync | khả năng durable/restore |
| Replication | link state, offset/lag, full resync | read staleness, bandwidth |
| Streams/jobs | oldest pending age, lag, failures | job age quan trọng hơn count đơn thuần |

`MONITOR` có overhead và có thể lộ arguments; không bật vô hạn trong production. `--bigkeys`/SCAN vẫn tạo tải, throttle và lấy mẫu khi dataset lớn. Metrics labels không dùng key/user ID tùy ý vì cardinality có thể làm hệ monitoring chết trước Redis.

### 12.3. Connection, retries và shutdown

Tái sử dụng client theo process/lifecycle; không tạo TCP connection mới cho mọi GET. Blocking commands (`BLMOVE`, `XREADGROUP BLOCK`) cần connection riêng để tránh head-of-line blocking trên client dùng chung. Subscriber cũng nên riêng. Offline queue có thể che outage rồi gửi dồn khi reconnect; xem safety của non-idempotent INCR/LPUSH khi response mất.

App có request deadline và retry budget, exponential backoff + jitter và giới hạn queue. Một `Promise.race` timeout không tự hủy lệnh đã gửi tới Redis; result có thể đến sau, side effect vẫn xảy ra. Không retry mutation mù nếu chưa biết lần trước thành công hay không.

Nguồn: [ACL](https://redis.io/docs/latest/operate/oss_and_stack/management/security/acl/), [SLOWLOG GET](https://redis.io/docs/latest/commands/slowlog-get/), [node-redis production usage](https://redis.io/docs/latest/develop/clients/nodejs/produsage/).


---

## 13. Replication và Sentinel — failover có khoảng gián đoạn

Replication Redis thông thường là bất đồng bộ. Primary đã trả OK nhưng replica chưa nhận hết mà primary mất có thể mất acknowledged write. `WAIT` giúp chờ replication acknowledgements cho writes **trước đó trên cùng connection**, nhưng không biến toàn hệ thống thành strongly consistent store hay thay durable fsync policy.

Sentinel giám sát và hỗ trợ bầu/promote primary khi không dùng Cluster. **Quorum** để xác nhận sự cố và **majority** để authorize failover là khái niệm liên quan nhưng không đồng nhất. Ba Sentinel trên cùng một VPS cho phép học election, không tạo ba failure domain.

### 13.1. Lab — primary, replica và ba Sentinel

Khuyến nghị tắt core DB/Redis sau khi lưu evidence để giảm RAM. Lab HA dùng project `be-redis-ha`, không public ports. Cần khoảng 1GiB RAM trống; với VPS nhỏ hãy làm trên laptop.

```bash {"name":"redis-29-sentinel-setup","excludeFromRunAll":"true"}
set -euo pipefail
cd "${HOME}/backend-labs/redis"
if [ -e compose.ha.yaml ]; then
  echo "HA lab đã có; không ghi đè Sentinel state. Dùng file hiện có hoặc archive lab trước."; exit 1
fi
mkdir -p ha/s1 ha/s2 ha/s3
for id in s1 s2 s3; do
  cat > "ha/$id/sentinel.conf" <<'CONF'
port 26379
bind 0.0.0.0
protected-mode no
dir /data
sentinel resolve-hostnames yes
sentinel announce-hostnames yes
sentinel monitor labmaster primary 6379 2
sentinel auth-pass labmaster ha-course-password
sentinel down-after-milliseconds labmaster 1500
sentinel failover-timeout labmaster 10000
sentinel parallel-syncs labmaster 1
CONF
done
cat > compose.ha.yaml <<'YAML'
name: be-redis-ha
x-sentinel: &sentinel
  image: redis:7.4-alpine
  command: ["redis-server", "/data/sentinel.conf", "--sentinel"]
  mem_limit: 96m
  depends_on: [primary, replica]
services:
  primary:
    image: redis:7.4-alpine
    command:
      - redis-server
      - --requirepass
      - ha-course-password
      - --masterauth
      - ha-course-password
      - --appendonly
      - "yes"
    volumes: ["ha_primary:/data"]
    mem_limit: 192m
  replica:
    image: redis:7.4-alpine
    command:
      - redis-server
      - --requirepass
      - ha-course-password
      - --masterauth
      - ha-course-password
      - --replicaof
      - primary
      - "6379"
      - --appendonly
      - "yes"
    volumes: ["ha_replica:/data"]
    mem_limit: 192m
  s1:
    <<: *sentinel
    volumes: ["./ha/s1:/data"]
  s2:
    <<: *sentinel
    volumes: ["./ha/s2:/data"]
  s3:
    <<: *sentinel
    volumes: ["./ha/s3:/data"]
volumes:
  ha_primary:
  ha_replica:
YAML
docker compose -p be-redis-ha -f compose.ha.yaml up -d
ready=0
for i in $(seq 1 60); do
  if docker compose -p be-redis-ha -f compose.ha.yaml exec -T s1 \
     redis-cli -p 26379 --raw -e SENTINEL CKQUORUM labmaster > evidence/sentinel-quorum.txt 2>/dev/null &&
     python3 -c 'from pathlib import Path; assert Path("evidence/sentinel-quorum.txt").read_text().startswith("OK")'; then
    ready=1; break
  fi
  sleep .5
done
test "$ready" = 1
docker compose -p be-redis-ha -f compose.ha.yaml exec -T s1 \
  redis-cli -p 26379 SENTINEL get-master-addr-by-name labmaster
```

Sentinel phải ghi lại config/state nên mount `/data` writable. Không đặt sentinel.conf read-only rồi bỏ qua lỗi rewrite. Auth giữa clients/data nodes khác auth bảo vệ Sentinel control plane; lab không publish Sentinel và chỉ để học, production phải kiểm soát cả hai.

### 13.2. Lab — WAIT đúng connection, rồi promote

```bash {"name":"redis-30-sentinel-failover","excludeFromRunAll":"true"}
set -euo pipefail
cd "${HOME}/backend-labs/redis"
synced=0
for i in $(seq 1 60); do
  docker compose -p be-redis-ha -f compose.ha.yaml exec -T \
    -e REDISCLI_AUTH=ha-course-password replica \
    redis-cli --no-auth-warning INFO replication > evidence/replica-info.txt
  if python3 -c 'from pathlib import Path; assert "master_link_status:up" in Path("evidence/replica-info.txt").read_text()'; then
    synced=1; break
  fi
  sleep .5
done
test "$synced" = 1
printf 'SET lab:ha:marker before-failover\nWAIT 1 2000\n' |
  docker compose -p be-redis-ha -f compose.ha.yaml exec -T \
    -e REDISCLI_AUTH=ha-course-password primary redis-cli --no-auth-warning --raw -e \
    > evidence/wait-reply.txt
test "$(tail -n 1 evidence/wait-reply.txt)" = 1
trap 'docker compose -p be-redis-ha -f compose.ha.yaml start primary >/dev/null' EXIT
docker compose -p be-redis-ha -f compose.ha.yaml stop primary
promoted=0
for i in $(seq 1 90); do
  docker compose -p be-redis-ha -f compose.ha.yaml exec -T \
    -e REDISCLI_AUTH=ha-course-password replica \
    redis-cli --no-auth-warning INFO replication > evidence/promoted-info.txt
  if python3 -c 'from pathlib import Path; assert "role:master" in Path("evidence/promoted-info.txt").read_text()'; then
    promoted=1; break
  fi
  sleep .5
done
test "$promoted" = 1
docker compose -p be-redis-ha -f compose.ha.yaml exec -T s1 \
  redis-cli -p 26379 SENTINEL get-master-addr-by-name labmaster
test "$(docker compose -p be-redis-ha -f compose.ha.yaml exec -T \
 -e REDISCLI_AUTH=ha-course-password replica redis-cli --no-auth-warning --raw GET lab:ha:marker)" = before-failover
docker compose -p be-redis-ha -f compose.ha.yaml start primary
trap - EXIT
```

**Đạt:** replica trở thành master, Sentinel trả địa chỉ mới, marker còn trong kịch bản đã WAIT này. Đo thời gian failover từ logs/timestamps; không ghi “luôn vài giây” cho mọi network partition. Kiểm tra sau đó node `primary` cũ được reconfigure thành replica. Client thật cần Sentinel-aware discovery/reconnect; hardcode hostname `primary` không tự đổi nghĩa sau promote.

**Các câu hỏi nâng cao:** Sentinel mất majority thì sao? Primary bị network partition nhưng vẫn nhận writes từ một nhóm client thì sao? `min-replicas-to-write`/lag policy đổi availability thế nào? Client đã nhận timeout có được retry INCR không? Thử từng fault trong sandbox và ghi rõ giả thuyết trước, không cùng lúc kéo nhiều link rồi đoán.

```bash {"name":"redis-31-stop-sentinel-lab","excludeFromRunAll":"true"}
set -euo pipefail
cd "${HOME}/backend-labs/redis"
docker compose -p be-redis-ha -f compose.ha.yaml down
```

Volume/config giữ lại topology đã biến đổi; muốn làm lại từ đầu cần archive state rồi khởi tạo namespace mới có chủ đích, không xóa data để “hết lỗi” mà không ghi nhận vì sao.

Nguồn: [replication](https://redis.io/docs/latest/operate/oss_and_stack/management/replication/), [WAIT](https://redis.io/docs/latest/commands/wait/), [Sentinel](https://redis.io/docs/latest/operate/oss_and_stack/management/sentinel/).

---

## 14. Cluster — slot, MOVED, CROSSSLOT và hot key

Cluster chia 16.384 slots giữa nodes. Multi-key command/Lua thường yêu cầu **cùng slot**, không chỉ tình cờ cùng máy. `{hash-tag}` quyết định phần key dùng để băm; chọn theo aggregate để colocate những keys cần atomic cùng nhau. Over-colocation tạo hot shard.

Sentinel ưu tiên failover cho một dataset primary-replica; Cluster thêm sharding và client redirection/topology. Không chọn theo ngưỡng cứng “32GB dùng Sentinel, 64GB dùng Cluster”; xem working set, throughput, hot keys, ops và consistency.

### Lab — ba primary, chỉ để học sharding

Không có replicas nên **không HA**. Không publish ports, client chạy trong Docker network để theo redirect tới node addresses. Tổng RAM limit 3×192MiB.

```bash {"name":"redis-32-cluster-setup","excludeFromRunAll":"true"}
set -euo pipefail
cd "${HOME}/backend-labs/redis"
if [ -e compose.cluster.yaml ]; then
  echo "Cluster file đã có; dùng cluster hiện tại, không chạy create lại lên state cũ."; exit 1
fi
cat > compose.cluster.yaml <<'YAML'
name: be-redis-cluster
x-node: &node
  image: redis:7.4-alpine
  command:
    - redis-server
    - --cluster-enabled
    - "yes"
    - --cluster-config-file
    - nodes.conf
    - --cluster-node-timeout
    - "5000"
    - --appendonly
    - "yes"
    - --protected-mode
    - "no"
  mem_limit: 192m
services:
  c1:
    <<: *node
    volumes: ["c1_data:/data"]
  c2:
    <<: *node
    volumes: ["c2_data:/data"]
  c3:
    <<: *node
    volumes: ["c3_data:/data"]
volumes:
  c1_data:
  c2_data:
  c3_data:
YAML
docker compose -p be-redis-cluster -f compose.cluster.yaml up -d
for node in c1 c2 c3; do
  ready=0
  for i in $(seq 1 40); do
    if [ "$(docker compose -p be-redis-cluster -f compose.cluster.yaml exec -T "$node" redis-cli --raw PING 2>/dev/null || true)" = PONG ]; then
      ready=1; break
    fi
    sleep .25
  done
  test "$ready" = 1
done
docker compose -p be-redis-cluster -f compose.cluster.yaml exec -T c1 \
  redis-cli --cluster create c1:6379 c2:6379 c3:6379 --cluster-replicas 0 --cluster-yes
ok=0
for i in $(seq 1 40); do
  docker compose -p be-redis-cluster -f compose.cluster.yaml exec -T c1 \
    redis-cli CLUSTER INFO > evidence/cluster-info.txt
  if python3 -c 'from pathlib import Path; assert "cluster_state:ok" in Path("evidence/cluster-info.txt").read_text()'; then
    ok=1; break
  fi
  sleep .25
done
test "$ok" = 1
```

```bash {"name":"redis-33-cluster-hash-tags","excludeFromRunAll":"true"}
set -euo pipefail
cd "${HOME}/backend-labs/redis"
dc=(docker compose -p be-redis-cluster -f compose.cluster.yaml)
slot1=$("${dc[@]}" exec -T c1 redis-cli --raw CLUSTER KEYSLOT 'cart:{user101}')
slot2=$("${dc[@]}" exec -T c1 redis-cli --raw CLUSTER KEYSLOT 'orders:{user101}')
test "$slot1" = "$slot2"
"${dc[@]}" exec -T c1 redis-cli -c -e MSET 'cart:{user101}' cart-value 'orders:{user101}' order-value
if "${dc[@]}" exec -T c1 redis-cli -c -e MSET 'cart:{user101}' a 'cart:{user202}' b; then
  echo "LỖI: mong đợi CROSSSLOT"; exit 1
fi
"${dc[@]}" exec -T c1 redis-cli CLUSTER NODES
```

**Đạt:** cùng hash tag → cùng slot; MSET across slots lỗi. Chạy `CLUSTER KEYSLOT` trên core standalone cluster-disabled có thể lỗi: không được giả vờ đó là bài Cluster đã chạy thành công. `-c` giúp redis-cli theo MOVED cho lab; client Node production cần cluster client và topology config đúng.

**Case VPS/NAT:** node announce address client không tới được → initial connect thành công nhưng request redirect thất bại. Kết nối một port bằng SSH tunnel không giải quyết mọi node addresses. Dùng private network/VPN/bastion cùng topology thích hợp; không mở cả cluster bus ra Internet để “hết timeout”.

```bash {"name":"redis-34-stop-cluster","excludeFromRunAll":"true"}
set -euo pipefail
cd "${HOME}/backend-labs/redis"
docker compose -p be-redis-cluster -f compose.cluster.yaml down
```

Nguồn: [Cluster specification](https://redis.io/docs/latest/operate/oss_and_stack/reference/cluster-spec/), [Cluster tutorial](https://redis.io/docs/latest/operate/oss_and_stack/management/scaling/).

---

## 15. Redis 8 — JSON, Search, Bloom và vector theo capability

Đây là phần mở rộng sau khi hiểu correctness và vận hành. Bạn không cần vector search để làm cache tốt. Không trộn server version, image Redis Stack, Redis Insight và module availability. Redis Insight là công cụ quản lý riêng; GUI đẹp không thay version/capability probe.

### Lab — tạo index và thực sự query

Instance tạm, network none, không public port, chạy commands bằng docker exec. Baseline dùng `redis:8.2`; không gọi tag này là newest. Giữ default command của image để không vô tình bỏ cấu hình tích hợp.

```bash {"name":"redis-35-modern-data-types","excludeFromRunAll":"true"}
set -euo pipefail
cd "${HOME}/backend-labs/redis"
name="be-redis-modern-$(python3 -c 'import uuid;print(uuid.uuid4().hex[:8])')"
trap 'docker rm -f "$name" >/dev/null 2>&1 || true' EXIT
docker run -d --rm --name "$name" --label course=be-redis \
  --network none --memory 256m redis:8.2 >/dev/null
ready=0
for i in $(seq 1 50); do
  if [ "$(docker exec "$name" redis-cli --raw PING 2>/dev/null || true)" = PONG ]; then ready=1; break; fi
  sleep .2
done
test "$ready" = 1
docker exec "$name" redis-cli INFO server > evidence/redis8-version.txt
docker exec "$name" redis-cli --json COMMAND INFO JSON.SET FT.CREATE BF.RESERVE VADD VSIM \
  > evidence/redis8-capabilities.json
python3 - <<'PY'
import json
from pathlib import Path
caps=json.loads(Path('evidence/redis8-capabilities.json').read_text())
assert len(caps)==5 and all(caps), 'Image lacks a required capability. Inspect version/image docs, do not silently skip.'
PY
docker exec "$name" redis-cli -e FT.CREATE lab:idx:products ON JSON PREFIX 1 'lab:json:product:' \
  SCHEMA '$.name' AS name TEXT '$.brand' AS brand TAG '$.price' AS price NUMERIC SORTABLE
docker exec "$name" redis-cli -e JSON.SET lab:json:product:1 '$' \
  '{"name":"Training laptop","brand":"Demo","price":150,"stock":{"hanoi":3}}'
docker exec "$name" redis-cli -e JSON.SET lab:json:product:2 '$' \
  '{"name":"Training keyboard","brand":"Demo","price":50,"stock":{"hanoi":8}}'
docker exec "$name" redis-cli -e JSON.NUMINCRBY lab:json:product:1 '$.stock.hanoi' 2
docker exec "$name" redis-cli -e FT.SEARCH lab:idx:products '@brand:{Demo} @price:[100 200]' RETURN 2 name price
docker exec "$name" redis-cli -e BF.RESERVE lab:bf:products 0.01 1000
docker exec "$name" redis-cli -e BF.ADD lab:bf:products product-1
test "$(docker exec "$name" redis-cli --raw BF.EXISTS lab:bf:products product-1)" = 1
docker exec "$name" redis-cli -e BF.EXISTS lab:bf:products never-added-id
docker exec "$name" redis-cli -e VADD lab:vectors VALUES 3 1 0 0 item-a
docker exec "$name" redis-cli -e VADD lab:vectors VALUES 3 0.95 0.05 0 item-b
docker exec "$name" redis-cli -e VADD lab:vectors VALUES 3 0 1 0 item-c
docker exec "$name" redis-cli -e VSIM lab:vectors ELE item-a COUNT 3 WITHSCORES
docker exec "$name" redis-cli MEMORY USAGE lab:vectors
docker rm -f "$name" >/dev/null
trap - EXIT
```

**Đạt:** search range trả product-1, stock Hanoi tăng thành 5, Bloom trả 1 cho item đã thêm, VSIM trả các neighbors với scores. Nếu search index chưa phản ánh ngay, xem `FT.INFO`/indexing progress và chờ có giới hạn; không đọc snapshot một lần rồi kết luận index sai.

**Bloom:** không có false negative đối với tập **đã được đưa đầy đủ vào filter** theo điều kiện thuật toán. Nếu app quên add product mới, Redis mất filter hoặc updater lag, “negative” không chứng minh product không tồn tại trong DB. False positive được phép; filter cần rebuild/versioning/fallback. Không dùng Bloom làm nguồn quyết định auth/thanh toán.

**Vector:** vector 3 chiều trong lab là dữ liệu nhân tạo để thấy API và khoảng cách; không tự có “ngữ nghĩa”. Chất lượng search phụ thuộc embedding model, dimension, normalization/metric, recall, filter và dữ liệu. Cần benchmark recall/latency/memory trước khi chọn native Vector Sets hay Query Engine vector index; đừng coi một kết quả gần nhau là hệ thống RAG đã đúng.

Nguồn: [Redis Docker](https://redis.io/docs/latest/operate/oss_and_stack/install/install-stack/docker/), [JSON](https://redis.io/docs/latest/develop/data-types/json/), [Query Engine](https://redis.io/docs/latest/develop/ai/search-and-query/), [Bloom filters](https://redis.io/docs/latest/develop/data-types/probabilistic/bloom-filter/), [Vector Sets](https://redis.io/docs/latest/develop/data-types/vector-sets/).


---

## 16. VPS — vận hành có giới hạn, backup ngoài máy và rehearsal

### 16.1. Trình tự triển khai

1. Hoàn thành core labs local; ghi Docker context/version/digest và memory footprint.
2. Chọn VPS lab, xác minh SSH host key bằng kênh độc lập. Cài Docker theo notebook 01; không đổi SSH/firewall mù.
3. Chạy preflight `hostname`, `free -m`, `df -h`, `df -i`, `ss -lnt`; tránh máy đang có Redis thật ở port tương tự.
4. Dùng VS Code Remote SSH để chạy setup ở `${HOME}/backend-labs/redis` trên VPS. Chạy chỉ core; dừng DB nếu không học bài cache.
5. Giữ bind `127.0.0.1:16379`, không public `6379`; dùng SSH tunnel hoặc client cùng private network.
6. Thực hiện backup/restore rehearsal như bài 25, rồi lưu bản backup thứ hai ngoài VPS bằng kênh được phép.
7. Cấu hình monitoring/rotation/retention và thử alert bằng fault nhỏ có đường lui.
8. Chỉ sau đó mới thử Sentinel/Cluster topology nếu RAM đủ; một VPS không chứng minh HA liên máy.

```bash {"name":"redis-36-vps-tunnel","interactive":"true","excludeFromRunAll":"true"}
set -euo pipefail
ssh -o ExitOnForwardFailure=yes -L 16379:127.0.0.1:16379 -N backend-training
```

Chạy trên laptop sau khi SSH alias đã cấu hình/verify. Tunnel tồn tại đến Ctrl+C. Nếu local core đang dùng 16379, đổi phía trái thành 26379. Redis Insight hoặc client local kết nối tunnel và credential lab; không mở dashboard ra Internet. Trên VS Code Remote SSH bạn có thể chạy `./rcli.sh` trực tiếp, không cần tunnel cho mỗi lệnh.

### 16.2. Kiểm tra từ VPS và ngoài VPS

```bash {"name":"redis-37-vps-operational-snapshot","excludeFromRunAll":"true"}
set -euo pipefail
cd "${HOME}/backend-labs/redis"
test "$(cat .course-id)" = be-redis-course-v1
hostname
free -m
df -h .
df -i .
ss -lnt
docker compose -p be-redis ps
docker compose -p be-redis stats --no-stream
./rcli.sh INFO server > evidence/vps-server.txt
./rcli.sh INFO memory > evidence/vps-memory.txt
./rcli.sh INFO persistence > evidence/vps-persistence.txt
./rcli.sh INFO stats > evidence/vps-stats.txt
docker compose -p be-redis logs --no-color --tail 100 redis > evidence/vps-redis.log
```

Từ một máy bạn kiểm soát ngoài VPS, thử kết nối **đúng public IP/port của mình** với timeout ngắn; 16379 không được mở public. Chỉ scan/check địa chỉ thuộc phạm vi bạn sở hữu hoặc được phép. Security group, host firewall và Docker port binding cần cùng phù hợp; auth không thay private exposure policy.

### 16.3. Nâng cấp Redis không chỉ đổi image tag

| Giai đoạn | Việc cần chứng minh |
|---|---|
| Trước nâng | version/client/module compatibility, backup restore, dung lượng/headroom |
| Staging | restore data thật đã ẩn thông tin, test scripts/ACL/commands/latency |
| Cutover | maintenance/rolling plan theo topology, discovery/reconnect và write policy |
| Sau nâng | invariants, command errors, memory, replication, persistence |
| Rollback | khả năng đọc format mới bằng binary cũ **không được giả định**; dùng backup/plan đã thử |

Không chạy hai major versions trên cùng writable data directory. Test restore sang phiên bản đích trước; nếu format mới không backward-compatible, rollback image đơn thuần có thể không khởi động. Với cache có thể rebuild, plan khác với queue state cần phục hồi chính xác.

### 16.4. Bảo vệ secrets và dữ liệu

Dùng ACL riêng cho cache app, queue worker, monitoring, backup/admin; command allowlist cần theo library thực dùng, không tự áp `+@write` rộng rồi nghĩ là least privilege. Khi rotate password có thể cho hai password trong thời gian chuyển tiếp, chuyển clients, quan sát, rồi gỡ old password; cú pháp `>`/`<` trong shell cần quote. Chỉ thu hồi khi biết clients cũ đã hết.

Redis TLS/private network bảo vệ đường truyền theo threat model; backup encryption và host disk encryption giải quyết rủi ro khác. SSH tunnel chỉ bảo vệ đoạn tunnel, không tự bảo vệ traffic từ API đến Redis nếu API dùng đường khác.

Không dùng Redis để lưu dữ liệu nhạy cảm rồi cho rằng “nằm trong RAM nên không để lại dấu vết”: AOF/RDB, replica, snapshot VPS, logs, crash dump và swap có thể chứa dữ liệu. Retention/erase phải xét toàn chuỗi.

Nguồn: [Redis security](https://redis.io/docs/latest/operate/oss_and_stack/management/security/), [TLS](https://redis.io/docs/latest/operate/oss_and_stack/management/security/encryption/), [Docker firewall](https://docs.docker.com/engine/network/packet-filtering-firewalls/).

---

## 17. Sổ xử lý sự cố production

| Triệu chứng | Bằng chứng đầu tiên | Mitigation | Sửa lâu dài và regression |
|---|---|---|---|
| p99 cao, CPU Redis cao | commandstats, slowlog, big key | giảm command/payload lớn, rate limit | bounded queries, data model, load theo shape |
| p99 cao, slowlog yên | RTT, client queue, Node event-loop lag | giảm concurrency/chờ queue | deadlines, connection reuse, client metrics |
| Cache hit giảm đột ngột | evictions, expired keys, versioned namespace | hạn chế fallback DB | TTL jitter, sizing, warmup có tốc độ |
| DB sập theo Redis outage | fallback requests/pool wait | shed load/circuit, stale nếu cho phép | budget chung và chaos test outage |
| BullMQ job biến mất | Redis eviction policy, instance sharing | ngừng enqueue nếu không giữ state | queue Redis noeviction riêng + durable semantics |
| `OOM command not allowed` | maxmemory policy, used_memory | giảm writes, tăng capacity có căn cứ | retention/key bound/admission |
| Container exit 137 | inspect OOMKilled, cgroup/host logs | khôi phục instance, giảm load | headroom CoW/RSS, tách workloads |
| AOF/RDB save fail | INFO persistence/error log/disk | giảm write/khôi phục disk đúng phạm vi | disk alerts, backup policy, restore drills |
| READONLY sau failover | role và endpoint client | rediscover primary | Sentinel/Cluster aware client, retry semantics |
| CROSSSLOT | keys và slot IDs | tách command hoặc colocate aggregate | hash-tag design, integration test Cluster |
| Streams pending tăng | oldest pending age, worker logs | reclaim theo min-idle hợp lý | heartbeat, idempotency, quarantine |
| Hot key khi shard tăng | per-key samples, shard CPU imbalance | local cache/read strategy | split aggregate nếu domain cho phép |
| Session biến mất | eviction/TTL/overwrite/restore | user re-auth theo policy | riêng state instance, TTL tests |
| Redis trông “xanh” nhưng app lỗi | ACL errors, client reconnects | restore quyền/endpoint hợp lệ | canary, ACL regression, startup validation |
| Memory không giảm ngay sau DEL | RSS/allocator/background reclaim | quan sát, không restart mù | bounded keys, UNLINK, allocator tuning có đo |
| Đọc replica thấy dữ liệu cũ | replication offset/lag | đọc primary cho path nhạy | consistency contract, read-your-writes policy |

### Mẫu incident report

- **Tác động:** endpoint/user/job nào, thời gian, scope.
- **Sự kiện:** deploy/config/data/load thay đổi gì trước lỗi?
- **Giả thuyết:** chỉ 2–3 giả thuyết ưu tiên, mỗi cái có phép kiểm tra.
- **Evidence:** metrics, log request/operation ID, config/version; không kèm secret.
- **Mitigation:** thay đổi gì, vì sao có thể đảo lại, tiêu chí dừng.
- **Root cause:** cơ chế tạo lỗi, không chỉ tên symptom “Redis chậm”.
- **Phòng tái diễn:** test, monitor, owner và thời hạn.
- **Recovery validation:** invariants dữ liệu, backlog/age và user-facing latency trở lại bình thường.

### Sáu tình huống để tự suy nghĩ sâu

**A. TTL của khóa cron 30 giây, job mất 90 giây.**  
Bạn có thể có nhiều worker cùng làm. Gia hạn lease bằng compare-token giúp, nhưng vẫn cần xử lý mất lease và side-effect idempotency. Bỏ TTL để “không bị trùng” sẽ làm dead owner giữ khóa vô hạn.

**B. Cache đã invalidate, nhưng user vẫn thấy dữ liệu cũ.**  
Reader cũ có thể repopulate, client/CDN còn cache khác, replica read lag hoặc cache key không đồng nhất. Xác định cache layer và version thực, không thêm DEL vào mọi nơi một cách ngẫu nhiên.

**C. HLL báo số visitor thấp hơn billing database.**  
HLL gần đúng và identity dedupe có semantics riêng; nó không là audit ledger exact. Xem yêu cầu trước khi chọn cấu trúc.

**D. Worker ACK trước DB commit để tăng tốc.**  
Crash sau ACK nhưng trước commit làm mất nghiệp vụ. Commit rồi ACK có thể redeliver sau crash; consumer cần unique inbox + business mutation cùng transaction. Notebook 05 thực hành chi tiết.

**E. Dùng SETNX operationId rồi gọi provider.**  
Crash sau SETNX trước provider làm lần retry bị chặn dù chưa có effect; TTL hết khi provider chậm có thể cho duplicate. Cần durable state machine, request hash, provider key, reconcile trạng thái unknown.

**F. Restore Redis từ backup hôm qua, SQL vẫn là hôm nay.**  
Jobs/cache/session/version có thể không cùng thời điểm. Rebuild cache từ source, replay jobs có idempotency, đối chiếu invariants; đừng khôi phục mù rồi mở traffic ngay.

---

## 18. Bài tập có lời giải định hướng

### Bài 1 — OTP một lần

**Đề:** OTP hết hạn sau 2 phút, chỉ dùng một lần, giới hạn thử sai, nhiều API instances.

**Thiết kế cần có:** không lưu/log OTP plaintext nếu tránh được; hash/HMAC theo threat model, key scope account/action, atomic compare+consume, counter attempts/expiry, resend policy, anti-enumeration và fail-closed khi không xác minh được. GET rồi DEL riêng có race. Không dùng OTP verification làm financial transaction lock.

**Thử nghiệm:** hai verification đồng thời cùng mã, chỉ một accepted; mã sai không xóa mã đúng tùy policy; quá attempts bị chặn; expiry, resend, Redis outage.

### Bài 2 — Cache multi-tenant

**Đề:** product ID 42 tồn tại ở hai tenant, user có roles khác nhau.

**Lời giải:** tenant đã xác minh trong key, xét permission-dependent representation; cache public projection riêng với private fields. Request hash/body không thay identity. Test hai tenants cùng ID, đổi role, revoke, invalidate và key migration v1→v2.

### Bài 3 — Batch API 100 products

**Đề:** 100 GET tuần tự chậm.

**Lời giải:** MGET khi cùng backend/slot phù hợp hoặc pipeline có giới hạn; xử lý partial miss bằng DB batch, giữ thứ tự input, negative cache rõ. Cluster MGET across slots cần client strategy/partition, không mặc định một MGET hoạt động với mọi keys. Benchmark cả payload/network/JSON parse.

### Bài 4 — Rate limiter cho đối tác

**Đề:** đối tác được 100 request/giây và tối đa 20 request đang chạy.

**Lời giải:** cần rate limit **và** concurrency limit, không thay nhau. Bucket key theo credential đã xác minh, retry-after, endpoint cost, bounds/cardinality, Redis failure policy. Lease concurrency cần đảm bảo release đúng token và timeout.

### Bài 5 — Cache “không được cũ quá 5 giây”

**Đề:** DB update liên tục, reader có thể pause.

**Lời giải:** TTL 5 giây từ lúc populate chưa đủ nếu snapshot cũ đã bị giữ 20 giây. Gắn source version/timestamp và hard freshness validation; invalidate/rebuild và fallback authoritative read khi vượt age. Clock differences cần xét nếu dùng wall time cross-host.

### Bài 6 — Queue 1 triệu jobs tồn lại

**Đề:** Redis đầy, worker tăng gấp 10 làm provider trả 429.

**Lời giải:** tăng workers không tăng capacity provider. Cần queue age, concurrency/rate budget, retries có jitter, poison quarantine, admission/backpressure, retention và dung lượng. Không xóa keys queue thủ công để giảm RAM. Xem notebook Message Queue.

### Bài 7 — “Lock dùng token ngẫu nhiên là an toàn tuyệt đối”

**Lời giải:** token chỉ chống unlock nhầm. Expired lease + stale worker vẫn có thể ghi. Cần resource fencing hoặc transaction/idempotency; thử case pause và resume vượt TTL.

### Bài 8 — backup 10GB hoàn thành, RTO 10 phút

**Lời giải:** chưa chứng minh. Restore lên máy trống, đo transfer/decrypt/load/warmup/client reconnect/business checks; phụ thuộc có thể làm thời gian tổng dài hơn. RPO được quyết định bởi thời điểm backup/persistence/replay, không phải chỉ dung lượng.

---

## 19. Capstone — cache đáng tin cho API catalog và job subsystem

### Phần bắt buộc

Tạo repository nhỏ theo kiến trúc bạn quen: controller → use case → cache adapter/repository, domain không import Redis client. DB giữ product version; Redis cache chỉ projection được phép cũ. Có key policy, TTL/jitter, negative cache, refresh coordination, bounded fallback và metrics.

Thêm Streams hoặc BullMQ cho một tác vụ không đồng bộ. State queue tách instance/policy cache. Consumer có operation ID bền vững và chống duplicate trong transaction; producer có outbox nếu DB và enqueue cần gắn nhau. Phần queue đầy đủ học tiếp ở notebook 05.

### Thử nghiệm phải vượt qua

1. Hai request cùng miss không tạo query storm ngoài budget.
2. Invalidations race được tái hiện; contract stale và mitigation được viết rõ.
3. Redis chết 1 phút: catalog giảm chất lượng có kiểm soát, auth không tự bỏ kiểm tra.
4. Key lớn/đầy RAM: quan sát rõ Redis OOM vs eviction vs container OOM.
5. Worker crash trước/sau effect: không mất nghiệp vụ và không nhân đôi effect.
6. Restart giữ data theo config; backup restore sang instance mới được kiểm chứng.
7. Sentinel failover hoặc Cluster redirect: client behavior phù hợp, không coi timeout là chắc chắn chưa ghi.
8. Không public service/admin port; credentials và logs không chứa dữ liệu thật.

### Rubric hoàn thành

| Mức | Bạn làm được |
|---|---|
| Nền tảng | Chọn đúng data structure và giải thích complexity/bounds |
| Lập trình | Atomic update có validate, pipeline khác transaction, WATCH connection isolation |
| Correctness | Phân biệt lease/idempotency/fencing và consistency với DB |
| Vận hành | Đọc metrics, restore, memory policy, failover/redirect |
| Chuyển giao | Runbook người khác dùng được và evidence tái hiện lỗi |

**Không cần nhớ mọi lệnh.** Cần biết invariant, assumption và failure mode; khi thiếu chắc chắn, tra đúng version và thiết kế một thí nghiệm nhỏ để kiểm chứng.

---

## 20. RESET — dừng project, giữ data/evidence

```bash {"name":"redis-38-stop-core","excludeFromRunAll":"true"}
set -euo pipefail
cd "${HOME}/backend-labs/redis"
test "$(cat .course-id)" = be-redis-course-v1
docker compose -p be-redis --profile database --profile tools down
```

Cell này giữ named volumes. Khi muốn **xóa dữ liệu riêng core lab** để học lại:

```bash {"name":"redis-39-delete-core-lab-volumes","interactive":"true","excludeFromRunAll":"true"}
set -euo pipefail
cd "${HOME}/backend-labs/redis"
test "$(cat .course-id)" = be-redis-course-v1
read -r -p "Đã giữ backup/evidence cần thiết? Gõ DELETE-REDIS-LAB-DATA: " answer
test "$answer" = DELETE-REDIS-LAB-DATA
docker compose -p be-redis --profile database --profile tools down --volumes
```

Không xóa mọi Redis trên host; HA/Cluster là project riêng, dùng đúng file và cân nhắc state cần giữ. Container tạm của restore/memory/Redis8 có trap cleanup; nếu bị kill -9 tool/shell thì trap có thể chưa chạy, dùng label `course=be-redis` để **liệt kê rồi kiểm tra từng container**, không tự rm tất cả theo pattern rộng.

## Danh mục nguồn chính thức

- [Redis commands](https://redis.io/docs/latest/commands/) — version “Since” và return semantics.
- [Node-redis project](https://github.com/redis/node-redis), [production usage](https://redis.io/docs/latest/develop/clients/nodejs/produsage/).
- [Transactions](https://redis.io/docs/latest/develop/using-commands/transactions/), [Lua](https://redis.io/docs/latest/develop/programmability/eval-intro/), [pipelining](https://redis.io/docs/latest/develop/using-commands/pipelining/).
- [Distributed locks](https://redis.io/docs/latest/develop/use/patterns/distributed-locks/).
- [Streams](https://redis.io/docs/latest/develop/data-types/streams/), [Pub/Sub](https://redis.io/docs/latest/develop/pubsub/).
- [Persistence](https://redis.io/docs/latest/operate/oss_and_stack/management/persistence/), [eviction](https://redis.io/docs/latest/develop/reference/eviction/).
- [Replication](https://redis.io/docs/latest/operate/oss_and_stack/management/replication/), [Sentinel](https://redis.io/docs/latest/operate/oss_and_stack/management/sentinel/), [Cluster](https://redis.io/docs/latest/operate/oss_and_stack/reference/cluster-spec/).
- [ACL](https://redis.io/docs/latest/operate/oss_and_stack/management/security/acl/), [latency](https://redis.io/docs/latest/operate/oss_and_stack/management/optimization/latency/).
- [JSON](https://redis.io/docs/latest/develop/data-types/json/), [Query Engine](https://redis.io/docs/latest/develop/ai/search-and-query/), [Vector Sets](https://redis.io/docs/latest/develop/data-types/vector-sets/).
- [Runme configuration](https://docs.runme.dev/configuration/cell-level/).

Bộ nguồn được dùng để kiểm tra cơ chế; ví dụ, bài tập, scenario và baseline trong giáo trình là thiết kế học tập, không phải cấu hình được hãng bảo đảm cho mọi production.








