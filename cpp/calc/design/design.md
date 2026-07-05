 C++多功能计算器后端设计文档
 —— 暨本人程序设计思路的记录与分享
> 本文档作者、著作权人、版权所有人江睿涛，旨在从一个程序设计师的角度，阐述该项目整个后端系统的设计思路，构建一个高性能、模块化、跨平台的Web服务后端。

---

 目录
1. [总体设计思路](#1-总体设计思路)
2. [架构设计](#2-架构设计)
3. [跨平台抽象层设计](#3-跨平台抽象层设计)
4. [网络通信层设计](#4-网络通信层设计)
5. [数据序列化层设计](#5-数据序列化层设计)
6. [业务逻辑层设计](#6-业务逻辑层设计)
7. [API接口层设计](#7-api接口层设计)
8. [模块间交互设计](#8-模块间交互设计)
9. [内存管理策略](#9-内存管理策略)
10. [错误处理策略](#10-错误处理策略)
11. [编译系统设计](#11-编译系统设计)
12. [设计决策记录](#12-设计决策记录)

---

 1. 总体设计思路
 1.1 设计哲学
人的需求思维是设计的起点。一个程序设计师在构思系统时，不会从代码开始，而是从问题域开始。面对"做一个多功能计算器的Web后端"这个需求，我的思维路径如下：
需求：跨平台Web计算器后端
  |
  +-> 核心问题1：如何让前端访问后端？ -> 需要HTTP服务器
  +-> 核心问题2：前端和后端用什么格式通信？ -> JSON
  +-> 核心问题3：后端要在哪些系统上跑？ -> Windows + Linux
  +-> 核心问题4：需要哪些计算功能？ -> 表达式/进制/排列组合/几何/高级
  +-> 核心问题5：前端文件怎么给浏览器？ -> 静态文件服务
  +-> 核心问题6：代码怎么组织？ -> 模块化，每个功能一个模块

基于这个思维路径，整个系统被划分为6个层次：
层次	职责	对应源码文件
L1 跨平台抽象层	抹平OS差异	platform.hpp/cpp
L2 数据序列化层	JSON解析/生成	json.hpp/cpp
L3 网络通信层	HTTP服务器	http_server.hpp/cpp
L4 业务逻辑层	所有计算引擎	calc_engine, base_converter, permutation, geometry, advanced
L5 API接口层	路由分发	api_handlers.hpp/cpp
L6 入口层	程序启动	main.cpp
 1.2 为什么不用第三方库
这是第一个关键设计决策。成熟的HTTP服务器有Boost.Beast/Asio，JSON有nlohmann/json，但本系统选择从零实现。原因如下：
1. 可控性：不依赖外部意味着编译时只需g++即可，无版本冲突
2. 学习价值：每个组件都展示了底层原理（HTTP协议、JSON语法、Socket编程）
3. 体积控制：不需要链接任何.so/.dll，最终二进制仅~350KB
4. 跨平台简化：只需处理g++标准编译，无需管理第三方库的跨平台编译
5. 完整性：无依赖意味着"拿到源码就能编译运行"
 1.3 设计原则
原则	含义	实例
单一职责（SRP）	每个文件只做一件事	platform只处理OS差异，json只处理JSON，不交叉
最小接口	只暴露必要接口	CalcEngine::evaluate()是唯一的公共接口
错误即异常	所有错误通过异常传播	DivisionByZeroException, OverflowException
RAII（资源获取即初始化）	资源与对象生命周期绑定	NetworkInit自动初始化和清理Winsock
零开销抽象	抽象不带来运行时开销	socket_t类型别名编译后无任何额外开销

---

 2. 架构设计
 2.1 系统架构图
                    客户端浏览器 / curl / 任意HTTP客户端
                           |
                    +------v------+
                    | HTTP Request |
                    +------+------+
                           |
         +-----------------+-----------------+
         |                                   |
    动态路由 (API)                     静态路由 (前端文件)
         |                                   |
   +-----v------+                    +------v------+
   | /api/health |                    | /index.html  |
   | /api/calculate                   | /style.css   |
   | /api/convert/*                   | /app.js      |
   | /api/permutation |               +------+------+
   | /api/geometry  |                        |
   | /api/matrix     |                        |
   | /api/numtheory  |                        |
   | /api/statistics |                        |
   | /api/equation/* +------+               |
   +-----+------+           |               |
         |                  |               |
   +-----v------------------v-----------+   |
   |          api_handlers              |   |
   |  (请求解析 -> 参数验证 -> 计算     |   |
   |   -> 结果封装 -> 响应序列化)        |   |
   +-----+------------------+-----------+   |
         |                  |               |
   +-----v------+    +------v------+  +----v----+
   |calc_engine |    |base_converter|  |http_server|
   |permutation |    |geometry      |  |(静态服务) |
   |advanced    |    +------+------+  +----+----+
   +------+-----+           |              |
          |                  |              |
   +------v------------------v--------------v--+
   |              json (序列化)                |
   +------+-----------------------------------+
          |
   +------v-----------------------------------+
   |              platform (跨平台)            |
   +------------------------------------------+

 2.2 数据流
一次完整的API调用数据流：
Client Request
  -> [platform] socket读取原始HTTP字节
  -> [http_server] 解析为HttpRequest对象 (请求行/头部/体)
  -> [http_server] Router::handle() 查找匹配路由
  -> [api_handlers] 解析JSON请求体 -> 参数验证
  -> [calc_engine/base_converter/...] 执行计算
  -> [api_handlers] 封装结果为统一JSON格式
  -> [json] 序列化为JSON字符串
  -> [http_server] 构建HttpResponse (状态码/头部/体)
  -> [http_server] 添加CORS头部
  -> [platform] socket发送响应字节
  -> Client

 2.3 为什么是这个架构
这个架构遵循了分层通信的直觉：
- 人想到"Web服务"，首先想到"浏览器发请求"
- 请求是HTTP格式的，所以需要一个HTTP解析层
- HTTP体里是JSON，所以需要一个JSON层
- JSON里的数据要交给计算引擎，所以需要一个业务层
- 计算结果要返回，所以需要一个API封装层
- 所有这些要跑在不同系统上，所以需要一个跨平台层
每一层只与相邻层交互，不跳层。这是人脑处理复杂系统的自然方式——分层抽象。

---

 3. 跨平台抽象层设计
 3.1 问题分析
Web服务需要Socket编程，但Windows和Linux的Socket API完全不同：
功能	Windows (Winsock)	Linux (POSIX)
头文件	<winsock2.h>	<sys/socket.h>
socket类型	SOCKET (64位无符号)	int (32位有符号)
无效socket	INVALID_SOCKET	-1
关闭	closesocket()	close()
错误码	WSAGetLastError()	errno
初始化	WSAStartup()	无需
非阻塞	ioctlsocket()	fcntl(F_SETFL, O_NONBLOCK)
 3.2 设计思路
人的思维是："这些本质上是同一件事，只是API名字不同。那我就定义一个统一的类型和一组统一的函数。"
三步抽象：
1. 类型统一：using socket_t = SOCKET (Win) / int (Linux)
2. 常量统一：INVALID_SOCKET_VAL, SOCKET_ERROR_VAL
3. 操作统一：socket_ops::close_socket(), socket_ops::set_nonblocking(), socket_ops::get_last_error()
RAII包装：NetworkInit类在构造时调用WSAStartup()，在析构时调用WSACleanup()。这样永远不会忘记清理。
 3.3 关键代码模式
// 条件编译自动选择正确的头文件和实现
#ifdef _WIN32
    // Windows实现
#else
    // Linux实现
#endif

这是跨平台C++的标准模式。关键是将条件编译集中在platform一个文件中，其他文件不需要关心平台差异。

---

 4. 网络通信层设计
 4.1 HTTP服务器设计
 4.1.1 为什么用select而非epoll/kqueue/IOCP
本系统选择select()模型，理由如下：
1. 简单：select是POSIX标准，所有平台都支持，代码量少
2. 足够：这是一个计算器服务，不是高并发Web服务，QPS预期<1000
3. 可移植：epoll(Linux)、kqueue(macOS)、IOCP(Windows)各自专属，select是通用的
4. 教学价值：select模型最能清晰展示IO多路复用的原理
 4.1.2 事件循环
while (running) {
    select(read_fds, timeout=100ms)  // 等待可读的socket
    if (server_socket readable) {
        accept() new connection       // 有新连接到来
        handle_client()               // 同步处理请求
        close() connection            // 关闭连接
    }
}

这是Reactor模式的最简实现。select监听服务器socket的可读事件，当有新连接时accept并处理。
 4.1.3 同步处理 vs 异步处理
本系统采用同步处理（每个请求在一个函数调用中完成），而非多线程/线程池。理由：
1. 计算请求都是CPU密集型短任务（毫秒级完成）
2. 无阻塞IO操作（不需要等数据库/文件系统）
3. 单线程避免了锁竞争和线程切换开销
4. 代码更简单，无并发bug风险
 4.2 HTTP协议实现
 4.2.1 请求解析
HTTP请求格式：
GET /api/calculate HTTP/1.1\r\n      <- 请求行 (方法 + 路径 + 版本)
Host: localhost\r\n                   <- 头部
Content-Type: application/json\r\n  <- 头部
Content-Length: 42\r\n              <- 头部
\r\n                                  <- 空行分隔
{"expression": "2 + 3 * 4"}       <- 请求体

解析分三步：
1. 请求行：GET /api/calculate HTTP/1.1 -> 提取method, path, version
2. 头部：逐行解析Key: Value，转为小写键名便于查找
3. 请求体：根据Content-Length读取指定字节数
 4.2.2 路由系统
人的思维是："收到请求后，看路径是什么，然后交给对应的处理器。"
路由表结构：{方法} -> {路径 -> 处理函数}
GET:
  "/api/health" -> health_handler
  "/api/convert/bases" -> list_bases_handler
POST:
  "/api/calculate" -> calculate_handler
  "/api/power" -> power_handler
  ...

 4.2.3 静态文件服务
人的思维是："前端文件放在某个目录里，当浏览器请求/index.html时，就去那个目录找到文件返回。"
关键点：
- 路径映射：URL /index.html -> 磁盘文件 ./frontend/dist/index.html
- SPA Fallback：React/Vue等单页应用刷新时，浏览器请求 /calculator，但磁盘上只有 index.html。如果文件不存在且路径无扩展名，返回 index.html。
- MIME类型：根据文件后缀判断Content-Type（.html -> text/html, .css -> text/css, .js -> application/javascript）
- 安全防护：拒绝包含 .. 的路径，防止目录遍历攻击

---

 5. 数据序列化层设计
 5.1 为什么不用nlohmann/json
nlohmann/json是C++最好的JSON库，但它：
- 需要下载和包含一个很大的头文件（~25000行）
- 编译时间显著增加
- 本系统的JSON需求并不复杂（简单的对象/数组/数值）
 5.2 手动JSON实现
 5.2.1 类型系统
JSON有7种类型：null, boolean, number, string, array, object。用枚举表示：
enum class JsonType { Null, Boolean, Number, String, Array, Object };

JsonValue类用type_字段标记当前存储的实际类型，用union风格的多个成员存储值。
 5.2.2 解析器（递归下降）
JSON语法可以用BNF描述：
value   -> null | true | false | number | string | array | object
array   -> "[" [value ("," value)*] "]"
object  -> "{" [string ":" value ("," string ":" value)*] "}"

递归下降解析器按照语法规则递归调用：
- parse_value() -> 根据首字符决定调用哪个解析函数
- parse_string() -> 处理转义序列（\n, \t, ", \, \uXXXX）
- parse_number() -> 处理整数/小数/科学计数法
- parse_array() -> 循环解析逗号分隔的value
- parse_object() -> 循环解析键值对
 5.2.3 序列化器
to_string()方法根据类型递归生成JSON字符串：
- Null -> "null"
- Boolean -> "true"/"false"
- Number -> 直接转字符串（整数无小数点，浮点智能截尾）
- String -> 加引号，转义特殊字符
- Array -> [elem1, elem2, ...]
- Object -> {"key1":val1, "key2":val2}
 5.3 统一响应格式
所有API响应使用统一格式，通过工具函数生成：
成功: { "success": true, "data": { ... } }
失败: { "success": false, "error": "错误描述" }

这样前端只需要检查 success 字段即可判断请求是否成功。

---

 6. 业务逻辑层设计
 6.1 表达式求值引擎
 6.1.1 算法选择：调度场算法（Shunting Yard）
人的思维过程：计算 2 + 3 * 4 时，人知道先算 3 * 4 再算加法。这是因为乘法优先级高于加法。
计算机模拟这个过程的经典算法是调度场算法（由Dijkstra发明），分三步：
Step 1 - 词法分析（Tokenizer）：
将字符串拆成token：["2", "+", "3", "*", "4"]
Step 2 - 中缀转后缀（Infix to Postfix）：
2 + 3 * 4 -> 2 3 4 * +
后缀表达式的好处是不需要括号就能确定计算顺序。
Step 3 - 后缀求值：
用一个栈依次处理token：
- 遇到数字：压入栈
- 遇到运算符：弹出两个操作数，计算，结果压入栈
- 最后栈中只剩一个值，就是结果
 6.1.2 优先级表
优先级	运算符	说明
4	_ (一元负号)	如 -5
3	^	幂运算，右结合
2	*, /	乘除
1	+, -	加减
 6.1.3 函数支持
内置函数（sin, cos, tan, sqrt, log等）在词法分析阶段识别为 TokenType::FUNCTION，在求值阶段直接调用对应的C++标准库函数。
 6.2 进制转换引擎
 6.2.1 核心算法
进制转换的核心思想：所有进制都可以先转为十进制，再从十进制转到目标进制。
N进制 -> 十进制：按位展开求和
"FF" (16进制) = 15*16^1 + 15*16^0 = 240 + 15 = 255

十进制 -> N进制：除基取余（整数部分）+ 乘基取整（小数部分）
255 / 16 = 15 余 15 -> 最低位 F
15 / 16 = 0 余 15  -> 最高位 F
结果: "FF"

 6.3 排列组合引擎
 6.3.1 关键公式
功能	公式	实现策略
排列 A(n,m)	n!/(n-m)!	循环连乘，不先算完整阶乘再除
组合 C(n,m)	n!/(m!*(n-m)!)	利用对称性C(n,m)=C(n,n-m)简化
圆排列	(n-1)!	固定一个元素，其余(n-1)个全排列
错位排列	D(n)=(n-1)*(D(n-1)+D(n-2))	迭代而非递归，避免栈溢出
 6.3.2 溢出处理
阶乘增长极快（170! > double最大值），所以：
- 排列/组合使用连乘边算边除，不先算完整阶乘
- 阶乘限制n <= 170
- 结果用double存储（精确到整数范围约2^53）
 6.4 几何公式引擎
 6.4.1 设计模式：统一接口
所有几何计算使用统一的 GeometryResult 结构返回，包含area（面积）、perimeter（周长）、volume（体积）三个字段。每个图形函数填充对应的字段。
 6.4.2 输入验证
每个函数开头验证参数为正数，使用 validate_positive() 统一处理。例如三角形海伦公式还要验证两边之和大于第三边。
 6.5 高级功能引擎
 6.5.1 矩阵运算
使用行优先的二维数组 std::vector<std::vector<double>> 存储。支持操作：加、减、乘、标量乘、转置、行列式。
行列式使用高斯消元法，时间复杂度O(n^3)。
 6.5.2 单位转换
使用转换因子法：所有单位都先转为基准单位（长度->米，重量->千克），再从基准单位转到目标单位。
温度需要特殊处理（不是线性比例）：
- C -> F: F = C * 9/5 + 32
- C -> K: K = C + 273.15

---

 7. API接口层设计
 7.1 RESTful设计
所有API遵循RESTful风格：
- POST 用于有请求体的操作（计算、转换）
- GET 用于查询类操作（支持的进制列表、单位列表）
- URL路径使用名词而非动词：/api/calculate 而非 /api/doCalculate
 7.2 路由注册
在 register_api_handlers() 函数中集中注册所有路由，使用Lambda表达式作为处理函数。这种设计的好处是：
- 所有路由定义在一个地方，便于查看和维护
- Lambda可以捕获局部变量，无需额外的上下文参数
- 每个Lambda内部结构一致：解析参数 -> 调用引擎 -> 封装响应
 7.3 CORS支持
跨域资源共享（CORS, Cross-Origin Resource Sharing）是前后端分离的必要配置。后端默认在所有响应中添加：
Access-Control-Allow-Origin: *
Access-Control-Allow-Methods: GET, POST, PUT, DELETE, OPTIONS
Access-Control-Allow-Headers: Content-Type, Authorization

这样前端可以部署在任何域名下，不限于后端同一域名。

---

 8. 模块间交互设计
 8.1 依赖关系图
main.cpp
  +-> api_handlers
    +-> http_server
    |   +-> platform
    +-> json
    |   +-> (无依赖)
    +-> calc_engine
    +-> base_converter
    +-> permutation
    +-> geometry
    +-> advanced

所有模块最终都依赖 platform（通过 http_server 间接）
json 是唯一零依赖的模块

 8.2 接口契约
每个模块的接口设计遵循最小暴露原则：
模块	暴露的接口	隐藏的实现
platform	socket_t, NetworkInit, socket_ops::*	OS特定的头文件和宏
json	JsonValue, JsonParser, json_utils::*	内部tokenize/infix_to_postfix
http_server	HttpServer, Router, HttpRequest, HttpResponse	socket读写细节
calc_engine	CalcEngine::evaluate(), CalcEngine::power(), ...	Token结构, 调度场算法
api_handlers	register_api_handlers()	每个Lambda的内部逻辑

---

 9. 内存管理策略
 9.1 零动态分配策略
本系统尽量在栈上分配内存，减少堆分配：
- JsonValue 使用值语义，拷贝时复制数据（数据量小，无性能问题）
- HTTP请求/响应在栈上构建
- 计算中间结果使用栈变量
 9.2 字符串策略
- JSON序列化使用 std::ostringstream 拼接，避免频繁的字符串拼接
- HTTP响应使用 std::ostringstream 一次性构建完整响应
 9.3 实测内存开销
指标	数值
启动后RSS	~900 KB
预热后RSS	~1.7 MB
Heap实际使用	~40 KB
VSZ	~2.6 MB
Swap	0 KB
线程数	1

---

 10. 错误处理策略
 10.1 异常层次
CalcException (基类)
  +-> DivisionByZeroException
  +-> OverflowException
  +-> InvalidInputException
  +-> DomainException

所有计算错误都通过异常传播，由 api_handlers 中的 safe_calc 工具函数统一捕获并转为JSON错误响应。
 10.2 异常 vs 错误码
选择异常而非错误码的原因：
1. 强制性：异常不能被忽略，错误码可以被忽略
2. 传播性：异常自动沿调用栈传播，不需要在每个函数中检查
3. 信息性：异常对象可以携带详细的错误信息
 10.3 前端友好的错误信息
所有错误信息使用中文，因为前端用户是中文使用者。例如：
- "错误：不能除以零"
- "输入错误：阶乘只定义于非负整数"
- "定义域错误：负数没有实数平方根"

---

 11. 编译系统设计
 11.1 为什么提供三种编译方式
方式	适用场景	优势
CMake	大型项目/团队协作	跨平台，IDE支持好
Makefile	快速编译	简单直接
直接g++	最快测试	一条命令
 11.2 Windows特殊处理
Windows需要链接 ws2_32.lib（Winsock库），通过 -lws2_32 指定。同时需要定义几个宏避免Windows头文件的副作用：
- WIN32_LEAN_AND_MEAN：排除不常用的Windows API
- NOMINMAX：禁用Windows的min/max宏，避免与STL冲突

---

 12. 设计决策记录
 ADR-001：纯C++实现，零第三方依赖
- 决策：不使用Boost、nlohmann/json、mongoose等第三方库
- 理由：可控性、学习价值、编译简单、最终体积小
- 影响：需要手动实现HTTP服务器和JSON解析器
 ADR-002：select模型而非epoll/IOCP
- 决策：使用select实现IO多路复用
- 理由：简单、可移植、足够满足计算器服务的并发需求
- 影响：高并发（>1000连接）时性能会下降
 ADR-003：同步处理而非多线程
- 决策：每个请求在主线程同步处理
- 理由：计算任务短、无阻塞IO、避免锁竞争
- 影响：无法利用多核CPU
 ADR-004：异常驱动错误处理
- 决策：使用C++异常而非错误码
- 理由：强制性、传播性、信息性
- 影响：需要理解C++异常机制
 ADR-005：RAII模式管理所有资源
- 决策：所有资源通过类的构造/析构管理
- 理由：防止资源泄漏，异常安全
- 影响：NetworkInit、HttpServer等类使用RAII
 ADR-006：前后端完全分离，CORS默认开启
- 决策：前端和后端独立运行，通过CORS允许跨域
- 理由：部署灵活、开发方便、前端可部署到CDN
- 影响：后端需要处理OPTIONS预检请求
 ADR-007：JSON作为统一数据交换格式
- 决策：所有API请求/响应使用JSON
- 理由：JavaScript原生支持、人类可读、RESTful标准
- 影响：需要手动实现JSON解析/生成
 ADR-008：统一API响应格式
- 决策：所有响应包含 {success, data/error} 结构
- 理由：前端处理逻辑统一
- 影响：需要在api_handlers中统一封装
