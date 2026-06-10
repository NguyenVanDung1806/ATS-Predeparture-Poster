# ATS Pre-departure Poster Webapp - Implementation Document

> **Mục tiêu:** Webapp cho phép học sinh tại event pre-departure điền thông tin, upload ảnh cá nhân, hệ thống tự động tạo poster mang brand ATS và gửi email kèm poster + lời chúc.
>
> **Phase 1:** HTML form tự host -> n8n webhook -> OpenAI Image API -> Google Drive -> SendGrid -> Google Sheets
> **Deploy:** VPS ATS, dự kiến `poster.ats.org.vn` cho form và `n8n.ats.org.vn` cho workflow
> **Scale:** 30-100 học sinh/event, nhiều event/năm

---

## Nguyên tắc chỉnh bản này

Bản này sửa theo hướng triển khai được thật ở event, không chỉ là outline.

- Không để form chờ GPT tạo ảnh. Frontend nhận submit xong thì hiện trang cảm ơn ngay.
- Không làm mất binary ảnh khi đi qua Code node trong n8n.
- Check trùng submission trước khi gọi OpenAI để tránh tốn tiền và gửi email lặp.
- Có consent rõ ràng vì hệ thống xử lý tên, email, trường, quốc gia và ảnh mặt học sinh.
- Drive link mặc định dùng cho staff, không public bừa bãi nếu không cần.
- OpenAI model/endpoint/price phải verify lại trên docs chính thức trước event vì phần này thay đổi theo thời gian.

---

## Kiến trúc Phase 1

```text
[QR code tai event]
        |
        v
[HTML form tu host - mobile first]
        |
        | POST multipart/form-data bang fetch()
        v
[n8n Webhook - respond immediately]
        |
        v
[Validate + normalize input]
        |
        v
[Check duplicate submission]
        |
        v
[Save original image to Google Drive]
        |
        v
[OpenAI Image API - edit input photo into poster]
        |
        v
[Convert b64 poster to binary]
        |
        v
[Save generated poster to Google Drive]
        |
        v
[SendGrid email with poster attachment]
        |
        v
[Google Sheets log: COMPLETED / FAILED / SKIPPED_DUPLICATE]
```

**Lưu ý quan trọng:** Poster binary trong n8n chỉ tồn tại trong execution data. Cần lưu ảnh gốc và poster output vào Drive để staff có thể review, resend, regenerate hoặc audit sau event.

---

## Phase 1 - Target thực tế

**Có thể làm trong 1 ngày nếu:**

- VPS/n8n/Traefik hoặc Nginx đã chạy ổn.
- Đã có credential OpenAI, Google Drive/Sheets, SendGrid.
- SendGrid domain authentication đã cấu hình trước.
- Chấp nhận UI form đơn giản, không có preview poster realtime.

**Nên chuẩn bị trước event ít nhất 3-7 ngày nếu dùng thật với học sinh**, vì cần test deliverability email, ảnh thật, prompt, quota, privacy wording và fallback vận hành.

---

## Bước 1 - HTML Form

Form tự build và deploy lên `https://poster.ats.org.vn`.

### Fields

| Field | Input type | Validate |
| --- | --- | --- |
| Họ và tên | `text` | Required, trim, max length |
| Email | `email` | Required, format check |
| Trường đại học sẽ học | `text` | Required, max length |
| Quốc gia | `select` | Required, dùng fixed options |
| Sở thích / hoạt động | `textarea` | Optional, max length |
| Ảnh của bạn | `file` | Required, `accept="image/jpeg,image/png,image/webp"`, max 10MB |
| Consent xử lý dữ liệu | `checkbox` | Required |
| Event ID | hidden | Giá trị cố định theo event, ví dụ `predeparture-2026-hcm-01` |
| Submission ID | hidden | UUID tạo khi load form |

### Consent text đề xuất

```text
Tôi đồng ý cho ATS Education sử dụng thông tin và ảnh tôi cung cấp để tạo poster pre-departure, gửi poster qua email, lưu trữ nội bộ phục vụ vận hành sự kiện và xử lý lại khi cần. Tôi hiểu ảnh và thông tin có thể được xử lý qua các dịch vụ bên thứ ba như OpenAI, Google Drive và SendGrid.
```

