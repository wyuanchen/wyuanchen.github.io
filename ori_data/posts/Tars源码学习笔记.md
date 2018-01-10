---
title: Tars源码学习笔记
date: 2017-06-21 22:36:06
categories: 后台
tags: C++
---


## `Application::main()`流程


1. 加载配置文件   
2. 初始化客户端
3. 初始化服务端
4. 绑定对象和端口    
5. 业务应用 的初始化
6. 设置`HandleGroup`分组，启动线程

<!-- more -->


### 1. 加载配置文件



```cpp
parseConfig(argc, argv);
```



### 2. 初始化 Proxy 部分



```cpp
initializeClient();
```



### 3. 初始化 Server 部分



```cpp
initializeServer();
```



### 4. 绑定对象和端口



```cpp
bindAdapter(adapters);
```



```cpp
void Application::bindAdapter(vector<TC_EpollServer::BindAdapterPtr>& adapters)
{
    size_t iBackPacketBuffLimit = TC_Common::strto<size_t>(toDefault(_conf.get("/tars/application/server<BackPacketBuffLimit>", "0"), "0"));

    string sPrefix = ServerConfig::Application + "." + ServerConfig::ServerName + ".";

    vector<string> adapterName;

    map<string, ServantHandle*> servantHandles;


    if (_conf.getDomainVector("/tars/application/server", adapterName))
    {
        for (size_t i = 0; i < adapterName.size(); i++)
        {
            string servant = _conf.get("/tars/application/server/" + adapterName[i] + "<servant>");

            checkServantNameValid(servant, sPrefix);

            ServantHelperManager::getInstance()->setAdapterServant(adapterName[i], servant);

            TC_EpollServer::BindAdapterPtr bindAdapter = new TC_EpollServer::BindAdapter(_epollServer.get());

            string sLastPath = "/tars/application/server/" + adapterName[i];

            bindAdapter->setName(adapterName[i]);

            bindAdapter->setEndpoint(_conf[sLastPath + "<endpoint>"]);

            bindAdapter->setMaxConns(TC_Common::strto<int>(_conf.get(sLastPath + "<maxconns>", "128")));

            bindAdapter->setOrder(parseOrder(_conf.get(sLastPath + "<order>","allow,deny")));

            bindAdapter->setAllow(TC_Common::sepstr<string>(_conf[sLastPath + "<allow>"], ";,", false));

            bindAdapter->setDeny(TC_Common::sepstr<string>(_conf.get(sLastPath + "<deny>",""), ";,", false));

            bindAdapter->setQueueCapacity(TC_Common::strto<int>(_conf.get(sLastPath + "<queuecap>", "1024")));

            bindAdapter->setQueueTimeout(TC_Common::strto<int>(_conf.get(sLastPath + "<queuetimeout>", "10000")));

            bindAdapter->setProtocolName(_conf.get(sLastPath + "<protocol>", "tars"));

            if (bindAdapter->isTarsProtocol())
            {
                bindAdapter->setProtocol(AppProtocol::parse);
            }

            bindAdapter->setHandleGroupName(_conf.get(sLastPath + "<handlegroup>", adapterName[i]));

            bindAdapter->setHandleNum(TC_Common::strto<int>(_conf.get(sLastPath + "<threads>", "0")));

            bindAdapter->setBackPacketBuffLimit(iBackPacketBuffLimit);

            _epollServer->bind(bindAdapter);

            adapters.push_back(bindAdapter);

            //队列取平均值
            if(!_communicator->getProperty("property").empty())
            {
                PropertyReportPtr p;
                p = _communicator->getStatReport()->createPropertyReport(bindAdapter->getName() + ".queue", PropertyReport::avg());
                bindAdapter->_pReportQueue = p.get();

                p = _communicator->getStatReport()->createPropertyReport(bindAdapter->getName() + ".connectRate", PropertyReport::avg());
                bindAdapter->_pReportConRate = p.get();

                p = _communicator->getStatReport()->createPropertyReport(bindAdapter->getName() + ".timeoutNum", PropertyReport::sum());
                bindAdapter->_pReportTimeoutNum = p.get();
            }
        }
    }
}
```



