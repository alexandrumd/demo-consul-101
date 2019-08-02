# Demo Consul Intentions
Simplified from the original demo code and microservices from the HashiCorp Consul 101 course - available at https://github.com/hashicorp/demo-consul-101
Scope of this demo is to check how Consul's service mesh and [intentions](https://www.consul.io/docs/connect/intentions.html) work (using the registered service identity rather than IP addresses to enforce ACLs).

## Prerequisites
- download Consul single binary from https://www.consul.io/downloads.html and make sure is available on the `PATH` using the guide [here](https://learn.hashicorp.com/consul/getting-started/install)
- install golang from https://golang.org/doc/install

## Startup consul and services

`cd` to where you have cloned the demo repo and start consul agent on local:
`consul agent -dev -config-dir="./demo-config-localhost" -node=laptop`

You will see in the output that health checks for the services defined in `/demo-config-localhost` (`counting.json` and `dashboard.json`) are failing. You can also go to Consul web UI to check running services http://localhost:8500/ui/dc1/services or by issuing `consul catalog services` via CLI 

Then start instances of counting-service and dashboard-service.
Configuration is supplied via environment variables, here for `PORT` (_be ready to open up multiple terminal windows_)
```
cd services/counting-service
PORT=9003 go run main.go
```
If you go now to http://localhost:9003 you should see some JSON. For example:
```
{"count":1,"hostname":"name.of.host"}
```

Configure the `dashboard-service` using the environment variable `COUNTING_SERVICE_URL` to look for the counting service on localhost:9001.
This ensures that the dashboard service only communicates locally, and relies on a proxy for communication with other services. In a future step, we'll define a proxy to supply this connection.
```
cd services/dashboard-service
PORT=9002 COUNTING_SERVICE_URL="http://localhost:9001" go run main.go
```
Now back in the Consul web UI you will see that the both counting and dashboard service health checks are marked OK.
If you go now to http://localhost:9002/ the dashboard service will report that the "Counting Service is Unreachable".

## Service Mesh with Consul Connect
You will start a proxy for the counting service first. You'll use Consul's built-in proxy, but for production applications you would want to use Envoy, HAProxy, or another proxy application.

The command to set up the counting service has a few important components.
- `service` declares the name of the service being provided by this proxy.
- `service-addr` describes where the service is (most likely on the local machine). We're running the counting-service at port 9003 so that is mentioned here.
- `listen` is an IP address and port that will be used to communicate with other proxies. The IP address cannot be 0.0.0.0 or localhost (or any variation). The port is arbitrary but must not conflict with other ports in use on this node. Pick a large number or a random number that is accessible from the network.
- `register` announces the service to the Consul catalog so it will appear in the Consul Web UI and be discoverable by other Connect proxies.

```
$ consul connect proxy \
      -service=counting \
      -service-addr=127.0.0.1:9003 \
      -listen=$YOUR_PRIVATE_IP:19003 \
      -register

==> Consul Connect proxy starting...
    Configuration mode: Flags
               Service: counting
       Public listener: 10.1.2.211:19003 => 127.0.0.1:9003
```

Finally, start a proxy for the dashboard service that surfaces the upstream counting service at localhost:9001.
- `service` describes the service we represent on this node (dashboard)
- `upstream` identifies the service we want to connect to (counting). It also includes the port number that we can read from on localhost in order to talk to the counting service (9001).
```
consul connect proxy \
       -service=dashboard \
       -upstream="counting:9001"

==> Consul Connect proxy starting...
    Configuration mode: Flags
              Service: dashboard
             Upstream: counting => :9001
      Public listener: Disabled
```
Verify that the services are connected by revisiting the dashboard service (http://localhost:9002/) in your web browser. It should show a valid number and the name of the host it is connected to.

# Enforce access with intentions
Consul Connect intentions define services that are allowed to (or denied from) communicating with other services. This can be done via the web UI, the command line, or the HTTP API. We will use the web UI.
Next, you will configure intentions to allow communication between only the dashboard and counting services.

## Step 1: Deny all by default
First, you should set a default policy of deny all to ensure only authorized services can communicate. Open the Consul Web UI. Go to the "Intentions" screen. Click the "Create" button.
Select * (All Services) from both the source and destination dropdown menus.
Click the DENY radio button and save the intention.
Go back to the dashboard service in your web browser. You should see that "Counting service is unreachable."
You can also check this via CLI by issuing `consul intention check [options] SRC DST` e.g. `consul intention check dashboard counting`

## Step 2: Allow dashboard to connect to counting
In the web UI, create a new intention that allows a source of dashboard to see a destination of counting. Save the intention.
The dashboard will connect and show a positive count again (and the name of the host it connects to).

# Further research
- explore consul DNS, e.g. `dig @127.0.0.1 -p 8600 counting.service.consul SRV`.
- go to https://learn.hashicorp.com/consul
