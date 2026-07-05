 AI编程Skill：C++多功能计算器Web服务端
 概述
本Skill（技能/指令集）作者、著作权人、版权所有人为江睿涛，用自身程序设计思路，指导AI生成一个生产级的跨平台Web服务后端。内容涵盖从架构设计到代码实现的完整技术细节。

---

 一、Skill总则
 1.1 技能目标
使用纯C++17（零第三方库）构建一个跨平台（Windows + Linux）的HTTP Web服务后端，提供计算器相关的RESTful API。
 1.2 技术约束
约束项	要求
C++标准	C++17
第三方库	零（不使用Boost、nlohmann/json、mongoose等）
编译器	g++ (Linux/MinGW) 或 MSVC (Windows)
平台	Windows 7+ / Linux 内核 3.0+
目标二进制大小	< 500KB
内存开销	启动时 < 2MB RSS
 1.3 架构总览
系统分为6层，每层有明确的职责边界：
L1: platform  — 跨平台抽象（Socket、错误处理、网络初始化）
L2: json      — JSON解析/生成
L3: http_server — HTTP协议解析、路由、静态文件服务
L4: calc_engine / base_converter / permutation / geometry / advanced — 业务逻辑
L5: api_handlers — API路由注册与参数封装
L6: main      — 入口与生命周期管理


---

 二、平台层Skill
 2.1 平台检测
使用条件编译 #ifdef _WIN32 区分Windows和Linux。Windows下需定义：
- _WIN32_WINNT=0x0601（目标Windows 7+）
- WIN32_LEAN_AND_MEAN（排除不常用API）
- NOMINMAX（禁用min/max宏，避免与STL冲突）
 2.2 Socket类型统一
定义类型别名统一两个平台的socket类型：
#ifdef _WIN32
    using socket_t = SOCKET;
    constexpr socket_t INVALID_SOCKET_VAL = INVALID_SOCKET;
    constexpr int SOCKET_ERROR_VAL = SOCKET_ERROR;
#else
    using socket_t = int;
    constexpr socket_t INVALID_SOCKET_VAL = -1;
    constexpr int SOCKET_ERROR_VAL = -1;
#endif

 2.3 网络初始化（RAII）
Windows需要WSAStartup，Linux无需操作。使用RAII类封装：
class NetworkInit {
public:
    NetworkInit() {
        // Windows: WSAStartup(MAKEWORD(2, 2), &wsaData)
        // Linux: 空操作
    }
    ~NetworkInit() {
        // Windows: WSACleanup()
        // Linux: 空操作
    }
    // 禁止拷贝和移动
};

 2.4 Socket操作函数集
命名空间 socket_ops 提供：
- close_socket(socket_t) — 关闭socket（Windows用closesocket，Linux用close）
- set_nonblocking(socket_t) — 设置非阻塞模式（Windows用ioctlsocket，Linux用fcntl）
- get_last_error() — 获取最后错误描述（Windows用WSAGetLastError+FormatMessage，Linux用strerror(errno)）
- get_platform_name() — 返回"Windows"或"Linux"

---

 三、JSON层Skill
 3.1 类型系统
实现完整的JSON类型枚举和值容器：
enum class JsonType { Null, Boolean, Number, String, Array, Object };

JsonValue 类需要：
- 存储当前值的类型标记 type_
- 用独立成员存储各类值：bool_value_, number_value_, string_value_, array_value_(vector), object_value_(map<string, JsonValue>)
- 提供构造函数重载（bool, int, double, string, vector, map）
- 提供类型查询方法：is_null(), is_number(), is_string() 等
- 提供类型安全取值方法：as_bool(), as_int(), as_double(), as_string(), as_array(), as_object()
- 提供 operator[] 重载（支持string键和size_t索引）
- 提供 to_string() 序列化方法（支持pretty格式化）
 3.2 解析器（递归下降）
实现 JsonParser 类，使用递归下降算法：
核心方法签名：
static JsonValue parse(const std::string& json);  // 入口
void skip_whitespace();                             // 跳过空白
Token parse_value();                                // 根据首字符分发
Token parse_null();                                 // 匹配 "null"
Token parse_bool();                                 // 匹配 "true"/"false"
Token parse_number();                               // 整数/小数/科学计数法
Token parse_string();                               // 处理转义序列
Token parse_array();                                // [value, ...]
Token parse_object();                               // {"key": value, ...}

