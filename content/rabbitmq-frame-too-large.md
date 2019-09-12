Title: Debugging RabbitMQ frame_too_large Error
Date: 2015-06-02 16:18
Tags: openstack, fedora
Authors: John Eckersberg
Category: tech

An [interesting
bug](https://bugzilla.redhat.com/show_bug.cgi?id=1222005) came to my
attention today, and upon digging into it, a very sane explanation
emerged for what initially to me looked like a crazy scenario.

Here's the relevant server-side error from the attachment in the bug:

    :::text
    =INFO REPORT==== 15-May-2015::08:49:46 ===
    accepting AMQP connection <0.783.0> (192.168.122.13:36399 -> 192.0.2.1:5672)
    
    =INFO REPORT==== 15-May-2015::08:49:56 ===
    accepting AMQP connection <0.839.0> (127.0.0.1:52346 -> 127.0.0.1:5672)
    
    =ERROR REPORT==== 15-May-2015::08:50:01 ===
    AMQP connection <0.783.0> (running), channel 19793 - error:
    {amqp_error,frame_error,
                "type 65, all octets = <<>>: {frame_too_large,1342177289,131064}",
                none}
    
    =ERROR REPORT==== 15-May-2015::08:50:04 ===
    closing AMQP connection <0.783.0> (192.168.122.13:36399 -> 192.0.2.1:5672):
    fatal_frame_error

The `{frame_too_large,1342177289,131064}` bit tells some details about
the frame: the maximum negotiated frame size is `131064` bytes
(default config value of
[frame_max](https://www.rabbitmq.com/configure.html)) and the frame we
received is `1342177289` bytes (1.25 GB).  That seems insanely large
on the surface.

Next, we see the channel logged is `19793`.  Channels are a means of
multiplexing logically distinct sessions over a single socket.  My gut
reaction is that you'd probably have some small number of channels,
and that `19173` is an atypically large value.  Checking an existing
RabbitMQ instance I have running in an undercloud seems to confirm:

    ::text
    [root@instack ~]# rabbitmqctl list_channels number | sort | uniq -c
         43 1
         15 2

So, 43 channels exist with channel number `1`, and 15 channels exist
with channel number `2`.  No channels with number `19173`.

Finally, make note that the frame type is `65`, we'll see that again
shortly.

To better understand what is going on, I checked out the [AMQP 0.9.1
specification](https://www.rabbitmq.com/resources/specs/amqp0-9-1.pdf).
Here's what I've gathered after reading through the Technical
Specifications starting on page 31.

When a client makes a new connection, the first thing it does is send
a protocol header to the server.  A protocol header looks like this:

    ::text
    +---+---+---+---+---+---+---+---+
    |'A'|'M'|'Q'|'P'| 0 | 0 | 9 | 1 |
    +---+---+---+---+---+---+---+---+
                8 octets

Broken down, the 8 bytes are the literal ASCII string "AMQP" (4
bytes), a null byte, the major version `0`, the minor version `9`, and
the revision `1`.  As hex values, it looks like this:

    ::text
    +------+------+------+------+------+------+------+------+
    | 0x41 | 0x4D | 0x51 | 0x50 | 0x00 | 0x00 | 0x09 | 0x01 |
    +------+------+------+------+------+------+------+------+

After this, the connection passes AMQP frames back and forth.  All
frames have the following format:

    ::text
    0      1         3         7                   size+7     size+8
    +------+---------+---------+   +-------------+   +-----------+
    | type | channel |  size   |   |   payload   |   | frame-end |
    +------+---------+---------+   +-------------+   +-----------+
     octet   short      long        'size' octets        octet

Furthermore, the specification defines that the only valid frame types
are `1`, `2`, `3`, or `4`.  So our frame type of `65` is invalid.
However as a programmer, `65` is drilled into my head as the ASCII
representation of capital 'A'.

By now, the pieces start falling into place, and the explanation
emerges.  This is the error produced if you have an existing,
established AMQP connection, and the client for some reason decides to
re-send the protocol header on the socket.  Possible explanations might be:

1.  Bad error handling/reconnect code

2.  Connection pooling weirdness

3.  Race conditions with multiple threads sharing a connection

4.  Etc.

Anyway, look what happens when we map the protocol header overtop of
the frame format.  We end up with:

*  type = 0x41 (decimal `65`), the 'A' in 'AMQP'

*  channel = 0x4D51 (decimal `19793`), the 'MQ' in 'AMQP'

*  size = 0x50000009 (decimal `1342177289`), the 'P' from 'AMQP' plus the version up to `9`

Which matches exactly with the error produced by the server.  So now I
at least have an explanation for the error.  Why exactly the client
resends the protocol header is another story, hopefully you will
eventually find the answer in the linked bugzilla bug.
