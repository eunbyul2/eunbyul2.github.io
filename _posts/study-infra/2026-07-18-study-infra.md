---
layout: post
title: "키보드에서 A를 누르면 화면에 A가 나타나기까지"
date: 2026-07-18 16:00:00 +0900
categories: [스터디, Linux, OperatingSystem]
tags: [Linux, Kernel, Keyboard, Interrupt, InputSubsystem, TTY, Terminal]
published: true
---

# 키보드에서 A를 누르면 화면에 A가 나타나기까지

키보드에서 `A` 키를 누르면 화면에 `A`가 나타난다. 사용자 입장에서는 너무 당연한 동작이지만, 컴퓨터 내부에서는 키보드 하드웨어, USB 컨트롤러, 커널 디바이스 드라이버, Linux Input Subsystem, 디스플레이 서버, 터미널 에뮬레이터, 애플리케이션, 그래픽 시스템이 차례로 관여한다.

이 과정을 이해할 때 가장 먼저 구분해야 하는 것은 **키보드 입력과 화면 출력은 하나의 동작이 아니라 서로 다른 두 개의 경로**라는 점이다. 키보드는 화면에 직접 문자를 그리지 않는다. 키보드는 어떤 물리적인 키가 눌렸다는 사실만 운영체제에 전달하고, 운영체제와 애플리케이션이 그 입력을 문자로 해석한 뒤 별도의 출력 경로를 통해 화면에 표시한다.

전체 흐름을 단순화하면 다음과 같다.

```text
키 입력
→ 키보드 컨트롤러
→ USB HID 또는 PS/2 드라이버
→ Linux Input Subsystem
→ TTY 또는 Wayland/X11
→ 애플리케이션
→ 터미널 또는 GUI 렌더링
→ GPU
→ 모니터
```

## 1. 키보드는 문자 A를 보내지 않는다

키보드에서 `A` 키를 눌렀다고 해서 키보드가 운영체제에 ASCII 문자 `A`를 직접 전송하는 것은 아니다. 키보드는 일반적으로 특정 물리 키가 눌렸거나 떼어졌다는 정보를 전송한다.

USB 키보드는 대부분 HID(Human Interface Device) 프로토콜을 사용한다. 키보드 내부의 컨트롤러는 키 스위치 상태를 확인한 뒤 현재 눌린 키의 Usage ID와 modifier 상태를 포함한 HID Report를 생성한다. 여기에는 `A`라는 문자 자체가 아니라 `Keyboard a and A`에 해당하는 키 식별 정보가 담긴다.

예를 들어 `A` 키만 누른 경우 키보드는 `A` 키에 해당하는 Usage ID를 전달한다. `Shift + A`를 누른 경우에는 Shift modifier 상태와 `A` 키의 Usage ID가 함께 전달된다. 대문자 `A`인지 소문자 `a`인지는 키보드가 결정하는 것이 아니라 이후 운영체제의 키보드 레이아웃과 입력 처리 계층이 판단한다.

```text
A 키 누름 → 물리 키 위치에 해당하는 코드 전달
Shift + A 키 누름 → Shift 상태 + A 키 코드 전달
```

PS/2 키보드에서는 일반적으로 Scan Code라는 표현을 사용한다. USB HID와 PS/2의 세부 전송 방식은 다르지만, 공통점은 키보드가 최종 문자를 보내는 것이 아니라 어떤 키가 눌렸는지를 나타내는 정보를 보낸다는 것이다.

## 2. 키보드 입력은 컴퓨터에 어떻게 전달되는가

PS/2 키보드에서는 키 입력이 발생하면 키보드 컨트롤러가 인터럽트를 발생시키고, CPU는 해당 인터럽트에 연결된 커널의 인터럽트 처리 루틴을 실행한다.

USB 키보드는 동작 방식이 조금 다르다. USB는 기본적으로 호스트가 장치에 데이터를 요청하는 구조이기 때문에 컴퓨터의 USB Host Controller가 일정한 주기로 키보드의 Interrupt Endpoint를 확인한다. 이름에는 Interrupt가 들어가지만, 키보드가 CPU에 직접 인터럽트를 발생시키는 PS/2 방식과는 구조가 다르다.

키 입력 데이터가 준비되면 USB Host Controller가 이를 메모리에 전달하고, 전송 완료 사실을 CPU에 알린다. 이후 Linux 커널의 USB Core와 HID 드라이버가 전달된 HID Report를 처리한다.

