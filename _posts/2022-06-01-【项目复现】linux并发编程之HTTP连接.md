---

layout:     post
title:      「项目复现」linux并发编程之HTTP连接
subtitle:   Linux系统
date:       2022-06-01
author:     MRL Liu
header-img: img/post-bg-hacker.png
catalog: true
tags:
    - 项目复现
---

​	   超文本传输协议HTTP（Hyper Text Transfer Protocol）是Web服务的应用层协议，超本文指的是含有超链接的文本，HTTP不仅可以传输文本，还可以传输图片、视频等更多信息形式。

# 一、HTTP请求和响应

## 1、使用浏览器访问服务器

假设我们在浏览器的搜索框输入一个 `"http://127.0.0.1:1316/"`

我们的命令默认创建一个GET方法的HTTP请求：

```http
GET / HTTP/1.1
Host: 127.0.0.1:1316
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/100.0.4896.88 Safari/537.3
Accept: text/html,application/xhtml+xml,application/xml;q=0.9
Accept-Language: zh-CN,zh;q=0.9
Accept-Encoding: gzip, deflate
Connection: keep-alive
Cookie: __yjs_duid=1_62b6e569b9117f77c6b908516fcc6d041621751020592; BAIDUID_BFESS=6270BCEFAB102EF717C046ECF7D9F772:FG=1; BIDUPSID=96E33A2E08D8CA03FDBFD42D0EB095F1; PSTM=1654071693; BAIDUID=96E33A2E08D8CA03DA0276C03470DCB1:FG=1; BD_LAST_QID=16758245400036680781
```

该HTTP请求的字段解释如下：

> GET指定从服务器获取一个资源
>
> HTTP/1.1指定HTTP协议的版本号
>
> Host指定目标服务器的域名（可能是同一台服务器上的不同网站）
>
> Connection指定本次连接使用长连接还是短连接
>
> Accept指定本次请求可以接收的数据类型，`*/*`表示任何类型
>
> Accept-Encoding指定本次请求可以接收的数据压缩格式
>
> Accept-Language指定本次请求可以接收的语言

服务器处理后，会向浏览器返回一个HTTP响应：

```http
HTTP/1.1 200 OK
Connection: keep-alive
Content-Encoding: gzip
Connect-Type: text/html; charset=utf-8
Connect-Length: 3385

<!DOCTYPE html>
<html lang="en">
<head>
</head>
...
```

该HTTP响应的字段解释如下：

> HTTP/1.1指定HTTP协议的版本号
>
> 200是状态码，不同的状态码表示本次响应的不同处理结果，200说明请求成功
>
> OK是短语，用来和状态码配合说明处理结果
>
> Connection指定本次连接使用长连接还是短连接
>
> Connect-Encoding指定本次响应的数据压缩格式，之后的字节就属于下一个回应了
>
> Connect-Type指定本次响应的数据类型
>
> Connect-Length指定本次响应的消息实体的数据长度，之后的字节就属于下一个响应

请注意，在浏览器上输入一次网址或者切换新的网页，并不会只发送一个HTTP请求，想要完整地获取一个网页，会发送多个HTTP请求：

请求HTML文档的GET类型的HTTP请求；

请求CSS文件的GET类型的HTTP请求；

请求JS文件的GET类型的HTTP请求；

请求图片的GET类型的HTTP请求；

请求视频的GET类型的HTTP请求等。

​        HTTP连接是建立在TCP连接上的，如果每次发送HTTP请求都需要建立一次TCP连接，那么网页的相应速度将会较慢；现在的浏览器在请求一个网页时，都是同时建立多个TCP连接，在每个TCP连接上多次发送HTTP请求，以便提高响应速度。每个HTTP连接建立的TCP连接在发送一个HTTP请求后并不会立即关闭，而是有一个时延，在规定时间内如果还有HTTP请求就继续发送，没有了再关闭，这就是HTTP的长连接。

## 2、使用简单的命令行访问

