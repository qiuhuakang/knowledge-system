#### signal(SIGPIPE, SIG_IGN)
当服务器close一个连接时，若client端接着发数据。根据TCP协议的规定，会收到一个RST响应，client再往这个服务器发送数据时，系统会发出一个SIGPIPE信号给进程，告诉进程这个连接已经断开了，不要再写了。
如:    signal(SIGPIPE,SIG_IGN)，这时SIGPIPE交给了系统处理。
服务器采用了fork的话，要收集垃圾进程，防止僵尸进程的产生，可以这样处理：signal(SIGCHLD,SIG_IGN);　交给系统init去回收。这里子进程就不会产生僵尸进程了
在Linux下测试，然后用C++写了客户端用千万级别数量的短链接进行压力测试. 但是服务器总是莫名退出，没有core文件. 
最后问题确定为, 对一个对端已经关闭的socket调用两次write, 第二次将会生成SIGPIPE信号, 该信号默认结束进程. 具体的分析可以结合TCP的”四次握手”关闭.
TCP是全双工的信道, 可以看作两条单工信道, TCP连接两端的两个端点各负责一条. 当对端调用close时, 虽然本意是关闭整个两条信道, 但本端只是收到FIN包. 
按照TCP协议的语义, 表示对端只是关闭了其所负责的那一条单工信道, 仍然可以继续接收数据. 也就是说, 因为TCP协议的限制, 一个端点无法获知对端的socket是调用了close还是shutdown.
对一个已经收到FIN包的socket调用read方法, 如果接收缓冲已空, 则返回0, 这就是常说的表示连接关闭. 但第一次对其调用write方法时, 如果发送缓冲没问题, 会返回正确写入(发送). 
但发送的报文会导致对端发送RST报文, 因为对端的socket已经调用了close, 完全关闭, 既不发送, 也不接收数据. 
所以, 第二次调用write方法(假设在收到RST之后), 会生成SIGPIPE信号, 导致进程退出. 为了避免进程退出, 可以捕获SIGPIPE信号, 或者忽略它,
给它设置SIG_IGN信号处理函数: signal(SIGPIPE, SIG_IGN); 这样, 第二次调用write方法时, 会返回-1, 同时errno置为SIGPIPE. 程序便能知道对端已经关闭. 
在linux下写socket的程序的时候，如果尝试send到一个disconnected socket上，就会让底层抛出一个SIGPIPE信号。 
这个信号的缺省处理方法是退出进程，大多数时候这都不是我们期望的。因此我们需要重载这个信号的处理方法。
调用以下代码，即可安全的屏蔽SIGPIPE： signal （SIGPIPE， SIG_IGN）； 
我的程序产生这个信号的原因是: client端通过 pipe 发送信息到server端后，就关闭client端, 这时server端,返回信息给 client 端时就产生Broken pipe 信号了，服务器就会被系统结束了。 
对于产生信号，我们可以在产生信号前利用方法 signal(int signum, sighandler_t handler) 设置信号的处理。如果没有调用此方法，
系统就会调用默认处理方法：中止程序，显示提示信息(就是我们经常遇到的问题)。我们可以调用系统的处理方法，也可以自定义处理方法。 
系统里边定义了三种处理方法： 
(1)SIG_DFL信号专用的默认动作: 　　 
(a)如果默认动作是暂停线程，则该线程的执行被暂时挂起。当线程暂停期间，发送给线程的任何附加信号都不交付，直到该线程开始执行，但是SIGKILL除外。 　　 
(b)把挂起信号的信号动作设置成SIG_DFL，且其默认动作是忽略信号 (SIGCHLD)。 
(2)SIG_IGN忽略信号 　　 
(a)该信号的交付对线程没有影响 　 
(b)系统不允许把SIGKILL或SIGTOP信号的动作设置为SIG_DFL 3)SIG_ERR 项目中我调用了signal(SIGPIPE, SIG_IGN), 这样产生 SIGPIPE 信号时就不会中止程序，直接把这个信号忽略掉。
