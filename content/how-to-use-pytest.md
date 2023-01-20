Title: pytest, 어떻게 사용할까?
Date: 2023-01-19 21:53
Modified: 2023-01-19 21:53
Category: 언어와 프레임워크
Tags: python, testing
Slug: how-to-use-pytest

# 테스트의 단계

테스트는 일반적으로 다음의 4단계의 절차로 이뤄진다고 생각할 수 있습니다.

1. Arrange: 테스트하려는 행동을 하기 전에 필요한 모든 것을 수행합니다.
2. Act: 테스트하려고 하는 행동(behavior)을 시스템에 수행합니다.
3. Assert: 행위의 결과가 우리가 원하는 대로였는지 확인하는 단계입니다.
4. Cleanup: 다른 테스트가 이 테스트에 영향을 받지 않도록 정리합니다.

# Arrange: fixture 사용

E2E 테스트를 진행하면 테스트용 DB를 준비하고, 해당 DB에 연결할 클라이언트가 준비되어야 합니다. 또 테스트하려는 행동을 할 때 필요한 데이터도 준비해야 합니다. 바로 위에서 말한 Arrange 단계입니다. pytest에서는 [fixture](https://docs.pytest.org/en/7.2.x/explanation/fixtures.html#about-fixtures)를 사용해 이 단계에 필요한 절차들을 함수로 작성해 사용할 수 있습니다.

fixture 데코레이터를 사용해 함수를 정의한 뒤 함수 이름과 같은 이름으로 테스트 함수의 input argument에 넣어주기만 하면, pytest가 해당 fixture 객체를 전체 테스트에서 사용할 수 있습니다. fixture는 테스트 함수 안에서 여러 개를 사용할 수도 있고 다른 fixture에 의존하는 fixture도 사용이 가능합니다. 하지만 fixture가 발생할 경우 예외를 raise하게 되고, pytest는 fixture 작동을 중단시키고 실패로 표시됩니다. 따라서 의존성을 신중히 만들 필요가 있습니다. 필수적인 것이 아니라면 제외해야할 것입니다. fixture의 결과는 캐시되기 때문에 같은 테스트 안에서 여러번 사용이 되어도 한번만 불러와서 리소스를 아낄 수 있습니다.

fixture의 특징을 요약하면 다음과 같습니다.

1.  함수에 `@pytest.fixture` 데코레이터를 사용해서 fixture를 만들 수 있습니다.
2.  테스트 함수의 input argument로 fixture 이름을 넣으면, 테스트 함수 내에서 사용할 수 있습니다.
3.  한번에 여러개의 fixture를 사용할 수 있습니다. fixture는 순서대로 실행됩니다.
4.  fixture는 다른 fixture를 요청할 수 있습니다.
5.  [autouse 옵션](https://docs.pytest.org/en/7.2.x/how-to/fixtures.html#autouse-fixtures-fixtures-you-don-t-have-to-request)으로 테스트에서 특정 fixture를 input argument에 명시적으로 사용하지 않아도 매번 실행되도록 할 수 있습니다. fixture decorator의 옵션으로 넣어주면 사용가능합니다. autouse fixture는 해당 scope 내에서 가장 먼저 실행됩니다.
6.  [scope 옵션](https://docs.pytest.org/en/7.2.x/how-to/fixtures.html#scope-sharing-fixtures-across-classes-modules-packages-or-session)으로 fixture를 function, class, module, pacakage, session 안에서 공유할 수 있습니다. fixture decorator의 옵션으로 넣어주면 사용 가능합니다. scope에는 callable을 넣을 수 있어 [동적으로 scope를 변경](https://docs.pytest.org/en/7.2.x/how-to/fixtures.html#dynamic-scope)할 수 있습니다. 아무 옵션도 넣지 않으면 scope는 function으로 설정됩니다. session scope로 되어 있는 fixture는 `conftest.py` 파일에서 관리하는 것이 좋습니다.
7.  한 스코프 안에서, fixture는 처음 한번만 실행되고, 그 결과가 캐시됩니다.

# Cleanup: fixture 함수 안에서 yield 활용

Arrange 단계에서 사용하는 fixture와 이를 마무리 짓는 Cleanup 단계가 밀접하게 관련이 있으므로, 먼저 살펴보도록 하겠습니다. pytest에서는 fixture에 [yield를 사용하는 것을 추천](https://docs.pytest.org/en/7.2.x/how-to/fixtures.html#yield-fixtures-recommended)합니다. pytest가 fixture를 순서대로 요청한 뒤, 테스트가 끝나면 이 순서의 역순으로 다시 돌아갑니다. 따라서 yield문 다음에 Cleanup 로직을 추가하면 됩니다.

yield 이후의 작업은 fixture의 scope가 끝날 때 실행됩니다. 아래 예시를 보면 그것을 알 수 있습니다.

```python
import pytest
import logging

# Create and configure logger
logging.basicConfig(filename="newfile.log",
                    format='%(asctime)s %(message)s',
                    filemode='w')

# Creating an object
logger = logging.getLogger()

# Setting the threshold of logger to DEBUG
logger.setLevel(logging.DEBUG)

logger.debug("set logger")

@pytest.fixture(scope="class")
def value_a():
    logger.debug("value_a called")
    yield 5
    logger.debug("value_a after yeild")

class TestArithmetic:
    @pytest.fixture(autouse=True)
    def autouse(self):
        logger.debug("autouse called")

    def test_addition(self, value_a):
        assert value_a + 1 == 6

    def test_subtraction(self, value_a):
        assert value_a - 1 == 4

    def test_multiplication(self, value_a):
        assert value_a * 2 == 10
```

위 코드를 `pytest -o log_cli=true` 의 명령어로 실행한 결과는 다음과 같습니다.

```
=============================================== test session starts ===============================================
platform darwin -- Python 3.10.8, pytest-7.2.1, pluggy-1.0.0
rootdir: /Users/feynubrick/dev/practice/pytest
collecting ...
----------------------------------------------- live log collection -----------------------------------------------
DEBUG    root:test_arithmetic.py:15 set logger
collected 3 items

test_arithmetic.py::TestArithmetic::test_addition
------------------------------------------------- live log setup --------------------------------------------------
DEBUG    root:test_arithmetic.py:19 value_a called
DEBUG    root:test_arithmetic.py:26 autouse called
PASSED                                                                                                      [ 33%]
test_arithmetic.py::TestArithmetic::test_subtraction
------------------------------------------------- live log setup --------------------------------------------------
DEBUG    root:test_arithmetic.py:26 autouse called
PASSED                                                                                                      [ 66%]
test_arithmetic.py::TestArithmetic::test_multiplication
------------------------------------------------- live log setup --------------------------------------------------
DEBUG    root:test_arithmetic.py:26 autouse called
PASSED                                                                                                      [100%]
------------------------------------------------ live log teardown ------------------------------------------------
DEBUG    root:test_arithmetic.py:21 value_a after yeild


================================================ 3 passed in 0.01s ================================================
```

# Act and Assert 정의: test function 사용

시스템에 어떤 작용을 해서 그 결과가 우리가 의도한 대로 나오는지 확인하는 부분은 함수에서 실행합니다. `test_*` 와 같은 형식의 이름의 함수에 테스트 내용을 정의하면, pytest가 해당 함수들만 실행해 결과를 확인합니다.

```python
# content of test_assert1.py
def f():
    return 3


def test_function():
    assert f() == 4
```

예외(Exception)가 발생했는지 확인하려면 pytest.raises()를 context manager로 사용하면 됩니다.

```python
import pytest


def test_zero_division():
    with pytest.raises(ZeroDivisionError):
        1 / 0
```

여러 테스트를 class 로 묶어서 실행할 수도 있습니다. 이때 class의 네이밍을 `Test*` 와 같이 해야합니다.

```python
class TestArithmetic:
  def test_addition():
      assert 1 + 1 == 2

  def test_subtraction():
      assert 4 - 1 == 3
```

# Test 실행하기

테스트 실행은 `pytest` 명령으로 수행합니다. 이 명령어는 현재 디렉토리 및 서브디렉토리 안에 있는 test\__.py 또는 _\_test.py의 형식으로 된 모든 파일을 실행합니다([standard test discovery rule](https://docs.pytest.org/en/7.2.x/explanation/goodpractices.html#conventions-for-python-test-discovery)). `pytest.ini`에 [testpaths](https://docs.pytest.org/en/7.2.x/reference/reference.html#confval-testpaths)를 설정하면 등록된 디렉토리에서만 테스트를 수행합니다.

이외에도 특정 테스트들만 골라서 수행할 수 있는 방법들이 아래와 같이 있습니다.

| 상황                           | 방법                                       | 설명                                                                                                                                     |
| ------------------------------ | ------------------------------------------ | ---------------------------------------------------------------------------------------------------------------------------------------- |
| 특정 모듈 테스트               | pytest test_mod.py                         | 파일 안의 테스트 전체 수행                                                                                                               |
| 특정 디렉토리 안의 모든 테스트 | pytest tests/                              | 디렉토리 안의 모든 테스트 수행                                                                                                           |
| 특정 키워드 테스트             | pytest -k “MyClass and not method”         | 현재 디렉토리와 그 서브 디렉토리에서 “MyClass”라는 키워드는 포함하지만 “method”라는 키워드는 포함하지 않는 테스트 수행 (대소문자 구별 X) |
| 노드 ID로 테스트               | pytest test_mod.py::TestClass::test_method | 특정 부분만 콕집어서 테스트                                                                                                              |
| 마커로 표시한 내용 테스트      | pytest -m slow                             | `@pytest.mark.slow` 데코레이터로 표시한 테스트만 수행. ([마커 사용법](https://docs.pytest.org/en/7.2.x/how-to/mark.html#mark))           |

명령어로 수행하는 방법 외에 python code에서 실행할 수 있는 방법도 있습니다. 해당 내용에 관해서는 [여기](https://docs.pytest.org/en/7.2.x/how-to/usage.html#other-ways-of-calling-pytest)를 참조해주세요.
