# Logging

- [Introduction](#introduction)
- [Configuration](#configuration)
    - [Building Log Stacks](#building-log-stacks)
- [Writing Log Messages](#writing-log-messages)
    - [Writing To Specific Channels](#writing-to-specific-channels)
- [Customizing Monolog For Channels](#customizing-monolog-for-channels)
- [Creating Custom Channels](#creating-custom-channels)

<a name="introduction"></a>
## Introduction

To help you learn more about what's happening within your application, Laravel provides robust logging services that allow you to log messages to files, the system error log, and even to Slack to notify your entire team.

Under the hood, Laravel utilizes the [Monolog](https://github.com/Seldaek/monolog) library, which provides support for a variety of powerful log handlers. Laravel makes it a cinch to configure these handlers, allowing you to mix and match them to customize your application's log handling.

<a name="configuration"></a>
## Configuration

All of the configuration for your application's logging system is housed in the `config/logging.php` configuration file. This file allows you to configure your application's log channels, so be sure to review each of the available channels and their options. Of course, we'll review a few common options below.

By default, Laravel will use the `stack` channel when logging messages. The `stack` channel is used to aggregate multiple log channels into a single channel. For more information on building stacks, check out the [documentation below](#building-log-stacks).

#### Configuring The Channel Name

By default, Monolog is instantiated with a "channel name" that matches the current environment, such as `production` or `local`. To change this value, add a `name` option to your channel's configuration:

    'stack' => [
        'driver' => 'stack',
        'name' => 'channel-name',
        'channels' => ['single', 'slack'],
    ],

#### Configuring The Slack Channel

The `slack` channel requires a `url` configuration option. This URL should match a URL for an [incoming webhook](https://slack.com/apps/A0F7XDUAZ-incoming-webhooks) that you have configured for your Slack team.

<a name="building-log-stacks"></a>
### Building Log Stacks

As previously mentioned, the `stack` driver allows you to combine multiple channels into a single log channel. To illustrate how to use log stacks, let's take a look at an example configuration that you might see in a production application:

    'channels' => [
        'stack' => [
            'driver' => 'stack',
            'channels' => ['syslog', 'slack'],
        ],

        'syslog' => [
            'driver' => 'syslog',
            'level' => 'debug',
        ],

        'slack' => [
            'driver' => 'slack',
            'url' => env('LOG_SLACK_WEBHOOK_URL'),
            'username' => 'Laravel Log',
            'emoji' => ':boom:',
            'level' => 'critical',
        ],
    ],

Let's dissect this configuration. First, notice our `stack` channel aggregates two other channels via its `channels` option: `syslog` and `slack`. So, when logging messages, both of these channels will have the opportunity to log the message.

#### Log Levels

Take note of the `level` configuration option present on the `syslog` and `slack` channel configurations in the example above. This option determines the minimum "level" a message must be in order to be logged by the channel. Monolog, which powers Laravel's logging services, offers all of the log levels defined in the [RFC 5424 specification](https://tools.ietf.org/html/rfc5424): **emergency**, **alert**, **critical**, **error**, **warning**, **notice**, **info**, and **debug**.

So, imagine we log a message using the `debug` method:

    Log::debug('An informational message.');

Given our configuration, the `syslog` channel will write the message to the system log; however, since the error message is not `critical` or above, it will not be sent to Slack. However, if we log an `emergency` message, it will be sent to both the system log and Slack since the `emergency` level is above our minimum level threshold for both channels:

    Log::emergency('The system is down!');

<a name="writing-log-messages"></a>
## Writing Log Messages

You may write information to the logs using the `Log` [facade](/docs/{{version}}/facades). As previously mentioned, the logger provides the eight logging levels defined in the [RFC 5424 specification](https://tools.ietf.org/html/rfc5424): **emergency**, **alert**, **critical**, **error**, **warning**, **notice**, **info** and **debug**:

    Log::emergency($message);
    Log::alert($message);
    Log::critical($message);
    Log::error($message);
    Log::warning($message);
    Log::notice($message);
    Log::info($message);
    Log::debug($message);

So, you may call any of these methods to log a message for the corresponding level. By default, the message will be written to the default log channel as configured by your `config/logging.php` configuration file:

    <?php

    namespace App\Http\Controllers;

    use App\User;
    use Illuminate\Support\Facades\Log;
    use App\Http\Controllers\Controller;

    class UserController extends Controller
    {
        /**
         * Show the profile for the given user.
         *
         * @param  int  $id
         * @return Response
         */
        public function showProfile($id)
        {
            Log::info('Showing user profile for user: '.$id);

            return view('user.profile', ['user' => User::findOrFail($id)]);
        }
    }

#### Contextual Information

An array of contextual data may also be passed to the log methods. This contextual data will be formatted and displayed with the log message:

    Log::info('User failed to login.', ['id' => $user->id]);

<a name="writing-to-specific-channels"></a>
### Writing To Specific Channels

Sometimes you may wish to log a message to a channel other than your application's default channel. You may use the `channel` method on the `Log` facade to retrieve and log to any channel defined in your configuration file:

    Log::channel('slack')->info('Something happened!');

If you would like to create an on-demand logging stack consisting of multiple channels, you may use the `stack` method:

    Log::stack(['single', 'slack'])->info('Something happened!');

<a name="customizing-monolog-for-channels"></a>
## Customizing Monolog For Channels

Sometimes you may need complete control over how Monolog is configured for an existing channel. For example, you may want to configure a custom Monolog `FormatterInterface` implementation for a given channel's handlers.

To get started, define a `tap` array on the channel's configuration. The `tap` array should contain a list of classes that should have an opportunity to customize (or "tap" into) the Monolog instance after it is created:

    'single' => [
        'driver' => 'single',
        'tap' => [App\Logging\CustomizeFormatter::class],
        'path' => storage_path('logs/laravel.log'),
        'level' => 'debug',
    ],

Once you have configured the `tap` option on your channel, you're ready to define the class that will customize your Monolog instance. This class only needs a single method: `__invoke`, which receives the Monolog instance:

    <?php

    namespace App\Logging;

    class CustomizeFormatter
    {
        /**
         * Customize the given Monolog instance.
         *
         * @param  \Monolog\Logger
         * @return void
         */
        public function __invoke($monolog)
        {
            foreach ($monolog->getHandlers() as $handler) {
                $handler->setFormatter(...);
            }
        }
    }

> {tip} All of your "tap" classes are resolved by the [service container](/docs/{{version}}/container), so any constructor dependencies they require will automatically be injected.

<a name="creating-custom-channels"></a>
## Creating Custom Channels

If you would like to define an entirely custom channel in which you have full control over Monolog's instantiation and configuration, you may specify a `custom` driver type in your `config/logging.php` configuration file. Additionally, your configuration should include a `via` option which specifies the class that should be invoked to create the Monolog instance:

    'channels' => [
        'custom' => [
            'driver' => 'custom',
            'via' => App\Logging\CreateCustomLogger::class,
        ],
    ],

Once you have configured the `custom` channel, you're ready to define the class that will create your Monolog instance. This class only needs a single method: `__invoke`, which should return the Monolog instance:

    <?php

    namespace App\Logging;

    use Monolog\Logger;

    class CreateCustomLogger
    {
        /**
         * Create a custom Monolog instance.
         *
         * @param  array  $config
         * @return \Monolog\Logger
         */
        public function __invoke(array $config)
        {
            return new Logger(...);
        }
    }
