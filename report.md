# Reliable Data Transfer

## 1. Abstract

The main objective of this experiment is to implement the Reliable Data Transfer (RDT) protocol at the transport layer, including the one-way Stop-and-Wait protocol and the Go-Back-N protocol. By writing the related programs in C language and conducting network simulation tests, stable and reliable data transmission from a virtual sender to a receiver can be achieved, with packet loss effectively reduced.

In terms of protocol design and implementation, by reading the experiment manual and combining it with the provided code framework, the functions of data reception (`input`), data sending (`output`), and timeout interruption (`timerinterrupt`) at both the sender and receiver sides were simulated, thereby realizing reliable data transmission. After completing the full two-week project, the program was able to simulate all cases of the two protocols, namely Stop-and-Wait and Go-Back-N, under the following conditions: no packet loss and no corruption, packet loss without corruption, corruption without packet loss, and both packet loss and corruption. Thus, the functionality of one-way RDT was realized.

**Keywords:** lexical analysis, syntax analysis, compiler

 

## 2. Task Requirement Analysis

### 2.1 Implementing the One-Way Stop-and-Wait Protocol

#### 2.1.1 Protocol Working Requirements

The general rule of Stop-and-Wait is as follows: when the two communicating parties exchange data, after the receiver correctly receives a data packet from the sender, it needs to send an acknowledgment message (ACK) back to the sender to confirm that the data has been received. After sending one packet, the sender stops and waits until it receives the acknowledgment from the receiver before sending the next packet, as shown in Figure 2-1. The Stop-and-Wait protocol in this experiment is intended to simulate the above behavior.

**Figure 2-1** Schematic diagram of the Stop-and-Wait protocol rules

#### 2.1.2 Protocol Application Scenarios

Because the RDT Stop-and-Wait protocol checks whether the other side has received each packet during transmission, otherwise triggering an error recovery mechanism, it features high reliability, meaning that it is unlikely to cause a large amount of packet loss or disorder. However, its implementation is relatively complex. Therefore, the Stop-and-Wait protocol implemented in this experiment is suitable for scenarios requiring secure and stable data transmission, such as the transfer of office documents, but it is not suitable for data transmission in scenarios such as online chatting.

### 2.2 Implementing the One-Way Go-Back-N Protocol

#### 2.2.1 Protocol Working Requirements

As shown in Figure 1-2, the general rule of Go-Back-N is as follows: under normal conditions, the sender sends data packets to the receiver; after receiving a packet, the receiver returns an ACK to the sender; once the sender receives the ACK, it can continue sending data. However, once the receiver detects that the received data are out of order, the sender must retransmit all unacknowledged packets after the last correctly received packet.

**Figure 1-2** Schematic diagram of the Go-Back-N protocol

If the sender has sent N packets and a previous packet times out, then that packet is considered lost or corrupted, and the sender will retransmit that erroneous packet and the subsequent sequence of packets.

The Go-Back-N experiment is intended to simulate the above behavior.

#### 2.2.2 Protocol Application Scenarios

Compared with the Stop-and-Wait protocol, the GBN protocol has more intuitive and simpler rules in implementation. GBN adopts a pipeline-like mechanism, so it can be used when the communication link quality is relatively high and a higher communication speed is required, such as in sending and receiving instant messaging data in a network.

 

## 3. Protocol Design

### 3.1 Framework Design

This experiment is divided into two modules: SW and GBN. In order to obtain an overall understanding of the protocol framework and facilitate the subsequent detailed implementation of internal logic, the following overall design was developed.

#### 3.1.1 Implementing the One-Way Stop-and-Wait Protocol

Based on the requirement analysis, the following FSM diagrams were drawn (as shown in Figures 3-1 and 3-2).

In the figures, after the sender or receiver jumps from state 0 to state 1, the corresponding values such as `checksum` and `ack` also change. Initially, transmission starts from `seq = 0` and `ack = 0`.

