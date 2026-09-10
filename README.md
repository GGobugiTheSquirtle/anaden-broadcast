# 어나덴 방송 아카이브

어나더에덴(Another Eden) 공식 생방송 요약 아카이브.
일본어 방송을 한국어 커뮤니티용으로 정리.

## 방송 목록

| 날짜 | 방송명 | 폴더 |
|------|--------|------|
| 2026-04-04 | アナデンまつり 2026 春 제1부 (ビギンズ) | `2026-04-04_마쓰리_1부/` |
| 2026-09-10 | Another Eden Live #59 (메인스토리 3부 Rhapsody / 3.17.0 / 비긴즈) | `2026-09-10_라이브59/` |

## 작업 파이프라인

### 1단계: 영상 다운로드 (`yt-dlp`)
```bash
# 생방송 중 (처음부터 받기)
yt-dlp --live-from-start -f "136+140" -o "stream.mp4" "URL"

# VOD (방송 종료 후)
yt-dlp -f "bestvideo[height<=720]+bestaudio" -o "stream.mp4" "URL"
```
- 생방송은 `--live-from-start`로 처음부터 받되, 원하는 시점에 kill해도 .part 파일 사용 가능
- 720p 추천 (화질/용량 균형)

### 2단계: 미디어 분리 (`ffmpeg`)
```bash
# 비디오+오디오 합치기
ffmpeg -i video.mp4.part -i audio.mp4.part -c copy -shortest merged.mp4

# 오디오 추출 (Whisper용 16kHz mono WAV)
ffmpeg -i audio.mp4.part -ar 16000 -ac 1 audio.wav

# 스크린샷 추출 — 60초 간격 (전체 구조 파악용)
ffmpeg -i video.mp4.part -vf "fps=1/60" -q:v 2 frames/frame_%04d.jpg

# 세부 스크린샷 — 15초 간격 (특정 구간 상세 분석용)
ffmpeg -ss 720 -t 480 -i video.mp4.part -vf "fps=1/15,scale=720:-1" -q:v 4 detail/pv_%03d.jpg
```

### 3단계: 음성 전사 (`faster-whisper`)
```python
from faster_whisper import WhisperModel
model = WhisperModel('medium', device='cuda', compute_type='float16')
segments, info = model.transcribe('audio.wav', language='ja', beam_size=5, vad_filter=True)
# → transcript.json (타임스탬프 + 일본어 텍스트)
```
- **medium 모델 사용** (large-v3는 RTX 3060에서 로딩 불안정)
- VAD 필터 켜서 무음 구간 제거
- ~73분 → 1407 세그먼트, 약 5분 소요

### 3.5단계: 슬라이드 영역 크롭 (2026-09-10 추가)

공식 방송은 OBS 씬이 고정이라 **슬라이드가 프레임의 일부만 차지한다**.
상단 ~118px 는 유튜브 채팅 오버레이, 우측 밴드는 키비주얼, 우하단은 진행자 웹캠 —
1280x720 프레임을 그대로 쓰면 슬라이드 각주가 웹캠에 가려지고 본문 글씨가 작아 판독·게재 양쪽에서 손해다.

```bash
py -3.14 _tools/media/broadcast_slide_crop.py     # 프레임별 레이아웃 분류 출력
```
- 판정: **우상단 "ANOTHER EDEN … LIVE" 로고 템플릿 상관계수**. `>= 0.85` 면 방송 레이아웃, 미만이면 PV 전체화면
  - 실측 상관계수 격차가 크다 (방송 레이아웃 0.85+ / PV 컷 최대 0.141) → 오분류 위험 낮음
- 크롭 영역 `SLIDE = (8, 118, 1070, 720)` — 1280x720 기준. 결과 1062x602
- **PV·인게임 전체화면 컷은 크롭하지 말 것** (Live #59 기준 190장 중 40장이 전체화면)
- 확대(업스케일) 금지. 크롭만으로 실효 글씨 크기가 ~20% 커진다

### 4단계: 프레임 분석 (Claude Vision 병렬 에이전트)
- **판독 세트 구성**: 5초 간격 전량 추출 → 다운스케일 그레이 프레임 차분(임계 6.0)으로 중복 제거
  → 정보 구간은 30초 그리드로 보강. Live #59 실측 809장 → 190장
- 챕터(유튜브 chapters)별로 Agent 병렬 발사. Live #59 는 6트랙
- 각 에이전트에 **등급(A/B/C) 판정**을 요구 → A등급만 게재. 중복 슬라이드가 60% 이상이다
- 각 에이전트가 프레임별 일본어 텍스트 추출 + 한국어 번역
- 출력: `analysis_pv.md`, `analysis_battle.md`, `analysis_chars.md`

### 5단계: QA 교차검증 (번역 품질 에이전트)
- 2개 에이전트 병렬: PV번역 QA / 배틀번역 QA
- 체크: 번역 오류, 어색한 표현, 캐릭터명 일관성, 게임 용어, 누락
- 출력: `qa_pv.md`, `qa_battle.md`

### 6단계: HTML 생성 (Python 스크립트)
- 인라인 CSS (네이버카페 복붙 호환)
- 이미지는 상대경로 (`images/*.jpg`) — base64나 외부 호스팅 사용 금지
- 라이트모드 에디토리얼 디자인
- 공식 한국판 공지의 Naver CDN 이미지 활용 가능

### 7단계: 배포 (GitHub Pages)
```bash
cd "어나덴 방송"
git add . && git commit -m "Add YYYY-MM-DD 방송명" && git push
```
- GitHub Pages 자동 배포
- URL: `https://ggobugithesquirtle.github.io/anaden-broadcast/폴더명/`

## 폴더 구조 규칙

```
YYYY-MM-DD_방송명/
├── index.html              ← 요약 페이지 (GitHub Pages 진입점)
├── images/                 ← 스크린샷 jpg (720px, ffmpeg q4)
│   ├── f0001.jpg~          ← 60초 간격 키프레임 (f=frame)
│   ├── pv_001.jpg~         ← PV 구간 15초 간격
│   ├── bt_001.jpg~         ← 배틀 구간 15초 간격
│   └── ch_001.jpg~         ← 캐릭터/가격 구간 15초 간격
├── transcript.json         ← Whisper 전사 결과
├── analysis_*.md           ← 프레임 분석 결과
├── qa_*.md                 ← 번역 QA 결과
└── stream_merged.mp4       ← 원본 영상 (.gitignore로 제외)
```

## 주의사항

- **catbox/imgur 등 외부 이미지 호스팅 사용 금지** — 깨짐 발생
- **webp 형식 금지** — 네이버카페 비호환
- **base64 파일매칭 금지** — JPEG 헤더 동일 문제로 전부 같은 이미지로 매핑됨
- **한국판 공식 공지 반드시 확인** — 한국어 트레일러 URL, 공식 이미지, 추가 정보
- **.gitignore에 mp4/wav/part/ytdl 제외** — 대용량 미디어 파일 업로드 방지
