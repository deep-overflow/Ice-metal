# A Freestanding Rust Binary

우리만의 operating system kernel을 만드는 과정에서의 첫 단계는 표준 라이브러리에 링크하지 않는 러스트 실행파일을 만드는 것이다. 이를 통해 bare metal에서 operating system 없이 러스트 코드를 실행할 수 있다.

* bare metal이란 로직 하드웨어에 operating system 없이 직접적으로 명령을 실행하는 컴퓨터를 의미한다.

## Introduction

operating system kernel을 만들기 위해 어떤 operating system의 특성에도 의존하지 않는 코드를 작성해야 한다. 이는 OS abstractions나 specific hardware에 의존적인 threads, files, heap memory, the network, random numbers, standard output 또는 어떠한 다른 특성들을 사용할 수 없다. 우리는 우리만의 OS와 우리만의 drivers를 작성하려는 것이기 때문에 당연한 것이다.

대부분의 Rust standard library를 사용할 수 없다. 그러나 우리가 사용할 수 있는 러스트의 특성들은 많다. 예를 들어, iterators, closures, pattern matching, option, result, string formatting, ownership system을 사용할 수 있다. 이러한 특성들을 통해 undefined behavior 또는 memory safety에 대한 걱정 없이 very expressive, high level way의 kernel을 작성할 수 있다.

러스트로 OS kernel을 만들기 위해, operating system 없이 실행할 수 있는 실행파일을 만들 수 있어야 한다. 그러한 실행파일은 "freestanding" 또는 "bare-metal" 실행파일이라고 불린다.

## Disabling the Standard Library

기본적으로 모든 러스트 크레이트들은 표준 라이브러리를 링크한다. 표준 라이브러리는 threads, files 또는 networking과 같은 특성을 위해 operating system에 의존한다. 표준 라이브러리는 C standard library인 libc에도 의존하는데, 이는 OS services와 밀접하게 상호작용한다. 우리의 목표는 operating system을 작성하는 것이기 때문에, 어떤 OS에 의존하는 라이브러리는 사용할 수 없다. 따라서 no_std attribute를 통해 표준 라이브러리가 자동으로 포함되는 것을 막을 수 있다.

새로운 cargo application project를 만든다.

    cargo new ice-metal --bin --edition 2018

--bin 플래그는 (라이브러리와 대조되는) 실행 가능한 바이너리를 만든다는 것을 명시하고 --edition 2018 플래그는 러스트의 2018 edition을 사용한다는 것을 명시한다. 위 명령어를 실행하면, cargo는 다음과 같은 디렉터리 구조를 생성한다.

    blog_os
    ├── Cargo.toml
    └── src
        └── main.rs

Cargo.toml은 crate name, the author, the semantic version number과 dependencies와 같은 crate configuration을 포함한다. src/main.rs 파일은 우리 크레이트의 루트 모듈과 main 함수를 포함한다. cargo build를 통해 크레이트를 컴파일할 수 있고 target/debug 하위 폴더 안에 있는 컴파일된 바이너리를 실행할 수 있다.

### The no_std Attribute

현재 우리 크레이트는 은연 중에 표준 라이브러리를 링크하고 있다. no_std attribute를 추가하여 이를 막을 수 있다.

    // main.rs

    #![no_std]

    fn main() {
        println!("Hello, world!");
    }

이것을 cargo build를 통해 빌드하면 다음과 같은 에러가 발생한다.

    error: cannot find macro `println!` in this scope
    --> src/main.rs:4:5
    |
    4 |     println!("Hello, world!");
    |     ^^^^^^^

이 에러의 원인은 println 매크로가 더 이상 포함되지 않는 표준 라이브러리의 일부이지 때문이다. 따라서 더 이상 무언가를 출력할 수 없다. println은 operating system으로부터 제공 받는 특별한 파일 작성자인 standard output을 통해 출력하기 때문에 당연한 소리이다.

따라서 출력을 제거하고 빈 main 함수에 대해 다시 시도해보자.

    // main.rs
    
    #![no_std]

    fn main() {}

이번에는 컴파일러가 #[panic_handler] 함수와 language item을 찾지 못하고 있다.

    > cargo build
    error: `#[panic_handler]` function required, but not found
    error: language item required, but not found: `eh_personality`

## Panic Implementation


## The eh_personality Language Item

### Disabling Unwinding

## The start attribute

### Overwriting the Entry Point

## Linker Errors

### Building for a Bare Metal Target

### Linker Arguments

## Summary
