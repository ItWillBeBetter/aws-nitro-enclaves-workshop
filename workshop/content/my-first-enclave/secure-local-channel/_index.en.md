+++
title = "Secure local channel"
chapter = false
weight = 13
+++

In this section, you will set up communication between the Nitro Enclave and an external endpoint. Out of the box, Nitro Enclaves can communicate only with parent instance and only through secure local communication channel - vsock. To allow the Nitro Enclave to communicate with the external endpoint, you will run a proxy server that runs on the host instance and Traffic-Forwarder in the Nitro Enclave. They will forward traffic from an Enclave coming through vsock to an external endpoint.

In this example, you will call a sample server application running inside the Nitro Enclave and configure a forwarder with vsock-proxy to allow the enclave to reach out to the external website.
The client application will be reaching out to a server, running inside the Nitro Enclave, and listening on port `5005`. Local loopback will forward all traffic from the server to the Traffic-Forwarder (on port `443`) and from there to the vsock-proxy (on port `8001`). The vsock-proxy routes the traffic to the target external endpoint. And rolling back response will trickle back to the client application. Both types of calls (client-server and enclave-endpoint) will go through the vsock socket.

![Architecture diagram](/images/secure-local-channel-arch.png)


{{% notice info %}}
**What is vsock, and how do I use it to communicate with an enclave?** - Vsock is a type of socket interface that is defined by a context ID (CID) and port number. The context ID is parallel to an IP address in a TCP/IP connection. Vsock utilizes standard, well-defined POSIX Sockets APIs (e.g., connect, listen, accept) to communicate with an enclave. Applications can use these APIs to communicate natively over vsock, or send HTTP requests over vsock through a proxy.  
**Vsock socket** - Vsock is a local communication channel between a parent instance and an enclave. It is the only channel of communication that an enclave can use to interact with external services. An enclave's vsock address is defined by a context identifier (CID) that you can set when launching an enclave. The CID used by the parent instance is always 3.  
**Vsock-Proxy** - Nitro enclave uses the vsock proxy to call the external endpoint through the parent instance's networking. In this workshop, you will use a vsock-proxy implementation that comes with the Nitro CLI. The example in this module will focus on reaching out to a custom endpoint, while later modules will use it to perform AWS KMS operations (`kms-decrypt`, `kms-generate-data-key`, and `kms-generate-random`) using the Nitro Enclaves SDK. Sessions with KMS are established logically between AWS KMS and the enclave itself, and all session traffic is protected from the parent instance.
{{% /notice %}}


### Review Source Code

But first, let's review provided sample code. Please navigate to the folder with code:

```sh
$ cd ~/environment/aws-nitro-enclaves-workshop/resources/code/my-first-enclave/secure-local-channel/
```

Look into the source code of the example. You can see that it only contains `Dockerfile` that describes target image, `traffic_forwarder.py`, `server.py` that contains "business logic" for our application, `run.sh` that setups and runs both server and forwarder, and `client.py` that will be calling the server from outside of the enclave.
 
Traffic-Forwarder is a custom code solution provided as-is with this workshop. It runs as a separate component in a Nitro Enclave and re-routes all the traffic from inside the Nitro Enclave to a vsock-proxy or other endpoints. 

You can find the complete code of Traffic-Forwarder in `code/traffic_forwarder.py`. It configures local port forwarding by:
1. Assign an IP address to the local loopback 
1. Add a hosts record, pointing target site call to local loopback.

This configuration will allow Traffic-Forwarder to route the traffic to vsock-proxy.


### Run Proxy
In the Prerequisites section, you already built and configured a vsock-proxy. You can validate if it is available in the system by running this command:
```sh
$ vsock-proxy --help
```
Take a look at the parameters that you can use to run the vsock-proxy.

