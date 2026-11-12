# 리눅스 디바이스 드라이버의 초기화·종료와 `open`/`release`, I/O 영역 예약

문자 디바이스 드라이버는 모듈이 적재될 때, 개별 장치에 연결될 때, 사용자 프로그램이 장치 파일을 열 때 각각 다른 상태를 준비한다. 리눅스 v6.12를 기준으로 각 상태를 언제 초기화하고 해제하는지, 여러 사용자의 동시 접근을 어떻게 처리하는지, I/O 포트와 MMIO 영역을 어떻게 예약하는지 정리한다.

학습일 **2026-11-12**

## 1. 자원이 필요한 기간으로 초기화 시점 정하기

하드웨어가 리셋되면 레지스터가 기본 상태로 돌아간다. 드라이버는 장치가 제공할 기능에 맞춰 레지스터와 동작 모드를 설정하고, 커널에서 사용할 상태 구조체와 락 등을 준비한다. 종료 시에는 각 자원을 참조하는 경로가 끝난 뒤 해당 자원을 반납한다.

| 자원이 필요한 기간 | 초기화 | 종료 |
| --- | --- | --- |
| 모듈 적재부터 제거까지 | `module_init()`에 지정한 함수에서 공용 상태를 준비하고 드라이버를 등록한다. | `module_exit()`에 지정한 함수에서 등록을 해제하고 공용 자원을 반납한다. |
| 장치 연결부터 분리까지 | 버스의 `probe()`에서 장치별 상태를 할당하고 지원 여부와 동작 조건을 확인한다. 필요에 따라 주소 영역을 예약하고 IRQ 처리기를 등록한다. | `remove()`에서 새 접근을 막고 진행 중인 작업을 정리한 뒤 IRQ, 주소 영역 등 장치 자원을 반납한다. |
| 파일 객체가 살아 있는 동안 | `.open`에서 부 번호에 해당하는 장치나 채널을 선택하고, 열기마다 필요한 상태를 준비한다. | `.release`에서 열린 파일의 마지막 참조가 사라지면 그 파일 객체가 사용한 상태를 정리한다. |
| 장치를 사용하는 파일 객체가 하나 이상인 동안 | 첫 번째 열기에서 장치를 가동한다. 필요한 자원만 이때 확보할 수 있다. | 마지막 파일 객체의 `.release`에서 장치를 정지시키고 가동 중에만 필요한 자원을 반납할 수 있다. |

