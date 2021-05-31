# Python-Cryptocurrency-investment-automation
유튜브 조코딩님의 파이썬 비트코인 투자 자동화



## 기술

거래소 - 업비트

전략 - 변동성 돌파 전략



## 목차

### 1. 업비트 가입하기

- 앱스토어를 통한 다운로드
- 카카오계정으로 로그인
- 보안등급 설정 - 최소 레벨4 이상 권장(원화출금 가능)
  - Kbank 계좌 연결(계좌 개설 필요)
    - 원화 가져오기(케이뱅크 ➡ 업비트)

### 2. API 키 발급받기

- UPbit 홈페이지(https://upbit.com/home) → 고객센터 → Open API 안내 → Open API 사용하기
- Open API 관리
  - Open API Key 관리
    - 자산조회, 주문조회, 주문하기 체크
    - 특정 IP에서만 실행 체크(공개 IP입력)
    - Access Key, Secret Key 별도 보관

### 3. 개발 환경 세팅

#### 1. 파이썬 3.x.x

- https://www.python.org/ → downloads → 운영체제에 맞게 선택 → Python 3.8.9 (64-bit)
- Install Python 3.8.9 (64-bit)
  - Add Python 3.8 to PATH 체크 → Customize installation
    - Optional Feature 전부 체크
    - Advanced Options 전부 체크, Customize install loaction ``C:\python38-64`` 
    - install → close(**Disable path length limit**를 클릭)
- 환경 변수
  - 시스템 변수 PATH 편집 → ``C:\python38-64`` 편집

#### 2. Visual Studio

- 작업 경로 선택: File - Open Folder...
- Terminal - New Terminal - Command Prompt
- Python Extension 설치
- Interpreter를 버젼에 맞게 설정(python 3.8.9 64-bit)
- pyupbit 라이브러리 설치
  - 해당 GitHub 접속 - https://github.com/sharebook-kr/pyupbit 
  - ``pip install pyupbit``



### 4. API로 잔고 조회하기

```python
import pyupbit

access = "input-your-access-key"          # 본인 값으로 변경
secret = "input-yout-secret-key"          # 본인 값으로 변경
upbit = pyupbit.Upbit(access, secret)

print(upbit.get_balance("KRW-XRP"))     # KRW-XRP 조회
print(upbit.get_balance("KRW"))         # 보유 현금 조회
```



### 5. 백테스팅

**과거의  데이터로  전략을 테스트 해보는 것**

- 위키독스 - 파이썬을 이용한 비트코인 자동매매 (개정판) GitHub 참고 
  (https://github.com/sharebook-kr/book-cryptocurrency)

  - ch07 
    07_13.py 수정 (``pip install openpyxl`` 설치)

    ```python
    import pyupbit
    import numpy as np
    
    #OHLCV(open, high, low, close, volume) 당일 시가, 고가, 저가, 종가, 거래량에 대한 데이터
    df = pyupbit.get_ohlcv("KRW-BTC", count=7)
    
    # 변동성 돌파 기준 범위 계산, (고가 - 저가) * k값
    df['range'] = (df['high'] - df['low']) * 0.5
    
    # target(매수가), range 컬럼을 한 칸씩 밑으로 내림(.shift(1))
    df['target'] = df['open'] + df['range'].shift(1)
    
    # ror(수익율), np.where(조건문, 참일 때 값, 거짓일 때 값)
    df['ror'] = np.where(df['high'] > df['target'],
                         df['close'] / df['target'],
                         1)
    
    # 누적 곱 계산(cumprod) => 누적 수익율
    df['hpr'] = df['ror'].cumprod()
    
    # Draw Down 계산 (누적 최대 값과 현재 hpr 차이 / 누적 최대값 * 100)
    df['dd'] = (df['hpr'].cummax() - df['hpr']) / df['hpr'].cummax() * 100
    
    # MOD 계산
    print("MDD(%): ", df['dd'].max())
    
    #엑셀로 출력    
    df.to_excel("dd.xlsx")
    ```

    

    07_11.py (가장 좋은 k값 구하기)

    ```python
    import pyupbit
    import numpy as np
    
    
    def get_ror(k=0.5):
        df = pyupbit.get_ohlcv("KRW-BTC", count=7)
        df['range'] = (df['high'] - df['low']) * k
        df['target'] = df['open'] + df['range'].shift(1)
    
        df['ror'] = np.where(df['high'] > df['target'],
                             df['close'] / df['target'],
                             1)
    
        ror = df['ror'].cumprod()[-2]
        return ror
    
    
    for k in np.arange(0.1, 1.0, 0.1):
        ror = get_ror(k)
        print("%.1f %f" % (k, ror))
    ```

    

### 6. 자동매매 코드 구현

- 조코딩의 GitHub 참고 (https://github.com/youtube-jocoding/pyupbit-autotrade)
  bitCoinAutoTrade.py

  ```python
  import time
  import pyupbit
  import datetime
  
  access = "your-access"
  secret = "your-secret"
  
  def get_target_price(ticker, k):
      """변동성 돌파 전략으로 매수 목표가 조회"""
      df = pyupbit.get_ohlcv(ticker, interval="day", count=2)
      target_price = df.iloc[0]['close'] + (df.iloc[0]['high'] - df.iloc[0]['low']) * k
      return target_price
  
  def get_start_time(ticker):
      """시작 시간 조회"""
      df = pyupbit.get_ohlcv(ticker, interval="day", count=1)
      start_time = df.index[0]
      return start_time
  
  def get_balance(ticker):
      """잔고 조회"""
      balances = upbit.get_balances()
      for b in balances:
          if b['currency'] == ticker:
              if b['balance'] is not None:
                  return float(b['balance'])
              else:
                  return 0
      return 0
  
  def get_current_price(ticker):
      """현재가 조회"""
      return pyupbit.get_orderbook(tickers=ticker)[0]["orderbook_units"][0]["ask_price"]
  
  # 로그인
  upbit = pyupbit.Upbit(access, secret)
  print("autotrade start")
  
  # 자동매매 시작
  while True:
      try:
          now = datetime.datetime.now()
          start_time = get_start_time("KRW-BTC")
          end_time = start_time + datetime.timedelta(days=1)
  
          if start_time < now < end_time - datetime.timedelta(seconds=10):
              target_price = get_target_price("KRW-BTC", 0.5)
              current_price = get_current_price("KRW-BTC")
              if target_price < current_price:
                  krw = get_balance("KRW")
                  if krw > 5000:
                      upbit.buy_market_order("KRW-BTC", krw*0.9995)
          else:
              btc = get_balance("BTC")
              if btc > 0.00008:
                  upbit.sell_market_order("KRW-BTC", btc*0.9995)
          time.sleep(1)
      except Exception as e:
          print(e)
          time.sleep(1)
  ```
  



### 7. 클라우드 서버에서 돌리기

개인 GCP 사용

#### Ubuntu 서버 명령어

- (*추가)한국 기준으로 서버 시간 설정: sudo ln -sf /usr/share/zoneinfo/Asia/Seoul /etc/localtime
- 현재 경로 상세 출력: ls -al
- 경로 이동: cd 경로
- vim 에디터로 파일 열기: vim bitcoinAutoTrade.py
- vim 에디터 입력: i
- vim 에디터 저장: :wq!
- 패키지 목록 업데이트: sudo apt update
- pip3 설치: sudo apt install python3-pip
- pip3로 pyupbit 설치: pip3 install pyupbit
- 백그라운드 실행: nohup python3 bitcoinAutoTrade.py > output.log &
- 실행되고 있는지 확인: ps ax | grep .py
- 프로세스 종료(PID는 ps ax | grep .py를 했을때 확인 가능): kill -9 PID