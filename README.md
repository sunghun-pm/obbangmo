# 오빵뭐 (오늘 빵 뭐 남았지?)

동네 빵집이 마감 전에 남은 빵을 **랜덤 마감박스**로 올리면, 근처 사람이 할인가로 예약하고 매장에서 직접 픽업하는 모바일 웹앱입니다.

- 사장님: 매장 등록 → 박스 상품 등록(한 번) → 매일 **버튼 하나로 오늘 판매 올리기** → 픽업 코드로 수령 처리
- 사용자: 근처 박스 지도·목록 → 상세(구성·알레르기) → 예약·결제 → 4자리 픽업 코드·QR 티켓

스택: React 19 + Vite + TypeScript + Tailwind v4 / Supabase(Postgres·Auth·Realtime·Storage·Edge Functions) / 토스페이먼츠 결제위젯 v2 (키가 없으면 데모 결제)

## 화면

| 홈 | 상세 | 예약 확인 | 픽업 티켓 |
| --- | --- | --- | --- |
| ![](docs/screenshots/04-home.png) | ![](docs/screenshots/05-detail.png) | ![](docs/screenshots/06-checkout.png) | ![](docs/screenshots/07-ticket.png) |

| 매장 등록 | 상품 등록 검증 | 오늘 판매 | 예약자·픽업 처리 |
| --- | --- | --- | --- |
| ![](docs/screenshots/01-store-setup.png) | ![](docs/screenshots/02-product-validation.png) | ![](docs/screenshots/03-owner-today.png) | ![](docs/screenshots/08-owner-orders.png) |

## 설계에서 설명할 만한 것

**재고는 DB 함수 안에서만 바뀐다.** 앱은 `listings`·`reservations`에 직접 쓰지 못하고(RLS + 권한 회수), `reserve`·`open_listing`·`confirm_pickup` 같은 Postgres 함수만 호출합니다.

**마지막 1개 동시 예약.** `reserve()`는 조건부 UPDATE 한 번으로 재고를 차감합니다.

```sql
update listings set qty_left = qty_left - p_qty
 where id = p_listing_id and qty_left >= p_qty and closed_at is null and now() < pickup_end
returning * into l;
if not found then raise exception 'SOLD_OUT'; end if;
```

행 잠금이 걸린 상태에서 조건을 다시 평가하므로 동시에 들어와도 한 명만 성공합니다. `scripts/race-test.mjs`로 20명이 동시에 마지막 1개를 누르는 상황을 재현하면 **성공 1 / SOLD_OUT 19 / 기타 0**이 나옵니다.

**선점 → 결제 → 확정.** 예약은 먼저 `held`(10분 선점)로 재고를 잡고, 결제 승인은 서버(Edge Function)에서만 처리해 `booked`로 바꿉니다. 결제하지 않은 선점은 pg_cron이 1분마다 만료시키고 재고를 되돌립니다.

```
held ──결제 승인──▶ booked ──픽업 코드 확인──▶ picked
 │                  ├─ 픽업 시작 전 취소 ──▶ canceled (재고 복구·환불)
 └─ 10분 미결제 ─▶ expired (재고 복구)
                    └─ 픽업 종료 시각 경과 ──▶ no_show
```

**보안 점검 (실제 API로 시도해 모두 차단 확인)**: 결제 없이 `mark_booked` 호출, 예약 상태 직접 수정, 재고 직접 수정, 예약 행 직접 생성, 남의 상품으로 판매 열기, 금액을 바꿔 결제 확정, 다른 사람 예약 조회.

## 폴더

```
src/                    앱 (pages/: 사용자, pages/owner/: 사장님)
supabase/migrations/    DB 스키마·권한·함수·예시 데이터·자동 상태 전환 (순서대로 실행)
supabase/functions/     Edge Functions: confirm-payment, cancel-reservation
scripts/race-test.mjs   동시 예약 테스트
tests/e2e.spec.ts       Playwright 핵심 시나리오 3개
```

---

## 배포하기 (무료 요금제 기준, 약 30분)

