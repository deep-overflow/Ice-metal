# A Minimal Rust Kernel

이번에는 x86 architecture을 위한 minimal 64-bit Rust kernel을 만든다. 이전 포스트에서 만든 freestanding Rust binary 위에 빌드하여 부팅이 가능한 무언가를 출력하는 disk image를 생성할 것이다.

## The Boot Process

사용자가 컴퓨터를 켜면, 마더보드 ROM에 저장되어 있는 firmware code가 실행되면서 시작된다. 이 code는 power-on self-test를 실행하고, 사용가능한 RAM을 감지하고 CPU와 하드웨어를 사전초기화한다. 이후에 부팅 가능한 disk를 탐색하고 operating system kernel을 부팅하기 시작한다.

x86에서, 2개의 firmware 표준이 있다: "Basic Input/Output System"(BIOS)와 "Unified Extensible Firmware Interface"(UEFI). BIOS 표준은 오래되었지만 간단하고 어떠한 x86 머신도 잘 지원한다. 반대로 UEFI는 현대적이고 더 많은 특성을 가지고 있지만 설정하기 너무 복잡하다.

현재, BIOS를 지원할 예정이지만, UEFI 지원도 예정되어 있다.

### BIOS Boot

대부분의 x86 system들은 BIOS booting을 지원한다. + 이는 지난 세기의 모든 머신에 대해 같은 부트 로직을 사용해도 됨을 의미한다. 하지만 이 넓은 호환성은 동시에 BIOS booting의 가장 큰 단점이기도 하다. 왜냐하면 이는 CPU가 부팅되기 전에 real mode라고 불리는 16-bit 호환 모드에 놓여 낡은 부트로더도 작동할 수 있게 하기 때문이다.

그러나 처음부터 시작해보자.

컴퓨터를 켜면, 마더보드에 위치한 특별한 flash memory로부터 BIOS를 로드한다. BIOS는 하드웨어의 self test와 초기화를 실행하고, 부팅 가능한 디스크를 탐색한다. 디스크를 찾으면 디스크의 시작 부분에 저장된 512-byte의 실행가능한 코드인 부트로더로 제어권이 넘어간다. 대부분의 부트로더는 512 bytes보다 커서 512 bytes인 첫 번째 부분과 나머지에 해당하는 두 번째 부분으로 나눠진다.

부트로더는 디스크 상에서 커널 이미지의 위치를 결정하고 메모리에 로드한다. CPU를 16-bit의 real mode에서 32-bit의 protected mode로 전환하고, 다시 64-bit registers와 완전한 메인 메모리가 사용 가능한 64-bit의 long mode로 전환할 필요가 있다. 세 번째 작업은 메모리 맵과 같은 특정한 정보를 BIOS로부터 OS kernel로 전달해줘야 한다.

부트로더를 작성하는 것은 어셈블리 언어를 요구하기도 하고 통찰력이 필요하지 않은 단계들 때문에 번거롭다. 따라서 이 포스트에서는 부트로더를 생성하는 것을 커버하지 않는다. 대신 자동으로 부트로더를 커널에 지원하는 bootimage라는 이름의 도구를 제공한다.

#### The Multiboot Standard

모든 operating system이 단 하나의 운영체제에만 적합한 자신만의 부트로더를 구현하는 것을 피하기 위해, Free Software Foundation은 Multiboot라고 불리는 오픈 부트로더 표준을 만들었다. 표준은 부트로더와 운영체제 간의 인터페이스를 정의하여 모든 Multiboot과 잘 맞는 부트로더는 모든 Multiboot과 잘 맞는 운영체제에도 로드할 수 있다. 참조할 수 있는 구현체는 리눅스 시스템에서 가장 유명한 부트로더인 GNU GRUB이다.

Multiboot에 잘 맞는 커널을 만들기 위해서는 Multiboot header라고 불리는 것을 커널 파일 초반에 넣으면 된다. 이것은 GRUB에서 어떤 OS를 부팅하기 쉽게 만든다. 그러나, GRUB와 Multiboot 표준은 몇 가지 문제를 가지고 있다.

* 32-bit protected mode만 지원한다. 이는 CPU configuration을 64-bit long mode로 전환해줘야 함을 의미한다.
* 커널 대신 부트로더를 간단하게 설계했다. ++
* +
* +

