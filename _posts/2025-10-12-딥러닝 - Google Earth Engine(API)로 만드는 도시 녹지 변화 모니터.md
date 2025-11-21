---
layout: post
title: 딥러닝 - Google Earth Engine(API)로 만드는 도시 녹지 변화 모니터
date: 2025-10-12 21:25:23 +0900
category: 딥러닝
---
# Google Earth Engine(API)로 만드는 “도시 녹지 변화 모니터” — 엔드투엔드 프로젝트 블로그(예제·코드·시각화·내보내기까지)

> 한 줄 요약
> **Google Earth Engine(GEE) Python API**만으로 서울 권역을 대상으로 **2019→2024년** 사이의 **녹지(식생) 변화**를 추정·시각화하고, 결과를 **지도/차트**로 보고 **이미지(GeoTIFF)**로 내보내는 엔드투엔드 프로젝트를 만듭니다.
> - 핵심 기술: Sentinel-2 SR, **NDVI**(Normalized Difference Vegetation Index), 클라우드 마스킹, 시즌별 합성, 변화(ΔNDVI) 지도, 타임시리즈, Folium 지도 시각화, Google Drive 내보내기
> - 모든 코드는 **Python + earthengine-api + folium + matplotlib**만 사용(추가 프레임워크 불필요)

---

## 프로젝트 목표·로드맵

- **목표**: “서울권 식생(녹지) 상태가 2019년 대비 2024년에 **어디서** 늘고/줄었는지”를 한 눈에 보여주는 **변화 지도**·**요약 차트**·**다운로드 가능한 결과물**을 만든다.
- **데이터**: Sentinel-2 SR(10m) — 빨강(RED=B4), 근적외(NIR=B8), 구름마스크(QA60)
- **절차**
  1) **환경 준비**(로그인/초기화)
  2) **ROI(관심영역)** 설정: 서울권(간단한 사각형 ROI로 시작)
  3) **클라우드 마스킹** + **NDVI 밴드** 추가 함수
  4) **시즌별 합성**: 2019/2024년 **‘잎이 무성한 계절’(4~10월)** median 합성
  5) **변화(ΔNDVI=2024−2019)** 지도 계산
  6) **Folium 지도 시각화**(타일 오버레이)
  7) **타임시리즈**(월별 평균 NDVI)와 **변화량 요약**
  8) **Google Drive 내보내기**(GeoTIFF)
  9) **품질 점검/유효성 체크** 및 확장 아이디어

---

## 환경 준비

