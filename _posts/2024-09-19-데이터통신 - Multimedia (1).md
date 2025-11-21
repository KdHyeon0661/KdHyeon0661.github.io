---
layout: post
title: 데이터 통신 - Multimedia (1)
date: 2024-09-19 20:20:23 +0900
category: DataCommunication
---
# Chapter 28. Multimedia

## Compression

### Lossless Compression

#### 1) 개념과 수학적 정의

**Lossless 압축**은 압축 → 해제 후에 **원본과 한 비트도 다르지 않은** 데이터를 복원할 수 있는 압축이다. 대부분의 실제 데이터가 갖고 있는 **통계적 중복(statistical redundancy)** 을 제거함으로써 가능하다.

전형적인 평가는 **압축 비율**(compression ratio)로 한다:

$$
\text{compression ratio}
= \frac{\text{original size}}{\text{compressed size}}
$$

또는 퍼센트 감소량:

$$
\text{size reduction(\%)} =
\left(1 - \frac{\text{compressed size}}{\text{original size}}\right)\times 100
$$

예: 100 MB 로그 파일이 20 MB로 줄었다면
압축 비율은 $$\frac{100}{20} = 5:1$$, 사이즈 감소는 $$80\%$$.

**Entropy** 관점에서 보면, 이상적인 압축은 **Shannon entropy**에 가까운 평균 비트수를 사용한다:

$$
H = -\sum_i p_i \log_2 p_i
$$

여기서 \(p_i\)는 심볼 \(i\)가 나올 확률이다. 이론적으로 평균 비트수는 최소 \(H\) 비트/심볼까지 줄일 수 있다.

---

#### 2) 대표적인 Lossless 알고리즘 계열

현대 시스템에서 자주 쓰이는 lossless 알고리즘은 대략 세 계열로 나뉜다:

1. **Entropy coding** (Huffman, Arithmetic coding 등)
2. **Dictionary-based** (LZ77/78, LZW, DEFLATE, LZMA, Zstd 등)
3. **Run-Length / 변환 기반** (RLE, Burrows–Wheeler Transform + Move-to-front 등)

최근 유럽계 연구에서 Huffman, LZW, Arithmetic 세 가지 entropy 코딩을 비교한 결과,
압축률·인코딩/디코딩 시간 관점에서 Huffman과 LZW가 여전히 좋은 trade-off를 보여준다는 분석도 있다.

##### (a) 엔트로피 코딩

- **Huffman coding**
  - 자주 나오는 심볼에 짧은 비트열, 드문 심볼에 긴 비트열 할당
  - Prefix-free 코드라서 디코딩이 모호하지 않다
  - PNG·GZIP·DEFLATE 내부, JPEG·MP3의 마지막 단계 등에서 사용

- **Arithmetic coding**
  - 전체 메시지를 [0,1) 구간의 실수 하나로 인코딩하는 개념
  - 이론적으로 entropy에 더 가깝게 근접 가능
  - 특허·복잡도 문제 때문에 실무에서는 Huffman보다 덜 보급됐지만, 최신 코덱 내부에는 변형된 형태로 많이 들어간다

##### (b) 사전 기반 코딩(LZ 계열)

- **LZ77 / LZ78 / LZW**
  - 데이터 안에서 이미 나온 패턴을 “사전(dictionary)”에 등록하고, 다음에는 “사전의 인덱스”만 보내는 방식
- **DEFLATE (ZIP, GZIP, PNG 내부)**
  - LZ77 + Huffman 결합
  - 범용적인 텍스트/바이너리 압축의 사실상 표준
- **LZMA / Zstd**
  - LZ 계열을 더 공격적으로 확장 + 엔트로피 코딩
  - Git, 패키지 매니저, 백업 등 고압축이 중요한 곳에서 많이 사용

##### (c) Run-length와 변환 기반

- **RLE (Run-Length Encoding)**: 같은 심볼이 연속될 때 `심볼+반복길이`로 압축
- **BWT(Burrows–Wheeler Transform)**:
  - 문자열을 재배열해 같은 문자가 뭉치도록 만든 뒤, RLE+Entropy 코딩을 적용
  - bzip2 같은 곳에서 사용

