---
layout: post
title: "libpci python wrapper를 만들자 (0)"
description: ""
date: 2017-07-26
tags: [python, cffi, libpci, pycparser]
comments: true
---

1. [libpci python wrapper를 만들자 (0)][1]
2. [libpci python wrapper를 만들자 (0-1)][2]

[1]: {{site.url}}/2017-07-26/how-to-make-libpci-python-wrapper-0
[2]: {{site.url}}/2017-07-28/how-to-make-libpci-python-wrapper-0-1

## 들어가기 전에

Python은 요즘 딥러닝이 인기를 얻으면서 다시 한번 핫해진 세상에서 가장 핫한 프로그래밍 언어 중
하나다. Python의 장점은 정말 다양하겠지만, 그 중에서도 특히 다른 언어와의 연계성은 python을 지금
위치에 올려놓는 데에 큰 영향력을 주었다는 데에 반대할 사람은 별로 없을 것이다. 사실상의 표준으로
작용하고 있는 Python의 C 언어 구현체인 CPython은 C 언어로 Python 확장을 구현하여 Python에서
호출하거나, 반대로 Python 모듈을 C에서 가져오거나, Python 함수를 C에서 호출할 수 있는 규약을
제공하고 있다. 다만, Python은 동적 타입 언어이며 C에 비해 다양한 형태의 자료형을 사용할 수 있기
때문에 규약을 숙지하고 사용하기가 그렇게 간단하지만은 않다.

이러한 문제를 해결하기 위해 여러 가지 라이브러리들이 제안되었는데, 이 중 Python의 표준 라이브러리에
포함되어 있는 `ctypes` 라이브러리의 경우, 이미 빌드된 so 파일이나 dll 파일을 불러들여 OS에서
제공하는 동적 링크 방법을 통해서 심볼을 찾고 함수를 가져와 사용할 수 있게 해주는 일련의 방법들을
제공한다. 간단한 정수형이나 스트링 등의 자료형은 특별한 조작 없이 자동으로 C에서 사용 가능한 형태로
변환해서 넘겨주며, 구조체도 정해진 방법대로 정의하면 사용 가능하고 필요에 따라 함수의 prototype을
적절히 지정해주는 것도 가능하여 일반적인 경우에는 별 탈 없이 편리하게 사용이 가능하다.

다만 `ctypes`의 문제점은,

1. so, dll 등 OS에서 지원하는 동적 링크 라이브러리 형태로 빌드된 경우에만 사용이 가능하며
2. 노출되지 않은/할 수 없는 심볼은 사용이 불가능하고
3. 기존 C header 파일을 손으로 직접 포팅하여 ctypes에 맞는 형태로 재정의가 필요하여

제대로 쓰려면 손이 많이 가는 단점이 있다.

이에 비해 [`cffi`][cffi]는 C 문법으로 구조체나 함수 등을 선언할 수 있고, 자동으로 CPython이나 PyPy의
C 확장 모듈용 소스 코드를 생성하여 빌드하는 형태라 함수나 변수의 형태로 사용할 수 있는 어떤 형태의
코드도 모두 노출이 가능하다. 이런 노출 가능한 심볼의 예로는 `#define` 을 통해 정의된 매크로 상수나 
매크로 함수 등이 있다.

그러나 [`cffi`][cffi]도 단점이 없는 것은 아닌데, 확장 모듈을 만들 때는 gcc나 MSVC 등의 적절한
컴파일러를 사용하여 빌드하지만 구조체나 함수 선언시에는 외부 컴파일러나 C 언어 파서를 사용하는 것이
아니라 [`pycparser`][pycparser] 모듈을 이용해 처리하다 보니 preprocessor나 각 컴파일러별 확장 기능
등이 거의 지원이 되지 않는다는 문제가 있다. C 언어의 형태로 함수를 선언하지만, 헤더파일을 있는 그대로
사용하는 것은 어렵고 결국 사람의 손을 타야 제대로 동작할 수 있다는 것이다.

Python에서 `libpci`를 사용할 필요가 생겨 포팅하기로 마음을 먹고, 좀 더 깔끔하게 포팅하기 위해
[`cffi`][cffi]를 사용하는 것 까지는 좋은데, 최대한 사람 손을 타지 않고 바인딩을 만드려면 위에서
언급한 문제를 해결해야 한다. 그래서, wrapper를 만들기 전에 먼저 C에서 사용하는 헤더파일을 분석하고,
함수의 프로토타입과 상수들을 추출하여 [`cffi`][cffi] 에 던져넣기만 하면 되는 형태로 정리해줄 추가적인
라이브러리가 필요하게 되었다.

