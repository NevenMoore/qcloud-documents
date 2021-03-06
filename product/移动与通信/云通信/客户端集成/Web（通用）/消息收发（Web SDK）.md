## 监听新消息


>注意：现调用过设置用户资料的接口设置过用户资料的，群消息里面是会下发设置的资料的。

示例：

```
//监听新消息事件
//newMsgList 为新消息数组，结构为[Msg]
function onMsgNotify(newMsgList) {
    //console.warn(newMsgList);
    var sess, newMsg;
    //获取所有聊天会话
    var sessMap = webim.MsgStore.sessMap();

    for (var j in newMsgList) {//遍历新消息
        newMsg = newMsgList[j];

        if (newMsg.getSession().id() == selToID) {//为当前聊天对象的消息
            selSess = newMsg.getSession();
            //在聊天窗体中新增一条消息
            //console.warn(newMsg);
            addMsg(newMsg);
        }
    }
    //消息已读上报，以及设置会话自动已读标记
    webim.setAutoRead(selSess, true, true);

    for (var i in sessMap) {
        sess = sessMap[i];
        if (selToID != sess.id()) {//更新其他聊天对象的未读消息数
            updateSessDiv(sess.type(), sess.id(), sess.unread());
        }
    }
}
```
其中参数newMsgList为 webim.Msg数组，即[ webim.Msg]

## 显示一条消息

示例：

```
//聊天页面增加一条消息
function addMsg(msg) {
    var isSelfSend, fromAccount, fromAccountNick, sessType, subType;

    fromAccount = msg.getFromAccount();
    if (!fromAccount) {
        fromAccount = '';
    }
    fromAccountNick = msg.getFromAccountNick();
    if (!fromAccountNick) {
        fromAccountNick = fromAccount;
    }
    isSelfSend = msg.getIsSend();//消息是否为自己发的

    var onemsg = document.createElement("div");
    onemsg.className = "onemsg";
    var msghead = document.createElement("p");
    var msgbody = document.createElement("p");
    var msgPre = document.createElement("pre");
    msghead.className = "msghead";
    msgbody.className = "msgbody";
    //如果是发给自己的消息
    if (!isSelfSend)
        msghead.style.color = "blue";
    //昵称  消息时间
    msghead.innerHTML = fromAccountNick + "&nbsp;&nbsp;" + webim.Tool.formatTimeStamp(msg.getTime());


    //解析消息
    //获取会话类型，目前只支持群聊
    //webim.SESSION_TYPE.GROUP-群聊，
    //webim.SESSION_TYPE.C2C-私聊，
    sessType = msg.getSession().type();
    //获取消息子类型
    //会话类型为群聊时，子类型为：webim.GROUP_MSG_SUB_TYPE
    //会话类型为私聊时，子类型为：webim.C2C_MSG_SUB_TYPE
    subType = msg.getSubType();



    switch (subType) {

        case webim.GROUP_MSG_SUB_TYPE.COMMON://群普通消息
            msgPre.innerHTML = convertMsgtoHtml(msg);
            break;
        case webim.GROUP_MSG_SUB_TYPE.REDPACKET://群红包消息
            msgPre.innerHTML = "[群红包消息]" + convertMsgtoHtml(msg);
            break;
        case webim.GROUP_MSG_SUB_TYPE.LOVEMSG://群点赞消息
            //业务自己可以增加逻辑，比如展示点赞动画效果
            msgPre.innerHTML = "[群点赞消息]" + convertMsgtoHtml(msg);
            //展示点赞动画
            //showLoveMsgAnimation();
            break;
        case webim.GROUP_MSG_SUB_TYPE.TIP://群提示消息
            msgPre.innerHTML = "[群提示消息]" + convertMsgtoHtml(msg);
            break;
    }

    msgbody.appendChild(msgPre);

    onemsg.appendChild(msghead);
    onemsg.appendChild(msgbody);
    //消息列表
    var msgflow = document.getElementsByClassName("msgflow")[0];
    msgflow.appendChild(onemsg);
    //300ms后,等待图片加载完，滚动条自动滚动到底部
    setTimeout(function () {
        msgflow.scrollTop = msgflow.scrollHeight;
    }, 300);

}
```


