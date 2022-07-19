---
title: CONNECT AXI Wrapper Documentation (Draft)
date: 2022-06-16 15:00:00 -0700
categories: [CONNECT]
tags: []
img_path: /assets/img/CONNECT/
---

# Introduction

The AXI4[-Stream] wrapper is built over the [CONNECT](https://research.ece.cmu.edu/calcm/new_connect/connect/) NoC, which is a network generator targeting FPGA.

# Getting Started

First, clone the CONNECT repository and build a sample network.

> TODO: Replace the git repository address in the future.

```bash
$ git clone https://github.com/shili2017/connect.git
$ cd connect
$ python gen_network.py -t double_ring -w 38 -n 4 -v 2 -d 4 --gen_rtl --gen_graph
```

The network will be generated and put in the `build` directory. Then we can run the sample test bench.

```bash
$ cp Prims/run_modelsim.sh build/
$ cd build
$ ./run_modelsim.sh
```

The default sample test bench is for AXI4-Stream protocol. To test on AXI4 standard protocol, in `run_modelsim.sh`, replace `CONNECT_testbench_sample_axi4stream` by `CONNECT_testbench_sample_axi4`.

To change (a) flit width or (b) number of virtual channels or (c) flit buffer depth, first re-generate the network.

```bash
$ python gen_network.py -t double_ring -w <flit_width> -n 4 -v <num_vcs> -d <flit_buffer_depth> --gen_rtl --gen_graph
```

Then change the parameters in `connect_parameters.v`.

```verilog
`define NUM_VCS <num_vcs>
`define FLIT_DATA_WIDTH <flit_depth>
`define FLIT_BUFFER_DEPTH <flit_buffer_depth>
```

> Do not change number of send or receive ports for now. So far the wrapper is still written manually, and only supports 4 terminals in the network.
>
> TODO: Consider generating the network wrapper automatically with scripts.

> To prevent potential deadlocks, AXI4 standard protocol requires at least 2 virtual channels, one for requests and the other for responses, and responses have higher priority. Meanwhile, AXI4-Stream only needs 1 virtual channel.

# Overall Design

The overall design is shown in the following blocks.

AXI4:

```
                                        CLK_FPGA                                     |                      CLK_NOC
---------------------------------------------------------------------------------------------------------------------

+--------+            +------------+             +--------------+             +-------------+             +---------+
| Device | <--AXI4--> | AXI4Bridge | ---flits--> |  Serializer  | ---flits--> | InPortFIFO  | ---flits--> |         |
+--------+            +------------+   97 bits   +--------------+    n bits   +-------------+    n bits   |         |
                            ^                                                                             | Network |
                            |                    +--------------+             +-------------+             |         |
                            +-----------flits--- | Deserializer | <--flits--- | OutPortFIFO | <--flits--- |         |
                                       97 bits   +--------------+    n bits   +-------------+    n bits   +---------+
```

AXI4-Stream:

```
                                                     CLK_FPGA                                      |                      CLK_NOC
-----------------------------------------------------------------------------------------------------------------------------------

+---------------+                   +------------+             +--------------+             +-------------+             +---------+
| Master Device | ---AXI4-Stream--> | AXI4Bridge | ---flits--> |  Serializer  | ---flits--> | InPortFIFO  | ---flits--> |         |
+---------------+                   +------------+   106 bits  +--------------+    n bits   +-------------+    n bits   |         |
                                                                                                                        | Network |
+---------------+                   +------------+             +--------------+             +-------------+             |         |
| Slave  Device | <--AXI4-Stream--- | AXI4Bridge | <--flits--- | Deserializer | <--flits--- | OutPortFIFO | <--flits--- |         |
+---------------+                   +------------+   106 bits  +--------------+    n bits   +-------------+    n bits   +---------+
```

## Related Source Files

- `NetworkAXI4Wrapper.sv`

- `NetworkAXI4StreamWrapper.sv`

# AXI4[-Stream] Bridge Design

An AXI4 bridge is used to convert a pair of send & recv port to a standard AXI4 protocol interface, either as a master device or a slave device. For example, for a network with 4 endpoints, we can have 2 AXI4 master devices and 2 AXI4 slave devices.

An AXI4-Stream bridge is used to convert a send or recv port to a AXI4-Stream protocol master or slave interface. For example, for a network with 4 endpoints, we can have 4 AXI4-Stream master devices and 4 AXI4-Stream slave devices.

## Related Parameters
Parameters are defined in `AXI4Package.sv`.

```verilog
parameter ADDR_WIDTH   = 32;
parameter DATA_WIDTH   = 64;
parameter STRB_WIDTH   = DATA_WIDTH / 8;
parameter ID_WIDTH     = 8;
parameter USER_WIDTH   = 8;
parameter KEEP_WIDTH   = DATA_WIDTH / 8;
parameter DEST_WIDTH   = 4;
```

Users also need to calculate the standard AXI4[-Stream] flit data width by themselves. For example, with the default parameters, we have

```verilog
parameter AXI4_FLIT_DATA_WIDTH  = 76;
parameter AXI4S_FLIT_DATA_WIDTH = 93;
```

## AXI4

AXI4 requests and responses are encoded in a single AXI4 flit. A standard AXI4 request or response with the aforementioned parameters is encoded into a 92-bit flit. The 3 most significant bits represent for the channel flag for the 5 AXI4 channels.

> User devices are required to mark the source ID in AXI4 ID field, and AXI4 ID field is put in the LSB part in the flit to be identified by the serializer & deserializer.

AXI4 requests are sent in virtual channel 1, and AXI4 responses are sent in virtual channel 0, which has a higher priority to prevent potential deadlocks.

### AW/AR Channel

```
AXI_x     - [75     ]
axid      - [74 : 67]
------ padding ------
axregion  - [60 : 57]
axqos     - [56 : 53]
axprot    - [52 : 50]
axcache   - [49 : 46]
axlock    - [45     ]
axburst   - [44 : 43]
axsize    - [42 : 40]
axlen     - [39 : 32]
axaddr    - [31 :  0]
```

### W Channel

```
AXI_W     - [75     ]
------ padding ------
wlast     - [72     ]
wstrb     - [71 : 64]
wdata     - [63 :  0]
```

### B Channel

```
AXI_W     - [75     ]
bid       - [74 : 67]
------ padding ------
bresp     - [65 : 64]
------ padding ------
```

### R Channel

```
AXI_R     - [75     ]
rid       - [74 : 67]
rlast     - [66     ]
rresp     - [65 : 64]
rdata     - [63 :  0]
```

## AXI4-Stream

AXI4-Stream requests are encoded in a single AXI4-Stream flit. A AXI4-Stream request with the aforementioned parameters is encoded into a 101-bit flit.

> User devices are required to mark the source ID in AXI4-Stream ID field, and AXI4-Stream ID field is put in the LSB part in the flit to be identified by the serializer & deserializer.

### T Channel

```
tdest - [92 : 89]
tid   - [88 : 81]
tlast - [80     ]
tkeep - [79 : 72]
tstrb - [71 : 64]
tdata - [63 :  0]
```

## Related Source Files

- `AXI4Bridge.sv`

- `AXI4StreamBridge.sv`

- `AXI4Interface.sv`

- `AXI4Package.sv`

# Serializer & Deserializer Design

To handle mismatched sizes of AXI4[-Stream] protocol flits and actual flits to be sent, a serializer & deserializer for each master & slave device is needed. For example, for a AXI4-Stream request flit of 101+5 bits (5 is metadata, including valid bit, tail bit, etc.) and actual flit width of 38+5 bits, a single request needs to be split into 3 small flits by the serializer on the sender side and then small flits can be reassembled to a complete AXI request on the receiver side. 

What makes things more complicated is that in the case that two master devices send data to a single slave device, flits may interleave (when virtual link is not enabled in CONNECT, or in general NoCs). Thus we need to encode source ID into the small flit such that the deserializer can distinguish interleaving flits. Fortunately, both AXI4 and AXI4-Stream supports interleaving requests and responses on the protocol level, so we don't need to store the whole packet, whose size is unknown at the beginnning. Namely, for every single transfer in AXI4[-Stream], we set the tail bit to 1.

For the serializer, the number of flits is `ceil(IN_FLIT_DATA_WIDTH / OUT_FLIT_EFF_DATA_WIDTH)`, where `OUT_FLIT_EFF_DATA_WIDTH = OUT_FLIT_DATA_WIDTH - SRC_BITS`. Similarly, for the deserializer, the number of flits is `ceil(OUT_FLIT_DATA_WIDTH / IN_FLIT_EFF_DATA_WIDTH)`, where `IN_FLIT_EFF_DATA_WIDTH = IN_FLIT_DATA_WIDTH - SRC_BITS`. The structure of small flit is shown as follows,

```
 -----------------------------------------------------------------------------------
 | <valid bit> | <is_tail> | <destination> | <virtual channel> | <source> | <data> |
 -----------------------------------------------------------------------------------
        1            1        `DEST_BITS         `VC_BITS       `SRC_BITS  OUT_FLIT_EFF_DATA_WIDTH
```

To handle interleaving flits, currently we adopt only a simple solution, as we know that flits are in-order and we know the number of senders. We make a table with one row per source ID and just fill in the table with successive flits until the tail flit arrives. At that moment the flits can be reassembled into a AXI4 flit and be processed by a later stage. We also implement a priority encoder as an arbiter to handle multiple valid reassembled flits in a single cycle. See the code below.

```verilog
// Use a priority encoder to decide which flit to send
logic [`SRC_BITS - 1 : 0] out_idx;
always_comb begin
  out_idx = 0;
  for (int i = `NUM_USER_SEND_PORTS - 1; i >= 0; i--) begin
    if (out_valid[i])
      out_idx = i;
  end
end
always_comb begin
  for (int i = 0; i < `NUM_USER_SEND_PORTS; i++) begin
    out_ready[i] = 0;
    out_flit = 0;
  end
  out_ready[out_idx] = out_flit_ready;
  out_flit = {flit_meta_reg[out_idx], flit_data_reg[out_idx][OUT_FLIT_DATA_WIDTH - 1 : 0]};