이러한 단점들 때문에 GRUB 또는 Multiboot 표준을 사용하지 않기로 결정했다. 그러나, Multiboot 지원을 bootimage 도구에 추가하기로 계획했기 때문에 GRUB system에 커널을 로드할 수 있다. 

### UEFI

## A Minimal Kernel

이제 대략적으로 어떻게 컴퓨터가 부트되는지 알게 되었다. 이번에는 최소한의 커널을 만들어보자. 목표는 부트되었을 때 화면에 "Hello World!"를 출력하는 디스크 이미지를 만드는 것이다. 이전 포스트의 freestanding Rust binary 위에 빌드할 것이다.

freestanding binary를 cargo를 통해 빌드하였지만 operating system에 의존하여 서로 다른 entry point 이름과 컴파일 플래그가 필요했다. 왜냐하면 cargo는 기본적으로 호스트 시스템을 위해 빌드하기 때문이다. 이는 커널은 다른 운영체제 위에서 동작하지 않기 때문에 커널을 위한 것이 아니다. 대신 완전히 정의된 target system에 대하여 컴파일해야 한다.

### Installing Rust Nightly

러스트는 3가지 배포 채널이 있다: stable, beta 그리고 nightly. operating system을 빌드하기 위해서는 nightly channel에서만 사용 가능한 특성들이 필요하다. 

러스트 설치를 관리하기 위해 rustup을 권장한다. rustup은 nightly, beta 그리고 stable 컴파일러를 설치할 수 있게 돕고 업데이트하는 것도 쉽다. `rustup override set nightly`를 실행함으로써 현재 디렉터리에서 nightly 컴파일러를 사용할 수 있다. 대안으로, `nightly`의 내용을 가진 `rust-toolchain`이라는 파일을 프로젝트 루트 디렉터리에 추가하면 된다. `rustc --version`를 실행해보면 nightly version이 설치되었는지 확인할 수 있다: 버전 넘버 마지막에 `-nightly`를 포함해야 한다.

nightly 컴파일러는 파일 상단에 feature flags를 사용하여 다양한 특성들을 사용할 수 있게 한다. 예를 들어, 인라인 어셈블리를 위해 `ams!` 매크로를 `#![feature(ams)]`를 추가하여 사용할 수 있게 한다. 이러한 실험적인 특성들은 불안정하다. 이러한 이유로 이러한 특성들이 절대적으로 필요한 경우에만 사용한다.

### Target Specification

Cargo는 다른 target system들을 `--target` 인자를 통해 지원한다. target은 target triple을 통해 표현된다. target triple은 CPU architecture, vendor, operating system 그리고 ABI를 나타낸다. 예를 들어, x86_64-unknown-linux-gnu라는 target triple은 시스템이 x86_64 CPU를 가지고 불분명한 vendor와 GNU ABI를 가지는 리눅스 운영체제를 가진다는 것을 나타낸다. 러스트는 다양한 target triple을 지원한다.

그러나 우리의 target system을 위해, 특별한 configuration parameter가 필요하고, 따라서 기존의 target triples은 맞는 것이 없다. 다행히, 러스트는 JSON 파일을 통해 우리만의 target을 정의할 수 있도록 한다. 예를 들어, x86_64-unknown-linux-gnu를 나타내는 JSON 파일은 다음과 같다.

```json
{
    "llvm-target": "x86_64-unknown-linux-gnu",
    "data-layout": "e-m:e-i64:64-f80:128-n8:16:32:64-S128",
    "arch": "x86_64",
    "target-endian": "little",
    "target-pointer-width": "64",
    "target-c-int-width": "32",
    "os": "linux",
    "executables": true,
    "linker-flavor": "gcc",
    "pre-link-args": ["-m64"],
    "morestack": false
}
```

대부분의 field는 LLVM이 해당 플랫폼을 위한 코드를 생성하기 위해 필요하다. 예를 들어 `data-layout` field는 다양한 정수, 부동 소수점 그리고 포인터 타입의 크기를 정의한다. 러스트가 조건이 있는 컴파일을 하기 위해 사용하는 `target-pointer-width`와 같은 field가 존재한다. + 예를 들어, `pre-link-args` field는 linker로 전달될 인자를 정의한다.

우리의 커널은 x86_64 system 또한 target으로 한다. 따라서 target specification은 위의 내용과 매우 유사하다. 먼저 아래 내용을 담은 json 파일로 시작하자.

