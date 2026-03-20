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
윗줄: y=0-14
복도: y=15-18
아랫줄: y=19-34
방 X 시작점:  [0, 13, 26, 39, 52, 65]
방 너비:      [12, 12, 12, 12, 12, 13]  (마지막은 13)
※ 계단(스테어웰) 없음 — 1층 구조, 위아래 2열 배치
```

### 부서 수에 따른 배치 전략
```
1-6개:  모두 아랫줄에 배치. 윗줄은 cafeteria + lounge 고정
7-10개: 윗줄에 5개, 아랫줄에 나머지. 남은 슬롯은 special room
11-12개: 양쪽 줄 꽉 채움
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
윗줄: doorTile = { x: roomX+6, y: 14 }
아랫줄: doorTile = { x: roomX+6, y: 18 }
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
  // ─── 윗줄 ───
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
  // ─── 아랫줄 ───
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

## Step 8: daily-log 기록 규칙 삽입

Pixel Office의 "오늘의 현황" 패널은 `.company/daily-log.md`를 Qwen이 분석해서 표시한다.
이 파일이 자동으로 채워지려면, **프로젝트의 CLAUDE.md에 기록 규칙을 삽입**해야 한다.

### 삽입할 내용

대상 파일에 `.company/CLAUDE.md` 가 있으면 그 파일에, 없으면 새로 생성해서 아래 내용을 추가한다:

```markdown
## 오늘의 현황 기록 규칙

큰 업데이트, 블로그 기사 작성/공개, 중요한 리서치 등의 작업을 완료하면
`.company/daily-log.md`에 해당 날짜 섹션으로 기록한다.

### 기록 형식
## YYYY-MM-DD

- [블로그] 기사 제목 — 공개/하서기 (키워드, SEO점수 등)
- [리서치] 조사 내용 요약
- [업데이트] 프로젝트명 — 변경 내용 요약
- [결정] 중요 결정 사항
- [기타] 기타 중요 활동

### 기록 타이밍
- 블로그 기사를 공개하거나 하서기로 저장했을 때
- 중요한 리서치(트렌드 분석, 시장 조사 등)를 완료했을 때
- 프로젝트에 큰 업데이트(새 기능, 중대 버그 수정 등)가 있을 때
- CEO 결정이나 조직 변경이 있을 때

### 주의사항
- 사소한 변경(오타 수정, 미세 조정)은 기록하지 않는다
- 날짜는 절대 날짜(YYYY-MM-DD)로 기재한다
```

### daily-log.md 초기 파일

대상 폴더에 `daily-log.md`가 없으면 생성한다:

```markdown
# Daily Log

## YYYY-MM-DD

- [기타] Pixel Office 시각화 시작
```

### 동작 원리

```
Claude Code가 작업 완료 시 daily-log.md에 자동 기록
      ↓
Pixel Office 서버가 /api/daily-status 요청을 받음
      ↓
daily-log.md에서 오늘 날짜 섹션 추출
      ↓
Qwen 2.5가 3~6개 불릿 포인트로 요약
      ↓
하단 "오늘의 현황" 패널에 표시
```

※ Qwen이 없으면 로그에서 직접 항목을 추출해서 표시 (폴백)

---

## 게임 내 기능 상세

### 직원 상태 (Character States)

| 상태 | 설명 |
|------|------|
| IDLE | 데스크에서 대기 |
| WALKING | 이동 중 |
| DELIVERING | 서류 배달 중 |
| RETURNING | 배달 후 귀환 |
| WANDERING | 방 안/복도 배회 |
| CHATTING | 동료와 잡담 |
| READING | 독서 중 |
| COMPUTING | PC 작업 중 |
| CORRIDOR_WALK | 복도 이동 (카페/수면실) |
| MEETING | 회의 참석 중 |
| SUMMONED | CEO실로 소환됨 |

---

### 체력 (HP) 시스템

**초기값:**
- 시작 HP: `80 + random(0~20)` → 80~100
- 최대 HP: 100

**소모 로직:**

| 행동 | HP 소모 | 계산식 |
|------|---------|--------|
| PC 작업 | -3~6 | `3 + random(0~3)` |
| 서류 배달 | -4~7 | `4 + random(0~3)` |
| 회의 | -6~11 | `6 + random(0~5)` |
| 자연 소모 | -1 | 12초마다 (작업 중만) |

