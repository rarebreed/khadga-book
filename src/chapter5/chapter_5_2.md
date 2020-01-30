# Letencrypt TLS certs

If you have an https server, you'll need a real TLS server.  Creating self-signed certs is ok for
testing purposes, but even then, unless you have good automation and CI/CD pipelines, there's a
danger of accidentally deploying a server with a self-signed cert to production.

Here, we will talk about how to create a Letsencrypt TLS cert that you can use for the khadga
backend

## Certbot

TODO: Show how to use certbot and docker to autogenerate and autorenew a TLS cert that we can deploy