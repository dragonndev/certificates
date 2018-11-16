# Examples

## Basic client usage

The basic-client example shows the functionality of the `ca.Client` type. The
methods work as an SDK for integrating services with the Certificate Authority (CA).

In [basic-client/client.go](/examples/basic-client/client.go) we see
the initialization of a client:

```go
client, err := ca.NewClient("https://localhost:9000", ca.WithRootSHA256("84a033e84196f73bd593fad7a63e509e57fd982f02084359c4e8c5c864efc27d"))
```

The previous code uses the CA address and the root certificate fingerprint.
The CA url will be present in the token, and the root fingerprint can be present
too if the `--root root_ca.crt` option is used in the creation of the token. If
the token does contain the root fingerprint then it is simpler to use:

```go
client, err := ca.Bootstrap(token)
```

After the initialization there are examples of all the client methods. These
methods are a convenient way to use the CA API. The first method, `Health`,
returns the status of the CA server. If the server is up it will return
`{"status":"ok"}`.

```go
health, err := client.Health()
// Health is a struct created from the JSON response {"status": "ok"}
```

The next method `Root` is used to get and verify the root certificate. We
pass a fingerprint and it downloads the root certificate from the CA and
verifies that the fingerprint matches. This method uses an insecure HTTP
client as it might be used in the initialization of the client, but the response
is considered secure because we have compared against the expected digest.

```go
root, err := client.Root("84a033e84196f73bd593fad7a63e509e57fd982f02084359c4e8c5c864efc27d")
```

Next we have the most important method; `Sign`. `Sign` will authorize and sign a
CSR (Certificate Signing Request) that we provide. To authorize this request we use
a provisioning token issued by an authorized provisioner.
You can build your own certificate request and add it in
the `*api.SignRequest`, but our CA SDK contains a method that will generate a
secure random key and create a CSR - combining the key with the information
provided in the provisioning token.

```go
// Create a CSR from a token and return the SignRequest, the private key,  and an
// error if something failed.
req, pk, err := ca.CreateSignRequest(token)
if err != nil { ... }

// Do the Sign request and return the signed certificate.
sign, err := client.Sign(req)
if err != nil { ... }
```

Next is the `Renew` method which is used to (you guessed it!) renew certificates.
Certificate renewal relies on a mTLS connection with using an existing certificate.
So, as input we will need to pass a transport with the current certificate.

```go
// Get a cancelable context to stop the renewal goroutines and timers.
ctx, cancel := context.WithCancel(context.Background())
defer cancel()
// Create a transport with the sign response and the private key.
tr, err := client.Transport(ctx, sign, pk)
if err != nil { ... }
// Renew the certificate. The return type is equivalent to the Sign method.
renew, err := client.Renew(tr)
if err != nil { ... }
```

The following methods are for inpsecting Provisioners.
One method that returns a list of provisioners or a the encrypted key of one provisioner.

```go
// Without options it will return the first 20 provisioners.
provisioners, err := client.Provisioners()
// We can also set a limit up to 100.
provisioners, err := client.Provisioners(ca.WithProvisionerLimit(100))
// With a pagination cursor.
provisioners, err := client.Provisioners(ca.WithProvisionerCursor("1f18c1ecffe54770e9107ce7b39b39735"))
// Or combine both.
provisioners, err := client.Provisioners(
    ca.WithProvisionerCursor("1f18c1ecffe54770e9107ce7b39b39735"),
    ca.WithProvisionerLimit(100),
)

// Return the encrypted key of one of the returned provisioners. The key
// returned is an encrypted JWE with the private key used to sign tokens.
key, err := client.ProvisionerKey("DmAtZt2EhmZr_iTJJ387fr4Md2NbzMXGdXQNW1UWPXk")
```

The following example shows how to create a
tls.Config object that can be injected into servers and clients. By default these
methods will spin off Go routines that auto-renew a certificate once (approximately)
two thirds of the duration of the certificate has passed.

