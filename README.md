# Meeting Action Brief

A personal community project for evidence-linked meeting decisions and actions.
Works with **Grok CLI**, or with **Grok Bot's own model** through the portable
prepare/validate interface. See [Grok Bot setup](BOT_SETUP.md) for the no-nested-CLI
route. Python 3.10+; standard library only.

- Apply only confirmed, scoped aliases before extraction. Keep ambiguous names
  and semantic synonyms separate, with immutable originals and a change ledger.
- Keep explicit commitments from anonymous speakers. Do not invent owners,
  deadlines, approvals, or amounts.
- Check source IDs, exact quotations, dates, and numeric evidence in code.
- Optionally use TypeSafe Jev for multi-label hints, missing-action candidates,
  and claim review. No text is discarded by its score. Your own API key and
  permission to send the input to TypeSafe are required.
- Default: Grok only. Jev is **not** a proven speed or accuracy improvement.
  All outputs require review; no messages, calendar events, or tasks are sent.

This is not an official Grok, Cursor, TypeSafe, or employer product. Synthetic
examples are included; private dictionaries, meeting data, and credentials are
not. Code is available under the [MIT license](LICENSE). Platform/model/API usage
may still cost money.

## 한국어 사용 안내

회의 원문과 용어 사전을 보존하면서 결정·할 일·미확정 사항을 정리하는 Python 표준 라이브러리 도구입니다. 기본 경로는 Grok CLI만 사용합니다. Jev 분류·검증은 `--backend grok-jev`로 선택합니다. 모든 결과는 사람 검토가 필요한 초안이며, 메일 발송·게시·일정 변경·태스크 등록은 수행하지 않습니다.

Grok Bot이 직접 작성한 JSON도 검증할 수 있습니다. 이 경로와 `--prepare-only`는 Grok CLI 설치·로그인이 필요 없습니다. 공유 Bot 템플릿은 이 로컬 Python 도구나 API 키를 자동으로 포함하지 않습니다.

## 실행

Python 3.10 이상이 필요합니다. 아래 명령은 이 README가 있는 디렉터리에서 실행합니다. 출력 경로는 매번 새 경로여야 하며, 기존 디렉터리가 있으면 덮어쓰지 않고 중단합니다.

```bash
# 원문·사전·프롬프트만 준비. 외부 요청과 자격증명 읽기를 하지 않습니다.
python3 meeting_brief.py --input fixtures/korean-edge-cases.json \
  --output-dir /tmp/meeting-brief-prepared-001 --prepare-only

# 설치된 Grok CLI로 전체 초안 작성.
python3 meeting_brief.py --input fixtures/korean-edge-cases.json \
  --output-dir /tmp/meeting-brief-grok-001 --backend grok

# TypeSafe에 회의 텍스트를 추가 전송하는 선택 경로.
python3 meeting_brief.py --input fixtures/korean-edge-cases.json \
  --output-dir /tmp/meeting-brief-jev-001 --backend grok-jev

# Bot이 prepared.json을 읽고 draft.schema.json에 맞춰 만든 초안을 검증.
# 기본 경로에서는 외부 모델을 호출하지 않습니다.
python3 meeting_brief.py --validate-result bot-draft.json \
  --prepared-dir /tmp/meeting-brief-prepared-001 \
  --output-dir /tmp/meeting-brief-reviewed-001
```

마지막 검증 명령에 `--backend grok-jev`를 추가하면 Jev의 문장별 힌트와 주장별 검증도 수행합니다. `--grok-bin`은 실행 파일 경로, `--grok-model`은 명시적 Grok 모델 ID를 지정합니다. 생략하면 PATH의 Grok와 CLI 설정 모델을 사용합니다. `--grok-timeout`은 기본 180초, `--jev-timeout`은 기본 20초이며 둘 다 600초 이하여야 합니다.

Grok 실행에는 `--prompt-file`, `--output-format json`, `--json-schema`, `--tools ''`, `--disable-web-search`, `--no-subagents`, `--permission-mode plan`, `--max-turns 2`를 사용합니다. 셸 문자열 실행이나 권한 우회 옵션을 사용하지 않습니다. 설치된 CLI가 이 옵션을 지원하는지는 `grok --help`로 확인합니다.

## 입력 계약

최소 입력은 `documents` 배열입니다. 각 문서의 `id`, `kind`, `text`가 필요합니다.

```json
{
  "documents": [
    {"id": "M1", "kind": "raw_transcript", "text": "화자 1: 지운에게 검토안을 전달하겠습니다."}
  ],
  "glossary": {
    "version": "user-confirmed-v1",
    "entries": [
      {
        "canonical": "지훈",
        "aliases": ["지운"],
        "type": "person",
        "scope": {"document_ids": ["M1"], "required_terms": ["검토안"]},
        "status": "confirmed",
        "provenance": "사용자가 참석자 이름을 직접 확인함",
        "exclusions": ["동명이인"]
      }
    ]
  }
}
```

