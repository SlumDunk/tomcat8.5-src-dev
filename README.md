## Welcome to Apache Tomcat!

## day01
    set JVM parameter:
        -Dcatalina.home="/Users/liuzhongda/IdeaProjects/apache-tomcat-8.5.23" // set the running directory, load configuration and lib, logs, and wars
    
    
    --the important components of tomcat
    
    --the progress of bootstrap
        --main()
            --init()
                --initClassLoaders() //null->commonLoader->catalinaLoader, null->commonLoader->sharedLoader
                --Catalina.class //catalinaDaemon [represents <Server></Server>]
                --
            --setAwait()
                Catalina.setAwait();
            --load()
                Catalina.load();
            --start()
                --Catalina.start();
                    --StandardServer.start();
                        --LifeCycleBase.start();
                        --init()
                          --StandardServer.initInternal()
                            --StandardService.init()
                              --StandardService.initInternal();
                                --StandardEngine.init() 
                                    --ContainerBase.initInternal()// JMX配置和初始化tomcat线程池用于启动子容器
                                --Executor.init()
                                    --StandardThreadExecutor.initInternal()
                                --mapperListener.init();
                                --Connector.init();
                                    -- adapter = new CoyoteAdapter(this);
                                    --protocolHandler.setAdapter(adapter);
                                    --protocolHandler.init();
                                        --AbstractProtocol.init();
                                            --AbstractEndpoint.init();
                                                --bind(); //开启serverSocket, 配置Selector
                                                --NioEndPoint.class // 阻塞模式
                                                --NioEndPoint2.class //真正的非阻塞模式
                        --StandardServer.startInternal();
                            --service.start() // StandardService
                                --eigine.start() //实际上调用的都是startInternal()
                                    -- ContainerBase.startInternal()
                                        --  Container children[] = findChildren(); //启动子容器
                                        -- results.add(startStopExecutor.submit(new StartChild(children[i])));
                                        -- threadStart(); //启动本线程 
                                            --ContainerBackgroundProcessor implements Runnable //定期加载本容器和子容器的资源
                                                --processChildren(Container container)
                                                    -- originalClassLoader = ((Context) container).bind(false, null); //设置WebApplicationClassLoader
                                                    --container.backgroundProcess(); //加载webapps StandardContext.class
                                                        --WebappLoader.class
                                                        --WebResourceRoot resources = getResources();
                                                        --resources.backgroundProcess(); //加载完之后清除不必要的缓存
                                                    --  super.backgroundProcess(); // ContainerBase.class, 处理Pipeline, Realm, Cluster    
                                                        
                                                    
                                        
                                --executor.start() //实际上调用的都是startInternal()
                                    //创建线程池，并预热核心线程 
                                    --executor.prestartAllCoreThreads(); // ThreadPoolExecutor
                                --connector.start() //实际上调用的都是startInternal()
                                    --protocolHandler.start(); // AbstractProtocol
                                        -- endpoint.start(); // AbstractEndPoint
                                            --NioEndPoint.startInternal()
                                                 // 128
                                                 //SocketProcessor [NioSocketWrapper, SocketEvent] handshake, SelectionKey.OP_READ, SelectionKey.OP_write
                                                 //Poller.register() event, ConnectionHandler 具体的处理逻辑
                                                 --processorCache = new SynchronizedStack<>(SynchronizedStack.DEFAULT_SIZE,
                                                                    socketProperties.getProcessorCache());
                                                 //PollEvent [Selector.register(), socket, NioSocketWrapper]  run() 对应事件的处理逻辑                                                                  
                                                 --eventCache = new SynchronizedStack<>(SynchronizedStack.DEFAULT_SIZE,
                                                                    socketProperties.getEventCache());
                                                 //对应socketChannel                                                                    
                                                 --nioChannels = new SynchronizedStack<>(SynchronizedStack.DEFAULT_SIZE,
                                                                    socketProperties.getBufferPool());
                                                 //start poller threads                                                                   
                                                 --pollers = new Poller[getPollerThreadCount()]; // 2*cpu cores
                                                    --this.selector = Selector.open();
                                                    --poller.run() // 处理积累的PollEvent, 具体就是处理感兴趣的Selection事件和注册新的感兴趣的Selection事件
                                                        --events()
                                                            --PollerEvent.run()
                                                                --processKey(sk, attachment); //attachment is NioSocketWrapper
                                                                    //处理读，写，上传文件等请求
                                                                    -- processSocket(attachment, SocketEvent.OPEN_READ, true)
                                                                        --SocketProcessorBase<S> sc = processorCache.pop();
                                                                        --executor.execute(sc);//线程池处理
                                                                        or
                                                                        --sc.run();
                                                                            --NioEndPoint.SocketProcessor.doRun()
                                                                                --state = getHandler().process(socketWrapper, SocketEvent.OPEN_READ); // 
                                                                                    --AbstractProtocol.process() //应用层处理 http层
                                                                                        --ContainerThreadMarker.set();//表明线程已经被占用
                                                                                            --state = processor.process(wrapper, status);
                                                                                                --state = dispatch(nextDispatch.getSocketStatus());//转发，对socket进一步修饰
                                                                                                    --state = service(socketWrapper); //真正处理的地方, Http11Processor
                                                                                                        --setSocketWrapper(socketWrapper); //绑定IO 
                                                                                                        --inputBuffer.init(socketWrapper);
                                                                                                        --outputBuffer.init(socketWrapper);
                                                                                                        --prepareRequest();//包装一下tomcat的request变成符合Servelet规范的request, 设置头，校验host, 设置InputFilters
                                                                                                            --rp.setStage(org.apache.coyote.Constants.STAGE_SERVICE);
                                                                                                            --getAdapter().service(request, response);     //CoyotoAdapter.service()
                                                                                                                --req.getRequestProcessor().setWorkerThreadName(THREAD_NAME.get());
                                                                                                                --postParseSuccess = postParseRequest(req, request, res, response);
                                                                                                                    --doConnectorAuthenticationAuthorization(req, request);
                                                                                                                //具体干活的地方了
                                                                                                                --connector.getService().getContainer().getPipeline().getFirst().invoke(
                                                                                                                                          request, response);                              
                                                                                                                   --StandardWrapperValve.invoke()
                                                                                                                    --servlet = wrapper.allocate(); // allocate a servlet instance
                                                                                                                        --instance = loadServlet();
                                                                                                                            --InstanceManager instanceManager = ((StandardContext)getParent()).getInstanceManager();
                                                                                                                            --servlet = (Servlet) instanceManager.newInstance(servletClass);
                                                                                                                            --initServlet(servlet);
                                                                                                                                --servlet.init(facade); //调用Servlet.init()方法                                                                                                                            
                                                                                                                        --initServlet(instance);//如果没有初始化的话
                                                                                                                    -- ApplicationFilterChain filterChain =
                                                                                                                                      ApplicationFilterFactory.createFilterChain(request, wrapper, servlet);
                                                                                                                    --filterChain.doFilter(request.getRequest(), response.getResponse());
                                                                                                                        --internalDoFilter(request,response);
                                                                                                                            --filter.doFilter(request, response, this); //执行filter
                                                                                                                            --  lastServicedRequest.set(request); //ThreadLocal
                                                                                                                            --  lastServicedResponse.set(response); //ThreadLocal
                                                                                                                            --servlet.service(request, response);
                                                                                                                    --filterChain.release();
                                                                                                                    --wrapper.deallocate(servlet);  
                                                                                                                    --wrapper.unload(); //调用Servelt的destroy方法，把相关的类缓存清除 以及从JMX中unregister                                                                                                 
                                                                                                                //不是Async的话，直接finish                              
                                                                                                                -- request.finishRequest();
                                                                                                                --response.finishResponse();                                                                                                                                          
                                                                                                            --rp.setStage(org.apache.coyote.Constants.STAGE_ENDINPUT);
                                                                                                            --rp.setStage(org.apache.coyote.Constants.STAGE_ENDOUTPUT);                                                                                                  
                                                                                                            --rp.setStage(org.apache.coyote.Constants.STAGE_KEEPALIVE); ////如果没出错且KeepAlive
                                                                                                        --rp.setStage(org.apache.coyote.Constants.STAGE_ENDED);
                                                                                       
                                                 --startAcceptorThreads()
                                                    --acceptors[i] = createAcceptor(); //最原始的接收Socket请求的地方 默认是一个线程负责这个处理，接了之后转给Poller线程处理
                                                        --Acceptor.run()
                                                            --socket = serverSock.accept();
                                                            --setSocketOptions(socket)
                                                                --channel = new NioChannel(socket, bufhandler);
                                                                --getPoller0().register(channel); //注册PollEvent
                                                                    --socket.setPoller(this); // 这里的socket就是传进来的channel
                                                                    -- PollerEvent r = eventCache.pop();
                                                                    -- ka.interestOps(SelectionKey.OP_READ);//this is what OP_REGISTER turns into.
                                                                    -- if ( r==null) r = new PollerEvent(socket,ka,OP_REGISTER);
                                                                     
                                        --asyncTimeout = new AsyncTimeout(); // async timeout thread
        
    --StandardServer
      --StandardService implements Service [Engine(Contiainer), Connector, Executor]
        --  StandardEngine->StandardHost-> StandardContext(ApplicationContext, 包括Servlet, Listener, Filter)->StandardWrapper(Servlet) [都继承ContainerBase, 都看作是容器]
        
    --组件的创建和依赖注入统一由Digest.class进行处理，通过反射 
                     
        
    
    