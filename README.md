## Edits

Added support for Tokens in Cookies in `websocketproxy.py` on `get_target`


```python
    def get_target(self, target_plugin, path):
        """
        Parses the path, extracts a token, and looks up a target
        for that token using the token plugin. Sets
        target_host and target_port if successful
        """
        # The files in targets contain the lines
        # in the form of token: host:port
		
        token = ''
		
		# Extract the token parameter from cookie
        if 'Cookie' in self.headers:
            cookiestr = self.headers['Cookie']
            cookiestr.replace( 'Cookie: ', '', 1 )
            cookies = cookiestr.split('; ')

            for cookie in cookies:
                cookie = cookie.split('=')
                if 'token' in cookie[0]:
                    token = cookie[1]
                    break

        # Extract the token parameter from url
        if token == '':
            args = parse_qs(urlparse(path)[4]) # 4 is the query from url

            if not 'token' in args or not len(args['token']):
                raise self.server.EClose("Token not present")

            token = args['token'][0].rstrip('\n')

        if token == '':
            raise self.server.EClose( "Empty Token defined" )

        # Search for the token
        result_pair = target_plugin.lookup( token )

        if result_pair is not None:
            return result_pair
        else:
            raise self.server.EClose( "Token '%s' not found" % token )
```


## websockify: WebSockets support for any application/server