{{% notice info %}}
Outside of this workshop, you can install CLI and vsock-proxy both from source code and the Amazon Linux repository. See [Nitro Cli Github Repo](https://github.com/aws/aws-nitro-enclaves-cli) for more details.
{{% /notice %}}

By default, the provided vsock-proxy allows routing of traffic only to port 443 of different KMS endpoints in regions all over the globe. The config file specifies allow list, that you can see at `/etc/vsock_proxy/config.yaml`. But for our example, we will point the proxy to a different website. You can use the `vsock-proxy.yaml` with sample code or create your custom file by running these commands:

```sh
$ echo "allowlist:" >> your-vsock-proxy.yaml
$ echo "- {address: ip-ranges.amazonaws.com, port: 443}" >> your-vsock-proxy.yaml
```

Such config file will allow the proxy to route traffic to `ip-ranges.amazonaws.com` on port 443.

Start proxy with the next command (you can pass the name of your config file as a parameter.):

```sh
$ vsock-proxy 8001 ip-ranges.amazonaws.com 443 --config your-vsock-proxy.yaml
```

### Run Server Application 
With vsock-proxy running in the current terminal window, please create a new terminal window to run the server application, navigate to `secure-local-channel`.

```sh
$ cd ~/environment/aws-nitro-enclaves-workshop/resources/code/my-first-enclave/secure-local-channel/
```
Remembering steps, that you learned in previous sections, let's build the server application. Please run the script below to: build your docker image -> create Enclave Imabe File -> Run new Nitro Enclave in debug-mode -> Connect to debug console:
```sh
$ docker build ./ -t secure-channel-example
$ nitro-cli build-enclave --docker-uri secure-channel-example:latest --output-file secure-channel-example.eif
$ nitro-cli run-enclave --cpu-count 2 --memory 2048 --eif-path secure-channel-example.eif --debug-mode
$ ENCLAVE_ID=`nitro-cli describe-enclaves | jq -r ".[0].EnclaveID"`
$ [ "$ENCLAVE_ID" != "null" ] && nitro-cli console --enclave-id ${ENCLAVE_ID}
```

{{% notice info %}}
If you encounter that your enclave requires more available memory, you will have to configure `/etc/nitro_enclaves/allocator.yaml` and restart allocator services. See [Nitro CLI Documentation](https://github.com/aws/aws-nitro-enclaves-cli). for more details.
{{% /notice %}}

### Run Client Application
With the vsock-proxy and server application running, let's call it from the client application, running on the host. The client application will obtain the list of published IP ranges for AWS S3 service and filter out to the region that you specify as the last parameter in the client call. 

Next comman will run the client application `client.py` and pass Enclave ID, configured port `5005` and region parameter (you can change it from default `us-east-1`). Running the client application should return to you current published IP Rages for S3 service, filtered to the region that you provided. Please open new terminal window and run:

```sh
$ cd ~/environment/aws-nitro-enclaves-workshop/resources/code/my-first-enclave/secure-local-channel/
$ ENCLAVE_ID=`nitro-cli describe-enclaves | jq -r ".[0].EnclaveID"`
$ [ "$ENCLAVE_ID" != "null" ] && python3 client.py client ${ENCLAVE_ID}  5005 "us-east-1"
```


{{% notice info %}}
It is important to note that the client application takes three parameters: `Enclave_CID`, `port`, and `query`. While you control `query` and `port` stay the same between restarts, `Enclave_CID` changes every time you create the Nitro Enclave. You can learn Enclave_Id by running `nitro-cli describe-enclaves`.
{{% /notice %}}

### Summary
You created a vsock-proxy in the parent host that listens to port `8001` and forwards all traffic to a custom HTTPS endpoint. Inside the enclave, you have Traffic-Forwarder accepting all traffic to localhost on port `443` and forwards it to the vsock-proxy.  

### Preparing for the next module
Before going to the next module, please stop existing Nitro Enclaves and restart vsock-proxy as a service with the default configuration (that will point it back to KMS endpoints).

### Stop running enclaves
Before continuing to the next section, Let's terminate created enclave by first describing them and then terminating them:
```sh
$ ENCLAVE_ID=`nitro-cli describe-enclaves | jq -r ".[0].EnclaveID"`
$ [ "$ENCLAVE_ID" != "null" ] && nitro-cli terminate-enclave --enclave-id ${ENCLAVE_ID}
```
### Start vsock-proxy as a service with the default configuration
In the terminal window with vsock-proxy, press `CTRL+C (^C)` to stop vsock-proxy, that your custom config.

After that run the command below to start is as a service with default configuration from `/etc/vsock_proxy/config.yaml`:
```sh
$ sudo systemctl enable nitro-enclaves-vsock-proxy.service
$ sudo systemctl start nitro-enclaves-vsock-proxy.service
```
Now the proxy runs using the default configuration from `/etc/vsock_proxy/config.yaml`, on local port `8000` and the AWS KMS endpoint corresponding to the instance's region.

---
#### Proceed to the [Cryptographic attestation](cryptographic-attestation.html) section to continue the workshop.