**회복 로직:**

| 행동 | HP 회복 | 계산식 | 소요 시간 |
|------|---------|--------|----------|
| 카페테리아 | +12~19 | `12 + random(0~7)` | 5~10초 |
| 낮잠 (일반) | +25~34 | `25 + random(0~9)` | 10초 |
| 수면실 (긴급) | +50~70 | `50 + random(0~19)` | 15~20초 |

**HP 바 색상:**
- HP > 60%: 초록 `0x44CC88`
- HP 30~60%: 노랑 `0xDDAA33`
- HP < 30%: 빨강 `0xCC4444`

**저HP 자동 행동:**
- HP ≤ 25: 35% 확률로 수면실 이동
- HP ≤ 45: 10% 확률로 수면실 이동
- HP > 45: 2% 확률로 수면실 이동

---

### 일상 행동 확률 분포

아이들 타이머: 2~6초 간격으로 랜덤 행동 결정.

**일반 직원:**

| 확률 범위 | 행동 |
|----------|------|
| 0% ~ loungeChance | 수면실 방문 (HP 의존) |
| ~ 20% | 방 안 배회 |
| ~ 32% | 복도 배회 |
| ~ 44% | 독서 📖 |
| ~ 56% | PC 작업 💻 |
| ~ 64% | 카페 (커피 ☕) |
| ~ 70% | 카페 (식사 🍽️) |
| 70%~ | 대기 애니메이션 |

**비서 특수 행동:**
- 10% 확률: CEO에게 커피 배달
- 5% 확률: CEO실 방문 잡담

**CEO 특수 행동:**
- 95% 확률: 사무실에 체류 (65% 대기 / 35% 스트레칭)
- 5% 확률: 외출 (50% 복도 / 50% 카페)

**PM 특수 대사:** 5% 확률로 위트 있는 대사 (11종)
**영업/마케팅 특수 대사:** 20% 확률로 부서 관련 대사

---

### 직원 프로필

직원 클릭 시 프로필 모달 표시:

| 항목 | 내용 |
|------|------|
| 이름 | 한국어 이름 (부서별 고정) |
| 부서 | 부서명 + 부서 색상 |
| 친밀도 | 0~100 (≥70 초록, ≥40 노랑, <40 빨강) |
| 연봉 | `(친밀도/50) × 3,000,000원` 으로 계산 |
| ENERGY 바 | 현재 HP |
| STRESS 바 | 100 - HP |

**직원 이름 목록:**

| 부서 | 이름 |
|------|------|
| research | 김민지, 이준혁 |
| marketing | 박서연, 최도윤 |
| blog | 정하은, 강시우, 윤채원, 임지호 |
| creative | 한소율, 오태양 |
| pm | 신유나, 조민재 |
| sales | 배지아, 류건우 |
| secretary | 문하린 |
| ceo | 稲邉舜太朗 |

**친밀도 시스템:**
- 초기값: 50
- 매일 -1 감쇠 (자동)
- 대화/상호작용으로 증가

---

### CEO실로 부르기 (Summon)

- 프로필 모달의 "CEO실로 부르기" 버튼 클릭
- 해당 직원이 현재 작업을 중단하고 CEO실로 이동
- 한 번에 1명만 소환 가능 (이미 소환 중이면 버튼 비활성화)
- CEO 본인은 소환 불가
- 도착 시 `employee-arrived-ceo` 이벤트 발생

---

### 회의 시스템

- 복수 직원을 회의실에 소집
- 회의 대화는 `/api/meeting-dialogue` 에서 Qwen LLM이 생성
- 턴 당 3초, + 5초 버퍼
- 회의 종료 시 `/api/meeting-minutes`로 회의록 생성
- 참석자는 회의 후 자동으로 데스크로 복귀

---

### 대화 기록

- 프로필 모달의 "대화 기록" 버튼으로 확인
- 직원별 대화 이력 저장 (서버 `/api/employee/:dept/:index`)
- 대화 수가 친밀도에 영향

---

### 숍 시스템