(1) Sender A  
**Figure 3-1** FSM diagram of sender A

(2) Receiver B  
**Figure 3-2** FSM diagram of receiver B

#### 3.1.2 Implementing the One-Way Go-Back-N Protocol

Based on the requirement analysis, the following FSM diagrams were drawn.

(1) Sender A

**Figure 3-3** FSM diagram of the sender

As shown in Figure 3-3, the sender waits for events to occur. If data packets need to be sent, it determines whether the buffer is full and then calls `tolayer3` to send the packets. If a packet is received, it must determine whether the packet is correct and then perform different operations accordingly, such as moving the window pointer. If a timeout occurs, `timerinterrupt` is called to retransmit all unacknowledged packets after the last correctly received packet.

(2) Receiver B  
**Figure 3-4** FSM diagram of the receiver

As shown in Figure 3-4, if the receiver gets a normal packet and its sequence number is also correct, then it extracts the data from the packet and uses `tolayer5` to deliver it upward. Then it encapsulates the outgoing packet, sets values such as `ACK`, and calls `tolayer3` to send it back to the sender.

### 3.2 Data Structure Design

The main data structures used in this experiment are as follows.

#### 3.2.1 `msg` and `pkt`

`msg` is the message transmitted between layer 5 and layer 4, consisting of a character array of 20 elements. `pkt` is the packet transmitted between layer 4 and layer 3, which contains the following fields:

- `seqnum`
- `acknum`
- `checksum`
- `payload`

**Figure 3-5** Schematic diagram of the `packet` structure

Among them, `seqnum` is the sequence number of the packet, `acknum` indicates the sequence number of the ACK message, `checksum` is the packet checksum, and `payload` records the header information of the packet so that packets are less likely to be lost during grouped transmission.

#### 3.2.2 Event List `event`

`event` is a data structure used in functions such as `main`, and it has already been encapsulated, as shown in Figure 3-6. The event list is used to link together each event assigned to the sender and receiver, triggering their corresponding responses and also participating in the final event trace output.

- `evtime`
- `evtype`
- `eventity`
- `pktptr`
- `*prev`
- `*next`

**Figure 3-6** Schematic diagram of the `event list` data structure

`evtime` and `evtype` denote the occurrence time and type of the event, respectively (`1` and `2`). `eventity` is an integer variable related to the location where the event occurs. The pointer `pktptr` points to the packet associated with that event. The last two fields, `prev` and `next`, are recursively defined pointers of type `event`, which point to the previous and next events, respectively, thereby forming the event list into a doubly linked list.

#### 3.2.3 Sender A

A is the simulated sender in the Stop-and-Wait implementation. Its data structure is shown in Figure 3-7.

- `nextSeq`
- `waitAck`
- `lastPkt`
- `status`

**Figure 3-7** Schematic diagram of the sender A structure

`nextSeq` is the sequence number of the next packet to be sent by the sender; `waitAck` is the ACK number the sender is waiting for, and the sender can continue subsequent transmission only after receiving that ACK; `lastPkt` is of type `pkt`, representing the last sent packet, so that its information can be reused if an error occurs during transmission; `status` is the sender state, including waiting (`WAIT`), ready (`READY`), and so on.

#### 3.2.4 Receiver B

- `seq`

**Figure 3-8** Schematic diagram of the receiver B structure

B is the simulated receiver in the Stop-and-Wait implementation. Its data structure is relatively simple, consisting only of the received sequence number `seq`.

#### 3.2.5 Buffer

The buffer is a newly defined data structure similar to a circular array and queue, required in the implementation of the Go-Back-N protocol to store packets, as shown in Figure 3-9.

**Figure 3-9** Schematic diagram of the packet buffer structure

The 0th position of the array is `base`, where the first packet is stored. Combined with the sender FSM drawn in Section 3.1, the following implementation of the buffer was designed in this experiment:

