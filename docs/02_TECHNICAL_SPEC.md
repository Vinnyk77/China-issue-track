# 기술 구현 명세
버전 3.0 | 빈 저장소 기준 | 코드 작성 시 공식 의존성 문서로 버전 호환성을 확인한다.

## 1. 구조
Frontend: React + TypeScript + Vite, ECharts timeline, Cytoscape.js network.
Backend: Python + FastAPI + Pydantic + SQLAlchemy + Alembic.
DB: PostgreSQL. Job: Celery + Redis. Raw/media: S3 호환 저장소, 로컬 MinIO.
Analysis: NumPy/SciPy/scikit-learn/NetworkX; 대량 중간 처리에 Parquet/DuckDB 선택.
OCR: 중국어 지원 로컬 엔진 adapter, 모델/버전/다운로드 출처 고정. OCR 불가여도 다른 분석은 가능.
LLM: 선택 adapter, 기본 OFF. 모델 없이 수동 주장 라벨과 템플릿 보고서 가능.
배포 단위: Docker Compose. 검증: pytest·프런트 정적 검사·핵심 브라우저 E2E.
전용 그래프/벡터 DB, Kubernetes는 최초 버전에 도입하지 않는다.

디렉터리:
backend/app/{api,auth,models,schemas,providers,collection,analysis,evidence,reports}
backend/tests/{fixtures,unit,integration}
frontend/src/{pages,components,api}
config/analysis_defaults.yaml
infra/compose.yaml
scripts/{dev,seed_demo,verify_export}
docs/와 prompts/는 이 패키지 구조 유지.
.env.example, .gitignore, README.md, lockfiles를 구현 시 생성.
Docker port는 개발 기본 localhost에 bind. 인증 우회가 있는 데모를 외부에 노출하지 않는다.

## 2. 데이터 불변 조건
모든 외부 ID는 문자열, 모든 내부 참조는 UUID.
계정 유일키: (platform,native_uid).
콘텐츠 유일키: (platform,object_type,native_id).
플랫폼 다른 같은 글은 별개 객체. 같은 플랫폼 공급자 중복은 객체 공유·receipt/관측 별도.
원문 관측은 append-only. 수정·삭제 상태도 새 관측.
analysis_run은 입력 observation ID와 config/code/seed를 고정한다.
조직 공통 객체를 저장하더라도 접근은 사건 멤버십으로 검사하고 다른 사건 존재 여부/관측을 노출하지 않는다.
검색·계정 상세·파일·그래프·보고서·다운로드 모두 서버 권한 검사.

