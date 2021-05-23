---
typora-root-url: ../../blog
---

다음은 윤성우의 열혈 TCP/IP 소켓 프로그래밍 책을 공부하고 정리한 내용입니다! 1주일에 두 챕터씩 공부할 예정이에요.



# Chapter 01 네트워크 프로그래밍과 소켓의 이해

## 01-1 네트워크 프로그래밍과 소켓의 이해

네트워크 프로그래밍(= 소켓프로그래밍)이란 서로 다른 두 컴퓨터가 데이터를 주고받을 수 있도록 하는 것이다. 

* 소켓이란?

  운영체제에서 제공하는 데이터 송수신에 사용할 수 있는 소프트웨어적인 장치, 또는 네트워크를 통한 두 컴퓨터의 연결

  전력망으로부터 전기를 공급받기 위한 소켓(플러그)로부터 유래되었다.

### TCP Socket

TCP: Transmission Control Protocol

#### Listening Socket(= Server Socket)

##### 1. 소켓 생성

```c
#inlcude <sys/socket.h>
int socket(int domain, int type, int protocol);

// 성공 시 파일 디스크립터, 실패 시 -1 반환
```

##### 2. 생성한 소켓에 주소정보(= IP와 port number) 부여

```c
#include <sys/socket.h>
int bind(int sockfd, struct sockaddr *myaddr, socklen_t addrlen);

// 성공 시 0, 실패 시 -1 반환
```

##### 3. 소켓을 연결요청이 가능한 상태로 변경

```c
#include <sys/socket.h>
int listen(int sockfd, int backlog);

// 성공 시 0, 실패 시 -1 반환
```

##### 4. 누군가 해당 소켓으로 연결 요청했을 때 수락

```c
#include <sys/socket.h>
int accept(int sockfd, struct sockaddr *addr, socklen_t *addrlen);

// 성공 시 파일 디스크립터, 실패 시 -1 반환
// 연결 요청이 없는 상태에서 호출되면 연결 요청이 있을 때까지 반환되지 않음
```

##### "Hello world!" 서버 프로그램 구현

```c
// hello_server.c

#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <arpa/inet.h>
#include <sys/socket.h>

void error_handling(char *message);

int main(int argc, char *argv[])
{
    int serv_sock;
    int clnt_sock;

    struct sockaddr_in serv_addr;
    struct sockaddr_in clnt_addr;
    socklen_t clnt_addr_size;

    char message[] = "Hello World!";

    if (argc != 2)
    {
        printf("Usage : %s <port>\n", argv[0]);
        exit(1);
    }

    serv_sock = socket(PF_INET, SOCK_STREAM, 0);
    if (serv_sock == -1)
        error_handling("socket() error");

    memset(&serv_addr, 0, sizeof(serv_addr));
    serv_addr.sin_family = AF_INET;
    serv_addr.sin_addr.s_addr = htonl(INADDR_ANY);
    serv_addr.sin_port = htons(atoi(argv[1]));

    if (bind(serv_sock, (struct sockaddr *)&serv_addr, sizeof(serv_addr)) == -1)
        error_handling("bind() error");

    if (listen(serv_sock, 5) == -1)
        error_handling("listen() error");

    clnt_addr_size = sizeof(clnt_addr);
    clnt_sock = accept(serv_sock, (struct sockaddr *)&clnt_addr, &clnt_addr_size);
    if (clnt_sock == -1)
        error_handling("accept() error");

    write(clnt_sock, message, sizeof(message));
    close(clnt_sock);
    close(serv_sock);
    return 0;
}

void error_handling(char *message)
{
    fputs(message, stderr);
    fputc('\n', stderr);
    exit(1);
}
```

#### Client Socket

##### 연결 요청

```c
#include <sys/socket.h>
int connect(int sockfd, struct sockaddr *serv_addr, socklen_t addrlen);

// 성공 시 0, 실패 시 -1 반환
```

##### 클라이언트 프로그램 구현

클라이언트 소켓의 port number는 랜덤하게 할당됨

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <arpa/inet.h>
#include <sys/socket.h>

void error_handling(char *message);

