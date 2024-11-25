+++
title = "File Servers"
description = "my experience with TFTP and FTP"
date = "2024-11-25"
+++

## Introduction

I've spent a week trying to implement a TFTP server and an FTP server in Rust. I did it for fun and to learn, so I'll admit I wasn't behaving like a good developer and didn't write tests. It was the first time I tried to do something without relying heavily on libaries (except for Tokio, as developing a runtime is really hard).

To do this, I went to [RFC Editor](https://rfc-editor.org), used the keyword search, and found the RFCs with Internet Standard status. For TFTP, it's RFC 1350 and for FTP it's RFC 959.

## TFTP

RFC 1350 is Revision 2 of TFTP, dated July 1992. RFC 1350 obsoletes RFC 783, which is also called Revision 2, as it obsoletes [IEN (Internet Experiment Notes) 133](https://www.potaroo.net/ietf/html/ienindex.html). It's a simple protocol built on top of UDP (commonly uses port 69) that transfers packets of data in a lockstep fashion. TFTP is mainly used by devices to boot from a network, particular Local Area Networks (LANs).

More or less, the client sends a read request (RRQ), the server sends 512-byte blocks of data (DATA), the client sends an acknowledgement of that block (ACK), then this continues until the last data packet is sent. The last data packet will be less than 512 bytes, which signals the end of the transfer.

It's also possible for a client to send a write request (WRQ), but this is considered unwise to support as the specification does not include any user authentication or security mechanisms (IEN 133 was published in January 1980, the Internet wasn't a large global public network yet).

There are five types of packets, which are read request (RRQ), write request (WRQ), data (DATA), acknowledgement (ACK), and error (ERROR), which have the 2 byte opcodes 1, 2, 3, 4, and 5 respectively.

Let's assume the perspective of a client that is communicating with a server that only serves files. We first need to send a RRQ packet over the network (don't forget about big-endianness). The RRQ packet consists of a 2 byte opcode (which takes the value of 1 in this case) followed by the filename as a null-terminated string, followed by the mode as a null-terminated string. 

In Rust, you can use the `CString` or `CStr` types in combination with the `as_bytes_with_nul` or `to_bytes_with_nul` methods from the `std::ffi` module to handle the strings. Or you could push the null byte to a String, whatever works for you. 

The mode can take the values "netascii", "octet", or "mail". Octet, or binary mode, is commonly used. Strictly speaking, octet mode means the server transfers file in it's own 8-bit format, but it would have to account for the client not using the same 8-bit format. This is to account for hardware, and the spec mentions the DEC-20, a 36-bit machine, which is no longer being produced, so let's not worry about it.

Once the server receives the RRQ, parses it, and verifies it has the file that the client wants, it sends DATA packets. DATA packets consist of the 2 byte opcode, the 2 byte block number, and n bytes of data (generally from 0 to 512, though there are extensions not mentioned here for larger window sizes). The block number starts at 1 and will increment for each successive block of data. Upon receiving the DATA packet, the client will send an ACK packet, which consists of the 2 byte opcode and the matching 2 byte block number. If the packet gets lost, then the client may timeout and re-transmit its last ACK or DATA packet, which indicates to the server that it should re-transmit the DATA packet.

In the happy path, the client and server exchange DATA and ACK packets until the last DATA packet under 512 bytes is sent. After the client sends the final ACK, it may cease communication, though the spec encourages the client to wait in case in needs to re-transmit the final ACK (which will happen if the server sends the final DATA packet again). There are still a few couple edge cases, which you can read more about in Section 6, Normal Termination.

It's possible that before or during the data transfer, an error occurs. For DATA packets sent by the server, the client may respond with an ERROR packet and for ACK packets send by the client, the server may respond with an ERROR packet. Of course, when the client sends the initial RRQ and the server doesn't have that file, it should also respond with an ERROR packet. The ERROR packet consists of the 2 byte opcode, 2 byte error code, and the null-terminated error message string. There are 8 error codes to choose from, such as "Not defined, see error message (if any)" and "File not found".

I want to emphasize that even though this is considered a simple protocol, there are more edge cases and intricacies that I haven't covered in this wall of text or in my own implementation, which would be too embarrassing to share. This includes handling different modes, re-transmissions, and robust error handling.

What I will say is that I was able to write a program with light error handling to load a couple of files into memory, listen for UDP datagrams, and send blocks of data. For re-transmissions, I would simply send the block of data that comes after whatever block number the client sent. I made a mistake by forgetting to null-terminate the error message in the ERROR packet which resulting in strange issues and confused me for a while; when I eventually fixed it, I swore to remember to use CString/CStr. And because UDP is the way it is, a little bit of work had to be done to save the client state (remembering what file it requested).

That's about it. Though this isn't the most exciting protocol out there, I think it's a gentle introduction to reading an RFC, figuring out how to construct packets, sending the packets over the network, and thinking about what could go wrong and dealing with those situations.

## FTP

I'll be honest, I was in way over my head and wasn't able to implement an FTP server, but I did try. I also read a good chunk of the RFC, so I'll just write about what I read. RFC 959 is 69 pages long, so it's quite complex.

