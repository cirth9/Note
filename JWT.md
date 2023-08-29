---
title: JWT-GO
date: 2023-07-03 16:41:38
tags:
- golang
- jwt
mathjax: true
categories: 
- golang
- jwt
---

### 什么是JWT

> JSON Web Token (JWT)是一个开放标准(RFC 7519) ，它定义了一种紧凑和自包含的方式，**`用于作为 JSON 对象在各方之间安全地传输信息`**。此信息可以进行验证和信任，因为它是经过数字签名的。JWT 可以使用机密(使用 HMAC 算法)或使用 RSA 或 ECDSA 的公钥/私钥对进行签名。
> 虽然可以对 JWT 进行加密，以便在各方之间提供保密性，但是我们将关注已签名的Token。签名Token可以验证其中包含的声明的完整性，而加密Token可以向其他方隐藏这些声明。当使用公钥/私钥对对令牌进行签名时，该签名还证明只有持有私钥的一方才是对其进行签名的一方( **`签名技术是保证传输的信息不被篡改,并不能保证信息传输的安全`** )。

***说人话的话，JWT就是一个是基于`JSON`格式用于网络传输的令牌，通过这个令牌可以去服务器上获取资源，也可以说是客户端和服务器端安全传输信息的一个标准。***



### BASE64

Base64是一种二进制到文本的编码方式。如果要更具体一点的话，可以认为它是一种将 `byte`数组编码为字符串的方法，而且编码出的字符串只包含ASCII基础字符。

例如字符串`ShuSheng007`对应的Base64为`U2h1U2hlbmcwMDc=`。其中那个`=`比较特殊，是填充符，这里仅作了解就好，不多做说明。

值得注意的是Base64不是加密算法，其仅仅是一种编码方式，算法也是公开的，所以不能依赖它进行加密。



编码和加密的区别：

* **编码是将一系列字符放入一种特殊格式以进行传输或存储的过程。**

* **加密是将数据转换成密码的过程**。



至于为什么这里会提到base64，因为jwt本质上就是一个用base64编码的字符串。

### JWT的结构

在其紧凑的形式中，JWT由以点(.)分隔的三个部分组成，它们是:

- Header（首部）
- Payload（负载）
- Signature（签名）

类似于xxxx.xxxx.xxxx格式,真实情况如下:

```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkpvaG4gRG9lIiwiaWF0IjoxNTE2MjM5MDIyfQ.SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c
```

并且你可以通过官网https://jwt.io/#debugger-io解析出三部分表示的信息( **`可使用 JWT.io Debugger 来解码、验证和生成 JWT`** ):
![在这里插入图片描述](https://ucc.alicdn.com/images/user-upload-01/98d3fc61aa364cefa3ec866275569377.png)

#### (1) Header

> 报头通常由两部分组成: Token的类型(即 JWT)和所使用的签名算法(如 HMAC SHA256或 RSA)。

例如:

```
{
  "alg": "HS256",
  "typ": "JWT"
}
```

最终这个 JSON 将由base64进行加密（该加密是可以对称解密的)，用于构成 JWT 的第一部分,eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9就是base64加密后的结果。

在`JWT`规范文件中称这些`Header`为`JOSE Header`，`JOSE`的全称为`Javascript Object Signature Encryption`，也就是`Javascript`对象签名和加密框架，`JOSE Header`其实就是`Javascript`对象签名和加密的头部参数。**`JWS`中常用的`Header`**：

|   简称   |                 全称                 |                             含义                             |
| :------: | :----------------------------------: | :----------------------------------------------------------: |
|   alg    |              Algorithm               |                  用于保护`JWS`的加解密算法                   |
|   jku    |             JWK Set URL              | 一组`JSON`编码的公共密钥的`URL`，其中一个是用于对`JWS`进行数字签名的密钥 |
|   jwk    |             JSON Web Key             |        用于对`JWS`进行数字签名的密钥相对应的公共密钥         |
|   kid    |                Key ID                |                    用于保护`JWS`进的密钥                     |
|   x5u    |              X.509 URL               |                         `X.509`相关                          |
|   x5c    |       X.509 Certificate Chain        |                         `X.509`相关                          |
|   x5t    |  X.509 Certificate SHA-1 Thumbprin   |                         `X.509`相关                          |
| x5t#S256 | X.509 Certificate SHA-256 Thumbprint |                         `X.509`相关                          |
|   typ    |                 Type                 |             类型，例如`JWT`、`JWS`或者`JWE`等等              |
|   cty    |             Content Type             |           内容类型，决定`payload`部分的`MediaType`           |