1. After the sender receives input data, it checks the buffer; if the array is not full, the packet is placed into it.
2. The buffer can also serve as the window. Changes in `base` determine changes in the window: after the ACK for the packet at `base` is received, the window moves backward; if the response for `base` times out, then all packets in the buffer are popped and retransmitted.
3. The lower boundary of the buffer (window) and the position of the next packet (adjacent in `seqnum`) within the window are maintained by the sender.

### 3.3 Protocol Rule Design

The rules designed for the two RDT protocols in this experiment are shown in the following tables.

#### 3.3.1 Stop-and-Wait Protocol

**Table 1. Rule design of the SW protocol**

| Event Situation | Sender | Receiver |
|--- | --- | --- |
| Normal case | 1. Send a packet into the window.  2. Wait for the returned ACK.  3. Receive the ACK and continue sending the next packet. | 1. Receive the incoming packet and return an ACK to the sender.  2. Receive the incoming packet and return an ACK to the sender. |
| Sender data loss (`data`) | 1. Packet is lost during transmission.  3. Timeout occurs because no ACK is received.  4. Retransmit the packet.  5. Receive the ACK and continue sending the next packet. | 1. Continue waiting for the incoming packet.  2. Receive the incoming packet and return an ACK to the sender.  3. Receive the incoming packet and return an ACK to the sender. |
| Sender data corruption (`data`) | 1. Send a packet into the window.  2. Packet is corrupted during transmission.  3. Fail to receive the correct ACK and a timeout occurs.  4. Retransmit the packet.  5. Receive the ACK and continue sending the next packet. | 1. Receive the incoming packet and detect an error.  2. Receive a duplicate packet and return an ACK to the sender.  3. Receive the incoming packet and return an ACK to the sender. |
| Receiver ACK loss | 1. Send a packet.  2. Wait for the ACK returned by B.  3. Timeout occurs because no ACK is received.  4. Retransmit the packet.  5. Receive the ACK and continue sending the next packet. | 1. Receive the incoming packet and return an ACK to the sender.  2. ACK packet is lost during transmission.  3. Receive a redundant packet and return an ACK to the sender.  4. Receive the incoming packet and return an ACK to the sender. |
| Receiver ACK corruption | 1. Send a packet.  2. Wait for the ACK returned by B.  3. The received ACK is incorrect, so retransmit the previous correct packet.  4. Receive the ACK and continue sending the next packet. | 1. Receive the incoming packet and return an ACK to the sender.  2. ACK packet is corrupted during transmission.  3. Receive a redundant packet and send the corresponding ACK.  4. Receive the incoming packet and return an ACK to the sender. |

(Continued table)

#### 3.3.2 Go-Back-N Protocol

**Table 2. Rule design of the GBN protocol**

| Buffer Status | Event Situation | Sender | Receiver |
|--- |--- | --- | --- |
| Buffer not full | Normal case | 1. Send a packet into the window.  2. Wait for the returned ACK.  3. Receive the ACK, move the window, and continue sending the next packet. | 1. Receive the incoming packet and return an ACK to the sender.  2. Receive the incoming packet and return an ACK to the sender. |
| Buffer not full | Sender data loss (`data`) | 1. Send a packet into the window.  2. Packet is lost during transmission.  3. Timeout occurs because no ACK is received.  4. Retransmit all sent packets within the window.  5. Receive the ACK, move the window, and continue sending the next packet. | 1. Continue waiting for the incoming packet.  2. Receive the incoming packet and return an ACK to the sender.  3. Receive the incoming packet and return an ACK to the sender. |
| Buffer not full | Sender data corruption (`data`) | 1. Send a packet into the window.  2. Packet is corrupted during transmission.  3. The received ACK is incorrect.  4. Retransmit `base`.  5. Receive the ACK, move the window, and continue sending the next packet. | 1. Receive the incoming packet, detect the error, and package a reply.  2. Receive the incoming packet and return an ACK to the sender.  3. Receive the incoming packet and return an ACK to the sender. |
| Buffer not full | Receiver ACK loss | 1. Send a packet into the window.  2. Wait for the returned ACK.  3. Timeout occurs because no ACK is received.  4. Retransmit all sent packets within the window.  5. Receive the ACK, move the window, and continue sending the next packet. | 1. Receive the incoming packet and return an ACK to the sender.  2. ACK packet is lost during transmission.  3. Receive the incoming packet and return an ACK to the sender.  4. Receive the incoming packet and return an ACK to the sender. |
| Buffer not full | Receiver ACK corruption | 1. Send a packet into the window.  2. Wait for the returned ACK.  3. The received ACK is corrupted, so discard this ACK packet.  4. No ACK is received, timeout occurs.  5. Retransmit all sent packets within the window.  6. Receive the ACK, move the window, and continue sending the next packet. | 1. Receive the incoming packet and return an ACK to the sender.  2. ACK packet is corrupted during transmission.  3. Receive the incoming packet and return an ACK to the sender.  4. Receive the incoming packet and return an ACK to the sender. |
| Buffer full | — | Return the packet without processing. | None |

