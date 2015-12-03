# 环信demo3.0安卓版

### 初始化
见HXSDKHelper

```
 public synchronized boolean onInit(Context context) {

       appContext = this;
       int pid = android.os.Process.myPid();
        String processAppName = getAppName(pid);
        // 如果app启用了远程的service，此application:onCreate会被调用2次
        // 为了防止环信SDK被初始化2次，加此判断会保证SDK被初始化1次
        // 默认的app会在以包名为默认的process name下运行，如果查到的process name不是app的process name就立即返回

        if (processAppName == null ||!processAppName.equalsIgnoreCase("com.easemob.chatuidemo")) {
            Log.e(TAG, "enter the service process!");
            //"com.easemob.chatuidemo"为demo的包名，换到自己项目中要改成自己包名

            // 则此application::onCreate 是被service 调用的，直接返回
            return false;
        }
       //初始化环信SDK
       Log.d("DemoApplication", "Initialize EMChat SDK");
       EMChat.getInstance().init(appContext);

       //获取到EMChatOptions对象
       EMChatOptions options = EMChatManager.getInstance().getChatOptions();
       //默认添加好友时，是不需要验证的，改成需要验证
       options.setAcceptInvitationAlways(false);
       //设置收到消息是否有新消息通知，默认为true
       options.setNotificationEnable(false);
       //设置收到消息是否有声音提示，默认为true
       options.setNoticeBySound(false);
       //设置收到消息是否震动 默认为true
       options.setNoticedByVibrate(false);
       //设置语音消息播放是否设置为扬声器播放 默认为true
       options.setUseSpeaker(false);
    }

    private String getAppName(int pID) {
            String processName = null;
            ActivityManager am = (ActivityManager) this
                    .getSystemService(ACTIVITY_SERVICE);
            List l = am.getRunningAppProcesses();
            Iterator i = l.iterator();
            PackageManager pm = this.getPackageManager();
            while (i.hasNext()) {
                ActivityManager.RunningAppProcessInfo info = (ActivityManager.RunningAppProcessInfo) (i
                        .next());
                try {
                    if (info.pid == pID) {
                        CharSequence c = pm.getApplicationLabel(pm
                                .getApplicationInfo(info.processName, PackageManager.GET_META_DATA));
                        processName = info.processName;
                        return processName;
                    }
                } catch (Exception e) {
                }
            }
            return processName;
    }

```
### 注册
见RegisterActivity，注意用户名会自动转为小写字母(强烈建议开发者通过后台调用rest接口去注册环信id，客户端注册方法不提倡使用)

```
new Thread(new Runnable() {
    public void run() {
      try {
         // 调用sdk注册方法
         EMChatManager.getInstance().createAccountOnServer(username, pwd);
      } catch (final Exception e) {
      }
   }
}).start();
```

### 登陆
见LoginActivity（必须在客户端调登陆方法）

```
//调用sdk登陆方法登陆聊天服务器
EMChatManager.getInstance().login(username, password, new EMCallBack() {

    @Override
    public void onSuccess() {
    // TODO Auto-generated method stub
    }

    @Override
    public void onProgress(int progress, String status) {
    // TODO Auto-generated method stub
    }

    @Override
    public void onError(int code, String message) {
    // TODO Auto-generated method stub
    }
});
```

### 注册listener
接收聊天消息,回执消息，透传消息，好友同意，好友请求等监听变化：见MainActivity.java

* 注册一个接收消息的BroadcastReceiver

```
msgReceiver = new NewMessageBroadcastReceiver();
IntentFilter intentFilter = new IntentFilter(EMChatManager.getInstance().getNewMessageBroadcastAction());
intentFilter.setPriority(3);
registerReceiver(msgReceiver, intentFilter);
```

* 注册一个ack回执消息的BroadcastReceiver

```
IntentFilter ackMessageIntentFilter = new IntentFilter(EMChatManager.getInstance().getAckMessageBroadcastAction());
ackMessageIntentFilter.setPriority(3);
registerReceiver(ackMessageReceiver, ackMessageIntentFilter);
```

