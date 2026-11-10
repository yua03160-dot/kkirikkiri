# 끼리끼리 — 공용 분리배출 안내 시스템

> 분리배출할 때 "이거 어디 버려?" 하고 멈칫하는 순간,
> 사진 한 장으로 품목을 찾고 배출 방법까지 알려줍니다.

| 항목 | 내용 |
| --- | --- |
| 과정 | 스마트인재개발원 피지컬 AI 과정 · 미니 프로젝트(해커톤) · A4팀 |
| 발표 | 2026.09.01 |
| 대상 품목 | 건전지 · 스티로폼 · 깨진 유리 · 전선 · 우산 (5종) |
| 모델 | YOLO26 nano (사전학습 모델 전이학습) |
| 시연 | OpenCV 웹캠 화면 · Gradio 웹 화면 |

---

## 1. 왜 만들었나

분리배출 규칙을 몰라서가 아니라, **눈앞의 물건이 어떤 품목인지 판단하기 어려운 경우**가 많습니다. 검색은 물건 이름을 알아야 쓸 수 있습니다. 끼리끼리는 이름을 몰라도 카메라에 비추면 품목을 먼저 찾고, 배출 안내로 이어집니다.

5개 품목은 일상에서 헷갈리기 쉽고, 잘못 버리면 안전 문제가 생기기 쉬운 것으로 골랐습니다.

## 2. 결과 (test 654장, 학습에 쓰지 않은 사진)

| Precision | Recall | mAP50 |
| --- | --- | --- |
| 0.899 | 0.748 | 0.849 |

- 모델이 찾았다고 한 결과는 대체로 맞았지만, 물건을 놓치는 경우가 남아 있습니다.
- 품목별로는 스티로폼(Recall 0.578)과 전선(0.625)을 자주 놓쳤습니다.
- 투명하거나 반사가 강한 깨진 유리는 실제 사진에서 인식하지 못한 경우가 있었습니다.

## 3. 전체 흐름

```
1. ZIP 정리          extract_source_archives.py   공개 데이터 ZIP → data/raw
2. 전처리            prepare_yolo_dataset.py      라벨 5종 통일 · 중복 제거 · 분할 → data/processed
3. 학습              train.py                     YOLO26 nano 학습 → best.pt, test 평가
4. 웹캠 시연         webcam_guide.py              노트북 카메라로 인식 + 안내 문구
5. 웹 화면           gradio_guide.py              사진 업로드 · 성능 그래프 · 혼동 행렬
6. 구조 검사         test_project_structure.py    필수 파일 누락 확인
```

## 4. 데이터

| 단계 | 내용 |
| --- | --- |
| 수집 | Kaggle, Roboflow 공개 데이터 6,322장 |
| 정제 | 대상 품목이 없는 사진 42장, 중복 4장 제외 → **6,276장** |
| 라벨 통일 | 출처마다 다른 품목 이름을 `battery`, `styrofoam`, `broken_glass`, `cable`, `umbrella` 5개로 변환 (`config/source_label_maps.json`) |
| 형식 변환 | Segmentation 라벨 → Bounding Box |
| 분할 | 사진 해시값 기준 train 4,415 / val 1,207 / test 654 (약 70 : 20 : 10) |
| 기록 | 품목별 개수와 중복 제거 결과를 `prepare_report.json`에 저장 |

**왜 해시로 나눴나?** 사진 내용으로 만든 고유 번호(해시)로 그룹을 정하면, 같은 사진이 학습용과 시험용에 동시에 들어가지 않습니다. 모델이 이미 본 사진을 시험에서 다시 맞혀 성능이 부풀려지는 것을 막기 위해서입니다.

품목별 객체 수: 건전지 6,198 · 스티로폼 3,219 · 깨진 유리 1,800 · 우산 1,746 · 전선 634

## 5. 학습 설정

| 항목 | 값 |
| --- | --- |
| 모델 | `yolo26n.pt` (사전학습 가중치) |
| 이미지 크기 | 640 × 640 |
| 에포크 | 40 |
| 배치 | 8 (발표 기준) |
| 환경 | RTX 3070 Laptop GPU · PyTorch · CUDA |

`train.py`는 `prepare_report.json`이 없으면 학습을 시작하지 않습니다. 전처리를 건너뛰고 학습하는 실수를 막기 위해서입니다.

## 6. 실행 방법

```bash
# 1. 공개 데이터 ZIP 정리
python src/extract_source_archives.py --downloads-dir <ZIP 폴더>

# 2. 전처리 (라벨 통일 · 중복 제거 · 분할)
python src/prepare_yolo_dataset.py

# 3. 학습 (기본 40 에포크, 640px)
python src/train.py

# 4. 웹캠 시연 (Q 키로 종료)
python src/webcam_guide.py

# 5. Gradio 웹 화면 → http://127.0.0.1:7860
python src/gradio_guide.py

# 6. 필수 파일 검사
pytest tests/test_project_structure.py
```

## 7. 화면 구성 (Gradio)

| 탭 | 내용 |
| --- | --- |
| 품목 인식과 안내 | 사진 업로드 또는 웹캠 촬영 → 탐지 상자, 인식 확률, 배출 안내. '인식 기준 점수' 슬라이더(기본 0.45)로 약한 예측을 거름 |
| 모델 성능과 검증 | 품목별 Precision · Recall · mAP50 그래프, 혼동 행렬 |
| 발표에서 설명할 흐름 | 데이터 통일 → 탐지 → 안내 → 성능 확인 순서 |

인식 기준 점수는 시연 화면에서 낮은 신뢰도 결과를 걸러내기 위한 값이며, 안전을 보장하는 기준은 아닙니다.

## 8. 한계와 다음 단계

- 공개 데이터와 실제 촬영 환경의 차이 → 다양한 각도·조명·배경으로 직접 촬영한 데이터 추가
- 스티로폼·전선·투명 유리 데이터 보강 후 재학습·재검증
- 지역마다 다른 배출 규칙 → 지역별 안내 데이터를 따로 관리하도록 확장

## 9. 폴더 구조

```
├── config/
│   ├── kkirikkiri.yaml
│   ├── source_label_maps.json
│   └── visualization_metrics.json
├── data/
│   ├── raw/          # 출처별 원본
│   └── processed/    # 학습용 (images, labels / train, val, test)
├── runs/             # 학습 결과, best.pt
├── src/
│   ├── extract_source_archives.py
│   ├── prepare_yolo_dataset.py
│   ├── train.py
│   ├── webcam_guide.py
│   └── gradio_guide.py
└── tests/
    └── test_project_structure.py
```

## 10. 팀

A4팀 · 팀장 김덕중 · [윤승기,정승민,장승식,조준혁]