(Continued table)

 

## 4. Protocol Implementation

### 4.1 Implementing the One-Way Stop-and-Wait Protocol

#### 4.1.1 Checksum Calculation

According to the explanation in the experiment manual and the classroom introduction to the RDT protocol, the checksum is calculated by summing elements such as `seqnum`, `acknum`, `checksum`, and each character value in `payload` within the packet, with additional handling for special cases.

#### 4.1.2 Initialization

The empty functions `A_init` and `B_init` are used to initialize the sender and receiver. The status of A is set to ready, and all other attributes are set to 0 (since no packet has yet been sent). The `seq` value of B is also initialized to 0.

#### 4.1.3 Input and Output

First, `A_input` and `B_input` are completed so that A receives the ACK packet returned by B, while whenever a packet sent by A arrives at B, `B_input` is called. The pseudocode is as follows:
```
A_input: BEGIN
    if checksum is non-zero and therefore incorrect then
        output packet corruption error information and report the actual checksum
        return
    endif
    else if the received ACK is the value A is waiting for then
        output success information
        set A.status to READY
        return
    endif
    excluding the above cases, the received ACK packet is erroneous
    print error information
END

B_input: BEGIN
    if checksum of the received packet != original checksum then
        output packet corruption error information
        obtain the checksum of the previous packet received by B and output it
        tolayer3 (1, previous packet received by B)
        return
    endif
    if the received packet number matches B then
        this means B has received the correct packet, so output correct information
        set fields such as acknum and checksum of the outgoing packet outpkt
        tolayer3 (1, outpkt)
        tolayer5 (payload of the returned packet)
        return
    endif
    excluding the above cases, the received packet is erroneous
    output error information
    tolayer3 (1, previous packet)
    return
END

A_output: BEGIN
    if sender A is still in WAIT state then
        this means it is waiting for ACK after timeout, so output related information
        return
    endif
    else
        set seqnum, acknum, and checksum of the outgoing packet outpkt
        toggle A.nextSeq and A.waitAck between 1 and 0
        set A to WAIT state and start timing to determine whether timeout occurs
        tolayer3(1, outpkt)
        send the packet and output related success information
        return
    endelse
END
```
#### 4.1.4 Timer Interrupt

When sender A is in the `WAIT` state, that is, waiting for the ACK sent by B, if no message is received for a long time, then an interruption is triggered (`A_timerinterrupt`). It calls `tolayer3` to perform the interrupt handling in preparation for retransmitting the packet. In this experiment, no timeout interrupt is set for receiver B.
```
A_timerinterrupt: BEGIN
    if sender A is not in WAIT state then
        this means it is not waiting for ACK, so output related information
        return
    endif
    print timeout information
    tolayer3(0, the previous packet sent by A)
    starttimer(0, 20.0)
    return
END
```
It can be seen that the above functions cover all four cases: packet loss, packet corruption, ACK loss, and ACK corruption.

