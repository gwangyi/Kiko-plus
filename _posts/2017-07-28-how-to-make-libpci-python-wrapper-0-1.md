---
layout: post
title: "libpci python wrapper를 만들자 (0-1)"
description: ""
date: 2017-07-28
tags: [python, cffi, libpci, pycparser]
comments: true
---

1. [libpci python wrapper를 만들자 (0)][1]
2. [libpci python wrapper를 만들자 (0-1)][2]

[1]: {{site.url}}/2017-07-26/how-to-make-libpci-python-wrapper-0
[2]: {{site.url}}/2017-07-28/how-to-make-libpci-python-wrapper-0-1

## 들어가기 전에

[지난 이야기][1]에서 먼저 일반적인 C 헤더파일을 `cffi`에서 사용 가능한 형태로 만들기 위한 전 단계로
외부 프로그램 없이 파이썬 내부에서 preprocessing 하는 방법에 대해 알아보았다. 원래 계획대로라면 이제
`libpci`의 헤더파일을 처리하여 native extension을 만들고 좀 더 pythonic하게 래퍼를 구현해가는 단계를
시작해야겠지만, preprocessing과 parsing만으로는 여전히 `cffi.FFI.cdef`에 전달하기엔 부족하다.

따라서 이번에는 앞에서 분석한 AST(Abstract Syntax Tree)를 바탕으로 우리가 필요로 하는 선언부만
추출해내는 작업을 시도해 보자.

## `pycparser.c_generator.CGenerator`

`pycparser` 라이브러리는 C99 표준에 맞는 C 언어 코드를 AST로 변환하거나, 이미 변환된 AST를 가지고
원하는 작업을 하는 것을 도와주는 라이브러리이다. C99 표준에 맞지 않는 코드는 제대로 처리하지 않고
에러를 뱉는데, 일반적인 C 언어 코드들은 순수 C99만을 쓰는게 아니다 보니 [지난번][1] 글을 통해서
확장을 약간 시켰던 것이다.

이미 파싱된 AST를 다시 C 코드로 변환시키려면 `pycparser.c_generator.CGenerator` 클래스를 사용하면
된다. 저 클래스를 상속받은 뒤, `visit_XXX` 형태의 메소드를 정의하면 AST를 traversal하다가 `XXX`에
해당하는 노드 타입이 나왔을 때 해당 노드를 패러미터로 `visit_XXX` 메소드를 호출해 준다. 우리가 관심
있는 노드는 함수 선언이나 구조체 선언, `enum` 선언, `typedef`에 한정된다. 앞의 셋은 `Decl` 노드이고
`typedef`는 Typedef 노드이므로 해당 노드를 제외한 나머지 노드에 대해서는 빈 스트링을 반환하게 하면
관심있는 저 문장들만 남고 나머지는 모두 지워버릴 수 있다. 다만, 파싱한 후의 AST는 `FileAST`
타입인데 이 것까지 빈 스트링을 반환시키면 애초에 아무것도 하지 못하는 문제가 생기므로, 이 경우는
원래의 함수를 호출하게 해 주자.

```python
class CDefGenerator(pycparser.c_generator.CGenerator):
    def visit(self, node):
        if node.__class__.__name__ == 'FileAST':
            return super().visit(node)
        elif node.__class__.__name__ in ('Decl', 'Typedef'):
            return super().visit(node)
        else:
            return ''
```

그런데, 선언된 함수나 변수 등에서 굳이 `cffi.FFI.cdef`로 노출하고 싶지 않은 심볼도 있을 수 있고,
`enum`의 경우는 초기값을 지정해주는 부분이 `cffi`에서 필요하지 않기 때문에 추가적인 처리를 더 해서
반환하는 편이 기능적으로 더 유용할 것이다.

