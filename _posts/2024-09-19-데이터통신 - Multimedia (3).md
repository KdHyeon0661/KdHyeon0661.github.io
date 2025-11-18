---
layout: post
title: 데이터 통신 - Multimedia (3)
date: 2024-09-19 22:20:23 +0900
category: DataCommunication
---
# 멀티미디어 — 압축과 멀티미디어 데이터(Compression & Multimedia Data)

## 28.1 Compression

### 28.1.1 압축 개요와 기본 개념

압축(compression)의 목표는 단순하다.

- **저장**: 같은 용량의 디스크/메모리에 더 많은 데이터를 저장
- **전송**: 같은 대역폭에서 더 많은 데이터를 전송, 혹은 같은 데이터를 더 빠르게 전송
- **비용**: 클라우드 스토리지·CDN 트래픽 비용 절감

압축 알고리즘은 대략 두 종류다.

- **Lossless compression (무손실 압축)**  
  - 압축 → 해제 후 **원본과 바이트 단위까지 완전히 동일**  
  - 텍스트, 실행 파일, 소스코드, 금융 데이터 등 “한 비트도 틀리면 안 되는” 데이터에 사용  
  - 예: ZIP, gzip, PNG, FLAC, 일부 PDF 등 
- **Lossy compression (손실 압축)**  
  - 사람 눈/귀가 잘 못 느끼는 정보는 과감히 버리고, **품질을 조금 희생해서 압축률을 극대화**  
  - 이미지·오디오·비디오 스트리밍에 필수  
  - 예: JPEG, MP3, AAC, Opus, H.264/AV1 비디오 코덱 

압축 성능을 보통 아래처럼 정의한다.

- **압축 비율(compression ratio)**  

  $$
  \text{compression ratio} = 
  \frac{\text{original size}}{\text{compressed size}}
  $$

  예를 들어 10 MB → 2 MB로 줄이면 압축 비율은 5:1, 또는 80% 절감이다.

- **공간 절감률(space saving)**  

  $$
  \text{space saving} = 1 - \frac{\text{compressed size}}{\text{original size}}
  $$

  위 예에서는  
  $$1 - \frac{2}{10} = 0.8 = 80\%$$

멀티미디어 스트리밍의 경우에는 보통 **비트레이트(bitrate)**로 이야기한다.

$$
\text{bitrate} = \frac{\text{전송해야 할 총 비트 수}}{\text{재생 시간(초)}}
$$

예를 들어 10분짜리(600초) 영상이 600 MB(= 600 × 8 Mbit = 4800 Mbit)라면,

$$
\text{bitrate} = \frac{4800\ \text{Mbit}}{600\ \text{s}} = 8\ \text{Mbit/s}
$$

---

### 28.1.2 Lossless Compression (무손실 압축)

무손실 압축 알고리즘의 핵심 아이디어는 크게 두 가지다.

1. **패턴의 반복을 이용**: 반복되는 문자열/비트열을 “참조”로 치환 (예: LZ77/LZ78 계열)
2. **심볼의 출현 확률을 이용**: 자주 나오는 심볼에는 짧은 코드, 드물게 나오는 심볼에는 긴 코드 (예: Huffman, Arithmetic coding)

#### (1) 간단한 예: 텍스트 RLE

**Run-Length Encoding (RLE)**는 가장 단순한 무손실 압축 예시다.

- 원본:  
  `AAABBBBCCDDDDDDDD`
- RLE 표현:  
  `3A4B2C8D`

길이 비교:

- 원본 길이: 16 문자
- 압축 후: `3A4B2C8D` → 8 문자  
→ 압축 비율 2:1

물론 `ABCDEFGH` 같이 반복이 없는 데이터는 **오히려 길어질** 수 있다. RLE는 **대부분의 값이 같은 비트맵 마스크, 흑백 팩스** 등에서 효과적이다.

#### (2) Huffman Coding 직관적 이해

