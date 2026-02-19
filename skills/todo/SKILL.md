---
name: todo
description: 옵시디언 볼트 기반 할일 관리. 할일 생성, 완료, 삭제를 지원합니다.
argument-hint: "할일 제목" 또는 complete/delete {번호}
---

# Obsidian Todo Manager

옵시디언 볼트의 todos/{오늘날짜}/ 디렉토리에 할일을 관리합니다.

## 사용법

- /todo "할일 제목" → 새 할일 생성
- /todo complete {번호} → 할일 완료 처리
- /todo delete {번호} → 할일 삭제

사용자의 입력: $ARGUMENTS

## 인수 파싱 규칙

$ARGUMENTS를 분석하여 다음 중 하나를 실행합니다:

1. "complete {번호}" 로 시작하면 → 완료 처리 워크플로우
2. "delete {번호}" 로 시작하면 → 삭제 워크플로우
3. 그 외 → 생성 워크플로우 (나머지 텍스트가 할일 제목)

## 저장 구조

    Obsidian Vault/
    └── todos/
        └── {YYYY-MM-DD}/           (오늘 날짜 폴더)
            ├── summary.md          (중요도별 체크리스트, 자동 생성)
            ├── 1-제목.md
            └── 2-제목.md

- 폴더 기준: 오늘 날짜 (todos/{오늘날짜}/)
- due_date는 frontmatter에 메타데이터로 저장
- 디렉토리가 없으면 obsidian_append_content가 자동 생성

## 번호(ID) 체계

- 날짜별 순차 번호 (1부터 시작)
- 파일명: {ID}-{sanitized-title}.md (예: 1-보고서-작성.md)
- max(기존 ID) + 1로 할당 (삭제된 번호 재사용 안 함)

---

## 워크플로우 1: 할일 생성

조건: $ARGUMENTS가 complete/delete로 시작하지 않을 때

단계 1. $ARGUMENTS에서 제목을 추출합니다 (앞뒤 따옴표 제거).

단계 2. AskUserQuestion 도구로 마감일과 중요도를 대화형으로 질문합니다.

질문 1 - 마감일:
  header: "마감일"
  question: "'{제목}' 할일의 마감일을 선택해주세요."
  options:
    - label: "오늘", description: "오늘 안에 완료"
    - label: "내일", description: "내일까지 완료"
    - label: "이번 주", description: "이번 주 내 완료"

질문 2 - 중요도:
  header: "중요도"
  question: "할일의 중요도를 선택해주세요. (1: 낮음 ~ 5: 높음)"
  options:
    - label: "3 - 보통 (Recommended)", description: "일반적인 할일"
    - label: "5 - 매우 높음", description: "최우선 처리 필요"
    - label: "4 - 높음", description: "빠른 시일 내 처리"
    - label: "2 - 낮음", description: "여유 있을 때 처리"

  "이번 주"는 다가오는 일요일 날짜로 계산합니다.
  "직접 입력" 선택 시 YYYY-MM-DD 형식으로 입력받습니다.

단계 3. obsidian_list_files_in_dir("todos/{today}/") 로 기존 파일 확인하고 다음 ID를 계산합니다.
  - 파일이 없으면 ID = 1
  - 파일명에서 숫자- 패턴을 찾아 최대값 + 1

단계 4. 제목 sanitize: 특수문자 제거, 공백을 하이픈으로 변환
  - 파일명: {ID}-{sanitized_title}.md

단계 5. 할일 파일 내용을 다음 형식으로 생성합니다:

    ---
    id: {ID}
    title: "{제목}"
    priority: {중요도}
    due_date: {마감일}
    status: pending
    created_at: {현재시간 ISO형식}
    completed_at: null
    ---

    # {제목}

    - **번호**: #{ID}
    - **중요도**: {별이모지} ({중요도}/5)
    - **마감일**: {마감일}
    - **상태**: 대기중
    - **생성일**: {현재시간}

단계 6. obsidian_append_content 도구로 todos/{today}/{filename} 경로에 저장합니다.

단계 7. summary.md 재생성 (아래 "summary.md 재생성" 섹션 참조)

단계 8. 완료 메시지를 출력합니다:

    ✅ 할일이 생성되었습니다!

    📋 #{ID} {제목}
    ⭐ 중요도: {별이모지} ({중요도}/5)
    📅 마감일: {마감일}
    💾 저장 위치: todos/{today}/{filename}