字符串转义处理： " -> \", \ -> \\, / -> \/, \b, \f, \n, \r, \t, \uXXXX (Unicode)
数字格式：支持整数（123）、小数（3.14）、负号（-5）、科学计数法（1e10, 2.5E-3）
 3.3 序列化器
to_string(bool pretty = false) 方法：
- Null -> "null"
- Boolean -> "true" / "false"
- Number -> 智能格式化（整数无小数点，浮点去除末尾零）
- String -> 加引号 + 转义
- Array -> [elem1, elem2]（pretty模式带换行缩进）
- Object -> {"key1":val1, "key2":val2}（pretty模式带换行缩进）
 3.4 工具函数
JsonValue make_success(const JsonValue& data);   // { "success": true, "data": ... }
JsonValue make_error(const std::string& msg);      // { "success": false, "error": ... }


---

 四、HTTP服务器层Skill
 4.1 HTTP请求解析
HttpRequest 类从原始HTTP字符串解析：
请求行格式：METHOD PATH HTTP/VERSION
- 示例：GET /api/calculate?key=val HTTP/1.1
- 解析出：method(GET/POST/PUT/DELETE/OPTIONS), path, version
- 从path中分离出查询参数（?key1=val1&key2=val2）
- 对path和查询参数进行URL解码（%20 -> 空格, + -> 空格）
头部解析：逐行解析 Key: Value，键名转为小写便于查找。
请求体读取：根据 Content-Length 头部读取指定字节数。
 4.2 HTTP响应构建
HttpResponse 类：
- status(code) 设置状态码（200/201/400/404/405/500）
- header(name, value) 添加响应头
- set_body(content, content_type) 设置响应体
- json(jsonValue) 快捷设置JSON响应
- text(text) 快捷设置文本响应
- html(html) 快捷设置HTML响应
- enable_cors() 添加CORS跨域头部（Access-Control-Allow-Origin: *）
- to_string() 序列化为完整HTTP响应字符串（含Content-Length）
 4.3 路由系统
Router 类：
- 路由表结构：map<method, map<path, handler_func>>
- 注册方法：get(path, handler), post(path, handler), put(path, handler), del(path, handler), options(path, handler)
- serve_static(url_prefix, dir_path) 注册静态文件服务目录
- handle(req, res) 查找并执行匹配的处理函数，未匹配返回404
静态文件服务逻辑：
1. URL路径以注册的prefix开头？
2. 去除prefix，拼接目录路径得到文件路径
3. 安全检查：拒绝包含 .. 的路径
4. 读取文件内容
5. 根据文件后缀推断MIME类型
6. SPA Fallback：如果文件不存在且路径无扩展名（可能是前端路由），返回index.html
7. 排除API路径：SPA Fallback不应用于 /api/ 开头的路径
 4.4 HTTP服务器主循环
HttpServer 类：
启动流程：
1. socket(AF_INET, SOCK_STREAM, 0) 创建TCP socket
2. setsockopt(SO_REUSEADDR) 允许地址重用
3. bind() 绑定IP和端口
4. listen(10) 开始监听，backlog=10
5. select() 事件循环，timeout=100ms
请求处理流程：
1. accept() 接受新连接
2. recv() 读取完整HTTP请求（读到 \r\n\r\n 后再根据Content-Length读body）
3. HttpRequest::parse() 解析请求
4. Router::handle() 路由分发
5. HttpResponse::to_string() 序列化响应
6. send() 发送响应
7. close_socket() 关闭连接
OPTIONS预检处理：收到OPTIONS请求直接返回200，带CORS头部，这是浏览器跨域预检所需。

---

 五、计算引擎层Skill
 5.1 表达式求值引擎