**Huffman 코딩**은 심볼의 출현 빈도에 따라 **가변 길이 코드**를 부여하는 무손실 압축이다.

예시 알파벳과 빈도:

| 심볼 | 빈도 | 확률 |
|------|------|------|
| A    | 50   | 0.5  |
| B    | 25   | 0.25 |
| C    | 15   | 0.15 |
| D    | 10   | 0.10 |

고정 길이(2비트)로 코딩하면:

- A: 00, B: 01, C: 10, D: 11  
- 평균 비트 수: 항상 **2비트/심볼**

Huffman은 자주 나타나는 A에 짧은 코드를 할당한다.

예시 트리(한 가지 가능한 경우):

- A: `0`
- B: `10`
- C: `110`
- D: `111`

평균 비트 수:

$$
E[L] = 0.5\cdot1 + 0.25\cdot2 + 0.15\cdot3 + 0.10\cdot3 
= 1.75\ \text{bit/symbol}
$$

고정 길이 2비트보다 약 12.5% 효율적이다.

#### (3) LZ77 / LZ78 / DEFLATE — 현대 압축의 근간

현대의 ZIP, gzip, PNG 등 상당수 포맷은 **LZ77 계열 + Huffman** 조합을 사용한다. 대표적으로 **DEFLATE** 포맷은 LZ77 슬라이딩 윈도우와 Huffman 코딩을 결합한 무손실 형식이다. 

간단한 아이디어:

- 최근 데이터 일부(슬라이딩 윈도우)를 탐색해서 **반복되는 부분 문자열을 “거리+길이”로 치환**
- 이렇게 토큰화한 스트림을 다시 Huffman 코딩으로 비트 단위 압축

예시 텍스트:

`ABRACADABRA ABRACADABRA`

앞에 나온 `ABRACADABRA` 문자열을 나중에 다시 사용할 때:

- “앞에서부터 11문자 뒤로 가서 길이 11만큼 복사” 같은 형태로 표현

이는 텍스트처럼 반복과 패턴이 많은 데이터에서 상당한 압축을 이끈다.  
연구들에 따르면 일반적인 영어 텍스트는 LZ77/Huffman 계열로 **30~70% 정도 크기 감소**를 얻을 수 있다. 

#### (4) PNG / FLAC 같은 포맷

- **PNG**: 무손실 비트맵 이미지 포맷, ISO/IEC 15948 및 RFC 2083에 정의되어 있으며, 각 스캔라인에 대한 예측 필터 + DEFLATE(= LZ77+Huffman)로 압축한다. 
- **FLAC**: 무손실 오디오 포맷, 오디오 신호를 예측(LPC 등)한 뒤 예측 오차를 엔트로피 코딩으로 압축

실제 웹에서는 “**텍스트 + PNG + gzip/deflate**” 조합이 기본이다.

#### (5) 간단 Python 예제: 텍스트 무손실 압축

텍스트를 gzip으로 압축해보는 예시:

```python
import gzip
import json

text = "ABRACADABRA " * 1000  # 인위적으로 반복 많은 텍스트
data = text.encode("utf-8")

print("원본 크기:", len(data), "bytes")

compressed = gzip.compress(data, compresslevel=9)
print("압축 후 크기:", len(compressed), "bytes")

# 검증
decompressed = gzip.decompress(compressed)
assert decompressed == data
```

이처럼 **gzip은 완전히 무손실**이며, `assert`가 실패하지 않는다.  
반복이 많을수록 압축 비율이 커진다.

---

### 28.1.3 Lossy Compression (손실 압축)

손실 압축은 인간의 지각 특성에 기반한 **“굴절된 정직함”**이다.

- 사람 눈은 **고주파 세부 패턴보다 큰 윤곽과 색 대비**에 더 민감
- 사람 귀는 **절대적인 진폭보다 특정 주파수 대역**에 더 민감

이를 활용해 **“덜 중요한” 정보는 버리고, “중요해 보이는” 정보만 상대적으로 많이 남겨** 비트 수를 줄인다. 