假设输入一个 `curl -v "http://127.0.0.1:1316/login"`

我们的命令默认创建一个GET方法的HTTP请求：

```http
GET /login HTTP/1.1
Host: 127.0.0.1:1316
Connection: close
User-Agent: curl/7.60.0
Accept: */*

```

该HTTP请求的字段解释如下：

> GET指定请求一个URL标志的文档
>
> Host指定目标服务器的域名（可能是同一台服务器上的不同地址）
>
> Connection指定本次连接使用长连接还是短连接





得到一个HTTP响应：

```http
HTTP/1.1 200 OK
Connection: close
Connect-type: text/html
Connect-length: 3385

<!DOCTYPE html>
<html lang="en">
<head>
</head>
...
```

该HTTP响应的字段解释如下：

> GET指定请求一个URL标志的文档
>
> Connect-length指定本次响应的数据长度，之后的字节就属于下一个回应了



# 二、HTTP请求的解析实现

现在尝试实现HTTP请求的解析

## 1、存储结果的数据结构

首先考虑定义存储解析HTTP请求的结果的数据结构：

```c++
string method_,path_,version_;// 请求行的解析结果
string body_;// 请求消息实体的解析结果
unordered_map<string,string> header_;// 请求头部字段
unordered_map<string,string> post_;// post方法的数据
```

## 2、基于有限状态机的逐步解析

基于有限状态机的逐步解析实现：

```C++
// HTTP请求的解析状态
enum PARSE_STATE{
    REQUEST_LINE,// 请求行
    HEADERS，// 请求头部字段
    BODY,// 请求消息实体
    FINISH,// 请求结束
}
PARSE_STATE state_;// 当前HTTP请求的解析状态
// 解析所有
bool parse(string line){
    switch(state_){
        case PARSE_STATE:
            ParseRequestLine(line);
            ParseRequestPath(line);
            break;
        case HEADERS:
            ParseRequestHeaders(line);
            // 如果line长度较少，说明body为空，直接结束
            if(line.length()<=2){
                state_=FINISH;
			}
            break;
        case BODY:
            ParseRequestBody(line);
            break;
        case FINISH:
            break;
        default:
            break;
    }
    printf("[%s],[%s],[%s]",method_.c_str(),path_.c_str(),version_.c_str())
}
```

## 3、请求行的解析

```c++
#include<regex> // 正则表达式
bool ParseRequestLine(const string& line){
    regex patten("^([^ ]*) ([^ ]*) HTTP/([^ ]*)$");
    smatch subMatch;//匹配结果
    // 全文匹配，line匹配patten，匹配结果放在subMatch中
    if(regex_match(line, subMatch, patten)) {   
        method_ = subMatch[1];//方法
        path_ = subMatch[2];//URL路径
        version_ = subMatch[3];//版本号
        state_ = HEADERS;//切换state_
        return true;
    }
    else{
        LOG_ERROR("RequestLine Error");
    	return false;
    }
    
}
```

## 4、请求头部的解析

```c++
void ParseRequestPath(const string& line) {
    regex patten("^([^:]*): ?(.*)$");
    smatch subMatch;
    // 如果全文匹配成功，保存匹配结果
    if(regex_match(line, subMatch, patten)) {
        header_[subMatch[1]] = subMatch[2];
    }
    // 匹配失败，切换到下一状态
    else {
        state_ = BODY;
    }
}
```

## 5、请求消息实体的解析

```c++
// 解析消息体
void HttpRequest::ParseBody_(const string& line) {
    body_ = line;
    ParsePost_();//解析表单
    state_ = FINISH;
    printf("Body:%s, len:%d", line.c_str(), line.size());
}
```

解析表单