### 4.2 Implementing the One-Way Go-Back-N Protocol

#### 4.2.1 Checksum Calculation

According to the explanation in the experiment manual and the classroom introduction to the RDT protocol, the checksum is calculated by summing `seqnum`, `acknum`, and each character value in `payload` within the packet, with additional handling for special cases.

#### 4.2.2 Initialization

The empty functions `A_init` and `B_init` are used to initialize the sender and receiver. It is only necessary to set the variables related to the sender and receiver to their default values.

#### 4.2.3 Input and Output

First, `A_input` and `B_input` are completed so that A receives the ACK packet returned by B, while whenever a packet sent by A arrives at B, `B_input` is called. The pseudocode is as follows:
```
A_input: BEGIN
    receive packet from the receiver and print information
    if packet checksum is incorrect then
        a corrupted packet has been received, so print error information
        discard the corrupted packet
    endif
    if packet.acknum < base then
        a duplicate packet has been received, so print error information
        discard the duplicate packet
    endif
    else if packet.acknum >= maximum sequence number nextSeq in the buffer then
        print error information
        discard the erroneous packet
    endif
    reset base and packet.acknum
    stoptimer(A)
    print information indicating successful reception
END

B_input: BEGIN
    receive packet from the sender and print information
    number of packets received by B++
    if packet checksum is incorrect then
        a corrupted packet has been received, so print error information
        discard the corrupted packet
    endif
    if packet checksum is correct && seqnum is correct then
        receiver expectedSeq++
        tolayer5(payload)
        number of packets sent by B++
        print success information
    endif
    else if packet seqnum is incorrect then
        packet is out of order, so print error information
        discard the out-of-order packet
    endif
    if receiver expectedSeq >= 1 then
        set packet ACK and checksum
        tolayer3(packet)
        print information
    endif
END

A_output: BEGIN
    receive message and add it to the msg list
    number of packets received by A++
    print reception information
    if the window is full then
        print information, return
    endif
    pop one message
    take one packet p from the buffer
    if p == NULL then
        the window is full, print information
    endif
    assign message to p->payload
    set p->seqnum, acknum, and checksum
    tolayer3 (p)
    number of packets sent by A++
    if base == buffer capacity then
        starttimer(20.0)
    endif
    buffer capacity++
    print information about sending the packet
END
```
#### 4.2.4 Timer Interrupt

When sender A is in the `WAIT` state, that is, waiting for the ACK from B, if no message is received for a long time, then an interruption is triggered (`A_timerinterrupt`). It calls `tolayer3` to perform the interrupt handling in preparation for retransmitting packets. In this experiment, no timeout interrupt is set for receiver B. The pseudocode is as follows:
```
A_timerinterrupt: BEGIN
    timeout occurs, print timeout information
    for i from base to nextSeq-1; i++ do
        take out packets[i] from the buffer
        use tolayer3 to retransmit packets[i]
        number of packets sent by A++
        print retransmission information
    endfor
    if there are still packets in the buffer then
        starttimer(20.0)
    endif
END
```
It can be seen that the above functions also cover the four cases of packet loss, packet corruption, ACK loss, and ACK corruption.

 

## 5. Experimental Results and Analysis

### 5.1 Implementing the One-Way Stop-and-Wait Protocol

Network simulation tests were conducted on the Stop-and-Wait protocol. The input parameters and corresponding situations are as follows.

**Table 3. Simulation test parameters for the SW protocol**

| Scenario | Number of messages sent | Packet loss rate | Corruption rate | Sending interval | Trace |
| --- | --- | ---| ---| ---| ---|
| No error, no loss | 20 | 0.0 | 0.0 | 1000 | 2 |
| Error, no loss | 20 | 0.0 | 0.5 | 1000 | 2 |
| No error, loss | 20 | 0.5 | 0.0 | 1000 | 2 |
| Error and loss | 20 | 0.5 | 0.5 | 1000 | 2 |

