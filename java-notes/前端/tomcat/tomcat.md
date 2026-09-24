# Servlet容器

servlet 容器是一个复杂的系统。不过，一个 servlet 容器要为一个 servlet 的请求提供服务，基本上有三件事要做：

- 创建一个 request 对象并填充那些有可能被所引用的 servlet 使用的信息，如参数、头部、cookies、查询字符串、URI 等等。一个 request 对象是 javax.servlet.ServletRequest或javax.servlet.http.ServletRequest接口的一个实例。
- 创建一个 response 对象，所引用的 servlet 使用它来给客户端发送响应。一个 response 对象 javax.servlet.ServletResponse 或 javax.servlet.http.ServletResponse 接口 的一个实例。

- 调用 servlet 的 service 方法，并传入 request 和 response 对象。在这里 servlet 会 从 request 对象取值，给 response 写值。

# Catalina架构

两个模块：容器+连接器

连接器接收HTTP请求，构造 request 和 response 对象，传递给容器。然后容器调用 servelet 的 service 方法用于响应。

# Socket demo

client:

```java
package tcpUdpThread;

import java.io.BufferedReader;
import java.io.InputStream;
import java.io.InputStreamReader;
import java.io.OutputStream;
import java.io.PrintWriter;
import java.net.InetAddress;
import java.net.Socket;

public class TCP_Client {

	public static void main(String[] args) throws Exception{
		// TODO Auto-generated method stub
		String userInput = null;
		String echoMessage = null;

		BufferedReader stdIn = new BufferedReader(new InputStreamReader(System.in));

		Socket socket = new Socket(InetAddress.getLocalHost(), 23333);
		System.out.println("Connected to Server");

		InputStream inStream = socket.getInputStream();
		OutputStream outStream = socket.getOutputStream();
		BufferedReader in = new BufferedReader(new InputStreamReader(inStream));
		PrintWriter out = new PrintWriter(outStream);


		while((userInput=stdIn.readLine())!=null)
        {
            out.println(userInput);
			out.flush();
			echoMessage = in.readLine();
			System.out.println("Echo from server: " + echoMessage);
		}
		socket.close();
	}
}
```

server:

```java
package tcpUdpThread;

import java.io.BufferedReader;
import java.io.IOException;
import java.io.InputStream;
import java.io.InputStreamReader;
import java.io.OutputStream;
import java.io.PrintWriter;
import java.net.ServerSocket;
import java.net.Socket;
import java.util.concurrent.ArrayBlockingQueue;
import java.util.concurrent.ThreadPoolExecutor;
import java.util.concurrent.TimeUnit;

public class TCP_Server_ThreadPool {

	public static void main(String[] args) throws Exception {
		// ServerSocket代表一个监听客户端socket的套接字
		ServerSocket listenSocket=new ServerSocket(23333);
		ThreadPoolExecutor executor = new ThreadPoolExecutor(5, 10, 200, TimeUnit.MILLISECONDS,
                new ArrayBlockingQueue<Runnable>(5));
        Socket socket=null;

        int count=0;
        System.out.println("Server listening at 23333");

        while(true){
            socket=listenSocket.accept();
			count++;
			System.out.println("The total number of clients now is " + count + ".");
            ServerTask serverTask=new ServerTask(socket);
            executor.execute(serverTask);
        }
	}
}
class ServerTask implements Runnable {

    Socket socket = null;

    public ServerTask(Socket socket) {
        this.socket = socket;
    }

    public void run(){
	InputStream inStream = null;
		OutputStream outStream = null;
		BufferedReader in = null;
		PrintWriter out = null;
        try {
	inStream = socket.getInputStream();
		outStream = socket.getOutputStream();
		in = new BufferedReader(new InputStreamReader(inStream));
		out = new PrintWriter(outStream);
            String info=null;
            while((info=in.readLine())!=null){
                System.out.println("Message from client:"+info);
				out.println(info+", server thread id: "+Thread.currentThread().getId());
				out.flush();
            }
			Thread.sleep(4000000);
        } catch (Exception e) {
            e.printStackTrace();
        }finally{
            try {
                if(in!=null)
                    in.close();
                if(out!=null)
                    out.close();
                if(inStream!=null)
	inStream.close();
                if(outStream!=null)
	outStream.close();
                if(socket!=null)
                    socket.close();
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
    }
}
```

# HTTP demo

HTTP server:

```java
package ex01.pyrmont;

import java.net.Socket;
import java.net.ServerSocket;
import java.net.InetAddress;
import java.io.InputStream;
import java.io.OutputStream;
import java.io.IOException;
import java.io.File;

public class HttpServer {

    /** WEB_ROOT is the directory where our HTML and other files reside.
   *  For this package, WEB_ROOT is the "webroot" directory under the working
   *  directory.
   *  The working directory is the location in the file system
   *  from where the java command was invoked.
   */
    public static final String WEB_ROOT =
        System.getProperty("user.dir") + File.separator  + "webroot";

    // shutdown command
    private static final String SHUTDOWN_COMMAND = "/SHUTDOWN";

    // the shutdown command received
    private boolean shutdown = false;

    public static void main(String[] args) {
        HttpServer server = new HttpServer();
        server.await();
    }

    public void await() {
        ServerSocket serverSocket = null;
        int port = 8080;
        try {
            serverSocket =  new ServerSocket(port, 1, InetAddress.getByName("127.0.0.1"));
        }
        catch (IOException e) {
            e.printStackTrace();
            System.exit(1);
        }

        // Loop waiting for a request
        while (!shutdown) {
            Socket socket = null;
            InputStream input = null;
            OutputStream output = null;
            try {
                socket = serverSocket.accept();
                input = socket.getInputStream();
                output = socket.getOutputStream();

                // create Request object and parse
                Request request = new Request(input);
                request.parse();

                // create Response object
                Response response = new Response(output);
                response.setRequest(request);
                response.sendStaticResource();

                // Close the socket
                socket.close();

                //check if the previous URI is a shutdown command
                shutdown = request.getUri().equals(SHUTDOWN_COMMAND);
            }
            catch (Exception e) {
                e.printStackTrace();
                continue;
            }
        }
    }
}
```

HTTP request:

```java
package ex01.pyrmont;

import java.io.InputStream;
import java.io.IOException;

public class Request {

    private InputStream input;
    private String uri;

    public Request(InputStream input) {
        this.input = input;
    }

    public void parse() {
        // Read a set of characters from the socket
        StringBuffer request = new StringBuffer(2048);
        int i;
        byte[] buffer = new byte[2048];
        try {
            i = input.read(buffer);
        }
        catch (IOException e) {
            e.printStackTrace();
            i = -1;
        }
        for (int j=0; j<i; j++) {
            request.append((char) buffer[j]);
        }
        System.out.print(request.toString());
        //设置URI
        uri = parseUri(request.toString());
    }
	//取HTTP请求报文中第一行，请求类型和HTTP协议版本中间的URI
    private String parseUri(String requestString) {
        int index1, index2;
        index1 = requestString.indexOf(' ');
        if (index1 != -1) {
            index2 = requestString.indexOf(' ', index1 + 1);
            if (index2 > index1)
                return requestString.substring(index1 + 1, index2);
        }
        return null;
    }

    public String getUri() {
        return uri;
    }
}
```

HTTP response:

```java
package ex01.pyrmont;

import java.io.OutputStream;
import java.io.IOException;
import java.io.FileInputStream;
import java.io.File;

/*
  HTTP Response = Status-Line
    *(( general-header | response-header | entity-header ) CRLF)
    CRLF
    [ message-body ]
    Status-Line = HTTP-Version SP Status-Code SP Reason-Phrase CRLF
*/

public class Response {

    private static final int BUFFER_SIZE = 1024;
    Request request;
    OutputStream output;

    public Response(OutputStream output) {
        this.output = output;
    }

    public void setRequest(Request request) {
        this.request = request;
    }

    public void sendStaticResource() throws IOException {
        byte[] bytes = new byte[BUFFER_SIZE];
        FileInputStream fis = null;
        try {
            File file = new File(HttpServer.WEB_ROOT, request.getUri());
            if (file.exists()) {
                fis = new FileInputStream(file);
                int ch = fis.read(bytes, 0, BUFFER_SIZE);
                while (ch!=-1) {
                    output.write(bytes, 0, ch);
                    ch = fis.read(bytes, 0, BUFFER_SIZE);
                }
            }
            else {
                // file not found
                String errorMessage = "HTTP/1.1 404 File Not Found\r\n" +
                    "Content-Type: text/html\r\n" +
                    "Content-Length: 23\r\n" +
                    "\r\n" +
                    "<h1>File Not Found</h1>";
                output.write(errorMessage.getBytes());
            }
        }
        catch (Exception e) {
            // thrown if cannot instantiate a File object
            System.out.println(e.toString() );
        }
        finally {
            if (fis!=null)
                fis.close();
        }
    }
}
```

# 一个简单的Servlet容器

Servelet编程是通过`javax.servlet`和`javax.servlet.http`这两个包的类和接口来实现的。

```java
public interface Servlet {
    // 在servlet容器初始化后被调用一次，init方法必须在servlet容器接收到任何请求之前执行完成
    void init(ServletConfig var1) throws ServletException;
	// 这个方法会返回由Servlet容器传给init（ ）方法的ServletConfig对象。
    ServletConfig getServletConfig();
	// servlet容器为servlet请求调用service方法，在servlet的生命周期中，它会被多次调用
    // 接收一个ServletRequest参数和ServletResponse参数
    void service(ServletRequest var1, ServletResponse var2) throws ServletException, IOException;
	// 这个方法会返回Servlet的一段描述，可以返回一段字符串。
    String getServletInfo();
	// 当从服务中移除一个servlet实例时，servlet容器需要调用destroy方法
    void destroy();
}
```

servlet容器工作流程：

- 第一次调用servlet构造方法时，加载该Servlet类并调用init方法
- 对每次请求，构造一个ServletRequest实例和一个ServletResponse实例
- 传递ServletRequest和ServletResponse参数，执行service方法
- 当servlet类关闭时执行destroy方法
