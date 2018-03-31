REST全称是Representational State Transfer，这个网上资料很多，我这里不讲理论。

通过Python可以很方便的实现RESTful的web服务，特别适用于作为API服务器。 本文主要参考了openstack的使用方式，或者说，是把openstack中RESTfull的web服务单独抽取出来研究了一下。

本文只讲整个架构，具体每个组件网上资料非常多，没必要再重复了。

## 测试代码

### user.py代码清单

    import eventlet
    import eventlet.wsgi
    import json
    import logging
    import os
    import paste.deploy
    import routes.middleware
    import webob.dec
    import webob.exc
    
    LOG = logging.getLogger(__name__)
    
    
    class RouterBase(object):
        @classmethod 
        def app_factory(cls, global_config, **local_config):
            return cls()
    
        def __init__(self):
            self._mapper = routes.Mapper()
            self._setup_routes(self._mapper)
            self._router = routes.middleware.RoutesMiddleware(self._dispatch, self._mapper)
    
        def _setup_routes(self, mapper):
            pass
    
        @webob.dec.wsgify
        def __call__(self, req):
            return self._router
    
        @staticmethod
        @webob.dec.wsgify
        def _dispatch(req):
            print "CLL: req=%s" % req
            print "CLL: req.environ=%s" % req.environ
            match = req.environ['wsgiorg.routing_args'][1]
            if match:
                return match['controller']
            return webob.exc.HTTPNotFound()
    
    
    class ControllerBase(object):
        @webob.dec.wsgify
        def __call__(self, req):
            arg_dict = req.environ['wsgiorg.routing_args'][1]
            action = arg_dict.pop('action')
            method = getattr(self, action)
    
            del arg_dict['controller']
            params = {}
            params.update(arg_dict)
    
            result = method(req, **params)
            response = webob.Response()
            response.status_int = 200
            response.headers['Content-Type'] = 'application/json'
            response.body = json.dumps(result)
            return response
    
    
    class UserRouter(RouterBase):
        def _setup_routes(self, mapper):
            controller=UserController()
            mapper.connect('/users/{id}',
                           controller=controller,
                           action='show',
                           conditions={'method': ['GET']})
            mapper.connect('/users',
                           controller=controller,
                           action='create',
                           conditions={'method': ['POST']})
    
    
    class UserController(ControllerBase):
        def show(self, req, id):
            return {'id': id,
                    'name': 'Ridge Chen',
                    'mail': 'ridge.chen@gmail.com'}
    
        def create(self, req):
            body = req.body
            print "CLL: body=%s" % json.loads(body)
            return {'id': 1,
                    'name': 'Ridge Chen',
                    'mail': 'ridge.chen@gmail.com'}
    
    
    if __name__ == '__main__':
        config="config:%s" % os.path.abspath('user.ini')
        appname="user-api"
        wsgi_app = paste.deploy.loadapp(config, appname)
        socket = eventlet.listen(('localhost', 8080))
        eventlet.wsgi.server(socket, wsgi_app)

### user.ini代码清单

    [DEFAULT]
    name=user
    
    [composite:user-api]
    use = egg:Paste#urlmap
    /: root
    
    [pipeline:root]
    pipeline = user
    
    [app:user]
    paste.app_factory = user:UserRouter.app_factory

### 测试

启动：

    python user.py

测试：

    curl -v 'http://127.0.0.1:8080/users/1' -X GET -H "X-Auth-Project-Id: admin" \
        -H "Content-Type: application/json" -H "Accept: application/json" \
        -H "X-Auth-Token: 2b3b61f328594ee3bc207e8ae33a705d"

    curl -v 'http://127.0.0.1:8080/users' -X POST -H "X-Auth-Project-Id: admin" \
        -H "Content-Type: application/json" -H "Accept: application/json" \
        -H "X-Auth-Token: 2b3b61f328594ee3bc207e8ae33a705d" \
        -d '{"name": "Ridge Chen", "mail": "ridge.chen@gmail.com"}'

## 这里使用了以下几种技术：

- WSGI
- webob
- routes
- paste deploy

##　WSGI

WSGI是Web Server Gateway Interface，Web服务器网关接口的简称。 简单的说WSGI是一个接口，把Web服务程序切分为两部分，Web服务器和Web应用。 Web服务器处理接受HTTP请求、解析HTTP请求、发送HTTP响应等通用的任务，而业务相关的逻辑由Web应用来实现。 事实上，Web应用也可以再次利用WSGI接口，把自己伪装成Web服务器，把其他Web应用纳入自己的下家， 自己只处理其中特定的一个功能（例如权限认证）。 于是就有了中间层（MiddleWare），实现所谓的切面开发。

上面的代码中，`eventlet.wsgi.server`实现了一个Web服务器，使用了eventlet中模块。 而wsgi_app实现了一个Web应用，是自己的代码。 Web应用本质上是一个可执行的对象，可以是python方法， 实现`__call__`方法的类，等等。

关于WSGI的信息，不是本文的主要内容，请查询网上资料。

## webob

webob是一个python的帮助库，对WSGI做了一系列的封装。 例如上面的代码中

- `webob.dec.wsgify`装饰器能够把普通方法转化为WSGI应用。
- `webob.exc.HTTPNotFound`能够直接构造一个404的WSGI响应。
- `webob.Response`能够让用户来定制一个WSGI响应，可以指定返回码，body，headers等。

## routes

如其名，`routes.Mapper`用来处理URL路由。

`connect`方法用来定义一条路由，下面的例子中，当URL匹配`/users/{id}`，且HTTP方法为GET，mapper将会把请求转给controller这个Web应用处理，并把路由信息存入`req.environ['wsgiorg.routing_args']`中，路由信息中包含id的参数。

    mapper.connect('/users/{id}',
                   controller=controller,
                   action='show',
                   conditions={'method': ['GET']})

同样，routes更多的信息及使用方式，也请查询网上资料。

## paste deploy

paste deploy负责根据配置文件生成Web应用。 前面说了， WSGI框架可以使用中间层来分割Web应用的各个功能模块。 而使用paste deploy，能够把各个功能模块的组合配置化，从而具有很好的伸缩性。 例如，openstack中有一个Debug的MiddleWare，用来打印请求和响应的内容，在开发的时候用paste deploy“串”到Web应用的前面，而在生产环境去除。 这些都只需要改改paste deploy的配置即可。

paste deploy支持使用composite（复合）、pipline（管道）、filter（过滤）、app（应用）等方式，来分割一个复杂的应用。 

（本文结束）

 