적재형 모듈은 `insmod` 등으로 적재될 때 초기화 함수가 실행되고, 정상적으로 `rmmod`가 진행되면 종료 함수가 실행된다. 내장 드라이버의 `module_init()`은 부팅 중에 실행되며, `module_exit()`은 모듈 제거 경로에 사용되므로 내장 드라이버에는 효과가 없다. 문자·블록·네트워크 드라이버는 해당 서브시스템의 등록·해제 API를 사용한다. 문자 장치는 주 번호와 부 번호로 구성된 장치 번호 범위에 `file_operations`를 연결할 수 있다. PCI처럼 버스에서 장치를 열거하는 경우, 모듈이 드라이버를 등록하면 커널이 일치하는 장치마다 `probe()`를 호출한다. 장치 하나의 `probe()`가 실패해도 등록된 드라이버는 다른 장치나 이후 연결될 장치와 결합할 수 있다. [드라이버 초기화와 종료](https://docs.kernel.org/6.12/driver-api/basics.html#driver-entry-and-exit-points), [드라이버 모델의 콜백](https://docs.kernel.org/6.12/driver-api/driver-model/driver.html), [문자 장치 등록 구현](https://github.com/torvalds/linux/blob/v6.12/fs/char_dev.c)

고정 장치 하나를 모듈 초기화 함수에서 직접 검출하는 구조에서는 장치가 없거나 복구하기 어려운 오류가 발생하면 이미 확보한 자원을 정리하고 적재를 실패시킬 수 있다. 버스 드라이버에서는 `probe()`가 장치별 검출과 오류를 처리한다. 검출에 레지스터 접근이나 기본 설정이 필요하면 연결 시점에 그 작업을 수행하고, 데이터 수집처럼 사용 중에만 필요한 동작은 첫 열기까지 미룰 수 있다. 인터럽트를 연결 시점부터 활성화한 장치는 열린 파일이 없어도 IRQ 처리기를 실행할 수 있다. [드라이버 모델의 검출과 오류 처리](https://docs.kernel.org/6.12/driver-api/driver-model/driver.html), [IRQ 관리 API](https://docs.kernel.org/6.12/core-api/genericirq.html)

## 2. 파일을 열 때와 마지막 참조를 해제할 때

### 2.1. `.open`에서 장치와 사용 상태 선택하기

사용자 프로그램이 장치 파일에 `open(path, flags)`를 호출하면 VFS는 문자 장치의 번호에 연결된 파일 연산을 찾는다. `<linux/fs.h>`의 `struct file_operations`에서 `.open`과 `.release`의 함수형은 모두 `int (struct inode *, struct file *)`이다. `inode`에서는 `iminor(inode)`로 부 번호를 얻을 수 있고, `file`에서는 열기 플래그인 `f_flags`와 드라이버가 설정하는 `private_data`를 사용할 수 있다. [VFS의 파일 연산](https://docs.kernel.org/6.12/filesystems/vfs.html), [문자 장치 열기 구현](https://github.com/torvalds/linux/blob/v6.12/fs/char_dev.c#L353-L403)

예를 들어 읽기와 쓰기를 모두 하려면 `O_RDWR`을 전달한다. 비블로킹 동작이 필요한 경우에는 `O_NONBLOCK`을 함께 지정할 수 있다. 이 플래그가 영향을 주는 연산과 대기 대신 반환할 결과는 장치 인터페이스에 따른다. [리눅스 `open(2)` 매뉴얼](https://man7.org/linux/man-pages/man2/open.2.html)

1. 부 번호로 사용할 장치나 채널을 찾고, 장치가 연결되어 있으며 요청한 방식으로 열 수 있는지 확인한다. 공용 상태와 부 번호 매핑은 파일 연산을 공개하기 전에 준비한다.
2. 배타적으로 여는 장치에서는 이미 사용 중인지 확인하고 필요하면 `-EBUSY`를 반환한다. 동시 열기를 허용한다면 장치별 락으로 사용자 수 확인과 첫 가동·마지막 정지 전환을 동기화한다. 그래야 동시에 들어온 두 열기가 모두 자신을 첫 사용자로 판단하는 일을 막을 수 있다.
3. 열기마다 필요한 상태를 할당하고 초기화해 `file->private_data`에 저장한다. 예를 들어 펄스 계수 장치는 열린 파일마다 읽기를 시작할 때의 계수값을 저장해 각 사용자가 자기 기준으로 증가량을 읽게 할 수 있다. 여러 파일 객체는 장치를 공유하면서도 각자의 기준값을 가진다.
4. 열기에 성공하면 콜백은 `0`을 반환하고 사용자 프로그램은 파일 디스크립터를 받는다. 장치가 없으면 `-ENODEV`, 메모리 할당이 실패하면 `-ENOMEM`처럼 음수 오류를 반환한다. 일반적인 C 라이브러리에서는 호출 결과 `-1`과 양수 `errno`를 확인한다. 열기 도중 실패하면 그 호출에서 확보한 자원과 사용 상태를 되돌린다. 실패한 열기의 정리를 `.release`에 맡길 수 없다. [열기와 실패 처리](https://github.com/torvalds/linux/blob/v6.12/fs/open.c), [파일 객체 해제 구현](https://github.com/torvalds/linux/blob/v6.12/fs/file_table.c#L376-L415)

부 번호마다 연산 자체가 달라지는 경우도 있다. `misc` 서브시스템은 여러 장치가 주 번호를 공유하도록 하고, 각 장치가 등록한 부 번호로 해당 `file_operations`를 선택한다. 열기 시 현재 파일 객체의 연산을 교체하고, 해당 장치가 자체 `.open`을 제공하면 호출한다. [`misc_open()` 구현](https://github.com/torvalds/linux/blob/v6.12/drivers/char/misc.c#L115-L160)

### 2.2. `.release`에서 열린 파일의 상태 정리하기

`close(fd)`는 파일 디스크립터 항목을 제거한다. `dup()`으로 복제하거나 `fork()`로 상속한 디스크립터가 같은 `struct file`을 계속 참조하면 파일 객체는 유지된다. VFS가 그 객체의 마지막 참조를 정리할 때 `.release`가 실행된다. 프로세스가 종료되면 커널이 그 프로세스가 가진 디스크립터를 정리하므로, `.release` 시점도 같은 참조 규칙을 따른다. [VFS의 `.release`](https://docs.kernel.org/6.12/filesystems/vfs.html), [파일 객체의 마지막 참조 처리](https://github.com/torvalds/linux/blob/v6.12/fs/file_table.c#L376-L415)

`.release`에서는 `private_data`에 연결한 열기별 메모리와 참조를 정리한다. 첫 열기에서 가동하는 설계라면 장치별 사용자 수를 줄이고 마지막 사용자가 떠날 때 장치를 정지시킨다. 이 사용자 수의 단위는 독립적으로 열린 파일 객체다. `.release`의 반환형은 `int`이며 관례적으로 `0`을 반환한다. VFS는 이 반환값을 버리고, `close()`에서는 구현된 `.flush`의 오류가 사용자 공간으로 전달될 수 있다. [`close()`와 `.flush` 구현](https://github.com/torvalds/linux/blob/v6.12/fs/open.c#L1429-L1487), [`.release` 호출 구현](https://github.com/torvalds/linux/blob/v6.12/fs/file_table.c#L395-L408)

## 3. 모듈 참조와 장치 분리 시의 정리

### 3.1. 파일 연산을 실행하는 모듈의 수명

모듈로 빌드한 문자 장치에서는 보통 `file_operations`의 `.owner = THIS_MODULE`을 지정한다. 커널은 파일 연산을 가져올 때 소유 모듈의 참조를 확보하고, 파일 객체가 정리될 때 참조를 반납한다. 이 참조가 남아 있으면 정상적인 모듈 제거가 제한된다. 별도의 커널 호출 경로에서 참조를 직접 관리할 때는 호출 전에 모듈의 수명이 보장되어 있어야 한다. `try_module_get(module)`이 성공하면 사용이 끝난 뒤 `module_put(module)`을 호출하고, 제거 중이라 참조를 얻지 못하면 해당 사용을 시작하지 않는다. [파일 연산의 모듈 참조 구현](https://github.com/torvalds/linux/blob/v6.12/include/linux/fs.h#L2435-L2444), [모듈 참조 API](https://docs.kernel.org/6.12/driver-api/basics.html#c.try_module_get)

리눅스 2.4 계열의 적재형 모듈에서는 `MOD_INC_USE_COUNT`와 `MOD_DEC_USE_COUNT`로 사용 횟수를 관리하고, `MOD_IN_USE`로 모듈이 사용 중인지 확인했다. 열기에 성공할 때 얻은 참조는 사용이 끝날 때 반납해야 했다. 이 방식은 참조를 증가시키기 전에 이미 모듈 코드에 진입할 수 있다는 경쟁 조건이 있었다. 2.6 계열에서는 호출하는 쪽이 먼저 모듈 참조를 확보하는 방식으로 바뀌었고, 기존 증감 매크로는 폐기 절차를 거쳤다. 배타적 열기 여부는 모듈 전체의 참조 수와 별개로 장치별 상태에서 판단한다. [리눅스 2.4 모듈 헤더](https://kernel.googlesource.com/pub/scm/linux/kernel/git/wtarreau/linux-2.4/%2B/refs/heads/master/include/linux/module.h#295), [기존 증감 매크로 제거 패치](https://lkml.iu.edu/0409.2/0800.html)

### 3.2. 자원 확보 실패와 핫플러그

등록과 할당은 대응하는 해제 API와 짝을 이룬다. 초기화 중 실패하면 성공한 단계만 의존 관계의 역순으로 정리한다. 예를 들어 상태 메모리를 할당하고 MMIO 영역을 예약했는데 매핑에 실패하면, 예약을 해제한 뒤 메모리를 반납한다. 아직 확보하지 않은 자원까지 해제하면 커널 오류가 날 수 있고, 해제를 빠뜨리면 반복 적재나 재연결 중에 메모리나 주소 영역이 누적해서 남을 수 있다. 장치에 귀속되는 자원에 `devm_*` API를 쓰면 `probe()` 실패나 장치 분리 시 커널이 등록된 자원을 해제한다. [장치 자원 관리와 오류 경로](https://docs.kernel.org/6.12/driver-api/driver-model/devres.html)

IRQ 처리기와 파일 연산이 상태를 공유한다면 공유 메모리와 락을 처리기 등록 전에 초기화한다. 열린 파일이 모듈을 참조하는 중에도 물리 장치는 빠질 수 있다. 이때 새 열기를 막고, 이미 열린 파일의 후속 입출력에서 장치가 사라졌음을 확인해 `-ENODEV` 등으로 응답할 수 있게 한다. 진행 중인 입출력과 비동기 작업이 장치 자원에 접근하지 않도록 동기화하고, 장치에 접근할 수 있다면 인터럽트 발생을 멈춘 뒤 처리기를 해제한다. IRQ 처리기가 사용하는 메모리와 레지스터 매핑은 처리기가 끝난 뒤 해제한다. 기존 파일의 콜백에 필요한 소프트웨어 상태는 마지막 참조까지 유지한다. [장치 분리 콜백](https://docs.kernel.org/6.12/driver-api/driver-model/driver.html#remove), [IRQ 관리 API](https://docs.kernel.org/6.12/core-api/genericirq.html), [문자 장치 등록 해제 후 열린 파일의 동작](https://github.com/torvalds/linux/blob/v6.12/fs/char_dev.c)

## 4. I/O 포트와 MMIO 영역의 충돌 방지

장치는 주소 영역 외에도 IRQ와 DMA 같은 자원을 사용할 수 있다. 주소가 고정된 장치끼리는 같은 범위를 요구할 수 있다. PCI에서는 커널이 열거한 장치의 BAR 자원 정보를 사용하지만, 드라이버는 사용하려는 영역을 `pci_request_region()` 같은 API로 예약한다. 주소 영역의 예약과 IRQ·DMA의 관리는 각각 해당 API가 담당한다. [PCI 드라이버의 자원 확보](https://docs.kernel.org/6.12/PCI/pci.html#request-mmio-iop-resources)

| 주소 영역 | 접근 방식 | 예약과 해제 | 확인 경로 |
| --- | --- | --- | --- |
| I/O 포트 | 별도의 포트 주소 공간을 사용한다. x86에서는 `inb()`, `outb()` 같은 접근 함수를 쓴다. | `request_region(start, len, name)`, `release_region(start, len)` | `/proc/ioports` |
| MMIO | 물리 주소 공간에 있는 장치 레지스터나 장치 메모리를 커널 주소 공간에 매핑한다. 레지스터에는 `readl()`, `writel()` 같은 접근 함수를 쓴다. | `request_mem_region(start, len, name)`, `release_mem_region(start, len)` | `/proc/iomem` |

두 예약 매크로는 `<linux/ioport.h>`에 있다. `start`는 시작 주소, `len`은 길이, `name`은 자원 목록에서 소유자를 식별할 이름이다. 예약에 성공하면 `struct resource *`를 반환하고, 기존에 점유된 범위와 충돌하거나 자원 구조체 할당에 실패하면 `NULL`을 반환한다. 해제할 때는 예약에 사용한 시작 주소와 길이를 그대로 전달한다. 예를 들어 가상의 장치가 포트 `0x680`부터 8개를 사용한다면 예약 범위는 `0x680`~`0x687`이다. 다른 드라이버가 그중 `0x684`부터 시작하는 범위를 요청하면 일부만 겹쳐도 충돌한다. 이 예약 전체를 해제할 때는 `release_region(0x680, 8)`을 호출한다. [I/O 영역 매크로](https://github.com/torvalds/linux/blob/v6.12/include/linux/ioport.h#L276-L295), [커널 자원 트리 구현](https://github.com/torvalds/linux/blob/v6.12/kernel/resource.c), [`/proc` 자원 목록](https://docs.kernel.org/6.12/filesystems/proc.html)

과거에는 `check_region()`과 `check_mem_region()`으로 범위가 비어 있는지 먼저 확인했다. 검사 결과가 `0`이어도 실제 예약 전에 다른 드라이버가 그 범위를 차지할 수 있었다. v6.12에서는 예약을 바로 요청하고 그 반환값으로 성공 여부를 판단한다. [기존 검사 API 제거 커밋](https://git.zx2c4.com/linux-dev/commit/?id=96831c0a6738f88f89e7012f4df0a747514af0a0), [v6.12의 예약 API](https://github.com/torvalds/linux/blob/v6.12/include/linux/ioport.h#L276-L295)

MMIO를 직접 관리할 때는 영역 예약 후 `ioremap()`으로 매핑하고, 사용이 끝나면 `iounmap()` 후 예약을 해제한다. 장치에 귀속된 MMIO라면 `devm_ioremap_resource()`로 예약과 매핑을 함께 관리할 수 있다. 장치가 연결된 동안 계속 사용하는 영역은 `probe()`에서 확보한다. 파일이 열려 있을 때만 쓰는 영역이라면 첫 `.open`에서 확보하고 마지막 `.release`에서 반납할 수 있다. 이 경우 다음 열기에서 예약이 다시 성공하는지 확인하고, 해당 영역을 사용하는 IRQ 처리기나 다른 작업도 같은 기간에 맞춰 정리한다. [장치 주소 매핑과 접근](https://docs.kernel.org/6.12/driver-api/device-io.html), [`devm_ioremap_resource()` 구현](https://github.com/torvalds/linux/blob/v6.12/lib/devres.c#L159-L183)
