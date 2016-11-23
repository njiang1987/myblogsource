title: '[转]iOS开发之--- NSURLProtocol'
date: 2016-11-22 19:40:06
tags: [iOS]
---

最近在项目里由于电信那边发生dns发生域名劫持，因此需要手动将URL请求的域名重定向到指定的IP地址，但是由于请求可能是通过NSURLConnection,NSURLSession或者AFNetworking等方式，因此要想统一进行处理，一开始是想通过Method Swizzling去hook cfnetworking底层方法，后来发现其实有个更好的方法--NSURLProtocol。

<!-- more -->

## NSURLProtocol

  NSURLProtocol能够让你去重新定义苹果的[URL加载系统](https://developer.apple.com/library/ios/documentation/Cocoa/Conceptual/URLLoadingSystem/URLLoadingSystem.html#//apple_ref/doc/uid/10000165-BCICJDHA) (URL Loading System)的行为，URL Loading System里有许多类用于处理URL请求，比如NSURL，NSURLRequest，NSURLConnection和NSURLSession等，当URL Loading System使用NSURLRequest去获取资源的时候，它会创建一个NSURLProtocol子类的实例，你不应该直接实例化一个NSURLProtocol，NSURLProtocol看起来像是一个协议，但其实这是一个类，而且必须使用该类的子类，并且需要被注册。

## 使用场景

  不管你是通过UIWebView, NSURLConnection 或者第三方库 (AFNetworking, MKNetworkKit等)，他们都是基于NSURLConnection或者 NSURLSession实现的，因此你可以通过NSURLProtocol做自定义的操作。

- 重定向网络请求
- 忽略网络请求，使用本地缓存
- 自定义网络请求的返回结果
- 一些全局的网络请求设置

## 拦截网络请求

#### 子类化NSURLProtocol并注册

```
@interface CustomURLProtocol : NSURLProtocol
@end
```

然后在application:didFinishLaunchingWithOptions:方法中注册该CustomURLProtocol，一旦注册完毕后，它就有机会来处理所有交付给URL Loading system的网络请求。

```
- (BOOL)application:(UIApplication *)application didFinishLaunchingWithOptions:(NSDictionary *)launchOptions {
    //注册protocol
    [NSURLProtocol registerClass:[CustomURLProtocol class]];
    return YES;
}
```

#### 实现CustomURLProtocol

注册好了之后，现在可以开始实现NSURLProtocol的一些方法：

- +canInitWithRequest:
  这个方法主要是说明你是否打算处理对应的request，如果不打算处理，返回NO，URL Loading System会使用系统默认的行为去处理；如果打算处理，返回YES，然后你就需要处理该请求的所有东西，包括获取请求数据并返回给 URL Loading System。网络数据可以简单的通过NSURLConnection去获取，而且每个NSURLProtocol对象都有一个NSURLProtocolClient实例，可以通过该client将获取到的数据返回给URL Loading System。
  这里有个需要注意的地方，想象一下，当你去加载一个URL资源的时候，URL Loading System会询问CustomURLProtocol是否能处理该请求，你返回YES，然后URL Loading System会创建一个CustomURLProtocol实例然后调用NSURLConnection去获取数据，然而这也会调用URL Loading System，而你在+canInitWithRequest:中又总是返回YES，这样URL Loading System又会创建一个CustomURLProtocol实例导致无限循环。我们应该保证每个request只被处理一次，可以通过+setProperty:forKey:inRequest:标示那些已经处理过的request，然后在+canInitWithRequest:中查询该request是否已经处理过了，如果是则返回NO。

```
+ (BOOL)canInitWithRequest:(NSURLRequest *)request
{
  //只处理http和https请求
    NSString *scheme = [[request URL] scheme];
    if ( ([scheme caseInsensitiveCompare:@"http"] == NSOrderedSame ||
     [scheme caseInsensitiveCompare:@"https"] == NSOrderedSame))
    {
        //看看是否已经处理过了，防止无限循环
        if ([NSURLProtocol propertyForKey:URLProtocolHandledKey inRequest:request]) {
            return NO;
        }

        return YES;
    }
    return NO;
}
```

- +canonicalRequestForRequest:
  通常该方法你可以简单的直接返回request，但也可以在这里修改request，比如添加header，修改host等，并返回一个新的request，这是一个抽象方法，子类必须实现。

```
+ (NSURLRequest *) canonicalRequestForRequest:(NSURLRequest *)request {
    NSMutableURLRequest *mutableReqeust = [request mutableCopy];
    mutableReqeust = [self redirectHostInRequset:mutableReqeust];
    return mutableReqeust;
}

+(NSMutableURLRequest*)redirectHostInRequset:(NSMutableURLRequest*)request
{
    if ([request.URL host].length == 0) {
        return request;
    }

    NSString *originUrlString = [request.URL absoluteString];
    NSString *originHostString = [request.URL host];
    NSRange hostRange = [originUrlString rangeOfString:originHostString];
    if (hostRange.location == NSNotFound) {
        return request;
    }
    //定向到bing搜索主页
    NSString *ip = @"cn.bing.com";

    // 替换域名
    NSString *urlString = [originUrlString stringByReplacingCharactersInRange:hostRange withString:ip];
    NSURL *url = [NSURL URLWithString:urlString];
    request.URL = url;

    return request;
}
```

- +requestIsCacheEquivalent:toRequest:
  主要判断两个request是否相同，如果相同的话可以使用缓存数据，通常只需要调用父类的实现。

```
+ (BOOL)requestIsCacheEquivalent:(NSURLRequest *)a toRequest:(NSURLRequest *)b
{
    return [super requestIsCacheEquivalent:a toRequest:b];
}
```

- -startLoading  -stopLoading
  这两个方法主要是开始和取消相应的request，而且需要标示那些已经处理过的request。

```
- (void)startLoading
{
    NSMutableURLRequest *mutableReqeust = [[self request] mutableCopy];
    //标示改request已经处理过了，防止无限循环
    [NSURLProtocol setProperty:@YES forKey:URLProtocolHandledKey inRequest:mutableReqeust];
    self.connection = [NSURLConnection connectionWithRequest:mutableReqeust delegate:self];
}

- (void)stopLoading
{
    [self.connection cancel];
}
```

- NSURLConnectionDataDelegate方法
  在处理网络请求的时候会调用到该代理方法，我们需要将收到的消息通过client返回给URL Loading System。

```
- (void) connection:(NSURLConnection *)connection didReceiveResponse:(NSURLResponse *)response {
    [self.client URLProtocol:self didReceiveResponse:response cacheStoragePolicy:NSURLCacheStorageNotAllowed];
}

- (void) connection:(NSURLConnection *)connection didReceiveData:(NSData *)data {
    [self.client URLProtocol:self didLoadData:data];
}

- (void) connectionDidFinishLoading:(NSURLConnection *)connection {
    [self.client URLProtocolDidFinishLoading:self];
}

- (void)connection:(NSURLConnection *)connection didFailWithError:(NSError *)error {
    [self.client URLProtocol:self didFailWithError:error];
}
```

现在你已经可以截取request并做你想做的事了，这里有个[demo](https://github.com/FreeMind-LJ/NSURLProtocolExample)可以参考一下，截取request并重新定向到新的地址，具体dns解析方法可以参看[DNS解析)](http://www.jianshu.com/p/d945454e3abc) ，如有不对，欢迎指正，哈～

文／树下的老男孩（简书作者）
原文链接：http://www.jianshu.com/p/7c89b8c5482a
著作权归作者所有，转载请联系作者获得授权，并标注“简书作者”。