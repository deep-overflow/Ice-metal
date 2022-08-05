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

```rust
// main.rs

#![no_std]

fn main() {
    println!("Hello, world!");
}
```

이것을 cargo build를 통해 빌드하면 다음과 같은 에러가 발생한다.

    error: cannot find macro `println!` in this scope
    --> src/main.rs:4:5
    |
    4 |     println!("Hello, world!");
    |     ^^^^^^^

이 에러의 원인은 println 매크로가 더 이상 포함되지 않는 표준 라이브러리의 일부이지 때문이다. 따라서 더 이상 무언가를 출력할 수 없다. println은 operating system으로부터 제공 받는 특별한 파일 작성자인 standard output을 통해 출력하기 때문에 당연한 소리이다.

따라서 출력을 제거하고 빈 main 함수에 대해 다시 시도해보자.

```rust
// main.rs

#![no_std]

fn main() {}
```

이번에는 컴파일러가 #[panic_handler] 함수와 language item을 찾지 못하고 있다.

    > cargo build
    error: `#[panic_handler]` function required, but not found
    error: language item required, but not found: `eh_personality`

## Panic Implementation

panic_handler attribute는 panic이 발생했을 때 컴파일러가 불러야 하는 함수를 정의한다. 표준 라이브러리는 자신만의 panic handler 함수를 제공하지만, no_std 환경에서 이를 정의할 필요가 있다.

```rust
// in main.rs

use core::panic::PanicInfo;

// This function is called on panic.
#[panic_handler]
fn panic(_info: &PanicInfo) -> ! {
    loop {}
}
```

PanicInfo 파라미터는 panic이 발생한 파일과 줄 그리고 선택적인 panic 메시지를 포함한다. 함수는 절대 반환하지 않으므로 "never" 타입인 !를 반환하는 diverging 함수이다. 현재 우리가 이 함수 안에서 할 수 있는 것이 많지 않으므로 그냥 무한루프를 생성하도록 한다.

## The eh_personality Language Item

language item들은 컴파일러에 의해 내부적으로 요구되는 특별한 함수와 타입들이다. ++ 예를 들어 Copy 트레이트는 어떤 타입들이 copy semantics를 가지는 지 컴파일러에게 알려주는 language item이다. 구현 내용을 보면, 그것을 language item으로 정의한 특별한 #[lang = "copy"] attribute를 가진다. ++

language items를 직접 구현하는 것도 가능하지만, 이는 최후의 수단으로 사용하는 것이 좋다. ++ 그 이유는 language items가 매우 불안정한 구현이고 타입 ++ 다행히, 위 language item error를 해결할 수 있는 더 안정적인 방법이 있다.

eh_personality language item은 stack unwinding을 구현하는 데에 사용된 함수를 나타낸다. 기본적으로, 러스트는 panic에서 모든 살아있는 스택 변수의 소멸자를 실행하기 위해 unwinding을 사용한다. 이는 모든 사용된 메모리가 해제되는 것과 부모 thread가 panic을 해결하고 실행을 지속할 수 있도록 보장한다. 하지만, unwinding은 복잡한 process이고 몇몇의 OS specific 라이브러리를 필요로 한다. 따라서 우리는 이를 사용하지 않을 것이다.

### Disabling Unwinding

unwinding이 바람직하지 않은 다른 경우들도 있다. 따라서 러스트는 panic을 abort하는 옵션을 제공한다. 이는 unwinding symbol 정보가 생성되는 것을 막고 따라서 바이너리 사이즈를 상당히 줄여준다. unwinding을 막는 가장 쉬운 방법은 Cargo.toml에 다음 코드를 넣는 것이다.

```toml
[profile.dev]
panic = "abort"

[profile.release]
panic = "abort"
```

이는 cargo build를 사용하는 dev 프로필과 cargo build --release를 사용하는 release 프로필에서 panic strategy를 abort로 설정하는 것이다. 이제 eh_personality language item으 더 이상 필요하지 않다.

이제 위에서 본 에러들은 해결했다. 하지만, 컴파일을 실행하면 새로운 에러가 발생한다.

    > cargo build
    error: requires `start` lang_item

## The start attribute

