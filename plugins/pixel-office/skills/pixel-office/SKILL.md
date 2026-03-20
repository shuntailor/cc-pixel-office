---
name: pixel-office
description: >
  지정한 폴더 구조를 분석해서 폴더 1개 = 방 1개로 동적 DeptConfig를 생성하고,
  Pixel Office를 그 구조에 맞게 시각화해서 시작하는 스킬.
trigger: /pixel-office
---

# Pixel Office — 폴더 시각화 스킬

## 이 스킬이 하는 것

어느 PC에서든:
1. 지정한 폴더를 스캔해서 서브폴더 목록을 가져온다
2. 서브폴더 수·파일 수를 분석해서 **동적으로 DeptConfig.js를 생성**한다
3. 기존 `DeptConfig.js`를 백업한 후 새 파일로 교체한다
4. `pixel-office --path [대상폴더]` 로 시작한다

---

## Step 1: pixel-office 위치 확인

`pwd` 로 현재 디렉토리를 확인한다.

현재 디렉토리에서 다음 중 하나를 찾는다:
- `./client/systems/DeptConfig.js` が存在する → **현재 디렉토리가 pixel-office 루트**
- `./pixel-office/client/systems/DeptConfig.js` が存在する → **하위 폴더에 pixel-office 존재**

どちらでもない場合:
> pixel-office 폴더를 찾지 못했습니다.
> pixel-office 루트 디렉토리 또는 그 부모 디렉토리에서 실행해주세요.

---

## Step 2: 시각화 대상 폴더 확인 (AskUserQuestion)

> 어느 폴더를 시각화할까요?
> 1. [현재 디렉토리]/.company/ — 회사 조직 폴더
> 2. 현재 디렉토리 전체 (서브폴더 = 방)
> 3. 직접 입력 (Other)

---

## Step 3: 폴더 구조 분석

지정된 폴더에서 다음을 수행한다:

```bash
# 서브폴더 목록 취득
ls -d [대상폴더]/*/

# 각 폴더의 파일 수 카운트 (마크다운 파일 기준)
find [대상폴더] -mindepth 2 -maxdepth 3 -name "*.md" | awk -F/ '{print $(NF-1)}' | sort | uniq -c | sort -rn
```

**분석 결과를 정리한다:**
- 폴더명 목록 (숨김 폴더 제외: `.`으로 시작하는 것 제외)
- 각 폴더의 `.md` 파일 수
- 최근 수정 파일 (직원 수 결정에 활용)

**제외하는 폴더:** `.`, `node_modules`, `.git`, `.venv`, `__pycache__`, `assets`

---

## Step 4: 방 레이아웃 생성 알고리즘

`references/layout-algorithm.md` 를 참조해서 아래 알고리즘으로 DEPARTMENTS 오브젝트를 생성한다.

### 기본 그리드 규칙
```
캔버스: 80×36 타일 (16px/타일)
Floor 2 (위): y=0-14
스테어웰:     col 30-33, y=14-19
Floor 1 (아래): y=19-34
방 X 시작점:  [0, 13, 26, 39, 52, 65]
방 너비:      [12, 12, 12, 12, 12, 13]  (마지막은 13)
```

### 부서 수에 따른 배치 전략
```
1-6개:  모두 Floor 1에 배치. Floor 2는 cafeteria + lounge 고정
7-10개: Floor 2에 5개, Floor 1에 나머지. 남은 슬롯은 special room
11-12개: 양 층 꽉 채움
```

### 직원 수(chars) 결정
```
파일 수 0:     chars=0 (특수룸)
파일 수 1-2:   chars=1
파일 수 3-5:   chars=2
파일 수 6-10:  chars=3
파일 수 11+:   chars=4
```

### 색상 팔레트 (순서대로 할당)
```javascript
const COLOR_PALETTE = [
  { color: 0x4A7C59, hex: '#4A7C59' },  // 녹색
  { color: 0xC25B5B, hex: '#C25B5B' },  // 빨강
  { color: 0xE8943A, hex: '#E8943A' },  // 주황
  { color: 0xB05BB0, hex: '#B05BB0' },  // 보라
  { color: 0x7B68EE, hex: '#7B68EE' },  // 인디고
  { color: 0xD4A017, hex: '#D4A017' },  // 금색
  { color: 0x6B8E9F, hex: '#6B8E9F' },  // 파랑회색
  { color: 0x8B6914, hex: '#8B6914' },  // 갈색
  { color: 0x3D7D3D, hex: '#3D7D3D' },  // 진녹색
  { color: 0xD4856A, hex: '#D4856A' },  // 살구색
];
```

