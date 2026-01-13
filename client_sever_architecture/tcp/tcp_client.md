**Note: The lines of code in this documentation is not mine, it has been reproduced following the book "Hands On Network Programming in C" by Lewis Van Wrikle which you can fork/copy from [https://github.com/codeplea/Hands-On-Network-Programming-with-C](Hand_On_Network_Programming_with_C).Here would be a simple explanation, on what I learned following this book. Enjoy :)**


## What is TCP?

I think that the best way to start this blog, would be to firstly answer what is TCP, and why is it important for us to learn it?
Based on CISCO PRESS TCP can be determined as the following: `TCP (Transmission Control Protocol) is a connection oriented protocol, which is assigned to the Transportation Layer in both OSI model and the TCP/IP model, used to transmitt, data frames (which here are known as packets) from one point of communication to another`

Pretty straightforward definition if you ask me. So it is a transportation layer protocol, and we will be using this protocol to transport our packets. How we will be doing this, well through creating a `tcp_client` and a `tcp_server` duh, so let us get started with it. 


## Geeting our Hands Dirty 

The whole idea of the project is to give us a better understanding on how tcp client works, so not really anything fancy, just a simple functional tcp client. This tcp client will take in a hostname (or IP address) and a service (or a port number) as arguments from the command line. It will attempt a connection to the TCP server at that address. If successul , it will relay data that's recieved from the server to the terminal and data inputted into the terminal to the server. It will continue until either it is terminated  ( with using the famous CTRL+C) or the server closes the connection. 

The flow that our tcp_client is going to follow is displayed in figure 1. 

[images/figure_1](figure_1)

*Note: This diagram was reproduced by Lewis Van Winkel book Hands On Network Programming with C, all credits to the original author*

Pretty standard and simple to understand if you ask me. In short flow it would be something like the following

```
Getting Remote address-> create a socket-> connect to the server-> check if it has input from the standard input (the termina) 
                                                                                    -> (yes) we get the input using and send it to the server
                                                                                       (no) we check if there is a input in the socket
                                                                                            ->(yes) we recieve it
                                                                                            ->(no) we check if there is input from the standard input. -> Chcking if the socket was closed (meaning the communication was closed) -> (yes) we close it
                                                                                 -> (no ) we print out the communication, and check if there is connection. 
```
Pretty standard stuff. The intersting thing here is that this tcp_client is meant to be **cross-platform** which is a fancy way would be sharing of data and resources between devices of different operating system, that means also windows :|, like Linux wasn't enough it would also needed tobe Window (Since MacOS is basically unix-based, anything that will be defined for linux, should also work on MacOS, based on a simple rule of three). 

EVerything that is needed in this project is defined in chap03.h header file. (I decided to not really change anything about the naming of the files, since I am lazy and there is no point in doing so, but you are free to do whatever you want). 


```chap03.h

#if defined(_WIN32)
#ifndef _WIN32_WINNT
#define _WIN32_WINNT 0x0600
#endif
#include <winsock2.h>
#include <ws2tcpip.h>
#pragma comment(lib, "ws2_32.lib")

#else
#include <sys/types.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>
#include <netdb.h>
#include <unistd.h>
#include <errno.h>

#endif


#if defined(_WIN32)
#define ISVALIDSOCKET(s) ((s) != INVALID_SOCKET)
#define CLOSESOCKET(s) closesocket(s)
#define GETSOCKETERRNO() (WSAGetLastError())

#else
#define ISVALIDSOCKET(s) ((s) >= 0)
#define CLOSESOCKET(s) close(s)
#define SOCKET int
#define GETSOCKETERRNO() (errno)
#endif

#include <stdio.h>
#include <string.h>

```
Quite networking standard if I am allowed to say so. The author also let us know that we will be needed the conio.h header file, which is required for the _kbhit()_ fuction which help us, by indicating weather there is input from the terminal line, or not. In Linux we won't be having this problem, since we will be using `select()` to monitor if we either have input from the socket or from the stdin , but you know Windows, and theirfamous AI bubble. 

Now we are ready to really get our hands quite dirty. We can start with the implementation of our `tcp client`. Ready for the cross-platform architecture, cool. be ready. 

We include chap03.h file and also <conio.h> file if we are in windows, also if we are operating in windows we would start with creating a window socket `WSADATA d;`, which if it failed to initialized, we would return one.

Earlier we said that we would like to write our program, in such a way that the hostname/IP_address and the port/service would be passed as command line argouments, which is why we would be need at least 3 argument passed from the command line (3 because argv[0] is the name of the file, argv[1] is the first line-argument, argv[2] is the second line-argument and so on. Argc is the variable where the number of command line arguments is stored, and we access them usign argv, which is an array of pointers to characters (strings).

*Look at the signature of main if you are confused: int main(int argc, char* argv[])*

It need to at least 3, if not, then the program your print a simple usage, on the order of information and return 1

```
 if (argc < 3) {
        fprintf(stderr, "usage: tcp_client hostname port\n");
        return 1;
    }

```
The whole puropse of the tcp_client is to be connected to a tcp server that is why a remote address shoul be establish. So we get to the first block on our flow chart, getaddrinfo(). 
To be fair the number of times I have seen this function, while trying to learn networking would be quite overwhelming, and all that for the right reasons. Following my first 'bible' definition, that being man (the second one beign : Ghost in the wired by kevin Mitnic and the third being the most well known books among wannabes-> Hacking: The art of Explotation, if you want to be a wannabe and you don't have this, i strongly suggest you to take it).
Back to the defenition of `getaddrinfo()` based on man: **Given node (host/IP_address) and service/port_number, which identify an Internet host and service, getaddrinfo() returns one or more addrinfo structures, each of which contains an Internet address that can be specified in a call to bind() or connect()**

Huh, what is a addrinfo structure? Well a very important structure in networking. addrinfo struct contains important information related to internet addresses. It would look something as the following.

```
struct addrinfo {
               int              ai_flags;
               int              ai_family;
               int              ai_socktype;
               int              ai_protocol;
               socklen_t        ai_addrlen;
               struct sockaddr *ai_addr;
               char            *ai_canonname;
               struct addrinfo *ai_next;
           };
```
Yes important information related to IP addresses. After getaddrinfo get the different remote addresses, we can choose one on our call to connect. Not bind(), we use bind() to bind local connections, while we use connect to connect to remote connections.*Do not worry, it is knida confusing but anything will be more than fine, as we go on everything will make sense.*

We create place in the memory for our structure using memset, and fill it up with zeros. Then we set the ai_socktype to be SOCK_STREAM, this is needed because in the next line we finally make use of getaddrinfo, and if you read this sectio on man documentation, it would have the following:

```
The hints argument points to an addringo structure that specifies criteria for selecting the socket address structures returned in the list pointed by res. If hints is not NULL it points to an addrinfo structure whose ai_family, ai_socktype, and ai_protocol specify criteria that limit the set of socket addresses returned by getaddrinfo().

```
Emphasise our work on specifies criteria for selecting the socket address structure, the criteria we set was on ai_socktype = SOCK_STREAM, meaning that we want those connections which uses TCP, rather than another transportation protocol such as UDP (User Datagram Protocol), which we will not be seeing. If we would let the ai_socktype not set, it would guide getaddrinfo() to displaying all conncection of all tcp and udp.

We also create a pointer to a struct address, that is because, also as mentioned, the socket address structures it will be return on a linked-list pointed by res, res in this case is the peer_address; Because the struct addrifo has a pointer to a struct addrinfo, named ai_next;

The function signature of the getaddrinfo is as follows 

```
int getaddrinfo(const char *restrict node,
                const char *restrict service,
                const struct addrinfo *restrict hints,
                struct addrinfo **restrict res);

```
It requires a node (either a host_name or ip_address), a service (a port number), an addrinfo structure and a pointer to an addrinfo structure, in short words 

`getaddrinfo(argv[1],argv[2],&hints,&peer_address))`. As a return value, getaddrinfo on success returns 0, so if anything rather than 0, we have failed to get a remmot connection, so we print the error. 

In the case that things go well, we get our address and service stored on the peer_address. getaddrinfo() is quite flexible about how it takes the input. The hostname could be a domain name like example.com or an IP address such as 192.168.17.23 or ::1. The port can be a number, such as 80, or a protocol such as http, it doesn't quite matter. 

After the getaddrinfo get the remote connection, we use getnameinfo() to convert it in a string, and print it, mostly for debugging reasons and it is actually a good debugging practice to print things on the terminal. According to man: getnameinfo() is a function which does address-to-name translation in a protcol independent manner.It is the inverse of getaddrinfo. The function signature of getnameinfo() is as follows

```
int getnameinfo(const struct sockaddr *restrict addr, socklen_t addrlen,
                       char host[_Nullable restrict .hostlen],
                       socklen_t hostlen,
                       char serv[_Nullable restrict .servlen],
                       socklen_t servlen,
                       int flags);
```
So we got the remote address, now we need to create a socket. From a simple point of view, a socket is what it is known as a point in communication and it is nothing more than just the ip_address:port -> example: example.com:80 is a socket, or 127.1.1:80 is also a socket. It is defined in the <sys/socket.h> header file, to create it we use the socket function which we store in the socket_peer variable of the socket type. 

The signature of the socket function is: int socket(int domain, int type, int protocol). For the domain, since we didn't specify nothing it wouldaccept both ipv4 and ipv6 ip addresses. THe socket type willl be of TCP only, the protocol, would be any: in the sense on what we have passed on the command line. 

Checking if the socket is valid, because example.com:-1 doesnt really tend to exist, unless one of you broke the internet, like Dan Kamisky almost did in 2008 with the DNS flaw. If not then we print the error, and return 1, indicating some errors. If no problem, then we move on to the nextstep of the flow, connection, which we do through the conenct function, which `int connect(int sockfd, const struct sockaddr *addr, socklen_t addrlen)` takes on a socketfd, the one that is stored in the socket_peer, the addr and the addrlength. 
(*In Unix and Unix-like computer operating systems, a file descriptor (FD, less frequently fildes) is a process-unique identifier (handle) for a file or other input/output resource, such as a pipe or network socket- WIkipedia. This descriptor tells the OS that something is open and need resources to use. Typically they are non-negative*)

On success connect return 0, meaning everything went off well.If not we print the error, if yes we free peer_address, sing the freeaddrinfo (like malloc and free), and print out connected.
Our program should now loop while checking both the terminal and socket for new data. If new data comes from the terminal, we send it over te server using send(). If the new data is comming over from the socket we will print in the terminal. Usually to recive data we will be using recv, but here is not possible due to the fact that recv block the connection if no new data is being send over the socket. Instead we use select(). 

Inside the while loop which will always be running, since 1 is always true.(*I find it actually quite cool that while(1) has been fully accepted from the community as i is a variable for the foor loop.*

We define an fd_set reads, which is going to be the set when we are to set our file descriptors (the socket and the stdin). In C the fd_set it is a structure, which is used in synchronus I/O multiplexing to monitor multiple file descriptors, waiting until one or more of them become "ready"
A good rule of thumb is that after the creating of the fd_set, is to clean it with FD_ZERO, since it can have some garbage and give us untrustworthy results. We set both file descriptors, by using FD_SET (*THe file descriptor for stdin is 0*). 

Now for windows, as we said earlier is a problem with the command line input, select can not monitior the command line input in windows, For thisreason, we set up a timeour to the select() call for 100miliseconds, If there is no activity after 100miliseconds , select() returns, and we can check for terminal input manually. More work, but it is needed to mainain cross platform. On success select return the number of file secriptors contained in the three returned descriptor sets (that is, the total number of bits that are in readfs (read in our case), writefds, exceptfds).Select can return 0 if the timeout expired before any file descriptor become ready. 

If monitoring is okay, we check if the socket file descriptor is in our read fd_set. If yes we read the data, using recieve (it is important to kow, that we should be able to read > 0 bytes of data). We print the received bytes of data in the terminal, and since recv() is not null terminated ('\0') and hence we do not know where a stream of characters begins and where it ends, for this reason we will be using this quite wired looking format specifier `%.*s`, which prints a string of defined length.

The line terminal in windows, would be made using _kbhit()_ function, to refering on windows documentation: Checks the console for keyboard inputand return a non-zero value, if the key have been pressed, otherwise return 0.

For Unix-based operating systems, we use fgets to read the command-line input. If this input is ready, we send it over to the server using send.
If a key interrupt such as Ctrl+C or connection is closed, than we quite the while loop and print closing socket... and finished. On windows we do not forget to WSACleanup the socket. 




## Running our program

IN unix-based operating systems compile the programm using gcc (or whatever compiler you like) `gcc tcp_client.c -o tcp_client`. ON whindows you would use MINGw as a environment development, and also link it with w2_32.`gcc tcp_client.c -o tcp_client.exe -lws2_32`

Run it : ./tcp_client node service (example-> ./tcp_client example.com 80), in Unix-based. 
         tcp_client.exe node service 

Enjoy :)



## Conclusion

This project it was a fun way to learn how the we can send data over a server using tcp. Still it has a lot to do, (three-hand shaking is also to be coverd, but for now it is cool. 

Thank you so much for reading. Hope you enjoyed :) 
















              