#### (2) Payload

> Token的第二部分是有效负载，其中包含声明。声明是关于实体(通常是用户)和其他数据的语句。有三种类型的声明: registered claims, public claims, and private claims。

例如:

```
{
  "sub": "1234567890",// 注册声明
  "name": "John Doe",// 公共声明
  "admin": true // 私有声明
}
```

这部分的声明也会通过base64进行加密,最终形成JWT的第二部分eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkpvaG4gRG9lIiwiaWF0IjoxNTE2MjM5MDIyfQ
**registered claims(注册声明)**

> 这些是一组预定义的声明，它们 **`不是强制性的，而是推荐的`** ，以 **`提供一组有用的、可互操作的声明`** 。

`JWT`的核心作用就是保护`Claims`的完整性（或者数据加密），保证`JWT`传输的过程中`Claims`不被篡改（或者不被破解）。`Claims`在`JWT`原始内容中是一个`JSON`格式的字符串，其中单个`Claim`是`K-V`结构，作为`JsonNode`中的一个`field-value`，这里列出常用的规范中预定义好的`Claim`：

| 简称 |      全称       |                 含义                  |
| :--: | :-------------: | :-----------------------------------: |
| iss  |     Issuer      |                发行方                 |
| sub  |     Subject     |                 主体                  |
| aud  |    Audience     |            （接收）目标方             |
| exp  | Expiration Time |               过期时间                |
| nbf  |   Not Before    | 早于该定义的时间的`JWT`不能被接受处理 |
| iat  |    Issued At    |          `JWT`发行时的时间戳          |
| jti  |     JWT ID      |            `JWT`的唯一标识            |

**`注意:声明名称只有三个字符，因为 JWT 意味着是紧凑的。`**

**Public claims(公共的声明)**

> 使用 JWT 的人可以随意定义这些声明( **`可以自己声明一些有效信息如用户的id,name等,但是不要设置一些敏感信息,如密码`** )。但是为了避免冲突，应该在 JWT注册表中定义它们，或者将它们定义为包含抗冲突名称空间的 URI。

**Private claims(私人声明)**

> 这些是创建用于在同意使用它们的各方之间共享信息的习惯声明，既不是注册声明，也不是公开声明( **`私人声明是提供者和消费者所共同定义的声明`** )。

**`注意:对于已签名的Token，这些信息虽然受到保护，不会被篡改，但任何人都可以阅读。除非加密，否则不要将机密信息放在 JWT 的有效负载或头元素中。`**

#### (3) Signature

> 要创建Signature，您必须获取编码的标头（header）、编码的有效载荷(payload)、secret、标头中指定的算法，并对其进行签名。

例如，如果您想使用 HMAC SHA256算法，签名将按以下方式创建:

```
HMACSHA256(
  base64UrlEncode(header) + "." +base64UrlEncode(payload),
  secret
  )
```

上面的JSON将会通过HMACSHA256算法结合secret进行加验签名(私钥加密)，其中header和payload将通过base64UrlEncode()方法进行base64加密然后通过字符串拼接 **`"."`** 生成新字符串,最终生成JWT的第三部分SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c

**`注意:secret是保存在服务器端的，jwt的签发生成也是在服务器端的，secret就是用来进行jwt的签发和验证，所以，它就是你服务端的私钥，在任何场景都不应该流露出去。一旦客户端得知这个secret, 那就意味着客户端是可以自我签发jwt了`**

#### (4) JWT的生成与解析

> JWT输出是三个由点分隔的 Base64-URL 字符串，这些字符串可以在 HTML 和 HTTP 环境中轻松传递，同时与基于 XML 的标准(如 SAML)相比更加紧凑。

下面显示了一个 JWT，该 JWT 对前一个头和有效负载进行了编码，并使用一个 secret 进行签名。