```c++
void ParsePost_() {
    // 如果该HTTP请求是POST，并且内容格式是浏览器默认，编码为key-value
    if(method_ == "POST" && header_["Content-Type"] == "application/x-www-form-urlencoded") {
        ParseFromUrlencoded_();//解析body,存入post_数组
        // 如果默认的字典中存在该访问路径
        if(DEFAULT_HTML_TAG.count(path_)) {
            int tag = DEFAULT_HTML_TAG.find(path_)->second;//查找该路径对应的tag
            LOG_DEBUG("Tag:%d", tag);
            if(tag == 0 || tag == 1) {
                bool isLogin = (tag == 1);
                // 注册或登录/验证数据
                if(UserVerify(post_["username"], post_["password"], isLogin)) {
                    path_ = "/welcome.html";//验证成功
                } 
                else {
                    path_ = "/error.html";
                }
            }
        }
    }   
}
void ParseFromUrlencoded_() {
    if(body_.size() == 0) { return; }
    string key, value;// key-value
    int num = 0;
    int n = body_.size();
    int i = 0, j = 0;
	// 遍历整个body
    for(; i < n; i++) {
        char ch = body_[i];//转换为字符
        switch (ch) {
        case '=':// 说明此时body_的[j,i)是一个key
            key = body_.substr(j, i - j);
            j = i + 1;
            break;
        case '+':// +对应空格
            body_[i] = ' ';
            break;
        case '':
            num = ConverHex(body_[i + 1]) * 16 + ConverHex(body_[i + 2]);
            body_[i + 2] = num % 10 + '0';
            body_[i + 1] = num / 10 + '0';
            i += 2;
            break;
        case '&':// 说明此时body_的[j,i)是一个key
            value = body_.substr(j, i - j);
            j = i + 1;
            post_[key] = value;
            prinf("%s = %s", key.c_str(), value.c_str());// 打印此时的key-value
            break;
        default:
            break;
        }
    }
    assert(j <= i);
    // 如果post_中没有添加过该数据
    if(post_.count(key) == 0 && j < i) {
        value = body_.substr(j, i - j);
        post_[key] = value;
    }
}
```

PostMan

# 三、HTTP响应的解析实现

根据是否解析成功来初始化应答：

```c++
Init(srcDir, request_.path(), request_.IsKeepAlive(), 200);//初始化一个解析成功的HTTP响应
response_.Init(srcDir, request_.path(), false, 400);//初始化一个解析失败的HTTP响应
```

## 1、初始化方法

```c++
// 初始化应答，Web资源路径、HTTP请求路径、是否长连接、状态码
void Init(const string& srcDir, string& path, bool isKeepAlive, int code){
    assert(srcDir != "");
    if(mmFile_) { UnmapFile(); }
    code_ = code;
    isKeepAlive_ = isKeepAlive;
    path_ = path;
    srcDir_ = srcDir;
    mmFile_ = nullptr; 
    mmFileStat_ = { 0 };
}
```

## 2、制作方法：

```C++
void MakeResponse(Buffer& buff) {
    /* 判断请求的资源文件 */
    if(stat((srcDir_ + path_).data(), &mmFileStat_) < 0 || S_ISDIR(mmFileStat_.st_mode)){
        code_ = 404;
    }
    else if(!(mmFileStat_.st_mode & S_IROTH)) {
        code_ = 403;
    }
    else if(code_ == -1) { 
        code_ = 200; 
    }
    ErrorHtml_();// 如果状态码是错误的，添加错误的code
    AddStateLine_(buff);// 添加状态行
    AddHeader_(buff);// 添加头部
    AddContent_(buff);// 添加内容
}
```

ErrorHtml_

```C++
void HttpResponse::ErrorHtml_() {
    if(CODE_PATH.count(code_) == 1) {
        path_ = CODE_PATH.find(code_)->second;// 根据错误代码找到对应的路径
        stat((srcDir_ + path_).data(), &mmFileStat_);
    }
}
```

添加状态行：