```go
// Get a cancelable context to stop the renewal goroutines and timers.
ctx, cancel := context.WithCancel(context.Background())
defer cancel()
// Get tls.Config for a server.
tlsConfig, err := client.GetServerTLSConfig(ctx, sign, pk)
// Get tls.Config for a client.
tlsConfig, err := client.GetClientTLSConfig(ctx, sign, pk)
// Get an http.Transport for a client; this can be used as a http.RoundTripper
// in an http.Client.
tr, err := client.Transport(ctx, sign, pk)
```

To run the example you need to start the certificate authority:

```sh
certificates $ bin/step-ca examples/pki/config/ca.json
2018/11/02 18:29:25 Serving HTTPS on :9000 ...
```

Then run client.go with a new token:
```sh
certificates $ export STEPPATH=examples/pki
certificates $ export STEP_CA_URL=https://localhost:9000
certificates $ go run examples/basic-client/client.go $(step ca token client.smallstep.com)
```

## Bootstrap Client & Server

In this example we are going run the CA alongside a simple Server using TLS and
a simple client making TLS requests to the server.

The examples directory already contains a sample pki configuration with the
password `password` hardcoded, but you can create your own using `step ca init`.

These examples show the use of some other helper methods - simple ways to
create TLS configured http.Server and http.Client objects. The methods are
`BootstrapServer`, `BootstrapServerWithMTLS` and `BootstrapClient`.

```go
// Get a cancelable context to stop the renewal goroutines and timers.
ctx, cancel := context.WithCancel(context.Background())
defer cancel()
// Create an http.Server
srv, err := ca.BootstrapServer(ctx, token, &http.Server{
    Addr: ":8443",
    Handler: handler,
})
if err != nil {
    panic(err)
}
srv.ListenAndServeTLS("", "")
```

```go
// Get a cancelable context to stop the renewal goroutines and timers.
ctx, cancel := context.WithCancel(context.Background())
defer cancel()
// Create an http.Server that requires a client certificate
srv, err := ca.BootstrapServerWithMTLS(ctx, token, &http.Server{
    Addr: ":8443",
    Handler: handler,
})
if err != nil {
    panic(err)
}
srv.ListenAndServeTLS("", "")
```

```go
// Get a cancelable context to stop the renewal goroutines and timers.
ctx, cancel := context.WithCancel(context.Background())
defer cancel()
// Create an http.Client
client, err := ca.BootstrapClient(ctx, token)
if err != nil {
    panic(err)
}
resp, err := client.Get("https://localhost:8443")
```

We will demonstrate the mTLS configuration in a different example. In this
examplefor we will configure the server to only verify client certificates
if they are provided.

To being with let's start the Step CA:

```sh
certificates $ bin/step-ca examples/pki/config/ca.json
2018/11/02 18:29:25 Serving HTTPS on :9000 ...
```

Next we will start the bootstrap-server and enter `password` prompted for the
provisioner password:

```sh
certificates $ export STEPPATH=examples/pki
certificates $ export STEP_CA_URL=https://localhost:9000
certificates $ go run examples/bootstrap-server/server.go $(step ca token localhost)
✔ Key ID: DmAtZt2EhmZr_iTJJ387fr4Md2NbzMXGdXQNW1UWPXk (mariano@smallstep.com)
Please enter the password to decrypt the provisioner key:
Listening on :8443 ...
```

Let's try to cURL our new bootstrap server with the system certificates bundle
as our root. It should fail.
```
certificates $ curl https://localhost:8443
curl: (60) SSL certificate problem: unable to get local issuer certificate
More details here: https://curl.haxx.se/docs/sslcerts.html

curl performs SSL certificate verification by default, using a "bundle"
 of Certificate Authority (CA) public keys (CA certs). If the default
 bundle file isn't adequate, you can specify an alternate file
 using the --cacert option.
If this HTTPS server uses a certificate signed by a CA represented in
 the bundle, the certificate verification probably failed due to a
 problem with the certificate (it might be expired, or the name might
 not match the domain name in the URL).
If you'd like to turn off curl's verification of the certificate, use
 the -k (or --insecure) option.
HTTPS-proxy has similar options --proxy-cacert and --proxy-insecure.
```

