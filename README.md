# 제주도 여행 수요조사

GitHub Pages에서 화면을 무료로 제공하고, Supabase 무료 플랜에서 실시간 집계와 안전한 투표 처리를 담당하는 정적 웹 앱입니다.

## 구현된 동작

- 이름과 분류(러버 / CD / MTF / TG)를 입력해 투표합니다. 이름과 분류는 공개 참여자 목록에 실시간으로 표시됩니다.
- 그래프는 **러버(파랑)** 와 **CD·MTF·TG 합계(분홍)** 를 전체 인원과 비율로 나타냅니다.
- 사용자는 투표할 때 쓴 이름과 4자 이상인 본인 비밀번호로 자기 투표를 삭제할 수 있습니다.
- 공개 화면에는 관리 기능을 표시하지 않습니다. 관리자는 주소 끝에 `#admin`을 붙인 뒤 대상을 선택하고 마스터 비밀번호로 이상 데이터를 삭제할 수 있습니다.
- 일반 투표 비밀번호는 각 투표마다 salt를 넣은 **bcrypt cost 12** 해시로만 데이터베이스에 저장됩니다. 원문 비밀번호나 해시는 공개 API에서 읽을 수 없습니다.
- 마스터 비밀번호는 소스·GitHub·데이터베이스에 넣지 않고, bcrypt 해시만 Supabase Edge Function의 비밀 환경변수에 저장합니다.

## 배포 순서

### 1. Supabase 프로젝트 만들기

1. [Supabase](https://supabase.com/dashboard)에서 새 무료 프로젝트를 만듭니다.
2. 프로젝트의 **SQL Editor**에서 [`supabase/migrations/20260910000100_create_travel_vote.sql`](supabase/migrations/20260910000100_create_travel_vote.sql) 전체를 실행합니다. 이 작업은 공개 참여 목록과 집계 테이블만 읽기 가능하게 하고, 비밀번호가 든 테이블은 브라우저에서 전혀 읽을 수 없게 합니다.
3. **Project Settings → API Keys**에서 Project URL과 publishable key(또는 legacy anon key)를 복사합니다.
4. [`site/config.js`](site/config.js)의 빈 문자열 두 곳에 각각 입력합니다. 이 publishable key는 웹 앱에 포함되어도 되는 공개 키입니다. `service_role` 키는 절대 여기에 넣지 마세요.

### 2. 비밀값 설정 및 Edge Function 배포

먼저 로컬 bcrypt 도구로 마스터 비밀번호의 cost 12 해시를 만드세요. 마스터 비밀번호 원문은 이 저장소·SQL·커밋 메시지 어디에도 적지 마세요. 그 해시를 Supabase Dashboard의 **Edge Functions → Secrets**에 다음과 같이 추가합니다.

| 이름 | 값 |
| --- | --- |
| `VOTE_MASTER_PASSWORD_HASH` | 마스터 비밀번호의 bcrypt cost 12 해시 (`$2...`로 시작) |
| `ALLOWED_ORIGIN` | GitHub Pages의 origin. 예: `https://<GitHub-아이디>.github.io` |
| `RATE_LIMIT_PEPPER` | 길고 무작위인 문자열 (IP 기반 요청 제한용) |

마스터 비밀번호 원문과 해시를 코드나 SQL 파일에 쓰지 않는 것이 핵심입니다. Supabase CLI를 연결한 후 다음을 실행해 함수를 배포합니다.

```powershell
supabase link --project-ref <프로젝트-REF>
supabase functions deploy travel-vote --no-verify-jwt
```

`--no-verify-jwt`는 로그인 없는 공개 투표를 받기 위한 설정입니다. 함수는 허용한 GitHub Pages 출처만 CORS로 응답하며, 투표·삭제 요청에는 IP 해시 기반 시간당 제한도 적용합니다. 실제 보안 경계는 비밀번호 검증, Edge Function의 비밀값, 데이터베이스 RLS입니다.

### 3. GitHub Pages 무료 배포

1. 이 폴더를 GitHub 저장소의 `main` 브랜치에 올립니다.
2. GitHub 저장소의 **Settings → Pages → Build and deployment**에서 **GitHub Actions**를 선택합니다.
3. `main` 브랜치에 push하면 [`.github/workflows/deploy-pages.yml`](.github/workflows/deploy-pages.yml)이 `site` 폴더를 Pages로 배포합니다.
4. 실제 Pages URL이 정해지면 `ALLOWED_ORIGIN`에 주소의 origin만 입력합니다. 예를 들어 `https://<GitHub-아이디>.github.io/<저장소-이름>/` 페이지라면 값은 경로 없이 `https://<GitHub-아이디>.github.io`입니다. 끝 슬래시는 넣지 않습니다.

## 보안 메모

SHA-256은 빠른 해시라서 사람이 정한 비밀번호 저장용으로 적합하지 않습니다. 이 앱은 bcrypt의 의도적인 연산 비용과 투표별 salt를 사용해 유출된 해시의 대입 공격 비용을 높입니다. 마스터 비밀번호도 bcrypt 해시만 배포 플랫폼의 비밀 환경변수로 격리합니다. 마스터 비밀번호가 이미 대화나 다른 곳에 노출됐다면, 실제 공개 운영 전에는 더 길고 고유한 값으로 교체하는 편이 안전합니다.

## 운영 전 확인

- 이름이 공개 목록에 보이므로, 실제 사용 전 참가자에게 공개 범위를 고지하세요.
- `site/config.js`에는 publishable key만 넣고 service-role key, 비밀번호, `RATE_LIMIT_PEPPER`는 절대 넣지 마세요.
- Supabase Dashboard의 Database → Replication에서 `vote_feed`, `vote_stats`가 Realtime publication에 추가됐는지 확인하세요. 마이그레이션이 이를 자동으로 처리합니다.