따라서 단순히 “키를 누르면 키보드가 CPU에 인터럽트를 발생시킨다”고 설명하면 PS/2 환경에는 비교적 적절하지만, 현대적인 USB 키보드의 동작을 정확히 설명하기에는 부족하다. 일반적인 USB 키보드에서는 USB Host Controller가 키보드를 주기적으로 확인하고, 데이터 전송이 완료되면 컨트롤러가 CPU에 인터럽트를 알리는 구조로 이해하는 것이 더 정확하다.

## 3. 커널의 키보드 드라이버가 입력을 처리한다

키보드에서 전달된 데이터는 Linux 커널의 디바이스 드라이버가 처리한다. USB 키보드라면 USB Core, HID Core, `usbhid` 계층이 관여하고, PS/2 키보드라면 `i8042`, `serio`, `atkbd` 같은 드라이버 계층이 관여할 수 있다.

드라이버는 하드웨어별 데이터 형식을 Linux가 공통으로 이해할 수 있는 입력 이벤트로 변환한다. 예를 들어 키보드에서 `A` 키가 눌렸다는 데이터가 들어오면 Linux Input Subsystem에는 다음과 유사한 이벤트가 전달된다.

```text
EV_KEY
KEY_A
value 1
```

여기서 `EV_KEY`는 키 입력 이벤트라는 의미이고, `KEY_A`는 A 위치의 키를 의미한다. `value 1`은 키가 눌렸다는 뜻이다. 일반적으로 값 `0`은 키를 뗀 상태, 값 `1`은 키를 누른 상태, 값 `2`는 키 반복 상태를 의미한다.

이 단계에서도 아직 최종 문자 `A`가 생성된 것은 아니다. 커널은 단지 `KEY_A`라는 키가 눌렸다는 사실만 알고 있다.

## 4. Linux Input Subsystem이 입력 장치를 통합한다

Linux는 키보드, 마우스, 터치패드, 게임패드와 같은 다양한 입력 장치를 Linux Input Subsystem이라는 공통 계층으로 관리한다. 각 장치의 드라이버는 하드웨어별 차이를 처리하고, 상위 계층에는 표준화된 입력 이벤트를 제공한다.

사용자 공간에서는 보통 `/dev/input/eventX` 장치 파일을 통해 이러한 이벤트를 읽을 수 있다.

```bash
ls -l /dev/input/
```

시스템에 따라 다음과 같은 장치가 보일 수 있다.

```text
event0
event1
event2
mouse0
mice
```

어떤 `eventX`가 키보드에 해당하는지는 시스템마다 다르다. `evtest` 명령을 사용하면 실제 입력 이벤트를 확인할 수 있다.

```bash
sudo evtest
```

키보드 장치를 선택한 뒤 `A` 키를 누르면 다음과 유사한 결과가 나타난다.

```text
Event: type 1 (EV_KEY), code 30 (KEY_A), value 1
Event: type 0 (EV_SYN), code 0 (SYN_REPORT), value 0
Event: type 1 (EV_KEY), code 30 (KEY_A), value 0
Event: type 0 (EV_SYN), code 0 (SYN_REPORT), value 0
```

첫 번째 `KEY_A value 1`은 키를 누른 이벤트이고, 이후의 `KEY_A value 0`은 키를 뗀 이벤트다. `SYN_REPORT`는 여러 입력 이벤트의 한 묶음이 끝났음을 알리는 동기화 이벤트다.

이 실습을 통해 키보드가 문자 `A`를 보내는 것이 아니라 `KEY_A`라는 키 이벤트를 전달한다는 사실을 직접 확인할 수 있다.

## 5. KEY_A는 언제 문자 A가 되는가

`KEY_A`가 실제 문자 `a` 또는 `A`로 변환되는 과정은 사용 환경에 따라 달라진다. 콘솔 TTY 환경인지, GUI 환경인지, 터미널 에뮬레이터 안인지에 따라 입력 경로가 다르다.

GUI 환경에서는 Wayland compositor나 X Server가 입력 장치 이벤트를 받아 키보드 레이아웃과 modifier 상태를 적용한다. 예를 들어 현재 키보드 레이아웃이 영문이고 Shift가 눌리지 않았다면 `KEY_A`는 소문자 `a`로 해석될 수 있다. Shift가 눌려 있다면 대문자 `A`로 해석된다.

```text
KEY_A + Shift 미입력 → a
KEY_A + Shift 입력 → A
```

Caps Lock 상태, 한글 입력기 상태, 키보드 레이아웃, 애플리케이션 단축키 설정에 따라서도 결과가 달라질 수 있다. 같은 물리 키를 눌러도 영문 환경에서는 `a`, 한글 두벌식 환경에서는 `ㅁ`, 단축키 환경에서는 특정 명령으로 해석될 수 있다.