- 문서 ID는 `M1`, `M2`처럼 양의 정수로 끝나며 중복을 허용하지 않습니다. 문서 순서를 바꿔도 명시적 ID는 그대로 사용합니다. `kind`는 `raw_transcript`, `manual_note`, `corrected_note`, `prior_brief` 중 하나입니다. 선택 필드는 `title`, ISO `meeting_date`, `timezone`입니다.
- 원문의 비어 있지 않은 각 줄을 화자 발언 또는 문단으로 삼아 `M1-S1` 등의 ID를 부여합니다. 줄을 삽입하거나 분할하면 후속 ID가 달라질 수 있으므로 비교할 때 같은 원문과 기존 prepare 결과를 유지합니다. 각 위치는 Unicode 문자 인덱스이며 끝 위치는 제외합니다. 문서 전체와 각 구간의 SHA-256을 기록합니다. 빈 줄과 개행도 문서 원문에 보존합니다.
- 사전은 선택 사항이며 기본은 비어 있습니다. 다른 사람의 이름·회사·기억·사전을 불러오지 않습니다. `type`은 `person`, `organization`, `product`, `term`, `semantic_synonym` 중 하나입니다. `status`는 `confirmed`, `candidate`, `rejected` 중 하나입니다.
- 적용 범위는 명시한 `scope.document_ids`로 제한합니다. `required_terms`가 있으면 해당 원문 줄에 모든 문자열이 존재해야 합니다. `exclusions` 중 하나라도 같은 줄에 존재하면 해당 치환을 보류합니다. 자유문장 범위·추정 주제·와일드카드는 허용하지 않습니다.
- `confirmed`이고 비어 있지 않은 `provenance`가 있는 정확한 alias만 적용합니다. provenance는 사용자가 제공한 확인 근거이며 이 도구가 외부 원장을 조회해 진위를 인증하지는 않습니다. 의미 유의어, STT 후보, 같은 alias의 복수 대상, 교차 중첩은 자동 치환하지 않습니다.
- 한글 조사·존칭의 제한된 목록을 허용하며 단어 내부의 짧은 일치는 치환하지 않습니다. 겹치는 후보 중 유일한 최장 명칭이 짧은 후보를 완전히 포함하면 최장 명칭을 우선합니다. 형태소 분석기가 아니므로 보류 내역을 확인해야 합니다.
- `normalized_view`는 매번 원문 `text`에서 새로 만듭니다. 원문을 정규화 결과로 교체하지 않습니다. `알파→베타`, `베타→감마`가 있어도 새로 만든 베타를 다시 감마로 치환하지 않습니다. 모든 변경에는 문서·위치·원문·표준명·근거를 기록합니다.

## 출력과 검증

`instructions.md`가 실행 지침의 단일 원본입니다. 공개 Meeting Action Brief v0.3의 Instructions 절 전체를 유지하고 앞에 CLI JSON 계약을 추가했습니다. 공개용 앱 이름·설명, 공개 전 체크리스트, 데모 절은 복사하지 않았습니다. 내장 가상 샘플은 실제 입력으로 전달하지 않는 한 근거가 아닙니다.

준비 결과는 `input.raw.json`, `prepared.json`, `prompt.txt`, `draft.schema.json`, `manifest.json`, `events.jsonl`입니다. 초안을 생성하거나 검증하면 `draft.raw.json`, `checks.json`, `coverage-review.json`, `brief.json`, `brief.md`가 추가됩니다. 실제 공급자 요청·응답·usage·반환 모델·지연을 별도 파일에 기록합니다. 비용이 공급자 응답에 없으면 `null`로 남기며 0원으로 간주하지 않습니다.

각 결정·할 일·미확정 항목은 고유 `id`, `text`, `owner`, `due`, `status`, `evidence`를 갖습니다. `owner`와 `due`는 모르면 `null`입니다. `status`는 `approved`, `proposed`, `pending`, `complete`, `unknown` 중 하나입니다. 인명 불명인 화자의 명시적 약속은 원문 화자명으로 할 일에 남기고, 실명·대상명·기한의 불확실성은 별도 질문으로 정리합니다.

근거는 `source_id`와 원문 그대로의 `quote`입니다. 코드는 ID 존재, 인용 문자열, 근거 안의 숫자·단위, 담당·기한의 문자 일치를 검사합니다. 날짜를 계산·정규화하지 않습니다. 표준 이름은 정규화 원문에서도 확인하지만 인용은 반드시 원래 철자여야 합니다. 구조 검사를 통과해도 제안과 확정의 혼동, 수치의 잘못된 귀속, 인용 문맥의 반전까지 보증하지 않습니다. 자동 검사는 `no_detected_issue`로 표시하며 의미상 확정이나 자동 승인으로 승격하지 않습니다.