```json
{
    "llvm-target": "x86_64-unknown-none",
    "data-layout": "e-m:e-i64:64-f80:128-n8:16:32:64-S128",
    "arch": "x86_64",
    "target-endian": "little",
    "target-pointer-width": "64",
    "target-c-int-width": "32",
    "os": "none",
    "executables": true
}
```

`llvm-target`에서 OS에 해당하는 부분과 `os` field를 `none`으로 바꾸었다. bare metal에서 실행할 것이기 때문이다.

다음과 같은 build-related entries를 추가하였다.

```json
"linker-flavor": "ld.lld",
"linker": "rust-lld",
```

플랫폼의 기본 linker를 사용하는 대신, 커널을 링크하기 위해 러스트와 함께 제공되는 cross platform LLD linker를 사용한다.

```json
"panic-strategy": "abort",
```

이 설정은 target이 panic에서 stack unwinding을 지원하지 않는 것을 명시하고 따라서 프로그램은 바로 abort해야 한다. 이는 `Cargo.toml`에서 `panic = "abort"` 옵션과 같은 효과를 가져온다. 따라서 해당 옵션을 제거해도 된다. (Cargo.toml에서의 옵션과 대조적으로 이 target 옵션은 이후에 `core` 라이브러리를 재컴파일할 때에도 적용된다. 따라서 Cargo.toml 옵션을 유지하고 싶다라도 이 옵션을 추가해줘야 한다.)

```json
"disable-redzone": true,
```

커널을 작성하면서 어떤 부분에서 인터럽트를 처리해줘야 한다. 이를 안전하게 하기 위해, stack corruptions를 발생시킬 수 있기 때문에 "red zone"이라고 불리는 stack pointer optimization을 막아야 한다. ++

```json
"features": "-mmx,-sse,+soft-float",
```

`features` field는 target features를 가능하게 하거나 가능하지 않게 한다. `mmx`와 `sse`라는 features는 가능하지 않게 하고 `soft-float`라는 feature는 가능하게 한다. 다른 플래그 사이에 빈 공간이 있으면 LLVM이 features string을 해석하는 데 실패할 수 있다.

`mmx`와 `sse`는 프로그램의 속도를 상당히 빠르게 해주는 Single Instruction Multiple Data(SIMD)에 대한 지원을 결정한다. 그러나 OS kernel에서 큰 SIMD register들은 성능 문제를 야기한다. 그 이유는 커널은 인터럽트된 프로그램을 지속하기 전에 모든 레지스터들을 원래대로 되돌려놔야 하기 때문이다. 이는 커널이 메인 메모리에 매 시스템 콜 또는 하드웨어 인터럽트마다 완전한 SIMD 상태를 저장해야 함을 의미한다. SIMD 상태는 매우 크고(512-1600bytes) 인터럽트는 매우 자주 발생하기 때문에 이러한 저장 및 회복 연산은 상당히 성능을 나쁘게 만든다. 이를 피하기 위해 SIMD를 막는다.

SIMD가 막힘으로 인해 발생하는 문제는 `x86_64`에서 부동 소수점 연산을 위해 SIMD 레지스터가 기본적으로 필요하다는 것이다. 이를 해결하기 위해 일반적인 정수에 기반한 소프트웨어 함수들을 통해 부동 소수점 연산을 하는 `soft-float` 특성이 필요하다.

#### Putting it Together

우리의 target specification file은 아래와 같다:

```json
{
    "llvm-target": "x86_64-unknown-none",
    "data-layout": "e-m:e-i64:64-f80:128-n8:16:32:64-S128",
    "arch": "x86_64",
    "target-endian": "little",
    "target-pointer-width": "64",
    "target-c-int-width": "32",
    "os": "none",
    "executables": true,
    "linker-flavor": "ld.lld",
    "linker": "rust-lld",
    "panic-strategy": "abort",
    "disable-redzone": true,
    "features": "-mmx,-sse,+soft-float"
}
```

### Building our Kernel

새로운 target을 컴파일할 때 리눅스 convention을 사용할 것이다. 이는 `_start`라는 entry point가 필요하다는 것을 의미한다.

