#### 1.数据库代理服务器db_proxy_server

​		主循环逻辑

````c
main(int argc,char*[]argv)
{
    //初始化redis数据库
    //初始化MySQL数据库
    //初始化任务队列
    //初始化同步中心
    //初始化网络监听10600端口
    //网络事件循环
}
````

##### 1.1 Redis初始化

````c
//初始化redis
CacheManager* pCacheManager = CacheManager::getInstance();
if (!pCacheManager)
{
    log("CacheManager init failed");
    return -1;
}
````

````c
//单例模型
CacheManager* CacheManager::getInstance()
{
	if (!s_cache_manager) {
		s_cache_manager = new CacheManager();//构建redis管理器
		if (s_cache_manager->Init()) {//初始化
			delete s_cache_manager;
			s_cache_manager = NULL;
		}
	}
	return s_cache_manager;
}
````

​		读取配置文件，根据CacheInstances配置数据库缓存池。将缓冲池保存在m_cache_pool_map中

````c
int CacheManager::Init()
{
	CConfigFileReader config_file("dbproxyserver.conf");//读取代理服务器的配置文件

    //CacheInstances=unread,group_set,token,sync,group_member
	char* cache_instances = config_file.GetConfigName("CacheInstances");//读取关键字
	if (!cache_instances) {
		log("not configure CacheIntance");
		return 1;
	}

	char host[64];
	char port[64];
	char db[64];
    char maxconncnt[64];
	CStrExplode instances_name(cache_instances, ',');
	for (uint32_t i = 0; i < instances_name.GetItemCnt(); i++) 
    {
        //根据redis服务器的配置，设置unread(未读消息计数器),group_set(群组设置),syn(同步控制),token(设备令牌),group_member(群组成员)等服务的host,port,db,maxconncnt等参数。
		char* pool_name = instances_name.GetItem(i);
		//printf("%s", pool_name);
		snprintf(host, 64, "%s_host", pool_name);
		snprintf(port, 64, "%s_port", pool_name);
		snprintf(db, 64, "%s_db", pool_name);
        snprintf(maxconncnt, 64, "%s_maxconncnt", pool_name);

		char* cache_host = config_file.GetConfigName(host);
		char* str_cache_port = config_file.GetConfigName(port);
		char* str_cache_db = config_file.GetConfigName(db);
        char* str_max_conn_cnt = config_file.GetConfigName(maxconncnt);
		if (!cache_host || !str_cache_port || !str_cache_db || !str_max_conn_cnt) {
			log("not configure cache instance: %s", pool_name);
			return 2;
		}
		//为每一个功能添加数据池。
		CachePool* pCachePool = new CachePool(pool_name, cache_host, atoi(str_cache_port),atoi(str_cache_db), atoi(str_max_conn_cnt));//创建数据池
		if (pCachePool->Init()) {//初始化数据池
			log("Init cache pool failed");
			return 3;
		}
		m_cache_pool_map.insert(make_pair(pool_name, pCachePool));//将缓存池以键值对的形式添加到系统数据池。
	}
	return 0;
}
````

​		实际连接数据库的接口，每个缓存池维持两个数据库连接。m_free_list链表保存着所有的和数据库的连接。

````c
int CachePool::Init()
{
	for (int i = 0; i < m_cur_conn_cnt; i++) //MIN_CACHE_CONN_CNT=2
    {
		CacheConn* pConn = new CacheConn(this);//创建数据库连接
		if (pConn->Init()) {//初始化连接
			delete pConn;
			return 1;
		}
		m_free_list.push_back(pConn);//将连接添加到链表中
	}
	log("cache pool: %s, list size: %lu", m_pool_name.c_str(), m_free_list.size());
	return 0;
}
````

​		redis数据库初始化连接和重连。每4秒重连一次，每次设置200ms超时，连接成功后，切换数据库号。

````c
/*
 * redis初始化连接和重连操作，类似mysql_ping()
 */
int CacheConn::Init()
{
	if (m_pContext) {
		return 0;
	}
	// 4s 尝试重连一次
	uint64_t cur_time = (uint64_t)time(NULL);
	if (cur_time < m_last_connect_time + 4) {
		return 1;
	}
	m_last_connect_time = cur_time;//记录重连时间
	// 200ms超时
	struct timeval timeout = {0, 200000};
	m_pContext = redisConnectWithTimeout(m_pCachePool->GetServerIP(), m_pCachePool->GetServerPort(), timeout);//连接redis数据库
	if (!m_pContext || m_pContext->err) {//连接失败
		if (m_pContext) {
			log("redisConnect failed: %s", m_pContext->errstr);
			redisFree(m_pContext);
			m_pContext = NULL;
		} else {
			log("redisConnect failed");
		}
		return 1;
	}
	//连接成功后,发出select db请求，切换数据库。
	redisReply* reply = (redisReply *)redisCommand(m_pContext, "SELECT %d", m_pCachePool->GetDBNum());
	if (reply && (reply->type == REDIS_REPLY_STATUS) && (strncmp(reply->str, "OK", 2) == 0)) {
		freeReplyObject(reply);
		return 0;
	} else {
		log("select cache db failed");
		return 2;
	}
}
````

​		根据以上的分析，CacheManager开辟了5个数据缓冲池，分别存储着unread，group_set等数据，每个缓存池有两路和redis连接的连接。所有的连接都保存在m_free_list链表里。



##### 1.2 MySQL初始化

````c
CDBManager* pDBManager = CDBManager::getInstance();//MySQL数据库管理单元
if (!pDBManager)
{
    log("DBManager init failed");
    return -1;
}
puts("db init success");
````

​		初始化MySQL数据管理器，单例

````c
CDBManager* CDBManager::getInstance()//单例
{
	if (!s_db_manager) 
    {
		s_db_manager = new CDBManager();//创建管理器
		if (s_db_manager->Init()) {//初始化
			delete s_db_manager;
			s_db_manager = NULL;
		}
	}
	return s_db_manager;
}
````

​			读取配置文件，配置teamtalk_master和teamtalk_slave两个和数据池。最后将 数据池 保存在m_dbpool_map这个map中

````c
int CDBManager::Init()
{
	CConfigFileReader config_file("dbproxyserver.conf");//读取配置文件
    //DBInstances=teamtalk_master,teamtalk_slave
	char* db_instances = config_file.GetConfigName("DBInstances");//获取DBInstances配置

	if (!db_instances) {
		log("not configure DBInstances");
		return 1;
	}

	char host[64];
	char port[64];
	char dbname[64];
	char username[64];
	char password[64];
    char maxconncnt[64];
	CStrExplode instances_name(db_instances, ',');

    //配置teamtalk_master和teamtalk_slave两个对象的数据库
	for (uint32_t i = 0; i < instances_name.GetItemCnt(); i++) //DBInstances=teamtalk_master,teamtalk_slave 
    {
		char* pool_name = instances_name.GetItem(i);
		snprintf(host, 64, "%s_host", pool_name);
		snprintf(port, 64, "%s_port", pool_name);
		snprintf(dbname, 64, "%s_dbname", pool_name);
		snprintf(username, 64, "%s_username", pool_name);
		snprintf(password, 64, "%s_password", pool_name);
        snprintf(maxconncnt, 64, "%s_maxconncnt", pool_name);

		char* db_host = config_file.GetConfigName(host);
		char* str_db_port = config_file.GetConfigName(port);
		char* db_dbname = config_file.GetConfigName(dbname);
		char* db_username = config_file.GetConfigName(username);
		char* db_password = config_file.GetConfigName(password);
        char* str_maxconncnt = config_file.GetConfigName(maxconncnt);

		if (!db_host || !str_db_port || !db_dbname || !db_username || !db_password || !str_maxconncnt) {
			log("not configure db instance: %s", pool_name);
			return 2;
		}

		int db_port = atoi(str_db_port);
        int db_maxconncnt = atoi(str_maxconncnt);
		CDBPool* pDBPool = new CDBPool(pool_name, db_host, db_port, db_username, db_password, db_dbname, db_maxconncnt);//创建数据池
		if (pDBPool->Init()) //初始化数据池
        {
			log("init db instance failed: %s", pool_name);
			return 3;
		}
		m_dbpool_map.insert(make_pair(pool_name, pDBPool));
	}
	return 0;
}
````

​		MySQL数据池初始化。每个数据池创建两路和MySQL连接的服务。并将创建的连接添加到m_free_list链表中。

````c
int CDBPool::Init()
{
	for (int i = 0; i < m_db_cur_conn_cnt; i++) //MIN_DB_CONN_CNT=2
    {
		CDBConn* pDBConn = new CDBConn(this);//创建与MySQL的连接
		int ret = pDBConn->Init();//连接MySQL
		if (ret) {
			delete pDBConn;
			return ret;
		}
		m_free_list.push_back(pDBConn);//将连接保存在m_free_list链表中
	}
	log("db pool: %s, size: %d", m_pool_name.c_str(), (int)m_free_list.size());
	return 0;
}
````

​		初始化mysql，设置重连，设置数据库字符集，然后连接mysql。

````c

int CDBConn::Init()
{
	m_mysql = mysql_init(NULL);
	if (!m_mysql)
    {
		log("mysql_init failed");
		return 1;
	}
	my_bool reconnect = true;
	mysql_options(m_mysql, MYSQL_OPT_RECONNECT, &reconnect);
	mysql_options(m_mysql, MYSQL_SET_CHARSET_NAME, "utf8mb4");
	if (!mysql_real_connect(m_mysql, m_pDBPool->GetDBServerIP(), m_pDBPool->GetUsername(), m_pDBPool->GetPasswrod(),m_pDBPool->GetDBName(), m_pDBPool->GetDBServerPort(), NULL, 0)) {
		log("mysql_real_connect failed: %s", mysql_error(m_mysql));
		return 2;
	}
	return 0;
}
````

​		以上CDBManager创建了两个数据池，teamtalk_master和teamtalk_slave，数据池保存在m_dbpool_map，每个数据处理维持两路和MySQL的连接，一共4路连接。所有的连接都保存在m_free_list链表中。每个数据池最多有16个连接，因此最多32路连接



##### 1.3 初始化代理任务队列

````c
...
CConfigFileReader config_file("dbproxyserver.conf");//读取配置文件
char* str_thread_num = config_file.GetConfigName("ThreadNum");//ThreadNum=48
uint32_t thread_num = atoi(str_thread_num);
...
init_proxy_conn(thread_num);//初始化代理任务队列
````

​		初始化代理服务器

````c
int init_proxy_conn(uint32_t thread_num)
{
	s_handler_map = CHandlerMap::getInstance();//2.命令和函数映射
	g_thread_pool.Init(thread_num);//1.初始化线程池
	netlib_add_loop(proxy_loop_callback, NULL);//添加网络事件处理函数，后面分析可以知道，实际上注册了响应客户端的回调函数
	signal(SIGTERM, sig_handler);//注册终止信息
	return netlib_register_timer(proxy_timer_callback, NULL, 1000);
}
````

​		1.线程池初始化。下面是线程池的一下基本操作接口。

````c
//线程池初始化
int CThreadPool::Init(uint32_t worker_size)
{
    m_worker_size = worker_size;
	m_worker_list = new CWorkerThread [m_worker_size];//初始化线程池
	if (!m_worker_list) {
		return 1;
	}
	for (uint32_t i = 0; i < m_worker_size; i++) {//48
		m_worker_list[i].SetThreadIdx(i);//设置线程号
		m_worker_list[i].Start();//启动线程
	}
	return 0;
}
//启动线程
void CWorkerThread::Start()
{
	(void)pthread_create(&m_thread_id, NULL, StartRoutine, this);//pthread_create
}
//线程池任务调度
void CWorkerThread::Execute()
{
	while (true) 
    {
		m_thread_notify.Lock();//互斥加锁
		// put wait in while cause there can be spurious wake up (due to signal/ENITR)
		while (m_task_list.empty()) {//任务队列为空，等待任务加入后唤醒任务调度
			m_thread_notify.Wait();
		}
		CTask* pTask = m_task_list.front();//唤醒后获取队列头
		m_task_list.pop_front();
		m_thread_notify.Unlock();//取出任务后，立即解锁，减小互斥量粒度
		pTask->run();//执行任务
		delete pTask;//删除任务
		m_task_cnt++;
		//log("%d have the execute %d task\n", m_thread_idx, m_task_cnt);
	}
}
//线程池添加任务
void CWorkerThread::PushTask(CTask* pTask)
{
	m_thread_notify.Lock();//获取互斥锁
	m_task_list.push_back(pTask);//添加任务到任务队列
	m_thread_notify.Signal();//唤醒任务队列
	m_thread_notify.Unlock();
}
````

​		2.任务队列初始化时，命令接口映射。

````c
CHandlerMap* CHandlerMap::getInstance()//命令Map
{
	if (!s_handler_instance) {
		s_handler_instance = new CHandlerMap();
		s_handler_instance->Init();//将命令和函数映射起来
	}
	return s_handler_instance;
}
````

````c
/**
 *  初始化函数,加载了各种commandId 对应的处理函数
 */
