# backend_readme.md

> **목표**: FastAPI 기반 뉴스 요약/트렌드 분석 백엔드를 **충돌 없는 의존성**, **프론트엔드 연동 용이성**, **에러 추적 용이성**을 고려해 실행/구성하는 방법을 정리한 문서입니다.

---

## 1. 폴더 구조

```
news_trend_project/
├── api/
│   ├── main.py                # FastAPI 앱 팩토리 및 엔트리포인트
│   └── routes/
│       └── news.py            # /news_trend API 라우트
├── core/
│   ├── constants.py           # 키워드-목적 매핑 등 상수
│   ├── evaluation.py          # ROUGE/BERTScore 평가 로직
│   ├── llm_chains.py          # LangChain 프롬프트/체인 정의
│   └── news_scraper.py        # 뉴스 링크/본문 추출
├── utils/
│   └── cleanre.py             # 텍스트 정제 함수
├── voice/                     # (선택) STT/TTS 관련. 서버와 분리됨
│   ├── stt.py
│   ├── tts.py
│   └── voice_chat.py
├── run_cli.py                 # (선택) CLI 클라이언트
├── requirements.txt           # 전체 패키지(개발/통합용)
├── requirements-api.txt       # 백엔드 서버 전용 최소 패키지
├── requirements-voice.txt     # STT/TTS 전용 패키지
├── .env.example               # 환경 변수 예시
└── README.md / backend_readme.md
```

---

## 2. 사전 준비

### 2.1 Python 가상환경 생성
```bash
python -m venv .venv
# Windows
.venv\Scripts\activate
# macOS/Linux
source .venv/bin/activate
```

### 2.2 의존성 설치 (백엔드 전용)
```bash
pip install -r requirements-api.txt
```
> **충돌 방지 TIP**: 음성 관련 패키지(PyAudio, webrtcvad 등)와 무거운 모델 다운로드가 필요 없으면 `requirements-voice.txt`는 설치하지 마세요.

### 2.3 환경 변수 설정
```bash
cp .env.example .env
# .env 파일 편집
OPENAI_API_KEY=YOUR_KEY_HERE
CORS_ORIGINS=http://localhost:3000,https://your-frontend-domain.com
```
- `CORS_ORIGINS`는 콤마(,)로 구분된 도메인 목록입니다.
- 운영 환경에서는 `*` 대신 실제 도메인으로 제한하세요.

---

## 3. 서버 실행

### 3.1 개발용 (hot reload)
```bash
uvicorn api.main:app --host 0.0.0.0 --port 8000 --reload
```

### 3.2 운영용 (예: gunicorn + uvicorn workers)
```bash
pip install gunicorn
gunicorn -k uvicorn.workers.UvicornWorker api.main:app -b 0.0.0.0:8000 --workers 2 --timeout 120
```

---

## 4. API 명세

### 4.1 Health Check
- **Endpoint**: `GET /health`
- **Response**
```json
{ "status": "ok" }
```

### 4.2 뉴스 트렌드 요약
- **Endpoint**: `POST /news_trend/`
- **Request Body**
```json
{
  "query": "윤석열"
}
```
- **Response (예시)**
```json
{
  "query": "윤석열",
  "extracted_keyword": "윤석열",
  "purpose": "윤석열 관련 뉴스 요약",
  "trend_digest": "…트렌드 종합 요약…",
  "trend_accuracy_scores": {
    "rouge1": 0.38,
    "rougeL": 0.36,
    "bertscore": 0.79
  },
  "trend_articles": [
    {
      "title": "기사 제목…",
      "url": "https://example.com/news/1",
      "summary": "기사 요약…",
      "accuracy_scores": { "rouge1": 0.4, "rougeL": 0.39, "bertscore": 0.8 }
    }
  ]
}
```
- **curl 예시**
```bash
curl -X POST http://127.0.0.1:8000/news_trend/   -H "Content-Type: application/json"   -d '{"query": "AI 트렌드"}'
```

---

## 5. 라이브러리 충돌 방지 전략

1. **의존성 분리**  
   - API: `requirements-api.txt`  
   - Voice/STT/TTS: `requirements-voice.txt`
2. **Optional Import**  
   - `evaluation.py`에서 transformers/rouge 미설치 시 0점 반환으로 fallback.
3. **고정 버전 사용 고려**  
   - 운영 환경에서는 각 패키지 버전을 고정(`==`)하여 빌드 재현성을 확보하세요.
4. **모델/데이터 캐시 디렉터리 분리**  
   - HuggingFace/Transformers 캐시를 프로젝트 외부 디스크로 분리해 충돌 및 용량 문제을 최소화.

---

## 6. 프론트엔드 연동 가이드

- **CORS 설정**: `.env`의 `CORS_ORIGINS`에 프론트 도메인을 추가.  
- **요청/응답 타입 정의**: TypeScript 예시
```ts
export interface NewsTrendRequest { query: string; }
export interface AccuracyScores { rouge1: number; rougeL: number; bertscore: number; }
export interface TrendArticle {
  title: string; url: string; summary: string; accuracy_scores: AccuracyScores;
}
export interface NewsTrendResponse {
  query: string;
  extracted_keyword: string;
  purpose: string;
  trend_digest: string;
  trend_accuracy_scores: AccuracyScores;
  trend_articles: TrendArticle[];
}
```
- **React fetch 예시**
```ts
async function fetchTrend(query: string) {
  const res = await fetch("http://localhost:8000/news_trend/", {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({ query })
  });
  if (!res.ok) throw new Error("API Error");
  return res.json() as Promise<NewsTrendResponse>;
}
```