> 사전 요건
> - [earthengine.google.com](https://earthengine.google.com/)에서 Earth Engine 계정을 활성화했습니다.
> - 로컬/Colab/Notebook 어느 환경이든 Python 실행 가능.

```python
# Colab/로컬 패키지 설치
# !pip install -q earthengine-api folium matplotlib

import ee, folium, math, time
import matplotlib.pyplot as plt

# 인증
# # 대화형 브라우저 인증 팝업 (주석 해제 후 1회 실행)

# 세션 초기화

ee.Initialize()
print("GEE initialized:", ee.Number(1).getInfo())
```

---

## 프로젝트 변수를 정하자 (ROI, 기간, 파라미터)

- **ROI**: 서울권을 대략 포함하는 사각형(간단화를 위해 대략 경계)
  - 위도 37.35~37.75, 경도 126.70~127.30 (임의·예시)
- **기간**: 2019/04/01~2019/10/31, 2024/04/01~2024/10/31
- **센서**: `COPERNICUS/S2_SR` (Sentinel-2 Surface Reflectance)

```python
# — 간단한 사각형(경도, 위도 순서)

roi = ee.Geometry.Rectangle([126.70, 37.35, 127.30, 37.75])  # (minLon, minLat, maxLon, maxLat)

# 분석 기간

periods = {
    "ref":  ("2019-04-01", "2019-10-31"),  # 기준연도(Reference)
    "curr": ("2024-04-01", "2024-10-31"),  # 비교연도(Current)
}

# 시각화 파라미터(색표현)

vis_truecolor = {"min": 0, "max": 3000, "bands": ["B4", "B3", "B2"]}
vis_ndvi      = {"min": 0.0, "max": 0.8,  "palette": ["brown","beige","yellow","lightgreen","green"]}
vis_diff      = {"min": -0.3, "max": 0.3, "palette": ["#313695","#74add1","#fefefe","#f46d43","#a50026"]}  # 블루(감소)↔레드(증가)
```

---

## 전처리 유틸 함수: 클라우드 마스킹 + NDVI 밴드

- **구름 마스크(QA60)**: Sentinel-2 SR의 QA60 밴드 **10, 11비트**에 구름/서큘러스 마스크 정보가 들어있습니다.
  - `cloudBitMask = 1 << 10`, `cirrusBitMask = 1 << 11`
  - 두 비트가 0인 픽셀만 유효(구름 없음)으로 간주.

- **NDVI**: \(\text{NDVI} = \frac{\mathrm{NIR}-\mathrm{RED}}{\mathrm{NIR}+\mathrm{RED}}\)
  - Sentinel-2: **NIR = B8**, **RED = B4**

```python
def s2_sr_cloud_mask(img: ee.Image) -> ee.Image:
    qa = img.select("QA60")
    cloudBitMask = 1 << 10
    cirrusBitMask = 1 << 11
    mask = qa.bitwiseAnd(cloudBitMask).eq(0).And(qa.bitwiseAnd(cirrusBitMask).eq(0))
    return img.updateMask(mask).copyProperties(img, img.propertyNames())

def add_ndvi(img: ee.Image) -> ee.Image:
    # Sentinel-2 SR은 스케일(1/10000)이 적용되어 있지만 비율 연산이라 NDVI 계산에는 영향 적음.
    ndvi = img.normalizedDifference(["B8", "B4"]).rename("NDVI")
    return img.addBands(ndvi)
```

---

## 연도별 시즌 합성(클린 모자이크)

> 목표: 2019/2024년의 **4~10월** 이미지에서 **구름 제거** 후, **median 합성**으로 대표 이미지를 만든다.

```python
S2 = ee.ImageCollection("COPERNICUS/S2_SR")

def seasonal_median(start_date: str, end_date: str, region: ee.Geometry) -> ee.Image:
    col = (S2.filterDate(start_date, end_date)
             .filterBounds(region)
             .map(s2_sr_cloud_mask)
             .map(add_ndvi))
    # 충분한 샘플만 남기고 median 합성
    composite = col.median().clip(region)
    return composite

ref_img  = seasonal_median(*periods["ref"],  roi)  # 2019 시즌
curr_img = seasonal_median(*periods["curr"], roi)  # 2024 시즌

# NDVI 밴드만 꺼내어 차이 계산(2024 - 2019)

ndvi_ref  = ref_img.select("NDVI")
ndvi_curr = curr_img.select("NDVI")
ndvi_diff = ndvi_curr.subtract(ndvi_ref).rename("NDVI_DIFF")
```

---

## Folium에 Earth Engine 레이어 붙이기

> **Folium**은 Leaflet 기반 파이썬 래퍼입니다. Earth Engine 이미지를 **타일 URL**로 가져와 오버레이할 수 있습니다.
> 아래 헬퍼는 `ee.Image.getMapId(vis_params)`에서 타일 URL을 얻고, Folium에 레이어로 추가합니다.

```python
def add_ee_layer(m, ee_image, vis_params, name, shown=True, opacity=1.0):
    # getMapId는 {mapid, tile_fetcher, token} 등을 반환
    map_id_dict = ee.Image(ee_image).getMapId(vis_params)
    folium.raster_layers.TileLayer(
        tiles = map_id_dict["tile_fetcher"].url_format,
        attr  = "Google Earth Engine",
        name  = name,
        overlay=True,
        control=True,
        show=shown,
        opacity=opacity
    ).add_to(m)

# 지도 생성 및 레이어 추가

center = [37.55, 127.00]  # 서울 중심부 근사
m = folium.Map(location=center, zoom_start=11, control_scale=True)

# 베이스맵 + 레이어

add_ee_layer(m, ref_img.select(["B4","B3","B2"]), vis_truecolor, name="TrueColor 2019", shown=False)
add_ee_layer(m, curr_img.select(["B4","B3","B2"]), vis_truecolor, name="TrueColor 2024", shown=False)
add_ee_layer(m, ndvi_ref,  vis_ndvi, name="NDVI 2019", shown=False)
add_ee_layer(m, ndvi_curr, vis_ndvi, name="NDVI 2024", shown=False)
add_ee_layer(m, ndvi_diff, vis_diff, name="ΔNDVI (2024-2019)", shown=True)

# ROI 테두리

folium.GeoJson(data=roi.getInfo(), name="ROI(Seoul-ish)").add_to(m)
folium.LayerControl(collapsed=False).add_to(m)

m
```

> 노트북/Colab에서 실행하면 인터랙티브 지도가 출력됩니다. 기본으로 **ΔNDVI** 레이어(녹지 증가=빨강, 감소=파랑)가 켜집니다.

---

## ΔNDVI 해석 가이드

- **양수(빨강)**: 2024년이 2019년보다 NDVI가 높음 → **녹지/식생 증가** 가능성(신규 조성, 계절·관리 영향 포함)
- **음수(파랑)**: 2024년이 더 낮음 → **녹지 감소** 가능성(공사, 포장, 건물 신축, 가뭄·계절 요인 등)
- **주의**:
  - **계절성** 오차를 줄이기 위해 **동일 시즌(4~10월)**만 비교했습니다.
  - 구름/연무/그림자, **관측 횟수 차이**(센서 가용성)도 영향.
  - 도시 지역의 **미세 단위(10m)** 변화를 해석할 때는 현장/부가 데이터와 교차검증 권장.

---

## + Matplotlib 차트

> ROI 전체 평균 NDVI의 월별 시계열을 만들고, 2019 vs 2024를 비교합니다.

```python
def monthly_ndvi_series(start_date, end_date, region):
    col = (S2.filterDate(start_date, end_date)
             .filterBounds(region)
             .map(s2_sr_cloud_mask)
             .map(add_ndvi))
    # 월별로 그룹핑
    months = ee.List.sequence(ee.Date(start_date).get("month"), ee.Date(end_date).get("month"))
    years  = ee.List.sequence(ee.Date(start_date).get("year"),  ee.Date(end_date).get("year"))
    # 실제로는 시작월~종료월 범위를 정교하게 구성해야 하지만 여기선 단순화:
    # 날짜 범위 내의 모든 이미지에 대해 월을 태그하고, 월별 평균 NDVI 산출
    def fmt(i):
        img = ee.Image(i)
        d = ee.Date(img.get("system:time_start"))
        m = d.format("YYYY-MM")
        mean = img.select("NDVI").reduceRegion(reducer=ee.Reducer.mean(), geometry=region, scale=10, maxPixels=1e12).get("NDVI")
        return ee.Feature(None, {"month": m, "ndvi": mean})
    feats = col.map(fmt).aggregate_array("properties")
    # Python 리스트로
    props = feats.getInfo()
    # 같은 month가 여러 번 나올 수 있으므로 평균
    from collections import defaultdict
    agg = defaultdict(list)
    for p in props:
        if p is None: continue
        agg[p["month"]].append(p["ndvi"])
    xs, ys = [], []
    for k in sorted(agg.keys()):
        vals = [v for v in agg[k] if v is not None]
        if not vals: continue
        xs.append(k); ys.append(sum(vals)/len(vals))
    return xs, ys

x19, y19 = monthly_ndvi_series(*periods["ref"],  roi)
x24, y24 = monthly_ndvi_series(*periods["curr"], roi)

plt.figure(figsize=(10,4))
plt.plot(x19, y19, marker="o", label="2019 NDVI(mean, ROI)")
plt.plot(x24, y24, marker="o", label="2024 NDVI(mean, ROI)")
plt.xticks(rotation=45); plt.grid(True); plt.legend()
plt.title("월별 평균 NDVI — ROI(서울권 근사)")
plt.tight_layout()
plt.show()
```

> 기대: 4~10월에 NDVI가 상승·피크·하강하는 시즌 패턴을 보며, **연도 간 차이**를 직관적으로 확인할 수 있습니다.

---

## 통계 요약(ROI 기준)

> ΔNDVI의 **평균/분위수/면적 비율**(증가·감소)을 ROI 차원에서 산출합니다.

```python
# ΔNDVI 이미지의 ROI 평균/표준편차/분위수

stats = ndvi_diff.reduceRegion(
    reducer = ee.Reducer.mean().combine(ee.Reducer.stdDev(), sharedInputs=True)
                            .combine(ee.Reducer.percentile([10,25,50,75,90]), sharedInputs=True),
    geometry = roi, scale = 10, maxPixels = 1e12
).getInfo()

print("ΔNDVI stats (ROI):")
for k,v in stats.items():
    print(f" - {k}: {v}")

# 증가/감소 면적 비율(단순 임계)

thr_inc = 0.05   # +0.05 이상 증가
thr_dec = -0.05  # -0.05 이하 감소

area_pixel = ee.Image.pixelArea().divide(1e6)  # km^2
inc_area = area_pixel.updateMask(ndvi_diff.gte(thr_inc)).reduceRegion(ee.Reducer.sum(), roi, 10, 1e12).get("area")
dec_area = area_pixel.updateMask(ndvi_diff.lte(thr_dec)).reduceRegion(ee.Reducer.sum(), roi, 10, 1e12).get("area")
tot_area = area_pixel.updateMask(ndvi_diff.mask()).reduceRegion(ee.Reducer.sum(), roi, 10, 1e12).get("area")

inc_km2 = (inc_area or ee.Number(0)).getInfo()
dec_km2 = (dec_area or ee.Number(0)).getInfo()
tot_km2 = (tot_area or ee.Number(0)).getInfo()

print(f"면적(km^2) — 증가(≥{thr_inc}): {inc_km2:.2f}, 감소(≤{thr_dec}): {dec_km2:.2f}, 총: {tot_km2:.2f}")
print(f"비율(%) — 증가: {100*inc_km2/max(1e-9,tot_km2):.2f}, 감소: {100*dec_km2/max(1e-9,tot_km2):.2f}")
```

> 임계(threshold)는 **도메인/잡음 수준**에 맞게 조정하세요(0.03~0.08 사이 탐색 권장).

---

## 결과 내보내기(Drive / Cloud Storage)

> 결과(GeoTIFF)를 **Google Drive**로 내보내는 방법입니다. (Drive 연결 필요)

```python
# 내보내기 이미지: ΔNDVI

export_img = ndvi_diff.clip(roi)

task = ee.batch.Export.image.toDrive(
    image = export_img,
    description = "NDVI_Diff_2019_2024_Seoul",
    folder = "GEE",                 # Drive의 'GEE' 폴더(없으면 자동 생성)
    fileNamePrefix = "ndvi_diff_2019_2024_seoul",
    region = roi,
    scale = 10,                     # 10m
    maxPixels = 1e13
)
task.start()
print("Export started:", task.id)

# 간단 폴링으로 상태 확인

while task.active():
    print("  Task state:", task.status().get("state"))
    time.sleep(5)
print("Done:", task.status())
```

> **팁**: 내보내기 전에 지도에서 ROI 범위를 적절히 좁혀 **용량/비용/시간**을 관리하세요.

---

## 유효성 검증(간단 체크리스트)

- [ ] ROI에 **물/바위/건물**만 있는 구역을 임의로 택해, NDVI가 낮게 나오는지 확인
- [ ] **대형 공원/산지**를 찍어 2019·2024 **둘 다 NDVI가 높고** ΔNDVI가 0 근처인지 확인
- [ ] **대규모 개발지역**(신도시/도로) 주변이 **음수(파랗게)** 나오는지 확인
- [ ] 구름 마스킹 실패(흰색 얼룩) 의심 구역은 **TrueColor** 레이어로 육안 확인

---

## 코드 한 번에(**함수화** + 깔끔 실행)

> 위 스니펫을 하나의 스크립트/노트북 셀로 묶습니다.

```python
import ee, folium, time
import matplotlib.pyplot as plt

ee.Initialize()

def build_project(roi, ref_range, curr_range):
    def s2_sr_cloud_mask(img):
        qa = img.select("QA60")
        cloudBitMask = 1 << 10
        cirrusBitMask = 1 << 11
        mask = qa.bitwiseAnd(cloudBitMask).eq(0).And(qa.bitwiseAnd(cirrusBitMask).eq(0))
        return img.updateMask(mask).copyProperties(img, img.propertyNames())

    def add_ndvi(img): return img.addBands(img.normalizedDifference(["B8","B4"]).rename("NDVI"))

    S2 = ee.ImageCollection("COPERNICUS/S2_SR")
    def seasonal_median(start, end):
        col = (S2.filterDate(start, end).filterBounds(roi)
                 .map(s2_sr_cloud_mask).map(add_ndvi))
        return col.median().clip(roi)

    ref  = seasonal_median(*ref_range)
    curr = seasonal_median(*curr_range)

    ndvi_ref  = ref.select("NDVI")
    ndvi_curr = curr.select("NDVI")
    ndvi_diff = ndvi_curr.subtract(ndvi_ref).rename("NDVI_DIFF")

    return ref, curr, ndvi_ref, ndvi_curr, ndvi_diff

def folium_map(roi, ref_img, curr_img, ndvi_ref, ndvi_curr, ndvi_diff):
    vis_true = {"min":0, "max":3000, "bands":["B4","B3","B2"]}
    vis_ndvi = {"min":0.0, "max":0.8, "palette":["brown","beige","yellow","lightgreen","green"]}
    vis_diff = {"min":-0.3, "max":0.3, "palette":["#313695","#74add1","#fefefe","#f46d43","#a50026"]}

    m = folium.Map(location=[37.55,127.00], zoom_start=11, control_scale=True)

    def add_layer(img, vis, name, shown=False, opacity=1.0):
        map_id = ee.Image(img).getMapId(vis)
        folium.raster_layers.TileLayer(
            tiles = map_id["tile_fetcher"].url_format,
            attr  = "Google Earth Engine", name=name, overlay=True, control=True, show=shown, opacity=opacity
        ).add_to(m)

    add_layer(ref_img.select(["B4","B3","B2"]), vis_true, "TrueColor 2019")
    add_layer(curr_img.select(["B4","B3","B2"]), vis_true, "TrueColor 2024")
    add_layer(ndvi_ref,  vis_ndvi, "NDVI 2019")
    add_layer(ndvi_curr, vis_ndvi, "NDVI 2024")
    add_layer(ndvi_diff, vis_diff, "ΔNDVI(2024-2019)", shown=True)
    folium.GeoJson(data=roi.getInfo(), name="ROI").add_to(m)
    folium.LayerControl(collapsed=False).add_to(m)
    return m

roi = ee.Geometry.Rectangle([126.70, 37.35, 127.30, 37.75])
ref_range  = ("2019-04-01", "2019-10-31")
curr_range = ("2024-04-01", "2024-10-31")

ref_img, curr_img, ndvi_ref, ndvi_curr, ndvi_diff = build_project(roi, ref_range, curr_range)
display(folium_map(roi, ref_img, curr_img, ndvi_ref, ndvi_curr, ndvi_diff))
```

---

## 포인트/구역별 드릴다운

> 지도에서 관심 지점을 찍어 **해당 지점의 NDVI 시계열**과 **ΔNDVI 값**을 보고 싶다면:

```python
# 관심 포인트(예: 남산타워 인근 근사 좌표)

pt = ee.Geometry.Point([126.988, 37.551])

# ΔNDVI 포인트 추출

val = ndvi_diff.sample(pt, scale=10).first().get("NDVI_DIFF").getInfo()
print("Point ΔNDVI (2024-2019):", val)

# 평균 NDVI 타임시리즈(2019/2024 각각)

buf = pt.buffer(100)
def series(start, end):
    col = (ee.ImageCollection("COPERNICUS/S2_SR").filterDate(start, end).filterBounds(buf)
             .map(s2_sr_cloud_mask).map(add_ndvi))
    def to_feat(img):
        d = ee.Date(img.get("system:time_start")).format("YYYY-MM")
        mean = img.select("NDVI").reduceRegion(ee.Reducer.mean(), buf, 10).get("NDVI")
        return ee.Feature(None, {"month": d, "ndvi": mean})
    props = col.map(to_feat).aggregate_array("properties").getInfo()
    from collections import defaultdict
    agg={}
    for p in props:
        if not p: continue
        agg.setdefault(p["month"], []).append(p["ndvi"])
    xs, ys = [], []
    for k in sorted(agg.keys()):
        vals = [v for v in agg[k] if v is not None]
        if vals: xs.append(k); ys.append(sum(vals)/len(vals))
    return xs, ys

x19, y19 = series(*ref_range)
x24, y24 = series(*curr_range)

plt.figure(figsize=(9,3.5))
plt.plot(x19, y19, marker="o", label="2019 (100m buffer)")
plt.plot(x24, y24, marker="o", label="2024 (100m buffer)")
plt.xticks(rotation=45); plt.grid(True); plt.legend(); plt.tight_layout()
plt.show()
```

---

## 정확도·강건성 팁

1. **계절 동치성**: 잎이 무성한 계절 같은 기간을 비교(여기선 4~10월) → 계절 편차 감소
2. **클라우드 마스킹**: QA60만으로 부족하면 **SCL(Scene Classification) 밴드** 활용(그림자·눈 등 추가 마스크)
3. **통계 요약**: ΔNDVI의 평균·분산·분위수로 **전체 경향** 파악 후, 임계 기반 증가/감소 지역 면적 요약
4. **다변량 보조**: **NDBI(도심화)**, **NDWI(수역)**도 함께 보면 해석력이 증가
   - NDBI ≈ (SWIR − NIR)/(SWIR + NIR) (S2: SWIR=B11)
   - NDWI ≈ (GREEN − NIR)/(GREEN + NIR) (S2: GREEN=B3)

---

## 오류·함정(트러블슈팅)

- **지도에 타일이 안 뜸**: 인증/초기화 누락(`ee.Initialize()`), 방화벽/네트워크 이슈, 혹은 Folium 렌더 경로 문제
- **Export 실패**: ROI 너무 큼 → `region`을 줄이고 `scale`을 키우거나, 타일 분할(여러 번 내보내기)
- **NDVI가 비정상**: 밴드명(B8/B4) 잘못 지정, NIR/RED 순서 바뀜, 마스크 후 픽셀 수 부족
- **속도 느림**: ROI 축소, 기간 축소, `median`→`mosaic` 또는 `qualityMosaic('NDVI')`로 단순화 시도

---

## 확장 아이디어

- **행정구역 단위 요약**: 구/동 경계(벡터)별 ΔNDVI 평균·면적 비율 계산 후 **Choropleth**
- **타임슬라이더 지도**: 2019/2024 레이어를 슬라이더로 전환(프런트 UI에서 지원)
- **산불·벌목 감시**: **NDVI 급감** 구간을 시계열로 탐지(변화점 탐지)
- **도시열섬 보조**: 야간 열관측·LST와 결합하여 ‘녹지 증가→열도 완화’ 상관 검증

---

## 라이선스·정책·운영

- **데이터 사용조건**: Sentinel-2 및 GEE의 이용약관/라이선스를 준수
- **재현성**: ROI/기간/스케일/임계/버전(스크립트 SHA) 기록
- **비용/쿼터**: 대규모 ROI/긴 기간/고해상도 Export는 시간 소요·쿼터 영향 → 사전 샘플링 권장

---

## — 한 번에 실행 가능한 최소 예제

```python
import ee, folium, time, matplotlib.pyplot as plt
ee.Initialize()

# Params

roi = ee.Geometry.Rectangle([126.70, 37.35, 127.30, 37.75])
REF = ("2019-04-01", "2019-10-31")
CUR = ("2024-04-01", "2024-10-31")

def s2_sr_cloud_mask(img):
    qa = img.select("QA60")
    cb, cr = 1<<10, 1<<11
    mask = qa.bitwiseAnd(cb).eq(0).And(qa.bitwiseAnd(cr).eq(0))
    return img.updateMask(mask)

def add_ndvi(img): return img.addBands(img.normalizedDifference(["B8","B4"]).rename("NDVI"))

def seasonal_median(r):
    col = (ee.ImageCollection("COPERNICUS/S2_SR")
           .filterDate(*r).filterBounds(roi).map(s2_sr_cloud_mask).map(add_ndvi))
    return col.median().clip(roi)

ref_img, cur_img = seasonal_median(REF), seasonal_median(CUR)
ndvi_ref, ndvi_cur = ref_img.select("NDVI"), cur_img.select("NDVI")
ndvi_diff = ndvi_cur.subtract(ndvi_ref).rename("NDVI_DIFF")

def add_ee_layer(m, img, vis, name, shown=False):
    mid = ee.Image(img).getMapId(vis)
    folium.raster_layers.TileLayer(
        tiles=mid["tile_fetcher"].url_format, attr="Google Earth Engine",
        name=name, overlay=True, control=True, show=shown).add_to(m)

vis_true = {"min":0, "max":3000, "bands":["B4","B3","B2"]}
vis_ndvi = {"min":0.0, "max":0.8, "palette":["brown","beige","yellow","lightgreen","green"]}
vis_diff = {"min":-0.3, "max":0.3, "palette":["#313695","#74add1","#fefefe","#f46d43","#a50026"]}

m = folium.Map(location=[37.55,127.00], zoom_start=11)
add_ee_layer(m, ref_img.select(["B4","B3","B2"]), vis_true, "TrueColor 2019")
add_ee_layer(m, cur_img.select(["B4","B3","B2"]), vis_true, "TrueColor 2024")
add_ee_layer(m, ndvi_ref,  vis_ndvi, "NDVI 2019")
add_ee_layer(m, ndvi_cur,  vis_ndvi, "NDVI 2024")
add_ee_layer(m, ndvi_diff, vis_diff, "ΔNDVI(2024-2019)", shown=True)
folium.GeoJson(roi.getInfo(), name="ROI").add_to(m)
folium.LayerControl(collapsed=False).add_to(m)
m
```

---

## 마무리

- 본 프로젝트는 **GEE Python API**로 **도시 녹지 변화**를 **정량화·시각화·내보내기**까지 구현한 **블로그형 튜토리얼**입니다.
- 여기서 출발해 **행정구역 레벨 통계, 변화점 탐지, 다른 지수(NDWI/NDBI)** 등을 곁들이면 즉시 실무/연구 대시보드의 토대가 됩니다.
- 필요시 **ROI(정확한 행정경계)**, **임계/시즌/마스크 전략**을 귀하의 도메인(SDG, 도시계획, ESG 등)에 맞춰 **커스터마이즈**해 드릴 수 있어요.
