---
layout: post
title: "쉽게 에이전트를 만들 수 있는 라이브러리, smolagents를 소개합니다."
author: jeong
categories: [Agent]
image: assets/images/blog/posts/2025-09-26-Introducing-smolagents/thumbnail.png
---
* TOC
{:toc}
<!--toc-->

_이 글은 Hugging Face 블로그의 [Introducing smolagents, a simple library to build agents](https://huggingface.co/blog/smolagents)를 한국어로 번역한 글입니다._

---
HuggingFace에서 LLM(Language Model)에 **에이전트 기능**을 부여하는 라이브러리 `smolagents`를 공개했습니다! 
간단히 살펴볼까요?

```python
from smolagents import CodeAgent, DuckDuckGoSearchTool, HfApiModel

agent = CodeAgent(tools=[DuckDuckGoSearchTool()], model=HfApiModel())

agent.run("How many seconds would it take for a leopard at full speed to run through Pont des Arts?") # 표범이 최고 속력으로 Pont des Arts 다리를 달린다면 몇 초 걸리는가?
```

![](https://huggingface.co/datasets/huggingface/documentation-images/resolve/main/smolagents/smolagents.gif)

## 목차

- [🤔 에이전트란 무엇인가?](https://huggingface.co/blog/smolagents#%F0%9F%A4%94-what-are-agents)
- [✅ 에이전트를 언제 써야하고 / ⛔ 언제 피해야 하는지](https://huggingface.co/blog/smolagents#%E2%9C%85-when-to-use-agents--%E2%9B%94-when-to-avoid-them)
- [**Code agents**](https://huggingface.co/blog/smolagents#code-agents)
- [***smolagents 소개* : 쉽게 에이전트 만들기  🥳**](https://huggingface.co/blog/smolagents#introducing-smolagents-making-agents-simple-%F0%9F%A5%B3)
    - [**에이전트 만들기**](https://huggingface.co/blog/smolagents#building-an-agent)
    - [**에이전틱 워크플로우에서 오픈 모델은 얼마나 강력할까?**](https://huggingface.co/blog/smolagents#how-strong-are-open-models-for-agentic-workflows)
- [**다음 단계 🚀**](https://huggingface.co/blog/smolagents#next-steps-%F0%9F%9A%80)

## **🤔 에이전트란 무엇인가?**

효율적인 AI 시스템은 **LLM이 외부 세계와 상호작용할 수 있는 방법**을 제공해야 합니다. 예: 검색 도구를 불러 외부 정보를 가져오거나, 프로그램을 실행해 특정 작업을 해결하는 것. 즉, LLM에는 **에이전시(agency)**가 필요합니다. 에이전틱한 프로그램은 LLM이 외부 세계와 연결되는 게이트웨이라고 할 수 있습니다.

AI 에이전트란 **LLM의 출력이 워크플로우를 제어하는 프로그램**입니다.

- LLM을 사용하는 시스템은 LLM출력을 코드로 통합합니다. LLM 출력이 코드 워크플로우에 어떤 영향을 주는 가가 곧 그 시스템의 에이전시 수준(단계) 입니다.

- 이는 agent 가 ‘0/1’ 같은 바이너리한 개념이 아니라, LLM에 권한을 얼마나 주느냐에 따라 **연속적인 agency 스펙트럼**을 가진다는 것을 보여줍니다.

시스템별 에이전시 단계표

| **Agency Level(에이전시 단계)** | **설명** | **명칭** | **예시**  |
| --- | --- | --- | --- |
| ☆☆☆ | LLM 출력이 프로그램 흐름에 아무런 영향을 줄 수 없음 | 단순 처리기 | `process_llm_output(llm_response)` |
| ★☆☆ | LLM 출력이 분기를 제어함 | 라우터 | `if llm_decision(): path_a() else: path_b()` |
| ★★☆ | LLM 출력이 함수 실행을 결정 | Tool call(툴 호출) | `run_function(llm_chosen_tool, llm_chosen_args)` |
| ★★★ | LLM 출력이 반복, 프로그램 지속을 제어함 | Multi-step(멀티스텝)에이전트 | `while llm_should_continue(): execute_next_step()` |
| ★★★ | 한 에이전틱 워크플로우가 다른 에이전트 실행 가능 | Multi-Agent(멀티 에이전트) | `if llm_trigger(): execute_agent()` |

멀티스텝 에이전트는 아래 구조를 가집니다:

```python
memory = [user_defined_task]
while llm_should_continue(memory): *# 반복이 멀티스텝 부분*
    action = llm_get_next_action(memory) *# 툴 호출 부분*
    observations = execute_action(action)
    memory += [action, observations]
```

이 시스템은 루프를 돌면서 각 단계마다 새로운 동작(Action)을 수행합니다.(함수 형태로 정의된 사전 지정 *도구* 호출 등). 그런 다음 관찰(Observe) 결과를 통해 주어진 작업을 해결하기에 충분한 상태에 도달했다고 판단될 때까지 반복합니다.

아래는 멀티스텝 에이전트가 간단한 수학 문제를 푸는 예시입니다:

![](https://huggingface.co/datasets/huggingface/documentation-images/resolve/main/transformers/Agent_ManimCE.gif)

## **✅ 에이전트를 언제 써야하고 / ⛔ 언제 피해야 하는지**

에이전트는 LLM이 애플리케이션의 워크플로우를 결정해야 할 때 유용합니다. 하지만 과잉일 때가 종종 발생하니, 고려해야할 질문점은 : “지금 당면한 과제를 효율적으로 해결하기 위해서는 **워크플로우의 유연성이** 필요할까?” 미리 정해둔 워크플로우로는 한계가 있다면, 더 큰 유연성이 필요하다는 뜻입니다. 예시를 들어볼게요. 여러분이 서핑 여행 웹사이트에서 고객 요청을 처리하는 앱을 만든다고 해봅시다.

사용자의 선택에 따라 요청이 두 가지 경우 중 반드시 하나에 속할 것이라는 걸 미리 알 수 있고, 각 경우마다 미리 정해둔 워크플로가 있다고 가정해 봅시다.

1. 여행 관련 정보를 알고 싶다 ⇒ 지식베이스를 검색할 수 있는 검색창 제공
2. 영업팀과 이야기하고 싶다 ⇒ 문의 양식을 작성하도록 안내

만약 이런 워크플로우가 모든 요청을 처리할 수 있다면, 모두 다 코드로 짜는 게 좋습니다! 

이렇게 하면 LLM 같은 예측 불가능한 요소가 개입해 오류를 만들 위험 없이 100% 안정적인 시스템을 얻을 수 있습니다. 에이전트 같은 행위자적 기능을 사용하지 않아야 앱을 단순하고 견고하게 만들 수 있죠.

하지만 워크플로우를 미리 정할 수 없는 경우는 어떨까요?

예를 들어, 한 사용자가 이렇게 묻는다고 해봅시다: `"나는 월요일에 올 수 있는데, 여권을 잊어버려서 수요일까지 늦어질 수도 있어. 화요일 아침에 내 짐과 나를 서핑하러 데려갈 수 있을까? 취소 보험도 포함해서?"`

이 질문은 여러 요인에 달려 있으며, 위에서 말한 사전 정의된 기준만으로는 대응하기 어렵습니다.

즉, 미리 정해둔 워크플로우로는 한계가 있다면, 더 많은 유연성이 필요하다는 뜻입니다.

바로 이럴 때 에이전틱 설정이 도움이 됩니다.

위 예시에서, 멀티스텝 에이전트를 만들어 날씨 예보를 위한 Weather API, 이동 거리 계산을 위한 Google Maps API, 직원 가용성 대시보드, 그리고 지식베이스를 위한 RAG 시스템에 접근하도록 할 수 있습니다.

최근까지 프로그램은 미리 정해진 워크플로우에 제한되어, 복잡한 액션을 처리하기 위해 if/else 조건문을 잔뜩 쌓는 방식에만 머물러 있었습니다. 이런 프로그램들은 “이 숫자들의 합을 계산하라”거나 “이 그래프에서 최단 경로를 찾아라” 같은 극도로 좁은 과제에 집중했습니다. 하지만 위의 여행 예시 같은 실생활 과제들은 미리 정해둔 워크플로에 잘 들어맞지 않습니다. 이런 관점에서 에이전틱 시스템은 프로그램에게 실생활의 다양한 과제를 연결해주는 열쇠입니다.

## **Code agents 코드 에이전트**

멀티스텝 에이전트에서는 각 단계마다 LLM이 **외부 도구를 호출하는 형태의 동작**을 작성할 수 있습니다.

Anthropic, OpenAI 등에서 쓰이는 일반적인 형식은 “도구 이름과 사용할 인자를 JSON 형태로 작성 → 이를 파싱해서 어떤 도구를 어떤 인자로 실행할지 결정”하는 방식입니다.

하지만 [**여러**](https://huggingface.co/papers/2402.01030) [**연구**](https://huggingface.co/papers/2411.01747) [**논문**](https://huggingface.co/papers/2401.00812)들은 **코드로 직접 도구 호출을 작성하는 방식이 훨씬 더 낫다**는 것을 보여주었는데, 이유는 간단합니다.

*우리는 컴퓨터가 수행할 동작을 표현하기 위해 가장 최적화된 방법으로 프로그래밍 언어를 만들어 왔기 때문입니다.*

만약 JSON이 더 나은 표현 수단이었다면, JSON이 최고의 프로그래밍 언어가 되었을 것이고 프로그래밍은 지옥이 되었을 겁니다.

아래 그림은 [**Executable Code Actions Elicit Better LLM Agents**](https://huggingface.co/papers/2402.01030) 논문에서 가져온 것으로, 동작을 코드로 작성했을 때의 장점을 보여줍니다.

![](https://huggingface.co/datasets/huggingface/documentation-images/resolve/main/transformers/code_vs_json_actions.png)

코드로 동작을 작성하는 것은 JSON과 같은 형식보다 이러한 장점을 가집니다:

- **조합성(Composability):** JSON 동작을 서로 중첩하거나, 재사용할 JSON 동작을 정의할 수 있는지? → 파이썬 함수로는 아주 쉽게 할 수 있습니다.
- **객체 관리(Object management):** `generate_image` 같은 함수/동작의 출력을 JSON에 어떻게 저장할지?
- **일반성(Generality):** 코드는 컴퓨터가 할 수 있는 모든 작업을 단순하게 표현하기 위해 설계되었습니다.
- **LLM 학습 데이터에서의 표현력:** 이미 LLM 학습 데이터에 양질의 코드 동작들이 다수 포함되어 있어, LLM이 이미 코드 기반 동작 작성에 훈련되어 있습니다.

## ***smolagents 소개*: 간단하게 에이전트 만들기 🥳**

우리는 다음 목표로 [**`smolagents`**](https://github.com/huggingface/smolagents)를 만들었습니다:

✨ **단순함**: 에이전트 로직은 약 수천 줄 정도의 코드에 담겨 있습니다([참고](https://github.com/huggingface/smolagents/blob/main/src/smolagents/agents.py)). 불필요한 추상화는 최소화하고, 원시 코드 위에 꼭 필요한 부분만 남겼습니다!

🧑‍💻 **코드 에이전트(Code Agents)에 대한 일급 지원 :** 동작을 코드로 작성하는 에이전트(“코드를 작성하는 데 쓰이는 에이전트”와는 다름)를 지원합니다. 보안을 위해 [**E2B**](https://e2b.dev/)를 통한 샌드박스 환경에서 실행할 수 있도록 했습니다.

- 이 [**`CodeAgent`**](https://huggingface.co/docs/smolagents/reference/agents#smolagents.CodeAgent) 클래스 위에,  JSON/text 블롭으로 동작을 작성하는 표준 [**`ToolCallingAgent`**](https://huggingface.co/docs/smolagents/reference/agents#smolagents.ToolCallingAgent)도 지원합니다.

🤗 **Hub 통합**: 도구를 허브에 공유하거나 불러올 수 있으며, 앞으로 더 많은 기능이 추가될 예정입니다!

🌐 **모든 LLM 지원**: 허브에 호스팅된 모델을 `transformers` 버전이나 추론 API를 통해 불러올 수 있을 뿐 아니라, OpenAI, Anthropic 등 다양한 모델도 [**LiteLLM](https://www.litellm.ai/)** 통합을 통해 지원합니다.

[**`smolagents`**](https://github.com/huggingface/smolagents) 는 [**`transformers.agents`](https://huggingface.co/blog/agents)** 의 후속 라이브러리로, 앞으로[**`transformers.agents`**](https://huggingface.co/blog/agents) 는 더이상 지원되지 않을 예정입니다.

# **Agent 만들기**

 에이전트를 만들기 위해 필요한 것은 최소 두가지 입니다 :

- `tools`: 에이전트가 접근 가능한 도구 목록
- `model`: 에이전트의 엔진이 되어줄 LLM

 `model` 의 경우, Hugging Face의 무료 추론 API를 활용하는`HfApiModel` 클래스를 사용해 오픈 모델을 쓸 수도 있고(위의 leopard 예시 참고), [**litellm**](https://github.com/BerriAI/litellm)을 활용하는 `LiteLLMModel` 을 사용해 100개 이상의 다양한 클라우드 LLM 중에서 선택할 수도 있습니다.

`tool`의 경우, 아래 두가지를 추가한 함수를 만든 뒤 `@tool` 데코레이터를 붙이면 도구로 기능합니다.

- 입력과 출력에 타입 힌트
- 입력에 대한 설명을 담은 docstring

Google Maps에서 이동 시간을 가져오는 **커스텀 도구**를 만들어, 이 도구를 활용하는 여행 계획 에이전트를 만들어 보겠습니다 :

```python
from typing import Optional
from smolagents import CodeAgent, HfApiModel, tool

def get_travel_duration(start_location: str, destination_location: str, transportation_mode: Optional[str] = None) -> str:
    """두 장소 사이의 이동 시간을 가져옵니다.

    Args:
        start_location: 출발지
        destination_location: 도착지
        transportation_mode: 이동 수단.'driving', 'walking', 'bicycling','transit'중 하나이며, Defaults값은 'driving'.
    """
    import os   *# 모든 import는 Hub 공유를 위해 함수 안에 배치합니다. **import googlemaps
    from datetime import datetime

    gmaps = googlemaps.Client(os.getenv("GMAPS_API_KEY"))

    if transportation_mode is None:
        transportation_mode = "driving"
    try:
        directions_result = gmaps.directions(
            start_location,
            destination_location,
            mode=transportation_mode,
            departure_time=datetime(2025, 6, 6, 11, 0), *# 6월 6일 오전 11시로 설정*
        )
        if len(directions_result) == 0:
            return "요청한 교통 수단으로는 이 두 장소 사이 경로를 찾을 수 없습니다."
        return directions_result[0]["legs"][0]["duration"]["text"]
    except Exception as e:
        print(e)
        return e

agent = CodeAgent(tools=[get_travel_duration], model=HfApiModel(), additional_authorized_imports=["datetime"])

agent.run("파리 주변에서 하루 동안 자전거로 여행할 수 있는 코스를 짜줄래? 시내든 외곽이든 상관없지만 하루 안에 끝나야 해. 나는 대여한 자전거만 타고 다녀.")
```

몇 단계 동안 이동 시간을 수집하고 계산을 실행한 뒤, 에이전트는 최종적으로 다음 일정을 제안했습니다:

```markdown
당일치기 파리 자전거 여행 일정:
1. 오전 9:00 에펠탑에서 출발
2. 오전 10:30까지 에펠탑 관광
3. 오전 10:46 노트르담 대성당으로 이동
4. 오후 12:16까지 노트르담 대성당 관광
5. 오후 12:41 몽마르트르로 이동
6. 오후 2:11까지 몽마르트르 관광
7. 오후 2:33 뤽상부르 공원으로 이동
8. 오후 4:03까지 뤽상부르 공원 관광
9. 오후 4:12 루브르 박물관으로 이동
10. 오후5:42까지 루브르 박물관 관광
11. 오후 6:12까지 저녁 식사
12. 종료 예정 시간: 오후 6:12
```

도구를 만든 뒤, Hub에 공유하는 방법은 매우 간단합니다:

```markdown
get_travel_duration.push_to_hub("{your_username}/get-travel-duration-tool")
```

결과는 [**이 스페이스**](https://huggingface.co/spaces/m-ric/get-travel-duration-tool)에서 확인할 수 있습니다. 도구의 로직은 [**스페이스 내 tool.py**](https://huggingface.co/spaces/m-ric/get-travel-duration-tool/blob/main/tool.py)에서 볼 수 있습니다.

보시다시피, 이 도구는 실제로 [**`Tool`**](https://huggingface.co/docs/smolagents/reference/tools#smolagents.Tool)클래스를 상속한 클래스로 내보내졌으며, 이는 모든 도구의 기본 구조입니다.

# **에이전틱 워크플로우에서 오픈 모델은 얼마나 강력할까?**

여러 최신 모델들로 [**`CodeAgent`**](https://huggingface.co/docs/smolagents/reference/agents#smolagents.CodeAgent) 인스턴스를 만들고, 다양한 벤치마크에서 질문을 모아 여러 유형의 과제를 제공하는 [**벤치마크**](https://huggingface.co/datasets/m-ric/agents_medium_benchmark_2)로 비교했습니다.

에이전트 설정에 대한 자세한 내용은 [**여기 벤치마크 노트북**](https://github.com/huggingface/smolagents/blob/main/examples/benchmark.ipynb)에서 확인할 수 있으며, 코드 에이전트와 툴 호출 에이전트의 성능 비교도 볼 수 있습니다.

(스포일러: **코드 방식이 더 잘 작동합니다.**)

![benchmark of different models on agentic workflows](https://huggingface.co/datasets/huggingface/documentation-images/resolve/main/blog/smolagents/benchmark_code_agents.png)

오픈소스 모델이 최고의 Closed 모델과 붙을 수준임을 볼 수 있습니다! 

## **Next steps 🚀**

- [**가이드 투어**](https://huggingface.co/docs/smolagents/guided_tour)로 smolagents 라이브러리를 익혀보세요
- 심화 튜토리얼로 [**도구**](https://huggingface.co/docs/smolagents/tutorials/tools) 사용법이나 [**일반적인 예제**](https://huggingface.co/docs/smolagents/tutorials/building_good_agents) 들을 학습하세요.
- 예제를 살펴보며 특정 시스템을 설정해 보세요: [**text-to-SQL**](https://huggingface.co/docs/smolagents/examples/text_to_sql), [**에이전틱 RAG**](https://huggingface.co/docs/smolagents/examples/rag), [**멀티 에이전트 오케스트레이션**](https://huggingface.co/docs/smolagents/examples/multiagents).
- 에이전트에 대해 더 읽어보기:
    - Anthropic [**블로그 포스팅](https://www.anthropic.com/research/building-effective-agents)  :** 탄탄한 기본 개념에 대해 알 수 있습니다.
    - [**컬렉션](https://huggingface.co/collections/m-ric/agents-65ba776fbd9e29f771c07d4e) :** 에이전트 관련 가장 영향력 있는 연구 논문들을 모아둔 자료입니다.
