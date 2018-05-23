---
title: Building my own http Client
date: 2018-02-26T16:39:27Z
lastmod: 2018-02-26T16:39:27Z
author: Jerry Caligiure
cover: /images/httpclient.jpeg
categories: ["Challenges"]
tags: ["web", "go", "dailyprogrammer", "challenge"]
---

Building an http client from scratch!

<!--more-->

Hello! Web clients are everywhere. You're reading this on one, most likely! Nearly everything
most poeple do on computers is making http requests, but how does that actually happen? What is 
going on when you open a website? Hopefully, this can provide some insight into that!

Today I'm going to try doing a challenge from the dailyprogrammer rather
than anything actually constructive ^_^! The challenge is 
[here](https://www.reddit.com/r/dailyprogrammer/comments/7jzy8k/20171215_challenge_344_hard_write_a_web_client/)
and it's basically to build an http client from the ground-up.

The code is going to be on my [github](https://github.com/Noofbiz/httpClient) if you want to take a look,
and I made a [c9](https://ide.c9.io/noofbiz/http-client) for it as well. Let's get started.

First off, go has an amazing http package in its standard library, so this would be too easy
if we were allowed to utilize those. I'll start from there and kind of work backwards to see how
it should work at the end of the day.

```go
package main

import (
	"fmt"
	"io/ioutil"
	"net/http"
)

func main() {
	var resp *http.Response
	var err error
	if resp, err = http.Get("http://httpbin.org/get"); err != nil {
		fmt.Printf("Unable to GET. %v\n", err)
	}
	defer resp.Body.Close()
	var b []byte
	if b, err = ioutil.ReadAll(resp.Body); err != nil {
		fmt.Printf("Malformed response body. %v\n", err)
	}
	fmt.Printf("%s\n", b)
}
```

So, this is super simple and against the rules! However, it does do some neat things that
will be useful later. It forms an http Request, which is what the client sends from itself
to the server. It essentially tells the server that it wants to tell it something. Then the
server reads the request and sends the client back a response. This is essentially how a
web browser works: it sends a request to a web server, and the web server responds by sending
the browser back the web page for it to display for the user.

To actually learn something about http clients,
I'm going to limit myself to just the net.Dial to create the connections, then do Read/Write on
the connections themselves from there.

```go
// PrintBody sends a generic GET request to the server at url and prints
// the body of the response to std Out
func PrintBody(url string) error {
	conn, err := net.Dial("tcp", url)
	if err != nil {
		return err
	}

	_, err = conn.Write([]byte("GET / HTTP/1.0\r\n\r\n"))
	if err != nil {
		return err
	}

	resp, err := bufio.NewReader(conn).ReadString('\n')
	if err != nil {
		return err
	}

	fmt.Println(resp)

	return nil
}
```

So, now I've got a start! The level for this library is around using net.Dial
and `net.Conn`. The Dial opens a connection with the server, and returs it as 
a `net.Conn`. By writing to Conn, you're sending out a request. You can then read
the server's response by reading from the Conn. 

Right now, however, the way this works is by carefully selecting a url. The url from the
first example has to be changed to `"[httpbin.org]:80"` in order to work. As most
people would like to enter a url, and not deal with port numbers, host names, usernames,
passwords, etc. themselves this information should be either put into the url or in the
request.

Also, more often than not, an http client will want to send more than just 
`"GET / HTTP/1.0\r\n\r\n"`. In fact, sending just this gets a 
400: Bad Request result from the server :P

Alright, let's start with parsing those urls. Most people will expect to put in a typical
url, so we want it to be of the form http[s]://[www].hostname.ext/file/path/stuff[:port]
where [] denotes optional parameters. I created a new page to handle this url.go

```go
package httpClient

import (
	"errors"
	"strings"
)

type protocol uint

const (
	protocolhttp protocol = iota
	protocolhttps
	protocolUnsupported
)

type url struct {
	scheme   protocol
	hostname string
	path     string
	port     string
}

func parseURL(in string) (url, error) {
	u := url{}
	if strings.HasPrefix(in, "http://") {
		u.scheme = protocolhttp
		in = in[7:]
	} else if strings.HasPrefix(in, "https://") {
		u.scheme = protocolhttps
		in = in[8:]
	} else {
		u.scheme = protocolUnsupported
		return u, errors.New("unsupported http protocol")
	}

	i := 0
	for i < len(in) {
		if in[i] == '/' {
			break
		}
		u.hostname += string(in[i])
		i++
	}

	for i < len(in) {
		if in[i] == ':' {
			u.port = in[i+1:]
			break
		}
		u.path += string(in[i])
		i++
	}

	if u.port == "" {
		u.port = "80"
	}

	return u, nil
}
```

All this really does is loop through the passed url and separate out the protocol,
a file path, and port number. If the port number isn't passed, it assumes it's
port 80, which is the standard web server port. I'll probably have to parse the file path
further down the road for Post and/or Put requests, but that'll look pretty similar,
just looking for a '?' and then putting the values into a map[string]string. There's also
stuff for url encoding of files and other things that could be done. The standard can
be seen [here](https://url.spec.whatwg.org), and a true http client would implement 
everything.

Now, to create the proper headers. [MDN](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers)
has a great list of the header file bits and what they do. Using ones relevent to
my GET request, my function ends up like this

```go
func PrintBody(url string) error {
	u, err := parseURL(url)
	if err != nil {
		return err
	}

	header := "GET " + u.path + " HTTP/1.1\r\n"
	header += "Host: " + u.hostname + "\r\n"
	header += "User-Agent: Noofbizzle\r\n"
	header += "Accept: text/html\r\n"
	header += "Accept-Language: en-us\r\n"
	header += "Accept-Encoding: gzip,deflate\r\n"
	header += "Accept-Charset: ISO-8859-1,utf-8\r\n\r\n"

	conn, err := net.Dial("tcp", "["+u.hostname+"]:"+u.port)
	if err != nil {
		return err
	}
	defer conn.Close()

	_, err = conn.Write([]byte(header))
	if err != nil {
		return err
	}

	resp := bufio.NewReader(conn)
	for {
		line, err := resp.ReadString('\n')
		if err == io.EOF {
			fmt.Print(line)
			break
		}
		if err != nil {
			return err
		}

		fmt.Print(line)
	}

	return nil
}
```

When I run it like this, it works! The server accepts my request and sends me the
proper response. However, it never breaks out of that loop. Unlike in file io, 
a server response never actually sends an EOF. It keeps the connection alive
and waits for further input. To fix this, I'll have to actually parse the header
of the response, find out the content length, and use that in my reader to properly
handle the response.

The for loop now looks like this

```go
	resp := bufio.NewReader(conn)
	respHeader := make(map[string]string)
	// get the header
	for {
		line, err := resp.ReadString('\n')
		if err != nil {
			return err
		}
		if line == "\r\n" {
			break //header ends with an empty line
		}
		if strings.HasPrefix(line, "HTTP/") {
			respHeader["first"] = line
			continue
		}
		nv := strings.Split(line, ": ")
		respHeader[nv[0]] = strings.TrimSuffix(nv[1], "\r\n")
	}

	n, err := strconv.Atoi(respHeader["Content-Length"])
	if err != nil {
		return err
	}

	buf := make([]byte, n)
	_, err = resp.Read(buf)
	if err != nil {
		return err
	}

	fmt.Printf("%s", buf)
```

This now properly returns the body when you make a GET request in the current URL! That's
about how an http GET client works! I'll probably add the bonus suff later down the road,
so keep a look out for that! Remember, the full code is available on 
[github!](https://github.com/Noofbiz/httpClient) Feel free to have a look!