Now lets use the root certificate generated for the Step PKI. It should work.

```sh
certificates $ curl --cacert examples/pki/secrets/root_ca.crt https://localhost:8443
Hello nobody at 2018-11-03 01:49:25.66912 +0000 UTC!!!
```

Notice that in the response we see `nobody`. This is because the server did not
detected a TLS client configuration.

But if we create a client with it's own certificate (generated by the Step CA),
we should see the Common Name of the client certificate:

```sh
certificates $ export STEPPATH=examples/pki
certificates $ export STEP_CA_URL=https://localhost:9000
certificates $ go run examples/bootstrap-client/client.go $(step ca token Mike)
✔ Key ID: DmAtZt2EhmZr_iTJJ387fr4Md2NbzMXGdXQNW1UWPXk (mariano@smallstep.com)
Please enter the password to decrypt the provisioner key:
Server responded: Hello Mike at 2018-11-03 01:52:52.678215 +0000 UTC!!!
Server responded: Hello Mike at 2018-11-03 01:52:53.681563 +0000 UTC!!!
Server responded: Hello Mike at 2018-11-03 01:52:54.682787 +0000 UTC!!!
...
```

## Bootstrap mTLS Client & Server

This example demonstrates a stricter configuration of the bootstrap-server. Here
we configure the server to require mTLS (mutual TLS) with a valid client certificate.

As always, we begin by starting the CA:

```sh
certificates $ bin/step-ca examples/pki/config/ca.json
2018/11/02 18:29:25 Serving HTTPS on :9000 ...
```

Next we start the mTLS server and we enter `password` when prompted for the
provisioner password:

```sh
certificates $ export STEPPATH=examples/pki
certificates $ export STEP_CA_URL=https://localhost:9000
certificates $ go run examples/bootstrap-mtls-server/server.go $(step ca token localhost)
✔ Key ID: DmAtZt2EhmZr_iTJJ387fr4Md2NbzMXGdXQNW1UWPXk (mariano@smallstep.com)
Please enter the password to decrypt the provisioner key:
Listening on :8443 ...
```

Now that the server is configured to require mTLS cURL-ing should fail even
if we use the correct root certificate bundle.

```sh
certificates $ curl --cacert examples/pki/secrets/root_ca.crt https://localhost:8443
curl: (35) error:1401E412:SSL routines:CONNECT_CR_FINISHED:sslv3 alert bad certificate
```

However, if we use our client (which requests a certificate from the Step CA
when it starts):

```sh
certificates $ export STEPPATH=examples/pki
certificates $ export STEP_CA_URL=https://localhost:9000
certificates $ go run examples/bootstrap-client/client.go $(step ca token Mike)
✔ Key ID: DmAtZt2EhmZr_iTJJ387fr4Md2NbzMXGdXQNW1UWPXk (mariano@smallstep.com)
Please enter the password to decrypt the provisioner key:
Server responded: Hello Mike at 2018-11-07 21:54:00.140022 +0000 UTC!!!
Server responded: Hello Mike at 2018-11-07 21:54:01.140827 +0000 UTC!!!
Server responded: Hello Mike at 2018-11-07 21:54:02.141578 +0000 UTC!!!
...
```

## Certificate rotation

We can use the bootstrap-server to demonstrate certificate rotation. We've
added a second provisioner, named `mike@smallstep.com`, to the CA configuration.
This provisioner is has a default certificate duration of 2 minutes.
Let's run the server, and inspect the certificate. We can should be able to
see the certificate rotate once approximately 2/3rds of it's lifespan has passed.

```sh
certificates $ export STEPPATH=examples/pki
certificates $ export STEP_CA_URL=https://localhost:9000
certificates $ go run examples/bootstrap-server/server.go $(step ca token localhost)
✔ Key ID: YYNxZ0rq0WsT2MlqLCWvgme3jszkmt99KjoGEJJwAKs (mike@smallstep.com)
Please enter the password to decrypt the provisioner key:
Listening on :8443 ...
```

