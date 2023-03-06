# Throw vs If 퍼포먼스
이 저장소는 java에서 Throw로 값 검증을 하는 것이 얼마나 더 성능에 악영향을 미치는 지에 대해서 측정하기 위해 제작되었습니다.

Throw 는 현대적인 CPU에서 검사를 위한 비용이 zero cost 이지만 throw 가 발생하는 경우 수백에 이르는 CPU 사이클을 낭비할 수도 있습니다.

반대로 If는 반드시 평균적으로 수십 사이클 가량의 성능을 소모하지만 일관적으로 성능이 나오는 특징이 있습니다.

따라서 예상되는 결과는 다음과 같습니다.

throw 가 자주 생기는 경우라면 throw 버전이 성능이 더 나쁠 것이다.(에러 스택 복원에 수백, 수천 CPU 사이클을 낭비하므로)
throw 가 자주 생기지 않는 경우라면 throw 버전은 if 버전보다 압도적으로 빠를 것이다. 

## 검증 절차

우선 throw로 조건을 검사하는 로직을 작성합니다.

로직은 아래와 같습니다.

```java
    public boolean checkWithThrow(String parameter) {
        try {
            int serverId = Integer.parseInt(parameter);
            if (serverId == 0)
                return false;
            return true;
        } catch (Exception e) {
            return true;
        }
    }
```

다음으로는 정규식으로 먼저 탐색한 다음 검사하는 로직을 작성합니다.

```java
    public static Pattern COMPLEX_NUMERIC = Pattern.compile("^(\\+|\\-)?[0-9]+$");
    public static Pattern SIMPLE_NUMERIC = Pattern.compile("^(\\+|\\-)?[0-9]+$");

    public boolean checkWithComplexRegex(String parameter) {
        if(COMPLEX_NUMERIC.matcher(parameter).matches()){
            int serverId = Integer.parseInt(parameter);
            if (serverId == 0)
                return false;
            return true;
        }else{
            return true;
        }
    }
    public boolean checkWithSimpleRegex(String parameter) {
        if(SIMPLE_NUMERIC.matcher(parameter).matches()){
            int serverId = Integer.parseInt(parameter);
            if (serverId == 0)
                return false;
            return true;
        }else{
            return true;
        }
    }
```

정규식은 양수 음수를 모두 판단 가능한 complex 버전과 부호가 없는 경우의 양수만 판단 가능한 simple 버전으로 검사했습니다.

이 모든 경우에서 각각의 함수가 100만번 실행되는데 걸리는 시간을 이용해 평균적인 퍼포먼스 측정과 throw 방식의 단점을 확인해 보고자 합니다.

## 예상되는 결론

만약 `parameter`값으로 대부분의 경우 숫자로 된 문자열이 넘어온다면 성능은 다음과 같을 것입니다.

1. `throw`
1. `if simple`
1. `if complex`

하지만 만약 문자형으로 된 자료가 넘어온다면? 그때는 `try/catch`문법의 오버헤드에 의해 퍼포먼스에 대한 전반적 하락이 있을 수도 있습니다.

이 경우에 대한 속도 예상은 다음과 같습니다.

1. `if simple`
1. `if complex`
1. `throw`

## 실험 결과

