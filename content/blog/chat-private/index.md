---
title: Chat App with NodeJs
date: "2020-08-18T22:12:03.284Z"
description: "Exercise  NodeJs course"
---


#I. Cấu hình chung

**1. Configs folder**

File configs > database.js

Tạo 1 collection mới để tương tác với database MongoDb

```javascript
col_chatPrivate: "chatPrivate",
```

**2. Link to Private message**
File views > chat > elements > navbar-collapse.ejs

```javascript
const linkPrivateMessage = systemConfig.prefixChat + `/private`;

...
<li class="divider"></li>
                <li><a href="<%= linkPrivateMessage %>">Private Message</a></li>
...
```

**3. Schema cho Private message**
File schemas > chatPrivate.js

```javascript
var schema = new mongoose.Schema({
  room: String,
  content: String,
  username: String,
  avatar: String,
  created: Date,
});
```

**4. Views cho Private message**
File views > chat > pages > private > index.ejs

```javascript
<% include ./../../helpers/show-box-all-users %>

<div class="col-md-4">   
    <div class="row">
        <div class="box box-primary">
            <div class="box-header with-border">
                <h3 class="box-title">All Users In System</h3>        
                <div class="box-tools pull-right">
                    <button type="button" class="btn btn-box-tool" data-widget="collapse"><i class="fa fa-minus"></i>
                    </button>
                </div>
            </div>
            <div class="box-body" id="list-users-in-system">
            <%- showBoxAllUsers(allUsers) %> 
            </div>
        </div>
               
            </div>
    
    
</div>
<div class="col-md-8 no-padding-right ">
    <%  if(privateId!==null)  {%>
    <%- include ./box-direct-chat %>   
    <% } %>      
</div>

<input type="hidden" name="prefixSocket" value="<%= prefixSocket %>">
<input type="hidden" name="privateId" value="<%= privateId %>">
<input type="hidden" name="username" value="<%= userInfo.username %>">
<input type="hidden" name="avatar" value="<%= userInfo.avatar %>">
<input type="hidden" name="socketId" value="<%= userInfo.id %>">


<script src="chat/js/socket-private.js"></script> 
```

**5. Routes cho Private message**

File routes > chat > private.js

Trước tiên tạo 2 routes:

```javascript
// cái này là khi mình vào /private, se thấy list các users co thể gửi tin nhắn tới
  router.get("/", async (req, res, next) => {
    let privateId = null;
    let usersInSystem = await UsersModel.listItemsChat({});
    let allUsers = usersInSystem.filter((user) => user.username !== req.user.username);

    res.render(`${folderView}index`, {
      layout: layoutChat,
      privateId,
      allUsers,
      prefixSocket,
    });
  });

// cái này là khi mình vào bấm vào nút send message cho 1 user nào đó, se thấy khung chat với user đó hiện ra
// privateId là tên của user được nhận tin nhắn, dùng để lưu tin nhắn sau này ở models
  router.get("/:private", async (req, res, next) => {
    let privateId = ParamsHelpers.getParam(req.params, "private", "");
    let usersInSystem = await UsersModel.listItemsChat({});
    let allUsers = usersInSystem.filter((user) => user.username !== req.user.username);

    const chatsRoom = await ChatPrivateModel.listItems(
      { room: privateId, currentUser: req.user.name },
      { task: "list-items-by-privacy" }
    );

    res.render(`${folderView}index`, {
      layout: layoutChat,
      privateId,
      allUsers,
      chatsRoom,
      prefixSocket,
    });
  });


```

**6. Models cho Private message**


File models > chatPrivate.js

