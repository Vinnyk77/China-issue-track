# 개발 순서와 완료 기준
버전 3.0. 빈 저장소에서 아래 순서로 구현한다. T/X 같은 이전 티켓 체계는 사용하지 않는다.
각 단계 완료 후 PROJECT_STATE·TEST_RESULTS를 갱신하고 다음 단계를 계속한다.
필수 외부 입력이 없으면 해당 작업을 blocked로 기록하고 독립 기능을 구현한다.
자연스러운 작업 단위에서 저장하되 검증 없이 “전체 완료”라고 하지 않는다.

## M0 — 기반과 실행 환경
작업:
- README/AGENTS/문서 읽기, 현재 파일·저장소 지침 확인.
- 프로젝트 구조·의존성 lockfile·Compose·.env.example·.gitignore.
- backend health, DB migration 기초, frontend shell.
- 안전한 로컬 관리자 초기화, 인증·사건별 권한 검사 기반.
- PROJECT_STATE/DECISIONS/TEST_RESULTS/RUNBOOK/PROVIDER_VALIDATION 생성.
완료:
- 문서의 실제 명령으로 서비스 기동/종료 가능.
- 빈 키 환경에서 기동. 비밀이 git/frontend/log에 없음.
- DB 준비 대기와 오류 메시지, auth/권한 smoke 확인.
- 외부 유료 호출 0회.
산출물: 실행 가능한 shell, 선택 이유, 실제 검증 결과.

## M1 — 공통 데이터·파일 입력·세 플랫폼 demo
의존: M0.
작업:
- platform/type/native_id 유일키, 계정·원본·관측·사건 멤버십 모델.
- 시간 구간/정밀도·수정·관측·원문 span.
- CSV/JSON preflight→오류 미리보기→commit, 중복/관측 충돌 처리.
- 가상 객체/원본 fixture와 demo seed. 실제 사용자 사건으로 오해되지 않게 DEMO 표시.
- 사건 생성·목록·콘텐츠/관측 상세·플랫폼별 품질.
완료:
- 3플랫폼 fixture를 한 사건에 입력하고 원문/receipt 열람.
- 다른 플랫폼 같은 ID/문구는 별개, 같은 원문 재수집은 객체 공유.
- 잘못된 ID/시각/parent/권한 입력에 오류 위치 표시.
- import 재실행에서 객체·관측 중복 규칙이 재현됨.
- 키·LLM 없이 작동.
산출물: 최소 수직 흐름. 이 단계는 분석/실제 연동 완료가 아님.

## M2 — 시간·콘텐츠 계보·주장 비교
의존: M1.
작업:
- 세 플랫폼 timeline 및 외부 사건 행.
- 직접 URL/ID 참조, 희귀 문구 후보, 이미지 exact hash/pHash, 기본 OCR adapter.
- 시간 구간·revision-aware 방향 검토, 공통 원본 대안.
- 콘텐츠 family, edge evidence, 후보/승인/기각 UI.
- claim span·문구 diff·수동 변화 라벨.
완료:
- TC01~TC10 관련 경로·시간·수정·의미 테스트 통과.
- edge→observation→raw receipt 역추적.
- 시간만 선행하면 전파/인과 확정 배지 없음.
- OCR 미설치/모델 미확보 상태를 기능 제한으로 표시하고 텍스트 분석 지속.
- 최소 중국어 sample에서 OCR 실제 엔진 smoke를 했는지 별도 기록.
산출물: 콘텐츠 과정 재구성. 조직성 통계는 다음 단계.

## M3 — 반복 협조 분석
의존: M2.
작업:
- 행동 이벤트·중복 제거·시간 정밀도 gate.
- 반복 공동대상/문구/매체·동기화 후보.
- 플랫폼/시간/행동 층화 null, seed, 검정 집합·효과·p/q.
- 클러스터/근거 표·지도. 신호 독립성 및 공개 template 처리.
완료:
- 알려진 가상 패턴에서 기대 edge, 일반 폭증 대조에서 해석 제한.
- 실제/null 동일 파이프라인, 후보 선택 편향 테스트.
- 고정 input/config/seed 결과 재현.
- descriptive_only/분해능 부족/자원 초과 표시.
산출물: 검토용 조직성 근거. 실제 사건 정확도 보장 아님.