---

#### 3) Lossless 압축의 사용 사례

- **텍스트 / 소스코드 / 로그**
  - UTF-8 텍스트, JSON, XML 등은 공백·반복 구조가 많아 gzip, zstd 등으로 3~10배 이상 압축 가능
- **이미지**
  - PNG(무손실, DEFLATE 기반), 무손실 WebP, JPEG XL의 lossless 모드 등은 스크린샷, UI 아이콘, 그래픽에서 많이 사용된다.
  - 최근 연구에서 PNG/JPEG/WebP/AVIF를 비교해 보면, 무손실·고품질 설정에서 WebP·AVIF가 JPEG 대비 더 작은 크기를 제공하는 사례가 다수 보고된다.
- **오디오**
  - FLAC, ALAC와 같은 무손실 오디오 코덱은 CD 품질 PCM(44.1 kHz, 16bit)을 대략 30–60% 정도로 줄이면서 완전히 복원 가능하다.

---

#### 4) 파이썬 Cookbook — 텍스트/로그 무손실 압축

**상황 예시**

- 하루에 수 GB씩 쌓이는 웹 서버 로그를 **저장 공간·전송 비용**을 줄이기 위해 압축하고 싶다.
- 로그는 어차피 사람이 직접 읽기보다는, 나중에 분석 스크립트로 처리하므로 압축 형식을 사용하는 것이 자연스럽다.

##### 레시피 1: `gzip`으로 텍스트 로그 압축하기

```python
import gzip
import shutil
from pathlib import Path

def compress_log(path: str) -> None:
    src = Path(path)
    dst = src.with_suffix(src.suffix + '.gz')

    with src.open('rb') as fin, gzip.open(dst, 'wb', compresslevel=9) as fout:
        shutil.copyfileobj(fin, fout)

    orig_size = src.stat().st_size
    comp_size = dst.stat().st_size
    ratio = orig_size / comp_size if comp_size else 1

    print(f"{src.name}: {orig_size/1024:.1f} KB -> {comp_size/1024:.1f} KB (x{ratio:.2f})")

compress_log("access.log")
```

- `compresslevel=9`는 최대 압축(느리지만 크기 최소)
- 나중에 분석할 때는 `gzip.open()`으로 그대로 텍스트 스트림처럼 읽으면 된다.

##### 레시피 2: 메모리 상의 데이터를 `zlib`으로 압축

```python
import zlib

data = b"ERROR: something happened\n" * 1000

compressed = zlib.compress(data, level=9)
decompressed = zlib.decompress(compressed)

assert decompressed == data
print(f"Original={len(data)}, Compressed={len(compressed)}")
```

---

### Lossy Compression

#### 1) 개념과 Rate–Distortion 관점

**Lossy 압축**은 압축 과정에서 **사람이 거의 구분하지 못하는 정보**를 의도적으로 버리는 방식이다. 압축 해제 후 데이터는 원본과 다르지만, 사람 눈·귀에는 충분히 비슷하게 느껴진다.

주로 이미지, 오디오, 비디오 같은 **멀티미디어**에 사용한다.

- **Rate(비트레이트)**: 초당 또는 픽셀당 쓰는 비트 수
- **Distortion(왜곡)**: 원본과 재생된 신호의 차이(PSNR, SSIM, PESQ 등으로 측정)

압축 설계의 목표는:

$$
\text{주어진 비트레이트에서 왜곡 최소화}
\quad\text{또는}\quad
\text{주어진 품질에서 비트레이트 최소화}
$$

---

#### 2) 이미지 Lossy 압축

대표적인 파이프라인(예: JPEG, AVIF, JPEG XL 등)은 아래 단계를 공유한다.

1. 컬러 공간 변환 (RGB → YCbCr, 혹은 다른 perceptual space)
2. 공간 도메인 → 주파수 도메인 변환 (DCT, DWT 등)
3. **Quantization(양자화)**: 사람 눈이 덜 민감한 성분을 더 과감히 줄임
4. Zigzag 스캔 + RLE + Entropy coding(Huffman/Arithmetic 등)