그래서 이 시리즈를 연재하기 전에 먼저 [`pycparserlibc`][pycparserlibc] 라이브러리에 대해 소개하고
코드를 정리해보자는 의미에서 0번 글이 되었다.

[cffi]: https://cffi.readthedocs.io/
[pycparser]: https://github.com/eliben/pycparser
[pycparserlibc]: https://github.com/gwangyi/pycparserlibc


## `pycparserlibc.cpp.Preprocessor`

`pycparserlibc`에서 해 주어야 하는 일은 다음과 같다.

1. Preprocessing
2. 표준 C 라이브러리에서 지원해주는 헤더파일들에 대한 처리
3. 표준 C 라이브러리에서 제공하는 다양한 매크로들에 대한 처리
4. 표준 C 라이브러리에서 미리 정의된 타입들에 대한 처리

`cffi`에서 `pycparser`를 쓸 때는 preprocessor를 제외하고 사용하기 때문에 제일 먼저 해야할 일은
preprocesor를 찾는 일이다.

다행히도, `pycparser`에서도 가져다 쓰는 [`ply`][ply] 라이브러리에 순수 파이썬 C preprocessor 구현이
번들되어 있다. 그런데 그냥 가져다 쓰면 정의되지 않은 `lex` 변수를 왜인지 참조하고 있어 실행이
안되므로 수정해줄 필요가 있다.

`pycparser` 소스트리를 보면 `utils/fake_libc_include` 안에 libc를 사용하는 C 소스를 파싱할 때 쓸 수
있는 fake 헤더파일들이 들어 있는 것을 볼 수 있다. 본체는 `_fake_defines.h`, `_fake_typedefs.h` 두
파일이며 나머지 헤더파일은 모두 더미로, 이 두 헤더파일을 include하는 것이 전부임을 볼 수 있다.
`pycparserlibc`에서는 이 더미 헤더파일이 include될 경우 fake 파일들을 사용하는 것으로 구현하면 libc
를 잘 처리할 수 있을 것이다.

새로운 클래스를 하나 만들어 `ply.cpp.Preprocessor`를 상속받는다. `lex` 문제를 해결하기 위해
`__init__`를, libc 헤더파일 처리를 위해 `#include`를 처리하는 `include`를 오버라이드한다.

먼저 `__init__`는 상대적으로 간단하게 해결할 수 있다.

```python
from ply import cpp, lex

class Preprocessor(cpp.Preprocessor):
    def __init__(self, lexer = None):
        if lexer is None:
            lexer = lex.lex(module=cpp)
        super().__init__(lexer)
```

lexer를 따로 지정하지 않은 경우 디폴트로 `cpp` 모듈에 정의된 lexer를 사용하도록 했다.

`include`의 경우 가능하다면 파일을 열 때 파일 이름을 체크하여 fake에 해당하는 경우 스킵하도록 하는
것이 가장 좋겠지만, 함수 구현상 그런 식으로 만들기가 어려워 `ply.cpp.Preprocessor.include` 내용을
복붙한 후 일부 수정을 가하는 선에서 마무리했다. `#include <...>` 문법으로 include했을 경우로 한정해
fake 리스트에 있는 헤더파일을 include 하면 그대로 리턴하는 방법으로 구현했다. 자세한 코드는
[`pycparserlibc`][pycparserlibc] github을 참조하기 바란다.


## Fake

위에서 libc 헤더를 단순히 스킵하는 방식으로 구현했으므로, `_fake_defines.h`나 `_fake_typedefs.h`의
내용을 똑같이 처리해줄 필요가 있다. 그러기 위해서 먼저 더미로 대체할 소스파일 이름, 미리 정의될
매크로의 이름과 내용, 미리 정의될 `typedef`들을 모아놓을 필요가 있는데, 이를 `pycparserlibc/fake.py`
안에 모았다. 파일 이름은 `utils/fake_libc_include` 안의 파일들의 이름으로 했고, 매크로와 `typedef`
들을 모두 수집했다.

매크로는 `ply.cpp.Preprocessor`에서 `define` 메소드로 추가로 정의할 수 있는데, 이 때는 공백문자들로
구분된 이름과 내용의 쌍을 받으므로 바로 넣을 수 있게 `"NULL 0"` 과 같은 형태로 정의해주었다.
`typedef`는 헤더파일 내용을 거의 그대로 옮긴 소스파일을 하나 준비하여 그대로 넣는다.

## Line directives

C 코드를 디버깅하려면 문제가 되는 코드가 어떤 파일의 몇 번째 줄에 있는지를 반드시 알아야 한다. 단일
소스 코드일 때는 그런 정보를 얻는 것이 어렵거나 복잡하지 않지만, `#include` 등을 통해 여러 파일이
섞이기라도 했으면 별도로 줄 번호를 붙여주는 것이 필요하다. `cpp`와 같은 일반적인 C preprocessor는
`#include` 등을 통해 다른 파일이 삽입이 되면 그런 줄 번호를 알아서 붙여주지만,
`ply.cpp.Preprocessor`는 안타깝게도 그런 동작을 전혀 해주지 않는다.

