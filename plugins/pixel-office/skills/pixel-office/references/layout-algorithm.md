# 방 레이아웃 생성 알고리즘 상세

## 그리드 좌표 기준

```
캔버스: 80타일 × 36타일 (1280px × 576px, 1타일=16px)

Floor 2 (위층):
  방 y 시작: 0
  방 h: 14
  복도 행: y=15-16 (실제 이동은 CORRIDOR_F2=15)

스테어웰:
  x: 30-33, y: 14-19

Floor 1 (아래층):
  방 y 시작: 19
  방 h: 15
  복도 행: y=17-18 (실제 이동은 CORRIDOR_F1=18)
```

## 방 X 위치 슬롯 (6개 슬롯/층)

| 슬롯 | x 시작 | 너비 | 도어 x |
|------|--------|------|--------|
| 0    | 0      | 12   | 6      |
| 1    | 13     | 12   | 19     |
| 2    | 26     | 12   | 32     |
| 3    | 39     | 12   | 45     |
| 4    | 52     | 12   | 58     |
| 5    | 65     | 13   | 71     |

## 부서 수에 따른 배치 전략

### Case 1: N ≤ 6
```
Floor 2: [cafeteria, lounge, lounge, lounge, lounge, lounge] (특수룸)
Floor 1: [dept0, dept1, ..., deptN-1, meeting, storage]
```

### Case 2: 7 ≤ N ≤ 10
```
Floor 2: dept0~dept4 (activity 높은 순 상위 5개)
Floor 2: 나머지 1슬롯 → cafeteria
Floor 1: dept5~deptN-1
Floor 1: 남는 슬롯 → meeting, storage (이 순서로)
```

### Case 3: N = 11
```
Floor 2: dept0~dept4 + cafeteria
Floor 1: dept5~dept10 + storage
```

### Case 4: N = 12
```
Floor 2: dept0~dept5 (cafeteria 없음)
Floor 1: dept6~dept11 (storage 없음)
lounge는 제거됨
```

### Case 5: N > 12
```
파일 수 기준 상위 10개만 선택
나머지 처리: Case 2 와 동일
```

## 특수룸 정의

```javascript
const SPECIAL_ROOMS = {
  cafeteria: { label: '카페테리아', color: 0xA0522D, colorHex: '#A0522D', chars: 0, deskTiles: [] },
  lounge:    { label: '수면실',    color: 0x2A2A5A, colorHex: '#2A2A5A', chars: 0, deskTiles: [] },
  meeting:   { label: '회의실',   color: 0x3D7D3D, colorHex: '#3D7D3D', chars: 0, deskTiles: [] },
  storage:   { label: '창고',     color: 0x6A6A5A, colorHex: '#6A6A5A', chars: 0, deskTiles: [] },
};
```

## 데스크 위치 계산 (절대 타일 좌표)

Floor 2 기준 (roomY=0):

| chars | deskTiles (절대 좌표) |
|-------|----------------------|
| 0     | []                   |
| 1     | [{x: roomX+6, y: 6}] |
| 2     | [{x: roomX+3, y: 5}, {x: roomX+8, y: 8}] |
| 3     | [{x: roomX+3, y: 4}, {x: roomX+8, y: 7}, {x: roomX+5, y: 10}] |
| 4     | [{x: roomX+3, y: 5}, {x: roomX+9, y: 5}, {x: roomX+3, y: 10}, {x: roomX+9, y: 10}] |

Floor 1 기준 (roomY=19):

| chars | deskTiles (절대 좌표) |
|-------|----------------------|
| 0     | []                   |
| 1     | [{x: roomX+6, y: 25}] |
| 2     | [{x: roomX+3, y: 24}, {x: roomX+8, y: 27}] |
| 3     | [{x: roomX+3, y: 23}, {x: roomX+8, y: 26}, {x: roomX+5, y: 29}] |
| 4     | [{x: roomX+3, y: 24}, {x: roomX+9, y: 24}, {x: roomX+3, y: 29}, {x: roomX+9, y: 29}] |

## 파일 수 → chars 매핑

| .md 파일 수 | chars |
|------------|-------|
| 0          | 0     |
| 1-2        | 1     |
| 3-5        | 2     |
| 6-10       | 3     |
| 11+        | 4     |

## 슬롯 X 위치 리스트

```javascript
const SLOT_X  = [0,  13, 26, 39, 52, 65];
const SLOT_W  = [12, 12, 12, 12, 12, 13];
```

## 알려진 폴더명 → 한국어 라벨 매핑

```javascript
const LABEL_MAP = {
  secretary: '비서실',   ceo: 'CEO',          pm: 'PM',
  research: '리서치',    marketing: '마케팅',  blog: '블로그',
  note: 'note',          sales: '영업',        finance: '경리',
  creative: '크리에이티브', hr: '인사',         engineering: '개발',
  design: '디자인',      legal: '법무',        support: '고객지원',
  data: '데이터',        reviews: '리뷰',      'audit-log': '감사로그',
  frontend: '프론트엔드', backend: '백엔드',   infra: '인프라',
  docs: '문서',          test: '테스트',       analytics: '분석',
};
```
