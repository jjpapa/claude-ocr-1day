# 🧾 Receipt Expense Tracker

영수증(JPG/PNG/PDF)을 업로드하면 **Upstage Vision LLM**이 자동으로 내용을 파싱하여 지출 내역을 관리해주는 경량 웹 애플리케이션입니다.

> 1일 스프린트 MVP — DB 없이 JSON 파일 기반으로 동작합니다.

---

## 기술 스택

| 구분 | 기술 |
|------|------|
| 프론트엔드 | React 18 + Vite + TailwindCSS + Axios |
| 백엔드 | Python FastAPI 0.136 |
| AI/OCR | Upstage `document-digitization-vision` (via LangChain 1.2) |
| 이미지 처리 | Pillow + pdf2image |
| 데이터 저장 | JSON 파일 (`backend/data/expenses.json`) |
| 배포 | Vercel (프론트 + 백엔드 서버리스) |

---

## 시작하기

### 1. 환경변수 설정

```bash
cp .env.example .env
# .env 파일에 Upstage API 키 입력
# UPSTAGE_API_KEY=your_key_here
```

### 2. 백엔드 실행

```bash
cd backend
python -m venv venv
source venv/Scripts/activate   # Windows
# source venv/bin/activate     # macOS/Linux
pip install -r requirements.txt
uvicorn main:app --reload
# → http://localhost:8000/docs (Swagger UI)
```

### 3. 프론트엔드 실행

```bash
cd frontend
npm install
npm run dev   # → http://localhost:5173
```

> 프론트엔드 개발 시 `frontend/.env.local` 파일에 `VITE_API_BASE_URL=http://localhost:8000` 추가

---

## 아키텍처

```
사용자 브라우저 (React + Vite + TailwindCSS)
        │  HTTP REST API
        ▼
FastAPI 백엔드
  ├─ POST /api/upload  → LangChain → Upstage Vision LLM → expenses.json 저장
  ├─ GET  /api/expenses              → 목록 조회 (날짜 필터)
  ├─ PUT/DELETE /api/expenses/{id}   → 수정 / 삭제
  └─ GET  /api/summary               → 합계 통계
        │
        ▼
GitHub → Vercel 자동 배포
```

### 백엔드 구조

```
backend/
├── main.py                  # FastAPI 앱, CORS, 라우터 등록
├── routers/
│   ├── upload.py            # POST /api/upload
│   ├── expenses.py          # GET, DELETE, PUT /api/expenses
│   └── summary.py           # GET /api/summary
├── services/
│   ├── ocr_service.py       # LangChain + ChatUpstage Vision LLM
│   └── storage_service.py   # expenses.json 읽기/쓰기
├── data/expenses.json
└── requirements.txt
```

### OCR 처리 흐름

```
파일 업로드 (JPG/PNG/PDF)
  → PIL / pdf2image → Base64 인코딩
  → LangChain Chain (ChatUpstage + PromptTemplate + JsonOutputParser)
  → 구조화 JSON 반환
  → UUID 부여 후 expenses.json에 append
```

---

## API 엔드포인트

| Method | Path | 설명 |
|--------|------|------|
| `POST` | `/api/upload` | 영수증 업로드 및 OCR 파싱 |
| `GET` | `/api/expenses` | 목록 조회 (`?from=YYYY-MM-DD&to=YYYY-MM-DD`) |
| `DELETE` | `/api/expenses/{id}` | 항목 삭제 |
| `PUT` | `/api/expenses/{id}` | 항목 수정 |
| `GET` | `/api/summary` | 합계 통계 (`?month=YYYY-MM`) |

---

## 데이터 스키마

```json
{
  "id": "uuid-v4",
  "created_at": "2025-07-15T14:30:00Z",
  "store_name": "이마트 강남점",
  "receipt_date": "2025-07-15",
  "receipt_time": "13:25",
  "category": "식료품",
  "items": [
    { "name": "신라면 멀티팩", "quantity": 2, "unit_price": 4500, "total_price": 9000 }
  ],
  "subtotal": 10800,
  "discount": 500,
  "tax": 0,
  "total_amount": 10300,
  "payment_method": "신용카드",
  "raw_image_path": "uploads/receipt_xxx.jpg"
}
```

**카테고리**: `식료품` `외식` `교통` `쇼핑` `의료` `기타`

---

## 배포 (Vercel)

Vercel 대시보드에서 환경변수 `UPSTAGE_API_KEY` 등록 후 GitHub 연동으로 자동 배포됩니다.

```json
// vercel.json
{
  "builds": [
    { "src": "frontend/package.json", "use": "@vercel/static-build" },
    { "src": "backend/main.py",       "use": "@vercel/python" }
  ],
  "routes": [
    { "src": "/api/(.*)", "dest": "backend/main.py" },
    { "src": "/(.*)",     "dest": "frontend/dist/$1" }
  ]
}
```

> **데이터 영속성 주의**: Vercel 서버리스는 파일 시스템이 요청 간 유지되지 않습니다.
> `VERCEL=1` 환경변수 감지 시 자동으로 `/tmp/expenses.json`을 사용합니다.
> 영속성이 필요하면 Railway / Render 배포 또는 Vercel KV(Redis) 도입을 권장합니다.

---

## 테스트용 영수증 샘플

`images/` 디렉토리에 다양한 실제 영수증이 포함되어 있습니다.

| 파일 | 매장 |
|------|------|
| `01_emart.png` | 이마트 |
| `02_starbucks.png` | 스타벅스 |
| `03_cu.jpg` | CU 편의점 |
| `03_lotte_depart.png` | 롯데백화점 |
| `04_lotteria.png` | 롯데리아 |
| `05_ikea.png` | IKEA |
| `06_uniqlo.png` | 유니클로 |
| `07_cgv.png` | CGV |
| `08_megabox.png` | 메가박스 |
| `09_medical.png` | 병원 |
| `11_taxi.png` | 택시 |
| `GS25편의점_영수증.pdf` | GS25 (PDF) |
