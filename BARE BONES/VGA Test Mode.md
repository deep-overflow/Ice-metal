VGA Text Mode
=============

VGA text 모드는 화면에 텍스트를 출력하는 가장 간단한 방식이다. 이 포스트에서는 안전하고 간단하게 사용하기 위한 인터페이스를 만들 것이다.

# The VGA Text Buffer

VGA text 모드에서 화면에 문자를 출력하기 위해서는 VGA hardware의 text buffer에 문자를 써야 한다. VGA text buffer는 화면에 바로 렌더링되는 25행 80열의 2차원 배열이다. 각각의 배열 entry는 다음과 같은 형식으로 화면 문자를 표현한다.

* 0-7 Bit(s) : ASCII code point
* 8-11 Bit(s) : Foreground color
* 12-14 Bit(s) : Background color
* 15 Bit(s) : Blink

첫 번째 바이트는 ASCII encoding으로 출력되는 문자를 표현한다. 정확하게는 ASCII는 아니고, code page 437이라는 문자 집합이다. 간단하게 하기 위해 이 포스트에서는 ASCII 문자로 부른다.

두 번째 바이트는 문자가 어떻게 나타나는지를 표현한다. 처음 4개 비트는 foreground color을, 다음 3개 비트는 background color을, 마지막 비트는 문자가 깜빡거리는지를 정의한다.

* 0x0 Black
* 0x1 Blue
* 0x2 Green
* 0x3 Cyan
* 0x4 Red
* 0x5 Magenta
* 0x6 Brown
* 0x7 Light Gray
* 0x8 Dark Gray
* 0x9 Light Blue
* 0xa Light Green
* 0xb Light Cyan
* 0xc Light Red
* 0xd Pink
* 0xe Yellow
* 0xf White

Bit 4는 bright bit로 예를 들어 blue를 light blue로 바꾼다. background color에서 이 비트는 blink bit로 바뀐다.

VGA text buffer는 memory-mapped I/O를 통해 `0x8000` 주소로 접근 가능하다. 이는 해당 주소를 읽고 쓰는 것이 RAM에 접근하는 것이 아니라 VGA hardware 상의 text buffer에 직접적으로 접근한다는 것을 의미한다.

memory-mapped hardware는 모든 일반적인 RAM 연산들을 지원하는 것은 아니다. 예를 들어, 디바이스가 byte-wise reads만 지원하고 `u64`를 읽으면 쓰레기값을 반환할 수도 있다. 다행히, text buffer는 일반적인 읽기, 쓰기를 지원하고 특별한 경우로 처리할 필요없다.

# A Rust Module

이제 VGA buffer가 어떻게 작동하는지 알기 때문에, 출력을 다루기 위한 러스트 모듈을 만들 수 있다.

```rust
// in src/main.rs
mod vga_buffer;
```

이 모듈의 내용을 위해 새로운 `src/vga_buffer.rs`를 생성한다. 아래의 모든 코드들은 새로운 모듈로 들어간다.

## Colors

먼저, enum을 통해 서로 다른 색을 나타낸다.

```rust
// in src/vga_buffer.rs

#[allow(dead_code)]
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
#[repr(u8)]
pub enum Color {
    Black = 0,
    Blue = 1,
    Green = 2,
    Cyan = 3,
    Red = 4,
    Magenta = 5,
    Brown = 6,
    LightGray = 7,
    DarkGray = 8,
    LightBlue = 9,
    LightGreen = 10,
    LightCyan = 11,
    LightRed = 12,
    Pink = 13,
    Yellow = 14,
    White = 15,
}
```

각각의 색깔에 대한 숫자를 명확하게 명시하기 위해 C-like enum을 사용한다. `repr(u8)` attribute 때문에 각각의 enum variant는 `u8`로 저장된다. 4 비트도 충분하지만, 러스트는 `u4`를 가지지 않는다.

일반적으로 컴파일러는 사용하지 않는 variant에 대해 경고를 보낸다. `#[allow(dead_code)]` attribute를 사용하여 `Color` enum에 대한 경고를 막는다.

++

full color code를 표현하기 위해

## Text Buffer
## Printing
## Bolatile
## Formatting Macros
## Newlines

# A Global Interface
## Lazy Statics
## Spinlocks
## Safety
## A println Macro
## Hello World using println
## Printing Panic Messages