## 解析一条消息

示例：

```
//把消息转换成Html
function convertMsgtoHtml(msg) {
    var html = "", elems, elem, type, content;
    elems = msg.getElems();//获取消息包含的元素数组
    for (var i in elems) {
        elem = elems[i];
        type = elem.getType();//获取元素类型
        content = elem.getContent();//获取元素对象
        switch (type) {
            case webim.MSG_ELEMENT_TYPE.TEXT:
                html += convertTextMsgToHtml(content);
                break;
            case webim.MSG_ELEMENT_TYPE.FACE:
                html += convertFaceMsgToHtml(content);
                break;
            case webim.MSG_ELEMENT_TYPE.IMAGE:
                html += convertImageMsgToHtml(content);
                break;
            case webim.MSG_ELEMENT_TYPE.SOUND:
                html += convertSoundMsgToHtml(content);
                break;
            case webim.MSG_ELEMENT_TYPE.FILE:
                html += convertFileMsgToHtml(content);
                break;
            case webim.MSG_ELEMENT_TYPE.LOCATION://暂不支持地理位置
                //html += convertLocationMsgToHtml(content);
                break;
            case webim.MSG_ELEMENT_TYPE.CUSTOM:
                html += convertCustomMsgToHtml(content);
                break;
            case webim.MSG_ELEMENT_TYPE.GROUP_TIP:
                html += convertGroupTipMsgToHtml(content);
                break;
            default:
                webim.Log.error('未知消息元素类型: elemType=' + type);
                break;
        }
    }
    return html;
}
```

## 解析文本消息元素

示例：

```
//解析文本消息元素
function convertTextMsgToHtml(content) {
    return content.getText();
}
```


## 解析表情消息元素

示例：

```
//解析表情消息元素
function convertFaceMsgToHtml(content) {
    var index = content.getIndex();
    var data = content.getData();
    var url=null;
    var emotion=webim.Emotions[index];
    if(emotion && emotion[1]){
        url=emotion[1];
    }
    if (url) {
        return	"<img src='" + url + "'/>";
    } else {
        return data;
    }
}
```

## 解析图片消息元素

示例：

```
//解析图片消息元素
function convertImageMsgToHtml(content) {
    var smallImage = content.getImage(webim.IMAGE_TYPE.SMALL);//小图
    var bigImage = content.getImage(webim.IMAGE_TYPE.LARGE);//大图
    var oriImage = content.getImage(webim.IMAGE_TYPE.ORIGIN);//原图
    if (!bigImage) {
        bigImage = smallImage;
    }
    if (!oriImage) {
        oriImage = smallImage;
    }
    return	"<img src='" + smallImage.getUrl() + "#" + bigImage.getUrl() + "#" + oriImage.getUrl() + "' style='CURSOR: hand' id='" + content.getImageId() + "' bigImgUrl='" + bigImage.getUrl() + "' onclick='imageClick(this)' />";
}
```

## 解析语音消息元素

示例：

```
//解析语音消息元素
function convertSoundMsgToHtml(content) {
    var second=content.getSecond();//获取语音时长
    var downUrl=content.getDownUrl();
    if (webim.BROWSER_INFO.type == 'ie' && parseInt(webim.BROWSER_INFO.ver) <= 8) {
        return '[这是一条语音消息]demo暂不支持ie8(含)以下浏览器播放语音,语音URL:' + downUrl;
    }
    return '<audio src="' + downUrl + '" controls="controls" onplay="onChangePlayAudio(this)" preload="none"></audio>';
}
```

## 解析文件消息元素

示例：

```
//解析文件消息元素
function convertFileMsgToHtml(content) {
    var fileSize = Math.round(content.getSize() / 1024);
    return '<a href="' + content.getDownUrl() + '" title="点击下载文件" ><i class="glyphicon glyphicon-file">&nbsp;' + content.getName() + '(' + fileSize + 'KB)</i></a>';
}
```

## 解析位置消息元素

示例：