int main(int argc, char *argv[])
{
    int sock;
    struct sockaddr_in serv_addr;
    char message[30];
    int str_len;

    if (argc != 3)
    {
        printf("Usage : %s <IP> <port>\n", argv[0]);
        exit(1);
    }

    sock = socket(PF_INET, SOCK_STREAM, 0);
    if (sock == -1)
        error_handling("socket() error");

    memset(&serv_addr, 0, sizeof(serv_addr));
    serv_addr.sin_family = AF_INET;
    serv_addr.sin_addr.s_addr = inet_addr(argv[1]);
    serv_addr.sin_port = htons(atoi(argv[2]));

    if (connect(sock, (struct sockaddr *)&serv_addr, sizeof(serv_addr)) == -1)
        error_handling("connect() error!");

    str_len = read(sock, message, sizeof(message) - 1);
    if (str_len == -1)
        error_handling("read() error!");

    printf("Message from server: %s \n", message);
    close(sock);
    return 0;
}

void error_handling(char *message)
{
    fputs(message, stderr);
    fputc('\n', stderr);
    exit(1);
}
```

##### 리눅스 기반에서 실행하기

```bash
$ gcc hello_server.c -o hserver
$ ./hserver 9190

# 아래 스크립트를 실행해서 연결 요청을 받아야 프로세스 종료됨
```

```bash
$ gcc hello_client.c -o hclient
$ ./hclient 127.0.0.1 9190
Message from server: Hello World!
```

## 01-2 리눅스 기반 파일 조작하기

리눅스는 소켓을 파일의 일종으로 구분한다. 따라서 파일 입출력 함수를 소켓 입출력(= 네트워크 상에서 데이터 송수신)에 사용할 수 있다.

### Low-level File Access & File Descriptor

>  Low-level: 표준에 관계 없이 운영체제가 독립적으로 제공한다는 의미이다. (ANSI 표준에서 정의한 함수가 아님)

* File descriptor: 리눅스가 시스템으로부터 할당 받은 파일(or 소켓)에 지칭하기 편하도록 부여한 정수. 표준 입출력 및 표준 에러에도 부여된다.

  | File descriptor |           대상            |
  | :-------------: | :-----------------------: |
  |        0        | 표준입력: Standard Input  |
  |        1        | 표준출력: Standard Output |
  |        2        | 표준에러: Standard Error  |

  파일과 소켓은 생성의 과정을 거쳐야 파일 디스크립터가 할당되는 반면, 위 세 가지 입출력 대상은 별도의 생성 과정을 거치지 않아도 프로그램이 실행되면 자동으로 파일 디스크립터가 할당된다. 이때문에 생성된 파일의 디스크립터는 3부터 시작한다.

  윈도우에서는 file handle이라고 한다.

  Os의 file descriptor table에 저장됨

  프로세스가 파일을 관리하는 게 아니고 OS가 관리함. 프로세스는 file descriptor 4 byte int만 관리함

### 파일 열기

데이터를 읽거나 쓰기 위해 파일을 열 때 사용하는 함수

```c
#include <sys/types.h>
#include <sys/stat.h>
#include <fcntl.h>

int open(const char *path, int flag);
// path: 파일 경로
// flag: 파일 오픈 모드(특성 정보)

// 성공 시 파일 디스크립터, 실패 시 -1 반환
```

* `flag` 에 전달할 수 있는 값

  | 오픈 모드 |                  의미                  |
  | :-------: | :------------------------------------: |
  |  O_CREAT  |          필요하면 파일을 생성          |
  |  O_TRUNC  |         기존 데이터 전부 삭제          |
  | O_APPEND  | 기존 데이터 보존하고, 뒤에 이어서 저장 |
  | O_RDONLY  |        읽기 전용으로 파일 오픈         |
  | O_WRONLY  |        쓰기 전용으로 파일 오픈         |
  |  O_RDWR   |      읽기 쓰기 겸용으로 파일 오픈      |


### 파일 닫기

```c
#include <unistd.h>

int close(int fd);
// fd: 닫고자 하는 파일(or 소켓)의 파일 디스크립터

// 성공 시 0, 실패 시 -1 반환
```

### 파일에 데이터 쓰기

파일에 데이터를 출력(전송)하는 함수. 소켓을 통해서 다은 컴퓨터에 데이터를 전송할 때도 사용한다.

```c
#include <unistd.h>

ssize_t write(int fd, const void * buf, size_t nbytes);
// fd: 데이터 전송대상을 나타내는 파일 디스크립터
// buf: 전송할 데이터가 저장된 버퍼의 주소 값
// nbytes: 전송할 데이터의 바이트 수

