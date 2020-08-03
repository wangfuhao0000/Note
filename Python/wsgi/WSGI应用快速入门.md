[官方文档](https://uwsgi-docs-zh.readthedocs.io/zh_CN/latest/WSGIquickstart.html)

### 安装uWSGI以及Python支持

uWSGI是一个（大的）C应用，因此，你需要一个**C编译器** (例如gcc或者clang)，以及**Python开发头文件**。

在基于Debian的发行版上按住啊个Python开发头文件：

```shell
apt-get install build-essential python-dev
```

安装uWSGI的方式有多种：

- 通过pip：

```shell
pip install uwsgi
```

- 使用网络安装器

```shell
curl http://uwsgi.it/install | bash -s default /tmp/uwsgi # 将uwsgi安装到/tmp/uwsgi内
```

- 下载源tarball文件，然后执行”make”命令

```shell
wget http://projects.unbit.it/downloads/uwsgi-latest.tar.gz

tar zxvf uwsgi-latest.tar.gz

cd <dir>

make
```

### 第一个WSGI应用

#### 编写application

首先写一个py文件：

```python
def application(env, start_response):
		start_response('200 OK', [('Content-Type','text/html')])
  	return [b"Hello World"]
```

而函数名字是application是因为它是uWSGI Python加载器将会搜索的默认函数（当然可以自定义）。

#### 将其部署在HTTP端口9090

启动uWSGI来运行一个HTTP服务器/路由器，它会传递请求到你的WSGI应用：

```shell
uwsgi --http :9090 --wsgi-file foobar.py
```

> 当你有一个前端web服务器，或者你正进行某些形式的基准时，不要使用 --http ，使用 --http-socket 。

#### 添加并发和监控

默认情况下，uWSGI启动一个单一的进程和一个单一的线程)。所以我们要先增加并发性，可以通过`--process`来增加进程，或者使用`--threads`添加线程（当然可同时添加）

```shell
uwsgi --http :9090 --wsgi-file foobar.py --master --processes 4 --threads 2
```

还有一个重要的任务是监控，stats子系统允许将uWSGI内部统计数据作为JSON导出：

```shell
uwsgi --http :9090 --wsgi-file foobar.py --master --processes 4 --threads 2 --stats 127.0.0.1:9191
```

#### 放在一个完整的web服务器后

uWSGI原生支持HTTP, FastCGI, SCGI及其特定的名为”uwsgi”的协议。最好的协议显然是uwsgi，nginx和Cherokee已经支持它了。一个常用的nginx配置如下：

```shell
location / {
    include uwsgi_params;
    uwsgi_pass 127.0.0.1:3031;
}
```

这表示“传递每一个请求给**绑定到3031端口**并**使用uwsgi协议**的服务器”。

现在，我们可以生成uWSGI来本地使用uwsgi协议：

```shell
uwsgi --socket 127.0.0.1:3031 --wsgi-file foobar.py --master --processes 4 --threads 2 --stats 127.0.0.1:9191
```

如果你要运行 `ps aux` ，那么你会看到一个进程。已经移除了HTTP路由器，因为我们的“worker” (被分配给uWSGI的进程) 本地使用uwsgi协议。

如果你的代理/web服务器/路由器使用HTTP，那么你必须告诉uWSGI本地使用http协议 (这与会自己生成一个代理的–http不同):

```shell
uwsgi --http-socket 127.0.0.1:3031 --wsgi-file foobar.py --master --processes 4 --threads 2 --stats 127.0.0.1:9191
```

#### 部署Django

假设Django工程位于 `/home/foobar/myproject`：

```shell
uwsgi --socket 127.0.0.1:3031 --chdir /home/foobar/myproject/ --wsgi-file myproject/wsgi.py --master --processes 4 --threads 2 --stats 127.0.0.1:9191
```

使用上述命令便可正确加载模块，但并不方便。uWSGI支持多种配置风格，我们先使用.ini文件：

```shell
[uwsgi]
socket = 127.0.0.1:3031
chdir = /home/foobar/myproject/
wsgi-file = myproject/wsgi.py
processes = 4
threads = 2
stats = 127.0.0.1:9191
```

然后我们只需要运行

```shell
uwsgi yourfile.ini
```

#### 部署Flask

假设有下面的Flask的应用：

```python
from flask import Flask

app = Flask(__name__)

@app.route('/')
def index():
    return "<span style='color:red'>I am app 1</span>"
```

Flask将其WSGI函数导出为”app”，因此，我们需要指示uWSGI使用它。我们仍然使用uwsgi socket：

```shell
uwsgi --socket 127.0.0.1:3031 --wsgi-file myflaskapp.py --callable app --processes 4 --threads 2 --stats 127.0.0.1:9191
```

唯一添加的是 `--callable` 选项

