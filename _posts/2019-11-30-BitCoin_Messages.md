---
layout:     post                    # 使用的布局（不需要改）
title:      Bitcoin Messages Protocol              # 标题 
subtitle:   analyse messages protocol of the Bitcoin  #副标题
date:       2019-11-30              # 时间
author:     BY Logici                      # 作者
header-img: img/post-bg-2015.jpg    #这篇文章标题背景图片
catalog: true                       # 是否归档
tags:                               #标签
    - Blockchain
---


# BitCoin 报文格式

**分析比特币源码：**

Main.cpp

ProcessMessage 处理不同类型的协议信息



chainParam.h	protocol.h 一些报文的参数

COMMAND_SIZE=12,				报文头，数据包类型的ASCII表示形式，如果数据包内容是空的，用NULL填充

MESSAGE_SIZE_SIZE=sizeof(int),	数据内容的长度

CHECKSUM_SIZE=sizeof(int),		校验项目

\#define MESSAGE_START_SIZE 4		报文头部长度



net.h net.cpp

PushMessage	封装数据包。报文头+数据内容+偏移长度+检验码





比特币网络通信中使用基于**Tcp的通信协议**。

**协议报文解析**

1.协议头

通信发送的数据包中有一个报文的头部，格式是固定的，头部中包含一些内容的简介，比如数据包的长度、校验等。

在protocol.h中，定义了class CMessageHeader报文头类，主要信息有：单位字节（Byte）

/** Message header.

 \* (4) message start.			报文头存放的字节长度

 \* (12) command.				 存放报文数据类别，根据command字段

 \* (4) size.								数据内容的长度

 \* (4) checksum.					校验值

 */

 COMMAND_SIZE=12,						//command字段的长度

MESSAGE_SIZE_SIZE=sizeof(int),	//报文长度的长度

CHECKSUM_SIZE=sizeof(int),			//校验值的长度



MESSAGE_SIZE_OFFSET=MESSAGE_START_SIZE+COMMAND_SIZE,	//偏移量，size字段的索引值

CHECKSUM_OFFSET=MESSAGE_SIZE_OFFSET+MESSAGE_SIZE_SIZE,//偏移量，checksum字段的索引值

HEADER_SIZE=MESSAGE_START_SIZE+COMMAND_SIZE+MESSAGE_SIZE_SIZE+CHECKSUM_SIZE//整个报文头的总长度

​        char pchMessageStart[MESSAGE_START_SIZE];			//

​        char pchCommand[COMMAND_SIZE];			//存放command的字符串数组



在chainparama.h中定义了\#define MESSAGE_START_SIZE 4

**版本交互**

以版本为例进行分析。首次与比特币网络节点（无论是和哪个节点通信，相互的交换版本信息是首先要进行的）通信发送的消息是版本信息。因为比特币网络中各种版本不兼容（去中心化的一个网络拓扑，版本不一致很正常）的话是不方便通信的。我们先来看看确认版本信息的几次握手示意过程：

- 本地节点（即将加入到比特币网络的节点）发送VERSION报文至某一个远程节点（网络节点）；
- 远程节点发送VERSION报文；
- 远程节点回复VERACK报文；
- 远程节点设置协议版本为两者的最小值；
- 本地节点回复VERACK报文；
- 本地节点设置协议版本为两者的最小值；

在net.cpp中，

```cpp
void CNode::PushVersion()

{

​    int nBestHeight = g_signals.GetHeight().get_value_or(0);


​    /// when NTP implemented, change to just nTime = GetAdjustedTime()

​    int64_t nTime = (fInbound ? GetAdjustedTime() : GetTime());

​    CAddress addrYou = (addr.IsRoutable() && !IsProxy(addr) ? addr : CAddress(CService("0.0.0.0",0)));

​    CAddress addrMe = GetLocalAddress(&addr);

​    RAND_bytes((unsigned char*)&nLocalHostNonce, sizeof(nLocalHostNonce));

​    LogPrint("net", "send version message: version %d, blocks=%d, us=%s, them=%s, peer=%s\n", PROTOCOL_VERSION, nBestHeight, addrMe.ToString(), addrYou.ToString(), addr.ToString());

​    PushMessage("version", PROTOCOL_VERSION, nLocalServices, nTime, addrYou, addrMe,

​                nLocalHostNonce, FormatSubVersion(CLIENT_NAME, CLIENT_VERSION, std::vector<string>()), nBestHeight, true);

}
```