#### (1) 기본 구조: Transform + Quantization + Entropy coding

대부분의 손실 코덱(JPEG, MP3, AAC, H.264, AV1 등)은 아래 구조를 가진다.

1. **블록 또는 프레임 단위 분할**
2. **변환(Transform)**  
   - 이미지: DCT(Discrete Cosine Transform), DWT(Wavelet)  
   - 오디오: MDCT(Modified DCT)
3. **양자화(Quantization)**  
   - 작은 계수(특히 고주파)는 0 또는 작은 정수로 잘라내기  
   - 이 단계가 “손실(loss)”이 발생하는 핵심
4. **엔트로피 코딩(Huffman, Arithmetic 등)**  
   - 남은 계수들을 무손실 압축

간단한 수식 예시:

- 원본 신호: $$x[n]$$
- DCT 계수: $$X[k] = \text{DCT}(x[n])$$
- 양자화: $$\tilde{X}[k] = \text{round}\left(\frac{X[k]}{Q[k]}\right)$$
- 저장 시에는 $$\tilde{X}[k]$$만 저장하고, 복원할 때 $$\hat{X}[k] = \tilde{X}[k]\cdot Q[k]$$

여기서 $$Q[k]$$가 크면 클수록 **더 과감한 양자화** → 더 높은 압축률, 더 큰 손실.

#### (2) JPEG 이미지 압축 예

JPEG는 수십 년간 웹 이미지의 대표 코덱이었다.

핵심 단계:

1. 이미지를 8×8 블록으로 분할
2. 각 블록에 대해 2D DCT 적용
3. 고주파 계수를 상대적으로 더 거칠게 양자화
4. 양자화된 계수를 지그재그 순회 → RLE → Huffman 코딩

실제 웹 이미지 최적화 도구에서, **원본 JPEG(예: 2 MB)를 손실 압축으로 40~80%까지 줄이면서 육안에는 큰 차이가 없게** 만드는 것이 흔하다. 실험 기사들에서는 **같은 이미지에서 손실 압축이 약 40% 파일 크기 절감, 무손실은 5~10% 절감** 정도로 보고하기도 한다. 

#### (3) 비디오 코덱: H.264, HEVC, AV1 등

비디오 코덱은 **공간(프레임 내부) + 시간(프레임 사이)** 중복을 모두 제거한다.

- 공간 중복: 각 프레임의 DCT/변환 코딩
- 시간 중복: 모션 보상, 이전 프레임과의 차이만 저장

대표적 코덱:

- **H.264/AVC**: 여전히 대부분의 스트리밍·회의에서 기본
- **HEVC/H.265**: 4K/HDR 콘텐츠에서 많이 사용, H.264보다 약 50% 가까운 비트 절감 보고 
- **AV1**: YouTube, Netflix, Facebook 등이 점점 도입 중. H.264 대비 **30~50% 파일 크기 절감**이 보고되며, 같은 품질에서 더 낮은 비트레이트를 제공. 

예시: 1080p(1920×1080) 30fps 비디오에 대해,

- H.264로 5 Mbps 정도면 “준수한” 스트리밍 품질
- AV1에서는 비슷한 품질을 **3~3.5 Mbps 정도**로 구현 가능 보고가 많다

또한, MPEG 계열 분석에 따르면 **1080i 비디오를 20 Mbit/s MPEG 스트림으로 담기 위해 최소 50:1 이상의 압축 비율**이 필요하다는 연구도 있다. 

#### (4) 오디오 코덱: MP3, AAC, Opus

- **MP3**: 오래된 표준, 오늘날은 효율 면에서 다소 뒤처짐
- **AAC**: 스트리밍, 방송에서 널리 사용
- **Opus**: IETF가 정의한 공개, 로열티 프리 코덱으로 **실시간 통신(VoIP, WebRTC, 게임 음성 채팅)**에 매우 적합한 저지연 코덱   

