# 캡스톤 작업 가이드 (Claude Code 전용)

> 이 파일은 매 Claude Code 세션 시작 시 먼저 읽혀야 한다.
> 위치: `/home/sprint/eunjung-rhwp-collab/CAPSTONE-WORKFLOW.md`

---

## 미션

전북도청 캡스톤 — 본가 rhwp v0.7.11에서 **다음 두 파일이 한컴오피스 2024와 시각적으로 동일하게 렌더링되도록** fix.

대상 파일 (절대 경로):

- `/home/sprint/eunjung-rhwp-collab/test-samples/(260403) 스마트행정팀 바인더(26.3월말 기준) ★.hwpx`
- `/home/sprint/eunjung-rhwp-collab/test-samples/241230 (회의자료)공직기강 확립 및 조직쇄신 방안.hwp`

마감: **2026-05-22**

---

## 작업 경로

- 본가 코드: `/home/sprint/eunjung-rhwp-collab/rhwp/`
- SVG 출력: `/home/sprint/eunjung-rhwp-collab/output-svg/`
- 빌드 결과: `./target/release/rhwp` (네이티브)
- 작업 브랜치: `local/capstone-fixes`

---

## 🚫 절대 건들지 말 것 (CRDT/협업 보호)

다음 디렉토리/파일은 **읽기만** 가능, **수정 절대 금지**:

- `src/document_core/` — CRDT 영향 가능성
- `tests/`, `samples/`, `pdf/`, `pdf-large/` — 테스트 자산
- `bindings/`, `npm/`, `rhwp-chrome/`, `rhwp-firefox/`, `rhwp-safari/`, `rhwp-vscode/` — 다른 패키지
- `README.md`, `CHANGELOG.md`, `THIRD_PARTY_LICENSES.md` — 사용자 문서
- `Cargo.toml`, `Cargo.lock` — 의존성 변경 금지 (필요 시 사용자 승인 요청)

CRDT/협업 코드는 본가 rhwp 레포에 **존재하지 않으며**, 별도 패키지(`~/eunjung-rhwp-collab/rhwp-collab-package/`)에만 있다.
본가 `src/` 만 수정하면 협업 기능은 자동 보호된다.

---

## ✅ 수정 가능 범위

- `src/renderer/` — 렌더링 엔진 (주 작업 영역)
- `src/model/` — 문서 IR (신중하게, 변경 시 사용자에게 영향 설명)
- `src/parser/` — 파서 (비표준 HWPX 처리 시만)

---

## 검증 절차 (각 수정 후 필수, 이 순서대로)

```bash
# 1. 빠른 컴파일 체크 (10-30초) — native + WASM 두 타겟
cd /home/sprint/eunjung-rhwp-collab/rhwp
cargo check --lib
cargo check --target wasm32-unknown-unknown --lib

# 2. 관련 테스트만 (예: shadow 관련)
cargo test renderer::shadow  # ★ 전체 cargo test는 금지 (시간 길어)

# 3. SVG 좌표 검증 (native CLI path — EmbeddedTextMeasurer)
./target/release/rhwp export-svg <파일> -p <페이지>
# python3 으로 좌표 분석

# 4. WASM --dev 빌드 (매 수정마다, 5분 이내 예상)
wasm-pack build --target web --dev

# 5. 여기서 멈추고 사용자에게 브라우저 시각 검증 요청.
#    사용자: rhwp-studio Ctrl+Shift+R → 한글 2024 캡쳐와 비교 → OK / NG 보고.
#    사용자가 OK 할 때까지 다음 단계 진행 금지.

# 마일스톤마다만 (--release, 10-20분 소요):
# wasm-pack build --target web --release
```

> 시각 검증은 사용자가 직접 한글 2024 / rhwp-studio 동일 페이지 캡쳐를 비교하여 수행한다. 자동 SVG 비교 도구는 사용하지 않는다.

> **★ 결정적 학습 (5/17)**: SVG 좌표 검증 (3단계) 만으로는 Canvas 시각 정합 보증 X. 반드시 5단계 (사용자 rhwp-studio 시각 검증) 까지 완료해야 OK 판정.

## ★ Native vs WASM 두 path 정책 (5/17 영구화)

rhwp 의 텍스트 측정/배치 코드는 두 구현이 병존한다:

| Path | 사용처 | 구현 | char_width 함수 |
|---|---|---|---|
| Native | `rhwp export-svg` CLI, native test | `EmbeddedTextMeasurer` (`text_measurement.rs:185+`, `estimate_text_width_unrounded:1051+`) | `measure_char_width_embedded` (Rust 임베디드 메트릭) + 폴백 분기 (is_narrow_punctuation 등) |
| WASM | rhwp-studio Canvas (브라우저) | `WasmTextMeasurer` (`text_measurement.rs:656+`) → `wasm_internals::measure_char_width_hwp` (`text_measurement.rs:604+`) | `measure_char_width_embedded` (공통 DB) → JS 폴백 (`js_measure_text_width`) |

**규칙 (5/17 영구화 + 5/18 강화)**:
1. 두 path 의 `compute_char_positions` / `\t` 처리 / char_width 영역 수정 시 **반드시 양쪽 동시 수정**.
2. **`is_narrow_punctuation` 같은 분류 함수**를 native 분기에 추가했으면, `wasm_internals::measure_char_width_hwp` 의 JS 폴백 직전에도 동기화 분기 추가 (5/18 C9 패턴).
3. **메트릭 DB 정정** (예: U+2018 fullwidth 잘못 기록) 은 native + WASM 양쪽 동시 영향 — `measure_char_width_embedded` 한 곳 수정으로 양 path 동시 fix 가능 (DB 공유).

위반 시 결과: 네이티브 CLI (SVG export) 검증은 OK 통과, 그러나 rhwp-studio Canvas 시각은 NG. 5/17 F-4 1차 fix 가 정확히 이 패턴으로 NG → 2차 fix 에 WASM 추가 적용으로 OK. **5/18 C9 가운뎃점 fix 가 동일 패턴 재현** — F-4 학습 적용으로 본질 즉시 발견 + WASM path 동기화 fix.

### 디폴트 pagination = TypesetEngine (5/18 영구화)
`rendering.rs:1142-1170` 가 `RHWP_USE_PAGINATOR=1` env 없으면 `TypesetEngine::typeset_section()` 호출. **Paginator (engine.rs) 는 fallback** 경로. 따라서:

| 시멘틱 | engine.rs (Paginator) | typeset.rs (TypesetEngine) |
|---|---|---|
| 컨트롤 sort | `process_controls` (line 1011-1028 vert_offset stable sort) | (없음 — 동기화 검토 영역) |
| 표 split v_offset | `split_table_rows` (line 1693-1703) | `typeset_block_table` (line 2034-2057) ← **5/18 fix 적용** |

**규칙**: pagination 영역 fix 시 두 엔진 동시 정합 점검 필수. 디폴트 (typeset.rs) 누락 시 fix 효과 영구 무력화.

---

## 권한 정책 (세션 자동 적용)

사용자 강력 정책 (2026-05-16 갱신). 다음 명령은 매번 묻지 말고 자동 실행한다:

**자동 실행 (확인 불요)**:
- 셸 진단: `grep`, `cat`, `ls`, `find`, `wc`, `head`, `tail`, `sort`, `uniq`, `python3`, `unzip`
- Rust 빌드/검증: `cargo check`, `cargo test --lib <module>` (전체 `cargo test` 는 금지 유지), `wasm-pack build --target web --dev`
- rhwp CLI: `rhwp dump`, `rhwp dump-pages`, `rhwp export-svg`, `rhwp ir-diff`, `rhwp info`, `rhwp diag`
- Git 읽기: `git diff`, `git status`, `git log`, `git show`, `git blame`
- 파일 읽기 (Read tool), 디렉토리 탐색
- 파일 수정 (Edit/Write tool): `rhwp/src/renderer/`, `rhwp/src/parser/`, `rhwp/src/model/`, `CAPSTONE-WORKFLOW.md`, `mydocs/orders/` 영역 한정

**위험 명령 (매번 확인)**:
- 파일 삭제/이동: `rm`, `mv` (다른 디렉토리로)
- 권한 상승: `sudo`
- Git 쓰기: `git commit`, `git push`, `git reset --hard`, `git checkout` (다른 브랜치)
- 빌드 (장시간): `wasm-pack build --target web --release` (10-20분), `cargo build --release` (재빌드 시)
- 의존성 변경: `cargo add`, `cargo remove`, `npm install`
- 외부 네트워크: `curl`, `wget` (github / cargo 패키지 fetch 제외)
- 환경 영향: `chmod`, `chown`, `kill`, `pkill`, 시스템 서비스 명령

---

## 절대 금지 (사용자 명시 승인 없이)