上述代码中，关键的一个函数是，PushMessage.（net.h中）

PushMessage("version", PROTOCOL_VERSION, nLocalServices, nTime, addrYou, addrMe,nLocalHostNonce, FormatSubVersion(CLIENT_NAME, CLIENT_VERSION, std::vector<string>()), nBestHeight, true);

"version"作为报文command，传参到PushMessage中的BeginMessage。（net.h中）

```cpp
template<typename T1, typename T2, typename T3, typename T4, typename T5, typename T6, typename T7, typename T8, typename T9>
    void PushMessage(const char* pszCommand, const T1& a1, const T2& a2, const T3& a3, const T4& a4, const T5& a5, const T6& a6, const T7& a7, const T8& a8, const T9& a9)
    {
        try
        {
            BeginMessage(pszCommand);
            ssSend << a1 << a2 << a3 << a4 << a5 << a6 << a7 << a8 << a9;
            EndMessage();
        }
        catch (...)
        {
            AbortMessage();
            throw;
        }
    }
```

PushMessage是一个多态重载函数，最多有9个模版变量作为参数变量。

先关心pszCommand,在pushversion函数中，传递的是“version”这个变量，进入了BeginMessage函数中。

```cpp
 void BeginMessage(const char* pszCommand) EXCLUSIVE_LOCK_FUNCTION(cs_vSend)
    {
        ENTER_CRITICAL_SECTION(cs_vSend);
        assert(ssSend.size() == 0);
        ssSend << CMessageHeader(pszCommand, 0);
        LogPrint("net", "sending: %s ", SanitizeString(pszCommand));
    }
```

ssSend是CNode类（/** Information about a peer 这个类中存放节点的一些信息*/）中，CDataStream ssSend;定义的一个数据流对象，这个类是用来管理底层数据格式编码流入流出的。具体先不用管。

ssSend << CMessageHeader(pszCommand, 0);表明要对报文头进行初始化。

```cpp
CMessageHeader::CMessageHeader(const char* pszCommand, unsigned int nMessageSizeIn)
{
    memcpy(pchMessageStart, Params().MessageStart(), MESSAGE_START_SIZE);
    strncpy(pchCommand, pszCommand, COMMAND_SIZE);
    nMessageSize = nMessageSizeIn;
    nChecksum = 0;
}
```

strncpy(pchCommand, pszCommand, COMMAND_SIZE);把参数pszCommand拷贝到了类CMessageHeader内变量pchCommand当中进行存储。把这个CMessageHeader对象 << 输入流 进入到ssSend当中。

我怀疑使用了以下的宏，把这些数据给封装起来了，但是这个宏我还暂时研究不透。

```cpp
IMPLEMENT_SERIALIZE
  (
  READWRITE(FLATDATA(pchMessageStart));
  READWRITE(FLATDATA(pchCommand));
  READWRITE(nMessageSize);
  READWRITE(nChecksum);
)
```



追溯到这里，完成了BeginMessage(pszCommand);这个函数的操作，即把pszCommand传进了报文头部，并进行了初始化在ssSend当中。

如果PushMessage没有其他参数，则表明只进行传递报文头，如果还有其他参数，则可以传递最多9个参数，现以version的9个参数为例。

ssSend << a1 << a2 << a3 << a4 << a5 << a6 << a7 << a8 << a9;

//9个不同的模版变量，依次传进ssSend对象变量当中。到时候也是依次对应解码。这里，version传递的是下面的9个字段变量：

| 字段                                                         | 字节 |                             描述                             |
| ------------------------------------------------------------ | ---- | :----------------------------------------------------------: |
| version——PROTOCOL_VERSION                                    | 4    |                    节点使用的协议版本标识                    |
| services——nLocalServices                                     | 8    | 该连接拥有的特点（比如可以接受header信息，还可以接受blcok信息） |
| timestamp——nTime                                             | 8    |                      时间戳（秒为单位）                      |
| addr_you——addrYou                                            | 26   |                       接收者的网络地址                       |
| addr_me—— addrMe                                             | 26   |                       发送者的网络地址                       |
| nonce——nLocalHostNonce                                       | 8    |                 节点的随机ID，用于侦测此连接                 |
| sub_version_num——FormatSubVersion(CLIENT_NAME, CLIENT_VERSION, std::vector<string>()) | 可变 |                         辅助版本信息                         |
| start_height——nBestHeight                                    | 4    |                       发送者的区块高度                       |
| 最后一个true，不知道是啥                                     |      |                                                              |

