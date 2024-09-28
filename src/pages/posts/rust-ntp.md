---
layout: ../../layouts/post.astro
title: 'Rust NTP Client'
author: 'Todd'
pubDate: 2022-11-09
---

# Introduction

Something that I thought would be interesting is to dabble a little bit with some obscure network programming, however something simple. This is the Network Time Protocol(NTP). NTP has been around for year, before 1985 and is a way to synchronize time across computers in networks. There have been 4 version of this protocol with the latest indepth RFC being 5905 and RFC 7822 with some updates. RFC 5905 established the standard for NTPv4. There is also a more light weight subset of NTP called SNTP, however, here I will focus on implementing a simple client using NTPv4.

# Datastructure

The primary packet for NTPv4 has one form. It consists of 48 bytes of data. Or atleat, those are the parts of the packet we care about. There are some option fields, however, we will not concern ourselves with this. The information for What we care about as far as the packet in RFC 5905 is located in page 17, chapter 7.
```sh
    | Field               | Size    |
    |:--------------------|:--------|
    | LI                  | 2 bits  |
    | VN                  | 3 bits  |
    | Mode                | 3 bits  |
    | Stratum             | 1 byte  |
    | Poll                | 1 byte  |
    | Precision           | 1 byte  |
    | Root delay          | 4 bytes |
    | Root Dispersion     | 4 bytes |
    | Reference ID        | 4 bytes |
    | Reference Timestamp | 4 bytes |
    | Origin Timestamp    | 4 bytes |
    | Receive Timestamp   | 4 bytes |
    | Transmit Timespace  | 4 bytes |
```
First is LI, which is short for *Leap Indicator*. This will indicate if there anything we need to concern ourselves about a leap second being added or subtracted. This takes up the first 2 bits. Next is VN, *Version Number*, encoded in 3 bits. After that is the *Mode* for 3 bits. When handling this packet, the first 3 fields will fit in an unsigned 8-bit integer, in rust as a u8.

In the rest of the packet, 1 byte can be treated as a *u8* in Rust and 4 bytes as a *u32*. 

*Stratum* is an interesting concept in NTP. The *Stratum Model* has to deal with the hierarchy of machines. Levels 0-15 indicate the device's distance from the reference clock. Stratum 0 can be directly connected to something such as an atomix clock. Stratum 0 devices are usually connected directly to a Startum 1 device that will serve as the time server to distribute time to Stratum 2 clients or servers and so on. Generally, it can be assumed that the higher the stratum number is, the less stability and accuracy there is. Any stratum greater than 15 is rejected.

*Poll* is the max interval between messages, this is taken in log2 seconds. 

*Precision* represents the precision of the system clock, also in log2 seconds.

For the remaining fields, I suggest taking a look at *RFC 5905*. 

# main.rs
```rust
  use std::net::UdpSocket;

  use rs_sntp::ntp_client::{write_data_packet, read_data_packet, NtpDataPacket};

  const HOST: &str = "0.pool.ntp.org";
  const PORT: &str = "123";

  fn main() {

    let mut packet = NtpDataPacket::new();
    
    let connection_string = HOST.to_owned() + ":" + PORT;

    let mut socket: UdpSocket = UdpSocket::bind("0.0.0.0:8000")
        .expect("Could not bind to socket");

    socket.connect(connection_string).expect("Could not connect to server");

    write_data_packet(&socket);

    println!("Data sent");

    let response = read_data_packet(&socket);

    println!("Data reseived");

    packet.parse_response(response);

    println!("{:?}", packet);

  }
```
Here is the main source file of the program. In the program we create an NtpDataPacket which is a structure I made to represent the packet for NTP. I also have some constants set for the host and port of one of the many publically availiable NTP servers. NTP uses UDP, so a UDPSocket is created, binded and connected. Next, write to the socket, which will be covered later in this article. Read the response, parse and display. Pretty straight forward and simple. THe magic for the protocol happens in the `ntp_client.rs` file.

# lib.rs

Really simple `lib.rs` file
```rust
       pub mod ntp_client;
```

