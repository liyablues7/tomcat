# 25. Advanced IO and Tomcat

## 目录  

- 简介  
- Comet 支持
	1. Comet 事件
	2. Comet 过滤器  
	3. 范例代码  
	4. Comet 超时  
- 异步写操作  

## 简介  

由于基于 APR 或 NIO API 来构建连接器，Tomcat 能在通常的阻塞 IO 之上提供一些扩展，从而支持 Servlet API。  

**重要说明：这些特性需要使用 APR 或 NIO HTTP 连接器。经典的 java.io HTTP 连接器 与 AJP 连接器并不支持它们。**  

## Comet 支持  

Comet 支持能让 Servlet 实现：对 IO 的异步处理；当连接可以读取数据时，接收事件（而不是总使用阻塞读取）；将数据异步地写入连接（很可能是响应其他一些源所产生的事件）。  

### 1. Comet 事件 

根据发生的具体事件，实现 `org.apache.catalina.comet.CometProcessor` 接口的 Servlet 将调用自己的事件方法，而非通常的服务方法。事件对象允许访问常见的请求与响应对象，使用方式与通常方式相同。主要的区别在于：在处理 BEGIN 事件到 END 或 ERROR 事件之间，这些事件对象能够保持有效和完整的功能性。事件类型如下：    

- `EventType.BEGIN` 在连接处理开始时被调用，用来初始化使用了请求和响应对象的相关字段。从处理完该事件后直到 END 或 ERROR 事件开始处理时的这段时间内，有可能使用响应对象在开放连接中写入数据。注意，响应对象以及所依赖的 OutputStream 和 Writer 仍不能同步，因此在通过多个线程访问它们时，需要进行强制实现同步操作。处理完初始化事件后，就可以提交请求对象了。  
- `EventType.READ` 该事件表明可以使用输入数据，读取过程不会阻塞。可以使用 InputStream 或 Reader 的 available 和 ready 方法来确定是否存在阻塞危险：当数据被报告可读时，Servlet 应该进行读取。当读取遇到错误时，servlet 可以通过正确传播 exception 属性来报告这一情况。抛出异常会导致 ERROR 事件的调用，连接就会关闭。另外，也有可能捕获一个异常，在 servlet 可能使用的数据结构上进行清理，然后使用事件的 close 方法。不允许从servlet对象执行方法外部去读取数据。  
	
	在一些平台（比如 Windows）上，利用 READ 事件来表示客户端断开连接。从流中读取的结果可能是 -1、IOException 异常或 EOFException异常。一定要正确处理这些情况。如果你没有捕捉到 IOException 异常，那么当 Tomcat 捕获到异常时，它会立刻调用你的事件队列生成一个ERROR 事件来存储这些错误，并且你会马上收到这个消息。  
	
- `EventType.END` 请求处理完毕时，就会调用 END 方法。Begin 方法初始化的字段也将被重置。在处理完这一事件后，请求和响应对象，以及它们所依赖的对象，都将被回收，以便再去处理其他请求。当数据可读取时，以及到达请求输入的文件末尾时（这通常表明客户端通过管线提交请求），也会调用 END。  
- `EventType.ERROR`：当连接上出现 IO 异常或类似的不可回收的错误时，容器就会调用 ERROR。在开始时候被初始化的字段在这时候被重置。在处理完这一事件后，请求和响应对象，以及它们所依赖的对象，都将被回收，以便再去处理其他请求。  


下面是一些事件子类别，通过它们可以对事件处理过程进行微调（注意：其中有些事件可能需要使用 `org.apache.catalina.valves.CometConnectionManagerValve` 值）：  

- `EventSubType.TIMEOUT`: 连接超时（`ERROR` 的子类别）。注意，这个 ERROR 子类型并不是必须的。除非 servlet 使用该事件的 close 方法，否则连接将不会关闭。
- `EventSubType.CLIENT_DISCONNECT`：客户端连接被关闭（`ERROR` 的子类别）。
- `EventSubType.IOEXCEPTION`：表示发生了 IO 异常（比如无效内容），例如无效的块阻塞（`ERROR` 的子类别）。
- `EventSubType.WEBAPP_RELOAD`：重新加载 Web 应用（`END` 的子类别）。
- `EventSubType.SESSION_END`：servlet 终止了会话（`END` 的子类别）。  

如上所述，Comet 请求的典型生命周期会包含一系列的事件：BEGIN -> READ -> READ -> READ -> ERROR/TIMEOUT。任何时候，servlet 都能用事件的 close 方法来终止对请求的处理。


### 2. Comet 过滤器  

跟一般的过滤器一样，当处理 Comet 事件时，就会调用一个过滤器队列。这些过滤器应该实现 `CometFilter` 接口（和常用的过滤器接口一样），在部署描述符文件中的声明与映像也都和通常的过滤器一样。当过滤器队列在处理事件时，它将只含有那些跟所有通常映射规则相匹配的过滤器，并且这些过滤器要实现 CometFilter 接口。  


### 3. 范例代码  

在下面的范例伪码中，通过使用上文所述的 API，servlet 实现了异步聊天功能。  