// 성공 시 전달한 바이트 수, 실패 시 -1 반환
```





# Chapter 02 소켓의 타입과 프로토콜의 설정

## 02-1 소켓의 프로토콜과 그에 따른 데이터 전송 특성

### Protocol

**컴퓨터 상호간의 대화에 필요한 통신규약.** 서로 데이터를 주고 받기 위해서 정의해 놓은 약속.

> 멀리 떨어져 있는 두 사람이 대화를 주고받으려 할 때, 한 사람이 전화를 통해 대화를 시도하면 다른 사람도 전화를 통해 대화해야 하지 편지를 쓰면 안 된다. 이 때 두 사람이 대화에 사용한 프로토콜은 전화라고 이야기할 수 있다.

### 소켓의 생성

`socket` 함수를 제대로 이해해보자.

```c
#include <sys/socket.h>

int socket(int domain, int type, int protocol);
/*
domain: 소켓이 사용할 프로토콜 체계(Protocol Family) 정보 전달
type: 소켓의 데이터 전송 방식에 대한 정보 전달
protocol: 두 컴퓨터간 통신에 사용되는 프로토콜 정보 전달

성공 시 file descriptor, 실패 시 -1 반환
*/
```

#### Protocol Family(프로토콜 체계)

다음은 헤더파일 `sys/socket.h` 에 선언되어있는 프로토콜 체계이다.

|   이름    |   Protocol Family(프로토콜 체계)    |
| :-------: | :---------------------------------: |
|  PF_INET  |      IPv4 인터넷 프로토콜 체계      |
| PF_INET6  |      IPv6 인터넷 프로토콜 체계      |
| PF_LOCAL  | 로컬 통신을 위한 UNIX 프로토콜 체계 |
| PF_PACKET | Low level 소켓을 위한 프로토콜 체계 |
|  PF_IPX   |       IPX 노벨 프로토콜 체계        |

PF_INET에 해당하는 IPv4 인터넷 프로토콜 체계가 이 책에서 주로 설명하는 프로토콜 체계이다. 나머지 프로토콜 체계는 중요도가 떨어지거나 아직 보편화되지 않은 프로토콜 체계이기 때문이다.

프로토콜 체계가 결정되었다고 해서 소켓의 데이터 전송 방식이 결정되는 것은 아니다. PF_INET에 해당하는 프로토콜 체계에도 둘 이상의 데이터 전송 방식이 존재한다.

#### 소켓의 타입

**소켓의 데이터 전송 방식**

##### 1. `SOCK_STREAM` : 연결지향형 소켓

**신뢰성 있는 순차적인 바이트 기반의 연결지향 데이터 전송 방식의 소켓**

* 중간에 데이터가 소멸되지 않고 목적지로 전송된다.

* 전송 순서대로 데이터가 수신된다.

* 전송되는 데이터의 경계(boundary)가 존재하지 않는다.

  데이터를 송수신하는 소켓은 내부적으로 버퍼(buffer), 즉, 바이트 배열을 가지고 있으며, 소켓을 통해 전송되는 데이터는 일단 이 배열에 저장된다. 따라서 데이터가 수신되었다고 해서 바로 `read` 함수를 호출해야 하는 것은 아니다.

  버퍼의 용량을 초과하지 않는 한, 데이터가 채워진 후에 한 번의 `read` 함수 호출을 통해서 데이터 전부를 읽어 들일 수도 있고, 반대로 한 번의 `write` 함수 호출로 전송된 데이터 전부를 여러 번의 `read` 함수 호출을 통해서 읽어들일 수도 있다. `read` 함수와 `write` 함수의 호출 횟수가 큰 의미가 없다.

* 소켓 대 소켓의 연결은 반드시 1:1이어야 한다: 연결지향형 소켓은 다른 연결지향형 소켓 하나와만 연결 가능하다.

> 예시) 독립된 별도의 전송 라인을 통해서 월병을 주고받는 상황
>
> ![월병토끼](/images/2021-05-23/conveyor_belt.gif)
>
> * 라인상의 문제만 없다면 데이터가 소멸되지 않음을 보장받을 수 있다.
> * 먼저 보내진 데이터보다 뒤에 보내진 데이터가 일찍 도착할 수 없다. 전송라인에 올려진 순서대로 데이터가 전달되기 때문이다.
> * 월병 100개가 여러 번에 걸쳐서 보내졌어도, 가져가는 사람은 월병 100개가 쌓인 다음에 한 번에 가져갈 수 있다.

**소켓에 존재하는 버퍼가 꽉 차면 데이터가 사라질까?**

> 수신되는 데이터의 저장을 위한 이 버퍼는 `read` 함수 호출을 통해서 데이터가 읽혀지면 읽혀진 만큼 버퍼에서 비워지게 된다. 만약 `read` 함수 호출로 읽혀지는 데이터의 양보다 많은 양의 데이터가 수신돼서 버퍼가 꽉 차면 어떻게 될까?

연결지향형 소켓은 자신과 연결된 상대 소켓의 상태를 파악해가면서 데이터를 전송한다. 데이터가 제대로 전송되지 않으면 데이터를 재전송하기까지 한다. 따라서 연결지향형 소켓의 데이터 손실은 특별한 경우가 아니면 발생하지 않는다.

##### 2. `SOCK_DGRAM` : 비 연결지향형 소켓

**신뢰성과 순차적 데이터 전송을 보장하지 않는, 고속의 데이터 전송을 목적으로 하는 소켓**

* 전송된 순서에 상관없이 가장 빠른 전송을 지향한다: 연결지향형 소켓에 비해 전송 속도가 빠르다.
* 전송된 데이터는 손실 및 파손의 우려가 있다.
* 한 번에 전송할 수 있는 데이터의 크기가 제한된다.
* 전송되는 데이터의 경계(boundary)가 존재한다: 두 번의 함수 호출로 데이터를 전송하면 데이터를 수신할 때에도 함수를 두 번 호출해야 한다.
* 연결지향형 소켓과 달리 연결이라는 개념이 존재하지 않는다. *다른 chapter에서 설명함*

> 예시) 오토바이 택배 서비스
>
> ![오토바이 배달](/images/2021-05-23/delivery.jpeg)
>
> * 택배는 속도가 생명이므로, 서로 다른 오토바이에 실려서 동일한 목적지를 향하는 두 개의 물건은 출발 순서에 상관 없이 최대한 빨리 목적지를 향하게 된다.
> * 택배는 특성상 손실 및 파손의 우려가 있다.
> * 오토바이에 실어서 보낼 수 있는 물건의 크기에 제한이 있으므로, 많은 양의 물건을 목적지로 보내기 위해서는 두 번 이상에 걸쳐서 물건을 나눠 보내야 한다.
> * 택배 물건 두 개가 각각 별도의 오토바이에 실려서 목적지로 향하면, 물건을 받는 사람도 두 번에 걸쳐서 물건을 수령해야 한다.

#### 프로토콜 최종 선택하기

> 대부분의 경우 세 번째 인자(프로토콜 정보)로 0을 넘겨줘도, 첫 번째 인자(프로토콜 체계)와 두 번째 인자(소켓 타입)만으로도 원하는 유형을 소켓을 생성할 수 있다.

하나의 프로토콜 체계 안에 데이터의 전송 방식이 동일한 프로토콜이 둘 이상 존재하는 경우 때문에 `socket` 함수에 프로토콜 정보도 전달해줘야 한다.

**TCP 소켓 생성하기**

IPv4 인터넷 프로토콜 체계에서 동작하는 연결지향형 데이터 전송 소켓

```c
int tcp_socket=socket(PF_INET, SOCK_STREAM, IPPROTO_TCP);
/*
PF_INET: IPv4인 프로토콜 체계
SOCK_STREAM: 연결지향형 데이터 전송
위 프로토콜 체계와 데이터 전송 방식을 만족하는 프로토콜은 IPPROTO_TCP 하나다.
*/
```

**UDP 소켓 생성하기**

UDP: User Datagram Protocol

IPv4 인터넷 프로토콜 체계에서 동작하는 비 연결지향형 데이터 전송 소켓

```c
int udp_socket=socket(PF_INET, SOCK_DGRAM, IPPROTO_UDP);
/*
PF_INET: IPv4인 프로토콜 체계
SOCK_DGRAM: 비 연결지향형 데이터 전송
위 프로토콜 체계와 데이터 전송 방식을 만족하는 프로토콜은 IPPROTO_UDP 하나다.
*/
```

#### 연결지향형 소켓, TCP 소켓의 예

*UDP 소켓은 별도의 chapter에서 설명*

> Chapter 1의 소스파일이 TCP 소켓 기반의 예제이므로 약간만 변경해서 사용함
>
> * `hello_server.c` -> `tcp_server.c` : 변경사항 없음
> * `hello_client.c` -> `tcp_client.c` : `read` 함수의 호출 방식 변경

**전송되는 데이터의 경계(boundary)가 존재하지 않는다**는 특성을 확인하기 위해, `write` 함수 호출 횟수와 `read` 함수 호출 횟수를 다르게 해서 실행시켜야 한다.

```c
// tcp_server.c

