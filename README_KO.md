# UltraTimTrack 한국어 사용 설명서

이 문서는 현재 폴더의 `UltraTimTrack.m`과 `UltraTimTrack.fig`를 직접 확인하여 작성한 한국어 사용 설명서입니다. 원본 `README.md`의 내용뿐 아니라 실제 코드의 처리 순서, GUI 설정값의 용도, 저장 형식, 현재 버전에서 확인된 제한 사항을 함께 설명합니다.

> 중요: 현재 코드에는 원본 README의 설명과 실제 동작이 일치하지 않는 부분이 있습니다. 특히 `Process all` 실행 전 수동으로 지정한 fascicle의 유지 여부와 역방향 추적은 반드시 이 문서의 **수동 보정** 및 **알려진 제한 사항** 절을 확인하십시오.

## 1. 프로그램 개요

UltraTimTrack은 다음 세 가지 방법을 결합하여 B-mode 초음파 영상의 근섬유 다발(fascicle)과 건막(aponeurosis)을 추적합니다.

- **TimTrack (TT)**: 각 프레임에서 Hough transform 등을 이용해 fascicle과 aponeurosis 선을 독립적으로 검출합니다. 장시간 누적 드리프트는 적지만 프레임별 결과가 흔들릴 수 있습니다.
- **UltraTrack (UT)**: KLT 특징점과 affine optical flow를 이용해 이전 프레임의 구조를 다음 프레임으로 이동시킵니다. 결과는 부드럽지만 오차가 프레임을 따라 누적될 수 있습니다.
- **UltraTimTrack (UTT)**: Kalman filter를 이용하여 UltraTrack의 예측값과 TimTrack의 측정값을 결합합니다.

GUI에서 최종적으로 표시되는 주요 결과는 다음과 같습니다.

- fascicle length: mm
- fascicle angle: deep aponeurosis를 기준으로 한 각도, degree
- superficial aponeurosis: 파란색
- deep aponeurosis: 초록색
- 최종 추적 fascicle: 빨간색
- 수동 입력 fascicle: 자홍색(magenta)
- optical-flow 특징점: 작은 빨간색 `+`

## 2. 현재 설치 위치와 실행 환경

저장소 내 설치 위치:

```text
FirstDayCode/2.2. UltraTimTrack
```

필수 구성:

- `UltraTimTrack.m`
- `UltraTimTrack.fig`
- `TimTrack_parms.mat`
- `ultrasound_tracking_settings.mat`
- `functions/`
- `ultrasound-automated-algorithm/`

필요한 MATLAB Toolbox:

- Computer Vision Toolbox
- Image Processing Toolbox
- Signal Processing Toolbox
- Parallel Computing Toolbox: 병렬 처리 기능을 사용할 때 필요

현재 확인한 PC에는 MATLAB R2025b와 Computer Vision, Image Processing, Signal Processing Toolbox가 설치되어 있지만 Parallel Computing Toolbox는 설치되어 있지 않습니다. 따라서 현재 PC에서는 **Parallel processing을 체크하지 마십시오.**

## 3. 실행 방법

MATLAB Command Window에서 다음을 실행합니다.

```matlab
projectDir = fullfile('<MILO repository>', 'FirstDayCode', ...
    '2.2. UltraTimTrack');
cd(projectDir)
addpath(genpath(projectDir))
UltraTimTrack
```

또는 MATLAB Current Folder를 위 폴더로 변경한 뒤 `UltraTimTrack.m`을 열고 **Run**을 눌러도 됩니다.

`UltraTimTrack.m`은 시작할 때 자신의 하위 폴더를 MATLAB path에 자동으로 추가합니다. 그러나 `TimTrack_parms.mat`을 상대 경로로 불러오기 때문에, 가장 안전한 방법은 위와 같이 UltraTimTrack 폴더로 이동한 뒤 실행하는 것입니다.

## 4. 지원 입력 형식

### 4.1 동영상 및 단일 파일

메뉴에서 **File → Open video or image file...**을 선택하거나 `Ctrl+N`을 누릅니다.

코드가 지원하는 입력은 다음과 같습니다.

- MATLAB `VideoReader`가 지원하는 동영상 형식: 예를 들어 MP4, AVI 등
- `.mat`
- `.b32`, `.b8`
- `.jpg`, `.jpeg`, `.png`, `.bmp`, `.tiff`

단일 이미지 파일은 내부적으로 같은 이미지를 2프레임으로 복제하고 frame rate를 1 Hz로 설정하며 TimTrack mode를 켭니다.

### 4.2 MAT 입력의 요구 구조

