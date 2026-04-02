# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

고랭이 배추 도매가격 예측 모델 — 강원 고랭지(강릉·대관령) 배추의 수확기(7~10월) 도매 가격 변동률을 4개 등급으로 분류 예측하는 머신러닝 프로젝트.

## Environment Setup

```bash
python -m venv .venv
source .venv/bin/activate
pip install -e .          # pyproject.toml 기반 의존성 설치
```

`pyproject.toml`에 정의된 주요 의존성: pandas, numpy, scikit-learn, openpyxl, lxml, seaborn, python-dotenv, ipykernel

## Running Notebooks

```bash
jupyter lab                      # 노트북 실행 환경
jupyter nbconvert --to notebook --execute notebooks/01_load_asos.ipynb  # 비대화형 실행
```

## Data Pipeline (notebooks/ 실행 순서)

각 노트북은 독립 실행 가능하나 아래 순서로 처리해야 함:

```
01_load_asos.ipynb     → raw_data/asos/     → processed/asos_{region}_2002_2025.csv
02_load_price.ipynb    → raw_data/.../xlsx  → processed/price_배추_가락시장_2002_2025.csv
02_load_spi.ipynb      → raw_data/spi/      → processed/spi_{region}_2002_2025.csv
          ↓
03_build_features.ipynb  → processed/dataset_{region}.csv  (2,337행 × 45열)
          ↓
04_train_model.ipynb     → 베이스라인 모델 (42개 피처 전체 사용)
05_feature_selection.ipynb → 최적 피처 수(N=5~30) 탐색 후 최종 모델 학습
```

## Architecture

### 핵심 설계 개념

**지역:** 강릉, 대관령 (지역별 독립 모델)

**입력 피처 (42개):** ASOS 기상변수 10개 + SPI 가뭄지수 4개 = 14개를 3개 생육기간 윈도우 평균값으로 집계
- 파종기: 수확일 -118 ~ -105일 (14일)
- 정식기: 수확일 -104 ~ -43일 (62일)
- 생육기: 수확일 -42 ~ -21일 (22일)

**타깃 (4클래스):** 전년 동일일 대비 가격변동률을 2007-2018 훈련 데이터 분위수로 구간화
- 0 (큰 하락) / 1 (하락) / 2 (상승) / 3 (큰 상승)
- 분위수 기준: Q1=-0.305, Q2=0.015, Q3=0.537

**학습/평가 분리:**
- Train: 2007–2018
- Test: 2019–2025

**모델:** SVM(RBF, MinMaxScaler 필요), Random Forest (GridSearchCV), Gradient Boosting (GridSearchCV)

### 데이터 전처리 주의사항
- ASOS CSV: cp949 인코딩, 연도별 컬럼명 불일치("합계 일사" vs "합계 일사량(MJ/m2)") 처리 필요
- 가격 데이터: 시장 휴장일(평균가격=0)을 NaN으로 변환 후 ffill
- SPI: 2005(강릉)/2007(대관령)년 이전 데이터 없음 → 보간으로 채움
- 가격 데이터가 null인 날짜는 학습 샘플에서 제외

### 파일 경로 (src/config.py 에서 관리 예정)
- 원시 데이터: `raw_data/`
- 전처리 결과: `processed/`
- 환경변수: `.env` (python-dotenv로 로드)