void CHandlerMap::Init()
{
    //DB_PROXY是命名空间，不是类名
	// Login validate
	m_handler_map.insert(make_pair(uint32_t(CID_OTHER_VALIDATE_REQ), DB_PROXY::doLogin));//登录
    m_handler_map.insert(make_pair(uint32_t(CID_LOGIN_REQ_PUSH_SHIELD), DB_PROXY::doPushShield));//push
    m_handler_map.insert(make_pair(uint32_t(CID_LOGIN_REQ_QUERY_PUSH_SHIELD), DB_PROXY::doQueryPushShield));//query
    // recent session
    m_handler_map.insert(make_pair(uint32_t(CID_BUDDY_LIST_RECENT_CONTACT_SESSION_REQUEST), DB_PROXY::getRecentSession));
    m_handler_map.insert(make_pair(uint32_t(CID_BUDDY_LIST_REMOVE_SESSION_REQ), DB_PROXY::deleteRecentSession));
    // users
    m_handler_map.insert(make_pair(uint32_t(CID_BUDDY_LIST_USER_INFO_REQUEST), DB_PROXY::getUserInfo));//userinfo
    m_handler_map.insert(make_pair(uint32_t(CID_BUDDY_LIST_ALL_USER_REQUEST), DB_PROXY::getChangedUser));//changeinfo
    m_handler_map.insert(make_pair(uint32_t(CID_BUDDY_LIST_DEPARTMENT_REQUEST), DB_PROXY::getChgedDepart));//changedepart
    m_handler_map.insert(make_pair(uint32_t(CID_BUDDY_LIST_CHANGE_SIGN_INFO_REQUEST), DB_PROXY::changeUserSignInfo));//user sign info
    // message content
    m_handler_map.insert(make_pair(uint32_t(CID_MSG_DATA), DB_PROXY::sendMessage));//send message
    m_handler_map.insert(make_pair(uint32_t(CID_MSG_LIST_REQUEST), DB_PROXY::getMessage));//get message
    m_handler_map.insert(make_pair(uint32_t(CID_MSG_UNREAD_CNT_REQUEST), DB_PROXY::getUnreadMsgCounter));//get unread msgcounter
    m_handler_map.insert(make_pair(uint32_t(CID_MSG_READ_ACK), DB_PROXY::clearUnreadMsgCounter));//cler unread msgcounter
    m_handler_map.insert(make_pair(uint32_t(CID_MSG_GET_BY_MSG_ID_REQ), DB_PROXY::getMessageById));//get message by id
    m_handler_map.insert(make_pair(uint32_t(CID_MSG_GET_LATEST_MSG_ID_REQ), DB_PROXY::getLatestMsgId));//get last messageid
    // device token
    m_handler_map.insert(make_pair(uint32_t(CID_LOGIN_REQ_DEVICETOKEN), DB_PROXY::setDevicesToken));//set devices token
    m_handler_map.insert(make_pair(uint32_t(CID_OTHER_GET_DEVICE_TOKEN_REQ), DB_PROXY::getDevicesToken));//get devices token
    //push 推送设置
    m_handler_map.insert(make_pair(uint32_t(CID_GROUP_SHIELD_GROUP_REQUEST), DB_PROXY::setGroupPush));//group push
    m_handler_map.insert(make_pair(uint32_t(CID_OTHER_GET_SHIELD_REQ), DB_PROXY::getGroupPush));//
    // group
    m_handler_map.insert(make_pair(uint32_t(CID_GROUP_NORMAL_LIST_REQUEST), DB_PROXY::getNormalGroupList));//get group list
    m_handler_map.insert(make_pair(uint32_t(CID_GROUP_INFO_REQUEST), DB_PROXY::getGroupInfo));//get group info
    m_handler_map.insert(make_pair(uint32_t(CID_GROUP_CREATE_REQUEST), DB_PROXY::createGroup));//create group
    m_handler_map.insert(make_pair(uint32_t(CID_GROUP_CHANGE_MEMBER_REQUEST), DB_PROXY::modifyMember));//modify member
    // file
    m_handler_map.insert(make_pair(uint32_t(CID_FILE_HAS_OFFLINE_REQ), DB_PROXY::hasOfflineFile));// has offline file
    m_handler_map.insert(make_pair(uint32_t(CID_FILE_ADD_OFFLINE_REQ), DB_PROXY::addOfflineFile));// add offline file
    m_handler_map.insert(make_pair(uint32_t(CID_FILE_DEL_OFFLINE_REQ), DB_PROXY::delOfflineFile));// del offline file
}

````

​		3.注册SIGTERM信号处理函数，一般用来在关闭程序前，做一些清理工作。这里貌似会同时MySQL和redis之间的同步服务。

````c
static void sig_handler(int sig_no)
{
	if (sig_no == SIGTERM) {
		log("receive SIGTERM, prepare for exit");
        CImPdu cPdu;
        IM::Server::IMStopReceivePacket msg;
        msg.set_result(0);
        cPdu.SetPBMsg(&msg);
        cPdu.SetServiceId(IM::BaseDefine::SID_OTHER);
        cPdu.SetCommandId(IM::BaseDefine::CID_OTHER_STOP_RECV_PACKET);
        for (ConnMap_t::iterator it = g_proxy_conn_map.begin(); it != g_proxy_conn_map.end(); it++) {
            CProxyConn* pConn = (CProxyConn*)it->second;
            pConn->SendPdu(&cPdu);
        }
        // Add By ZhangYuanhao
        // Before stop we need to stop the sync thread,otherwise maybe will not sync the internal data any more
        CSyncCenter::getInstance()->stopSync();
        
        // callback after 4 second to exit process;
		netlib_register_timer(exit_callback, NULL, 4000);
	}
}
````



##### 1.4 同步MySQL数据到Redis

````c
CSyncCenter::getInstance()->init();	//初始化同步
CSyncCenter::getInstance()->startSync();//开始同步
````

​		init，加载上一次的同步信息

````c
/*
 * 初始化函数，从cache里面加载上次同步的时间信息等
 */
void CSyncCenter::init()
{
    // Load total update time
    CacheManager* pCacheManager = CacheManager::getInstance();//获取redis数据库的信息
    // increase message count
    CacheConn* pCacheConn = pCacheManager->GetCacheConn("unread");//读取unread数据池
    if (pCacheConn)
    {
        string strTotalUpdate = pCacheConn->get("total_user_updated");

        string strLastUpdateGroup = pCacheConn->get("last_update_group");
        pCacheManager->RelCacheConn(pCacheConn);
	    if(strTotalUpdate != "")
        {
            m_nLastUpdate = string2int(strTotalUpdate);
        }
        else
        {
            updateTotalUpdate(time(NULL));
        }
        if(strLastUpdateGroup.empty())
        {
            m_nLastUpdateGroup = string2int(strLastUpdateGroup);
        }
        else
        {
            updateLastUpdateGroup(time(NULL));
        }
    }
    else
    {
        log("no cache connection to get total_user_updated");
    }
}
````

​		开启同步，开启单独的线程区同步数据。

````c
/**
 *  开启内网数据同步以及群组聊天记录同步
 */
void CSyncCenter::startSync()
{
#ifdef _WIN32
    (void)CreateThread(NULL, 0, doSyncGroupChat, NULL, 0, &m_nGroupChatThreadId);
#else
    (void)pthread_create(&m_nGroupChatThreadId, NULL, doSyncGroupChat, NULL);
#endif
}

/**
 *  同步群组聊天信息
 *  @param arg NULL
 *  @return NULL
 */
void* CSyncCenter::doSyncGroupChat(void* arg)
{
    m_bSyncGroupChatRuning = true;
    CDBManager* pDBManager = CDBManager::getInstance();//获取MySQL对象
    map<uint32_t, uint32_t> mapChangedGroup;
    do{
        mapChangedGroup.clear();
        CDBConn* pDBConn = pDBManager->GetDBConn("teamtalk_slave");//1.从teamtalk_slaves数据池取出 数据库连接 ，若没有空闲的连接则新建一个连接，若连接数申请到最大，只能等待工作的连接结束后再返回连接。
        if(pDBConn)
        {
            string strSql = "select id, lastChated from IMGroup where status=0 and lastChated >=" + int2string(m_pInstance->getLastUpdateGroup());
            CResultSet* pResult = pDBConn->ExecuteQuery(strSql.c_str());//构建sql语句查询最后更新群组信息时间之后的，status==0(正常)消息
            if(pResult)
            {
                while (pResult->Next()) {
                    uint32_t nGroupId = pResult->GetInt("id");
                    uint32_t nLastChat = pResult->GetInt("lastChated");
                    if(nLastChat != 0)
                    {
                        mapChangedGroup[nGroupId] = nLastChat;//添加到map里面
                    }
                }
                delete pResult;
            }
            pDBManager->RelDBConn(pDBConn);//将 数据库连接 重新添加到m_free_list(空闲连接池)里面
        }
        else
        {
            log("no db connection for teamtalk_slave");
        }
        m_pInstance->updateLastUpdateGroup(time(NULL));//2.将MySQL数据同步到Redis
        for (auto it=mapChangedGroup.begin(); it!=mapChangedGroup.end(); ++it)//将MySQL群组聊天信息
        {
            uint32_t nGroupId =it->first;//群组id
            list<uint32_t> lsUsers;
            uint32_t nUpdate = it->second;//群组最后聊天时间
            CGroupModel::getInstance()->getGroupUser(nGroupId, lsUsers);//获得群组用户
            for (auto it1=lsUsers.begin(); it1!=lsUsers.end(); ++it1)//遍历所有群组用户，
            {
                uint32_t nUserId = *it1;
                uint32_t nSessionId = INVALID_VALUE;
                //获得群组用户的session id
                nSessionId = CSessionModel::getInstance()->getSessionId(nUserId, nGroupId, IM::BaseDefine::SESSION_TYPE_GROUP, true);
                if(nSessionId != INVALID_VALUE)
                {
                    CSessionModel::getInstance()->updateSession(nSessionId, nUpdate);//若是正常的id，则更新数据
                }
                else
                {
                    CSessionModel::getInstance()->addSession(nUserId, nGroupId, IM::BaseDefine::SESSION_TYPE_GROUP);//否则发送
                }
            }
        }
//    } while (!m_pInstance->m_pCondSync->waitTime(5*1000));
    } while (m_pInstance->m_bSyncGroupChatWaitting && !(m_pInstance->m_pCondGroupChat->waitTime(5*1000)));//同步中,同时5s内每收到超时信号。
//    } while(m_pInstance->m_bSyncGroupChatWaitting);
    m_bSyncGroupChatRuning = false;
    return NULL;
}
		
````

​		1.从MySQL取出连接

````c
CDBConn* CDBManager::GetDBConn(const char* dbpool_name)  //从连接池里找到dbpool_name名称的数据库连接,若没有空闲的连接则新建一个连接，若已满，则等待
{
	map<string, CDBPool*>::iterator it = m_dbpool_map.find(dbpool_name);
	if (it == m_dbpool_map.end()) {//找不到
		return NULL;
	} else {
		return it->second->GetDBConn();//找到
	}
}
CDBConn* CDBPool::GetDBConn()
{
	m_free_notify.Lock();//访问共享资源，加锁

	while (m_free_list.empty()) //没有空闲连接
    {
		if (m_db_cur_conn_cnt >= m_db_max_conn_cnt) {
			m_free_notify.Wait();
		} else {
			CDBConn* pDBConn = new CDBConn(this);//创建一个新的连接
			int ret = pDBConn->Init();//初始化
			if (ret) {//失败
				log("Init DBConnecton failed");
				delete pDBConn;
				m_free_notify.Unlock();
				return NULL;
			} else {//成功
				m_free_list.push_back(pDBConn);//添加到空闲连接池
				m_db_cur_conn_cnt++;
				log("new db connection: %s, conn_cnt: %d", m_pool_name.c_str(), m_db_cur_conn_cnt);
			}
		}
	}

	CDBConn* pConn = m_free_list.front();//获得头
	m_free_list.pop_front();//弹出

	m_free_notify.Unlock();

	return pConn;
}
````

​		2.更新Redis群组信息

````c
/**
 *  更新上次同步群组信息时间
 *
 *  @param nUpdated 时间
 */
void CSyncCenter::updateLastUpdateGroup(uint32_t nUpdated)
{
    CacheManager* pCacheManager = CacheManager::getInstance();//获得redis CacheManager
    CacheConn* pCacheConn = pCacheManager->GetCacheConn("unread");	//取出连接，取出的逻辑和上面的CDBManager是一样的
    if (pCacheConn) {
        last_update_lock_.lock();
        m_nLastUpdateGroup = nUpdated;
        string strUpdated = int2string(nUpdated);
        last_update_lock_.unlock();
  
        pCacheConn->set("last_update_group", strUpdated);//更新群组最后更新时间信息
        pCacheManager->RelCacheConn(pCacheConn);//将工作连接返还m_free_list
    }
    else
    {
        log("no cache connection to get total_user_updated");
    }
}
````



##### 1.5 在10600端口启动侦听，监听新连接。

````c
...
char* listen_ip = config_file.GetConfigName("ListenIP");//ListenIP=0.0.0.0
char* str_listen_port = config_file.GetConfigName("ListenPort");//ListenPort=10600
uint16_t listen_port = atoi(str_listen_port);
...
CStrExplode listen_ip_list(listen_ip, ';');//多个IP可以以;分隔
for (uint32_t i = 0; i < listen_ip_list.GetItemCnt(); i++)//遍历侦听的ip列表，启动侦听
{
    ret = netlib_listen(listen_ip_list.GetItem(i), listen_port, proxy_serv_callback, NULL);
    if (ret == NETLIB_ERROR)
        return ret;
}
````