根据配置文件先创建所有的`Adapter`，然后把`Adapter`绑定到`Server`，   

其中`Server`的绑定是如下操作：   

```cpp
int  TC_EpollServer::bind(TC_EpollServer::BindAdapterPtr &lsPtr)
{
    int iRet = 0;

    for(size_t i = 0; i < _netThreads.size(); ++i)
    {
        if(i == 0)
        {
            iRet = _netThreads[i]->bind(lsPtr);
        }
        else
        {
            //当网络线程中listeners没有监听socket时，list使用adapter中设置的最大连接数作为初始化
            _netThreads[i]->setListSize(lsPtr->getMaxConns());
        }
    }

    return iRet;
}
```



把`Adapter`绑定到`Server`内部线程组的第一个线程，我猜测线程组的一个线程是作为`accept`的线程，然后线程组的其他线程则是增加每个线程的最大连接数





### 6. 设置 HandleGroup 分组，启动线程



```cpp
        for (size_t i = 0; i < adapters.size(); ++i)
        {
            string name = adapters[i]->getName();

            string groupName = adapters[i]->getHandleGroupName();

            if(name != groupName)
            {
                TC_EpollServer::BindAdapterPtr ptr = _epollServer->getBindAdapter(groupName);

                if (!ptr)
                {
                    throw runtime_error("[TARS][adater `" + name + "` setHandle to group `" + groupName + "` fail!");
                }

            }
            setHandle(adapters[i]);
        }
```



我看了实例的配置文件，发现往往情况下，`name`和`groupName`确实是一样的



然后在看`setHandle(adapter[i]);`

```cpp
void Application::setHandle(TC_EpollServer::BindAdapterPtr& adapter)
{
    adapter->setHandle<ServantHandle>();
}
```



再看`adapter->setHandle<ServantHandle>();`

```cpp
template<typename T> void setHandle()
{
  _pEpollServer->setHandleGroup<T>(_handleGroupName, _iHandleNum, this);
}
```



再看`_pEpollServer->setHandleGroup<T>(_handleGroupName, _iHandleNum, this);`

```cpp
    /**
     * 创建一个handle对象组，如果已经存在则直接返回
     * @param name
     * @return HandlePtr
     */
    template<class T> void setHandleGroup(const string& groupName, int32_t handleNum, BindAdapterPtr adapter)
    {
        map<string, HandleGroupPtr>::iterator it = _handleGroups.find(groupName);

        if (it == _handleGroups.end())
        {
            HandleGroupPtr hg = new HandleGroup();

            hg->name = groupName;

            adapter->_handleGroup = hg;

            for (int32_t i = 0; i < handleNum; ++i)
            {
                HandlePtr handle = new T();

                handle->setEpollServer(this);

                handle->setHandleGroup(hg);

                hg->handles.push_back(handle);
            }

            _handleGroups[groupName] = hg;

            it = _handleGroups.find(groupName);
        }
        it->second->adapters[adapter->getName()] = adapter;

        adapter->_handleGroup = it->second;
    }
```



上面的调用的顺序有点像是委托模式是：   

1. `Application`对象调用成员函数`setHandle(adapters[i])`把一个`Adapter`传进去
2. `adapter[i]`对象调用自己成员函数`setHandle<ServantHandle>()`
3. 最后由`_pEpollServer`对象来调用自己成员函数`setHandleGroup<T>(_handleGroupName, _iHandleNum, this);`来完成实际的操作。

## Handle 处理完逻辑后返回响应数据的流程



 `BIndAdapter`只是负责把接收到的字节流进行协议解码后传递给`Handle`，所以`Handle`在`run()`中会遍历`BindAdpater`是否有数据进来（注意这些数据都是已经经过解码的）

