# pdf-inspector 한국어 정리 문서

> PDF를 AI가 읽기 좋은 **마크다운**으로 초고속 변환해주는 Rust 라이브러리.
> 설치 · 사용법 · PHP 연동 · 수익화 아이디어까지 한 번에 정리한 문서입니다.

## 🔗 GitHub 저장소

| 구분 | 주소 |
|---|---|
| **이 저장소 (포크)** | https://github.com/bmshin94/pdf-inspector |
| **원본 (upstream)** | https://github.com/firecrawl/pdf-inspector |
| 제작사 | [Firecrawl](https://firecrawl.dev) |

### 배포 채널

| 플랫폼 | 패키지 | 주소 |
|---|---|---|
| PyPI | `pdf-inspector` | https://pypi.org/project/pdf-inspector/ |
| npm | `@firecrawl/pdf-inspector` | https://www.npmjs.com/package/@firecrawl/pdf-inspector |
| npm (WASM) | `@firecrawl/pdf-inspector-wasm` | https://www.npmjs.com/package/@firecrawl/pdf-inspector-wasm |
| crates.io | `pdf-inspector` | https://crates.io/crates/pdf-inspector |

**라이선스: MIT** — 상업적 이용 · 수정 · 재배포 · 판매 모두 자유. 저작권 고지문만 포함하면 됩니다.

---

## 1. 이게 뭐 하는 거야?

PDF는 사람 눈에는 잘 보이지만, 컴퓨터에게는 **그림 덩어리**에 가깝습니다.
어디가 제목인지, 어디가 표인지 알 수 없죠.

pdf-inspector는 그걸 **구조가 살아있는 마크다운으로 번역**해줍니다.

```
📄 복잡한 PDF  →  [pdf-inspector]  →  # 제목
   (그림덩어리)                          - 목록
                                        | 표 | 표 |
```

### 핵심 기능 3가지

**① PDF 종류 판별 (10~50ms)**
- `TextBased` (텍스트가 이미 들어있음) / `Scanned` (스캔 이미지) / `ImageBased` / `Mixed`
- **몇 번 페이지가 OCR이 필요한지** 페이지 번호까지 알려줌

**② 위치 인식 텍스트 추출**
- 글자별 X/Y 좌표, 폰트 크기, 볼드/이탤릭 정보
- 신문식 2단·3단 컬럼도 읽는 순서를 맞춰서 정리
- CID/Type0 폰트(한중일), RTL(히브리어/아랍어) 지원

**③ 마크다운 변환**
- 제목(H1~H4), 불릿/번호/문자 목록, 코드블럭, **표**, 링크
- 페이지 번호·목차 점선(`......`)·머리말/꼬리말 같은 노이즈 자동 제거

### 왜 중요한가 — OCR 비용 절감

세상 PDF의 **약 54%는 이미 텍스트가 들어있어 OCR이 필요 없습니다.**
그런데 대부분의 파이프라인은 구분 없이 전부 OCR을 돌립니다 (건당 2~10초 + API 비용).

```
PDF 도착
  → pdf-inspector가 판별 (~20ms)
  → 텍스트 기반 + 신뢰도 높음?
      YES → 로컬에서 추출 (~150ms), 끝  ✅ 공짜
      NO  → OCR 서비스로 전송 (2~10초)  💸 유료
```

---

## 2. 성능 벤치마크

opendataloader-bench 코퍼스(PDF 200개), 로컬 엔진, OCR 비활성. 점수 0~1, 높을수록 좋음.

| 엔진 | 종합 | 읽기순서 | 표(TEDS) | 제목 | 200개 처리속도 |
|---|---|---|---|---|---|
| **pdf-inspector** | **0.875** | **0.915** | **0.814** | 0.788 | **0.470s** |
| liteparse | 0.873 | 0.913 | 0.693 | **0.811** | 0.750s |
| opendataloader | 0.831 | 0.902 | 0.489 | 0.739 | 2.569s |
| pymupdf4llm | 0.735 | 0.886 | 0.401 | 0.424 | 17.117s |
| markitdown | 0.589 | 0.844 | 0.273 | 0.000 | 16.165s |

> 2026-07-31 기준, Apple M4 Pro. 출처: [원본 README](https://github.com/firecrawl/pdf-inspector#benchmark)

**요약: pymupdf4llm보다 약 36배 빠르고, 표 인식은 약 2배 정확.**

---

## 3. 폴더 구조

```
pdf-inspector/
├── src/                  🧠 Rust 본체 (약 98,000줄)
│   ├── lib.rs            공개 API, PdfOptions 빌더
│   ├── detector.rs       PDF 타입 판별 (전체 로딩 없이)
│   ├── extractor/        텍스트 추출 (폰트, 레이아웃, 컬럼, 링크, 밑줄)
│   ├── tables/           표 감지 3종 (rect → line → heuristic 우선순위)
│   ├── markdown/         마크다운 변환 (분석 → 전처리 → 변환 → 후처리)
│   ├── vision/           선택적 OCR (PDFium 렌더링 + PP-OCRv6)
│   ├── tounicode.rs      CID 폰트 디코딩 (한중일)
│   └── bin/              CLI: pdf2md, detect-pdf, dump_ops
├── napi/                 🔌 Node.js / Bun 바인딩
├── wasm/                 🌐 브라우저 WebAssembly 바인딩
├── docs/                 📖 API 문서 (python.md, rust-api.md, ocr-runtime.md 등)
├── examples/             🎬 basic_usage.py
├── tests/                ✅ 픽스처 PDF 29개 + 스냅샷 테스트
├── external/bcmaps/      CJK 폰트 CMap 데이터
├── site/                 소개용 랜딩 페이지
├── AGENTS.md             AI 에이전트용 개발 가이드
└── Cargo.toml            Rust 패키지 정의 (v1.17.0)
```

### 처리 파이프라인

```
PDF bytes
  ├─► detector          → PdfType 판별
  └─► extractor
        ├─ fonts          → 폰트 폭, 인코딩
        ├─ content_stream → PDF 오퍼레이터 해석 → TextItem + PdfRect
        ├─ xobjects       → Form XObject 텍스트
        ├─ links          → 하이퍼링크, AcroForm
        └─ layout         → 컬럼 감지 → 줄 묶기 → 읽기 순서
              ├─► tables    → rect / line / heuristic → 마크다운 표
              └─► markdown  → 분석 → 전처리 → 변환 → 후처리
```

> 문서는 **한 번만 로드**되어 판별 단계와 추출 단계가 공유합니다 (중복 I/O 없음).

---

## 4. 설치 및 사용법

> 아래 예시는 모두 이 환경에서 **실제로 설치·실행하여 검증**한 결과입니다.

### 4-1. CLI (가장 간단)

```bash
cargo install pdf-inspector
```
Rust가 없다면 먼저:
```bash
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
```

명령어는 2개입니다.

```bash
pdf2md 문서.pdf              # PDF → 마크다운
detect-pdf 문서.pdf           # 타입 판별만
```

**실제 실행 결과:**

```
$ detect-pdf tests/fixtures/nexo-price-en.pdf
Type: TEXT-BASED (extractable text)
Confidence: 100%
OCR recommended: NO
Detection time: 2ms
```

```
$ detect-pdf tests/fixtures/scan_with_native_header_text.pdf
Type: IMAGE-BASED (mostly images, OCR may help)
Confidence: 80%
OCR recommended: YES
Pages needing OCR: all (of 1)
  page 1: scanned
```

```
$ pdf2md tests/fixtures/nexo-price-en.pdf      # 0.032초 소요
|Classification|Selling price before tax|Selling price after tax|...|
|---|---|---|---|
|Exclusive|80,509,000 ...|76,435,000 ...|...|
|Prestige|87,893,000 ...|83,445,000 ...|...|
```

**주요 옵션**

| 옵션 | 설명 |
|---|---|
| `결과.md` | 두 번째 인자로 출력 파일 지정 |
| `--json` | JSON으로 출력 (프로그램 연동용) |
| `--items-json` | 위치 정보가 있는 TextItem JSON |
| `--raw` | 헤더 없이 순수 마크다운만 |
| `--compact` | 토큰 절약 모드 (점선 등 축약) — AI 입력용 권장 |
| `--pages` | 페이지 구분 주석 삽입 (`<!-- Page N -->`) |
| `--select-pages 1,3,5-10` | 특정 페이지만 처리 |
| `--password PW` | 암호화된 PDF |
| `--detect-only` | 판별만 |
| `--analyze` | 판별 + 레이아웃 분석 (마크다운 생성 안 함) |
| `--ocr off\|auto\|force` | OCR 모드 (`--features ocr` 빌드 필요) |

### 4-2. Python (실무에서 가장 많이 사용)

```bash
pip install pdf-inspector
```
> 검증: 5.3MB 휠 다운로드 후 즉시 설치 완료. **Rust 툴체인 불필요** (미리 빌드된 휠 제공).
> 지원: CPython ≥3.8 / Linux(x86_64, aarch64) / macOS(Intel, Apple Silicon) / Windows(x64)

```python
import pdf_inspector

result = pdf_inspector.process_pdf("문서.pdf")

print(result.pdf_type)          # 'text_based' | 'scanned' | 'image_based' | 'mixed'
print(result.page_count)        # 페이지 수
print(result.confidence)        # 0.0 ~ 1.0
print(result.markdown)          # 마크다운 결과
print(result.pages_with_tables) # 표가 있는 페이지 번호
print(result.pages_needing_ocr) # OCR이 필요한 페이지 번호
print(result.has_encoding_issues)
```

**실제 실행 결과:**
```
타입: text_based
페이지: 4
신뢰도: 100%
걸린시간: 33 ms
표 있는 페이지: [3]
OCR 필요: 없음
```

**자주 쓰는 API**

```python
# 특정 페이지만
r = pdf_inspector.process_pdf("문서.pdf", pages=[1, 3, 5])

# 바이트로 처리 (웹 업로드 대응)
with open("문서.pdf", "rb") as f:
    r = pdf_inspector.process_pdf_bytes(f.read())

# 판별만 초고속으로
info = pdf_inspector.detect_pdf("문서.pdf")

# 순수 텍스트
text = pdf_inspector.extract_text("문서.pdf")

# 위치 정보 포함
items = pdf_inspector.extract_text_with_positions("문서.pdf", pages=[1])
for it in items[:5]:
    print(it.text, it.x, it.y, it.font_size, it.is_bold)

# 영역 지정 추출
regions = pdf_inspector.extract_text_in_regions("문서.pdf", [(0, [[0, 0, 600, 200]])])
```

**핵심 실전 패턴 — OCR 비용 절감**

```python
import pdf_inspector

def process(path):
    r = pdf_inspector.process_pdf(path)
    if r.pdf_type == "text_based" and not r.has_encoding_issues:
        return r.markdown           # 무료 + 0.1초
    return expensive_ocr_service(path)   # 정말 필요할 때만
```

### 4-3. Node.js

```bash
npm install @firecrawl/pdf-inspector
```
> 검증: `added 2 packages in 2s`

```javascript
import { readFileSync } from 'fs';
import { processPdf, processPdfWithOcr } from '@firecrawl/pdf-inspector';

const pdf = readFileSync('문서.pdf');
const r = processPdf(pdf);

console.log(r.pdfType);          // 'TextBased'
console.log(r.pageCount);        // 3
console.log(r.pagesWithTables);  // [2, 3]
console.log(r.markdown);
```

**실제 실행 결과:**
```
타입: TextBased | 페이지: 3 | 표: [ 2, 3 ]
##### Technical Information
## l T-12 SI
#### Thermodynamic Properties
# Freon 12
```

### 4-4. 브라우저 (WebAssembly)

```bash
npm install @firecrawl/pdf-inspector-wasm
```

```javascript
import init, { processPdf } from '@firecrawl/pdf-inspector-wasm';

await init();
const result = processPdf(new Uint8Array(await file.arrayBuffer()));
console.log(result.markdown);
```

> 서버 왕복 없이 브라우저에서 처리 → 민감 문서 취급 시 유리.

### 4-5. Rust

```bash
cargo add pdf-inspector
```

```rust
use pdf_inspector::process_pdf;

let result = process_pdf("document.pdf")?;
println!("Type: {:?}", result.pdf_type);
if let Some(markdown) = &result.markdown {
    println!("{}", markdown);
}
```

### 4-6. OCR 사용 (선택 사항)

기본 설치로는 **스캔 PDF의 글자를 읽지 못합니다.** 단, "OCR이 필요하다"는 **판별은 됩니다.**

```bash
export PDFIUM_LIB_PATH=/경로/libpdfium.so
export ORT_DYLIB_PATH=/경로/libonnxruntime.so
```

```python
ocr = pdf_inspector.process_pdf_with_ocr("스캔.pdf")
print(ocr.pages_routed_to_ocr)
```

Rust/CLI는 빌드 시 옵트인:
```bash
cargo install pdf-inspector --features ocr --bin pdf2md
pdf2md scan.pdf --ocr auto --json
```

> 상세: [docs/ocr-runtime.md](./ocr-runtime.md)
> **권장:** 세팅이 번거로우므로, 초기에는 "판별 + 텍스트 PDF 추출"만 사용하고
> 스캔본은 기존 OCR 서비스로 넘기는 구성이 효율적입니다.

### 4-7. 디버깅

```bash
RUST_LOG=pdf_inspector::extractor::layout=debug pdf2md file.pdf   # 컬럼/읽기순서
RUST_LOG=pdf_inspector::tables=debug pdf2md file.pdf              # 표 감지
RUST_LOG=pdf_inspector=debug pdf2md file.pdf                      # 전체
```

---

## 5. PHP 연동

### 연동 방식 비교 (검증 완료)

| 방식 | 가능 여부 | 비고 |
|---|---|---|
| **① CLI 호출 (`proc_open`)** | ✅ **검증됨** | 실측 왕복 **25ms**. 권장 |
| ② 마이크로서비스 (HTTP) | ✅ 가능 | 트래픽 증가 시 |
| ③ PHP FFI 직접 연동 | ❌ **불가** | C ABI 미노출 |

**③이 불가능한 이유** — 소스에 C 인터페이스가 없습니다.
```bash
$ grep -rn 'no_mangle\|extern "C"' src/
(결과 없음)
```
이 라이브러리는 Python(PyO3) · Node(napi-rs) · WASM(wasm-bindgen) 바인딩만 제공하며,
PHP FFI가 요구하는 C ABI는 노출하지 않습니다.
(필요하면 Rust에 `extern "C"` 래퍼를 직접 추가하면 가능하지만, ①로 충분합니다.)

**①이 실용적인 이유** — 바이너리 의존성이 거의 없습니다.
```bash
$ ldd target/release/pdf2md
libc.so.6, libm.so.6, libgcc_s.so.1     # 리눅스 기본 3개뿐
$ ls -lh target/release/pdf2md
8.2M                                     # 단일 파일
```
→ **바이너리 파일 하나만 서버에 복사**하면 동작합니다. PHP 확장 컴파일 불필요.

### PHP 래퍼 클래스

```php
<?php
/**
 * pdf-inspector PHP 래퍼 — CLI 바이너리를 호출하는 방식
 */
class PdfInspector
{
    public function __construct(
        private string $pdf2md = '/usr/local/bin/pdf2md',
        private string $detect = '/usr/local/bin/detect-pdf',
        private int $timeout = 30,
    ) {}

    /** PDF 종류 판별 (초고속) */
    public function detect(string $path): array
    {
        return $this->run($this->detect, [$path, '--json']);
    }

    /** PDF → 마크다운 + 메타데이터 */
    public function process(string $path, array $opts = []): array
    {
        $args = [$path, '--json'];
        if (!empty($opts['pages']))    { $args[] = '--select-pages'; $args[] = $opts['pages']; }
        if (!empty($opts['compact']))  { $args[] = '--compact'; }
        if (!empty($opts['password'])) { $args[] = '--password'; $args[] = $opts['password']; }
        return $this->run($this->pdf2md, $args);
    }

    /** 업로드된 바이트로 바로 처리 */
    public function processBytes(string $bytes, array $opts = []): array
    {
        $tmp = tempnam(sys_get_temp_dir(), 'pdf_') . '.pdf';
        file_put_contents($tmp, $bytes);
        try   { return $this->process($tmp, $opts); }
        finally { @unlink($tmp); }
    }

    private function run(string $bin, array $args): array
    {
        $cmd = escapeshellcmd($bin);
        foreach ($args as $a) { $cmd .= ' ' . escapeshellarg($a); }

        $descriptors = [1 => ['pipe', 'w'], 2 => ['pipe', 'w']];
        $proc = proc_open($cmd, $descriptors, $pipes);
        if (!is_resource($proc)) { throw new RuntimeException('실행 실패'); }

        $stdout = stream_get_contents($pipes[1]);
        $stderr = stream_get_contents($pipes[2]);
        fclose($pipes[1]); fclose($pipes[2]);
        $code = proc_close($proc);

        if ($code !== 0)     { throw new RuntimeException("pdf-inspector 오류(code $code): $stderr"); }
        $json = json_decode($stdout, true);
        if ($json === null)  { throw new RuntimeException('JSON 파싱 실패: ' . substr($stdout, 0, 200)); }
        return $json;
    }
}
```

**사용 예 (파일 업로드 처리)**

```php
$pi = new PdfInspector();
$r  = $pi->processBytes(file_get_contents($_FILES['pdf']['tmp_name']));

if ($r['pdf_type'] === 'text_based') {
    echo $r['markdown'];                 // 무료 · 즉시
} else {
    // 유료 OCR 서비스로 라우팅
}
```

**실제 실행 결과 (PHP 8.4)**

```
=== 1) 판별 ===
타입: text_based / 신뢰도: 1 / 페이지: 1

=== 2) 마크다운 변환 ===
타입: text_based / 페이지: 3
표 있는 페이지: 2,3
마크다운 길이: 5708자
PHP 왕복 총 시간: 25ms

=== 3) 업로드 시뮬레이션 (바이트로 처리) ===
타입: text_based / 페이지: 4 / OCR필요: 없음
```

### 보안 체크리스트

- `escapeshellarg()` / `escapeshellcmd()` 필수 (위 래퍼에 적용됨)
- 업로드 파일은 MIME 및 매직바이트(`%PDF-`) 검증
- 임시 파일은 `finally`에서 반드시 삭제
- 대용량 PDF는 큐(예: Laravel Queue)로 비동기 처리
- 프로세스 타임아웃 및 동시 실행 수 제한

---

## 6. 수익화 아이디어

### 전제

> **"PDF → 마크다운" 변환 자체는 상품이 되기 어렵습니다.** 누구나 무료로 설치해 쓸 수 있습니다.
> 수익은 **이 기술 + 고유한 무언가**의 결합에서 나옵니다.

### 🥇 1. PHP 생태계 플러그인 (권장)

**근거:** PHP 진영에는 이 수준의 PDF 파서가 없습니다.
기존 `smalot/pdfparser`는 표·다단 컬럼 인식이 사실상 불가능합니다.

| 만들 것 | 시장 |
|---|---|
| WordPress 플러그인 | 전 세계 웹사이트의 약 43% |
| 그누보드 / XE 모듈 | 국내 커뮤니티·기관 사이트 |
| Laravel 패키지 | PHP 개발자 |
| Composer 패키지 (Packagist) | PHP 생태계 전체 |

**모델:** 무료 배포 + Pro 버전(일회성 $29~99) — 배치 처리, OCR 연동, 우선 지원
**장점:** 진입장벽 낮음, 직접 경쟁자 거의 없음, 기존 PHP 역량 활용

### 🥈 2. OCR 비용 절감 미들웨어

**타겟:** 이미 OCR API 비용을 지출 중인 기업

```
현재:   PDF 1,000장 → 전량 OCR       → 월 100만원
도입 후: PDF 1,000장 → 판별 후 선별   → 월 46만원 (540장은 로컬 무료 처리)
```

**모델:** 절감액의 일정 비율(예: 20%) 성과 과금
**장점:** 고객 PDF 100장만 돌려보면 효과가 즉시 숫자로 증명됩니다.

### 🥉 3. 도메인 특화 SaaS

범용 변환기가 아니라 한 분야를 깊게 파는 방식.

| 분야 | 서비스 |
|---|---|
| 세무/회계 | 세금계산서·거래명세서 → 엑셀/ERP 자동 입력 |
| 부동산 | 등기부등본·계약서 → 구조화 데이터 |
| 금융 | 사업보고서·재무제표 → 표 데이터 |
| 학술 | 논문 → 마크다운 + 참고문헌 정리 |
| 보험 | 청구 서류 자동 분류 |

**모델:** 월 구독 (₩29,000 ~ ₩199,000)
**핵심 자산:** 해당 분야의 표 구조를 정확히 해석하는 **후처리 로직**

### 4. 사내 문서 AI 챗봇 구축 (SI/외주)

```
회사 PDF 문서 → pdf-inspector → 마크다운 → 벡터DB → AI 챗봇
```

- 구축비 500만 ~ 3,000만원 / 유지보수 월 50~200만원
- 단기 현금흐름이 가장 큰 모델
- 경쟁사 대부분이 pymupdf 계열이라 표가 깨짐 → **"문서 품질"이 명확한 차별점**

### 5. 웹 서비스 (`pdf2md` 형태)

하루 3건 무료 / 무제한 월 ₩5,900 수준.
직접 수익은 작지만 **포트폴리오 + 리드 수집 채널**로서 가치가 있습니다.
여기서 유입된 기업 문의가 2번·4번 고객으로 전환됩니다.

### ⚠️ 리스크 3가지

1. **스캔 PDF는 기본 미지원** — 국내 관공서·법원·의료 문서는 스캔본 비중이 높습니다.
   → 클로바 OCR / Upstage 등과 조합 필요. 이 제약이 오히려 **아이디어 2번의 근거**가 됩니다.
2. **경쟁자 존재** — LlamaParse, Unstructured, Upstage Document AI, Firecrawl 본체 등.
   → 범용으로 정면 승부하지 말고 **1번(생태계) 또는 3번(도메인)** 으로 우회.
3. **원본이 Firecrawl 소유** — 언제든 자체 SaaS를 강화할 수 있습니다.
   → 가치를 **파싱 엔진이 아니라 그 위에 얹는 레이어**에 둘 것.

### 추천 실행 순서

```
1주차     PHP 래퍼 정리 → Composer 패키지로 Packagist 등록
2~3주차   데모 웹서비스 오픈 (리드 수집용)
1~2개월   WordPress 플러그인 출시 → 무료로 사용자 확보
병행       주변 기업에 "OCR 비용 절감" 제안 (아이디어 2번)
```

**시작은 1번, 매출은 2번·4번에서.**

---

## 7. 참고 문서

| 문서 | 경로 |
|---|---|
| Python API | [docs/python.md](./python.md) |
| Rust API | [docs/rust-api.md](./rust-api.md) |
| Node.js API | [napi/README.md](../napi/README.md) |
| WASM API | [wasm/README.md](../wasm/README.md) |
| OCR 런타임 설정 | [docs/ocr-runtime.md](./ocr-runtime.md) |
| 벤치마크 방법 | [docs/benchmarking.md](./benchmarking.md) |
| 디버깅 | [docs/debugging.md](./debugging.md) |
| 개발 가이드 | [AGENTS.md](../AGENTS.md) |
