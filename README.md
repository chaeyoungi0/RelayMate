# RelayMate

팀원들이 한 명씩 "바통(baton)"을 이어받아 함께 거리를 채워나가는 팀 이어달리기 러닝 앱입니다.

혼자 뛰는 러닝 앱과 달리, 한 팀 안에서 한 번에 한 사람만 바통(=러닝 권한)을 가질 수 있고, 그 사람이 뛴 거리가 팀의 누적 거리에 더해집니다. 팀을 만들거나 나와 성향이 맞는 팀에 들어가고, GPS로 실시간 경로를 기록하며, 팀 간 월간 랭킹으로 경쟁하는 것이 이 앱의 핵심입니다.

## Core Features

- **팀 생성 / 매칭 기반 팀 가입** — 활동 시간대, 목표 페이스, 목표 거리, 활동 요일을 입력하면 이를 벡터로 변환해 각 팀과의 매칭률(%)을 계산하고, 매칭률이 높은 순으로 정렬해 보여줍니다. 팀 목록이 없을 경우 더미 팀 데이터를 생성해 화면을 채웁니다.
- **팀 채팅** — 가입한 팀 내에서 Firestore 실시간 구독(`onSnapshot`) 기반으로 메시지를 주고받습니다.
- **바통(Baton) 릴레이 러닝**
  - 팀당 바통은 한 번에 한 명만 보유할 수 있으며, 바통을 받은 뒤 48시간 안에 러닝을 시작해야 합니다. 시간 내 시작하지 않으면 자동으로 바통이 반납됩니다.
  - 러닝 중에는 GPS 위치를 2초 간격으로 수집하고, 정확도(accuracy)가 25m 이내인 값만 신뢰합니다. 한 번에 0.5m~50m 사이로 이동한 경우만 실제 이동 거리로 인정해, GPS 튐이나 차량 이동 등 비정상적인 거리 증가를 걸러냅니다.
  - 일시정지는 최대 1시간까지 가능하며, 초과 시 바통이 자동 반납됩니다.
  - 러닝을 종료하면 뛴 거리(km)가 팀의 누적 거리에 더해지고(덮어쓰지 않고 합산), 바통은 다음 사람에게 넘어갈 수 있도록 반납됩니다.
- **팀 랭킹** — 팀들의 이번 달 누적 거리를 기준으로 순위를 매겨 보여주며, 매월 1일 00:00에 초기화됩니다. 월 초에는 지난달 우승 안내가 표시됩니다.
- **마라톤 캘린더** — 국내 마라톤 대회 일정을 달력 형태로 보여주고, 대회 링크로 바로 이동할 수 있습니다. 사용자가 새 일정을 추가하거나 삭제하면 Firestore에 저장되어 기본 제공 일정과 함께 표시됩니다.
- **마이페이지(프로필)**
  - 닉네임, 프로필 사진 변경
  - 레벨/경험치(EXP) 및 누적 거리 표시, 이번 달 랭킹 순위와 예상 보상 안내(1~3위에 한해 리워드 문구 표시)
  - **AI 러닝 코치 챗봇** — Gemini(`gemini-2.5-flash`)에게 닉네임·레벨·누적 거리·현재 순위를 프롬프트로 함께 전달해, 사용자 맞춤형 러닝/마라톤 코칭 답변을 받습니다.
- **로그인 / 회원가입** — Firebase Authentication(이메일·비밀번호) 기반이며, 가입 시 레벨 1·경험치 0·누적 거리 0으로 사용자 문서가 생성됩니다.

## Architecture

**클라이언트 ↔ Firebase 직접 연동 구조**

별도의 자체 백엔드 서버 없이, React Native(Expo) 앱이 Firebase Authentication과 Firestore에 직접 연결되는 구조입니다. 팀, 사용자, 바통 상태, 채팅, 마라톤 일정이 모두 Firestore 컬렉션(`users`, `teams`, `teams/{id}/messages`, `marathons`)에 저장되고, 화면들은 `onSnapshot` 실시간 구독으로 상태를 동기화합니다.

**매칭 알고리즘**

`calculateMatchPct` 함수는 사용자와 팀의 [시간대, 페이스, 목표거리, 요일(월~일) 7개] 벡터 중 시간대·요일만 비교합니다. 페이스와 목표 거리는 매칭 점수 계산에는 반영되지 않습니다. 시간대가 다르면 40%, 활동 요일이 다를수록 최대 60%의 가중치가 더해져, **서로 다를수록 매칭률이 높게(70~99% 범위) 나오도록** 설계되어 있습니다. 코드 주석에도 이 방향성이 "다를수록 매칭률 상승"이라고 명시되어 있습니다.

