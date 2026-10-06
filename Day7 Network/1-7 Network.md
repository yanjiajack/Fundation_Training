# Linux Socket

In Linux, a socket is a communication endpoint. It enables two processes to exchange data—whether they are running on the same machine or across a network.

- **Application Level**: You can think of a socket as a virtual connection tunnel between two points (for example, `1.1.1.1:100 - 2.2.2.2:200`). A process writes data into one end, and another process reads it from the other end.
    
- **Kernel Level**: Linux follows the philosophy that "Everything is a file." Inside the system, a socket is treated as a special type of file, represented by a **File Descriptor (FD)**.
    

## Socket as a File

To see how Linux treats a socket as a file descriptor, we can write a minimal C program that creates a socket and pauses execution:

```C
#include <stdio.h> 
#include <sys/socket.h> 
#include <netinet/in.h> 

int main() { 
	int fd = socket(AF_INET, SOCK_STREAM, 0); 
	printf("fd: %d\n", fd); 
	while(1); 
	return 0; 
}
```

Compile and run the program in the background:

```shell
$ gcc 1.c 
$ ./a.out & 
[1] 46396 
fd: 3
```

Because `0`, `1`, and `2` are reserved for standard input, output, and error, the system assigns `FD 3` to our new socket.

We can inspect the process's open files under `/proc`:

```Shell
$ ll /proc/46396/fd 
total 0 
lrwx------. 1 dadmin dadmin 64 Sep 23 10:39 0 -> /dev/pts/1 
lrwx------. 1 dadmin dadmin 64 Sep 23 10:39 1 -> /dev/pts/1 
lrwx------. 1 dadmin dadmin 64 Sep 23 10:39 2 -> /dev/pts/1 
lrwx------. 1 dadmin dadmin 64 Sep 23 10:39 3 -> 'socket:[7371078]'
```

Notice that `FD 3` points to `socket:[7371078]`, where `7371078` is the internal inode number. Using `lsof`, we can confirm that Linux identifies it as a TCP socket:

```Shell
$  lsof | grep 7371078
COMMAND     PID  TID TASKCMD             USER   FD      TYPE             DEVICE  SIZE/OFF       NODE NAME
a.out     46396                        dadmin    3u     sock                0,9       0t0    7371078 protocol: TCP
```

## Client-Server Communication

To see sockets in action, we build a typical Client-Server model.

### Key System Calls

To achieve it, we need to understand a few Linux system calls

- `socket()`: creates an endpoint for communication and returns a file descriptor that refers to that endpoint.
    
- `bind()`: used to assign the address to the socket.
    
- `listen()`: marks the socket as a passive socket
    
- `accept()`: extracts the first connection request on the queue of pending connections for the listening socket, create a new connected socket, and return a new file descriptor referring to that socket.
    
- `recv()`: used to receive messages from a socket, same as `read()`
    
- `send()`: used to send messages to a socket, same as `write()`
    
- `connect()`: connects the socket to the address.
    

### Server Code Example

The server creates a socket, binds it to port `8888`, listens for connections, reads a message from a client, sends a reply, and closes the connection.

```C
#include <stdio.h>
#include <string.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <unistd.h>

int main() {
    // 1. Create a listening socket
    int listen_fd = socket(AF_INET, SOCK_STREAM, 0);

    // 2. Bind to 127.0.0.1:8888
    struct sockaddr_in addr;
    memset(&addr, 0, sizeof(addr));
    addr.sin_family = AF_INET;
    addr.sin_port = htons(8888);
    addr.sin_addr.s_addr = htonl(INADDR_ANY);

    bind(listen_fd, (struct sockaddr *)&addr, sizeof(addr));

    // 3. Start listening
    listen(listen_fd, 5);
    printf("Server started. Waiting for connections...\n");

    // 4. Accept a client connection
    int conn_fd = accept(listen_fd, NULL, NULL);
    printf("Client connected!\n");

    // 5. Receive message from client
    char buf[128] = {0};
    recv(conn_fd, buf, sizeof(buf) - 1, 0);
    printf("Received from client: %s\n", buf);

    // 6. Send a reply back to the client
    char *reply = "Hello Client, message received!";
    send(conn_fd, reply, strlen(reply), 0);
    printf("Reply sent to client.\n");

    // 7. Close sockets
    close(conn_fd);
    close(listen_fd);
    return 0;
}
```

