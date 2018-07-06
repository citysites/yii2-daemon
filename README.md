Daemons system for Yii2
=======================
Extension provides functionality for simple daemons creation and control. There is also a daemons supervisor.

Installation
------------

The preferred way to install this extension is through [composer](http://getcomposer.org/download/).

Either run

```
php composer.phar require --prefer-dist advissmedia/yii2-daemon "*"
```

or add

```
"advissmedia/yii2-daemon": "*"
```

to the require section of your `composer.json` file.

### Setting WatcherDaemon (Supervisor)
WatcherDaemon is the main daemon and provides from box. This daemon check another daemons and run, if it need.
Do the following steps:

1. Create in your console controllers path file for ex. DaemonsSupervisorController.php with following content:
```
<?php

namespace console\controllers;
use \advissmedia\daemon\controllers\WatcherDaemonController;

class DaemonsSupervisorController extends WatcherDaemonController
{
    /**
     * @return array
     */
    protected function getDaemonsList()
    {
        //TODO: modify list, or get it from config, it does not matter
        $daemons = [
            ['className' => 'FirstDaemonController', 'enabled' => true, 'hardKill' => false],
            ['className' => 'AnotherDaemonController', 'enabled' => false, 'hardKill' => true]
        ];
        return $daemons;
    }
}
```
or get this jobs array from file, for example json array
```
<?php

namespace console\controllers;
use \advissmedia\daemon\controllers\WatcherDaemonController;

class DaemonsSupervisorController extends WatcherDaemonController
{
    /**
     * @return array
     */
    protected function getDaemonsList()
    {
        $string = @file_get_contents("/url/or/uri/to/json/file.json");
        if ($string === false || empty($daemons = json_decode($string,true)))
            return [];
        return $daemons;
    }
}
```
And `file.json` format:
```
[
{
    "className":"FirstDaemonController",
	"enabled":"true",
	"hardKill":"false"
},
{
    "className":"AnotherDaemonController",
	"enabled":"false",
	"hardKill":"false"
}
]
```
or get running daemon list from database

2. No one checks the Watcher. Watcher should run continuously. Add it to your crontab:
```
* * * * * /path/to/yii/project/yii watcher-daemon --demonize=1
```
Watcher can't start twice, only one instance can work in the one moment.

Usage
-----
### Create new daemons
1. Create in your console controllers path file {NAME}DaemonController.php with following content:
```
<?php

namespace console\controllers;

use \advissmedia\daemon\DaemonController;

class {NAME}DaemonController extends DaemonController
{
    /**
     * @return array
     */
    protected function defineJobs()
    {
        /*
        TODO: return task list, extracted from DB, queue managers and so on. 
        Extract tasks in small portions, to reduce memory usage.
        */
    }
    /**
     * @return jobtype
     */
    protected function doJob($job)
    {
        /*
        TODO: implement you logic
        Don't forget to mark task as completed in your task source
        */
    }


    const DEF_FIRST_JOB = 1;
    const DEF_SOCOND_JOB = 3;

    /**
     * {@inheritdoc}
     */
    public function init()
    {
        parent::init();
        /*
         * Set type of jobs list can be processed
         */
        $this->isJobsListPersistent = true;
        /*
         * Array connection name for renew connections for daemon process
         */
        $this->connections = ['pgdb', 'sqltdb'];
    }

    /**
     * @return array
     */
    protected function defineJobs()
    {
        /*
        TODO: return task list, extracted from DB, queue managers or jast array. 
        Extract tasks in small portions, to reduce memory usage.
        */

        return [
            ['job_id' => self::DEF_FIRST_JOB],
            ['job_id' => self::DEF_SECOND_JOB]
        ];
    }

    /**
     * @param array $job
     * @return bool
     * @throws \Exception
     * @throws \Throwable
     * @throws \yii\db\Exception
     */
    protected function doJob($job)
    {
        /*
        TODO: implement you logic
        Don't forget to mark task as completed in your task source
        */

        if (array_key_exists('job_id', $job)) {
            switch ($job['job_id']) {
                case self::DEF_FIRST_JOB:
                    $ret = $this->processFirstJob();
                    break;
                case self::DEF_SECOND_JOB:
                    $ret = $this->processSecondJob();
                    break;
                default:
                    $ret = false;
            }
            return $ret;
        }
        return false;
    }
}
```
2. Implement logic. 
3. Add new daemon to daemons list in watcher.

### Working with RabbitMQ (this example needs "videlalvaro/php-amqplib" package)
```
<?php

namespace console\controllers\daemons;

use console\components\controllers\BaseDaemonController;
use PhpAmqpLib\Channel\AMQPChannel;
use PhpAmqpLib\Connection\AMQPStreamConnection;
use PhpAmqpLib\Message\AMQPMessage;

/**
 * Class SomeRabbitQueueController
 */
class SomeRabbitQueueController extends BaseDaemonController
{
    /**
     *  @var $connection AMQPStreamConnection
     */
    protected $connection;

    /**
     *  @var $connection AMQPChannel
     */
    protected $channel;


    /**
     * @return array|bool
     */
    protected function defineJobs()
    {
        $channel = $this->getQueue();
        while (count($channel->callbacks)) {
            try {
                $channel->wait(null, true, 5);
            } catch (\PhpAmqpLib\Exception\AMQPTimeoutException $timeout) {

            } catch (\PhpAmqpLib\Exception\AMQPRuntimeException $runtime) {
                \Yii::error($runtime->getMessage());
                $this->channel = null;
                $this->connection = null;
            }
        }
        return false;
    }

    /**
     * @param AMQPMessage $job
     * @return bool
     * @throws NotSupportedException
     * @SuppressWarnings(PHPMD.UnusedFormalParameter)
     */
    protected function doJob($job)
    {
        $result = false;

        //do somethink here and set $result

        if ($result) {
            $this->ask($job);
        } else {
            $this->nask($job);
        }
        return $result;
    }


    /**
     * @return AMQPChannel
     * @throws InvalidParamException
     */
    protected function getQueue()
    {
        if ($this->channel == null) {
            if ($this->connection == null) {
                if (isset(\Yii::$app->params['rabbit'])) {
                    $rabbit = \Yii::$app->params['rabbit'];
                } else {
                    throw new InvalidParamException('Bad config RabbitMQ');
                }
                $this->connection = new AMQPStreamConnection($rabbit['host'], $rabbit['port'], $rabbit['user'], $rabbit['password']);
            }

            $this->channel = $this->connection->channel();

            $this->channel->exchange_declare($this->exchange, $this->type, false, true, false);

            $args = [];

            if ($this->dlx) {
                $args['x-dead-letter-exchange'] = ['S', $this->exchange];
                $args['x-dead-letter-routing-key'] = ['S',$this->dlx];
            }
            if ($this->max_length) {
                $args['x-max-length'] = ['I', $this->max_length];
            }
            if ($this->max_bytes) {
                $args['x-max-length-bytes'] = ['I', $this->max_bytes];
            }
            if ($this->max_priority) {
                $args['x-max-priority'] = ['I', $this->max_priority];
            }

            list($queue_name, ,) = $this->channel->queue_declare($this->queue_name, false, true, false, false, false, $args);

            foreach ($this->binding_keys as $binding_key) {
                $this->channel->queue_bind($queue_name, $this->exchange, $binding_key);
            }

            $this->channel->basic_consume($queue_name, '', false, false, false, false, [$this, 'doJob']);
        }

        return $this->channel;
    }


    /**
     * @param $job
     */
    protected function ask($job)
    {
        $job->delivery_info['channel']->basic_ack($job->delivery_info['delivery_tag']);
    }

    /**
     * @param $job
     */
    protected function nask($job)
    {
        $job->delivery_info['channel']->basic_nack($job->delivery_info['delivery_tag']);
    }
}
```
### Daemon settings (propeties)
In your daemon you can override parent properties:
* `$demonize` - if 0 daemon is not running as daemon, only as simple console application. It's needs for debug.
* `$memoryLimit` - if daemon reach this limit - daemon stop work. It prevent memory leaks. After stopping WatcherDaemon run this daemon again.
* `$sleep` - delay between checking for new task, daemon will not sleep if task list is full.
* `$pidDir` - dir where daemons pids is located
* `$logDir` - dir where daemons logs is located
* `$isMultiInstance` - this option allow daemon create self copy for each task. That is, the daemon can simultaneously perform multiple tasks. This is useful when one task requires some time and server resources allows perform many such task.
* `$maxChildProcesses` - only if `$isMultiInstance=true`. The maximum number of daemons instances. If the maximum number is reached - the system waits until at least one child process to terminate.
* `$isJobsListPersistent` - change jobs list processing method. If false job list processing always while have job and free worker. If true - jobs list processing from first job to last and sleep workers for sleep time after which worker starts again with the processing of the first job.   
### Daemon control
Daemon catch a some signal and do some action. Below are the list of catch signals and the actions of the daemon when they are received:
* **SIGINT**, **SIGTERM** - daemon shutdown, break from loop write log data and closure used resources.
* **SIGCHLD** - shutdown all child workers, main daemon process still work.
* **SIGUSR1** - write to linux syslog information about daemon uptime.
### Logger settings
If you want to change logging preferences, you may override the function initLogger. Example:
```
    /**
     * Adjusting logger. You can override it.
     */
    protected function initLogger()
    {

        $targets = \Yii::$app->getLog()->targets;
        foreach ($targets as $name => $target) {
            $target->enabled = false;
        }
        $config = [
            'levels' => ['error', 'warning', 'trace', 'info'],
            'logFile' => \Yii::getAlias($this->logDir) . DIRECTORY_SEPARATOR . $this->shortClassName() . '.log',
            'logVars'=>[], // Don't log all variables
            'exportInterval'=>1, // Write each message to disk
            'except' => [
                'yii\db\*', // Don't include messages from db
            ],
        ];
        $targets['daemon'] = new \yii\log\FileTarget($config);
        \Yii::$app->getLog()->targets = $targets;
        \Yii::$app->getLog()->init();
        // Flush each message
        \Yii::$app->getLog()->flushInterval = 1;
    }
```
    
    
### Installing Proctitle on PHP < 5.5.0

You will need proctitle extension on your server to be able to isntall yii2-daemon. For a debian 7 you can use these commands:

```
# pecl install channel://pecl.php.net/proctitle-0.1.2
# echo "extension=proctitle.so" > /etc/php5/mods-available/proctitle.ini
# php5enmod proctitle
```
