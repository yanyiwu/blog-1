# 介绍

几个月前，思索了一套安全的服务器交互模型，随着几个月的不停的思索，又有很多新的心得体会。抽象来说，我认为，对于一个好的服务器架构，简单，可扩展性强，是必不可少的。

- 简单，其实很好理解，一个服务就干一件事情，不同的功能逻辑别糅在一个服务里面实现。短期来看，一个服务器实现一堆功能逻辑，编码容易简单，但是弊端也能立马显现，可维护性差，单点故障等。我们的项目就出现了这样的问题，一个童鞋当初为了方便，将本来几个服务，组织架构服务，登陆验证服务，账号管理服务等几个服务的代码糅合在一个服务里面，编码是简单了，但是后续一堆问题出来了。举个最简单的例子，我们想使用外部的一个登陆验证模块，使用外部的组织架构数据，现在我们完全无法实现。就因为这个，我们又准备将这几个服务进行拆分。所以，无论怎样，保证服务的简单，是为了更好的为了后续的扩展

- 扩展性强，一是服务功能可扩展强，因为一个服务一个功能，当我们需要实现另一个功能的时候就增加一个服务，每个服务通过标准的通信协议（一般就是http）进行通信就行。另一个就是服务本身的可扩展性强，单个服务里面实现的功能都应该是状态无关的，也就是说服务的api提供不能有时序关系。就因为无时序状态，所以每个服务的个数可以动态扩展

# 安全交互

在一套安全的服务器交互模型中，我提出使用key + stub的方式进行高效安全的交互，这套机制虽然没有问题，但是我现在觉得是对第三方开发不友好的方式。如果不光我们提供服务，客户端也由我们编写，那无所谓，但是如果另一个客户端想对接我们的服务，就会面临一个很郁闷的问题，必须保存每一个服务的stub，这个随着服务的增多会越来越来管理。所以思索了很久，还是采用业界比较通用的签名方法，这个足够高效，也足够安全。

- 部署一个访问验证服务器，用于管理access id和security key
- 当用户安全的登陆之后，我们会给访问客户端一个用于访问我们所有服务的access id和security key。因为登陆一般采用的是https，所以id和key不会被网络窃听获取到，所以是安全的
- 客户端收到该id和key之后，对于后续任何的http请求，使用key对url签名，然后将id与签名放置在http header里面一同发送给server
- server收到http请求之后，首先通过header里面的id在访问验证server里面找到对应的key，在使用同样的方式对url进行签名，如果两次签名一样，表明这是一个合法的url，如果不一样，可能其他人篡改了该url，我们拒绝这次http请求
- 当用户注销登出，id和key就无效了。如果为了安全性，客户端也可以强制的每隔一段时间进行id和key的更换

签名机制足够的简单高效，但也必须得注意几个问题：

- 每次对url进行签名最好带上一个随机数，譬如当前时间，这样不容易被窃听之后暴力破解得到key
- 为了更好的保证安全性，这套机制还是应该在https上面使用

# 访问验证服务器

这里我引入访问验证服务器，也是我认为架构简单性的考量，每个服务干自己的事情，所以需要验证了就提供一个验证服务器。

但这里又有一个问题，当服务器收到http请求之后，在什么时候进行url的验证？为了保证简单性，各个服务提供api是不会出现任何验证逻辑，那这套验证逻辑放置在什么地方呢？我觉得可以在两个地方处理：

- 服务框架的统一http入口处理里面，铁定我们会写一套框架代码，各个服务在这套框架之上运行。譬如我们采用tornado wsgi的方式搭建服务器，他对于任何http请求都是有一个统一的入口的，我们只要在这个入口里面首先进行访问权限的判断，然后在继续进行后续操作就行
- 在引入一个总控服务器，该服务器收到http请求之后首先进行访问验证，然后在进行后续操作

这两种方式，第一种是我们项目现在使用stub采用的，而第二种则是我现在推荐的方式。第一种方式有一个不好的地方在于，验证逻辑跟框架代码耦合了，虽然我们可以通过很多方式让框架跟验证逻辑进行解耦，但是我仍然觉得有点别扭。现今我们stub为什么可以这么是因为验证控制这套逻辑本来就是由底层框架提供，而不是外部服务提供。

而采用第二种方式，则需要引入一个总控服务器，可以使用openresty解决，因为我们所有的http都要经过nginx，所以直接在nginx里面做就可以了。因为nginx性能很高，同时在openresty里面使用lua做到非常容易开发，而且我们实现的只是简单的流程控制，所以通过nginx绰绰有余。

# 原子级别的api服务

因为我们的功能都是由不同的服务器提供，这里就有一个比较郁闷的问题，一个功能如果需要不同服务器协同，怎么处理。譬如我现在想实现一个功能，我需要a，b，c三个服务以此进行处理，只有都处理成功了这个功能才算完成。如果只有我们自己写客户端，那没问题，但是如果有一个第三方需要对接实现这个功能，那么他们就会很困惑，为啥我要这么麻烦来实现这个功能？并且还不能保证他们能理解正确，写出正确的功能代码。

这个问题现在已经比较严重了，导致了我们有时候不得不花上很多时间来给其他开发人员解释api如何使用。所以每一个功能有一个原子级别的api来提供，是一个不错的办法。怎么说呢，因为引入了openresty，所以我们完全可以在nginx这层就写好该功能需要访问的服务。这个其实跟上面的访问验证类似，就是由openresty做总控，来进行流程的控制。

# 总控nginx

既然我们使用openresty来进行服务的总控控制，就需要注意一个最重要的问题：openresty负责的功能只能是简单的流程控制，它不负责其他任何的功能逻辑，这样才能保证足够的简单。我们不希望在nginx里面有太多重型的逻辑。

# 总结

这些算是近端时间以来对服务架构的一些理解，还有一些，后续慢慢补充。
