# yii2-notifications
Notification system for Yii 2+

# Usage

Default storage is your database. For that you need to insert migrations from `src/migrations` folder.

To insert those migrations you need to run migration command:

```bash
yii migrate --migrationPath=@vendor/dvamigos/yii2-notifications/src/migrations
```

This will create `notifications` table in your database. If you don't plan to store your notifications in database then this step is not necessary.

Add notification component configuration in your application config:

```php
[
    'components' => [
        'notifications' => [
            'class' => '\dvamigos\Yii2\Notifications\NotificationManager',
            'types' => [
                'text' => [
                     'my_notification' => 'This is my notification'
                ]
            ]
        ]
    ]
]

```

Then in your code you can push notifications directly using:

```php
Yii::$app->notifications->push('my_notification');
```

This will save the notification for current logged in user.

## Notification types

You can define arbitrary number of types for your notification to store as much as data per notification as needed.

Below is an example of a notification having a `title` and a `message`.

```php
[
    'components' => [
        'notifications' => [
            'class' => '\dvamigos\Yii2\Notifications\NotificationManager',
            'types' => [
                'new_user' => [
                    'text' => [
                        'title' => 'New user created!',
                        'message' => 'New user {username} is created.'
                    ],
                    'default' => [
                        'username' => ''
                    ]
                ]
            ]
        ]
    ]
]

```

Field `{username}` will be replaced from data passed to notification on its creation. 

To pass data just use:

```php
Yii::$app->notifications->push('new_user', [
    'username' => 'JohnDoe94'
]);
```

You do not have to always pass every necessary key into data. If you do not pass required key it's value will be taken from `'default'` of that
notification.

You can also use this array to pass any arbitrary information which can be serialized into JSON string, effectively
allowing you to store any required data in order to display or use your notification.

## Using in models/controllers/components

You can use `PushNotification`, `UpdateNotification` or `ReplaceNotification` classes inside your every component which has `events()` function.

To include `events()` function use `EventListAwareTrait`.

For example to set it inside a model you can define following:

```php
public function events()
{
    self::EVENT_AFTER_INSERT => [
        new \dvamigos\Yii2\Notifications\events\PushNotification([
            'type' => 'my_notification',
            'data' => ['my_data' => 1]
        ])
    ]
}
```

Types can be resolved later using:
```php
public function events()
{
    self::EVENT_AFTER_INSERT => [
        new \dvamigos\Yii2\Notifications\events\PushNotification([
            'type' => function(PushNotification $n) {
                return 'my_type';
            },
            'data' => function(PushNotification $n) {
                return ['my_key' => $this->getPrimaryKey()];
            }
        ])
    ]
}
```

If you wish to use Behavior approach, that is also available via `NotificationBehavior` class.

To use that class you can simply add in your model/component which supports behaviors:

```php
public function behaviors() 
{
    return [
        'notification' => [
            'class' => \dvamigos\Yii2\Notifications\events\NotificationBehavior::class,
            'events' => [
                self::EVENT_AFTER_INSERT => [
                    [
                       'class' => \dvamigos\Yii2\Notifications\events\PushNotification,
                       'type' => function(PushNotification $n) {
                            return 'my_type';
                        },
                        'data' => function(PushNotification $n) {
                            return ['my_key' => $this->getPrimaryKey()];
                        }
                    ]
                ]
            ]
        ]
    ];
}
```

If you want to encapsulate your own logic, then extending `\dvamigos\Yii2\Notifications\events\PushNotification` with your own class is also a possibility.

Example:
```php
class MyNotification extends \dvamigos\Yii2\Notifications\events\PushNotification {
    public $type = 'my_notification_type';
    
    public function init() {
        $this->data = [$this, 'handleMyData'];
        parent::init();
    }
    
    public function handleMyData(MyNotification $instance, \yii\base\Event $event)
    {
        // You logic for returning data here...
    }
}
```
```php
public function behaviors() 
{
    return [
        'notification' => [
            'class' => \dvamigos\Yii2\Notifications\events\NotificationBehavior::class,
            'events' => [
                self::EVENT_AFTER_INSERT => [
                    MyNotification::class
                ]
            ]
        ]
    ];
}
```

Then you can add your notification simply as:


# Displaying notifications

You can use `NotificationList` widget to display your notifications.

Please note that depending on your use case you will need to configure this widget to suit your needs. This widget assumes that
your component name is `notification` although you can pass a different name during widget creation.

You need to specify template for your notifications using this widget.

Example of a simple list where notifications are defined as.

```php
[
    'components' => [
        'notifications' => [
            'class' => '\dvamigos\Yii2\Notifications\NotificationManager',
            'types' => [
                'new_user' => [
                    'text' => [
                        'title' => 'New user created!',
                        'message' => 'New user {username} is created.'
                    ],
                    'default' => [
                        'username' => '',
                        'fullName' => 'Unknown',
                        'gender' => 'male'
                    ]
                ]
            ]
        ]
    ]
]

```

```php
<?= \dvamigos\Yii2\Notifications\widgets\NotificationList::widget([
    'containerTemplate' => '<ul>{notifications}{emptyText}</ul>',
    'emptyText' => '<li>No notifications available.</li>',
    'itemTemplate' => '
        <li>
            <span class="title">{text.title}</span>
            <span class="message">{text.message}</span>
            <span class="at">{timestamp}</span>
        </li>
    '
]); ?>
```

Custom sections are also supported. And you can define them as:

```php
<?= \dvamigos\Yii2\Notifications\widgets\NotificationList::widget([
    'containerTemplate' => '<ul>{notifications}{emptyText}</ul>',
    'emptyText' => '<li>No notifications available.</li>',
    'itemTemplate' => '
        <li>
            <span class="title">{text.title}</span>
            <span class="message">{text.message}</span>
            <span class="message">User full name: {section.fullName}</span>
        </li>
    ',
    'sections' => [
        'fullName' => function($context) {
            /** @var \dvamigos\Yii2\Notifications\NotificationInterface $n */
            $n = $context['notification'];
            
            if ($n->getType() !== 'new_user') {
                return '';
            }
            
            return $n->getData()['fullName'];
        }
    ]
]); ?>
```

Please refer to `NotificationList` documentation in the code for more information on what is available.

Best practice is to extend this `NotificationList` widget with your own and implement this functionality
based on needs for your own project.