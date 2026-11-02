[![推广活动](https://www.bright.cn/static/github_promo_15.png?md5=105367-daeb786e)](https://www.bright.cn/?promo=github15) 
# PHP 代理服务器：如何在 PHP 中设置代理

了解如何使用 cURL、`file_get_contents()` 和 Symfony 在 PHP 中设置代理。你还将了解如何在 PHP 中使用 Bright Data 的[住宅代理](https://www.bright.cn/proxy-types/residential-proxies)进行网页爬虫和 IP 轮换。本指南也可在 [Bright Data 博客](https://www.bright.cn/blog/how-tos/php-proxy-servers)上查看。

- [要求](#要求)
  - [在 Apache 中设置本地代理服务器](#在-apache-中设置本地代理服务器)
- [在 PHP 中使用代理](#在-php-中使用代理)
  - [使用 cURL 集成代理](#使用-curl-集成代理)
  - [使用 `file_get_contents()` 集成代理](#使用-file_get_contents-集成代理)
  - [在 Symfony 中集成代理](#在-symfony-中集成代理)
- [测试 PHP 中的代理集成](#测试-php-中的代理集成)
- [在 PHP 中集成 Bright Data 代理](#在-php-中集成-bright-data-代理)
  - [设置住宅代理](#设置住宅代理)
  - [通过身份验证代理进行网页爬虫的示例](#通过身份验证代理进行网页爬虫的示例)
  - [测试 IP 轮换](#测试-ip-轮换)

## 要求

确认你的计算机上已安装 [PHP 8+](https://www.php.net/downloads.php)、[Composer](https://getcomposer.org/download/) 和 [Apache](https://httpd.apache.org/download.cgi)。如果尚未安装，请点击前面的链接下载安装程序，启动安装程序并按照说明操作。

确保 Apache 服务已启动并正常运行。

为你的 PHP 项目创建一个文件夹，进入该文件夹，然后在其中初始化一个新的 Composer 应用程序：

```bash
mkdir <PHP_PROJECT_FOLDER_NAME>
cd <PHP_PROJECT_FOLDER_NAME>
composer init
```

**注意**：在 Windows 上，我们建议使用 WSL（[适用于 Linux 的 Windows 子系统](https://learn.microsoft.com/en-us/windows/wsl/install)）。

### 在 Apache 中设置本地代理服务器

将你的本地 Apache Web 服务器配置为正向代理服务器。

首先，使用以下命令启用 [`mod_proxy`](https://httpd.apache.org/docs/2.4/mod/mod_proxy.html)、[`mod_proxy_http`](https://httpd.apache.org/docs/2.4/mod/mod_proxy_http.html) 和 [`mod_proxy_connect`](https://httpd.apache.org/docs/2.4/mod/mod_proxy_connect.html) 模块：

```bash
sudo a2enmod proxy
sudo a2enmod proxy_http
sudo a2enmod proxy_connect
```

然后，在 `/etc/apache2/sites-available/` 中创建一个名为 `proxy.conf` 的新[虚拟主机配置文件](https://httpd.apache.org/docs/2.4/vhosts/)，内容从默认虚拟主机配置文件 `000-default.conf` 复制：

```bash
cd /etc/apache2/sites-available/
sudo cp 000-default.conf proxy.conf
```

使用以下代理定义逻辑初始化 `proxy.conf`：

```
<VirtualHost *:80>
    # set the server name to localhost
    ServerName localhost
    # set the server admini email to admin@localhost
    ServerAdmin admin@localhost

    # if the SSL module is enabled
    <IfModule mod_ssl.c>
        # disable SSL to avoid certificate errors
        SSLEngine off
    </IfModule>

    # specify the error log file location
    ErrorLog ${APACHE_LOG_DIR}/error.log
    # specify the access log file location and format
    CustomLog ${APACHE_LOG_DIR}/access.log combined

    # enable proxy logic
    ProxyRequests On
    ProxyVia On

    # define a proxy for all requests
    <Proxy *>
        Order deny,allow
        Allow from all
    </Proxy>
</VirtualHost>
```

使用以下命令注册新的 Apache 虚拟主机：

```bash
sudo a2ensite proxy.conf
```

最后，重新加载 Apache 服务器：

```bash
service apache2 reload
```

现在，你已有一个在 `http://localhost:80` 上侦听的本地代理服务器。

## 在 PHP 中使用代理

了解如何将代理集成到 PHP 的以下技术中：

- [cURL](https://www.php.net/manual/en/book.curl.php)
- [`file_get_contents()`](https://www.php.net/manual/en/function.file-get-contents.php)
- [Symfony](https://symfony.com/)

### 使用 cURL 集成代理

使用 cURL 库时，可通过 [`CURLOPT_PROXY`](https://curl.se/libcurl/c/CURLOPT_PROXY.html) 选项在 PHP 中指定代理服务器，如 `curl_proxy.php` 代码片段所示：

```php
// the URL of the proxy server
$proxyUrl = 'http://localhost:80';
// the URL of the target site
$targetUrl = 'https://httpbin.org/get';

// initialize a cURL session
$ch = curl_init();

// set the target URL
curl_setopt($ch, CURLOPT_URL, $targetUrl);
// set the proxy server to be used for routing the request
curl_setopt($ch, CURLOPT_PROXY, $proxyUrl);
// return the response as a string
curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);

// disable SSL certificate verification to avoid certificate errors
curl_setopt($ch, CURLOPT_SSL_VERIFYPEER, false);
curl_setopt($ch, CURLOPT_SSL_VERIFYHOST, false);

// execute the cURL request
$response = curl_exec($ch);

// if the response is not successful
if (curl_errno($ch)) {
    echo 'cURL Error: ' . curl_error($ch);
} else {
    echo $response;
}

// close the cURL session
curl_close($ch);
```

### 使用 `file_get_contents()` 集成代理

在 `file_get_contents()` 中使用 `proxy` 选项设置代理服务器，如下面的 `file_get_contents_proxy.php` 代码片段所示：

```php
// define the proxy server to be used for HTTP/HTTPS requests
$options = [
    'http' => [
        'proxy' => 'tcp://localhost:80',
        // force the use of the full URI when making the request
        'request_fulluri' => true,
    ],
];
// create a stream context with the defined options
$context = stream_context_create($options);

// the URL of the target site
$url = 'https://httpbin.org/get';
// perform an HTTP request with the defined context
$response = file_get_contents($url, false, $context);

// if the response was not successful
if ($response === false) {
    echo "Failed to retrieve data from $url";
} else {
    echo $response;
}
```

**注意**：`proxy` 选项中的代理服务器协议必须是 `tcp`，而不是 `http`。

### 在 Symfony 中集成代理

安装 Symfony 的 [`BrowserKit`](https://symfony.com/doc/current/components/browser_kit.html) 和 [`HTTP Client`](https://symfony.com/doc/current/http_client.html) 组件：

```bash
composer require symfony/browser-kit symfony/http-client
```

使用 `HttpBrowser` 发起 HTTP 请求时，通过 `HttpClient` 中的 [`proxy`](https://symfony.com/doc/current/http_client.html#http-proxies) 选项指定代理服务器，如 `symfony_proxy.php` 代码片段所示：

```php
// include the Composer autoload file
require './vendor/autoload.php';

// load the required load Symfony components
use Symfony\Component\BrowserKit\HttpBrowser;
use Symfony\Component\HttpClient\HttpClient;

// define the proxy server and port
$proxyServer = 'http://localhost';
$proxyPort = '80';

// create an HTTP client with a proxy configuration
$client = new HttpBrowser(HttpClient::create(['proxy' => sprintf('%s:%s', $proxyServer, $proxyPort)]));

// make a GET request to the target URL
$client->request('GET', 'https://httpbin.org/get');

// get the content of the response
$content = $client->getResponse()->getContent();

// output the content
echo $content;
```

## 测试 PHP 中的代理集成

使用以下命令启动三个 PHP 代理集成脚本中的任意一个：

```bash
php <PHP_SCRIPT_NAME>
```

无论执行哪个脚本，结果都将类似如下：

```json
{
  "args": {},
  "headers": {
    "Accept": "*/*",
    "Host": "httpbin.org",
    "X-Amzn-Trace-Id": "Root=1-661ab837-40de4746307643415ec9c659"
  },
  "origin": "XX.YY.ZZ.AA",
  "url": "https://httpbin.org/get"
}
```

查看 `access.log`，这是记录代理所发请求的 Apache 日志文件：

```
tail -n 50 /var/log/apache2/access.log
```

最后一行表示请求已成功通过代理转发至 `httpbin.org`，响应状态码为 `200`：

```
::1 - - [13/Apr/2024:18:53:22 +0200] "CONNECT httpbin.org:443 HTTP/1.0" 200 6138 "-" "-"
```

## 在 PHP 中集成 Bright Data 代理

Bright Data 提供可自动为你轮换出口 IP 的[高级代理](https://www.bright.cn/proxy-types)。下面来看看如何在使用 cURL 的 PHP 脚本中将其用于网页爬虫。

### 设置住宅代理

[注册 Bright Data](https://www.bright.cn/cp/start) 以开始免费试用。前往“代理和抓取基础设施”控制面板，然后点击“住宅代理”卡片上的“开始使用”。

按照流程设置住宅代理，并获取以下凭据：

- `<BRIGHTDATA_PROXY_HOST>`
- `<BRIGHTDATA_PROXY_PORT>`
- `<BRIGHTDATA_PROXY_USERNAME>`
- `<BRIGHTDATA_PROXY_PASSWORD>`

### 通过身份验证代理进行网页爬虫的示例

使用 Bright Data 的身份验证住宅代理连接到[维基百科的“代理服务器”页面](https://en.wikipedia.org/wiki/Proxy_server)，并使用 [`DOMDocument`](https://www.php.net/manual/en/class.domdocument.php) 从中抓取数据，如 `curl_proxy_scraping.php` 代码片段所示：

```php
// Bright Data proxy details
$proxyUrl = '<BRIGHTDATA_PROXY_HOST>:<BRIGHTDATA_PROXY_PORT>';
$proxyUser = '<BRIGHTDATA_PROXY_USERNAME>:<BRIGHTDATA_PROXY_PASSWORD>';

// target scraping page
$targetUrl = 'https://en.wikipedia.org/wiki/Proxy_server';

// perform a GET request to the target page
// through the Bright Data proxy
$ch = curl_init();

curl_setopt($ch, CURLOPT_URL, $targetUrl);
curl_setopt($ch, CURLOPT_PROXY, $proxyUrl);
curl_setopt($ch, CURLOPT_PROXYUSERPWD, $proxyUser);
curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);
curl_setopt($ch, CURLOPT_SSL_VERIFYPEER, false);

$response = curl_exec($ch);

if (curl_errno($ch)) {
    echo 'cURL Error: ' . curl_error($ch);
} else {
    // parse the HTML document returned by the server
    $dom = new DOMDocument();
    @$dom->loadHTML($response);

    // extract the text content on the page
    $content = $dom->getElementById('mw-content-text')->textContent;

    // extract the titles from the H2 headings
    $headings = [];
    $headingsNodeList = $dom->getElementsByTagName('h2');
    foreach ($headingsNodeList as $heading) {
        $headings[] = $heading->textContent;
    }

    // extract the titles from the H3 headings
    $headingsNodeList = $dom->getElementsByTagName('h3');
    foreach ($headingsNodeList as $heading) {
        $headings[] = $heading->textContent;
    }

    // print the scraped data
    echo "Content:\n";
    echo $content . "\n\n";

    echo "Headings:\n";
    foreach ($headings as $index => $heading) {
        echo ($index + 1) . ". $heading\n";
    }
}

curl_close($ch);
```

输出如下：

```
Content:
Computer server that makes and receives requests on behalf of a user
.mw-parser-output .hatnote{font-style:italic}.mw-parser-output div.hatnote{padding-left:1.6em;margin-bottom:0.5em}.mw-parser-output .hatnote i{font-style:normal}.mw-parser-output .hatnote+link+.hatnote{margin-top:-0.5em}For Wikipedia's policy on editing from open proxies, please see Wikipedia:Open proxies. For other uses, see Proxy.


Communication between two computers connected through a third computer acting as a proxy server.
// omitted for brevity...

Headings:
1. Contents
2. Types[edit]
3. Uses[edit]
// omitted for brevity...
```

### 测试 IP 轮换

运行以 [`http://lumtest.com/myip.json`](http://lumtest.com/myip.json) 为目标的 PHP 代理脚本 `curl_proxy_brightdata.php`。这是一个返回你的 IP 信息的特殊端点：

```php
<?php

$proxyUrl = '<BRIGHTDATA_PROXY_HOST>:<BRIGHTDATA_PROXY_PORT>';
$proxyUser = '<BRIGHTDATA_PROXY_USERNAME>:<BRIGHTDATA_PROXY_PASSWORD>';

$targetUrl = 'http://lumtest.com/myip.json';

$ch = curl_init();

curl_setopt($ch, CURLOPT_URL, $targetUrl);
curl_setopt($ch, CURLOPT_PROXY, $proxyUrl);
curl_setopt($ch, CURLOPT_PROXYUSERPWD, $proxyUser);
curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);
curl_setopt($ch, CURLOPT_SSL_VERIFYPEER, false);

$response = curl_exec($ch);

if (curl_errno($ch)) {
    echo 'cURL Error: ' . curl_error($ch);
} else {
    echo $response;
}

curl_close($ch);
```

多次执行该脚本。每次你都会看到来自不同位置的不同 IP。
