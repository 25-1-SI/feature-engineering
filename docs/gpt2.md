# 지반 침하 예측 모델링 프로젝트

이 문서는 주어진 데이터를 활용하여 지반 침하 고위험 지역을 예측하는 모델링 프로젝트의 핵심적인 접근 방식을 요약합니다.

---

## 1. 데이터 목록 및 좌표 유무

| 파일명 | 좌표 칼럼 | 시간 칼럼 | 용도 요약 |
|---|---|---|---|
| `지반침하_고위험지역_주소.csv` | `lat1`, `lon1`, `lat2`, `lon2` (일부 null) | 없음 (사고 날짜 없음) | 라벨 생성 + "과거 사고" 피처 |
| `subway_depth.csv` | `위도`, `경도` | 없음 | 역 메타 (깊이, 형식) |
| `road_construction_processed.csv` | 없음 | `착공일`, `준공일` | 도로 굴착 이벤트 (행정구/동 단위) |
| `construction_processed.csv` | `위치좌표(위도/경도)` | `사업시작일`, `사업종료일` | 대규모 건설 프로젝트 |
| `construction_underground_water_processed.csv` | `X좌표`, `Y좌표` (TM) | `등록년도` | 지하수 관련 공사 |
| `building_underground_water_processed.csv` | 없음 | `조사년도` | 건축물 지하수 배출량 |
| `sinkhole_map.html` | — | — | 지도 스팟 육안 검수용 |

---

## 2. 전처리 핵심 요약

### 1. 인코딩 자동 감지
다양한 인코딩을 처리하기 위해 `read_kor_csv` 함수를 사용하여 `utf-8-sig`, `cp949`, `euc-kr` 순으로 시도합니다.

```python
def read_kor_csv(path):
    for enc in ['utf-8-sig','cp949','euc-kr']:
        try: return pd.read_csv(path, encoding=enc)
        except UnicodeDecodeError: pass
    raise ValueError("인코딩 실패")
```

### 2. 좌표 통일
* **WGS84**: `지반침하_고위험지역_주소.csv`, `subway_depth.csv`, `construction_processed.csv`는 이미 WGS84 좌표계입니다.
* **TM → WGS84**: `construction_underground_water_processed.csv`의 TM 좌표(`EPSG:5179`)는 `pyproj.Transformer.from_crs("EPSG:5179","EPSG:4326")`를 사용하여 WGS84(`EPSG:4326`)로 변환합니다.

### 3. 날짜 파싱
`착공일`, `준공일`, `사업시작일`, `사업종료일`과 같은 날짜 칼럼은 `pd.to_datetime`을 사용하여 datetime 객체로 변환합니다. `errors='coerce'` 옵션으로 파싱 오류 시 `NaT`로 처리합니다.

```python
for c in ['착공일','준공일','사업시작일','사업종료일']:  
    df[c] = pd.to_datetime(df[c], errors='coerce')
```

### 4. 숫자 문자열 처리
숫자 문자열 내의 쉼표 제거 및 `연장(m)`와 같은 단위 문자열을 처리하여 숫자형으로 변환합니다.

---

## 3. 타깃(라벨) 정의

### 방법 A (정석)
`sinkhole.csv`의 사고 지점과 지하철 역 150m 버퍼를 공간 조인합니다. 각 역별로 지난 12개월간의 사고 건수(`acc_cnt_last_12m`)를 집계하여, 사고 건수가 0이면 0, 1 이상이면 1로 이진 분류 라벨을 생성합니다.

### 방법 B (더 세분화)
역-월 단위로 끊어 `acc_cnt`를 회귀 목표로 사용할 수도 있습니다.

**주의**: 사고 데이터에 날짜 칼럼이 없으므로 '최근 n 개월' 기준을 직접 적용할 수 없습니다. 이를 해결하기 위해 `sinkhole.csv`를 50% 랜덤 분할하여 "과거 vs 미래" 시퀀스를 만들어 라벨과 피처로 재활용하는 꼼수를 고려할 수 있습니다.

---

## 4. 피처 설계

각 데이터 파일별로 생성 가능한 피처는 다음과 같습니다.

### 4-1. `subway_depth.csv` (역 메타)