````c
int netlib_listen(const char* server_ip, uint16_t port,callback_t callback,void* callback_data)
{
	CBaseSocket* pSocket = new CBaseSocket();
	if (!pSocket)
		return NETLIB_ERROR;
	int ret =  pSocket->Listen(server_ip, port, callback, callback_data);
	if (ret == NETLIB_ERROR)
		delete pSocket;
	return ret;
}
````

````c
//侦听IPserver_ip，侦听端口port，侦听回调函数callback，
int CBaseSocket::Listen(const char* server_ip, uint16_t port, callback_t callback, void* callback_data)
{
	m_local_ip = server_ip;
	m_local_port = port;
	m_callback = callback;
	m_callback_data = callback_data;

	m_socket = socket(AF_INET, SOCK_STREAM, 0);
	if (m_socket == INVALID_SOCKET)
	{
		printf("socket failed, err_code=%d\n", _GetErrorCode());
		return NETLIB_ERROR;
	}

	_SetReuseAddr(m_socket);//serverfd addr reuse
	_SetNonblock(m_socket);//serverfd non block

	sockaddr_in serv_addr;
	_SetAddr(server_ip, port, &serv_addr);
    int ret = ::bind(m_socket, (sockaddr*)&serv_addr, sizeof(serv_addr));//bind
	if (ret == SOCKET_ERROR)
	{
		LOG__(NET,  _T("bind failed, err_code=%d"), _GetErrorCode());
		closesocket(m_socket);
		return NETLIB_ERROR;
	}

	ret = listen(m_socket, 64);//listen
	if (ret == SOCKET_ERROR)
	{
		LOG__(NET,  _T("listen failed, err_code=%d"), _GetErrorCode());
		closesocket(m_socket);
		return NETLIB_ERROR;
	}

	m_state = SOCKET_STATE_LISTENING;//侦听socket,区别IO socket

	LOGA__(NET, "CBaseSocket::Listen on %s:%d", server_ip, port);

	AddBaseSocket(this);
	CEventDispatch::Instance()->AddEvent(m_socket, SOCKET_READ | SOCKET_EXCEP);//添加异常事件侦听
	return NETLIB_OK;
}
````

​		添加到全局数据区

````c
typedef hash_map<net_handle_t, CBaseSocket*> SocketMap;//hash_map，查询事件复杂度是O(1)
SocketMap	g_socket_map;
void AddBaseSocket(CBaseSocket* pSocket)
{
	g_socket_map.insert(make_pair((net_handle_t)pSocket->GetSocket(), pSocket));
}
````

​		epoll，ET模式，read要读完全部数据，write最好的做法是先写，写不完再添加可读事件监听。

````c
void CEventDispatch::AddEvent(SOCKET fd, uint8_t socket_event)
{
	struct epoll_event ev;
	ev.events = EPOLLIN | EPOLLOUT | EPOLLET | EPOLLPRI | EPOLLERR | EPOLLHUP;
	ev.data.fd = fd;
	if (epoll_ctl(m_epfd, EPOLL_CTL_ADD, fd, &ev) != 0)
	{
		log("epoll_ctl() failed, errno=%d", errno);
	}
}
````



##### 1.6 事件监听循环。

````c
printf("server start listen on: %s:%d\n", listen_ip, listen_port);
printf("now enter the event loop...\n");
writePid();
netlib_eventloop(10);
````

````c
void netlib_eventloop(uint32_t wait_timeout)
{
	CEventDispatch::Instance()->StartDispatch(wait_timeout);
}
````

````c
//主要逻辑
while(退出条件)
{
    //io复用
    //遍历可用socket
    //可读
    //可写
    //异常
    //定时器
    //其他事件(如响应客户端)
}

//监听服务器事件，并进行处理
void CEventDispatch::StartDispatch(uint32_t wait_timeout)
{
	struct epoll_event events[1024];
	int nfds = 0;

    if(running)
        return;
    running = true;
    
	while (running)
	{
		nfds = epoll_wait(m_epfd, events, 1024, wait_timeout);
		for (int i = 0; i < nfds; i++)
		{
			int ev_fd = events[i].data.fd;
			CBaseSocket* pSocket = FindBaseSocket(ev_fd);
			if (!pSocket)
				continue;
            //Commit by zhfu @2015-02-28
            #ifdef EPOLLRDHUP
            if (events[i].events & EPOLLRDHUP)//客户端关闭(貌似没触发过)
            {
                //log("On Peer Close, socket=%d, ev_fd);
                pSocket->OnClose();
            }
            #endif
            // Commit End
			if (events[i].events & EPOLLIN)//数据可读
			{
				//log("OnRead, socket=%d\n", ev_fd);
				pSocket->OnRead();
			}
			if (events[i].events & EPOLLOUT)//数据可写
			{
				//log("OnWrite, socket=%d\n", ev_fd);
				pSocket->OnWrite();
			}
			if (events[i].events & (EPOLLPRI | EPOLLERR | EPOLLHUP))//其他数据
			{
				//log("OnClose, socket=%d\n", ev_fd);
				pSocket->OnClose();
			}

			pSocket->ReleaseRef();
		}
		_CheckTimer();//检查定时器
        _CheckLoop();
	}
}
````
##### 新客户端连接|读事件

````c
void CBaseSocket::_AcceptNewSocket()
{
	SOCKET fd = 0;
	sockaddr_in peer_addr;
	socklen_t addr_len = sizeof(sockaddr_in);
	char ip_str[64];
	while ( (fd = accept(m_socket, (sockaddr*)&peer_addr, &addr_len)) != INVALID_SOCKET )
	{
		CBaseSocket* pSocket = new CBaseSocket();//创建CBaseSocket
		uint32_t ip = ntohl(peer_addr.sin_addr.s_addr);
		uint16_t port = ntohs(peer_addr.sin_port);
		snprintf(ip_str, sizeof(ip_str), "%d.%d.%d.%d", ip >> 24, (ip >> 16) & 0xFF, (ip >> 8) & 0xFF, ip & 0xFF);
		log("AcceptNewSocket, socket=%d from %s:%d\n", fd, ip_str, port);
		pSocket->SetSocket(fd);//添加fd
		pSocket->SetCallback(m_callback);
		pSocket->SetCallbackData(m_callback_data);
		pSocket->SetState(SOCKET_STATE_CONNECTED);//set state
		pSocket->SetRemoteIP(ip_str);//set remote ip
		pSocket->SetRemotePort(port);//set remote port
		_SetNoDelay(fd);//设置TCP_NODELAY,即禁用Nagle算法，允许小包发送，适合数据包比较小的场景。Nagle算法，只有写缓冲达到一定量的时候才会写出。
		_SetNonblock(fd);//no block
		AddBaseSocket(pSocket);//添加到g_socket_map
		CEventDispatch::Instance()->AddEvent(fd, SOCKET_READ | SOCKET_EXCEP);//添加可读和异常事件
		m_callback(m_callback_data, NETLIB_MSG_CONNECT, (net_handle_t)fd, NULL);//设置回调事件，也就是proxy_serv_callback
	}
}

//listen时，将serverfd的回调函数设置成了proxy_serv_callback
void proxy_serv_callback(void* callback_data, uint8_t msg, uint32_t handle, void* pParam)//handle=fd
{
    if (msg == NETLIB_MSG_CONNECT)
    {
        CProxyConn* pConn = new CProxyConn();//创建一个CProxyConnection
        pConn->OnConnect(handle);//执行
    }
    else
    {
        log("!!!error msg: %d", msg);
    }
}

//修改clientfd的各种事件的回调函数
void CProxyConn::OnConnect(net_handle_t handle)//fd
{
	m_handle = handle;

	g_proxy_conn_map.insert(make_pair(handle, this));

	netlib_option(handle, NETLIB_OPT_SET_CALLBACK, (void*)imconn_callback);//修改回调函数为imconn_callback
	netlib_option(handle, NETLIB_OPT_SET_CALLBACK_DATA, (void*)&g_proxy_conn_map);//传入map<fd,proxy_connection>
	netlib_option(handle, NETLIB_OPT_GET_REMOTE_IP, (void*)&m_peer_ip);
	netlib_option(handle, NETLIB_OPT_GET_REMOTE_PORT, (void*)&m_peer_port);

	log("connect from %s:%d, handle=%d", m_peer_ip.c_str(), m_peer_port, m_handle);
}
//////////////////////////////////////////
//以上的逻辑是一个fd对应一个CBaseSocket ，和一个CProxyCon，他们都可以通过map<fd,XXX>来获得
//////////////////////////////////////////
````

​		客户端的新的回调函数。

````c
void imconn_callback(void* callback_data, uint8_t msg, uint32_t handle, void* pParam)
{
	NOTUSED_ARG(handle);
	NOTUSED_ARG(pParam);
	if (!callback_data)
		return;
	ConnMap_t* conn_map = (ConnMap_t*)callback_data;//map<fd,proxy_connection>
	CImConn* pConn = FindImConn(conn_map, handle);//proxyconnection继承CImConn，class CProxyConn : public CImConn {...},并且含有虚函数因此可以使用多态。
	if (!pConn)
		return;
	//log("msg=%d, handle=%d ", msg, handle);
	switch (msg)
	{
	case NETLIB_MSG_CONFIRM:
		pConn->OnConfirm();
		break;
	case NETLIB_MSG_READ:
		pConn->OnRead();
		break;
	case NETLIB_MSG_WRITE:
		pConn->OnWrite();
		break;
	case NETLIB_MSG_CLOSE:
		pConn->OnClose();
		break;
	default:
		log("!!!imconn_callback error msg: %d ", msg);
		break;
	}
	pConn->ReleaseRef();
}
////////////////////////////////////////////////////////
// 上面提到fd对应一个 CProxyConn，实际上，map<fd,connection>中的connect是 CImConn 类型。
````

​		处理可读事件。

````c
// 
// 由于数据包是在另一个线程处理的，所以不能在主线程delete数据包，所以需要Override这个方法
// CProxyConn继承于CImConn，每一路CImConn都有自己独立的读缓冲区，和写缓冲区。
void CProxyConn::OnRead()
{
	for (;;) 
    {
		uint32_t free_buf_len = m_in_buf.GetAllocSize() - m_in_buf.GetWriteOffset();//获得读缓冲区的剩余空间
		if (free_buf_len < READ_BUF_SIZE)
			m_in_buf.Extend(READ_BUF_SIZE);//空间不足则拓展
		int ret = netlib_recv(m_handle, m_in_buf.GetBuffer() + m_in_buf.GetWriteOffset(), READ_BUF_SIZE);
		if (ret <= 0)//读取完成
			break;
		m_recv_bytes += ret;
		m_in_buf.IncWriteOffset(ret);//写入读缓冲区
		m_last_recv_tick = get_tick_count();//计时
	}
	uint32_t pdu_len = 0;
    try {
        while ( CImPdu::IsPduAvailable(m_in_buf.GetBuffer(), m_in_buf.GetWriteOffset(), pdu_len) ) {//读取包长度
            HandlePduBuf(m_in_buf.GetBuffer(), pdu_len);//处理读包中的命令
            m_in_buf.Read(NULL, pdu_len);//处理完成后会将已读数据删除
        }
    } catch (CPduException& ex) {
        log("!!!catch exception, err_code=%u, err_msg=%s, close the connection ",
            ex.GetErrorCode(), ex.GetErrorMsg());
        OnClose();
    }
	
}
````

​		处理协议包。从读缓冲区里读取数据，转换成协议包，再生成一个包任务，添加到线程池的任务队列处理。

````c
CImPdu{
    ...
    CSimpleBuffer	m_buf;
    PduHeader_t		m_pdu_header;//消息头
    ...
}
typedef struct {
    uint32_t 	length;		  // the whole pdu length
    uint16_t 	version;	  // pdu version number
    uint16_t	flag;		  // not used
    uint16_t	service_id;	  //
    uint16_t	command_id;	  //
    uint16_t	seq_num;     // 包序号
    uint16_t    reversed;    // 保留
} PduHeader_t;

void CProxyConn::HandlePduBuf(uchar_t* pdu_buf, uint32_t pdu_len)
{
    CImPdu* pPdu = NULL;//Instant message protocol data unit , 即时通信消息协议数据单元，即一个协议包
    pPdu = CImPdu::ReadPdu(pdu_buf, pdu_len);//读取
    if (pPdu->GetCommandId() == IM::BaseDefine::CID_OTHER_HEARTBEAT) {//心跳包，不进行处理
        return;
    }
    
    pdu_handler_t handler = s_handler_map->GetHandler(pPdu->GetCommandId());//
    
    if (handler) {
        CTask* pTask = new CProxyTask(m_uuid, handler, pPdu);//生成一个包处理任务
        g_thread_pool.AddTask(pTask);//添加到线程池任务队列处理
    } else {
        log("no handler for packet type: %d", pPdu->GetCommandId());
    }
}
````

​		线程池会将任务随机丢到一个任务队列处理。