`.mat` 파일을 직접 열 때 현재 코드는 `TVDdata` 구조체가 있다고 가정합니다. 주요 필드는 다음과 같습니다.

- `TVDdata.Im`: 영상 배열
- `TVDdata.Height`, `TVDdata.Width`
- `TVDdata.Fnum`: 프레임 수
- `TVDdata.Time`: timestamp

일반적인 임의 MAT 파일에 위 구조가 없으면 로드에 실패할 수 있습니다.

### 4.3 MP4와 같은 이름의 MAT sidecar

동영상과 같은 폴더에 이름이 같은 MAT 파일이 있으면 프로그램은 해당 파일의 `TVDdata`를 확인합니다.

예:

```text
trial_001.mp4
trial_001.mat
```

`TVDdata.cmPerPixY`가 있으면 image depth를 자동 계산하고, `TVDdata.Time`이 있으면 timestamp 기반 시간축을 사용할 수 있습니다. Telemed 영상에서는 이 조합을 사용하는 것이 유리합니다.

### 4.4 여러 이미지 불러오기

**File → Open multiple images**는 폴더 안의 이미지들을 영상 시퀀스로 불러옵니다. 이 경로에서는 frame rate가 코드상 100 Hz로 고정되며 TimTrack mode가 자동으로 켜집니다. 실제 촬영 frame rate가 다르면 결과 시간축이 부정확할 수 있으므로 주의하십시오.

## 5. 화면 구성과 색상

### 5.1 영상 위 표시

| 색상/표시 | 의미 |
|---|---|
| 파란 선 | superficial aponeurosis |
| 초록 선 | deep aponeurosis |
| 빨간 선 | 현재 최종 추적 fascicle |
| 자홍 선 | 사용자가 `Set manual`로 입력한 fascicle |
| 빨간 `+` | UltraTrack optical-flow 특징점 |
| 파란 반투명 상자 | superficial aponeurosis 검색 범위 |
| 초록 반투명 상자 | deep aponeurosis 검색 범위 |

처리 후 빨간 선과 자홍 선이 동시에 보이면, 자홍 선은 수동 입력이고 빨간 선은 Kalman filter를 거친 최종 결과입니다. 두 선이 반드시 완전히 겹치는 것은 아닙니다. `Manual variance`가 작을수록 최종 결과가 수동 입력에 더 가까워집니다.

### 5.2 그래프

- 왼쪽 아래: fascicle length, mm
- 오른쪽 아래: fascicle angle, degree
- 빨간 곡선: 최종 추적값
- 자홍색 `x`: 수동 입력값
- 검은 세로선: 현재 표시 중인 프레임

## 6. 가장 안정적인 기본 사용 순서: 정방향 추적

현재 코드에서 가장 안정적이고 문서화가 잘 된 방식은 첫 프레임에서 시작하는 정방향 추적입니다.

### 6.1 영상 불러오기

1. `File → Open video or image file...`을 선택합니다.
2. 영상이 모두 메모리에 로드될 때까지 기다립니다.
3. frame rate, resolution, image depth를 확인합니다.

### 6.2 영상 방향 확인

TimTrack의 Hough transform은 fascicle 방향에 영향을 받습니다.

- superficial 쪽 fascicle 끝이 영상 오른쪽에 있고 deep 쪽이 왼쪽에 있는 방향이 일반적으로 검출에 유리합니다.
- 반대 방향으로 보이면서 fascicle 검출이 실패하면 `Flip image`를 체크합니다.
- `Flip image`는 가능하면 추적 전에 결정하십시오.

### 6.3 aponeurosis 검색 상자 설정

1. 파란 상자를 superficial aponeurosis 주변에 둡니다.
2. 초록 상자를 deep aponeurosis 주변에 둡니다.
3. 각 상자 중앙에 목표 aponeurosis 하나만 포함되도록 합니다.
4. 전체 영상에서 목표 aponeurosis가 상자 밖으로 빠져나가지 않는지 확인합니다.
5. 다른 aponeurosis가 검색 상자 안으로 들어오지 않는 것이 좋습니다.

`Process all`을 누르면 현재 상자 위치가 TimTrack parameter로 저장되어 사용됩니다.

### 6.4 기본 설정

처음 실행할 때 권장하는 출발값:

| 항목 | 권장 출발값 |
|---|---:|
| Image depth | 초음파 장비의 실제 값 |
| Resolution reduction | 1 또는 2 |
| ROI type | Hough - local |
| Process noise covariance | 0.01 또는 0.1 |
| Cut-off frequency | 2–5 Hz 범위에서 기존 설정 유지 |
| Measurement noise covariance x-coordinate | 100 |
| # stationary frames | 1 |
| Manual variance | 0.01 또는 0.1 |
| TimTrack mode | 해제 |
| Track backwards | 해제 |
| Flip image | 필요한 경우만 체크 |
| Parallel processing | 현재 PC에서는 해제 |

