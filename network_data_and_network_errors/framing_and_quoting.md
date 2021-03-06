# Framing and Quoting

If you have made the far more common option of using a TCP stream for communication, then
you will face the issue of framing, that is, the issue of how to delimit your messages so that the receiver can tell where
one message ends and the next begins.

There is a **first pattern (streaming)** that can be used by extremely simple network protocols that involve only the
delivery of data—no response is expected, so there never has to come a time when the receiver decides
“Enough!” and turns around to send a response. In this case, the sender can loop until all of the outgoing
data has been passed to sendall() and then close() the socket. The receiver need only call recv()
repeatedly until the call finally returns an empty string, indicating that the sender has finally closed the
socket. You can see this pattern in `streamer.py`:
```python
import socket, sys
s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)

HOST = sys.argv.pop() if len(sys.argv) == 3 else '127.0.0.1'
PORT = 1060

if sys.argv[1:] == ['server']:
    s.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
    s.bind((HOST, PORT))
    s.listen(1)
    print 'Listening at', s.getsockname()
    sc, sockname = s.accept()
    print 'Accepted connection from', sockname
    sc.shutdown(socket.SHUT_WR)
    message = ''
    while True:
        more = sc.recv(8192)  # arbitrary value of 8k
        if not more:  # socket has closed when recv() returns ''
            break
        message += more
    print 'Done receiving the message; it says:'
    print message
    sc.close()
    s.close()

elif sys.argv[1:] == ['client']:
    s.connect((HOST, PORT))
    s.shutdown(socket.SHUT_RD)
    s.sendall('Beautiful is better than ugly.\n')
    s.sendall('Explicit is better than implicit.\n')
    s.sendall('Simple is better than complex.\n')
    s.close()

else:
    print >>sys.stderr, 'usage: streamer.py server|client [host]'
    ```
    If you run this script as a server and then, at another command prompt, run the client version, you
will see that all of the client's data makes it intact to the server, with the end-of-file event generated by
the client closing the socket serving as the only framing that is necessary:

```
root@erlerobot:~/Python_files#  python streamer.py server
Listening at ('127.0.0.1', 1060)
Accepted connection from ('127.0.0.1', 49592)
Done receiving the message; it says:
Beautiful is better than ugly.
Explicit is better than implicit.
Simple is better than complex.
```

There is a **second pattern** is a variant on the first: **streaming in both directions**. The socket is initially left
open in both directions. First, data is streamed in one direction—exactly and
then that direction alone is shut down. Second, data is then streamed in the other direction, and the
socket is finally closed.

A **third pattern**, which we have already seen, is to use fixed-length messages, as illustrated in
`tcp_sixteen.py`. You can use the Python `sendall()` method to keep sending parts of a string until the whole
thing has been transmitted, and then use a `recv()` loop of our own devising to make sure that you
receive the whole message.

A **fourth pattern** is to somehow delimit your messages with special characters. The receiver would
wait in a recv() loop like the one just cited, but wait until the reply string it was accumulating finally
contained the delimiter indicating the end-of-message.

A **fifth pattern** is to prefix each message with its length. This is a very popular choice for highperformance
protocols since blocks of binary data can be sent verbatim without having to be analyzed,
quoted, or interpolated. Of course, the length itself has to be framed using one of the techniques given
previously—often it is simply a fixed-width binary integer, or else a variable-length decimal string
followed by a delimiter. But either way, once the length has been read and decoded, the receiver can
enter a loop and call `recv()` repeatedly until the whole message has arrived.

There is sixth pattern for which the unknown lengths are no problem. Instead of sending just one,
try sending several blocks of data that are each prefixed with their length. This means that as each chunk
of new information becomes available to the sender, it can be labeled with its length and placed on the
outgoing stream. When the end finally arrives, the sender can emit an agreed-upon signal—perhaps a
length field giving the number zero—that tells the receiver that the series of blocks is complete.

Following(`blocks.py`) you can find an example of this sixth pattern.Like the previous one, this sends data
in only one direction—from the client to the server—but the data structure is much more interesting.
Each message is prefixed with a 4-byte length; in a struct, 'I' means a 32-bit unsigned integer, meaning
that these messages can be up to 4GB in length. A series of three such messages is sent to the server,
followed by a zero-length message—which is essentially just a length field with zeros inside and then no
message data after it—to signal that the series of blocks is over.
```

import socket, struct, sys
s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)

HOST = sys.argv.pop() if len(sys.argv) == 3 else '127.0.0.1'
PORT = 1060
format = struct.Struct('!I')  # for messages up to 2**32 - 1 in length

def recvall(sock, length):
    data = ''
    while len(data) < length:
        more = sock.recv(length - len(data))
        if not more:
            raise EOFError('socket closed %d bytes into a %d-byte message'
                           % (len(data), length))
        data += more
    return data

def get(sock):
    lendata = recvall(sock, format.size)
    (length,) = format.unpack(lendata)
    return recvall(sock, length)

def put(sock, message):
    sock.send(format.pack(len(message)) + message)

if sys.argv[1:] == ['server']:
    s.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
    s.bind((HOST, PORT))
    s.listen(1)
    print 'Listening at', s.getsockname()
    sc, sockname = s.accept()
    print 'Accepted connection from', sockname
    sc.shutdown(socket.SHUT_WR)
    while True:
        message = get(sc)
        if not message:
            break
        print 'Message says:', repr(message)
    sc.close()
    s.close()

elif sys.argv[1:] == ['client']:
    s.connect((HOST, PORT))
    s.shutdown(socket.SHUT_RD)
    put(s, 'Beautiful is better than ugly.')
    put(s, 'Explicit is better than implicit.')
    put(s, 'Simple is better than complex.')
    put(s, '')
    s.close()

else:
    print >>sys.stderr, 'usage: streamer.py server|client [host]'
    ```


 Running first the server and then the client in different terminals, resulto on:



```
root@erlerobot:~/Python_files# python blocks.py server
Listening at ('127.0.0.1', 1060)
Accepted connection from ('127.0.0.1', 49692)
Message says: 'Beautiful is better than ugly.'
Message says: 'Explicit is better than implicit.'
Message says: 'Simple is better than complex.'
root@erlerobot:~/Python_files#
```