| 아이템 | 가격 |
|--------|------|
| 🪑 ERGONOMIC CHAIR | 300G |
| 🖥️ GAMING DESK | 500G |
| 🚰 WATER COOLER | 150G |
| 🪴 POTTED PLANT | 100G |
| 📋 WHITEBOARD | 250G |
| ☕ COFFEE MACHINE | 200G |

- 시작 잔고: 1,200G
- 이벤트 발생·태스크 완료로 화폐 적립

---

### 일일 브리핑 (오늘의 현황)

- `.company/daily-log.md`를 Qwen LLM이 요약
- 3~6개 불릿 포인트, 각 15자 이내
- 수동 새로고침 버튼 (↻)
- LLM 실패 시 로그에서 직접 항목 추출 (폴백)

---

### 말풍선 시스템

| 종류 | 대사 수 | 예시 |
|------|---------|------|
| 일상 잡담 (IDLE_CHATS) | 46종 | "오늘 트렌드 봤어?", "카페 새 메뉴 맛있다" |
| 잡담 응답 (IDLE_RESPONSES) | 25종 | "좋아요!", "저도요~" |
| 수면 전 (SLEEP_LINES) | 7종 | "피곤해 죽겠네…", "5분만 자야지" |
| 기상 후 (WAKE_LINES) | 7종 | "개운하다~~", "꿈에서 코딩했어" |
| 저HP (LOW_HP_LINES) | 8종 | "체력이 바닥이야...", "에너지 제로..." |

- 일상 잡담은 Qwen LLM이 문맥에 맞게 추가 생성 가능

---

### 방별 가구 (Room.js에서 자동 생성)

각 방은 부서 특성에 맞는 가구가 자동으로 배치된다.

| 방 | 가구 |
|-----|------|
| research | 화이트보드, 지구본, 실험 플라스크, 논문 더미 |
| marketing | 프로젝터 스크린, 트로피, 메가폰, 파이차트 |
| blog | 삼각대 카메라, 링라이트, 콘텐츠 보드 |
| cafeteria | 카운터, 테이블 4개, 커피머신 2대, 자판기, 화분 |
| creative | 이젤+캔버스, 물감 팔레트, 색상 견본, 디자인 도구 |
| secretary | 서류 캐비닛, 시계, 전화기, 수신함 트레이 |
| pm | 칸반 보드, 타임라인 차트, 커피머신 |
| sales | 목표 보드, 트로피 선반, 전화기, 악수 포스터 |
| meeting | 대형 테이블, 의자 6개, 프로젝터, 화이트보드, 정수기 |
| lounge | 침대 6개 (2행×3열), 무드등, 화분 |
| ceo | 대형 책상, 가죽 의자, 금색 명판, 책장, 화분 |

모든 가구는 Phaser.js Graphics API로 픽셀 단위 드로잉. 외부 이미지 파일 없음.

---

### 이벤트 매핑 (파일 변경 → 게임 이벤트)

| 파일 경로 | 이벤트 타입 | 게임 내 동작 |
|----------|-----------|-------------|
| research/trends/*.md | trend_delivery | 리서치→블로그·마케·영업·PM 서류 배달 |
| ceo/decisions/*.md | ceo_decision | CEO→관련 부서 지시서 전달 |
| pm/tickets/*.md | ticket_created | PM→담당 부서 이동 |
| blog/articles/*.md | article_published | 블로그→마케·비서 보고 |
| marketing/campaigns/*.md | content_plan | 마케→블로그 공유 |
| sales/proposals/*.md | sales_proposal | 영업→CEO 상신 |
| secretary/todos/*.md | todo_update | 비서→PM 전달 |

---

### LLM 설정

| 항목 | 값 |
|------|-----|
| 엔진 | Ollama (로컬) |
| 모델 | qwen2.5:1.5b |
| URL | http://localhost:11434 |
| 타임아웃 | 30초 |
| 비용 | 0원 (완전 무료) |
| 폴백 | 정형문 표시 (LLM 없이도 작동) |

---

### 협업 그래프

| 항목 | 값 |
|------|-----|
| 계산식 | `상호참조 × 0.25 + 담당배분 × 0.35 + 최신성 × 0.40` |
| 윈도우 | 14일 (시간 감쇠 적용) |
| 소스 | research/trends, ceo/decisions, pm/tickets, secretary/todos |

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