### 6.5 Detect fascicle로 미리 확인

`Detect fascicle`은 현재 프레임에서 TimTrack을 한 번 실행하여 fascicle과 두 aponeurosis를 미리 확인합니다.

다음 사항을 검사하십시오.

- 파란 선이 superficial aponeurosis를 따르는가?
- 초록 선이 deep aponeurosis를 따르는가?
- 빨간 fascicle이 실제 fascicle 방향과 대략 일치하는가?

> 현재 구현에서 `Detect fascicle` 결과는 `Process all`의 영구적인 수동 초기값이 아닙니다. `Process all`은 시작할 때 기존 추적 구조를 삭제한 뒤 시작 프레임에서 자동 검출을 다시 수행합니다. 검색 상자와 설정값은 유지되므로, 미리보기가 올바르면 재검출 결과도 대체로 같습니다.

### 6.6 전체 처리

1. `Process all`을 누릅니다.
2. TimTrack 처리 진행 표시가 끝날 때까지 기다립니다.
3. UltraTrack 처리 진행 표시가 끝날 때까지 기다립니다.
4. MATLAB Command Window에 오류가 없는지 확인합니다.
5. 아래 두 그래프가 생성되는지 확인합니다.

현재 코드에서 `Process all`의 내부 순서는 다음과 같습니다.

1. 기존 `Region`과 `geofeatures` 삭제
2. 정방향이면 첫 프레임, 역방향이면 마지막 프레임으로 이동
3. 시작 프레임 자동 검출
4. ROI type에 `Hough`가 포함되면 모든 프레임에서 TimTrack 실행
5. TimTrack mode가 꺼져 있으면 UltraTrack optical flow 실행
6. Hough 방식이면 Kalman state estimation과 smoothing 실행
7. length 및 angle 그래프 갱신

### 6.7 결과 검사

1. 첫 프레임으로 이동합니다.
2. 재생 버튼 `>`을 누릅니다.
3. 전체 영상에서 빨간 fascicle과 파란색·초록색 aponeurosis를 확인합니다.
4. 다음과 같은 실패를 찾습니다.
   - 선이 실제 구조에서 갑자기 이탈
   - fascicle 방향이 반전
   - 부착점이 화면 밖으로 급격히 이동
   - length/angle 그래프에 비현실적인 spike
   - 점차 누적되는 drift

그래프만 보고 판단하지 말고 영상 overlay를 전 프레임에서 확인하십시오.

## 7. Settings 상세 설명

### 7.1 Frame rate (Hz)

동영상 메타데이터 또는 timestamp에서 읽은 초당 프레임 수입니다. 다음 계산에 사용됩니다.

- 시간축
- 재생 및 저장 동영상 frame rate
- cut-off frequency 정규화
- 결과 TXT/MAT의 시간 정보

회색 표시값이므로 일반적으로 GUI에서 직접 수정하지 않습니다.

### 7.2 Resolution (pix)

영상의 width와 height입니다. 회색 표시값이며 직접 수정하지 않습니다. crop을 적용하면 내부 `vidWidth`와 `vidHeight`가 바뀝니다.

### 7.3 Image depth (mm)

현재 영상 세로 방향의 실제 깊이입니다. fascicle length는 다음 비율로 계산됩니다.

```text
mm per pixel = Image depth / 영상 높이(pixel)
```

따라서 image depth가 10% 틀리면 계산된 fascicle length도 약 10% 비례하여 틀어집니다.

변경 방법:

- 왼쪽 Settings의 `Image depth (mm)` 입력창
- 메뉴 `Settings → Set image depth`
- 메뉴 `Image → Spatial calibration`

`Spatial calibration`에서는 영상에서 실제 거리를 알고 있는 두 점을 선택하고 mm 값을 입력합니다. 프로그램이 mm/pixel과 전체 image depth를 계산합니다.

### 7.4 Resolution reduction

TimTrack 입력 영상을 `1 / 값` 배율로 줄입니다.

- `1`: 원본 해상도
- `2`: 가로와 세로를 각각 절반으로 축소
- 값이 커질수록 TimTrack은 빨라지지만 작은 구조 검출 능력이 떨어질 수 있음

검출 후 좌표와 길이 일부는 원래 해상도로 다시 환산됩니다. 새로운 데이터셋에서는 먼저 `1`과 `2`를 비교하십시오.