#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <arpa/inet.h>
#include <sys/socket.h>

void error_handling(char *message);

int main(int argc, char *argv[])
{
    int serv_sock;
    int clnt_sock;

    struct sockaddr_in serv_addr;
    struct sockaddr_in clnt_addr;
    socklen_t clnt_addr_size;

    char message[] = "Hello World!";

    if (argc != 2)
    {
        printf("Usage : %s <port>\n", argv[0]);
        exit(1);
    }

    serv_sock = socket(PF_INET, SOCK_STREAM, 0);
    if (serv_sock == -1)
        error_handling("socket() error");

    memset(&serv_addr, 0, sizeof(serv_addr));
    serv_addr.sin_family = AF_INET;
    serv_addr.sin_addr.s_addr = htonl(INADDR_ANY);
    serv_addr.sin_port = htons(atoi(argv[1]));

    if (bind(serv_sock, (struct sockaddr *)&serv_addr, sizeof(serv_addr)) == -1)
        error_handling("bind() error");

    if (listen(serv_sock, 5) == -1)
        error_handling("listen() error");

    clnt_addr_size = sizeof(clnt_addr);
    clnt_sock = accept(serv_sock, (struct sockaddr *)&clnt_addr, &clnt_addr_size);
    if (clnt_sock == -1)
        error_handling("accept() error");

    write(clnt_sock, message, sizeof(message));
    close(clnt_sock);
    close(serv_sock);
    return 0;
}

