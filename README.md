## 🤖 LLM 튜터 API 가이드라인

1. 아키텍처 개요

본 API는 **Azure Functions(HTTP Trigger)**를 기반으로 구축하며, Azure OpenAI Service와 상호작용합니다. 문맥 정보 주입(RAG)을 위해 Azure AI Search를 벡터 데이터베이스로 활용합니다.

    데이터 흐름

    클라이언트(App/Web) ↔ Azure Function API ↔ Azure AI Search (RAG) ↔ Azure OpenAI (LLM)

2. API 엔드포인트 명세

    Base URL: https://<your-function-app>.azurewebsites.net/api

POST /tutor/hint

    설명: 학생이 막혔을 때, 정답을 노출하지 않고 다음 단계로 나아갈 수 있는 소크라틱 방식의 힌트를 제공합니다.

    Request Body:
    JSON

{
  "question_uid": "00041_42232",
  "student_answer_history": ["9", "15", "27"]
}

Success Response (200 OK):
JSON

    {
      "status": "success",
      "type": "hint",
      "content": "좋은 시도예요! 혹시 '소수'가 정확히 어떤 수인지 정의를 다시 한번 생각해볼까요?"
    }

POST /tutor/explain

    설명: 학생이 제출한 답안에 대해 정/오답 원인을 요약하고, 핵심 개념을 설명합니다.

    Request Body:
    JSON

{
  "question_uid": "00041_42232",
  "student_answer": "9, 15, 27, 33"
}

Success Response (200 OK):
JSON

    {
      "status": "success",
      "type": "explanation",
      "feedback": "거의 다 맞았어요! 하지만 9, 15, 27, 33은 1과 자기 자신 외에 다른 수(3 또는 5 등)로도 나누어지기 때문에 소수가 아니랍니다.",
      "core_concept": "소수는 1보다 큰 자연수 중 1과 자기 자신만을 약수로 가지는 수입니다."
    }

3. 핵심 구현 전략

가. RAG (Retrieval-Augmented Generation) 컨텍스트 주입

    사전 작업 (Indexing): silver.question_dim의 모든 question_text를 Azure OpenAI의 임베딩 모델(text-embedding-ada-002)을 사용하여 벡터로 변환하고, question_uid와 함께 Azure AI Search에 저장합니다.

    실시간 검색 (Retrieval): API 요청이 오면, question_uid에 해당하는 질문 텍스트를 임베딩하여 Azure AI Search에 유사도 검색을 요청합니다. Top-K(예: 3개) 유사 문항의 텍스트를 검색 결과로 얻습니다.

    컨텍스트 생성: LLM에 전달할 프롬프트에 **[현재 문항] + [유사 문항 텍스트 요약]**을 포함하여, 더 정확하고 풍부한 답변을 생성하도록 유도합니다.

나. 프롬프트 엔지니어링 (Prompt Engineering)

LLM이 원하는 역할을 정확히 수행하도록 시스템 프롬프트에 명확한 규칙을 설정해야 합니다.

    시스템 프롬프트 (예시):

    You are 'Math-Bot', a friendly and patient middle school math tutor.
    Your goal is to help students think for themselves.

    CRITICAL RULES:
    1. NEVER give the final numerical answer or list the correct options directly.
    2. Always respond in the requested JSON format.
    3. Keep your language encouraging and easy for a 14-year-old to understand.

다. 안전장치 구현

    JSON 스키마 출력 강제: Azure OpenAI 호출 시, JSON 모드를 활성화하고 프롬프트에 원하는 출력 JSON 구조를 명시하여 응답 형식을 강제합니다.

    폴백(Fallback) 로직: LLM API 호출이 실패하거나, 응답이 JSON으로 파싱되지 않거나, 응답 시간이 2초를 초과할 경우, 미리 준비된 규칙 기반(Rule-based) 힌트를 반환합니다.

        예: silver.question_hints 테이블에 question_uid별로 "이 문제는 소수의 정의를 묻는 문제입니다." 와 같은 기본 힌트를 저장해 둡니다.

