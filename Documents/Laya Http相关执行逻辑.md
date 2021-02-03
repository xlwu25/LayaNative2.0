## HTTP相关执行逻辑

### 简单讲解
在AjaxManager中，我们所使用类为HttpRequest，通过查看laya.core.js中源码可以发现，代码为：「this._http=new Browser.window.XMLHttpRequest();」该类使用的是挂载到window上的一个类，而这个挂载的类在不同的运行环境下，有不同的实现方式，而在app中，类似于之前我们讲过的ConchInput自定义输入框，是在apploader.js文件中挂载到window上的，代码如下：

```
class XMLHttpRequest extends EventTarget {
    ...
}
XMLHttpRequest.UNSENT = 0;
XMLHttpRequest.OPENED = 1;
XMLHttpRequest.HEADERS_RECEIVED = 2;
XMLHttpRequest.LOADING = 3;
XMLHttpRequest.DONE = 4;
window.XMLHttpRequest = XMLHttpRequest;
```

所以，我们所使用的类就是从apploader.js这里来的，调用的也是这里的方法。通过查看其constructor方法，得到_XMLHttpRequest类。

```
constructor() {
    super();
    this._hasReqHeader = false;
    this.withCredentials = false;
    this.setResponseHeader = function (name, value) {
        this._head = value;
    };
    this.xhr = new _XMLHttpRequest();
    this._readyState = 0;
    this._responseText = this._response = this._responseType = this._url = "";
    this._responseType = "text";
    this._method = "GET";
    this.xhr._t = this;
    this.xhr.set_onreadystatechange(function (r) {
        var _t = this._t;
        if (r == 1) {
            _t._readyState = 1;
        }
        if (_t._onrchgcb) {
            var e = new _lbEvent("readystatechange");
            e.target = _t;
            _t._onrchgcb(e);
        }
        var ev;
        if (_t._status == 200) {
            ev = new _lbEvent("load");
            ev.target = _t;
            _t.dispatchEvent(ev);
        }
        else if (_t._status == 404) {
            ev = new _lbEvent("error");
            ev.target = _t;
            _t.dispatchEvent(ev);
        }
    });
}
```

搜索全局，发现在XMLHttpRequest.cpp方法中，注册了_XMLHttpRequest类到JS，「JSP_INSTALL_CLASS("_XMLHttpRequest", XMLHttpRequest);」，所以XMLHttpRequest类就是我们要找的类的C++实现。

通过查看代码发现，主要的HTTP相关代码在于common/source/downloadMgr文件夹内，涉及的文件分别为JCHttpHeader、JCDownloadMgr、JCCurlWrap。JCHttpHeader类看的不多，估么着主要是用于封装发送和收到的Http的Header；JCDownloadMgr是比较主要的类，里面定义了下载文件、获取数据完成后回调等方法，主要和XMLHttpRequest和JCCurlWrap类交互，JCCurlWrap是对第三方库curl的二次封装，真正的请求在curl中执行。

### AjaxManager Get方法执行过程

#### Apploader.js的XMLHttpRequest类send方法
```
    var onPostLoad = function (buf, strbuf) {
        var _t = this._t;
        if (_t.responseType == 'arraybuffer') {
            _t._response = buf;
            _t._responseText = strbuf;
        }
        else {
            _t._response = _t._responseText = buf;
        }
        _t._readyState = 4;
        _t._status = 200;
        _t.xhr._changeState(4);
        if (_t._onloadcb) {
            _t._loadsus();
        }
        onPostLoad.ref = onPostError.ref = null;
    };
    var onPostError = function (e1, e2) {
        var _t = this._t;
        _t._readyState = 4;
        _t._status = 404;
        _t.xhr._changeState(4);
        if (_t.onerror) {
            var ev = new _lbEvent("error");
            ev.target = _t;
            ev['ecode1'] = e1;
            ev['ecode2'] = e2;
            _t.onerror(ev);
        }
        onPostLoad.ref = onPostError.ref = null;
    };

    if (this._method == 'POST' && body) {    //POST
        onPostLoad.ref = onPostError.ref = this.xhr;
        this.xhr.setPostCB(onPostLoad, onPostError);  //设置回调方法
        this.xhr.postData(this.url, body);
    }
    else if (this._hasReqHeader) {           //GET，当header参数不为空时
        onPostLoad.ref = onPostError.ref = this.xhr;
        this.xhr.setPostCB(onPostLoad, onPostError);
        this.xhr.getData(this.url);
    }
    else {
        ...
    }
```

