# PetLog Frontend

## 🚀 배포 주소

**Production**: [https://petlog-nu.vercel.app](https://petlog-nu.vercel.app)

- **플랫폼**: Vercel
- **자동 배포**: GitHub main/dev 브랜치 푸시 시 자동 재배포
- **환경 변수**: Vercel Dashboard에서 관리

---

## 🛠️ 로컬 개발 환경

### 1. 프론트엔드 실행

```bash
# 의존성 설치
pnpm install

# 개발 서버 실행
pnpm run dev
```

접속: http://localhost:5173

### 2. 백엔드 서비스 실행 (Docker)

프론트엔드와 연동하려면 백엔드 서비스도 실행해야 합니다.

```bash
# deployment 폴더로 이동
cd ../deployment

# 모든 백엔드 서비스 실행
docker compose up -d

# 서비스 상태 확인
docker compose ps
```

**실행되는 서비스**:
- **Infrastructure** (7개): PostgreSQL, MongoDB, Kafka, Zookeeper, Milvus, etcd, MinIO
- **Backend** (4개): user-service, diary-service, social-service, petmate-service
- **Gateway**: api-gateway (포트 8000)

**API Gateway**: http://localhost:8000

### 3. 서비스 중지

```bash
cd ../deployment
docker compose down
```

---

## FASD 아키텍처 마이그레이션 구현 계획

PetLog 프론트엔드를 **Hybrid FASD (Feature-App-Shared-Design)** 구조로 재구성하여 MSA 스타일의 유지보수성과 백엔드 연동 유연성을 확보합니다.

## 현재 구조 분석

```
src/
├── App.tsx           # 라우팅 및 프로바이더 설정
├── main.tsx          # 엔트리 포인트
├── components/       # 79개 파일 (UI + 도메인 혼합)
│   ├── ui/           # 57개 shadcn/ui 컴포넌트
│   ├── shop/         # 5개 쇼핑 컴포넌트
│   └── *.tsx         # 16개 기타 컴포넌트
├── pages/            # 54개 페이지 파일
├── lib/              # 8개 유틸리티/API 파일
└── hooks/            # 2개 공통 훅
```

> [!WARNING]
> 현재 구조의 문제점:
> - 도메인 간 코드가 섞여 있어 특정 기능 수정 시 여러 폴더를 탐색해야 함
> - API 로직과 UI가 분리되어 있어 백엔드 연동 시 추적이 어려움
> - 팀원 간 코드 충돌 가능성 높음

---

## 제안하는 FASD 구조

```
src/
├── app/                      # (App) 앱 설정 및 라우팅
│   ├── App.tsx               # 라우터 + 프로바이더
│   ├── main.tsx              # 엔트리 포인트
│   └── routes.tsx            # 라우트 정의 (선택)
│
├── shared/                   # (Shared) 공통 모듈
│   ├── ui/                   # shadcn/ui 컴포넌트 (57개)
│   ├── lib/                  # 유틸리티 (utils.ts)
│   ├── hooks/                # 공통 훅 (use-mobile, use-toast)
│   └── components/           # 공통 레이아웃 (Header, Logo)
│
├── features/                 # (Feature) 도메인별 기능 모듈
│   ├── auth/                 # 인증/사용자
│   │   ├── components/
│   │   ├── hooks/
│   │   ├── api/
│   │   └── pages/
│   ├── diary/                # AI 다이어리/리캡
│   ├── healthcare/           # 건강 관리
│   ├── shop/                 # 쇼핑몰
│   ├── petmate/              # 펫메이트/산책메이트
│   ├── social/               # 피드/탐색
│   └── chatbot/              # 챗봇
│
└── pages/                    # (Pages) 라우트 페이지 (features 조립)
```

---

## Path Alias 업데이트

`tsconfig.json`에 경로 별칭 추가:

```json
{
  "compilerOptions": {
    "paths": {
      "@/*": ["./src/*"],
      "@/app/*": ["./src/app/*"],
      "@/shared/*": ["./src/shared/*"],
      "@/features/*": ["./src/features/*"]
    }
  }
}
```

---

## Verification Plan

### 빌드 테스트
```bash
cd c:\Final_Project\Frontend
pnpm run build
```
- 빌드 에러 없이 완료되어야 함

### 개발 서버 실행
```bash
cd c:\Final_Project\Frontend
pnpm run dev
```
- `http://localhost:5173`에서 정상 로딩 확인

---

## 참고: Feature 폴더 내부 컨벤션

각 feature 폴더는 다음 구조를 따릅니다:

```
features/[domain]/
├── api/           # API 호출 함수 (백엔드 연동)
├── components/    # 해당 도메인 전용 UI 컴포넌트
├── context/       # React Context (상태 관리)
├── hooks/         # 커스텀 훅
├── data/          # 목 데이터, 상수
├── pages/         # 라우트 페이지
└── index.ts       # Public exports (선택)
```

> [!TIP]  
> 이 구조의 장점:
> - **담당자 독립성**: 헬스케어 담당자는 `features/healthcare/` 폴더만 보면 됨
> - **백엔드 연동 용이**: API 파일이 도메인별로 분리되어 엔드포인트 추적 쉬움
> - **코드 충돌 최소화**: 각자 다른 feature 폴더에서 작업

## 향후 API 연동 시 사용법
1. import httpClient from '@/shared/api/http-client';

2. GET 요청
const pets = await httpClient.get<Pet[]>('/pets');

3. POST 요청
const newPet = await httpClient.post<Pet>('/pets', { name: '몽치' });