Nếu có khả năng người tham gia dưới 18 tuổi, cần thêm câu hỏi xác nhận hoặc quy trình phụ huynh/guardian consent theo chính sách ATS.

### Submit UX

Không để browser submit thẳng rồi chờ redirect từ n8n. Dùng JavaScript `fetch()`:

1. User bấm submit.
2. Disable nút submit, hiện trạng thái "Đang tải ảnh lên...".
3. POST `multipart/form-data` đến n8n webhook.
4. Nếu webhook trả `202 Accepted`, hiện thank-you state ngay trên trang:

```text
Poster của bạn đang được tạo!
Email sẽ đến trong khoảng 30-60 phút. Nhớ kiểm tra cả Spam/Promotions nhé.
```

5. Nếu upload lỗi, cho user thử lại, không reset form.

### Client-side validation

Client-side chỉ để UX tốt hơn, không được xem là bảo mật.

- Check file size <= 10MB.
- Check MIME trong browser: JPEG/PNG/WebP.
- Check email format.
- Check required fields và consent.
- Chặn double-click submit bằng disabled state.

### Server/proxy config cần nhớ

Nếu form upload trực tiếp vào n8n qua Nginx/Traefik:

- Set request body limit lớn hơn max file một chút, ví dụ 12-15MB.
- Set upload timeout đủ cho mạng yếu tại event.
- Bật HTTPS.
- Không hardcode webhook test URL trên QR code. QR phải trỏ production URL.

---

## Bước 2 - n8n Workflow

### Node 1 - Webhook Trigger

```text
Type: Webhook
HTTP Method: POST
Path: predeparture
Response Mode: Immediately
Response Code: 202
Response Body:
  { "ok": true, "message": "Poster generation queued" }
Binary Data: ON
```

Payload kỳ vọng:

```text
$json.body.ten
$json.body.email
$json.body.truong
$json.body.quocgia
$json.body.soThich
$json.body.consent
$json.body.eventId
$json.body.submissionId
$binary.anhCuaBan
```

Tên field trong form phải khớp chính xác với `name` attribute của input.

---

### Node 2 - Code: Normalize + Build Prompt

Node này phải preserve binary từ input. Nếu chỉ return JSON, các node sau có thể mất `$binary.anhCuaBan`.

Khuyến nghị cấu hình Code node ở mode **Run Once for Each Item**.

```javascript
const item = $input.item;
const body = item.json.body || {};

const clean = (value, max = 200) =>
  String(value || "")
    .trim()
    .replace(/\s+/g, " ")
    .slice(0, max);

const name = clean(body.ten, 80);
const email = clean(body.email, 120).toLowerCase();
const university = clean(body.truong, 120);
const country = clean(body.quocgia, 60);
const interests = clean(body.soThich, 180) || "học tập và khám phá văn hóa mới";
const consent = body.consent === "on" || body.consent === "true" || body.consent === true;
const eventId = clean(body.eventId, 80);
const submissionId = clean(body.submissionId, 80);
const shortName = name ? name.split(" ").pop() : "bạn";

const prompt = `
Create a polished study-abroad celebration poster for ATS Education.

Use the uploaded photo as the main subject. Preserve the student's face, identity, age impression, and natural appearance. Do not invent a different person.

Student details:
- Name: ${name}
- University: ${university}
- Destination country: ${country}
- Personal interests: ${interests}

