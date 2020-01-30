# Authentication

We don't just want anyone to come in and use our service.  We could allow guests to access some
parts of the system, but not everything.  The first step is having a user sign up and register
themselves to the service.

Once a user is registered, we need a way to validate that the entity accessing the service, is who
they actually say they are.  This is authentication.  This is not to be confused with authorization,
which means that we have know you are John Doe, but we need to know what permissions John Doe has to
our system.

## Classic database authentication

We will be implementing a classic password + database security.  While perhaps not the best system,
we can later augment this.

TODO: Discuss how to send data from web page, to backend, and then to the mongodb.

## 2FA

TODO: using 2FA or TPM

Talk about stragies for stronger authN

## Oauth

TODO: Talk about possibly implementing OAuth with google