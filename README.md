# testpage

A page that tells you what the web server sees about your request: your
address and port, the IP version, whether the connection is encrypted, the
TLS parameters, the server address and name, the date, and the request
headers.

This fork renders the page with **nginx's native SSI module** and no longer
supports Apache. The original Apache version lives at
<https://github.com/bernd289/testpage>.

## How it works

`index.shtml`, `full.txt` and `ip.txt` are Server Side Include templates.
nginx evaluates them in-process, so the same template produces correct HTML
*and* correct plain text. No `sub_filter`, no external process, no CSS tricks
to hide the branch that should not have been rendered.

## Setup

1. nginx must be built with the SSI module. It is part of every standard
   build:

   ```shell
   nginx -V 2>&1 | grep -q with-http_ssi_module && echo "SSI enabled"
   ```

2. Point a server at this directory. `ssi_types` is what makes the `.txt`
   endpoints work, since nginx only processes `text/html` by default:

   ```nginx
   geo $ipversion {
       ::ffff:0:0/96 ipv4;
       ::0/0        ipv6;
       0.0.0.0/0    ipv4;
   }

   server {
       listen 80;
       listen [::]:80;
       listen 443 ssl;
       listen [::]:443 ssl;
       http2 on;

       server_name test.example.com;
       root /var/www/testpage;
       index index.shtml;

       location / {
           ssi on;
           ssi_types text/plain;

           add_header Cache-Control "max-age=0, no-cache, no-store, must-revalidate" always;
           add_header Pragma "no-cache" always;
           add_header Expires "Mon, 1 Jan 2019 01:00:00 GMT" always;
       }
   }
   ```

   `geo` lives in the `http` context. Site configs included from
   `sites-enabled/` already are, so it can sit at the top of the file.

3. Endpoints:

   | Path         | Content                              |
   |--------------|--------------------------------------|
   | `/`          | the HTML page                        |
   | `/full.txt`  | the same information as plain text   |
   | `/ip.txt`    | just the client address              |

## Differences from the Apache version

- **No reverse DNS.** nginx has no built-in reverse lookup, so `REMOTE_HOST`
  does not exist and the page always reports "(without DNS name)". Getting it
  back needs a separate process in front of or behind nginx.
- **No cipher key size.** nginx exposes `$ssl_cipher` but no variable for the
  negotiated key size, so the "… bit" part is gone.
- **Values are escaped.** nginx's `#echo` HTML-escapes by default. That also
  closes the hole where Apache's version rendered a `User-Agent` containing
  HTML markup verbatim. The `.txt` templates opt out with
  `encoding="none"`.
- **`$ipversion` comes from `geo`**, not from a regex on the address, so the
  `geo` block above is required.

## Test instances

These run the upstream Apache version, not this fork:

- <https://test.matthiashille.com>
  (IPv4 only: <https://test4.matthiashille.com>, IPv6 only: <https://test6.matthiashille.com>)
- <https://test.mazdermind.de>
  (IPv4 only: <https://test4.mazdermind.de>, IPv6 only: <https://test6.mazdermind.de>)
- <https://test.dssr.ch>