Not only does FTP use TCP, but the client-server (or server-server when controlled by a client) communication requires 2 connections. These connections are known as the control and data connections. The server must use port 21 for the control connection and port 20 for the data connection. So let's talk about them.

To get some terminology out of the way, both the client and server have a Protocol Interpreter (PI) and DTP (Data Transfer Process). The user PI initiates a connection with the server PI for the control connection, and the server DTP initiates a connection with the user DTP for the data connection. Now the server DTP initiating the connection with the client DTP is called active mode and is the default behavior. However, the client can send the PASV command to get the address the server is *passively* listening on; the client DTP then connects to the server DTP instead in what is called passive mode.

With that out of the way, the FTP server control connection receives commands from the client, which are Telnet strings. Basically they are ASCII strings that are terminated with CRLF (Carriage Return, Line Feed). Note that there are significantly more commands listed in the spec and its extensions, but in the minimum implementation specified in section 5.1, the server should support the following commands:
  - USER
  - QUIT
  - PORT
  - TYPE
  - MODE
  - STRU
  - RETR
  - STOR
  - NOOP

If you're wondering if the USER command authenticates a user, then the answer is maybe. While it technically conforms to the spec, some servers require the PASS command to enter a password after the USER command is accepted. But even if the PASS command is supported and used, there is no encryption. Sorry, you'll have to look at RFC 2228 for security extensions.

Let's go into a little more detail now. Commands are categorized as access control commands, transfer parameter commands, and FTP service commands. The USER and QUIT commands are access control commands, the PORT, TYPE, MODE, and STRU commands are transfer parameter commands, and RETR, STOR, and NOOP are FTP service commands. Each command may have required and optional arguments.

USER sets the user for the "session". I use the word session because the client PI can actually send the USER command again, which resets current user information and begins the login sequence again. This has the side effect of using the previous user information for file transfer that are already in-progress. However, beginning the login sequence again does not affect the transfer parameters.

QUIT terminates a user and allows the server to close the control connection if there is no file transfer in progress. If there is, then the control connection remains open for the server to send a response before closing it.

PORT allows the client to specify where the server DTP should send data (remember, the default is active mode so the server connects to the client). The command requires a comma-delimited string of numbers in the form h1,h2,h3,h4,p1,p2 where h1,h2,h3,h4 is the 32-bit internet host address (h1 being the high order 8 bits), and p1,p2 is the 16-bit TCP port address. The specification states that it's not required to use this command because there are defaults for both the user and server data ports. In practice, firewalls may block ports from being opened and since routers use Network Address Translation (NAT), it's more practical to set the server in passive mode by issuing the PASV command and having the client initiate the data connection.

TYPE allows the client to set the representation type - A for ASCII, E for EBCDIC, I for Image, and L for local byte. ASCII and EBCDIC take optional second parameters N for Non-print, T for Telnet format effectors, and C for Carriage Control (ASA). Local byte takes a second parameter for byte size. Though there's many options here, ASCII Non-print is most commonly used and is the default type.

MODE allows the client to set the data transfer mode - S for Stream, B for Block, and C for Compressed. Stream is the default.

STRU allows the client to set the file structure - F for File (no record structure), R (Record structure), and P (Page structure). The minimum implementations requires support for files and records. While EOF is used to mark the end of files, EOR is used to mark the end of records in the record-structure, where records comprise a file.

RETR allows the client to specify a pathname for a file to retrieve from the server. It causes the server DTP to transfer a copy of the file to the user DTP.

STOR causes the server DTP to accept data transferred from the client DTP and store the data in its filesystem. Files will be replaced if they already exist, or created if not.

NOOP is a no-op, so it doesn't do anything. The server will send an OK reply.

In summary, the client PI sends a command to the server PI, then the server PI may store or update state such as user information or transfer parameters or instruct the server DTP to send data to the client using some transfer parameters. Or, the server DTP may wait for the client DTP to send it data and save it.

The server PI will also send replies to the client after commands. Replies are Telnet strings and may be single line or multi-line. Single line replies consists of the 3 digit code, a space, the null-terminated message string, and the Telnet EOL. For multi-line replies, the first line contains the 3 digit code, a hyphen, the first line of the message. The lines in the middle contain more message text. The final line contains the 3 digit code again, a space, optional text, and the Telnet EOL. More information about FTP Replies can be found in section 4.2.

That's as much as I'll explain for FTP. I only scratched the surface with commands (there are more I didn't list above) and it will get more complicated when you have sequences of commands. Communication between the client PI, client DTP, server PI, and server DTP boils down to establishing TCP connections between processes; as long as you can wrap your head around the active and passive mode, this is pretty straightforward.

To reiterate, there's many edge cases and small details that I haven't included here and the best way to learn about them is to read the spec or by implementing it yourself! The spec includes much more information, such as BNF notation for command parameters, formats for reply codes, state ASCII diagrams for common scenarios, and more.

I hope you learned something new or found part of this post interesting, thanks for reading!