```rust
// src/main.rs

#![no_std] // don't link the Rust standard library
#![no_main] // disable all Rust-level entry points

use core::panic::PanicInfo;

/// This function is called on panic.
#[panic_handler]
fn panic(_info: &PanicInfo) -> ! {
    loop {}
}

#[no_mangle] // don't mangle the name of this function
pub extern "C" fn _start() -> ! {
    // this function is the entry point, since the linker looks for a function
    // named `_start` by default
    loop {}
}
```

호스트 운영체제에 상관 없이 entry point는 _start로 불려야 한다.

json 파일의 이름을 `--target`으로 전달함으로써 새로운 target에 대하여 커널을 빌드할 수 있다.

    > cargo build --target x86_64-blog_os.json

    error[E0463]: can't find crate for `core`

에러는 러스트 컴파일러가 `core` 라이브러리를 찾을 수 없다고 말하고 있다. 이 라이브러리는 Result, Option 그리고 iterators와 기본 러스트 타입을 포함하고 있고 모든 `no_std` 크레이트에 링크되어 있다.

문제는 core 라이브러리는 사전에 컴파일된 라이브러리로 러스트 컴파일러와 함께 나눠졌다는 것이다. 따라서, 지원되는 호스트 triples에 대해서는 유효하지만 우리의 target에 대해서는 유효하지 않다. 만약 다른 target에 대하여 컴파일된 코드를 원한다면, 먼저 다른 target에 대해 `core`을 재컴파일해야 한다.

#### The `build-std` Option

이것이 cargo의 `build-std` feature이 등장하는 시점이다. 이를 통해 러스트 설치와 함께 설치된 사전에 컴파일된 버전을 사용하는 대신 `core`와 다른 표준 라이브러리 크레이트들을 필요에 따라 재컴파일할 수 있다. 이 특성은 불안정하여 nightly Rust compiler에서만 사용 가능하다.

이 특성을 하용하기 위해 `.cargo/config.toml`에 다음과 같은 내용을 담은 cargo configuration 파일을 만들어야 한다.

    # in .cargo/config.toml

    [unstable]
    build-std = ["core", "compiler_builtins"]

이것은 cargo가 `core`와 `compiler_builtins` 라이브러리들을 재컴파일해야 함을 알려준다. 후자는 `core`의 dependency이기 때문에 필요하다. 이러한 라이브러리들을 재컴파일하기 위해, cargo는 우리가 `rustup component add rust-src`를 통해 설치할 수 있는 러스트 소스 코드에 접근해야 한다.

unstable.build-std configuration을 설정하고 rust-src component를 설치한 이후, 빌드를 다시 실행하면 된다.

    > cargo build --target x86_64-blog_os.json
    Compiling core v0.0.0 (/…/rust/src/libcore)
    Compiling rustc-std-workspace-core v1.99.0 (/…/rust/src/tools/rustc-std-workspace-core)
    Compiling compiler_builtins v0.1.32
    Compiling blog_os v0.1.0 (/…/blog_os)
        Finished dev [unoptimized + debuginfo] target(s) in 0.29 secs

++

#### Memory-Related Intrinsics

러스트 컴파일러는 built-in 함수들의 특정 집합은 모든 시스템에서 사용 가능하다고 가정한다. 이런 함수들 중 대부분은 우리가 재컴파일한 `compiler_builtins` 크레이트가 제공한다. 그러나, 이 크레이트에는 기본적으로 사용할 수 없는 메모리와 관련된 함수들이 있는데, 이는 시스템 상에 있는 C 라이브러리에 의해 제공되기 때문이다. 이러한 함수들에는 메모리 블록에 있는 모든 바이트를 주어진 값으로 설정하는 `memset`, 하나의 메모리 블록을 다른 메모리 블록으로 복사하는 `memcpy`, 두 개의 메모리 블록을 비교하는 `memcmp` 등이 포함된다. 지금 당장은 커널을 컴파일하기 위해 이러한 함수들이 필요하진 않지만 코드를 더 추가하면 필요할 것이다.

operating system의 C 라이브러리를 링크할 수 없기 때문에, 이러한 함수들을 컴파일러에 제공하기 위한 대안이 필요하다. 가능한 방식 중 하나는 우리만의 `memset` 등의 함수를 구현하고 `#[no_mangle]` attribute를 적용하는 것이다. 하지만 이러한 함수들의 구현에 작은 실수는 예상치 못한 동작으로 이어질 수 있기 때문에 위험하다. ++ 따라서 기존의 검증된 구현을 재사용하는 것이 좋다.