```cpp
tagRecvData* recv = NULL;

        map<string, BindAdapterPtr>& adapters = _handleGroup->adapters;

        for (map<string, BindAdapterPtr>::iterator it = adapters.begin(); it != adapters.end(); ++it)
        {
            BindAdapterPtr& adapter = it->second;

            try
            {

                while (adapter->waitForRecvQueue(recv, 0))
                {
                    //上报心跳
                    heartbeat();

                    //为了实现所有主逻辑的单线程化,在每次循环中给业务处理自有消息的机会
                    handleAsyncResponse();

                    tagRecvData& stRecvData = *recv;

                    int64_t now = TNOWMS;


                    stRecvData.adapter = adapter;

                    //数据已超载 overload
                    if (stRecvData.isOverload)
                    {
                        handleOverload(stRecvData);
                    }
                    //关闭连接的通知消息
                    else if (stRecvData.isClosed)
                    {
                        handleClose(stRecvData);
                    }
                    //数据在队列中已经超时了
                    else if ( (now - stRecvData.recvTimeStamp) > (int64_t)adapter->getQueueTimeout())
                    {
                        handleTimeout(stRecvData);
                    }
                    else
                    {
                        handle(stRecvData);
                    }
                     handleCustomMessage(false);

                    delete recv;
                    recv = NULL;
                }
            }
            catch (exception &ex)
            {
                if(recv)
                {
                    close(recv->uid, recv->fd);
                    delete recv;
                    recv = NULL;
                }

                getEpollServer()->error("[Handle::handleImp] error:" + string(ex.what()));
            }
            catch (...)
            {
                if(recv)
                {
                    close(recv->uid, recv->fd);
                    delete recv;
                    recv = NULL;
                }

                getEpollServer()->error("[Handle::handleImp] unknown error");
            }
        }
```





而 `Handle`的响应数据则通过 `TarsCurrentPtr`来返回，并没有用到`BindAdpater`：

```cpp
current->sendResponse(ret, buffer, TarsCurrent::TARS_STATUS(), sResultDesc);
```



下面是`current->sendResponse(...)`的实现

```cpp
void TarsCurrent::sendResponse(int iRet, const vector<char>& buffer, const map<string, string>& status, const string & sResultDesc)
{
    _ret = iRet;

    //单向调用不需要返回
    if (_request.cPacketType == TARSONEWAY)
    {
        return;
    }

    TarsOutputStream<BufferWriter> os;

    if (_request.iVersion != TUPVERSION)
    {
        ResponsePacket response;

        response.iRequestId     = _request.iRequestId;
        response.iMessageType   = _request.iMessageType;
        response.cPacketType    = TARSNORMAL;
        response.iVersion       = TARSVERSION;
        response.status         = status;
        response.sBuffer        = buffer;
        response.sResultDesc    = sResultDesc;
        response.context        = _responseContext;
        response.iRet           = iRet;

        TLOGINFO("[TARS]TarsCurrent::sendResponse :"
                   << response.iMessageType << "|"
                   << _request.sServantName << "|"
                   << _request.sFuncName << "|"
                   << response.iRequestId << endl);

        response.writeTo(os);
    }
    else
    {
        //tup回应包用请求包的结构
        RequestPacket response;

        response.iRequestId     = _request.iRequestId;
        response.iMessageType   = _request.iMessageType;
        response.cPacketType    = TARSNORMAL;
        response.iVersion       = _request.iVersion;
        response.status         = status;
        response.context        = _responseContext;
        response.sBuffer        = buffer;
        response.sServantName   = _request.sServantName;
        response.sFuncName      = _request.sFuncName;

        //异常的情况下buffer可能为空，要保证有一个空UniAttribute的编码内容
        if(response.sBuffer.size() == 0)
        {
            tup::UniAttribute<> tarsAttr;
            tarsAttr.setVersion(_request.iVersion);
            tarsAttr.encode(response.sBuffer);
        }
        //iRet为0时,不记录在status里面,节省空间
        if(iRet != 0)
        {
            response.status[ServantProxy::STATUS_RESULT_CODE] = TC_Common::tostr(iRet);
        }
        //sResultDesc为空时,不记录在status里面,节省空间
        if(!sResultDesc.empty())
        {
            response.status[ServantProxy::STATUS_RESULT_DESC] = sResultDesc;
        }

        TLOGINFO("[TARS]TarsCurrent::sendResponse :"
                   << response.iMessageType << "|"
                   << response.sServantName << "|"
                   << response.sFuncName << "|"
                   << response.iRequestId << endl);

        response.writeTo(os);
    }

    tars::Int32 iHeaderLen = htonl(sizeof(tars::Int32) + os.getLength());

    string s = "";

    s.append((const char*)&iHeaderLen, sizeof(tars::Int32));

    s.append(os.getBuffer(), os.getLength());

    _servantHandle->sendResponse(_uid, s, _ip, _port, _fd);
}
```



