---
layout: post
title:  "Javascript를 이용해 프로그래밍 방식으로 소리 생성하기"
categories: translate
published: true
---

<link rel="stylesheet" type="text/css" href="/assets/2018-06-02-generate-sounds-programmatically-with-javascript/style.css" />
<script src="/assets/2018-06-02-generate-sounds-programmatically-with-javascript/music.js"></script>

원문: [Generate Sounds Programmatically With Javascript](http://marcgg.com/blog/2016/11/01/javascript-audio/)

최근에 있었던 해커톤에서 나는 [Web Audio API](https://developer.mozilla.org/en-US/docs/Web/API/Web_Audio_API)를 사용해 프로그래밍 방식으로 생성된 소리를 이용한 [멀티플레이어 8비트 시퀀서](http://marcgg.com/blog/2016/11/21/chiptune-sequencer-multiplayer/)를 만들기로 결정했다. HTML 5의 `audio` 태그만 사용하는 것은 원치 않았는데 너무 제한적이란 걸 알았기 때문이다… 하지만 처음 발견했던 것은 제대로 된 소리를 내는 것이 전혀 간단하지 않았다는 것이다. 특히 당신이 나처럼 아주 기초적인 음악적 소양 정도만 가지고 있다면 말이다. 그럼 정확히 어떻게 해야 깨끗하고 낭랑하며 멋진 소리가 나는 음을 만들 수 있을까?

이 글에서는 몇몇 조언과 사용 가능한 코드를 제공한다. [당신이 사용 중인 브라우저에서 지원](http://caniuse.com/#feat=audio-api)한다면 실제로 실행할 수 있는 예제도 있다.[^1]

[^1]: (역주) 해당 페이지에도 나와 있지만 Safari의 경우 `webkit` prefix가 필요, 즉 `webkitAudioContext`과 같이 사용해야 합니다. 따라서 본 글의 예제도 제대로 작동하지 않습니다.

## 간단한 삐 소리 만들기

첫 번째로 [사인파](https://en.wikipedia.org/wiki/Sine_wave)를 이용해 아주 기본적인 삐 소리를 만들어 보자. 우리는 소리를 생성하는 데 가장 중요한 객체인 [audio context](https://developer.mozilla.org/en-US/docs/Web/API/AudioContext)를 초기화한 후에, 사인파를 생성하는 [oscillator](https://developer.mozilla.org/en-US/docs/Web/API/OscillatorNode)를 생성할 것이다. 마지막으로 oscillator를 context에 연결하고 start 한다.

```javascript
var context = new AudioContext()
var o = context.createOscillator()
o.type = "sine"
o.connect(context.destination)
o.start()
```

<a href="#!" class="js_play_sound" onclick="example1()">재생</a>
<a href="#!" class="js_stop_sound" onclick="o.stop()">정지</a>

여기서 생성한 소리가 썩 좋지 않다는 것을 알아차렸을 것이다. 전화기를 들었을 때 같은 소리가 나고, 멈출 때 "뚝"(click) 소리를 듣게 되는데 이는 전혀 즐겁지 않다. 이유는 사람의 청각은 이 [훌륭한 글](http://alemangui.github.io/blog//2015/12/26/ramp-to-value.html)에서 설명한 것과 같이 반응하기 때문이다. 기본적으로, 사인파가 0이 되는 지점 외에서 소리를 멈췄을 때 당신은 이 뚝 소리를 듣게 된다.

![사인파의 뚝 하는 잡음](/assets/2018-06-02-generate-sounds-programmatically-with-javascript/audio_click.jpg)

## 뚝 소리 없애기

뚝 소리를 없애기 위한 최고의 해답은 지수함수를 이용해 사인파를 감소시키는 것이다. [이 문서](https://developer.mozilla.org/en-US/docs/Web/API/AudioParam/exponentialRampToValueAtTime)에 나오는 `AudioParam.exponentialRampToValueAtTime()`를 사용한다.

이번엔 우리는 oscillator에 [gain node](https://developer.mozilla.org/en-US/docs/Web/API/GainNode)를 추가할 필요가 있다. Gain node를 사용하면 문서에 있는 이 도표에 나온 것처럼 신호의 볼륨을 바꿀 수 있다.

![Gain node](/assets/2018-06-02-generate-sounds-programmatically-with-javascript/gain.jpg)

소리를 시작하는 코드는 이제 이렇게 바뀌었다:

```javascript
var context = new AudioContext()
var o = context.createOscillator()
var  g = context.createGain()
o.connect(g)
g.connect(context.destination)
o.start(0)
```

<a href="#!" class="js_play_sound" onclick="example2()">재생</a>

소리를 멈추기 위해 우리는 gain 값을 변경해 효과적으로 볼륨을 낮출 수 있다. 이 함수에는 값이 항상 양수여야 하는 제약이 있기 때문에 0으로 감소시키지 않는다는 것을 참고하자.

```javascript
g.gain.exponentialRampToValueAtTime(
  0.00001, context.currentTime + 0.04
)
```

<a href="#!" class="js_play_sound" onclick="example2Stop(0.04)">정지</a>

직접 들었듯이 뚝 소리가 사라졌다! 하지만 지수 감소를 이용해 할 수 있는 흥미로운 일은 이뿐만이 아니다.

## 울리는 효과 설정하기

위 예제에서는 소리를 `0.04`초 동안 아주 빠르게 멈췄다. 하지만 이 `X` 값을 바꾸면 어떻게 될까?

```javscript
g.gain.exponentialRampToValueAtTime(0.00001, context.currentTime + X)
```

<a href="#!" class="js_play_sound" onclick="example2()">재생</a>
<a href="#!" class="js_play_sound" onclick="example2Stop(0.1)">정지 (X=0.1)</a>
<a href="#!" class="js_play_sound" onclick="example2Stop(1)">정지 (X=1)</a>
<a href="#!" class="js_play_sound" onclick="example2Stop(5)">정지 (X=5)</a>

### 타격음

소리가 작아질 때의 시간을 길게 주면 전혀 다른 느낌을 준다. 신호가 시작하자마자 정지해 보면 더 확연히 드러난다.

<a href="#!" class="js_play_sound" onclick="example3('sine', 0.08)">시작하고 빠르게 정지</a>
<a href="#!" class="js_play_sound" onclick="example3('sine', 1.5)">시작하고 느리게 정지</a>

첫 번째는 톡 하는 잡음 같고 다른 소리는 실제 악기에서 연주되는 음 같다.

### 다양한 Oscilator

지금까지 우리는 주로 사인파 신호를 사용하였다. 하지만 다른 선택지도 있다:

![Oscilator의 종류](/assets/2018-06-02-generate-sounds-programmatically-with-javascript/signals.png)

`o.type = type`를 설정하여 oscilator의 종류를 바꿔가며 재생해 보면 더욱 흥미롭다.

<a href="#!" class="js_play_sound" onclick="example3('sine', 1.5)">Sine</a>
<a href="#!" class="js_play_sound" onclick="example3('square', 1.5)">Square</a>
<a href="#!" class="js_play_sound" onclick="example3('triangle', 1.5)">Triangle</a>
<a href="#!" class="js_play_sound" onclick="example3('sawtooth', 1.5)">Sawtooth</a>

# 실제 음계 재생하기

위 코드로 멋진 음을 내는 것이 매우 간단해졌다. 하지만 대체 무슨 음을 재생하고 있던 것일까? 이제 진동수를 고려해봐야 할 때다. 이를테면 어느 사람은 A4가 440Hz라는 것을 알지만, 더 많은 음이 존재한다.

### 음의 Hz 진동수

다음의 표를 이용하면 어떤 음이라도 진동수를 이용하여 재생 가능한 매핑을 쉽게 생성할 수 있다. 나는 해커톤에서 간단한 해시 매핑을 사용하였고 이 [gist](https://gist.github.com/marcgg/94e97def0e8694f906443ed5262e9cbb)에서 볼 수 있다.

<table class="data_table">
<tbody>
    <tr>
        <td></td><th>C</th><th>C#</th><th>D</th><th>Eb</th><th>E</th><th>F</th><th>F#</th><th>G</th><th>G#</th><th>A</th><th>Bb</th><th>B</th>
    </tr>
    <tr>
        <th>0</th><td>16.35</td><td>17.32</td><td>18.35</td><td>19.45</td><td>20.60</td><td>21.83</td><td>23.12</td><td>24.50</td><td>25.96</td><td>27.50</td><td>29.14</td><td>30.87</td>
    </tr>
    <tr>
        <th>1</th><td>32.70</td><td>34.65</td><td>36.71</td><td>38.89</td><td>41.20</td><td>43.65</td><td>46.25</td><td>49.00</td><td>51.91</td><td>55.00</td><td>58.27</td><td>61.74</td>
    </tr>
    <tr>
        <th>2</th><td>65.41</td><td>69.30</td><td>73.42</td><td>77.78</td><td>82.41</td><td>87.31</td><td>92.50</td><td>98.00</td><td>103.8</td><td>110.0</td><td>116.5</td><td>123.5</td>
    </tr>
    <tr>
        <th>3</th><td>130.8</td><td>138.6</td><td>146.8</td><td>155.6</td><td>164.8</td><td>174.6</td><td>185.0</td><td>196.0</td><td>207.7</td><td>220.0</td><td>233.1</td><td>246.9</td>
    </tr>
    <tr>
        <th>4</th><td>261.6</td><td>277.2</td><td>293.7</td><td>311.1</td><td>329.6</td><td>349.2</td><td>370.0</td><td>392.0</td><td>415.3</td><td>440.0</td><td>466.2</td><td>493.9</td>
    </tr>
    <tr>
        <th>5</th><td>523.3</td><td>554.4</td><td>587.3</td><td>622.3</td><td>659.3</td><td>698.5</td><td>740.0</td><td>784.0</td><td>830.6</td><td>880.0</td><td>932.3</td><td>987.8</td>
    </tr>
    <tr>
        <th>6</th><td>1047</td><td>1109</td><td>1175</td><td>1245</td><td>1319</td><td>1397</td><td>1480</td><td>1568</td><td>1661</td><td>1760</td><td>1865</td><td>1976</td>
    </tr>
    <tr>
        <th>7</th><td>2093</td><td>2217</td><td>2349</td><td>2489</td><td>2637</td><td>2794</td><td>2960</td><td>3136</td><td>3322</td><td>3520</td><td>3729</td><td>3951</td>
    </tr>
    <tr>
        <th>8</th><td>4186</td><td>4435</td><td>4699</td><td>4978</td><td>5274</td><td>5588</td><td>5920</td><td>6272</td><td>6645</td><td>7040</td><td>7459</td><td>7902</td>
    </tr>
</tbody>
</table>

이를 구현하려면 oscilator에 진동수를 추가하기만 하면 된다.

```javascript
var frequency = 440.0
o.frequency.value = frequency
```

진동수 값을 변경한다면 어떤 음도 재생할 수 있다. 예를 들면:

<a href="#!" class="js_play_sound" onclick="example4(261.6, 'sine')">261.6Hz (C4)</a>
<a href="#!" class="js_play_sound" onclick="example4(440.0, 'sine')">440Hz (A4)</a>
<a href="#!" class="js_play_sound" onclick="example4(830.6, 'sine')">830.6Hz (G#5)</a>

볼륨이 감소하는 시점과 서로 다른 신호를 조합하면 더 흥미로운 소리를 만들 수 있다.

<a href="#!" class="js_play_sound" onclick="example4(261.6, 'square')">174.6Hz (F3) - Square</a>
<a href="#!" class="js_play_sound" onclick="example4(1109, 'sawtooth')">1109Hz (C#6) - Sawtooth</a>
<a href="#!" class="js_play_sound" onclick="example4(87.31, 'triangle')">87.31 Hz (F2) - Triangle</a>

---