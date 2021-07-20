---
layout: post
title: "TypeScript로 간단한 팬게임 제작: 세아랜더"
categories: log
published: true
---

> 게임은 하라고 만드는 게 아닙니다. 플레이어 빡치라고 만드는 겁니다.

## 서문

갑자기 소재가 떠올라서 [세아랜더](https://juo6442.github.io/sea_lander)라는 팬 게임을 제작했다. 달 착륙 게임으로 알려진 루나 랜더의 게임 방식을 따랐다. 최대한 빠르게 완성하는 것을 목표로 했고 제작은 약 2주 가량 걸렸다. 더위 때문에 녹아가면서도 퇴근 후 밤늦게까지 의욕적으로 코딩을 했던 것 같다.

아래는 만들고 나서 떠오른 이야깃거리를 떠오르는 대로 적은 것이다.

## 리소스 제작

### 이미지

![한 그림에 모든 게임내 스프라이트를 모아 놓았다](/assets/2021-07-19-make-a-simple-fan-game-using-ts/res_img.png){: width="480px"}

본격적으로 제작에 들어가기 전 이런 식의 그림을 그렸다. 간단한 게임이라 한 캔버스 안에 모든 엔터티를 그릴 수 있었고 거의 그대로 사용했다. 이렇게 해 두면 서로간의 상대 크기를 파악하기 좋다. 제작 중에 새로 생각난 리소스(주로 UI)는 그때그때 추가로 그리기도 했다.

HTTP request를 최대한 줄이기 위해 막바지에 [TexturePacker](https://www.codeandweb.com/texturepacker)를 이용해 스프라이트를 모두 합칠 생각이었는데, 스프라이트의 메타데이터를 읽어들이는 것을 고려하지 않고 코드를 작성했기 때문에 일일이 좌표를 입력해야 했다. 귀찮아져서 그냥 두었다. 메타데이터를 자동으로 뽑아준다는 사실을 미리 알았다면 이런 일은 일어나지 않았을 것이다...

### 효과음

효과음은 아이패드용 GarageBand와 맥용 Logic Pro로 제작했다. 필요한 효과음을 찾는 시간이나 직접 만드는 시간이나 비슷할 것 같다는 이유였지만... 사용법을 거의 몰라서 그때그때 필요한 부분을 검색해 익혔다.

특히 인트로에서 SEGA를 SEHA로 패러디한 요소는 예전부터 생각하고 있었던 부분이라 피치 조절 기능까지 날림으로 배웠다. 생각한 대로는 뽑힌 것 같다.

## 구현

### 개발 언어

간편하게 웹 페이지에서 바로 플레이할 수 있도록 제작하기로 했다. 요새 잘 모르는 사람이 만든 실행 파일을 받는 것은 거부감이 든다.

게임 제작을 위한 라이브러리나 엔진은 전혀 쓰지 않았는데 쓸 줄 아는 것이 없어서다. 그렇다고 처음부터 공부하느니 차라리 현재 알고 있는 지식으로 처음부터 짜는 게 빠를 것 같았다. 『게임 프로그래밍 알고리즘』이라는 책의 내용을 기억하고 있던 게 도움이 되었다.

제작이 끝난 지금, 실제로 빨랐을지는 모르겠지만 다음부터는 꼭 유니티라도 배워 놓아야겠다는 생각이 들었다.

언어는 TypeScript로 진행했다. 가끔 JS를 쓸 일이 있을 때마다 그 느슨한 타입 시스템 때문에 혼란을 겪은 적이 많았기 때문에. [타입스크립트 핸드북 번역본](https://typescript-kr.github.io)을 잠깐 훑어보고 개발을 시작했다.

### 게임 메인 루프

모든 게임 내 객체에는 `update`와 `render` 메서드를 두어 상태 업데이트와 그리기를 분리했다. 굳이 각각을 `Updatable`, `Renderable`과 같은 인터페이스로 나눌 필요는 없을 것 같았다.

처음엔 `setInterval`로 한 프레임당 한 번씩 게임 내 로직 업데이트를 하고 `requestAnimationFrame`에서 그렸다.

```typescript
// Game.ts
public start(): void {
    setInterval(this.update.bind(this), 1000 / Environment.FPS);
    window.requestAnimationFrame(() => this.render());
    // ...
}
```

하지만 FireFox에서 성능이 좋지 않았다. `update`가 제때 실행되지 않아서였을까? 그래서 좀더 정밀한 `requestAnimationFrame`에서 `update`와 `render` 둘 다 처리하도록 바꿨다.

[MDN requestAnimationFrame 문서](https://developer.mozilla.org/ko/docs/Web/API/Window/requestAnimationFrame)를 보면 "콜백의 수는 보통 1초에 60회지만, 일반적으로 대부분의 브라우저에서는 W3C 권장사항에 따라 그 수가 디스플레이 주사율과 일치하게 됩니다."라고 한다. 이렇게 되면 `update`가 프레임당 한 번 이상 호출되어 게임 속도가 빨라질 수 있으므로 마지막 호출 시점으로부터 변화한 시간을 계산해서 적절히 처리를 해 줘야 한다.

```typescript
// Game.ts
this.interval = 1000 / Environment.FPS;
// ...

private loop(timestamp: number): void {
    const timeDiff = timestamp - this.lastUpdateTimestamp;
    const elapsedFrame = Math.floor(timeDiff / this.interval);

    for (let i = 0; i < elapsedFrame; i++) {
        this.currentScene?.update(this.keyListener.keyStatus);
    }
    if (elapsedFrame > 0) this.currentScene?.render(this.screen.context);

    this.lastUpdateTimestamp = timestamp - (timeDiff % this.interval);

    window.requestAnimationFrame(this.loop.bind(this));
}
```

『게임 프로그래밍 알고리즘』에서는 업데이트 간격이 일정하지 못해도 실제로 지나간 시간에 맞게 객체가 업데이트될 수 있도록 하는 방법을 소개하고 있다. `obj.position.x += 150 * deltaTime`와 같이 변화량에 델타 시간을 곱해주는 것. 하지만 귀찮았기 때문에 여기서는 지나간 프레임만큼 `update`를 반복해서 불러주었다.

### 랜더링

처음 구현할 때는 `screen`, `buffer` 캔버스를 두었다. 그리고 `buffer`에 모든 내용이 그려지면 `buffer`의 내용을 `screen`으로 복사해 실제 화면에 출력되도록 했다. 혹시 tearing이 일어날 지 모른다는 생각에서였다.

```typescript
// Game.ts
private render(): void {
    this.currentScene?.render(this.buffer.context);

    this.screen.context.drawImage(this.buffer.canvas,
            0, 0, this.screen.width, this.screen.height);
    // ...
}
```

쓸데없는 짓이었다. 나중에 성능 측정을 해 보니 화면을 복사하는 과정이 게임을 느리게 만들었으며, 이런 번거로운 과정 없어도 문제 없이 화면이 출력된다. 결국 `buffer`는 역사의 뒤안길로 사라졌다.

### Scene

화면에 표시되는 장면. `Game`에 장면 교체를 요청할 수 있다. 처음엔 entity의 list를 가지고 `update`와 `render`에서 한번에 처리하도록 했다.

```typescript
// Scene.ts
public update(keyStatus: KeyStatus): void {
    this.entities.forEach(e => e.update(keyStatus));
}

public render(context: CanvasRenderingContext2D): void {
    this.entities.forEach(e => e.render(context));
}
```

하지만 각 `Scene`에서 entity 하나하나를 제어해야 하는 일이 많아졌다. 또 그리기 순서를 조절해야 하는 문제도 있어 그냥 없애고 각 subclass에서 알아서들 관리하는 걸로 했다.

### 리소스

`Resource`는 게임에 사용되는 이미지와 오디오를 가지고 있다. 인스턴스를 적당히 따로 둘 수도 있지만 여기서는 간편하게 static으로 관리했다.

인스턴스는 `Loader`를 통해서만 생성할 수 있다. `Loader`는 주어진 모든 리소스를 불러오기 위한 `Promise`를 만들어 반환한다.

```typescript
export default class Resource {
    public static global: Resource | undefined;

    private images: Map<string, HTMLImageElement>;
    private audios: Map<string, AudioBuffer>;

    private constructor() {
        // ...
    }

    static Loader = class Loader {
        public async load(): Promise<Resource> {
            await Promise.all(this.resourcesToLoad);
            return this.resource;
        }
        // ...
    }
}
```

실제 사용할 때는 로딩을 위한 `Scene`을 따로 두어 게임의 모든 리소스를 로드했다. 수가 얼마 안 되므로 이래도 된다.

불러오기에 실패해도 게임이 멈추지 않도록 각 엔터티에서 적절히 처리를 해 두었다. 스프라이트의 경우에는 이미지가 없으면 랜더링을 하지 않고, 오디오의 경우 특이 케이스 패턴으로 인터페이스를 호출해도 아무런 동작이 발생하지 않도록 하였다.

```typescript
// AudioResource.ts
public build(): AudioResource {
    return this.buffer ?
            new AudioResourceImpl(this.buffer, this.loop ?? false) :
            new EmptyAudioResource();
}

// ...

class EmptyAudioResource extends AudioResource {
    public play(): void {}
    public stop(): void {}
}
```

### 오디오

처음엔 `HTMLAudioElement`를 생성해 사용했으나 재생과 멈춤, 루프에 딜레이가 있었다. 게임에 사용하려면 Web Audio API가 더 좋다.

현대 웹 브라우저는 사용자 입력이 없는 상태에서의 오디오 자동 재생을 막고 있다. 그래서 게임을 시작할 때 버튼을 한 번 클릭하도록 만들었다. 하지만 Safari는 이것만으론 안 되고, user interaction으로 야기된 메서드 내에서 play 한 `AudioContext`에 대해서만 이후 자동 재생을 허용한다.

[이 글](https://curtisrobinson.medium.com/how-to-auto-play-audio-in-safari-with-javascript-21d50b0a2765)을 참고하여 시작할 때 빈 오디오를 플레이하면 Safari에서도 오디오가 정상 출력된다.

```typescript
// Bgm.ts (AudioResource.ts에도 비슷한 내용이 있다)
/**
 * Enables Safari's audio auto-play for its context.
 * Need be called by a user interection.
 */
public static enableAutoPlay(): void {
    Bgm.getInstance().play(Bgm.context.createBuffer(1, 1, 8000));
    Bgm.getInstance().stop();
}
```

### 엔터티

화면을 구성하는 기본 오브젝트. 생성에 필요한 파라미터가 많아 읽기 쉽도록 빌더 패턴을 적용했다.

```typescript
// InGameScene.ts
new ParticleGenerator.Builder()
        .setAngle(0, 20)
        .setColor(255, 199, 0)
        .setSpeed(5)
        .setInterval(3)
        .setRadius(0, 20)
        .setDuration(50)
        .build();
```

`Drawable`이라는 슈퍼클래스를 만들어 position, angle, color 등의 공통 프로퍼티를 넣을까 했지만 `Label`에 적용할 수 없는 프로퍼티가 좀 있어서 굳이 통일하지 않았다. 대신 TS의 구조적 서브타이핑을 이용한 인터페이스를 활용했다.

```typescript
// Entity.ts
export interface EntityWithPosition {
    position: Position;
}

export interface EntityWithSize {
    size: Size;
}

export interface EntityWithColor {
    color: Color;
}

// CommonScript.ts
export class Fade extends Script {
    private entity: EntityWithColor;
    // ...
}
```

### 각도

머리의 각도가 0도에서 벗어난 정도에 따라 점수가 계산된다. 따라서 각도는 0 ~ 360도 범위가 아니라 -180 ~ 180도 범위로 정규화해야 계산이 용이하다.

```typescript
// SeaHead.ts
private normalizeAngle(radianAngle: number): number {
    if (radianAngle > +Math.PI) return radianAngle - 2 * Math.PI;
    if (radianAngle < -Math.PI) return radianAngle + 2 * Math.PI;
    return radianAngle;
}
```

추진 방향을 계산할 때 각도에 따른 벡터의 방향은 다음과 같다. (HTML canvas에서는 각도가 시계방향으로 돌아가고, 아래로 내려갈 수록 y축이 증가한다는 것을 유념하자)

- 0도: (0, -1)
- 90도: (1, 0)
- 180도: (0, 1)
- 270도: (-1, 0)

따라서 아래와 같이 삼각함수를 쓰면 된다.

```typescript
// SeaHead.ts
const xProjection = +Math.sin(this.radianAngle);
const yProjection = -Math.cos(this.radianAngle);
```

## 게임 디자인

자고로 팬게임은 똥겜이어야 한다. 게임이 너무 쉬우면 재미없다는 말도 있지 않은가.

### 도킹 조건

Lunar Lander 장르의 게임이니 당연히 x, y축 속력과 각속도가 너무 빠르지 않아야 하고, 각도가 너무 틀어져있지 않아야 한다. 값은 직접 테스트를 하면서 조금씩 조절했는데 막상 릴리즈했을 때 ~~이거 게임아님~~ 어렵다는 의견이 많았다. 이후 거의 눈치채지 못할 정도로 아주 약간만 완화시켰고 더 이상 쉽게 만들 생각은 없다.

대신 도킹 위치를 좀 널널하게 해서 거북목 세아도 허용했다. 여기에는 충돌 판정 계산을 구체-구체로 일원화하려는 목적이 있었고, 다른 이유는 거북목이 되면 웃길 것 같아서다.

### 추진

처음에는 머리의 속도를 점점 아래 방향으로 증가시키는 중력 가속도(`gravity`), 속도와 각속도를 조금씩 감소시키는 공기 저항(`airResistance`)만 두었다. 하지만 이렇게 하니 시작 직후 위 방향 추진만 했을 경우 머리가 대기권을 돌파해서 다시 내려오는데 한세월이 걸리는 문제가 있었다.

그래서 `angleInstability`, `angleGravityRate`를 추가로 두어 추진할 때 각도를 랜덤하게 떨리게 했으며, 기울어져 있다면 아래를 향해 더 기울어지도록 하였다. 이제 만족할 만큼 게임이 어려워졌다.

```typescript
// SeaHead.ts
private boostUp(keyStatus: KeyStatus): boolean {
    // ...
    this.radianAngleVelocity +=
            NumberUtil.random(-this.angleInstability, +this.angleInstability);
    this.radianAngle *= this.angleGravityRate;
    // ...
}
```

### 스코어링

짧은 게임을 좀 더 즐길 수 있게 하는 간단하고 게으르며 고전적인 방법은 스코어링이다. 점수를 내는 기준은 남은 연료, 도킹 위치의 정확성, 각도로 하였다. 위치와 각도는 지수함수로 계산하여 많이 틀어질수록 처참한 점수가 나온다. 첫 릴리즈 기준으로 위치 점수가 너무 민감해서 잠수함 패치로 조절한 상태이다.

각 스테이지별로 얻을 수 있는 점수의 총량에 대해서는 깊이 생각하지 않고 배율을 정했다.

```typescript
// Score.ts
this.fuelScore = fuel;
this.positionScore = Math.floor(Math.pow(2, -Math.abs(positionDiff / 15)) * 500);
this.angleScore = Math.floor(Math.pow(2, -Math.abs(angle * 5)) * 500);
```

점수는 `localStorage`에 별도의 암호화 처리 없이 저장하고 있다. 코드가 다 공개되어서 에디트할 사람은 할 거고, 이런 게임에 그런 짓까지 해서 자존감 챙기려는 사람이 있을까 싶어서 굳이 하지 않았다. 뭐 이런 걸로 자존감이 높아진다면 나름대로 좋은 일이 아닐까(...)

### 스테이지 구성

생각한 게임 시스템만으론 난이도 구성에 한계가 있다는 생각이 들어 깔끔하게 10스테이지를 마지막으로 정했다. 스테이지에 따라 달라지는 것은 몸통(들)의 위치와 적 머리의 위치이다.

몸통은 스테이지별로 배치되는 개수와 타입이 정해져 있으며, 위치는 랜덤하다. 구석에 있거나 겹치거나 너무 가깝지 않아야 하므로 화면의 가로 길이를 적당히 140픽셀 단위로 등분하여 배치하였다. 몸통이 시작 위치 바로 아래 있을 경우 게임이 너무 쉽게 끝나는 경우가 있지만 운빨X망겜이라는 극찬을 받기 위해 그대로 놔두었다.

```typescript
// ActorGenerator.ts
abstract class BaseActorGenerator {
    private static HORIZONTAL_SEGMENT_LENGTH = 140;
    private static SEA_BODY_TOP = Environment.VIEWPORT_HEIGHT - 100;

    /**
     * Returns n random position within the screen width.
     * @param n - Numbers to return
     * @returns Array of positions
     */
    protected getRandomHorizontalPositions(n: number): number[] {
        const segments = new Array();   // [140, 280, 420, ...] 이 들어갈 것이다
        for (let i = BaseActorGenerator.HORIZONTAL_SEGMENT_LENGTH;
                i <= Environment.VIEWPORT_WIDTH - BaseActorGenerator.HORIZONTAL_SEGMENT_LENGTH;
                i += BaseActorGenerator.HORIZONTAL_SEGMENT_LENGTH) {
            segments.push(i);
        }

        // 섞은 후 앞에서부터 필요한 개수만 잘라서 리턴한다
        const shuffledSegments = segments.sort(() => Math.random() - 0.5);
        return shuffledSegments.slice(0, n);
    }
}
```

적 머리는 완전 랜덤하게 배치하면 플레이어의 시작 지점과 너무 가깝거나 겹쳐서 시작하자마자 불합리하게 한 몫을 날릴 수 있다. 때문에 수동으로 배치했는데, 이동이 빠른 타입의 머리는 시작 지점에서 떨어진 곳에 놓고 이동이 느린 머리는 비교적 가까이에 놓았다. 대신 방향에는 랜덤성을 두었다.

## 배포

처음에는 급하게 릴리즈하느라 `dist` 디렉토리까지 main branch에 포함시킨 후 github pages에서 main branch를 사용하도록 했다. 하지만 이렇게 하면 commit log가 지저분해진다. 저 중간중간에 낀 "Dist"를 보라.

![일반적인 커밋 중간중간에 끼어 있는 "Dist"](/assets/2021-07-19-make-a-simple-fan-game-using-ts/commit_log_dist.png)

그래서 이후 짬이 좀 났을 때 [copy-webpack-plugin](https://webpack.js.org/plugins/copy-webpack-plugin)을 이용해 빌드시 웹 페이지용 파일과 리소스 파일을 배포용 디렉토리에 복사하도록 만들었다. 거창한 프레임워크를 쓴 것이 아니므로 이 정도로 충분하다.

```typescript
// webpack.config.js
const CopyPlugin = require("copy-webpack-plugin");
// ...
plugins: [
    new CopyPlugin({
        patterns: [
            { from: "web" },
            { from: "res", to: "res" },
        ],
    }),
],
```

그리고 [gh-pages](https://www.npmjs.com/package/gh-pages)를 사용한다. 아래와 같이 설정하고 `npm run deploy`를 실행하면 `predeploy`의 내용(빌드)이 실행된 후 `dist` 디렉토리의 내용이 기본값인 `gh-pages` 브랜치로 배포된다.

```typescript
// package.json
"scripts": {
    "build": "webpack --mode=development",
    "dist": "webpack --mode=production",
    // 아래 두 줄 추가
    "predeploy": "npm run dist",
    "deploy": "gh-pages -d dist"
},
"homepage": "https://juo6442.github.io/sea_lander",
```

## 후기

스트리머가 키보드 샷건을 치는 모습을 보니 뿌듯함이 느껴졌다. 왜 사람들이 똥겜을 만드는지 알 수 있었던 유익한 시간이었다.