| 피처 | 타입 | 설명 |
|---|---|---|
| `정거장깊이`, `선로기준정거장깊이` | 수치 | 깊이가 깊을수록 굴착 난이도 상승 및 지하 안정성 저하 가능성 |
| `지하층수` | 수치 | `층수(지상/지하)` 칼럼을 '/'로 분리 후 뒷부분 값 |
| `형식(섬식/상대식)` | 원-핫 인코딩 | 터널 형태는 지반 절개 방식과 관련 |
| `환승역_dummy` | bool | `비고` 칼럼의 `isna()` 여부로 간략하게 추출 |

### 4-2. `지반침하_고위험지역_주소.csv` (사고 이력)

1.  **좌표 보강**: `lat1`, `lon1`이 `null`인 경우 `lat2`, `lon2`로 보강합니다.
    ```python
    df['lat'] = df['lat1'].fillna(df['lat2'])
    df['lon'] = df['lon1'].fillna(df['lon2'])
    ```
2.  **역 거리 기반 피처 (150m 원형 버퍼 예)**:
    | 피처 | 설명 |
    |---|---|
    | `acc_cnt_total` | 전체 사고 건수 |
    | `acc_cnt_near` | 0 - 75m 사이 사고 건수 (가중치 부여) |
    | `acc_dist_min` | 사고 최단 거리 (m) |

### 4-3. `construction_processed.csv` (대규모 프로젝트)

| 피처 | 윈도 | 집계 식 |
|---|---|---|
| `big_const_cnt_ongoing` | 현재 ∈ [사업시작일, 사업종료일] | 건수 |
| `big_const_amt_sum` | 동일 | 사업금액 합 (결측은 0) |
| `big_const_amt_log` | 동일 | `log1p(합)` |

거대 프로젝트의 경우, 단일 사고보다 영향력이 길게 지속될 수 있으므로 `(남은기간/총기간)`과 같은 기간 가중치를 값에 곱하는 것을 고려할 수 있습니다.

### 4-4. `construction_underground_water_processed.csv`

1.  좌표 변환(`TM` → `WGS84`) 후 역 버퍼에 합류합니다.
2.  **피처**: `uw_const_cnt` (총 건수), `uw_const_recent` (최근 3년 내 비중).

### 4-5. `road_construction_processed.csv` (좌표 없음)

좌표가 없으므로 구/동 단위로만 접근합니다.

| 피처 | 구현 |
|---|---|
| `same_gu_cnt` | 역의 자치구와 같은 굴착 공사 중 (착공일 ≤ today ≤ 준공일)인 건수. `groupby(['구']).size()` 후 매핑 |
| `same_dong_cnt` | 역의 동과 같은 굴착 공사 중인 건수. `groupby(['동']).size()` 후 매핑 |
| `공사일수_평균` | 공사 기간이 길수록 지반 장기 교란 가능성. `mean()` |

**Devil's advocate**: 행정구 단위는 공간 해상도가 낮아 노이즈가 클 수 있습니다. 좌표를 얻어야 진정한 거리 기반 피처를 활용할 수 있습니다.

### 4-6. `building_underground_water_processed.csv`

1.  **동 추출**: '건축물 위치' 칼럼에서 정규표현식을 사용하여 동을 추출합니다. (`.str.extract(r'(\w+동)')`)
2.  **집계**: 역-동 매핑 후 `bld_uw_sum`, `bld_uw_max`, `bld_uw_cnt` 피처를 생성합니다.
    * "일평균 톤/일"은 log 변환 후 표준화를 추천합니다.

---

## 5. 파이프라인 스케치

주어진 데이터만 활용한 파이프라인 스케치는 다음과 같습니다.