![在这里插入图片描述](https://ucc.alicdn.com/images/user-upload-01/fe7a3884df1443329a1f35b2a7ae838c.png)

真实情况,一般是在请求头里加入Authorization，并加上Bearer标注最后是JWT(格式:Authorization: Bearer **`<token>`**)：

![在这里插入图片描述](https://ucc.alicdn.com/images/user-upload-01/2aa1d619cd3c4dbcbc0c7eb587f678a1.png)

#### (5) 总结

第一部分是header, 是一个json字符串, 描述了自己是什么和生成的算法。这一串字符是经过base64编码后的结果。

第二部分是 payload, 翻译过来就是负载，简而言之也就是令牌token的内容，其内部的字段是可以自定义的。

第三部分是签名。所谓签名, 就是对第二部分的payload进行取哈希值。由于哈希函数具有 单向性和 很强的方碰撞性，所以可以防止有人串改第二部分的 payload。

当接受jwt的一方需要对签名进行验证, 这个过程叫做验签。只有经过验签的jwt才是有效和真实的。

jwt的签名和验签都需要密钥的参与，要不然谁都可以生成一个 jwt, 服务端无法确认 jwt 的真实性，也就无法使用了。

### 应用

一般是在请求头里加入Authorization，并加上Bearer标注：

```php
fetch('api/user/1', {  headers: {    'Authorization': 'Bearer ' + token  }})
```

服务端会验证token，如果验证通过就会返回相应的资源。

### 常见的签名和验签 算法

#### 对称式签名（验签）

对称式最常见的当属 HS256。简单来说, 签名过程 就是对 header,payload和密钥的拼接 进行一次取SHA256哈希值, 作为 jwt 的第三部分。 用公式来写, 就是这样:

$signedString=SHA256(header+payload+key)$

验签过程需要拿到 密钥 key(提前约定好的), 根据 header 和 payload  计算SHA256哈希值，如果和 jwt 的第三部分一致, 说明 jwt 真实, 否则说明 jwt 被篡改。

#### 非对称式签名(验签)

非对称式最常见的当属 RS256。简单来说, 就是对  header 和 payload  计算SHA256哈希值, 随后对这个哈希值使用私钥进行加密。 用公式来写就是这样:

$signedString=SHA256(header+payload)$

$cipherText=RSAencrypt(signedString,privateKey)$

验签的时候使用公钥解密出 signedString, 再根据 header 和 payload  计算SHA256哈希值,  随后对这个哈希值和 jwt的部分进行比对。如果和 jwt 的第三部分一致, 说明 jwt 真实, 否则说明 jwt 被篡改。

#### HS256 与 RS256 区别

HS256 需要双方严格保管密钥, 如果有一方泄露了密钥, 那么就可以伪造出 jwt. 而 RS256 签名的时候使用私钥, 验签的时候使用公钥，只要私钥不泄露, 那么jwt是不能被伪造的, 充其量只是公钥泄露, 谁都验证jwt而已。

`其实也就是对称加密和非对称加密的区别`

### jwt-go

#### 安装 jwt-go

```javascript
go get -u github.com/dgrijalva/jwt-go@latest
```

#### 生成token

使用 `jwt-go` 库生成` token`，我们需要定义需求（`claims`），也就是说我们需要通过 jwt 传输的数据。假如我们需要传输 ID 和 Username，我们可以定义 Claims 结构体，其中包含 ID 和 Username 字段，还有在 `jwt-go` 包预定义的 `jwt.StandardClaims`。

```go
type Claims struct {
	ID       int64
	Username string
	jwt.StandardClaims
}
```

`jwt.StandardClaims` 包含的字段：

```go
type StandardClaims struct {
  Audience  string `json:"aud,omitempty"`
  ExpiresAt int64  `json:"exp,omitempty"`
  Id        string `json:"jti,omitempty"`
  IssuedAt  int64  `json:"iat,omitempty"`
  Issuer    string `json:"iss,omitempty"`
  NotBefore int64  `json:"nbf,omitempty"`
  Subject   string `json:"sub,omitempty"`
}
```

`jwt.NewWithClaims` 方法：

```javascript
func jwt.NewWithClaims(method jwt.SigningMethod, claims jwt.Claims) *jwt.Token
```

`jwt.NewWithClaims` 方法根据 `Claims` 结构体创建 `Token` 示例。

参数 1 是 jwt.SigningMethod，

其中包含

* `jwt.SigningMethodHS256`，

* `jwt.SigningMethodHS384`，

* `jwt.SigningMethodHS512` 

三种`crypto.Hash` 加密算法的方案。

参数 2 是 Claims，包含自定义类型和 StandardClaim，StandardClaim 嵌入在自定义类型中，以方便对标准声明进行编码，解析和验证。

SignedString 方法：

```go
func (*jwt.Token).SignedString(key interface{}) (string, error)
```

SignedString 方法根据传入的空接口类型参数 key，返回完整的签名令牌。

##### 示例

```go
//claims.go

package claims

import "github.com/dgrijalva/jwt-go"

type Claims struct {
	ID       int64
	Username string
	jwt.StandardClaims
}
```



```go
package Token

import (
	"github.com/dgrijalva/jwt-go"
)

// GenerateTokenString
// 获取CLAIMS结构体对应的token值
func GenerateTokenString(claims jwt.Claims) (string, error) {
	//func NewWithClaims(method SigningMethod, claims Claims) *Token
	//参数解析：method表示加密方法，claims表示对应的claims结构体，后续跟着.SignedString([]byte("golang"))，其中的参数其实是代表私钥
	token, err := jwt.NewWithClaims(jwt.SigningMethodHS256, claims).SignedString([]byte("golang"))
	return token, err
}

```



```go
//main.go
package main

import (
	"GO_JWT/Token"
	"GO_JWT/claims"
	"fmt"
	"github.com/dgrijalva/jwt-go"
)

func main() {
	TokenString, err := Token.GenerateTokenString(claims.Claims{
		ID:             1,
		Username:       "Tom",
		StandardClaims: jwt.StandardClaims{},
	})
	if err != nil {
		panic(err)
	}
	fmt.Printf("%v", TokenString)
}

//output:eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJJRCI6MSwiVXNlcm5hbWUiOiJUb20ifQ.2rzyQl4GG3DSXC7BbNKgClh0iMYA2dIz7c35hoTxhcQ
```

#### 解析token

使用 jwt-go 库解析 token，主要用到两个方法，分别用通过与解析传入的 token 字符串，和根据 `MyCustomClaims` 结构体定义的相关属性要求进行校验。

`jwt.ParseWithClaims` 方法：

```javascript
func jwt.ParseWithClaims(tokenString string, claims jwt.Claims, keyFunc jwt.Keyfunc) (*jwt.Token, error)
```

`jwt.ParseWithClaims` 方法用于解析鉴权的声明，返回 `*jwt.Token。

`Valid` 方法用于校验鉴权的声明。

```go
//ParseToken.go
package Token

import (
	"GO_JWT/claims"
	"github.com/dgrijalva/jwt-go"
)

// ParseTokenString
// 解析Token字符串，将其转换为claims结构体对象
func ParseTokenString(tokenString string) (interface{}, error) {
	//func jwt.ParseWithClaims(tokenString string, claims jwt.Claims, keyFunc jwt.Keyfunc) (*jwt.Token, error)
	//参数解析：tokenString token字符串，claims 目标结构体对象，keyFunc表示通过这个token返回私钥
	tokenClaims, err := jwt.ParseWithClaims(tokenString, &claims.Claims{}, func(token *jwt.Token) (interface{}, error) {
		return []byte("golang"), nil
	})

	if err != nil {
		return nil, err
	}

	if tokenClaims != nil {
		if value, ok := tokenClaims.Claims.(*claims.Claims); ok && tokenClaims.Valid {
			return value, nil
		}
	}

	return nil, err
}
```

```go
package main

import (
	"GO_JWT/Token"
	"GO_JWT/claims"
	"fmt"
	"github.com/dgrijalva/jwt-go"
)

func main() {
	TokenString, err := Token.GenerateTokenString(claims.Claims{
		ID:             1,
		Username:       "Tom",
		StandardClaims: jwt.StandardClaims{},
	})
	if err != nil {
		panic(err)
	}
	fmt.Printf("%v\n", TokenString)

	value, err := Token.ParseTokenString(TokenString)
	fmt.Printf("%#v", value)
}
//output:
//eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJJRCI6MSwiVXNlcm5hbWUiOiJUb20ifQ.2rzyQl4GG3DSXC7BbNKgClh0iMYA2dIz7c35hoTxhcQ
//&claims.Claims{ID:1, Username:"Tom", StandardClaims:jwt.StandardClaims{Audience:"", ExpiresAt:0, Id:"", IssuedAt:0, Issuer:"", NotBefore:0, Subject:""}}
```

主要参考自https://developer.aliyun.com/article/995894