최근 연구들 요약:

- 전통 JPEG 대비, **WebP·AVIF**는 같은 시각적 품질에서 평균 15–25% 정도 더 작은 파일 크기를 보이는 경향.
- **JPEG XL**은 기존 JPEG/PNG 대비 20~50% 수준의 추가 절감 + 무손실 JPEG 재압축 기능을 제공하도록 설계되었다.

##### 예시: 웹 썸네일 이미지

- 원본: 4000×3000, DSLR 사진, 8-bit RGB PNG, 8 MB
- JPEG(품질 80): 1.2 MB
- WebP(품질 80): 800 KB
- AVIF(품질 40 정도): 600 KB

브라우저·코덱에 따라 수치는 달라지지만, 전통 JPEG 대비 최신 코덱이 확실히 이득이 있다는 것이 여러 유럽 연구들에서 확인된다.

---

#### 3) 오디오 Lossy 압축

Lossy 오디오는 **인간 청각 시스템(Human Auditory System)** 의 특성을 이용한다.

핵심 아이디어:

- 사람이 거의 못 듣는 주파수/음량 영역은 과감히 버린다.
- 시간·주파수 마스킹 효과를 이용해, 강한 소리 주변의 약한 소리는 생략.
- 결과적으로 원본 대비 10~20배 이상 비트레이트를 줄이면서도 “거의 같은 것처럼” 들리게 한다.

대표 코덱:

- **MP3**: 고전적 표준, 여전히 광범위하게 지원
- **AAC(Advanced Audio Coding)**:
  - MP3보다 더 정교한 psychoacoustic 모델 사용
  - 같은 품질에서 30~50% 정도 비트레이트 절감 가능
  - 대형 스트리밍 서비스(YouTube, Apple 등)에서 사실상 표준.
- **Opus**:
  - IETF 표준, 완전 오픈·로열티 프리
  - 음성부터 음악까지 모두 커버, 저지연 특화
  - WebRTC·게임·VoIP 등에 널리 사용.

---

#### 4) 비디오 Lossy 압축

비디오는 **시간 축이 있는 이미지 + 오디오**로 볼 수 있다. 현대 비디오 코덱(H.264, H.265, VP9, AV1, VVC 등)은 각각의 프레임이 아니라, **예측 + 잔차(residual)** 를 전송한다.

구조 개요:

- I-frame: 독립적으로 디코딩 가능한 프레임(이미지)
- P/B-frame: 이전/이후 프레임으로부터 **motion compensation** 을 통해 예측, 예측 오차만 압축
- GOP(Group of Pictures): I + P/B 프레임의 그룹

최근 4K/8K 비디오에서의 비교 연구들에 따르면:

| 코덱 | 기준 대비 비트레이트 절감(대략) | 비고 |
|------|--------------------------------|------|
| H.264/AVC | 기준 | 호환성 최고, 여전히 지배적 |
| H.265/HEVC | H.264 대비 30–50% 절감 | UHD, HDR에서 많이 사용 |
| VP9 | H.264 대비 ~30% 절감 | 웹 스트리밍용, 오픈 |
| AV1 | VP9 대비 ~30% 추가 절감 | 오픈, CPU 부담 큼 |
| H.266/VVC | H.265 대비 30–50% 절감 | 최신, 아직 보급 중 |

---

#### 5) 파이썬 Cookbook — 이미지 Lossy 압축 실험

**상황 예시**

- 블로그에 올릴 썸네일 이미지를 자동으로 JPEG/WebP 두 버전으로 만들고, 어느 쪽이 더 작은지 비교하고 싶다.
- 서버는 Python + Pillow를 사용한다.

