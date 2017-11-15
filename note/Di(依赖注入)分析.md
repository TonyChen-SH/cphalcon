### 什么叫依赖注入(Dependency Injection)?
下面我们会用一个比较长的例子，详细解释一下为什么Phalcon用依赖注入的方式加载服务.
由于业务需要，我们需要开发一个处理任务的功能类，这个类需要依赖数据库的连接。不出意外的话，你可以会像下面这样写。尽快看上去也没什么大的问题，是一个可以正常使用的类。但是，它有一个比较大的缺点：对象不能改变数据库的连接信息和数据库类型，因为连接是写死在代码里面的。
```php
class SomeComponent
{
    /**
     * 采取硬编码的形式，把数据库配置连接写代码里
     * 修改代码才能修改连接地址，不符合面向对象的开闭原则
     */
    public function someDbTask()
    {
        $connection = new Connection(
            [
                'host'     => 'localhost',
                'username' => 'root',
                'password' => 'secret',
                'dbname'   => 'invo',
            ]
        );
        // ...
    }
}

$some = new SomeComponent();
$some->someDbTask();
```

为了解决这个问题，我使用setConnection方法来注入一个额外的连接对象$connection，用来配置数据库连接
```php
<?php

class SomeComponent
{
    private $connection;

    /**
     * 外部设置数据连接
     *
     * @param Connection $connection
     */
    public function setConnection(Connection $connection)
    {
        $this->connection = $connection;
    }

    public function someDbTask()
    {
        $connection = $this->connection;
        // ...
    }
}

$some = new SomeComponent();
// 创建连接对象
$connection = new Connection(
    [
        'host'     => 'localhost',
        'username' => 'root',
        'password' => 'secret',
        'dbname'   => 'invo',
    ]
);

// 注入到SomeComponent的对象中
$some->setConnection($connection);
$some->someDbTask();
```

现在考虑我们使用该组件在应用程序的不同部分,然后我们需要创建连接几次之前将它传递给组件。使用全局注册表模式,我们可以在那里存储连接对象和重用,当我们需要它。

```php
<?php

class Registry
{
    /**
     * Returns the connection
     */
    public static function getConnection()
    {
        return new Connection(
            [
                'host'     => 'localhost',
                'username' => 'root',
                'password' => 'secret',
                'dbname'   => 'invo',
            ]
        );
    }
}

class SomeComponent
{
    protected $connection;

    /**
     * Sets the connection externally
     *
     * @param Connection $connection
     */
    public function setConnection(Connection $connection)
    {
        $this->connection = $connection;
    }

    public function someDbTask()
    {
        $connection = $this->connection;

        // ...
    }
}

$some = new SomeComponent();

// Pass the connection defined in the registry
$some->setConnection(Registry::getConnection());

$some->someDbTask();
```

现在,让我们假设我们在组件必须实现两个方法,第一个总是需要创建一个新的连接,第二总是需要使用一个共享的连接
```php
<?php

class Registry
{
    protected static $connection;

    /**
     * Creates a connection
     *
     * @return Connection
     */
    protected static function createConnection(): Connection
    {
        return new Connection(
            [
                'host'     => 'localhost',
                'username' => 'root',
                'password' => 'secret',
                'dbname'   => 'invo',
            ]
        );
    }

    /**
     * Creates a connection only once and returns it
     *
     * @return Connection
     */
    public static function getSharedConnection(): Connection
    {
        if (self::$connection === null) {
            self::$connection = self::createConnection();
        }

        return self::$connection;
    }

    /**
     * Always returns a new connection
     *
     * @return Connection
     */
    public static function getNewConnection(): Connection
    {
        return self::createConnection();
    }
}

class SomeComponent
{
    protected $connection;

    /**
     * Sets the connection externally
     *
     * @param Connection $connection
     */
    public function setConnection(Connection $connection)
    {
        $this->connection = $connection;
    }

    /**
     * This method always needs the shared connection
     */
    public function someDbTask()
    {
        $connection = $this->connection;

        // ...
    }

    /**
     * This method always needs a new connection
     *
     * @param Connection $connection
     */
    public function someOtherDbTask(Connection $connection)
    {

    }
}

$some = new SomeComponent();

// This injects the shared connection
$some->setConnection(
    Registry::getSharedConnection()
);

$some->someDbTask();

// Here, we always pass a new connection as parameter
$some->someOtherDbTask(
    Registry::getNewConnection()
);
```