```javascript
const ChatsPrivateModel = require(__path_schemas + "chatPrivate");
const databaseConfig = require(__path_configs + "database");

module.exports = {
  listItems: (params, options = null) => {
    if (options.task == "list-items-by-privacy") {
      let sort = { created: "asc" };

      return ChatsPrivateModel.find({
        $or: [
          {
            room: params.room,
            username: params.currentUser,
          },
          {
            room: params.currentUser,
            username: params.room,
          },
        ],
      })
        .select("content created username avatar room")
        .sort(sort);
    }
  },
  countItem: (params, options = null) => {
    let objWhere = {};
    return ChatsPrivateModel.countDocuments(objWhere);
  },
  saveItem: (item, options = null) => {
    if (options.task == "add") {
      item.created = Date.now();
      return new ChatsPrivateModel(item).save();
    }
  },
};


```

Model này dùng để hiện những tin nhắn đã được lưu trước đó bên trong ChatsPrivate collection mà mình có trước đó trong database
Đoạn code trên là từ việc e dùng cách sau để lưu tin nhắn vào database

+ User A gửi tin nhắn cho user B private, mình lưu theo cách:
-> room = user B
-> user = user A
+ hoặc ngược lại khi B gửi cho A, mình lưu theo cách:
-> room = user B
-> user = user A

Thì 1 trong 2 tình huống trên xảy ra, nghĩa là tin nhắn có cùng 1 room (room này là room của riêng A và B thôi) 
=> Trong database mình sẽ lấy hết những tin nhắn thỏa điều kiện ở trên.

**7. Routes cho Private message (tiếp theo)**

```javascript
let prefixSocket = "PRIVATE_";
  //let users = new UserConnect();

  io.on("connection", function (socket) {
    // let srvSockets = io.sockets.sockets;
    // console.log(Object.values(srvSockets).length);

    socket.on(`${prefixSocket}USER_CONNECT`, async (data) => {
      console.log("on private connect test ");
      users.addUser(socket.id, data.username, data.avatar);

      //return list of users currently in current page ~ home
      io.emit(`${prefixSocket}SERVER_SEND_CURRENT_ONLINE_USERS`, users.getListUsers());
    });

    socket.on(`disconnect`, async () => {
      let disConnectUser = users.removeUser(socket.id);
      //update current list of user connect back to client, if there is a disconnected user
      if (disConnectUser) io.emit(`${prefixSocket}SERVER_SEND_LIST_ALL_USERS`, users.getListUsers());
    });

    socket.on(`${prefixSocket}CLIENT_SEND_PRIVATE_MESSAGE`, async (data) => {
      if (data.content.length > 0) {
        const result = await ChatPrivateModel.saveItem(data, { task: "add" });

        //io emit send to the user itself
        io.to(socket.id).emit(`${prefixSocket}SERVER_RETURN_PRIVATE_MESSAGE`, {
          room: result.room,
          content: result.content,
          username: result.username,
          avatar: result.avatar,
          created: moment(result.created).format(systemConfig.format_time_chat),
        });

        //io emit send to socketId, if the receiver is currently online
        if (data.receiver && data.receiver.length > 0) {         

          io.to(data.receiver[0].id).emit(`${prefixSocket}SERVER_RETURN_PRIVATE_MESSAGE`, {
            room: result.room,
            content: result.content,
            username: result.username,
            avatar: result.avatar,
            created: moment(result.created).format(systemConfig.format_time_chat),
          });         

          //show notify
          io.to(data.receiver[0].id).emit(`${prefixSocket}SERVER_SEND_PRIVATE_MESSAGE_NOTIFY`, {
            sender: result.username,
            senderAvatar: result.avatar,
            content: result.content,
            room: result.room,
            created: moment(result.created).format(systemConfig.format_time_chat),
          });
        }
      } 
      .... //phần còn lại giống home hoặc room
    });
    
  });

```
Những code trên em làm là:

+ Em cũng lưu những user online vào object, object này lấy từ user online của default route "/" (cái này e bị lỗi nên có lần em hỏi anh cách sửa để co user online ở tất cả các route khi vào /home, /private, /room)