4. DoD (Definition of Done) 달성 방안

    ✅ P95 응답 ≤ 2.5s:

        Azure Function은 프리미엄 플랜을 사용하여 Cold Start 최소화.

        Azure AI Search의 인덱스를 최적화하여 벡터 검색 속도 확보.

        LLM 응답 생성 시 max_tokens를 제한하여 불필요하게 긴 답변 방지.

    ✅ JSON 스키마 파싱 100% 성공:

        Azure OpenAI의 JSON 모드 활용.

        JSON 파싱 실패 시, 즉시 규칙 기반 폴백 로직으로 전환하여 에러 대신 정상 응답 반환.

    ✅ 금지어/정답 누설 0건:

        시스템 프롬프트에 강력한 제약 조건 명시.

        (선택) LLM 응답을 사용자에게 보내기 전, Azure Function 코드에서 정답에 해당하는 숫자나 금지어(예: '정답은')가 포함되어 있는지 마지막으로 스캔하는 로직 추가.

---

## 🏁 최종 결과물: 실시간 상호작용 API 엔드포인트

이 프로세스의 최종 결과물은 정적인 데이터 테이블이 아니라, '실시간으로 학생과 상호작용하는 지능형 API 엔드포인트' 입니다.

쉽게 비유하자면, 우리는 '똑똑한 AI 웨이터(Azure Function)' 를 만드는 것입니다. 학생이라는 손님이 질문(Request)을 하면, AI 웨이터는 주방장(LLM)과 레시피북(AI Search)을 참고하여, 손님에게 가장 적절한 답변(Response)을 서빙하는 역할을 합니다.

🙋‍♂️ 사용자 경험 예시: '힌트' 기능의 작동 방식

학생이 문제를 풀다가 막혀서 앱에서 '힌트' 버튼을 누르는 순간부터의 전체 흐름과 최종 결과물은 다음과 같습니다.

1. 클라이언트 (학생의 앱/웹)

    학생이 '힌트' 버튼을 누르면, 앱은 우리가 배포한 Azure Function API에 다음과 같은 요청을 보냅니다.

        요청: POST https://<우리-함수앱-주소>.azurewebsites.net/api/tutor/hint

        내용(Body):
        JSON

        {
          "question_uid": "00041_42232",
          "student_answer_history": ["9", "15", "27"]
        }

2. Azure Function (우리가 만든 백엔드)

    요청을 받은 Azure Function은 다음과 같이 동작합니다.

        question_uid를 사용하여 silver.question_dim에서 문제 원본 텍스트를 조회합니다.

        Azure AI Search에 접속하여 해당 문제와 유사한 문제 Top 3개를 검색합니다 (RAG).

        이 모든 정보(원본 문제, 유사 문제, 학생의 오답 기록)를 종합하여 Azure OpenAI의 LLM(주방장) 에게 전달할 정교한 프롬프트(요리 주문서)를 만듭니다.

        LLM으로부터 답변을 받습니다.

3. 최종 API 응답 (백엔드가 앱에게 보내는 답변)

    Azure Function은 LLM이 생성한 답변을 가이드라인에 따라 검증하고(정답 노출 금지 등), 최종적으로 다음과 같은 구조화된 JSON 형식으로 클라이언트(앱)에게 응답을 보냅니다.

        최종 결과물 (JSON Response):
        JSON

        {
          "status": "success",
          "type": "hint",
          "content": "좋은 시도예요! 혹시 '소수'가 정확히 어떤 수인지 정의를 다시 한번 생각해볼까요?"
        }

4. 클라이언트 (학생의 앱/웹)

    앱은 이 JSON 응답을 받아서, content 부분의 텍스트("좋은 시도예요!...")를 추출하여 학생의 화면에 예쁜 채팅 말풍선이나 팝업 형태로 보여줍니다.

결론적으로, 이 프로세스가 모두 배포되었을 때의 최종 산출물은 '요청에 따라 지능적인 JSON 답변을 실시간으로 제공하는 백엔드 API' 입니다. 프론트엔드 개발자는 이 API가 주는 구조화된 답변을 활용하여 사용자에게 다양한 형태의 피드백(힌트, 설명, 격려 메시지 등)을 보여줄 수 있게 됩니다.