### 데스크 위치 생성 (방 상대 좌표 → 절대 타일 좌표)
```
chars=1: [{x: roomX+6, y: floorBaseY+6}]
chars=2: [{x: roomX+3, y: floorBaseY+5}, {x: roomX+8, y: floorBaseY+8}]
chars=3: [{x: roomX+3, y: floorBaseY+4}, {x: roomX+8, y: floorBaseY+7}, {x: roomX+5, y: floorBaseY+10}]
chars=4: [{x: roomX+3, y: floorBaseY+5}, {x: roomX+9, y: floorBaseY+5}, {x: roomX+3, y: floorBaseY+10}, {x: roomX+9, y: floorBaseY+10}]
```

### 도어 위치
```
Floor 2: doorTile = { x: roomX+6, y: 14 }
Floor 1: doorTile = { x: roomX+6, y: 18 }
```

---

## Step 5: DeptConfig.js 생성

분석 결과로 새 `DeptConfig.js` 를 생성한다.

**사전 작업: 백업**
```bash
cp [pixel-office경로]/client/systems/DeptConfig.js \
   [pixel-office경로]/client/systems/DeptConfig.js.backup-[날짜]
```

**생성할 파일 형식 (기존 형식 그대로 유지):**
```javascript
/**
 * Dynamic configuration: generated from [대상폴더]
 * Folders found: [폴더 목록]
 * Generated at: [날짜]
 */

export const TILE = 16;
export const CANVAS_W = 80 * TILE;
export const CANVAS_H = 36 * TILE;

export const CORRIDOR_F2 = 15;
export const CORRIDOR_F1 = 18;

export const DEPARTMENTS = {
  // ─── Floor 2 ───
  [부서명]: {
    label: '[폴더명 또는 표시 이름]',
    color: 0x[hex],
    colorHex: '#[hex]',
    floor: 2,
    room: { x: [X], y: 0, w: [W], h: 14 },
    doorTile: { x: [X+6], y: 14 },
    deskTiles: [...],
    chars: [N],
  },
  // ─── Floor 1 ───
  ...
  // ─── Special rooms (cafeteria / lounge) ───
  cafeteria: {
    label: '카페테리아',
    color: 0xA0522D,
    colorHex: '#A0522D',
    floor: 2,
    room: { x: [빈슬롯X], y: 0, w: 12, h: 14 },
    doorTile: { x: [빈슬롯X+6], y: 14 },
    deskTiles: [],
    chars: 0,
  },
};

export const DEPT_NAMES = Object.keys(DEPARTMENTS);

export function tileToWorld(tx, ty) {
  return { x: tx * TILE + TILE / 2, y: ty * TILE + TILE / 2 };
}
```

**라벨 생성 규칙:**
- 폴더명에 알려진 한국어 매핑이 있으면 적용 (아래 참조)
- 없으면 폴더명 그대로 사용

```javascript
const FOLDER_LABEL_MAP = {
  secretary: '비서실', ceo: 'CEO', pm: 'PM', research: '리서치',
  marketing: '마케팅', blog: '블로그', note: 'note', sales: '영업',
  finance: '경리', creative: '크리에이티브', hr: '인사', engineering: '개발',
  design: '디자인', legal: '법무', support: '고객지원', data: '데이터',
  reviews: '리뷰', 'audit-log': '감사로그',
};
```

---

## Step 6: parser.js 패치 (선택)

`server/parser.js` 의 타입 추론 테이블에 새 폴더가 포함되지 않은 경우, 아래 기본 규칙을 추가한다:

```javascript
// 알 수 없는 dept/subdir → generic_update 타입으로 처리
if (!typeMap[`${dept}/${subdir}`]) {
  return 'generic_update';
}
```

`eventMapper.js` 에도 `generic_update` 이벤트 핸들러가 없으면 추가:
```javascript
case 'generic_update':
  return [{ type: 'document_delivery', from: dept, to: 'secretary', summary: title }];
```

---

## Step 7: pixel-office 시작

```bash
cd [pixel-office경로]
npm start -- --path [대상폴더] --name "[폴더명] Office"
```

브라우저가 자동으로 열리면:
> 시각화를 시작합니다!
> http://localhost:3101 에서 확인하세요.
>
> 생성된 방 수: [N]개
> [폴더명 → 방이름 목록 표시]
>
> 파일을 추가·수정하면 실시간으로 캐릭터가 움직입니다.

---

## 원래대로 복원하는 방법

```bash
cd [pixel-office경로]
cp client/systems/DeptConfig.js.backup-[날짜] client/systems/DeptConfig.js
```

---

## 중요 주의사항

- 기존 `DeptConfig.js` 는 반드시 백업 후 교체한다
- `node_modules`, `.git`, 숨김 폴더는 분석에서 제외한다
- 폴더가 12개를 넘으면 파일 수 상위 10개만 방으로 생성하고, 나머지는 무시 (cafeteria/lounge로 대체)
- `parser.js`, `eventMapper.js` 수정 시 기존 동작을 깨지 않도록 기존 케이스는 유지
- pixel-office가 실행 중이면 먼저 종료 (`Ctrl+C`) 후 재시작