따라서 `KEY_A`와 문자 `A`는 같은 것이 아니다. `KEY_A`는 물리적인 키 입력을 표현하는 Linux 입력 이벤트이고, 문자 `A`는 키보드 레이아웃과 입력 상태를 적용한 결과다.

## 6. GUI 애플리케이션에서 A가 입력되는 과정

웹 브라우저나 VS Code 같은 GUI 애플리케이션에서는 대략 다음과 같은 흐름으로 입력이 전달된다.

```text
키보드
→ USB HID 드라이버
→ Linux Input Subsystem
→ libinput
→ Wayland compositor 또는 X Server
→ GUI toolkit
→ 애플리케이션
```

`libinput`은 Wayland compositor나 X.Org 환경에서 키보드와 마우스 같은 입력 장치를 통합적으로 처리하기 위해 널리 사용되는 라이브러리다. Wayland 환경에서는 compositor가 입력 이벤트를 받아 포커스를 가진 클라이언트 애플리케이션에 전달한다. X11 환경에서는 X Server가 입력 이벤트를 처리해 해당 X 클라이언트에 전달한다.

애플리케이션은 키 이벤트를 수신한 뒤 현재 커서 위치와 입력 상태를 기준으로 문자를 문서나 입력창에 삽입한다. 이때 애플리케이션은 글꼴에서 해당 문자의 glyph를 찾고, GUI toolkit과 그래픽 라이브러리를 통해 화면에 렌더링한다.

## 7. 터미널에서 A가 입력되는 과정

터미널 환경에서는 물리 키보드, 터미널 에뮬레이터, pseudo terminal, shell을 구분해야 한다.

GNOME Terminal, Konsole, iTerm2, Windows Terminal과 같은 프로그램은 터미널 에뮬레이터다. 터미널 에뮬레이터는 GUI 애플리케이션으로 실행되며, 키보드 입력을 받아 문자 바이트로 변환한 뒤 pseudo terminal의 master 측에 기록한다.

Pseudo Terminal은 master와 slave의 한 쌍으로 구성된다. 터미널 에뮬레이터는 master 측을 사용하고, Bash 같은 shell은 slave 측을 자신의 표준 입력과 표준 출력으로 사용한다.

```text
키보드 입력
→ Wayland/X11
→ 터미널 에뮬레이터
→ PTY master
→ TTY line discipline
→ PTY slave
→ Bash의 stdin
```

예를 들어 터미널 에뮬레이터가 `A`라는 문자 바이트를 PTY master에 기록하면, 커널의 TTY 계층과 line discipline이 이를 처리한다. Bash는 PTY slave에 연결된 표준 입력을 `read()` 시스템 콜로 읽는다.

하지만 터미널에서 키를 누르자마자 화면에 문자가 나타나는 현상은 Bash가 직접 `write()`를 호출해서 발생하는 것이 아닐 수 있다. 일반적인 canonical mode에서는 TTY의 `ECHO` 설정이 활성화되어 있기 때문에 커널의 TTY line discipline이 입력 문자를 자동으로 다시 출력 방향으로 보내는 echo 동작을 수행한다.

즉, 터미널에서 `A`를 눌렀을 때 화면에 즉시 `A`가 보이는 일반적인 흐름은 다음과 같다.

```text
터미널 에뮬레이터가 A 입력
→ PTY master에 기록
→ TTY line discipline이 입력 처리
→ ECHO 설정에 따라 A를 출력 큐로 전달
→ 터미널 에뮬레이터가 A를 읽음
→ 화면에 A 렌더링
```

Bash는 사용자가 Enter를 누르기 전까지 완성된 입력 행을 전달받지 못할 수도 있다. 이는 TTY가 canonical mode로 동작할 때 입력을 한 줄 단위로 버퍼링하기 때문이다.

따라서 터미널에서 `A`가 보이는 이유를 단순히 “Bash가 `read()`로 A를 읽고 `write()`로 다시 출력하기 때문”이라고 설명하면 정확하지 않다. 일반적인 설정에서는 TTY line discipline의 echo 기능이 화면 표시를 담당한다.

## 8. Canonical mode와 Raw mode

TTY 입력 방식은 크게 canonical mode와 non-canonical 또는 raw mode로 구분할 수 있다.

Canonical mode에서는 커널의 TTY line discipline이 입력을 한 줄 단위로 처리한다. 사용자가 `A`, `B`, `C`를 누른 뒤 Enter를 입력하면 애플리케이션은 보통 `ABC\n`을 한 번에 읽는다. Backspace 처리, 줄 단위 버퍼링, 입력 echo도 TTY 계층에서 수행할 수 있다.