## 3. 핵심 테이블
공통 PK/created_at/FK/index는 생략. 실제 migration에서 제약·삭제 정책을 명시.
| 테이블 | 필수 필드 |
|---|---|
| users,roles,case_permissions | user_id, role, case_id, allowed_actions |
| cases | org_id,title,start_at,end_at,status,purpose,retention_policy |
| case_queries | case_id,platform,surface,text,aliases,exclude_terms,sort,version |
| external_events | case_id,time_interval,event_type,public_source,review_status |
| collection_plans | case_id,version,platform_configs,budget_limits,query_versions |
| collection_jobs | plan_id,capability,window,status,checkpoint,lease,request_budget |
| provider_requests | provider,platform,surface,endpoint_version,sanitized_params,attempt,status,captured_at,raw_id,cost_status,currency,cost |
| raw_objects | sha256,storage_key,content_type,encryption_version,byte_size |
| accounts | platform,native_uid,first_seen,last_seen |
| account_observations | account_id,request_id,public_fields,field_presence,observed_at |
| containers | platform,type(group/subject/topic),native_id,public_name,provenance |
| content_objects | platform,object_type,native_id,author_id,parent_id,root_id,container_id,canonical_url,first_observed_at |
| content_observations | content_id,request_id,revision_hash,title_raw,text_raw,text_clean,time_fields,metrics,availability |
| case_content | case_id,content_id,discovery_query_id,source_kind,membership_reason |
| content_edges | from_content,to_content,type(repost/reply/quote),direct_or_inferred,evidence_id |
| engagement_snapshots | content_id,request_id,observed_at,metric_name,value,unit |
| trend_snapshots/entries | provider,platform,board,collected_at,provider_time,topic_id/text,rank,metric_name/value,labels |
| observation_windows | case_id,platform,surface,query,start/end,cap,failure,time_resolution |
| media_assets | sha256,phash,storage_key,mime,width,height,source_receipt |
| ocr_segments | asset_id,bbox,text,confidence,engine_version,human_verified |
| media_derivations | input_asset,crop_region,transform,output_hash,tool_version |
| analysis_runs | case_id,input_manifest_hash,code_version,config_hash,seed,status,started_at |
| behavior_events | run_id,account_id,action,target_key,content_observation_id,event_time_interval |
| coordination_edges | run_id,actor_a/b,behavior,observed,expected,effect_size,p,q,evidence_ids |
| clusters/members | run_id,algorithm,parameters,cluster_id,account_id,membership_strength |
| content_families/members | run_id,method,content_observation_id,signals,review_status |
| claim_units | case_id,observation_id,span,proposition,negation,modality,speaker,attribution |
| claim_relations | claim_a/b,relation,evidence,review_status |
| propagation_candidates | run_id,parent_revision,child_revision,signals,time_status,alternatives |
| propagation_links | run_id,from_revision,to_revision,type,direction_status,evidence,review_status |
| evidence_items | case_id,observation_id,raw_id,excerpt,creator,captured_at |
| hypotheses/hypothesis_evidence | claim,subject,scope,status,supports/contradicts,alternatives,missing_evidence |
| public_entities/entity_relations | public_name,type,provenance,relation_type,source,reviewer,validity |
| account_link_assertions | account_a/b,public_link_evidence,status,reviewer |
| reviews | target_id,reviewer,decision,rationale,version |
| report_versions | case_id,run_id,immutable_manifest,status,artifact_hash |
| audit_events | actor,operation,target,time,sanitized_diff |

object_type: post,note,group_topic,review,short_comment,comment,reply.
source_kind: live_api,manual_import,fixture. 화면/보고서에서 항상 구분.
트렌드는 동일 보드인 경우에만 공급자 비교. 다른 metric은 단위와 원래 이름을 보존.
null은 0이 아니다. 알려지지 않은 작성자·날짜·지역을 추정값으로 채우지 않는다.
분석 입력 해시에 코드뿐 아니라 사전/template 목록·모형·OCR 버전 포함.

## 4. 시간 모델
UTC timestamptz로 저장하고 Asia/Shanghai 표시.
published_at_raw, published_lower, published_upper, timestamp_precision,
time_basis, timezone_assumption, edited_lower/upper, observed_at 필수 구조.
정밀도: second/minute/hour/day/unknown.
시간은 반열린 구간 [lower,upper), minute면 1분, day면 해당 로컬 날짜 전체.
relative time은 기준 관측 시각·반올림 규칙을 알아야 구간 환산. 아니면 unknown.
time_basis: provider_source,visible_page,capture_metadata,ocr_claim,manual_assertion,inferred.
캡처에 보이는 시각은 OCR claim으로 보존. 업로드 시각에 대입하지 않는다.
공급자 캐시 시각·수집 시각·원문 게시 시각을 구분한다.

A=[aL,aU), B=[bL,bU), aU≤bL이면 게시 시간상 A 선행.
구간 겹치면 방향 미확정. 지연 하한/상한은 보수적 범위 bL−aU, bU−aL로 표시하고 열린 경계가 있을 수 있음을 데이터 계약에 표시.
unknown은 정렬 편의를 위해 관측 시각을 쓸 수 있으나 게시 순서 판정에는 쓰지 않는다.
중요: 현재 문구의 최초 존재 시각이 게시물 생성 시각과 다를 수 있다.
수정 이력·이전 관측이 없으면 문구 차용 방향은 별도 unresolved_revision으로 표시.
게시물-level 상호 인용 cycle은 revision/event로 분해하거나 unresolved. 임의 edge 삭제 금지.

