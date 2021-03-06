# kdb-pcap-decoder
### Packet Sniffing

Packet sniffing is the practice of gathering, collecting, and logging some or all packets that pass through a computer network, regardless of how the packet is addressed. The code in this repository uses kdb+ to decode a .pcap file, generated by a tcpdump, into a table that has relevant information for easy viewing.


### Generating a Packet

Tcpdump is a command line utility that will capture and analyse network traffic in your system which can be used for security or to troubleshoot network issues. 

Check if tcpdump is installed on your linux system with the following command:

```
$ which tcpdump
/usr/sbin/tcpdump
```

To capture packets, tcpdump requires elevated permissions, so `sudo` must be used in command line executions.
There are many different flags that can be used to operate tcpdump and an article explaining some of the flags can be found here:
https://opensource.com/article/18/10/introduction-tcpdump

Below is an example of a command line expression that is used to listen to port 2222, caps the number of packets at 25 and also writes the packets to a .pcap file:
```
sudo tcpdump -i any -c25 -nn -w webserver.pcap port 2222
```

### Example Usage
Start a q session in the base directory of the git repository using:
```
q common/decoder.q
```
Once inside the session, the table can be built using the following:
```
.pcap.buildtable[`:example/kdbcapture.pcap]
```
Which will produce the following output:
```
time                          src       dest      srcport destport protocol flags    se..
---------------------------------------------------------------------------------------..
2020.05.14D12:10:02.639526000 127.0.0.1 127.0.0.1 55196   18596    TCP      `ACK`PSH 31..
2020.05.14D12:10:02.639641000 127.0.0.1 127.0.0.1 18596   55196    TCP      `ACK`PSH 26..
2020.05.14D12:10:02.639662000 127.0.0.1 127.0.0.1 55196   18596    TCP      ,`ACK    31..
2020.05.14D12:10:04.027428000 127.0.0.1 127.0.0.1 55196   18596    TCP      `ACK`PSH 31..
2020.05.14D12:10:04.027504000 127.0.0.1 127.0.0.1 18596   55196    TCP      `ACK`PSH 26..
2020.05.14D12:10:04.027516000 127.0.0.1 127.0.0.1 55196   18596    TCP      ,`ACK    31..
2020.05.14D12:10:18.598946000 127.0.0.1 127.0.0.1 55196   18596    TCP      `ACK`PSH 31..
2020.05.14D12:10:18.599056000 127.0.0.1 127.0.0.1 18596   55196    TCP      `ACK`PSH 26..
2020.05.14D12:10:18.599079000 127.0.0.1 127.0.0.1 55196   18596    TCP      ,`ACK    31..
2020.05.14D12:10:22.959434000 127.0.0.1 127.0.0.1 55196   18596    TCP      `ACK`PSH 31..
2020.05.14D12:10:22.959547000 127.0.0.1 127.0.0.1 18596   55196    TCP      `ACK`PSH 26..
...
```


### Packet Structure
The structure of a .pcap file is formed as follows:
* Global header
  * Packet header
    * Packet data
  * Packet header
    * Packet data
  * ...

#### Global Header
24 bytes

#### Packet Header
16 bytes
Contains the timestamp and the length of the packet data. The length field is used by the kdb+ code to iterate through each of the packets and grab the same fields each time.

More information on the global and packet headers can be found here: https://wiki.wireshark.org/Development/LibpcapFileFormat

#### Packet Data
The length of this section will be outlined in the packet header length field. 

Below is an explanation of the columns in the table that the function produces:
* time = The time the packet was captured
* src = The IP the packet was sent from
* dest = The IP the packet was sent to
* srcport = The port the packet was sent from
* destport = The port the packet was sent to
* protocol = The type of protocol
* flags = The list of tcp flags attached to packet
* seq = Sequence number (raw), which tracks bytes in each direction of the connection, the sequence number of the next packet from A to B = the sequence number of the previous A to B packet + the previous A to B len (relative seqs start at 1)
* ack = Acknowledgment number (raw), tracks bytes sent from the other side of the connection, the ack number sent from A should be the same as the last seq number received by A
* win = Window size value (calculated after scaling applied), window size of the packets from A to B indicate how much buffer space is available on A for receiving packets. So when B receives a packet with window size 1, it would tell B how many bytes it is allowed to send to A.
* tsval = Timestamp value, arbitrary time value to send to B from A and await echo reply, e.g. in packet 1
* tsecr = Timestamp echo reply, tsval number from previous packet sent back to A from B, e.g. in packet 2.
  * tsval and tsecr values gradually increment as packets are sent back and forth, they let tcp know the state of the network connection between src and dest, e.g. to establish network latency or to improve throughput
* length = The length, in bytes, of the packet data (not including packet header)
* len = The length, in bytes, of the data content (payload)
* data = The data content (payload)

All of the columns are populated from the packet data apart from time and length which are populated from the packet header.