Raw mode에서는 입력 문자가 가공되지 않고 애플리케이션에 더 직접적으로 전달된다. `vim`, `less`, `top`, `htop`과 같은 대화형 프로그램은 터미널을 raw mode에 가깝게 설정해 키 입력을 즉시 처리한다.

다음 명령으로 현재 터미널 설정을 확인할 수 있다.

```bash
stty -a
```

출력에서 `icanon`은 canonical mode 활성화 여부를 나타내고, `echo`는 입력 문자를 자동으로 화면에 다시 표시할지 나타낸다.

```bash
stty -echo
```

위 명령을 실행한 뒤 키보드로 문자를 입력하면 실제 입력은 전달되지만 화면에 문자가 보이지 않는다. 다시 echo를 활성화하려면 다음 명령을 사용한다.

```bash
stty echo
```

이 실습을 통해 터미널에서 입력한 문자가 화면에 나타나는 동작이 키보드 자체의 기능이 아니라 TTY 설정에 의해 제어된다는 점을 확인할 수 있다.

## 9. 애플리케이션이 입력을 읽는 과정

Bash 같은 애플리케이션은 키보드 하드웨어를 직접 읽지 않는다. 애플리케이션은 자신의 표준 입력에 연결된 파일 디스크립터를 통해 데이터를 읽는다.

일반적으로 표준 입력은 파일 디스크립터 `0`이다.

```c
char buf[2];
read(0, buf, 1);
```

이 코드에서 `read()`는 키보드 드라이버를 직접 호출하는 것이 아니다. 현재 프로세스의 파일 디스크립터 `0`이 PTY slave나 실제 TTY 장치에 연결되어 있다면, `read()`는 그 장치에서 데이터를 읽는다.

시스템 콜의 흐름을 단순화하면 다음과 같다.

```text
애플리케이션
→ read() 시스템 콜
→ 커널 VFS 계층
→ TTY 드라이버
→ line discipline 버퍼
→ 사용자 공간으로 데이터 반환
```

Canonical mode에서는 Enter가 입력될 때까지 `read()`가 대기할 수 있고, raw mode에서는 키 입력 직후 한 글자씩 반환될 수 있다.

## 10. 문자는 어떻게 화면에 그려지는가

애플리케이션이나 TTY 계층이 출력 데이터를 생성하면 터미널 에뮬레이터나 GUI 애플리케이션은 해당 문자를 화면에 그려야 한다.

터미널 에뮬레이터는 PTY master에서 출력 바이트를 읽고 ANSI escape sequence를 해석한다. 일반 문자 `A`라면 현재 커서 위치에 `A`를 표시하고, `\n`, `\r`, 색상 코드, 커서 이동 명령 등이 포함되어 있다면 터미널 상태를 그에 맞게 변경한다.

이후 터미널 에뮬레이터는 글꼴 시스템을 이용해 문자 `A`에 해당하는 glyph를 찾는다. Glyph는 글자의 시각적 모양이다. 같은 문자 `A`라도 사용하는 글꼴에 따라 화면에 그려지는 모양은 달라진다.

GUI 애플리케이션은 FreeType, HarfBuzz, Pango와 같은 글꼴 및 텍스트 처리 라이브러리를 사용할 수 있다. 렌더링된 결과는 Wayland compositor 또는 X Server를 거쳐 최종 화면 이미지에 합성된다.

Linux 그래픽 스택에서는 DRM(Direct Rendering Manager)과 KMS(Kernel Mode Setting)가 GPU와 디스플레이 출력을 관리한다. GPU는 최종 프레임을 framebuffer 또는 scanout buffer에 준비하고, 디스플레이 컨트롤러는 이 데이터를 모니터로 전송한다.

```text
문자 A
→ 글꼴 glyph 선택
→ 그래픽 버퍼에 렌더링
→ compositor가 화면 합성
→ DRM/KMS
→ GPU 또는 디스플레이 컨트롤러
→ 모니터
```

이 단계에 도달해야 사용자는 실제 픽셀로 그려진 `A`를 볼 수 있다.

## 11. 입력 경로와 출력 경로는 분리되어 있다

키보드에서 `A`를 누르고 화면에서 `A`를 보는 과정은 하나의 직선적인 동작처럼 보이지만, 실제로는 입력 경로와 출력 경로가 분리되어 있다.

입력 경로는 다음과 같다.

