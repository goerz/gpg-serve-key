# gpg-serve-key

This script allows to transfer a public/private GPG key from a server to
another device where communication is only possible over the https. Note that
in general this should not be a first choice. For example, if you have ssh
access, a better way to transfer a key would be

    ssh user@remote gpg2 --export-secret-key KEYID | gpg2 --import

However, transfer over https is usually a better choice than e.g. emailing an
exported secret-key file to yourself. The one particular use case motivating
this script was the import of a secret key into the [Pass iOS app][1]

While transfer over https in principle makes it accessible to anyone, the
script takes strong measures to protect the key data:

*   They key is directly read through a pip from the `gpg` executable. The
    secret key is never written to disk

*   The server encrypts the communication with SSL (that is, using the https
    protocol) by default. While this creates the additional overhead of
    requiring valid SSL certificates for the public hostname under which the
    server will be reached, it is essential to guarantee that the key cannot be
    sniffed in transit. For use within a trusted network, the encryption can be
    disabled, although you are strongly discouraged from doing so.

*   The key is exposed at a url that contains a random token and using a random
    port number (by default), e.g. for the KEYID 57A6CAA6

        https://michaelgoerz.net:47409/v1f4Y7XixMQ/57A6CAA6-secret.key

*   Brute-forcing the token is prevented through rate limiting, that is, by an
    exponentially increasing delay after an invalid request

*   The server responds with HTTP headers that disable caching by the client.
*
*   The server writes log messages about every served request. This allows to
    monitor for unexpected access and to detect if the key has been compromised
    (as a last resort)



## Requirements ##

*  Python >3.5
*  [click package][2]
*  A server that is accessible through a public hostname, with GPG installed
   and the private key for the KEYID that is to be exported in its keychain
*  SSL certificates for the public hostname. It is recommended to use
   [Let's Encrypt][3]. You may use an existing certificate for a webserver
   running on the host


## Usage ##

Run the script directly as e.g.

    ./gpg-serve-key \
        --cert-file=/etc/letsencrypt/live/michaelgoerz.net/cert.pem \
        --key-file=/etc/letsencrypt/live/michaelgoerz.net/privkey.pem \
        --host=michaelgoerz.net 57A6CAA6

See `./gpg-serve-key --help` for more details.

This will start temporary webserver at a random port and serve both the public
and the private key at URLs such as

    https://michaelgoerz.net:47409/v1f4Y7XixMQ/57A6CAA6-public.key
    https://michaelgoerz.net:47409/v1f4Y7XixMQ/57A6CAA6-secret.key

After importing the keys from these URLs, stop the serve by hitting `ctrl+c`.

[1]: https://mssun.github.io/passforios/
[2]: http://click.pocoo.org/5/
[3]: https://letsencrypt.org