### 7.5 ROI type

선택지는 세 가지입니다.

#### Optical flow

- TimTrack 전체 프레임 처리와 Kalman 결합을 사용하지 않는 비교 모드입니다.
- 시작 fascicle은 자동 검출로 생성되고 이후 UltraTrack optical flow로 이동합니다.
- 결과가 부드러울 수 있지만 장시간 영상에서 drift가 누적될 수 있습니다.

#### Hough - global

- TimTrack을 모든 프레임에서 실행합니다.
- optical-flow 특징점을 두 aponeurosis 사이의 비교적 넓은 영역에서 찾습니다.
- 자동 fascicle 선 주변에 좋은 특징점이 부족할 때 `Hough - local`과 비교할 수 있습니다.

#### Hough - local

- TimTrack의 fascicle 후보 선 주변 약 ±5 pixel 영역을 사용해 optical-flow 특징점을 제한합니다.
- 주변에 다른 fascicle이나 무관한 질감이 많은 경우 유리할 수 있습니다.
- 일반적인 기본 선택입니다.
- TimTrack이 fascicle을 완전히 잘못 검출하면 잘못된 위치 주변에서 특징점을 고를 수 있으므로 이때는 `Hough - global`을 비교하십시오.

### 7.6 Process noise covariance parameter, Q

UltraTrack 예측과 TimTrack 측정 중 무엇을 더 신뢰할지 조절하는 핵심 값입니다. 실제 process noise는 코드에서 다음처럼 계산됩니다.

```text
Q_actual = Q × 프레임 간 이동량²
```

- Q가 작음: UltraTrack 예측을 더 신뢰, 부드럽지만 drift 가능
- Q가 큼: TimTrack 측정을 더 신뢰, drift는 줄지만 결과가 더 흔들릴 수 있음
- Q가 0에 가까움: UltraTrack 쪽에 가까움
- Q가 `Inf`: TimTrack 쪽에 가까움

권장 튜닝 순서:

1. 먼저 `0.01` 또는 `0.1`로 전체 처리
2. 결과를 재생하여 drift와 jitter 확인
3. 처리 후 Q를 `0.01 → 0.1 → 1` 등으로 변경
4. 값을 입력하고 Enter를 눌러 재계산
5. 모든 조건에서 동일한 평가 기준을 사용

### 7.7 SA 버튼

`SA`는 sensitivity analysis입니다. 자동 최적화 버튼이 아닙니다.

현재 코드는 Q를 약 `10^-7`부터 `10^3`, 그리고 `Inf`까지 반복하여:

- state estimation 재실행
- 각 Q 결과 MAT 저장
- length와 angle 곡선 비교
- noise와 drift 지표 계산
- 가중치에 따른 최적 Q 그래프 생성

많은 계산과 파일 저장이 발생하므로 일반 추적에서는 누르지 않는 것이 좋습니다.

### 7.8 Cut-off frequency (Hz)

TimTrack 측정값의 고주파 성분으로 measurement variance를 추정할 때 사용합니다. 최종 결과에 단순 low-pass filter를 적용하는 값은 아닙니다.

코드는 `1.5 × cut-off frequency`를 Nyquist frequency로 정규화한 뒤 2차 high-pass Butterworth filter를 사용합니다. 특별한 검증 없이 먼저 조절하기보다는 Q를 먼저 튜닝하십시오.

### 7.9 Measurement noise covariance x-coordinate sup. attachment

fascicle의 superficial attachment x-coordinate에 대한 측정 noise입니다.

- 작은 값: 시작 위치 또는 anchor를 더 강하게 신뢰
- 큰 값: optical-flow 이동을 더 허용
- `Inf`: 이 좌표에서 optical-flow 예측을 사실상 그대로 사용

기본 출발값으로 `100`을 유지하고, superficial attachment가 비현실적으로 좌우로 움직일 때 조절하십시오.

### 7.10 # stationary frames

초기 fascicle angle의 평균과 분산을 계산할 프레임 수입니다.

- 시작 부분에 근육 길이와 힘이 일정한 정지 구간이 있을 때만 1보다 크게 설정
- 시작부터 움직이는 영상은 `1`
- 정지하지 않은 프레임을 포함하면 초기 angle과 covariance가 왜곡될 수 있음

현재 코드에서는 정방향일 때 시작 프레임부터, 역방향일 때 마지막 프레임 부근부터 해당 수만큼 사용합니다.

### 7.11 Manual variance

수동 입력을 Kalman filter가 얼마나 신뢰할지 정합니다.