+ socket.on(`${prefixSocket}CLIENT_SEND_PRIVATE_MESSAGE`...
=> cái này xử lí khi phía client gửi tin nhắn, user bấm nút "Send" trong khung chat

-> Lưu tin nhắn vào database: const result = await ChatPrivateModel.saveItem(data, { task: "add" });

-> io.to(socket.id).emit(`${prefixSocket}SERVER_RETURN_PRIVATE_MESSAGE`... Gửi tin nhắn này về cho chính user đó, 
Vi du user A gửi tin nhắn cho B, thì cái code trên sẽ gửi chính tin nhắn cho A, để ở khung chat của A hiện lên tin nhắn A đã gửi

-> if (data.receiver && data.receiver.length > 0) {
       io.to(data.receiver[0].id).emit(`${prefixSocket}SERVER_RETURN_PRIVATE_MESSAGE`, {...
       io.to(data.receiver[0].id).emit(`${prefixSocket}SERVER_SEND_PRIVATE_MESSAGE_NOTIFY`, {...
    
Đoạn code này kiểm tra nếu:
Khi A gửi cho B và B đang online
Thì B sẽ:
+ nhận tin nhắn trong khung chat
+ Hiện notify trên menu bar là đã có 1 tin nhắn gửi tới từ A

**8. Xử lí phía client phần socketIO**

File public > backend > Chat -> JS > socket-private.js

1. Khi user bấm nút Send:
Thì mình gửi lên server nội dung tin nhắn, tên người gửi, người nhận
Và post 1 cái API -> save unread message để dùng cho notification
```javascript
elmFormChat.submit(function () {
    let receiver = null;
    console.log(onlineUsers);
    if (onlineUsers)
      receiver = Object.values(onlineUsers).filter((user) => user.username === elmInputRoom.val());
    // console.log(receiver);
    socket.emit(
      `${prefixSocket}CLIENT_SEND_PRIVATE_MESSAGE`,
      paramsUserSendPrivateMessage(elmInputMessage, elmInputUsr, elmInputAvatar, elmInputRoom, receiver)
    );

    $.ajax({
      method: "POST",
      url: "/api/save-unread-message",
      dataType: "json",
      data: {
        sender: elmInputUsr.val(),
        senderAvatar: elmInputAvatar.val(),
        content: elmInputMessage.val(),
        receiver: elmInputRoom.val(),
        created: Date.now(),
      },
    }).done((data) => {});

    elmInputMessage.val("");
    emojioneAreas.data("emojioneArea").setText("");
    $("div#area-notification").remove();
    return false;
  });
```

Lưu unread message
2.

File routes > chat > api.js
```
router.post("/save-unread-message", async (req, res, next) => {
  let item = {};
  item.sender = req.body.sender;
  item.senderAvatar = req.body.senderAvatar;
  item.content = req.body.content;
  item.receiver = req.body.receiver;
  item.created = req.body.created;

  await UsersModel.saveItem(item, { task: "unread-message" });

  res.json(item);
});
```

File models > users.js
```
 if (options.task == "unread-message") {
      await UsersModel.updateOne(
        {
          username: item.receiver,
        },
        {
          $pull: {
            unreadMessage: {
              username: item.sender,
            },
          },
        }
      );

      return await UsersModel.updateOne(
        {
          username: item.receiver,
        },
        {
          $push: {
            unreadMessage: {
              username: item.sender,
              avatar: item.senderAvatar,
              content: item.content,
              created: item.created,
            },
          },
        }
      );
```
Cái này lưu unread message khi có user A gửi tin nhắn cho B và B đang online / offline
Nếu B đã có 1 unread message trước đó thì sẽ lấy tin nhắn cũ ra, và lưu tin nhắn mới vào array


**9. Todo**

Hiện tại em còn thiếu:
+ Xử lí user online:
Ví dụ user A đang online và ở /, /home, /room. User B gửi tin nhắn private cho user A, vì hiện tại ko lấy được user A online cho nên là sẽ ko có notify được cho user A ngay tại lúc đó

+ Xử lí unread message:
Khi user B bấm vào message notify thì phải chuyển trạng thái tin nhắn này vào "đã đọc", hiện tại em còn thiếu trạng thái tin nhắn là đã đọc / chưa đọc