可以看到最后又调用` _servantHandle->sendResponse(_uid, s, _ip, _port, _fd);`，并且此时发送的数据应该是已经经过编码的。



下面是` _servantHandle->sendResponse(_uid, s, _ip, _port, _fd);`的实现

```cpp
void TC_EpollServer::Handle::sendResponse(uint32_t uid, const string &sSendBuffer, const string &ip, int port, int fd)
{
    _pEpollServer->send(uid, sSendBuffer, ip, port, fd);
}
```



再看`_pEpollServer->send(uid, sSendBuffer, ip, port, fd);`

```cpp
void TC_EpollServer::send(unsigned int uid, const string &s, const string &ip, uint16_t port, int fd)
{
    TC_EpollServer::NetThread* netThread = getNetThreadOfFd(fd);

    netThread->send(uid, s, ip, port);
}
```

在这里先去获取`fd`绑定的`IO线程`，然后调用该线程的`send()`



下面是`netThread->send(uid, s, ip, port);`的实现

```cpp
void TC_EpollServer::NetThread::send(uint32_t uid, const string &s, const string &ip, uint16_t port)
{
    if(_bTerminate)
    {
        return;
    }

    tagSendData* send = new tagSendData();

    send->uid = uid;

    send->cmd = 's';

    send->buffer = s;

    send->ip = ip;

    send->port = port;

    _sbuffer.push_back(send);

    //通知epoll响应, 有数据要发送
    _epoller.mod(_notify.getfd(), H64(ET_NOTIFY), EPOLLOUT);
}
```



这里可以看到`IO线程`会先通过提前创建的`notify`文件来完成通知数据发送的功能，这样就可以在后续的`processPipe()`来响应该操作

```cpp
void TC_EpollServer::NetThread::processPipe()
{
    send_queue::queue_type deSendData;

    _sbuffer.swap(deSendData);

    send_queue::queue_type::iterator it = deSendData.begin();

    send_queue::queue_type::iterator itEnd = deSendData.end();

    while(it != itEnd)
    {
        switch((*it)->cmd)
        {
        case 'c':
            {
                Connection *cPtr = getConnectionPtr((*it)->uid);

                if(cPtr)
                {
                    if(cPtr->setClose())
                    {
                        delConnection(cPtr,true,EM_SERVER_CLOSE);
                    }
                }
                break;
            }
        case 's':
            {
                Connection *cPtr = getConnectionPtr((*it)->uid);

                if(cPtr)
                {
                    int ret = sendBuffer(cPtr, (*it)->buffer, (*it)->ip, (*it)->port);

                    if(ret < 0)
                    {
                        delConnection(cPtr,true,(ret==-1)?EM_CLIENT_CLOSE:EM_SERVER_CLOSE);
                    }
                }
                break;
            }
        default:
            assert(false);
        }
        delete (*it);
        ++it;
    }
}
```

`int ret = sendBuffer(cPtr, (*it)->buffer, (*it)->ip, (*it)->port);`会调用系统`IO`操作来执行真正的写入操作。



下面是`NetThread`的`run()`实现

