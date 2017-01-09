Guide To Choosing A Ruby App Server
===========
This guide will review what a web server and app server are, the difference between them, why you need an app server, discuss various security concerns, and then provide contexts where you might choose a different app server than the default.

Web Servers
-------
The traditional way to serve sites on the Internet is via a *web server*, for example using Apache or Nginx. Apache is more broadly used and has more features, while Nginx is smaller, faster, and has fewer (but newer) features. Web Servers serve static files (those that sit on the server) and send requests on to other servers or programs. Web apps, of which Rails apps are an example, generate content based on your code. An app server allows your web server to efficiently send requests to your web app, and to receive the content your web app generates.

Neither Apache nor Nginx can serve Ruby web apps straight out-of-the-box. This functionality is provided by an application server. The demands on your server are very different in a development vs production environment, with production requiring significantly higher performance and security than during development. Therefore for production use it is recommended to use a combination of a web server and an app server. They can cooperate by setting up the web server as a *reverse proxy* in front of the app server, or in the case of Passenger by directly plugging the app server into the web server. A reverse proxy setup means that the web server accepts an incoming HTTP request (a request for a webpage) and forwards it to the app server, which also speaks HTTP. The app server sends the HTTP response (the webpage content generated using your code) back to Apache/Nginx, which will forward the response back to the client (e.g. a web browser).

Despite being able to respond to HTTP requests, an app server usually shouldn’t be allowed to directly accept HTTP requests from the internet because they don’t have sufficient concurrency to handle high traffic, adequate security to deal with malicious requests, and raw performance when serving static files. This will be covered in greater detail in the following sections.

App Servers
-------
The default app server as of Rails 5 is Puma, it's what runs your app when you start your web app with `rails server`. There are many additional app servers available today, the most popular is Passenger. Each app server will be described in a later section, and how they differ from each other.

The App Server and the World
-------
All current Ruby app servers speak HTTP; however, Passenger may be directly exposed to the Internet, while Puma may not and must be put behind a reverse proxy web server like Apache or Nginx, for performance and security reasons that will be explained below.

### Why must some app servers be put behind a reverse proxy?
One reason is security. Using Apache or Nginx in front of your app server is good security practice, because they are very mature and can shield the app server from (perhaps maliciously) corrupted requests. The web server can also buffer requests and responses, protecting the app server from "slow clients" - HTTP clients that don't send or accept data very quickly. You don't want your app server to do nothing while waiting for the client to send the full request or to receive the full response, because during that time the app server could be serving a different request.

Another reason is performance. Most app servers can serve static files, but are not particularly good at doing it quickly, whereas Apache and Nginx can do it faster. People typically set up Apache/Nginx to serve static files directly, and forward requests that don't correspond with static files to the app server.

Puma has good protection against slow clients in most cases, however it doesn’t protect websocket connections, and it doesn’t serve static files as quickly as a dedicated web server which is why it was placed on this list.

### Why can some app servers be directly exposed to the Internet?
Passenger integrates with the battle-tested Apache and Nginx web servers and thereby benefits from the protection and features of these web servers, and is therefore considered safe to expose to the Internet.

Application Servers Compared
--------
### Puma
Puma was forked from Mongrel, and has seen extensive improvement since. And as of Rails 5 Puma has been made the default app server for Rails. While Puma is fast, it is recommended to  use a web server configured as a reverse-proxy to speed up static file serving. Puma was intended to be a purely multithreading app server but has also gained a cluster mode for multiprocessing. Puma is written in Ruby, and as such can experience unpredictable GC pauses. Puma, only has minimal documentation. When used with a web server acting as a reverse proxy, both the web server and Puma must be configured separately to proxy only non-static URIs, and minimize proxy overhead. Puma runs one web app per cluster, so you must deploy puma for each web app on your server. Puma's management tooling is fairly simplistic but covers the basics.

### Passenger Open Source
Passenger integrates directly into Apache or Nginx, similar to `mod_php` for Apache. Just like `mod_php` allows Apache to serve PHP apps, Passenger allows Apache or Nginx to easily serve Ruby (and Node.js and Python) apps. Passenger has been designed with an emphasis on stability, reliability and ease-of-use. These qualities are enabled by its out-of-process architecture, its integrated buffering reverse proxy and a powerful set of administration tools. By virtue of its architecture Passenger is able to self-monitor and deal with failures in a robust way, as well as support multiple different programming languages and frameworks. Passenger’s integrated buffering reverse proxy both shields it from slow client attackers and generally improves performance. Passenger is written primarily in C++, and uses a hybrid evented I/O model, and as a result it scales well and has great performance. Passenger can run the Ruby <a href="https://en.wikipedia.org/wiki/Garbage_collection_(computer_science)">garbage collector</a> when it is not handling a request from a client, potentially reducing request times by hundreds of milliseconds. Passenger has good documentation coverage. Passenger can be configured together with Apache or Nginx, and handles most of the details for you. Passenger can handle multiple web apps just as easily as one. Passenger offers comprehensive tools to administer your server.