算法：调度场算法（Shunting Yard）—— 中缀表达式转后缀表达式，再用栈求值。
词法分析：将字符串拆分为token列表
- 数字：123, 3.14, 1e10（支持小数和科学计数法）
- 运算符：+, -, *, /, ^
- 括号：(, )
- 函数名：sin, cos, tan, sqrt, cbrt, log, log10, ln, abs, floor, ceil, round
- 常量：pi (3.14159...), e (2.71828...)
- 逗号：,（函数参数分隔）
中缀转后缀规则：
- 遇到数字：直接输出到后缀队列
- 遇到左括号：压入运算符栈
- 遇到右括号：弹出栈顶到输出，直到遇到左括号
- 遇到运算符：当栈顶运算符优先级 >= 当前运算符时，弹出栈顶到输出，然后当前运算符入栈
- 幂运算 ^ 是右结合（不同于加减乘除的左结合）
- 一元正负号处理：当 + 或 - 出现在表达式开头或另一个运算符后，视为一元运算符
后缀求值：使用 std::stack<double>
- 数字token：压栈
- 运算符token：弹出两个操作数，计算，结果压栈
- 函数token：弹出一个参数，计算函数值，结果压栈
运算符优先级（数值越大优先级越高）：
4: _ (一元负号)
3: ^ (幂运算, 右结合)
2: *, /
1: +, -

错误处理：
- DivisionByZeroException：除数为0
- OverflowException：结果超出double范围
- InvalidInputException：非法字符、括号不匹配、操作数不足
- DomainException：负数开偶次方、对数的真数<=0
 5.2 进制转换引擎
N进制 -> 十进制：逐位转换
result = result * base + digit_value;

十进制 -> N进制：
- 整数部分：除基取余，逆序排列
- 小数部分：乘基取整，顺序排列（最多10位精度）
字符转换：0-9 -> 0-9, A-Z/a-z -> 10-35
验证：is_valid_number() 检查字符串中每个字符是否都在目标进制的有效范围内。
 5.3 排列组合引擎
排列 A(n,m)：直接连乘 n * (n-1) * ... * (n-m+1)，不先算完整阶乘。
组合 C(n,m)：利用对称性 C(n,m) = C(n,n-m) 选择较小的m计算；连乘边乘边除。
阶乘 n!：循环乘法，限制 n <= 170（171! > 1.8e308）。
错位排列 D(n)：递推公式 D(n) = (n-1) * (D(n-1) + D(n-2))，用迭代而非递归。
 5.4 几何公式引擎
每个图形函数接收参数，返回 GeometryResult 结构：
struct GeometryResult {
    double area = 0;       // 面积/表面积
    double perimeter = 0;  // 周长
    double volume = 0;     // 体积
    std::string formula;   // 使用的公式说明
    std::string notes;     // 额外说明
};

输入验证：所有长度参数必须为正数。三角形海伦公式额外验证三边满足三角不等式。
 5.5 高级功能引擎
矩阵：使用 std::vector<std::vector<double>> 存储。行列式使用高斯消元法O(n^3)。
单位转换：转换因子法。温度转换需要特殊处理（非线性）。
数论：GCD用欧几里得算法，LCM用 |a*b|/gcd(a,b)，质数判断用试除到sqrt(n)。
统计：标准算法（均值=sum/n，方差=sum((x-mean)^2)/n）。
方程：一元二次方程求根公式，判别式<0时返回复数根。

---

 六、API接口层Skill
 6.1 路由注册模式
在 register_api_handlers() 函数中集中注册所有路由。每个处理器使用Lambda，遵循统一模式：
router.post("/api/xxx", [](const HttpRequest& req, HttpResponse& res) {
    try {
        // 1. 解析JSON请求体
        JsonValue body = req.json_body();
        // 2. 提取参数
        double param = get_double(body, "key");
        // 3. 参数验证
        if (param <= 0) throw InvalidInputException("参数必须为正数");
        // 4. 调用计算引擎
        double result = CalcEngine::xxx(param);
        // 5. 封装响应
        JsonValue data(JsonObjectMap());
        data["result"] = JsonValue(result);
        res.json(json_utils::make_success(data));
    } catch (const CalcException& e) {
        res.json(json_utils::make_error(e.what()));
    }
});

 6.2 参数提取工具函数
double get_double(const JsonValue& json, const std::string& key, double default_val = 0);
int get_int(const JsonValue& json, const std::string& key, int default_val = 0);
std::string get_string(const JsonValue& json, const std::string& key, const std::string& default_val = "");

 6.3 安全计算包装
template<typename Func>
static void safe_calc(Func calc_func, HttpResponse& res) {
    try {
        JsonValue result = calc_func();
        res.json(json_utils::make_success(result));
    } catch (const CalcException& e) {
        res.json(json_utils::make_error(e.what()));
    } catch (const std::exception& e) {
        res.json(json_utils::make_error(std::string("计算错误: ") + e.what()));
    }
}


