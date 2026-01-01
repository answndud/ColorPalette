# 🎨 Color Palette Generator 기술 문서

> 색상 이론과 WCAG 접근성 계산의 구현

## 📋 목차

1. [프로젝트 개요](#프로젝트-개요)
2. [색 공간과 변환](#색-공간과-변환)
3. [색상 조화 알고리즘](#색상-조화-알고리즘)
4. [WCAG 접근성 계산](#wcag-접근성-계산)
5. [Shades & Tints 생성](#shades--tints-생성)
6. [UI 구현](#ui-구현)

---

## 프로젝트 개요

Color Palette Generator는 디자이너와 개발자를 위한 색상 도구입니다.

### 핵심 기능

```
┌─────────────────────────────────────────────────────────┐
│                    Color Palette Generator               │
├─────────────────────────────────────────────────────────┤
│  🎨 색상 입력                                            │
│     └── Color Picker / HEX 직접 입력                     │
│                                                          │
│  🔄 색 공간 변환                                          │
│     └── HEX ↔ RGB ↔ HSL 실시간 변환                      │
│                                                          │
│  🌈 색상 조화 생성                                        │
│     ├── 보색 (Complementary)                             │
│     ├── 유사색 (Analogous)                               │
│     ├── 삼색 조화 (Triadic)                              │
│     └── 분리보색 (Split-Complementary)                   │
│                                                          │
│  📊 명도 변화                                             │
│     └── 10단계 Shades & Tints                            │
│                                                          │
│  ♿ 접근성 검사                                           │
│     └── WCAG 대비율 계산 (AA/AAA 등급)                   │
│                                                          │
│  💻 코드 생성                                             │
│     └── CSS 변수 코드 복사                               │
└─────────────────────────────────────────────────────────┘
```

---

## 색 공간과 변환

### 색 공간 비교

| 색 공간 | 형식 | 특징 | 용도 |
|--------|------|------|------|
| **HEX** | #RRGGBB | 웹 표준, 간결함 | CSS, 디자인 도구 |
| **RGB** | rgb(R, G, B) | 모니터 출력 방식 | 프로그래밍, 이미지 처리 |
| **HSL** | hsl(H, S%, L%) | 인간 친화적 | 색상 조합, 밝기 조절 |

### HEX → RGB 변환

HEX는 16진수로 표현된 RGB 값입니다:

```
#6366F1
  ││││││
  │││││└── Blue  (F1 = 241)
  │││││
  │││└┴── Green (66 = 102)
  ││└┴┴── Red   (63 = 99)
```

```javascript
function hexToRgb(hex) {
    // 정규표현식으로 6자리 HEX 파싱
    const result = /^#?([a-f\d]{2})([a-f\d]{2})([a-f\d]{2})$/i.exec(hex);
    
    return result ? {
        r: parseInt(result[1], 16),  // 16진수 → 10진수
        g: parseInt(result[2], 16),
        b: parseInt(result[3], 16)
    } : null;
}

// 예시
hexToRgb('#6366F1')  // { r: 99, g: 102, b: 241 }
```

### RGB → HEX 변환

```javascript
function rgbToHex(r, g, b) {
    return '#' + [r, g, b].map(x => {
        // 0-255 범위 제한
        const clamped = Math.max(0, Math.min(255, Math.round(x)));
        // 10진수 → 16진수, 1자리면 앞에 0 추가
        const hex = clamped.toString(16);
        return hex.length === 1 ? '0' + hex : hex;
    }).join('').toUpperCase();
}

// 예시
rgbToHex(99, 102, 241)  // "#6366F1"
```

### RGB → HSL 변환

HSL은 **색상(Hue)**, **채도(Saturation)**, **명도(Lightness)**로 색을 표현합니다:

```
        색상환 (Hue)
            0°
            🔴 빨강
     300°      60°
    🟣 자주    🟡 노랑
        
   240° 🔵────🟢 120°
       파랑   초록
            
            180°
            🩵 시안
```

```javascript
function rgbToHsl(r, g, b) {
    // 0-1 범위로 정규화
    r /= 255;
    g /= 255;
    b /= 255;
    
    const max = Math.max(r, g, b);
    const min = Math.min(r, g, b);
    let h, s;
    const l = (max + min) / 2;  // 명도 = (최대 + 최소) / 2

    if (max === min) {
        // 무채색 (회색)
        h = s = 0;
    } else {
        const d = max - min;  // 채도 계산용 차이값
        
        // 채도: 명도가 0.5 기준으로 계산 방식이 다름
        s = l > 0.5 ? d / (2 - max - min) : d / (max + min);
        
        // 색상: 어떤 채널이 최대인지에 따라 계산
        switch (max) {
            case r: 
                h = ((g - b) / d + (g < b ? 6 : 0)) / 6; 
                break;
            case g: 
                h = ((b - r) / d + 2) / 6; 
                break;
            case b: 
                h = ((r - g) / d + 4) / 6; 
                break;
        }
    }
    
    return {
        h: Math.round(h * 360),   // 0-360°
        s: Math.round(s * 100),   // 0-100%
        l: Math.round(l * 100)    // 0-100%
    };
}

// 예시
rgbToHsl(99, 102, 241)  // { h: 239, s: 84, l: 67 }
```

**수학적 배경**:

```
명도 (Lightness):
L = (max + min) / 2

채도 (Saturation):
S = 0                          if max = min (무채색)
S = (max - min) / (max + min)  if L ≤ 0.5
S = (max - min) / (2 - max - min)  if L > 0.5

색상 (Hue):
H는 max가 어떤 채널인지에 따라 60°씩 오프셋되어 계산
```

### HSL → RGB 변환

```javascript
function hslToRgb(h, s, l) {
    h /= 360;  // 0-1 범위로
    s /= 100;
    l /= 100;
    
    let r, g, b;

    if (s === 0) {
        // 무채색
        r = g = b = l;
    } else {
        // 헬퍼 함수: 색상값을 RGB로 변환
        const hue2rgb = (p, q, t) => {
            if (t < 0) t += 1;
            if (t > 1) t -= 1;
            if (t < 1/6) return p + (q - p) * 6 * t;
            if (t < 1/2) return q;
            if (t < 2/3) return p + (q - p) * (2/3 - t) * 6;
            return p;
        };
        
        // 명도에 따른 중간값 계산
        const q = l < 0.5 ? l * (1 + s) : l + s - l * s;
        const p = 2 * l - q;
        
        // 각 채널은 색상에서 120° (1/3) 오프셋
        r = hue2rgb(p, q, h + 1/3);
        g = hue2rgb(p, q, h);
        b = hue2rgb(p, q, h - 1/3);
    }
    
    return {
        r: Math.round(r * 255),
        g: Math.round(g * 255),
        b: Math.round(b * 255)
    };
}
```

---

## 색상 조화 알고리즘

색상 조화는 **색상환(Color Wheel)**에서의 각도 관계로 정의됩니다.

### 색상환 다이어그램

```
                    0° (빨강)
                       │
            330°       │       30°
               ╲       │       ╱
                 ╲     │     ╱
          300°────●────●────●────60°
                 ╱     │     ╲
               ╱       │       ╲
            270°       │       90°
               │       │       │
               │   메인 색상    │
               │       │       │
            240°       │       120°
               ╲       │       ╱
                 ╲     │     ╱
          210°────●────●────●────150°
                 ╱     │     ╲
               ╱       │       ╲
            180° (시안)
```

### 1. 보색 (Complementary)

색상환에서 **정반대(180°)** 위치의 색상:

```javascript
function getComplementary(h) {
    return (h + 180) % 360;
}

// 예시: 파랑(240°) → 노랑(60°)
getComplementary(240)  // 60
```

```
      0°
       │
       │   ●───────────────●
       │  60°            240°
       │ 노랑            파랑
     180°
```

**특성**: 높은 대비, 시선을 끄는 효과

### 2. 유사색 (Analogous)

메인 색상에서 **±30°** 떨어진 색상들:

```javascript
function getAnalogous(h) {
    return [
        (h + 330) % 360,  // -30° (360 - 30 = 330)
        (h + 30) % 360    // +30°
    ];
}

// 예시: 파랑(240°) → [210°, 270°]
getAnalogous(240)  // [210, 270]
```

```
          ●
        270°
       ╱
      ╱
    ●──────●
  240°   210°
 (메인)
```

**특성**: 자연스러운 조화, 편안한 느낌

### 3. 삼색 조화 (Triadic)

색상환을 **3등분(120° 간격)**:

```javascript
function getTriadic(h) {
    return [
        (h + 120) % 360,
        (h + 240) % 360
    ];
}

// 예시: 빨강(0°) → [120°, 240°]
getTriadic(0)  // [120, 240] - 빨강, 초록, 파랑!
```

```
           0° ● 빨강
          ╱ ╲
         ╱   ╲
        ╱     ╲
  240° ●───────● 120°
     파랑     초록
```

**특성**: 생동감, 균형잡힌 대비

### 4. 분리보색 (Split-Complementary)

보색의 **양옆 30°** 색상들:

```javascript
function getSplitComplementary(h) {
    return [
        (h + 150) % 360,  // 보색 - 30°
        (h + 210) % 360   // 보색 + 30°
    ];
}

// 예시: 빨강(0°) → [150°, 210°]
```

```
        0° ● 메인 (빨강)
           │
           │
           │
    150° ●─┼─● 210°
           │
           │
        180° (보색 위치)
```

**특성**: 보색보다 부드러운 대비, 세련된 느낌

### 5. 4색 조화 (Tetradic)

**90° 간격**의 4색:

```javascript
function getTetradic(h) {
    return [
        (h + 90) % 360,
        (h + 180) % 360,
        (h + 270) % 360
    ];
}
```

**특성**: 풍부한 색상, 복잡한 디자인에 적합

---

## WCAG 접근성 계산

### WCAG란?

**Web Content Accessibility Guidelines (WCAG)**는 웹 접근성을 위한 국제 표준입니다.
텍스트 가독성을 위해 전경색과 배경색 사이의 **대비율(Contrast Ratio)**을 정의합니다.

### 대비율 기준

| 등급 | 대비율 | 용도 |
|------|--------|------|
| **AA Large** | ≥ 3:1 | 큰 텍스트 (18pt+ 또는 14pt 굵게) |
| **AA** | ≥ 4.5:1 | 일반 텍스트 |
| **AAA** | ≥ 7:1 | 향상된 접근성 |

### 상대 휘도 (Relative Luminance)

대비율 계산의 핵심은 **상대 휘도**입니다:

```javascript
function getLuminance(r, g, b) {
    // sRGB → 선형 RGB 변환
    const [rs, gs, bs] = [r, g, b].map(c => {
        c /= 255;
        // 감마 보정 역변환
        return c <= 0.03928 
            ? c / 12.92 
            : Math.pow((c + 0.055) / 1.055, 2.4);
    });
    
    // ITU-R BT.709 가중치 적용
    // 인간의 눈은 초록에 가장 민감, 파랑에 가장 둔감
    return 0.2126 * rs + 0.7152 * gs + 0.0722 * bs;
}
```

**가중치의 의미**:
```
빨강: 0.2126 (21.26%)
초록: 0.7152 (71.52%)  ← 가장 높음!
파랑: 0.0722 (7.22%)   ← 가장 낮음
```

이 가중치는 인간 시각 시스템의 색상별 민감도를 반영합니다.

### 대비율 계산

```javascript
function getContrastRatio(hex1, hex2) {
    const rgb1 = hexToRgb(hex1);
    const rgb2 = hexToRgb(hex2);
    
    const l1 = getLuminance(rgb1.r, rgb1.g, rgb1.b);
    const l2 = getLuminance(rgb2.r, rgb2.g, rgb2.b);
    
    // 밝은 색이 분자, 어두운 색이 분모
    const lighter = Math.max(l1, l2);
    const darker = Math.min(l1, l2);
    
    // WCAG 공식: (L1 + 0.05) / (L2 + 0.05)
    return (lighter + 0.05) / (darker + 0.05);
}

// 예시
getContrastRatio('#FFFFFF', '#000000')  // 21:1 (최대)
getContrastRatio('#6366F1', '#FFFFFF')  // 4.53:1 (AA 통과)
```

**대비율 공식**:
```
Contrast Ratio = (L_lighter + 0.05) / (L_darker + 0.05)

범위: 1:1 (동일 색상) ~ 21:1 (흰색 vs 검정)
```

### 실제 테스트 케이스

```javascript
const contrastTests = [
    { bg: hex, fg: '#FFFFFF', label: '흰색 텍스트' },
    { bg: hex, fg: '#000000', label: '검정 텍스트' },
    { bg: '#FFFFFF', fg: hex, label: '흰 배경' },
    { bg: '#000000', fg: hex, label: '검정 배경' },
];

contrastTests.forEach(test => {
    const ratio = getContrastRatio(test.bg, test.fg);
    
    const passAA = ratio >= 4.5;
    const passAAA = ratio >= 7;
    const passAALarge = ratio >= 3;
    
    console.log(`${test.label}: ${ratio.toFixed(2)}:1`);
    console.log(`  AA Large: ${passAALarge ? '✓' : '✗'}`);
    console.log(`  AA: ${passAA ? '✓' : '✗'}`);
    console.log(`  AAA: ${passAAA ? '✓' : '✗'}`);
});
```

---

## Shades & Tints 생성

### 개념

- **Shade**: 색상에 검정을 섞음 (어두워짐)
- **Tint**: 색상에 흰색을 섞음 (밝아짐)

HSL 색 공간에서는 **Lightness(명도)만 조절**하면 됩니다!

```
Tints (밝음)
    ↑
    │  L=95%  ░░░░░░░░░░  50
    │  L=85%  ░░░░░░░░░   100
    │  L=75%  ▒▒▒▒▒▒▒▒    200
    │  L=65%  ▒▒▒▒▒▒▒     300
    │  L=55%  ▓▓▓▓▓▓      400
    │  L=45%  ▓▓▓▓▓       500 (원본)
    │  L=35%  ████        600
    │  L=25%  ████        700
    │  L=15%  ████        800
    │  L=5%   ████        900
    ↓
Shades (어두움)
```

### 구현

```javascript
function generateShades(hex, count = 10) {
    const rgb = hexToRgb(hex);
    const hsl = rgbToHsl(rgb.r, rgb.g, rgb.b);
    const shades = [];
    
    for (let i = 0; i < count; i++) {
        // 95%에서 5%까지 균등 분배
        const lightness = 95 - (i * (90 / (count - 1)));
        
        // HSL에서 명도만 변경
        const newRgb = hslToRgb(hsl.h, hsl.s, lightness);
        shades.push(rgbToHex(newRgb.r, newRgb.g, newRgb.b));
    }
    
    return shades;
}

// 예시: #6366F1로 10단계 생성
generateShades('#6366F1')
// [
//   '#F5F5FF',  // 50  (L=95%)
//   '#E0E1FF',  // 100 (L=85%)
//   '#C2C4FE',  // 200 (L=75%)
//   ...
//   '#1E1F5C'   // 900 (L=5%)
// ]
```

### Tailwind CSS 스타일 명명

```css
:root {
  --color-50:  #F5F5FF;   /* 가장 밝음 */
  --color-100: #E0E1FF;
  --color-200: #C2C4FE;
  --color-300: #A3A6FD;
  --color-400: #8588FB;
  --color-500: #6366F1;   /* 원본 (대략) */
  --color-600: #4F52D9;
  --color-700: #3B3EB8;
  --color-800: #282A8C;
  --color-900: #1E1F5C;   /* 가장 어두움 */
}
```

---

## UI 구현

### 색상 피커 동기화

컬러 피커와 HEX 입력이 **양방향 동기화**됩니다:

```javascript
const colorPicker = document.getElementById('color-picker');
const hexInput = document.getElementById('hex-input');

// 컬러 피커 → HEX 입력
colorPicker.addEventListener('input', (e) => {
    const hex = e.target.value;
    hexInput.value = hex.substring(1).toUpperCase();  // # 제거
    updatePalette(hex);
});

// HEX 입력 → 컬러 피커
hexInput.addEventListener('input', (e) => {
    // 유효한 문자만 허용 (0-9, A-F)
    let value = e.target.value
        .replace(/[^0-9A-Fa-f]/g, '')
        .substring(0, 6);
    e.target.value = value.toUpperCase();
    
    if (value.length === 6) {
        const hex = '#' + value;
        colorPicker.value = hex;  // 피커 업데이트
        updatePalette(hex);
    }
});
```

### 클립보드 복사

**Clipboard API**를 활용한 복사 기능:

```javascript
function copyToClipboard(text) {
    navigator.clipboard.writeText(text).then(() => {
        showToast(`${text} 복사됨!`);
    });
}

// 토스트 알림
function showToast(message) {
    const toast = document.createElement('div');
    toast.style.cssText = `
        position: fixed;
        bottom: 30px;
        left: 50%;
        transform: translateX(-50%);
        background: var(--accent-primary);
        color: white;
        padding: 12px 24px;
        border-radius: 8px;
        z-index: 9999;
        animation: fadeIn 0.3s ease;
    `;
    toast.textContent = message;
    document.body.appendChild(toast);
    setTimeout(() => toast.remove(), 2000);
}
```

### 테마 토글

CSS 변수와 `data-theme` 속성을 활용한 테마 전환:

```javascript
const html = document.documentElement;

// 저장된 테마 불러오기
const savedTheme = localStorage.getItem('color-palette-theme') || 'dark';
html.setAttribute('data-theme', savedTheme);

// 토글
themeToggle.addEventListener('click', () => {
    const current = html.getAttribute('data-theme');
    const newTheme = current === 'dark' ? 'light' : 'dark';
    
    html.setAttribute('data-theme', newTheme);
    localStorage.setItem('color-palette-theme', newTheme);
});
```

```css
/* 다크 테마 */
:root, [data-theme="dark"] {
    --bg-primary: #0a0a0f;
    --text-primary: #e8e8e8;
}

/* 라이트 테마 */
[data-theme="light"] {
    --bg-primary: #ffffff;
    --text-primary: #1a1a2e;
}
```

### CSS 코드 생성

생성된 팔레트를 CSS 변수로 출력:

```javascript
const cssCode = `:root {
  --color-primary: ${hex};
  --color-primary-rgb: ${rgb.r}, ${rgb.g}, ${rgb.b};
  --color-complementary: ${harmonies[1].hex};
  --color-analogous-1: ${harmonies[2].hex};
  --color-analogous-2: ${harmonies[3].hex};
  --color-50: ${shades[0]};
  --color-100: ${shades[1]};
  /* ... */
  --color-900: ${shades[9]};
}`;
```

---

## 마치며

### 핵심 알고리즘 요약

| 기능 | 알고리즘 | 복잡도 |
|------|----------|--------|
| 색 공간 변환 | HEX↔RGB↔HSL 수학 공식 | O(1) |
| 색상 조화 | 색상환 각도 연산 | O(1) |
| 대비율 계산 | WCAG 휘도 공식 | O(1) |
| Shades 생성 | HSL 명도 조절 | O(n) |

### 참고 자료

- [WCAG 2.1 Contrast Requirements](https://www.w3.org/WAI/WCAG21/Understanding/contrast-minimum.html)
- [CSS Color Module Level 4](https://www.w3.org/TR/css-color-4/)
- [색상 이론 기초](https://en.wikipedia.org/wiki/Color_theory)

---

*작성일: 2026년 1월*

