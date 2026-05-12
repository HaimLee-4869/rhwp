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
# 1. 빠른 컴파일 체크 (10-30초)
cd /home/sprint/eunjung-rhwp-collab/rhwp
cargo check --lib

# 2. 관련 테스트만 (예: shadow 관련)
cargo test renderer::shadow  # ★ 전체 cargo test는 금지 (시간 길어)

# 3. WASM --dev 빌드 (매 수정마다, 5분 이내 예상)
wasm-pack build --target web --dev

# 4. 여기서 멈추고 사용자에게 브라우저 시각 검증 요청.
#    사용자: rhwp-studio 새로고침 → 한글 2024 캡쳐와 비교 → OK / NG 보고.
#    사용자가 OK 할 때까지 다음 단계 진행 금지.

# 마일스톤마다만 (--release, 10-20분 소요):
# wasm-pack build --target web --release
```

> 시각 검증은 사용자가 직접 한글 2024 / rhwp-studio 동일 페이지 캡쳐를 비교하여 수행한다. 자동 SVG 비교 도구는 사용하지 않는다.

---

## 절대 금지 (사용자 명시 승인 없이)

- `git commit` — 사용자가 검토 후 명시적으로 요청해야 함
- `git push` — 동일
- `wasm-pack build --target web --release` — 시간 길어 (10-20분). 매 수정에는 `--dev`만, `--release`는 마일스톤마다만
- **여러 버그 동시 처리** — 한 번에 하나만, 회귀 방지
- 본가 메인 브랜치(`main`)에 직접 작업 — 항상 `local/capstone-fixes` 브랜치
- `cargo test` 전체 실행 — 시간 길어. 관련 모듈만

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

## 발견된 버그 12건 (우선순위)

### A. 콘텐츠 누락
- **A-1**: 표 장식 바(파란색 선) 미표시
- **A-2**: 업무협약서 이미지 완전 누락
- **A-3**: 꺾쇠 괄호 누락
- **A-4**: 색칠 네모 + 화살표 기호 깨짐
- **A-5**: ▶ 삼각형 기호 미표시
- **A-6**: 목차 그림자 효과 미적용
- **A-7**: 표지 타이틀 글자 drop shadow 미렌더링 (HWPX `<hh:shadow>` offsetX/Y 파싱 누락) ★ 첫 타깃

### B. 레이아웃 어긋남
- **B-1**: 텍스트 줄바꿈 위치 불일치 (전반)
- **B-2**: 이미지 위치 어긋남
- **B-3**: 자간/행간 불일치 (글자 겹침)
- **B-4**: 글자 위치 전반 차이

### C. 글꼴
- **C-1**: 숫자에 HCI Poppy 폰트 적용 (한글/숫자 폰트 분리 미처리)

### D. 변환/회전
- **D-1**: 가로 회전 페이지 세로 표시

**권장 작업 순서**: A-6 (그림자, 가장 isolated) → A-3/4/5 (기호 매핑, 같은 패턴) → A-1 (장식 바) → A-2 (이미지 누락) → 나머지

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