* 注册一个透传消息的BroadcastReceiver（透传消息不会存入db，可以理解为一条不存db的，不显示在ui的普通消息，或者理解成就是发送的一条指令）

```
IntentFilter cmdMessageIntentFilter = new IntentFilter(EMChatManager.getInstance().getCmdMessageBroadcastAction());
cmdMessageIntentFilter.setPriority(3);
registerReceiver(cmdMessageReceiver, cmdMessageIntentFilter);
```

* 注册一个好友请求同意好友请求等的BroadcastReceiver

```
IntentFilter inviteIntentFilter = new IntentFilter(EMChatManager.getInstance().getContactInviteEventBroadcastAction());
registerReceiver(contactInviteReceiver, inviteIntentFilter);
```

* 监听联系人的变化等

```
EMContactManager.getInstance().setContactListener(new MyContactListener());
```

* 注册一个监听连接状态的listener

```
EMChatManager.getInstance().addConnectionListener(new MyConnectionListener());
```

* 发送聊天消息并显示

```
//创建一个消息(本条信息是一条文本，可以通过EMMessage.Type选择其他类型)
EMMessage msg = EMMessage.createSendMessage(EMMessage.Type.TXT);
//设置消息的接收方
msg.setReceipt(username);
//设置消息内容。本消息类型为文本消息。
TextMessageBody body = new TextMessageBody(tvMsg.getText().toString());
msg.addBody(body);
try {
   //发送消息
   EMChatManager.getInstance().sendMessage(msg,callback);
} catch (Exception e) {
   e.printStackTrace();
}
```

* 接收聊天消息并显示

```
private class NewMessageBroadcastReceiver extends BroadcastReceiver {
    @Override
    public void onReceive(Context context, Intent intent) {
        //消息id
        String msgId = intent.getStringExtra("msgid");
        ......
        ......
        ......

        //注销广播，否则在ChatActivity中会收到这个广播
        abortBroadcast();
    }
}
```

### 消息回执BroadcastReceiver
见MainActivity.java

```
private BroadcastReceiver ackMessageReceiver = new BroadcastReceiver() {

    @Override
    public void onReceive(Context context, Intent intent) {
        //消息id
        String msgId = intent.getStringExtra("msgid");
        ......
        ......
        ......
        abortBroadcast();
	}
};
```

### 透传消息BroadcastReceiver
见MainActivity.java

```
private BroadcastReceiver cmdMessageReceiver = new BroadcastReceiver() {

    @Override
    public void onReceive(Context context, Intent intent) {
        //消息id
        String msgId = intent.getStringExtra("msgid");
        ......
        ......
        ......
        abortBroadcast();
	}
};
```

### 联系人变化listener
见MainActivity.java

```
private class MyContactListener implements EMContactListener{
     @Override
     public void onContactAdded(List<String> usernameList) {
     }
     @Override
     public void onContactDeleted(List<String> usernameList) {
     }
}
```

### 监听连接状态和账号多处登录被迫下线
见MainActivity.java

```
    private class MyConnectionListener implements EMConnectionListener {
        @Override
		public void onConnected() {
		}
		@Override
		public void onDisconnected(final int error) {
			runOnUiThread(new Runnable() {

				@Override
				public void run() {
					if(error == EMError.USER_REMOVED){
						// 显示帐号已经被移除
					}else if (error == EMError.CONNECTION_CONFLICT) {
						// 显示帐号在其他设备登陆
					} else {
		                 //"连接不到聊天服务器"
					}
				}
			});
		}
    }

```

### 退出登陆

```
EMChatManager.getInstance().logout();//此方法为同步方法
```

或者

```
EMChatManager.getInstance().logout(new EMCallBack(){})//此方法为异步方法
//后文中，如遇到new EMCallBack()即为new EMCallBack(){}
```

# EaseUI使用指南
EaseUI是一个UI库，封装了IM功能常用的控件、fragment等等，旨在帮助开发者快速集成环信sdk
easeui及demo的github下载地址为：https://github.com/easemob/easeui；https://github.com/easemob/sdkdemoapp3.0_android

