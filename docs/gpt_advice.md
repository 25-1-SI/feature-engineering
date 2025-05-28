# 서울시 싱크홀 고위험지역 예측  
### ― 피처 엔지니어링 설계 개요

---

## 1. 데이터 스냅샷 ⸺ 헤더·주요 컬럼
| 파일명 | 공간/주소 컬럼 | 시계열 컬럼 | 값 예시 |
|-------|----------------|------------|---------|
| `road_construction_processed.csv` | `구·동`, `주소` | **착공일·준공일**, `공사일수`, `도로`, `포장` | “성동구 행당동…”, 차도/보도 |
| `construction_processed.csv` | `위도`, `경도` | **사업착수일·종료일**, `사업금액(억원)` | 37.5667, 126.9786 |
| `construction_underground_water_processed.csv` | `X좌표`, `Y좌표` (TM중부, m) | **등록년도** | 203 831 / 449 269 |
| `building_underground_water_processed.csv` | `주소` | **총·일평균발생량(톤)** | 21 481 t / 118 t·day |
| `subway_depth.csv` | `위도`, `경도` | `정거장깊이(m)` | 37.5531, 126.9725 |
| `지반침하_고위험지역_주소.csv` (Target) | `lat1,lon1`, `lat2,lon2` | 선정사유 | “지반침하사고 발생 빈도 높음” |

> **좌표계 통일 필수** ― TM중부(5186) ↔︎ WGS84 변환, 주소-lat/lon 정규화

---

## 2. 공간 피처 후보
| 아이디어 | 구현 방식 |
|----------|-----------|
| **도로굴착 밀도** | `착공일 ≥ t−1y` 필터 → 반경 _r_ m 버퍼 내 건수·평균 `공사일수` |
| **대형 건설 인접도** | 반경 _r_ m 존재 여부(0/1), 총 `사업금액`, 평균 기간 |
| **지하수 대량 배출 영향** | 반경 _r_ m `일평균발생량` 합, 상위 _k_건물까지 거리 |
| **지하 구조물 복합 위험** | (지하철 깊이 > 15 m) ∧ (굴착 진행 중) 교집합 여부 |
| **행정구역/토지용도 더미** | `시군구`, `동` one-hot + 토지용도 레이어 spatial join |

> 💡 **Devil’s advocate** : 반경 _r_ 하나 고정 → 과적합 우려. (_50 / 100 / 300 m_) 다중 스케일 생성 후 SHAP 검증.

---

## 3. 시간·빈도 피처
| 아이디어 | 포인트 |
|----------|--------|
| **공사-Sinkhole 지연시간** | 최근 **완료** 공사의 Δt(일) 최소·평균·가중(1/Δt) |
| **누적 굴착일수 지수화** | `공사일수 × exp(−λ Δt)` 가중 합 |
| **강우·지하수 시계열** | AWS 강우 7d·30d 누적, 배출량 이동평균 |

---

## 4. 상호작용·비선형 피처
* `굴착밀도 × 배출수합`
* `도로분류 × 지하철 깊이`
* `포장(차도/보도) × 토지용도`

---

## 5. 파이프라인 예시 (GeoPandas)

```python
import geopandas as gpd
from shapely.geometry import Point

# 1) 좌표계 통일
road = gpd.read_file('road_construction_processed.csv', encoding='cp949')
road = road.dropna(subset=['경도', '위도']).set_geometry(
        gpd.points_from_xy(road['경도'], road['위도']), crs='EPSG:4326'
).to_crs(5186)  # TM중부

# 2) 반경 100 m 버퍼 밀도
road['buffer'] = road.geometry.buffer(100)
buffers = road.set_geometry('buffer')
road_density = gpd.overlay(target_gdf, buffers, how='intersection') \
                  .groupby('target_id').size()