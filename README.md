# MRI 추간공 협착증(Foraminal Stenosis) 객체 탐지 프로젝트

`04_Foraminal_Stenosis_Data` 데이터셋을 활용해, MRI 영상에서 추간공(Foramen) 영역을 탐지하고
협착 등급(Grade)을 함께 식별하는 **YOLOv8 기반 객체 탐지(Object Detection)** 프로젝트입니다.

## 프로젝트 개요

- **목표**: MRI 축상/시상 슬라이스 영상에서 추간공 위치를 Bounding Box로 검출하고,
  좌/우 측면(Side) 및 협착 등급(Grade 0~3)을 분류
- **데이터 형식**: 환자별 폴더 구조 (`case_id` 단위) 내에 MRI 슬라이스(PNG) + Pascal VOC XML 주석
- **핵심 파이프라인**: XML 주석 파싱 → 데이터 품질 점검 → Pascal VOC → YOLO 포맷 변환 → YOLOv8 학습/평가

## 노트북 구성

| 노트북 | 주요 내용 |
|---|---|
| `mri_객체인식_1_.ipynb` | 객체 탐지 이론(Bounding Box 표현법, IoU, 2-Stage/1-Stage 모형), XML 주석 파싱 → 정형데이터 변환, 데이터 품질 점검(누락 이미지, 이미지 크기, Small Object 비율) |
| `mri_객체인식_2_.ipynb` | 위 내용에 이어 박스 시각화, IoU/NMS 직접 구현, Pascal VOC → YOLO 형식 변환, YOLOv8 모형 학습 및 평가까지 전체 파이프라인 수행 |

## 데이터 처리 과정

1. **XML 파싱**: `case_id`(환자), `slice_id`(슬라이스), `side`(L/R), `grade`(협착 등급), `level`(추간공 위치), Bounding Box 좌표를 추출해 하나의 데이터프레임(`df_box`)으로 평탄화 → `04_xml_bndbox.csv`로 저장
2. **데이터 품질 점검**
   - 주석은 있으나 이미지 파일이 없는 케이스 확인
   - 이미지 크기(width/height) 일관성 확인
   - 박스 면적 비율 확인 → 평균 약 0.78%로, **Small Object Detection**에 해당하는 어려운 과제임을 확인
3. **박스 시각화**: 등급별 색상(`GRADE_COLOR`)으로 원본 MRI 위에 Bounding Box 오버레이
4. **평가 지표 직접 구현**: IoU(Intersection over Union), NMS(Non-Maximum Suppression)를 직접 함수로 구현하여 원리 학습
5. **YOLO 포맷 변환**
   - 클래스 정의는 `CLASS_MODE`로 선택 가능: `single`(단일 클래스) / `side`(L·R) / `grade`(FS0~FS3) / `full`(측면+등급 조합)
   - 좌표를 중심점·정규화 방식(cx, cy, w, h)으로 변환, 범위(0~1) 초과 여부 검증
   - 환자(`case_id`) 단위로 train/val 분리(80:20)하여 **데이터 누수 방지**
   - `data.yaml` 생성 후 YOLO 학습용 디렉토리 구조(`images/`, `labels/`) 구성
6. **변환 결과 역검증**: 생성된 YOLO 라벨(txt)을 다시 픽셀 좌표로 복원해 원본과 비교 시각화

## 모델 학습 및 평가

- **모델**: `YOLOv8n` (Ultralytics)
- **학습 설정**: `epochs=5`, `imgsz=512`, `batch=8`
- **평가 지표**: Precision, Recall, mAP50, mAP75, mAP50-95
- IoU 임계값(0.5~0.95)별 AP 변화 곡선 시각화
- 검증 이미지에 대한 예측 결과 시각화

## 사용 라이브러리

- `ultralytics` (YOLOv8)
- `tensorflow`, `keras`
- `numpy`, `pandas`
- `matplotlib`, `plotly`
- `PIL` (이미지 처리)
- `sklearn` (train_test_split)
- `xml.etree.ElementTree` (Pascal VOC XML 파싱)
- `glob`, `os`, `shutil` (파일/디렉토리 처리)

## 폴더 구조 (예시)

```
data/04_Foraminal_Stenosis_Data/
├── 0001/                     # 환자별 폴더
│   ├── IM000003.png
│   └── IM000003.xml
├── ...
├── 04_xml_bndbox.csv          # 파싱된 주석 정형데이터
└── yolo_foramina_<CLASS_MODE>/
    ├── images/{train,val}
    ├── labels/{train,val}
    ├── data.yaml
    └── yolo_runs/             # 학습/평가 결과
```

## 참고 사항

- Small Object Detection 특성상 IoU 지표가 위치 오차에 매우 민감하므로, 박스 크기 및 정규화 좌표 검증 단계가 중요합니다.
- `CLASS_MODE`를 변경하면 동일한 파이프라인으로 단일/측면/등급/전체 조합 등 다양한 분류 세분화 실험이 가능합니다.
