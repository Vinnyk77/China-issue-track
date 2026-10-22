# 파일 입력 및 가상 데이터 계약
버전 3.0. 이 문서는 구현할 계약이며 아래 예시는 실제 수집 자료가 아니다.

## 1. 입력 모드
POST imports는 preflight와 commit을 분리.
preflight: schema·시간·유일키·parent·URL·크기·행 수·충돌·비밀 유무를 확인해 오류/경고 표시.
commit: preflight 입력 해시와 정책이 같을 때 확정. 변경되면 새 preflight 요구.
기본 정책은 오류 행이 있으면 전체 commit 안 함. 명시적으로 valid_rows_only를 선택하면 제외 행/이유를 import receipt에 남김.
case_id는 서버 context, 입력 필드로 다른 사건 접근 불가.
URL만 제공되면 출처 bookmark 상태이며 원문·시간을 수집한 게시물로 세지 않는다.

## 2. JSON 계약
envelope: schema_version="1.0", source_kind="manual_import"|"fixture", records=[].
record 필수:
platform, object_type, native_id, observed_at, source_url,
published {raw,lower,upper,precision,basis,timezone},
text_raw 또는 title_raw 또는 media 목록 중 하나.
선택:
author {native_uid,public_name}, container {type,native_id,public_name},
parent {platform,object_type,native_id}, root {...},
edited {lower,upper}, title_raw,text_raw,metrics[],media[],source_note.
시각 unknown은 lower/upper null, precision unknown, basis와 raw는 가능한 범위로 제공.
fixture는 실제 source URL 대신 example.invalid 사용 가능하며 실제 링크 fetch 금지.
manual source_url은 허용 public host 또는 검토 경고. URL 검증이 콘텐츠 진위를 보증하지는 않음.
metrics 항목은 name,value,unit,observed_at. unknown과 0 구분.
media는 파일 sha256/로컬 첨부 참조 또는 public URL; 첨부 안전 검사 적용.
원문을 확실히 확보하지 못한 수동 주장은 source_note/미검증 증거로 관리.

예시:
```json
{
  "schema_version": "1.0",
  "source_kind": "fixture",
  "records": [{
    "platform": "xiaohongshu",
    "object_type": "note",
    "native_id": "demo_note_001",
    "source_url": "https://example.invalid/xhs/demo_note_001",
    "observed_at": "2026-10-20T03:00:00Z",
    "published": {
      "raw": "2026-10-20 10:00",
      "lower": "2026-10-20T02:00:00Z",
      "upper": "2026-10-20T02:01:00Z",
      "precision": "minute",
      "basis": "manual_assertion",
      "timezone": "Asia/Shanghai"
    },
    "author": {"native_uid": "demo_actor_001", "public_name": "가상 계정"},
    "text_raw": "这是一条虚构测试内容。",
    "source_note": "가상 자료이며 실제 사건과 무관함"
  }]
}
```

## 3. CSV 계약
필수 column:
platform,object_type,native_id,source_url,observed_at,
published_raw,published_lower,published_upper,published_precision,published_basis,
title_raw,text_raw.
선택:
author_native_uid,author_public_name,container_type,container_native_id,
parent_platform,parent_object_type,parent_native_id,source_note.
UTF-8, quoted newline 지원. ID는 처음부터 문자열로 파싱.
복잡한 media/metrics는 JSON 입력 또는 별도 첨부 mapping 사용.
업로드에서 CSV 수식을 실행하지 않고 export는 위험 prefix를 안전 처리.

## 4. 중복과 관측
object key가 같고 동일 관측 payload hash이면 재입력은 logical duplicate.
새 import receipt는 남기되 같은 관측/행동으로 중복 가산하지 않는다.
내용이나 관측 시각이 바뀌면 새 관측을 남긴다. event 추출은 source event ID와 정규화 행동으로 다시 중복 제거.
동일 관측 시각에 충돌한 내용은 둘 다 보존하고 conflict 상태.
parent가 파일 후반에 있으면 2-pass 해석, 외부 parent면 unresolved reference.
같은 플랫폼/type/native_id에 다른 author는 validation warning 및 사람 검토.
hash만으로 object identity를 새로 만들거나 다른 플랫폼 객체를 합치지 않는다.

## 5. 구현할 fixture 세트
F01 direct_chain: 3플랫폼 10~20 콘텐츠, 직접 참조와 수정 없는 경로.
F02 common_news: 외부 사건 하나, 세 플랫폼 동시 확산, 직접 링크 없음.
F03 public_fandom: 공개 template·총공 공지·조직적이지만 기만성 미확인.
F04 coordinated_repeat: 여러 target·시간창에서 같은 계정 쌍 반복.
F05 uncertain_revision: 날짜 정밀·수정 이력·캡처 시각·순환 인용.
F06 claim_changes: 추측→단정·인과 추가·부정·반박.
F07 provider_failures: 200 error·429·timeout·cursor 반복·cap·스키마 변경.
F08 identity_and_dup: 동일 닉네임·긴 ID·동일 원문 양 공급자.
모든 객체는 가상 이름, 인위 시각, example.invalid. 실제 개인 사건을 가상 정답으로 쓰지 않는다.
생성 seed와 expected findings/expected non-findings를 사람이 읽는 manifest로 제공.
수집 receipt fixture와 정규화 import fixture를 분리. 테스트에서 어느 계층을 검증했는지 명시.
가상 패턴을 잘 찾는다는 이유로 실제 탐지 정확도 수치를 보고하지 않는다.