*注意： 因为这是一个ui库，后续很可能还会继续改动，新旧版本在api的兼容上不会像im sdk那样绝对的兼容。*


该模块包含了一个接近微信的完整的聊天app的所有功能, 包括发文字，表情，图片，语音，文件，视频，位置，群聊，登录，注册，退出登录等。


主要fragment

* EaseConversationList - 聊天页面，最主要的fragment
* EaseContactListFragment - 联系人页面
* EaseConversationListFragment - 会话列表页面


主要控件

* EaseTitleBar - 标题栏
* EaseChatMessageList - 聊天消息列表控件
* EaseConversationList - 会话列表控件
* EaseContactList - 联系人列表页面
* EaseChatInputMenu - 聊天输入菜单栏
* 其他子控件，后文会做详细介绍
这里对聊天页面几个控件做简单图示：
![1](http://docs.easemob.com/lib/exe/fetch.php?media=start:200androidcleintintegration:chat_widget_simple.png)

## 正式使用

### 初始化

使用EaseUI需要先调用初始化方法，在Application的oncreate里调用初始化

`EaseUI.getInstance().init(applicationContext)`

### 启动聊天会话fragment

new出EaseChatFragment或者其子类， 调用setArguments方法传入chatType(会话类型)和userId(用户或群id)，通过getSupportFragmentManager()启动fragment

```
 //new出EaseChatFragment或其子类的实例
 EaseChatFragment chatFragment = new EaseChatFragment();
 //传入参数
 Bundle args = new Bundle();
 args.putInt(EaseConstant.EXTRA_CHAT_TYPE, EaseConstant.CHATTYPE_GROUP);
 args.putString(EaseConstant.EXTRA_USER_ID, "zw123");
 chatFragment.setArguments(args);
 getSupportFragmentManager().beginTransaction().add(R.id.container, chatFragment).commit();
 ```

### 聊天会话fragment功能扩展

EaseUI提供现成的聊天相关能直接使用的fragment，app无需任何改动，可以直接使用(可参考simpledemo)。也可以通过继承easeui里的fragment亦或是直接使用easeui里提供的widget扩展自己的需要的功能

#### 自组合widget实现会话fragment

自组合控件时可参考EaseChatFragment类，注意：使用easeui中的自定义控件时，如果需要xml中设置其属性(具体哪些属性可查看attrs文件)，务必在xml根节点中加上xmlns:easemob=“http://schemas.android.com/apk/res-auto”
* 标题栏控件EaseTitleBar使用


在xml中声明标题栏控件，可以在xml直接设置标题内容，左右图片，在java文件中亦可以设置这些属性以及相关的点击事件。 示例：
```
<com.easemob.easeui.widget.EaseTitleBar
        android:id="@+id/title_bar"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        easemob:titleBarLeftImage="@drawable/ease_mm_title_back" />

titleBar = (EaseTitleBar) getView().findViewById(R.id.title_bar);
titleBar.setTitle("张建国");
titleBar.setRightImageResource(R.drawable.ease_mm_title_remove);
```
* 聊天消息列表控件EaseChatMessageList使用
![1](http://docs.easemob.com/lib/exe/fetch.php?media=start:200androidcleintintegration:chat_message_list.png)

EaseChatMessageList默认包含文字、表情、图片、语音、视频、文件消息的显示。
使用该控件，可以在xml中设置其item(chatrow)的背景图片，是否显示用户头像、昵称等属性
```
<com.easemob.easeui.widget.EaseChatMessageList
            android:id="@+id/message_list"
            android:layout_width="match_parent"
            android:layout_height="match_parent"
            easemob:msgListShowUserNick="false"
             />
 ```

##### 常用API

```
messageList = (EaseChatMessageList) getView().findViewById(R.id.message_list);
//初始化messagelist
messageList.init(toChatUsername, chatType, null);
//设置item里的控件的点击事件
messageList.setItemClickListener(new EaseChatMessageList.MessageListItemClickListener() {

            @Override
            public void onUserAvatarClick(String username) {
                //头像点击事件
            }

            @Override
            public void onResendClick(final EMMessage message) {
                //重发消息按钮点击事件
            }

            @Override
            public void onBubbleLongClick(EMMessage message) {
                //气泡框长按事件
            }

            @Override
            public boolean onBubbleClick(EMMessage message) {
                //气泡框点击事件，easeui有默认实现这个事件，如果需要覆盖，return值要返回true
                return false;
            }
        });
//获取下拉刷新控件
swipeRefreshLayout = messageList.getSwipeRefreshLayout();
//刷新消息列表
messageList.refresh();
messageList.refreshSeekTo(position);
messageList.refreshSelectLast();
```

* 自定义EaseChatMessageList的item(chatrow)
![2](http://docs.easemob.com/lib/exe/fetch.php?media=start:200androidcleintintegration:chat_message_list_item.png)

在easeui里，每一个messagelist的item称为一个chatrow，开发者可以覆盖默认的chatrow或者自定义自己的chatrow
简单示例(具体可参考的chatfragment和easechatfragment类)：
```
private final class CustomChatRowProvider implements EaseCustomChatRowProvider {
        @Override
        public int getCustomChatRowTypeCount() {
            //音、视频通话发送、接收共4种
            return 4;
        }

        @Override
        public int getCustomChatRowType(EMMessage message) {
            if(message.getType() == EMMessage.Type.TXT){
                //语音通话类型
                if (message.getBooleanAttribute(Constant.MESSAGE_ATTR_IS_VOICE_CALL, false)){
                    return message.direct == EMMessage.Direct.RECEIVE ? MESSAGE_TYPE_RECV_VOICE_CALL : MESSAGE_TYPE_SENT_VOICE_CALL;
                }else if (message.getBooleanAttribute(Constant.MESSAGE_ATTR_IS_VIDEO_CALL, false)){
                    //视频通话
                    return message.direct == EMMessage.Direct.RECEIVE ? MESSAGE_TYPE_RECV_VIDEO_CALL : MESSAGE_TYPE_SENT_VIDEO_CALL;
                }
            }
            return 0;
        }

        @Override
        public EaseChatRow getCustomChatRow(EMMessage message, int position, BaseAdapter adapter) {
            if(message.getType() == EMMessage.Type.TXT){
                // 语音通话,  视频通话
                if (message.getBooleanAttribute(Constant.MESSAGE_ATTR_IS_VOICE_CALL, false) ||
                    message.getBooleanAttribute(Constant.MESSAGE_ATTR_IS_VIDEO_CALL, false)){
                    //ChatRowVoiceCall为一个继承自EaseChatRow的类
                    return new ChatRowVoiceCall(getActivity(), message, position, adapter);
                }
            }
            return null;
        }
    }
//初始化的时候把provider传入
messageList.init(toChatUsername, chatType, new CustomChatRowProvider());
```

* 底部操作栏EaseChatInputMenu使用
![4](EaseUI是一个UI库，封装了IM功能常用的控件、fragment等等，旨在帮助开发者快速集成环信sdk
easeui及demo的github下载地址为：https://github.com/easemob/easeui；https://github.com/easemob/sdkdemoapp3.0_android
注意： 因为这是一个ui库，后续很可能还会继续改动，新旧版本在api的兼容上不会像im sdk那样绝对的兼容。

)

EaseChatInputMenu包含3个控件:EaseChatPrimaryMenu(主菜单栏，包含文字输入、发送等功能)
EaseChatExtendMenu(扩展栏，点击加号按钮出来的小宫格的菜单栏)
以及EaseEmojiconMenu(表情栏)
xml中使用示例
```
<com.easemob.easeui.widget.EaseChatInputMenu
        android:id="@+id/input_menu"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:layout_alignParentBottom="true" />
```
##### EaseChatPrimaryMenu扩展
考虑的EaseChatPrimaryMenu变动的复杂性，easeui为提供具体修改的api，但是提供了一个api让开发者替换成自己写的控件

```
    /**
     * 设置自定义的表情栏，该控件需要继承自EaseEmojiconMenuBase，
     * 以及回调你想要回调出去的事件给设置的EaseEmojiconMenuListener
     * @param customEmojiconMenu
     */
    public void setCustomEmojiconMenu(EaseEmojiconMenuBase customEmojiconMenu){
        this.emojiconMenu = customEmojiconMenu;
    }
```
通过inputmenu对象调用此方法就行。

##### EaseChatExtendMenu扩展
扩展菜单栏默认是不包含任何按钮的(EaseChatFragment中会默认加上拍照、图片、位置)，通过调用registerExtendMenuItem就行

```
inputMenu = (EaseChatInputMenu) getView().findViewById(R.id.input_menu);
//注册底部菜单扩展栏item
//传入item对应的文字，图片及点击事件监听，extendMenuItemClickListener实现EaseChatExtendMenuItemClickListener
inputMenu.registerExtendMenuItem(R.string.attach_video, R.drawable.em_chat_video_selector, ITEM_VIDEO, extendMenuItemClickListener);
inputMenu.registerExtendMenuItem(R.string.attach_file, R.drawable.em_chat_file_selector, ITEM_FILE, extendMenuItemClickListener);
...
```

##### EaseEmojiconMenu扩展:
EaseEmojiconMenu默认包含一套默认的表情，通过api可替换或增加表情 两种方法： 1、调用inputmenu init方法的时候传入表情组列表，传null会显示一套默认表情，inputMenu.init(null); * * 通过获取到EmojiconMenu对象添加或删除表情组。

''((EaseEmojiconMenu)inputMenu.getEmojiconMenu()).addEmojiconGroup(EmojiconExampleGroupData.getData());
((EaseEmojiconMenu)inputMenu.getEmojiconMenu()).removeEmojiconGroup(1);''
如果想完全自己实现这套表情的布局，通过调用setCustomEmojiconMenu实现

```
    /**
     * 设置自定义的表情栏，该控件需要继承自EaseEmojiconMenuBase，
     * 以及回调你想要回调出去的事件给设置的EaseEmojiconMenuListener
     * @param customEmojiconMenu
     */
    public void setCustomEmojiconMenu(EaseEmojiconMenuBase customEmojiconMenu){
        this.emojiconMenu = customEmojiconMenu;
    }
```

整体代码示例：

```
inputMenu = (EaseChatInputMenu) getView().findViewById(R.id.input_menu);
//注册底部菜单扩展栏item
//传入item对应的文字，图片及点击事件监听，extendMenuItemClickListener实现EaseChatExtendMenuItemClickListener
inputMenu.registerExtendMenuItem(R.string.attach_video, R.drawable.em_chat_video_selector, ITEM_VIDEO, extendMenuItemClickListener);
inputMenu.registerExtendMenuItem(R.string.attach_file, R.drawable.em_chat_file_selector, ITEM_FILE, extendMenuItemClickListener);

//初始化，此操作需放在registerExtendMenuItem后
inputMenu.init();
//设置相关事件监听
inputMenu.setChatInputMenuListener(new ChatInputMenuListener() {

            @Override
            public void onSendMessage(String content) {
                // 发送文本消息
                sendTextMessage(content);
            }

            @Override
            public boolean onPressToSpeakBtnTouch(View v, MotionEvent event) {
                ////把touch事件传入到EaseVoiceRecorderView 里进行录音
                return voiceRecorderView.onPressToSpeakBtnTouch(v, event, new EaseVoiceRecorderCallback() {
                    @Override
                    public void onVoiceRecordComplete(String voiceFilePath, int voiceTimeLength) {
                        // 发送语音消息
                        sendVoiceMessage(voiceFilePath, voiceTimeLength);
                    }
                });
            }
        });
  ```
更多api及使用请参考api文档及demo
[EaseUI使用指南](http://docs.easemob.com/doku.php?id=start:200androidcleintintegration:170easeuiuseguide)