# ntp_client.rs
```rust
  use std::net::UdpSocket;

  #[derive(Debug)]
  pub struct NtpDataPacket {
    li_vn_mode: u8,
    strat: u8, 
    poll: u8,
    prec: u8,
    root_delay: u32,
    root_dispersion: u32,
    ref_id: u32,
    ref_time_seconds: u32,
    ref_time_fraction: u32,
    origin_ts_seconds: u32,
    origin_ts_fraction: u32,
    recv_ts_seconds: u32,
    recv_ts_fraction: u32,
    tx_ts_seconds: u32,
    tx_ts_fraction: u32,
  }

  pub fn write_data_packet(socket: &UdpSocket) {
    let mut buf = [0; 48];
    buf[0] = 0x1b;
    socket.send(&buf).expect("Issue with sending to server");
  }

  pub fn read_data_packet(socket: &UdpSocket) -> [u8; 48] {
    let mut buf: [u8; 48] = [0; 48];
    if let Ok(r) = socket.recv(&mut buf) { 
        println!("Received: {} bytes", r);
    } else { 
        println!("Error receiving from server!");
    }

    return buf;
  }

  impl NtpDataPacket {

    pub fn new() -> Self {
        Self { li_vn_mode: 0, strat: 0, poll: 0, prec: 0, root_delay: 0, root_dispersion: 0, ref_id: 0, ref_time_seconds: 0, ref_time_fraction: 0, origin_ts_seconds: 0, origin_ts_fraction: 0, recv_ts_seconds: 0, recv_ts_fraction: 0, tx_ts_seconds: 0, tx_ts_fraction: 0 }
    }

    pub fn parse_response(&mut self, resp: [u8; 48]) {
        self.li_vn_mode = resp[0];
        self.strat = resp[1];
        self.poll = resp[2];
        self.prec = resp[3];
        self.root_delay |= (resp[4] as u32) << 24;
        self.root_delay |= (resp[5] as u32) << 16;
        self.root_delay |= (resp[6] as u32) << 8;
        self.root_delay |= resp[7] as u32;
        self.root_dispersion |= (resp[8] as u32) << 24;
        self.root_dispersion |= (resp[9] as u32) << 16;
        self.root_dispersion |= (resp[10] as u32) << 8;
        self.root_dispersion |= resp[11] as u32;
        self.ref_id |= (resp[12] as u32) << 24;
        self.ref_id |= (resp[13] as u32) << 16;
        self.ref_id |= (resp[14] as u32) << 8;
        self.ref_id |= resp[15] as u32;
        self.ref_time_seconds |= (resp[16] as u32) << 24;
        self.ref_time_seconds |= (resp[17] as u32) << 16;
        self.ref_time_seconds |= (resp[18] as u32) << 8;
        self.ref_time_seconds |= resp[19] as u32;
        self.ref_time_fraction |= (resp[20] as u32) << 24;
        self.ref_time_fraction |= (resp[21] as u32) << 16;
        self.ref_time_fraction |= (resp[22] as u32) << 8;
        self.ref_time_fraction |= resp[23] as u32;
        self.origin_ts_seconds |= (resp[24] as u32) << 24;
        self.origin_ts_seconds |= (resp[25] as u32) << 16;
        self.origin_ts_seconds |= (resp[26] as u32) << 8;
        self.origin_ts_seconds |= resp[27] as u32;
        self.origin_ts_fraction |= (resp[28] as u32) << 24;
        self.origin_ts_fraction |= (resp[29] as u32) << 16;
        self.origin_ts_fraction |= (resp[30] as u32) << 8;
        self.origin_ts_fraction |= resp[31] as u32;
        self.recv_ts_seconds |= (resp[32] as u32) << 24;
        self.recv_ts_seconds |= (resp[33] as u32) << 16;
        self.recv_ts_seconds |= (resp[34] as u32) << 8;
        self.recv_ts_seconds |= resp[35] as u32;
        self.recv_ts_fraction |= (resp[36] as u32) << 24;
        self.recv_ts_fraction |= (resp[37] as u32) << 16;
        self.recv_ts_fraction |= (resp[38] as u32) << 8;
        self.recv_ts_fraction |= resp[39] as u32;
        self.tx_ts_seconds |= (resp[40] as u32) << 24;
        self.tx_ts_seconds |= (resp[41] as u32) << 16;
        self.tx_ts_seconds |= (resp[42] as u32) << 8;
        self.tx_ts_seconds |= resp[43] as u32;
        self.tx_ts_fraction |= (resp[44] as u32) << 24;
        self.tx_ts_fraction |= (resp[45] as u32) << 16;
        self.tx_ts_fraction |= (resp[46] as u32) << 8;
        self.tx_ts_fraction |= resp[47] as u32;
    }

  }
```

## NtpDataPacket

This is a structure that defines the packet for the protocol. Some of the fields are described some above. This structure is 48 bytes in total. To keep communication simple, an array of 48 bytes (u8) is created, sent and we received it. A function is created in the implementation block of NtpDataPacket to parse the 48 byte block received into the data structure for better handling. Using bitwise logical operators to ensure that data is placed appropraitely. 

## What is required by the packet

In the send function, before sending the 48 byte packet, the first byte is set to value `0x1b`. This is `0b00011011`. This will set the appropriate LI, VN and Mode bits.
```sh
   | Key  | Value | Meaning                                                                     |
   |:-----|:------|-----------------------------------------------------------------------------|
   | LI   | 00    | This is for server use, server decide is leap second is inserted or deleted |
   | VN   | 011   | Version number, set to 3 in this example, could also be set to 4            |
   | Mode | 011   | Mode is set to 3, which means client mode                                   |
```

The version and mode need to be set in order to get a response.

# Output

	NtpDataPacket { li_vn_mode: 28, strat: 2, poll: 3, prec: 236, root_delay: 3802, root_dispersion: 1545, ref_id: 2886875149, ref_time_seconds: 3877304791, ref_time_fraction: 1240575520, origin_ts_seconds: 0, origin_ts_fraction: 0, recv_ts_seconds: 3877306283, recv_ts_fraction: 2288430625, tx_ts_seconds: 3877306283, tx_ts_fraction: 2288611839 }

Above is the output from the server. The primary field toget the time is the `tx_ts_seconds` field. However, converting that will not give the current time. `3877306283` yields a date and time of `November 12th, 2092 5:31:23 AM.` So, how do we get the correct time? We substract 70 years worth of seconds from the time. This is 1900 to the UNIX epoch of Janruary 1st, 1970. 70 years in seconds is `2208988800`. This will give us the proper time.