I/O Concurrency Models
-------
I/O is the task of reading or writing data to or from any device that is not directly on the CPU. It is orders of magnitude slower than non-I/O tasks and therefore involves the processor waiting for the I/O task to complete. Because of this waiting a large amount of I/O can potentially be done concurrently, as the CPU doesn’t have to do extra work while additional I/O tasks wait.
### Single-Threaded Multi-Process
This is traditionally the most popular I/O model for Ruby app servers, partially because many gems were not thread safe for a long time. Each process can handle exactly one request at a time. The web server load-balances between processes. This model is very robust and there is little chance for the programmer to introduce concurrency bugs. However, its I/O concurrency is limited by the number of processes, and its memory use is very high (each process contains an entire <a href="https://en.wikipedia.org/wiki/Ruby_(programming_language)#Implementations">Ruby VM</a> and application with all gems). This model is suitable for fast, short-running workloads. It is however unsuitable for slow, long-running blocking I/O workloads, e.g. workloads involving the calling of HTTP APIs.

Passenger Open Source is a single-threaded multi-process app server, however Passenger will queue requests once it reaches its configured concurrency, and automatically scales the number of running processes based on the load on the web site.

### Purely Multi-Threaded
Nowadays the Ruby ecosystem has excellent multithreading support, so this I/O model has become very viable. Multithreading allows high I/O concurrency, making it suitable for both short-running and long-running blocking I/O workloads. The programmer is more likely to introduce concurrency bugs, but luckily most web frameworks are designed in such a way that makes this very unlikely. One thing to note however is that the MRI Ruby interpreter cannot leverage multiple CPU cores even when there are multiple threads, due to the use of the Global Interpreter Lock (GIL). You can work around this by using multiple multi-threaded processes, because each process can leverage a CPU core.

Puma is a multi-threaded app server which will scale with a lower Ram footprint. However Puma also drops all new requests as soon as it has reached its configured concurrency, and returns errors more often under high load.

Performance
--------
### Your bottleneck is most likely elsewhere
When discussing Ruby app server performance, it is important to keep a few things in mind. Ruby is slow, but that's usually okay because I/O is much slower. For example, if your Ruby web app talks to a database or receives a network request then in most cases the time spent there  already eclipses the time your web app will take performing non-I/O tasks. A similar principle is true for app servers. Even a slow app server will likely be much faster than your Ruby app, so app server performance is mainly a factor once you are serving very large numbers of users simultaneously. No app server will speed up a slow app. A server can only provide tools to work around that slowness, such as caching, load balancing, multiple threads and/or processes, and request timeouts, but you will always see the best returns by speeding up your app (which generally means reducing I/O). If, after profiling (measuring the speed of all the pieces involved in serving requests, in a controlled and scientific manner), you find that a significant proportion of your request latency (time taken before responding) is due to your app server, that is when you should consider choosing a faster app server.
### Scaling
There are two constraints to keep in mind when talking about scaling (serving more requests at the same time). First, the available RAM (memory); and second, the speed and number of CPU cores. The more resources you have, the bigger you can scale. In Ruby apps, the RAM usage is almost always the source of the bottleneck that prevents scaling bigger. The amount of RAM being used boils down to your web app and which I/O model you are using. Adding threads requires less additional RAM than adding processes, so a multithreaded app server will usually minimize the RAM usage the most, and therefore will allow the biggest scale.

Comparison Chart
--------

| App Server | Ease of Use | Multithreading | Requires Reverse Proxy | Resiliency under load | Windows Support | Tooling | Other Notable Points |
|------------|-------------|----------------|------------------------|-----------------------|-----------------|---------|----------------------|
|Passenger Open Source | ✔ | ✗ | ✗ | Good | ✗ | Great | Superior Documentation |
|Puma | ✗ | ✔ | ✔ | Poor | ✔ | Good | Default Rails Server |

Conclusion
-------
This guide explained that in order to serve Rails applications, you need an application server, usually in combination with a traditional web server. We compared Puma and Passenger Open Source and examined various security and performance characteristics of each app server. In the end each app server brings something different and you may not need every feature. Depending on your specific web app’s needs, the volume of requests you need to handle, and the amount of time you are able to put into configuring your server there will be an app server that can keep you safe and handle everything that the internet can throw at it.
