# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 프로젝트 개요

영수증(JPG/PNG/PDF) 업로드 → Upstage Vision LLM 자동 파싱 → 지출 내역 관리 웹앱.
DB 없이 JSON 파일(`backend/data/expenses.json`)에 데이터를 저장하는 경량 구조.

## 주요 명령어

### 백엔드 (Python FastAPI)

```bash
cd backend
python -m venv venv
source venv/Scripts/activate  # Windows: venv\Scripts\activate
pip install -r requirements.txt
uvicorn main:app --reload
# → http://localhost:8000/docs (Swagger UI)
```

### 프론트엔드 (React + Vite)

```bash
cd frontend
npm install
npm run dev      # http://localhost:5173
npm run build    # dist/ 생성
```

### 환경변수 설정

```bash
# 루트 .env (백엔드용)
UPSTAGE_API_KEY=<your-upstage-api-key>

# frontend/.env.local (프론트 개발용)
VITE_API_BASE_URL=http://localhost:8000
```

## 아키텍처

```
frontend/src/
├── api/axios.js          # Axios 인스턴스 (VITE_API_BASE_URL 기반)
├── pages/
│   ├── Dashboard.jsx     # / — SummaryCard + FilterBar + ExpenseList
│   ├── UploadPage.jsx    # /upload — DropZone → OCR 파싱 → ParsePreview
│   └── ExpenseDetail.jsx # /expense/:id — 수정/삭제
└── components/           # Badge, Modal, Toast, ProgressBar 등

backend/
├── main.py               # FastAPI 앱, CORS, 라우터 등록
├── routers/
│   ├── upload.py         # POST /api/upload
│   ├── expenses.py       # GET/DELETE/PUT /api/expenses[/{id}]
│   └── summary.py        # GET /api/summary
├── services/
│   ├── ocr_service.py    # LangChain + ChatUpstage Vision LLM 연동
│   └── storage_service.py# expenses.json 읽기/쓰기/append
└── data/expenses.json    # 런타임 데이터 저장소 (배열 형태)
```

### OCR 처리 흐름

```
파일 업로드 → PIL/pdf2image → Base64 인코딩
  → LangChain Chain (ChatUpstage + PromptTemplate + JsonOutputParser)
  → 구조화 JSON → UUID 부여 → expenses.json append
```

### 데이터 스키마 핵심 필드

`id` (UUID v4), `store_name`, `receipt_date` (YYYY-MM-DD), `category`
(식료품|외식|교통|쇼핑|의료|기타), `items[]`, `total_amount`, `payment_method`

## API 엔드포인트

| Method | Path | 설명 |
|--------|------|------|
| POST | `/api/upload` | 영수증 업로드 및 OCR 파싱 |
| GET | `/api/expenses` | 목록 조회 (`?from=&to=` 날짜 필터) |
| DELETE | `/api/expenses/{id}` | 항목 삭제 |
| PUT | `/api/expenses/{id}` | 항목 수정 |
| GET | `/api/summary` | 합계 통계 (`?month=YYYY-MM`) |

## 배포 (Vercel)

`vercel.json`에서 프론트(`@vercel/static-build`) + 백엔드(`@vercel/python` + Mangum) 라우팅.

Vercel 환경변수 필수 등록: `UPSTAGE_API_KEY`

> **주의**: Vercel 서버리스는 파일 시스템이 요청 간 유지되지 않음.
> 백엔드에서 `VERCEL=1` 환경변수 감지 시 `/tmp/expenses.json` 자동 사용.
> 데이터 영속성이 필요하면 클라이언트 localStorage 병행 저장 또는 Railway/Render 배포 고려.

## 스타일 가이드

- **폰트**: `Pretendard` (fallback: `Noto Sans KR`)
- **Primary 색상**: `indigo-600` (#4F46E5)
- **카드**: `bg-white rounded-xl border border-gray-200 shadow-sm`
- **TailwindCSS 커스텀 애니메이션**: `slide-up`, `scale-in`, `fade-in` (`tailwind.config.js`에 keyframes 등록)

## 테스트용 영수증 샘플

`images/` 디렉토리에 이마트, 스타벅스, GS25, 롯데백화점, CGV 등 실제 영수증 이미지 포함.
