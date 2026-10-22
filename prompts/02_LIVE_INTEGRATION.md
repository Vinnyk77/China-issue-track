# 실제 API 연동 프롬프트

CPEL의 M5 수집 인프라를 확인하고 M6 실제 공급자 연동을 진행하세요.
먼저 AGENTS.md, docs/02, docs/05, PROVIDER_VALIDATION.md, PROJECT_STATE.md를 읽으세요.

서버 비밀 환경변수에 TikHub/JustOneAPI 키가 있는지 값 노출 없이 확인하세요.
LIVE_PROVIDER_CALLS_ENABLED와 사건/공급자별 호출 상한을 확인하세요.
설정이 갖춰졌으면 그 범위 안에서 실제 요청을 수행하세요. 반복 확인을 요구하지 마세요.
필수 값이 없으면 무엇이 빠졌는지만 기록하고 mock 계약·normalization·독립 개발을 계속하세요.
키를 대화나 frontend에 붙여넣도록 요구하지 마세요.

최신 공식 공급자 문서에서 provider×platform×surface×capability를 확인하세요.
문서만 확인한 상태와 실제 응답을 검증한 상태를 구분하세요.
초기에는 공개 씨앗 URL 상세와 최소 pagination으로 시간·ID·오류·cache를 확인하세요.
응답 fixture는 비밀을 제거하고 계약 테스트를 만드세요.
단가/청구 규칙이 불명이면 0원으로 표시하지 말고 호출 상한과 cost_unknown을 사용하세요.
재시도도 상한에 포함하고 timeout은 uncertain charge로 남기세요.
검증한 동등 기능만 fallback에 등록하세요.

微博/小红书는 실제 사용할 기능을 우선 검증하세요.
豆瓣小组와 영화 리뷰 API를 혼동하지 마세요. 小组 미확인은 manual_import 유지.
추가 서비스 결제·가입·비공개 수집·우회 접근은 하지 마세요.
실제 검증한 endpoint/기능/수집 범위/요청 사용량/제약/미검증을 기록하고,
해당 상태가 UI·보고서에 반영되는지 확인하세요.
