# 공급자 연결과 운영 설정
버전 3.0 | 문서 검토 기준일 2026-10-22 | 실제 키/호출/단가 검증 미수행.

## 1. 현재 확인 상태
| 플랫폼/surface | TikHub | JustOneAPI | 초기 경로 |
|---|---|---|---|
| 微博 | 검색/상세/댓글/프로필/트렌드 등 안내 | 관련 기능 문서 있음 | 기능별 시험 후 기본 공급자 결정 |
| 小红书 | 공개 노트/검색/댓글/프로필 등 안내 | 검색/상세/댓글/답글 문서 있음 | 기능별 시험 후 기본 공급자 결정 |
| 豆瓣 movie | 이번 기획에서 지원 확정 못 함 | 영화 상세/댓글/리뷰 상세 문서 확인 | 필요할 때 해당 기능만 검증 |
| 豆瓣 group | 미확인 | 미확인 | manual_import, 향후 adapter |

미확인은 미지원 단정이 아니다. 최신 문서·계정 지원을 확인한다.
공급자 계정이 유료라는 사실은 필요한 범위의 무제한 조회 권한을 의미하지 않는다.
두 공급자가 같은 상위 데이터를 사용할 수 있으므로 독립된 원천 증거 두 개로 계산하지 않는다.
수집 합집합도 전체 플랫폼 데이터라고 부르지 않는다.

## 2. 확인한 문서
- TikHub 微博: https://tikhub.io/weibo-api
- TikHub advanced search: https://docs.tikhub.io/381269425e0
- TikHub realtime search: https://docs.tikhub.io/381269427e0
- TikHub comprehensive: https://docs.tikhub.io/410358109e0
- TikHub 小红书: https://tikhub.io/xiaohongshu-api
- TikHub 댓글: https://docs.tikhub.io/420136394e0
- TikHub 답글: https://docs.tikhub.io/420748830e0
- JustOneAPI 微博: https://docs.justoneapi.com/en/api/weibo/
- JustOneAPI 微博 검색 V2: https://docs.justoneapi.com/en/api/weibo/keyword-search-v2
- JustOneAPI 小红书: https://docs.justoneapi.com/en/api/xiaohongshu-rednote/
- JustOneAPI 小红书 검색 V2: https://docs.justoneapi.com/en/api/xiaohongshu-rednote/note-search-v2
- JustOneAPI 豆瓣 상세: https://docs.justoneapi.com/en/api/douban-movie/subject-details-v1
- JustOneAPI 豆瓣 댓글: https://docs.justoneapi.com/en/api/douban-movie/comments-v1
- JustOneAPI 豆瓣 리뷰: https://docs.justoneapi.com/en/api/douban-movie/review-details-v1
경로/응답을 이 목록만 보고 추측 구현하지 않는다. 실제 최신 operation 문서 확인 필요.
참고 방법론:
https://ojs.aaai.org/index.php/ICWSM/article/view/18075
https://link.springer.com/article/10.1007/s42001-024-00260-z
연구는 설계 참고이며 CPEL의 웨이보/小红书/豆瓣 정확도 입증 아님.

## 3. 기능 검증 기록
provider,platform,surface,capability,docs_url,checked_at,status,
auth_scheme,endpoint/method,schema_version,pagination,time_filter_resolution,
sorts,cache_behavior,time_precision,observed_max_page,rate_limit_source,
unit_price,currency,billing_rule,actual_test_receipt_id,limitations.
status=documented/tested/unknown/unsupported/temporarily_unavailable.
권한 실패/미문서 상태는 표에 남기고 fake empty response로 대체하지 않는다.
fixture는 실제 response의 비밀 제거 최소본, 구조 변경마다 계약 테스트.
query token·signed URL·개인 연락처 등은 fixture에서 제거.
샘플 자료에는 실제 운영 API 응답임과 synthetic임을 구분.

## 4. 실제 API 설정
서버 전용 환경변수 이름:
TIKHUB_API_KEY, JUSTONEAPI_API_KEY,
LIVE_PROVIDER_CALLS_ENABLED=false,
APP_TIMEZONE=Asia/Shanghai,
DATABASE_URL, REDIS_URL,
OBJECT_STORAGE_ENDPOINT, OBJECT_STORAGE_BUCKET,
OBJECT_STORAGE_ACCESS_KEY, OBJECT_STORAGE_SECRET_KEY,
LLM_ENABLED=false, LLM_API_KEY(선택).

서버측 plan budget:
max_requests(필수), provider_request_limits, platform_request_limits,
max_cost_by_currency(단가 알려진 경우), max_attempts, concurrency, rate_limit.
기본 live OFF. 예제 구성에는 빈 키와 비활성 호출만 넣는다.
키가 있고 사용자가 live ON과 명시 호출 상한을 설정하면 그 범위 안에서 작업을 진행한다.
매 단계마다 동일 허가를 다시 받는 절차는 만들지 않는다.
불명 단가는 cost_unknown과 요청 상한으로 관리하며 무료/0원으로 표시하지 않는다.
가격이 확인되면 정확한 통화·청구 규칙·출처를 기록하고 금액 하드캡 적용.

## 5. 처음 실제 연결
1. 최신 문서·인증·필수 매개변수·지원 surface 확인.
2. 호출 없는 plan estimate와 호출 상한 설정.
3. 공개 씨앗 URL 하나의 상세로 ID/시간/필드 시험.
4. 소량 검색/댓글 pagination 시험, 비밀 제거 fixture 생성.
5. 빈 결과·제한·오류·cache·정밀도 확인.
6. 기능별 default/fallback 선정.
7. 실제 사건에서 기간/쿼리를 제한해 수집.
8. 요청 수·추정/실제 비용·누락·지원 범위 보고.
키가 없으면 blocked만 기록하고 개발 중단 금지.

## 6. 스케줄/수집 상한
초기에는 수동 실행과 명시 계획만, 장기 예약은 후속 기능.
热搜 5분 주기는 예시이며 하루 288회 요청. 실제 갱신·비용에 맞게 조정.
과거 热搜는 사전 snapshot/검증된 과거 데이터 없으면 복구 불가.
프로필/관계는 전체 자동 확장 금지, 분석가 지정·선택된 계정군만.
팔로우 관계는 최초 버전 기본 OFF.
전체 이중 수집 기본 OFF, 기능 검증용 제한 표본만 교차 조회.
수집창·page cap·실패·신선도·필드 누락률 표시. 분모 불명 시 수집률 % 표시 금지.

## 7. 장애·복구
rate limit: 대기/상한 준수. auth: 중단 후 설정 수정. schema change: adapter 격리.
budget: paused_budget. unknown availability: unavailable_unknown.
업스트림 중단이면 검증된 동일 기능 fallback 또는 partial.
Redis 장애로 job message 유실해도 PostgreSQL의 durable job/checkpoint로 복구.
워커 lease 만료 재시도는 외부 호출 중복 가능성을 남긴다.
가용성 회복 뒤 새 관측과 새 분석을 생성하며 과거 보고서 자동 변경 금지.

## 8. 인수에 필요한 사용자 값
실제 호출 전 키를 서버 비밀 변수로 제공.
사건별 주제어/기간/씨앗, 공급자/플랫폼별 최대 호출 또는 확인된 금액 예산.
내부 설치 환경·사용자 역할·보존 기간.
LLM 외부 전송은 별도 선택; 초기 OFF.
새 유료 공급자 계약·공개 사이트 배포는 별도 사용자 결정이다.