Opus는 기본 설정에서 약 20 ms 프레임, 26.5 ms 수준의 알고리즘 지연을 제공하여, 4G/5G 네트워크의 추가 지연(수십 ms)에도 실시간 통신 품질을 확보한다.

일반적으로:

- 음악 스트리밍: 128~256 kbps (압축 비율 수십:1)
- 음성 통화: 16~32 kbps

#### (5) Python 예제: 단순 품질 비교(개념 코드)

아래 코드는 구조만 보여준다(실제 실행을 위해서는 `Pillow`, `ffmpeg` 등이 필요).

```python
from PIL import Image
import os

img = Image.open("input.png")

# 고품질 JPEG
img.save("out_q95.jpg", format="JPEG", quality=95)
# 저품질 JPEG
img.save("out_q40.jpg", format="JPEG", quality=40)

def size(path):
    return os.path.getsize(path) / 1024  # KB

print("원본 PNG:", size("input.png"), "KB")
print("JPEG Q=95:", size("out_q95.jpg"), "KB")
print("JPEG Q=40:", size("out_q40.jpg"), "KB")
```

실제 테스트를 해보면:

- PNG(무손실) → 예: 1000 KB
- JPEG Q=95 → 예: 300 KB
- JPEG Q=40 → 예: 80 KB

같은 이미지를 **손실 압축으로 수배~수십배 줄일 수 있다**는 감각만 기억하면 된다.

---

## 28.2 Multimedia Data

이제 압축 알고리즘을 멀티미디어 데이터 유형별로 묶어 보자.

- Text
- Image
- Video
- Audio

각 데이터가 갖는 **통계적/지각적 특성**이 **어떤 압축 방식에 어울리는지**가 포인트다.

---

### 28.2.1 Text

#### (1) 텍스트의 특징

- 심볼: 보통 문자(char)
- 알파벳 크기: 영문 기준 26자 + 숫자 + 기호 → 수십~수백
- 강한 **규칙성과 중복**, 예측 가능성:  
  - 영어에서 e, t, a, o, i 등은 매우 자주, z, q, x는 드물게 등장  
  - 단어, 구, 문장 구조도 반복적

#### (2) 인코딩: ASCII, Unicode, UTF-8

오늘날 웹에서 텍스트는 거의 대부분 **UTF-8**로 인코딩한다.

- UTF-8은 **ASCII와 호환되면서** 모든 유니코드 문자를 표현 가능
- 영어 알파벳은 1바이트, 유럽/아시아 문자는 2~4바이트 등 가변 길이
- 2025년 기준, 웹 사이트 문자 인코딩의 **약 98.8%가 UTF-8**을 사용하고 있다고 W3Techs가 보고한다.   

UTF-8 자체는 “압축”은 아니지만, **영어 중심 텍스트에 매우 효율적**이다. 예를 들어 UTF-32는 모든 문자를 4바이트로 표현하는 반면, UTF-8은 영어 텍스트에서 1바이트만 사용한다.

#### (3) 텍스트 압축에 잘 맞는 알고리즘

텍스트는 규칙성과 중복이 많아서 **무손실 압축**으로도 상당한 절감이 가능하다.

- **Huffman + LZ77 (DEFLATE)**: ZIP, gzip, HTTP Content-Encoding에 기본 사용
- **BWT + MTF + RLE + Huffman**: bzip2 계열

예시: 간단한 로그 파일 압축

```python
import gzip

with open("server.log", "rb") as f:
    data = f.read()

print("원본:", len(data)/1024, "KB")

compressed = gzip.compress(data, compresslevel=9)
print("압축:", len(compressed)/1024, "KB")

# 복원 검증
assert gzip.decompress(compressed) == data
```

실제 웹 서버 로그는 패턴이 많아서 **3~10배**까지도 줄어드는 경우가 많다.

---

### 28.2.2 Image

이미지는 “2D 신호”이다.

