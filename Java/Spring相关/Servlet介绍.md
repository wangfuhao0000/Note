## Servlet生命周期

Servlet生命周期可被定义为从创建直到毁灭的整个过程，如下：

- Servlet通过调用`init()`方法进行初始化
- Servlet调用`service()`方法处理客户端的请求
- Servlet通过调用`destroy()`方法终止（结束）
- 最后，Servlet是由JVM的垃圾回收器进行回收的

然后我们详细讨论一下各个生命周期

### init()方法

此方法被设计成**只调用一次，第一次创建Servlet时被调用，后续每次用户请求时不再调用**。Servlet创建于用户第一次调用对应于该Servlet的URL时，调用一个Servlet时就会创建一个Servlet实例，**每个用户请求都会产生一个新的线程**，适当的时候交给`doGet()`或者`doPos()`t方法。**init方法简单的创建或加载一些数据，这些数据被用于Servlet的整个生命周期**。

### service()方法

它是执行任务的主要方法，Servlet容器调用`service()`方法来处理来自客户端的请求，并把格式化的响应写回给客户端。**每次服务器接收到一个Servlet请求时，服务器就会产生一个新的线程并调用服务**。`service()`方法检查HTTP请求类型，并在适当的时候调用doGet、doPost、doPut等方法。

我们最常用的就是**doGet()**方法和**doPost**方法

### destroy()方法

**只会被调用一次**，在Servlet生命周期结束时被调用，它可以让您的Servlet关闭数据库连接、停止后台线程、把Cookie列表写入到磁盘等类似的清理活动。

`destory()` 方法被调用后，servlet 被销毁，**但是并没有立即被回收，再次请求时，并没有重新初始化**。

### 架构图

下图显示了一个典型的 Servlet 生命周期方案。

- 第一个到达服务器的 HTTP 请求被委派到 Servlet 容器。
- Servlet 容器在调用 service() 方法之前加载 Servlet。
- 然后 Servlet 容器处理由多个线程产生的多个请求，**每个线程执行一个单一的 Servlet 实例的 service() 方法**。