end
```

## Related Source Files

- `FlitSerializer.sv`

# Flow Control Layer Design

Credit-based flow control is supported in CONNECT and the wrapper. To support flow control, a synchronous FIFO, i.e., `SCFIFO`, is implemented as a buffer between the user devices (or more accurately, the serailizer or deserializer) and the network. On the sender side, we need to maintain a credit counter. Each time we send a flit, we need to decrement the counter, and each time we receive a credit from the network, we increment the counter. On the receiver side, each time we dequeue a flit from the FIFO to the user device, we send a credit to the network.

The network itself can run at a higher clock frequency as ASIC, and the remaining FPGA logic may run at a lower clock frequency. Thus a FIFO with different read and write clock, i.e., `DCFIFO`, is also needed. The block diagram of a FIFO is shown below.

`InPortFIFO`:

```
            (WR) CLK_FPGA   CLK_NOC (RD)             CLK_NOC
                        |   |                        |
                        v   v                        v                             +---------------+
                     +--------+                    +--------+                      |               | ---send_ports_putFlit_flit_in-->
---put_flit--------> |        | ---sc_enq_data---> |        | ---deq_flit_data---> | Combinational | ---EN_send_ports_putFlit------->
---put_flit_valid--> | DCFIFO | ---sc_enq_valid--> | SCFIFO | ---deq_flit_valid--> | Logic         | <--send_ports_getCredits--------
<--put_flit_ready--- |        | <--sc_enq_ready--- |        | <--deq_flit_ready--- |               | ---EN_send_ports_getCredits---->
                     +--------+                    +--------+                      +---------------+