从apploader.js的XMLHttpRequest类send方法开始执行，看以上代码，得知不是POST方法的时候，当有Head参数的时候，就是走的getData（即约等于GET方法），除了调用getData还调用了setPostCB，该方法用于设置成功和失败的回调方法，即onPostLoad方法和onPostError方法，而且在onPostError方法中我们可以看到，设置了错误码为404，这就是为什么我们获取到的错误码都是404的原因。接下去我们看XMLHttpRequest.cpp中的setPostCB和getData方法。

#### setPostCB方法
```
void XMLHttpRequest::setPostCB(JSValueAsParam p_onOK, JSValueAsParam p_onError) 
{
    //m_jsfunPostComplete用于后面在_onPostComplete_JSThread方法回调，_onPostComplete_JSThread方法在_onPostComplete中调用
    m_jsfunPostComplete.set(oncompleteid, this, p_onOK);  //HTTP完成回调
    m_jsfunPostComplete.__BindThis(m_This);
    m_jsfunPostError.set(onerrid, this, p_onError);  //HTTP错误回调
    m_jsfunPostError.__BindThis(m_This);
    std::weak_ptr<int> cbref(m_CallbackRef);

    //m_funcPostComplete绑定_onPostComplete，用于postData或getData方法中(download)直接传入，到download方法中变为_QueryDownload对象的mOnEnd方法，拿到数据之后mOnEnd存在则执行回调
    m_funcPostComplete = std::bind(_onPostComplete, this, isBin(),m_pCmdPoster, 
        std::placeholders::_1, 
        std::placeholders::_2,
        std::placeholders::_3,
        std::placeholders::_4,
        std::placeholders::_5,
        std::placeholders::_6,
        cbref);
}
```

看以上代码得知，setPostCB方法接收两个参数，一个是成功回调，另一个是失败回调。m_jsfunPostComplete用于在_onPostComplete_JSThread回调，_onPostComplete_JSThread方法在_onPostComplete中调用；m_funcPostComplete绑定_onPostComplete，用于postData或getData方法中(download)直接传入，到download方法中变为_QueryDownload对象的mOnEnd方法，拿到数据之后mOnEnd存在则执行回调。
#### getData方法

```
    void XMLHttpRequest::getData(const char* p_sUrl) 
    {
        JCDownloadMgr* pdmgr = JCDownloadMgr::getInstance();
        if (!pdmgr)
        {
            m_pCmdPoster->postToJS(std::bind(_onPostError_JSThread, this, -1, 0, std::weak_ptr<int>(m_CallbackRef)));
            return;
        }
        else 
        {
            std::vector<std::string> headers;
            HTTPHeaderMap::iterator it = m_requestHeaders.begin();
            while (it != m_requestHeaders.end()) 
            {
                std::string head = (*it).first;
                head += ": ";//标准是可以有任意个空格
                head += (*it).second;
                headers.push_back(head);
                it++;
            }
            pdmgr->download(p_sUrl, 0, JCDownloadMgr::defProgressFunc, m_funcPostComplete,0,headers);  //设置回调方法m_funcPostComplete
        }
    }

    void JCDownloadMgr::download(const char *p_pszURL, int p_nPriority,
        onProgressFunc p_ProgCb, onEndFunc p_CompleteCb,
        const char* p_pPostData, int p_nPostLen,  //可以为0
        bool p_bOnlyHeader,                       //一般是false
        int p_nTimeout, int p_nConnTimeout,       //为0则缺省
        std::vector<std::string> p_vHeaders,      //size()==0则忽略
        const char* p_pLocalFile,                 //一边下载，一边保存，一般用在大文件下载，0则忽略
        bool p_bChkRemoteChange             //检查远端文件是否改变了，大文件用
        ) {
#ifndef THIN_COMMON_WITHOUT_DOWNLOAD
        m_bCancelTask = false;
        if (0 != p_pszURL) {
            if (strlen(p_pszURL) <= 0) {
                LOGE("Error! downloadMgr::download url len=0");
                return;
            }
            static int nTh = 0;
            int nThNum = m_ThreadPool.getThreadNum();// _l_PoolQuery->GetSize();
            if (nThNum <= 0)
                return;
            _QueryDownload* pQuery = new _QueryDownload(p_pszURL);
            pQuery->mOnEnd = p_CompleteCb;
            pQuery->mOnProg = p_ProgCb;
            pQuery->mnOptTimeout = p_nTimeout>0?p_nTimeout:s_nOptTimeout;
            pQuery->mnConnTimeout = p_nConnTimeout > 0 ? p_nConnTimeout : s_nConnTimeout;
            pQuery->mHeaders = p_vHeaders;
            pQuery->mbOnlyHeader = p_bOnlyHeader;
            if(p_pPostData)
                pQuery->setPostData(p_pPostData, p_nPostLen);
            if (p_pLocalFile) {
                pQuery->mstrLocalFile = p_pLocalFile;
            }
            if (p_nPriority == 1 || nThNum == 1)
                m_ThreadPool.sendToThread(pQuery, 0);  //设置到线程中并立即执行
            else {
                nTh %= (nThNum - 1);
                m_ThreadPool.sendToThread(pQuery, nTh + 1);
            }
            nTh++;
        }
#endif
    }
```
看以上代码可知，这里实际上是调用了JCDownloadMgr类的download方法，该方法接收了url、优先级（该参数决定在线程中什么时候执行）、进度、完成回调、是否二进制等参数，这里主要关注的是执行回调，传入的回调参数为p_CompleteCb，从pQuery->mOnEnd = p_CompleteCb;这句代码中可知，回调函数赋值给了pQuery对象的mOnEnd属性，pQuery为_QueryDownload类的实例，后面的sendToThread(pQuery, 0)应该是设置需要执行的代码到线程中执行。后续线程调用的逻辑不是很清楚，后面就调用到了JCDownloadMgr类中的__WorkThread，在这个方法中通过代码while (!pQuery->run(&_lCurl))执行了_QueryDownload类的run方法，接下去我们继续看run方法。