EndMessage函数：

```cpp
   void EndMessage() UNLOCK_FUNCTION(cs_vSend)
    {
        // The -*messagestest options are intentionally not documented in the help message,
        // since they are only used during development to debug the networking code and are
        // not intended for end-users.
        if (mapArgs.count("-dropmessagestest") && GetRand(GetArg("-dropmessagestest", 2)) == 0)
        {
            LogPrint("net", "dropmessages DROPPING SEND MESSAGE\n");
            AbortMessage();
            return;
        }
        if (mapArgs.count("-fuzzmessagestest"))
            Fuzz(GetArg("-fuzzmessagestest", 10));

        if (ssSend.size() == 0)
            return;

        // Set the size
        unsigned int nSize = ssSend.size() - CMessageHeader::HEADER_SIZE;
        memcpy((char*)&ssSend[CMessageHeader::MESSAGE_SIZE_OFFSET], &nSize, sizeof(nSize));

        // Set the checksum
        uint256 hash = Hash(ssSend.begin() + CMessageHeader::HEADER_SIZE, ssSend.end());
        unsigned int nChecksum = 0;
        memcpy(&nChecksum, &hash, sizeof(nChecksum));
        assert(ssSend.size () >= CMessageHeader::CHECKSUM_OFFSET + sizeof(nChecksum));
        memcpy((char*)&ssSend[CMessageHeader::CHECKSUM_OFFSET], &nChecksum, sizeof(nChecksum));

        LogPrint("net", "(%d bytes)\n", nSize);

        std::deque<CSerializeData>::iterator it = vSendMsg.insert(vSendMsg.end(), CSerializeData());
        ssSend.GetAndClear(*it);
        nSendSize += (*it).size();

        // If write queue empty, attempt "optimistic write"
        if (it == vSendMsg.begin())
            SocketSendData(this);

        LEAVE_CRITICAL_SECTION(cs_vSend);
    }
```

```cpp
        // Set the size
        unsigned int nSize = ssSend.size() - CMessageHeader::HEADER_SIZE;
        memcpy((char*)&ssSend[CMessageHeader::MESSAGE_SIZE_OFFSET], &nSize, sizeof(nSize));
```

此时，ssSend：报文头部+a1+a2+a3+a4+a5+a6+a7+a8+a9，则ssSend.size()-CMessageHeader::HEADER_SIZE=(a1+a2+a3+a4+a5+a6+a7+a8+a9)的长度，这个长度需要存放在偏移地址为CMessageHeader::MESSAGE_SIZE_OFFSET的地方，这个地方长sizeof(nSize)。

```cpp
 // Set the checksum
        uint256 hash = Hash(ssSend.begin() + CMessageHeader::HEADER_SIZE, ssSend.end());
        unsigned int nChecksum = 0;
        memcpy(&nChecksum, &hash, sizeof(nChecksum));
        assert(ssSend.size () >= CMessageHeader::CHECKSUM_OFFSET + sizeof(nChecksum));
        memcpy((char*)&ssSend[CMessageHeader::CHECKSUM_OFFSET], &nChecksum, sizeof(nChecksum));
```

要存放校验值checksum，这个值是通过hash计算得到的。把这个hash值编码到nchecksum当中，再存进CMessageHeader::CHECKSUM_OFFSET偏移地址上，这个存放的长度为sizeof(nChecksum)。

至此，PushMessage函数的主要功能就捋清楚了。在PushVersion当中，封装了带有“version”的报文头，还有节点使用的协议版本标识、该连接拥有的特点（比如可以接受header信息，还可以接受blcok信息）、时间戳（秒为单位）、接收者的网络地址、发送者的网络地址、节点的随机ID、辅助版本信息、发送者的区块高度、还有一个true不知道是干啥的。

接下来，看处理报文的函数。

在net.cpp当中，有一个处理messages的线程。

```cpp
  // Process messages
    threadGroup.create_thread(boost::bind(&TraceThread<void (*)()>, "msghand", &ThreadMessageHandler));

```

线程中循环处理  ProcessMessages(pnode) 这个函数。