준비물: Supabase 계정, Vercel 계정, GitHub 계정 (모두 무료). 토스페이먼츠·카카오맵 키는 선택입니다.

### 1. Supabase 프로젝트 만들기

1. [supabase.com](https://supabase.com) → New project. 지역은 **Northeast Asia (Seoul)** 를 고릅니다.
2. 왼쪽 **SQL Editor**에서 `supabase/migrations/` 파일 4개를 **이름 순서대로** 하나씩 붙여 넣고 Run 합니다.
   - `..._schema.sql` → `..._demo_data.sql` → `..._state_jobs.sql` → `..._harden.sql`
   - 세 번째 파일이 pg_cron 오류를 내면 **Database → Extensions**에서 `pg_cron`을 켠 뒤 다시 실행하세요.
3. **Authentication → Sign In / Providers → Email**에서 데모용으로 **Confirm email을 끕니다** (켜 두면 가입 후 메일 인증이 필요해요).
4. **Project Settings → API**에서 `Project URL`과 `anon public` 키를 메모합니다.

### 2. Edge Functions 올리기 (웹에서 가능)

1. **Edge Functions → Deploy a new function → Via Editor**
2. 이름 `confirm-payment`, 내용은 `supabase/functions/confirm-payment/index.ts` 전체를 붙여 넣고 Deploy.
3. 같은 방법으로 `cancel-reservation`도 올립니다.
4. **Edge Functions → Secrets**에 추가합니다.
   - `DEMO_PAYMENTS` = `true` (토스 키 없이 데모 결제 허용)
   - 토스 테스트 결제를 쓰려면 `TOSS_SECRET_KEY` = 토스 개발자센터의 **테스트 시크릿 키**

> CLI를 쓴다면: `npx supabase login && npx supabase link --project-ref <ref> && npx supabase db push && npx supabase functions deploy confirm-payment cancel-reservation`

### 3. GitHub에 올리기

GitHub에서 새 저장소를 만들고 **Add file → Upload files**로 이 폴더의 내용을 끌어다 놓습니다 (`node_modules`는 없어도 됩니다).

### 4. Vercel로 배포

1. [vercel.com](https://vercel.com) → Add New → Project → 방금 만든 저장소 Import
2. Framework는 Vite로 자동 인식됩니다. **Environment Variables**에 추가:
   - `VITE_SUPABASE_URL` = 1-4의 Project URL
   - `VITE_SUPABASE_ANON_KEY` = 1-4의 anon public 키
   - (선택) `VITE_TOSS_CLIENT_KEY` = 토스 **테스트 클라이언트 키** (결제위젯용)
   - (선택) `VITE_KAKAO_MAP_KEY` = 카카오 JavaScript 키. Kakao Developers → 플랫폼 → Web에 Vercel 주소를 등록해야 지도가 뜹니다. 없으면 간이 지도가 나옵니다.
3. Deploy → 발급된 주소로 접속.

### 5. 시연 준비

- 사장님 계정 1개, 사용자 계정 1개를 만들어 두고 README나 포트폴리오에 적어 두면 평가자가 양쪽을 바로 체험할 수 있어요.
- 예시 매장 5곳의 판매는 매일 0시 5분(한국 시간)에 자동으로 새로 만들어집니다.

## 로컬 개발

```bash
npm install
cp .env.example .env.local   # 값 채우기
npm run dev
npm run test:e2e             # 앱이 떠 있는 상태에서
SUPABASE_URL=... SUPABASE_ANON_KEY=... SUPABASE_SERVICE_ROLE_KEY=... npm run test:race
```

## 알려진 한계 (MVP)

- 결제는 테스트 모드(또는 데모)만 지원합니다. 정산·수수료·세금계산서는 없습니다.
- 위치는 망원역 기준으로 고정입니다(거리 계산·지도 중심).
- 매장 위치는 위도·경도를 직접 입력합니다.
- 실제 운영 전에는 통신판매중개업 신고, 식품 표시 규정 검토가 필요합니다.
