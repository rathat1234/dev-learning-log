# 리눅스 문자 디바이스의 등록·해제와 `struct file_operations`

가상의 문자 장치가 여러 입력 채널을 제공한다고 가정한다. 리눅스 v6.12에서 채널에 장치 번호를 배정하고, 노드를 열 때 VFS가 어떤 콜백을 찾으며, 등록을 해제할 때 열린 파일에 어떤 영향이 있는지 살펴본다.

학습일 **2026-11-11**

## 1. `register_chrdev()`와 `unregister_chrdev()`

두 함수는 `<linux/fs.h>`에 선언된다. v6.12의 `register_chrdev()`는 주 번호 하나의 기본 부 번호 범위를 예약하고, 내부에서 `cdev`를 생성해 `fops`와 장치 번호 범위를 연결한다. 번호를 예약할 때는 주 번호에서 계산한 해시 버킷을 찾아 같은 주 번호에서 부 번호 범위가 겹치는지 확인한다. `name`은 `/proc/devices`와 커널 진단 메시지에서 등록 항목을 식별하는 데 쓰인다. `/dev`에 표시할 노드 이름과 노드의 생성·삭제는 별도로 관리한다. 사용자 프로그램이 열 노드에는 등록 범위에 맞는 문자 장치 유형과 번호가 들어 있어야 한다. [등록 함수 선언](https://github.com/torvalds/linux/blob/v6.12/include/linux/fs.h#L2634-L2652), [문자 장치 등록 구현](https://github.com/torvalds/linux/blob/v6.12/fs/char_dev.c), [장치 번호와 노드 이름](https://docs.kernel.org/admin-guide/devices.html)

| API와 조건 | 동작과 반환값 |
| --- | --- |
| `register_chrdev(0, name, fops)` | 비어 있는 주 번호를 동적으로 할당한다. 성공하면 양수인 실제 주 번호를 반환하므로 노드 생성과 해제에 사용할 값을 보관한다. |
| `register_chrdev(major, name, fops)` (`major > 0`) | 지정한 주 번호의 범위를 예약한다. 성공하면 `0`을 반환하므로 지정한 번호를 계속 사용한다. |
| 등록 실패 | 장치 번호 범위가 충돌하거나 동적 할당에 쓸 주 번호가 없으면 `-EBUSY`, 유효 범위를 벗어나면 `-EINVAL`, 관리 객체 할당에 실패하면 `-ENOMEM`이 발생할 수 있다. |
| `unregister_chrdev(major, name)` | 등록할 때 사용한 이름과 실제 주 번호를 전달한다. 반환형은 `void`다. v6.12 구현은 주 번호와 부 번호 범위로 항목을 찾아 예약을 해제하고 대응하는 `cdev`를 제거한다. |

부 번호 범위를 직접 정하려면 `alloc_chrdev_region()`이나 `register_chrdev_region()`으로 번호를 확보한 뒤 `cdev_init()`과 `cdev_add()`로 연산을 연결할 수 있다. 이 방식의 해제는 `cdev_del()`과 `unregister_chrdev_region()`이 맡는다. `cdev_del()`은 이후의 열기를 막는다. 이미 열린 파일에서는 콜백이 계속 호출될 수 있으므로, 열린 파일이 참조하는 상태의 수명도 관리해야 한다. [문자 장치 번호 예약과 해제 구현](https://github.com/torvalds/linux/blob/v6.12/fs/char_dev.c), [`cdev_del()` 동작](https://docs.kernel.org/core-api/kernel-api.html#c.cdev_del)

## 2. 장치 번호를 파일 연산에 연결하는 과정

1. VFS는 `open()`에 전달된 경로를 탐색하고 접근 권한을 확인한다. 문자 장치 노드의 inode에는 유형과 `i_rdev`가 들어 있다. `i_rdev`는 주 번호(major)와 부 번호(minor)를 담은 `dev_t` 값이다. 주 번호는 등록 범위를 찾는 데 쓰이고, 부 번호는 그 범위에 속한 장치나 채널을 식별할 수 있게 한다.
2. 커널의 문자 장치 열기 경로는 장치 번호가 속한 범위의 `struct cdev`를 찾는다. 해당 `cdev`의 `file_operations`를 열린 파일 객체인 `struct file`의 `f_op`에 연결하고, 드라이버가 `.open`을 지정했다면 `struct inode *`와 `struct file *`를 넘겨 호출한다.
3. 열기에 성공하면 사용자 프로그램은 파일 디스크립터 `fd`를 받는다. 이후 호출에서 커널은 프로세스의 디스크립터 테이블을 통해 `struct file`을 찾고 `f_op`의 해당 콜백을 호출한다. 따라서 장치 번호로 연산을 선택하는 단계는 열 때 수행하며, 이후 입출력은 열린 파일 객체를 통해 처리한다.

예를 들어 같은 드라이버가 이벤트 센서 두 개를 담당하면 주 번호를 공유하고 부 번호로 센서를 구분할 수 있다. `.open`은 `iminor(inode)`로 센서를 선택해 그 상태를 `file->private_data`에 저장한다. 이후 `.read`는 `private_data`가 가리키는 센서의 큐에서 이벤트를 가져온다. 장치 번호는 장치 노드의 식별자이고, `fd`는 열린 파일 객체를 가리키는 프로세스의 핸들이다. [문자 장치의 열기 경로](https://github.com/torvalds/linux/blob/v6.12/fs/char_dev.c#L353-L403), [VFS의 파일 객체](https://docs.kernel.org/filesystems/vfs.html)

## 3. `struct file_operations`의 콜백과 반환값

`<linux/fs.h>`의 `struct file_operations`에는 연산별 함수 포인터와 소유 모듈을 나타내는 `.owner`, 파일 연산 플래그인 `fop_flags` 등이 있다. 모듈에서는 보통 `static const` 객체를 만들고 `.owner = THIS_MODULE`과 구현한 콜백을 지정 초기화한다. 생략한 콜백은 `NULL`로 초기화된다. 생략했을 때의 동작은 연산별로 다르다. `.open`은 생략할 수 있고, 읽기는 `.read_iter`로도 구현할 수 있다. 탐색을 지원하지 않는 파일에서 `lseek()`를 호출하면 `ESPIPE`가 발생한다. `.poll`, `.mmap`, `.fasync` 같은 연산도 장치가 제공하는 기능에 따라 선택한다. [파일 연산 선언](https://github.com/torvalds/linux/blob/v6.12/include/linux/fs.h#L1929-L1984), [탐색과 읽기 구현](https://github.com/torvalds/linux/blob/v6.12/fs/read_write.c)

| 사용자 공간 호출 | 대응하는 콜백과 인자 | 사용자 공간에서 받는 결과 |
| --- | --- | --- |
| `open(pathname, flags)` | `.open(inode, file)`. 커널이 경로를 해석하며, 드라이버는 `inode`에서 장치 번호를, `file`에서 `f_flags` 등 열기 상태를 확인한다. | 콜백의 `0`은 초기화 성공을 뜻한다. 전체 열기가 성공하면 커널이 할당한 `fd`가 돌아온다. |
| `read(fd, buf, count)` | `.read(file, buf, count, ppos)`. `buf`는 결과를 받을 사용자 버퍼이고, `count`는 요청 바이트 수다. | 실제로 읽은 바이트 수가 돌아온다. 양수 길이를 요청했을 때 `0`은 일반적으로 EOF를 뜻한다. |
| `write(fd, buf, count)` | `.write(file, buf, count, ppos)`. `buf`는 데이터를 담은 사용자 버퍼이고, `count`는 요청 바이트 수다. | 실제로 쓴 바이트 수가 돌아온다. 요청량보다 작을 수 있다. |
| `lseek(fd, offset, whence)` | `.llseek(file, offset, whence)`. `whence`가 지정한 기준 위치에서 `offset`을 적용한다. | 새 파일 위치가 돌아온다. |
| `ioctl(fd, request, ...)` | `.unlocked_ioctl(file, cmd, arg)`. `request`는 `cmd`에 대응한다. `arg`의 의미는 명령이 정하며, 인자가 없는 명령은 `arg`를 읽지 않는다. | 성공은 보통 `0`이다. 지원하지 않는 장치 명령에는 `-ENOTTY`를 반환한다. |
| `close(fd)` | `fd` 항목을 제거하고 `.flush`가 있으면 호출한다. `.release(inode, file)`은 열린 파일 객체의 마지막 참조가 해제될 때 실행된다. | `.flush`의 오류는 `close()`에 전달될 수 있다. `.release`는 반환형이 `int`지만 VFS는 그 값을 사용하지 않는다. |

콜백이 반환한 `-EFAULT` 같은 음수 오류는 시스템 호출의 오류가 된다. 일반적인 C 라이브러리 래퍼를 사용하면 프로그램에서는 반환값 `-1`과 양수 `errno`를 확인한다. VFS는 콜백을 호출하기 전에 유효한 `fd`인지, 읽거나 쓸 수 있는 모드인지 등을 검사할 수 있다. `dup()` 등으로 하나의 파일 객체를 여러 `fd`가 공유하면 하나를 닫은 뒤에도 다른 `fd`로 입출력할 수 있으며, 마지막 참조에서 `.release`가 실행된다. [VFS의 파일 연산](https://docs.kernel.org/filesystems/vfs.html), [`close()` 경로](https://github.com/torvalds/linux/blob/v6.12/fs/open.c#L1429-L1488), [파일 객체의 정리](https://github.com/torvalds/linux/blob/v6.12/fs/file_table.c#L395-L408)

`.read`는 `copy_to_user()`로 커널 데이터를 사용자 버퍼에 복사하고, `.write`는 `copy_from_user()`로 사용자 데이터를 커널로 가져온다. `buf`는 사용자 공간 주소이므로 이러한 접근 함수를 사용한다. 두 함수는 복사하지 못한 바이트 수를 반환하므로 `0`이면 전부 복사한 것이다. 위치 기반 장치는 커널이 넘긴 `ppos`로 현재 위치를 읽고 실제 처리량만큼 갱신한다. 스트림으로 열린 파일은 위치 인자가 `NULL`일 수 있다. [사용자 공간 메모리 접근](https://docs.kernel.org/kernel-hacking/hacking.html#copy-to-user-copy-from-user-get-user-put-user), [읽기·쓰기와 파일 위치 처리](https://github.com/torvalds/linux/blob/v6.12/fs/read_write.c)

센서의 샘플링 간격을 바꾸는 `ioctl()` 명령이라면 세 번째 인자로 새 간격을 담은 사용자 구조체의 주소를 넘길 수 있다. 드라이버는 `cmd`로 명령을 확인하고 구조체를 복사한 뒤 값을 검증한다. 파일 연산 구조체의 필드와 함수형은 커널 버전에 따라 바뀐다. 구형 커널에는 `.ioctl` 필드가 있었고, v6.12에서는 `.unlocked_ioctl`과 32비트 사용자 공간 호환 처리를 위한 `.compat_ioctl`을 확인할 수 있다. [커널 ioctl 인터페이스](https://docs.kernel.org/driver-api/ioctl.html), [v6.12 파일 연산 선언](https://github.com/torvalds/linux/blob/v6.12/include/linux/fs.h#L1929-L1946)

## 4. 사용자 프로그램에서 장치에 접근하는 경로

사용자 공간의 프로세스는 시스템 호출을 통해 커널에 장치 작업을 요청한다. 일반 파일의 내용은 파일 시스템이 관리한다. 문자 디바이스 파일은 장치 유형과 장치 번호를 담은 특수 파일이다. VFS는 이 정보를 이용해 커널에 등록된 드라이버를 찾고, 드라이버는 요청에 따라 하드웨어를 제어하거나 내부 버퍼를 처리한다. 드라이버는 커널에 내장하거나 실행 중 적재하는 모듈로 구성할 수 있다. [리눅스 VFS](https://docs.kernel.org/filesystems/vfs.html), [드라이버 초기화와 종료](https://docs.kernel.org/driver-api/basics.html#driver-entry-and-exit-points)

| 유형 | 처리 단위와 일반적인 접근 경로 |
| --- | --- |
| 문자 | 장치가 정의한 바이트 스트림이나 레코드를 다룬다. 사용자 프로그램은 장치 파일을 열어 `read()`, `write()`, `ioctl()` 등을 호출한다. |
| 블록 | 저장장치의 데이터를 블록 단위로 처리한다. 파일 시스템은 블록 계층을 통해 입출력을 요청한다. 권한이 있으면 사용자 프로그램도 블록 장치 파일을 열 수 있다. |
| 네트워크 | 패킷을 송수신한다. 사용자 프로그램은 주로 소켓을 사용하고, 커널의 네트워크 스택이 장치 드라이버와 통신한다. |

문자 장치의 읽기·쓰기 단위와 종료 조건은 장치의 인터페이스에 따라 정해진다. 예를 들어 센서 이벤트는 계속 들어올 수 있으므로, 이벤트 큐가 비었을 때 블로킹 읽기는 다음 이벤트를 기다리고 비블로킹 읽기는 `EAGAIN`을 반환하도록 구현할 수 있다. 스트림 장치에서는 임의 위치로 이동하는 연산도 생략할 수 있다. [블록 계층](https://docs.kernel.org/block/blk-mq.html), [네트워크 장치](https://docs.kernel.org/networking/netdevices.html), [커널의 대기 큐](https://docs.kernel.org/driver-api/basics.html#wait-queues-and-wake-events)

## 5. 모듈 적재부터 제거까지

1. **구현**. 필요한 커널 헤더를 포함하고 장치 상태, 파일 연산 콜백, `file_operations` 객체를 준비한다. 하드웨어가 인터럽트를 사용하면 인터럽트 서비스 루틴을 별도로 둔다. 파일 연산 콜백은 사용자 요청을 처리하고, 인터럽트 서비스 루틴은 장치가 알린 이벤트를 처리한다.
2. **적재와 초기화**. `insmod`로 모듈을 적재하면 커널이 필요한 공개 심볼을 연결한 뒤 `module_init()`에 지정된 초기화 함수를 실행한다. 초기화 함수는 콜백이 사용할 상태를 준비하고 `register_chrdev()`로 연산을 등록한다. 필요한 경우 장치 노드를 제공한다. 중간에 실패하면 이미 확보한 자원을 역순으로 정리하고 음수 오류를 반환한다.
3. **사용**. 사용자 프로그램이 장치 노드를 열고 얻은 `fd`로 읽기, 쓰기, 제어를 요청한다. `.owner = THIS_MODULE`을 설정하면 열린 파일이 파일 연산을 참조하는 동안 모듈 참조가 유지된다.
4. **제거와 정리**. `rmmod`로 모듈 제거를 요청하고 참조가 남아 있지 않으면 `module_exit()`에 지정된 종료 함수가 실행된다. 종료 함수는 생성한 장치 노드를 정리하고 `unregister_chrdev()`로 연결을 해제한다. 인터럽트를 등록했다면 처리기를 해제한 뒤 그 처리기가 참조하던 상태를 정리한다.

[모듈의 초기화와 종료](https://docs.kernel.org/driver-api/basics.html#driver-entry-and-exit-points), [커널의 심볼 공개와 모듈 사용](https://docs.kernel.org/kernel-hacking/hacking.html), [인터럽트 관리 API](https://docs.kernel.org/core-api/genericirq.html), [모듈 참조와 파일 연산](https://github.com/torvalds/linux/blob/v6.12/include/linux/fs.h#L2437-L2445)
