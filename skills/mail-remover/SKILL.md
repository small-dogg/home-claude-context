---
name: mail-remover
description: POP3 메일 서버(localhost:8888)의 메일을 스캔하여 옵시디언에 제거 대상을 분류/정리하고, 사용자가 체크한 항목을 API로 삭제하는 스킬입니다.
---

# Mail Remover

POP3 연동 메일 애플리케이션(localhost:8888)의 메일을 스캔하여 4단계로 분류하고, 옵시디언에서 체크한 메일을 일괄 삭제하는 skill입니다.

## 주요 기능

1. **메일 스캔**: 서버의 전체 메일을 페이지 단위로 읽어 4단계 분류
2. **재스캔**: 마지막 스캔 이후 새로 도착한 메일만 추가 스캔
3. **선택 삭제**: 옵시디언에서 체크(`[x]`)한 메일을 API로 일괄 삭제
4. **이력 보존**: 삭제된 항목은 ~~취소선~~으로 표시하여 이력 유지

## 사용 방법

- `/mail-remover scan` - 메일 스캔 및 분류
- `/mail-remover delete` - 체크된 메일 삭제

## API 서버

- **Base URL**: `http://localhost:8888`
- **인증**: 없음 (로컬)

### API 엔드포인트

| Method | Path | 설명 |
|--------|------|------|
| GET | `/api/mail/servers` | 서버 목록 조회 |
| GET | `/api/mail/{serverName}/messages?page=0&size=20&sort=desc` | 메일 목록 (페이징, sort=asc\|desc) |
| GET | `/api/mail/{serverName}/messages/{messageId}` | 메일 상세 |
| DELETE | `/api/mail/{serverName}/messages` | 메일 일괄 삭제 (body: `{"messageIds": [...]}`) |

### API 응답 구조

```json
{
  "success": true,
  "data": { ... }
}
```

메일 목록 응답의 `data`는 페이지 응답 형태:
```json
{
  "content": [
    {
      "messageId": 142,
      "subject": "[광고] 특가 세일!",
      "sender": "sender@spam.com",
      "date": "2026-02-10",
      "size": 1024
    }
  ],
  "totalElements": 150,
  "totalPages": 8,
  "number": 0,
  "size": 20,
  "last": false
}
```

## 저장 구조

```
Obsidian Vault/
└── mails/
    └── 2026-02-11/
        ├── scan-result.md     # 분류별 체크리스트
        └── .state.md          # last_message_id 추적
```

## 분류 기준

| 등급 | 아이콘 | 판별 기준 |
|------|--------|----------|
| 삭제 강력 권장 | 🔴 | [광고], AD, 특가, 세일, 무료, 수신거부, 대량발송, 대출, 당첨 |
| 삭제 권장 | 🟠 | no-reply/noreply 발신자, notification, alert, 시스템 자동 메시지, 인증코드 |
| 삭제 검토 | 🟡 | newsletter, 구독, unsubscribe, 정기 발송, 주간/월간 |
| AI 추천 삭제 | 🤖 | Claude가 제목/발신자를 직접 판단하여 불필요하다고 판단한 메일 |
| 보존 권장 | 🟢 | 위 패턴 및 AI 추천 미해당 |

패턴 매칭 상세: `scripts/mail_helpers.py`의 `classify_mail()` 함수 참조.
AI 분류: Claude가 워크플로우 단계 6.5에서 직접 수행.

## 워크플로우

### 1. `/mail-remover scan` - 메일 스캔

**단계:**

1. 서버 목록을 조회합니다:

```bash
curl -s http://localhost:8888/api/mail/servers
```

2. 서버가 1개면 자동선택, **복수면 AskUserQuestion으로 선택**합니다:

```
AskUserQuestion을 사용하여 서버를 선택합니다:

  header: "메일 서버"
  question: "스캔할 메일 서버를 선택해주세요."
  options:
    - label: "{server1.name}"
      description: "{server1.host}:{server1.port}"
    - label: "{server2.name}"
      description: "{server2.host}:{server2.port}"
    (최대 4개, 초과 시 처음 4개만 표시)
```

3. 옵시디언에서 기존 상태를 확인합니다:

```python
# .state.md 확인
state = obsidian_get_file_contents(f"mails/{today}/.state.md")
# frontmatter에서 last_message_id 추출
```

4. **첫 스캔** (state가 없는 경우): page=0부터 마지막 페이지까지 **전체 순회**합니다:

```bash
# page 0
curl -s "http://localhost:8888/api/mail/{serverName}/messages?page=0&size=20&sort=desc"
# sort=desc: 최신 메일부터 조회 (기본값)
# 응답의 last == false 이면 다음 페이지 계속
# page 1
curl -s "http://localhost:8888/api/mail/{serverName}/messages?page=1&size=20&sort=desc"
# ... last == true 까지 반복
```

5. **재스캔** (state가 있는 경우): page=0부터 시작하되, `last_message_id`를 만나면 **즉시 중단**합니다:

```python
last_known_id = state['last_message_id']
new_mails = []
page = 0
stop = False

while not stop:
    response = curl_get(f"...?page={page}&size=20&sort=desc")
    for mail in response['content']:
        if mail['messageId'] <= last_known_id:
            stop = True
            break
        new_mails.append(mail)
    if response['last']:
        break
    page += 1
```

6. 각 메일의 제목(`subject`)과 발신자(`sender`)로 **패턴 매칭 분류**를 수행합니다:

```python
# scripts/mail_helpers.py의 classify_mail() 사용
category = classify_mail(mail['subject'], mail['sender'])
# 'red' | 'orange' | 'yellow' | 'green'
```

6.5. **AI 추천 삭제 분류**: 패턴 매칭으로 🟢(보존 권장)이 된 메일 중 Claude가 직접 제목/발신자를 분석하여 불필요하다고 판단한 메일을 `🤖 AI 추천 삭제` 카테고리로 재분류합니다.

**AI 판단 기준** (Claude가 자체 판단):
- 마케팅성 메일이지만 패턴에 없는 경우 (이벤트 안내, 리워드, 포인트 등)
- 오래된 서비스 알림 (앱 업데이트 안내, 약관 변경 등)
- 일회성 확인 메일 (가입 환영, 주문 확인 등 이미 용도가 끝난 메일)
- 자동 생성된 리포트/통계 메일
- SNS 알림 메일 (좋아요, 팔로우, 댓글 알림 등)

**중요**: Claude는 메일 제목과 발신자만 보고 판단합니다. 확실하지 않은 메일은 🟢(보존 권장)에 그대로 둡니다.

```python
# 패턴 매칭 후 green으로 분류된 메일들을 모아서 Claude가 직접 판단
green_mails = [m for m in classified_mails if m['category'] == 'green']

# Claude가 green_mails의 제목/발신자를 보고 AI 판단 수행
# 불필요하다고 판단한 메일은 category를 'ai'로 변경
for mail in green_mails:
    # Claude의 자체 판단 로직 (LLM 추론)
    # 예: "포인트 적립 안내" from "reward@service.com" → 'ai'
    # 예: "프로젝트 미팅 일정" from "colleague@company.com" → 'green' 유지
    if claude_judges_as_deletable(mail):
        mail['category'] = 'ai'
```

AI 추천 항목에는 추천 사유를 간단히 표기합니다:
```markdown
## 🤖 AI 추천 삭제
- [ ] `#150` **포인트 적립 안내** - reward@service.com (2026-02-11) `마케팅`
- [ ] `#146` **앱 업데이트 v3.2 안내** - update@app.com (2026-02-10) `서비스 안내`
```

7. **scan-result.md를 생성**합니다:

   - **첫 스캔**: `generate_scan_result()` 함수로 전체 생성
   - **재스캔**: `merge_new_mails_into_scan_result()` 함수로 기존 문서에 새 메일 추가

```python
# 첫 스캔
content = generate_scan_result(server_name, scanned_at, last_id, total, classified_mails)
obsidian_append_content(filepath=f"mails/{today}/scan-result.md", content=content)

# 재스캔 (기존 문서가 있는 경우)
existing = obsidian_get_file_contents(f"mails/{today}/scan-result.md")
updated = merge_new_mails_into_scan_result(existing, new_classified_mails, new_last_id, scanned_at)
obsidian_delete_file(f"mails/{today}/scan-result.md")
obsidian_append_content(filepath=f"mails/{today}/scan-result.md", content=updated)
```

8. **.state.md를 업데이트**합니다:

```python
state_content = generate_state_content(server_name, last_message_id, scanned_at)
# 기존 .state.md 삭제 후 재생성
obsidian_delete_file(f"mails/{today}/.state.md")  # 없어도 무시
obsidian_append_content(filepath=f"mails/{today}/.state.md", content=state_content)
```

9. **결과를 출력**합니다:

```
📬 메일 스캔 완료!

📊 서버: {serverName}
📧 스캔: {total}건
🆕 신규: {new_count}건 (재스캔 시)

🔴 삭제 강력 권장: {red_count}건
🟠 삭제 권장: {orange_count}건
🟡 삭제 검토: {yellow_count}건
🤖 AI 추천 삭제: {ai_count}건
🟢 보존 권장: {green_count}건

💾 저장: mails/{today}/scan-result.md
📝 옵시디언에서 삭제할 메일을 체크([x])한 후 '/mail-remover delete'를 실행하세요.
```

### 2. `/mail-remover delete` - 체크된 메일 삭제

**단계:**

1. 오늘 날짜의 **scan-result.md를 읽습니다**:

```python
content = obsidian_get_file_contents(f"mails/{today}/scan-result.md")
```

2. 파일이 없으면 에러를 출력합니다:

```
❌ 스캔 결과가 없습니다.
먼저 '/mail-remover scan'으로 메일을 스캔하세요.
```

3. **체크된 항목([x])에서 messageId를 추출**합니다:

```python
# scripts/mail_helpers.py의 extract_checked_message_ids() 사용
checked_ids = extract_checked_message_ids(content)
# 이미 취소선(~~)이 적용된 항목은 제외
# 패턴: - [x] `#142` (취소선 없는 것만)
```

이미 삭제된 항목(취소선 적용)은 제외합니다:

```python
already_deleted = []
for match in re.finditer(r'- \[x\]\s*~~`#(\d+)`', content):
    already_deleted.append(int(match.group(1)))