---

## 워크플로우 2: 할일 완료

조건: $ARGUMENTS가 "complete {번호}" 패턴일 때

단계 1. 번호를 파싱합니다.

단계 2. obsidian_list_files_in_dir("todos/{today}/") 로 파일 목록을 조회합니다.

단계 3. "{번호}-" 로 시작하는 파일을 찾습니다.
  - 못 찾으면 에러: ❌ #{번호}번 할일을 찾을 수 없습니다. '/todos' 로 현재 할일 목록을 확인하세요.

단계 4. obsidian_read_file("todos/{today}/{파일명}") 로 내용을 읽습니다.

단계 5. frontmatter에서 status를 확인합니다.
  - 이미 completed이면: ℹ️ #{번호} {제목}은(는) 이미 완료된 할일입니다.

단계 6. status를 completed로, completed_at을 현재 시간으로 변경한 새 내용을 생성합니다.
  - 본문의 상태도 "완료"로 변경합니다.
  - 완료일 항목을 추가합니다.

단계 7. 기존 파일을 삭제하고 재생성합니다:
  - obsidian_delete_file("todos/{today}/{파일명}")
  - obsidian_append_content(filepath="todos/{today}/{파일명}", content=새내용)

단계 8. summary.md를 재생성합니다.

단계 9. 완료 메시지: ✅ #{번호} {제목} 할일을 완료했습니다!

---

## 워크플로우 3: 할일 삭제

조건: $ARGUMENTS가 "delete {번호}" 패턴일 때

단계 1. 번호를 파싱합니다.

단계 2. obsidian_list_files_in_dir("todos/{today}/") 로 파일 목록을 조회합니다.

단계 3. "{번호}-" 로 시작하는 파일을 찾습니다.
  - 못 찾으면 에러: ❌ #{번호}번 할일을 찾을 수 없습니다. '/todos' 로 현재 할일 목록을 확인하세요.

단계 4. obsidian_delete_file("todos/{today}/{파일명}") 로 삭제합니다.

단계 5. summary.md를 재생성합니다.

단계 6. 완료 메시지: 🗑️ #{번호} {제목} 할일을 삭제했습니다.

---

## summary.md 재생성 (공통)

모든 변경(생성/완료/삭제) 후 실행합니다.

단계 1. obsidian_list_files_in_dir("todos/{today}/") 로 모든 파일을 조회합니다.

단계 2. summary.md를 제외한 각 파일을 obsidian_read_file로 읽어서 frontmatter를 파싱합니다.

단계 3. 할일이 0개면 기존 summary.md만 삭제하고 종료합니다.

단계 4. summary.md 내용을 생성합니다 (중요도 내림차순, 같은 중요도 내 ID 순):

    # {날짜} 할일 요약

    > 총 {N}개 | 완료 {M}개 | 남은 할일 {N-M}개

    ---

    ## ⭐⭐⭐⭐⭐ 우선순위 5
    - [x] #1 보고서 작성 (마감: 2026-02-09) ✅
    - [ ] #4 이메일 보내기 (마감: 2026-02-09)

    ## ⭐⭐⭐⭐ 우선순위 4
    - [ ] #2 회의 준비 (마감: 2026-02-10)

    ---

    *마지막 업데이트: {현재시간}*

단계 5. 기존 summary.md를 삭제 후 재생성합니다:
  - obsidian_delete_file("todos/{today}/summary.md")
  - obsidian_append_content(filepath="todos/{today}/summary.md", content=...)

## 에러 처리

- 할일 없음: 📭 오늘의 할일이 없습니다. '/todo "할일 제목"' 으로 새 할일을 추가하세요.
- 잘못된 번호: ❌ #{번호}번 할일을 찾을 수 없습니다. '/todos' 로 현재 할일 목록을 확인하세요.
- 이미 완료: ℹ️ #{번호} {제목}은(는) 이미 완료된 할일입니다.
- 옵시디언 연결 실패: ⚠️ 옵시디언에 연결할 수 없습니다. MCP 서버 설정을 확인해주세요.

## MCP 도구

| 작업 | 도구 |
|------|------|
| 파일 생성 | obsidian_append_content |
| 파일 읽기 | obsidian_read_file |
| 파일 삭제 | obsidian_delete_file |
| 디렉토리 조회 | obsidian_list_files_in_dir |
