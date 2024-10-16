using System;
using System.Net;
using System.Net.Sockets;
using System.Text;

class AsyncTcpServer
{
    private Socket _listenSocket;
    private const int Port = 8080;

    public void Start()
    {
        // 初始化监听套接字
        _listenSocket = new Socket(AddressFamily.InterNetwork, SocketType.Stream, ProtocolType.Tcp);
        _listenSocket.Bind(new IPEndPoint(IPAddress.Any, Port));
        _listenSocket.Listen(100); // 设置等待队列大小

        Console.WriteLine($"Server started on port {Port}...");

        // 开始接受连接
        StartAccept();
    }

    private void StartAccept()
    {
        // 创建 SocketAsyncEventArgs 对象用于异步接受连接
        SocketAsyncEventArgs acceptEventArg = new SocketAsyncEventArgs();
        acceptEventArg.Completed += AcceptCompleted;

        // 异步接受连接
        if (!_listenSocket.AcceptAsync(acceptEventArg))
        {
            // 如果操作立即完成，直接调用完成处理函数
            ProcessAccept(acceptEventArg);
        }
    }

    private void AcceptCompleted(object sender, SocketAsyncEventArgs e)
    {
        // 处理接受连接完成
        ProcessAccept(e);
    }

    private void ProcessAccept(SocketAsyncEventArgs e)
    {
        Socket clientSocket = e.AcceptSocket;

        if (clientSocket != null && clientSocket.Connected)
        {
            Console.WriteLine("Client connected.");

            // 创建新的 SocketAsyncEventArgs 用于接收数据
            SocketAsyncEventArgs receiveEventArg = new SocketAsyncEventArgs();
            receiveEventArg.SetBuffer(new byte[1024], 0, 1024);
            receiveEventArg.Completed += IOCompleted;

            // 异步接收数据
            if (!clientSocket.ReceiveAsync(receiveEventArg))
            {
                // 如果操作立即完成，直接处理接收数据
                ProcessReceive(receiveEventArg);
            }

            // 继续接受新的连接
            StartAccept();
        }
    }

    private void IOCompleted(object sender, SocketAsyncEventArgs e)
    {
        // 根据操作类型处理不同的结果
        switch (e.LastOperation)
        {
            case SocketAsyncOperation.Receive:
                ProcessReceive(e);
                break;
            case SocketAsyncOperation.Send:
                ProcessSend(e);
                break;
        }
    }

    private void ProcessReceive(SocketAsyncEventArgs e)
    {
        // 获取接收到的数据
        Socket clientSocket = e.AcceptSocket;
        int bytesRead = e.BytesTransferred;

        if (bytesRead > 0 && e.SocketError == SocketError.Success)
        {
            // 处理接收到的数据
            string receivedData = Encoding.UTF8.GetString(e.Buffer, e.Offset, bytesRead);
            Console.WriteLine("Received: " + receivedData);

            // 准备发送响应
            byte[] sendData = Encoding.UTF8.GetBytes("Echo: " + receivedData);
            e.SetBuffer(sendData, 0, sendData.Length);

            // 异步发送响应
            if (!clientSocket.SendAsync(e))
            {
                ProcessSend(e);
            }
        }
        else
        {
            // 关闭客户端连接
            clientSocket.Shutdown(SocketShutdown.Both);
            clientSocket.Close();
        }
    }

    private void ProcessSend(SocketAsyncEventArgs e)
    {
        // 发送完成，继续接收新的数据
        Socket clientSocket = e.AcceptSocket;

        // 清空缓冲区，准备接收下一次的数据
        e.SetBuffer(new byte[1024], 0, 1024);

        if (!clientSocket.ReceiveAsync(e))
        {
            ProcessReceive(e);
        }
    }
}