```cpp
void ThreadMessageHandler()
{
    SetThreadPriority(THREAD_PRIORITY_BELOW_NORMAL);
    while (true)
    {
        bool fHaveSyncNode = false;

        vector<CNode*> vNodesCopy;
        {
            LOCK(cs_vNodes);
            vNodesCopy = vNodes;
            BOOST_FOREACH(CNode* pnode, vNodesCopy) {
                pnode->AddRef();
                if (pnode == pnodeSync)
                    fHaveSyncNode = true;
            }
        }

        if (!fHaveSyncNode)
            StartSync(vNodesCopy);

        // Poll the connected nodes for messages
        CNode* pnodeTrickle = NULL;
        if (!vNodesCopy.empty())
            pnodeTrickle = vNodesCopy[GetRand(vNodesCopy.size())];

        bool fSleep = true;

        BOOST_FOREACH(CNode* pnode, vNodesCopy)
        {
            if (pnode->fDisconnect)
                continue;

            // Receive messages
            {
                TRY_LOCK(pnode->cs_vRecvMsg, lockRecv);
                if (lockRecv)
                {
                    if (!g_signals.ProcessMessages(pnode))
                        pnode->CloseSocketDisconnect();

                    if (pnode->nSendSize < SendBufferSize())
                    {
                        if (!pnode->vRecvGetData.empty() || (!pnode->vRecvMsg.empty() && pnode->vRecvMsg[0].complete()))
                        {
                            fSleep = false;
                        }
                    }
                }
            }
            boost::this_thread::interruption_point();

            // Send messages
            {
                TRY_LOCK(pnode->cs_vSend, lockSend);
                if (lockSend)
                    g_signals.SendMessages(pnode, pnode == pnodeTrickle);
            }
            boost::this_thread::interruption_point();
        }

        {
            LOCK(cs_vNodes);
            BOOST_FOREACH(CNode* pnode, vNodesCopy)
                pnode->Release();
        }

        if (fSleep)
            MilliSleep(100);
    }
}

```

```cpp
// requires LOCK(cs_vRecvMsg)
bool ProcessMessages(CNode* pfrom)
{
    //if (fDebug)
    //    LogPrintf("ProcessMessages(%u messages)\n", pfrom->vRecvMsg.size());

    //
    // Message format
    //  (4) message start
    //  (12) command
    //  (4) size
    //  (4) checksum
    //  (x) data
    //
    bool fOk = true;

    if (!pfrom->vRecvGetData.empty())
        ProcessGetData(pfrom);

    // this maintains the order of responses
    if (!pfrom->vRecvGetData.empty()) return fOk;

    std::deque<CNetMessage>::iterator it = pfrom->vRecvMsg.begin();
    while (!pfrom->fDisconnect && it != pfrom->vRecvMsg.end()) {
        // Don't bother if send buffer is too full to respond anyway
        if (pfrom->nSendSize >= SendBufferSize())
            break;

        // get next message
        CNetMessage& msg = *it;

        //if (fDebug)
        //    LogPrintf("ProcessMessages(message %u msgsz, %u bytes, complete:%s)\n",
        //            msg.hdr.nMessageSize, msg.vRecv.size(),
        //            msg.complete() ? "Y" : "N");

        // end, if an incomplete message is found
        if (!msg.complete())
            break;

        // at this point, any failure means we can delete the current message
        it++;

        // Scan for message start
        if (memcmp(msg.hdr.pchMessageStart, Params().MessageStart(), MESSAGE_START_SIZE) != 0) {
            LogPrintf("PROCESSMESSAGE: INVALID MESSAGESTART %s\n", SanitizeString(msg.hdr.GetCommand()));
            fOk = false;
            break;
        }

        // Read header
        CMessageHeader& hdr = msg.hdr;
        if (!hdr.IsValid())
        {
            LogPrintf("PROCESSMESSAGE: ERRORS IN HEADER %s\n", SanitizeString(hdr.GetCommand()));
            continue;
        }
        string strCommand = hdr.GetCommand();

        // Message size
        unsigned int nMessageSize = hdr.nMessageSize;

        // Checksum
        CDataStream& vRecv = msg.vRecv;
        uint256 hash = Hash(vRecv.begin(), vRecv.begin() + nMessageSize);
        unsigned int nChecksum = 0;
        memcpy(&nChecksum, &hash, sizeof(nChecksum));
        if (nChecksum != hdr.nChecksum)
        {
            LogPrintf("ProcessMessages(%s, %u bytes): CHECKSUM ERROR nChecksum=%08x hdr.nChecksum=%08x\n",
               SanitizeString(strCommand), nMessageSize, nChecksum, hdr.nChecksum);
            continue;
        }

        // Process message
        bool fRet = false;
        try
        {
            fRet = ProcessMessage(pfrom, strCommand, vRecv);
            boost::this_thread::interruption_point();
        }
        catch (std::ios_base::failure& e)
        {
            pfrom->PushMessage("reject", strCommand, REJECT_MALFORMED, string("error parsing message"));
            if (strstr(e.what(), "end of data"))
            {
                // Allow exceptions from under-length message on vRecv
                LogPrintf("ProcessMessages(%s, %u bytes): Exception '%s' caught, normally caused by a message being shorter than its stated length\n", SanitizeString(strCommand), nMessageSize, e.what());
            }
            else if (strstr(e.what(), "size too large"))
            {
                // Allow exceptions from over-long size
                LogPrintf("ProcessMessages(%s, %u bytes): Exception '%s' caught\n", SanitizeString(strCommand), nMessageSize, e.what());
            }
            else
            {
                PrintExceptionContinue(&e, "ProcessMessages()");
            }
        }
        catch (boost::thread_interrupted) {
            throw;
        }
        catch (std::exception& e) {
            PrintExceptionContinue(&e, "ProcessMessages()");
        } catch (...) {
            PrintExceptionContinue(NULL, "ProcessMessages()");
        }

        if (!fRet)
            LogPrintf("ProcessMessage(%s, %u bytes) FAILED\n", SanitizeString(strCommand), nMessageSize);

        break;
    }

    // In case the connection got shut down, its receive buffer was wiped
    if (!pfrom->fDisconnect)
        pfrom->vRecvMsg.erase(pfrom->vRecvMsg.begin(), it);

    return fOk;
}
```