- 공간 해상도: 가로×세로 픽셀
- 색 표현: 색상 공간(RGB, YCbCr 등), 비트 깊이(8bit, 10bit, 12bit …)

#### (1) 비트맵 vs 벡터

- **비트맵(bitmap, raster)**: 각 픽셀을 명시
  - 예: PNG, JPEG, WebP, AVIF
- **벡터(vector)**: 기하 도형, 곡선, 텍스트 등을 수식으로 표현
  - 예: SVG, PDF 내부 벡터 객체

이번 장의 압축 논의는 대부분 **비트맵**에 초점.

#### (2) 이미지 포맷과 압축

대표 포맷:

| 포맷 | 손실/무손실 | 주요 용도 |
|------|-------------|----------|
| BMP  | 무압축 또는 단순 | 테스트, 내부 처리용 |
| PNG  | 무손실 압축(DEFLATE) | 아이콘, UI, 스크린샷, 투명도 필요할 때  |
| JPEG | 손실(옵션으로 무손실) | 사진, 웹 이미지 |
| WebP | 손실+무손실 모두 지원 | 웹 최적화 이미지 |
| AVIF | AV1 기반, 최신 손실/무손실 | 고효율 이미지 |

PNG는 RFC 2083에서 정의된 대로 **무손실, 특허 프리**, 다양한 비트 깊이(1~16bit)와 색상 타입을 지원한다. 

#### (3) 예제: 사진 vs UI 아이콘

- 사진(jpeg)  
  - 카메라로 찍은 풍경 사진: 노이즈·텍스처·그라디언트가 많다  
  - 손실 압축으로 고주파(세부 노이즈)를 일부 제거해도 **눈으로는 큰 차이를 못 느끼는 경우가 많음**
- UI 아이콘(png)  
  - 단색 영역, 선, 문자 등  
  - 손실 압축으로 블록 아티팩트가 생기면 바로 눈에 띈다  
  - 무손실 PNG + 알파 채널이 안전

예를 들면, 동일한 1024×1024 이미지라도:

- 사진(JPEG Q=80): 200~400 KB
- PNG(무손실): 1~3 MB

#### (4) Python 예제: 이미지 품질과 포맷 선택

```python
from PIL import Image
import os

img = Image.open("photo_raw.png")

img.save("photo_jpeg_q90.jpg", quality=90)
img.save("photo_jpeg_q60.jpg", quality=60)
img.save("photo_png.png", optimize=True)

def size(path):
    return os.path.getsize(path) / 1024

for path in ["photo_raw.png", "photo_png.png", "photo_jpeg_q90.jpg", "photo_jpeg_q60.jpg"]:
    print(path, ":", size(path), "KB")
```

직접 열어보면:

- Q=90 JPEG는 육안상 거의 원본과 유사
- Q=60 JPEG는 블록/흐림이 조금씩 보임
- PNG는 가장 큰 파일이지만 완전 무손실

---

### 28.2.3 Video

비디오는 **연속된 이미지(프레임) + 오디오**를 묶은 것이다.

- 해상도: 1920×1080(1080p), 3840×2160(4K), 7680×4320(8K) 등
- 프레임 레이트: 24, 30, 60 fps 등
- 비디오 코덱: H.264, HEVC, VP9, AV1, VVC …
- 오디오 코덱: AAC, Opus, AC-3 등
- 컨테이너: MP4, MKV, WebM, TS 등

#### (1) 비디오 데이터의 엄청난 크기

압축 전 **원시(raw) 비디오**는 말도 안 되게 크다.

예시: 1080p, 24비트 RGB, 30fps

- 한 프레임 픽셀 수:  
  $$1920 \times 1080 \approx 2{,}073{,}600\ \text{pixels}$$
- 프레임당 비트수:  
  $$2{,}073{,}600 \times 24 \approx 49{,}766{,}400\ \text{bits} \approx 47.5\ \text{Mbit}$$
