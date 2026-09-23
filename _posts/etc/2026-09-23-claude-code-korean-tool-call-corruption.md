---
title: Claude Code 도구 호출에서 한글이 깨지는 이유와 우회 방법
author: hkh7670
date: 2026-09-23 00:00:00 +0900
categories: [etc]
tags: [Claude Code, Unicode, 한글]
---

한글이 깨지면 보통 파일 인코딩이나 터미널 설정부터 확인하게 된다. 그런데 Claude Code에서 도구를 호출할 때 한글이 이상해지는 현상 중에는 조금 다른 사례가 있다. 모델이 한글을 유니코드 이스케이프로 쓰면서 코드 값을 잘못 생성하는 경우다.

[관련 LinkedIn 게시물](https://www.linkedin.com/posts/woohyungchoi_bug-sonnet-5-writes-korean-tool-call-parameters-share-7489171964033425408-kZnF/)을 계기로, 같은 제목의 [Claude Code GitHub 이슈 #83033](https://github.com/anthropics/claude-code/issues/83033)을 확인했다. 아래 내용은 해당 이슈의 실험 보고를 바탕으로 정리했다. 직접 재현한 결과는 아니다.

### 어떤 증상인가

이슈에서 다루는 대상은 Sonnet 5가 생성한 도구 호출 인자다. `AskUserQuestion`의 질문이나 `TodoWrite`의 작업 내용에 엉뚱한 한글 음절이 섞이는 현상이 보고됐다.

글자가 아예 표시되지 않는 것과는 다르다. 한글로 읽히기는 하지만, 단어를 이루는 일부 글자가 다른 글자로 바뀐다. 보고자는 Claude Code를 거치지 않는 Bedrock API 호출에서도 이를 재현했다고 설명한다. 이 사례를 터미널 표시 문제만으로 보기는 어려운 이유다. [실험 보고](https://github.com/anthropics/claude-code/issues/83033)

### JSON은 정상인데 내용은 틀릴 수 있다

JSON 문자열에서 한글은 그대로 쓸 수도 있고, `\uXXXX` 형식으로 표현할 수도 있다. 아래 두 JSON은 파싱하면 같은 값을 갖는다.

```json
{"message": "가"}
```

```json
{"message": "\uac00"}
```

이스케이프 자체가 문제는 아니다. 정확한 코드 값을 쓰면 된다.

다만 `\uac00`을 `\uac01`로 쓰면 이야기가 달라진다. 둘 다 유효한 JSON이지만, 후자는 `가`가 아니라 `각`이다. 파서는 작성자가 어떤 글자를 의도했는지 알 수 없으므로 오류 없이 받아들인다.

다음은 이 차이를 확인하기 위한 Python 예제다. Claude의 출력을 재현하는 코드는 아니다.

```python
import json

literal = json.loads('{"message": "가"}')
escaped = json.loads(r'{"message": "\uac00"}')
wrong = json.loads(r'{"message": "\uac01"}')

print(literal == escaped)  # True
print(escaped["message"])  # 가
print(wrong["message"])    # 각
```

이렇게 다른 글자가 들어간 문자열은 UTF-8로 다시 저장해도 그대로다. 이미 `각`이라는 유효한 문자가 되었기 때문이다. 파일 인코딩을 바꾸는 작업으로 원래 의도한 `가`를 되찾을 수는 없다.

### 보고된 실험 결과

이슈 작성자는 도구 인자의 한글을 이스케이프로 쓰도록 한 조건과, 한글 그대로 쓰도록 한 조건을 비교했다.

- 실제로 이스케이프를 사용한 45회에서는 모두 한글 손상이 관찰됐다.
- 한글 그대로 쓰도록 한 조건에서는 같은 유형의 손상이 관찰되지 않았다. 다만 별개 유형의 오류 1건은 있었다.

여기서 45회 모두라는 수치는 **해당 실험에서 실제로 이스케이프를 사용한 실행**에 대한 결과다. Sonnet 5의 모든 한글 응답이 깨진다는 뜻은 아니다. [실험 조건과 결과](https://github.com/anthropics/claude-code/issues/83033)

### CLAUDE.md에 추가할 지침

이슈에서 제안한 우회 방법은 도구 호출 인자에 한글을 그대로 쓰도록 지시하는 것이다. 다음은 그 취지를 옮긴 예시다.

```markdown
도구 호출 인자에 들어가는 한글은 글자 그대로 작성한다.
한글을 유니코드 이스케이프(\uXXXX)로 변환해서 작성하지 않는다.
```

프로젝트에 적용하려면 루트의 `CLAUDE.md`에 추가한다. 여러 프로젝트에서 개인 지침으로 사용하려면 `~/.claude/CLAUDE.md`에 넣을 수 있다. 기존 파일이 있다면 내용을 유지하고 위 지침을 덧붙이면 된다. 파일 위치별 적용 범위는 [Claude Code 공식 문서](https://code.claude.com/docs/en/memory#choose-where-to-put-claude-md-files)에 나와 있다.

이 지침은 모델의 출력 방식을 유도하는 문장이다. 프로그램이 강제하는 인코딩 설정은 아니며, 공식 문서도 `CLAUDE.md`를 강제 설정이 아닌 문맥으로 설명한다. [CLAUDE.md 동작 설명](https://code.claude.com/docs/en/memory#claude-md-vs-auto-memory)

또한 이슈 작성자는 이스케이프를 쓰지 않아도 드물게 음절이 바뀌는 별도 현상을 보고했다. 따라서 이 방법을 모든 한글 깨짐의 해결책으로 볼 수는 없다. [우회 방법과 남은 문제](https://github.com/anthropics/claude-code/issues/83033)

도구 호출이 성공했다고 해서 그 안의 문장까지 정확한 것은 아니다. 한글 문서나 메시지를 생성하는 작업이라면, 실행 성공 여부와 함께 실제로 저장된 내용도 확인하는 편이 좋겠다.