void error_handling(char *message)
{
    fputs(message, stderr);
    fputc('\n', stderr);
    exit(1);
}
```

```c
// tcp_client.c

#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <arpa/inet.h>
#include <sys/socket.h>

void error_handling(char *message);

int main(int argc, char *argv[])
{
    int sock;
    struct sockaddr_in serv_addr;
    char message[30];
    int str_len = 0, idx = 0, read_len = 0;

    if (argc != 3)
    {
        printf("Usage : %s <IP> <port>\n", argv[0]);
        exit(1);
    }

    sock = socket(PF_INET, SOCK_STREAM, 0);
    if (sock == -1)
        error_handling("socket() error");

    memset(&serv_addr, 0, sizeof(serv_addr));
    serv_addr.sin_family = AF_INET;
    serv_addr.sin_addr.s_addr = inet_addr(argv[1]);
    serv_addr.sin_port = htons(atoi(argv[2]));

    if (connect(sock, (struct sockaddr *)&serv_addr, sizeof(serv_addr)) == -1)
        error_handling("connect() error!");

    while ((read_len = read(sock, &message[idx++], 1))) // read 함수가 호출될 때마다 데이터를 1byte씩 읽어들인다.
    {
        if (read_len == -1)
            error_handling("read() error!");

        str_len += read_len; // while문이 실행되는 동안 read_len은 항상 1이므로 str_len은 읽어들인 바이트 수가 저장된다.
    }

    printf("Message from server: %s \n", message);
    printf("Function read call count: %d \n", str_len);
    close(sock);
    return 0;
}

void error_handling(char *message)
{
    fputs(message, stderr);
    fputc('\n', stderr);
    exit(1);
}
```

실행 결과

```bash
$ gcc tcp_server.c -o tcp_server

$ ./tcp_server 9190

```

```bash
$ gcc tcp_client.c -o tcp_client

$ ./tcp_client 127.0.0.1 9190
Message from server: Hello World!
Function read call count: 13

```

서버가 전송한 13바이트짜리 데이터를 총 13회의 `read` 함수 호출로 읽어들였음을 알 수 있다.