```text
키보드
→ 하드웨어 인터페이스
→ 커널 디바이스 드라이버
→ Linux Input Subsystem
→ Wayland/X11 또는 TTY
→ 애플리케이션
```

출력 경로는 다음과 같다.

```text
애플리케이션 또는 TTY echo
→ 터미널 에뮬레이터 또는 GUI toolkit
→ 글꼴 렌더링
→ compositor
→ DRM/KMS
→ GPU
→ 모니터
```

두 경로가 연결되어 동작하기 때문에 사용자는 키를 누른 즉시 같은 문자가 화면에 나타난다고 느낀다. 그러나 입력한 키가 항상 그대로 출력되는 것은 아니다.

예를 들어 `Ctrl + C`는 문자 `c`로 표시되지 않고 실행 중인 프로세스에 `SIGINT` 시그널을 발생시킬 수 있다. `Ctrl + D`는 EOF 조건으로 처리될 수 있고, 방향키는 일반 문자 대신 escape sequence로 전달된다. 비밀번호 입력 시에는 `ECHO`가 비활성화되어 키 입력이 화면에 나타나지 않는다.

이는 키보드 입력과 화면 출력 사이에 운영체제와 애플리케이션의 해석 과정이 존재한다는 것을 보여준다.

## 12. 전체 과정 정리

GUI 애플리케이션에서 `A` 키를 누른 경우의 전체 흐름은 다음과 같이 정리할 수 있다.

```text
1. 사용자가 키보드의 A 키를 누른다.
2. 키보드 컨트롤러가 물리 키 상태를 감지한다.
3. USB HID Report 또는 PS/2 Scan Code가 생성된다.
4. 커널의 USB HID 또는 키보드 드라이버가 데이터를 처리한다.
5. Linux Input Subsystem에 KEY_A 이벤트가 등록된다.
6. libinput과 Wayland compositor 또는 X Server가 이벤트를 전달받는다.
7. 키보드 레이아웃과 Shift, Caps Lock, 입력기 상태가 적용된다.
8. KEY_A가 문자 a 또는 A로 변환된다.
9. 포커스를 가진 애플리케이션이 문자 입력 이벤트를 받는다.
10. 애플리케이션이 글꼴의 glyph를 이용해 문자를 렌더링한다.
11. compositor와 DRM/KMS가 최종 화면을 구성한다.
12. 모니터에 A가 픽셀로 표시된다.
```

터미널 환경에서는 다음과 같이 정리할 수 있다.

```text
1. 사용자가 A 키를 누른다.
2. 키 입력이 Linux Input Subsystem을 거쳐 터미널 에뮬레이터에 전달된다.
3. 터미널 에뮬레이터가 키 입력을 문자 A로 변환한다.
4. 터미널 에뮬레이터가 A를 PTY master에 기록한다.
5. 커널 TTY line discipline이 입력을 처리한다.
6. ECHO가 활성화되어 있다면 A가 출력 큐로 다시 전달된다.
7. 터미널 에뮬레이터가 PTY master에서 A를 읽는다.
8. 터미널 에뮬레이터가 글꼴을 사용해 A를 렌더링한다.
9. GPU와 디스플레이 시스템을 거쳐 모니터에 A가 표시된다.
10. Canonical mode라면 Bash는 Enter가 입력된 이후 완성된 입력 행을 읽는다.
```

## 마무리

키보드에서 `A`를 누르면 화면에 `A`가 나타나는 과정은 단순한 입력과 출력처럼 보이지만, 실제로는 하드웨어, 커널, 사용자 공간, 그래픽 시스템이 모두 연결된 결과다.

키보드는 문자 자체를 보내는 것이 아니라 물리 키에 대한 정보를 전달한다. 커널의 디바이스 드라이버와 Linux Input Subsystem은 이를 표준화된 키 이벤트로 변환하고, Wayland, X11, TTY 같은 상위 계층이 키보드 레이아웃과 modifier 상태를 적용해 실제 문자로 해석한다. 이후 애플리케이션이나 TTY echo 기능이 출력 데이터를 만들고, 터미널 에뮬레이터 또는 GUI 애플리케이션이 글꼴 glyph를 렌더링한다. 마지막으로 compositor, DRM/KMS, GPU를 거쳐 모니터에 문자가 표시된다.

결국 화면에 보이는 `A` 한 글자는 키보드가 직접 출력한 결과가 아니다. 여러 하드웨어와 소프트웨어 계층이 입력 이벤트를 전달하고 해석한 뒤, 별도의 출력 경로를 통해 다시 화면에 그려낸 결과다.