누군가는 main 함수가 프로그램을 실행할 때 불리는 첫 번째 함수라고 생각할 것이다. 하지만, 대부분의 언어는 garbage collection이나 software threads와 같은 것들과 관련 있는 runtime system을 가진다. 이 runtime은 main 전에 실행되고, 따라서 스스로 초기화해야 할 필요가 있다.

표준 라이브러리를 링크하는 전형적인 러스트 바이너리에서, 실행은 C 어플리케이션을 위한 환경을 만드는 C runtime 라이브러리인 crt0("C runtime zero")에서 시작한다. 이것은 스택을 만들고 인자들을 정확한 레지스터에 등록하는 것을 포함한다. 이후 C runtime은 start language item에 의해 표시된 Rust runtime의 entry point를 실행시킨다. 러스트는 stack overflow guards나 backtrace on panic을 출력하는 작은 것들을 관리하는 최소한의 runtime만 가진다. runtime은 마지막에 main 함수를 실행한다.

freestanding 실행파일은 Rust runtime과 crt0에 대해 접근하지 않는다. 따라서 우리만의 entry point를 정의해야 한다. start language item을 구현하는 것은 start language item이 crt0을 필요로 하기 때문에 안된다. 대신 crt0 entry point를 직접 overwrite할 수 있다.

### Overwriting the Entry Point

Rust 컴파일러에게 표준 entry point chain을 사용하지 않을 것을 명시하기 위해 #![no_main] attribute를 추가한다.

```rust
#![no_std]
#![no_main]

use core::panic::PanicInfo;

/// This function is called on panic.
#[panic_handler]
fn panic(_info: &PanicInfo) -> ! {
    loop {}
}
```

main 함수는 underlying runtime 없이 필요 없기 때문에 제거한다. 대신 _start 함수를 통해 operating system entry point를 overwrite한다.

```rust
#[no_mangle]
pub extern "C" fn _start() -> ! {
    loop {}
}
```

#[no_mangle] attribute를 사용하여 name mangling을 막고 이를 통해 러스트 컴파일러가 _start라는 이름 그대로 함수를 만들도록 보장한다. 이 attribute 없이, 컴파일러는 모든 함수에 유일한 이름을 부여하기 위해 이 함수의 실제 이름을 _ZN3blog_os4_start7hb173fedf945531caE라는 이름으로 바꿔 생성한다. 이 attribute는 링커에게 entry point 함수의 이름을 전달하기 위해 필요하다.

함수를 extern "C"로 표시하여 컴파일러에게 (unspecified Rust calling convention 대신) C calling convention을 사용하도록 명시한다. 함수의 이름이 _start인 이유는 대부분의 시스템에서 기본 entry point 이름이 _start이기 때문이다.

! 반환 타입은 함수가 diverge함을 의미한다. 이렇게 하는 이유는 entry point는 어떤 함수에 의해 실행되는 것이 아니라 operating system이나 bootloader에 의해 직접적으로 실행되기 때문이다. 따라서 반환하는 것 대신, entry point는 operating system의 exit system call을 실행해야 한다. 우리의 경우, 머신을 셧다운시키는 것이 적절한 동작이지만 현재는 무한히 반복하는 것으로 필요사항을 충족시킨다.

cargo build를 실행하면 ungly linker error를 얻게 된다.

## Linker Errors

linker는 생성된 코드를 실행파일로 결합하는 프로그램이다. 시스템마다 실행파일의 형식이 다르기 때문에 각각의 시스템은 다른 에러를 던지는 각 시스템만의 linker를 가진다. 이 에러의 본질적인 원인은 같다: linker의 기본 configuration이 우리의 프로그램이 C runtime에 의존한다고 가정하는 것이다. (실제로 그렇지 않다.)

이 에러를 해결하기 위해, linker가 C runtime을 포함하지 않도록 알려줘야 한다. 이는 특정 인자들의 집합을 linker에 전달하거나 bare metal target을 위해 빌드하는 방식으로 할 수 있다.

### Building for a Bare Metal Target

기본적으로 러스트는 빌드한 실행파일이 현재 시스템 환경에서 동작할 수 있게 한다. 예를 들어, x86_64의 윈도우를 사용한다면, 러스트는 x86_64의 명령을 따르는 .exe 윈도우 실행파일을 만들 것이다.