```
//解析位置消息元素
function convertLocationMsgToHtml(content) {
    return '经度='+content.getLongitude()+',纬度='+content.getLatitude()+',描述='+content.getDesc();
}
```

## 解析自定义消息元素

示例：

```
//解析自定义消息元素
function convertCustomMsgToHtml(content) {
    var data = content.getData();
    var desc = content.getDesc();
    var ext = content.getExt();
    return "data=" + data + ", desc=" + desc + ", ext=" + ext;
}
```

## 解析群提示消息元素

当有用户被邀请加入群组，或者有用户被移出群组时，群内会产生有提示消息，调用方可以根据需要展示给群组用户，或者忽略。

示例：

```
//解析群提示消息元素
function convertGroupTipMsgToHtml(content) {
    var WEB_IM_GROUP_TIP_MAX_USER_COUNT=10;
    var text ="";
    var maxIndex = WEB_IM_GROUP_TIP_MAX_USER_COUNT - 1;
    var opType,opUserId,userIdList;
    opType=content.getOpType();//群提示消息类型（操作类型）
    opUserId=content.getOpUserId();//操作人id
    switch (opType) {
        case webim.GROUP_TIP_TYPE.JOIN://加入群
            userIdList=content.getUserIdList();
            //text += opUserId + "邀请了";
            for (var m in userIdList) {
                text += userIdList[m] + ",";
                if (userIdList.length > WEB_IM_GROUP_TIP_MAX_USER_COUNT && m == maxIndex) {
                    text += "等" + userIdList.length + "人";
                    break;
                }
            }
            text += "加入该群";
            break;
        case webim.GROUP_TIP_TYPE.QUIT://退出群
            text += opUserId + "主动退出该群";
            break;
        case webim.GROUP_TIP_TYPE.KICK://踢出群
            text += opUserId + "将";
            userIdList=content.getUserIdList();
            for (var m in userIdList) {
                text += userIdList[m] + ",";
                if (userIdList.length > WEB_IM_GROUP_TIP_MAX_USER_COUNT && m == maxIndex) {
                    text += "等" + userIdList.length + "人";
                    break;
                }
            }
            text += "踢出该群";
            break;
        case webim.GROUP_TIP_TYPE.SET_ADMIN://设置管理员
            text += opUserId + "将";
            userIdList=content.getUserIdList();
            for (var m in userIdList) {
                text += userIdList[m] + ",";
                if (userIdList.length > WEB_IM_GROUP_TIP_MAX_USER_COUNT && m == maxIndex) {
                    text += "等" + userIdList.length + "人";
                    break;
                }
            }
            text += "设为管理员";
            break;
        case webim.GROUP_TIP_TYPE.CANCEL_ADMIN://取消管理员
            text += opUserId + "取消";
            userIdList=content.getUserIdList();
            for (var m in userIdList) {
                text += userIdList[m] + ",";
                if (userIdList.length > WEB_IM_GROUP_TIP_MAX_USER_COUNT && m == maxIndex) {
                    text += "等" + userIdList.length + "人";
                    break;
                }
            }
            text += "的管理员资格";
            break;

        case webim.GROUP_TIP_TYPE.MODIFY_GROUP_INFO://群资料变更
            text += opUserId + "修改了群资料：";
            var groupInfoList=content.getGroupInfoList();
            var type,value;
            for (var m in groupInfoList) {
                type = groupInfoList[m].getType();
                value = groupInfoList[m].getValue();
                switch (type) {
                    case webim.GROUP_TIP_MODIFY_GROUP_INFO_TYPE.FACE_URL:
                        text += "群头像为" + value + "; ";
                        break;
                    case webim.GROUP_TIP_MODIFY_GROUP_INFO_TYPE.NAME:
                        text += "群名称为" + value + "; ";
                        break;
                    case webim.GROUP_TIP_MODIFY_GROUP_INFO_TYPE.OWNER:
                        text += "群主为" + value + "; ";
                        break;
                    case webim.GROUP_TIP_MODIFY_GROUP_INFO_TYPE.NOTIFICATION:
                        text += "群公告为" + value + "; ";
                        break;
                    case webim.GROUP_TIP_MODIFY_GROUP_INFO_TYPE.INTRODUCTION:
                        text += "群简介为" + value + "; ";
                        break;
                    default:
                        text += "未知信息为:type=" + type + ",value=" + value + "; ";
                        break;
                }
            }
            break;

        case webim.GROUP_TIP_TYPE.MODIFY_MEMBER_INFO://群成员资料变更(禁言时间)
            text += opUserId + "修改了群成员资料:";
            var memberInfoList=content.getMemberInfoList();
            var userId,shutupTime;
            for (var m in memberInfoList) {
                userId = memberInfoList[m].getUserId();
                shutupTime = memberInfoList[m].getShutupTime();
                text += userId + ": ";
                if (shutupTime != null && shutupTime !== undefined) {
                    if (shutupTime == 0) {
                        text += "取消禁言; ";
                    } else {
                        text += "禁言" + shutupTime + "秒; ";
                    }
                } else {
                    text += " shutupTime为空";
                }
                if (memberInfoList.length > WEB_IM_GROUP_TIP_MAX_USER_COUNT && m == maxIndex) {
                    text += "等" + memberInfoList.length + "人";
                    break;
                }
            }
            break;
        default:
            text += "未知群提示消息类型：type=" + opType;
            break;
    }
    return text;
}
```


