# DesignPack 트렌드 갤러리 — 작업 핸드오프

## 이 문서의 목적
`designpack-gallery.html`(트렌드 디자인 템플릿 갤러리)을 DesignPack 코드베이스로 옮겨 세 가지 기능을 추가하기 위한 인수인계 문서입니다. 이 문서와 함께 `designpack-gallery.html`, `templates.json`이 전달됩니다.

---

## 1. 현재 상태

단일 파일 `designpack-gallery.html` (약 328줄, 순수 HTML/CSS/vanilla JS, 의존성 없음. 폰트만 Google Fonts CDN 사용).

화면 구성은 헤더 + 30개 템플릿 카드 그리드입니다. 카드 하나는 다음으로 이루어집니다.

- **preview**: 해당 디자인 스타일을 표현하는 230px 높이의 미니 목업
- **meta**: 번호·태그·이름·설명
- **palettes**: 5개의 컬러 조합 버튼 (각 버튼은 4색 칩 + 이름)
- **usebtn**: "이 스타일로 생성하기" 버튼 (현재는 alert 데모)

30개 템플릿은 모두 2026년 트렌드 기반입니다: Tactile Brutalism, Refined Glass, Dark Premium, Y2K, Nature Distilled, Dopamine Max, Bento Grid, Kinetic Type, Soft Neumorphism, Editorial Serif, Claymorphism, Aurora UI, Spatial/AR, Anti-Design, Mono Minimal, Cyber HUD, Pastel Dream, Vintage Print, Liquid Chrome, Organic Blob, High Contrast, Gradient Mesh, Skeuomorphic, Neon Cyberpunk, Warm Minimal, 3D Render, Blueprint, Playful Sticker, Frost Light, Duotone Bold.

## 2. 핵심 아키텍처 (반드시 이해할 것)

### 데이터 구조
모든 템플릿은 JS 배열 `templates`에 들어 있고, 각 항목은 다음 형태입니다.

```js
{
  n: "01",                       // 2자리 ID (문자열)
  tag: "Tactile Brutalism",      // 스타일 분류명
  name: "Raw Grid",              // 표시 이름
  desc: "1px 보더, 단단한...",    // 한글 설명
  palettes: [                    // 정확히 5개의 조합, 각 조합은 정확히 4색
    ["#111111","#FF4D2E","#ECECE4","#FFFFFF"],  // [0] Signature (원본)
    ["#14110F","#3D5AFE","#EDEAE3","#FFFFFF"],  // [1] Cool
    ["#1A1A1A","#00C566","#F0EFEA","#FFFFFF"],  // [2] Fresh
    ["#121212","#FFB300","#EAE7DF","#FFFFFF"],  // [3] Warm
    ["#0F0F14","#E91E63","#ECEAE6","#FFFFFF"]   // [4] Vivid
  ],
  html: '<div class="p1" style="background:var(--c1)">...'  // var(--c1~c4) 사용
}
```

이 데이터는 별도 파일 `templates.json`으로도 export되어 있습니다. 리팩터링 시 이걸 그대로 import 모듈로 쓰면 됩니다.

### 색상 적용 메커니즘 (가장 중요)
색은 preview HTML에 **하드코딩되어 있지 않습니다.** 모든 preview는 CSS 변수 `--c1, --c2, --c3, --c4`를 참조합니다 (`style="background:var(--c1)"` 형태). 팔레트를 바꾸는 것은 곧 그 카드 preview 요소의 네 변수를 교체하는 것입니다.

```js
const PAL_NAMES = ["Signature","Cool","Fresh","Warm","Vivid"];

function applyPal(n, idx, btn){
  const t = templates.find(x=>x.n===n);
  const pv = document.getElementById('pv-'+n);   // 카드별 preview 요소
  const p = t.palettes[idx];
  pv.style.setProperty('--c1',p[0]);
  pv.style.setProperty('--c2',p[1]);
  pv.style.setProperty('--c3',p[2]);
  pv.style.setProperty('--c4',p[3]);
  if(btn){ /* active 클래스 토글 */ }
}
```

CSS의 `.pN { ... }` 규칙은 **구조(위치·크기·폰트)만** 담당하고 색은 일절 담지 않습니다. 색 역할은 인라인 `var()`로만 지정됩니다. 일부 그라데이션/믹스는 `color-mix(in srgb, var(--c2) 40%, transparent)`를 씁니다.

