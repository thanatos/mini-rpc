=======
MiniRPC
=======

*MiniRPC* is an on-the-wire protocol for doing RPC calls. Some points:

* Compression
* Binary, so small, but,
* doesn't dictate what format your call is; use JSON, XML, whatever suits you.
* Can do introspection.


Protocol
========


Basic Types
-----------

The following are basic types used in the protocol's serialization:

* ``<uint8>``: A single 8-bit unsigned integer.
* ``<uinteger>``: A variable-length unsigned integer. (See below for encoding.)
* ``<string>``: A text string. Encoded as:

  ::

      <string> ::= <string-length> <string-data>
      <string-length> ::= <uinteger>
      <string-data> ::= <octet>*

  String data is UTF-8, always. The data is length prefixed (i.e.,
  ``<string-data>`` should be ``<string-length>`` octets). The string data is
  never null terminated.

Variable-length integers
************************

Some integers in the protocol are encoded using a variable length encoding to
attempt to keep the number of bytes required on the wire to a minimum.

If the most significant bit of an octet is 0, then this is last octet in the
integer. If the most significant bit of an octet is 1, then there are more
octets to be read. The bits in each octet are joined together; the least
significant bits are sent first. (This is a kind of backwards from network byte
order, but much easier to encode.)

To send 127:

::

    0x7f

To send 128:

::

    0x80 0x01

To send 129:

::

    0x81 0x01


General overview
----------------

Messages are sent in packets; packets are, roughly:

::

    <packet> ::= <command-and-data-size> <compression-type> <command-and-data>

    <command-and-data-size> ::= <uinteger>
    <compression-type> ::= <uint8>

``<command-and-data-size>`` is the length, in octets, of ``<command-and-data>``.

``<compression-type>`` is an integer representing the compression algorithm
used to compress ``<command-and-data>`` — it must be the ID of an algorithm
from the server hello packet. (The first two packets to be send, client-hello
and server-hello, always use a compression type ID of 0, which means no
compression, as compression schemes haven't been negoiated at that point.)

``<command-and-data>`` is the bulk of the message, containing what the command
is, and any data associated with that command:

::

   <command-and-data> ::= <command-id> <command-data>

   <command-id> ::= <uint8>
   <command-data> ::= <octet>*

``<command-id>`` is the type of packet being sent; ``<command-data>`` is data
associated with that command.

The protocol starts with a "client hello"; the response to this is a "server
hello". From there, the client can send any command except "client hello".

Compression Algorithms
**********************

The protocol is built to support any compression algorithm. The server and
client, in order to use one, must agree on a name for the algorithm, and
transmit that during the client and server hello packets. The following names
are given to well known compression algorithms by this document, clients SHOULD
use these names when using these algorithms:

* ``zlib`` — `zlib <http://www.zlib.net/>`_
* ``bzip2`` — `bzip2 <http://www.bzip.org/>`_
* ``lzma`` — `LZMA <http://en.wikipedia.org/wiki/Lempel%E2%80%93Ziv%E2%80%93Markov_chain_algorithm>`_


Client Hello
------------

A client hello (type ``0x01``) is an initial hello, along with the list of
compression algorithms supported by the client:

::

    <client-hello> ::= 0x0c "mini-rpc-1.0" <compression-support-list>

    <compression-support-list> ::= <count> (<compression-name>*)
    <count> ::= <uint8>
    <compression-name> ::= <string>

The client starts with a ``<string>`` set to ``mini-rpc-1.0``; this simply serves
to validate that the client is speaking what the server expects, and perhaps
vice versa.

Next, the client lists the set of available compression algorithms. Clients
SHOULD make support at least ``zlib`` compression.

Note that this data is wrapped in the ``<packet>`` structure, thus, a
compression algorithm must be sent with it. Clients MUST send ``0x00`` as the
compression, and leave the body of the client hello packet uncompressed.
(``0x00`` corresponds to no compression during normal protocol flow.)


Server Hello
------------

The server sends a server hello (type ``0x81``) in response to a client hello.
In it, the server lists the acceptable compression mechanisms for the duration
of the connection.

The compression algorithms listed by the server are associated with IDs for
this connection; the IDs start at 1, with the first compression algorithm in
the server's response, and increment from there.

The server also associates these with an ID, which is the ID
sent in the ``<compression>`` field of the general packet structure.

The server should not include any compression types that the client did not
list in the return list. (The returned list should be
*compression_algorithms_supported_by_client* ∩
*compression_algorithms_supported_by_server*.)

::

    <server-hello> ::= 0x0c "mini-rpc-1.0" <compression-list>

    <compression-list> ::= <count> (<compression-name>*)
    <count> ::= <uint8>
    <compression-name> ::= <string>


Call Function
-------------

The client calls a function with a type ``0x02`` message:

::

    <function-call> ::= <function-name> <id> <data-size> <data>

    <function-name> ::= <string>
    <id> ::= <uinteger>
    <data-size> ::= <uinteger>
    <data> ::= <octet>*


The server responds with a ``0x82`` message.

If the function call is successful, the server returns the following:

::

    <function-result> ::= 0x01 <id> <result>

    <id> ::= <uinteger>
    <result> ::= <count> <data>
    <count> ::= <uinteger>
    <data> ::= <octet>*


If the function call is unsuccessful, the server returns the following:

::

    <failed-call> ::= 0x01 <id> <error-type> <error-number> <error-message> <error-data>

    <id> ::= <uinteger>
    <error-type> ::= <uint8>
    <error-number> ::= <uinteger>
    <error-message> ::= <string>
    <error-data> ::= <count> <data>
    <count> ::= <uinteger>
    <data> ::= <octet>*