这里就把之前pushMessage拼接的包给拆解了。

获取了strCommand,包中command字段已经被解析。

```cpp
// Read header
        CMessageHeader& hdr = msg.hdr;
        if (!hdr.IsValid())
        {
            LogPrintf("PROCESSMESSAGE: ERRORS IN HEADER %s\n", SanitizeString(hdr.GetCommand()));
            continue;
        }
        string strCommand = hdr.GetCommand();


```

报文长度

```cpp
 // Message size
        unsigned int nMessageSize = hdr.nMessageSize;

```

检查校验值是否改变

```cpp
 // Checksum
        CDataStream& vRecv = msg.vRecv;
        uint256 hash = Hash(vRecv.begin(), vRecv.begin() + nMessageSize);
        unsigned int nChecksum = 0;
        memcpy(&nChecksum, &hash, sizeof(nChecksum));
        if (nChecksum != hdr.nChecksum)
        {
            LogPrintf("ProcessMessages(%s, %u bytes): CHECKSUM ERROR nChecksum=%08x hdr.nChecksum=%08x\n",
               SanitizeString(strCommand), nMessageSize, nChecksum, hdr.nChecksum);
            continue;
        }
```

ProcessMessage针对不同的command字段，做不同的处理

```cpp
// Process message
        bool fRet = false;
        try
        {
            fRet = ProcessMessage(pfrom, strCommand, vRecv);
            boost::this_thread::interruption_point();
        }
```

在ProcessMessage中，举version的例子：