Design requirements:
- Format: vertical social poster, 1024x1536.
- Visual style: modern, energetic, premium education-event design.
- Brand colors: navy blue #003087, gold #FFB81C, white accents.
- Main subject: the uploaded student photo, smiling or confident, centered.
- Background: recognizable but tasteful scenery or landmark associated with ${country}.
- Text overlay: "${name}" as the primary title.
- Text overlay: "${university}" as secondary text.
- Branding: subtle "ATS Education" text/logo area in the top-right.
- Decorative elements: graduation cap, stars, travel/study-abroad accents, light confetti.
- Keep all text legible. Avoid misspelling the student's name and university.
- No extra people, no fake logos of the university, no distorted face, no inappropriate content.
`.trim();

const congratsByCountry = {
  Australia: `Chúc ${shortName} có một hành trình tuyệt vời tại Australia! ATS Education luôn đồng hành cùng bạn trên chặng đường mới.`,
  Canada: `Chúc ${shortName} bắt đầu hành trình tại Canada với thật nhiều năng lượng, trải nghiệm đẹp và cơ hội mới.`,
  "United States": `Chúc ${shortName} chinh phục hành trình học tập tại Mỹ thật trọn vẹn. ATS Education tin bạn sẽ tạo nên nhiều dấu ấn đáng tự hào.`,
  "United Kingdom": `Chúc ${shortName} có những năm tháng học tập đáng nhớ tại Vương quốc Anh. ATS Education luôn sẵn sàng đồng hành khi bạn cần.`,
  "New Zealand": `Chúc ${shortName} tận hưởng hành trình học tập tại New Zealand với thật nhiều trải nghiệm đáng nhớ.`
};

const congratsMessage =
  congratsByCountry[country] ||
  `Chúc ${shortName} có một hành trình du học thật ý nghĩa. ATS Education tự hào đồng hành cùng ước mơ của bạn.`;

return {
  json: {
    ...item.json,
    name,
    email,
    university,
    country,
    interests,
    consent,
    eventId,
    submissionId,
    prompt,
    congratsMessage,
    status: "NORMALIZED"
  },
  binary: item.binary
};
```

---

### Node 3 - Validate Required Fields

Đặt trước mọi node tốn tiền hoặc lưu dữ liệu.

Điều kiện AND:

| Field | Condition |
| --- | --- |
| `{{ $json.name }}` | is not empty |
| `{{ $json.email }}` | is not empty |
| `{{ $json.university }}` | is not empty |
| `{{ $json.country }}` | is not empty |
| `{{ $json.consent }}` | equals true |
| `{{ $json.eventId }}` | is not empty |
| `{{ $json.submissionId }}` | is not empty |
| `{{ $binary.anhCuaBan }}` | exists |

Nhánh FALSE:

- Append row vào Google Sheets với `Status = VALIDATION_FAILED`.
- Không gọi OpenAI.
- Không gửi email.

### Validate file kỹ hơn

Thêm IF hoặc Code node để check metadata binary:

- MIME chỉ cho phép `image/jpeg`, `image/png`, `image/webp`.
- File size <= 10MB.
- Nếu n8n không expose size ổn định, vẫn giữ proxy body limit + browser check.

---

### Node 4 - Check Duplicate

Trước khi gọi OpenAI, lookup Google Sheets theo:

1. `submissionId`
2. hoặc `email + eventId`

Nếu tìm thấy row có `Status = COMPLETED` hoặc `Email Sent = TRUE`:

- Append/update log `SKIPPED_DUPLICATE`.
- Stop workflow.

Không nên chỉ check email global, vì cùng một học sinh có thể tham gia event khác trong tương lai. Thêm field `eventId`, ví dụ `predeparture-2026-hcm-01`, sẽ sạch hơn.

---

### Node 5 - Upload Original Image To Google Drive

Upload ảnh gốc trước khi gọi OpenAI để còn dữ liệu xử lý lại nếu API fail.

```text
Type: Google Drive
Operation: Upload File
Parent Folder: ATS Posters/{eventId}/originals
File Name: original_{{ $json.eventId }}_{{ $json.submissionId }}_{{ $json.email }}.[ext]
File Content: $binary.anhCuaBan
```

Ghi chú:

- Đừng giả định file luôn là `.jpg`; tốt hơn map extension theo MIME.
- Folder originals nên private, chỉ staff có quyền xem.
- Không đưa link ảnh gốc vào email học sinh nếu không cần.

---

### Node 6 - OpenAI Image API