다행히, `compiler_builtins` 크레이트는 모든 필요한 함수에 대한 구현을 포함하고 있고, C 라이브러리의 구현과 충돌하지 않도록 기본적으로 사용할 수 없게 설정되어 있다. cargo의 `build-std-features` 플래그를 `["compiler-builtins-mem"]`로 설정하여 사용할 수 있다. `build-std` 플래그와 유사하게, 이 플래그는 커맨드 라인에서 `-Z` 플래그처럼 전달하거나 `.cargo/config.toml` 내에서 `unstable` 테이블에서 설정할 수 있다. 

```toml
# in .cargo/config.toml

[unstable]
build-std-features = ["compiler-builtins-mem"]
build-std = ["core", "compiler_builtins"]
```

이 플래그는 `compiler_builtins` 크레이트의 `mem` 특성을 사용할 수 있게 한다. 효과는 `memcpy` 등의 함수에 `#[no_mangle]` attribute가 적용되어 링커가 사용 가능하도록 만드는 것이다.

이 변화로 컴파일러가 필요한 함수에 대해 유효한 구현을 가지고 따라서 더 복잡한 코드도 컴파일 가능하다.

#### Set a Default Target

매 `cargo build`마다 `--target` 인자를 전달하는 것을 피하기 위해 기본 target을 override할 수 있다. 이를 위해 다음을 `.cargo/config.toml`에 추가해야 한다.

```toml
# in .cargo/config.toml

[build]
target = "x86_64-blog_os.json"
```

이는 cargo가 `x86_64-blog_os.json` target을 사용하도록 명시한다. 이제 `cargo build`만으로 커널을 빌드할 수 있다.

아직 `_start` entry point는 비어 있다. 이제 뭔가 화면에 출력하도록 해보자.

### Printing to Screen

이 단계에서 텍스트를 화면에 출력하는 가장 쉬운 방법은 VGA text buffer이다. 이것은 화면에 나타나는 내용을 포함하는 VGA 하드웨어에 대응하는 특별한 메모리 공간이다. 일반적으로 80개의 문자 셀을 포함하는 25개의 줄로 구성되어 있다. 각 문자 셀은 foreground 색과 background 색과 함께 ASCII 문자를 표시한다.

VGA buffer의 정확한 레이아웃은 다음 포스트에서 논의한다. "Hello, World!"를 출력하기 위해, buffer는 `0xb8000`에 위치한다는 것과 각 문자 셀은 ASCII byte와 color byte로 구성되어 있다는 것을 알아야 한다.

구현은 다음과 같다:

```rust
static HELLO: &[u8] = b"Hello World!";

#[no_mangle]
pub extern "C" fn _start() -> ! {
    let vga_buffer = 0xb8000 as *mut u8;

    for (i, &byte) in HELLO.iter().enumerate() {
        unsafe {
            *vga_buffer.offset(i as isize * 2) = byte;
            *vga_buffer.offset(i as isize * 2 + 1) = 0xb;
        }
    }

    loop {}
}
```

먼저, 정수 `0xb8000`을 raw pointer로 캐스트한다. 이후 static `HELLO` byte string의 byte들을 iterate한다. for loop에서 `offset` 메서드를 사용하여 string byte와 color byte를 출력한다.

모든 메모리 작성 주위에는 `unsafe` 블록이 있다. 이는 러스트 컴파일러가 우리가 생성한 raw pointer가 유효한지 증명할 수 없기 때문이다. raw pointer는 어느 곳이든 가리킬 수 있고 data corruption으로 이어질 수 있다. `unsafe` 블록 안에 두어서 컴파일러에게 해당 연산이 유효함을 보장할 수 있다. `unsafe` 블록이 러스트의 safety check를 끄는 것은 아니다. 

++

따라서 우리는 가능한 `unsafe`의 사용을 줄이고 싶다. 러스트는 safe abstraction을 생성하여 이를 할 수 있게 한다. 예를 들어 모든 unsafety를 encapsulate하고 외부로부터 모든 잘못된 동작이 불가능하도록 보장할 수 있는 VGA buffer type을 생성할 수 있다. 이 방식은 최소한의 `unsafe`가 필요하고 memory safety를 어기지 않는 것을 보장할 수 있다. 다음 포스트에서 VGA buffer abstraction을 생성할 것이다.