```cpp
if (strCommand == "version")
    {
        // Each connection can only send one version message
  			// 一个连接只能发送一个版本信息，不能发送多个，不等于0说明之前已经处理过一次了，已经接收过一次了
        if (pfrom->nVersion != 0)
        {
            pfrom->PushMessage("reject", strCommand, REJECT_DUPLICATE, string("Duplicate version message"));
            Misbehaving(pfrom->GetId(), 1);
            return false;
        }

        int64_t nTime;
        CAddress addrMe;
        CAddress addrFrom;
        uint64_t nNonce = 1;
  		//解析发过来的包，和封包时的顺序一致
        vRecv >> pfrom->nVersion >> pfrom->nServices >> nTime >> addrMe;
  		// 如果发过来的版本比约定的最低版本还小，则发送reject，并断开连接
        if (pfrom->nVersion < MIN_PEER_PROTO_VERSION)
        {
            // disconnect from peers older than this proto version
            LogPrintf("partner %s using obsolete version %i; disconnecting\n", pfrom->addr.ToString(), pfrom->nVersion);
            pfrom->PushMessage("reject", strCommand, REJECT_OBSOLETE,
                               strprintf("Version must be %d or greater", MIN_PEER_PROTO_VERSION));
            pfrom->fDisconnect = true;
            return false;
        }

        if (pfrom->nVersion == 10300)
            pfrom->nVersion = 300;
  
  			//继续解析包
        if (!vRecv.empty())
            vRecv >> addrFrom >> nNonce;
        if (!vRecv.empty()) {
            vRecv >> LIMITED_STRING(pfrom->strSubVer, 256);
            pfrom->cleanSubVer = SanitizeString(pfrom->strSubVer);
        }
        if (!vRecv.empty())
            vRecv >> pfrom->nStartingHeight;
  		//这个是前面我没搞明白的true的包解析
  		// 说是为了设置之后的第一个信息过滤
        if (!vRecv.empty())
            vRecv >> pfrom->fRelayTxes; // set to true after we get the first filter* message
        else
            pfrom->fRelayTxes = true;

  // 上面是解析数据
  // 下面就是获取发来过的报文之后的处理了
        if (pfrom->fInbound && addrMe.IsRoutable())
        {
            pfrom->addrLocal = addrMe;
            SeenLocal(addrMe);
        }

        // Disconnect if we connected to ourself
        if (nNonce == nLocalHostNonce && nNonce > 1)
        {
            LogPrintf("connected to self at %s, disconnecting\n", pfrom->addr.ToString());
            pfrom->fDisconnect = true;
            return true;
        }

        // Be shy and don't send version until we hear
        if (pfrom->fInbound)
            pfrom->PushVersion();

        pfrom->fClient = !(pfrom->nServices & NODE_NETWORK);


        // Change version
        pfrom->PushMessage("verack");			//这里push是仅仅带了报文头而已
  			// 改变版本号
        pfrom->ssSend.SetVersion(min(pfrom->nVersion, PROTOCOL_VERSION));

        if (!pfrom->fInbound)
        {
            // Advertise our address
            if (!fNoListen && !IsInitialBlockDownload())
            {
                CAddress addr = GetLocalAddress(&pfrom->addr);
                if (addr.IsRoutable())
                    pfrom->PushAddress(addr);
            }

            // Get recent addresses
            if (pfrom->fOneShot || pfrom->nVersion >= CADDR_TIME_VERSION || addrman.size() < 1000)
            {
                pfrom->PushMessage("getaddr");
                pfrom->fGetAddr = true;
            }
            addrman.Good(pfrom->addr);
        } else {
            if (((CNetAddr)pfrom->addr) == (CNetAddr)addrFrom)
            {
                addrman.Add(addrFrom, addrFrom);
                addrman.Good(addrFrom);
            }
        }

        // Relay alerts
        {
            LOCK(cs_mapAlerts);
            BOOST_FOREACH(PAIRTYPE(const uint256, CAlert)& item, mapAlerts)
                item.second.RelayTo(pfrom);
        }

        pfrom->fSuccessfullyConnected = true;

        LogPrintf("receive version message: %s: version %d, blocks=%d, us=%s, them=%s, peer=%s\n", pfrom->cleanSubVer, pfrom->nVersion, pfrom->nStartingHeight, addrMe.ToString(), addrFrom.ToString(), pfrom->addr.ToString());

        AddTimeData(pfrom->addr, nTime);
    }
```

太久远了，这里在提醒一下：

- 本地节点（即将加入到比特币网络的节点）发送VERSION报文至某一个远程节点（网络节点）；
- 远程节点发送VERSION报文；
- 远程节点回复VERACK报文；
- 远程节点设置协议版本为两者的最小值；
- 本地节点回复VERACK报文；
- 本地节点设置协议版本为两者的最小值；



这里是处理收到的报文是verack的情况：取两者的最小值


```cpp
    else if (strCommand == "verack")
    {
        pfrom->SetRecvVersion(min(pfrom->nVersion, PROTOCOL_VERSION));
    }
```



A:pushVersion	===>		B:provessVersion 	===>    B:(provessVersion中)pushVersion    ===>  A:provessVersion & B:(provessVersion中)pushVerAck    ===>     A:(provessVersion中)pushVersion(但是B已经process一次了，退出)    &    A:(processVerAck)SetVersion    &	B:(provessVersion中)SetVersion   ===>.    A:(provessVersion中)pushVerAck    ===>  A:(provessVersion中)SetVersion 	&	B:(provessVersion中)SetVersion  

有点怪怪的。。。


参考：

1.https://github.com/bitcoin/bitcoin

2.https://www.gopherliu.com/2018/12/02/details-of-bitcoin-network/