Theo docs OpenAI hiện tại, có thể generate/edit image bằng Image API hoặc Responses API. Với n8n Phase 1, dùng Image API `POST /v1/images/edits` là hướng đơn giản vì nhận multipart file trực tiếp.

**Trước event phải verify lại chính thức:**

- Model image đang dùng: ví dụ `gpt-image-1`, `gpt-image-1.5`, hoặc model image mới hơn nếu account hỗ trợ.
- Supported sizes: tài liệu hiện có size dọc `1024x1536`.
- Pricing/rate limit theo project/account.
- Response shape có `data[0].b64_json` với GPT image models.

```text
Type: HTTP Request
Method: POST
URL: https://api.openai.com/v1/images/edits
Authentication: Header Auth
  Name: Authorization
  Value: Bearer {{ $env.OPENAI_API_KEY }}
Content-Type: multipart/form-data
```

Form fields:

| Key | Value | Type |
| --- | --- | --- |
| `model` | model đã verify trước event | Text |
| `prompt` | `{{ $json.prompt }}` | Text |
| `n` | `1` | Text |
| `size` | `1024x1536` | Text |
| `image` | `$binary.anhCuaBan` | File binary |

Retry:

- Retry on fail: ON
- Max tries: 2
- Wait between tries: 10-15s

Nếu rate limit thấp, thêm Wait/Queue trước node OpenAI. Không chỉ thêm Wait sau node OpenAI, vì lúc đó request đã bắn đi rồi.

---

### Node 7 - Convert Poster Base64 To Binary

OpenAI trả poster dạng base64. Cần convert sang binary trước khi upload Drive hoặc attach SendGrid.

```text
Type: Move Binary Data / Convert to File
Mode: JSON to Binary
Source Key: data[0].b64_json
Destination Binary Property: posterBinary
MIME Type: image/png
File Name: poster_{{ $json.eventId }}_{{ $json.submissionId }}.png
```

Sau node này phải kiểm tra `$binary.posterBinary` tồn tại. Nếu không, log `OPENAI_EMPTY_OUTPUT`.

---

### Node 8 - Upload Generated Poster To Google Drive

```text
Type: Google Drive
Operation: Upload File
Parent Folder: ATS Posters/{eventId}/generated
File Name: poster_{{ $json.eventId }}_{{ $json.submissionId }}_{{ $json.email }}.png
File Content: $binary.posterBinary
```

Permission đề xuất:

- Folder `generated` private cho staff là mặc định an toàn hơn.
- Email học sinh nhận attachment nên không cần public link.
- Chỉ bật "Anyone with the link can view" nếu ATS thật sự muốn học sinh tải qua link Drive thay vì attachment.

Nếu cần share link Drive:

```text
POST https://www.googleapis.com/drive/v3/files/{{ posterFileId }}/permissions
Body: { "role": "reader", "type": "anyone" }
```

Nhưng phải cân nhắc privacy vì poster có tên, trường và ảnh mặt học sinh.

---

### Node 9 - SendGrid Email

```text
Type: SendGrid
Operation: Send Email
From Name: ATS Education
From Email: hello@ats.edu.vn
To: {{ $json.email }}
Subject: [ATS] Poster du học của {{ $json.name }} đã sẵn sàng
```

Dynamic Template Data:

```text
student_name:     {{ $json.name }}
university:       {{ $json.university }}
country:          {{ $json.country }}
congrats_message: {{ $json.congratsMessage }}
```

Attachment:

```text
Content:  $binary.posterBinary
Filename: poster-{{ $json.name }}.png
Type:     image/png
```

Lưu ý:

- Với SendGrid API raw, attachment content thường cần base64 string.
- Với n8n SendGrid node, kiểm tra node nhận binary property hay base64 expression theo version n8n đang chạy.
- Test bằng Gmail, Outlook và iCloud nếu có thể.
- Domain authentication SPF/DKIM phải xong trước event.

---

### Node 10 - Google Sheets Log

Sheet nên là nguồn vận hành cho staff trong Phase 1.

Columns đề xuất:

| Column | Value |
| --- | --- |
| Timestamp | `{{ $now.toISO() }}` |
| Event ID | `{{ $json.eventId }}` |
| Submission ID | `{{ $json.submissionId }}` |
| Tên | `{{ $json.name }}` |
| Email | `{{ $json.email }}` |
| Trường | `{{ $json.university }}` |
| Quốc gia | `{{ $json.country }}` |
| Status | `COMPLETED` / `FAILED` / `VALIDATION_FAILED` / `SKIPPED_DUPLICATE` |
| Email Sent | `TRUE` / `FALSE` |
| Error Code | nếu có |
| Original Drive URL | staff-only |
| Poster Drive URL | staff-only hoặc public nếu đã quyết định |
| OpenAI Model | model thực tế đã dùng |
| Created At | timestamp |

Nếu workflow fail giữa chừng, cần Error Workflow riêng để log status. Không chờ đến node cuối mới ghi mọi thứ, vì fail trước node Sheets sẽ mất dấu submission.

---

## SendGrid Dynamic Template

Template nên nói rõ poster nằm trong attachment, tránh phụ thuộc link Drive public.

```html
<!DOCTYPE html>
<html>
  <head>
    <meta charset="utf-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1" />
  </head>
  <body style="margin:0;padding:0;background:#f5f5f5;font-family:Arial,sans-serif;">
    <table width="100%" cellpadding="0" cellspacing="0">
      <tr>
        <td align="center" style="padding:32px 16px;">
          <table width="600" cellpadding="0" cellspacing="0" style="max-width:600px;width:100%;background:#fff;border-radius:8px;overflow:hidden;">
            <tr>
              <td style="background:#003087;padding:24px 28px;">
                <h1 style="color:#FFB81C;margin:0;font-size:24px;">ATS Education</h1>
                <p style="color:#fff;margin:6px 0 0;font-size:14px;">Pre-departure journey</p>
              </td>
            </tr>
            <tr>
              <td style="padding:28px 28px 12px;">
                <h2 style="color:#003087;margin:0 0 12px;font-size:20px;">Xin chào {{student_name}},</h2>
                <p style="color:#444;line-height:1.6;margin:0;">{{congrats_message}}</p>
              </td>
            </tr>
            <tr>
              <td style="padding:12px 28px 24px;">
                <div style="background:#f0f4ff;border-radius:8px;padding:16px;border-left:4px solid #003087;">
                  <p style="color:#003087;font-weight:bold;margin:0 0 6px;font-size:14px;">Poster của bạn đã được đính kèm trong email này.</p>
                  <p style="color:#666;font-size:13px;margin:0;line-height:1.5;">Bạn có thể tải file poster về và chia sẻ lên mạng xã hội nếu muốn.</p>
                </div>
              </td>
            </tr>
            <tr>
              <td style="background:#f0f0f0;padding:18px 28px;border-top:1px solid #e0e0e0;">
                <p style="color:#777;font-size:12px;margin:0;line-height:1.6;">
                  Email gửi bởi ATS Education sau sự kiện Pre-departure.<br />
                  Trường: {{university}} | Hotline: 1800 XXX XXX
                </p>
              </td>
            </tr>
          </table>
        </td>
      </tr>
    </table>
  </body>
</html>
```

Unsubscribe:

- Không ghi chắc `{{{unsubscribe}}}` nếu chưa cấu hình SendGrid Unsubscribe Group.
- Với email transactional/event fulfillment, hỏi lại team marketing/legal của ATS xem có cần unsubscribe link không.
- Nếu dùng unsubscribe, cấu hình trong SendGrid trước rồi test template thật.

---

## Privacy, Data Retention, Security

Đây là phần không nên bỏ qua vì dữ liệu có ảnh mặt.

### Data collected

- Họ tên
- Email
- Trường
- Quốc gia
- Sở thích/hoạt động
- Ảnh cá nhân
- Submission ID, event ID, timestamps

### Third-party processors

- OpenAI: xử lý ảnh và prompt để tạo poster.
- Google Drive/Sheets: lưu ảnh, poster, log.
- SendGrid: gửi email và attachment.

