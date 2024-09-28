---
layout: ../../layouts/post.astro
title: 'Roku ECP Programming with Swift 2'
author: 'Todd'
pubDate: 2023-05-23
---

In the previous [article](https://enigmaticbits.com/post/swift-ecp-programming-1/), I went over a little bit about discovery of Roku devices on the network. Accomplished by sending out a UDP call on port 239.255.255.250 on port 1900. A response comes back with the a message with two important blocks of information, the unique device identifier and the IP address for the device on the local area network. Now, for a little bit of code to make the device do something.

Looking at the [documentation](https://developer.roku.com/docs/developer-program/dev-tools/external-control-api.md), there are a number of endpoints we can hit on the device's HTTP REST server. The initial one we will target is the keypress, since this is probably the most basic thing we want to do. We want to be able to navigate the screen. In the table where it talks about the endpoint, we can also see there is a link which takes up to how we can specify the [key](https://developer.roku.com/docs/developer-program/dev-tools/external-control-api.md#keypress-key-values).

A little bit below that on the page, it shows some examples using curl. While I did this for some initial playing, the main point of interest is to be able to control the thing via code written in swift.

```swift
  import Foundation

  let host = "192.168.0.14"
  let port = 8060

  let direction = "Right"

  let url: URL? = URL(string: "http://\(host):\(port)/keypress/\(direction)")
  var request: URLRequest = URLRequest(url: url!)
  request.httpMethod = "POST"
  print("Sending request")

  let (data, response) = try await URLSession.shared.data(for: request)

  print("Response: \(response)")
```

The code is simple. Just a quick script to run in Swift playgrounds. The request for a keypress must be sent as a *POST* request, so the method is set. Using string interpolation, we set the url. The full url looks like so
```sh
  http://192.169.0.14:8060/keypress/Right
```
Using swift's async/await, make the request and print out the response. The following is the response we get back
```swift
  <NSHTTPURLResponse: 0x1286d4790> { URL: http://192.168.0.14:8060/keypress/Left } { Status Code: 200, Headers {
    "Content-Length" =     (
        0
    );
    Server =     (
        "Roku/12.0.0 UPnP/1.0 Roku/12.0.0"
    );
  } }
```
Not a whole lot of exciting data, the most important bit is the status code of 200 telling us everything went well.

The API provides a lot of different endpoints with my main focus being on keypress. It provides the usualy expected navigation such as left, right, up, and down. Along with an endpoint for home, back and several others.
