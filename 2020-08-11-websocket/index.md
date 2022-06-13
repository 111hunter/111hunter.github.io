# 基于 Socket.IO 的实时聊天室


WebSocket 是一种服务端和客户端之间的双向通信协议，和 HTTP 一样是基于 TCP 协议的应用层协议，并且 WebSocket 在握手阶段依赖于 HTTP 连接。

WebSocket 广泛应用于多用户实时交流，服务端数据持续变化的场景。比如社交聊天、弹幕、多玩家游戏、协同编辑、股票基金实时报价、体育实况更新、视频会议/聊天、基于位置的应用、在线教育、智能家居等需要高实时的场景。

学习 WebSocket，请看这篇[教程](http://www.ruanyifeng.com/blog/2017/05/websocket.html)。在 node.js 中，通常使用 socket.io 这个库。socket.io 封装了 WebSocket 服务端 JS 库，同时也提供客户端的 JS 库。Socket.IO 支持以事件为基础的实时双向通讯。它可以兼容各种浏览器或移动设备，从而让开发者可以聚焦到功能的实现而不是平台的兼容性。

## Socket.IO 常用 api

常用服务端 api：

```js

socket.on('eventName', msg => {}) 
/*服务器端监听客户端emit的事件，事件名称可以和客户端是重复的，但是并没有任何关联。
socket.io内置了一些事件比如connection，disconnect，exit事件*/

socket.emit('eventName', msg) 
//服务端各自的socket向各自的客户端发送数据

socket.broadcast('eventName', msg) 
//服务端向其他客户端发送消息，不包括自己的客户端

socket.join(channel) 
//创建一个频道（非常有用，尤其做分频道的时候，比如斗地主这种实时棋牌游戏）

io.sockets.in(channel) 
//加入一个频道

io.to(channel).emit('eventName', msg)
//向一个频道发送消息，包括自己的客户端

socket.broadcast.to(channel).emit('eventName', msg) 
//向一个频道发送消息，不包括自己的客户端

io.emit('eventName', msg)
//向所有客户端发送数据

io.sockets.adapter.rooms 
//获取所有的频道
```

常用客户端 api：

```js
//客户端
 
io.connect(url) 
//客户端连接上服务器端，可简写为 io(url)，无跨域时为 io()

socket.on('eventName', msg => {}) 
//客户端监听服务器端事件

socket.emit('eventName', msg) 
//客户端向服务器端发送数据

socket.disconnect() 
//客户端断开链接
```

更多的 api 请参阅 Socket.IO 的[官方文档](https://socket.io/)。这里有一篇搭建实时聊天室的[文章](https://www.cnblogs.com/AlexZha/p/6101035.html)，注意文中的 index.html 和 client.js 中的线上服务器地址 realtime.plhwin.com:3000 已经没有了，改为本地地址 localhost:3000 就能运行代码了。index.html 里的 
```html
<script src="/socket.io/socket.io.js"></script>
```
指向的文件是其实是
```html
<script src="../server/node_modules/socket.io-client/dist/socket.io.js"></script>
```

整体的开发思路就是服务端和客户端其中一端触发事件，另一端就监听事件。文中的示例程序只用到了事件触发 socket.emit 和事件监听 socket.on。下文的示例程序展示了 Socket.IO 中更多 api 的用法。用户进入聊天室时需要选择房间，进入相同房间的用户才能内部交流，不同房间之间的内部信息不能互通。

![ChatCord实时聊天室](/img/socketio.png "ChatCord实时聊天室")

## 服务端实现

WebSocket 依赖于 http，这里需要安装 socket.io 和 express

```js
// server.js

const path = require('path');
const http = require('http');
const express = require('express');
const socketio = require('socket.io');
const formatMessage = require('./utils/messages');
const {
  userJoin,
  getCurrentUser,
  userLeave,
  getRoomUsers
} = require('./utils/users');

const app = express();
const server = http.createServer(app);
const io = socketio(server);

// Set static folder
app.use(express.static(path.join(__dirname, 'public')));

const botName = 'ChatCord Bot';

// Run when client connects
io.on('connection', socket => {
  socket.on('joinRoom', ({ username, room }) => {
    const user = userJoin(socket.id, username, room);

    socket.join(user.room);

    // Welcome current user
    socket.emit('message', formatMessage(botName, 'Welcome to ChatCord!'));

    // Broadcast when a user connects
    socket.broadcast
      .to(user.room)
      .emit(
        'message',
        formatMessage(botName, `${user.username} has joined the chat`)
      );

    // Send users and room info
    io.to(user.room).emit('roomUsers', {
      room: user.room,
      users: getRoomUsers(user.room)
    });
  });

  // Listen for chatMessage
  socket.on('chatMessage', msg => {
    const user = getCurrentUser(socket.id);

    io.to(user.room).emit('message', formatMessage(user.username, msg));
  });

  // Runs when client disconnects
  socket.on('disconnect', () => {
    const user = userLeave(socket.id);

    if (user) {
      io.to(user.room).emit(
        'message',
        formatMessage(botName, `${user.username} has left the chat`)
      );

      // Send users and room info
      io.to(user.room).emit('roomUsers', {
        room: user.room,
        users: getRoomUsers(user.room)
      });
    }
  });
});

const PORT = process.env.PORT || 3000;

server.listen(PORT, () => console.log(`Server running on port ${PORT}`));
```


## 客户端实现

需要先在 html 中引入 socket.io-client，才能使用 io

```html
<!-- public/chat.html -->

<script src="/socket.io/socket.io.js"></script>
<script src="js/main.js"></script>
```

这里的静态资源文件是由 express 加载的，没有跨域，可省略 io 括号里的地址

```js
// public/js/main.js

const chatForm = document.getElementById('chat-form');
const chatMessages = document.querySelector('.chat-messages');
const roomName = document.getElementById('room-name');
const userList = document.getElementById('users');

// Get username and room from URL
const { username, room } = Qs.parse(location.search, {
  ignoreQueryPrefix: true
});

const socket = io();

// Join chatroom
socket.emit('joinRoom', { username, room });

// Get room and users
socket.on('roomUsers', ({ room, users }) => {
  outputRoomName(room);
  outputUsers(users);
});

// Message from server
socket.on('message', message => {
  console.log(message);
  outputMessage(message);

  // Scroll down
  chatMessages.scrollTop = chatMessages.scrollHeight;
});

// Message submit
chatForm.addEventListener('submit', e => {
  e.preventDefault();

  // Get message text
  const msg = e.target.elements.msg.value;

  // Emit message to server
  socket.emit('chatMessage', msg);

  // Clear input
  e.target.elements.msg.value = '';
  e.target.elements.msg.focus();
});

// Output message to DOM
function outputMessage(message) {
  const div = document.createElement('div');
  div.classList.add('message');
  div.innerHTML = `<p class="meta">${message.username} <span>${message.time}</span></p>
  <p class="text">
    ${message.text}
  </p>`;
  document.querySelector('.chat-messages').appendChild(div);
}

// Add room name to DOM
function outputRoomName(room) {
  roomName.innerText = room;
}

// Add users to DOM
function outputUsers(users) {
  userList.innerHTML = `
    ${users.map(user => `<li>${user.username}</li>`).join('')}
  `;
}
```
更多内容请看源码。

附：[源码地址](https://github.com/bradtraversy/chatcord)

**参考资料**

- [WebSocket 教程](http://www.ruanyifeng.com/blog/2017/05/websocket.html)
- [Socket.IO 官方文档](https://socket.io/)
- [ChatCord 源码](https://github.com/bradtraversy/chatcord)
