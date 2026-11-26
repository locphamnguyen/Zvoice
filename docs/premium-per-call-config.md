# Cấu hình theo cuộc cho voice agent Premium

Tài liệu vận hành cho đường «cấu hình theo cuộc» giữa backend ZCRM (Go) và máy voice agent Premium tự host
(zvoicebot). Mục đích: mỗi bot dùng **đúng lời chào, lời dặn, giọng và mô hình của chính nó**, thay vì persona mặc
định của agent cố định trên máy.

> Repo này công khai: tài liệu chỉ dùng chỗ giữ chỗ (`<backend-container>`, `<agent-host>`…), không ghi tên khách,
> địa chỉ máy hay khoá.

## Cách hoạt động

1. Bot engine `premium` được gọi (Zalo, trunk SIP, gọi thử…). Backend đọc cấu hình đã xuất bản của bot.
2. Nếu bật `ZALO_VOICEBOT_PREMIUM_SESSIONS=1`, backend gửi `POST /v1/zcrm/sessions` tới máy agent, kèm header
   `xi-api-key`. Thân gồm `customer_id`, `system_prompt`, `first_message`, `language`, `voice`, `llm`, `expressive`,
   `idle_prompt`, `goodbye`, `pronounce`, `stt_fix`, `knowledge_sha256` / `knowledge`, `allow_end_call` và, khi cuộc
   gọi đi qua khối «AI điều phối» hoặc chuyển máy theo KB TCRM, `route_targets`.
3. Máy trả `200 {session_id: "zs-…", signed_url}` ⇒ backend mở WebSocket tới `signed_url` và agent nói lời chào của
   bot.
4. Máy trả **404** (bản agent cũ, chưa có endpoint) hoặc **422** (từ chối cấu hình) ⇒ backend **rơi về
   `get-signed-url`** với agent cố định (`ZALO_VOICEBOT_SELFHOST_AGENT_ID`). Cuộc gọi vẫn chạy nhưng **mất lời dặn
   và lời chào của bot**: agent nói câu chào persona mặc định của máy.

Bước 4 là lý do lỗi này khó thấy: cuộc gọi không hỏng, chỉ «sai người nói».

## Sự cố 26/11/2026: lời chào persona mặc định thay cho lời chào của bot

**Triệu chứng.** Mỗi cuộc gọi tới bot Premium trên trunk có chuyển máy theo KB TCRM, log backend ghi:

```text
[voicebot] máy Premium không nhận cấu hình theo cuộc (voicebot: máy từ chối cấu hình phiên) — dùng get-signed-url cũ
INFO voicebot: máy Premium phục vụ … agent=<agent-cố-định> endpoint="Máy 1"
INFO voicebot: Agent trả lời … text="<lời chào của persona mặc định>"
```

**Nguyên nhân gốc.** Khi bot nghe máy trên trunk có callback CCMS, backend gửi kèm `route_targets` = mọi nút KB
TCRM có action Transfer, với `key` = **mã nút TCRM nguyên văn**. Mã nút thật có chữ hoa
(dạng `smartbot-TenNut_ChuyenMay`), trong khi máy agent chỉ nhận key chữ thường (`^[a-z0-9][a-z0-9_-]*$`). Một key
sai ⇒ máy trả 422 cho **cả phiên** ⇒ backend rơi về agent cố định. Prompt và lời chào của bot không có lỗi gì: gửi
lại đúng cấu hình đó mà bỏ `route_targets` thì máy nhận ngay.

Log backend lúc đó không ghi lý do 422 (chỉ «máy từ chối cấu hình phiên»), nên phải đọc log của máy agent mới biết:

```text
WARNING agent.server: POST /v1/zcrm/sessions ⇒ 422 invalid_request: route_targets: key chữ thường/số/-/_ (≤ 128 ký tự), không trùng
```

**Đã sửa.**