````c
void CThreadPool::AddTask(CTask* pTask)
{
	/*
	 * select a random thread to push task
	 * we can also select a thread that has less task to do
	 * but that will scan the whole thread list and use thread lock to get each task size
	 */
	uint32_t thread_idx = random() % m_worker_size;
	m_worker_list[thread_idx].PushTask(pTask);
}
````

​		由以上的读取包的流程，可以得出以下的伪码：

````c
while(退出条件)
{
    //监听socket 可读事件
    //执行读回调函数
    //accept时，生成CBaseSocket,添加到g_socket_map;重定位回调函数
    //read时，生成CProxyConn,添加到 g_proxy_conn_map.执行回调函数，将数据写入m_in_buf(读缓冲区),数据长度不足包头大小，跳出循环。从包头获得包体大小，若缓冲区无法满足包头+包体大小，跳出循环，否则开始解包。根据包命令处理数据，并转换成一个任务，传递给任务队列执行。
    //清除刚才处理的数据。
}
````

​		

##### 消息响应

对于任务队列处理完数据后的应答流程，以登录数据包来进行分析。

````C
void CHandlerMap::Init()
{
    //DB_PROXY是命名空间，不是类名
	// Login validate
	m_handler_map.insert(make_pair(uint32_t(CID_OTHER_VALIDATE_REQ), DB_PROXY::doLogin));
    ...
}
//login
void doLogin(CImPdu* pPdu, uint32_t conn_uuid)
{
    
    CImPdu* pPduResp = new CImPdu;
    
    IM::Server::IMValidateReq msg;//请求消息
    IM::Server::IMValidateRsp msgResp;//响应消息
    if(msg.ParseFromArray(pPdu->GetBodyData(), pPdu->GetBodyLength()))
    {
        
        string strDomain = msg.user_name();//用户名称
        string strPass = msg.password();//用户密码
        
        msgResp.set_user_name(strDomain);//响应用户名称
        msgResp.set_attach_data(msg.attach_data());//响应数据
        
        do
        {
            CAutoLock cAutoLock(&g_cLimitLock);
            list<uint32_t>& lsErrorTime = g_hmLimits[strDomain];
            uint32_t tmNow = time(NULL);
            
            //清理超过30分钟的错误时间点记录
            /*
             清理放在这里还是放在密码错误后添加的时候呢？
             放在这里，每次都要遍历，会有一点点性能的损失。
             放在后面，可能会造成30分钟之前有10次错的，但是本次是对的就没办法再访问了。
             */
            auto itTime=lsErrorTime.begin();
            for(; itTime!=lsErrorTime.end();++itTime)//遍历所有的错误的时间，30分钟之前有错误，退出遍历。
            {
                if(tmNow - *itTime > 30*60)
                {
                    break;
                }
            }
            if(itTime != lsErrorTime.end())//中途存在30分钟之前的错误，则将错误删除
            {
                lsErrorTime.erase(itTime, lsErrorTime.end());
            }
            // 判断30分钟内密码错误次数是否大于10
            if(lsErrorTime.size() > 10)
            {
                itTime = lsErrorTime.begin();
                if(tmNow - *itTime <= 30*60)
                {
                    msgResp.set_result_code(6);
                    msgResp.set_result_string("用户名/密码错误次数太多");
                    pPduResp->SetPBMsg(&msgResp);
                    pPduResp->SetSeqNum(pPdu->GetSeqNum());
                    pPduResp->SetServiceId(IM::BaseDefine::SID_OTHER);
                    pPduResp->SetCommandId(IM::BaseDefine::CID_OTHER_VALIDATE_RSP);
                    CProxyConn::AddResponsePdu(conn_uuid, pPduResp);//添加错误响应包
                    return ;
                }
            }
        } while(false);
        //登录
        log("%s request login.", strDomain.c_str());
        IM::BaseDefine::UserInfo cUser;
        if(g_loginStrategy.doLogin(strDomain, strPass, cUser))//1.从MySQL数据库获取数据，进行用户名和密码校验
        {
            IM::BaseDefine::UserInfo* pUser = msgResp.mutable_user_info();
            pUser->set_user_id(cUser.user_id());
            pUser->set_user_gender(cUser.user_gender());
            pUser->set_department_id(cUser.department_id());
            pUser->set_user_nick_name(cUser.user_nick_name());
            pUser->set_user_domain(cUser.user_domain());
            pUser->set_avatar_url(cUser.avatar_url());
            pUser->set_email(cUser.email());
            pUser->set_user_tel(cUser.user_tel());
            pUser->set_user_real_name(cUser.user_real_name());
            pUser->set_status(0);
            pUser->set_sign_info(cUser.sign_info());
            msgResp.set_result_code(0);//result code
            msgResp.set_result_string("成功");//result string
            //如果登陆成功，则清除错误尝试限制
            CAutoLock cAutoLock(&g_cLimitLock);
            list<uint32_t>& lsErrorTime = g_hmLimits[strDomain];
            lsErrorTime.clear();
        }
        else//错误，记录一次错误
        {
            //密码错误，记录一次登陆失败
            uint32_t tmCurrent = time(NULL);
            CAutoLock cAutoLock(&g_cLimitLock);
            list<uint32_t>& lsErrorTime = g_hmLimits[strDomain];
            lsErrorTime.push_front(tmCurrent);
            
            log("get result false");
            msgResp.set_result_code(1);
            msgResp.set_result_string("用户名/密码错误");
        }
    }
    else
    {
        msgResp.set_result_code(2);
        msgResp.set_result_string("服务端内部错误");
    }
    pPduResp->SetPBMsg(&msgResp);
    pPduResp->SetSeqNum(pPdu->GetSeqNum());
    pPduResp->SetServiceId(IM::BaseDefine::SID_OTHER);
    pPduResp->SetCommandId(IM::BaseDefine::CID_OTHER_VALIDATE_RSP);
    CProxyConn::AddResponsePdu(conn_uuid, pPduResp);//2.添加消息响应
}
````

​		1.用户名和密码的校验

````c
bool CInterLoginStrategy::doLogin(const std::string &strName, const std::string &strPass, IM::BaseDefine::UserInfo& user)
{
    bool bRet = false;
    CDBManager* pDBManger = CDBManager::getInstance();
    CDBConn* pDBConn = pDBManger->GetDBConn("teamtalk_slave");//从数据库获得一个连接
    if (pDBConn) {
        string strSql = "select * from IMUser where name='" + strName + "' and status=0";//MySQL查询语句
        CResultSet* pResultSet = pDBConn->ExecuteQuery(strSql.c_str());//进行查询
        if(pResultSet)
        {
            string strResult, strSalt;
            uint32_t nId, nGender, nDeptId, nStatus;
            string strNick, strAvatar, strEmail, strRealName, strTel, strDomain,strSignInfo;
            while (pResultSet->Next()) {
                nId = pResultSet->GetInt("id");
                strResult = pResultSet->GetString("password");
                strSalt = pResultSet->GetString("salt");
                strNick = pResultSet->GetString("nick");
                nGender = pResultSet->GetInt("sex");
                strRealName = pResultSet->GetString("name");
                strDomain = pResultSet->GetString("domain");
                strTel = pResultSet->GetString("phone");
                strEmail = pResultSet->GetString("email");
                strAvatar = pResultSet->GetString("avatar");
                nDeptId = pResultSet->GetInt("departId");
                nStatus = pResultSet->GetInt("status");
                strSignInfo = pResultSet->GetString("sign_info");
            }
            string strInPass = strPass + strSalt;//密码+混淆码
            char szMd5[33];
            CMd5::MD5_Calculate(strInPass.c_str(), strInPass.length(), szMd5);
            string strOutPass(szMd5);
            //去掉密码校验
            //if(strOutPass == strResult)
            {
                bRet = true;
                user.set_user_id(nId);
                user.set_user_nick_name(strNick);
                user.set_user_gender(nGender);
                user.set_user_real_name(strRealName);
                user.set_user_domain(strDomain);
                user.set_user_tel(strTel);
                user.set_email(strEmail);
                user.set_avatar_url(strAvatar);
                user.set_department_id(nDeptId);
                user.set_status(nStatus);
  	        	user.set_sign_info(strSignInfo);
            }
            delete  pResultSet;
        }
        pDBManger->RelDBConn(pDBConn);//归还数据库的连接
    }
    return bRet;
}
````

​		2.消息的响应

````c
void CProxyConn::AddResponsePdu(uint32_t conn_uuid, CImPdu* pPdu)
{
	ResponsePdu_t* pResp = new ResponsePdu_t;//创建一个响应
	pResp->conn_uuid = conn_uuid;
	pResp->pPdu = pPdu;//x协议包
	s_list_lock.lock();
	s_response_pdu_list.push_back(pResp);//添加到CProxyConn对象的回复队列。
	s_list_lock.unlock();
}
````

​		至于s_response_pdu_list队列里面的怎么发送出去，可以查看下面

````c
init_proxy_conn(thread_num);
int init_proxy_conn(uint32_t thread_num)
{
	s_handler_map = CHandlerMap::getInstance();
	g_thread_pool.Init(thread_num);
	netlib_add_loop(proxy_loop_callback, NULL);//重点
	signal(SIGTERM, sig_handler);
	return netlib_register_timer(proxy_timer_callback, NULL, 1000);
}

//
int netlib_add_loop(callback_t callback, void* user_data)
{
	CEventDispatch::Instance()->AddLoop(callback, user_data);
	return 0;
}
//向m_loop_list添加一个item处理项。
void CEventDispatch::AddLoop(callback_t callback, void* user_data)//proxy_loop_callback
{
    TimerItem* pItem = new TimerItem;
    pItem->callback = callback;//Item设置回调函数
    pItem->user_data = user_data;
    m_loop_list.push_back(pItem);//将item添加到m_loop_list
}
//被注册的回调函数
void proxy_loop_callback(void* callback_data, uint8_t msg, uint32_t handle, void* pParam)
{
	CProxyConn::SendResponsePduList();
}
//实际上执行的函数
void CProxyConn::SendResponsePduList()
{
	s_list_lock.lock();
    //一旦队列不为空，就会取出响应队列的数据
	while (!s_response_pdu_list.empty()) {
		ResponsePdu_t* pResp = s_response_pdu_list.front();
		s_response_pdu_list.pop_front();
		s_list_lock.unlock();
		CProxyConn* pConn = get_proxy_conn_by_uuid(pResp->conn_uuid);//获得uuid对应的CProxyConn
		if (pConn) {
			if (pResp->pPdu) {
				pConn->SendPdu(pResp->pPdu);//3.通过CProxyConn发送消息
			} else {
				log("close connection uuid=%d by parse pdu error\b", pResp->conn_uuid);
				pConn->Close();
			}
		}
		if (pResp->pPdu)//删除响应消息
			delete pResp->pPdu;
		delete pResp;//删除响应

		s_list_lock.lock();
	}
	s_list_lock.unlock();
}
````

​		而什么时候发送执行这个回调函数，可以看

````c
//事件分发函数
void CEventDispatch::StartDispatch(uint32_t wait_timeout)
{
    ...
   	_CheckTimer();
	_CheckLoop();
    ...
}
//检查是否有其他事件需要执行
void CEventDispatch::_CheckLoop()
{
    for (list<TimerItem*>::iterator it = m_loop_list.begin(); it != m_loop_list.end(); it++) {
        TimerItem* pItem = *it;//取出item
        pItem->callback(pItem->user_data, NETLIB_MSG_LOOP, 0, NULL);//执行item的回调函数
    }
}
````

##### 消息的发送

​		如果无法发送完，会将数据写到发送缓冲区，并设置m_busy标志，同时添加可写事件监听。（但是它在WIN和APPLE平台都设置的事件监听，唯独Linux平台没有设置监听）。

````c
CProxyConn* pConn = get_proxy_conn_by_uuid(pResp->conn_uuid);
pConn->SendPdu(pResp->pPdu);
//
class CImConn : public CRefObject
{
    ...
    int SendPdu(CImPdu* pPdu) { return Send(pPdu->GetBuffer(), pPdu->GetLength()); }
    ...
}
//send
int CImConn::Send(void* data, int len)
{
	m_last_send_tick = get_tick_count();//记录最后的发送事件
//	++g_send_pkt_cnt;
	if (m_busy)//如果仍让忙碌，则将数据写入发送缓冲区
	{
		m_out_buf.Write(data, len);
		return len;
	}

	int offset = 0;
	int remain = len;
	while (remain > 0) {//只要还剩下就继续发
		int send_size = remain;
		if (send_size > NETLIB_MAX_SOCKET_BUF_SIZE) {//每次最多发送NETLIB_MAX_SOCKET_BUF_SIZE=(128 * 1024) 128k
			send_size = NETLIB_MAX_SOCKET_BUF_SIZE;
		}
		int ret = netlib_send(m_handle, (char*)data + offset , send_size);//发送
		if (ret <= 0) {//出错
			ret = 0;
			break;
		}
		offset += ret;
		remain -= ret;
	}
	if (remain > 0)//检查remain判断是否正常
	{
		m_out_buf.Write((char*)data + offset, remain);//不正常则将数据写入发送缓冲区
		m_busy = true;//置忙碌标记
		log("send busy, remain=%d ", m_out_buf.GetWriteOffset());
	}
    else
    {
        OnWriteCompelete();//执行发送完成函数，虚函数，会被继承覆盖
    }
	return len;
}
//底层的send
int netlib_send(net_handle_t handle, void* buf, int len)
{
	CBaseSocket* pSocket = FindBaseSocket(handle);
	if (!pSocket)
	{
		return NETLIB_ERROR;
	}
	int ret = pSocket->Send(buf, len);
	pSocket->ReleaseRef();
	return ret;
}
int CBaseSocket::Send(void* buf, int len)
{
	if (m_state != SOCKET_STATE_CONNECTED)
		return NETLIB_ERROR;

	int ret = send(m_socket, (char*)buf, len, 0);
	if (ret == SOCKET_ERROR)
	{
		int err_code = _GetErrorCode();
		if (_IsBlock(err_code))//阻塞，说明发送错误
		{
#if ((defined _WIN32) || (defined __APPLE__))
			CEventDispatch::Instance()->AddEvent(m_socket, SOCKET_WRITE);//添加可写事件，为什么linux平台就不添加可写事件?这很奇怪
#endif
			ret = 0;
			//log("socket send block fd=%d", m_socket);
		}
		else
		{
			log("!!!send failed, error code: %d", err_code);
		}
	}
	return ret;
}
//is block
bool CBaseSocket::_IsBlock(int error_code)
{
#ifdef _WIN32
	return ( (error_code == WSAEINPROGRESS) || (error_code == WSAEWOULDBLOCK) );
#else
	return ( (error_code == EINPROGRESS) || (error_code == EWOULDBLOCK) );
#endif
}
````

