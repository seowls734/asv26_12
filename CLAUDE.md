# 도트몬 어드벤처 — CLAUDE.md

8비트 픽셀 스타일의 몬스터 수집·트레이너 배틀 RPG. **HTML 파일 하나**(`dotmon-adventure.html`)로 동작하며 빌드 과정·외부 라이브러리가 없다. PC(키보드)와 모바일(화면 터치 패드)을 모두 지원한다.

## 실행

- `dotmon-adventure.html`을 브라우저로 열면 바로 실행된다. 서버 불필요.
- 외부 리소스는 Google Fonts의 `Noto Sans KR` 하나뿐이다. 오프라인이면 시스템 한글 글꼴로 대체된다.
- 저장: `localStorage` 키 `dotmon-adventure-v1`. 센터 이용·배틀 종료 후 자동 저장, 메뉴 → 기록하기로 수동 저장.

## 조작

| 동작 | 키보드 | 모바일 |
|---|---|---|
| 이동 | 방향키 / WASD | 십자키 |
| 결정 (A) | Z · Space · Enter | A 버튼 |
| 취소 (B) | X · Esc · Backspace | B 버튼 |
| 메뉴 | C · M · Shift | MENU 버튼 |
| 거대화 | 배틀 기술 선택 화면에서 메뉴 키 | MENU 버튼 |

## 저작권 원칙 (중요)

- 실제 포켓몬 캐릭터·이름·로고·고유 용어를 쓰지 않는다. 몬스터 12종은 모두 오리지널 디자인이다.
- "다이맥스/거다이맥스" 대신 **거대화/초거대화**, "몬스터볼" 대신 **캡슐**, "이상한사탕" 대신 **레벨업 젤리**처럼 고유 용어를 쓴다.
- 새 몬스터를 추가할 때도 기존 작품의 캐릭터를 닮게 그리지 않는다.

## 화면·렌더링 규칙

- 내부 해상도 **240×176** 논리 픽셀, 캔버스는 2배(480×352)로 그리고 CSS `image-rendering: pixelated`로 확대한다. `ctx.setTransform(2,…)`이 기본 상태(`resetT()`).
- 픽셀 글자: `text(str,x,y,color,size)`는 글자를 2배 해상도로 그린 뒤 알파를 임계값 처리해 계단 모양 가장자리를 만든다. 결과는 `tcache`에 캐시된다. 크기 단위는 논리 픽셀(보통 7~9).
- 한국어 조사: `josa(단어,'을','를')`처럼 받침에 따라 자동 선택한다. 메시지에 이름을 넣을 때 반드시 사용.
- 대화창은 `box()` + `wrap()`(폭 214 기준 줄바꿈), 커서는 `arrowR()` / `arrowD()`.
- 색은 `INK`, `SUB`, `PAPER`, 타입색 `TYPES`, 타일색 `C`에 모여 있다.

## 코드 구조 (스크립트 안 섹션 순서)

1. **core** — 상수, `rect/text/box`, 글꼴 로딩
2. **sprites** — `SP`(종 데이터), `MOVES`, `CHART`(상성), `ITEMS`, `GMAX`(초거대화), 이미지 캐시 `IMG / SIL(실루엣) / GIMG(초거대 색)`, 주인공 `PL / PIMG`
3. **tiles & map** — 타일 이미지 `TL`, 지도 `MAP`(104×70)을 `fill / put / bld` 로 코드로 생성, `SIGNS`, `TRAINERS`, `VILLAGERS`, `HOUSES`, `LEGENDS`
4. **state** — 세이브 객체 `G`, 플레이어 `P`, `stats()`, `mkMon()`, `newGame()`, `save / loadSave / applySave`
5. **input** — `held`(누르고 있는 방향), `pressed`(이번 프레임에 눌린 버튼, 프레임 끝에 비움)
6. **audio** — WebAudio 비프음 `beep / seq / sfx.*`. 첫 입력 이후에만 재생
7. **async helpers** — 게임 흐름은 `async/await`로 쓴다
   - `msg(text, auto)` 대화창 표시, A로 넘김 (auto>0이면 자동 진행)
   - `choose(options, {x,y,w,cancel,start})` 선택지 → 인덱스(취소 시 -1)
   - `wait(n)`, `waitUntil(fn)` 프레임 단위 대기
   - `transition()`, `fadeIn()`, `fx`(flash 등) 화면 전환 효과
   - 오버레이 스택 `overlays` — 맨 위 오버레이만 입력을 받는다
8. **modes** — `modes.title / starter / world / battle` 각각 `update()`와 `draw()`. 입력 우선순위: 대화창 > 오버레이 > 현재 모드
9. **world** — 격자 이동(8프레임/칸), 문 진입 이벤트(`D` 센터, `m` 상점, `d·e` 체육관, `h` 집), 트레이너 시야 판정, 지역 이름 토스트, 메뉴(몬스터·도감·가방·지도·기록하기·소리)
10. **battle** — 상태 `Bt`, `doTurn()` → `attack()` → `checkFaints()` 흐름, 포획 `throwBall()`, 경험치 `gainExp / levelCheck`, 거대화 `startGiant / endGiant / clearGiant`
11. **loop** — 60fps 고정 스텝 `step()` + `render()`

## 게임 데이터 요약

