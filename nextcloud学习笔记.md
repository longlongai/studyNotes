## 数据库查询

```php
$db = \OC::$server->getDatabaseConnection();
$qb = $db->getQueryBuilder();

$qb->select(...$fileds)
    ->from($table[0], 'a')
    ->leftJoin('a', $table[1], 'b', $condition)
    ->where($where);

$result = $qb->execute();

return $result->fetchAll();
```

​      

#### 获取请求数据

````php
$request = \OC::$server->getRequest();
````

#### 路由加载

以URL：`http://192.168.158.128/company/bserver/ocs/v2.php/ga/officer` 为例。

```php
//第一步 v2.php ==> v1.php
require_once __DIR__ . '/v1.php';

//第二步 v1.php
OC::$server->getRouter()->match('/ocsapp'.\OC::$server->getRequest()->getRawPathInfo());

//第三步 bserver/lib/private/Server.php
$this->registerService(IRouter::class, function (Server $c) {
			$cacheFactory = $c->getMemCacheFactory();
			$logger = $c->getLogger();
			if ($cacheFactory->isLocalCacheAvailable()) {
				$router = new \OC\Route\CachingRouter($cacheFactory->createLocal('route'), $logger);
			} else {
				$router = new \OC\Route\Router($logger);
			}
			return $router;
		});
//第四步 bserver/lib/private/Route/Router.php
public function match($url){
    $this->loadRoutes();
}
 
public function loadRoutes($app = null) {
    $this->requireRouteFile($file, $app);
}

private function requireRouteFile($file, $appName) {
    $this->setupRoutes(include_once $file, $appName);
}

private function setupRoutes($routes, $appName) {
    if (class_exists($applicationClassName)) {
        $application = \OC::$server->query($applicationClassName);
    } else {
        $application = new App($appName);
    }
    $application->registerRoutes($this, $routes);
}

//第五步 bserver/lib/public/AppFramework/App.php
public function registerRoutes(IRouter $router, array $routes) {
    $routeConfig = new RouteConfig($this->container, $router, $routes);
    $routeConfig->register();
}

//第六步 bserver/lib/private/AppFramework/Routing/RouteConfig.php
public function register() {
    //...
}

```

#### 文件分块上传

以`http://192.168.37.31/company/bserver/remote.php/dav/uploads/28c8edde3d61a0411511d3b1866f0636/e10adc3949ba59abbe56e057f20f883e/0000000000000003-0000000000000004` 为例。

url解释：`PUT /remote.php/dav/uploads/用户名/文件标识/分块开始位置-分块结束位置`

```php
//第一步 bserver/remote.php
function resolveService($service) {
	$services = [
		'webdav' => 'dav/appinfo/v1/webdav.php',
		'dav' => 'dav/appinfo/v2/remote.php',	//根据dav 找到对于引入文件
		//...
	];
	if (isset($services[$service])) {
		return $services[$service];
	}

	return \OC::$server->getConfig()->getAppValue('core', 'remote_' . $service);
}

//第二步 bserver/apps/dav/appinfo/v2/remote.php
$request = \OC::$server->getRequest();
$server = new \OCA\DAV\Server($request, $baseuri);	//添加控件
$server->exec();

//第三步 bserver/apps/dav/lib/Server.php
public function __construct(IRequest $request, $baseUri) {
		$root = new RootCollection();
		$this->server = new \OCA\DAV\Connector\Sabre\Server(new CachingTree($root));
}

//第四步 bserver/apps/dav/lib/Connector/Sabre/Server.php
public function __construct($treeOrNode = null) {
    parent::__construct($treeOrNode);
    self::$exposeVersion = false;
    $this->enablePropfindDepthInfinity = true;
}

//第五步 bserver/3rdparty/sabre/dav/lib/DAV/Server.php
public function __construct($treeOrNode = null)
{
    $this->addPlugin(new CorePlugin());
}

public function addPlugin(ServerPlugin $plugin)
{
    $this->plugins[$plugin->getPluginName()] = $plugin;
    $plugin->initialize($this);
}

//第六步 bserver/3rdparty/sabre/dav/lib/DAV/CorePlugin.php
public function initialize(Server $server)
{
    //...
    $server->on('method:PUT', [$this, 'httpPut']);
}
```



#### 控制器中某个方法不登陆即可访问

实例：`http://192.168.158.128/company/bserver/ocs/v2.php/ga/verifyIp`

```php
//第一步 v2.php ==> v1.php
require_once __DIR__ . '/v1.php';

//第二步 lib/base.php
class OC {
{
    public static function init() {
        //...
        
        // setup the basic server
		self::$server = new \OC\Server(\OC::$WEBROOT, self::$config);
        \OC::$server->getEventLogger()->log('autoloader', 'Autoloader', $loaderStart, $loaderEnd);
		\OC::$server->getEventLogger()->start('boot', 'Initialize');
    }
}
OC::init();
    
//第三步 lib/private/Server.php
public function getEventLogger() {
    return $this->query(IEventLogger::class);
}
  
//第四步 lib/private/ServerContainer.php
public function query(string $name, bool $autoload = true) {
    //...
    $appContainer = $this->getAppContainer(strtolower($segments[1]), $segments[1]);
}
    
protected function getAppContainer(string $namespace, string $sensitiveNamespace): DIContainer {
    //...
    return new DIContainer($this->namespaces[$namespace]);
}
    
//第五步 lib/private/AppFramework/DependencyInjection/DIContainer.php
public function __construct($appName, $urlParams = [], ServerContainer $server = null) {
    //...
    $securityMiddleware = new SecurityMiddleware(
        $c->query(IRequest::class),
        $c->query(IControllerMethodReflector::class),
        $c->query(INavigationManager::class),
        $c->query(IURLGenerator::class),
        $server->getLogger(),
        $c['AppName'],
        $server->getUserSession()->isLoggedIn(),
        $server->getGroupManager()->isAdmin($this->getUserId()),
        $server->getUserSession()->getUser() !== null && $server->query(ISubAdmin::class)->isSubAdmin($server->getUserSession()->getUser()),
        $server->getAppManager(),
        $server->getL10N('lib')
    );
    $dispatcher->registerMiddleware($securityMiddleware);
}    

//第六步 lib/private/AppFramework/Middleware/Security/SecurityMiddleware.php
class SecurityMiddleware extends Middleware {
    public function __construct(IRequest $request,
								ControllerMethodReflector $reflector,
								INavigationManager $navigationManager,
								IURLGenerator $urlGenerator,
								ILogger $logger,
								string $appName,
								bool $isLoggedIn,
								bool $isAdminUser,
								bool $isSubAdmin,
								IAppManager $appManager,
								IL10N $l10n
	) {
		$this->reflector = $reflector;	//通过反射获取注释
	}
    
    public function beforeController($controller, $methodName) {
        //...

        // security checks
        $isPublicPage = $this->reflector->hasAnnotation('PublicPage');
        if (!$isPublicPage) {
            //...
        }
    }
}


//第七步lib/private/AppFramework/Utility/ControllerMethodReflector.php
public function reflect($object, string $method) {
    //...
}
```



