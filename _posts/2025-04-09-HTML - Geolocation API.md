---
layout: post
title: HTML - Geolocation API
date: 2025-04-09 21:20:23 +0900
category: HTML
---
# 🌍 HTML5 Geolocation API 완벽 가이드

## ✅ Geolocation이란?

**Geolocation API**는 사용자의 **현재 위치(위도/경도)**를 가져오는 HTML5 API입니다.  
웹 애플리케이션에서 **지도 표시, 날씨 제공, 위치 기반 검색, 근처 서비스 추천** 등 다양한 위치 기반 기능을 구현할 수 있게 해줍니다.

> 📌 사용자의 **명시적인 허가** 없이 위치 정보를 가져올 수 없습니다.  
> 대부분의 브라우저는 위치 접근 시 **팝업을 통해 동의 여부**를 확인합니다.

---

## 🧱 기본 사용법

### 1️⃣ 현재 위치 가져오기 (`getCurrentPosition()`)

```javascript
navigator.geolocation.getCurrentPosition(successCallback, errorCallback, options);
```

### 예제

```html
<script>
function showLocation() {
  if (navigator.geolocation) {
    navigator.geolocation.getCurrentPosition(
      position => {
        const lat = position.coords.latitude;
        const lon = position.coords.longitude;
        alert(`현재 위치: 위도 ${lat}, 경도 ${lon}`);
      },
      error => {
        alert("위치 정보를 가져올 수 없습니다: " + error.message);
      }
    );
  } else {
    alert("이 브라우저는 Geolocation을 지원하지 않습니다.");
  }
}
</script>

<button onclick="showLocation()">내 위치 가져오기</button>
```

---

## 📌 반환 객체 구조

```javascript
position = {
  coords: {
    latitude: 37.123,
    longitude: 127.456,
    altitude: null,              // 고도 (일부 장치만 제공)
    accuracy: 50,                // 위치 정확도 (미터 단위)
    altitudeAccuracy: null,
    heading: null,               // 방향 (도 단위)
    speed: null                  // 속도 (m/s)
  },
  timestamp: 1624567890000       // 타임스탬프
}
```

---

## ⚙️ 옵션 설정 (세 번째 인자)

```javascript
{
  enableHighAccuracy: true,  // 더 정확한 위치 (GPS) 요청
  timeout: 10000,            // 최대 응답 대기 시간 (ms)
  maximumAge: 0              // 이전 위치 캐시 사용 시간 (ms)
}
```

예:

```javascript
navigator.geolocation.getCurrentPosition(success, error, {
  enableHighAccuracy: true,
  timeout: 5000,
  maximumAge: 0
});
```

---

## 🔄 위치 지속 추적 (`watchPosition()`)

사용자의 위치가 **실시간으로 바뀔 때마다 추적**하고 싶은 경우 `watchPosition()`을 사용합니다.

```javascript
const watchId = navigator.geolocation.watchPosition(successCallback, errorCallback);
```

> 위치 추적 중지: `navigator.geolocation.clearWatch(watchId)`

---

## 🚫 에러 처리

`errorCallback` 함수의 `error.code`는 다음과 같은 값을 가질 수 있습니다:

| 코드 | 의미 |
|------|------|
| 1    | PERMISSION_DENIED (사용자가 위치 접근을 거부함) |
| 2    | POSITION_UNAVAILABLE (위치 정보 사용 불가) |
| 3    | TIMEOUT (시간 초과) |

예:

```javascript
function error(err) {
  switch (err.code) {
    case err.PERMISSION_DENIED:
      alert("위치 접근이 거부되었습니다.");
      break;
    case err.POSITION_UNAVAILABLE:
      alert("위치를 찾을 수 없습니다.");
      break;
    case err.TIMEOUT:
      alert("요청 시간이 초과되었습니다.");
      break;
  }
}
```

---

## 🔐 보안 및 HTTPS 요구사항

- **Geolocation API는 반드시 HTTPS에서만 동작**합니다. (또는 localhost)
- 이는 사용자의 **위치 정보 보호를 위한 보안 정책**입니다.
- HTTP에서 동작하지 않으며, 콘솔에 에러가 출력됩니다.

---

## 🌐 브라우저 지원

| 브라우저 | 지원 여부 |
|----------|------------|
| Chrome | ✅ |
| Firefox | ✅ |
| Edge | ✅ |
| Safari | ✅ |
| IE 9+ | ✅ (구식 API 사용 가능) |

> 📱 모바일 브라우저에서도 폭넓게 지원되며, GPS 센서와 연동됩니다.

---

## 📍 예시: Google Maps에 사용자 위치 표시하기

```html
<div id="map" style="width:100%;height:300px;"></div>

<script>
function initMap() {
  if (navigator.geolocation) {
    navigator.geolocation.getCurrentPosition(position => {
      const lat = position.coords.latitude;
      const lng = position.coords.longitude;
      const location = { lat, lng };
      
      const map = new google.maps.Map(document.getElementById("map"), {
        center: location,
        zoom: 14
      });

      new google.maps.Marker({
        position: location,
        map: map
      });
    });
  }
}
</script>

<!-- Google Maps API -->
<script async defer
  src="https://maps.googleapis.com/maps/api/js?key=YOUR_API_KEY&callback=initMap">
</script>
```

---

## ✅ Geolocation 사용 시 유의사항

- 사용자의 **명시적 동의**가 필수
- 배터리 소모에 주의 (고정밀 추적 시)
- 위치 정보를 외부 서버로 전송할 경우 **보안 및 개인정보 보호 법규** 준수 필요
- 위치 정보를 캐시할 경우 **최대한 익명화** 필요

---

## 🧠 활용 예시

| 분야 | 예시 |
|------|------|
| 여행 | 현재 위치 기반 관광지 추천 |
| 날씨 | 지역 날씨 자동 제공 |
| 내비게이션 | 실시간 길찾기, 위치 공유 |
| 커머스 | 근처 매장 찾기, 배달 범위 제한 |
| 헬스 | 운동 경로 추적 (watchPosition 사용)

---

## 📚 참고 링크

- [MDN Web Docs - Geolocation API](https://developer.mozilla.org/ko/docs/Web/API/Geolocation_API)
- [W3C Geolocation API 명세](https://www.w3.org/TR/geolocation-API/)
- [Can I Use - Geolocation](https://caniuse.com/?search=geolocation)