- `git commit` — 사용자가 검토 후 명시적으로 요청해야 함
- `git push` — 동일
- `wasm-pack build --target web --release` — 시간 길어 (10-20분). 매 수정에는 `--dev`만, `--release`는 마일스톤마다만
- **여러 버그 동시 처리** — 한 번에 하나만, 회귀 방지
- 본가 메인 브랜치(`main`)에 직접 작업 — 항상 `local/capstone-fixes` 브랜치
- `cargo test` 전체 실행 — 시간 길어. 관련 모듈만

---

## 폰트 정책

본 캡스톤에서 한컴 시각 정합을 위해 **rhwp 코드 안에서 직접 추가하면 안 되는 폰트** 와 **주무관님께 요청해야 하는 폰트** 를 구분한다.

### 임베딩/적용 금지
- **HCI Poppy** — 한글 본문 폰트가 아닌 영문/숫자 전용 폰트로, **rhwp 가 자동 적용하면 안 됨** (현재 C-1 본질). 본 캡스톤 두 대상 파일 모두 한컴은 숫자도 한글 본문 폰트 (휴먼명조 등) 와 같은 폰트로 표시. rhwp 가 숫자 영역에 HCI Poppy 자동 매핑하는 동작이 C-1 의 본질이며, 이는 매핑 분리 정정 영역 — 임베딩 아님.

### 이미 작동
- **함초롬** — 표준 폰트 fallback 으로 정상 작동. 추가 작업 불필요.

### 주무관님께 요청 영역
- **한컴 자체 PUA 글리프 폰트 (HY견고딕, HCRBatang 등)** — 한컴 자체 폰트의 PUA 영역 (0xF02B1~F02C4 사각 안 숫자 같은 글리프) 은 표준 Unicode 매핑으로 시각 정합 불가 (F-1, F-6 본질). 한컴 라이선스 폰트 제공 후 SVG `@font-face` 임베딩 가능.
- rhwp 코드 안에서 추측 매핑 추가 금지 (회귀 위험).

---

## 디버깅 도구 (본가 제공)

본가 rhwp는 코드 수정 없이 진단할 수 있는 CLI 도구를 제공한다:

```bash
# 페이지 경계와 인덱스 시각화
./target/release/rhwp export-svg --debug-overlay <파일>

# 특정 페이지의 문단 배치 결과
./target/release/rhwp dump-pages -p <페이지번호> <파일>

# 특정 문단의 ParaShape, LINE_SEG, 표 속성 상세
./target/release/rhwp dump -s <섹션> -p <문단> <파일>

# HWPX↔HWP IR 차이 (같은 문서의 두 포맷 비교 시)
./target/release/rhwp ir-diff <파일.hwpx> <파일.hwp>
```

본가 메인테이너의 권장 순서:
1. `--debug-overlay` → 어떤 문단/표에서 문제 발생하는지 식별
2. `dump-pages -p N` → 페이지 배치 확인
3. `dump -s N -p M` → 문제 문단의 ParaShape 등 조사

---

## 작업 단위

한 세션 = **한 버그**. 다음 형식으로 명시되어야 함:

- **버그 ID** (예: A-6)
- **증상** (한 문장)
- **한컴 결과 vs rhwp 결과** (스크린샷 또는 SVG 경로)
- **영향 페이지/파일**
- **재현 명령** (예: 어느 파일 어느 페이지)

---

## 발견된 버그 (우선순위)

### A. 콘텐츠 누락
- **A-1**: 표 장식 바(파란색 선) 미표시
- **A-2**: 업무협약서 이미지 완전 누락
- **A-3**: 꺾쇠 괄호 누락 (raw `&lt;` `&gt;` 디코딩 후 ASCII < > — char-by-char SVG 위치 매칭 필요)
- **A-4 ✅**: U+F007E "사각 외곽 + 안 화살표" 합성 글리프 — raw PUA passthrough + `generic_fallback()` 함초롬 한국어 family name fallback chain. 1차 ■ 매핑 / 2차 ➡ 매핑 모두 NG, 3차 raw passthrough + 폰트 fallback 정공법 OK — **완료 5/16**
- **A-5**: ▶ 삼각형 기호 미표시 (잠정 매핑 완료 후 F-2 에서 정정)
- **A-6**: 목차 그림자 효과 미적용
- **A-7 ✅**: 표지 타이틀 글자 drop shadow 미렌더링 (HWPX `<hh:shadow>` offsetX/Y 파싱 누락) — **완료 5/12**

### B. 레이아웃 어긋남
- **B-1**: 텍스트 줄바꿈 위치 불일치 (전반, 페이지 분할 누적 — 5/18-21 정면 돌파 예정)
- **B-2**: 이미지 위치 어긋남
- **B-3**: 자간/행간 불일치 (글자 겹침)
- **B-4**: 글자 위치 전반 차이
- **B-5 ✅**: HWPX paragraph 의 비-인라인 control sentinel push 누락 — 표지 인라인 라벨 위치 — **완료 5/12**

