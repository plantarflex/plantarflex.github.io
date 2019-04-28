---
layout: post
title: 파이썬의 namespace
tags: [python, namespace, scope]
comments: true
---

보통 space라 하면, 수학에서는 특정한 구조(structure)를 따르는 수의 집합이라 간단히 말할 수 있다. 이런 수학적 정의를 참고하자면, namespace는 그것이 디자인된 맥락에서 굉장히 이름을 적절하게 잘 지은 것으로 보인다. 다른 포스트에서 다루었듯이 파이썬에서는 변수든 함수든 클래스든 변수(name)가 객체(object; value)를 참조(refer)하는 상황으로 보아야 한다. **namespace는 이러한 name들의 집함임과 동시에, 이들이 참조하는 object와의 매핑 관계가 저장되어있는 개념적인 집합으로 요약할 수 있다.**

scope는 위의 namespace에 직접적으로 접근할 수 있는 스크립트 상의 영역을 뜻한다. 아름답게도, 파이썬에서는 indent 자체로 scope의 범위를 정의한다. **그리고 우리가 OOP에서 습관적으로 사용하는 dot(.) 연산은 그 피연산자 내부의 local namespace를 지정하는 scope에 들어가겠다는 의미이다.**

파이썬의 built-in 함수인 dir는 그것을 콜한 local namespace, 즉 current scope 범위의 모든 name을 리턴한다.

```python
# some_module.py

import bar

def foo(a=1, b=2):
    c = 3
    return dir()


print(dir())
# ['__annotations__', '__builtins__', '__cached__', '__doc__', '__file__', '__loader__', '__name__', '__package__', '__spec__', 'math', 'foo']

print(foo())
# ['a', 'b', 'c']
```

위와 같이, some_module.py에서 임포트한 다른 모듈, 정의한 함수, 정의한 클래스 등이 some_module.py의 local namespace에 살고 있다. 그리고 dir을 콜한 시점에서 그 라인까지 같은 indent만큼 떨어져 있는 object들이 current scope에 잡히게 된다.

함수 foo를 정의하며 dir를 콜했을 때도 마찬가지이다. def 이하의 indent 범위에서 선언된 변수 a, b, c가 foo의 local namespace에 속한 name이 된다.

이제 dot 연산을 통해 함수 foo의 local namespace에 들어가보자. 앞서 dir에 파라미터를 주지 않고 콜하는 경우, 그 해당 영역의 local namespace를 리턴한다고 하였다. 파이썬 공식 레퍼런스에 따르면, dir에 파라미터를 주는 경우, 해당 파라미터의 valid attribute를 모두 반환한다.

```python
# some_module.py (continued)

foo.d = 4

print(foo())
# ['a', 'b', 'c']

print(dir(foo))
'''
['__annotations__',
 '__call__',
 '__class__',
 '__closure__',
 '__code__',
 '__defaults__',
 '__delattr__',
 '__dict__',
 '__dir__',
 '__doc__',
 '__eq__',
 '__format__',
 '__ge__',
 '__get__',
 '__getattribute__',
 '__globals__',
 '__gt__',
 '__hash__',
 '__init__',
 '__init_subclass__',
 '__kwdefaults__',
 '__le__',
 '__lt__',
 '__module__',
 '__name__',
 '__ne__',
 '__new__',
 '__qualname__',
 '__reduce__',
 '__reduce_ex__',
 '__repr__',
 '__setattr__',
 '__sizeof__',
 '__str__',
 '__subclasshook__']
'''
```

dot 연산을 통해 foo의 local namespace에 d 라는 변수를 정의했다(== foo의 attribute를 지정했다). 하지만 이상하게도, 다시 foo 안에서 dir를 콜해서 그 local namespace를 리턴하게 해도 d는 출력되지 않는다. 반대로, foo 바깥에서 dir의 파라미터에 foo를 전달할 경우 새로운 변수 d는 나타나지만 a, b, c 는 온데간데없다. a, b, c와 d는 둘 다 foo의 local namespace가 아니었던가! 이에 대해 명확히 답할 수 있는 레퍼런스를 찾지 못했지만, 잠정적으로 내린 나의 결론은 다음과 같다.