#### 5.1.1 No Error, No Loss

Set the first parameter to 20, the second parameter (packet loss rate) and the third parameter (corruption rate) to 0, the fourth parameter to 1000 (that is, a time interval of 1 s), and the fifth parameter `trace` to its default value 2. The following result segment is obtained:

**Figure 5-1** Test result with no error and no loss

It can be seen that 20 data packets were successfully sent, and all were correctly transmitted and received. The yellow mark indicates that A sent one packet, and the green mark indicates that B correctly received that packet.

#### 5.1.2 Error, No Loss

Based on the input of Section 5.1.1, change the corruption rate to 0.5 while keeping the other inputs unchanged, and the following result is obtained:

(a)

(b)

**Figure 5-2** Test result with error but no loss

From Figures (a) and (b), it can be seen that some packets were corrupted, resulting in the message **The packet B received is damaged** (marked in blue). Meanwhile, some ACKs were also corrupted, resulting in the message **The packet A received is damaged** (marked in yellow). In addition, the ports retransmit after receiving erroneous packets, and Figure 5-2(b) shows that the second transmission succeeded.

#### 5.1.3 No Error, Loss

Based on the input of Section 5.1.1, change the packet loss rate to 0.5 while keeping the other inputs unchanged, and the following result is obtained:

(a)

(b)

**Figure 5-3** Test result with no error but with loss

As shown in Figure 5-3, since the corruption rate is 0, the reason for timeout at A is that the ACK packet from B to A was lost (marked in yellow); the reason for retransmission at B is that the packet from A to B was lost (marked in green).

#### 5.1.4 Error and Loss

Based on the input of Section 5.1.1, change both the packet loss rate and the corruption rate to 0.5, and the following result is obtained:

**Figure 5-4** Test result with both error and loss

It can be seen from the figure that under the presence of both errors and packet loss, all situations occur, including correct sending and receiving, packet loss and corruption, and ACK loss and corruption, all marked with different colors.

Through the controlled-variable tests in Sections 5.1.2 and 5.1.3, it has already been confirmed that the program can identify loss or error in different parts and output corresponding error messages. The test in Section 5.1.4 further verifies these cases. In summary, the experimental test results of the Stop-and-Wait protocol are generally successful.

### 5.2 Implementing the One-Way Go-Back-N Protocol

Network simulation tests were conducted on the Stop-and-Wait protocol. The input parameters and corresponding situations are as follows. Because GBN can roll back multiple messages at a time, the sending interval was adjusted to 100 in this experiment.

**Table 4. Simulation test parameters for the GBN protocol**

| Scenario | Number of messages sent | Packet loss rate | Corruption rate | Sending interval | Trace |
| --- | --- |---|---|  --- | --- |
| No error, no loss | 20 | 0.0 | 0.0 | 100 | 2 |
| Error, no loss | 20 | 0.0 | 0.5 | 100 | 2 |
| No error, loss | 20 | 0.5 | 0.0 | 100 | 2 |
| Error and loss | 20 | 0.5 | 0.5 | 100 | 2 |

#### 5.2.1 No Error, No Loss

Set the first parameter to 20, the second parameter (packet loss rate) and third parameter (corruption rate) to 0, the fourth parameter to 100 (that is, a time interval of 0.1 s), and the fifth parameter `trace` to its default value 2. The following result segment is obtained:

**Figure 5-5** Test result with no error and no loss

It can be seen that 20 data packets were successfully sent, and all were correctly transmitted and received. The yellow mark indicates that A sent a packet to layer 3, whose sequence number is `seq = 16`, and the green mark indicates that B correctly received that packet.

#### 5.2.2 Error, No Loss

Based on the input of Section 5.2.1, change the corruption rate to 0.5 while keeping the other inputs unchanged, and the following results are obtained:

**Figure 5-6(a)** Data packet error, no loss

Figure 5-6(a) shows that some data packets were corrupted, resulting in `[Sender] TOLAYER3: packet being corrupted` and the corresponding `[Receiver] Packet is corrupted` (both marked in blue).

**Figure 5-6(b)** ACK packet error, no loss

Figure 5-6(b) shows that some returned ACK packets were corrupted, resulting in `[Receiver] TOLAYER3: packet being corrupted` and the corresponding `[Sender] ACK Packet is corrupted` (both marked in yellow).

#### 5.2.3 No Error, Loss

Based on the input of Section 5.2.1, change the packet loss rate to 0.5 while keeping the other inputs unchanged, and the following results are obtained:

**Figure 5-7(a)** Data packet loss, no error

As shown in Figure 5-7(a), the sender loses a packet during transmission, namely `[Sender] TOLAYER3: packet being lost`, which then triggers a timeout interrupt because the ACK is not received, causing all packets in the window `[1,2)` to be retransmitted (marked by the blue box at the top of Figure a). After that, the receiver receives the correct packet and sends back an ACK (marked by the blue box at the bottom of Figure a).

**Figure 5-7(b)** ACK packet loss, no error

As shown in Figure 5-7(b), after the receiver sends an ACK packet, that ACK is lost, namely `[Receiver] TOLAYER3: packet being lost` (marked by the green box at the top of Figure b). This causes the sender to trigger a timeout interrupt because no ACK is received, thereby retransmitting the data packet(s) in window `[8,9)` (marked by the green box at the bottom of Figure b).

#### 5.2.4 Error and Loss

Based on the input of Section 5.2.1, change both the packet loss rate and corruption rate to 0.5, and the following result is obtained:

**Figure 5-8** Test result with both error and loss

It can be seen from the figure that under the presence of both error and packet loss, all situations occur, including correct sending and various packet loss and corruption cases, all marked with different colors.

Through the controlled-variable tests in Sections 5.2.2 and 5.2.3, it has already been confirmed that the program can identify loss or error in different parts and output corresponding error messages. The test in Section 5.2.4 further verifies these cases. In summary, the experimental test results of the Go-Back-N protocol are generally successful.

 

## 6. Conclusion

In this experiment, the design and implementation of reliable data transfer protocols were reproduced, leading to a deeper understanding of the sending and receiving mechanisms of RDT. Through two weeks of practical work, two one-way RDT protocols were basically implemented, and all performance indicators required by the tests were met.

### 6.1 Problems Encountered in the Experiment and Their Solutions

(1) In the first week, the Dev-C++ IDE was used for network simulation testing, but it was found that the framework source code provided by the experiment could not run directly and required modifications in the function definitions, such as adding `void` type declarations. After consulting classmates, it was learned that the framework code could run in the GCC compilation environment of the virtual machine, so this method was adopted for the subsequent work.

(2) The internal details of the framework code were not sufficiently familiar at first, which led to deviations between the test results and expectations in the later debugging stage, especially under packet loss or corruption conditions. After checking the source code, it was found that the issue was caused by an incorrect timer start position, which made timeout detection fail on some occasions at the sender side. After adjusting the code, the problem was resolved.

### 6.2 Reflections on Completing the Experiment

This was our first practical project related to transport-layer knowledge. Through this RDT programming exercise, we gained a great deal: our practical ability, programming ability, and ability to complete relatively large-scale projects were all improved. It is hoped that in future study, our understanding of computer networks and network programming will continue to deepen, thereby enhancing our technical competence.

During this experiment, I would especially like to thank our teacher and the two teaching assistants for their efforts. Through their explanations of the relevant knowledge and the weekly centralized discussions they organized, we were able to learn more from the classroom, and our ability to write technical reports also improved significantly.

 

## Notes

Some modifications and improvements were made in this report to the work reports of the first two weeks, and these have been marked in yellow in the original document.