- 초당 30 프레임:  
  $$47.5\ \text{Mbit/frame} \times 30 \approx 1{,}425\ \text{Mbit/s} \approx 1.4\ \text{Gbit/s}$$

즉, **압축 없는 1080p 영상은 1.4 Gbit/s 정도**가 된다.

하지만 실제 스트리밍 서비스는:

- 1080p: 3~8 Mbit/s
- 4K: 10~25 Mbit/s

즉, **수십~수백 배의 압축**이 들어간다. MPEG 계열 분석에서는 1080i를 20 Mbit/s에 실으려면 최소 50:1 압축이 필요하다고 본다. 

#### (2) 프레임 타입(I/P/B)과 GOP

비디오 코덱은 프레임을 크게 세 종류로 나눈다.

- I-Frame (Intra)  
  - 완전한 이미지, JPEG와 비슷한 방식
  - 임의 위치에서 재생을 시작하기 위한 앵커
- P-Frame (Predicted)  
  - 이전 프레임(I 또는 P)을 참조해서 차이만 저장
- B-Frame (Bi-directional)  
  - 앞뒤 프레임을 모두 참조해 차이만 저장

일련의 프레임 묶음을 **GOP(Group Of Pictures)**라고 부른다.

예: `I B B P B B P ... I ...`

- I-프레임은 크지만 드물게
- B/P-프레임은 작지만 많이

#### (3) 현대 비디오 코덱과 효율

2025년 기준:

- **H.264/AVC**: 거의 모든 디바이스 지원, 유튜브·회의 솔루션의 기본
- **HEVC/H.265**: 4K TV, UHD 방송, 블루레이 등에서 널리 사용, H.264 대비 25~50% 절감 보고 
- **AV1**: 오픈, 로열티 프리 코덱으로 H.264 대비 30~50% 절감, HEVC 대비도 우위 보고들 존재 

스트리밍미디어 업계 분석에 따르면, 2025년 시점에 AV1은 주요 플랫폼(YouTube, Netflix 등)에서 4K/모바일 콘텐츠에 점점 더 많이 사용되고 있다. 

#### (4) Python 예시: ffmpeg로 비트레이트 줄이기(개념)

실제 인코딩은 외부 도구(ffmpeg)를 호출해야 한다.

```python
import subprocess

input_file = "input_1080p_raw.yuv"
output_h264 = "output_h264.mp4"
output_av1 = "output_av1.mkv"

# H.264 인코딩 (약 5 Mbps)
subprocess.run([
    "ffmpeg", "-y",
    "-s", "1920x1080", "-pix_fmt", "yuv420p", "-r", "30",
    "-i", input_file,
    "-c:v", "libx264", "-b:v", "5M",
    output_h264
])

# AV1 인코딩 (같은 품질 목표, 더 낮은 비트레이트)
subprocess.run([
    "ffmpeg", "-y",
    "-s", "1920x1080", "-pix_fmt", "yuv420p", "-r", "30",
    "-i", input_file,
    "-c:v", "libaom-av1", "-crf", "30", "-b:v", "0",
    output_av1
])
```

테스트를 해보면:

- H.264: 약 4~6 Mbps
- AV1: 2~3.5 Mbps 정도에서 비슷하거나 더 나은 주관적 품질을 얻는 경우가 많다.

---

### 28.2.4 Audio

오디오는 시간 축의 1D 신호다.

- 샘플링 주파수: 44.1 kHz, 48 kHz, 96 kHz …
- 채널: mono, stereo, 5.1, 7.1 …
- 해상도: 16bit, 24bit …

#### (1) 샘플링과 비트레이트

가장 기본적인 PCM 오디오 비트레이트:

$$
\text{bitrate} =
\text{sampling rate} \times \text{bit depth} \times \text{channels}
$$

예: CD 품질(44.1 kHz, 16bit, 스테레오)의 비트레이트