### SPL中的ArrayAccess接口
[文档地址](http://php.net/manual/zh/class.arrayaccess.php)
> 实现此接口的对象，可以像访问数组一样访问对象。

```php
<?php
class obj implements arrayaccess {
    private $container = array();
    public function __construct() {
        $this->container = array(
            "one"   => 1,
            "two"   => 2,
            "three" => 3,
        );
    }
    public function offsetSet($offset, $value) {
        if (is_null($offset)) {
            $this->container[] = $value;
        } else {
            $this->container[$offset] = $value;
        }
    }
    public function offsetExists($offset) {
        return isset($this->container[$offset]);
    }
    public function offsetUnset($offset) {
        unset($this->container[$offset]);
    }
    public function offsetGet($offset) {
        return isset($this->container[$offset]) ? $this->container[$offset] : null;
    }
}

$obj = new obj;

var_dump(isset($obj["two"])); // bool(true)
var_dump($obj["two"]); // int(2)
unset($obj["two"]);                         //==> 触发 offsetUnset方法
var_dump(isset($obj["two"]));// bool(false)   ==> 触发 offsetExists方法
$obj["two"] = "A value";                    //==> 触发 offsetSet方法
var_dump($obj["two"]);// string(7) "A value"//==> 触发 offsetGet方法
$obj[] = 'Append 1';
$obj[] = 'Append 2';
$obj[] = 'Append 3';
print_r($obj);
//obj Object
//(
//    [container:obj:private] => Array
//        (
//            [one] => 1
//            [three] => 3
//            [two] => A value
//            [0] => Append 1
//            [1] => Append 2
//            [2] => Append 3
//        )
//)
```

### 

> 由于实现了ArrayAcess接口，所以di可以这样用
> $di["request"] = new \Phalcon\Http\Request();
> 
> 
> 
> 

### 服务的单例模式实现
shared 

/**
 * Check if the service is shared
 */
if shared {
	let sharedInstance = this->_sharedInstance;
	if sharedInstance !== null {
		return sharedInstance;
	}
}

### 事件机制
get(name, parameters)，在获取服务的时候，会判断eventsManager是否存在，若存在就调用事件接口

```
public function get(string! name, parameters = null) -> var
{
    let eventsManager = <ManagerInterface> this->_eventsManager;
    
    // 解析服务前的事件触发
    if typeof eventsManager == "object" {
    	let instance = eventsManager->fire(
    		"di:beforeServiceResolve",
    		this,
    		["name": name, "parameters": parameters]
    	);
    }
    
    
    // code
    // ...
    
    // 解析服务以后的事件触发
    if typeof eventsManager == "object" {
    		eventsManager->fire(
    			"di:afterServiceResolve",
    			this,
    			[
    				"name": name,
    				"parameters": parameters,
    				"instance": instance
    			]
    		);
    }
}

/**
 * 设置内部事件管理器
 * 实际主要是Phalcon\Events\Manager 承担了事件管理器的角色
 * Sets the internal event manager
 */
public function setInternalEventsManager(<ManagerInterface> eventsManager)
{
	let this->_eventsManager = eventsManager;
}
```


接下来，看看Phalcon\Events\Manager的fire内部实现

```
   /**
	 * Fires an event in the events manager causing the active listeners to be notified about it
	 *
	 *<code>
	 *	$eventsManager->fire("db", $connection);
	 *</code>
	 *
	 * @param string eventType 事件类型:
	 * @param object source 事件源，从哪个对象里面触发的事件
	 * @param mixed  data   传的参数
	 * @param boolean cancelable 
	 * @return mixed
	 */
	public function fire(string! eventType, source, data = null, boolean cancelable = true)
	{
		var events, eventParts, type, eventName, event, status, fireEvents;

      // 所有事件
		let events = this->_events;
		if typeof events != "array" {
			return null;
		}

		// eventType中必须有冒号分隔符
		// 像这样的 "di:beforeServiceResolve"
		if !memstr(eventType, ":") {
			throw new Exception("Invalid event type " . eventType);
		}

		let eventParts = explode(":", eventType),
		   // “di”赋值给事件类型type
			type = eventParts[0],
			// 事件名称
			eventName = eventParts[1];

		let status = null;

		// Responses must be traced?
		if this->_collect {
			let this->_responses = null;
		}

		let event = null;

		// 如果di在events中注入了事件
		if fetch fireEvents, events[type] {
        // 如果是事件对象，或者是事件数组
			if typeof fireEvents == "object" || typeof fireEvents == "array" {

				// Create the event context
				// 创建一个事件的上下文
				let event = new Event(eventName, source, data, cancelable);

				// Call the events queue
				// 请求事件队列
				let status = this->fireQueue(fireEvents, event);
			}
		}

		// Check if there are listeners for the event type itself
		if fetch fireEvents, events[eventType] {

			if typeof fireEvents == "object" || typeof fireEvents == "array" {

				// Create the event if it wasn't created before
				if event === null {
					let event = new Event(eventName, source, data, cancelable);
				}

				// Call the events queue
				let status = this->fireQueue(fireEvents, event);
			}
		}

		return status;
	}
	
	
	
	/**
	 * Internal handler to call a queue of events
	 *
	 * @param \SplPriorityQueue|array queue
	 * @param \Phalcon\Events\Event event
	 * @return mixed
	 */
	public final function fireQueue(var queue, <EventInterface> event)
	{
		var status, arguments, eventName, data, iterator, source, handler;
		boolean collect, cancelable;

		if typeof queue != "array" {
			if typeof queue == "object" {
				if !(queue instanceof SplPriorityQueue) {
					throw new Exception(
						sprintf(
							"Unexpected value type: expected object of type SplPriorityQueue, %s given",
							get_class(queue)
						)
					);
				}
			} else {
				throw new Exception("The queue is not valid");
			}
		}

		let status = null, arguments = null;

		// Get the event type
		let eventName = event->getType();
		if typeof eventName != "string" {
			throw new Exception("The event type not valid");
		}

		// Get the object who triggered the event
		let source = event->getSource();

		// Get extra data passed to the event
		let data = event->getData();

		// Tell if the event is cancelable
		let cancelable = (boolean) event->isCancelable();

		// Responses need to be traced?
		let collect = (boolean) this->_collect;

		if typeof queue == "object" {

			// We need to clone the queue before iterate over it
			let iterator = clone queue;

			// Move the queue to the top
			iterator->top();

			while iterator->valid() {

				// Get the current data
				let handler = iterator->current();
				iterator->next();

				// Only handler objects are valid
				if typeof handler == "object" {

					// Check if the event is a closure
					if handler instanceof \Closure {

						// Create the closure arguments
						if arguments === null {
							let arguments = [event, source, data];
						}

						// Call the function in the PHP userland
						let status = call_user_func_array(handler, arguments);

						// Trace the response
						if collect {
							let this->_responses[] = status;
						}

						if cancelable {

							// Check if the event was stopped by the user
							if event->isStopped() {
								break;
							}
						}

					} else {

						// Check if the listener has implemented an event with the same name
						if method_exists(handler, eventName) {

							// Call the function in the PHP userland
							let status = handler->{eventName}(event, source, data);

							// Collect the response
							if collect {
								let this->_responses[] = status;
							}

							if cancelable {

								// Check if the event was stopped by the user
								if event->isStopped() {
									break;
								}
							}
						}
					}
				}
			}

		} else {

			for handler in queue {

				// Only handler objects are valid
				if typeof handler == "object" {

					// Check if the event is a closure
					if handler instanceof \Closure {

						// Create the closure arguments
						if arguments === null {
							let arguments = [event, source, data];
						}

						// Call the function in the PHP userland
						let status = call_user_func_array(handler, arguments);

						// Trace the response
						if collect {
							let this->_responses[] = status;
						}

						if cancelable {

							// Check if the event was stopped by the user
							if event->isStopped() {
								break;
							}
						}

					} else {

						// Check if the listener has implemented an event with the same name
						if method_exists(handler, eventName) {

							// Call the function in the PHP userland
							let status = handler->{eventName}(event, source, data);

							// Collect the response
							if collect {
								let this->_responses[] = status;
							}

							if cancelable {

								// Check if the event was stopped by the user
								if event->isStopped() {
									break;
								}
							}
						}
					}
				}
			}
		}

		return status;
	}
```










### 参考文章
- [phalcon di](https://docs.phalconphp.com/zh/3.2/di)

