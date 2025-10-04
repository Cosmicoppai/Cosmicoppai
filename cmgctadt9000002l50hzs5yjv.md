---
title: "Traceroute: Not so hard"
seoTitle: "Traceroute: Not so hard"
seoDescription: "Writing traceroute from scratch"
datePublished: Sat Oct 04 2025 21:55:14 GMT+0000 (Coordinated Universal Time)
cuid: cmgctadt9000002l50hzs5yjv
slug: traceroute-not-so-hard
cover: https://cdn.hashnode.com/res/hashnode/image/stock/unsplash/_Cvt8w6Uiso/upload/985ca87ddf052b98485f6cb656536c7a.jpeg
tags: c, networking, ipv4, traceroute

---

I’m sure most of you might have already heard about [traceroute](https://en.wikipedia.org/wiki/Traceroute).

If not, it’s a command line tool generally used to trace the path of the packet over the network.

I recently came across the implementation detail of it, and it’s surprisingly simple to implement and reason about.

In OSI or TCP/IP model, when we create an IP packet to send across the network, we specify the bunch of parameters in headers. One of them is `TTL` in IPV4 and `HOP` in IPV6. By definition `TTL` indicates the time to live in seconds and `HOP` indicates the number of hops. But in practice both of them work as hop counter. Both of them indicates the number of machines that should process the packet before dropping.

```bash
                    IPv4 Header Format
       
    0                   1                   2                   3
    0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
   |Version|  IHL  |Type of Service|          Total Length         |
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
   |         Identification        |Flags|      Fragment Offset    |
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
   |  Time to Live |    Protocol   |         Header Checksum       |
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
   |                       Source Address                          |
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
   |                    Destination Address                        |
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
   |                    Options                    |    Padding    |
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

```bash
                    IPv6 Header Format
   
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
   |Version| Traffic Class |           Flow Label                  |
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
   |         Payload Length        |  Next Header  |   Hop Limit   |
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
   |                                                               |
   +                                                               +
   |                                                               |
   +                         Source Address                        +
   |                                                               |
   +                                                               +
   |                                                               |
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
   |                                                               |
   +                                                               +
   |                                                               |
   +                      Destination Address                      +
   |                                                               |
   +                                                               +
   |                                                               |
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

For example: I transmitted a packet with TTL/HOP of 3, when the next machine will reduce it by one and transmit it to the next machine, if it’s not the final destination. This process will be repeated until destination is reached, or the packet is dropped. Once the packet is dropped by the machine/node, the same machine informs the source about it via the `ICMP` message. `ICMP - Time exceeded` message, more specifically.

*<mark>ICMP messages are used to send control messages and error reporting messages. Thet are not used for any data sending part but used for the feedback mechanism across the devices.</mark>*

The reason why we have this TTL/HOP in the first place is to avoid the infinite roaming of packet. In a misconfigured looped network, a packet will be passed around for eternity, wasting resources.

The traceroute leverages this particular field to get the complete route of the packet over the Internet. It sends the packets continuously starting with TTL `1` up to a max user defined limit. If the packet reaches the destination the returned error will be `ICMP PORT UNREACHABLE`. Any node in between will give the `ICMP Time exceeded` message, effectively sending back it’s IP as per the protocol.

A simplified pseudocode will look like this:

```python
sock = create_socket()
for i in range(1, 30):
    ip_packet = IP_PACKET
    ip.packet.ttl = i
    status = send_packet(sock, remote_server, ip_packet)
    if status:
        err_msg = sock.read_error()
        if err_msg.type == ICMP_TIME_EXCEEDED:
            print(f"{i}: {err_msg.remote_addr.ip}")
        elif err_msg.type == ICMP_UNREACHABLE:
            print("Destination Reached")
            print(f"{i}: {err_msg.remote_addr.ip}")
            return;
```

I have tried to write the whole thing in C, just to make things bit hard for myself as someone who’s not used to writing C daily.

```c
#include <stdio.h>
#include <sys/socket.h>
#include <arpa/inet.h>
#include <sys/types.h>
#include <netdb.h>
#include <string.h>
#include <unistd.h>
#include <netinet/in.h>
#include <linux/errqueue.h>
#include <netinet/ip_icmp.h>


int get_udp_socket() {
	int fd = socket(AF_INET, SOCK_DGRAM, IPPROTO_UDP);
	if (fd < 0) {
		printf("failed to create socket: %d\n", fd);
		return -1;
	}
	return fd;
}

int get_udp_address(char *server_name, char server_port[], struct addrinfo **server_addr_info) {

	struct addrinfo hint = {0};

        hint.ai_family = AF_INET;
        hint.ai_socktype = SOCK_DGRAM;

        int status = getaddrinfo(server_name, server_port, &hint, server_addr_info);

        if (status != 0) {
                printf("getaddrinfo failed: %d\n", status);
                return status;
        }
	return status;

}

int main(int argc, char *argv[]) {
	if ( argc != 2) {
		printf("Need server address\n");
		return 1;
	}

	char *server_name = argv[1];
	// I initially kept this 50000, but the success rate wasn't good enough,
	// so I kept it same as traceroute default port, looks like more fw allows traffic on it
	char server_port[] = "33434";
	struct addrinfo *server_addr_info;

	int status = get_udp_address(server_name, server_port, &server_addr_info);

	if ( status != 0) {
		return 1;
	};


	int socket_fd = get_udp_socket();

	char recv_buffer[1024];
	const char *msg = "辿";

	// Each packet size will be: ipv4 header (20 bytes) + udp header(8 bytes) + msg (3 bytes) = Total(31 bytes)

	fd_set readfds;
	struct timeval tv;

	// TODO: allow dynamic size packets later
	printf("%s を辿る, 32 byte packets\n", server_name);

	for (int hop = 1; hop < 31; hop++) { // staring from 1 as ttl 0 not valid
		int status_code = setsockopt(socket_fd, IPPROTO_IP, IP_TTL, &hop, sizeof(hop));
		if ( status_code !=0) {
			 perror("setsockopt");
			return 1;
		}

		int on = 1;
		status_code = setsockopt(socket_fd, IPPROTO_IP, IP_RECVERR, &on, sizeof(on));
		if ( status_code !=0) {
			perror("setsockopt");
			return 1;
		}

		FD_ZERO(&readfds);
		FD_SET(socket_fd, &readfds);

		tv.tv_sec = 2;
		tv.tv_usec = 0;

		sendto(socket_fd, msg, strlen(msg), 0, server_addr_info->ai_addr, server_addr_info->ai_addrlen);

		int ret = select(socket_fd+1, &readfds, NULL, NULL, &tv);

		if ( ret > 0 && FD_ISSET(socket_fd, &readfds) ) {
			
			struct msghdr resp_msg = {0};
			struct iovec iov;
			struct sockaddr_in sender_addr; // for remote addr
			
			iov.iov_base = recv_buffer;  // to store the data
			iov.iov_len = sizeof(recv_buffer) - 1;

			resp_msg.msg_name = &sender_addr;  // to store the ip and other info of the sender
			resp_msg.msg_namelen = sizeof(sender_addr);
			resp_msg.msg_iov = &iov;
			resp_msg.msg_iovlen = 1;

			char control_buffer[512];  // to store the errors
			resp_msg.msg_control = control_buffer;
			resp_msg.msg_controllen = sizeof(control_buffer);

			ssize_t bytes_read = recvmsg(socket_fd, &resp_msg, MSG_ERRQUEUE);
			if (bytes_read < 0) {
				perror("recvmsg");
			}
			struct cmsghdr *cmsg;
				for (cmsg = CMSG_FIRSTHDR(&resp_msg); cmsg != NULL; cmsg = CMSG_NXTHDR(&resp_msg, cmsg)) {

					// checking message level is ipv4 and type is error
					if ( cmsg->cmsg_level == SOL_IP && cmsg->cmsg_type == IP_RECVERR) {
						struct sock_extended_err *err_msg = (struct sock_extended_err *)CMSG_DATA(cmsg);
						if ( err_msg->ee_origin == SO_EE_ORIGIN_ICMP ) {
							struct sockaddr_in *sin = (struct sockaddr_in *)(err_msg+1);
							char ip_str[INET_ADDRSTRLEN];
							inet_ntop(AF_INET, &sin->sin_addr, ip_str, sizeof(ip_str));
							if ( err_msg->ee_type == ICMP_TIME_EXCEEDED) {
								printf("%d: %s\n", hop, ip_str);
							} else if ( err_msg->ee_type == ICMP_UNREACH) {
								printf("%d: Dest reached: %s\n", hop, ip_str);
								return 0;
							}
						}
					}
				}
		} else {
			printf("%d: * * *\n", hop);
		}
	}
	close(socket_fd);
	freeaddrinfo(server_addr_info);
}
```

What I’m doing is, creating a socket and changing the TTL on each iteration and allowing the control messages on the socket explicitly, as only responses are buffered in the socket by the kernel, the error messages are discarded by default. I’m also setting a read timeout for the sanity.

Some pitfalls that I encountered:

* Usage of ports other than `33434` has very less success rate, my guess is that firewalls on the internet accustomed to it because of the historical reason of traceroute, idk.
    
* `select()` notifying when I set `read_fd` not `error_fd`.
    
* The signature of `recvmsg`, `msghdr` and `cmsghdr`, aren’t very intuitive. I tried to understand the reasoning behind its design but wasn’t able to find any solid reading material.
    

The above code is also available at my GitHub: [Cosmicoppai/tadoru: Traceroute clone built for fun](https://github.com/cosmicoppai/tadoru)