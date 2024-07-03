---
layout: ../../layouts/MarkdownPostLayout.astro
title: 'Roku ECP Programming with Swift 1'
author: 'Todd'
pubDate: 2023-05-22
---

# Introduction

Here recently, I have taken to dabble in swift a little bit. A little bit of a shout out to [Hacking With Swift](https://www.hackingwithswift.com/) for some great tutorials to get started with swift. This came about with my recently acquiring an Apple watch SE. I figured, I had a M1 Macbook Air, I might as well play around in the ecosystem some. One thing I've ran into recently as a headache is dealing with my Roku. Great little devices, but with small kids, I always seem to permanently loose the Roku remote. So we started using the Roku app on our phones in lieu. However, the app is rather clunky and a pain to deal with. However, I found some neat [documentation](https://developer.roku.com/docs/developer-program/dev-tools/external-control-api.md) on how to interact with the Rokue with my own code.

# Simple Service Discovery Protocol

First thing we may want to do is to be able to do is discover any roku devices on our network. Roku make their devices discoverable using [SSDP](https://en.wikipedia.org/wiki/Simple_Service_Discovery_Protocol) (Simple Service Discovery Protocol). It is a simple protocol to be able to query on a network for any kind of devices and to get some information on them. In the context of a roku device, it will send us information on how to communicate further over TCP directly with the device. Lets take a look at the request and response in Wireshark.

## Sending out the query for Devices

We can make thie query by sending a UDP packet to 239.255.255.250 on port 1900. All Roku devices listen on this. The packet we send is fairly simple.
```sh
  M-SEARCH * HTTP/1.1
  Host: 239.255.255.250:1900
  Man: "ssdp:discover"
  ST: roku:ecp
```
That goes into the body of the UDP packet being sent out. The bottom of the packet does require a carriage return line feed (\r\n). So it looks more like this
```sh
  M-SEARCH * HTTP/1.1\r\n
  Host: 239.255.255.250:1900\r\n
  Man: "ssdp:discover"\r\n
  ST: roku:ecp\r\n
  \r\n
```
## What we get in response

In response we get the following, at least for the details we care about.
```sh
  Frame 5110: 268 bytes on wire (2144 bits), 268 bytes captured (2144 bits) on interface en0, id 0
  Ethernet II, Src: Roku_b3:68:af (ac:ae:19:b3:68:af), Dst: Apple_46:9e:c8 (3c:a6:f6:46:9e:c8)
  Internet Protocol Version 4, Src: 192.168.0.14, Dst: 192.168.0.88
  User Datagram Protocol, Src Port: 1900, Dst Port: 57101
  Simple Service Discovery Protocol
    HTTP/1.1 200 OK\r\n
    Cache-Control: max-age=3600\r\n
    ST: roku:ecp\r\n
    USN: uuid:roku:ecp:YH000T494575\r\n
    Ext: \r\n
    Server: Roku/12.0.0 UPnP/1.0 Roku/12.0.0\r\n
    LOCATION: http://192.168.0.14:8060/\r\n
    device-group.roku.com: 3022633342B2DCFCBCB2\r\n
    \r\n
    [HTTP response 1/1]
```
In the response, we get a HTTP 200 OK. The two important fields we care about are the USN and the LOCATION. The USN has a uuid which in the above is YH000T494575. This uniquely identifies an individual Roku on a network with several other Rokus. The location is going to tell us where it's webserver lives. All Roku webservers live on port 8060 according to the documentation. So we don't really need the query for that. What we care most about is the IP address tied to that specific Roku. With this, we can now use the REST API that each Roku runs in order to fully control it.

# Code

The following is some simple code drafted up in a playground.
```swift
  import Foundation
  import Network

  let hostUDP: NWEndpoint.Host = "239.255.255.250"
  let portUDP: NWEndpoint.Port = 1900

  let packet = "M-SEARCH * HTTP/1.1\r\nHost: \(hostUDP):\(portUDP)\r\nMan: \"ssdp:discover\"\r\nST: roku:ecp\r\n\r\n"

  print("Packet to send:\n\(packet)")

  let connection: NWConnection? = NWConnection(host: hostUDP, port: portUDP, using: .udp)

  connection?.stateUpdateHandler = { (newState) in
    switch(newState) {
    case .ready:
      print("State: ready\n")
      let content = packet.data(using: String.Encoding.utf8)
      connection?.send(content: content, completion: NWConnection.SendCompletion.contentProcessed(({ (NWError) in
        if (NWError == nil) {
          print("Data was sent")
        } else {
          print("Error: \(String(describing: NWError))")
        }
      })))
    case .setup:
      print("Setup\n")
    case .cancelled:
      print("Cancelled\n")
    case .preparing:
      print("Preparing\n")
    default:
      print("State not defined\n")
    }
  }

  connection?.start(queue: .global())

  while(true) { }
```

Here you will need to bear with me since its been awhile since I've done any programming with UDP and more specifically, my first time with Swift. The beginning of the code is faily simple, import some libraries and create some variables for the host and port. I also type out the search packet with string interpolation for the port and host. Next, using NWConnection, start creating the connection. After that is the handler. One thing of importance here is, the connection must be in the *ready* state in order to send or receive. In this program, receiving is not handled. I did the receiving in wireshark. At the bottom, start the connection on the global DispactchQueue and enter in an infinite while loop. This is just the keep the program running as the DispatchQueue works it's way through the various states in the UDP socket and sends our request packet.



