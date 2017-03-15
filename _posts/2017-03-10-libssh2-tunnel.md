---
layout: post
title:  "Reverse SSH tunnel with libssh2"
date:   2017-03-10 22:45:00
tags: tunnel ssh libssh2
---

Recently, I had to fix a problem in a mobile library that uses [libssh2](https://www.libssh2.org/) to open a reverse tunnel with a remote server. The implementation was based in the [tcpip-forward.c](https://github.com/libssh2/libssh2/blob/master/example/tcpip-forward.c) example provided in the libssh2 code package. The problem was that in this example the SSH tunnel is closed at the end of each client connection. This article will show how to keep the tunnel permanently opened.

<!--more-->

## 1. The libssh2 library

libssh2 is a lean client-only C library that implements the SSH2 protocol. Given its small size, stability, and good performance, it's well suitable for embedded systems and is also used in many open source projects like [curl](https://curl.haxx.se/).

Unfortunately, there is not much documentation about how to use the library. Despite the API docs and some example files on the project's [website](https://www.libssh2.org), I didn't find any tutorial or a good blog post about how to use it. So I decided to write this one to share what I have learned about libssh2 :-)

Also, note that there is another open source library for SSH called [libssh](https://www.libssh.org/) which, despite the similar name, has nothing to do with libssh2. This ends up causing a lot of confusion when searching for information in the web.

## 2. Reverse tunnels

SSH is an encrypted network protocol widely used for remote command execution and secure data communication. In a SSH tunnel, a unencrypted traffic is wrapped around the SSH protocol and sent over an encrypted and secure connection.

A reverse tunnel is a technique used to access a server in an internal network which is not accessible from the outside world. For example, a server behind a firewall that denies incoming connections.

To set up a reverse tunnel, you need to have another server which is publicly accessible and have SSH access to it. Then, from your server, you open a SSH connection to the publicly accessible server and tell it to forward all data and connection requests from a specific port to a local port on your server.

This way you can, for example, forward all HTTP requests to the port 4000 on the accessible server to your web application running on port 8080 on your server.


<p align="center">
<img src="/img/reverse-tunnel.png">
</p>


## 3. Permanent reverse tunnel with libssh2

Following, I will show you step-by-step how to create a permanent reverse tunnel with libssh2. As already mentioned, this code was base on the [tcpip-forward.c](https://github.com/libssh2/libssh2/blob/master/example/tcpip-forward.c) example which comes in the libssh2 package.

### 3.1 Initialization
First of all, you should call `libssh2_init` to initialize the libssh2 functions.

{% highlight c %}
rc = libssh2_init(0);
if (rc != 0) {
    fprintf (stderr, "libssh2 initialization failed (%d)\n", rc);
    return 1;
}
{% endhighlight %}

The `0` passed to `libssh2_init` will ask it to also initialize the crypto library. By default, libssh2 will attempt to locate the crypto libraries automatically. The supported crypto libraries are [OpenSSL](https://www.openssl.org), [Libgcrypt](https://www.gnupg.org/), [WinCNG](Windows Vista+), and [mbedTLS](https://tls.mbed.org/).

### 3.2 Connecting to the remote SSH server
Next, you should open a socket connection to the remote SSH server.

{% highlight c %}
const char *server_ip = "127.0.0.1";  /* the remote IP address */

sock = socket(PF_INET, SOCK_STREAM, IPPROTO_TCP);
if (sock == -1) {
    fprintf(stderr, "Error opening socket\n");
    return -1;
}

sin.sin_family = AF_INET;
if (INADDR_NONE == (sin.sin_addr.s_addr = inet_addr(server_ip))) {
    fprintf(stderr, "Invalid remote IP address\n");
    return -1;
}
sin.sin_port = htons(22); /* the SSH port */
if (connect(sock, (struct sockaddr*)(&sin),
      sizeof(struct sockaddr_in)) != 0) {
    fprintf(stderr, "Failed to connect!\n");
    return -1;
}
{% endhighlight %}

### 3.3 Creating a SSH session
The next step is to create and start a SSH session under the socket connection.

{% highlight c %}
LIBSSH2_SESSION *session;

/* Create a session instance */
session = libssh2_session_init();
if(!session) {
    fprintf(stderr, "Could not initialize the SSH session!\n");
    return -1;
}

/* ... start it up. This will trade welcome banners, exchange keys,
 * and setup crypto, compression, and MAC layers
 */
rc = libssh2_session_handshake(session, sock);
if(rc) {
    fprintf(stderr, "Error when starting up SSH session: %d\n", rc);
    return -1;
}
{% endhighlight %}

### 3.4 Authenticating the connection
Once connected, you should authenticate yourself to receive access to several resources such as port forwarding, shell, sftp, and so on.

{% highlight c %}
const char *username = "username";
const char *password = "";

if (libssh2_userauth_password(session, username, password)) {
    fprintf(stderr, "Authentication by password failed.\n");
    goto shutdown;
}
{% endhighlight %}

For simplicity, the code above shows only how to authenticate with password, but libssh2 also supports key-based authentication.

### 3.5 Listening for inbound TCP/IP connections
Now, you can start to listen for inbound TCP/IP connections on a specific port of the remote server.

{% highlight c %}
LIBSSH2_LISTENER *listener = NULL;
const char *remote_listenhost = "localhost";
int remote_wantport = 4000;
int remote_listenport;

fprintf(stderr, "Asking server to listen on remote %s:%d\n",
    remote_listenhost, remote_wantport);

listener = libssh2_channel_forward_listen_ex(session, remote_listenhost,
              remote_wantport, &remote_listenport, 1);
if (!listener) {
  fprintf(stderr, "Could not start the tcpip-forward listener!\n"
          "(Note that this can be a problem at the server!"
          " Please review the server logs.)\n");
  goto shutdown;
}

fprintf(stderr, "Server is listening on %s:%d\n", remote_listenhost,
    remote_listenport);
{% endhighlight %}

Besides the session, the `libssh2_channel_forward_listen_ex` method receives:

- the `remote_listenhost` specifying the address to bind to on the remote host. In this case, the remote host address itself (localhost).
- the `remote_wantport` which specifies the port that we want to listen on the remote server. When `0` is passed, the remote host will select the first available dynamic port.
- the `remote_listenport` which will be populated with the actual port bound on the remote host. In this case, the value will be the same passed on `remote_wantport`.
- and a number indicating the maximum number of pending connections to queue before rejecting further attempts.

New connections will be queued by the library until accepted by `libssh2_channel_forward_accept` as shown in the next step.

### 3.6 Creating a SSH Channel
Whenever the remote server receives a TCP/IP connection request, a new SSH channel is created using the `libssh2_channel_forward_accept`, and the `forward_tunnel` function is called to forward the request to the local service.

{% highlight c %}
while (1) {
    fprintf(stderr, "Waiting for remote connection\n");
    channel = libssh2_channel_forward_accept(listener);
    if (!channel) {
        fprintf(stderr, "Could not accept connection!\n"
                "(Note that this can be a problem at the server!"
                " Please review the server logs.)\n");
        goto shutdown;
    }

    forward_tunnel(session, channel);

    libssh2_channel_free(channel);
}
{% endhighlight %}

`libssh2_channel_forward_accept` is a blocking function. This means that a channel instance will be created (and the next instruction processed) only when the listener enqueues/receives a connection.

### 3.7 Forwarding connection to local port
The `forward_tunnel` function is responsible for reading the content received in the channel and forward it to the local port. It also forwards all local service's responses to the channel.

First, it opens a socket connection to the local port to be able to send and receive data from it:
{% highlight c %}
const char *local_destip = "127.0.0.1"; /* local IP address */
int local_destport = 8080;  /* local port */

forwardsock = socket(PF_INET, SOCK_STREAM, IPPROTO_TCP);
if (forwardsock == -1) {
    fprintf(stderr, "Error opening socket\n");
    goto shutdown;
}

sin.sin_family = AF_INET;
sin.sin_port = htons(local_destport);
if (INADDR_NONE == (sin.sin_addr.s_addr = inet_addr(local_destip))) {
    fprintf(stderr, "Invalid local IP address\n");
    goto shutdown;
}
if (connect(forwardsock, (struct sockaddr*) &sin,
      sizeof(struct sockaddr_in)) != 0) {
    fprintf(stderr, "Failed to connect!\n");
    goto shutdown;
}
{% endhighlight %}

Next, it starts a loop where it first reads (`recv`) from the `forwardsock`, saves the content in a buffer, and then writes all buffer's content in the channel (`libssh2_channel_write`). Another while loop inside the first one does the opposite. It reads from the channel (`libssh2_channel_read`) and writes (`send`) the content to the local socket.

{% highlight c %}
/* Setting session to non-blocking IO */
libssh2_session_set_blocking(session, 0);

while (1) {
    FD_ZERO(&fds);
    FD_SET(forwardsock, &fds);
    tv.tv_sec = 0;
    tv.tv_usec = 100000;
    rc = select(forwardsock + 1, &fds, NULL, NULL, &tv);
    if (-1 == rc) {
        fprintf(stderr, "Socket not ready!\n");
        goto shutdown;
    }
    if (rc && FD_ISSET(forwardsock, &fds)) {
        len = recv(forwardsock, buf, sizeof(buf), 0);
        if (len < 0) {
            fprintf(stderr, "Error reading from the forwardsock!\n");
            goto shutdown;
        } else if (0 == len) {
            fprintf(stderr, "The local server at %s:%d disconnected!\n",
                local_destip, local_destport);
            goto shutdown;
        }
        wr = 0;
        do {
            i = libssh2_channel_write(channel, buf, len);
            if (i < 0) {
                fprintf(stderr, "Error writing on the SSH channel: %d\n", i);
                goto shutdown;
            }
            wr += i;
        } while(i > 0 && wr < len);
    }
    while (1) {
        len = libssh2_channel_read(channel, buf, sizeof(buf));
        if (LIBSSH2_ERROR_EAGAIN == len)
            break;
        else if (len < 0) {
            fprintf(stderr, "Error reading from the SSH channel: %d\n", (int)len);
            goto shutdown;
        }
        wr = 0;
        while (wr < len) {
            i = send(forwardsock, buf + wr, len - wr, 0);
            if (i <= 0) {
                fprintf(stderr, "Error writing on the forwardsock!\n");
                goto shutdown;
            }
            wr += i;
        }
        if (libssh2_channel_eof(channel)) {
            fprintf(stderr, "The remote client at %s:%d disconnected!\n",
                remote_listenhost, remote_listenport);
            goto shutdown;
        }
    }
}

shutdown:
    close(forwardsock);
    /* Setting the session back to blocking IO */
    libssh2_session_set_blocking(session, 1);
    return rc;
}
{% endhighlight %}

One important point to be noticed is the calls to `libssh2_session_set_blocking`. According to the libssh2 API doc:

> This function is responsible for setting the session's block mode. If a read is performed on a session with no data currently available, a blocking session will wait for data to arrive and return what it receives. A non-blocking session will return immediately with an empty buffer. If a write is performed on a session with no room for more data, a blocking session will wait for room. A non-blocking session will return immediately without writing anything.

So in this case, I'm setting the session to non-blocking before the while loop because I don't want it to be blocked on reading or writing operations.

When the client disconnects (`libssh2_channel_eof`), the `shutdown` instructions will be called. The `forwardsock` is then closed and the session is set back to the blocking mode. This way the `libssh2_channel_forward_accept` will be blocked on waiting for a new connection.

## 4. Source code

You can found the complete code for the reverse tunnel in the [libssh2-tunnel-example](https://github.com/marianafranco/libssh2-tunnel-example) repository. Enjoy! :D
