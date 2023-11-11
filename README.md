# gpg-serve-key

This script allows to transfer a public/private GPG key from a server to
another device where communication is only possible over `https`. Note that
in general this should not be a first choice. For example, if you have `ssh`
access, a better way to transfer a key would be

    ssh user@remote gpg2 --export-secret-key KEYID | gpg2 --import

However, transfer over `https` is usually a better choice than e.g. emailing an
exported secret-key file to yourself. The one particular use case motivating
this script was the import of a secret key into the [Pass iOS app][1].

While transfer over `https` in principle makes it accessible to anyone, the
script takes strong measures to protect the key:

*   They key is directly read through a pipe from the `gpg` executable. The
    secret key is never written to disk

*   The server encrypts the communication with SSL (that is, using the `https`
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

*   The server writes log messages about every served request. This allows to
    monitor for unexpected access and to detect if the key has been compromised
    (as a last resort)

Through the `--serve-file` option, files in addition to the GPG key may be
served (e.g. a private SSH key)


## Requirements ##

*  Python >= 3.5
*  [click package][2]
*  A server that is accessible through a public hostname, with GPG installed
   and the private key for the KEYID that is to be exported in its keychain.
   Check available private keys with `gpg2 -K`.
*  SSL certificates for the public hostname. It is recommended to use
   [Let's Encrypt][3]. You may use an existing certificate for a webserver
   running on the host. Since `gpg-serve-key` will run on a non-standard port,
   it will not be necessary to temporarily suspend the web server.


## Usage ##

Run the script directly as e.g.

    ./gpg-serve-key \
        --cert-file=/etc/letsencrypt/live/michaelgoerz.net/cert.pem \
        --key-file=/etc/letsencrypt/live/michaelgoerz.net/privkey.pem \
        --host=michaelgoerz.net 57A6CAA6

See `./gpg-serve-key --help` for more details. You may use either the short
8-digit key KEYID, or the full length KEYID as shown by `gpg -K`.

The command will start a temporary webserver at a random port and serve both
the public and the private key at URLs such as

    https://michaelgoerz.net:47409/v1f4Y7XixMQ/57A6CAA6-public.key
    https://michaelgoerz.net:47409/v1f4Y7XixMQ/57A6CAA6-secret.key

If using a Cloudflare proxy for the domain, it must be temporarily disabled.
Make sure any firewall running on the server is set up allow access to the
port. On Ubuntu, to allow access to, e.g., port `47409`, run

    sudo ufw allow 47409

After importing the keys from the above URLs, stop the server by hitting
`ctrl+c`.

If applicable, remove the firewall rule (`sudo ufw delete allow 47409`), and
re-enable the Cloudflare proxy.

[1]: https://mssun.github.io/passforios/
[2]: http://click.pocoo.org/5/
[3]: https://letsencrypt.org
