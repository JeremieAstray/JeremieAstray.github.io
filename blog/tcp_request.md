## tcp请求建立连接，结束连接握手过程
建立连接握手过程（3次）：  
1、客户端向服务端发出SYN(x)报文  
2、服务端返回ACK(x+1)-SYN(y)报文  
3、客户端再向服务端发出ACK(y+1)，客户端确定连接已经建立，当服务端收到后也确定连接已经建立  
之后就可以发送数据了  

结束连接握手过程（4次）：  
1、客户端向服务端发出FIN报文  
2、服务端收到FIN报文后向客户端发出应答ACK报文  
3、服务端向客户端发出FIN报文  
4、客户端收到FIN报文后向服务端发出ACK报文  
服务端抽到最后一个ACK报文后连接终止，客户端发出FIN报文后终止  

注：结束连接握手过程其实为可逆，客户端和服务端的关系可以反转  