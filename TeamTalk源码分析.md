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
	netlib_add_loop(proxy_loop_callback, NULL);//添加网络事件处理函数
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
​		新客户端连接。

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
		AddBaseSocket(pSocket);//
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
	netlib_option(handle, NETLIB_OPT_SET_CALLBACK_DATA, (void*)&g_proxy_conn_map);
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
	ConnMap_t* conn_map = (ConnMap_t*)callback_data;
	CImConn* pConn = FindImConn(conn_map, handle);
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
````

​		处理可读事件。

````c
#define closesocket close
#define ioctlsocket ioctl
void CBaseSocket::OnRead()
{
	if (m_state == SOCKET_STATE_LISTENING)//serverfd
	{
		_AcceptNewSocket();
	}
	else//clientfd
	{	
		u_long avail = 0;
		if ( (ioctlsocket(m_socket, FIONREAD, &avail) == SOCKET_ERROR) || (avail == 0) )
		{
			m_callback(m_callback_data, NETLIB_MSG_CLOSE, (net_handle_t)m_socket, NULL);//close
		}
		else
		{
			m_callback(m_callback_data, NETLIB_MSG_READ, (net_handle_t)m_socket, NULL);//read
		}
	}
}
````

​		处理可写事件。

````c
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
````





