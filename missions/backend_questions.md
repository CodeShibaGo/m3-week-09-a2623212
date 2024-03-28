# 說明什麼是 WSGI，跟 ASGI 又有甚麼關係？
### 為什麼我們需要 WSGI？
WSGI，全名 Web Server Gateway Interface，他定義了 web server 和 Python web application 之間溝通的規範。只要 web server 和 application 有符合 WSGI 規範，就可以搭配使用。這因此提供開發者彈性，不用因為 server 和 application 不相容，就要重新開發。也可以把開發切開，開發 web server 的人需要去處理多個請求進到 server 狀況，但是 web application(framework) 的開發者，就不需要處理這種事情，可以更專注在 framework 的開發。 -> [What is WSGI?](https://wsgi.readthedocs.io/en/latest/what.html)
### WSGI 跟 WSGI Server 有什麼不一樣？
WSGI server 指的是支援 WSGI 規範的 server。例如 gunicorn 或 uwsgi。
### ASGI 又是什麼？
全名 Asynchronous Server Gateway Interface，是 WSGI 的繼承者，既存的 WSGI 應用可以直接在 ASGI 服務器中運行。為了解決現有的 WSGI 不支持的新的協議標準，它為異步和同步的應用程式提供標準，除了HTTP、HTTP2，也支持 WebSocket。

# 說明 Gunicorn 是什麼？該如何使用它？
### Gunicorn 是什麼？
是一個適用於 UNIX 的 Python WSGI HTTP 伺服器，可以和大多數的 Python Web 框架兼容，支援 WSGI、Django、Paster。
### 如何使用它？
- python 3.5 版本以上，可以直接 `pip` 下載它；
- 新增一個配置的檔案 `gunicorn.conf.py`；
- 運行 Python application:
```shell
$ gunicorn app:app -c gunicorn.conf.py
```

# 使用守護進程 systemd 啟動 Gunicorn 

`Systemd` 取代了`initd`，成為第一個步驟，Systemd 的優點是功能強大，使用方便；缺點是體系龐大，非常複雜，是強耦合。
```shell=
sudo nano /etc/systemd/system/myproject.service
```
```service=
[Unit]
Description=Gunicorn instance to serve myproject
After=network.target

[Service]
User=ubuntu
Group=www-data
WorkingDirectory=/home/ubuntu/myproject
Environment="PATH=/home/ubuntu/myproject/myprojectenv/bin"
ExecStart=/home/ubuntu/myproject/myprojectenv/bin/gunicorn --workers 3 --bind unix:myproject.sock -m 007 wsgi:app

[Install]
WantedBy=multi-user.target

```
```shell=
## Start the Gunicorn service that you created and enable it so that it starts at boot:
sudo systemctl start myproject
sudo systemctl enable myproject
## Check the status
sudo systemctl status myproject
```
# HTTP 如何加密？
> Sot this approach is just for testing purposes, do not protect your website this way, it would look more suspicious than when using plain unsecured http.

## What is HTTPS?
The encryption and security functionality for HTTP is implemented through the TLS (Transport Layer Security).

Client establishes a connection with the server and requests an encrypted connection, the **server responds** with its **SSL Certificate**.

After client knows and trusts the CA, it can confirm the certificate signature comes from this entity, the connection is legitimate.

The client verifies the certificate and creates an encryption key using a public key that is inclued with the server certificate. This key used for any communication between this client and **the server**, which **owns the private key** to decrypt the package.

## Methods to be HTTPS
1、Add `ssl_context='adhoc'` in `app.run()`

First, install ` pip install pyopenssl` in the virtual env's shell.

Second, in the python file `app.py`.
```python
from flask import Flask
app = Flask(__name__)

@app.route("/")
def hello():
    return "Hello World!"

if __name__ == "__main__":
    app.run(debug=True, ssl_context='adhoc')
```

And use command line `python3 app.py`

:-1: The browser does not like this type of certificate, it shows a big and scary warning before you access the app. This is convenient for quick and dirty tests, but not for any real use.

2、 Use Self-signed Certificates （自簽章憑證｜自己署名証明書）

In the command line:
```shell
(flaskssl_env) $ which openssl
(flaskssl_env) $ openssl req -x509 -newkey rsa:4096 -nodes -out cert.pem keyout priv_key.pem -days 3650
```
A new certificate file  is `cert.pem`, private key is in `priv_key.pem`, with a validity period of 3650 days. 

Then you will be asked some the questions.
```shell
Generating a 4096 bit RSA private key
......................++
.............++
writing new private key to 'priv_key.pem'
-----
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
-----
Country Name (2 letter code) [AU]:US
State or Province Name (full name) [Some-State]:Oregon
Locality Name (eg, city) []:Portland
Organization Name (eg, company) [Internet Widgits Pty Ltd]:Miguel Grinberg Blog
Organizational Unit Name (eg, section) []:
## Please at least fill this value --> localhost
Common Name (e.g. server FQDN or YOUR name) []:localhost
Email Address []:
```
Keypair will be generated in current directory:
```shell
(flaskssl_env) $ ls
app_keypair.py  app.py  cert.pem  flaskssl_env  priv_key.pem  __pycache__
```
Then, we can use this new self-signed certificate in `app_keypair.py`:
```python
from flask import Flask
app = Flask(__name__)

@app.route("/")
def hello():
    return "Hello World!"

if __name__ == "__main__":
    app.run(debug=True, ssl_context=('cert.pem', 'priv_key.pem'))
```
And use command line `python3 app_keypair.py`

3、 Using "Real" Certificates.
[參考這篇文章！](https://blog.miguelgrinberg.com/post/running-your-flask-application-over-https)

# 設定 nginx 並加入 SSL 
But how do we install an SSL certificate on a production server?

1、 If we are using `gunicorn`:
`$ gunicorn --certfile cert.pem --keyfile key.pem -b 0.0.0.0:8000 hello:app`

2、If we use nginx as a reverse proxy, the configuration items for nginx are as follows:
```shell
server {
    listen 443 ssl http2;
    server_name example.com;
    ssl_certificate /path/to/cert.pem;
    ssl_certificate_key /path/to/key.pem;
    # ...
}

## In the HTTP case
server {
    listen 80;
    server_name example.com;
    location / {
        return 301 https://$host$request_uri;
    }
}
```