​		剩下的无法发送的有epoll_wait监听可写事件

````c
void CEventDispatch::StartDispatch(uint32_t wait_timeout)
{
    nfds = epoll_wait(m_epfd, events, 1024, wait_timeout);
    ...
    if (events[i].events & EPOLLOUT)
    {
        //log("OnWrite, socket=%d\n", ev_fd);
        pSocket->OnWrite();
    }
    ...
}
//执行CBaseSocket的write函数
void CBaseSocket::OnWrite()
{
#if ((defined _WIN32) || (defined __APPLE__))
	CEventDispatch::Instance()->RemoveEvent(m_socket, SOCKET_WRITE);
#endif
	if (m_state == SOCKET_STATE_CONNECTING)
	{
		int error = 0;
		socklen_t len = sizeof(error);
#ifdef _WIN32
		getsockopt(m_socket, SOL_SOCKET, SO_ERROR, (char*)&error, &len);
#else
		getsockopt(m_socket, SOL_SOCKET, SO_ERROR, (void*)&error, &len);
#endif
		if (error) {
			m_callback(m_callback_data, NETLIB_MSG_CLOSE, (net_handle_t)m_socket, NULL);
		} else {
			m_state = SOCKET_STATE_CONNECTED;
			m_callback(m_callback_data, NETLIB_MSG_CONFIRM, (net_handle_t)m_socket, NULL);
		}
	}
	else
	{
		m_callback(m_callback_data, NETLIB_MSG_WRITE, (net_handle_t)m_socket, NULL);
	}
}

//这里在客户端连接的时候就已经将回调函数重定位了
void CProxyConn::OnConnect(net_handle_t handle)//fd
{
    ...
	netlib_option(handle, NETLIB_OPT_SET_CALLBACK, (void*)imconn_callback);//修改回调函数为imconn_callback
    ...
}
//可以参考这里的函数
void imconn_callback(void* callback_data, uint8_t msg, uint32_t handle, void* pParam)
{
    ...
	ConnMap_t* conn_map = (ConnMap_t*)callback_data;//map<fd,proxy_connection>
	CImConn* pConn = FindImConn(conn_map, handle);
    ...
	switch (msg)
	{
	...
	case NETLIB_MSG_WRITE:
		pConn->OnWrite();
		break;
	...
	}
	...
}

//因此实际还是执行了
void CImConn::OnWrite()
{
	if (!m_busy)//如果不忙碌了，直接返回，这里只有无法发送出去的时候才会重新发送
		return;
	while (m_out_buf.GetWriteOffset() > 0) {//
		int send_size = m_out_buf.GetWriteOffset();
		if (send_size > NETLIB_MAX_SOCKET_BUF_SIZE) {//NETLIB_MAX_SOCKET_BUF_SIZE=(128 * 1024)
			send_size = NETLIB_MAX_SOCKET_BUF_SIZE;
		}
		int ret = netlib_send(m_handle, m_out_buf.GetBuffer(), send_size);//发送，如果还是无法发送，继续注册监听发送事件
		if (ret <= 0) {
			ret = 0;
			break;
		}
		m_out_buf.Read(NULL, ret);//发送成功，清除掉已发送的数据
	}
	if (m_out_buf.GetWriteOffset() == 0) {//发送缓冲区已清理，清除忙碌标志
		m_busy = false;
	}
	log("onWrite, remain=%d ", m_out_buf.GetWriteOffset());
}
//////////////////////////////////////////
// 因此，对于发送流程，可以总结
// 1.将响应数据丢到其他任务队列中。
// 2.epoll_执行完毕后，处理其他任务队列,执行其他任务队列的回调函数
// 3.找到响应对应的CProxyConn，执行数据发送
// 4.先一次性发送数据，若无法发出错，则将数据写到发送缓冲区，并注册监听可写事件
// 5.epoll监听到可写事件，则将发送缓冲区的数据继续发送出去，如果还是无法发送全部，则继续注册可写事件....直到全部数据发送出去。
// 6.数据全部发送出去，取消可写事件监听，避免无数据可写也触发可写事件(实际上很频繁我的)！
````

##### 心跳包的处理

````c
int init_proxy_conn(uint32_t thread_num)
{
	...
	return netlib_register_timer(proxy_timer_callback, NULL, 1000);
}
//注册定时器回调函数proxy_timer_callback
int netlib_register_timer(callback_t callback, void* user_data, uint64_t interval)
{
	CEventDispatch::Instance()->AddTimer(callback, user_data, interval);
	return 0;
}
//
void CEventDispatch::AddTimer(callback_t callback, void* user_data, uint64_t interval)
{
	list<TimerItem*>::iterator it;
	for (it = m_timer_list.begin(); it != m_timer_list.end(); it++)//遍历所有定时器，若已经存在这样的定时器，则更新定时器
	{
		TimerItem* pItem = *it;
		if (pItem->callback == callback && pItem->user_data == user_data)
		{
			pItem->interval = interval;
			pItem->next_tick = get_tick_count() + interval;
			return;
		}
	}
	TimerItem* pItem = new TimerItem;//创建定时器
	pItem->callback = callback;//设置回调函数
	pItem->user_data = user_data;
	pItem->interval = interval;
	pItem->next_tick = get_tick_count() + interval;
	m_timer_list.push_back(pItem);//添加到定时器队列
}
````

​		在IO事件监听事件处理循环里

````c
void CEventDispatch::StartDispatch(uint32_t wait_timeout)
{    
	while (running)
	{
		nfds = epoll_wait(m_epfd, events, 1024, wait_timeout);
		for (int i = 0; i < nfds; i++)
		{
			...
		}
		_CheckTimer();//检查定时器事件
		...
	}
}
//定时器检查
void CEventDispatch::_CheckTimer()
{
	uint64_t curr_tick = get_tick_count();
	list<TimerItem*>::iterator it;
	//遍历定时器队列
	for (it = m_timer_list.begin(); it != m_timer_list.end(); )
	{
		TimerItem* pItem = *it;
		it++;		// iterator maybe deleted in the callback, so we should increment it before callback
		if (curr_tick >= pItem->next_tick)//定时时间到达
		{
			pItem->next_tick += pItem->interval;//更新时间
			pItem->callback(pItem->user_data, NETLIB_MSG_TIMER, 0, NULL);//执行定时器函数，proxy_timer_callback
		}
	}
}
//定时器函数
void proxy_timer_callback(void* callback_data, uint8_t msg, uint32_t handle, void* pParam)
{
	uint64_t cur_time = get_tick_count();
	for (ConnMap_t::iterator it = g_proxy_conn_map.begin(); it != g_proxy_conn_map.end(); ) {//遍历map
		ConnMap_t::iterator it_old = it;
		it++;
		CProxyConn* pConn = (CProxyConn*)it_old->second;//获取CProxyConn
		pConn->OnTimer(cur_time);//执行对应连接的定时函数
	}
}
/////////
void CProxyConn::OnTimer(uint64_t curr_tick)
{
    //对比上一次服务器发数据的时间，若超过心跳间隔，则主动发送一帧心跳包 , SERVER_HEARTBEAT_INTERVAL=5000
	if (curr_tick > m_last_send_tick + SERVER_HEARTBEAT_INTERVAL) {// m_last_send_tick 上一次发送数据的时间
        CImPdu cPdu;
        IM::Other::IMHeartBeat msg;
        cPdu.SetPBMsg(&msg);
        cPdu.SetServiceId(IM::BaseDefine::SID_OTHER);
        cPdu.SetCommandId(IM::BaseDefine::CID_OTHER_HEARTBEAT);//设置为心跳包
		SendPdu(&cPdu);//发送心跳包
	}

    //对比上一次收到数据的时间，如超过指定间隔，则认为客户端已离线，关闭这个连接
	if (curr_tick > m_last_recv_tick + SERVER_TIMEOUT) {// m_last_recv_tick 上一次收到数据的时间
		log("proxy connection timeout %s:%d", m_peer_ip.c_str(), m_peer_port);
		Close();
	}
}

///////////////////////////////////////////////////
// 我挺喜欢这种心跳包的处理方式。
// 1.服务器定时向客户端发一次心跳包，若有数据包发送，则心跳包推迟下一个时间发送，这样可以充分利用服务器的带宽，而不是使用TCP自带的心跳包的方式，这样会很大占用带宽，但是实际上都是发送一些无意义的数据。
// 2.服务器在接收客户端数据时，会更新发送时间，若超过一段时间，客户端没有发来数据，则任务客户端已掉线。
/////////////////////////////////////////////////
````





#### 2.Msg_server 消息服务器

​		TeamTalk各个服务的功能介绍。TeamTalk是一个分布式部署的聊天服务器，通过分布式部署，可以实现分流以及支持高数量的用户同时在线。MsgServer是整个系统的核心。不同的用户根据各个消息服务器的负载，选择一个服务器连接。RouteServer实现一个MsgServer用户消息转发给其他MsgServer用户，可以拓展。

<img src="TeamTalk框架.png" style="zoom:80%;" />

````c
LoginServer(C++)	: //负载均衡服务器，会分配一个负载小的MsgServer给客户端使用
MsgServer(C++)		: //消息服务器，提供客户端的信令功能，提供群聊，私聊等功能。
RouteServer(C++)	: //路由服务器，提供在不同MsgServer之间消息的转发。
FileServer(C++)		: //文件服务器，提供客户端之间的文件传输服务，包括离线和在线文件。
MsfsServer(C++)		: //图片存储服务器，提供图片存储和头像等服务。
DBProxyServer(C++)	: //数据库代理服务器，提供redis和MySQL等数据库的访问，屏蔽其他服务器和redis与MySQL的访问。
HttpMsgServer(C++)	: //对外接口，只是一个框架
PushServer(C++)		: //消息推送服务器，提供IOS系统消息推送(IOS消息推送必须走apns)
````



​		MsgServer的连接流程：

````c
int maint(int argc char*[] argv)
{
    //读取配置文件系统
    //初始化当前服务器
    //连接其他的服务器：FileServer,DBProxyServer,loginServer,RouteServer,PushServer.
    //启动事件监听循环
}
````



##### 2.1 配置文件

````c
CConfigFileReader config_file("msgserver.conf");//读取msgserver.conf
char* listen_ip = config_file.GetConfigName("ListenIP");// ListenIP=0.0.0.0
char* str_listen_port = config_file.GetConfigName("ListenPort");//ListenPort=8000
char* ip_addr1 = config_file.GetConfigName("IpAddr1");	// 电信IP
char* ip_addr2 = config_file.GetConfigName("IpAddr2");	// 网通IP
char* str_max_conn_cnt = config_file.GetConfigName("MaxConnCnt");//MaxConnCnt=100000
char* str_aes_key = config_file.GetConfigName("aesKey");
uint32_t db_server_count = 0;
serv_info_t* db_server_list = read_server_config(&config_file, "DBServerIP", "DBServerPort", db_server_count);//DBServerIP1=127.0.0.1,DBServerPort1=10600 ; DBServerIP2=127.0.0.1,BServerPort2=10600
uint32_t login_server_count = 0;
serv_info_t* login_server_list = read_server_config(&config_file, "LoginServerIP", "LoginServerPort", login_server_count);
uint32_t route_server_count = 0;
serv_info_t* route_server_list = read_server_config(&config_file, "RouteServerIP", "RouteServerPort", route_server_count);
uint32_t push_server_count = 0;
serv_info_t* push_server_list = read_server_config(&config_file, "PushServerIP","PushServerPort", push_server_count);
uint32_t file_server_count = 0;
serv_info_t* file_server_list = read_server_config(&config_file, "FileServerIP","FileServerPort", file_server_count);
````



##### 2.2 MsgServer服务器初始化

````c
uint16_t listen_port = atoi(str_listen_port);//8000
uint32_t max_conn_cnt = atoi(str_max_conn_cnt);//100000

int ret = netlib_init();

if (ret == NETLIB_ERROR)
    return ret;