## M4 — 조사 검토·보고서
의존: M2, M3.
작업:
- 증거·가설·지지/반대·대안·미확인·공개 관계.
- 리뷰 이력·승인 상태·자체 검토 표시.
- Markdown/HTML/CSV/JSON/ZIP와 manifest 검증.
- 보고서 input 고정과 새 run/report 버전.
완료:
- case→import→analysis→edge evidence→hypothesis→review→export E2E.
- 모든 수치/문장 원문 추적, 후보와 승인 분리.
- 새 수집 후 이전 보고서 hash 불변.
- 인과/지시자 미확인을 확인된 사실처럼 출력하지 않음.
산출물: mock/import 기반 내부 조사 앱.
이 시점에 “파일 입력 기반 흐름 완료”라고 보고할 수 있으나 “실제 3플랫폼 자동 수집 완료”는 불가.

## M5 — 공급자 공통 수집·예산·상태
의존: M1; M2~M4와 계약 호환 유지.
작업:
- 공통 adapter 및 mock HTTP 계약.
- 공급자×플랫폼×surface capability matrix.
- dry-run estimate, 요청 예약/정산, timeout 불명 청구, lease/checkpoint/retry.
- page cap·cursor 반복·schema 오류·fallback·품질 dashboard.
완료:
- mock HTTP로 장애/재개/동시 worker 예산 상한 테스트.
- 키/유료 예산 없으면 live 차단, 파일 분석은 계속.
- 검증된 동등 기능만 fallback; 실패를 빈 데이터로 숨기지 않음.
산출물: 실제 연동 전 수집 인프라.

## M6 — 실제 공급자 검증·제한 수집
의존: M5. 사용자 환경에 키와 호출 상한이 있을 때 실행.
작업:
- 최신 공급자 문서 확인, 인증/endpoint/response 최소 계약 확정.
- 예산 내 소량 호출, 비밀 제거 fixture, 정규화 mapping.
- 微博/小红书 검색·상세·댓글·프로필·트렌드 중 실제 사용할 기능 검증.
- 豆瓣 group 기능 확인 시 구현. 미확인은 manual_import 유지.
- 비용·검색 정렬/과거 범위·캐시·시간 필터·페이지 cap 실측.
완료:
- LIVE 검증 기능별 실제 응답·검증일·제약·예산 사용을 기록.
- provider 2개 모두 adapter 경계와 문서가 있고 검증 상태 명시.
- 한 공급자의 장애 시 기능별 검증한 fallback만 사용.
- 豆瓣 movie만 검증한 경우 group 지원을 주장하지 않음.
외부 차단 시 M6 integration_blocked, 이전 단계 완료는 유지하고 M7 독립 검증 진행.
실제 연동에 새 API 서비스 결제/가입이 필요하면 자동 수행하지 않는다.

## M7 — 파일럿·운영 인수
의존: M4/M5, M6 가능한 범위.
작업:
- 의심 사건/공개 총공/자연 확산을 비교. 실제 사건에 정답이 없다면 정답으로 학습하지 않음.
- 실제 자료 확보 범위에 따라 계보 후보와 오탐을 사람이 검토.
- 보안·접근·키 redaction·SSRF·HTML/CSV·백업 복원.
- 비용·성능·자료 누락·분석 수용률 기록.
- demo/live/manual/unsupported 상태별 인수 목록과 사용자 기동 설명.
완료:
- docs/04 필수 테스트 및 인수 흐름을 실제 실행한 결과 제공.
- 실패/미검증/수동 기능을 명확히 구분.
- 내부 운영 버전과 향후 기능의 경계를 문서화.
- 공개 배포는 수행하지 않음.
M6 차단이 있으면 “파일 입력 운영 가능/실제 연동 일부 미완료”로 보고.

## 이후 기능
다사건 비교, 장기 schedule, 임베딩, Granger/Hawkes, 인과 연구, 고급 영상, PDF.
선택 기능을 먼저 구현하느라 M0~M7 핵심 흐름을 미루지 않는다.
큰 그래프 DB·분산 시스템·자동 identity resolution은 필요성 확인 후 별도 요구로 검토.
