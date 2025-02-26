[![Promo](https://brightdata.com/static/github_promo_15.png?md5=105367-daeb786e)](https://www.bright.cn/?promo=github15)  
# PHP 代理服务器：如何在 PHP 中设置代理

了解如何在 PHP 中使用 cURL、`file_get_contents()` 和 Symfony 来设置代理。你还将看到如何在 PHP 中使用 Bright Data 的[住宅代理](https://www.bright.cn/proxy-types/residential-proxies)进行网络抓取和 IP 轮换。本指南也可在 [Bright Data 博客](https://www.bright.cn/blog/how-tos/php-proxy-servers)上查看。

- [环境要求](#环境要求)
  - [在 Apache 中设置本地代理服务器](#在-Apache-中设置本地代理服务器)
- [在 PHP 中使用代理](#在-PHP-中使用代理)
  - [使用 cURL 集成代理](#使用-cURL-集成代理)
  - [使用 `file_get_contents()` 集成代理](#使用-file_get_contents()-集成代理)
  - [在 Symfony 中集成代理](#在-Symfony-中集成代理)
- [在 PHP 中测试代理集成](#在-PHP-中测试代理集成)
- [在 PHP 中集成 Bright Data 代理](#在-PHP-中集成-Bright-Data-代理)
  - [住宅代理设置](#住宅代理设置)
  - [通过身份验证代理的网络抓取示例](#通过身份验证代理的网络抓取示例)
  - [测试 IP 轮换](#测试-IP-轮换)

## 环境要求

确保你已在本机安装了 [PHP 8+](https://www.php.net/downloads.php)、[Composer](https://getcomposer.org/download/) 和 [Apache](https://httpd.apache.org/download.cgi)。如果还未安装，可点击前面链接下载安装包，运行并按照指示进行安装。

确认 Apache 服务已启动并正常运行。

创建一个用于存放 PHP 项目的文件夹，进入该文件夹，在其中初始化一个新的 Composer 应用：

```bash
mkdir <PHP_PROJECT_FOLDER_NAME>
cd <PHP_PROJECT_FOLDER_NAME>
composer init
```

**注意**：在 Windows 上，推荐使用 WSL（[Windows Subsystem for Linux](https://learn.microsoft.com/en-us/windows/wsl/install)）。

### 在 Apache 中设置本地代理服务器

将你的本地 Apache 服务器配置为正向代理服务器。

首先，通过以下命令启用 [`mod_proxy`](https://httpd.apache.org/docs/2.4/mod/mod_proxy.html)、[`mod_proxy_http`](https://httpd.apache.org/docs/2.4/mod/mod_proxy_http.html) 和 [`mod_proxy_connect`](https://httpd.apache.org/docs/2.4/mod/mod_proxy_connect.html) 模块：

```bash
sudo a2enmod proxy
sudo a2enmod proxy_http
sudo a2enmod proxy_connect
```

然后，在 `/etc/apache2/sites-available/` 目录下，基于默认的虚拟主机配置文件 `000-default.conf` 复制并创建一个新的 [虚拟主机配置文件](https://httpd.apache.org/docs/2.4/vhosts/) `proxy.conf`：

```bash
cd /etc/apache2/sites-available/
sudo cp 000-default.conf proxy.conf
```

在 `proxy.conf` 中添加以下代理定义逻辑：

```
<VirtualHost *:80>
    # 将服务器名称设置为 localhost
    ServerName localhost
    # 将服务器管理员邮箱设置为 admin@localhost
    ServerAdmin admin@localhost

    # 如果 SSL 模块已启用
    <IfModule mod_ssl.c>
        # 禁用 SSL 以避免证书错误
        SSLEngine off
    </IfModule>

    # 指定错误日志文件位置
    ErrorLog ${APACHE_LOG_DIR}/error.log
    # 指定访问日志文件位置和格式
    CustomLog ${APACHE_LOG_DIR}/access.log combined

    # 启用代理功能
    ProxyRequests On
    ProxyVia On

    # 对所有请求定义代理
    <Proxy *>
        Order deny,allow
        Allow from all
    </Proxy>
</VirtualHost>
```

使用以下命令注册新建的 Apache 虚拟主机：

```bash
sudo a2ensite proxy.conf
```

最后，重载 Apache 服务：

```bash
service apache2 reload
```

现在就有一个本地的代理服务器在 `http://localhost:80` 上监听了。

## 在 PHP 中使用代理

以下示例演示如何在 PHP 中将代理与下列技术结合：

- [cURL](https://www.php.net/manual/en/book.curl.php)
- [`file_get_contents()`](https://www.php.net/manual/en/function.file-get-contents.php)
- [Symfony](https://symfony.com/)

### 使用 cURL 集成代理

通过在 cURL 中使用 [`CURLOPT_PROXY`](https://curl.se/libcurl/c/CURLOPT_PROXY.html) 指定代理服务器，示例如下 `curl_proxy.php` 示例脚本：

```php
// 代理服务器的 URL
$proxyUrl = 'http://localhost:80';
// 目标网站的 URL
$targetUrl = 'https://httpbin.org/get';

// 初始化 cURL 会话
$ch = curl_init();

// 设置目标 URL
curl_setopt($ch, CURLOPT_URL, $targetUrl);
// 设置要使用的代理服务器来路由请求
curl_setopt($ch, CURLOPT_PROXY, $proxyUrl);
// 将响应内容返回为字符串
curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);

// 禁用 SSL 证书验证以避免证书错误
curl_setopt($ch, CURLOPT_SSL_VERIFYPEER, false);
curl_setopt($ch, CURLOPT_SSL_VERIFYHOST, false);

// 执行 cURL 请求
$response = curl_exec($ch);

// 如果响应不是成功的
if (curl_errno($ch)) {
    echo 'cURL Error: ' . curl_error($ch);
} else {
    echo $response;
}

// 关闭 cURL 会话
curl_close($ch);
```

### 使用 `file_get_contents()` 集成代理

在 `file_get_contents()` 中使用 `proxy` 选项来设置代理服务器，如 `file_get_contents_proxy.php` 示例脚本所示：

```php
// 为 HTTP/HTTPS 请求定义代理服务器
$options = [
    'http' => [
        'proxy' => 'tcp://localhost:80',
        // 在发起请求时使用完整 URI
        'request_fulluri' => true,
    ],
];
// 使用定义好的选项创建一个流上下文
$context = stream_context_create($options);

// 目标网站的 URL
$url = 'https://httpbin.org/get';
// 使用定义好的上下文执行 HTTP 请求
$response = file_get_contents($url, false, $context);

// 如果响应为 false，则表示请求失败
if ($response === false) {
    echo "Failed to retrieve data from $url";
} else {
    echo $response;
}
```

**注意**：在 `proxy` 选项中，代理服务器协议需要写为 `tcp` 而不是 `http`。

### 在 Symfony 中集成代理

安装 [`BrowserKit`](https://symfony.com/doc/current/components/browser_kit.html) 和 [`HTTP Client`](https://symfony.com/doc/current/http_client.html) 这两个 Symfony 组件：

```bash
composer require symfony/browser-kit symfony/http-client
```

在 `HttpClient` 中通过 [`proxy`](https://symfony.com/doc/current/http_client.html#http-proxies) 选项指定代理服务器，并在使用 `HttpBrowser` 发起 HTTP 请求时生效，如 `symfony_proxy.php` 示例脚本所示：

```php
// 引入 Composer 的自动加载文件
require './vendor/autoload.php';

// 导入所需的 Symfony 组件
use Symfony\Component\BrowserKit\HttpBrowser;
use Symfony\Component\HttpClient\HttpClient;

// 定义代理服务器和端口
$proxyServer = 'http://localhost';
$proxyPort = '80';

// 使用包含代理配置的方式创建 HTTP client
$client = new HttpBrowser(HttpClient::create(['proxy' => sprintf('%s:%s', $proxyServer, $proxyPort)]));

// 发起 GET 请求到目标 URL
$client->request('GET', 'https://httpbin.org/get');

// 获取响应内容
$content = $client->getResponse()->getContent();

// 输出内容
echo $content;
```

## 在-PHP-中测试代理集成

使用以下命令来运行任一前面提到的 PHP 代理示例脚本：

```bash
php <PHP_SCRIPT_NAME>
```

无论运行哪个脚本，都可能得到类似这样的结果：

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

可以查看 Apache 的代理访问日志文件 `access.log`，其中记录了通过代理发起的请求：

```
tail -n 50 /var/log/apache2/access.log
```

最后一行指示该请求已成功代理到 `httpbin.org`，且响应状态码为 `200`：

```
::1 - - [13/Apr/2024:18:53:22 +0200] "CONNECT httpbin.org:443 HTTP/1.0" 200 6138 "-" "-"
```

## 在 PHP 中集成 Bright Data 代理

Bright Data 提供[高级代理](https://www.bright.cn/proxy-types)，它们会自动替你轮换出口 IP。下面演示如何在 PHP 脚本中使用 cURL 来进行网络抓取。

### 住宅代理设置

[注册 Bright Data](https://www.bright.cn/cp/start) 并开始免费试用。进入 “Proxies & Scraping Infrastructure” 仪表盘，在“Residential Proxy”卡片上点击 “Get Started”。

按照向导完成设置步骤并获取以下凭证：

- `<BRIGHTDATA_PROXY_HOST>`
- `<BRIGHTDATA_PROXY_PORT>`
- `<BRIGHTDATA_PROXY_USERNAME>`
- `<BRIGHTDATA_PROXY_PASSWORD>`

### 通过身份验证代理的网络抓取示例

使用 Bright Data 的住宅代理进行身份验证后访问 [“Proxy server” 维基百科页面](https://en.wikipedia.org/wiki/Proxy_server)，并使用 [`DOMDocument`](https://www.php.net/manual/en/class.domdocument.php) 来抓取页面内容。示例脚本 `curl_proxy_scraping.php` 如下：

```php
// Bright Data 代理详情
$proxyUrl = '<BRIGHTDATA_PROXY_HOST>:<BRIGHTDATA_PROXY_PORT>';
$proxyUser = '<BRIGHTDATA_PROXY_USERNAME>:<BRIGHTDATA_PROXY_PASSWORD>';

// 目标抓取页面
$targetUrl = 'https://en.wikipedia.org/wiki/Proxy_server';

// 通过 Bright Data 代理对目标页面执行 GET 请求
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
    // 解析服务器返回的 HTML 文档
    $dom = new DOMDocument();
    // 使用 @ 屏蔽 HTML 解析产生的警告
    @$dom->loadHTML($response);

    // 提取页面文本内容
    $content = $dom->getElementById('mw-content-text')->textContent;

    // 从 H2 标签中提取标题
    $headings = [];
    $headingsNodeList = $dom->getElementsByTagName('h2');
    foreach ($headingsNodeList as $heading) {
        $headings[] = $heading->textContent;
    }

    // 从 H3 标签中提取标题
    $headingsNodeList = $dom->getElementsByTagName('h3');
    foreach ($headingsNodeList as $heading) {
        $headings[] = $heading->textContent;
    }

    // 输出抓取到的数据
    echo "Content:\n";
    echo $content . "\n\n";

    echo "Headings:\n";
    foreach ($headings as $index => $heading) {
        echo ($index + 1) . ". $heading\n";
    }
}

curl_close($ch);
```

脚本的输出如下所示：

```
Content:
Computer server that makes and receives requests on behalf of a user
.mw-parser-output .hatnote{font-style:italic}.mw-parser-output div.hatnote{padding-left:1.6em;margin-bottom:0.5em}.mw-parser-output .hatnote i{font-style:normal}.mw-parser-output .hatnote+link+.hatnote{margin-top:-0.5em}For Wikipedia's policy on editing from open proxies, please see Wikipedia:Open proxies. For other uses, see Proxy.


Communication between two computers connected through a third computer acting as a proxy server.
// 省略中间内容...

Headings:
1. Contents
2. Types[edit]
3. Uses[edit]
// 省略中间内容...
```

### 测试 IP 轮换

运行名为 `curl_proxy_brightdata.php` 的 PHP 脚本，目标 URL 为 [http://lumtest.com/myip.json](http://lumtest.com/myip.json)，该地址可返回你的 IP 信息：

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

多次执行该脚本，每次你都会看到来自不同位置、不同 IP 的结果。
