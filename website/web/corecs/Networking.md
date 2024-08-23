# Networking

1. IP vs UDP vs TCP ? 
| TCP   | UDP |
| -------- | ------- |
| Applications include File Transfer, Media, Email, SSH | Gaming, DNS lookup, Real time multi media streaming |
| Reliable but slower    | UDP is faster, simpler, and more efficient than TCP. but no gaurantee data is delivered to destination    |  
| congestion control, It can reduce the speed of data based on the speed of the receiver.    | PC1    |  
| Established via 3 way handshake    | connectionless    |  
| no multicasting    | Supports multicasting    |  
| extensive error checking    | basic error checking    |  
| PC1    | PC1    |  

<br />

2. What happens when you hit enter on a url?
http or https specifies the protocol. The domain name will be resolved to the ip address via the domain name system lookup. Some caches might be involved on websites frequently visited in the browser, router etc.
Establish a connection with the server via TCP 3 way handshake.

3. What is a TCP 3 / 4 way handshake?
**3 way** -> client send SYN msg, server send SYN-ACK msg, client send ACK message back to establish connection.
**4 way** -> for termination of a connection