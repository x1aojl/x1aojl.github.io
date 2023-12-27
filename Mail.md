### 邮件系统
#### 目标
游戏中的邮件主要是用于向玩家发送消息或发放奖励。

#### 分类
游戏中的邮件按照发送类型分为私发邮件和广播邮件。
1. 私发邮件: 通常是玩家完成某个任务后，系统通过邮件向玩家发放任务奖励。
2. 广播邮件: 通常是服务器维护等影响全体玩家游戏体验或利益的情况发生后，系统通过邮件向全体玩家发放补偿奖励。

#### 流程
1. 私发邮件：
	1. 找到玩家：弱联网单机游戏中，此时玩家一定在线
	2. 发送邮件

2. 广播邮件：
	1. 将邮件加入到邮件缓存中，并存入数据库（服务器重启后找回数据）
	2. 找到所有玩家
		1. 玩家在线：发送邮件，玩家记录收到的最新邮件的时间戳
		2. 玩家不在线：等到下一次上线时，通过与邮件缓存中所有邮件的时间戳进行对比，拉取所有未收到的新邮件

MAIL.gs
```
#pragma parallel

#include "Game/Include/Rid.gsh"
#include "Game/Include/Common.gsh"

import gs.util.*;
import Common.RID;
import Common.TIME;
import External.MONGO;
import Game.Manager.ACCOUNT;

// 所有广播邮件缓存(与数据库保持同步)
readonly share_value mails_;

void create()
{
    // 从game数据库中导出邮件数据
    let string err, array records = MONGO.find_many(GAME_DB, MAIL_COLL, READONLY({ }));
    if (err)
        error("Game导出邮件数据失败. error=%O", err);

    // 保存邮件数据
    map mails = { };
    for (map data : records)
    {
        mails.set(data["rid"], {
            "rid" : data["rid"],
            "title" : data["title"],
            "content" : data["content"],
            "extra" : data["extra"],
            "timestamp" : data["timestamp"]
        });
}

// 初始化邮件缓存
mails_ := share_value.allocate("mails_", mails);
}

// 发送邮件
public void send(mixed receiver, string title, string content, array extra = [ ])
{
    // 获取玩家对象
    object user = ACCOUNT.query(receiver);
    if (user)
    {
        // 创建邮件
        map mail = create_mail(title, content, extra);

        // 发送邮件给玩家
        user => send_mail(mail);
    }
    else
    {
        // 邮件发送失败
        TRACE_WARN("mail send failed. receiver=%O, title=%O", receiver, title);
    }
}

// 广播邮件
public void broadcast(string title, string content, array extra = [ ])
{
    // 创建邮件
    map mail = create_mail(title, content, extra);

    // RO
    READONLY(mail);

    // 加锁
    try_lock(mails_)
    {
        // 写入邮件缓存
        mails_.set(mail["rid"], mail);
    }

    // 写入数据库
    if (MONGO.insert_one(GAME_DB, MAIL_COLL, mail))
    {
        // 成功直接广播所有在线玩家
        map users = ACCOUNT.query_all();
        for (object user : users.values())
            user => send_mail(mail.deep_dup());
    }
    else
    {
        // 加锁
        try_lock(mails_)
        {
            // 从邮件缓存中移除邮件
            mails_.delete_by_key(mail["rid"]);
        }

        // 邮件广播失败
        TRACE_WARN("mail broadcast failed. title=%O", title);
    }
}

// 拉取邮件
public array pull(int timestamp)
{
    array result = [ ];

    // 所有邮件
    map mails;

    // 加锁
    try_lock(mails_)
    {
        mails = mails_.fetch_value();
    }

    // 遍历所有邮件
    for (string rid, map mail : mails)
    {
        // 对比时间戳，将新邮件全部取走
        if (mail["timestamp"] > timestamp)
            result.push_back(mail.deep_dup());
    }

    return result;
}

// 创建邮件
map create_mail(string title, string content, array extra = [ ])
{
    // 分配rid
    string rid = RID.alloc(RID_MAIL);

    // 申请时间
    int timestamp = TIME.seconds();

    // 初始化邮件数据
    map mail = {
        "rid" : rid,
        "title" : title,
        "content" : content,
        "extra" : extra,
        "timestamp" : timestamp
    };

    return mail;
}
```

FMail.gs
```
#include "Game/Include/Event.gsh"
#include "Game/Include/NetCmd.gsh"

import gs.util.*;
import Game.Manager.MAIL;
import Game.Manager.ACCOUNT;
import Game.ETC.GLOBAL_SETTINGS_CFG;

void create()
{
    this.subscribe_event(EID_GAME_START_OK, (: process_game_start_ok:));
}

// 发送邮件
public void send_mail(map mail)
{
    // 获取所有邮件
    mixed mails = this.query("mails");
    if (mails.length() >= GLOBAL_SETTINGS_CFG.query("mail_max"))
    {
        // 超过邮件数量上限，顶掉最前面的
        map rid = mails[0]["rid"];
        mails.delete(mails[0]);

        // 通知客户端删除被顶掉的邮件
        this.send(MSG_REMOVE_MAIL, rid);
    }

    // 添加新邮件
    mails.push_back(mail);

    // 更新玩家记录的邮件时间戳
    if (this.query("mail_timestamp") < mail["timestamp"])
        this.set("mail_timestamp", mail["timestamp"]);

    // 通知客户端有一封新邮件
    this.send(MSG_NEW_MAIL, mail);
}

// 阅读邮件
public bool read_mail(string mail_rid)
{
    // 获取邮件数据
    map mail = get_mail(mail_rid);
    if (mail == nil)
    {
        // 阅读邮件失败
        TRACE_WARN("mail read failed. mail_rid=%O", mail_rid);
        return false;
    }

    // 取出邮件奖励
    array rewards = mail["extra"];

    // 先删除邮件
    mixed mails = this.query("mails");
    mails.delete(mail);

    // 再发放邮件奖励
    if (rewards)
        this.reward_raw(rewards);

    // 通知客户端删除已阅读的邮件
    this.send(MSG_REMOVE_MAIL, mail_rid);

    return true;
}

// 阅读所有邮件
public bool read_all_mail()
{
    // 获取所有邮件
    mixed mails = this.query("mails");
    for (int i = mails.length() - 1; i >= 0; --i)
        // 逆序阅读所有邮件
        read_mail(mails[i]["rid"]);

    return true;
}

// 获取邮件
map get_mail(string mail_rid)
{
    // 获取所有邮件
    mixed mails = this.query("mails");
    for (map mail : mails)
    {
        if (mail["rid"] == mail_rid)
            return mail;
    }

    // 没有找到邮件，返回空
    return nil;
}

void process_game_start_ok()
{
    // 获取玩家记录的邮件时间戳
    int timestamp = this.query("mail_timestamp");

    // 玩家没有记录的邮件时间戳，则取玩家注册时间
    if (timestamp == 0)
        timestamp = this.query("create_time");

    // 拉取邮件列表，发送到玩家邮箱中
    array mails = MAIL.pull(timestamp);
    for (map mail : mails)
        send_mail(mail);
}
```