```python
from pathlib import Path
from PIL import Image

def export_image_variants(path: str):
    src = Path(path)
    img = Image.open(src).convert("RGB")

    # JPEG (품질 80)
    jpeg_path = src.with_suffix(".jpg")
    img.save(jpeg_path, "JPEG", quality=80, optimize=True, progressive=True)

    # WebP (품질 80)
    webp_path = src.with_suffix(".webp")
    img.save(webp_path, "WEBP", quality=80, method=6)

    s_jpeg = jpeg_path.stat().st_size
    s_webp = webp_path.stat().st_size

    print(f"JPEG: {s_jpeg/1024:.1f} KB, WebP: {s_webp/1024:.1f} KB")
    if s_webp < s_jpeg:
        print("WebP가 더 효율적이다.")
    else:
        print("JPEG가 더 작거나 비슷하다.")

export_image_variants("photo.png")
```

---

## Multimedia Data

### Text

#### 1) 구조와 특성

텍스트는 보통 **문자열 + 인코딩**(UTF-8 등)으로 표현된다.

- 로그, 소스코드, JSON, XML, HTML, CSV 등
- 많은 **반복 구조**와 **문자 패턴**을 갖고 있어 lossless 압축에 매우 적합
- 작은 오타 하나도 치명적일 수 있기 때문에 일반적으로 **Lossy 압축을 하지 않는다**

텍스트의 정보량을 거칠게 추정해 보면:

- 알파벳 26자 + 공백/구두점 → 대략 40~60 심볼
- 균등 분포 가정 시 entropy는 \( \log_2(60) \approx 5.9\) 비트/문자
- 실제 영어는 더 불균형(‘e’, ‘space’가 훨씬 자주 등장)해서 1~2비트/문자까지 근접 가능하다는 실험도 있다.

#### 2) 파이썬 Cookbook — JSON 로그 압축/해제

```python
import gzip
import json
from pathlib import Path

def write_compressed_json(path: str, records):
    path = Path(path)
    with gzip.open(path, "wt", encoding="utf-8") as f:
        for rec in records:
            f.write(json.dumps(rec, ensure_ascii=False))
            f.write("\n")

def read_compressed_json(path: str):
    path = Path(path)
    with gzip.open(path, "rt", encoding="utf-8") as f:
        for line in f:
            yield json.loads(line)

# 사용 예시

records = [{"user": i, "event": "login"} for i in range(100000)]
write_compressed_json("logs.json.gz", records)

print(sum(1 for _ in read_compressed_json("logs.json.gz")))
```

- S3에 gzip된 텍스트를 그대로 올려두고, Athena/BigQuery 등에서 바로 읽을 수도 있다.
- 네트워크 전송과 저장 비용 모두 감소.

---

### Image

#### 1) 구조: 비트맵 vs 벡터, 색 공간

**비트맵 이미지(raster)**

- 픽셀 그리드: width × height
- 각 픽셀은 색상 채널(RGB, RGBA, YCbCr 등)과 비트를 가진다.
- 예: 1920×1080, 24bit RGB
  - 픽셀 수 = 2,073,600
  - 원시 데이터 크기:
    $$
    2{,}073{,}600 \times 24 \text{ bits} \approx 49{,}766{,}400 \text{ bits} \approx 6 \text{ MB}
    $$

**벡터 이미지**

- 도형, 곡선, 텍스트 등의 수학적 정의
- 확대해도 깨지지 않지만, 사진 같은 데이터에는 적합하지 않다.

#### 2) 주요 이미지 포맷과 압축

최근 유럽/북미권 연구에서 자주 다루는 포맷들은 다음과 같다.

| 포맷 | 압축 | 특징(요약) |
|------|------|-----------|
| PNG | 무손실 | DEFLATE 기반, 웹 그래픽/스크린샷에 적합 |
| JPEG | 손실 | 전통 사진용. DCT 기반. 호환성 최고 |
| WebP | 손실/무손실 | 구글 개발, JPEG·PNG 대비 월등한 압축 성능 보고 다수 |
| AVIF | 손실/무손실 | AV1 기반, 매우 높은 압축 효율, 인코딩 느림 |
| JPEG XL | 손실/무손실 | JPEG·PNG 대비 20~50% 추가 절감, 무손실 JPEG 재압축 지원 |