- 작은 값: 수동 입력을 강하게 신뢰
- 큰 값: 수동 입력의 영향이 약해짐
- 수동 입력이 없으면 결과에 영향을 주지 않음

현재 `0.01`은 수동 입력을 강하게 신뢰하는 설정입니다.

### 7.12 TimTrack mode

체크하면 UltraTrack과 Kalman 결합을 건너뛰고 TimTrack 결과만 사용합니다.

- 독립된 이미지 처리
- TimTrack 자체 결과 비교
- 단일 이미지 또는 이미지 폴더 처리

프레임별 노이즈가 클 수 있으며 일반적인 동영상의 UltraTimTrack 분석에서는 보통 해제합니다.

### 7.13 Track backwards

마지막 프레임에서 시작하여 첫 프레임 방향으로 UltraTrack optical flow와 state estimation을 실행합니다.

코드에는 역방향 함수가 구현되어 있지만 원본 README에는 forward tracking만 지원한다고 적혀 있습니다. 또한 현재 버전에는 역방향 수동 초기화 문제가 있으므로 **실험적 기능으로 취급**하십시오.

- 추적 전에 체크 여부를 결정하십시오.
- 이미 전체 추적이 끝난 뒤에는 방향 변경을 제한합니다.
- 마지막 프레임의 구조가 첫 프레임보다 명확할 때 사용할 수 있습니다.
- 결과를 정방향 추적 및 수동 확인과 반드시 비교하십시오.

### 7.14 Flip image

영상을 좌우 반전하며, 이미 추적 결과가 있으면 관련 좌표도 함께 변환합니다.

- fascicle의 superficial 쪽이 왼쪽에 있어 Hough 검출 방향이 맞지 않을 때 사용
- 가능하면 영상을 불러온 직후, `Detect fascicle` 및 `Process all` 전에 설정
- 체크를 변경한 뒤 기존 결과가 혼란스럽다면 `Clear` 후 다시 처리

### 7.15 Parallel processing

TimTrack의 프레임별 계산을 `parfor`로 실행합니다. UltraTrack 전체가 병렬화되는 것은 아닙니다.

- Parallel Computing Toolbox 필요
- 결과의 의미는 동일하고 처리 시간만 단축
- 현재 PC에서는 Toolbox가 없으므로 해제

처리 예상 시간이 60초보다 길고 체크가 꺼져 있으면 프로그램이 병렬 처리를 사용할지 묻습니다. Toolbox가 없는 환경에서는 `No`를 선택하십시오.

## 8. 수동 보정의 정확한 사용법

### 8.1 Set manual의 두 번 클릭 방식

`Set manual`은 한 번 누르는 버튼처럼 보이지만 실제로는 두 단계로 동작합니다.

1. 첫 번째 클릭: 드래그 가능한 선 세 개 생성
   - 빨강: fascicle
   - 파랑: superficial aponeurosis
   - 초록: deep aponeurosis
2. 각 선의 끝점을 실제 구조에 맞게 이동
3. 두 번째 클릭: 현재 좌표를 manual measurement로 저장하고 선 편집 종료

수동 fascicle은 두 aponeurosis와 교차하도록 충분히 길게 놓는 것이 좋습니다.

### 8.2 현재 코드에서 권장되는 수동 보정 순서

현재 코드는 `Process all` 시작 시 기존 추적 데이터를 삭제합니다. 따라서 다음 순서를 사용하십시오.

1. 먼저 `Process all`을 완료합니다.
2. 잘못 추적된 프레임으로 이동합니다.
3. `Set manual`을 한 번 누릅니다.
4. 빨강·파랑·초록 선을 실제 구조에 맞춥니다.
5. `Set manual`을 다시 눌러 확정합니다.
6. Kalman 결과와 그래프가 다시 계산될 때까지 기다립니다.
7. 수동선은 자홍색, 최종 결과는 빨간색으로 표시되는지 확인합니다.
8. 이후 **`Process all`을 다시 누르지 않습니다.**
9. 재생하여 전체 결과를 검사한 뒤 저장합니다.

### 8.3 Clear manual

`Clear manual`은 수동 입력 필드를 지우고 원래 자동 추적값을 복원한 뒤 state estimation을 다시 시도합니다.

`Clear`와 다릅니다.

- `Clear manual`: 수동 보정만 제거
- `Clear`: `Region`, `geofeatures`, optical-flow points 및 그래프 등 전체 추적 결과 제거

### 8.4 처리 전 수동 초기화가 유지되지 않는 이유

현재 `process_all_Callback`은 시작하자마자 `menu_clear_tracking_Callback`을 호출합니다. 이 함수는 다음 항목을 제거합니다.