### Retention đề xuất

- Ảnh gốc: giữ 30-90 ngày sau event, sau đó xóa hoặc archive theo policy ATS.
- Poster generated: giữ lâu hơn nếu phục vụ CRM/marketing, nhưng cần consent rõ.
- Google Sheets log: giữ theo policy CRM.

### Access control

- Drive folders private theo event.
- Chỉ staff liên quan có quyền xem originals/generated.
- Không public folder originals.
- Không share spreadsheet rộng hơn nhóm vận hành.
- API keys nằm trong n8n credentials hoặc environment variables, không hardcode trong HTML.

---

## Edge Cases & Cách xử lý

### Form & Input

| Edge case | Xử lý |
| --- | --- |
| Thiếu required field | Client-side required + n8n validate |
| Không tick consent | Không xử lý, log `VALIDATION_FAILED` |
| Email sai format | Browser check + n8n regex basic |
| Ảnh > 10MB | Browser reject + proxy body limit |
| MIME không hợp lệ | Reject trước OpenAI |
| Submit 2 lần | Check `submissionId` và `email + eventId` trước OpenAI |
| Tên tiếng Việt có dấu | Form UTF-8, test với Nguyễn/Trần/Phượng/Đặng |
| Mạng yếu | Disable button, show upload state, cho retry |

### OpenAI Image API

| Edge case | Xử lý |
| --- | --- |
| API timeout | Retry 2 lần, sau đó log `OPENAI_TIMEOUT` |
| Rate limit | Queue/Wait trước node OpenAI, không bắn request đồng thời |
| Content policy block | Fallback prompt ít nhấn mạnh photorealistic, hoặc log để staff regenerate |
| Output không có `b64_json` | Log `OPENAI_EMPTY_OUTPUT`, không gửi email |
| Poster sai mặt/sai chữ | Staff review Drive/Sheets, regenerate thủ công |
| Chi phí/quota thay đổi | Verify trên OpenAI dashboard 1 ngày trước event |

### Email

| Edge case | Xử lý |
| --- | --- |
| Email vào spam | SPF/DKIM, test mail-tester, gửi từ domain uy tín |
| Hard bounce | SendGrid Event Webhook -> update Sheets `BOUNCED` |
| Attachment lỗi | Test binary/base64 theo n8n SendGrid node thực tế |
| Email gửi trùng | Dedupe trước OpenAI + check `Email Sent` trước SendGrid |
| SendGrid quota thiếu | Kiểm tra quota 1 ngày trước event |

### Google Drive / Sheets

| Edge case | Xử lý |
| --- | --- |
| Drive upload fail | Retry. Nếu original fail, dừng trước OpenAI; nếu poster fail, vẫn có thể gửi attachment nhưng log rõ |
| Permission link không xem được | Staff dùng private Drive; chỉ public generated nếu đã quyết định |
| File name trùng | Dùng `eventId + submissionId + email` |
| Sheets append fail | Error Workflow hoặc retry, vì Sheets là nguồn vận hành Phase 1 |

### n8n / Infrastructure

| Edge case | Xử lý |
| --- | --- |
| n8n crash giữa chừng | Xem executions, dùng original Drive + Sheets để xử lý lại |
| VPS quá tải | Với 30-100 người thường ổn, nhưng cần queue OpenAI |
| Webhook bị spam | Hidden submissionId không đủ bảo mật; có thể thêm simple event code hoặc rate limit |
| Form không load | Chuẩn bị Tally/Google Form backup nhận data + ảnh |

---

## Checklist trước Event

### 1 tuần trước