#### _QueryDownload类run方法
```
virtual bool run(Curl* pCurl) {
    if (JCDownloadMgr::m_bCancelTask) {
        return true;
    }
    pCurl->setProgressCB(_OnProgress, this);

    unsigned char *pContentBuff = nullptr;
    size_t iContentBuffSize = 0;
    bool bBigFile = mstrLocalFile.length() > 0;
    LOGI("Download [%c%c]:%s", mbOnlyHeader ? 'H' : ' ', bBigFile ? 'B' : ' ', mUrl.c_str());
    // 下载文件
    JCUrl url(mUrl.c_str());
    std::string encodeUrl = mUrl;// url.encode();
    bool bHasQuery = url.m_Query.length() > 0;
    char *pszUrlBuff = gDownloadMgr->getFinalUrl(encodeUrl.c_str());
    __Buffer* _pContentBuffer = NULL;

    pCurl->query(pszUrlBuff, &_pContentBuffer,
        mpPostData, mnPostDataLen,
        mbOnlyHeader,
        mnOptTimeout, mnConnTimeout,
        mHeaders,
        bBigFile ? mstrLocalFile.c_str() : nullptr,
        bBigFile);
    if (_pContentBuffer) {
        iContentBuffSize = _pContentBuffer->GetDataSize();
        pContentBuff = (unsigned char*)_pContentBuffer->SwapBuff(0, 0);
    }
    if(mpPostData)
        delete mpPostData;
    mpPostData = NULL;

    if (mOnEnd) {
        if (pCurl->m_nCurlRet != CURLE_OK) {
            //curl 执行失败
            static std::string nullstr;
            JCBuffer jb;
            mOnEnd(jb, pCurl->m_strLocalAddr, pCurl->m_strSvAddr, pCurl->m_nCurlRet, pCurl->m_nResponseCode, nullstr);
        }
        else {
            //bool bHttpOK = pCurl->m_nResponseCode >= 200 && pCurl->m_nResponseCode < 300;
            LOGI("Download end:%d", pCurl->m_nResponseCode);
            if (bBigFile || iContentBuffSize <= 0) {//大文件没有buffer
                JCBuffer jb;
                mOnEnd(jb, pCurl->m_strLocalAddr, pCurl->m_strSvAddr,
                    CURLE_OK, pCurl->m_nResponseCode,
                    pCurl->m_strResponseHead);
            }
            else {
                if(pContentBuff && iContentBuffSize>0){
                    int sz = iContentBuffSize;
                    unsigned char* posthandleData = pContentBuff;
                    gDownloadMgr->postDownload(pszUrlBuff, posthandleData, sz);
                    iContentBuffSize = sz;
                    if (posthandleData != pContentBuff) {
                        delete[] pContentBuff;
                        pContentBuff = posthandleData;
                    }
                }

                JCBuffer buf((void*)pContentBuff, iContentBuffSize, false, true);
                mOnEnd(buf, pCurl->m_strLocalAddr, pCurl->m_strSvAddr,
                    CURLE_OK, pCurl->m_nResponseCode,
                    pCurl->m_strResponseHead);
            }
        }
    }
    delete[] pszUrlBuff;
    return true;
}

```

在这个方法里我们看到，执行了Curl类的query方法，执行完成后判断mOnEnd，如果有mOnEnd则执行回调。下面看Curl类的query方法。