웹 트래픽 분석 결과, 여전히 JPEG/PNG가 대부분을 차지하지만,
WebP와 AVIF가 점점 점유율을 늘리면서 페이지 로딩 시간을 15–21% 정도 줄이는 사례가 보고된다.

#### 3) 파이썬 Cookbook — 웹 이미지 파이프라인 뼈대

**상황 예시**

- 원본 이미지를 업로드하면, 서버가 자동으로 “원본 보정용 PNG + 웹용 WebP/AVIF 썸네일”을 생성하는 파이프라인을 만들고 싶다.

```python
from pathlib import Path
from PIL import Image

def make_web_images(src_path: str, max_width=1280):
    src = Path(src_path)
    img = Image.open(src).convert("RGB")

    # 크기 조정
    w, h = img.size
    if w > max_width:
        ratio = max_width / w
        img = img.resize((max_width, int(h * ratio)), Image.LANCZOS)

    # 마스터 PNG (무손실)
    png_path = src.with_suffix(".png")
    img.save(png_path, "PNG", optimize=True)

    # WebP (손실)
    webp_path = src.with_suffix(".webp")
    img.save(webp_path, "WEBP", quality=80, method=6)

    # AVIF (Pillow가 빌드 옵션에 따라 지원)
    avif_path = src.with_suffix(".avif")
    try:
        img.save(avif_path, "AVIF", quality=35)
    except ValueError:
        print("AVIF를 지원하지 않는 Pillow 빌드입니다.")

    for p in [png_path, webp_path, avif_path]:
        if p.exists():
            print(p.name, p.stat().st_size / 1024, "KB")
```

---

### Video

#### 1) 비디오 데이터 구조

비디오는 **시간 축을 가진 이미지 시퀀스 + 오디오**다.

- 해상도: 1920×1080, 3840×2160(4K) 등
- 프레임 레이트: 24/30/60 fps
- 색 공간: YUV 4:2:0(서브샘플링) 등

압축되지 않은 1080p 30fps YUV 4:2:0 8bit 비디오의 대략적인 비트레이트:

- 픽셀/프레임: 1920×1080 ≈ 2.07M
- YUV 4:2:0에서 평균 12bit/픽셀 정도로 잡으면
  - 2.07M × 12 ≈ 24.8M bit/프레임
  - 30 fps → 약 744 Mbit/s ≈ 93 MB/s

실제 스트리밍에서는 이걸 **수십~수백 배** 줄여야 한다.

#### 2) 비디오 코덱과 컨테이너

- 컨테이너: MP4, MKV, WebM 등
- 코덱: H.264/AVC, H.265/HEVC, VP9, AV1, VVC/H.266 등

최근 4K·8K 환경에서의 코덱 성능 비교 연구에 따르면:

- H.265/HEVC: H.264 대비 약 30–50% 비트레이트 절감
- AV1: VP9 대비 대략 30% 추가 절감
- H.266/VVC: HEVC 대비 30–50% 절감 가능하지만, 인코딩 복잡도와 특허 문제 등으로 아직 도입 속도는 제한적

#### 3) 파이썬 Cookbook — FFmpeg로 비디오 트랜스코딩 호출

**상황 예시**

- 로컬에서 촬영한 H.264 MP4 파일을 **저장용 HEVC 또는 AV1**로 다시 인코딩하고, 크기·비트레이트를 비교하고 싶다.

```python
import subprocess
from pathlib import Path

def transcode_to_hevc(src_path: str):
    src = Path(src_path)
    out = src.with_name(src.stem + "_hevc.mp4")

    cmd = [
        "ffmpeg",
        "-i", str(src),
        "-c:v", "libx265",
        "-crf", "28",   # 품질 조정
        "-c:a", "copy",
        str(out),
    ]
    subprocess.run(cmd, check=True)

    print("H.264:", src.stat().st_size / (1024*1024), "MB")
    print("HEVC :", out.stat().st_size / (1024*1024), "MB")

transcode_to_hevc("input_h264.mp4")
```

