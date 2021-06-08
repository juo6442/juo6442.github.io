---
layout: post
title: "TypeScript try/catch/finally and Custom Errors"
categories: translate
published: true
---

원문: [TypeScript try/catch/finally and Custom Errors](https://joefallon.net/2018/09/typescript-try-catch-finally-and-custom-errors/)

몇 년 전 미국의 한 유명 항공사의 모든 예약과 체크인 시스템이 혼잡한 평일 아침 한 시간 이상 중지되었다. 이로 인해 전국적으로 하루종일 비행편이 지연되었다. 이는 처리되지 않은(unhandled) 에러 때문에 항공편 조회 서비스가 완전히 멈춰버렸던 것으로 밝혀졌다.

TypeScript와 JavaScript에서의 에러 핸들링은 개발자가 경험해보았을 일 중 가장 기본적인 것 중 하나다. 이는 코드의 다른 부분만큼이나 중요하며 간과하거나 과소평가되어서도 안 된다. 이 글은 새내기 개발자가 어떻게 산업 표준에 따라 에러를 처리하고, 던지고(throwing), try/catch/finally를 사용할지와, 어떻게 처리하면 안 될지에 대한 가이드다.

### 모범 사례 - 에러를 catch할 때 타입 정보 얻기

여기 에러를 catch할 때 타입 정보를 얻는 메서드가 있다.

```typescript
try {
    // 에러를 throw할 수도 있는 코드...
}
catch(e) {
    if(e instanceof Error) {
        // 여기서 IDE 타입 힌팅이 활성화된다
        // 에러 e를 알맞게 처리한다
    }
    else if(typeof e === 'string' || e instanceof String) {
        // 여기서 IDE 타입 힌팅이 활성화된다
        // 에러 e를 알맞게 처리하거나...문자열을 그대로 throw하는 라이브러리를 쓰지 말 것
    }
    else if(typeof e === 'number' || e instanceof Number) {
        // 여기서 IDE 타입 힌팅이 활성화된다
        // 에러 e를 알맞게 처리하거나...숫자를 그대로 throw하는 라이브러리를 쓰지 말 것
    }
    else if(typeof e === 'boolean' || e instanceof Boolean) {
        // 여기서 IDE 타입 힌팅이 활성화된다
        // 에러 e를 알맞게 처리하거나...불린을 그대로 throw하는 라이브러리를 쓰지 말 것
    }
    else {
        // 만약 뭘 처리해야 하는지 알 수 없을 경우
        // 아마 복구가 안 될 것이다...그러므로 다시 throw한다
        // 나에게 남기는 노트: 내 인생을 다시 생각해 보고 더 쓸만한 라이브러리를 고를 것
        throw e;
    }
}
```

위에서 봤듯 당신이 핸들링하는 에러에 대한 타입을 찾으려 하는 것은 꽤 지루할 수 있다. 대부분의 경우 복구 가능한 종류의 에러 발생 상황에 대해서는 이미 알고 있을 것이다. 그 외의 경우는 삼키지(swallow) 말고 다시 throw해야 한다. (나중에 더 알아볼 것이다) 여기 더 간단하고 전형적인 버전이 있다:

```typescript
try {
    // 에러를 throw할 수도 있는 코드...
}
catch(e) {
    if(e instanceof Error) {
        // 여기서 IDE 타입 힌팅이 활성화된다
        // 에러 e를 알맞게 처리한다
    }
    else {
        // 아마 복구가 안 될 것이다...그러므로 다시 throw한다
        throw e;
    }
}
```

### 모범 사례 - 항상 에러 객체를 사용하라, 스칼라는 쓰지 말자

에러를 던지거나 리턴할 때 항상 Error 객체나, Error 객체를 상속받은 객체의 인스턴스를 사용하자. 절대 스칼라는 throw하지 말자. (숫자, 문자열 자체 등) 그러면 에러를 핸들링할 때 스택 트레이스를 수집하고 에러 타이핑을 사용할 수 있다.

이것은 code utilizing 콜백으로 Error를 리턴하는 예제이다:

```typescript
function example1(param: number, callback: (err: Error, result: number) => void) {
    try {
        // 에러를 throw할 수도 있는 코드...
        const exampleResult = 1;
        callback(null, exampleResult);
    }
    catch(e) {
        callback(e, null); // Caller가 에러를 처리한다
    }
}
```

이것은 async 함수에서 에러를 던지는 예제이다:

```typescript
async function example2() {
    await functionReturningPromise();
    throw new Error('example error message');
}
```

얼마나 코드가 줄어들고 간단해지는지 느껴보길. 이것이 async/await을 쓸 때의 공통 테마이다. 정말 추천한다. 에러로 돌아가자...

### 모범 사례 - finally에서 리소스를 해제하자

finally 절에서는 리소스를 해제해야 한다.

```typescript
async function example2() {
    let resource: ExampleResource = null;

    try {
        resource = new ExampleResource();
        resource.open();
        // 리소스를 사용...
    }
    catch(e) {
        if(e instanceof Error) {
            // 여기서 IDE 타입 힌팅이 활성화된다
            // 가능하면 에러를 처리한다
            // 적절히 처리할 수 없다면 다시 던진다
        }
        else {
            // 적절히 문자열이나 숫자를 처리한다
            // 가능하다면 처리하고 아니면 다시 던진다.
            throw e;
        }
    }
    finally {
        // 항상 실행된다
        // 필요하다면 리소스를 해제한다 (데이터베이스를 닫는 등)
        if(resource != null) {
            resource.close();
        }
    }
}
```

### 모범 사례 - Error를 세분화해라

종종 일반적인 Error 객체 말고 더 세분화된 에러를 던지거나 리턴하고 싶을 것이다. 표준 에러 객체를 확장(extend)하면 쉽다.

```typescript
class InvalidArgumentError extends Error {
    constructor(message?: string) {
        super(message);
        // 참고: typescriptlang.org/docs/handbook/release-notes/typescript-2-2.html
        Object.setPrototypeOf(this, new.target.prototype); // 프로토타입 체인 복구
        this.name = InvalidArgumentError.name; // 이제 스택트레이스가 제대로 표시된다
    }
}
```

세분화된 에러를 사용하는 목적은 현재 범위 내에서 에러를 처리할 수 있는 가능성을 높이고, 코드 실행을 계속할 수 있게 하는 것이다. 이렇게 세분화된 에러를 처리할 수 있다.

```typescript
try {
    throw new InvalidArgumentError();
}
catch(e) {
    if(e instanceof InvalidArgumentError) {
        // InvalidArgumentError 처리
    }
    else if(e instanceof Error) {
        // 일반적인 에러 처리
        // 중요: 가장 세분화된 에러부터 순서대로 나열해야 한다 (child ~> parent)
    }
    else {
        throw e;
    }
}
```

"if" 조건의 에러는 가장 세분화된 에러부터 덜 세분화된 에러 순으로 나열되어야 한다. InvalidArgumentError도 Error의 인스턴스이기 때문이다.

### 모범 사례 - 예외를 삼키지(swallow) 말자

Error를 잡아서(caught) 브라우저에게 주의시키는 한 브라우저는 기쁘게 당신의 코드를 계속해서 실행할 것이다. 사실 catch 블록에서 Error를 다루는 것은 전적으로 선택 사항이다. 예외를 삼키는 것은 Error를 catch하지만 아무 것도 하지 않는 행위를 말한다. 여기 Error 삼키기의 예제가 있다.

```typescript
function exceptionSwallower1() {
    try {
        throw new Error('my error');
    }
    catch(e) { /* 우리끼리의 비밀 */ }
}
```

코드는 유효하지만 이것은 아주 나쁜 관행이다. 최소한 에러를 다시 던지기라도 해야 한다.

코드가 유효하기 때문에 컴파일러는 불평하지 않는다. 하지만 정말로 잘 문서화된 이유가 없다면 절대로 하지 말자. 예외가 잡혔지만 발생한 이슈에 대해 아무 것도 하지 않았다. 결국 잡힌 에러에서 추출할 수 있는 유용한 정보를 잃어버리게 된다.

흔하게 볼 수 있지만 별로 도움이 되지 않는 다른 관행은 콘솔에 에러를 로깅하고 계속 실행하는 것이다:

```typescript
function exceptionSwallower2() {
    try {
        throw new Error('my error');
    }
    catch(e) {
        console.log(err);
    }
}
```

finally 블록에서 갑자기 리턴하는 것도 에러를 삼키는 행위이다.

```typescript
function exceptionSwallower3() {
    try {
        throw new Error('my error');
    }
    finally {
        return null;
    }
}
```

처음에 에러가 던져지고 finally 블록이 실행된다. 그리고 갑자기 리턴을 하였으므로 모든 에러에 대한 정보가 사라진다. 여기서도 에러는 삼켜지게 된다.

에러는 섀도잉으로도 삼켜질 수 있다.

```typescript
function exceptionSwallower4() {
    try {
        throw new Error('my error');
    }
    finally {
        throw new Error('different error'); // 첫 번째 에러는 완전히 숨겨진다
    }
}
```

### 모범 사례 - throw를 GoTo처럼 쓰지 말 것

가끔 실제 에러 조건이 아닌데 코드 흐름을 제어할 목적으로 try/catch 메커니즘을 사용해놓곤 스스로 똑똑하다고 생각하는 사람이 있다.

```typescript
function badGoto() {
    try {
        // some code 1

        if(some_condition) {
            throw new Error();
        }
        // some code 2
    }
    catch(e) {
        // some_condition이 true였음
        // some code 3 (에러 처리 코드가 아님)
    }
}
```

이 코드의 목적은 "some code 2"를 실행하지 않고 넘기는 것이다. try/catch는 일반적인 프로그램 로직의 흐름을 위한 것이 아니며 에러 조건에 사용하도록 설계되었기 때문에 이런 방법은 효과적이지 않을 뿐 아니라 느리다. JavaScript/TypeScript는 이미 충분하고도 남을 흐름 제어 로직을 제공하고 있다. 따라서 이렇게 goto문을 흉내내는 것은 좋은 생각이 아니다.

### 모범 사례 - 로깅 후 다시 던질 목적으로 catch를 사용하지 말 것

당신의 어플리케이션이 멈추는 이유를 알아내려고 할 때, 예외에 대한 로그를 출력하자마자 다시 던지지 말아라:

```typescript
function clogMyLogs() {
    try {
        throw new Error('my error');
    }
    catch(e) {
        console.log(err);
        throw err;
    }
}
```

불필요할 뿐 아니라 중복된 로그 메시지가 출력되어 수많은 텍스트로 당신의 로그를 무겁게 할 것이다.
