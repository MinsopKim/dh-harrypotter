# AGENTS.md — 이 저장소에서 일하는 AI 에이전트 지침

이 저장소는 영화 〈해리 포터와 마법사의 돌〉 지식그래프를 기반으로 한 **Next.js GraphRAG 챗봇** 프로젝트입니다.
(COLA173-19 디지털인문학응용 · B팀)

Google Docs로 작성한 프로젝트 기획안과 Google Sheets의 구조화 데이터를 Neo4j AuraDB 지식그래프로 옮겼고,
이 그래프를 탐색하고 그래프에 근거해 질문에 답하는 웹 챗봇을 만드는 것이 목표입니다.
Antigravity를 비롯한 모든 AI 코딩 에이전트는 아래 지침을 따라야 합니다.

## 1. 프로젝트 구조와 목표

* **프론트엔드 / API 서버:** 루트 디렉터리의 Next.js(App Router) 애플리케이션
* **지식그래프 DB:** Neo4j AuraDB (스키마 조회와 Cypher 실행 대상)
* **참고 자료:** `uploads/` 폴더의 프로젝트 기획안(`.md`)과 데이터 파일(`.csv`, `.cypher`)
* **데이터 범위:** 영화 1편 한정, 해리의 호그와트 입학부터 마법사의 돌 사건 해결까지
* **데이터 모델(기획안 기준):**
  * 5개 클래스: 인물(Person) · 집단(Group) · 장소(Place) · 사건(Event) · 사물(Object)
  * 14종 관계: `BELONGS_TO`(소속되다), `FAMILY_OF`(가족이다), `FRIEND_OF`(친구이다), `RIVAL_OF`(대립하다),
    `SERVES`(섬기다), `PROTECTS`(보호하다), `SUSPECTS`(의심하다), `PARTICIPATES_IN`(참여하다),
    `OCCURS_AT`(일어나다), `OWNS`(소유하다), `GIVES`(주다), `USED_IN`(사용되다),
    `GUARDS`(지키다), `LOCATED_IN`(위치하다)
* **주의:** 실제 AuraDB의 라벨 이름·대소문자, 속성 이름(`name`, `name_ko`, `name_eng` 등)은 위 설계와 다를 수 있습니다.
  쿼리를 작성하기 전에 반드시 `CALL db.schema.visualization()`, `CALL db.labels()`, `CALL db.relationshipTypes()`,
  `CALL db.propertyKeys()`로 실제 스키마를 확인하고, 실제 값에 맞춰 코드를 작성하세요. 추측으로 라벨이나 속성을 만들지 마세요.
* **에이전트의 주요 임무:**
  1. 현재 Neo4j 그래프 스키마와 프로젝트 기획안을 함께 분석해, 기획안의 핵심 질문에 답할 수 있는 그래프 탐색·질의 시나리오를 찾습니다.
  2. 자연어 질문을 Cypher 쿼리로 바꾸고(Text-to-Cypher), 조회된 그래프 결과에 근거해 답변을 생성하는 RAG 챗봇 파이프라인을 구현합니다.

## 2. 개발 서버 규칙 (`npm run build` 실행 금지)

* **항상 개발 서버(`npm run dev`)를 사용하세요.** HMR(Hot Module Replacement)로 변경 사항을 실시간으로 확인하며 작업합니다.
* **에이전트 세션 안에서는 절대 `npm run build`를 실행하지 마세요.** 프로덕션 빌드는 `.next` 폴더를 덮어써서 핫 리로드가 깨지고
  개발 서버가 불안정해집니다. 프로덕션 빌드는 배포 직전에 에이전트 작업 밖에서만 실행합니다.

## 3. Neo4j 및 데이터 안전 지침 (매우 중요)

* **읽기 전용 쿼리 우선:** Text-to-Cypher 로직을 구현하거나 MCP로 DB를 탐색할 때는 읽기 전용 쿼리(`MATCH`, `RETURN`, `CALL db.schema.*` 등)만 생성·실행합니다.
* **파괴적 작업 차단:** 사용자의 명시적 승인 없이 기존 그래프 데이터를 삭제하거나 덮어쓰는 쿼리(`DELETE`, `DETACH DELETE`, `DROP`, `SET`, `MERGE`, `CREATE` 등)를 실행하지 마세요.
  챗봇이 생성한 Cypher에 쓰기 키워드가 들어 있으면 실행 전에 차단하는 검증 로직을 넣으세요.
* **근거 기반 답변(환각 방지):** 챗봇은 Neo4j에서 조회한 노드와 관계만을 근거로 답해야 합니다. 조회 결과에 없는 내용은 추측하지 말고
  "그래프 데이터에서 찾을 수 없습니다"라고 답하게 하세요. 영화·원작 소설에 대한 일반 지식으로 빈칸을 채우지 마세요.

## 4. 환경 변수와 보안

* Neo4j 접속 정보(`NEO4J_URI`, `NEO4J_USERNAME`, `NEO4J_PASSWORD`, `NEO4J_DATABASE`)와 LLM API 키(`GEMINI_API_KEY`, `EXAONE_API_KEY`)는 **`.env.local`에만** 저장합니다.
* `.env.local`이 `.gitignore`에 포함되어 있는지 확인하고, 비밀 값을 GitHub 원격 저장소에 **절대** 커밋·푸시하지 마세요.
* 새 환경 변수를 추가하면 `.env.example`에는 키 이름만 적고, 값은 사용자가 직접 로컬에 입력하도록 안내하세요.

## 5. 의존성과 코딩 규칙

* 패키지(`neo4j-driver`, `@google/genai` 등)를 추가·업데이트할 때는 `package-lock.json`을 함께 맞추고, 필요하면 개발 서버를 재시작하세요.
* 컴포넌트, API 라우트, 유틸리티는 TypeScript(`.tsx` / `.ts`)로 일관되게 작성합니다.
* 사용자에게 보이는 UI 문구, 챗봇 시스템 프롬프트, 오류 메시지는 기본적으로 **한국어**로 작성합니다.
* 인물 이름 등은 화면에 한국어로 보여 주되, 그래프에 영어 이름만 있는 노드는 `name_ko` 같은 보조 속성이 있으면 그것을 우선 사용합니다.

## 6. 자주 쓰는 명령어

| 명령어 | 용도 |
| :--- | :--- |
| `npm run dev` | HMR이 켜진 Next.js 개발 서버 실행 (기본) |
| `npm run lint` | ESLint 코드 품질·문법 검사 |
| `npm run test` | 테스트 실행 (설정된 경우) |
| `npm run build` | **프로덕션 빌드 — 에이전트 세션에서는 실행 금지** |
