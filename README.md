# AI-Agent-Exercise
멀티에이전트 강의를 듣고 실습하는 레포지토리

## 01. Colab_CrewAI_실행예제_togetherai사용.ipynb
회사 노트북으로 주피터를 쓰려니 cmd 대부분의 기능이 보안프로그램으로 차단되어서 구글 코랩을 사용했습니다. OpenAI API는 선결제가 필요하여 무료 크레딧을 주는 TogetherAI API를 이용했고, Meta-Llama-3-8B-Instruct-Turbo 모델을 선택했습니다. 익숙한 모델명이어서 택한거고 다른 모델도 많아서 실습하실 땐 liteLLM에서 값싼 모델로 돌리면 되겠습니다.기사 써보라고 목차 설계자와 본문 작성자를 둬서 시켜보니 씁니다.
삽질했던 부분은 아래와 같습니다.
1. crewai가 litellm을 사용하여 LLM들에 연결되기 때문에 트랜스포머를 이용해 로컬 모델에 접근하는건 불가능합니다(litellm 목록 내 모델만 사용 가능)
2. Provider 제공 오류는 모델 경로를 상세히 지정해주어야 됩니다. 예를 들어 togherai를 사용할 경우 provider에 togherai를 적고 model에는 모델명만 적었는데 이러면 연결이 안되고, model 경로에도 최상단에 제공자명을 포함해야 됩니다
## 02. Custom_Tool_생성.ipynb
- 야후 파인내스 라이브러리는 API 사용에 제한이 있으므로 개인 용도로만 사용할 것

`crewai_tools` 라이브러리가 없는 현상 발생하여 아래 방식들을 시도하였습니다.

```python

# 1. pip 명령어로 설치
!pip install 'crewai[tools]'
# 2. 다른 방식으로 불러오기
from crewai.tools import BaseTool # --> 이 방법의 경우 추상클래스 부분이 꼬임
```
이에 수반해서 tool 모듈을 통해 decorator를 지정하는 방식이 실행되지 않았습니다. 따라서 Pydantic의 BaseModel을 상속받는 클래스를 생성하는 방식으로 해결했습니다.

`@tool` → `BaseTool` 상속으로 방식이 바뀐 것 같음. 따라서 데코레이터 대신 클래스를 정의하고 BaseTool을 상속(*이에 대해 자세한 이해가 필요하다면 Pydantic의 데이터 모델 클래스 안에서 사용되는 Field 클래스를 키워드로 검색) -> `_run` 함수도 지정해줘야되며 `name` 필드에 타입도 지정해줘야 오류가 안 남. -> 인스턴스 생성 후 실행하면 결과값을 얻을 수 있음.

```python
# 1. 라이브러리 넣고
from crewai.tools import BaseTool

# 2.  클래스 재정의 필요!
class LatestStockPriceTool(BaseTool):
    name: str = "latest_stock_price"  # ✅ 타입 힌트 추가
    description: str = "주어진 주식 티커의 최근 종가를 조회합니다."  # ✅ 타입 힌트 추가

    def _run(self, ticker: str) -> str:
        import yfinance as yf
        ticker_obj = yf.Ticker(ticker)
        historical = ticker_obj.history(period='5d', interval='1d')
        latest_price = historical['Close'].iloc[-1]
        return f"{ticker}의 최근 종가는 {latest_price:.2f}입니다."

    def _arun(self, *args, **kwargs):
        raise NotImplementedError("비동기 실행은 지원하지 않습니다.")

# 3. 만든 클래스에 대한 인스턴스 생성 후 실행
lastest_stock_price = LatestStockPriceTool() # ✅ 인스턴스 생성
latest_stock_price("AAPL") # __call__() 간접 호출
```
다른 툴 정의도 같은 방식으로 기존 코드에서 변환하면 실행됩니다.

![image](https://github.com/user-attachments/assets/f92acb55-57eb-4dc6-8e9c-a03ca1b61057)

