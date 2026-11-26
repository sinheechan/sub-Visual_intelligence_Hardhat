# Visual_intelligence_Hardhat

YOLOv8 기반 CCTV 사람 · 안전모 인식 분석 서버

<br/>

<img src="img/hardhat_result.jpg">

<br/><br/>

## Object

<br/>

- 건설 현장 CCTV 영상에서 사람과 안전모 착용 여부 인식
- 안전지대 진입 시 헬멧 착용 여부를 1초 단위로 검출
- 화면 표시는 프론트엔드 역할, 본 프로젝트는 백엔드 분석
- 클라이언트(영상 수집)와 서버(분석) 분리 구조

<br/><br/>

## Content

<br/>

1. Module 소개

2. 진행 과정

3. Result

4. 실행 방법

5. Version

6. Feedback

<br/><br/>

## 1. Module 소개

<br/>

- `predict_api.py` : 분석 서버 메인 모듈 (Flask)
  - `/predict` 로 받은 프레임을 YOLOv8로 검출
  - `save_txt=T` : 결과를 JSON(클래스·신뢰도·좌표)으로 반환
  - 그 외 : 박스를 그린 이미지 반환
  - 신뢰도·IoU·이미지 크기 등은 실행 인자로 조정
- `my_client.py` : 1초마다 프레임을 서버에 전송하는 클라이언트
- `general.py` : 요청에서 분석 대상·결과 형식 추출
- `send_mod.py` : 관제용 메타정보 생성, 이미지 Base64 변환
- `watcher.py` : 결과 폴더 감시 후 파일 자동 삭제 (watchdog)
- `hardhat.pt` : 안전모 검출 가중치 (head, helmet, person)

<br/><br/>

## 2. 진행 과정

<br/>

### ① 영상 수집 → 분석 요청

<br/>

- OpenCV로 영상을 읽어 프레임을 이미지로 저장
- 1초 간격으로 서버에 전송

<br/>

### ② 객체 검출

<br/>

- 사람, 헬멧, 머리(헬멧 미착용) 검출
- 검출 결과가 없는 프레임은 예외 처리 후 다음 프레임 진행

<br/>

### ③ 결과 전달

<br/>

- 클래스, 신뢰도, 박스 좌표를 JSON으로 정리
- 관제 화면용 Base64 이미지 + 메타정보 전송 로직 작성
- 전송 로직은 현재 주석 처리

<br/>

### ④ 결과 파일 정리

<br/>

- 1초마다 분석하면서 `runs/detect` 결과 폴더가 계속 누적
- `watcher.py` 로 새 파일 생성 시 자동 삭제

<br/><br/>

## 3. Result

<br/>

<img src="img/hardhat_img.png">

<br/>

- 테스트 영상 : `fallen.mp4` (위 이미지)
- 맨 위 이미지 : `hardhat.pt` 예측 결과
- 앉은 사람의 모자를 helmet으로 오검출 (신뢰도 0.68)
- 바닥에 누운 사람은 person 미검출
- 모자·쓰러진 자세 데이터를 학습에 추가해야 현장 적용 가능

<br/><br/>

## 4. 실행 방법

<br/>

```bash
python predict_api.py --weights hardhat.pt          # 분석 서버
python my_client.py                                 # 클라이언트
python predict_api.py --weights hardhat.pt --show   # 결과 화면 확인
```

<br/><br/>

## 5. Version

<br/>

- python 3.10
- Flask 3.0.0
- numpy 1.26.4
- pandas 2.2.2
- ultralytics 8.2.0
- opencv-python, requests, watchdog

<br/><br/>

## 6. Feedback

<br/>

- Base64 변환 시 원본보다 용량 증가
- 위험 상황이 검출된 프레임만 전송하는 방식이 적합
- 현재 첫 번째 검출 객체만 처리, 다수 작업자 대응 필요
- `watcher.py` 감시 경로가 로컬 절대경로라 환경별 수정 필요