In this case, the certificate will rotate after 74-80 seconds.
The exact formula is `<duration>-<duration>/3-rand(<duration>/20)` (`duration=120`
in our example).

We can use the following command to check the certificate expiration and to make
sure the certificate  changes after 74-80 seconds.

```sh
certificates $ step certificate inspect --insecure https://localhost:8443
```

## NGINX with Step CA certificates

The example under the `docker` directory shows how to combine the Step CA
with NGINX to serve or proxy services using certificates created by the
Step CA.

This example creates 3 different docker images:

* nginx-test: docker image with NGINX and a script using inotify-tools to watch
  for changes in the certificate to reload NGINX.
* step-ca-test: docker image with the Step CA
* step-renewer-test: docker image with the step cli tool - it creates the
  certificate and sets a cron that renews the certificate (the cron
  runs every minute for testing purposes).

To run this test you need to have the docker daemon running. With docker running
swith to the `examples/docker directory` and run `make`:

```
certificates $ cd examples/docker/
docker $ make
GOOS=linux go build -o ca/step-ca github.com/smallstep/certificates/cmd/step-ca
GOOS=linux go build -o renewer/step github.com/smallstep/cli/cmd/step
docker build -t nginx-test:latest nginx
...
docker-compose up
WARNING: The Docker Engine you're using is running in swarm mode.

Compose does not use swarm mode to deploy services to multiple nodes in a swarm. All containers will be scheduled on the current node.

To deploy your application across the swarm, use `docker stack deploy`.

Creating network "docker_default" with the default driver
Creating docker_ca_1 ... done
Creating docker_renewer_1 ... done
Creating docker_nginx_1   ... done
Attaching to docker_ca_1, docker_renewer_1, docker_nginx_1
ca_1       | 2018/11/12 19:39:16 Serving HTTPS on :443 ...
nginx_1    | Setting up watches.
nginx_1    | Watches established.
...
```

Make will build the binaries for step and step-ca, create the images, create the
containers and start them using docker composer.

NGINX will be listening on your local machine on https://localhost:4443, but to
make sure the cert is right we need to add the following entry to `/etc/hosts`:

```
127.0.0.1   nginx
```

Now we can use cURL to verify:

```sh
docker $ curl --cacert ca/pki/secrets/root_ca.crt https://nginx:4443/
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```

We can use `make inspect` to witness the certificate being rotated every minute.

```sh
docker $ make inspect | head
step certificate inspect https://localhost:4443 --insecure
Certificate:
    Data:
        Version: 3 (0x2)
        Serial Number: 220353801925419530569669982276277771655 (0xa5c6993a7e110e6f009c83c79edc1d87)
    Signature Algorithm: ECDSA-SHA256
        Issuer: CN=Smallstep Intermediate CA
        Validity
            Not Before: Nov 10 02:13:00 2018 UTC
            Not After : Nov 11 02:13:00 2018 UTC
docker $ make inspect | head
step certificate inspect https://localhost:4443 --insecure
Certificate:
    Data:
        Version: 3 (0x2)
        Serial Number: 207756171799719353821615361892302471392 (0x9c4c621c04d3e8be401ff0d14c5440e0)
    Signature Algorithm: ECDSA-SHA256
        Issuer: CN=Smallstep Intermediate CA
        Validity
            Not Before: Nov 10 02:14:00 2018 UTC
            Not After : Nov 11 02:14:00 2018 UTC
```

Finally, to cleanup the containers and volumes created in this demo use `make down`:

```sh
docker $ make down
docker-compose down
Stopping docker_nginx_1   ... done
Stopping docker_renewer_1 ... done
Stopping docker_ca_1      ... done
Removing docker_nginx_1   ... done
Removing docker_renewer_1 ... done
Removing docker_ca_1      ... done
Removing network docker_default
```