```cpp
void TC_EpollServer::NetThread::run()
{
    //循环监听网路连接请求
    while(!_bTerminate)
    {
        _list.checkTimeout(TNOW);

        int iEvNum = _epoller.wait(2000);

        for(int i = 0; i < iEvNum; ++i)
        {
            try
            {
                const epoll_event &ev = _epoller.get(i);

                uint32_t h = ev.data.u64 >> 32;

                switch(h)
                {
                case ET_LISTEN:
                    {
                        //监听端口有请求
                        map<int, BindAdapterPtr>::const_iterator it = _listeners.find(ev.data.u32);
                        if( it != _listeners.end())
                        {
                            if(ev.events & EPOLLIN)
                            {
                                bool ret;
                                do
                                {
                                    ret = accept(ev.data.u32);
                                }while(ret);
                            }
                        }
                    }
                    break;
                case ET_CLOSE:
                    //关闭请求
                    break;
                case ET_NOTIFY:
                    //发送通知
                    processPipe();
                    break;
                case ET_NET:
                    //网络请求
                    processNet(ev);
                    break;
                default:
                    assert(true);
                }
            }
            catch(exception &ex)
            {
                error("run exception:" + string(ex.what()));
            }
        }
    }
}
```







## AdapterProxy

主要方法为：

- `invoke`
- `finishInvoke`


并没有负责真正IO的读写，只是调用`Transver`来执行真正`IO`

也没有负责真正的数据编解码，而是调用`ObjectProxy`来实现协议编解码



`AdapterProxy`的主要作用应该是调用`ObjectProxy`进行对数据进行协议的编码，然后调用`Transver`来把编码后的数据输出。   

然后负责调用响应数据（是已经经过解码的）自身的回调函数进行i响应，至于`AdapterProxy`有没有负责响应数据流的解码我还不清楚，不过估计应该是没有的。



###  invoke(ReqMessage * msg)

```cpp
int AdapterProxy::invoke(ReqMessage * msg)
{
    assert(_trans != NULL);

    TLOGINFO("[TARS][AdapterProxy::invoke objname:" << _objectProxy->name() << ",desc:" << _endpoint.desc() << endl);

    //未发链表有长度限制
    if(_timeoutQueue->getSendListSize() >= _noSendQueueLimit)
    {
        TLOGERROR("[TARS][AdapterProxy::invoke fail,ReqInfoQueue.size > " << _noSendQueueLimit << ",objname:" << _objectProxy->name() <<",desc:"<< _endpoint.desc() << endl);
        msg->eStatus = ReqMessage::REQ_EXC;

        finishInvoke(msg);

        return 0;
    }

    //生成requestid
    //tars调用 而且 不是单向调用
    if(!msg->bFromRpc)
    {
        msg->request.iRequestId = _objectProxy->generateId();
    }

    _objectProxy->getProxyProtocol().requestFunc(msg->request,msg->sReqData);

    //交给连接发送数据,连接连上,buffer不为空,直接发送数据成功
    if(_timeoutQueue->sendListEmpty() && _trans->sendRequest(msg->sReqData.c_str(),msg->sReqData.size()) != Transceiver::eRetError)
    {
        TLOGINFO("[TARS][AdapterProxy::invoke push (send) objname:" << _objectProxy->name() << ",desc:" << _endpoint.desc() << ",id:" << msg->request.iRequestId << endl);

        //请求发送成功了，单向调用直接返回
        if(msg->eType == ReqMessage::ONE_WAY)
        {
            delete msg;
            msg = NULL;

            return 0;
        }

        bool bFlag = _timeoutQueue->push(msg, msg->request.iRequestId, msg->request.iTimeout + msg->iBeginTime);
        if(!bFlag)
        {
            TLOGERROR("[TARS][AdapterProxy::invoke fail1 : insert timeout queue fail,queue size:" << _timeoutQueue->size() << ",objname" <<_objectProxy->name() << ",desc" << _endpoint.desc() <<endl);
            msg->eStatus = ReqMessage::REQ_EXC;

            finishInvoke(msg);
        }
    }
    else
    {
        TLOGINFO("[TARS][AdapterProxy::invoke push (no send) " << _objectProxy->name() << ", " << _endpoint.desc() << ",id " << msg->request.iRequestId <<endl);

        //请求发送失败了
        bool bFlag = _timeoutQueue->push(msg,msg->request.iRequestId, msg->request.iTimeout+msg->iBeginTime, false);
        if(!bFlag)
        {
            TLOGERROR("[TARS][AdapterProxy::invoke fail2 : insert timeout queue fail,queue size:" << _timeoutQueue->size() << "," <<_objectProxy->name() << ", " << _endpoint.desc() <<endl);
            
            msg->eStatus = ReqMessage::REQ_EXC;

            finishInvoke(msg);
        }
    }

    return 0;
}
```