## Running out Kernel

이제 상당한 뭔가를 실행할 수 있는 실행파일을 실행해보자. 먼저 컴파일된 커널을 부트로더와 링크하여 부팅 가능한 디스크 이미지로 바꿔야 한다. 이후에 QEMU 가상 머신에서 디스크 이미지를 실행하거나 USB를 사용하여 실제 하드웨어에서 부트할 수 있다.

### Creating a Bootimage

컴파일된 커널을 부팅 가능한 디스크 이미지로 바꾸기 위해서는 컴파일된 커널을 부트로더와 링크해야 한다. 부트로더는 CPU를 초기화하고 커널을 로드하는 역할을 한다.

우리만의 부트로더를 만드는 대신 `bootloader` 크레이트를 사용한다. 이 크레이트는 C dependencies가 없는 기본적인 BIOS 부트로더를 구현하고 있다. 커널을 부팅하기 위해 이것을 사용하려면 dependency를 추가해야 한다.

```toml
# in Cargo.toml

[dependencies]
bootloader = "0.9.8"
```

bootloader를 dependency로 추가하는 것은 실제로 부팅 가능한 디스크 이미지를 만드는 데에 충분하지 않다. 문제는 커널과 부트로더를 컴파일 이후에 링크해야 하지만 cargo는 post-build script를 지원하지 않는다.

이를 해결하기 위해, 커널과 부트로더를 먼저 컴파일하고 이후에 부팅 가능한 디스크 이미지를 생성하기 위해 이들을 링크하는 `bootimage`라는 도구를 만들었다. 이 도구를 설치하기 위해 다음 명령을 터미널에 실행하면 된다.

```
cargo install bootimage
```

`bootimage`를 실행하고 부트로더를 빌드하기 위해 `llvm-tools-preview` rustup component를 가져야 한다. `rustup component add llvm-tools-preview`를 실행하면 된다.

`bootimage`를 설치하고 `llvm-tools-preview` component를 추가한 이후에 다음을 실행하여 부팅 가능한 디스크 이미지를 생성할 수 있다.

```
cargo bootimage
```

해당 도구가 `cargo build`를 사용하여 커널을 재컴파일한다. 이후에 부트로더를 컴파일한다. 다른 크레이트 dependencies처럼 한 번 빌드된 후에 캐시된다. 따라서 이어지는 빌드는 더 빠르게 진행된다. 최종적으로 `bootimage`는 부트로더와 커널을 부팅 가능한 디스크 이미지로 결합한다.

위 명령어를 실행한 후, `target/x86_64-blog_os/debug` 디렉터리 내에서 `bootimage-blog_os.bin`이라는 부팅 가능한 디스크 이미지를 볼 수 있다. 이 이미지를 가상 머신에서 부팅하거나 실제 하드웨어 상에서 USB를 통해 부팅할 수 있다.

#### How does it work?

`bootimage`는 다음 단계를 거쳐 실행된다.

- 커널을 ELF 파일로 컴파일한다.
- 부트로더 dependency를 standalone executable로 컴파일한다.
- 커널 ELF 파일을 부트로더에 링크한다.

부팅이 시작되면, 부트로더는 ELF 파일을 읽고 분할한다. 그리고 페이지 테이블에서 프로그램 조각들을 가상 주소에 연결하고, `.bss` 부분을 0으로 초기화하고, 스택을 초기화한다. 최종적으로 entry point 주소를 읽은 후에 그곳으로 이동한다.

### Booting it in QEMU

이제 디스크 이미지를 가상 머신에서 부팅할 수 있다. QEMU에서 부팅하기 위해, 다음 명령어를 실행하면 된다.

```
> qemu-system-x86_64 -drive format=raw,file=target/x86_64-blog_os/debug/bootimage-blog_os.bin
warning: TCG doesn't support requested feature: CPUID.01H:ECX.vmx [bit 5]
```

### Real Machine

USB에 저장하고 실제 머신 상에서 부팅할 수 있다.

```
> dd if=target/x86_64-blog_os/debug/bootimage-blog_os.bin of=/dev/sdX && sync
```

`sdX`는 USB의 디바이스 이름이다.

++

### Using cargo run

++