```c++
void AddStateLine_(Buffer& buff) {
    string status;
    // 如果当前状态码是否找到对应短语
    if(CODE_STATUS.count(code_) == 1) {
        status = CODE_STATUS.find(code_)->second;
    }
    // 如果没有找到，重设状态码
    else {
        code_ = 400;
        status = CODE_STATUS.find(400)->second;
    }
    buff.Append("HTTP/1.1 " + to_string(code_) + " " + status + "\r\n");
}
```

添加头部字段：

```C++
void AddHeader_(Buffer& buff) {
    buff.Append("Connection: ");
    if(isKeepAlive_) {
        buff.Append("keep-alive\r\n");
        buff.Append("keep-alive: max=6, timeout=120\r\n");
    } else{
        buff.Append("close\r\n");
    }
    buff.Append("Content-type: " + GetFileType_() + "\r\n");
}
```

添加内容实体：

```C++
void AddContent_(Buffer& buff) {
    // 以只读的方式打开对应的文件
    int srcFd = open((srcDir_ + path_).data(), O_RDONLY);
    if(srcFd < 0) { 
        ErrorContent(buff, "File NotFound!");
        return; 
    }

    /* 将文件映射到内存提高文件的访问速度 
        MAP_PRIVATE 建立一个写入时拷贝的私有映射*/
    LOG_DEBUG("file path %s", (srcDir_ + path_).data());
    int* mmRet = (int*)mmap(0, mmFileStat_.st_size, PROT_READ, MAP_PRIVATE, srcFd, 0);
    // 如果映射错误
    if(*mmRet == -1) {
        ErrorContent(buff, "File NotFound!");
        return; 
    }
    mmFile_ = (char*)mmRet;
    close(srcFd);
    buff.Append("Content-length: " + to_string(mmFileStat_.st_size) + "\r\n\r\n");
}
```

# 四、HTTP连接的对象实现

## 1、HTTP连接对象的公有数据变量

```c++
static bool isET;// 公共的是否ET模式
static atomic<int> userCount;// 公共的当前用户数（原子性）
static const char* srcDir;// 公共的当前工作目录
```

## 2、HTTP连接对象的私有数据变量

```c++
int fd_; //连接socket
struct  sockaddr_in addr_;// socket地址
bool isClose_;// 是否关闭
Buffer readBuff_; // 读缓冲区
Buffer writeBuff_; // 写缓冲区
HttpRequest request_;// HTTP请求
HttpResponse response_;// HTTP响应
```

## 3、当一个客户端连接的epoll事件发生时

```c++
/*初始化Http连接*/
void init(int fd, const sockaddr_in& addr) {
    assert(fd > 0);
    fd_ = fd;
    addr_ = addr;
    isClose_ = false;
    userCount++;// 增加用户数
    writeBuff_.RetrieveAll();// 清空
    readBuff_.RetrieveAll();// 清空
}
```

## 4、当一个客户端关闭的epoll事件发生时

```c++
void Close() {
    // response_.UnmapFile();
    if(isClose_ == false){
        isClose_ = true; 
        userCount--;// 减少用户数
        std::close(fd_);// 关闭套接字
    }
}
```

## 5、当一个客户端读取的epoll事件发生时

```c++
// 返回本次读取成功的数据
ssize_t read(int* saveErrno) {
    ssize_t len = -1;
    // 尝试从fd_的缓存区读取数据
    do {
        len = readBuff_.ReadFd(fd_, saveErrno);
        if (len <= 0) {
            break;
        }
    } while (isET);
    return len;
}
```

## 6、当一个客户端写入的epoll事件发生时