| Nơi | Thay đổi |
|---|---|
| Máy agent (zvoicebot `agent/zsessions.py`) | Key nơi chuyển nhận chữ hoa và `.` `:` ngoài chữ thường/số/`-`/`_`, **giữ nguyên hoa/thường** (backend so khớp đúng chuỗi khi agent gọi `route_call`). Vẫn chặn khoảng trắng, xuống dòng, ký tự lạ, key trùng. |
| Backend (zcrm-core `internal/zago/call/voicebot/zsession.go`) | Lỗi 422 mang lý do của máy: `máy từ chối cấu hình phiên: <mã>: <câu>`. `name` / `description` của nơi chuyển được cắt theo trần hợp đồng (80 / 400 ký tự) thay vì để máy 422 cả phiên. |

## Kiểm tra nhanh

**1. Backend có rơi về đường cũ không?**

```sh
docker logs --since 30m <backend-container> 2>&1 | grep -i voicebot | grep -E "cấu hình theo cuộc|phục vụ|Agent trả lời"
```

Tốt: không còn dòng «không nhận cấu hình theo cuộc», và câu `Agent trả lời` đầu tiên là lời chào của bot.

**2. Máy agent từ chối vì sao?** Từ bản backend có lý do 422, đọc ngay trong dòng log ở bước 1. Bản cũ hơn: đọc log
máy agent:

```sh
grep -n "zcrm/sessions ⇒ 4" <thư-mục-agent>/logs/agent.log | tail
```

**3. Thử trực tiếp một phiên** (từ máy được phép gọi agent; agent có danh sách IP cho phép nên WebSocket từ IP lạ bị
403):

```sh
curl -s -H "xi-api-key: $KEY" -H 'Content-Type: application/json' \
  --data-binary @session.json https://<agent-host>/v1/zcrm/sessions
```

`session.json` dựng từ cấu hình bot (`prompt` → `system_prompt`, `first_message`, `language`, `llm`, `expressive`),
thêm `route_targets` nếu cuộc gọi có chuyển máy. Gửi lần lượt có / không từng trường để khoanh trường bị từ chối.

## Khắc phục theo mã lỗi 422

| Mã / câu của máy | Nguyên nhân thường gặp | Cách xử lý |
|---|---|---|
| `invalid_request: route_targets: key …` | Mã nút / nơi chuyển có ký tự máy không nhận, hoặc trùng | Cập nhật máy agent lên bản có sửa ở trên; nếu key có khoảng trắng / ký tự lạ, sửa mã nút ở nguồn (KB TCRM, khối «AI điều phối»). |
| `invalid_request: route_targets phải là danh sách tối đa N` | Quá nhiều nút Transfer | Gộp nút ở KB, hoặc nâng `MAX_ROUTES` ở cả máy agent và test hợp đồng. |
| `invalid_request: thiếu first_message` | Bot chưa có lời chào (hoặc khối điều phối ghi đè bằng chuỗi rỗng) | Điền lời chào cho bot rồi xuất bản lại. |
| `too_large: … quá dài` | Lời dặn / lời chào / tri thức vượt trần | Rút gọn; trần máy phải khớp trần của backend (`maxPromptRunes`, `maxFirstMessageRunes`). |
| `unknown_voice` | `ZALO_VOICEBOT_PREMIUM_VOICES` trỏ tới giọng máy không có | Sửa ánh xạ hoặc chép tệp giọng vào `voices/` của máy. |
| `sha_mismatch` | Văn bản tri thức và sha không khớp | Lỗi backend; báo lại kèm log. |

Sau khi vá mã trên máy agent: sao lưu tệp, kiểm tra `healthz` có `"sessions":0` (không có cuộc đang chạy) rồi mới khởi
động lại tiến trình agent; chờ `"status":"ok"` trước khi cho nhận cuộc gọi.

## Nguyên tắc để không lặp lại

- Trần và định dạng của `/v1/zcrm/sessions` là **hợp đồng hai đầu**: đổi ở máy agent thì đổi test hợp đồng ở backend và
  ngược lại.
- Mã đến từ hệ thống ngoài (mã nút TCRM, mã nơi chuyển) phải được nhận nguyên văn hoặc ánh xạ ở backend, không được làm
  hỏng cả phiên.
- Log backend luôn ghi lý do từ chối (mã + câu chung), không bao giờ ghi lời dặn, lời chào hay tri thức.