앞서 dir(param)은 param의 valid attribute를 반환한다는, 공식 레퍼런스 치곤 정의가 모호한 답변을 내놓았다. 이러한 valid attribute를 dot 연산을 통해 접근할 수 있는 attribute라고 단순히 생각하면서 다음과 같은 인스펙션으로 흥미로운 사실을 발견할 수 있다.

```python
def foo(a=1, b=2):
    c = 3
    def bar():
        pass
    return dir()

print(foo())            # ['a', 'b', 'bar', 'c']
print(foo.__class__)    # <class 'function'>
print(foo.__call__())   # ['a', 'b', 'bar', 'c']
print(foo.__dict__)     # {'d': 4}
```

foo는 function이라는 클래스를 상속하였고, 그에 바인딩된 __call__은 우리가 알고 있는 매직메서드의 기능을 한다. 또한 __dict__는 우리가 알고 있는 클래스 attribute를 딕셔너리 형태로 리턴한다.

이것을 통해 foo라는 함수는 사실 클래스이며, 우리가 함수를 정의하는 def 이하에 서술한 스크립트는 그 클래스에 바인딩된 __call__메서드의 local nameapce에 해당한다고 생각할 수 있다. 즉, 억지로 표현하자면 아래와 유사한 구조일 것으로 추측된다.

```python
class foo(function):
    d = 4

    @classmethod
    def __call__(cls, a=1, b=2):
        c = 3
        def bar():
            pass
        return dir()
```

다음과 같은 인스펙션은 심지어 더 흥미로운 사실을 알 수 있다.

```python
def foo(a=1, b=2):
    c = 3
    def bar():
        pass
    return dir()

help(foo.__call__)
'''
Help on method-wrapper object:

__call__ = class method-wrapper(object)
 |  Methods defined here:
 |
 |  __call__(self, /, *args, **kwargs)
 |      Call self as a function.
 |
 | (...이하 생략...)
 '''
```

즉, foo.__call__은 단순 function이 아니라, method-wrapper로, foo 자체가 이미 self 파라미터로 전달된 형태의 메서드라는 것이다. 따라서 foo는 그 자체로 클래스가 아니라, 이미 어떤 클래스가 인스턴스화된 형태로 보아야 할 것이다. 따라서 위에서 만든 가상의 구현 모델은 다음과 같이 수정되어야 한다.

```python
class foo(function):
    d = 4

    def __init__(self, *args, **kwargs):
        # do something

    def __call__(self, a=1, b=2):
        c = 3
        def bar():
            pass
        return dir()
```

어떤 마법이 일어난 것인지 이 이상 알기 위해서는 Cython 구현 코드를 뜯어보아야 할 것 같다. 어쨌든 위의 모델을 통해 표현하고자 한 의도는 다음과 같다. **보통 우리가 지칭하는 함수의 local namespace는, 함수를 정의하는 def 이하의 namespace 입장에서는 indent가 다른 영역(아마도 enclosing scope)에 위치하리라는 것이다.**

우리가 단순히 foo에 dot 연산을 하는 것 만으로는, 그 깊은 구석에 indent된 namespace에 살고 있는 a, b, c에 접근할 수 없다. 단순 dot 연산으로는 그저 함수 attribute인 d에만 접근이 가능하다. def 아래에서 콜한 dir는 foo가 콜해지는 동안에 사용되는 namespace의 a, b, c를 보여준다.

foo를 콜하지 않고 a, b, c에 접근하기 위해서는 foo의 local namespace에 있는 특수한 attribute인 __code__를 경유해야만 한다.

```python
def foo(a=1, b=2):
    c = 3
    def bar():
        pass
    return dir()

print(foo.__code__.co_varnames)
# ('a', 'b', 'c', 'bar')
```

휴대할 수 있는 형태로 결론을 내보자.

1. **dot 연산자를 통해 피연산자의 local namespace에 직접 접근할 수 있다.**

2. **dir(foo)는 foo.attr 의 방식으로 접근할 수 있는 local namespace의 모든 attr을 보여준다.**

3. **이러한 namespace는 indent의 간격으로 정해진다.**

4. **클래스 상속이란 이러한 local namespace를 공유 받는다는 뜻이다.(다른 포스트에서 추가 서술)**