`--validate-result`는 준비 당시 manifest의 해시와 원문·prepared 파일을 대조하고 원문에서 사전 처리를 재계산합니다. 기존 prepare 결과를 수정하지 않으며 새 결과 폴더에 원문과 초안을 보존합니다. 해시는 우발적 손상 탐지용이며 서명이나 위변조 방지 인증이 아닙니다.

| 종료 코드 | 의미 |
|---|---|
| 0 | 준비 완료 또는 검토용 초안 산출. 사람이 검증을 마쳤다는 뜻이 아님 |
| 1 | 입력·준비본 손상·Grok·스키마 등 필수 단계 실패. 이전 단계 증거 보존 |
| 2 | Jev 사용을 요청했으나 일부/전체 호출 실패. 검토용 결과와 degraded 상태 보존 |

## Jev 선택 경로와 정보 전송

`--backend grok`는 회의 원문·정규화 자료·제공된 사전을 인증된 Grok CLI로 보냅니다. Grok 계정의 사용량·비용 조건이 적용됩니다. `--backend grok-jev`는 추가로 해당 자료와 생성된 주장을 TypeSafe의 `https://api.typesafe.ai/v1/systemone`에 전송하며 별도 API 비용이 발생할 수 있습니다. 허용된 자료만 입력합니다. `--prepare-only`는 어느 공급자에도 전송하지 않습니다.

TypeSafe 키는 프로세스의 `TYPESAFE_API_KEY` 또는 기준 workspace의 `credentials/typesafe.env`에서 읽습니다. 기준 workspace는 `WORKSPACE_ROOT`를 우선하고, 없으면 `git --git-common-dir`의 부모를 사용합니다. 키 파일을 복사하거나 프롬프트에 넣지 않습니다. 키가 없거나 API가 실패하면 상태에 기록하고 Grok 원문 전체 경로 또는 외부 초안의 결정론 검증 결과를 유지합니다. 인증 키는 로그에서 제거하지만 원본 입력 자체는 그대로 보존하므로 입력 자료에 비밀 키를 넣지 않습니다. 실행 결과 폴더는 비공개 자료로 취급합니다.

모델은 공식 문서에서 2026-09-27 확인한 `jev-1.13.0`으로 고정했습니다. 반환 버전이 다르면 검토가 필요한 실패로 처리합니다. 각 문장의 결정·행동·미확정을 각각 Noul로 묻기 때문에 한 문장이 여러 역할을 가질 수 있습니다. 동일 문서의 독립 질문을 묶어서 전송하며 어느 문장도 점수 때문에 제거하지 않습니다. 주장 검증은 제공된 전체 문맥을 확인합니다. TypeSafe의 문맥 제한을 넘을 가능성이 있으면 자르지 않고 해당 단계를 degraded로 남깁니다.

Noul은 참일 확률이며 별도 confidence가 없습니다. `0.30–0.70`은 미보정 검토 구간이고 높은 값도 `high_support_unverified`로 남깁니다. 행동 확률이 0.70을 넘는데 대응하는 action 근거가 없으면 누락 후보를 표시하며 자동으로 할 일을 만들지 않습니다. 누락 후보가 없다고 모든 할 일을 추출했다는 뜻은 아닙니다. 숫자·날짜 계산은 Jev에 맡기지 않습니다. 한국어·적대적 입력·긴 문맥의 성능은 별도 실제 평가가 필요합니다.

공식 근거: [HTTP API](https://docs.typesafe.ai/api), [Noul](https://docs.typesafe.ai/primitives/noul), [모델](https://docs.typesafe.ai/models), [Jev 1.13 약점](https://docs.typesafe.ai/model-jaggedness/jev-1.13), [인용 검사](https://docs.typesafe.ai/cookbooks/citation_check), [일관성·불확실성](https://docs.typesafe.ai/cookbooks/consistency_noul_cookbook).

## 오프라인 검증

```bash
python3 -m unittest discover -s . -p 'test_*.py' -v
```

테스트는 외부 API·실제 키를 사용하지 않습니다. 한글 조사·부분 문자열·중첩·모호한 alias·연속 치환·범위·원문 위치, 익명 화자 약속, 근거/ID/숫자/응답 손상, 준비본 손상, 타임아웃·Jev 실패 fallback을 확인합니다. 실제 모델의 의미 정확도나 선정·보상 가능성을 검증하는 벤치마크는 아닙니다.