```python
class CDefGenerator(c_generator.CGenerator):
    class Extractor(c_generator.CGenerator):
        def __init__(self, exclude: Container[str]):
            super().__init__()
            self._exclude = exclude

        def visit_Decl(self, n, **kwargs):
            if n.name in self._exclude:
                return ''
            return super().visit_Decl(n)

        def visit_Enum(self, n):
            s = ['enum']
            if n.name:
                s.append(' ')
                s.append(n.name)
            if n.values:
                s.append(' {')
                s.append(', '.join(
                    f'{enumerator.name} = ...'
                    for _, enumerator in enumerate(n.values.enumerators)))
                s.append('}')
            return ''.join(s)

    def __init__(self, exclude: Container[str]=()):
        super().__init__()
        self._extractor = self.Extractor(exclude)

    def visit(self, node):
        if node.__class__.__name__ == 'FileAST':
            return super().visit(node)
        elif node.__class__.__name__ in ('Decl', 'Typedef'):
            return self._extractor.visit(node)
        else:
            return ''
```

`Decl`, `Typedef` 의 경우는 별도의 다른 generator를 통해 내용물을 처리하게 했다. 제외시키기 원하는
심볼은 `exclude` 속성을 통해 전달시키고 이를 내부 generator에 전달한다. 전달받은 내부 generator는
해당 심볼이 선언되면 빈 칸을 반환시킨다. `visit_Enum`을 통해서 `enum` 선언이 들어오면, 초기값을
컴파일러가 찾아 반환하게끔 `...` 으로 지정한 것으로 바꿨다.

## Macro constants

Preprocessing은 외부 preprocessor(ex. cpp, gcc -E)를 통해 충분히 처리 가능하지만, [지난번 글][1]에서
python 내부에서 처리하도록 했었다. 그렇게 한 이유는, 직접 preprocessing을 해야 매크로들의 이름과
값을 얻기 쉽기 때문이다. 

`ply.cpp.Preprocessor` 는 처리가 끝나면 `macros`라는 `dict`에 정의됐던 매크로를 모두 모은다. 정의된
값은 lex/yacc에 의해 tokenize된 토큰 형태로 들어 있다. 여기서 한가지 문제가 있는데, 매크로는 타입이
없지만 `cffi.FFI.cdef`에서 정의되는 상수는 타입을 가져야 한다는 것이다.  `ply.cpp.Preprocessor`의
`#if` 구문 처리하는 코드를 보게 되면 참고할 만한 코드가 있다.

다만 여기서 문제는 preprocessor단에서 알 수 없는 정보가 포함된 경우(ex. user-defined type으로 type
casting 등) 분석이 어려우므로, 전체 소스를 파싱했던 파서를 통해 AST를 만들어서 필요없는 부분을 지워
수식을 만드는 방법으로 처리했다.

```python
class CExprEval(CGenerator):
    def visit_Cast(self, node):
        return self.visit(node.expr)


def _cdef_macro(preprocessor: Any, parser: Any):
    def gen():
        pp = preprocessor

        for m, val in pp.macros.items():
            if not m.startswith('_') and val.value and val.arglist is None:
                e = ''.join(tok.value for tok in pp.expand_macros(pp.tokenize(m)))
                ast = parser.cparser.parse(input=f'int a = {e};', lexer=parser.clex, debug=0)
                expr = CExprEval().visit(ast.ext[0].init)
                expr = expr.replace("&&"," and ")
                expr = expr.replace("||"," or ")
                expr = expr.replace("!"," not ")
                try:
                    v = eval(expr)
                    
                    if isinstance(v, int):
                        yield f'const int64_t {m};'
                    elif isinstance(v, float):
                        yield f'const double {m};'
                    elif isinstance(v, str):
                        yield f'const char * const {m};'
                except Exception:
                    pass
    return '\n'.join(gen())

```

## 조립하기

preprocessor와 parser로 소스를 처리한 후, AST로부터 cdef를 추출하고 부산물을 이용해 매크로 정의를
수집하면 `cffi.FFI.cdef`에 전달할 코드를 생성할 수 있다.
코드는 [cffi_ext](https://github.com/gwangyi/cffi_ext/)를 참조하면 되겠다.

## 수정 내역

* 2017-07-28
  - 최초 버전