- `handles.Region`
- `handles.geofeatures`

manual fascicle과 manual aponeurosis도 `handles.Region` 안에 저장되므로 함께 사라집니다. 그 뒤 프로그램은 시작 프레임에서 자동 검출을 다시 실행합니다.

따라서 현재 버전에서는 다음 순서가 의도대로 동작하지 않습니다.

```text
Detect fascicle
→ Set manual 두 번으로 초기 fascicle 수정
→ Process all
```

이 순서를 사용하면 `Process all`에서 수동선이 초기값으로 보존되지 않고 자동 검출선으로 다시 시작할 수 있습니다.

### 8.5 역방향 수동 초기화의 추가 문제

역방향 추적에서는 마지막 프레임이 시작점입니다. 그러나 manual callback은 현재 다음 경우에만 수동선을 실제 추적선으로 즉시 복사합니다.

```text
현재 프레임이 start_frame이거나 TimTrack mode인 경우
```

마지막 프레임은 `start_frame`이 아니므로 일반 UltraTimTrack 역방향 모드에서는 이 조건을 만족하지 않습니다. 즉, 처리 전 마지막 프레임을 수동 지정하는 방식은 현재 코드에서 완전히 지원되지 않습니다.

역방향에서 수동 보정이 필요하면:

1. `Track backwards`를 체크한 상태로 먼저 `Process all` 실행
2. 처리 완료 후 마지막 프레임으로 이동
3. `Set manual` 두 번으로 보정
4. `Process all`을 다시 누르지 않고 재계산 결과 확인

## 9. ROI 검색 상자와 Detect fascicle

`Detect fascicle`은 다음 작업을 수행합니다.

1. 파란색·초록색 상자 위치를 normalized aponeurosis 범위로 저장
2. 현재 프레임을 resolution reduction 값에 따라 축소
3. `auto_ultrasound`를 호출하여 TimTrack 실행
4. 검출 좌표를 원래 영상 해상도로 환산
5. fascicle과 aponeurosis 교차점 계산
6. 해당 프레임 결과 표시

aponeurosis는 잘 검출되지만 fascicle이 잘못 검출될 때:

- `Flip image` 전환 후 `Clear` 및 재검출
- `Resolution reduction`을 2에서 1로 낮춰 세부 구조 보존
- `Settings → Set TimTrack settings`에서 Hough 관련 parameter 조정
- `Hough - global`과 `Hough - local` 비교
- 전체 처리 후 `Set manual`로 해당 프레임 보정

## 10. 영상 구간 자르기

### 10.1 Cut before

1. 추적을 시작할 프레임으로 이동합니다.
2. `Cut before`를 한 번 누릅니다.
3. 선택한 프레임이 새 첫 프레임이 됩니다.

### 10.2 Cut after

1. 추적을 끝낼 프레임으로 이동합니다.
2. `Cut after`를 한 번 누릅니다.
3. 선택한 프레임이 새 마지막 프레임이 됩니다.

주의 사항:

- 원본 README는 각 버튼을 한 번만 사용할 것을 권장합니다.
- 여러 번 자르면 `start_frame`, `NumFrames`, 시간축과 저장 index가 어긋날 수 있습니다.
- 잘못 잘랐다면 영상을 다시 불러오고 처음부터 한 번만 자르는 것이 안전합니다.

## 11. 영상 crop

메뉴 `Image`에는 다음 기능이 있습니다.

- Automatically crop image
- Manually crop image
- Reset image
- Spatial calibration

crop은 내부 영상 배열과 width/height를 변경합니다. fascicle length 환산은 `Image depth / vidHeight`를 사용하므로 crop 후 image depth가 물리적으로 무엇을 의미하는지 반드시 다시 확인해야 합니다.

개발 메모에는 crop과 ROI/fascicle 위치에 관한 문제가 기록되어 있습니다. 중요한 분석에서는 다음을 권장합니다.

1. 가능하면 원본 영상으로 먼저 검증
2. crop 전후 결과 비교
3. crop 후 spatial calibration 또는 image depth 재확인
4. 기존 추적 결과가 있는 상태에서 crop하지 않기

## 12. Optical-flow block size

메뉴 `Settings → Set block size`에서 optical-flow tracker의 block size를 바꿀 수 있습니다. 시작값은 코드상 `[21 71]`입니다.

- 큰 block: 넓은 질감을 사용하고 노이즈에 비교적 강하지만 작은 국소 움직임을 놓칠 수 있음
- 작은 block: 국소 움직임에 민감하지만 노이즈와 특징 부족에 취약할 수 있음
- 이미 전체 추적이 끝난 상태에서 block size를 변경하면 optical flow와 state estimation을 다시 실행할 수 있음