```c++
// 返回本次写入成功的数据
ssize_t write(int* saveErrno) {
    ssize_t len = -1;
    // 尝试传输一次数据，但是不一定可以立即全部传输：
    do {
        // 向fd_的缓存区写入iovCnt_个IO指针iov_指向的缓存区数据，先写IO[0],后写IO[1]
        len = writev(fd_, iov_, iovCnt_);// len表示已经写入完成的数据
        // 如果写入失败，直接返回
        if(len <= 0) {
            *saveErrno = errno;
            break;
        }
        // 如果传输已经结束：两个IO向量指向的剩余数据已经为0
        if(iov_[0].iov_len + iov_[1].iov_len  == 0) { break; } /* 传输结束 */
        // 如果传输没有结束但是iov_[0]已经传输完成、iov_[1]传输了部分数据：
        else if(static_cast<size_t>(len) > iov_[0].iov_len) {
            // 重新设置iov_[1]的数据
            iov_[1].iov_base = (uint8_t*) iov_[1].iov_base + (len - iov_[0].iov_len);
            iov_[1].iov_len -= (len - iov_[0].iov_len);
            // 如果写缓存区的数据还没有清空，清空写缓存区
            if(iov_[0].iov_len) {
                writeBuff_.RetrieveAll();
                iov_[0].iov_len = 0;//设为0
            }
        }
        // 如果传输没有结束并且iov_[0]还没有传输完成
        else {
            // 重新设置iov_[0]的数据
            iov_[0].iov_base = (uint8_t*)iov_[0].iov_base + len; 
            iov_[0].iov_len -= len; 
            // 清空写缓存区的前len数据
            writeBuff_.Retrieve(len);
        }
    } while(isET || ToWriteBytes() > 10240);
    return len;
}
```

当处理HTTP请求时：

1、扫描读缓存区，如果读缓存区不存在数据，返回false

2、尝试将读缓存区的数据解析为HTTP请求

3、在写缓存区写入HTTP响应的头部数据

4、将HTTP响应的头部数据和文件数据映射为2个IO向量

```c++
bool HttpConn::process() {
    request_.Init();//初始化HTTP请求
    // 如果可读缓存的数据为0，直接返回
    if(readBuff_.ReadableBytes() <= 0) {
        return false;
    }
    // 解析可读缓存中的数据
    else if(request_.parse(readBuff_)) {
        LOG_DEBUG("%s", request_.path().c_str());//解析成功后的初始化HTTP应答
        //初始化一个成功的HTTP响应
        response_.Init(srcDir, request_.path(), request_.IsKeepAlive(), 200);
    } else {
        //初始化一个失败的HTTP响应
        response_.Init(srcDir, request_.path(), false, 400);//解析失败的初始化HTTP应答
    }
	// 制作HTTP响应
    response_.MakeResponse(writeBuff_);//制作HTTP应答
    /* 将用户空间的响应头数据放入第一个IO向量 */
    iov_[0].iov_base = const_cast<char*>(writeBuff_.Peek());
    iov_[0].iov_len = writeBuff_.ReadableBytes();
    iovCnt_ = 1;
    // 将内核空间的响应文件数据放入第二个IO向量*/
    if(response_.FileLen() > 0  && response_.File()) {
        iov_[1].iov_base = response_.File();// 数据指针
        iov_[1].iov_len = response_.FileLen();// 数据长度
        iovCnt_ = 2;
    }
    LOG_DEBUG("filesize:%d, %d  to %d", response_.FileLen() , iovCnt_, ToWriteBytes());
    return true;
}
```

write和writev

read和write

# 五、缓存区的实现

缓存区Buffer的需求：

读和写的速率不一致，起个缓冲作用

一块连续的地址空间，有一部分专门用来写，有一部分专门用来读

可以向其中添加一整块数据、清除一块数据

## 1、缓存区的主要数据结构

```C++
std::vector<char> buffer_;
std::atomic<std::size_t> readPos_;// 可读位置
std::atomic<std::size_t> writePos_;// 可写位置
```

## 2、缓存区的初始化

```c++
Buffer::Buffer(int initBuffSize) : buffer_(initBuffSize), readPos_(0), writePos_(0) {}
```

## 3、缓存区的可写区间和可读区间

缓存区的可写区间是`[writePos_,len-1]`，可读区间是`[readPos_ ,writePos_-1]`，已读区间是`[0,readPos_-1]`