## 发送消息（文本+表情）

本节主要介绍sdk 发消息sendMsg api。

函数名：

```
webim.sendMsg
```

定义：

```
webim.sendMsg(msg,cbOk, cbErr)
```

参数列表：

| 名称    | 说明         | 类型        |
| ----- | ---------- | --------- |
| msg   | 消息对象       | webim.Msg |
| cbOk  | 调用接口成功回调函数 | Function  |
| cbErr | 调用接口失败回调函数 | Function  |



示例：

```
//发送消息(文本或者表情)
function onSendMsg() {

    if (!selToID) {
        alert("你还没有选中好友或者群组，暂不能聊天");
        $("#send_msg_text").val('');
        return;
    }
    //获取消息内容
    var msgtosend = document.getElementsByClassName("msgedit")[0].value;
    var msgLen = webim.Tool.getStrBytes(msgtosend);

    if (msgtosend.length < 1) {
        alert("发送的消息不能为空!");
        $("#send_msg_text").val('');
        return;
    }
    var maxLen, errInfo;
    if (selType == webim.SESSION_TYPE.C2C) {
        maxLen = webim.MSG_MAX_LENGTH.C2C;
        errInfo = "消息长度超出限制(最多" + Math.round(maxLen / 3) + "汉字)";
    } else {
        maxLen = webim.MSG_MAX_LENGTH.GROUP;
        errInfo = "消息长度超出限制(最多" + Math.round(maxLen / 3) + "汉字)";
    }
    if (msgLen > maxLen) {
        alert(errInfo);
        return;
    }
    if (!selSess) {
      var  selSess = new webim.Session(selType, selToID, selToID, friendHeadUrl, Math.round(new Date().getTime() / 1000));
    }
    var isSend = true;//是否为自己发送
    var seq = -1;//消息序列，-1表示sdk自动生成，用于去重
    var random = Math.round(Math.random() * 4294967296);//消息随机数，用于去重
    var msgTime = Math.round(new Date().getTime() / 1000);//消息时间戳
    var subType;//消息子类型
    if (selType == webim.SESSION_TYPE.C2C) {
        subType = webim.C2C_MSG_SUB_TYPE.COMMON;
    } else {
        //webim.GROUP_MSG_SUB_TYPE.COMMON-普通消息,
        //webim.GROUP_MSG_SUB_TYPE.LOVEMSG-点赞消息，优先级最低
        //webim.GROUP_MSG_SUB_TYPE.TIP-提示消息(不支持发送，用于区分群消息子类型)，
        //webim.GROUP_MSG_SUB_TYPE.REDPACKET-红包消息，优先级最高
        subType = webim.GROUP_MSG_SUB_TYPE.COMMON;
    }
    var msg = new webim.Msg(selSess, isSend, seq, random, msgTime, loginInfo.identifier, subType, loginInfo.identifierNick);
    
    var text_obj, face_obj, tmsg, emotionIndex, emotion, restMsgIndex;
    //解析文本和表情
    var expr = /\[[^[\]]{1,3}\]/mg;
    var emotions = msgtosend.match(expr);
    if (!emotions || emotions.length < 1) {
        text_obj = new webim.Msg.Elem.Text(msgtosend);
        msg.addText(text_obj);
    } else {

        for (var i = 0; i < emotions.length; i++) {
            tmsg = msgtosend.substring(0, msgtosend.indexOf(emotions[i]));
            if (tmsg) {
                text_obj = new webim.Msg.Elem.Text(tmsg);
                msg.addText(text_obj);
            }
            emotionIndex = webim.EmotionDataIndexs[emotions[i]];
            emotion = webim.Emotions[emotionIndex];
            
            if (emotion) {
                face_obj = new webim.Msg.Elem.Face(emotionIndex, emotions[i]);
                msg.addFace(face_obj);
            } else {
                text_obj = new webim.Msg.Elem.Text(emotions[i]);
                msg.addText(text_obj);
            }
            restMsgIndex = msgtosend.indexOf(emotions[i]) + emotions[i].length;
            msgtosend = msgtosend.substring(restMsgIndex);
        }
        if (msgtosend) {
            text_obj = new webim.Msg.Elem.Text(msgtosend);
            msg.addText(text_obj);
        }
    }

    webim.sendMsg(msg, function (resp) {
        if (selType == webim.SESSION_TYPE.C2C) {//私聊时，在聊天窗口手动添加一条发的消息，群聊时，长轮询接口会返回自己发的消息
            addMsg(msg);
        }
        webim.Tool.setCookie("tmpmsg_" + selToID, '', 0);
        $("#send_msg_text").val('');
        turnoffFaces_box();
    }, function (err) {
        alert(err.ErrorInfo);
        $("#send_msg_text").val('');
    });
}
```