### C. 글꼴
- **C-1**: rhwp 가 숫자에 HCI Poppy 를 자동 분리 적용하는 버그. 한글 2024 에서는 숫자가 본문 폰트 (휴먼명조 등) 로 통일되는데, rhwp 는 숫자만 HCI Poppy 로 잘못 적용해 시각이 어긋남. fix 방향: rhwp 의 숫자 폰트 자동 분리 로직 제거 또는 비활성화.

### D. 변환/회전
- **D-1**: 가로 회전 페이지 세로 표시
- **D-2**: 목차 페이지 쪽번호 누락 + 본문 시작 페이지 번호 차이 (page_number.rs / pagination/engine.rs 영역, isolated 아님)

### E. 렌더링 중복
- **E-1 ✅**: 인라인 TAC Table 두 번 렌더링 (paragraph_layout + 별도 PageItem::Table entry) — `typeset.rs:1797-1808` 가드 — **완료 5/13**

### F. HWP 파일 발견
- **F-1 ✅**: U+F02B1~F02C4 사각 안 숫자 한컴 자체 PUA 글리프 — 기존 표준 ①~⑳ 매핑 (Task #509) 은 fallback chain 효과 못 받음 (1순위 폰트가 표준 ① 글리프로 즉시 렌더링). 매핑 entry 제거 → raw PUA passthrough + A-4 fallback chain 재활용. PowerShell 디코딩으로 codepoint 확정 — **완료 5/16**
- **F-2 / F-7 ✅**: U+F02FB 본문 박스 마커 ▶ → ▸ (Black small right-pointing triangle) 매핑 정정 — **완료 5/13**
- **F-3**: 「참고2」 헤더 페이지 위치 — B-1 영역
- **F-4 ✅**: 목차 페이지 숫자 정렬 — 단일-run \t 케이스 의 `body_right = available_width - line_x_offset` 가 effective_margin_left 미포함 + seg_w 의 leading space 미포함 → cell right inner 미달. **결정적 학습**: rhwp 에 `EmbeddedTextMeasurer` (native) 와 `WasmTextMeasurer` (WASM) 두 `compute_char_positions` 구현 존재, 양쪽 동시 수정 필수. TextStyle 에 effective_margin_left 필드 추가 + targeted fix (cell right inner 정렬, leading space 포함 seg_w) — **완료 5/17**
- **F-5 ✅**: U+F007E 두 파일 글리프 충돌 가설 — 실은 매핑 오류 환상. 두 파일 동일 글리프, A-4 fix 로 자연 해소 — **완료 5/16**
- **F-6**: ◊ U+25CA — **rhwp 메트릭 영역 (본가 영역)**. 한글 2024 안에서도 같은 모양, rhwp 에서만 길쭉. 폰트 매핑 영역 아닌 SVG 메트릭 처리 영역. 캡스톤 마감 내 fix 어려움 — **보류**
- **F-8**: 폰트 메트릭 미세 차이 — F-6 와 동질, 메트릭 영역 (본가 영역) — **보류**

**5/12-13 완료 (5건)**: A-7, B-5, A-4 (1차 매핑), E-1, F-2/F-7
**5/16 완료 (3건)**: A-4 (3차 fallback chain 정공법), F-1, F-5 (자연 해소)
**5/16 보류 분류**: F-6, F-8 — 메트릭 영역, 본가 영역으로 분류
**5/17 ~ 5/18 새벽 완료 (6건 + 1 부분 완료)**:
- F-4 (단일-run RIGHT+leader, native + WASM 두 path)
- 카테고리 A (따옴표 / 로마자 / 원 안 숫자 → 자간 누적 → FontLoader OS 폰트 + @font-face local())
- Z = C-1 (`detect_lang_category` 숫자 분리 제거 → 한글 본문 폰트 통일)
- X (사전검증 ① 정렬 → hanging-marker guard, 진짜 본질은 본가 PR 영역)
- Y-2 (numbering "1." + 공백 → 3 cell render path + Number/Outline trailing space)
- F-3 = Y-1 부분 완료 (참고2 박스 위치 → `is_tac_table_inline` 조건 확장 + controls vert_offset sort, **anchored 표 vert_offset 적용은 5/18 trace 영역**)

**5/18 완료 (3건)**:
- Y-1 후속 ✅ (참고2 박스 layout 완성 → `has_inline_tac` 가드 + `paragraph_layout` Control::Table inline 분기 추가)
- 페이지 분할 정합 ✅ (pi=407 LAYOUT_OVERFLOW 31.2px → `typeset.rs` 어울림 표 v_offset_px 가용 높이 차감)
- C8+C9 ✅ (좁은 구두점 U+2018/U+2019/U+2027 폭 분류 → native `is_narrow_punctuation` + `measure_char_width_embedded` halfwidth_punct override + WASM `measure_char_width_hwp` 폴백 동기화)

**5/19~5/22 영역**:
- 5/22 마감 prep: 발표 내용 정리, 본가 PR 후보 도큐먼트화

**본가 PR 후보 (5/22 이후 도큐먼트화) 분류**:
- ★★★★: anchored shape vert_offset, paragraph_layout TAC inline routing (Control::Table 분기 포함), FontLoader OS 폰트 + @font-face local(), **TypesetEngine + Paginator 시멘틱 동기화** (5/18), **메트릭 DB U+2018/U+2019 fullwidth 정정 + native+WASM 동기화** (5/18 C8+C9)
- ★★★: Z, F-4 (native/WASM 일관성)
- ★★ specific guard: X (hanging-marker), Y-2 공백 (Bullet mirror)
- 보류 (본가 메트릭/CRDT 영역): estimate_text_width 과대 (X 본질), FieldMarkerType::Numbering variant 부재 (Y-2 inner mech), F-6/F-8 글리프 메트릭

**5/17 ~ 5/18 학습 framework 17개** (5/17 14개: mydocs/orders/20260517.md, 5/18 3개 추가: mydocs/orders/20260518.md):
1. native + WASM 두 path / 2. SVG ≠ Canvas / 3. 단일-run vs cross-run / 4. targeted fix /
5. PowerShell + console 결정적 단서 / 6. 본질 분류 정정 패턴 / 7. generic vs specific /
8. 시각 영역 vs inner mech / 9. F-4 동질 패턴 메트릭 영역 / 10. 사용자 시각 평가가 진단 정정 /
11. 본질 위치 정정 다중 (5번 chain) / 12. native vs WASM 진짜 영역 /
13. capstone scope framing 정정 (두 파일 → 모든 hwp 호환) /
14. **정밀 검증 패턴 — "회귀로 보였던 것이 실은 fix 효과 + 잔존 본질"** (Y-1 결정적, 사용자 "표 지우기" 실험) /
15. **디폴트 pagination = TypesetEngine** (Paginator 는 fallback) — 양 엔진 동시 점검 필수 (5/18 페이지 분할) /
16. **F-4 패턴 일반화 — 모든 char/layout fix 는 native + WASM 두 path 동시 점검** (5/18 C9 재현) /
17. **inspect 도구로 메트릭 DB 잠재 본질 발견** (5/18 휴먼명조 U+2018/2019 fullwidth 잘못 기록 확정)

---

## 기록

완료된 fix는 다음 형식으로 `mydocs/orders/yyyymmdd.md` 에 한 줄씩 기록:

```markdown
# 2026-05-13

## 완료
- A-6: 박스 그림자 효과 추가 (`src/renderer/shape_shadow.rs`)
  - 영향 파일: 스마트행정팀 바인더 1쪽 헤더, 2쪽 목차
  - 검증: SVG 비교 완료, 사용자 OK

## 진행 중
- A-3: 꺾쇠 괄호 매핑 추가 (PUA 매핑 패턴)
```

---

## 검증 통과 기준

1. 두 대상 파일에서 해당 버그가 시각적으로 해소
2. `cargo check --lib` 통과
3. 관련 테스트 통과 (있는 경우)
4. **다른 페이지에 회귀가 보이지 않음** (사용자가 SVG로 빠르게 훑어서 확인)
5. 사용자 명시적 OK

---

## 권한 정책 (모든 세션 적용)

다음 read-only 진단 명령은 매번 묻지 않고 자동 실행:
- grep, cat, ls, find, head, tail, wc
- python3 -c (read-only XML/JSON 분석)
- cargo check, cargo test
- ./target/release/rhwp dump, dump-pages, ir-diff, export-svg (출력만)
- git diff, git status, git log, git branch
- file, stat, du -sh
- unzip -p (read-only)

위험 명령은 매번 확인:
- rm, mv (파일 삭제/이동)
- sudo
- git push, git reset --hard, git checkout
- 외부 네트워크 (curl, wget — github/cargo 제외)
- chmod +x, chown