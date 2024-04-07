# Installing Pixlet

## Install on Linux (Debian "Bookworm")

Download the pixlet binary from the [latest](https://github.com/tidbyt/pixlet/releases) release and move it into your PATH.

```
# Download the archive.
curl -LO https://github.com/tidbyt/pixlet/releases/download/v0.30.2/pixlet_0.30.2_linux_arm64.tar.gz

# Unpack the archive.
tar -xvf pixlet_0.30.2_linux_arm64.tar.gz

# Ensure the binary is executable.
chmod +x ./pixlet

# Move the binary into your path.
sudo mv pixlet /usr/local/bin/pixlet
```

# BitCoin Ticker

Create a new file:

`sudo nano bitcoin_ticker.star`

and paste the following in the file:

```
load("render.star", "render")
load("http.star", "http")
load("encoding/base64.star", "base64")

COINDESK_PRICE_URL = "https://api.coindesk.com/v1/bpi/currentprice.json"

BTC_ICON = base64.decode("""
iVBORw0KGgoAAAANSUhEUgAAABEAAAARCAYAAAA7bUf6AAAAlklEQVQ4T2NkwAH+H2T/jy7FaP+
TEZtyDEG4Zi0TTPXXzoDF0A1DMQRsADbN6MZdO4NiENwQbAbERh1lWLzMmgFGo5iFZBDYEFwuwG
sISCPUIKyGgDRjAyBXYXMNIz5XgDQga8TpLboYgux8DO/AwoUuLiEqTLBFMcmxQ7V0gssgklIsL
AYozjsoBoE45OZi5DRBSnkCAMLhlPBiQGHlAAAAAElFTkSuQmCC
""")

def main():
    rep = http.get(COINDESK_PRICE_URL, ttl_seconds = 240) # cache for 4 minutes
    if rep.status_code != 200:
        fail("Coindesk request failed with status %d", rep.status_code)
    rate = rep.json()["bpi"]["USD"]["rate_float"]

    # for development purposes: check if result was served from cache or not
    if rep.headers.get("Tidbyt-Cache-Status") == "HIT":
        print("Hit! Displaying cached data.")
    else:
        print("Miss! Calling CoinDesk API.")

    return render.Root(
        child = render.Box(
            render.Row(
                expanded=True,
                main_align="space_evenly",
                cross_align="center",
                children = [
                    render.Image(src=BTC_ICON),
                    render.Text("$%d" % rate),
                ],
            ),
        ),
    )
```
Run it with the pixlet serve command:

`pixlet serve bitcoin_ticker.star`

You can view the result by navigating to

`http://localhost:8080`

# Installing Nginx.
To make the Bitcoin ticker accessible for the Pixelrunner, we need a proxy server sinds it's running on localhost. We can achieve this by using Nginx.

### Install Nginx (if not already installed):

```
sudo apt update
sudo apt install nginx
```

### Configure Nginx:

Create a new Nginx configuration file:

`sudo nano /etc/nginx/sites-available/pixelrunner`

Here's an example configuration to forward requests from `*.*.*.*:8080 to 127.0.0.1:8080`: Where `*.*.*.*` the IP of your Raspberry is (`ifconfig`)

```
server {
    listen *.*.*.*:8080;
    server_name *.*.*.*;

    location / {
        proxy_pass http://127.0.0.1:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```
### Enable the Configuration:
Once you've created your configuration file, create a symbolic link to it in the `/etc/nginx/sites-enabled/` directory:

`sudo ln -s /etc/nginx/sites-available/mywebsite /etc/nginx/sites-enabled/`

### Test Configuration:
Before restarting Nginx, it's a good idea to test the configuration for syntax errors:

`sudo nginx -t`

### Reload Nginx:
If the configuration test is successful, reload Nginx to apply the changes:

`sudo systemctl reload nginx`

Now you should be able to see the Bitcoin ticker at

`http://*.*.*.*:8080`