---

 七、主入口Skill
 7.1 启动流程
int main(int argc, char* argv[]) {
    // 1. 创建NetworkInit（RAII，自动初始化和清理Winsock）
    NetworkInit net_init;
    // 2. 解析命令行参数（端口、静态文件目录）
    int port = (argc > 1) ? atoi(argv[1]) : 8080;
    std::string static_dir = (argc > 2) ? argv[2] : "./frontend/dist";
    // 3. 创建HttpServer
    HttpServer server;
    // 4. 注册所有API路由
    register_api_handlers(server.router(), static_dir);
    // 5. 启动服务器（阻塞）
    server.start(port);
    return 0;
}

 7.2 信号处理
注册 SIGINT (Ctrl+C) 和 SIGTERM 的信号处理器，在收到信号时调用 server.stop() 优雅退出。
 7.3 命令行参数
用法: calc_server [端口] [静态文件目录]

示例:
  ./calc_server 8080 ./frontend/dist    # 监听8080，服务前端文件
  ./calc_server 3000                    # 监听3000，不服务静态文件
  ./calc_server                         # 默认端口8080


---

 八、代码组织Skill
 8.1 文件命名规范
类型	命名	示例
类声明	PascalCase	class HttpServer
函数/方法	snake_case	void handle_client()
变量	snake_case	int port_number
常量/枚举	UPPER_SNAKE	INVALID_SOCKET_VAL
命名空间	小写	namespace calc
宏	大写下划线	#define PLATFORM_WINDOWS
 8.2 注释规范
- 文件头注释：说明文件用途和作者
- 类注释：/** @brief 一句话描述 */
- 函数注释：/** @brief 描述 @param 参数 @return 返回值 @throw 异常 */
- 代码内注释：解释"为什么"而非"做什么"
- 所有注释使用中文
 8.3 异常安全
所有可能失败的操作都应该：
1. 抛出异常而不是返回错误码
2. 使用RAII管理资源（确保异常时资源被释放）
3. 在API层捕获所有异常并转为友好的错误响应

---

 九、调试与测试Skill
 9.1 编译选项
场景	编译命令
开发调试	g++ -std=c++17 -g -O0 -Wall -Wextra
发布优化	g++ -std=c++17 -O2 -Wall -Wextra
Windows	g++ -std=c++17 -O2 -o calc_server.exe *.cpp -lws2_32
 9.2 快速测试
# 启动服务器
./calc_server 8080 ./frontend/dist &

# 健康检查
curl http://localhost:8080/api/health

# 表达式计算
curl -X POST http://localhost:8080/api/calculate \
  -H "Content-Type: application/json" \
  -d '{"expression":"2+3*4"}'

# 进制转换
curl -X POST http://localhost:8080/api/convert/base \
  -H "Content-Type: application/json" \
  -d '{"number":"FF","from_base":16,"to_base":2}'

 9.3 内存检查
使用 /proc/PID/status 查看内存使用情况，关注 VmRSS（实际物理内存）。

---

 十、完整API端点列表
端点	方法	功能	请求体
/api/health	GET	健康检查	无
/api/calculate	POST	表达式计算	{"expression": "2+3*4"}
/api/power	POST	幂运算	{"base": 2, "exponent": 10}
/api/root	POST	开方	{"value": 27, "n": 3}
/api/trig	POST	三角函数	{"function": "sin", "angle": 1.57, "mode": "rad"}
/api/logarithm	POST	对数	{"value": 100, "base": "10"}
/api/convert/base	POST	进制转换	{"number": "FF", "from_base": 16, "to_base": 2}
/api/convert/bases	GET	进制列表	无
/api/permutation	POST	排列组合	{"type": "combination", "n": 10, "m": 3}
/api/geometry	POST	几何公式	{"shape": "circle", "radius": 5}
/api/matrix	POST	矩阵运算	{"operation": "add", "matrix_a": [[1,2],[3,4]], "matrix_b": [[5,6],[7,8]]}
/api/convert/unit	POST	单位转换	{"category": "length", "value": 100, "from": "cm", "to": "m"}
/api/numtheory	POST	数论	{"operation": "gcd", "a": 48, "b": 18}
/api/statistics	POST	统计	{"operation": "mean", "data": [1,2,3,4,5]}
/api/equation/quadratic	POST	二次方程	{"a": 1, "b": -5, "c": 6}