checked_ids = [mid for mid in checked_ids if mid not in already_deleted]
```

4. 체크된 항목이 없으면 안내합니다:

```
ℹ️ 삭제할 메일이 없습니다.
옵시디언에서 삭제할 메일을 체크([x])해주세요.
```

5. **frontmatter에서 서버명을 획득**합니다:

```python
fm = parse_frontmatter(content)
server_name = fm['server']
```

6. **DELETE API를 호출**합니다:

```bash
curl -s -X DELETE "http://localhost:8888/api/mail/{serverName}/messages" \
  -H "Content-Type: application/json" \
  -d '{"messageIds": [142, 145, 147]}'
```

7. 삭제 성공한 항목에 **취소선을 적용**합니다:

```python
# scripts/mail_helpers.py의 apply_strikethrough() 사용
updated = apply_strikethrough(content, deleted_ids)
```

변경 전:
```markdown
- [x] `#142` **[광고] 특가 세일!** - sender@spam.com (2026-02-10)
```

변경 후:
```markdown
- [x] ~~`#142` **[광고] 특가 세일!** - sender@spam.com (2026-02-10)~~ ✅ 삭제됨
```

8. **scan-result.md를 갱신**합니다:

```python
obsidian_delete_file(f"mails/{today}/scan-result.md")
obsidian_append_content(filepath=f"mails/{today}/scan-result.md", content=updated)
```

9. **결과를 출력**합니다:

```
🗑️ 메일 삭제 완료!

📊 서버: {serverName}
✅ 삭제: {count}건
  - #{142} [광고] 특가 세일!
  - #{145} 배송 완료 알림

💾 scan-result.md가 업데이트되었습니다.
```

## scan-result.md 포맷

```markdown
---
server: {serverName}
scanned_at: 2026-02-11T10:30:00
last_message_id: 150
total_scanned: 150
---

# Mail Scan - 2026-02-11

> Server: {serverName} | 스캔: 150건

---

## 🔴 삭제 강력 권장 (스팸/광고)
- [ ] `#142` **[광고] 특가 세일!** - sender@spam.com (2026-02-10)
- [x] ~~`#139` **무료 쿠폰 지급** - promo@deal.com (2026-02-09)~~ ✅ 삭제됨

## 🟠 삭제 권장 (알림/자동발송)
- [ ] `#145` **배송 완료 알림** - no-reply@delivery.com (2026-02-11)

## 🟡 삭제 검토 (뉴스레터/구독)
- [ ] `#143` **주간 뉴스레터 #52** - news@tech.com (2026-02-10)

## 🤖 AI 추천 삭제
- [ ] `#150` **포인트 적립 안내** - reward@service.com (2026-02-11) `마케팅`
- [ ] `#146` **앱 업데이트 v3.2 안내** - update@app.com (2026-02-10) `서비스 안내`

## 🟢 보존 권장 (개인/업무)
- [ ] `#148` **프로젝트 미팅 일정** - colleague@company.com (2026-02-11)

---
*마지막 업데이트: 2026-02-11 10:30*
```

## .state.md 포맷

```markdown
---
server: {serverName}
last_message_id: 150
updated_at: 2026-02-11T10:30:00
---
```

## 에러 처리

### API 서버 연결 실패
```
⚠️ 메일 서버(localhost:8888)에 연결할 수 없습니다.
서버가 실행 중인지 확인해주세요.
```

### 스캔 결과 없음
```
❌ 스캔 결과가 없습니다.
먼저 '/mail-remover scan'으로 메일을 스캔하세요.
```

### 체크된 메일 없음
```
ℹ️ 삭제할 메일이 없습니다.
옵시디언에서 삭제할 메일을 체크([x])해주세요.
```

### 삭제 API 실패
```
⚠️ 메일 삭제 중 오류가 발생했습니다.
서버 응답: {error_message}
```

### 메일이 0건
```
📭 메일함이 비어있습니다.
```

## MCP 도구 사용

| 작업 | 사용 도구 |
|------|-----------|
| 파일 생성 | `obsidian_append_content` |
| 파일 읽기 | `obsidian_get_file_contents` |
| 파일 삭제 | `obsidian_delete_file` |
| 디렉토리 조회 | `obsidian_list_files_in_dir` |
| API 호출 | `Bash` (curl) |
| 서버 선택 | `AskUserQuestion` |

## 주의사항

1. **API 호출은 전부 curl(Bash)을 사용**합니다 (WebFetch 사용하지 않음).
2. **서버 복수 시** AskUserQuestion으로 선택받습니다.
3. **삭제 후** 취소선(~~) 표시로 이력을 보존합니다.
4. **재스캔 시** 기존 scan-result.md에 새 메일만 추가합니다.
5. **날짜 기준**: 모든 조회/조작은 **오늘 날짜** 폴더 기준입니다.
6. **이미 삭제된 항목**(취소선 적용)은 재삭제하지 않습니다.
