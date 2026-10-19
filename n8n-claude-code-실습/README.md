# n8n × Claude Code 실습 자료

2026-10-19 작성

## 준비물과 비용

n8n, n8n-mcp, 실습에 쓰는 API는 모두 무료이고 가입이나 API 키가 필요 없어요. 유료일 수 있는 건 Claude Code 하나뿐이에요. Claude Code는 Claude 유료 플랜(Pro 이상) 계정이 있어야 써요.

| 준비물 | 용도 | 비용 |
| --- | --- | --- |
| [Node.js](https://nodejs.org) 22 LTS | n8n-mcp 실행 (npx) | 무료 |
| 수업용 n8n 서버 (n8n.aitestbed.kr) | 워크플로우 도구, 브라우저로 접속 (설치 없음) | 무료 |
| [n8n-mcp](https://github.com/czlonkowski/n8n-mcp) | Claude Code와 n8n을 잇는 MCP 서버 | 무료 (오픈소스) |
| Claude Code (CLI 또는 데스크톱 앱) | 워크플로우를 대신 만들어 주는 AI | Claude 유료 플랜 필요 |
| [Open-Meteo](https://open-meteo.com) | 실습 1 날씨 데이터 | 무료, 키 없음 |
| [PokeAPI](https://pokeapi.co) | 실습 2 포켓몬 데이터 | 무료, 키 없음 |

수업 전에 수강생이 확인할 것:

- [ ] 터미널에서 `node -v`를 입력하면 v20 이상이 나온다
- [ ] 터미널에서 `claude --version`이 나온다 (Claude Code 설치 완료)
- [ ] Claude Code에 로그인되어 있다
- [ ] `https://n8n.aitestbed.kr`에 강사에게 받은 계정으로 로그인된다

## 실습 0: Claude Code에 n8n 연결하기 (20분)

목표는 Claude한테 말로 시켰는데 n8n 화면에 워크플로우가 생기는 걸 눈으로 확인하는 거예요. 연결 구조는 `Claude Code → n8n-mcp → 수업용 n8n 서버(n8n.aitestbed.kr)` 예요.

### 1단계. 수업용 n8n 서버에 로그인하기

1. 브라우저에서 `https://n8n.aitestbed.kr`에 접속해요. n8n을 내 PC에 설치할 필요는 없어요.
2. 강사에게 받은 계정으로 로그인해요.
3. 이 서버는 모든 수강생이 같이 써요. 워크플로우 이름과 웹훅 주소에는 꼭 내 영문 이니셜(예: cs)을 붙여서 다른 사람 것과 겹치지 않게 해요.

### 2단계. n8n API 키 만들기

1. n8n 화면 왼쪽 아래에서 **Settings → n8n API**로 가요.
2. **Create an API key**를 누르고 만들어진 키를 복사해 메모장에 붙여 둡니다. 다시 보여주지 않으니 꼭 복사해요.

API 메뉴가 안 보이면 서버 관리자(강사)가 n8n API를 막아 둔 상태예요. 수업 전에 관리자 설정에서 허용해 두세요.

### 3단계. Claude Code에 n8n-mcp 등록하기

1. 새 터미널을 열고 실습 폴더를 만들어 들어가요.

```
mkdir n8n-lab
cd n8n-lab
```

2. 아래 명령에서 `내API키`만 바꿔서 실행해요 (Mac/Linux).

```
claude mcp add n8n-mcp -e MCP_MODE=stdio -e LOG_LEVEL=error -e DISABLE_CONSOLE_OUTPUT=true -e N8N_API_URL=https://n8n.aitestbed.kr -e N8N_API_KEY=내API키 -- npx -y n8n-mcp
```

3. Windows는 마지막 부분만 `-- cmd /c npx -y n8n-mcp`로 바꿔요.

### 4단계. 연결 확인하기

1. 같은 폴더에서 `claude`를 실행하고 `/mcp`를 입력해요. `n8n-mcp`가 connected로 보이면 성공이에요.
2. Claude한테 이렇게 말해 봐요.

```
n8n 연결 상태 확인하고, 지금 n8n에 있는 워크플로우 목록 보여줘
```

### 미션: 첫 워크플로우 말로 만들기

```
n8n에 "헬로 클로드-내이니셜" 워크플로우를 만들어줘. 수동 실행 트리거 다음에 Edit Fields(Set) 노드로 message 필드에 "안녕 n8n, 나는 클로드야"를 넣어줘.
```

- [ ] n8n 화면을 새로고침하면 "헬로 클로드-내이니셜"이 보인다
- [ ] 열어서 **Execute workflow**를 누르면 message가 출력된다

## 실습 1: 날씨 보고 점심 메뉴 뽑아주는 봇 (25분)

브라우저 주소를 열면 지금 서울 날씨를 보고 오늘 점심 메뉴를 랜덤으로 추천해 주는 봇을 만들어요. 웹훅, 외부 API 호출, 코드, 응답까지 n8n의 핵심 노드 4개를 한 번에 연습해요.

워크플로우 모양: `Webhook → HTTP Request(Open-Meteo) → Code(메뉴 뽑기) → Respond to Webhook`

### 1단계. Claude에게 주문하기

```
n8n에 "점심 메뉴 봇" 워크플로우를 만들어줘.
1. Webhook 노드: GET, path는 lunch-내이니셜(예: lunch-cs), 응답은 Respond to Webhook 노드로 보내기
2. HTTP Request 노드: Open-Meteo에서 서울 현재 날씨 가져오기
   https://api.open-meteo.com/v1/forecast?latitude=37.57&longitude=126.98&current=temperature_2m,precipitation&timezone=Asia/Seoul
3. Code 노드: 비가 오면 국물류, 25도 이상이면 시원한 메뉴, 아니면 일반 메뉴 목록에서 하나를 랜덤으로 뽑기. 각 목록에 한식 메뉴 5개씩.
4. Respond to Webhook: "지금 서울 OO도! 오늘 점심은 OO 어때요?" 형식의 텍스트로 응답
다 만들면 워크플로우를 활성화해줘.
```

### 2단계. 실행해 보기

1. n8n 화면에서 워크플로우가 활성화됐는지 확인해요. 안 됐으면 오른쪽 위 Active 토글(버전에 따라 Publish 버튼)을 켜요.
2. 브라우저에서 `https://n8n.aitestbed.kr/webhook/lunch-내이니셜`를 열어요.
3. 새로고침할 때마다 메뉴가 바뀌면 성공이에요.

### 3단계. 말로 고치기 (보너스 미션)

n8n을 직접 만지지 말고 Claude에게만 말해서 고쳐 보세요. 한 번 고칠 때마다 n8n 화면을 새로고침해서 노드가 어떻게 바뀌었는지 확인해요.

- [ ] "점심 메뉴 봇에서 비 오면 우산 챙기라는 말도 붙여줘"
- [ ] "주소 뒤에 ?name=철수를 붙이면 '철수님, 오늘 점심은...'으로 부르게 해줘"
- [ ] "주소 뒤에 ?city=부산을 붙이면 부산 날씨로 추천하게 해줘. 좌표는 Open-Meteo 무료 지오코딩 API로 찾아줘"

## 실습 2: 내 이름으로 뽑는 포켓몬 카드 (30분)

이름을 넣으면 “나의 운명의 포켓몬”을 골라 그림이 들어간 카드 웹페이지로 보여 주는 워크플로우예요. 같은 이름은 항상 같은 포켓몬이 나오니까 옆자리 친구와 결과를 비교하면서 하면 재밌어요. 실습 1이 “만들기”였다면 실습 2는 Claude에게 말로 두세 번 고쳐 가며 완성하는 게 포인트예요.

워크플로우 모양: `Webhook → Code(이름→번호) → HTTP Request(PokeAPI) → Code(HTML 카드) → Respond to Webhook`

### 1단계. Claude에게 주문하기

```
n8n에 "내 포켓몬 카드" 워크플로우를 만들어줘.
1. Webhook 노드: GET, path는 pokemon-내이니셜(예: pokemon-cs), ?name= 쿼리로 이름을 받기. 응답은 Respond to Webhook 노드로.
2. Code 노드: 이름 글자들을 숫자로 바꿔 더해서 1~151 사이 번호를 만들기 (같은 이름이면 항상 같은 번호)
3. HTTP Request 노드: https://pokeapi.co/api/v2/pokemon-species/{번호} 에서 한국어 이름 가져오기
4. HTTP Request 노드: https://pokeapi.co/api/v2/pokemon/{번호} 에서 그림(sprites.other.official-artwork)과 타입 가져오기
5. Code 노드: "OO님의 운명의 포켓몬은 OO!" 제목, 그림, 타입이 들어간 예쁜 HTML 카드 만들기
6. Respond to Webhook: Content-Type을 text/html; charset=utf-8로 해서 HTML을 응답
다 만들면 활성화해줘.
```

### 2단계. 실행해 보기

1. 브라우저에서 `https://n8n.aitestbed.kr/webhook/pokemon-내이니셜?name=내이름`을 열어요.
2. 카드가 나오면 옆 사람 이름도 넣어 보고, 누가 제일 멋진 포켓몬을 뽑았는지 비교해요.
3. 글자가 깨지거나 HTML 코드가 그대로 보이면 Claude에게 “카드가 이렇게 보여, 고쳐줘”라고 화면 상황을 그대로 설명해요. 이것도 실습의 일부예요.

### 3단계. 말로 업그레이드하기

한 번에 하나씩 시키고, 매번 브라우저를 새로고침해서 바뀐 카드를 확인해요.

- [ ] "포켓몬 타입별로 카드 배경색을 다르게 해줘 (불꽃은 빨강, 물은 파랑...)"
- [ ] "HP, 공격, 방어, 스피드 능력치를 막대 그래프로 카드에 넣어줘"
- [ ] 도전: "?name=철수&vs=영희로 두 사람을 넣으면 두 카드를 나란히 보여주고 능력치 합이 높은 쪽을 승자로 발표해줘"

마지막에 n8n 화면에서 완성된 워크플로우를 열고, 노드를 하나씩 눌러 입력과 출력을 확인하며 Claude가 무엇을 만들었는지 해석해 봐요.

## 문제 해결 FAQ와 강사 팁

| 증상 | 해결 |
| --- | --- |
| `/mcp`에 n8n-mcp가 없거나 failed | `claude mcp add`를 한 폴더에서 `claude`를 실행했는지 확인. `claude mcp list`로 등록 여부 확인. 잘못 넣었으면 `claude mcp remove n8n-mcp` 후 다시 추가 |
| Windows에서 연결 실패 | 명령 끝을 `-- cmd /c npx -y n8n-mcp`로 등록했는지 확인 |
| Claude가 “API 연결 안 됨”이라고 함 | 서버에 로그인되는지, API 키를 빠짐없이 복사했는지 확인. URL은 `https://n8n.aitestbed.kr`까지만 씁니다 (끝에 `/home/workflows`나 `/api/v1` 붙이지 않기) |
| 웹훅 주소가 404 | 워크플로우가 활성화돼 있는지 확인. 편집 화면에서 테스트할 땐 `/webhook-test/...` 주소를 쓰고 먼저 Execute workflow를 눌러 둡니다 |
| 웹훅이 “이미 사용 중”이라며 활성화 안 됨 | 다른 수강생과 path가 겹친 거예요. path 뒤에 내 이니셜을 붙여 달라고 Claude에게 요청 |
| 한글이 깨져 보임 | Respond to Webhook의 Content-Type에 `charset=utf-8`을 넣어 달라고 Claude에게 요청 |
| 노드가 빨간색(에러) | n8n의 Executions 탭에서 에러 메시지를 복사해 Claude에게 그대로 붙여넣기 |

강사 팁:

- 실습 0에서 시간이 가장 많이 걸려요. Node.js와 Claude Code 설치, 수업용 서버 계정 발급은 수업 전에 끝내 두면 수업이 훨씬 매끄러워요.
- 서버를 다 같이 쓰기 때문에 워크플로우 이름과 웹훅 path에 이니셜을 붙이는 규칙을 시작할 때 꼭 강조하세요.
- Claude는 매번 조금씩 다른 워크플로우를 만들어요. 수강생마다 결과가 다른 게 정상이라고 미리 알려 주세요.
- “만들어 주면 끝”이 아니라, 완성된 노드를 열어 입력과 출력을 확인하는 시간을 꼭 주세요. 기본 n8n 강의 내용과 이어지는 부분이에요.
- n8n과 n8n-mcp는 업데이트가 잦아서 메뉴 이름이 바뀔 수 있어요. 수업 전에 한 번 직접 리허설해 보세요.