websockify was formerly named wsproxy and was part of the
[noVNC](https://github.com/novnc/noVNC) project.

At the most basic level, websockify just translates WebSockets traffic
to normal socket traffic. Websockify accepts the WebSockets handshake,
parses it, and then begins forwarding traffic between the client and
the target in both directions.

### News/help/contact

Notable commits, announcements and news are posted to
<a href="http://www.twitter.com/noVNC">@noVNC</a>

If you are a websockify developer/integrator/user (or want to be)
please join the <a
href="https://groups.google.com/forum/?fromgroups#!forum/novnc">noVNC/websockify
discussion group</a>

Bugs and feature requests can be submitted via [github
issues](https://github.com/novnc/websockify/issues).

If you want to show appreciation for websockify you could donate to a great
non-profits such as: [Compassion
International](http://www.compassion.com/), [SIL](http://www.sil.org),
[Habitat for Humanity](http://www.habitat.org), [Electronic Frontier
Foundation](https://www.eff.org/), [Against Malaria
Foundation](http://www.againstmalaria.com/), [Nothing But
Nets](http://www.nothingbutnets.net/), etc. Please tweet <a
href="http://www.twitter.com/noVNC">@noVNC</a> if you do.

### WebSockets binary data

Starting with websockify 0.5.0, only the HyBi / IETF
6455 WebSocket protocol is supported. There is no support for the older
Base64 encoded data format.


### Encrypted WebSocket connections (wss://)

To encrypt the traffic using the WebSocket 'wss://' URI scheme you need to
generate a certificate and key for Websockify to load. By default, Websockify
loads a certificate file name `self.pem` but the `--cert=CERT` and `--key=KEY`
options can override the file name. You can generate a self-signed certificate
using openssl. When asked for the common name, use the hostname of the server
where the proxy will be running:

```
openssl req -new -x509 -days 365 -nodes -out self.pem -keyout self.pem
```

For a self-signed certificate to work, you need to make your client/browser
understand it. You can do this by installing it as accepted certificate, or by
using that same certificate for a HTTPS connection to which you navigate first
and approve. Browsers generally don't give you the "trust certificate?" prompt
by opening a WSS socket with invalid certificate, hence you need to have it
acccept it by either of those two methods.

If you have a commercial/valid SSL certificate with one ore more intermediate
certificates, concat them into one file, server certificate first, then the
intermediate(s) from the CA, etc. Point to this file with the `--cert` option
and then also to the key with `--key`. Finally, use `--ssl-only` as needed.


### Additional websockify features

These are not necessary for the basic operation.

* Daemonizing: When the `-D` option is specified, websockify runs
  in the background as a daemon process.

* SSL (the wss:// WebSockets URI): This is detected automatically by
  websockify by sniffing the first byte sent from the client and then
  wrapping the socket if the data starts with '\x16' or '\x80'
  (indicating SSL).

* Session recording: This feature that allows recording of the traffic
  sent and received from the client to a file using the `--record`
  option.

* Mini-webserver: websockify can detect and respond to normal web
  requests on the same port as the WebSockets proxy. This functionality
  is activated with the `--web DIR` option where DIR is the root of the
  web directory to serve.

* Wrap a program: see the "Wrap a Program" section below.

* Log files: websockify can save all logging information in a file.
  This functionality is activated with the `--log-file FILE` option
  where FILE is the file where the logs should be saved.

* Authentication plugins: websockify can demand authentication for
  websocket connections and, if you use `--web-auth`, also for normal
  web requests. This functionality is activated with the
  `--auth-plugin CLASS` and `--auth-source ARG` options, where CLASS is
  usually one from auth_plugins.py and ARG is the plugin's configuration.

* Token plugins: a single instance of websockify can connect clients to
  multiple different pre-configured targets, depending on the token sent
  by the client using the `token` URL parameter, or the hostname used to
  reach websockify, if you use `--host-token`. This functionality is
  activated with the `--token-plugin CLASS` and `--token-source ARG`
  options, where CLASS is usually one from token_plugins.py and ARG is
  the plugin's configuration.

### Other implementations of websockify

The primary implementation of websockify is in python. There are
several alternate implementations in other languages available in
our sister repositories [websockify-js](https://github.com/novnc/websockify-js)
(JavaScript/Node.js) and [websockify-other](https://github.com/novnc/websockify-other)
 (C, Clojure, Ruby).

In addition there are several other external projects that implement
the websockify "protocol". See the alternate implementation [Feature
Matrix](https://github.com/novnc/websockify/wiki/Feature_Matrix) for
more information.


### Wrap a Program

In addition to proxying from a source address to a target address
(which may be on a different system), websockify has the ability to
launch a program on the local system and proxy WebSockets traffic to
a normal TCP port owned/bound by the program.

The is accomplished with a small LD_PRELOAD library (`rebind.so`)
which intercepts bind() system calls by the program. The specified
port is moved to a new localhost/loopback free high port. websockify
then proxies WebSockets traffic directed to the original port to the
new (moved) port of the program.

The program wrap mode is invoked by replacing the target with `--`
followed by the program command line to wrap.

    `./run 2023 -- PROGRAM ARGS`

The `--wrap-mode` option can be used to indicate what action to take
when the wrapped program exits or daemonizes.

Here is an example of using websockify to wrap the vncserver command
(which backgrounds itself) for use with
[noVNC](https://github.com/novnc/noVNC):

    `./run 5901 --wrap-mode=ignore -- vncserver -geometry 1024x768 :1`

Here is an example of wrapping telnetd (from krb5-telnetd). telnetd
exits after the connection closes so the wrap mode is set to respawn
the command:

    `sudo ./run 2023 --wrap-mode=respawn -- telnetd -debug 2023`

The `wstelnet.html` page in the [websockify-js](https://github.com/novnc/websockify-js)
project demonstrates a simple WebSockets based telnet client (use
'localhost' and '2023' for the host and port respectively).


### Installing websockify

Download one of the releases or the latest development version, extract
it and run `python setup.py install` as root in the directory where you
extracted the files. Normally, this will also install numpy for better
performance, if you don't have it installed already. However, numpy is
optional. If you don't want to install numpy or if you can't compile it,
you can edit setup.py and remove the `install_requires=['numpy'],` line
before running `python setup.py install`.

Afterwards, websockify should be available in your path. Run
`websockify --help` to confirm it's installed correctly.