데이터셋별 검증 없이 임의로 변경하지 않는 것이 좋습니다.

## 13. 결과 저장

### 13.1 Save tracking as TXT

메뉴: `File → Save tracking as TXT`

저장 내용:

- Time
- Fascicle Length
- Fascicle/Pennation Angle

탭으로 구분된 텍스트 파일입니다.

### 13.2 Save tracking as MAT

메뉴: `File → Save tracking as MAT`, 단축키 `Ctrl+S`

자동 파일명 형식:

```text
원본파일명_tracked_Q=값.mat
```

주요 변수:

- `TrackingData`
  - `res`: image depth
  - `start_frame`
  - `NumFrames`
  - `ProcessingTime`
  - `BlockSize`
  - `Parallel`
  - aponeurosis 검색 영역 `S`, `D`
- `Fdat`
  - `Region`
  - `FL`: fascicle length
  - `PEN`: fascicle angle
  - `ANG`: horizontal 기준 fascicle angle이 존재할 때
  - `Time`
  - `geofeatures`: TimTrack 결과가 존재할 때
  - `R`: 측정 variance가 계산된 경우

### 13.3 Save video

메뉴: `File → Save video`, 단축키 `Ctrl+V`

추적 overlay가 포함된 MPEG-4 동영상을 생성합니다. frame rate는 입력 영상의 frame rate를 사용합니다.

### 13.4 Save image

메뉴: `File → Save image`, 단축키 `Ctrl+I`

현재 표시 프레임을 TIFF로 저장합니다.

### 13.5 Save fascicle / Load fascicle

현재 프레임의 fascicle, aponeurosis, ROI 및 가능한 경우 TimTrack geofeatures를 별도의 MAT 파일로 저장하거나 불러옵니다.

다른 프레임 또는 다른 영상에 불러온 fascicle은 영상 좌표와 해상도가 동일한지 확인해야 합니다. geofeatures가 없는 fascicle 파일은 optical-flow 방식으로만 안전하게 사용할 수 있다는 경고가 Command Window에 출력됩니다.

## 14. 결과 품질 확인 체크리스트

각 영상에 대해 다음을 기록하는 것이 좋습니다.

- [ ] 원본 파일명
- [ ] frame rate
- [ ] resolution
- [ ] image depth와 calibration 근거
- [ ] Flip image 여부
- [ ] crop 여부 및 crop 범위
- [ ] Cut before/after 여부
- [ ] Resolution reduction
- [ ] ROI type
- [ ] Q
- [ ] Cut-off frequency
- [ ] x-coordinate measurement covariance
- [ ] stationary frames
- [ ] Manual variance
- [ ] Block size
- [ ] 정방향/역방향
- [ ] TimTrack mode 여부
- [ ] 수동 보정 프레임 목록
- [ ] 전체 overlay 육안 검토 완료
- [ ] 저장된 MAT/TXT/동영상 경로

## 15. 자주 발생하는 문제

### 15.1 aponeurosis는 맞지만 fascicle이 반대 방향으로 검출됨

1. `Clear`
2. `Flip image` 전환
3. 검색 상자 재확인
4. `Resolution reduction = 1`로 비교
5. `Detect fascicle` 재실행

### 15.2 수동 초기 fascicle이 Process all 후 사라짐

현재 코드 동작입니다. `Process all`이 기존 `Region`을 삭제하기 때문입니다.

해결:

- 먼저 `Process all`
- 그다음 잘못된 프레임에서 `Set manual` 두 번
- 이후 `Process all`을 다시 누르지 않음

### 15.3 수동선과 최종 빨간 선이 다름

- 자홍색: manual measurement
- 빨간색: Kalman filter 최종 추정

`Manual variance`를 줄이면 manual을 더 강하게 신뢰합니다. 단, 지나치게 작은 값을 무조건 사용하는 대신 앞뒤 프레임 연결이 자연스러운지 확인하십시오.

### 15.4 역방향 마지막 프레임 수동 초기화가 반영되지 않음

현재 manual callback이 첫 프레임만 초기 프레임으로 인정하는 코드 조건 때문입니다. 먼저 역방향 `Process all`을 실행한 뒤 마지막 프레임을 수동 보정하십시오.

### 15.5 Parallel processing 오류

Parallel Computing Toolbox가 없으면 체크를 해제하고 병렬 처리 질문에 `No`를 선택하십시오.

### 15.6 length 단위가 이상함

다음을 확인하십시오.