### Client Code Example

The client creates a socket, connects to `127.0.0.1:8888`, sends a greeting, prints the server's reply, and exits.

```C
#include <stdio.h>
#include <string.h>
#include <sys/socket.h>
#include <arpa/inet.h>
#include <unistd.h>

int main() {
    // 1. Create socket
    int fd = socket(AF_INET, SOCK_STREAM, 0);

    // 2. Configure server address
    struct sockaddr_in server_addr;
    memset(&server_addr, 0, sizeof(server_addr));
    server_addr.sin_family = AF_INET;
    server_addr.sin_port = htons(8888);
    inet_pton(AF_INET, "127.0.0.1", &server_addr.sin_addr);

    // 3. Connect to the server
    connect(fd, (struct sockaddr *)&server_addr, sizeof(server_addr));
    printf("Connected to server!\n");

    // 4. Send message to server
    char *msg = "Hello Server!";
    send(fd, msg, strlen(msg), 0);
    printf("Sent to server: %s\n", msg);

    // 5. Receive reply from server
    char buf[128] = {0};
    recv(fd, buf, sizeof(buf) - 1, 0);
    printf("Received from server: %s\n", buf);

    // 6. Close socket
    close(fd);
    return 0;
}
```

### Running the Test

Compile both programs:

```
$ gcc server.c -o server
$ gcc client.c -o client
```

Start the server in Terminal 1:

```
$ ./server
Server started. Waiting for connections...
```

Run the client in Terminal 2:

```
$ ./client
Connected to server!
Sent to server: Hello Server!
Received from server: Hello Client, message received!
```

Output in Terminal 1 updates:

```
$ ./server
Server started. Waiting for connections...
Client connected!
Received from client: Hello Server!
Reply sent to client.
```

To explore more, we can use the built-in system tools, such as `strace`, `tshark`/`tcpdump` or `ss`/`netstat`

## I/O Multiplexing with `epoll`

Our previous server program had a major limitation: it could only process **one connection at a time**. If a client connected and stayed idle, the server remained blocked at `recv()`, unable to serve anyone else.

To handle thousands of concurrent client connections efficiently, Linux provides **I/O Multiplexing**. Instead of using **multi-threading** or **multi-processing** (which consumes significant CPU memory), we can use `epoll`.

`epoll` is an event notification mechanism in Linux. It allows a single process to monitor multiple file descriptors to see if any of them are ready for I/O operations (such as reading data or accepting a new connection).

### Key System Calls

- `epoll_create1()`: Creates an `epoll` instance in the kernel and returns a FD referencing it.
    
- `epoll_ctl()`: Controls the interest list of an `epoll` instance. It adds (`EPOLL_CTL_ADD`), modifies (`EPOLL_CTL_MOD`), or removes (`EPOLL_CTL_DEL`) target FDs and their associated events.
    
- `epoll_wait()`: Blocks and waits for registered events to occur on any monitored file descriptors.
    

### Non-blocking Multi-Client Server

Here is how we modify the server to handle multiple clients concurrently using an `epoll` event loop:

- **Register Listening Socket**: We register `listen_fd` into `epoll`. When a client initiates a connection, `epoll_wait()` unblocks and signals that `listen_fd` has data to read (an incoming connection request).
    
- **Accept & Add Client**: When `listen_fd` is triggered, we execute `accept()` to get `conn_fd`, then register `conn_fd` into the same epoll instance.
    
- **Handle Data**: When a client sends a message, `epoll_wait()` unblocks again, this time identifying `conn_fd`. The server reads the data, replies, and closes the connection.
    