실험은 [openjdk/jmh](https://github.com/openjdk/jmh)를 이용해 측정되었습니다.
실험에 사용된 코드는 다음과 같습니다.

[코드 보러가기](./app/src/jmh/java/com/egoavara/throwifperf/AppTest.java)

```
Benchmark                            Mode  Cnt     Score    Error  Units
AppTest.checkWithComplexRegexNumber  avgt   10    47.000 ±  3.036  ms/op
AppTest.checkWithComplexRegexString  avgt   10    34.709 ±  2.050  ms/op
AppTest.checkWithSimpleRegexNumber   avgt   10    47.285 ±  3.678  ms/op
AppTest.checkWithSimpleRegexString   avgt   10    32.959 ±  0.309  ms/op
AppTest.checkWithThrowNumber         avgt   10    ≈ 10⁻⁶           ms/op
AppTest.checkWithThrowString         avgt   10  1385.552 ± 11.133  ms/op
```

## 원인 분석

### 직접 분석

우선 몇가지 이야기하면 예상과 큰 그림은 일치했지만 생각과는 다르게 정규식을 좀 더 복잡하게 쓴 정도로는 성능에 유의미한 영향이 거의 없었다는 것이다.

그리고 예상보다 throw를 안걸릴 때는 너무 빠르고 catch에 잡혔을 때는 너무 느리다는 것도 신경쓰였다.

결과를 보면 catch에 걸리면 30배 정도 느려지지만 걸리지 않으면 거의 제로타임이라 부를 정도로 빨랐다.

이런 것들을 보면 정규식이 아니라 좀 더 최적화된 조건 검사를 통하면 더 빠른 결과를 낼 수도 있을지도 모든다는 생각도 들고, 동시에 throw가 예상보다도 느리다는 실감이 들었다.

### ChatGPT님의 고견

If the input to this method mostly consists of non-integer values, then there may be some performance implications.
> 만약 대다수의 입력이 정수형이 아니라면 약간의 성능에 영향이 있을 수도 있습니다.


The parseInt() method will throw a NumberFormatException if the input string cannot be parsed as an integer. 
> 만약 parseInt() 메서드가 정수형 분석에 실패하면 `NumberFormatException`에러를 발생시킵니다.

If the input to this method mostly consists of non-integer values, then this method will spend a lot of time catching and handling NumberFormatException exceptions. 
> 만약 비정수형 자료로 메서드를 호출하면 이 메서드는 많은 시간을 `NumberFormatException`를 처리하는데 할애하게 됩니다.

This could potentially slow down the method, especially if the input strings are very long.
> 이는 성능 하락의 가능성이 있으며, 특히 입력값이 매우 긴 경우에 더욱 그렇습니다.

In this case, it might be better to check if the input string can be parsed as an integer before actually parsing it. 
> 이 경우 파싱 전 입력값을 사전 검증하는 것이 도움이 될 것입니다.

One way to do this would be to use a regular expression to check if the string consists only of digits. Here's an example:
> 이런 방법의 하나로, 정규식을 이용하는 것이 있습니다. 아래 예시가 있습니다.

```java
public boolean checkWithThrow(String parameter) {
    if (!parameter.matches("\\d+")) {
        // Input string contains non-numeric characters
        return false;
    }

    int serverId = Integer.parseInt(parameter);
    if (serverId == 0)
        return false;
    return true;
}
````

This version of the code first checks if the input string contains only digits using the matches() method and a regular expression. 
> 이 버전의 코드는 먼저 입력값이 오직 숫자로만 이뤄졌는지 `matches`메서드를 이용해 검증합니다.

If the input string contains non-numeric characters, it immediately returns false without trying to parse the string as an integer. 
> 만약 숫자형 문자 이외의 문자가 사용된다면 메서드는 파싱을 하지 않고 즉시 false를 반환합니다.

This should be faster than parsing the string and catching NumberFormatException exceptions for every non-numeric input.
> 이는 모든 숫자가 아닌 형에 대해 `NumberFormatException`를 발생시키지 않음으로서 파싱을 더욱 빠르게 만듭니다.

***ChatGPT이 가라사대, 딱히 문제될 법한 건 아닌데 좀 더 느릴 수도 있다 하더라.***


## 결론
결론적으로, 만약 입력값이 대부분의 경우가 숫자형이 아닐 경우 이 코드는 잠재적인 성능 문제를 발생시킬 가능성이 있습니다.

물론 이 사례는 100만건당 1.4초 정도로 매우 작은 영향만이 있다는 사실은 중요합니다.

대다수의 경우에는 문제가 없을 것으로 생각됩니다.

## 한계
이 실험에서는 입력으로 `"hello"`, `"1"`같은 매우 간단한 자료만 넣었습니다.
이는 더 복잡한 숫자가 들어왔을 때 어떤 영향이 있을지 예측하는 데에 한계가 있습니다.

입력값이 길어지면 성능 하락의 영향이 정규식 버전이 더 클지, throw 버전이 더 클지 확인해 보는 것도 재미있을 것 입니다.

또 위의 사례는 매우 간단한 컨텍스트를 가진 경우의 복구 사례입니다. 함수 호출이 깊거나, 상태 변수가 많거나, 다른 비동기 호출이 이 함수에 영향을 받을 수 있는 경우라면 
성능은 현재보다 더 낮아질 가능성도 있습니다. 

이런 사례에 대해 조사해보는 것도 좋을 것입니다.