```python
import geopandas as gpd, pandas as pd, numpy as np
from shapely.geometry import Point
from pyproj import Transformer

EPSG_TM, EPSG_WGS = 5179, 4326
tf = Transformer.from_crs(EPSG_TM, EPSG_WGS, always_xy=True)

# ① 사고 GDF 생성
acc = read_kor_csv('...고위험지역...csv')
acc['lat'] = acc['lat1'].fillna(acc['lat2'])
acc['lon'] = acc['lon1'].fillna(acc['lon2'])
g_acc = gpd.GeoDataFrame(acc.dropna(subset=['lat','lon']),
                         geometry=gpd.points_from_xy(acc.lon, acc.lat),
                         crs=f"EPSG:{EPSG_WGS}")

# ② 역 GDF 생성
stn = read_kor_csv('subway_depth.csv')
g_stn = gpd.GeoDataFrame(stn, geometry=gpd.points_from_xy(stn.경도, stn.위도),
                         crs=f"EPSG:{EPSG_WGS}")

# ③ 역 버퍼 150m 생성
g_stn['buf'] = g_stn.geometry.buffer(0.15)  # degree ≈ 150 m; 정확히 하려면 to_crs 5186 사용

# ④ 사고 count 조인
join = gpd.sjoin(g_stn.set_geometry('buf'), g_acc, predicate='contains')
sink_cnt = join.groupby('역명').size().reindex(g_stn.역명).fillna(0)

# ⑤ 대규모 공사 합
big = read_kor_csv('construction_processed.csv').dropna(subset=['위치좌표(위도)'])
g_big = gpd.GeoDataFrame(big,
                         geometry=gpd.points_from_xy(big['위치좌표(경도)'], big['위치좌표(위도)']),
                         crs=f"EPSG:{EPSG_WGS}")
g_big_on = g_big[g_big.사업시작일 <= pd.Timestamp.today()]
g_big_on = g_big_on[g_big_on.사업종료일 >= pd.Timestamp.today()]
big_join = gpd.sjoin(g_stn.set_geometry('buf'), g_big_on, predicate='contains')
big_sum = big_join.groupby('역명')['사업금액(억원)'].sum().fillna(0)

# ... 같은 패턴으로 ⑤, ⑥ 데이터도 처리 ...
dataset = g_stn[['역명','정거장깊이']].assign(
    acc_cnt_total=sink_cnt,
    big_const_amt=big_sum,
    # etc.
)
```

---

## 6. 모델 및 평가 (TL;DR)

### 알고리즘
* **LightGBM / CatBoost**: 범주형과 수치형 피처가 혼합된 데이터셋에 빠르고 효율적이며, 결측치 자동 처리 기능이 있습니다.
* **L1-Logistic Regression**: 베이스라인 모델로 활용하며, 해석성이 좋고 피처 선택에 유용합니다.

### 교차 검증
* **XGBoost time-split CV**: 사고 빈도가 적어 일반적인 `stratifiedkfold`가 적합하지 않을 수 있으므로, "구 단위 블록 CV"를 권장합니다.

### 메트릭
* **PR-AUC (Precision-Recall AUC)** 또는 **F1@Top-k 역 (예: k=10개 역)**: 불균형한 데이터셋에서 현실적인 평가를 위해 적합합니다.

---

## 7. 한계점

현재 데이터만으로는 다음과 같은 한계가 있습니다.

* **좌표가 없는 주소 피처의 해상도 한계**: `도로 굴착` 및 `건물 지하수` 데이터의 주소 피처는 좌표가 없어 행정구 단위 수준의 낮은 공간 해상도를 가집니다. 이는 "거리가 가까울수록 위험"이라는 직관을 살리지 못하게 합니다.
    * **개선 방안**: 가능한 한 빨리 **지오코딩**을 수행하여 좌표를 얻는 것이 모델 성능과 해석력 모두를 급상승시키는 데 중요합니다.
* **사고 CSV의 날짜 정보 부재**: `지반침하_고위험지역_주소.csv`에 사고 날짜 칼럼이 없어 시간적 인과 관계를 파악하기 어렵습니다. 모델이 단순히 "사고가 많았던 역에서 또 사고가 난다"는 패턴만 학습할 위험이 있습니다.
* **좌표계 변환 검수 필요**: TM에서 WGS84로 좌표계 변환 시 한 줄의 실수로 남북이 뒤집히는 등 괴이한 현상이 발생할 수 있습니다. **꼭 샘플 데이터를 Folium 맵으로 시각화하여 육안 검수**하는 것이 중요합니다.

---

## 한 줄 요약

"좌표가 있는 데이터는 모두 버퍼-조인을 활용하고, 좌표가 없는 데이터는 행정구 단위로 그룹바이하여 피처를 생성합니다."

현재 가진 자료만으로는 이 방법이 최대치입니다. 추후 좌표를 얻을 때마다 해당 피처를 버퍼-조인 방식으로 통합하면 모델 성능이 단계적으로 향상될 것입니다.