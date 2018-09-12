## 无缝监听安装手机通知栏推送消息

### 1.添加<inter-filter>
```
 <service
            android:name=".service.NotifyService"
            android:label="@string/app_name"
            android:permission="android:permission.BIND_NOTIFICATION_LISTENER_SERVICE">
            <intent-filter>
                <action android:name="android.service.notification.NotificationListenerService" />
            </intent-filter>
        </service>
```
### 2.打开通知监听设置

```
try {
        Intent intent;
        if (android.os.Build.VERSION.SDK_INT >= android.os.Build.VERSION_CODES.LOLLIPOP_MR1) {
            intent = new Intent(Settings.ACTION_NOTIFICATION_LISTENER_SETTINGS);
        } else {
            intent = new Intent("android.settings.ACTION_NOTIFICATION_LISTENER_SETTINGS");
        }
        startActivity(intent);
    } catch (Exception e) {
        e.printStackTrace();
    }

```
### 3.然后重写一下三个方法：
- onNotificationPosted(StatusBarNotification sbn, RankingMap rankingMap) ：当有新通知到来时会回调；
- onNotificationRemoved(StatusBarNotification sbn) ：当有通知移除时会回调；
- onListenerConnected() ：当 NotificationListenerService 是可用的并且和通知管理器连接成功时回调。

而我们要获取通知栏的信息则需要在onNotificationPosted方法内获取 ，之前在网上查了一些文章有的通过判断API是否大于18来采取不同的办法，大致是=18则利用反射获取 Notification的内容,>18则通过Notification.extras来获取通知内容，而经测试在部分安卓手机上即使API>18 Notification.extras是等于null的。因此不能通过此方法获取通知栏信息

### 4.获取消息通知
```
String content = null;
        if (sbn.getNotification().tickerText != null) {
            content = sbn.getNotification().tickerText.toString();
        }

```
在onNotificationPosted方法内通过上面的方法即可获取部分手机的通知栏消息，但是但是重点来了，在部分手机上，比如华为荣耀某系列sbn.getNotification().tickerText == null，经调试发现仅在StatusBarNotification对象内部的一个view的成员变量上有推送消息内容，因此不得不用上了反射去获取view上的内容

```
private Map<String, Object> getNotiInfo(Notification notification) {
       int key = 0;
       if (notification == null)
           return null;
       RemoteViews views = notification.contentView;
       if (views == null)
           return null;
       Class secretClass = views.getClass();

       try {
           Map<String, Object> text = new HashMap<>();

           Field outerFields[] = secretClass.getDeclaredFields();
           for (int i = 0; i < outerFields.length; i++) {
               if (!outerFields[i].getName().equals("mActions"))
                   continue;

               outerFields[i].setAccessible(true);

               ArrayList<Object> actions = (ArrayList<Object>) outerFields[i].get(views);
               for (Object action : actions) {
                   Field innerFields[] = action.getClass().getDeclaredFields();
                   Object value = null;
                   Integer type = null;
                   for (Field field : innerFields) {
                       field.setAccessible(true);
                       if (field.getName().equals("value")) {
                           value = field.get(action);
                       } else if (field.getName().equals("type")) {
                           type = field.getInt(action);
                       }
                   }
                   // 经验所得 type 等于9 10为短信title和内容，不排除其他厂商拿不到的情况
                   if (type != null && (type == 9 || type == 10)) {
                       if (key == 0) {
                           text.put("title", value != null ? value.toString() : "");
                       } else if (key == 1) {
                           text.put("text", value != null ? value.toString() : "");
                       } else {
                           text.put(Integer.toString(key), value != null ? value.toString() : null);
                       }
                       key++;
                   }
               }
               key = 0;

           }
           return text;
       } catch (Exception e) {
           e.printStackTrace();
       }
       return null;
   }


```

那么经过以上方法：先获取sbn.getNotification().tickerText，如果为空，则尝试使用反射获取view上的内容，目前测试了主流机型，暂无任何兼容性问题。