**바통 상태 관리**

바통 보유자(`batonHolder`)는 팀 문서 하나에만 저장되는 단일 락(lock) 구조입니다. 바통을 가진 사람만 러닝을 시작할 수 있고, 러닝 종료·48시간 만료·1시간 초과 일시정지 세 가지 경우에 자동으로 반납되도록 되어 있습니다.

**캐릭터 성장(준비 중)**

## Tech Stack

- **프레임워크** — React Native 0.81 · Expo ~54 (expo-router, TypeScript)
- **인증/데이터베이스** — Firebase Authentication · Firestore (`firebase` SDK, 실시간 리스너 기반)
- **위치/지도** — `expo-location`, `react-native-maps`, `geolib` (GPS 기반 거리 계산 및 경로 표시)
- **AI** — `@google/generative-ai` (Gemini `gemini-2.5-flash`, 프로필 화면의 러닝 코치 챗봇)
- **UI/기타** — `@expo/vector-icons`, `react-native-reanimated`, `react-native-gesture-handler`, `expo-image-picker`
- **이미지 전처리(오프라인)** — Python · Pillow (`scripts/split_character.py`, 캐릭터 스프라이트 분할용)

## Getting Started

### 설치

```bash
npm install
```

### 실행

```bash
npx expo start
```

터미널에 뜨는 QR코드를 Expo Go 앱으로 스캔하거나, 아래 명령으로 특정 플랫폼을 바로 실행할 수 있습니다.

```bash
npm run ios      # iOS 시뮬레이터
npm run android  # Android 에뮬레이터
npm run web      # 웹
```

### 유의사항

- 러닝 기능은 위치 권한이 필요합니다. iOS는 `NSLocationWhenInUseUsageDescription` / `NSLocationAlwaysUsageDescription`, Android는 `ACCESS_FINE_LOCATION` / `ACCESS_COARSE_LOCATION` 권한이 앱에 설정되어 있습니다.
- Firebase 프로젝트와 Gemini API 키가 `firebaseConfig.ts` / `app/(tabs)/profile.tsx`에 직접 값으로 들어 있습니다. 자신의 Firebase 프로젝트로 포크해서 쓰려면 이 값들을 본인 프로젝트 값으로 교체해야 합니다.

### 린트

```bash
npm run lint
```

## Project Structure

```
RelayMate/
├── app/
│   ├── _layout.tsx              # 루트 레이아웃, Firebase Auth 상태에 따른 라우팅 가드
│   ├── login.tsx                # 로그인 화면
│   ├── signup.tsx               # 회원가입 화면 (Firebase Auth + 초기 유저 문서 생성)
│   ├── modal.tsx
│   └── (tabs)/
│       ├── _layout.tsx          # 하단 탭 구성 (Home / Character / Running / Ranking / Marathon / My Page)
│       ├── index.tsx            # 홈: 팀 생성/매칭 가입, 팀 채팅
│       ├── running.tsx          # 러닝: 바통 관리, GPS 트래킹, 지도
│       ├── ranking.tsx          # 팀 랭킹 (월간 누적 거리 기준)
│       ├── marathon.tsx         # 마라톤 대회 캘린더
│       ├── profile.tsx          # 마이페이지, 레벨/EXP, Gemini 러닝 코치 챗봇
│       ├── character.tsx        # 캐릭터 성장 화면 (현재 빈 파일, 미구현)
│       └── explore.tsx          # Expo 기본 템플릿 화면 (탭에서는 숨김 처리됨)
├── components/                  # 공통 UI 컴포넌트 (themed-text, parallax-scroll-view 등)
├── constants/theme.ts           # 색상/폰트 등 테마 상수
├── hooks/                       # 컬러 스킴 등 커스텀 훅
├── scripts/
│   ├── reset-project.js         # Expo 기본 프로젝트 리셋 스크립트
│   └── split_character.py       # 캐릭터 이미지를 10단계로 분할·배경 투명화하는 전처리 스크립트
├── firebaseConfig.ts            # Firebase 초기화 (Auth, Firestore)
├── app.json                     # Expo 앱 설정
└── package.json
```