## 上传图片 （高版本浏览器）

目前demo采用了H5 FileAPI读取图片，并将图片二进制数据转换成base64编码进行分片上传，理论上没有大小限制。

```
/* function uploadPic  
 *   上传图片
 * params:
 *   cbOk	- function()类型, 成功时回调函数
 *   cbErr	- function(err)类型, 失败时回调函数, err为错误对象
 * return:
 *   (无)
 */
uploadPic: function(options, cbOk, cbErr) {},
```

**示例：**

```
//上传图片
function uploadPic() {
    var uploadFiles = document.getElementById('upd_pic');
    var file = uploadFiles.files[0];
    var businessType;//业务类型，1-发群图片，2-向好友发图片
    if (selType == SessionType.C2C) {//向好友发图片
        businessType = UploadPicBussinessType.C2C_MSG;
    } else if (selType == SessionType.GROUP) {//发群图片
        businessType = UploadPicBussinessType.GROUP_MSG;
    }
    //封装上传图片请求
    var opt = {
        'file': file, //图片对象
        'onProgressCallBack': onProgressCallBack, //上传图片进度条回调函数
        //'abortButton': document.getElementById('upd_abort'), //停止上传图片按钮
        'From_Account': loginInfo.identifier, //发送者帐号
        'To_Account': selToID, //接收者
        'businessType': businessType//业务类型
    };
    //上传图片
    webim.uploadPic(opt,
            function (resp) {
                //上传成功发送图片
                sendPic(resp);
                $('#upload_pic_dialog').modal('hide');
            },
            function (err) {
                alert(err.ErrorInfo);
            }
    );
}
```

## 上传图片 （低版本浏览器IE8、9）

在低版本浏览器（IE8、9）中，demo采用了表单来上传图片，最大支持10M图片的上传。

```
/* function submitUploadFileForm  
 *   上传图片(低版本ie)
 * params:
 *   cbOk	- function()类型, 成功时回调函数
 *   cbErr	- function(err)类型, 失败时回调函数, err为错误对象
 * return:
 *   (无)
 */
submitUploadFileForm: function(options, cbOk, cbErr) {},
```

**示例：**