int main()
{
    ...
    //在8000端口号上侦听客户端连接
    CStrExplode listen_ip_list(listen_ip, ';');
    for (uint32_t i = 0; i < listen_ip_list.GetItemCnt(); i++)
    {
        ret = netlib_listen(listen_ip_list.GetItem(i), listen_port, msg_serv_callback, NULL);
        if (ret == NETLIB_ERROR)
            return ret;
    }
    init_msg_conn();//1
    ...
}

int netlib_listen(const char*	server_ip, uint16_t	port,callback_t	callback,void*		callback_data)
{
	CBaseSocket* pSocket = new CBaseSocket();
	if (!pSocket)
		return NETLIB_ERROR;
	int ret =  pSocket->Listen(server_ip, port, callback, callback_data);
	if (ret == NETLIB_ERROR)
		delete pSocket;
	return ret;
}

//listen
int CBaseSocket::Listen(const char* server_ip, uint16_t port, callback_t callback, void* callback_data)
{
	m_local_ip = server_ip;
	m_local_port = port;
	m_callback = callback;
	m_callback_data = callback_data;
	m_socket = socket(AF_INET, SOCK_STREAM, 0);//tcp
	if (m_socket == INVALID_SOCKET)
	{
		printf("socket failed, err_code=%d\n", _GetErrorCode());
		return NETLIB_ERROR;
	}
	_SetReuseAddr(m_socket);//reuse
	_SetNonblock(m_socket);//non block
	sockaddr_in serv_addr;
	_SetAddr(server_ip, port, &serv_addr);
    int ret = ::bind(m_socket, (sockaddr*)&serv_addr, sizeof(serv_addr));//bind
	if (ret == SOCKET_ERROR)
	{
		log("bind failed, err_code=%d", _GetErrorCode());
		closesocket(m_socket);
		return NETLIB_ERROR;
	}
	ret = listen(m_socket, 64);//listen
	if (ret == SOCKET_ERROR)
	{
		log("listen failed, err_code=%d", _GetErrorCode());
		closesocket(m_socket);
		return NETLIB_ERROR;
	}
	m_state = SOCKET_STATE_LISTENING;
	log("CBaseSocket::Listen on %s:%d", server_ip, port);
	AddBaseSocket(this);
	CEventDispatch::Instance()->AddEvent(m_socket, SOCKET_READ | SOCKET_EXCEP);//添加事件监听
	return NETLIB_OK;
}

````

````c
void init_msg_conn()
{
	g_last_stat_tick = get_tick_count();
	signal(SIGUSR1, signal_handler_usr1);
	signal(SIGUSR2, signal_handler_usr2);
	signal(SIGHUP, signal_handler_hup);
	netlib_register_timer(msg_conn_timer_callback, NULL, 1000);//注册定时器，这里是和DBProxyServe一样，是注册定时器来进行心跳包检测和发送
	s_file_handler = CFileHandler::getInstance();
	s_group_chat = CGroupChat::GetInstance();
}

int netlib_register_timer(callback_t callback, void* user_data, uint64_t interval)
{
	CEventDispatch::Instance()->AddTimer(callback, user_data, interval);
	return 0;
}
````





##### 2.3 连接其他服务器

````c
init_file_serv_conn(file_server_list, file_server_count);//连接文件服务器
init_db_serv_conn(db_server_list2, db_server_count2, concurrent_db_conn_cnt);//连接数据库代理服务器
init_login_serv_conn(login_server_list, login_server_count, ip_addr1, ip_addr2, listen_port, max_conn_cnt);//连接登录服务器
init_route_serv_conn(route_server_list, route_server_count);//连接路由服务器
init_push_serv_conn(push_server_list, push_server_count);//连接消息推送服务器
printf("now enter the event loop...\n");
````

​		连接流程都一样，以文件服务器为例

````c
void init_file_serv_conn(serv_info_t* server_list, uint32_t server_count)
{
	g_file_server_list = server_list;//g_file_server_list文件服务器队列
	g_file_server_count = server_count;
	serv_init<CFileServConn>(g_file_server_list, g_file_server_count);//1.初始化和文件服务器的连接，并将文件服务器连接添加到队列
	netlib_register_timer(file_server_conn_timer_callback, NULL, 1000);//2.注册定时器
	s_file_handler = CFileHandler::getInstance();
}
//1.
template <class T>
void serv_init(serv_info_t* server_list, uint32_t server_count)//连接服务器
{
	for (uint32_t i = 0; i < server_count; i++) {
		T* pConn = new T();//创建一个连接对象
		pConn->Connect(server_list[i].server_ip.c_str(), server_list[i].server_port, i);
		server_list[i].serv_conn = pConn;
		server_list[i].idle_cnt = 0;
		server_list[i].reconnect_cnt = MIN_RECONNECT_CNT / 2;//MIN_RECONNECT_CNT=4
	}
}
//连接服务器
void CFileServConn::Connect(const char* server_ip, uint16_t server_port, uint32_t idx)
{
	log("Connecting to FileServer %s:%d ", server_ip, server_port);
	m_serv_idx = idx;
	m_handle = netlib_connect(server_ip, server_port, imconn_callback, (void*)&g_file_server_conn_map);//连接服务器，并设置回调函数
	if (m_handle != NETLIB_INVALID_HANDLE) {
		g_file_server_conn_map.insert(make_pair(m_handle, this));//添加到文件服务器map
	}
}
net_handle_t netlib_connect(const char* server_ip, uint16_t	port, callback_t	callback, void*		callback_data)
{
	CBaseSocket* pSocket = new CBaseSocket();
	if (!pSocket)
		return NETLIB_INVALID_HANDLE;
	net_handle_t handle = pSocket->Connect(server_ip, port, callback, callback_data);
	if (handle == NETLIB_INVALID_HANDLE)
		delete pSocket;
	return handle;
}
net_handle_t CBaseSocket::Connect(const char* server_ip, uint16_t port, callback_t callback, void* callback_data)
{
	log("CBaseSocket::Connect, server_ip=%s, port=%d", server_ip, port);

	m_remote_ip = server_ip;
	m_remote_port = port;
	m_callback = callback;//设置回调函数
	m_callback_data = callback_data;
	m_socket = socket(AF_INET, SOCK_STREAM, 0);//创建一个client句柄
	if (m_socket == INVALID_SOCKET)
	{
		log("socket failed, err_code=%d", _GetErrorCode());
		return NETLIB_INVALID_HANDLE;
	}
	_SetNonblock(m_socket);//非阻塞
	_SetNoDelay(m_socket);//禁用Nagle算法
	sockaddr_in serv_addr;
	_SetAddr(server_ip, port, &serv_addr);
	int ret = connect(m_socket, (sockaddr*)&serv_addr, sizeof(serv_addr));//连接服务器
	if ( (ret == SOCKET_ERROR) && (!_IsBlock(_GetErrorCode())) )
	{	
		log("connect failed, err_code=%d", _GetErrorCode());
		closesocket(m_socket);
		return NETLIB_INVALID_HANDLE;
	}
	m_state = SOCKET_STATE_CONNECTING;
	AddBaseSocket(this);
	CEventDispatch::Instance()->AddEvent(m_socket, SOCKET_ALL);//监听所有事件
	return (net_handle_t)m_socket;
}

//在事件监听中立即出发可写（服务器没那么快发来数据，因此可读还无法触发）
void CEventDispatch::StartDispatch(uint32_t wait_timeout)
{	
    if (events[i].events & EPOLLOUT)
    {
        //log("OnWrite, socket=%d\n", ev_fd);
        pSocket->OnWrite();
    }
}
//执行可写
void CBaseSocket::OnWrite()
{
#if ((defined _WIN32) || (defined __APPLE__))
	CEventDispatch::Instance()->RemoveEvent(m_socket, SOCKET_WRITE);
#endif

	if (m_state == SOCKET_STATE_CONNECTING)
	{
		int error = 0;
		socklen_t len = sizeof(error);
#ifdef _WIN32
		getsockopt(m_socket, SOL_SOCKET, SO_ERROR, (char*)&error, &len);
#else
		getsockopt(m_socket, SOL_SOCKET, SO_ERROR, (void*)&error, &len);
#endif
		if (error) {
			m_callback(m_callback_data, NETLIB_MSG_CLOSE, (net_handle_t)m_socket, NULL);
		} else {
			m_state = SOCKET_STATE_CONNECTED;//修改为已连接状态
			m_callback(m_callback_data, NETLIB_MSG_CONFIRM, (net_handle_t)m_socket, NULL);//执行回调函数
		}
	}
	else
	{
		m_callback(m_callback_data, NETLIB_MSG_WRITE, (net_handle_t)m_socket, NULL);
	}
}