### finishInvoke(ReqMessage * msg)

`finishInvoke`有3个版本：

- `void finishInvoke(ResponsePacket &rsp)`
- `void finishInvoke(ReqMessage * msg)`
- `void finishInvoke(bool bTimeout)`  

不过主要操作还是放在`void finishInvoke(ReqMessage * msg)`



```cpp
void AdapterProxy::finishInvoke(ReqMessage * msg)
{
    assert(msg->eStatus != ReqMessage::REQ_REQ);

    TLOGINFO("[TARS][AdapterProxy::finishInvoke(ReqMessage) objname:" << _objectProxy->name() << ",desc:" << _endpoint.desc() << " ,id:" << msg->response.iRequestId << endl);

    //单向调用
    if(msg->eType == ReqMessage::ONE_WAY)
    {
        delete msg;
        msg = NULL;
        return ;
    }

    //stat 上报调用统计
    stat(msg);

    //超时屏蔽统计,异常不算超时统计
    if(msg->eStatus != ReqMessage::REQ_EXC && !msg->bPush)
    {
        finishInvoke(msg->response.iRet != TARSSERVERSUCCESS);
    }

    //同步调用，唤醒ServantProxy线程
    if(msg->eType == ReqMessage::SYNC_CALL)
    {
        if(!msg->bCoroFlag)
        {
            assert(msg->pMonitor);

            TC_ThreadLock::Lock sync(*(msg->pMonitor));
            msg->pMonitor->notify();
            msg->bMonitorFin = true;
        }
        else
        {
            msg->sched->put(msg->iCoroId);
        }

        return ;
    }

    //异步调用
    if(msg->eType == ReqMessage::ASYNC_CALL)
    {
        if(!msg->bCoroFlag)
        {
            if(msg->callback->getNetThreadProcess())
            {
                //如果是本线程的回调，直接本线程处理
                //比如获取endpoint
                ReqMessagePtr msgPtr = msg;
                try
                {
                    msg->callback->onDispatch(msgPtr);
                }
                catch(exception & e)
                {
                    TLOGERROR("[TARS][AdapterProxy::finishInvoke(ReqMessage) ex:" << e.what() << ",line:" << __LINE__ << endl);
                }
                catch(...)
                {
                    TLOGERROR("[TARS]AdapterProxy::finishInvoke(ReqMessage) ex:unknown,line:" << __LINE__ << endl);
                }
            }
            else
            {
                //异步回调，放入回调处理线程中
                _objectProxy->getCommunicatorEpoll()->pushAsyncThreadQueue(msg);
            }
        }
        else
        {
            CoroParallelBasePtr ptr = msg->callback->getCoroParallelBasePtr();
            if(ptr)
            {
                ptr->insert(msg);
                if(ptr->checkAllReqReturn())
                {
                    msg->sched->put(msg->iCoroId);
                }
            }
            else
            {
                TLOGERROR("[TARS][AdapterProxy::finishInvoke(ReqMessage) coro parallel callback error,obj:" << _objectProxy->name() << ",desc:" << _endpoint.desc() 
                    << ",id:" << msg->response.iRequestId << endl);
                delete msg;
                msg = NULL;
            }
        }
        return;
    }

    assert(false);

    return;
}
```