```
//上传图片(用于低版本IE)
function uploadPicLowIE() {
    var businessType;//业务类型，1-发群图片，2-向好友发图片
    if (selType == webim.SESSION_TYPE.C2C) {//向好友发图片
        businessType = webim.UPLOAD_PIC_BUSSINESS_TYPE.C2C_MSG;
    } else if (selType == webim.SESSION_TYPE.GROUP) {//发群图片
        businessType = webim.UPLOAD_PIC_BUSSINESS_TYPE.GROUP_MSG;
    }
    //封装上传图片请求
    var opt = {
        'formId': 'updli_form', //上传图片表单id
        'fileId': 'updli_file', //file控件id
        'To_Account': selToID, //接收者
        'businessType': businessType//图片的使用业务类型
    };
    webim.submitUploadFileForm(opt,
                        function (resp) {
                            $('#upload_pic_low_ie_dialog').modal('hide');
                            //发送图片
                            sendPic(resp); 
                        },
                        function (err) {
                            $('#upload_pic_low_ie_dialog').modal('hide');
                            alert(err.ErrorInfo);
                        }
    );
}
```

## 发送消息（图片） 

在IE9（含）以下浏览器，sdk采用了jsonp方法解决ajax跨域问题，由于jsonp是采用get方法传递数据的，且get存在数据大小限制（不同浏览器不一样），所以暂不支持异步发送图片。

```
/* function sendMsg
 *   发送一条消息
 * params:
 *   msg	- webim.Msg类型, 要发送的消息对象
 *   cbOk	- function()类型, 当发送消息成功时的回调函数
 *   cbErr	- function(err)类型, 当发送消息失败时的回调函数, err为错误对象
 * return:
 *   (无)
 */
sendMsg: function(msg, cbOk, cbErr) {},
```

**示例： **

```
//发送图片
function sendPic(images) {
    if (!selToID) {
        alert("您还没有好友，暂不能聊天");
        return;
    }
    if (!selSess) {
        selSess = new webim.Session(selType, selToID, selToID, friendHeadUrl, 
        Math.round(new Date().getTime() / 1000));
    }
    var msg = new webim.Msg(selSess, true);
    var images_obj = new webim.Msg.Elem.Images(images.File_UUID);
    for (var i in images.URL_INFO) {
        var img = images.URL_INFO[i];
        var newImg;
        var type;
        switch (img.PIC_TYPE) {
            case 1://原图
                type = 1;//原图
                break;
            case 2://小图（缩略图）
                type = 3;//小图
                break;
            case 4://大图
                type = 2;//大图
                break;
        }
        newImg = new webim.Msg.Elem.Images.Image(type, img.PIC_Size, img.PIC_Width, 
        img.PIC_Height, img.DownUrl);
        images_obj.addImage(newImg);
    }
    msg.addImage(images_obj);
    //调用发送图片接口
    webim.sendMsg(msg, function (resp) {
        addMsg(msg);
        curMsgCount++;
    }, function (err) {
        alert(err.ErrorInfo);
    });
}
```

## 播放语音 

目前web端只支持显示并播放android或ios im demo发的语音消息，暂不支持上传并发送语音消息。使用audio控件来播放语音，注意，确保其他终端上传的语音格式是mp3格式（所有主流浏览器下的audio控件都兼容mp3，除了IE8下不支持使用audio标签播放语音）。

解析语音消息请参考第7节。


## 下载文件 

目前web端只支持显示并下载android或ios im demo发的文件消息，暂不支持上传并发送文件消息。 

解析文本消息请参考第8节。


## 发送消息(自定义)

```
/* function sendMsg
	 *   发送一条消息
	 * params:
	 *   msg	- webim.Msg类型, 要发送的消息对象
	 *   cbOk	- function()类型, 当发送消息成功时的回调函数
	 *   cbErr	- function(err)类型, 当发送消息失败时的回调函数, err为错误对象
	 * return:
	 *   (无)
	 */
	sendMsg: function(msg, cbOk, cbErr) {},
```

**示例： **

