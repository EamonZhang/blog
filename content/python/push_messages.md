---
title: "消息推送"
date: 2021-11-15T13:47:33+08:00
draft: false
toc: false
categories: ["python"]
tags: []
---

## 基于jango channel 实现推送

在官方demo 的基础上自定义推送 Consumer 。

由于只是服务端到客户端单方向推送信息。类中只实现如下3个方法即可

```

"""
connect 建立连接
disconnet 断开连接
push_messages 推送消息
"""

from channels.generic.websocket import  AsyncWebsocketConsumer

# # 推送consumer
class PushConsumer(AsyncWebsocketConsumer):
    async def connect(self):
      # 将username名称设定为 group 名
      # self.group_name = self.scope['url_route']['kwargs']['username']
      self.room_name = self.scope['url_route']['kwargs']['room_name']
      self.room_group_name = 'chat_%s' % self.room_name
      await self.channel_layer.group_add(
        self.room_group_name,
        self.channel_name
      )
      await self.accept()

    async def disconnect(self, close_code):
      await self.channel_layer.group_discard(
        self.room_group_name,
        self.channel_name
      )
    # print(PushConsumer.chats)
    async def push_message(self, event):
      message = event['message']
      # 可记录消息是否推送成功
      print("push messsage  %s to %s " % (message, self.room_group_name))
      await self.send(text_data=json.dumps({
        "message": message
      }))
```

## 推送方法实现

```
### python manage.py shell
from channels.layers import get_channel_layer
from asgiref.sync import async_to_sync

def push(username, message):
    channel_layer = get_channel_layer()
    async_to_sync(channel_layer.group_send)(
        username,
        {
            "type": "push.message",
            "message": message
        }
    )
```
    
## 调用推送

```
push("username","your message")
```

## 点对点通信的一些思路

聊天组的命名形式为user_a的id加上下划线_加上user_b的id，其中id值从小到大放置，例如： 123_125。 通信双方可以确定唯一groupname。

在全局。比如redis中记录groupname中双方是否同时在线。如果同时在线，则发送方直接将信息发送的group中。如果对方不在先。则向对方发送一条推送消息唤醒对方连接到group中。


```
async def receive_json(self, message, **kwargs):
   # 收到信息时调用
   to_user = message.get('to_user')
   # 信息发送
   length = redis.connection.get[self.group_name]
   # 判断对方是否在group中存在连接
   if length == 2:
     await self.channel_layer.group_send(
       self.group_name,
       {
         "type": "chat.message",
         "message": message.get('message'),
       },
     )
   else:
     await self.channel_layer.group_send(
       to_user,
       {
         "type": "push.message",
         "event": {'message': message.get('message'), 'group': self.group_name}
       },
     )
 async def chat_message(self, event):
   # Handles the "chat.message" event when it's sent to us.
   await self.send_json({
     "message": event["message"],
   })
```