- [ ] Chốt domain form `poster.ats.org.vn`.
- [ ] Deploy HTML form mobile-first.
- [ ] Thêm consent checkbox và event ID.
- [ ] Tạo Google Drive folders theo event: `originals`, `generated`.
- [ ] Tạo Google Sheets đúng columns.
- [ ] Kết nối n8n credentials: OpenAI, Google Drive, Google Sheets, SendGrid.
- [ ] Build workflow n8n end-to-end.
- [ ] Setup Error Workflow để log fail.
- [ ] Verify OpenAI model/endpoint/size/pricing/rate limit trên docs/dashboard chính thức.
- [ ] Test prompt với 15-20 ảnh thật hoặc ảnh nội bộ được phép dùng.
- [ ] Setup SendGrid Dynamic Template.
- [ ] Verify SPF/DKIM.
- [ ] Test email deliverability.

### 1 ngày trước

- [ ] Test QR code trên iOS và Android.
- [ ] Test upload ảnh 2MB, 5MB, 10MB.
- [ ] Test tên tiếng Việt có dấu.
- [ ] Test duplicate submission.
- [ ] Test một case OpenAI fail giả lập.
- [ ] Confirm OpenAI balance/quota.
- [ ] Confirm SendGrid quota.
- [ ] Confirm Drive/Sheets permission.
- [ ] Chuẩn bị backup form.

### Trong event

- [ ] Mở Google Sheets monitor realtime.
- [ ] Check status mỗi 15-20 phút.
- [ ] Staff có quyền xem Drive folders.
- [ ] Có người phụ trách resend/regenerate manual.
- [ ] Nếu lỗi hàng loạt, tắt QR hoặc chuyển sang backup form.

### Sau event

- [ ] Review toàn bộ `FAILED`, `BOUNCED`, `OPENAI_*`.
- [ ] Resend manual cho case cần thiết.
- [ ] Export/import CRM nếu có consent phù hợp.
- [ ] Xóa/archive ảnh gốc theo retention policy.
- [ ] Ghi lại lỗi và cải tiến prompt/workflow.

---

## Phase 2 - Proper Build

Sau khi Phase 1 validate được nhu cầu, nên chuyển sang build chuẩn hơn.

### Frontend

- Next.js app mobile-first, brand ATS đầy đủ.
- Upload progress thật.
- Submission status page.
- Có thể preview poster nếu backend xử lý async job.

### Backend

- Next.js API routes hoặc FastAPI.
- Job queue cho OpenAI.
- PostgreSQL thay Google Sheets.
- Object storage như Cloudflare R2/S3 thay Drive.
- Idempotency key chuẩn theo `eventId + submissionId`.
- Admin resend/regenerate endpoint.

### Admin Dashboard

- Filter theo event/status.
- Preview original/poster.
- Resend email.
- Regenerate poster.
- Export CSV.
- Audit log.

### Quality / Safety

- Face/image quality pre-check nếu cần.
- Rate limit theo IP/event code.
- Consent/versioning.
- Retention automation.
- Monitoring + alert.

---

## Ước tính chi phí Phase 1

Chi phí OpenAI, SendGrid và quota thay đổi theo account/plan. Bảng dưới chỉ là estimate vận hành, phải verify trước event.

| Dịch vụ | Chi phí | Ghi chú |
| --- | --- | --- |
| HTML form tự host | Miễn phí | Dùng VPS có sẵn |
| n8n self-host | Miễn phí | Nếu VPS đã có |
| OpenAI Image API | Cần verify | Tính theo model/quality/size tại thời điểm event |
| SendGrid | Tùy plan | Kiểm tra daily/monthly quota |
| Google Drive | Miễn phí/tùy workspace | Ảnh gốc + poster có thể tốn vài MB mỗi học sinh |
| Google Sheets | Miễn phí/tùy workspace | Tạm dùng làm ops dashboard |

---

## Nguồn cần verify trước khi triển khai

- OpenAI Images guide: `https://platform.openai.com/docs/guides/images`
- OpenAI Images API reference: `https://platform.openai.com/docs/api-reference/images`
- SendGrid template + attachment behavior theo node n8n version đang dùng
- Google Drive permission behavior theo Google account/workspace ATS

---

_Document version 4.0 - Sửa luồng n8n để preserve binary, thêm consent/privacy, thêm dedupe trước OpenAI, sửa prompt giữ identity, dùng poster dọc 1024x1536, làm rõ Drive permission và checklist vận hành._