**c1~c4의 역할은 템플릿마다 다릅니다.** 예를 들어 Brutalism에서 c1=잉크/보더, c2=액센트, c3=배경, c4=서피스. Dark Premium에서는 c1=배경, c2=강한 CTA, c3=밝은 그라데이션 텍스트, c4=텍스트. 커스텀 컬러 기능을 만들 때 이 역할 차이를 사용자에게 어떻게 노출할지가 핵심 설계 포인트입니다 (아래 3-2 참고).

## 3. 추가할 기능 (우선순위 순)

### 3-0. 선행: 리팩터링 (먼저 할 것)
단일 HTML을 그대로 옮기지 말고 다음으로 분리하세요. (스택은 DesignPack 본 코드에 맞출 것 — Next.js라면 아래처럼)
- `data/templates.ts` (또는 .json) — `templates` 데이터
- `components/TemplateCard.tsx` — 카드 한 장 (preview + meta + palettes)
- `components/PalettePicker.tsx` — 팔레트 버튼 묶음
- preview의 CSS 변수 적용은 React state로 관리 (현재의 `pv.style.setProperty` DOM 직접 조작을 state 기반으로 전환). 카드별로 `selectedPaletteIdx`와 `customColors` 상태를 가짐.
- 색 변수는 인라인 `style={{ '--c1': c[0], ... }}`로 주입.

기존 CSS의 `.p1`~`.p30` 구조 규칙과 preview HTML 문자열은 그대로 재사용 가능합니다. preview를 `dangerouslySetInnerHTML`로 넣거나, 더 깔끔하게는 컴포넌트화하세요(시간 여유가 있을 때).

### 3-1. 팔레트 이름 한글화
`PAL_NAMES`를 한글로 교체. 제안: `["시그니처","쿨","프레시","웜","비비드"]`. 단순 문자열 교체라 가장 쉬움. i18n을 쓴다면 그 체계에 맞출 것.

### 3-2. 사용자 커스텀 컬러 슬롯
각 카드에 6번째 옵션 "직접 입력"을 추가. 클릭 시 4개의 `<input type="color">` (또는 HEX 텍스트 입력)을 노출하고, 입력값을 그 카드의 `--c1~c4`로 실시간 반영.

설계 주의점:
- 슬롯에 **역할 라벨**을 붙일 것. c1~c4가 템플릿마다 다른 역할이므로 "색 1/2/3/4"는 사용자에게 무의미함. 각 템플릿 데이터에 `roles: ["배경","잉크","액센트","서피스"]` 같은 필드를 추가하는 것을 권장 (30개에 대해 채워야 함 — 위 2절의 역할 설명과 각 preview HTML을 보고 유추 가능).
- 커스텀 값은 카드별 state에 저장. 다른 카드에 영향 없도록.
- 가독성 가드(선택): 배경 대비 텍스트 명도차가 너무 낮으면 경고 표시. WCAG 4.5:1 기준 권장.

### 3-3. HEX 복사 버튼
현재 선택된 조합(또는 커스텀 값)의 4개 HEX를 클립보드로 복사. `navigator.clipboard.writeText()` 사용. 복사 형식은 용도에 맞게 — 예: `#111111, #FF4D2E, #ECECE4, #FFFFFF` 또는 디자인토큰 형태(`--c1: #111111; ...`). 복사 성공 시 "복사됨" 토스트/체크 피드백 추가.

## 4. 검증 방법
- 30개 카드 전부 5개 팔레트가 정상 스왑되는지 (특히 그라데이션/`color-mix`를 쓰는 카드: Refined Glass, Dark Premium, Aurora, Claymorphism, 3D Render, Skeuomorphic 등)
- 커스텀 입력이 다른 카드에 누수되지 않는지
- HEX 복사가 현재 활성 조합과 일치하는지
- 모바일 1열 레이아웃 (`@media(max-width:780px)`)

## 5. 향후 연동 (이번 작업 범위 밖, 참고용)
`usebtn`은 현재 alert 데모. 실서비스에서는 "선택된 템플릿 n + 선택된 팔레트 idx(또는 커스텀 4색)"를 들고 `/studio`로 이동해 디자인팩 생성 프리셋으로 전달하는 것이 목표. 데이터 구조를 이 전달에 맞게 유지할 것.
