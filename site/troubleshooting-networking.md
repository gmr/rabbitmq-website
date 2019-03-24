<!--
Copyright (c) 2007-2019 Pivotal Software, Inc.

All rights reserved. This program and the accompanying materials
are made available under the terms of the under the Apache License,
Version 2.0 (the "License”); you may not use this file except in compliance
with the License. You may obtain a copy of the License at

https://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.
-->

# Troubleshooting Network Connectivity

## <a id="overview" class="anchor" href="#overview">Overview</a>

This guide accompanies the one [on networking](/networking.html) and focuses on troublshooting of
network connections.

For connections that use TLS there is an additional [guide on troubleshooting TLS](/troubleshooting-ssl.html).

## <a id="methodology" class="anchor" href="#methodology">Methodology</a>

Troubleshooting Client Connection Issues: the Methodology

Troubleshooting of network connectivity issues is a broad topic. There are entire
books written about it. This guide explains a methodology and widely available networking tools
that help narrow most common issues down efficiently.

Networking protocols are [layered](https://en.wikipedia.org/wiki/OSI_model#Comparison_with_TCP.2FIP_model).
So are problems with them. An effective troubleshooting
strategy typically uses the process of elimination to pin point the issue (or multiple issues),
starting at higher levels. Specifically for messaging technologies, the following steps
are often effective and sufficient:

 * [Verify client configuration](#verify-client)
 * [Verify server configuration](#verify-server) using <code>[rabbitmq-diagnostics](/rabbitmq-diagnostics.8.html) listeners</code>,
   <code>[rabbitmqctl](rabbitmqctl.8.html) status</code>, <code>rabbitmqctl environment</code>
 * Inspect [server logs](#server-logs)
 * Verify [hostname resolution](#hostname-resolution)
 * Verify what TCP [port are used and their accessibility](#ports)
 * Verify [IP routing](#ip-routing)
 * If needed, [take and analyze a traffic dump](#traffic-capture) (traffic capture)

These steps, when performed in sequence, usually help identify the root cause of
the vast majority of networking issues. Troubleshooting tools and techniques for
levels lower than the [Internet (networking) layer](https://en.wikipedia.org/wiki/Internet_protocol_suite#Internet_layer)
are outside of the scope of this guide.

Certain problems only happen in environments with a [high degree of connection churn](#detecting-high-connection-churn).
Client connections can be inspected using the [management UI](/management.html).
It is also possible to [inspect all TCP connections of a node and their state](#inspecting-connections).
That information collected over time, combined with server logs, will help detect connection churn,
file descriptor exhaustion and related issues.

## <a id="verify-client" class="anchor" href="#verify-client">Verify Client Configuration</a>

All developers and operators have been there: typos,
outdated values, issues in provisioning tools, mixed up
public and private key paths, and so on. Step one is to
double check application and client library
configuration.

## <a id="verify-server" class="anchor" href="#verify-server">Verify Server Configuration</a>

Verifying server configuration helps prove that RabbitMQ is running
with the expected set of settings related to networking. It also verifies
that the node is actually running. Here are the recommended steps:

 * Make sure the node is running using <code>[rabbitmqctl](/rabbitmqctl.8.html) status</code>
 * Verify [config file is correctly placed and has correct syntax/structure](/configure.html#configuration-files)
 * Inspect listeners using <code>[rabbitmq-diagnostics](/rabbitmq-diagnostics.8.html) listeners</code>
   or the <code>listeners</code> section in <code>[rabbitmqctl](/rabbitmqctl.8.html) status</code>
 * Inspect effective configuration using <code>[rabbitmqctl](/rabbitmqctl.8.html) environment</code>

The listeners sections will look something like this:

<pre class="lang-ini">
Interface: [::], port: 25672, protocol: clustering, purpose: inter-node and CLI tool communication
Interface: [::], port: 5672, protocol: amqp, purpose: AMQP 0-9-1 and AMQP 1.0
Interface: [::], port: 5671, protocol: amqp/ssl, purpose: AMQP 0-9-1 and AMQP 1.0 over TLS
Interface: [::], port: 15672, protocol: http, purpose: HTTP API
Interface: [::], port: 15671, protocol: https, purpose: HTTP API over TLS (HTTPS)
Interface: [::], port: 1883, protocol: mqtt, purpose: MQTT
</pre>

With <code>rabbitmqctl status</code> it will look like so:

<pre class="lang-erlang">
% ...
{listeners,
   [{clustering,25672,"::"},
    {amqp,5672,"::"},
    {'amqp/ssl',5671,"::"},
    {http,15672,"::"}]}
% ...
</pre>

In this example, there are 4 TCP listeners on the node:

 * Inter-node and CLI tool communication port, <code>25672</code>
 * AMQP 0-9-1 (and 1.0, if enabled) listener for non-TLS connections, <code>5672</code>
 * AMQP 0-9-1 (and 1.0, if enabled) listener for TLS-enabled connections, <code>5671</code>
 * [HTTP API](/management.html), 15672

All listeners are bound to all available interfaces.

Inspecting TCP listeners used by a node helps spot non-standard port configuration,
protocol plugins (e.g. [MQTT](/mqtt.html)) that are supposed to be configured but aren't,
cases when the node is limited to only a few network interfaces, and so on. If a port is not on the
listener list it means the node cannot accept any connections on it.

## <a id="server-logs" class="anchor" href="#server-logs">Inspect Server Logs</a>

RabbitMQ nodes will [log](/logging.html) key
client [connection lifecycle events](/logging.html#connection-lifecycle-events).
A TCP connection must be successfully established and at least 1 byte of data must be
sent by the peer for a connection to be considered (and
logged as) accepted. If no events are logged, this means
that either there were no successful TCP connections or they
sent no data.

## <a id="hostname-resolution" class="anchor" href="#hostname-resolution">Hostname Resolution</a>

It is very common for applications to use hostnames or URIs with hostnames when connecting
to RabbitMQ. [dig](https://en.wikipedia.org/wiki/Dig_(command)) and [nslookup](https://en.wikipedia.org/wiki/Nslookup) are
commonly used tools for troubleshooting hostnames resolution.

## <a id="ports" class="anchor" href="#ports">Port Access</a>

Besides hostname resolution and IP routing issues,
TCP port inaccessibility for outside connections is a common reason for
failing client connections. [telnet](https://en.wikipedia.org/wiki/Telnet) is a commonly
used, very minimalistic tool for testing TCP connections to a particular hostname and port.

The following example uses <code>telnet</code> to connect to host <code>localhost</code> on port <code>5672</code>.
There is a running node with stock defaults running on <code>localhost</code> and nothing blocks access to the port, so
the connection succeeds. <code>12345</code> is then entered for input followed by Enter. Since <code>12345</code> is not a correct AMQP protocol header,
so the server closes TCP connection:

<pre class="lang-bash">
telnet localhost 5672
# => Trying ::1...
# => Connected to localhost.
# => Escape character is '^]'.
12345 # enter this and hit Enter to send
# => AMQP	Connection closed by foreign host.
</pre>

After <code>telnet</code> connection succeeds, use <code>Control + ]</code> and then <code>Control + D</code> to
quit it.

The following example connects to <code>localhost</code> on port <code>5673</code>.
The connection fails (refused by the OS) since there is no process listening on that port.

<pre class="lang-bash">
telnet localhost 5673
# => Trying ::1...
# => telnet: connect to address ::1: Connection refused
# => Trying 127.0.0.1...
# => telnet: connect to address 127.0.0.1: Connection refused
# => telnet: Unable to connect to remote host
</pre>

Failed or timing out <code>telnet</code> connections
strongly suggest there's a proxy, load balancer or firewall
that blocks incoming connections on the target port. It
could also be due to RabbitMQ process not running on the
target node or uses a non-standard port. Those scenarios
should be eliminated at the step that double checks server
listener configuration.

There's a great number of firewall, proxy and load balancer tools and products.
[iptables](https://en.wikipedia.org/wiki/Iptables) is a commonly used
firewall on Linux and other UNIX-like systems. There is no shortage of <code>iptables</code>
tutorials on the Web.

Open ports, TCP and UDP connections of a node can be inspected using [netstat](https://en.wikipedia.org/wiki/Netstat),
[ss](https://linux.die.net/man/8/ss), [lsof](https://en.wikipedia.org/wiki/Lsof).

The following example uses <code>lsof</code> to display OS processes that listen on port 5672 and use IPv4:

<pre class="lang-ini">
sudo lsof -n -i4TCP:5672 | grep LISTEN
</pre>

Similarly, for programs that use IPv6:

<pre class="lang-ini">
sudo lsof -n -i6TCP:5672 | grep LISTEN
</pre>

On port 1883:

<pre class="lang-ini">
sudo lsof -n -i4TCP:1883 | grep LISTEN
</pre>

<pre class="lang-ini">
sudo lsof -n -i6TCP:1883 | grep LISTEN
</pre>


If the above commands produce no output then no local OS processes listen on the given port.

The following example uses <code>ss</code> to display listening TCP sockets that use IPv4 and their OS processes:

<pre class="lang-ini">
sudo ss --tcp -f inet --listening --numeric --processes
</pre>

Similarly, for TCP sockets that use IPv6:

<pre class="lang-ini">
sudo ss --tcp -f inet6 --listening --numeric --processes
</pre>

For the list of ports used by RabbitMQ and its various
plugins, see above. Generally all ports used for external
connections must be allowed by the firewalls and proxies.

<code>rabbitmq-diagnostics listeners</code> and <code>rabbitmqctl status</code> can be
used to list enabled listeners and their ports on a RabbitMQ node.

## <a id="ip-routing" class="anchor" href="#ip-routing">IP Routing</a>

Messaging protocols supported by RabbitMQ use TCP and require IP routing between
clients and RabbitMQ hosts to be functional. There are several tools and techniques
that can be used to verify IP routing between two hosts. [traceroute](https://en.wikipedia.org/wiki/Traceroute) and [ping](https://en.wikipedia.org/wiki/Ping_(networking_utility))
are two common options available for many operating systems. Most routing table inspection tools are OS-specific.

Note that both <code>traceroute</code> and <code>ping</code> use [ICMP](https://en.wikipedia.org/wiki/Internet_Control_Message_Protocol)
while RabbitMQ client libraries and inter-node connections use TCP.
Therefore a successful <code>ping</code> run alone does not guarantee successful client connectivity.

Both <code>traceroute</code> and <code>ping</code> have Web-based and GUI tools built on top.

## <a id="traffic-capture" class="anchor" href="#traffic-capture">Capturing Traffic</a>

All network activity can be inspected, filtered and analyzed using a traffic capture.

[tcpdump](https://en.wikipedia.org/wiki/Tcpdump) and its GUI sibling [Wireshark](https://www.wireshark.org)
are the industry standards for capturing traffic, filtering and analysis. Both support all protocols supported by RabbitMQ.
See the [Using Wireshark with RabbitMQ](/amqp-wireshark.html) guide for an overview.

## <a id="tls" class="anchor" href="#tls">TLS Connections</a>

For connections that use TLS there is a separate [guide on troubleshooting TLS](/troubleshooting-ssl.html).

When adopting TLS it is important to make sure that clients
use correct port to connect (see the list of ports above)
and that they are instructed to use TLS (perform TLS
upgrade). A client that is not configured to use TLS will
successfully connect to a TLS-enabled server port but its connection
will then time out since it never performs the TLS upgrade that the server
expects.

A TLS-enabled client connecting to a non-TLS enabled port will successfully
connect and try to perform a TLS upgrade which the server does not expect, this
triggering a protocol parser exception. Such exceptions will be logged by the server.

## <a id="inspecting-connections" class="anchor" href="#inspecting-connections">Inspecting Connections</a>

Open ports, TCP and UDP connections of a node can be inspected using [netstat](https://en.wikipedia.org/wiki/Netstat),
[ss](https://linux.die.net/man/8/ss), [lsof](https://en.wikipedia.org/wiki/Lsof).

The following example uses <code>netstat</code> to list all TCP connection sockets regardless of their state and interface.
IP addresses will be displayed as numbers instead of being resolved to domain names. Program names will be printed next
to numeric port values (as opposed to protocol names).

<pre class="lang-bash">
sudo netstat --all --numeric --tcp --programs
</pre>

Both inbound (client, peer nodes, CLI tools) and outgoing (peer nodes,
Federation links and Shovels) connections can be inspected this way.

<code>rabbitmqctl list_connections</code> and management UI can be used to inspect
more connection properties, some of which are RabbitMQ- or messaging protocol-specific:

 * Network traffic flow, both inbound and outbound
 * Messaging (application-level) protocol used
 * Connection virtual host
 * Time of connection
 * Username
 * Number of channels
 * Client library details (name, version, capabilities)
 * Effective heartbeat timeout
 * TLS details

Combining connection information from management UI or CLI tools with those of <code>netstat</code> or <code>ss</code>
can help troubleshoot misbehaving applications, application instances and client libraries.

## <a id="detecting-high-connection-churn" class="anchor" href="#detecting-high-connection-churn">Detecting High Connection Churn</a>

High connection churn (lots of connections opened and closed after a brief
period of time) [can lead to resource exhaustion](/networking.html#dealing-with-high-connection-churn).
It is therefore important to be able to identify such scenarios. <code>netstat</code> and <code>ss</code>
are most popular options for [inspecting TCP connections](troubleshooting-inspecting-connections).
A lot of connections in the <code>TIME_WAIT</code> state is a likely symptom of high connection churn.
Lots of connections in states other than <code>ESTABLISHED</code> also might be a symptom worth investigating.

Evidence of short lived connections can be found in RabbitMQ log files. E.g. here's an example
of such connection that lasted only a few milliseconds:

<pre class="lang-ini">
2018-06-17 16:23:29.851 [info] &lt;0.634.0&gt; accepting AMQP connection &lt;0.634.0&gt; (127.0.0.1:58588 -> 127.0.0.1:5672)
2018-06-17 16:23:29.853 [info] &lt;0.634.0&gt; connection &lt;0.634.0&gt; (127.0.0.1:58588 -> 127.0.0.1:5672): user 'guest' authenticated and granted access to vhost '/'
2018-06-17 16:23:29.855 [info] &lt;0.634.0&gt; closing AMQP connection &lt;0.634.0&gt; (127.0.0.1:58588 -> 127.0.0.1:5672, vhost: '/', user: 'guest')
</pre>