#### Curl类query方法
```
void Curl::query(const char *p_pszUrl, __Buffer **p_ppResBuff,
        const char* p_pPostData, int p_nPostLen,  //可以为0
        bool p_bOnlyHeader,                       //一般是false
        int p_nTimeout, int p_nConnTimeout,       //为0则缺省
        const std::vector<std::string>& p_vHeaders,      //size()==0则忽略
        const char* p_pLocalFile,                 //一边下载，一边保存，一般用在大文件下载，0则忽略
        bool p_bChkRemoteChange                   //检查远端文件是否改变了，大文件用
        ){
    if(p_ppResBuff)
        *p_ppResBuff = 0;
    m_nResponseCode = 0;
    FILE* fp = nullptr;
    do {
        if (!_Prepare())
            break;
        begin();
        //如果要保存到本地的处理
        if (p_pLocalFile) {
            m_nLocalFileLen = GetLocalFileLenth(p_pLocalFile);
            if (p_bChkRemoteChange) {
                //要检查远程是否改变了
                unsigned int remoteLen=0;
                std::string lastModified, etag;
                //注意：这个会改变很多状态，所以需要在最前面执行。
                bool br = getRemoteFileInfo(m_pCurlHandle, p_pszUrl, remoteLen, lastModified, etag);
                //TODO 检查文件是否改变了，这个也可以在脚本中做
                if (m_nLocalFileLen > 0 && (long)remoteLen == m_nLocalFileLen) {
                    m_nCurlRet = CURLE_OK;
                    m_nResponseCode = 200;
                    break;
                }
            }
            fp = fopen(p_pLocalFile, "a+b");
            if (!fp) {
                LOGW("Open file error:%s", p_pLocalFile);
                m_nCurlRet = CURLE_GOT_NOTHING;
                break;
            }
            fseek(fp, 0, SEEK_END);

            set_OnData(downLoadPackage,fp);
            //if(m_nLocalFileLen>0)
                curl_easy_setopt(m_pCurlHandle, CURLOPT_RESUME_FROM, m_nLocalFileLen);
        }
        else {
            set_OnData(Curl::_WriteCallback, this);
            curl_easy_setopt(m_pCurlHandle, CURLOPT_RESUME_FROM, 0);
        }

        m_nOptTimeout = p_nTimeout;
        int opttimeout = p_nTimeout == 0 ? DEF_OPTTIMEOUT : p_nTimeout;
        set_Url(p_pszUrl);
        set_Header(p_vHeaders);
        ApplyHeaders();
        if (p_pPostData && p_nPostLen > 0) {
            set_PostData(p_pPostData, p_nPostLen);
        }
        else {
            set_GET();
        }
        set_OnlyHead(p_bOnlyHeader);
        set_Timeout(opttimeout);
        int connTimeout = p_nConnTimeout==0? 8: p_nConnTimeout;
        set_ConnectTimeout(connTimeout);

        m_nCurlRet = curl_easy_perform(m_pCurlHandle);  //同步执行传输
        if (!checkResult(p_pszUrl)) {
            m_Buffer.ClearData();
        }
        else if (p_bOnlyHeader) {
            m_Buffer.ClearData();
            m_Buffer.AddData(m_strResponseHead.c_str(), m_strResponseHead.length());
        }
        if(p_ppResBuff)
            *p_ppResBuff = &m_Buffer;
        //curl_easy_setopt(m_pCurlHandle, CURLOPT_WRITEHEADER, 0);
    } while (false);
    end();

    if (fp)
        fclose(fp);

    curl_easy_setopt(m_pCurlHandle, CURLOPT_HTTPHEADER, 0);
    curl_easy_setopt(m_pCurlHandle, CURLOPT_POSTFIELDS, 0);
    curl_easy_setopt(m_pCurlHandle, CURLOPT_POSTFIELDSIZE, 0);
    curl_easy_setopt(m_pCurlHandle, CURLOPT_POST, 0L);
    //不能调用这个。会丢失之前的有效设置。例如session
    //curl_easy_reset(m_pCurlHandle);
}
```
这个方法就是对第三方Curl库的一些操作了，如curl_easy_setopt和curl_easy_perform，curl_easy_setopt方法是配置一些信息，如请求的URL等，比较复杂。curl_easy_perform用于同步地发送数据，如果一直没有响应，该方法会堵塞后面的代码。

### 附录：libcurl库使用步骤
1. 调用curl_global_init()初始化libcurl
2. 根据curl_easy_setopt()设置的传输选项，实现回调函数以完成用户特定任务
3. 调用curl_easy_perform()函数完成传输任务
4. 调用curl_easy_cleanup()释放内存
5. 调用curl_global_cleanup()析构libcurl