## 5. 공급자 인터페이스
ProviderAdapter(platform,surface):
 capabilities()
 search_content(query,start,end,sort,cursor)
 get_content(native_id)
 list_comments(native_id,sort,cursor)
 list_replies(comment_id,cursor) # optional
 get_account(native_uid) # optional
 list_account_content(native_uid,cursor) # optional
 get_trends(board) # optional
 list_reposts(native_id,cursor) # optional

Page[T]: items,next_cursor,has_more(bool|null),completeness,
provider_total_claimed,raw_receipt_id,warnings.
completeness: exhausted,capped,partial,unknown; exhausted는 조회 계획 소진이지 전체 확보가 아님.
오류: auth_failed,permission_denied,rate_limited,quota_exhausted,
transient_error,schema_changed,invalid_request,not_found,unavailable_unknown,unsupported.
HTTP200 본문 오류 확인. 빈 결과와 조회 실패 분리.
삭제 근거가 없으면 unavailable_unknown. 양 공급자 일치는 독립된 원천 증거 두 개가 아님.

## 6. 수집과 비용
단계: 계획 추정→예산 예약→요청→원본 저장→정규화 커밋→checkpoint 갱신→정산.
모든 실제 요청은 DB 트랜잭션에서 공급자/사건/플랫폼별 잔여 호출량을 예약한다.
알려진 단가는 금액도 예약. 불명 단가는 금액 0으로 취급하지 않고 호출 수 하드캡 및 cost_unknown 표시.
상한은 재시도를 포함. 서로 다른 통화 합산 금지(명시 환율 없을 때).
외부 호출은 DB와 exactly-once 불가: timeout/워커 장애 후 uncertain charge 기록, 보수적으로 예산 차감/예약 유지.
checkpoint resume는 논리 객체 중복 방지와 청구 중복 가능성을 분리.
max 3회 일시 오류 재시도+jitter, 429 Retry-After, 인증 오류 재시도 금지.
결과 저장 실패 시 성공 처리 금지. 예산/실패/부분 수집 경고 보존.
cursor 반복·페이지 반복·cap을 별도 종료 사유로 저장.
시간창은 공급자 최소 단위까지 분할하고 원문 시각으로 후필터. 경계 중첩+중복 제거.
수집 실패 구간을 시계열 0으로 채우지 않는다.
live 기능은 명시적 설정·키·호출 상한 필요. 없으면 mock/import 계속.

## 7. 정규화와 후보 검색
원문 유지. 파생 비교용 HTML 해제·Unicode·공백·간번체 정규화.
부정어·숫자·인명 보존. 인용 원문과 신규 작성문 분리.
URL 원문/정규화/redirect 결과를 별도로 보존. 추적 토큰이 필요한 원문은 제한 raw 영역.
중국어 char n-gram TF-IDF/cosine + 정확 해시 + MinHash 후보.
초기 탐색 예: 12자 이상, cosine≥0.85. 검증된 판정 기준이 아니며 config와 민감도 기록.
흔한 구호/공식 공지/기사 인용은 template registry로 낮은 정보량 표시.
이미지 SHA256는 동일 바이트, pHash는 후보, OCR는 원문 비교 보조.
이미지/OCR/phash가 같은 매체에서 나온 경우 독립 신호로 중복 가산하지 않는다.
URL/희귀 n-gram/media LSH/대상별 시간 index로 후보 생성. 전 계정/콘텐츠 N² 행렬 금지.
후보 cap에 걸리면 truncated 상태·규칙을 기록하고 전체 분석으로 표시하지 않는다.