$$
44{,}100 \times 16 \times 2 
= 1{,}411{,}200\ \text{bit/s}
\approx 1.411\ \text{Mbit/s}
$$

이를 **1.4 Mbps** 정도로 부르며, 저장 시 약 176 KB/s, 10분이면 105 MB 정도가 된다.

#### (2) 무손실 vs 손실 오디오

- **무손실 (FLAC, ALAC, WAV(비압축))**
  - 스튜디오 녹음, 아카이빙
  - 원본과 완전히 동일
  - FLAC은 보통 30~60% 크기 절감을 달성 (소스에 따라 다름)
- **손실 (MP3, AAC, Ogg Vorbis, Opus)**
  - 스트리밍, 모바일, VoIP
  - 심리음향 모델을 사용해 귀에 덜 들리는 주파수를 제거
  - 음성 통화는 16~32 kbps, 음악은 128~256 kbps 정도에서 “실용적인” 품질

#### (3) Opus와 실시간 통신

**Opus**는 WebRTC, VoIP, 게임 음성 채팅 등에서 사실상 표준에 가까운 위치를 차지하고 있다.   

특징:

- **저지연**: 26.5 ms 수준의 기본 지연, 5 ms까지 줄일 수 있음
- 유연한 비트레이트: 6 kbps ~ 수백 kbps
- 음악과 음성 모두에 최적화

예시: 음성 회의 서비스

- 인코딩: Opus 24 kbps, 20 ms 프레임
- 한 통화의 평균 비트레이트는 24 kbps + 패킷 헤더 오버헤드 (예: 총 30 kbps 안팎)
- 10명 동시 통화라도 300 kbps 수준으로 충분히 운용 가능

#### (4) Python 예제: pydub로 비트레이트 비교(개념)

```python
from pydub import AudioSegment
import os

song = AudioSegment.from_file("input.wav")

# 320 kbps MP3
song.export("out_320.mp3", format="mp3", bitrate="320k")

# 128 kbps MP3
song.export("out_128.mp3", format="mp3", bitrate="128k")

def size(path):
    return os.path.getsize(path) / 1024

for p in ["input.wav", "out_320.mp3", "out_128.mp3"]:
    print(p, ":", size(p), "KB")
```

직접 들어보면:

- 320 kbps: 거의 CD급 품질
- 128 kbps: 대부분의 상황에서 충분하지만, 고급 스피커에서 차이가 느껴질 수 있음

---

## 28.2.x 정리: 데이터 타입별 전략

마지막으로, 지금까지의 이야기를 “실무 전략” 관점에서 요약해보자.

### (1) Text

- **인코딩**: UTF-8 고정 (이미 웹의 사실상 표준)
- **압축**: HTTP/2, HTTP/3에서 `gzip` 또는 `brotli` 사용
  - HTML, CSS, JS, JSON 등 텍스트는 무손실 압축으로 30~80% 절감 가능

### (2) Image

- 사진, 배너, 썸네일
  - JPEG/AVIF/WebP (손실)
  - 화질–파일 크기 트레이드오프를 실험해 최적 값 찾기
- UI 아이콘, 로고, 스크린샷
  - PNG/WebP(무손실)  
  - 투명도와 sharp edge가 중요할 때 무손실 사용

### (3) Video

- 온라인 스트리밍
  - H.264 기본, 점진적으로 AV1 도입
  - 1080p는 3~8 Mbps, 4K는 10~25 Mbps 정도에서 화질–비용 균형 잡기
- 회의/화상통화
  - 상대적으로 낮은 해상도(720p 등) + 낮은 비트레이트(1~3 Mbps)  
  - 지연(latency)과 패킷 손실에 강한 프로파일 사용

### (4) Audio

- VOD, 음악 스트리밍
  - AAC 128~256 kbps, 필요시 FLAC 무손실 제공
- 실시간 음성(VoIP, 게임)
  - Opus 16~32 kbps (저지연·손실)
- 아카이빙
  - FLAC/ALAC 무손실