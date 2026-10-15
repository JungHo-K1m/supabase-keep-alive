# supabase-keep-alive

Supabase 무료 플랜 프로젝트가 비활성으로 정지되지 않도록 GitHub Actions가 매일 2회(06:00, 18:00 UTC)
각 프로젝트의 `public.keep_alive` 테이블에 upsert를 보낸다.

- 워크플로: `.github/workflows/supabase-keep-alive-multi.yml`
- 대상 목록: 저장소 시크릿 `SUPABASE_PROJECTS` (JSON 배열) — **저장 후 다시 볼 수 없으므로** 아래 표를 항상 최신으로 유지할 것
- 정지 기준(공식 문서): 지난 7일간 매일 몇 건 수준의 DB 활동이 없으면 정지 → 주 1~2회 ping으로는 부족

## 등록된 프로젝트 (2026-10-15 기준)

| name | url | 비고 |
|---|---|---|
| flowgence-mvp | https://biouwhfczktkfdkfxpxt.supabase.co | |
| safety-report | https://kaqdxgevgucfdqrxrshk.supabase.co | |
| solar-agri | https://khnhmpvgnxdxhcqxlxkp.supabase.co | |
| dream-bridge | https://hjxqnikqvtvfqzeltpcm.supabase.co | |
| health-hero | https://uasphzqbluxctnukoazy.supabase.co | |
| connect-h | https://dffsaricmhtkjcamiucz.supabase.co | 2026-10-15 추가 (누락으로 정지됐던 프로젝트) |

제외: digiray (gzligjvcmdrneodfhkmb) — 더 이상 사용하지 않아 정지 상태로 둠.

## 프로젝트 추가 절차

1. 대상 프로젝트 Supabase 대시보드 → SQL Editor에서 실행:

   ```sql
   create table if not exists public.keep_alive (
     id        integer primary key,
     last_ping timestamptz not null default now()
   );
   alter table public.keep_alive enable row level security;  -- service_role은 RLS 우회, 정책 불필요
   insert into public.keep_alive (id, last_ping) values (1, now()) on conflict (id) do nothing;
   ```

2. 대시보드 → Settings → API에서 `service_role` 키 복사.
3. GitHub → Settings → Secrets and variables → Actions → `SUPABASE_PROJECTS` 수정.
   수정 화면은 항상 비어 있으므로 **전체 목록을 다시 입력**해야 한다:

   ```json
   [
     {"name":"flowgence-mvp","url":"https://biouwhfczktkfdkfxpxt.supabase.co","key":"<service_role_key>"},
     {"name":"safety-report","url":"https://kaqdxgevgucfdqrxrshk.supabase.co","key":"<service_role_key>"},
     {"name":"solar-agri","url":"https://khnhmpvgnxdxhcqxlxkp.supabase.co","key":"<service_role_key>"},
     {"name":"dream-bridge","url":"https://hjxqnikqvtvfqzeltpcm.supabase.co","key":"<service_role_key>"},
     {"name":"health-hero","url":"https://uasphzqbluxctnukoazy.supabase.co","key":"<service_role_key>"},
     {"name":"connect-h","url":"https://dffsaricmhtkjcamiucz.supabase.co","key":"<service_role_key>"}
   ]
   ```

4. 위 표에 프로젝트를 추가하고 커밋.
5. Actions → "Supabase Keep Alive (Multi-Project)" → Run workflow로 수동 실행 후 실행 요약 표에서 전부 ✅인지 확인.

## 결과 확인 / 장애 진단

- 실행 요약(Step Summary) 표에 프로젝트별 상태가 표시된다.
  - `401/403`: service_role 키 오류
  - `404`: `keep_alive` 테이블 없음 또는 url 오류
  - `000`: DNS/네트워크 실패 — 프로젝트가 이미 정지됐을 가능성 큼
- 한 프로젝트라도 실패하면 워크플로가 실패로 표시된다.
- 정지된 프로젝트는 Supabase 대시보드에서 Restore 후, 7일 이내에 시크릿에 등록돼 있는지 확인.
- heartbeat 커밋(`keepalive-heartbeat.txt`)은 저장소 60일 비활성으로 schedule이 꺼지는 것을 막기 위한 것으로, 주 1회만 생성된다.
