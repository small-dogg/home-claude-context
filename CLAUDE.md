# Global Claude Code Context

## 응답 규칙
- 모든 응답은 **한국어**로 작성할 것
- 기술 용어와 코드 식별자는 원문 그대로 유지

## 소스코드 변경 규칙
- 소스코드 수정 시 반드시 **Before & After**를 보여줄 것
- 변경 내용을 확인시킨 뒤 사용자의 **진행 여부 승인**을 받고 진행할 것
- 승인 없이 코드를 수정하지 말 것

## 프롬프트 로깅
- 모든 프롬프트와 결과는 기록할 것
- Obsidian MCP 연동 시: Obsidian 볼트의 `prompt/{YYYY-MM-DD}/{세션 제목}/`
- Obsidian MCP 미연동 시: `~/prompt/{YYYY-MM-DD}/{세션 제목}/`
- 세션 제목은 작업 내용을 요약하여 Claude가 자동 생성
- 파일 형식: Markdown

## 사용자 프로필
- 이름: Jerry
- 주요 언어: Java, TypeScript
- 선호 IDE: IntelliJ IDEA
- OS: macOS

## 코딩 컨벤션
- 변수/함수명: camelCase
- 클래스명: PascalCase
- 상수: UPPER_SNAKE_CASE
- 들여쓰기: 2 spaces (JS/TS), 4 spaces (Java)
- 문자열: single quote (JS/TS)
- 주석(comment)은 반드시 **영어**로 작성할 것

## Git 커밋 규칙
- Conventional Commits 형식 사용
- 형식: `type(scope): description`
- type: feat, fix, docs, style, refactor, test, chore
- description은 한국어 허용
- 예시: `feat(auth): 소셜 로그인 추가`

## 금지 사항
- 사용자 승인 없이 파일 삭제 금지
- .env, credentials 등 민감 파일 커밋 금지
- force push 금지 (명시적 요청 제외)
- 불필요한 주석, docstring 자동 추가 금지
- 과도한 에러 핸들링/추상화 금지
