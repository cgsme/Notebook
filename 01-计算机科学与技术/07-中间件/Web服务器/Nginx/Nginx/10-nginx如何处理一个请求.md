# nginx如何处理一个请求

## 基于名称的虚拟服务器

nginx 首先需要决定让哪个服务器来处理请求。我们从一个简单的配置开始，其中所有三个虚拟服务器都侦听端口 *:80：

```conf
server {
    listen      80;
    server_name example.org www.example.org;
    ...
}

server {
    listen      80;
    server_name example.net www.example.net;
    ...
}

server {
    listen      80;
    server_name example.com www.example.com;
    ...
}
```

在此配置中，nginx 仅测试请求头字段`Host`，以确定请求应路由到哪个服务器。如果 `Host`值与任何`server_name`都不匹配，或者请求根本不包含此标头字段，则 nginx 会将请求路由到此端口的默认服务器。在上面的配置中，默认服务器是第一个（这是 nginx 的默认行为）。可以使用`listen`指令中的`default_server`参数显式设置默认服务器：

```conf
server {
    listen      80 default_server;              # default_server 显示地指定默认服务器
    server_name example.net www.example.net;
    ...
}
```

> `default_server` 参数从 0.8.21 版本开始可用。在早期版本中，应使用 `default` 参数。

**注意！**，`default_server` 是 `listen` 的属性，而不是 `server_name` 的属性。

## 如何防止处理未定义服务器名称的请求

如果不允许没有`Host`请求头字段的请求，则可以定义仅用作删除请求的服务器：

```conf
server {
    listen      80;
    server_name "";   # 服务器名称置为空字符串，用来匹配不带Host请求头的请求
    return      444;
}
```

这里，`server_name` 设置为一个空字符串，它将匹配没有 “Host” 请求头字段的请求，并返回一个特殊的 nginx 的非标准代码 444 来关闭连接。

> 从0.8.48版本开始，上门的这种配置是 `server_name` 的默认设置，因此可以省略 `server_name ""`。而在早期版本中，计算机的主机名用作默认服务器名称，所以需要显示地配置 `server_name ""`。

## 基于名称和基于 IP 的混合虚拟服务器

下面是更复杂的配置，其中一些虚拟服务器侦听不同的地址：

```conf
server {
    listen      192.168.1.1:80;
    server_name example.org www.example.org;
    ...
}

server {
    listen      192.168.1.1:80;
    server_name example.net www.example.net;
    ...
}

server {
    listen      192.168.1.2:80;
    server_name example.com www.example.com;
    ...
}
```

在此配置中，nginx 首先根据 `server` 块的 `listen` 指令测试请求的 IP 地址和端口。然后根据与 IP 地址和端口匹配的 `server` 块的 `server_name` 条目来测试请求的 “Host” 请求头字段。如果未找到 `server_name` ，则请求将由默认服务器处理。例如，在 192.168.1.1:80 端口上收到的对 `www.example.com` 的请求将由 192.168.1.1:80 端口的默认服务器（即第一台服务器）处理，因为没有 `www.example.com` 为此端口定义。

如前所述，默认服务器是 `listen port` 的一个属性，可以为不同的端口定义不同的默认服务器：

```conf
server {
    listen      192.168.1.1:80;
    server_name example.org www.example.org;
    ...
}

server {
    listen      192.168.1.1:80 default_server;
    server_name example.net www.example.net;
    ...
}

server {
    listen      192.168.1.2:80 default_server;
    server_name example.com www.example.com;
    ...
}
```

## 一个简单的 PHP 站点配置

nginx 如何选择一个位置来处理一个典型的简单 PHP 站点的请求：

```conf
server {
    listen      80;
    server_name example.org www.example.org;
    root        /data/www;

    location / {
        index   index.html index.php;
    }

    location ~* \.(gif|jpg|png)$ {
        expires 30d;
    }

    location ~ \.php$ {
        fastcgi_pass  localhost:9000;
        fastcgi_param SCRIPT_FILENAME
                      $document_root$fastcgi_script_name;
        include       fastcgi_params;
    }
}
```

nginx 首先搜索最具体的 location 前缀（和顺序无关）。在上面的配置中，唯一的 location 前缀是 “/”，由于它能匹配任何的请求，因此将用作最后的处理手段。然后 nginx 按照配置文件中列出的顺序检查以正则表达式给出的 location 。遇到第一个匹配的表达式会停止继续搜索，nginx 将使用该 location。如果没有正则表达式与请求匹配，则 nginx 使用之前找到的最具体的 location。

注意，所有类型的 location 仅测试不带参数的请求行的 URI 部分。这样做是因为查询字符串中的参数多种多样的，例如：

```text
/index.php?user=john&page=1
/index.php?page=1&user=john
```

此外，任何人都可以在查询字符串请求任何内容：

```text
/index.php?page=1&something+else&user=john
```

我们看看在上面的配置中如何处理请求：

- 请求 `/logo.gif` 首先与 `location /` 匹配，然后与正则表达式 `\.(gif|jpg|png)$` 匹配，所以将由正则表达式对应的 location 进行处理。使用指令 `root /data/www`，请求被映射到文件 `/data/www/logo.gif`，并且该文件被发送到客户端。
- 请求 `/index.php` 也首先与 `location /` 匹配，然后与正则表达式 `\.(php)$` 匹配。所以由正在表达式对应的 location 处理，并将请求传递到监听 `localhost:9000` 的 FastCGI 服务器。`fastcgi_param` 指令将 FastCGI 参数 `SCRIPT_FILENAME` 设置为 `/data/www/index.php`，FastCGI 服务器将执行该文件。变量 `$document_root` 等于根指令的值，变量 `$fastcgi_script_name` 等于请求 URI，即 `/index.php`。
- 请求 `/about.html` 仅与 `location /` 匹配，因此在该 location 处理。使用指令 `root /data/www`，请求被映射到文件 `/data/www/about.html`，并且该文件被发送到客户端。
- 处理请求 `/` **比较复杂**。仅与 `location \` 匹配，因此由该 location 处理。然后 `index` 指令根据其参数和 `root /data/www` 指令测试索引文件是否存在。如果文件 `/data/www/index.html` 不存在，并且文件 `/data/www/index.php` 存在，则该指令会执行**内部重定向**到 `/index.php`，nginx 将再次搜索这些 locations，就像请求是从客户端发送的一样。重定向的请求最终将由 FastCGI 服务器处理。