다른 환경을 표현하기 위해 러스트는 target triple이라고 불리는 문자열을 사용한다. rustc --version --verbose를 실행하면 host system의 target triple을 볼 수 있다.

    rustc 1.35.0-nightly (474e7a648 2019-04-07)
    binary: rustc
    commit-hash: 474e7a6486758ea6fc761893b1a49cd9076fb0ab
    commit-date: 2019-04-07
    host: x86_64-unknown-linux-gnu
    release: 1.35.0-nightly
    LLVM version: 8.0

위 출력은 x86_64 리눅스 시스템의 것이다. 호스트의 triple이 x86_64-unknown-linux-gnu라는 것을 볼 수 있다. 이는 CPU architecture는 x86_64, vendor는 unknown, operating system은 linux 그리고 ABI는 gnu라는 것을 말한다.

host triple에 대해 컴파일하면 러스트 컴파일러와 linker는 C runtime을 기본적으로 사용하는 리눅스와 윈도우와 같은 operating system이 존재한다고 가정하고 때문에 linker error가 발생한다. 따라서 linker error를 피하기 위해, underlying operating system이 없는 다른 환경에 대해 컴파일할 수 있다.

bare metal 환경을 위한 예시는 embedded ARM system을 나타내는 thumbv7em-none-eabihf target triple이다. 구체적인 것은 중요하지 않고, 중요한 것은 저 target triple은 target triple에서 none이 표시하고 있는 underlying operating system이 없다는 것이다. 이것을 컴파일하기 위해서는 rustup에 포함해야 한다.

    rustup target add thumbv7em-none-eabihf

해당 시스템의 표준(그리고 핵심) 라이브러리의 사본을 다운받게 된다. 이제 freestanding 실행파일을 빌드할 수 있다.

    cargo build --target thumbv7em-none-eabihf

--target 인자를 전달하여 bare metal target system을 위한 실행파일을 cross compile할 수 있다. target system이 operating system이 없기 때문에, linker는 C runtime과 링크하지 않아도 되고 어떤 linker error 없이 빌드를 성공할 수 있다.

이것이 OS kernel을 빌드하기 위해 사용할 접근방식이다. thumbv7em-none-eabihf 대신, x86_64 bare metal environment를 나타내는 custom target을 사용할 것이다.

### Linker Arguments

bare metal system을 위해 컴파일하는 방식 대신, linker에 인자 집합을 전달하여 linker error들을 해결할 수 있다. 이것은 우리의 kernel을 위해 사용할 접근방식은 아니다.

## Summary

최소한의 freestanding Rust binary는 다음과 같다:

`src/main.rs`
```rust
#![no_std] // don't link the Rust standard library
#![no_main] // disable all Rust-level entry points

use core::panic::PanicInfo;

#[no_mangle] // don't mangle the name of this function
pub extern "C" fn _start() -> ! {
    // this function is the entry point, since the linker looks for a function
    // named `_start` by default
    loop {}
}

/// This function is called on panic.
#[panic_handler]
fn panic(_info: &PanicInfo) -> ! {
    loop {}
}
```

`Cargo.toml`
```rust
[package]
name = "crate_name"
version = "0.1.0"
authors = ["deep-overflow 2020320120@korea.ac.kr"]

# the profile used for `cargo build`
[profile.dev]
panic = "abort" # disable stack unwinding on panic

# the profile used for `cargo build --release`
[profile.release]
panic = "abort" # disable stack unwinding on panic
```

이 바이너리를 빌드하기 위해서는 `thumbv7em-none-eabihf`와 같은 bare metal target에 대해 컴파일해야 한다.

    cargo build --target thumbv7em-none-eabihf

대신, 추가적인 linker argument들을 전달하여 host system에 대하여 컴파일할 수 있다.

    # Linux
    cargo rustc -- -C link-arg=-nostartfiles
    # Windows
    cargo rustc -- -C link-args="/ENTRY:_start /SUBSYSTEM:console"
    # macOS
    cargo rustc -- -C link-args="-e __start -static -nostartfiles"

이것은 freestanding Rust binary의 가장 단순한 예시이다. 이 바이너리는 다양한 기능을 기대한다. 예를 들면, `_start`함수가 실행되면 stack이 초기화된다. 