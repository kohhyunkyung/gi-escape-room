# GI Escape Room — 최종 과제 리포트

> **게임 링크**: https://kohhyunkyung.github.io/gi-escape-room/
> **GitHub**: https://github.com/kohhyunkyung/gi-escape-room

---

## 1. 게임 개요

### 기획 의도

GI Escape Room은 **DDGI(Dynamic Diffuse Global Illumination)** 기술을 게임플레이의 핵심 메커닉으로 활용한 1인칭 방 탈출 게임이다. 단순한 시각 효과로서의 GI가 아닌, **조명을 켜야만 단서가 보이는** 퍼즐 구조를 통해 GI 기술이 게임플레이에 직접 통합되도록 설계하였다.

### 게임 시작 화면

![게임 시작](https://raw.githubusercontent.com/kohhyunkyung/gi-escape-room/main/escape%20room/%EA%B2%8C%EC%9E%84%20%EC%8B%9C%EC%9E%91.png)

### 게임 흐름

```
시작 화면 → 연구실(방 1) → 지하실(방 2) → 보안실(방 3) → 탈출 성공
```

| 방 | 배경 | GI 활용 방식 | 탈출 방법 |
|---|---|---|---|
| 연구실 | Cornell Box 스타일 (빨강/초록/파랑 벽) | GI ON → 빨간 벽에 숫자 826 등장 | 노트 읽기 → GI 켜서 826 확인 → 금고 열기 → 열쇠 획득 → 문 열기 |
| 지하실 | 정전으로 어두운 공간 | GI ON → 간접광으로 방 전체가 밝아짐 + 뒷벽에 5272 등장 | GI 켜서 탐색 → 퓨즈 줍기 → 퓨즈박스 수리 → 발전기 5272 → 배터리 획득 → 문 열기 |
| 보안실 | SF 분위기 보안 구역 | GI ON → 4방향 벽/천장에 각각 다른 색 숫자 등장 | GI 켜서 4곳 탐색 → 4321 조합 → 패널 해제 → 탈출 |

### 조작법

| 키 | 기능 |
|---|---|
| WASD | 이동 |
| 마우스 | 시점 회전 |
| E | 조작하기 (줍기 / 읽기 / 열기) |
| G | 조명(GI) ON/OFF |
| P | DDGI 프로브 시각화 |

---

## 2. 강의 내용과 구현 내용 매핑

### 2.1 렌더링 방정식 (Rendering Equation)

강의에서 다룬 렌더링 방정식은 다음과 같다:

$$L_o(\mathbf{x}, \omega_o) = L_e(\mathbf{x}, \omega_o) + \int_\Omega f_r(\mathbf{x}, \omega_i, \omega_o) L_i(\mathbf{x}, \omega_i)(\omega_i \cdot \mathbf{n}) d\omega_i$$

Phong 모델 같은 직접 조명(Direct Lighting)은 광원에서 오는 빛만 계산하므로 간접광을 표현하지 못한다. 본 게임에서는 이 한계를 극복하기 위해 DDGI를 적용하였다.

### 2.2 Real-time Global Illumination

실시간 GI는 매 프레임 모든 광선을 추적하는 대신, **조명 정보를 캐시에 저장하고 재사용**하는 방식으로 구현된다. 본 게임에서는 DDGI 방식을 채택하였다.

| GI 방식 | 조명 저장 방식 | 간접광 샘플링 | 특징 |
|---|---|---|---|
| **DDGI** (본 게임) | 프로브 SH (공간 격자) | 8개 프로브 trilinear interpolation | 동적 씬, 확산 GI |
| SurfelGI | Surfel SH (표면 부착) | 주변 Surfel 가중 평균 | 표면 기반, 효율적 |
| VoxelGI | Voxel Radiance (3D 격자) | Cone tracing | 정반사 지원 |

### 2.3 DDGI (Dynamic Diffuse Global Illumination)

#### 프로브 격자 구성

본 게임에서 각 방마다 **4×3×4 = 48개의 프로브**로 구성된 격자를 배치하였다.

```javascript
// 방 1 (연구실) 프로브 격자
new DDGI(
  new THREE.Vector3(-4.5, 0.2, -5.5),  // 격자 시작점
  new THREE.Vector3(9, 3.6, 10.5),      // 격자 크기
  [4, 3, 4]                              // 해상도 (x, y, z)
)
```

**P키로 프로브 격자 시각화:**

![프로브 시각화](https://raw.githubusercontent.com/kohhyunkyung/gi-escape-room/main/escape%20room/%ED%94%84%EB%A1%9C%EB%B8%8C.png)

#### 구면조화함수 (Spherical Harmonics)

각 프로브는 L1 SH로 방사 방향의 조명 정보를 저장한다. L1 SH는 **4개의 계수 × RGB = 12 floats**로 전방향 확산광을 압축 저장한다.

```javascript
// SH 투영: 방향벡터 + 색상 → SH 계수
proj(dir, color) {
  return [
    0.282095 * color.r,           // L0 (Y_0^0)
    0.488603 * dir.y * color.r,   // L1 (Y_1^{-1})
    0.488603 * dir.z * color.r,   // L1 (Y_1^0)
    0.488603 * dir.x * color.r,   // L1 (Y_1^1)
    // G, B 채널 반복...
  ];
}

// SH 복원: 노말 방향 → irradiance (단순 내적)
eval(sh, normal) {
  const c0 = 0.886227, c1 = 1.023328;
  return Math.max(0,
    c0*sh[0] + c1*(sh[1]*n.y + sh[2]*n.z + sh[3]*n.x)
  );
}
```

#### Spherical Fibonacci 샘플링

각 프로브는 매 프레임 **16개의 방향**으로 radiance를 캡처한다. Spherical Fibonacci 방법으로 방향을 균일하게 분포시킨다.

```javascript
const N = 16, gr = (1 + Math.sqrt(5)) / 2;  // 황금비
for (let s = 0; s < N; s++) {
  const theta = Math.acos(1 - 2*(s+0.5)/N);
  const phi   = 2 * Math.PI * s / gr;
  const dir   = new THREE.Vector3(
    Math.sin(theta)*Math.cos(phi),
    Math.cos(theta),
    Math.sin(theta)*Math.sin(phi)
  );
}
```

#### Temporal Blending (시간적 수렴)

매 프레임 새로 계산된 SH 값을 이전 프레임과 지수 이동 평균으로 블렌딩하여 간접광이 점진적으로 수렴한다.

```javascript
const alpha = 0.09;
for (let k = 0; k < 12; k++) {
  probe.sh[k] = probe.sh[k] * (1 - alpha) + newSH[k] * alpha;
}
```

우측 상단 패널에서 수렴 과정을 실시간으로 확인할 수 있다 (수렴 중 → 완료).

#### 8-probe Trilinear Interpolation

임의 위치에서의 irradiance는 주변 8개 프로브의 SH를 trilinear interpolation으로 합산하여 계산한다.

```javascript
sample(pos, normal) {
  // 격자 좌표 계산
  const lp = (pos - origin) / size * (resolution - 1);
  let result = new THREE.Color(0, 0, 0);
  // 8개 인접 프로브 보간
  for (dz of [0,1]) for (dy of [0,1]) for (dx of [0,1]) {
    const w = (dx?tx:1-tx) * (dy?ty:1-ty) * (dz?tz:1-tz);
    result += evalSH(probe[x0+dx][y0+dy][z0+dz], normal) * w;
  }
}
```

---

## 3. 게임 개발 상세 설명

### 3.1 방 1 — 연구실

**Cornell Box 스타일**의 실내 환경으로, 빨강/초록/파랑 세 가지 색상 벽이 간접광 색 번짐(Color Bleeding) 효과를 극대화하도록 설계하였다.

#### GI OFF vs GI ON 비교

| GI OFF | GI ON |
|---|---|
| ![GI OFF](https://raw.githubusercontent.com/kohhyunkyung/gi-escape-room/main/escape%20room/%EB%B0%A91%EC%97%B0%EA%B5%AC%EC%8B%A4%20GI%20%20OFF%20.png) | ![GI ON](https://raw.githubusercontent.com/kohhyunkyung/gi-escape-room/main/escape%20room/%EB%B0%A91%20%EC%97%B0%EA%B5%AC%EC%8B%A4%20GI%20ON.png) |

GI ON 상태에서 초록 벽의 빛이 바닥과 천장에 초록빛으로 번지고, 빨간 벽의 빛이 인접 면에 붉게 번지는 **Color Bleeding** 효과가 나타난다.

#### GI 퍼즐 — 빨간 벽의 숨겨진 숫자

금고 암호 826은 DDGI가 계산한 각 벽면의 간접광 반사값에서 유래한다:
- 빨간 벽 반사값: **8**
- 초록 벽 반사값: **2**
- 파랑 벽 반사값: **6**

GI를 활성화하면 빨간 벽에 숫자 **8, 2, 6**이 나타난다.

![방1 P키](https://raw.githubusercontent.com/kohhyunkyung/gi-escape-room/main/escape%20room/%EB%B0%A91(P%ED%82%A4).png)

#### 금고 시스템

826 입력 시 금고 문이 경첩을 기준으로 열리며 내부의 황금 열쇠가 드러난다.

![금고 열림](https://raw.githubusercontent.com/kohhyunkyung/gi-escape-room/main/escape%20room/%EB%B0%A91%20%EA%B8%88%EA%B3%A0%20%EC%97%B4%EB%A6%BC.png)

![열쇠 보이는 장면](https://raw.githubusercontent.com/kohhyunkyung/gi-escape-room/main/escape%20room/%EB%B0%A9%201%20%EC%97%B0%EA%B5%AC%EC%8B%A4%20%EC%97%B4%EC%87%A0%EB%B3%B4%EC%9D%B4%EB%8A%94%20%EC%9E%A5%EB%A9%B4.png)

---

### 3.2 방 2 — 지하실

**정전 상황**을 연출하여 GI OFF 상태에서는 방이 거의 완전히 어둡고, GI ON 시 간접광으로 공간을 파악할 수 있도록 설계하였다.

#### GI OFF vs GI ON 비교

| GI OFF | GI ON |
|---|---|
| ![GI OFF](https://raw.githubusercontent.com/kohhyunkyung/gi-escape-room/main/escape%20room/%EB%B0%A92%20%EC%A7%80%ED%95%98%EC%8B%A4%20GI%20OFF.png) | ![GI ON](https://raw.githubusercontent.com/kohhyunkyung/gi-escape-room/main/escape%20room/%EB%B0%A92%20%EC%A7%80%ED%95%98%EC%8B%A4%20GI%20ON.png) |

GI ON 시 DDGI 프로브의 간접광 계산으로 방 전체가 밝아지며, 뒷벽에 발전기 코드 **5272**가 파란 형광으로 나타난다.

#### 메모 힌트

![방2 메모](https://raw.githubusercontent.com/kohhyunkyung/gi-escape-room/main/escape%20room/%EB%B0%A92%20%EB%A9%94%EB%AA%A8.png)

#### 퍼즐 체인

```
메모 읽기 → 퓨즈 줍기 → 퓨즈박스 수리 → 발전기 코드 입력(5272) → 배터리 획득
```

| 발전기 가동 전 | 발전기 가동 후 |
|---|---|
| ![가동 전](https://raw.githubusercontent.com/kohhyunkyung/gi-escape-room/main/escape%20room/%EB%B0%A92%20%EB%B0%9C%EC%A0%84%EA%B8%B0%20%EA%B0%80%EB%8F%99%20%EC%A0%84.png) | ![가동 후](https://raw.githubusercontent.com/kohhyunkyung/gi-escape-room/main/escape%20room/%EB%B0%A92%20%EB%B0%9C%EC%A0%84%EA%B8%B0%20%EA%B0%80%EB%8F%99%20%ED%9B%84%20%EC%B4%88%EB%A1%9D%20%ED%91%9C%EC%8B%9C%EB%93%B1%20%EC%BC%9C%EC%A7%90.png) |

---

### 3.3 방 3 — 보안실

**SF 보안 구역** 분위기로, 4개의 숫자가 방 곳곳에 흩어져 있어 GI를 켜고 탐색하며 찾아야 한다.

#### GI OFF vs GI ON

| GI OFF | GI ON (4개 숫자 모두) |
|---|---|
| ![GI OFF](https://raw.githubusercontent.com/kohhyunkyung/gi-escape-room/main/escape%20room/%EB%B0%A93%20GI%20OFF.png) | ![4개 숫자](https://raw.githubusercontent.com/kohhyunkyung/gi-escape-room/main/escape%20room/%EB%B0%A93%204%EA%B0%9C%20%EC%88%AB%EC%9E%90%20%ED%95%9C%EB%B2%88%EC%97%90%20%EB%B3%B4%EC%9D%B4%EA%B8%B0.png) |

#### GI ON 시 각 위치별 숫자

| 위치 | 색상 | 숫자 | 캡처 |
|---|---|---|---|
| 뒷벽 | 🔴 빨강 | 4 | ![빨간4](https://raw.githubusercontent.com/kohhyunkyung/gi-escape-room/main/escape%20room/%EB%B0%A93%20GI%20ON%20%EB%92%B7%EB%B2%BD%20%EB%B9%A8%EA%B0%84%20%EC%88%AB%EC%9E%90%204.png) |
| 오른쪽 벽 | 🟢 초록 | 3 | ![초록3](https://raw.githubusercontent.com/kohhyunkyung/gi-escape-room/main/escape%20room/%EB%B0%A93%20GI%20ON%20%EC%98%A4%EB%A5%B8%EC%AA%BD%20%EB%B2%BD%20%EC%B4%88%EB%A1%9D%20%EC%88%AB%EC%9E%90%203.png) |
| 천장 | 🔵 파랑 | 2 | ![파란2](https://raw.githubusercontent.com/kohhyunkyung/gi-escape-room/main/escape%20room/%EB%B0%A9%203%20%EC%B2%9C%EC%9E%A5%20%ED%8C%8C%EB%9E%80%20%EC%88%AB%EC%9E%90%202.png) |
| 왼쪽 벽 | 🟡 노랑 | 1 | ![노란1](https://raw.githubusercontent.com/kohhyunkyung/gi-escape-room/main/escape%20room/%EB%B0%A93%20GI%20ON%20%EC%99%BC%EC%AA%BD%20%EB%B2%BD%20%EB%85%B8%EB%9E%80%EC%88%AB%EC%9E%901.png) |

보안 매뉴얼에 "빨강→초록→파랑→노랑 순서"라는 힌트가 있어 코드 **4321**을 도출한다.

![보안 매뉴얼](https://raw.githubusercontent.com/kohhyunkyung/gi-escape-room/main/escape%20room/%EB%B0%A93%20%EB%B3%B4%EC%95%88%EB%A9%94%EB%89%B4%EC%96%BC.png)

---

## 4. DDGI 기술 적용 요약

### 구현된 DDGI 파이프라인

```
1. 프로브 격자 초기화 (방마다 4×3×4 = 48개)
         ↓
2. 매 프레임: Spherical Fibonacci로 16방향 radiance 캡처
         ↓
3. SH 투영 (L1, 12 floats/probe)
         ↓
4. Temporal Blending으로 점진적 수렴
         ↓
5. 렌더링 시: 8-probe trilinear interpolation으로 irradiance 샘플링
         ↓
6. emissive로 간접광 적용 (Color Bleeding 표현)
```

### GI가 게임플레이에 미치는 영향

| 방 | GI OFF | GI ON | 게임플레이 역할 |
|---|---|---|---|
| 연구실 | 숫자 안 보임 | 826 등장 | 금고 암호 획득 |
| 지하실 | 거의 암흑 | 방 전체 밝아짐 + 5272 등장 | 탐색 가능 + 발전기 코드 획득 |
| 보안실 | 숫자 없음 | 4색 숫자 등장 | 보안 코드 4321 획득 |

본 게임에서 DDGI는 단순한 시각 효과가 아닌, **퍼즐의 핵심 메커닉**으로 활용되었다. 플레이어는 반드시 GI를 활성화해야만 게임을 진행할 수 있다.

---

## 5. 탈출 성공 화면

![탈출 성공](https://raw.githubusercontent.com/kohhyunkyung/gi-escape-room/main/escape%20room/%ED%83%88%EC%B6%9C%EC%84%B1%EA%B3%B5%20%EC%97%94%EB%94%A9%20%ED%99%94%EB%A9%B4.png)

---

## 6. 기술 스택

| 항목 | 내용 |
|---|---|
| 렌더링 엔진 | Three.js r128 |
| GI 방식 | DDGI (커스텀 구현) |
| 프로브 수 | 방당 48개 (4×3×4) |
| SH 차수 | L1 (4계수 × RGB = 12 floats) |
| 배포 | GitHub Pages |
| 파일 구성 | 단일 HTML 파일 |

---

*게임 링크: https://kohhyunkyung.github.io/gi-escape-room/*