```C++
// 获取此时缓存区可写入的字节数（可写区间的长度）
size_t Buffer::WritableBytes() const {
    return buffer_.size() - writePos_;
}
// 获取此时缓存区可读入的字节数（可读区间的长度）
size_t Buffer::ReadableBytes() const {
    return writePos_ - readPos_;
}
// 获取此时缓存区预备的字节数（已读区间的长度）
size_t Buffer::PrependableBytes() const {
    return readPos_;
}
// 获取已读区间的第一个字符所在地址
char* Buffer::BeginPtr_() {
    return &*buffer_.begin();// 取第一个迭代器指向字符的地址
}
// 获取可读区间的第一个字符所在地址
const char* Buffer::Peek() const {
    return BeginPtr_() + readPos_;
}
```

假如写入`len_`的长度，确保`len_`的数据可以被写入：

```C++
// 确保len的长度可写入
void Buffer::EnsureWriteable(size_t len) {
    // 如果可写区间的长度小于len
    if(WritableBytes() < len) {
        MakeSpace_(len);
    }
    assert(WritableBytes() >= len);// 安全检查
}
void Buffer::MakeSpace_(size_t len) {
    // 如果可写区间+已读区间的总长都不够
    if(WritableBytes() + PrependableBytes() < len) {
        buffer_.resize(writePos_ + len + 1);//扩长可写区间
    } 
    // 如果可写区间+已读区间的总长够
    else {
        size_t readable = ReadableBytes();// 获取待读区间的长度
        // 将可读区间的数据拷贝到已读区间
        std::copy(BeginPtr_() + readPos_, BeginPtr_() + writePos_, BeginPtr_());
        readPos_ = 0;// 待读位置归零（已读区间为空）
        writePos_ = readPos_ + readable;// 可写位置调整
        assert(readable == ReadableBytes());// 安全检查，如果没有相等，说明拷贝出错
    }
}
```

## 4、向缓存区添加数据

```C++
void Buffer::Append(const std::string& str) {
    Append(str.data(), str.length());
}
void Buffer::Append(const char* str, size_t len) {
    assert(str);// 安全检查，如果str不存在，报错
    EnsureWriteable(len);// 确保复制前有足够的内容空间
    std::copy(str, str+len, BeginWrite());//将str的[0,len-1]数据拷贝到可写区间
    writePos_ += len;// 调整可写位置
}
```

## 5、从缓存区读取数据

```c++
void Buffer::Retrieve(size_t len) {
    assert(len <= ReadableBytes());// 安全检查，本次读取的数据长度小于可读区间
    readPos_ += len;// 调整可读位置
}
// 恢复所有缓存区
void Buffer::RetrieveAll() {
    bzero(&buffer_[0], buffer_.size());
    readPos_ = 0;
    writePos_ = 0;
}
```

将数据

```c++
// 将数据从fd的缓存区读取到buff和buffer_所在的缓存区
ssize_t Buffer::ReadFd(int fd, int* saveErrno) {
    char buff[65535];// 额外缓存区
    struct iovec iov[2];// 2个IO向量
    const size_t writable = WritableBytes();
    /* 分散读， 保证数据全部读完 */
    iov[0].iov_base = BeginPtr_() + writePos_;// 读取可读区间
    iov[0].iov_len = writable;
    iov[1].iov_base = buff;// 额外缓存区
    iov[1].iov_len = sizeof(buff);
    const ssize_t len = readv(fd, iov, 2);//读fd到iov
    // 如果填充出错
    if(len < 0) {
        *saveErrno = errno;
    }
	// 如果iov[0]没有填满，调整可写位置
    else if(static_cast<size_t>(len) <= writable) {
        writePos_ += len;
    }
    // 如果iov[0]已经填满，调整可写位置
    else {
        writePos_ = buffer_.size();//可写区间已满
        Append(buff, len - writable);// 向缓存区添加额外缓存区填充的数据
    }
    return len;
}
```

缓存区的数据：



ToWriteBytes()



分散读和集中写：