//回调函数
void imconn_callback(void* callback_data, uint8_t msg, uint32_t handle, void* pParam)//设置回调函数
{
    ...
	ConnMap_t* conn_map = (ConnMap_t*)callback_data;
	CImConn* pConn = FindImConn(conn_map, handle);
	...
	switch (msg)
	{
	case NETLIB_MSG_CONFIRM://NETLIB_MSG_CONFIRM
		pConn->OnConfirm();
		break;
	...
}

//发布连接确认信息
void CFileServConn::OnConfirm()
{
	log("connect to file server success ");
	m_bOpen = true;
	m_connect_time = get_tick_count();
	g_file_server_list[m_serv_idx].reconnect_cnt = MIN_RECONNECT_CNT / 2;
    
    //连上file_server以后，给file_server发送获取ip地址的数据包
    IM::Server::IMFileServerIPReq msg;
    CImPdu pdu;
    pdu.SetPBMsg(&msg);
    pdu.SetServiceId(SID_OTHER);
    pdu.SetCommandId(CID_OTHER_FILE_SERVER_IP_REQ);
    SendPdu(&pdu);
}
    
///////////////////////////////////////////////////////////
// 上面的这些步骤，实现的功能是在连接FileServer之后，发起获取文件服务器信息的请求。

//2.注册定时器
int netlib_register_timer(callback_t callback, void* user_data, uint64_t interval)
{
	CEventDispatch::Instance()->AddTimer(callback, user_data, interval);
	return 0;
}
//文件服务器定时器回调函数
void file_server_conn_timer_callback(void* callback_data, uint8_t msg, uint32_t handle, void* pParam)
{
	ConnMap_t::iterator it_old;
	CFileServConn* pConn = NULL;
	uint64_t cur_time = get_tick_count();
	for (ConnMap_t::iterator it = g_file_server_conn_map.begin(); it != g_file_server_conn_map.end();)
    {
        it_old = it;
        it++;
		pConn = (CFileServConn*)it_old->second;
		pConn->OnTimer(cur_time);//发送心跳包，若超过一定时间没有回复，则会将连接关闭，并踢出g_file_server_conn_map
	}
	// reconnect FileServer
	serv_check_reconnect<CFileServConn>(g_file_server_list, g_file_server_count);
}

//检测是否挂了。若挂了，重新连接
template <class T>
void serv_check_reconnect(serv_info_t* server_list, uint32_t server_count)
{
	T* pConn;
	for (uint32_t i = 0; i < server_count; i++) {
		pConn = (T*)server_list[i].serv_conn;
		if (!pConn) {
			server_list[i].idle_cnt++;
			if (server_list[i].idle_cnt >= server_list[i].reconnect_cnt) {
				pConn = new T();
				pConn->Connect(server_list[i].server_ip.c_str(), server_list[i].server_port, i);
				server_list[i].serv_conn = pConn;
			}
		}
	}
}

````

​		关于LoginServer的一些特殊情况

````c
// 连接LoginServer之后，会告诉服务器当前MsgServer的ip地址，端口，已登录的用户数量，最大容纳的用户数量。
void CLoginServConn::OnConfirm()
{
	log("connect to login server success ");
	m_bOpen = true;
	g_login_server_list[m_serv_idx].reconnect_cnt = MIN_RECONNECT_CNT / 2;

	uint32_t cur_conn_cnt = 0;
	uint32_t shop_user_cnt = 0;
    
    //连接login_server成功以后,告诉login_server自己的ip地址、端口号
    //和当前登录的用户数量和可容纳的最大用户数量
    list<user_conn_t> user_conn_list;
    CImUserManager::GetInstance()->GetUserConnCnt(&user_conn_list, cur_conn_cnt);
	char hostname[256] = {0};
	gethostname(hostname, 256);
    IM::Server::IMMsgServInfo msg;
    msg.set_ip1(g_msg_server_ip_addr1);
    msg.set_ip2(g_msg_server_ip_addr2);
    msg.set_port(g_msg_server_port);
    msg.set_max_conn_cnt(g_max_conn_cnt);
    msg.set_cur_conn_cnt(cur_conn_cnt);
    msg.set_host_name(hostname);
    CImPdu pdu;
    pdu.SetPBMsg(&msg);
    pdu.SetServiceId(SID_OTHER);
    pdu.SetCommandId(CID_OTHER_MSG_SERV_INFO);
	SendPdu(&pdu);
}
````



##### 2.4  IO事件循环

````c
// IO事件的循环和DBProxyServer类似，下面是FileServe对于消息协议包的处理。
void CFileServConn::HandlePdu(CImPdu* pPdu)
{
	switch (pPdu->GetCommandId()) {
        case CID_OTHER_HEARTBEAT:
            break;
        case CID_OTHER_FILE_TRANSFER_RSP:
            _HandleFileMsgTransRsp(pPdu);
            break;
        case CID_OTHER_FILE_SERVER_IP_RSP:
            _HandleFileServerIPRsp(pPdu);
            break;
        default:
            log("unknown cmd id=%d ", pPdu->GetCommandId());
            break;
	}
}
````





#### 3.LoginServer 登录分流服务器

​		登录服务器最准确的名称应该是登录分流服务器，它连接所有的消息服务器，在接收客户端的连接请求之后，选择负载最小的消息服务器MsgServer给客户端。

````c
int main()
{
    //读取配置文件
    //在8008端口监听客户端的连接
    //在8100端口监听MsgServer的连接
    //在8080端口监听客户端http连接
    //初始化login connection
    //初始化http connection
    //进行事件循环
}
````

##### 3.1 配置文件

````c
	CConfigFileReader config_file("loginserver.conf");//读取loginserver.conf
    char* client_listen_ip = config_file.GetConfigName("ClientListenIP");//0.0.0.0
    char* str_client_port = config_file.GetConfigName("ClientPort");//8008
    char* http_listen_ip = config_file.GetConfigName("HttpListenIP");//0.0.0.0
    char* str_http_port = config_file.GetConfigName("HttpPort");//8080
	char* msg_server_listen_ip = config_file.GetConfigName("MsgServerListenIP");//0.0.0.0
	char* str_msg_server_port = config_file.GetConfigName("MsgServerPort");//8100
    char* str_msfs_url = config_file.GetConfigName("msfs");
    char* str_discovery = config_file.GetConfigName("discovery");
	if (!msg_server_listen_ip || !str_msg_server_port || !http_listen_ip
        || !str_http_port || !str_msfs_url || !str_discovery) {
		log("config item missing, exit... ");
		return -1;
	}
	uint16_t client_port = atoi(str_client_port);
	uint16_t msg_server_port = atoi(str_msg_server_port);
    uint16_t http_port = atoi(str_http_port);
````

##### 3.2 监听客户端连接 && msg_server的连接

````c++
//在8080端口监听客户端连接
char* str_client_port = config_file.GetConfigName("ClientPort");//8008
uint16_t client_port = atoi(str_client_port);
CStrExplode client_listen_ip_list(client_listen_ip, ';');
for (uint32_t i = 0; i < client_listen_ip_list.GetItemCnt(); i++)
{
    ret = netlib_listen(client_listen_ip_list.GetItem(i), client_port, client_callback, NULL);
    if (ret == NETLIB_ERROR)
        return ret;
}
int netlib_listen(const char* server_ip, uint16_t port,callback_t callback,void* callback_data)
{
	CBaseSocket* pSocket = new CBaseSocket();
	if (!pSocket)
		return NETLIB_ERROR;
	int ret =  pSocket->Listen(server_ip, port, callback, callback_data);
	if (ret == NETLIB_ERROR)
		delete pSocket;
	return ret;
}

//这个函数已经分析过很多次了
int CBaseSocket::Listen(const char* server_ip, uint16_t port, callback_t callback, void* callback_data)
{
	m_local_ip = server_ip;
	m_local_port = port;
	m_callback = callback;
	m_callback_data = callback_data;
	m_socket = socket(AF_INET, SOCK_STREAM, 0);
	if (m_socket == INVALID_SOCKET)
	{
		printf("socket failed, err_code=%d\n", _GetErrorCode());
		return NETLIB_ERROR;
	}
	_SetReuseAddr(m_socket);
	_SetNonblock(m_socket);
	sockaddr_in serv_addr;
	_SetAddr(server_ip, port, &serv_addr);
    int ret = ::bind(m_socket, (sockaddr*)&serv_addr, sizeof(serv_addr));
	if (ret == SOCKET_ERROR)
	{
		log("bind failed, err_code=%d", _GetErrorCode());
		closesocket(m_socket);
		return NETLIB_ERROR;
	}
	ret = listen(m_socket, 64);
	if (ret == SOCKET_ERROR)
	{
		log("listen failed, err_code=%d", _GetErrorCode());
		closesocket(m_socket);
		return NETLIB_ERROR;
	}
	m_state = SOCKET_STATE_LISTENING;
	log("CBaseSocket::Listen on %s:%d", server_ip, port);
	AddBaseSocket(this);
	CEventDispatch::Instance()->AddEvent(m_socket, SOCKET_READ | SOCKET_EXCEP);//监听可读事件
	return NETLIB_OK;
}

//客户端会发送数据，socket产生可读事件，会执行回调函数
void client_callback(void* callback_data, uint8_t msg, uint32_t handle, void* pParam)
{
	if (msg == NETLIB_MSG_CONNECT)
	{
		CLoginConn* pConn = new CLoginConn();//创建登录连接
		pConn->OnConnect2(handle, LOGIN_CONN_TYPE_CLIENT);
	}
	else
	{
		log("!!!error msg: %d ", msg);
	}
}
//OnConnect2
void CLoginConn::OnConnect2(net_handle_t handle, int conn_type)
{
	m_handle = handle;
	m_conn_type = conn_type;
	ConnMap_t* conn_map = &g_msg_serv_conn_map;
	if (conn_type == LOGIN_CONN_TYPE_CLIENT) {//客户端登录，这是map为g_client_conn_map
		conn_map = &g_client_conn_map;
	}else
		conn_map->insert(make_pair(handle, this));

	netlib_option(handle, NETLIB_OPT_SET_CALLBACK, (void*)imconn_callback);//设置客户端服务器连接的回调函数
	netlib_option(handle, NETLIB_OPT_SET_CALLBACK_DATA, (void*)conn_map);
}

//实际上执行的回调函数，之前Accept又注册了可读事件监听，会触发NETLIB_MSG_READ，由于CImConn的子类对于Read都没有覆盖，因此对协议包读取的方式是一样的，但是他们对于包的处理是有覆盖的。
void imconn_callback(void* callback_data, uint8_t msg, uint32_t handle, void* pParam)
{
    ...
	ConnMap_t* conn_map = (ConnMap_t*)callback_data;
	CImConn* pConn = FindImConn(conn_map, handle);
	switch (msg)
	{
	case NETLIB_MSG_CONFIRM:
		pConn->OnConfirm();
		break;
	case NETLIB_MSG_READ:
		pConn->OnRead();
		break;
	case NETLIB_MSG_WRITE:
		pConn->OnWrite();
		break;
	case NETLIB_MSG_CLOSE:
		pConn->OnClose();
		break;
	default:
		log("!!!imconn_callback error msg: %d ", msg);
		break;
	}
	pConn->ReleaseRef();
}
//处理协议包
void CLoginConn::HandlePdu(CImPdu* pPdu)
{
	switch (pPdu->GetCommandId()) {
        case CID_OTHER_HEARTBEAT://心跳命令
            break;
        case CID_OTHER_MSG_SERV_INFO:
            _HandleMsgServInfo(pPdu);//处理服务器信息
            break;
        case CID_OTHER_USER_CNT_UPDATE:
            _HandleUserCntUpdate(pPdu);//用户上下线数据更新
            break;
        case CID_LOGIN_REQ_MSGSERVER:
            _HandleMsgServRequest(pPdu);//回复客户端服务器信息
            break;
        default:
            log("wrong msg, cmd id=%d ", pPdu->GetCommandId());
            break;
	}
}

//处理服务器信息
void CLoginConn::_HandleMsgServInfo(CImPdu* pPdu)
{
	msg_serv_info_t* pMsgServInfo = new msg_serv_info_t;
    IM::Server::IMMsgServInfo msg;//服务器信息
    msg.ParseFromArray(pPdu->GetBodyData(), pPdu->GetBodyLength());
    
	pMsgServInfo->ip_addr1 = msg.ip1();//记录服务器信息
	pMsgServInfo->ip_addr2 = msg.ip2();
	pMsgServInfo->port = msg.port();
	pMsgServInfo->max_conn_cnt = msg.max_conn_cnt();
	pMsgServInfo->cur_conn_cnt = msg.cur_conn_cnt();
	pMsgServInfo->hostname = msg.host_name();
	g_msg_serv_info.insert(make_pair(m_handle, pMsgServInfo));//记录在g_msg_serv_info
	g_total_online_user_cnt += pMsgServInfo->cur_conn_cnt;
	log("MsgServInfo, ip_addr1=%s, ip_addr2=%s, port=%d, max_conn_cnt=%d, cur_conn_cnt=%d, hostname: %s. ",
		pMsgServInfo->ip_addr1.c_str(), pMsgServInfo->ip_addr2.c_str(), pMsgServInfo->port,pMsgServInfo->max_conn_cnt,
		pMsgServInfo->cur_conn_cnt, pMsgServInfo->hostname.c_str());
}
map<uint32_t, msg_serv_info_t*> g_msg_serv_info;
typedef struct  {
    string		ip_addr1;	// 电信IP
    string		ip_addr2;	// 网通IP
    uint16_t	port;
    uint32_t	max_conn_cnt;
    uint32_t	cur_conn_cnt;
    string 		hostname;	// 消息服务器的主机名
} msg_serv_info_t;

//用户上下线信息更新
void CLoginConn::_HandleUserCntUpdate(CImPdu* pPdu)
{
	map<uint32_t, msg_serv_info_t*>::iterator it = g_msg_serv_info.find(m_handle);
	if (it != g_msg_serv_info.end()) {
		msg_serv_info_t* pMsgServInfo = it->second;
        IM::Server::IMUserCntUpdate msg;
        msg.ParseFromArray(pPdu->GetBodyData(), pPdu->GetBodyLength());
		uint32_t action = msg.user_action();
		if (action == USER_CNT_INC) {//上限
			pMsgServInfo->cur_conn_cnt++;
			g_total_online_user_cnt++;
		} else {//下线
			pMsgServInfo->cur_conn_cnt--;
			g_total_online_user_cnt--;
		}
		log("%s:%d, cur_cnt=%u, total_cnt=%u ", pMsgServInfo->hostname.c_str(),
            pMsgServInfo->port, pMsgServInfo->cur_conn_cnt, g_total_online_user_cnt);
	}
}

//回复客户端关于msg_server的信息
void CLoginConn::_HandleMsgServRequest(CImPdu* pPdu)
{
    IM::Login::IMMsgServReq msg;
    msg.ParseFromArray(pPdu->GetBodyData(), pPdu->GetBodyLength());
	log("HandleMsgServReq. ");
	// no MessageServer available
	if (g_msg_serv_info.size() == 0) {//暂时没有msg_server服务器连接上
        IM::Login::IMMsgServRsp msg;
        msg.set_result_code(::IM::BaseDefine::REFUSE_REASON_NO_MSG_SERVER);
        CImPdu pdu;
        pdu.SetPBMsg(&msg);
        pdu.SetServiceId(SID_LOGIN);
        pdu.SetCommandId(CID_LOGIN_RES_MSGSERVER);
        pdu.SetSeqNum(pPdu->GetSeqNum());
        SendPdu(&pdu);
        Close();
		return;
	}
	// return a message server with minimum concurrent connection count
	msg_serv_info_t* pMsgServInfo;
	uint32_t min_user_cnt = (uint32_t)-1;
	map<uint32_t, msg_serv_info_t*>::iterator it_min_conn = g_msg_serv_info.end(),it;
	//分流
	for (it = g_msg_serv_info.begin() ; it != g_msg_serv_info.end(); it++) {
		pMsgServInfo = it->second;
		if ( (pMsgServInfo->cur_conn_cnt < pMsgServInfo->max_conn_cnt) &&
			 (pMsgServInfo->cur_conn_cnt < min_user_cnt))
        {
			it_min_conn = it;
			min_user_cnt = pMsgServInfo->cur_conn_cnt;
		}
	}
    //全部服务器都已经满载
	if (it_min_conn == g_msg_serv_info.end()) {
		log("All TCP MsgServer are full ");
        IM::Login::IMMsgServRsp msg;
        msg.set_result_code(::IM::BaseDefine::REFUSE_REASON_MSG_SERVER_FULL);
        CImPdu pdu;
        pdu.SetPBMsg(&msg);
        pdu.SetServiceId(SID_LOGIN);
        pdu.SetCommandId(CID_LOGIN_RES_MSGSERVER);
        pdu.SetSeqNum(pPdu->GetSeqNum());
        SendPdu(&pdu);
	}
    else
    {//负载找到负载低的服务器，返回服务器连接。
        IM::Login::IMMsgServRsp msg;
        msg.set_result_code(::IM::BaseDefine::REFUSE_REASON_NONE);
        msg.set_prior_ip(it_min_conn->second->ip_addr1);
        msg.set_backip_ip(it_min_conn->second->ip_addr2);
        msg.set_port(it_min_conn->second->port);
        CImPdu pdu;
        pdu.SetPBMsg(&msg);
        pdu.SetServiceId(SID_LOGIN);
        pdu.SetCommandId(CID_LOGIN_RES_MSGSERVER);
        pdu.SetSeqNum(pPdu->GetSeqNum());
        SendPdu(&pdu);
    }
    //找到负载低的服务器，关闭当前连接
	Close();	// after send MsgServResponse, active close the connection
}
````

 		msgserver和上面的类似

````c
//在8100上监听msg_server的连接
CStrExplode msg_server_listen_ip_list(msg_server_listen_ip, ';');
uint16_t msg_server_port = atoi(str_msg_server_port);
for (uint32_t i = 0; i < msg_server_listen_ip_list.GetItemCnt(); i++)
{
    ret = netlib_listen(msg_server_listen_ip_list.GetItem(i), msg_server_port, msg_serv_callback, NULL);//
    if (ret == NETLIB_ERROR)
        return ret;
}
// this callback will be replaced by imconn_callback() in OnConnect()
void msg_serv_callback(void* callback_data, uint8_t msg, uint32_t handle, void* pParam)
{
    log("msg_server come in");
	if (msg == NETLIB_MSG_CONNECT)
	{
		CLoginConn* pConn = new CLoginConn();
		pConn->OnConnect2(handle, LOGIN_CONN_TYPE_MSG_SERV);
	}
	else
	{
		log("!!!error msg: %d ", msg);
	}
}
````

##### 3.3 监听http连接

````c++
//在8080上监听客户端http连接
char* str_http_port = config_file.GetConfigName("HttpPort");
uint16_t http_port = atoi(str_http_port);
CStrExplode http_listen_ip_list(http_listen_ip, ';');
for (uint32_t i = 0; i < http_listen_ip_list.GetItemCnt(); i++)
{
    ret = netlib_listen(http_listen_ip_list.GetItem(i), http_port, http_callback, NULL);
    if (ret == NETLIB_ERROR)
        return ret;
}
//连接上回调函数后
void http_callback(void* callback_data, uint8_t msg, uint32_t handle, void* pParam)
{
    if (msg == NETLIB_MSG_CONNECT)
    {
        CHttpConn* pConn = new CHttpConn();
        pConn->OnConnect(handle);
    }
    else
    {
        log("!!!error msg: %d ", msg);
    }
}

void CHttpConn::OnConnect(net_handle_t handle)
{
    printf("OnConnect, handle=%d\n", handle);
    m_sock_handle = handle;
    m_state = CONN_STATE_CONNECTED;
    g_http_conn_map.insert(make_pair(m_conn_handle, this));
    
    netlib_option(handle, NETLIB_OPT_SET_CALLBACK, (void*)httpconn_callback);
    netlib_option(handle, NETLIB_OPT_SET_CALLBACK_DATA, reinterpret_cast<void *>(m_conn_handle) );
    netlib_option(handle, NETLIB_OPT_GET_REMOTE_IP, (void*)&m_peer_ip);
}

void httpconn_callback(void* callback_data, uint8_t msg, uint32_t handle, uint32_t uParam, void* pParam)
{
	NOTUSED_ARG(uParam);
	NOTUSED_ARG(pParam);
	// convert void* to uint32_t, oops
	uint32_t conn_handle = *((uint32_t*)(&callback_data));
    CHttpConn* pConn = FindHttpConnByHandle(conn_handle);
    if (!pConn) {
        return;
    }
	switch (msg)
	{
	case NETLIB_MSG_READ:
		pConn->OnRead();
		break;
	case NETLIB_MSG_WRITE:
		pConn->OnWrite();
		break;
	case NETLIB_MSG_CLOSE:
		pConn->OnClose();
		break;
	default:
		log("!!!httpconn_callback error msg: %d ", msg);
		break;
	}
}

//触发读事件
void CHttpConn::OnRead()
{
	for (;;)
	{
		uint32_t free_buf_len = m_in_buf.GetAllocSize() - m_in_buf.GetWriteOffset();
		if (free_buf_len < READ_BUF_SIZE + 1)
			m_in_buf.Extend(READ_BUF_SIZE + 1);
		int ret = netlib_recv(m_sock_handle, m_in_buf.GetBuffer() + m_in_buf.GetWriteOffset(), READ_BUF_SIZE);
		if (ret <= 0)
			break;
		m_in_buf.IncWriteOffset(ret);
		m_last_recv_tick = get_tick_count();
	}
	// 每次请求对应一个HTTP连接，所以读完数据后，不用在同一个连接里面准备读取下个请求
	char* in_buf = (char*)m_in_buf.GetBuffer();
	uint32_t buf_len = m_in_buf.GetWriteOffset();
	in_buf[buf_len] = '\0';
    // 如果buf_len 过长可能是受到攻击，则断开连接
    // 正常的url最大长度为2048，我们接受的所有数据长度不得大于1K
    if(buf_len > 1024)
    {
        log("get too much data:%s ", in_buf);
        Close();
        return;
    }
	//log("OnRead, buf_len=%u, conn_handle=%u\n", buf_len, m_conn_handle); // for debug
	m_cHttpParser.ParseHttpContent(in_buf, buf_len);

	if (m_cHttpParser.IsReadAll()) {//http://192.168.226.128:8080/msg_server，请求msgserver
		string url =  m_cHttpParser.GetUrl();
		if (strncmp(url.c_str(), "/msg_server", 11) == 0) {
            string content = m_cHttpParser.GetBodyContent();
            _HandleMsgServRequest(url, content);//处理请求
		} else {
			log("url unknown, url=%s ", url.c_str());
			Close();
		}
	}
}

// Add By Lanhu 2014-12-19 通过登陆IP来优选电信还是联通IP
void CHttpConn::_HandleMsgServRequest(string& url, string& post_data)
{
    msg_serv_info_t* pMsgServInfo;
    uint32_t min_user_cnt = (uint32_t)-1;
    map<uint32_t, msg_serv_info_t*>::iterator it_min_conn = g_msg_serv_info.end();
    map<uint32_t, msg_serv_info_t*>::iterator it;
    if(g_msg_serv_info.size() <= 0)
    {
        Json::Value value;
        value["code"] = 1;
        value["msg"] = "没有msg_server";
        string strContent = value.toStyledString();
        char* szContent = new char[HTTP_RESPONSE_HTML_MAX];
        snprintf(szContent, HTTP_RESPONSE_HTML_MAX, HTTP_RESPONSE_HTML, strContent.length(), strContent.c_str());
        Send((void*)szContent, strlen(szContent));
        delete [] szContent;
        return ;
    }
    
    for (it = g_msg_serv_info.begin() ; it != g_msg_serv_info.end(); it++) {
        pMsgServInfo = it->second;
        if ( (pMsgServInfo->cur_conn_cnt < pMsgServInfo->max_conn_cnt) &&
            (pMsgServInfo->cur_conn_cnt < min_user_cnt)) {
            it_min_conn = it;
            min_user_cnt = pMsgServInfo->cur_conn_cnt;
        }
    }
    
    if (it_min_conn == g_msg_serv_info.end()) {
        log("All TCP MsgServer are full ");
        Json::Value value;
        value["code"] = 2;
        value["msg"] = "负载过高";
        string strContent = value.toStyledString();
        char* szContent = new char[HTTP_RESPONSE_HTML_MAX];
        snprintf(szContent, HTTP_RESPONSE_HTML_MAX, HTTP_RESPONSE_HTML, strContent.length(), strContent.c_str());
        Send((void*)szContent, strlen(szContent));
        delete [] szContent;
        return;
    } else {
        Json::Value value;
        value["code"] = 0;
        value["msg"] = "";
        if(pIpParser->isTelcome(GetPeerIP()))
        {
            value["priorIP"] = string(it_min_conn->second->ip_addr1);
            value["backupIP"] = string(it_min_conn->second->ip_addr2);
            value["msfsPrior"] = strMsfsUrl;
            value["msfsBackup"] = strMsfsUrl;
        }
        else
        {
            value["priorIP"] = string(it_min_conn->second->ip_addr2);
            value["backupIP"] = string(it_min_conn->second->ip_addr1);
            value["msfsPrior"] = strMsfsUrl;
            value["msfsBackup"] = strMsfsUrl;
        }
        value["discovery"] = strDiscovery;
        value["port"] = int2string(it_min_conn->second->port);
        string strContent = value.toStyledString();
        char* szContent = new char[HTTP_RESPONSE_HTML_MAX];
        uint32_t nLen = strContent.length();
        snprintf(szContent, HTTP_RESPONSE_HTML_MAX, HTTP_RESPONSE_HTML, nLen, strContent.c_str());
        Send((void*)szContent, strlen(szContent));//发送响应
        delete [] szContent;
        return;
    }
}
````

##### 3.5 初始化login connection && httpconnection

````c++
init_login_conn();//从后面的分析来看，它的作用是向客户端和msgserver发送心跳包
void init_login_conn()
{
	netlib_register_timer(login_conn_timer_callback, NULL, 1000);
}
int netlib_register_timer(callback_t callback, void* user_data, uint64_t interval)
{
	CEventDispatch::Instance()->AddTimer(callback, user_data, interval);
	return 0;
}
//login connection定时向每一个连接的客户端发送心跳包，向每一个连接的msg_server发送心跳包
void CEventDispatch::AddTimer(callback_t callback, void* user_data, uint64_t interval)
{
	list<TimerItem*>::iterator it;
	for (it = m_timer_list.begin(); it != m_timer_list.end(); it++)
	{
		TimerItem* pItem = *it;
		if (pItem->callback == callback && pItem->user_data == user_data)
		{
			pItem->interval = interval;
			pItem->next_tick = get_tick_count() + interval;
			return;
		}
	}
	TimerItem* pItem = new TimerItem;
	pItem->callback = callback;
	pItem->user_data = user_data;
	pItem->interval = interval;
	pItem->next_tick = get_tick_count() + interval;
	m_timer_list.push_back(pItem);
}

//定时器执行函数
void login_conn_timer_callback(void* callback_data, uint8_t msg, uint32_t handle, void* pParam)
{
	uint64_t cur_time = get_tick_count();
	for (ConnMap_t::iterator it = g_client_conn_map.begin(); it != g_client_conn_map.end(); ) {
		ConnMap_t::iterator it_old = it;
		it++;

		CLoginConn* pConn = (CLoginConn*)it_old->second;
		pConn->OnTimer(cur_time);//执行每一个connection的定时函数
	}

	for (ConnMap_t::iterator it = g_msg_serv_conn_map.begin(); it != g_msg_serv_conn_map.end(); ) {
		ConnMap_t::iterator it_old = it;
		it++;

		CLoginConn* pConn = (CLoginConn*)it_old->second;
		pConn->OnTimer(cur_time);
	}
}

//发送心跳包
void CLoginConn::OnTimer(uint64_t curr_tick)
{
	if (m_conn_type == LOGIN_CONN_TYPE_CLIENT) {
		if (curr_tick > m_last_recv_tick + CLIENT_TIMEOUT) {
			Close();
		}
	} else {
		if (curr_tick > m_last_send_tick + SERVER_HEARTBEAT_INTERVAL) {
            IM::Other::IMHeartBeat msg;
            CImPdu pdu;
            pdu.SetPBMsg(&msg);
            pdu.SetServiceId(SID_OTHER);
            pdu.SetCommandId(CID_OTHER_HEARTBEAT);
			SendPdu(&pdu);
		}
		if (curr_tick > m_last_recv_tick + SERVER_TIMEOUT) {
			log("connection to MsgServer timeout ");
			Close();
		}
	}
}
````

````c++
init_http_conn();
void init_http_conn()
{
	netlib_register_timer(http_conn_timer_callback, NULL, 1000);
}
//http连接定时函数，遍历每一个http连接，超时自动关闭连接，不会发送心跳包，这个符合http的连接的性质。
void http_conn_timer_callback(void* callback_data, uint8_t msg, uint32_t handle, void* pParam)
{
	CHttpConn* pConn = NULL;
	HttpConnMap_t::iterator it, it_old;
	uint64_t cur_time = get_tick_count();
	for (it = g_http_conn_map.begin(); it != g_http_conn_map.end(); ) {
		it_old = it;
		it++;
		pConn = it_old->second;
		pConn->OnTimer(cur_time);
	}
}

void CHttpConn::OnTimer(uint64_t curr_tick)
{
	if (curr_tick > m_last_recv_tick + HTTP_CONN_TIMEOUT) {
		log("HttpConn timeout, handle=%d ", m_conn_handle);
		Close();
	}
}
````

##### 3.6 事件分发

````c++
//和之前分析的是一样的
netlib_eventloop();
void netlib_eventloop(uint32_t wait_timeout)
{
	CEventDispatch::Instance()->StartDispatch(wait_timeout);
}
void CEventDispatch::StartDispatch(uint32_t wait_timeout)
{
    ...
	while (running)
	{
		nfds = epoll_wait(m_epfd, events, 1024, wait_timeout);
		for (int i = 0; i < nfds; i++)
		{
			int ev_fd = events[i].data.fd;
			CBaseSocket* pSocket = FindBaseSocket(ev_fd);
			...
			if (events[i].events & EPOLLIN)
			{
				//log("OnRead, socket=%d\n", ev_fd);
				pSocket->OnRead();
			}
			if (events[i].events & EPOLLOUT)
			{
				//log("OnWrite, socket=%d\n", ev_fd);
				pSocket->OnWrite();
			}
			...
			pSocket->ReleaseRef();
		}
		_CheckTimer();//定时器事件
        _CheckLoop();//响应事件
	}
}
````



#### 4.msfs服务器 聊天图片的上传和下载服务器。

​		msfs服务器是单独的服务器，被客户端直接访问，不与其他服务器连接。

​		客户端以http的方式上传和下载聊天图片。



















