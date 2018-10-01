---
layout: post
title: "Fizzlefade"
categories: translate
published: true
---

<style type="text/css">
.center {
    display: block;
    margin: auto;
}
</style>

원문: [Fizzlefade](http://fabiensanglard.net/fizzlefade/)

# FIZZLEFADE

저는 많은 소스 코드를 읽는 것을 좋아합니다. 그리고 15년 동안 일하면서도 충분한 양을 본 것 같습니다. 정규직으로 일하면서도 저는 저녁 시간을 여기저기서 읽으면서 보내려 했습니다. 제가 멈추는 것을 볼 수 없죠. 그것은 항상 누군가의 생각 과정을 따라가며 새로운 것을 배울 기회입니다.

가끔 어떤 문제에 대한 "아름답다" 말고는 표현할 다른 말이 없는 매우 우아하고 창의적인 해결책과 우연히 마주치곤 합니다. "Inverse Square Root"로 더 많이 알려져 있고 Quake 3로 인해 유명해진 [Q_rsqrt](https://en.wikipedia.org/wiki/Fast_inverse_square_root)도 분명 놀라운 코드에 속합니다. 제가 [Game Engine Black Book: Wolfenstein 3D](https://play.google.com/store/books/details/Fabien_Sanglard_Game_Engine_Black_Book?id=Lq4yDwAAQBAJ)를 작업하던 중에 또 하나와 마주쳤는데요, Fizzlefade입니다.

Fizzlefade는 Wolfenstein 3D에서 한 장면에서 다른 장면으로 페이드하는 역할을 맡고 있는 함수의 이름입니다. 이 함수는 스크린의 픽셀을 무작위처럼 보이게 한 번에 하나씩 골라 단색으로 바꿉니다.

### // Fizzle이 뭐지 ?!

Wolfenstein 3D에서 대부분의 화면 전환은 (팔레트를 시프팅하여) 검정색으로 페이드되는데요, fizzling으로 화면 전환이 이루어지는 경우는 두 가지가 있습니다.

- 죽었을 때
- 보스를 죽였을 때

아래는 fizzling을 보여드리기 위한 움직이는 스크린샷입니다. 전환 중에 화면 상의 각 픽셀은 붉은 색 (죽었을 때) 또는 파랑색 (보스를 처치했을 때) 으로 변합니다. 각 픽셀은 한 번에 하나씩 칠해지고 무작위로 정해지는 것처럼 보입니다.

![](/assets/2018-10-01-fizzlefade/die.gif){: width="640px" height="480px"}{: .center}

이 효과를 구현하기 위한 나이브한 접근법으로는, 유사난수 생성기 [US_RndT](https://github.com/id-Software/wolf3d/blob/05167784ef009d0d0daefe8d012b027f39dc8541/WOLFSRC/ID_US_A.ASM#L85)를 사용하면서 어느 픽셀이 fizzle되었는지 추적하면 될 것입니다. 하지만 그렇게 하면 페이드 시간이 비결정적이 될 것이며 동일한 픽셀 좌표 (X,Y) 가 몇 번 발생할 것이므로 CPU 사이클을 낭비할 것입니다. 유사난수 생성기를 구현하기 위한 위한 더 빠르고 더 우아한 방법이 있습니다. 이 효과를 담당하는 코드는 [id_vh.cpp](https://github.com/id-Software/wolf3d/blob/05167784ef009d0d0daefe8d012b027f39dc8541/WOLFSRC/ID_VH.C) 안의 함수 [FizzleFade](https://github.com/id-Software/wolf3d/blob/05167784ef009d0d0daefe8d012b027f39dc8541/WOLFSRC/ID_VH.C#L471)에서 찾을 수 있습니다. 처음에는 어떻게 동작하는지 이해가 잘 되지 않습니다.

```c
boolean FizzleFade {
  long rndval = 1;
  int x , y ;
  do
  {
    // seperate random value into x/y pair
    asm mov ax ,[ WORD PTR rndval ]
    asm mov dx ,[ WORD PTR rndval +2]
    asm mov bx , ax
    asm dec bl
    asm mov [ BYTE PTR y ], bl // low 8 bits - 1 = y
    asm mov bx , ax
    asm mov cx , dx
    asm mov [ BYTE PTR x ], ah // next 9 bits = x
    asm mov [ BYTE PTR x +1] , dl

    // advance to next random element
    asm shr dx ,1
    asm rcr ax ,1
    asm jnc noxor
    asm xor dx ,0x0001
    asm xor ax ,0x2000

    noxor :
    asm mov [ WORD PTR rndval ] , ax
    asm mov [ WORD PTR rndval +2] , dx

    if (x > width || y > height ) continue ;

    fizzle_pixel (x , y ) ;

    if ( rndval == 1) return false ; // end sequence

  } while (1)
}
```

당신이 16비트 TASM을 읽을 수 없다면 (비난하지는 않겠습니다), 여기 C로 작성된 코드가 있습니다.

```c
boolean fizzlefade(void)
{
  uint32_t rndval = 1;
  uint16_t x,y;
  do
  {
     y =  rndval & 0x000FF;  /* Y = low 8 bits */
     x = (rndval & 0x1FF00) >> 8;  /* X = High 9 bits */
     unsigned lsb = rndval & 1;   /* Get the output bit. */
     rndval >>= 1;                /* Shift register */
     if (lsb) {                 /* If the output is 0, the xor can be skipped. */
          rndval ^= 0x00012000;
      }
      if (x < 320 && y < 200)
        fizzle_pixel(x , y) ;
  } while (rndval != 1);

  return 0;
}
```

코드를 읽어 보면 이렇습니다:

- rndval을 1로 초기화한다.
- 이것을 9 + 8 비트로 분해한다. 8비트를 Y좌표 생성에 사용하고 9비트를 X좌표에 사용한다. 이 픽셀을 붉게 만든다.
- rndval에 XOR 연산을 끼얹는다.
- rndval의 값이 어찌저찌 1이 되면 멈춘다. 화면은 완전히 붉은 색이다.

<br />

흑마법같은 느낌이 듭니다. 어떻게 rndval이 1로 돌아간다고 보장할 수 있을까요? 이 기법은 [Linear Feedback Shift Register](https://en.wikipedia.org/wiki/Linear-feedback_shift_register)라고 합니다. 상태를 저장하고 다음 상태를 생성하고 값까지 생성하는 데에 하나의 레지스터를 사용합니다. 다음 값을 얻기 위해서는 오른쪽 시프트를 하면 됩니다. 가장 오른쪽의 비트가 사라지므로 왼쪽에 새로운 비트가 필요합니다. 이 새 비트를 생성하기 위해 레지스터는 XOR에 사용될 비트들의 오프셋인 "탭"을 사용합니다. 여기 두 개의 탭을 가진 피보나치 형식의 간단한 LFSR이 있습니다.

![](/assets/2018-10-01-fizzlefade/4bits_lfsr.svg){: .center}

(0번과 2번 비트가 탭인) 이 레지스터는 6개의 값을 생성하고 원래의 상태로 돌아옵니다. 아래 생성하는 값의 목록이 있습니다. (별표는 탭의 위치를 나타냅니다)

```
 * * | value
======================
0001 | 1
1000 | 8
0100 | 4
1010 | A
0101 | 5
0010 | 2
0001 | 1 -> CYCLE

Sequence of 6 numbers before cycling .
```

탭을 다양하게 조절하면 다른 수열을 얻을 수 있습니다. 이 4비트 레지스터의 경우 최대로 가질 수 있는 값의 종류는 16-1 = 15개입니다 (0은 될 수 없습니다.) 이렇게 하려면 0, 1번 비트를 탭으로 정하면 됩니다. 이는 "최대 길이" LFSR라고 합니다.

```
  ** | value
======================
0001 | 1
1000 | 8
0100 | 4
0010 | 2
1001 | 9
1100 | C
0110 | 6
1011 | B
0101 | 5
1010 | A
1101 | D
1110 | E
1111 | F
0111 | 7
0011 | 3
0001 | 1 -> CYCLE

Sequence of 15 numbers before cycling .
```

Wolf는 유사난수 수열을 생성하기 위해 2개의 탭이 있는 17비트 최대 길이 LFSR을 사용했습니다. 여기서는 각 단계마다 8비트를 Y좌표 생성에, 9비트를 X좌표에 사용합니다. 그리고 화면 상의 대응하는 픽셀을 빨강/파랑색으로 바꿉니다.

![](/assets/2018-10-01-fizzlefade/fibonacci.svg){: .center}

이 피보나치 형식의 LFSR은 전반적인 이해에 도움을 줍니다. 하지만 LFSR을 소프트웨어로 구현할 때는 보통 이런 식으로 만들지 않습니다. 이렇게 하면 탭의 숫자에 따라 선형 증가를 보이기 때문입니다. 4개의 탭을 사용한다면 연속한 3개의 XOR 연산이 필요합니다.

![](/assets/2018-10-01-fizzlefade/fibonacci_hard.svg){: .center}

"갈루아"라고 하는, LFSR를 기술하기 위한 또 다른 방법이 있는데, 탭 숫자와 상관없이 하나의 XOR만 있으면 됩니다. Wolfenstein 3D에서는 이 방법을 사용하여 320x200=64000개의 픽셀을 결정론적 시간에 정확히 한 번씩만 칠합니다.

![](/assets/2018-10-01-fizzlefade/galois.svg){: .center}

**<u>참고 :</u>** 개발자들이 이 게임을 하드웨어 가속을 지원하는 GPU로 포팅하면서 이 효과를 모방하기가 어려웠는데, 픽셀을 하나하나 그려내기 때문입니다. Wolf4SDL에서는 320x200보다 높은 해상도를 지원하는 LFSR 탭 설정을 찾아냈고, 이를 제외하면 fizzlefade를 모방해낸 포팅은 존재하지 않습니다.

**<u>참고 :</u>** 이 17비트에서의 탭 설정은 사이클 전까지 131,072개의 값을 생성합니다. 320x200=64000이므로 이는 16,15,13,4를 탭으로 사용한 ("갈루아" 형식의) 16비트 최대 길이 레지스터로 구현할 수도 있었습니다. 제 생각에는 LFSR에 대한 문헌을 1991/1992년에는 찾아보기 힘들었으며 16비트 최대 길이 레지스터에 대한 알맞은 탭을 찾는 노력을 할 가치는 없었을 것입니다.