## 8. 전파 계보
그래프 3종: 콘텐츠 전파, 계정 행동 협조, 공개 운영 관계. 서로 자동 전환 금지.
link type:
explicit_reference: 특정 원문 URL/ID를 명시.
embedded_capture: 원문과 맞는 캡처 영역, 출처·위조/오인 검토 상태.
strong_derivation_candidate: 희귀 오타/연속 문구/편집 흔적+시간 양립.
narrative_similarity: 의미 유사, 직접 전파 미확정.
temporal_precedence_only: 시간 선행만, 인과 화살표 금지.
unresolved_direction: 시간 겹침/문구 수정 이력 불명.
처리: 후보→시간/수정 검토→claim 비교→근거/대안 저장→사람 승인/기각/보류.
공통 미관측 원본을 unknown_source 또는 대안 가설로 둔다.
직접 링크도 인용 비판인지 구분한다.
텍스트/매체/시간 특징은 독립 값으로 저장하고 임의 확률로 합치지 않는다.
각 edge는 revision·원문 span·receipt까지 역추적.

## 9. 주장 변형
claim은 원문의 정확한 observation/span을 참조.
주어/서술/목적어, negation, modality, speaker, attribution, quote를 보존.
relation: verbatim_quote,paraphrase,added_detail,omitted_qualification,
speculation_to_assertion,negation_changed,misattributed_quote,
causal_link_added,rebuttal,unrelated.
초기 수동 라벨+문구 diff 필수. 규칙/선택 LLM 자동 제안은 candidate.
“왜곡”은 검토 후만, 초안은 구체적 변화로 표시.
동일 인물 언급만으로 같은 주장으로 군집화하지 않는다.
자동 truth 판정·의도·의뢰인 생성 금지.

## 10. 협조 행동과 우연 기준선
이벤트: authored_template,commented_on,reposted,shared_url,shared_media.
배우=플랫폼 계정 UUID. 서로 다른 플랫폼 계정은 공개 연결 검증 전 별도 배우.
수집 집계 좋아요로 누가 좋아요했는지 이벤트 생성 금지.
같은 배우-target-time bucket 중복 제거. 시간 정밀도보다 작은 동시성 창 금지.
초기 창: 30초/2분/10분, 모두 분석 config.
반복 공동대상·희귀 문구·동시성 특징을 분리. 같은 해시태그 하나는 충분하지 않음.

