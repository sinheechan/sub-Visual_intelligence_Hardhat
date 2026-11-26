# Visual_intelligence_Hardhat

YOLOv8 기반 CCTV 영상 객체 인식(사람 · 헬멧 · 머리) 및 Flask 분석 서버 구현

<br/>

<img src="img/hardhat_img.png">

테스트 영상(`fallen.mp4`) 프레임

<br/><br/>

## Object

<br/>

이 프로젝트는 건설 현장 CCTV 영상에서 **사람과 안전모(헬멧) 착용 여부**를 실시간으로 인식하는 것을 목표로 합니다.

영상을 수집하는 클라이언트와 분석을 수행하는 서버를 분리하고, 클라이언트가 1초마다 프레임을 서버로 보내면 서버가 YOLOv8로 객체를 검출해 결과(JSON)를 돌려주는 구조로 구현했습니다.

검출 결과는 이후 관제 화면(프론트엔드)에서 경고 신호로 활용할 수 있도록 Base64 이미지와 메타정보 형태로 전송하는 로직까지 설계했습니다.

<br/><br/>

## Content

<br/>

1. Module 소개

2. 프로세스 구성

3. 실행 방법

4. Version

5. Feedback

<br/><br/>

## 1. Module 소개

<br/>

- `predict_api.py` : 분석 서버를 구동하는 메인 모듈입니다. (Flask)

  ① `/predict` (GET · POST) : 업로드된 프레임을 `data/raw/` 에 저장한 뒤 YOLOv8로 검출

  ② `save_txt=T` 이면 검출 결과를 JSON(클래스 · 신뢰도 · 박스 좌표)으로, 아니면 박스가 그려진 이미지를 스트리밍으로 반환

  ③ 신뢰도(conf) · IoU · 이미지 크기 · 장치(device) 등 추론 옵션을 실행 인자로 조정

- `my_client.py` : 영상 수집 클라이언트입니다. 영상을 프레임 단위로 읽어 1초마다 서버에 분석을 요청하고 결과를 정리합니다.

- `general.py` : GET / POST 요청에서 분석 대상(source) · 결과 형식(save_txt) 인자를 꺼내는 공통 함수입니다.

- `send_mod.py` : 관제 시스템 전송용 메타정보(채널 · 요청 ID · 요청 시각)와 이미지 Base64 인코딩 함수입니다.

- `watcher.py` : 감시 폴더(`runs/detect`)에 결과 파일이 쌓이면 자동으로 정리하는 모듈입니다. (watchdog)

- `hardhat.pt` : 안전모 검출용 YOLOv8 학습 가중치 (클래스 : `head`, `helmet`, `person`)

<br/><br/>

## 2. 프로세스 구성

<br/>

```
[영상 수집 · my_client.py]  ──(1초마다 프레임 POST)──▶  [분석 서버 · predict_api.py]
                                                         │  YOLOv8 (hardhat.pt)
        ◀──────────(JSON : 클래스 · 신뢰도 · 박스)─────────┘
        │
        └─▶ [시각화 전송] 메타정보 + Base64 이미지 → 관제 화면
```

<br/>

### ① 영상 수집

- 클라이언트는 OpenCV로 영상을 읽고, 프레임을 임시 이미지로 저장한 뒤 1초 간격으로 서버에 전송합니다.

### ② 객체 검출

- 서버는 받은 프레임을 저장하고 YOLOv8로 사람 · 헬멧 · 머리(헬멧 미착용)를 검출합니다.
- 검출 결과가 없는 프레임은 예외 처리하여 다음 프레임으로 넘어갑니다.

### ③ 결과 전송

- 첫 번째 검출 객체의 클래스명 · 신뢰도 · 박스 좌표를 JSON으로 정리합니다.
- 관제 화면 연동을 위해 채널 ID · 요청 시각 · Base64 이미지를 묶어 보내는 로직을 작성했습니다. (현재 코드에서는 주석 처리)

### ④ 결과 정리

- `watcher.py` 가 결과 폴더를 감시하며, 실시간 분석으로 쌓이는 결과 파일을 자동 삭제해 디스크 사용량을 관리합니다.

<br/><br/>

## 3. 실행 방법

<br/>

```bash
# 1. 분석 서버 실행 (http://127.0.0.1:5000)
python predict_api.py --weights hardhat.pt

# 2. 다른 터미널에서 클라이언트 실행 (fallen.mp4 를 1초 단위로 분석 요청)
python my_client.py

# 결과 이미지를 서버에서 바로 확인할 때
python predict_api.py --weights hardhat.pt --show
```

<br/><br/>

## 4. Version

<br/>

- python 3.10
- Flask 3.0.0
- numpy 1.26.4
- pandas 2.2.2
- ultralytics 8.2.0
- opencv-python, requests, watchdog

<br/><br/>

## 5. Feedback

<br/>

- Base64 문자열로 이미지를 전송하면 원본보다 용량이 커지므로, 실제 서비스에서는 검출 시점의 프레임만 전송하는 방식이 필요합니다.
- 현재는 첫 번째 검출 객체만 결과로 정리하므로, 한 화면에 여러 작업자가 있을 때 모든 객체를 판단하도록 확장이 필요합니다.
- `watcher.py` 의 감시 경로가 로컬 절대경로로 지정되어 있어, 환경에 맞게 수정 후 사용해야 합니다.
