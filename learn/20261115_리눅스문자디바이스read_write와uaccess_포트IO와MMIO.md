# 리눅스 문자 디바이스의 `read`·`write`와 `uaccess`, 포트 I/O와 MMIO

문자 장치의 `read()`와 `write()`가 VFS에서 드라이버 콜백으로 이어지는 과정과, 콜백이 사용자 버퍼와 장치 레지스터 사이에서 데이터를 옮기는 방법을 공부했다. 전송 방향에 맞는 `uaccess` 함수, 실제로 처리한 바이트 수를 반환하는 규칙, 포트 I/O와 MMIO에서 접근 폭과 주소를 고르는 기준을 정리했다. 마지막으로 주 번호와 부 번호가 열린 장치를 어떻게 식별하는지 살펴봤다.

학습일 **2026-11-15** · PDF 뷰어 기준 **217~253쪽**

## 1. 장치 파일을 열면 VFS가 어떤 콜백을 호출하는가

응용 프로그램은 장치 노드를 `open()`으로 열어 파일 디스크립터 `fd`를 얻고, `read(fd, buf, count)` 또는 `write(fd, buf, count)`를 호출한 뒤 `close()`로 닫는다. 장치 노드를 열 때 커널은 문자 장치의 주·부 번호에 연결된 파일 연산을 찾는다. 이어지는 입출력에서는 VFS가 `fd`로 열린 `struct file`을 찾고 접근 모드와 요청 범위를 검사한다. 읽기 전용으로 연 파일에 쓰기를 요청하면 VFS는 드라이버 콜백을 호출하기 전에 `-EBADF`를 반환한다. 일반적인 동기 입출력 경로에서는 등록된 `.read`나 `.write`를 호출하며, 그 콜백이 없으면 `.read_iter`나 `.write_iter`가 있는 경우 해당 연산을 사용한다. 장치가 제공하는 별도 명령은 `ioctl()`, 매핑 가능한 메모리나 버퍼의 공유는 `mmap()` 인터페이스로 표현할 수 있다. [VFS 파일 연산](https://docs.kernel.org/filesystems/vfs.html), [문자 장치 열기](https://github.com/torvalds/linux/blob/master/fs/char_dev.c), [VFS 읽기·쓰기 구현](https://github.com/torvalds/linux/blob/master/fs/read_write.c), [`ioctl` 인터페이스](https://docs.kernel.org/driver-api/ioctl.html)

`<linux/fs.h>`의 `struct file_operations`에서 두 콜백은 `ssize_t`를 반환하고 `struct file *`, 사용자 버퍼, `size_t count`, `loff_t *ppos`를 차례로 받는다. `.read`의 버퍼는 결과를 저장할 `char __user *`이고, `.write`의 버퍼는 입력을 읽을 `const char __user *`다. `count`는 요청 길이를 바이트로 나타내며, 콜백은 사용자 버퍼와 장치가 실제로 다룰 수 있는 범위 안에서 처리한다. 장치에 필요한 설정 명령이라면 `.write`가 받은 바이트가 드라이버 상태를 변경할 수도 있다. [콜백 선언](https://github.com/torvalds/linux/blob/master/include/linux/fs.h), [VFS 파일 연산](https://docs.kernel.org/filesystems/vfs.html)

| 열린 파일에서 확인할 값 | 읽기·쓰기에서의 역할 |
| --- | --- |
| `f_flags` | `O_NONBLOCK` 등 현재 파일 상태 플래그를 확인한다. 접근 모드를 직접 구분해야 하면 `f_flags & O_ACCMODE`의 결과를 `O_RDONLY`, `O_WRONLY`, `O_RDWR`과 비교한다. `O_RDONLY`의 값은 0이다. `O_SYNC`를 지원하는 장치는 쓰기의 완료 조건을 인터페이스에 맞게 정의하고 적용한다. |
| `f_pos`와 콜백의 `ppos` | 위치를 지원하는 장치는 `*ppos`를 기준으로 처리하고, 위치가 바뀌면 그 값을 갱신한다. 예를 들어 위치 24에서 3바이트를 읽으면 다음 위치는 27이다. `.llseek`를 구현한 장치에서는 사용자가 읽기·쓰기 위치를 이동할 수 있다. 위치는 일반 파일의 바이트 오프셋처럼 사용할 수도 있고, `/dev/mem`처럼 물리 주소를 나타낼 수도 있다. 스트림으로 연 파일에서는 `ppos`가 `NULL`일 수 있다. |
| `private_data` | 열린 파일에 연결한 장치나 열린 파일별 상태를 `.open`, `.read`, `.write`, `.release` 사이에서 참조한다. 예를 들어 각 열기에 다른 채널 설정을 둘 수 있다. 여러 열기가 공유하는 큐나 레지스터는 장치별 상태에 둔다. |
| `f_op` | 해당 열린 파일에 적용할 연산 테이블이다. 부 번호에 따라 구현 전체가 달라지는 경우 `misc`처럼 열기 시 대상을 고르고 참조를 관리하는 `replace_fops()` 경로를 사용할 수 있다. 동일한 연산을 공유하는 채널이라면 상태만 골라 `private_data`에 둘 수 있다. |

[열기 플래그 매뉴얼](https://man7.org/linux/man-pages/man2/open.2.html), [파일 위치 처리](https://github.com/torvalds/linux/blob/master/fs/read_write.c), [스트림 열기 구현](https://github.com/torvalds/linux/blob/master/fs/open.c), [`misc_open()` 구현](https://github.com/torvalds/linux/blob/master/drivers/char/misc.c), [`/dev/mem` 매뉴얼](https://man7.org/linux/man-pages/man4/mem.4.html)

## 2. 준비 여부와 반환값을 함께 설계한다

- **읽기**는 요청 시점의 장치 상태를 레지스터에서 얻거나, 드라이버가 앞서 축적한 데이터를 꺼낸다. 현재 상태만 보면 되는 장치는 콜백에서 직접 읽을 수 있다. 짧게 발생하는 신호나 연속 수신 데이터는 인터럽트 처리기가 커널 큐에 담고 읽는 쪽을 깨우도록 설계할 수 있다.
- **쓰기**는 사용자 데이터를 커널로 가져와 검증한 다음 레지스터에 쓰거나 송신 큐에 넣는다. 큐에 넣는 것이 완료인 인터페이스와 장치의 처리 완료까지 기다리는 인터페이스는 성공을 반환하는 시점이 다르다. 여러 호출자나 인터럽트 처리기가 같은 큐 또는 레지스터를 만지면 준비 조건 확인과 상태 갱신을 동기화한다. 잠들 수 있는 사용자 복사는 인터럽트 처리기와 스핀락으로 보호하는 구간 밖에서 수행한다.
- **대기**는 요청을 진행할 조건으로 판단한다. 블로킹 호출은 데이터나 공간이 준비될 때까지 잠들 수 있으며, 준비된 분량을 처리하고 `count`보다 작은 양수로 끝날 수도 있다. `O_NONBLOCK`으로 연 스트림에서 당장 한 바이트도 진행할 수 없으면 `-EAGAIN`을 반환하고, 일부가 가능하면 실제 전송량을 반환할 수 있다. 새 코드에서는 비블로킹 플래그로 `O_NONBLOCK`을 쓰며, 리눅스에서 `O_NDELAY`는 같은 용도로 쓰이는 이름이다. 이벤트를 기다리는 앱은 `.poll`을 제공하는 장치에서 `poll()`로 준비 신호를 받은 뒤 입출력을 재시도할 수 있다.
- **결과**에서 양수는 실제 전달하거나 받아들인 바이트 수다. 예를 들어 쓰기 요청 12바이트 중 5바이트를 받아들였다면 앱은 남은 7바이트를 이어서 보낼 수 있다. 입력이 끝났음을 표현하는 장치에서 읽기 결과 `0`은 EOF이고, 길이 0인 읽기 요청도 오류가 없으면 `0`을 반환한다. 요청을 진행하지 못했다면 장치 I/O 오류에 `-EIO`, 사용자 버퍼 접근 실패에 `-EFAULT`, 실제 저장 공간 부족에 `-ENOSPC`처럼 이유에 맞는 오류를 사용한다. 커널 콜백의 음수 오류는 통상 libc의 `read()`와 `write()`에서 `-1`과 `errno`로 나타난다.

[`read(2)`](https://man7.org/linux/man-pages/man2/read.2.html), [`write(2)`](https://man7.org/linux/man-pages/man2/write.2.html), [`open(2)`](https://man7.org/linux/man-pages/man2/open.2.html), [VFS의 `.poll`](https://docs.kernel.org/filesystems/vfs.html), [사용자 메모리 접근](https://docs.kernel.org/core-api/mm-api.html#user-space-memory-access)

## 3. 사용자 주소와 커널 주소 사이에서 복사한다

시스템 호출로 커널에 진입해도 보통 같은 태스크의 실행이 이어진다. CPU가 커널 권한으로 전환되어 VFS와 드라이버 코드를 실행한 뒤 사용자 코드로 돌아간다. 진입 명령은 아키텍처에 따라 달라지며, 예를 들어 x86-64의 일반 경로는 `SYSCALL`을 사용한다. 프로세스의 사용자 가상 주소 공간은 별도로 관리되고 사용자 모드의 커널 메모리 접근은 제한된다. 드라이버가 받은 사용자 주소에는 매핑이나 권한 문제가 있을 수 있고, 아직 메모리에 올라오지 않았거나 스왑에 있는 페이지라면 접근 중 폴트가 발생할 수 있다. `<linux/uaccess.h>`의 헬퍼로 사용자 주소를 접근하고 실패를 처리한다. [x86-64 시스템 호출 진입 코드](https://github.com/torvalds/linux/blob/master/arch/x86/entry/entry_64.S), [리눅스 메모리 관리 개념](https://docs.kernel.org/admin-guide/mm/concepts.html), [`uaccess.h` 구현](https://github.com/torvalds/linux/blob/master/include/linux/uaccess.h)

| 데이터 크기 | 커널에서 사용자로 | 사용자에서 커널로 | 반환값 |
| --- | --- | --- | --- |
| 배열·버퍼 | `copy_to_user(user_dst, kernel_src, n)` | `copy_from_user(kernel_dst, user_src, n)` | 복사하지 못한 바이트 수다. `0`이면 `n`바이트 전부를 복사했다. |
| 스칼라 하나 | `put_user(value, user_ptr)` | `get_user(value, user_ptr)` | 성공하면 `0`, 실패하면 `-EFAULT`다. `get_user()`는 실패 시 목적 변수에 0을 넣는다. |

`get_user()`와 `put_user()`의 접근 폭은 사용자 포인터가 가리키는 스칼라의 타입으로 결정된다. 예를 들어 16비트 값과 두 개의 8비트 필드로 이루어진 4바이트 명령이라면 전체 길이를 확인한 뒤 규약에 맞춰 필드 크기와 바이트 순서대로 해석한다. 바이트 배열이나 구조체 전체를 처리할 때는 블록 복사가 적합하다. 일반 `copy_*_user()`, `get_user()`, `put_user()`는 각자의 경로에서 사용자 주소를 검사한다. 별도의 범위 검사만 필요한 경로에서는 `access_ok(addr, len)`을 사용하고, 실제 메모리 접근의 성공 여부는 해당 접근 함수의 반환값으로 확인한다. [스칼라 접근 문서](https://docs.kernel.org/core-api/mm-api.html#user-space-memory-access), [블록 복사와 범위 검사 구현](https://github.com/torvalds/linux/blob/master/include/linux/uaccess.h)

예를 들어 읽기에서 18바이트를 복사하려고 했는데 `copy_to_user()`가 `4`를 반환했다면, 사용자에게 전달된 분량은 14바이트다. 바이트 스트림이 부분 전송을 허용한다면 전송한 14바이트를 읽기 결과로 돌려주고, 복사 실패 전까지 아무 바이트도 전달하지 못했다면 `-EFAULT`를 반환한다. 읽기 큐의 소비량도 성공한 분량과 일치시켜야 한다. 명령 전체가 도착해야 실행할 수 있는 장치에서는 명령 버퍼가 완전히 복사된 경우에만 장치에 적용한다. `copy_from_user()`는 짧게 복사됐을 때 목적 버퍼의 남은 부분을 0으로 채우므로, 입력의 유효 길이는 반환값으로 계산한다. 임시 버퍼가 필요하면 요청 상한을 정하고 할당 실패를 검사하며, 읽기 결과에 포함할 바이트를 초기화한 뒤 성공·실패 경로 모두에서 버퍼를 정리한다. [사용자 복사 구현](https://github.com/torvalds/linux/blob/master/include/linux/uaccess.h), [메모리 할당 가이드](https://docs.kernel.org/core-api/memory-allocation.html)

버퍼를 경유하는 경로는 장치와 커널 버퍼 사이의 전송에 더해 사용자 버퍼로 옮기는 비용이 든다. 드라이버가 매핑을 지원하는 경우 `mmap()`으로 지정된 버퍼를 사용자 주소 공간에 연결해 반복 복사를 줄일 수 있다. `/dev/mem`을 통한 물리 주소 접근과 매핑은 커널 구성과 아키텍처의 제한을 받는다. [VFS의 매핑 연산](https://docs.kernel.org/filesystems/vfs.html), [`/dev/mem` 접근 제한](https://man7.org/linux/man-pages/man4/mem.4.html)

## 4. 레지스터가 속한 주소 공간과 접근 폭을 고른다

x86의 포트 I/O는 CPU의 `in`·`out` 명령이 사용하는 별도 주소 공간을 갖는다. MMIO는 장치 자원이 메모리 주소 범위에 배치되는 방식이다. 한 시스템에서 두 방식이 함께 쓰일 수 있으므로, 장치가 제공한 리소스 종류와 레지스터 명세에 따라 접근 방식을 결정한다. 드라이버에서는 `<linux/io.h>`를 포함하고 아키텍처별 구현을 감싼 접근 함수를 사용한다. [리눅스 장치 I/O 문서](https://docs.kernel.org/driver-api/device-io.html), [커널 I/O 헤더](https://github.com/torvalds/linux/blob/master/include/linux/io.h)

| 전송 형태 | 포트 I/O | MMIO |
| --- | --- | --- |
| 레지스터 한 번 읽기 | `inb()`, `inw()`, `inl()` | `readb()`, `readw()`, `readl()` |
| 레지스터 한 번 쓰기 | `outb()`, `outw()`, `outl()` | `writeb()`, `writew()`, `writel()` |
| 같은 레지스터에서 반복 읽기 | `insb()`, `insw()`, `insl()` | `readsb()`, `readsw()`, `readsl()` |
| 같은 레지스터에 반복 쓰기 | `outsb()`, `outsw()`, `outsl()` | `writesb()`, `writesw()`, `writesl()` |
| 연속된 주소 범위에 대한 블록 연산 | | `memcpy_fromio()`, `memcpy_toio()`, `memset_io()` |

`b`, `w`, `l`은 각각 8·16·32비트 폭을 뜻한다. 한 번 읽는 함수는 레지스터 값을 반환하고, 한 번 쓰는 함수는 값과 포트 번호 또는 매핑된 주소를 받는다. 반복 전송은 장치 주소를 고정하고 커널 메모리 버퍼를 전진시킨다. 예를 들어 16비트 FIFO에 `count = 6`으로 반복 접근하면 같은 레지스터에서 6개 원소, 총 12바이트를 전송한다. 연속 주소에 놓인 버퍼를 옮길 때는 MMIO 블록 함수를 사용하며, 그 길이는 바이트 단위다. [입출력 함수의 폭과 동작](https://docs.kernel.org/driver-api/device-io.html)

포트 I/O는 드라이버가 소유한 유효 포트 번호를 사용한다. 이식 가능한 MMIO 코드에서는 장치 리소스를 `ioremap()` 또는 관리형 리소스 매핑 함수로 매핑하고, `void __iomem *`로 받은 주소에 레지스터 오프셋을 더해 접근 함수에 전달한다. 접근 함수가 순서와 바이트 순서에 필요한 아키텍처별 처리를 맡는다. PCI처럼 두 주소 형태가 모두 가능한 환경에서 공통 코드를 만들 때는 매핑 방식에 맞는 `ioread*()`·`iowrite*()`와 반복 전송용 `*_rep()` 계열을 사용할 수 있다. [매핑과 `__iomem` 주소](https://docs.kernel.org/driver-api/device-io.html)

## 5. 상태 입력을 확인한 뒤 출력 레지스터를 갱신한다

시험 지그의 리미트 스위치가 눌리면 검사 장비에 약 250ms 길이의 시작 펄스를 보내는 문자 장치를 생각해 봤다. 예제의 상태 바이트에서 비트 2가 액티브 로우 입력이고, 출력 래치의 비트 5가 시작 신호다. 상태 입력과 출력 데이터는 각기 다른 레지스터로 처리한다.

1. 앱은 `O_RDWR | O_NONBLOCK`으로 장치를 열고 상태 바이트 하나를 요청한다. 드라이버는 상태 레지스터를 읽어 `put_user()`로 넘긴 뒤 성공 시 `1`을 반환한다. 앱은 반환값이 `1`일 때만 상태 바이트를 16진수로 기록하고 비트 2를 검사한다. 입력이 활성화될 때까지 정해진 간격으로 상태를 다시 확인한다. 큐에 쌓인 이벤트를 읽는 방식이라면 데이터가 없는 시점에 `-EAGAIN`을 처리하거나 `.poll`로 준비를 기다릴 수 있다.
2. 입력을 확인하면 앱은 시작 신호를 올리는 명령을 쓰고 250ms 후 신호를 내리는 명령을 쓴다. 드라이버는 사용자 주소에서 명령값을 성공적으로 가져온 뒤 장치 상태를 잠그고 해당 비트만 바꾼다. 여러 바이트를 지원하는 인터페이스라면 다음 입력 바이트로 주소를 전진시키고, 성공적으로 적용한 분량만 반환한다. 앱은 쓰기 결과를 확인한 뒤 장치를 닫는다.
3. 실제 병렬 프린터 포트에서는 회로와 상태 레지스터 정의에 따라 접지에 연결된 입력선을 활성 상태로 해석하고, 데이터 레지스터의 비트로 출력선을 구동할 수 있다. 포트가 `parport`에 등록된 환경에서는 클라이언트를 `parport_register_dev_model()`로 등록한다. `parport_claim()`이 성공하면 `parport_read_status()`와 `parport_write_data()`를 호출할 수 있고, 사용을 마치면 `parport_release()`로 반납한다. 종료 시에는 `parport_unregister_device()`로 클라이언트 등록을 해제한다. 드라이버가 포트 리소스를 직접 소유하는 구성에서는 할당된 포트 주소에 `inb()`와 `outb()`를 사용할 수 있다. [병렬 포트 API](https://docs.kernel.org/driver-api/parport-lowlevel.html), [현재 병렬 포트 헤더](https://github.com/torvalds/linux/blob/master/include/linux/parport.h)

현행 커널에서는 외부 모듈을 실행 대상 커널의 빌드 트리에 맞춰 Kbuild로 빌드해 `.ko`를 만든다. `module_init()`과 `module_exit()`로 등록·해제 진입점을 지정한다. `file_operations.owner`에 `THIS_MODULE`을 설정하면 VFS의 파일 연산 참조가 모듈 참조를 유지한다. `.open`과 `.release`는 열린 파일을 위한 초기화나 정리가 필요할 때 구현한다. 사용자 프로그램은 일반 C 컴파일러로 별도로 빌드한다. 설치하지 않은 모듈은 의존 모듈을 준비한 뒤 `insmod`로 적재해 `lsmod`로 확인하고, 장치 사용을 마친 뒤 `rmmod`로 내릴 수 있다. 노드를 수동으로 만든다면 `mknod`는 종류와 장치 번호를 기록하고, `open()` 시점에 일치하는 드라이버 등록이 필요하다. [외부 모듈 빌드](https://docs.kernel.org/kbuild/modules.html), [모듈 선언과 참조](https://github.com/torvalds/linux/blob/master/include/linux/module.h), [문자 장치 열기 구현](https://github.com/torvalds/linux/blob/master/fs/char_dev.c)

## 6. 주 번호와 부 번호로 장치와 채널을 식별한다

문자·블록 장치 노드는 종류와 주·부 번호를 가진다. 주 번호는 등록된 장치 범위를 찾는 데 쓰이고, 부 번호는 그 안에서 개별 장치, 채널 또는 동작을 구분하는 데 쓸 수 있다. 같은 계수 장치가 입력 채널을 둘 제공한다면 각 노드에 다른 부 번호를 부여하고, `.open`에서 부 번호에 대응하는 채널 상태를 선택할 수 있다. 네트워크 인터페이스의 식별과 등록은 별도 경로를 따른다. [리눅스 장치 번호 목록](https://docs.kernel.org/admin-guide/devices.html), [문자 장치 번호 등록과 조회](https://github.com/torvalds/linux/blob/master/fs/char_dev.c)

커널 2.4에서 사용하던 `kdev_t`의 주·부 각 8비트 표현은 2.6에서 `dev_t` 기반 표현으로 바뀌었다. 현행 커널 내부의 `dev_t`는 32비트이며 `MINORBITS`는 20이다. `MAJOR()`와 `MINOR()`로 나누고 `MKDEV()`로 합친다. [2.4의 장치 번호 헤더](https://kernel.googlesource.com/pub/scm/linux/kernel/git/wtarreau/linux-2.4/%2B/refs/heads/master/include/linux/kdev_t.h), [2.6.12의 장치 번호 헤더](https://github.com/torvalds/linux/blob/v2.6.12/include/linux/kdev_t.h), [현행 `dev_t` 선언](https://github.com/torvalds/linux/blob/master/include/linux/types.h), [현행 장치 번호 매크로](https://github.com/torvalds/linux/blob/master/include/linux/kdev_t.h)

문자 장치 번호 범위를 직접 관리하려면 `alloc_chrdev_region()`과 `cdev_add()`를 사용할 수 있다. 장치 노드 하나를 공개하는 예라면 `misc_register()`와 `MISC_DYNAMIC_MINOR`로 부 번호를 동적 할당받을 수 있고, devtmpfs가 `/dev`에 마운트된 환경에서 커널의 디바이스 등록으로 노드가 생성된다. 고정 번호가 필요한 경우 할당된 번호의 목록과 충돌 여부를 확인한다. [문자 장치 등록 구현](https://github.com/torvalds/linux/blob/master/fs/char_dev.c), [misc 장치 등록](https://docs.kernel.org/driver-api/misc_devices.html), [devtmpfs 구현](https://github.com/torvalds/linux/blob/master/drivers/base/devtmpfs.c)