null: 플랫폼/surface/행동/사건 시간 블록 안에서 (target,time) 슬롯에 actor label을 섞어
해당 블록의 actor 활동 수와 target/time 슬롯을 보존. 실제·null 같은 중복 처리.
seed 고정, block 5/15/60분 등 민감도.
짧은 블록이 coordination을 보존하거나 큰 블록이 공통 충격을 희석할 수 있음을 명시.
탐색 B=200, 검토 B≥1000 제안. p=(1+#null≥observed)/(B+1).
최소 p 분해능·검정 집합·BH-FDR q·효과 크기·기대값·반복수 함께 표시.
관측 기반 후보 선택은 null에도 적용하거나 발견/검증 기간 분리.
관측 후보만 검정한 q를 전체 계정 쌍 q라고 표시하지 않는다.
기준선이 부적합하면 descriptive_only. 유의하지 않은 값을 숨기지 않는다.
반복≥3·독립 신호≥2 등은 검토 우선순위 기본값일 뿐 조작 판정 규칙 아님.
행동 edge와 원문 repost 방향 edge 분리. 클러스터 방식·해상도·seed 기록.
centrality는 지도상 위치이며 지휘권이 아니다.
identity 연결이 없어도 플랫폼 간 반복 동시 게시를 분석 가능, 동일 운영자라고 표현 금지.

## 11. 인과 분석 범위
필수: 근거 기반 시간/콘텐츠 과정 재구성, 공통 충격·미관측 원본 대안.
후속: 수집 품질을 통제한 시계열 탐색. 단위가 다른 플랫폼 반응을 단순 합산 금지.
Granger는 조건부 예측, Hawkes는 모형상 excitation이며 직접 전파/의도/지시 입증 아님.
선택 모형은 데이터 충분성 gate, 자기상관/계절성/잔차·홀드아웃·민감도·다중검정 기록.
실제 인과 효과는 별도 연구 질문·개입/노출·결과·비교집단·식별 가정 필요.
조건 불충족 status=not_identified. MVP는 인과 효과 추정 기능이 없어도 완료 가능하나 그 한계는 표시.

## 12. 내부 API
prefix /api/v1. 모든 UUID는 사건 권한 확인, 리스트는 cursor pagination.
POST/GET /cases ; GET/PATCH /cases/{case_id}
POST /cases/{case_id}/queries
POST /cases/{case_id}/external-events
GET /providers/capabilities
POST /cases/{case_id}/collection-plans/estimate
POST /cases/{case_id}/collection-runs # plan_version,idempotency_key
GET /collection-runs/{id}
POST /collection-runs/{id}/pause|resume|cancel # 실제로는 각 route
POST /cases/{case_id}/imports # preflight/commit 모드
GET /cases/{case_id}/coverage
POST /cases/{case_id}/analysis-runs # cutoff,config_version
GET /analysis-runs/{id}
GET /analysis-runs/{id}/timeline|content-families|propagation-graph|clusters # 각 route
GET /analysis-runs/{id}/edges/{edge_id}/evidence
GET /cases/{case_id}/content/{uuid}/observations
GET /cases/{case_id}/accounts/{uuid}
POST /cases/{case_id}/claims
POST /cases/{case_id}/claim-relations
POST /cases/{case_id}/evidence
POST/PATCH /cases/{case_id}/hypotheses[/{id}]
POST /cases/{case_id}/account-link-assertions
POST /reviews
POST /cases/{case_id}/reports # run_id
POST /reports/{id}/approve
GET /reports/{id}/download
GET /operations/budgets|provider-health # 각 route
장기 작업은 202+job_id. schema에 actual run/input/config reference 포함.
오류는 code,message,request_id. 누락이 있는 응답은 caveats 포함.
idempotency key는 조직/사용자/operation으로 scope하고 같은 키 다른 payload는 409.
다른 사건의 존재를 오류 메시지로 누출하지 않는다.

## 13. 보안/운영
세션/비밀번호 안전한 라이브러리 사용, CSRF 및 cookie 정책, 역할·사건 권한.
서버/worker만 키를 읽는다. 토큰이 query에 있으면 access log/trace/error에서 redaction.
Raw는 제한 암호화 영역, export는 정제본. raw와 정제본 해시를 구분.
게시물 HTML escape, CSV formula 방지, 업로드 크기/행 수/타입 제한.
공유 URL 해석·media fetch는 host allowlist·사설 IP 차단·매 redirect 재검사·용량/timeout 제한.
redirect를 통한 키 전송 금지. API 클라이언트와 public media fetch client 분리.
실제 private 정보는 추가 수집하지 않고 export에서 마스킹.
LLM 입력은 최소 발췌, 비신뢰 원문 지시는 실행하지 않음. 키·도구권한 제공 금지.
LLM output은 evidence ID·수치·인용 validator, 검토 전 초안.
기본 보존 제안 90일, 사건별 변경/보존 잠금/만료 삭제/백업 만료 이력.
append-only는 보존 기간 안의 규칙이며 영구 저장 강제 아님.
백업: DB+raw/media+manifest, 초기 내부 배포에서 복원 실제 테스트.
로그: request/analysis ID, 비용, schema 오류, 누락률, 큐 지연. 비밀·사적 원문 로그 금지.

## 14. 보고서/재현
Markdown+HTML+CSV+JSON manifest+ZIP.
모든 주장 evidence ID, 수치 분모/시점/범위, 후보/승인 분리.
report hash 고정. 입력 추가는 새 run/report.
verify_export는 manifest sha256와 내부 근거 참조 무결성 검증.
동일 seed/config/input으로 핵심 표/edge 재현, 생성 시각처럼 가변 metadata는 분리.
잠정 벤치마크: 8vCPU/32GB, 10만 이벤트/2만 계정 기본 후보 분석 30분 목표.
OCR/임베딩/대량 permutation 시간은 별도 측정. 실측 전 보장 수치로 표시하지 않는다.
일반 목록 p95 2초 목표, 그래프 상위 노드 표시 수 명시. 자원 초과는 failed_resource_limit.