```  
public class ChatServlet
    extends HttpServlet implements CometProcessor {

    protected ArrayList<HttpServletResponse> connections =
        new ArrayList<HttpServletResponse>();
    protected MessageSender messageSender = null;

    public void init() throws ServletException {
        messageSender = new MessageSender();
        Thread messageSenderThread =
            new Thread(messageSender, "MessageSender[" + getServletContext().getContextPath() + "]");
        messageSenderThread.setDaemon(true);
        messageSenderThread.start();
    }

    public void destroy() {
        connections.clear();
        messageSender.stop();
        messageSender = null;
    }

    /**
     * Process the given Comet event.
     *
     * @param event The Comet event that will be processed
     * @throws IOException
     * @throws ServletException
     */
    public void event(CometEvent event)
        throws IOException, ServletException {
        HttpServletRequest request = event.getHttpServletRequest();
        HttpServletResponse response = event.getHttpServletResponse();
        if (event.getEventType() == CometEvent.EventType.BEGIN) {
            log("Begin for session: " + request.getSession(true).getId());
            PrintWriter writer = response.getWriter();
            writer.println("<!DOCTYPE html>");
            writer.println("<head><title>JSP Chat</title></head><body>");
            writer.flush();
            synchronized(connections) {
                connections.add(response);
            }
        } else if (event.getEventType() == CometEvent.EventType.ERROR) {
            log("Error for session: " + request.getSession(true).getId());
            synchronized(connections) {
                connections.remove(response);
            }
            event.close();
        } else if (event.getEventType() == CometEvent.EventType.END) {
            log("End for session: " + request.getSession(true).getId());
            synchronized(connections) {
                connections.remove(response);
            }
            PrintWriter writer = response.getWriter();
            writer.println("</body></html>");
            event.close();
        } else if (event.getEventType() == CometEvent.EventType.READ) {
            InputStream is = request.getInputStream();
            byte[] buf = new byte[512];
            do {
                int n = is.read(buf); //can throw an IOException
                if (n > 0) {
                    log("Read " + n + " bytes: " + new String(buf, 0, n)
                            + " for session: " + request.getSession(true).getId());
                } else if (n < 0) {
                    error(event, request, response);
                    return;
                }
            } while (is.available() > 0);
        }
    }

    public class MessageSender implements Runnable {

        protected boolean running = true;
        protected ArrayList<String> messages = new ArrayList<String>();

        public MessageSender() {
        }

        public void stop() {
            running = false;
        }

        /**
         * Add message for sending.
         */
        public void send(String user, String message) {
            synchronized (messages) {
                messages.add("[" + user + "]: " + message);
                messages.notify();
            }
        }

        public void run() {

            while (running) {

                if (messages.size() == 0) {
                    try {
                        synchronized (messages) {
                            messages.wait();
                        }
                    } catch (InterruptedException e) {
                        // Ignore
                    }
                }

                synchronized (connections) {
                    String[] pendingMessages = null;
                    synchronized (messages) {
                        pendingMessages = messages.toArray(new String[0]);
                        messages.clear();
                    }
                    // Send any pending message on all the open connections
                    for (int i = 0; i < connections.size(); i++) {
                        try {
                            PrintWriter writer = connections.get(i).getWriter();
                            for (int j = 0; j < pendingMessages.length; j++) {
                                writer.println(pendingMessages[j] + "<br>");
                            }
                            writer.flush();
                        } catch (IOException e) {
                            log("IOExeption sending message", e);
                        }
                    }
                }

            }

        }

    }

}

```  


### 4. Comet 超时  

如果使用 NIO 连接器，你可以为不同的 comet 连接设置单独的超时。只需设置一个如下所示的请求属性即可设置超时：  

`CometEvent event.... event.setTimeout(30*1000);`  

或 

`event.getHttpServletRequest().setAttribute("org.apache.tomcat.comet.timeout", new Integer(30 * 1000));`  

超时被设置为 30 秒。重要说明：为了设置超时，必须完成 BEGIN 事件。默认值为 `soTimeout`。  

如果使用 APR 连接器，所有的 Comet 连接将拥有统一的超时值：`soTimeout*50`。  

## 异步写操作  

当 APR 或 NIO 可用时，Tomcat 支持使用 sendfile 方式去发送大型静态文件。只要系统负载一增加，就会异步地高效执行写操作。作为一种使用阻塞写操作发送大型响应的替代方式，有可能使用 sendfile 代码来将内容写入静态文件。缓存值将利用这一点将响应数据缓存至文件而非存储在内存中。如果请求属性 `org.apache.tomcat.sendfile.support` 设为 `Boolean.TRUE`，则表示支持 sendfile。  

通过合适的请求属性，任何 servlet 都可以指示 Tomcat 执行 sendfile 调用。正确地设置响应长度也是很有必要的。在使用 sendfile 时，最好确定请求与响应都没有被包装起来。因为稍后连接器本身将发送响应主体，所以不能够过滤响应主体。除了设置 3 个所需的请求属性之外，servlet 不应该发送任何响应数据，但能使用一些能够修改响应报头的方法（比如设定 cookie）。  

- `org.apache.tomcat.sendfile.filename` 作为字符串发送的标准文件名。  
- `org.apache.tomcat.sendfile.start`开始位置偏移值，长整型值。  
- `org.apache.tomcat.sendfile.end` 结束位置偏移值，长整型值。  

除了设置这些属性，还有必要设置内容长度报头。不要指望 Tomcat 来处理，因为你可能已经将数据写入输出流了。  

注意，使用 sendfile 将禁止 Tomcat 可能在响应中执行的压缩操作。