---

## 7. 로깅 & 모니터링 (옵션)

- 기본적으로 FastAPI/Uvicorn 로거가 동작.
- 세밀한 추적이 필요하면 `logging` 모듈로 중앙 로거 구성:
```python
import logging
logger = logging.getLogger("news")
logger.setLevel(logging.INFO)
```
- Sentry, Prometheus 등을 통한 에러/메트릭 수집 고려 가능.

---

## 8. 테스트

- **단위 테스트**: pytest 추천
```bash
pip install pytest
pytest -q
```
- **예시**: `tests/test_news_scraper.py`, `tests/test_evaluation.py` 작성

---

## 9. 배포 체크리스트

- [ ] `.env`에 실제 키/도메인 반영
- [ ] `requirements-api.txt` 버전 고정
- [ ] 헬스체크 구성 (로드밸런서/쿠버네티스 readinessProbe)
- [ ] 로그 수집/모니터링
- [ ] TLS/HTTPS 적용 (Nginx/Cloudflare 등)

---

## 10. 트러블슈팅

| 증상 | 원인 | 해결 |
|------|------|------|
| `OPENAI_API_KEY is not set` | .env 미설정 | .env에 키 추가 후 재실행 |
| `요약 실패: 본문 추출 실패` | 사이트 구조 변경 | `news_scraper.extract_article_text`의 CSS 선택자 수정 |
| 500 에러 | 외부 요청/LLM 실패 | 로그 확인 → 재시도, 타임아웃 조정 |
| 패키지 설치 실패(PyAudio 등) | OS 별 빌드 툴 요구 | 윈도우: 휠 파일 사용 / 리눅스: ALSA dev 패키지 설치 |

---

## 11. 라이선스/저작권

- 각 라이브러리(Transformers, RougeScore 등) 라이선스 준수
- 뉴스 크롤링은 각 포털/언론사 서비스 이용약관 확인 후 사용

---

**문의/개선 요청**: 언제든지 말씀해주세요!



### 2.4 MeloTTS 설치 (한국어 TTS) & mecab-ko 의존성

> **요약**: `melo-tts` + `mecab-python3`가 필요합니다. OS마다 mecab-ko 설치 방식이 다르므로 아래를 참고하세요.

#### 2.4.1 공통 (pip 설치)
```bash
pip install git+https://github.com/myshell-ai/MeloTTS.git#egg=melo-tts
pip install mecab-python3
```

#### 2.4.2 Windows
1. **PyAudio 준비**: 미리 컴파일된 휠 다운로드 후 설치 (예: `PyAudio‑0.2.12‑cp310‑cp310‑win_amd64.whl`).
   ```bash
   pip install PyAudio-0.2.12-cp310-cp310-win_amd64.whl
   ```
2. **mecab-python3 설정(선택)**: 대부분 자동 설치되지만, `MECABRC` 경로가 필요할 수 있습니다.
   ```bash
   set MECABRC=%USERPROFILE%\AppData\Local\Programs\Python\Python310\Lib\site-packages\unidic_lite\dicdir\mecabrc
   ```

#### 2.4.3 Ubuntu/Debian/Raspberry Pi
```bash
sudo apt update
sudo apt install -y build-essential libsndfile1 portaudio19-dev mecab libmecab-dev mecab-ipadic-utf8
# 한국어 mecab 사전(옵션, 필요 시)
# git clone https://github.com/jonghwanhyeon/mecab-ko-dic.git && cd mecab-ko-dic && sudo make install

# 가상환경 안에서
pip install mecab-python3
pip install git+https://github.com/myshell-ai/MeloTTS.git#egg=melo-tts
```

> **처음 TTS 실행 시 모델이 자동 다운로드** 됩니다. 예:
```python
from melo.api import TTS
tts = TTS(language='KR', device='cpu')
tts.tts_to_file("테스트 문장입니다.", 0, "test.wav")
```

---

### 2.5 faster-whisper (STT) 설치

`faster-whisper`는 CTranslate2 기반이므로 **torch 없이도 동작**합니다. (torch는 BERTScore용)

#### 2.5.1 기본 설치
```bash
pip install faster-whisper
```

#### 2.5.2 Raspberry Pi / ARM 환경 팁
- `pip install faster-whisper --extra-index-url https://www.piwheels.org/simple` (라즈베리파이에서 빌드 시간 단축)
- OpenBLAS 성능 향상을 위해 `sudo apt install libopenblas-dev` 권장

#### 2.5.3 모델 다운로드
코드에서 자동으로 HuggingFace에서 다운로드합니다. 네트워크가 차단된 환경이라면 수동 다운로드 후 폴더 지정:
```python
from faster_whisper import WhisperModel
model = WhisperModel("models/faster-whisper-base", device="cpu", compute_type="int8")
```
> `models/faster-whisper-base` 디렉터리에 모델 파일을 넣으면 로컬에서 바로 로드.

---

### 2.6 기타 오디오 관련 의존성

| 패키지 | 용도 | 설치 팁 |
|--------|------|---------|
| PyAudio | 마이크/스피커 I/O | Windows는 휠 파일 권장, Linux는 `portaudio19-dev` 필요 |
| soundfile | WAV I/O | `libsndfile1` 패키지 필요 (Ubuntu: `sudo apt install libsndfile1`) |
| webrtcvad | 음성 구간 검출 | pip로 설치 가능, ARM 빌드 시 gcc 필요 |