发送自定义消息入口：
![](//mccdn.qcloud.com/static/img/6ad98f6243363ced652f46a9fed727ba/image.png)
```
//弹出发自定义消息对话框
function showEditCustomMsgDialog() {
    $('#ecm_form')[0].reset();
    $('#edit_custom_msg_dialog').modal('show');
}
//发送自定义消息
function sendCustomMsg() {
    if (!selToID) {
        alert("您还没有好友或群组，暂不能聊天");
        return;
    }
    var data = $("#ecm_data").val();
    var desc = $("#ecm_desc").val();
    var ext = $("#ecm_ext").val();

    var msgLen = webim.Tool.getStrBytes(data);

    if (data.length < 1) {
        alert("发送的消息不能为空!");
        return;
    }
    var maxLen, errInfo;
    if (selType == webim.SESSION_TYPE.C2C) {
        maxLen = webim.MSG_MAX_LENGTH.C2C;
        errInfo = "消息长度超出限制(最多" + Math.round(maxLen / 3) + "汉字)";
    } else {
        maxLen = webim.MSG_MAX_LENGTH.GROUP;
        errInfo = "消息长度超出限制(最多" + Math.round(maxLen / 3) + "汉字)";
    }
    if (msgLen > maxLen) {
        alert(errInfo);
        return;
    }

    if (!selSess) {
        selSess = new webim.Session(selType, selToID, selToID, friendHeadUrl, Math.round(new Date().getTime() / 1000));
    }
    var msg = new webim.Msg(selSess, true,-1,-1,-1,loginInfo.identifier,0,loginInfo.identifierNick);
    var custom_obj = new webim.Msg.Elem.Custom(data, desc, ext);
    msg.addCustom(custom_obj);
    //调用发送消息接口
    webim.sendMsg(msg, function (resp) {
        if(selType==webim.SESSION_TYPE.C2C){//私聊时，在聊天窗口手动添加一条发的消息，群聊时，长轮询接口会返回自己发的消息
            addMsg(msg);
        }
        $('#edit_custom_msg_dialog').modal('hide');
    }, function (err) {
        alert(err.ErrorInfo);
    });
}
```

## 获取未读c2c消息 

```
/* function syncMsgs
 *   拉取最新C2C消息
 *   一般不需要使用方直接调用, SDK底层会自动同步最新消息并通知使用方,
 *   一种有用的调用场景是用户手动触发刷新消息
 * params:
 *   cbOk	- function(notifyInfo)类型, 当同步消息成功时的回调函数, 
*     notifyInfo同上面cbNotify中的说明,
 *    如果此参数为null或undefined则同步消息成功后会像自动同步那样回调cbNotify
 *   cbErr	- function(err)类型, 当同步消息失败时的回调函数, err为错误对象
 * return:
 *   (无)
 */
syncMsgs: function(cbOk, cbErr) {},
```

**示例: **

```
webim.syncMsgs(onMsgNotify);
```

## 获取c2c历史消息

```
/* function getC2CHistoryMsgs
 * 拉取C2C漫游消息
 * params:
 *   options	- 请求参数
 *   cbOk	- function(msgList)类型, 成功时的回调函数, msgList为消息数组，格式为[Msg对象],
 *   cbErr	- function(err)类型, 失败时回调函数, err为错误对象
 * return:
 *   (无)
 */
getC2CHistoryMsgs: function(options, cbOk, cbErr) {},
```

示例:

```
 //获取最新的c2c历史消息,用于切换好友聊天，重新拉取好友的聊天消息
var getLastC2CHistoryMsgs = function (cbOk, cbError) {

    if (selType == webim.SESSION_TYPE.GROUP) {
        alert('当前的聊天类型为群聊天，不能进行拉取好友历史消息操作');
        return;
    }
    var lastMsgTime = 0;//第一次拉取好友历史消息时，必须传0
    var msgKey = '';
    var options = {
        'Peer_Account': selToID, //好友帐号
        'MaxCnt': reqMsgCount, //拉取消息条数
        'LastMsgTime': lastMsgTime, //最近的消息时间，即从这个时间点向前拉取历史消息
        'MsgKey': msgKey
    };
    webim.getC2CHistoryMsgs(
            options,
            function (resp) {
                var complete = resp.Complete;//是否还有历史消息可以拉取，1-表示没有，0-表示有
                var retMsgCount = resp.MsgCount;//返回的消息条数，小于或等于请求的消息条数，小于的时候，说明没有历史消息可拉取了

                if (resp.MsgList.length == 0) {
                    webim.Log.error("没有历史消息了:data=" + JSON.stringify(options));
                    return;
                }
                getPrePageC2CHistroyMsgInfoMap[selToID] = {//保留服务器返回的最近消息时间和消息Key,用于下次向前拉取历史消息
                    'LastMsgTime': resp.LastMsgTime,
                    'MsgKey': resp.MsgKey
                };
                if (cbOk)
                    cbOk(resp.MsgList);
            },
            cbError
            );
};
```

## 获取群历史消息

```
/* function syncGroupMsgs
 * 拉取群漫游消息
 * params:
 *   options	- 请求参数
 *   cbOk	- function()类型, 成功时回调函数
 *   cbErr	- function(err)类型, 失败时回调函数, err为错误对象
 * return:
 *   (无)
 */
syncGroupMsgs: function(options, cbOk, cbErr) {},
```

**示例: **

```
//获取最新的群历史消息,用于切换群组聊天时，重新拉取群组的聊天消息
var getLastGroupHistoryMsgs = function (cbOk, cbError) {

    if (selType == webim.SESSION_TYPE.C2C) {
        alert('当前的聊天类型为好友聊天，不能进行拉取群历史消息操作');
        return;
    }

    getGroupInfo(selToID, function (resp) {
        //拉取最新的群历史消息
        var options = {
            'GroupId': selToID,
            'ReqMsgSeq': resp.GroupInfo[0].NextMsgSeq - 1,
            'ReqMsgNumber': reqMsgCount
        };
        if (options.ReqMsgSeq == null || options.ReqMsgSeq == undefined || options.ReqMsgSeq<=0) {
            webim.Log.warn("该群还没有历史消息:options=" + JSON.stringify(options));
            return;
        }
        webim.syncGroupMsgs(
                options,
                function (msgList) {
                    if (msgList.length == 0) {
                        webim.Log.error("该群没有历史消息了:options=" + JSON.stringify(options));
                        return;
                    }
                    getPrePageGroupHistroyMsgInfoMap[selToID] = {
                        "ReqMsgSeq": msgList[msgList.length - 1].MsgSeq - 1
                    };
                    if (cbOk)
                        cbOk(msgList);
                },
                function (err) {
                    alert(err.ErrorInfo);
                }
        );
    });
};
```

## 获取所有会话 

webim.Session对象,简单理解为最近会话列表的一个条目 .

webim.MsgStore是消息数据的Model对象,它提供接口访问当前存储的会话和消息数据。 

**示例： **

```
var sessMap = webim.MsgStore.sessMap();
for (var i in sessMap) {
       sess = sessMap[i];
       if (selToID != sess.id()) {//更新其他聊天对象的未读消息数
           updateSessDiv(sess.type(), sess.id(), sess.unread());
       }
}
```

## 获取会话 

可以根据会话类型和会话ID取得相应会话。 

**示例：** 

```
selSess = webim.MsgStore.sessByTypeId(selType, selToID);
```
## 获取最近联系人

可以拉取最近联系人的会话列表。 

**示例：** 

```
webim.getRecentContactList({
	'Count': 10 //最近的会话数 ,最大为100
},function(resp){
	//业务处理
},function(resp){
	//错误回调
});
```
请参考demo代码 recentcontact/recent_contact_list_manager.js

## 删除最近联系人

删除最近联系人中的一条会话。 

**示例：** 

注：
```
sess_type == 'C2C' ? 1 : 2;
```

```
	function delChat(sess_type, to_id) {
	    var data = {
	        'To_Account': to_id,
	        'chatType': sess_type
	    }
	    webim.deleteChat(
	        data,
	        function(resp) {
	            $("#sessDiv_" + to_id).remove();
	        }
	    );
	}
```

请参考demo代码 switch_chat_obj.js