- **몬스터 12종**: 일반 8 (`fox, seal, bunny, bird, shroom, fly, jelly, crab`) + 전설 4 (`astra, marea, silva, ignis`). 도감 순서는 `ORDER`.
- **타입 5종**: 노말·불꽃·물·풀·전기. 상성은 `CHART[공격][방어]`.
- **능력치**: `stats(m)` — HP = ⌊종족HP×2×Lv/50⌋+Lv+10 (거대화 중 ×2). 필요 경험치 `expFor(L)=⌊L³×0.8⌋`.
- **대미지**: ((2Lv/5+2)×위력×공격/방어)/50+2 × 자속 1.5 × 상성 × 급소 1.5 × 난수 0.85~1.
- **포획률**: (3·최대HP−2·현재HP)/(3·최대HP) × 종 `rate` × 1.6 × 캡슐 배율(슈퍼 1.8). 흔들림 4번 판정.
- **코인**: 시작 1000, 트레이너 상금 = 최고 레벨 × 60 (관장 ×150), 야생 = Lv×12, 전멸 시 1/4 잃음.
- **보관함**: 최대 `BOX_CAP`=120, 쪽당 24. 포획할 때마다 동료·보관함을 레벨 내림차순 정렬(`sortByLevel`).

## 지역

| 지역 | 좌표(대략) | 야생 Lv | 비고 |
|---|---|---|---|
| 도트 마을 | x0–31, y0–23 | 2–9 | 시작, 센터·상점·도트 체육관(관장 도윤) |
| 솔바람 숲 / 바위 들판 | x32–62, y0–22 | 8–13 | 산들 마을(센터·상점) |
| 반짝 호숫가 / 갈대 벌판 | y24–44 | 12–15 | 호수 섬 = 아스트라 (다리 해금) |
| 모래 언덕길 | y46–55 | 14–18 | 노을 마을(SW), 별빛 도시(SE, 별빛 체육관·관장 하늘) |
| 전설의 땅 | x64–103 | 18–24 | 불꽃 산(이그나) · 고대의 숲(실바린) · 심해 해안(마레아) |

## 진행 흐름

1. 파트너 선택(불꼬리 / 퐁당이 / 새순이, 모두 초거대화 가능)
2. 북부 트레이너 8명 + 관장 도윤 → 도트 배지 + **거대화 밴드**
3. 배지 + 북부 8명 전승 → 호수 섬 다리 열림 → **아스트라**
4. 남부 트레이너 2명 + 관장 하늘 → 별빛 배지 → **마레아**
5. 일반 8종 포획 → **실바린**, 배지 2개 + 아스트라 포획 → **이그나**

## 자주 하는 수정 방법

- **몬스터 추가**: `SP`에 `{n,t,hp,atk,def,spd,exp,rate,learn:[[레벨,'기술id']],d,pal,px}` 추가. `px`는 **12×12 문자열 배열**(`.`=투명, 그 외 문자는 `pal`의 색). `ORDER`에 넣고, 야생 출현은 `wildFor()`에 추가. 전설이면 `legend:true`.
- **기술 추가**: `MOVES`에 `{n,t,p,a}`. `a:101`이면 반드시 명중.
- **트레이너 추가**: `TRAINERS`에 `{id,name,x,y,dir,pal,team:[['종',Lv]],hello,lose,after}`. 전설 해금 조건에서 빼려면 `extra:true`, 마지막 몬스터 거대화는 `giant:true`. 서 있는 칸은 이동 가능한 타일이어야 한다.
- **마을 주민**: `VILLAGERS`에 `{name,x,y,dir,pal,lines:[…]}`. 집 문 대사·선물은 `HOUSES['x,y']={t, gift:['아이템id',개수]}`.
- **건물**: `bld(x0,x1,지붕y,'지붕','벽',문x,'문')`. 문 문자: `D` 센터, `m` 상점, `h` 집, `d/e` 체육관.
- **전설 추가**: `LEGENDS`에 `{sp,x,y,lv,cond,place,spark,cry,bg,unlock,near,roar,appear,gone}`. `cond()`가 참이 되면 맵에 나타나고, 배틀 종료 후 `legendNotices()`가 해금 메시지를 띄운다.
- **아이템 추가**: `ITEMS`와 `ITEM_ORDER`에 추가하고 `use` 종류(`ball/heal/revive/jelly`)를 고른다. 세이브 호환을 위해 `applySave`가 없는 키를 0으로 채운다.
- **세이브 호환**: `G`에 새 필드를 넣으면 `newGame()`과 `applySave()` 양쪽에 기본값을 넣는다.

## 검증 방법

```bash
# 스크립트 문법 검사
sed -n '/<script>/,/<\/script>/p' dotmon-adventure.html | sed '1d;$d' > /tmp/g.js && node --check /tmp/g.js
```

- Playwright로 페이지를 열고 키 입력(`z`, 방향키)으로 진행하며 `canvas` 스크린샷을 찍어 확인한다.
- 새 지역을 만든 뒤에는 시작점(16,10)에서 BFS로 모든 문·NPC·전설 옆 칸에 닿을 수 있는지 검사한다(`tileAt`, `walkable`, `npcAt`, `legendAt` 사용).
- 콘솔 오류(`pageerror`)가 없어야 한다.