```C
#include <stdio.h>
#include <string.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <sys/epoll.h>
#include <unistd.h>

int main() {
    // 1. Setup socket
    int listen_fd = socket(AF_INET, SOCK_STREAM, 0);

    struct sockaddr_in addr = {0};
    addr.sin_family = AF_INET;
    addr.sin_port = htons(8888);
    addr.sin_addr.s_addr = htonl(INADDR_ANY);

    bind(listen_fd, (struct sockaddr *)&addr, sizeof(addr));
    listen(listen_fd, 5);
    printf("Server started on 8888...\n");

    // 2. Setup epoll
    int epoll_fd = epoll_create1(0);
    struct epoll_event ev, events[10];

    ev.events = EPOLLIN;
    ev.data.fd = listen_fd;
    epoll_ctl(epoll_fd, EPOLL_CTL_ADD, listen_fd, &ev);

    // 3. Event Loop
    while (1) {
        int nfds = epoll_wait(epoll_fd, events, 10, -1);

        for (int i = 0; i < nfds; i++) {
            if (events[i].data.fd == listen_fd) {
                // New connection
                int conn_fd = accept(listen_fd, NULL, NULL);
                printf("Client connected!\n");

                ev.events = EPOLLIN;
                ev.data.fd = conn_fd;
                epoll_ctl(epoll_fd, EPOLL_CTL_ADD, conn_fd, &ev);
            } else {
                // Client sent data
                int conn_fd = events[i].data.fd;
                char buf[128] = {0};

                recv(conn_fd, buf, sizeof(buf) - 1, 0);
                printf("Received: %s\n", buf);

                char *reply = "Hello Client, message received via Epoll!";
                send(conn_fd, reply, strlen(reply), 0);
                printf("Reply sent.\n");

                epoll_ctl(epoll_fd, EPOLL_CTL_DEL, conn_fd, NULL);
                close(conn_fd);
            }
        }
    }

    close(listen_fd);
    close(epoll_fd);
    return 0;
}
```

## How the Linux Kernel Receives a Network Packet

### 1. Packet Ingestion by the NIC

The NIC receives an incoming network packet and writes it directly into host memory via **DMA (Direct Memory Access)** into a pre-allocated Ring Buffer (RX Queue), requiring no CPU intervention.

Once the packet is stored in memory, the NIC generates a **Hardware Interrupt (HardIRQ)** to inform the CPU.

### 2. Interrupt Handling & NAPI Scheduling

The CPU executes the **Interrupt Service Routine (ISR)** triggered by the hardware interrupt, which disables further hardware interrupts from that NIC to prevent an "interrupt storm."

The ISR marks the network receive soft interrupt (NET_RX_SOFTIRQ) as pending and returns immediately to keep interrupt latency low.

The Linux kernel handles the **Soft Interrupt (SoftIRQ)** using NAPI (New API, a hybrid interrupt-and-polling mechanism) executing the polling routine either directly on the current CPU or via the `ksoftirqd` kernel thread under heavy load.

Once all pending packets are processed and the Ring Buffer is empty, NAPI re-enables hardware interrupts for the NIC to listen for new incoming packets.

### 3. Packet Processing in the Kernel Network Stack

NAPI polls packets directly from the Ring Buffer and wraps each packet into a `sk_buff` (Socket Buffer) structure.

The kernel calls `napi_gro_receive()` (for Generic Receive Offload optimization) or `netif_receive_skb()` to pass the `sk_buff` up the protocol stack.

The packet travels through the protocol layers (Link Layer > Network Layer/IP > Transport Layer/TCP/UDP). Each layer strips its respective header by simply advancing internal data pointers inside the `sk_buff` without copying the data.

### 4. Delivering to the Socket Receive Queue

The Transport Layer uses the source/destination IPs and ports to locate the corresponding struct `sock` (Socket instance).

The kernel places the `sk_buff` directly into the socket's receive queue (`sk_receive_queue`).

If the owning process is blocked in a system call (like `epoll_wait()`, `read()`, or `recv()`), the kernel wakes up the process to copy the data into user-space memory.

## Practice

1. Inspect open sockets using `/proc/<pid>/fd` and `lsof` to understand how Linux treats sockets as files.
    
2. Write and understand the **client-server** example
    
3. Write and understand the **epoll** example
    
4. Gain a basic understanding of a packet's path from `NIC > HardIRQ > SoftIRQ > Network Subsystem > Process`
    
5. Summarize your learnings into technical notes, and push both the code and notes to GitHub.