# pattern

匹配请求 URL 的表达式，支持域名、路径、通配符、正则等匹配方式


## 请求 URL

请求 URL 有三种类型：

1. **隧道代理：** `tunnel://domain:port`
    > 示例：`tunnel://www.test.com:443`
2. **WebSocket：** `ws[s]://domain[:port]/[path/to[?query]]`
    > 示例：`wss://www.test.com/path?a=1&b=2`
3. **普通 HTTP/HTTPS：** `http[s]://domain[:port]/[path/to[?query]]`
    > 示例：`https://www.test.com/path?a=1&b=2`

## 域名匹配
1. 正常域名：
   - `www.example.com`、
   - `1.2.3.4`
   - `//www.example.com`、
   - `//1.2.3.4`
      > IP 也可作为域名
2. 带端口域名：
   - `www.example.com:8080`、
   - `//www.example.com:8080`、
3. 带协议域名：
   - `tunnel://www.example.com[:port]`
   - `ws[s]://www.example.com[:port]`
   - `http[s]://www.example.com[:port]`

## 路径匹配
1. 无协议路径：
   - `www.example.com[:port]/[path/to[?query]]`
   - `//www.example.com[:port]/[path/to[?query]]`
2. 带协议路径：
   - `ws[s]://www.example.com[:port]/[path/to[?query]]`
   - `http[s]://www.example.com[:port]/[path/to[?query]]`
      > TUNNEL 请求没有路径

## 通配符匹配

### 域名通配符规则

1. **基础通配符**：`*.example.com[:port][/path][?query]`
    > `*` 匹配任意非分隔符字符（正则：`/[^./?]*/`）
    > 
    > 示例：`api.example.com`、`shop.example.com:8080`
2. **多级通配符**：`**.example.com[:port][/path][?query]`
    > `**` 匹配任意多级子域（正则：`/[^/?]*/`）
    > 
    > 示例：`a.b.example.com`、`x.y.z.example.com/path`
3. **混合通配符**：`test.abc**.com[:port][/path][?query]`
    > `**` 固定前缀 + 多级通配（正则：`/[^/?]*/`）
    > 
    > 示例：`test.abc123.com`、`test.abc123.x.com`、`test.abc.a.b.com`
4. **协议通配符**：`http*://test.abc**.com[:port][/path][?query]`
    > 协议中的 `*` 匹配任意字母或冒号（正则：`/[a-z:]*/`）
    > 
    > 示例：`https://...`、`http://...`
5. **特殊规则**：`***.example.com[:port][/path][?query]`
   > 相当于同时匹配：根域名（example.com）+ 多级子域（**.example.com）
   > 
   > 示例：example.com、a.example.com、a.b.example.com/path?q=1

除上述特殊规则（`***.`）外，协议或域名部分出现3个及以上连续的星号（如或 `***`、`****`）时，其功能等同于2个星号（`**`）

### 路径通配符
由于 `*` 是合法的 URL 路径字符，当需要将其作为通配符使用时，在表达式前面加 `^` 显式声明：

``` txt
^[[schema:]//]domain[:port]/pa**th?qu*ery
```
> 示例：`^http*://**.example.com/data/*/result?q=*23`

协议和域名的通配符功能同上面的域名通配符规则，纯路径及请求参数的通配符规则如下：

##### 路径部分的通配符

| 通配符 | 正则等价   | 匹配范围                    | 示例匹配                                 |
| ------ | ---------- | --------------------------- | ----------------------------------- |
| `*`    | `/[^?/]*/` | 单级路径（不含 `/` 和 `?`） | `^.../*/*.js` -> `.../a/b.js`          |
| `**`   | `/[^?]*/`  | 多级路径（不含 `?`）        | `^.../**file` -> `.../a/b/c/test-file` |
| `***`  | `/.*/`     | 任意字符（含 `/` 和 `?`）   | `^.../data/***file` -> `.../a/b/c?test=file` |



##### 请求参数部分的通配符

| 通配符 | 正则等价  | 匹配范围             | 示例匹配            |
| ------ | --------- | -------------------- | ------------------- |
| `*`    | `/[^&]*/` | 单参数值（不含 `&`） | `^...?q=*123` -> `...?q=abc123`  |
| `**`   | `/.*/`    | 任意字符（含 `&`）   | `^...?q=**123` -> `...?q=abc&test=123` |

## 正则匹配
除简单匹配规则外，Whistle 提供完整的正则表达式支持，语法与 JavaScript 正则完全兼容：
``` txt
/pattern/[flags]
```

- pattern：正则表达式主体
- flags：匹配模式修饰符（可选）支持：
    - `i`	忽略大小写 `/abc/i` 匹配 "AbC"
    - `u`	启用 Unicode 支持	`/\p{Emoji}/u` 匹配 "😀"

示例：
``` txt
/\.test\./          # 匹配 ".test."
/key=value/i        # 忽略大小写匹配 "key=value"
/\/statics\//ui     # Unicode 模式匹配 "/statics/"
```

## 子匹配传值

在 Whistle 的规则配置中，可以通过 $0、$1 至 $9 引用通配符或正则表达式匹配的子匹配内容，并将其传递到操作值中：

``` txt
pattern protocol://$0_$1_$2_..._$1
```

- **$0**：完整匹配结果
- **$1 - $9**：对应捕获组的内容

#### 通配符匹配传值
``` txt
^http://*.example.com/v0/users/** file:///User/xxx/$1/$2
```

- **匹配**：`http://www.example.com/v2/users/alice/test.html?q=1`
- **传值**：
  - `$1` = `www`
  - `$2` = `users/alice`
- 结果：替换本地文件 `/User/xxx/www/alice/test.html` 的内容

#### 正则匹配传值
``` txt
/regexp\/(user|admin)\/(\d+)/ reqHeaders://X-Type=$1&X-ID=$2
```

- **匹配**：`.../regexp/admin/123`
- **传值**：
  - `$1` = `admin`
  - `$2` = `123`
- **结果**：添加请求头 `X-Type: admin` 和 `X-ID: 123`


