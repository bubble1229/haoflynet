---
title: "OAuth和SSO"
date: 2016-06-16 14:08:00
categories: 编程之路
---
参考[理解OAuth 2.0](http://www.ruanyifeng.com/blog/2014/05/oauth_2_0.html)

目前网络上的数据传输普遍基于HTTP/HTTPS，然而，他们都是无状态的协议，也就是说你本次的请求与下一次的请求是毫无关系的，为了使得我们能够保存用户的状态，于是诞生了cookie，诞生了session，诞生了token，当然，这三者的概念其实都差不多，总体思想就是，在第一次请求过后，给用户生成唯一的一个标识，用户在下一次请求时，携带这个标识，这样服务端就能通过它来判断用户的状态以及合法性。

注意: OAuth确实增加了安全性，但是也增加了应用的复杂性，对于安全要求没那么高的应用，依然可以使用简单的加密算法进行双向加解密认证。

## OAuth
OAuth主要用于社会化登录，仅仅需要一个系统来存储用户的信息，其他的系统或者第三方系统均可以使用，其他系统不需要维护自己的用户系统，免去了用户注册账号的麻烦，并且用户的密码等重要信息都集中保存在信任方，提高了安全性。

根据阮一峰的博客，客户端获取授权有以下几种认证方式:

1. 授权码模式（authorization code）: 是功能最完整、流程最严密的授权模式。它的特点就是通过客户端的后台服务器，与"服务提供商"的认证服务器进行互动。
2. 简化模式（implicit）: 不通过第三方应用程序的服务器，直接在浏览器中向认证服务器申请令牌，跳过了"授权码"这个步骤，因此得名。所有步骤在浏览器中完成，令牌对访问者是可见的，且客户端不需要认证。
3. 密码模式（resource owner password credentials）:用户向客户端提供自己的用户名和密码。客户端使用这些信息，向"服务商提供商"索要授权。
4. 客户端模式（client credentials）:指客户端以自己的名义，而不是以用户的名义，向"服务提供商"进行认证。严格地说，客户端模式并不属于OAuth框架所要解决的问题。在这种模式中，用户直接向客户端注册，客户端以自己的名义要求"服务提供商"提供服务，其实不存在授权问题。

![举个例子](http://7xnc86.com1.z0.glb.clouddn.com/oauth-and-sso-1.png)
具体流程(感觉不用图也能说清楚呀):

1. 第三方站点获取未授权的Request Token
2. 获取用户授权的Request Token
3. 用授权的Request Token换取Access Token
4. 通过Access Token访问该用户的资源

另外，对于客户端的每一次获取token的请求，都对应着一定的权限，比如微博的第三方认证，通常仅有获取用户基本信息的权限，或者如github的权限就更加详细一点，包括获取基本信息，公开库的访问权限和私有库的访问权限(github的认证请求权限使用的是`X-OAuth-Scopes`作为参数放在请求头部，例如`user`，表示能获取用户的所有信息，而`user.email`则表示只能获取用户的邮箱)。

对于第三方应用，仅需要维护token的值，一般来说，有一种用户基本信息表，多张token存储表对应着多个公开应用。例如接入了微博的第三方可以有这么一张表: (id, user_id, weibo_id, weibo_access_token, weibo_expires)

注: 社会化登录的回调地址(callback_url)与返回地址(next)的区别，回调地址指用户点击第三方登录过后，由资源服务器回调到的地址，而返回地址则是用户欲访问的地址，一般为应用自己的首页，但也有可能是用户直接知道url后想去的地址，这时候可以在请求的时候带上next参数，然后由资源服务器原样返回过去。

最后，几乎所有关于OAuth的文章都会遗漏的一点，那就是资源服务器和认证服务器之间的认证。当访问者获取了token过后，向资源服务器发送请求时，资源服务器肯定会向认证服务器验证这个token的合法性，这也是一个很危险的步骤，在实际项目中，我将token与访问者的appkey一起进行认证，并且作为认证服务器同样要对资源服务器的认证请求一起认证。具体可参考[stackoverflow](http://stackoverflow.com/questions/12296017/how-to-validate-an-oauth-2-0-access-token-for-a-resource-server)

## SSO(单点登录)
解决的是跨域的用户校验问题，常用于一个公司内部有多个子网站，然后提供统一的登录，一个站点登录过后其他站点不用重复登录。多用于存在多个子系统的公司。
具体流程:
1. 用户请求任意一个子站的login页面
2. 用户输入用户名密码，子站将它传递给认证中心SSO Server
3. SSO Server验证成功后

虽然SSO和OAuth是不一样的东西，但是我认为SSO只是OAuth的一种简化模式，可以归为OAuth的一类。通常由于子系统的域名不一样，不方便设置各自的cookie，于是这里又有两种解决方案:

1. 使用公共加解密函数进行双向加解密(加密字符串直接放在GET请求中)
2. 同样使用公共的加解密函数，但是是使用JSOPN解决跨域问题(用户登录子应用时，会带上父应用域名下的cookie访问父应用提供的JSONP接口，父应用验证登录状态，最后返回加密后的用户信息)

例如，本人在一个项目中所使用的认证流程:

![图片](http://7xnc86.com1.z0.glb.clouddn.com/oauth-and-sso-2.png)

## 签名算法的设计

无论是OAuth或是SSO都涉及到认证的过程，一般都不推荐使用可逆的加密算法，而使用单向的加密算法，只要双方对数据进行加密后的结果一致就表示该请求是合法的。参考网上很多的签名算法，最后总结了一下使用最广也是最安全的一种签名算法步骤:

1. 将请求的参数转换为'key=value'形式的字符串，如: "k1=v1", "k2=v2"
2. 将格式化后的字符串以字典升序进行排序，然后拼接在一起，如: "k1=v1k2=v2"
3. 在拼接后的字符串的前后加上Secret Key，如："keyk1=v1k2=v2key"
4. 对拼接后的字符串进行MD5加密得到最终的结果

### Python实现

```python
import hashlib
def apiSign(parameters, secret):
	keys = list(parameters.keys())
	keys.sort()

	parameters_str = "%s%s%s" % (
		secret,
		str().join('%s%s' % (key, parameters[key]) for key in keys),
		secret)
	return hashlib.md5(parameters_str.encode('utf8')).hexdigest()j
```

### PHP实现

```php
function apiSign($parameters, $secret)
{
	$str = '';

	if (gettype($parameters) === 'array') {
		ksort($parameters);
		foreach ($parameters as $k => $v) {
			$str .= $k.$v;
		}
	} elseif (gettype($parameters) === 'string') {
		$str = $parameters;
	} else {
		return false;
	}
	$str = $secret.$str.$secret;
	
	return md5($str);
}
```

### Java实现

```java
private static String apiSign(List<NameValuePair> nvps) throws NoSuchAlgorithmException, IOException {
    List<String> nvs = Lists.newArrayList();
    for (NameValuePair nvp: nvps) {
        boolean noSignValue = nvp == null || nvp.getValue() == null ||
                SIGN_EXCEPTIONS.contains(nvp.getName());
        if (noSignValue) continue;
        nvs.add(String.format("%s=%s", nvp.getName(), nvp.getValue()));
    }
    Collections.sort(nvs);
    StringBuilder sb = new StringBuilder();
    for (String nv: nvs) sb.append(String.format("%s", nv));
    String encodeSource = String.format("%s%s", sb.toString(), APP_SECRET);
    return MD5Util.MD5Encode(encodeSource, ENCRYPT_CHARSET).toLowerCase();
}
```