![img](https://note.youdao.com/yws/public/resource/c26f28c1bdaa229936f4dc10ffdd5934/xmlnote/WEB6b3c167bb6a73b2d2dd764f49fbc4844/E56F50B52ABA46C486DFDDE609CCF8A9/22846)

## Servlet实例

我们在使用Servlet时通常是继承`javax.servlet.HttpServlet`来编写自己的Servlet，并实现相应的方法来处理HTTP请求，如下：

```java
public class HelloWorld extends HttpServlet {

    private String message;

    @Override
    public void init() throws ServletException {
        message = "Hello World";
    }

    @Override
    protected void doGet(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {
        // 设置响应内容类型
        resp.setContentType("text/html");

        // 实际的逻辑是在这里
        PrintWriter out = resp.getWriter();
        out.println("<h1>" + message + "</h1>");
    }

    @Override
    protected void doPost(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {
        this.doGet(req, resp);
    }

    @Override
    public void destroy() {
        // 销毁
    }
}
```

### Servlet部署

在部署前需要先编译Servlet，可以选择使用`javac`进行编译，但要保证所需要的类都在类路径中；也可以选择使用IDEA来直接编译为对应的Class文件。默认情况下，Servlet 应用程序位于路径` <Tomcat-installation-directory>/webapps/ROOT `下，且类文件放在 `<Tomcat-installation-directory>/webapps/ROOT/WEB-INF/classes` 中。如果您有一个完全合格的类名称 **com.myorg.MyServlet**，那么这个 Servlet 类必须位于 **WEB-INF/classes/com/myorg/MyServlet.class** 中。

现在，让我们把 HelloWorld.class 复制到 `<Tomcat-installation-directory>/webapps/ROOT/WEB-INF/classes` 中，并在位于 `<Tomcat-installation-directory>/webapps/ROOT/WEB-INF/` 的 **web.xml** 文件中创建以下条目：

```xml-dtd
<servlet>  <!--先定义一个servlet-->
    <servlet-name>HelloWorld</servlet-name>
    <servlet-class>HelloWorld</servlet-class>
</servlet>

<servlet-mapping>  <!--然后决定把这个servlet映射到哪个url上-->
    <servlet-name>HelloWorld</servlet-name>
    <url-pattern>/HelloWorld</url-pattern>
</servlet-mapping>
```

这样启动完Tomcat后，就可以直接访问对应的url（/HelloWorld）来访问对应Servlet的`doGet()`方法了。

### 使用 Servlet 读取表单数据

Servlet 处理表单数据，这些数据会根据不同的情况使用不同的方法自动解析：

- **getParameter()：**您可以调用 request.getParameter() 方法来获取表单参数的值。
- **getParameterValues()：**如果参数出现一次以上，则调用该方法，并返回多个值，例如复选框。
- **getParameterNames()：**如果您想要得到当前请求中的所有参数的完整列表，则调用该方法。

## Servlet过滤器

Servlet 过滤器可以动态地拦截请求和响应，以变换或使用包含在请求或响应中的信息。可以将一个或多个 Servlet 过滤器附加到一个 Servlet 或一组 Servlet。调用 Servlet 前调用所有附加的 Servlet 过滤器。Servlet 过滤器是可用于 Servlet 编程的 Java 类，可以实现以下目的：

- 在客户端的请求访问后端资源之前，拦截这些请求。
- 在服务器的响应发送回客户端之前，处理这些响应。

根据规范建议的各种类型的过滤器：

- 身份验证过滤器（Authentication Filters）。
- 数据压缩过滤器（Data compression Filters）。
- 加密过滤器（Encryption Filters）。
- 触发资源访问事件过滤器。
- 图像转换过滤器（Image Conversion Filters）。
- 日志记录和审核过滤器（Logging and Auditing Filters）。
- MIME-TYPE 链过滤器（MIME-TYPE Chain Filters）。
- 标记化过滤器（Tokenizing Filters）。
- XSL/T 过滤器（XSL/T Filters），转换 XML 内容。

过滤器通过 Web 部署描述符（web.xml）中的 XML 标签来声明，然后映射到您的应用程序的部署描述符中的 Servlet 名称或 URL 模式。当 Web 容器启动 Web 应用程序时，它会**为您在部署描述符中声明的每一个过滤器创建一个实例。Filter的执行顺序与在web.xml配置文件中的配置顺序一致，一般把Filter配置在所有的Servlet之前**。

### Servlet过滤器方法

过滤器是一个实现了 `javax.servlet.Filter` 接口的 Java 类。此接口定义了三个方法：

- `public void doFilter (ServletRequest, ServletResponse, FilterChain)`：该方法**完成实际的过滤操作**，当客户端请求方法与过滤器设置匹配的URL时，Servlet容器将先调用过滤器的doFilter方法。FilterChain用户访问后续过滤器。
- `public void init(FilterConfig filterConfig)`：web 应用程序启动时，web 服务器将创建Filter 的实例对象，并调用其init方法，读取web.xml配置，完成对象的初始化功能，从而为后续的用户请求作好拦截的准备工作（**filter对象只会创建一次，init方法也只会执行一次**）。开发人员通过init方法的参数，可获得代表当前filter配置信息的FilterConfig对象。
- `public void destroy()`：Servlet容器在销毁过滤器实例前调用该方法，在该方法中释放Servlet过滤器占用的资源。

#### FilterConfig的使用

Filter 的 init 方法中提供了一个 FilterConfig 对象。假如 web.xml 文件配置如下：

```xml
<filter>
    <filter-name>LogFilter</filter-name>
    <filter-class>com.runoob.test.LogFilter</filter-class>
    <init-param>
        <param-name>Site</param-name>
        <param-value>菜鸟教程</param-value>
    </init-param>
</filter>
```

在 init 方法使用 FilterConfig 对象获取参数：

```java
public void  init(FilterConfig config) throws ServletException {
    // 获取初始化参数
    String site = config.getInitParameter("Site"); 
    // 输出初始化参数
    System.out.println("网站名称: " + site); 
}
```

### Servlet过滤器实例

主要看一下Servlet过滤器在Web.xml中是如何配置的：

```xml
<!-- 首先定义一个过滤器 -->
<filter>
  <filter-name>LogFilter</filter-name>
  <filter-class>com.runoob.test.LogFilter</filter-class>
  <init-param>
    <param-name>Site</param-name>
    <param-value>菜鸟教程</param-value>
  </init-param>
</filter>
<!-- 指定这个过滤器会过滤哪些Servlet -->
<filter-mapping>
  <filter-name>LogFilter</filter-name>
  <url-pattern>/*</url-pattern>
</filter-mapping>
```

上述过滤器适用于所有的 Servlet，因为我们在配置中指定 **/\*** 。如果您只想在少数的 Servlet 上应用过滤器，您可以指定一个特定的 Servlet 路径。

当然也可以使用相同的方式创建多个过滤器，并分别指定各自需要过滤的Servlet。

### 过滤器的应用顺序

web.xml 中的 **filter-mapping 元素的顺序决定了 Web 容器应用过滤器到 Servlet 的顺序**。若要反转过滤器的顺序，您只需要在 web.xml 文件中反转 filter-mapping 元素即可。例如，下面的实例将先应用 AuthenFilter，然后再应用 LogFilter：

```xml
<filter-mapping>
   <filter-name>AuthenFilter</filter-name>
   <url-pattern>/*</url-pattern>
</filter-mapping>

<filter-mapping>
   <filter-name>LogFilter</filter-name>
   <url-pattern>/*</url-pattern>
</filter-mapping>
```

### 过滤器在Web.xml配置各节点说明

- `<filter>`

  指定一个过滤器。

  - `<filter-name>`用于为过滤器指定一个名字，该元素的内容不能为空。
  - `<filter-class>`元素用于指定过滤器的完整的限定类名。
  - `<init-param>`元素用于为过滤器指定初始化参数，它的子元素`<param-name>`指定参数的名字，`<param-value>`指定参数的值。
  - 在过滤器中，可以使用`FilterConfig`接口对象来访问初始化参数。

- `<filter-mapping>`

  元素用于设置一个 Filter 所负责拦截的资源。一个Filter拦截的资源可通过两种方式来指定：**Servlet 名称**和资源访问的**请求路径**

  - `<filter-name>`子元素用于设置filter的注册名称。该值必须是在`<filter>`元素中声明过的过滤器的名字
  - `<url-pattern>`设置 filter 所拦截的请求路径(过滤器关联的URL样式)

- `<servlet-name>`指定过滤器所拦截的Servlet名称。

- `<dispatcher>`指定过滤器所拦截的资源被 Servlet 容器调用的方式，可以是`REQUEST`,`INCLUDE`,`FORWARD`和`ERROR`之一，默认`REQUEST`。用户可以设置多个`<dispatcher>`子元素用来指定 Filter 对资源的多种调用方式进行拦截。

- `<dispatcher>`

  子元素可以设置的值及其意义

  - `REQUEST`：当用户直接访问页面时，Web容器将会调用过滤器。如果目标资源是通过RequestDispatcher的include()或forward()方法访问时，那么该过滤器就不会被调用。
  - `INCLUDE`：如果目标资源是通过RequestDispatcher的include()方法访问时，那么该过滤器将被调用。除此之外，该过滤器不会被调用。
  - `FORWARD`：如果目标资源是通过RequestDispatcher的forward()方法访问时，那么该过滤器将被调用，除此之外，该过滤器不会被调用。
  - `ERROR`：如果目标资源是通过声明式异常处理机制调用时，那么该过滤器将被调用。除此之外，过滤器不会被调用。

## Servlet异常处理

当一个 Servlet 抛出一个异常时，Web 容器在使用了 exception-type 元素的 **web.xml** 中搜索与抛出异常类型相匹配的配置。**您必须在 web.xml 中使用 error-page 元素来指定对特定异常 或 HTTP 状态码 作出相应的 Servlet 调用**。

```xml
<!-- servlet 定义 -->
<servlet>
        <servlet-name>ErrorHandler</servlet-name>
        <servlet-class>ErrorHandler</servlet-class>
</servlet>
<!-- servlet 映射 -->
<servlet-mapping>
        <servlet-name>ErrorHandler</servlet-name>
        <url-pattern>/ErrorHandler</url-pattern>
</servlet-mapping>

<!-- error-code 相关的错误页面 -->
<error-page>
    <error-code>404</error-code>
    <location>/ErrorHandler</location>
</error-page>
<error-page>
    <error-code>403</error-code>
    <location>/ErrorHandler</location>
</error-page>

<!-- exception-type 相关的错误页面 -->
<error-page>
    <exception-type>
          javax.servlet.ServletException
    </exception-type >
    <location>/ErrorHandler</location>  <!-- 出现此异常，则由相应的Servlet进行处理 -->
</error-page>

<error-page>
    <exception-type>java.io.IOException</exception-type >
    <location>/ErrorHandler</location>
</error-page>
```

以下是关于上面的 web.xml 异常处理要注意的点：

- Servlet ErrorHandler 与其他的 Servlet 的定义方式一样，且在 web.xml 中进行配置。
- 如果有错误状态代码出现，不管为 404（Not Found 未找到）或 403（Forbidden 禁止），则会调用 ErrorHandler 的 Servlet。
- 如果 Web 应用程序抛出 *ServletException* 或 *IOException*，那么 Web 容器会调用 ErrorHandler 的 Servlet。
- 您可以定义不同的错误处理程序来处理不同类型的错误或异常。上面的实例是非常通用的，希望您能通过实例理解基本的概念。

## Servlet处理Cookie

识别返回用户包括三个步骤：

- 服务器脚本向浏览器发送一组 Cookie。例如：姓名、年龄或识别号码等。
- 浏览器将这些信息存储在本地计算机上，以备将来使用。
- 当下一次浏览器向 Web 服务器发送任何请求时，浏览器会把这些 Cookie 信息发送到服务器，服务器将使用这些信息来识别用户。

### 通过Servlet读取Cookie

通过 Servlet 设置 Cookie 包括三个步骤：

1. **创建一个 Cookie 对象**：您可以调用带有 cookie 名称和 cookie 值的 Cookie 构造函数，cookie 名称和 cookie 值都是字符串。

```java
Cookie cookie = new Cookie("key","value");
```

请记住，无论是名字还是值，都不应该包含空格或以下任何字符：

```
[ ] ( ) = , " / ? @ : ;
```

2. 设置最大生存周期：您可以使用 `setMaxAge` 方法来指定 cookie 能够保持有效的时间（以秒为单位）。下面将设置一个最长有效期为 24 小时的 cookie。

```java
cookie.setMaxAge(60*60*24); 
```

3. 发送 Cookie 到 HTTP 响应头：您可以使用 `response.addCookie` 来添加 HTTP 响应头中的 Cookie，如下所示：

```java
response.addCookie(cookie);
```

### 通过Servlet读取Cookie

要读取 Cookie，您需要通过调用 **HttpServletRequest 的 `getCookies()`** 方法创建一个 *javax.servlet.http.Cookie* 对象的数组。然后循环遍历数组，并使用 `getName()` 和 `getValue()`方法来访问每个 cookie 和关联的值。

```java
// 获取与该域相关的 Cookie 的数组
cookies = request.getCookies();
for (int i = 0; i < cookies.length; i++){
   cookie = cookies[i];
   if((cookie.getName( )).compareTo("name") == 0 ){
        cookie.setMaxAge(0);
        response.addCookie(cookie);
        out.print("已删除的 cookie：" + 
                     cookie.getName( ) + "<br/>");
   }
   out.print("名称：" + cookie.getName( ) + "，");
   out.print("值：" +  URLDecoder.decode(cookie.getValue(), "utf-8") +" <br/>");
}
```

### 通过Servlet删除Cookie

删除 Cookie 是非常简单的。如果您想删除一个 cookie，那么您只需要按照以下三个步骤进行：

- 读取一个现有的 cookie，并把它存储在 Cookie 对象中。
- 使用 `setMaxAge()` 方法**设置 cookie 的年龄为零**，来删除现有的 cookie。
- 把这个 cookie 添加到响应头。

## Servlet 的Session跟踪