```

`OutPortFIFO`:

```
            (RD) CLK_FPGA   CLK_NOC (WR)             CLK_NOC
                        |   |                        |
                        v   v                        v                             +---------------+
                     +--------+                    +--------+                      |               | <--recv_ports_getFlit------------
<--get_flit--------- |        | <--sc_deq_data---- |        | <--enq_flit_data---- | Combinational | ---EN_recv_ports_getFlit-------->
<--get_flit_valid--- | DCFIFO | <--sc_deq_valid--- | SCFIFO | <--enq_flit_valid--- | Logic         | ---recv_ports_putCredits_cr_in-->
---get_flit_ready--> |        | ---sc_deq_ready--> |        | ---enq_flit_ready--> |               | ---EN_recv_ports_putCredits----->
                     +--------+                    +--------+                      +---------------+
```

> TODO: 1) Call AXI4 flit a packet. 2) One FIFO for each VC. 3) Add one more VC for W channel. 4) Do read and write in the same cycle? 5) #rows in deserializer should be #src * #vc. 6) One DCFIFO, handle credits approximately. 7) Serializer & deserializer should be close to NoC.

## Related Parameters

Parameters of both `DCFIFO` and `SCFIFO` can be modified by users.

`SCFIFO`:

```verilog
parameter DEPTH               = `FLIT_BUFFER_DEPTH;
parameter DATA_WIDTH          = `FLIT_WIDTH;
parameter ALMOST_FULL_VALUE   = `FLIT_BUFFER_DEPTH - 1;
parameter ALMOST_EMPTY_VALUE  = 1;
```

`DCFIFO`:

```verilog
parameter DEPTH               = `FLIT_BUFFER_DEPTH;
parameter DATA_WIDTH          = `FLIT_WIDTH;
```

## Related Source Files

- `FlitFIFO.sv`

# Testbench

To test the AXI4[-Stream] wrapper, we also need AXI4[-Stream] devices to send requests or receive responses. We also include testbenches to verify the correctness of AXI4[-Stream] wrapper design. See related source files.

## Related Source Files

- `AXI4Device.sv`

- `AXI4StreamDevice.sv`

- `testbench_sample_axi4.sv`

- `testbench_sample_axi4stream.sv`