- 같은 시각적 품질에서 HEVC가 얼마나 비트레이트를 줄여주는지 직접 확인 가능하다.

---

### Audio

#### 1) 기본 개념: 샘플링, 양자화, 비트레이트

아날로그 오디오를 디지털로 변환할 때는:

1. **샘플링**: 시간 축을 일정 간격으로 나눠, 각 시점의 신호값을 측정
2. **양자화**: 측정된 값(실수)을 유한한 비트수로 근사

CD 품질 PCM의 비트레이트:

- 샘플링 레이트: 44.1 kHz
- 비트 깊이: 16 bit
- 채널: 2(스테레오)

$$
\text{bitrate} = 44{,}100 \times 16 \times 2 \approx 1.41 \text{ Mbit/s}
$$

압축되지 않은 WAV는 대략 1.4 Mbit/s.
Lossy 코덱을 사용하면 같은 체감 품질에서 128~256 kbit/s 정도까지 줄일 수 있다.

#### 2) Lossless vs Lossy 오디오 코덱

- **Lossless**: FLAC, ALAC
  - 복원 시 원본 비트스트림과 완전히 동일
  - CD 리핑·아카이브·마스터링에 사용
- **Lossy**: MP3, AAC, Opus 등
  - 스트리밍·다운로드 음악·게임/VoIP에 사용
  - 효율 차이는 코덱, 프로파일, 비트레이트에 따라 큼

#### 3) 파이썬 Cookbook — WAV → FLAC/AAC/Opus 변환 자동화

실제 인코딩은 보통 FFmpeg나 외부 도구를 호출한다.

```python
import subprocess
from pathlib import Path

def encode_audio_variants(path: str):
    src = Path(path)
    assert src.suffix.lower() == ".wav"

    # FLAC (무손실)
    flac = src.with_suffix(".flac")
    subprocess.run(["ffmpeg", "-y", "-i", str(src), "-c:a", "flac", str(flac)], check=True)

    # AAC (손실, 128kbps)
    aac = src.with_suffix(".m4a")
    subprocess.run([
        "ffmpeg", "-y", "-i", str(src),
        "-c:a", "aac", "-b:a", "128k",
        str(aac)
    ], check=True)

    # Opus (손실, 96kbps)
    opus = src.with_suffix(".opus")
    subprocess.run([
        "ffmpeg", "-y", "-i", str(src),
        "-c:a", "libopus", "-b:a", "96k",
        str(opus)
    ], check=True)

    for f in [flac, aac, opus]:
        print(f.name, f.stat().st_size / 1024, "KB")

encode_audio_variants("input.wav")
```

- FLAC은 원본 WAV 대비 40–70% 정도 크기를 줄이면서 무손실.
- AAC/Opus는 그보다 훨씬 더 작은 크기를 제공하지만 손실.

---

## 전체 정리

1. **Lossless 압축**은 텍스트, 소스코드, 데이터베이스 덤프, 일부 이미지(스크린샷, UI 그래픽) 등에 필수적이다.
   - Huffman, LZ 계열, Zstd, PNG, FLAC 같은 기술로 구현된다.
2. **Lossy 압축**은 이미지·비디오·오디오에서 사람의 인지 특성을 이용해 매우 높은 압축률을 제공한다.
   - JPEG/WebP/AVIF/JPEG XL, AAC/Opus, H.264/HEVC/AV1/VVC 등의 현대 코덱이 대표적이다.
3. **텍스트/이미지/비디오/오디오** 각각은 데이터 구조와 인간의 지각 특성이 다르기 때문에, **서로 다른 압축 전략**이 적용된다.
4. 파이썬 Cookbook 관점에서:
   - `gzip`, `zlib`으로 텍스트·로그 압축
   - Pillow로 이미지 포맷 변환 및 품질 조정
   - FFmpeg를 subprocess로 호출해 비디오/오디오 코덱 변환
   - 이런 레시피들을 조합하면, **웹 서비스·스트리밍·백업 시스템 전체의 저장·전송 비용을 체계적으로 줄이는** 자동화 파이프라인을 만들 수 있다.