아이디어는 다음과 같다. 매번 토큰을 반환해주는 제네레이터에서 반환되는 토큰을 모니터링 하다가,
토큰이 반환된 후 `__FILE__` 매크로 값이 변경되면 해당 토큰보다 먼저 line directive의 토큰을 반환시켜
줄 번호를 달아주면 될 것이다.

토큰을 반환시켜주는 함수는 `ply.cpp.Preprocessor.parsergen()` 함수이므로 이를 override한다. 그 다음
마지막으로 사용된 `__FILE__`값을 저장하고 있는 멤버 변수를 만들어서, 원래 함수가 호출된 뒤 그 값이
변경되었으면 line directive를 출력해 줄 번호와 파일 이름을 잘 따라가도록 한다.

Line directive는 C99에서 다음과 같이 정의하고 있다.

```C
# line digit-sequence "s-char-sequence_opt" new-line
```

`s-char-sequence_opt`는 옵셔널한 값으로 소스코드 이름을 지정해줄 수 있다. 즉, `__FILE__` 값을
변경하는데 사용한다. digit-sequence는 바로 다음 행의 행 번호를 지정해준다. 즉, `__LINE__` 값을
변경할 수 있다. 표준에서는 이 행 번호는 0이 될 수 없다고 한다.

그래서 다음과 같이 `parsegen` 함수를 override하면 되겠다.

```python
    def parsegen(self, input, source=None):
        for tok in super().parsegen(input, source):
            if self._prev_source != self.source:
                self._prev_source = self.source
                for t in self.tokenize(f'# {tok.lineno} "{self.source}"'):
                    yield t
            yield tok
```

`self._prev_source` 는 적절히 초기화해주면 될 것이다.

## 조립하기

`ply.cpp.Preprocessor` 클래스에는 `add_path` 함수와 `define` 함수가 있다. `define`은 위에서 언급한
것 처럼 매크로를 정의할 때 쓰는 함수이고, `add_path` 함수는 `#include` 할 때 참조할 디렉토리를 추가
할 때 쓸 수 있다. 즉 `cpp`의 옵션으로 `-I`, `-D` 가 들어온 경우 두 함수로 처리할 수 있다.

libc 를 fake 할 때는 위의 Fake 문단에서 만들었던 fake macro 들을 `define` 함수로 정의해준 뒤
`ply.cpp.Preprocessor` 문단에서 만든 preprocessor 로 처리해주면 preprocessing을 수행할 수 있다.
거기에 보태서, fake하기 위해 만든 매크로는 사용자 입장에서는 큰 관심이 없으므로 preprocessing 후
정의된 매크로 목록에서 빼주면 좀 더 유용하다.

Preprocessing 후 C 언어 parsing을 할 때는, fake에서 수집한 `typedef`들을 소스코드 머리에 붙인 뒤
파싱을 진행해야 한다. 처음부터 type check list에 넣으면 좋지만, 그게 잘 안돼서 소스코드에 삽입시켜
처리했다. 대신 맨 마지막에 미리 정해놓은 코드를 붙여서, 파싱된 결과에서는 정해놓은 코드가 나타나기
전까지는 모두 무시하도록 처리해 주었다.

조립한 코드는 `pycparserlibc/__init__.py` 안에 들어있다.

## 사용법

`pycparserlibc.preprocess` 함수로 preprocessing 한 뒤 `pycparserlibc.parse` 함수에 preprocessing 된
내용물을 넣고 파싱하면 파싱된 AST 객체가 반환된다. `pycparserlibc.preprocess` 에 `fake_defs=True`
옵션을 넣고 `pycparserlibc.parse` 에 `fake_typedefs=True` 옵션을 넣으면 libc 참조하는 코드도 에러
없이 파싱할 수 있다.

```python
import pycparserlibc

header = pycparserlibc.preprocess(r"""
typedef uint8_t u8;
typedef uint16_t u16;
typedef uint32_t u32;
typedef uint64_t u64;
#include <pci/pci.h>
""", fake_defs=True, cpp_args=["-I/usr/include", "-DPCI_HAVE_Uxx_TYPES"])

ast = pycparserlibc.parse(header, 'pypci.h', fake_typedefs=True)

print(ast)
```

## 수정 내역

* 2017-07-28
  - 맨 처음에 관련글 목록을 추가
* 2017-07-27
  - Line directive에 대한 언급 추가
  - 예제에 `cpp_args` 항목 추가
