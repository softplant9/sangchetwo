# 클라우드 동기화 설정 (Supabase)

로그인하면 정답/헷갈림/오답 채점 기록이 계정에 저장되어 **다른 브라우저·기기에서도 그대로 이어집니다.**
설정 전(placeholder 상태)에는 기존처럼 이 브라우저(localStorage)에만 저장됩니다.

---

## 1. Supabase 프로젝트 만들기
1. https://supabase.com 가입 후 **New project** 생성 (무료 플랜).
2. 프로젝트가 준비되면 **Project Settings → Data API(또는 API)** 에서 **Project URL** 복사.
   - 형태: `https://<프로젝트참조>.supabase.co`

## 2. 키 (이미 발급됨)
- `env` 파일에 저장된 키:
  - `supabase_default_key` (`sb_publishable_...`) → **공개용**. 코드에 넣음.
  - `supabase_secret_key` (`sb_secret_...`) → **서버 전용. 절대 깃/클라이언트에 넣지 않음.**
- `env` 파일은 `.gitignore`에 등록되어 커밋되지 않습니다.

## 3. supabase-config.js 채우기
`supabase-config.js` 에서 `SUPABASE_URL` 을 1단계에서 복사한 Project URL 로 교체합니다.
```js
window.SUPABASE_URL = 'https://<프로젝트참조>.supabase.co';
window.SUPABASE_ANON_KEY = 'sb_publishable_11J2U67BNtQBGtzMEcrSFg_oMFb6VlT'; // 이미 입력됨
```
(publishable 키는 공개되어도 안전 — 데이터는 아래 RLS로 보호)

## 4. 테이블 + 보안규칙(RLS) 만들기
Supabase 대시보드 **SQL Editor** 에서 아래를 실행:
```sql
create table if not exists public.quiz_marks (
  user_id    uuid not null references auth.users(id) on delete cascade,
  qhash      text not null,
  mark       text not null check (mark in ('correct','partial','incorrect')),
  updated_at timestamptz not null default now(),
  primary key (user_id, qhash)
);

alter table public.quiz_marks enable row level security;

-- 본인 행만 읽기/쓰기 가능
create policy "own rows - select" on public.quiz_marks
  for select using (auth.uid() = user_id);
create policy "own rows - insert" on public.quiz_marks
  for insert with check (auth.uid() = user_id);
create policy "own rows - update" on public.quiz_marks
  for update using (auth.uid() = user_id) with check (auth.uid() = user_id);
create policy "own rows - delete" on public.quiz_marks
  for delete using (auth.uid() = user_id);
```

## 5. 이메일+비밀번호 로그인 켜기
- **Authentication → Providers → Email** 활성화 (기본 켜져 있음).
- 빠른 사용을 원하면 **Confirm email** 을 끄면 가입 즉시 로그인됩니다.
  (켜두면 가입 시 받은 인증 메일을 클릭해야 로그인 가능)
- **Authentication → URL Configuration → Site URL** 에 배포 주소
  `https://softplant9.github.io/sangchetwo/` 추가.

## 6. 배포
`supabase-config.js` 수정 후 커밋·푸시하면 사이트 상단에 로그인 바가 나타납니다.
- **가입**(최초 1회) → 자동 로그인 → 기존 로컬 채점이 클라우드로 업로드됨
- 다른 기기에서 같은 계정으로 **로그인** → 채점 기록이 그대로 따라옵니다

---

### 동작 방식 요약
- 로그인 상태: 채점할 때마다 즉시 클라우드 저장. 로그인 시 클라우드 ↔ 로컬 병합.
- 로그아웃/미설정: 이 브라우저에만 저장(localStorage).
- 충돌 시: 클라우드 값 우선, 로컬에만 있던 기록은 클라우드로 보존 업로드.