- Image depth가 실제 촬영 설정과 같은가?
- crop 후 image depth/calibration을 다시 확인했는가?
- MP4 sidecar MAT의 `cmPerPixY`와 `Height`가 올바른가?
- 표시된 resolution의 세로 pixel 수가 예상과 같은가?

### 15.7 결과가 부드럽지만 점점 이동함

UltraTrack drift 가능성이 있습니다.

- Q를 조금 증가
- Hough 방식 사용 확인
- TimTrack fascicle/aponeurosis 검출 품질 확인
- `Hough - local`과 `Hough - global` 비교

### 15.8 결과가 프레임마다 심하게 흔들림

TimTrack 영향을 너무 강하게 받거나 검출 품질이 낮을 수 있습니다.

- Q를 감소
- 검색 상자에 목표 aponeurosis만 포함
- Resolution reduction을 1로 비교
- Flip image 방향 확인
- TimTrack settings 검토

## 16. 현재 코드에서 확인된 제한 사항

다음 항목은 사용자의 조작 실수와 구분해야 합니다.

1. **Process all이 수동 초기값을 삭제함**
   `process_all_Callback`이 시작 시 `menu_clear_tracking_Callback`을 호출하여 `Region`과 `geofeatures`를 삭제합니다.

2. **역방향 manual 시작점 조건 누락**
   manual 확정 코드는 첫 프레임 또는 TimTrack mode만 즉시 적용하며, 역방향 시작점인 마지막 프레임은 조건에 포함하지 않습니다.

3. **README와 역방향 코드의 불일치**
   README에는 forward tracking만 지원한다고 적혀 있지만 소스에는 backward optical-flow 및 backward state-estimation 함수가 있습니다. 충분히 검증된 기능으로 단정해서는 안 됩니다.

4. **crop 관련 개발 메모**
   개발용 `functions/README.md`에는 crop 후 fascicle/ROI 위치 문제와 시간축/index 문제 가능성이 기록되어 있습니다.

5. **여러 이미지 입력의 frame rate 고정**
   이미지 폴더 입력은 실제 촬영 속도와 무관하게 100 Hz로 설정됩니다.

6. **SA가 다수 파일을 자동 저장함**
   sensitivity analysis는 단순 미리보기가 아니며 각 Q 결과를 저장합니다.

## 17. 권장 분석 원칙

1. 모든 영상에 동일한 parameter를 무조건 적용하기 전에 대표 영상으로 검증하십시오.
2. 그래프뿐 아니라 전 프레임 overlay를 확인하십시오.
3. image depth와 timestamp를 원자료와 대조하십시오.
4. Q 조절 전 TimTrack 검출 자체가 올바른지 먼저 확인하십시오.
5. 수동 보정은 `Process all` 이후에 수행하십시오.
6. 수동 보정 후에는 `Process all`을 다시 누르지 마십시오.
7. 역방향 결과는 가능하면 정방향 결과와 비교하십시오.
8. 분석에 사용한 설정과 수동 보정 프레임을 기록하십시오.

## 18. 관련 소스 위치

동작을 다시 확인할 때 참고할 주요 함수:

| 기능 | `UltraTimTrack.m` 함수 |
|---|---|
| GUI 초기화 | `UltraTimTrack_OpeningFcn` |
| 파일 로드 | `menu_load_file_Callback` |
| 전체 초기화 | `menu_clear_tracking_Callback`, `clear_fascicle_Callback` |
| 전체 처리 | `process_all_Callback` |
| 정방향 UltraTrack | `process_all_UltraTrack` |
| 역방향 UltraTrack | `process_all_UltraTrack_backwards` |
| TimTrack 전체 처리 | `process_all_TimTrack` |
| Kalman 결합 | `do_state_estimation`, `run_kalman_filter` |
| fascicle state update | `state_estimator`, `update_Fascicle` |
| 길이·각도 계산 | `calc_fascicle_length_and_pennation` |
| 현재 프레임 자동 검출 | `Auto_Detect_Callback` |
| manual 설정 | `manual_estimate_Callback`, `extract_estimates` |
| manual 제거 | `clear_manual_Callback` |
| 좌우 반전 | `do_flip` |
| 결과 표시 | `show_image`, `show_data` |
| MAT 저장 | `Save_As_Mat_Callback` |
| TXT 저장 | `Save_As_Txt_Callback` |

---

이 설명서는 현재 로컬에 복제된 UltraTimTrack 코드의 실제 동작을 기준으로 작성되었습니다. upstream 코드가 변경되면 함수 동작과 GUI 구성이 달라질 수 있으므로, 버전을 업데이트한 뒤에는 이 문서의 제한 사항을 다시 확인해야 합니다.
