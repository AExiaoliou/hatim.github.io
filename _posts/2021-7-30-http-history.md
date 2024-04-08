---
layout : post
title : "HTTP的演变，1->1.1->2->3(QUIC)"
date :  +0800
categories: other
---

本博客部分内容来自mdn文章[HTTP 的发展](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Basics_of_HTTP/Evolution_of_HTTP)

>HTTP（HyperText Transfer Protocol）是万维网（World Wide Web）的基础协议。自 Tim Berners-Lee 博士和他的团队在 1989-1991 年间创造出它以来，HTTP 已经发生了太多的变化，在保持协议简单性的同时，不断扩展其灵活性。如今，HTTP 已经从一个只在实验室之间交换文件的早期协议进化到了可以传输图片，高分辨率视频和 3D 效果的现代复杂互联网协议。

## 最开始的版本——HTTP0.9
HTTP最一开始的版本其实没有版本号，不够后来把这个版本定为0.9。

这个版本的HTTP非常地原始，构成也很简单，请求由一个简单的`GET`+地址组成，响应也是直接返回整个html内容，连头部都没有，突出一个能用就行。

请求：
```
GET /mypage.html
```
响应：
```html
<html>
  这是一个非常简单的 HTML 页面
</html>
```

现在已经很难找到http0.9的兼容实现了，毕竟个响应头都没有，不过curl如今依然保留了一个http0.9的兼容实现。

```c
    if(!end_ptr) {
      /* Not a complete header line within buffer, append the data to
         the end of the headerbuff. */
      result = Curl_dyn_addn(&data->state.headerb, buf, blen);
      if(result)
        return result;
      *pconsumed += blen;

      if(!k->headerline) {
        /* check if this looks like a protocol header */
        statusline st =
          checkprotoprefix(data, conn,
                           Curl_dyn_ptr(&data->state.headerb),
                           Curl_dyn_len(&data->state.headerb));

        if(st == STATUS_BAD) {
          /* this is not the beginning of a protocol first header line */
          k->header = FALSE;
          streamclose(conn, "bad HTTP: No end-of-message indicator");
          if(conn->httpversion >= 10) {
            failf(data, "Invalid status line");
            return CURLE_WEIRD_SERVER_REPLY;
          }
          if(!data->set.http09_allowed) {
            failf(data, "Received HTTP/0.9 when not allowed");
            return CURLE_UNSUPPORTED_PROTOCOL;
          }
          leftover_body = TRUE;
          goto out;
        }
      }
      goto out; /* read more and try again */
    }
```

curl默认不支持http0.9的版本，必须打开选项`--http0.9`，实现也很简单，发现这个响应头有点怪并且开启了选项的话，就把剩下的当html读就完事了。

可以实现一个简单的http0.9服务器看下这个功能能不能用
```rust
use std::{
    error::Error,
    io::{BufRead, BufReader, Write},
    net::TcpListener,
};

fn main() -> Result<(), Box<dyn Error>> {
    let listener = TcpListener::bind("127.0.0.1:7878")?;

    for stream in listener.incoming() {
        let mut stream = stream?;
        let lines: Vec<_> = BufReader::new(&stream)
            .lines()
            .map(|result| result.unwrap())
            .take_while(|line| !line.is_empty())
            .collect();
        println!("{lines:#?}");

        // 直接返回html
        stream.write_all("<html> 这是一个非常简单的 HTML 页面</html>".as_bytes())?;
    }

    Ok(())
}
```

命令`curl --http0.9 -vvv localhost:7878`可以看到html输出，如果不带http0.9选项会告知不支持http0.9

## 混乱的http1实现

> 文档 RFC 1945 定义了 HTTP/1.0，但它是狭义的，并不是官方标准。

现在开始http长得和我们熟悉的样子差不多了，响应头状态码...感觉回到了家的感觉。    

> - 协议版本信息现在会随着每个请求发送（HTTP/1.0 被追加到了 GET 行）。
> - 状态码会在响应开始时发送，使浏览器能了解请求执行成功或失败，并相应调整行为（如更新或使用本地缓存）。
> - 引入了 HTTP 标头的概念，无论是对于请求还是响应，允许传输元数据，使协议变得非常灵活，更具扩展性。
> - 在新 HTTP 标头的帮助下，具备了传输除纯文本 HTML 文件以外其他类型文档的能力（凭借 Content-Type 标头）。

相比http1.1, http1.0最大的问题就是它是事实不是一份官方标准而且在很多地方留下了歧义比如下面这部分有关状态码的标准。

```
9.1  Informational 1xx

   This class of status code indicates a provisional response,
   consisting only of the Status-Line and optional headers, and is
   terminated by an empty line. HTTP/1.0 does not define any 1xx status
   codes and they are not a valid response to a HTTP/1.0 request.
   However, they may be useful for experimental applications which are
   outside the scope of this specification.
```

是的...这份标准只要求你返回一个1xx的状态码就行了，并没有像后续的http标准明确规定了100、101到底是什么意思并且在什么情况下返回。

这导致了很多地方都按自己的理解实现，类似的用途你返回一个100我返回一个199，能不被淘汰就见鬼了。

## http1.1的时代

> HTTP/1.1 在 1997 年 1 月以 RFC 2068 文件发布。

终于回到了我们最熟悉的http1.1，相比http1.0最明显的一点是...标准变长了:)  
为了避免歧义并增加各项新特性，标准页数成功由60页膨胀到了162页，这还没算因为后续发展增加的其他相关标准。

> 连接可以复用，节省了多次打开 TCP 连接加载网页文档资源的时间。
增加管线化技术，允许在第一个应答被完全发送之前就发送第二个请求，以降低通信延迟。
> 支持响应分块。
> 引入额外的缓存控制机制。
> 引入内容协商机制，包括语言、编码、类型等。并允许客户端和服务器之间约定以最合适的内容进行交换。
> 凭借 Host 标头，能够使不同域名配置在同一个 IP 地址的服务器上。

## 发展与进步

## 更进一步——http2

http2相比之前的第一大进步是使用二进制进行传输数据，不过有人认为把明文传输改成二进制增加了debug的复杂性，个人认为这个理由非常离谱且站不住脚，毕竟众所周知http2及以前是基于tcp的，而tcp显然是一个二进制传输协议，不如说在http2之前使用明文进行传输更加奇怪。

## 这真的是http吗——QUIC与http3

