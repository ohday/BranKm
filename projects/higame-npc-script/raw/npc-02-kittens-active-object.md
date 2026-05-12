---
id: npc-02
title: Kittens — ActiveObject 与 Mail
sources:
  - E:/HiProject/PerforceDev/UnrealEngine/Projects/HiGame/Content/Script/kittens/active_object/active_object.lua
  - E:/HiProject/PerforceDev/UnrealEngine/Projects/HiGame/Content/Script/kittens/active_object/active_object_const.lua
  - E:/HiProject/PerforceDev/UnrealEngine/Projects/HiGame/Content/Script/kittens/active_object/active_object_mgr.lua
  - E:/HiProject/PerforceDev/UnrealEngine/Projects/HiGame/Content/Script/kittens/active_object/active_object_proxy_ref.lua
  - E:/HiProject/PerforceDev/UnrealEngine/Projects/HiGame/Content/Script/kittens/active_object/active_object_ref.lua
  - E:/HiProject/PerforceDev/UnrealEngine/Projects/HiGame/Content/Script/kittens/active_object/mail.lua
  - E:/HiProject/PerforceDev/UnrealEngine/Projects/HiGame/Content/Script/kittens/active_object/mail_location.lua
  - E:/HiProject/PerforceDev/UnrealEngine/Projects/HiGame/Content/Script/kittens/active_object/mail_queue.lua
  - E:/HiProject/PerforceDev/UnrealEngine/Projects/HiGame/Content/Script/kittens/active_object/mail_stub.lua
  - E:/HiProject/PerforceDev/UnrealEngine/Projects/HiGame/Content/Script/kittens/active_object/mail_switcher.lua
  - E:/HiProject/PerforceDev/UnrealEngine/Projects/HiGame/Content/Script/kittens/active_object/stash_box.lua
updated: 2026-05-11
---

# npc-02 — Kittens ActiveObject 与 Mail

## Sources

全部 11 个文件均位于 `Content/Script/kittens/active_object/` 目录下,见 frontmatter `sources` 字段。

## 1. 类层次 (ActiveObject / ActiveObjectMgr / ActiveObjectRef / ProxyRef)

| 类 | 文件 | Kittens 基类 | 职责 |
|---|---|---|---|
| `ActiveObject` | active_object.lua | `Kittens.class('ActiveObject', nil)` | Actor 模型核心:持有 MailQueue、mail_switcher_stack,对外暴露 `become`/`unbecome`/`stash`/`unstash_all` |
| `ActiveObjectMgr` | active_object_mgr.lua | `Kittens.class('ActiveObjectMgr', nil)` | 本地 AO 注册表,管理 `__all_active_objects`(local)、`__remote_active_object_refs`(remote)、`__ao_proxy_refs`(proxy) 三张表 |
| `ActiveObjectRef` | active_object_ref.lua | `Kittens.class('ActiveObjectRef', nil)` | 邮箱地址 / 间接层,提供 location transparency;持有 `__mail_filter` 和 `__mail_dispatcher` 两个可选 MailSwitcher |
| `ActiveObjectProxyRef` | active_object_proxy_ref.lua | `Kittens.class('ActiveObjectProxyRef', nil)` | 跨生命周期代理;AO 未绑定时将 Mail 缓存到 `__pending_mail_queue`,绑定后自动 flush |

**ActiveObject 关键成员**:

```lua
self.__mail_queue = MailQueue:new(_context, self, _effect_handlers)
self.__mail_switcher_stack = {}          -- become/unbecome 栈
self.__curr_mail_switcher = nil          -- 栈顶快捷引用
```

**ActiveObjectMgr 创建流程**(`create_active_object`):分配 ID -> `MailStub:make_location` 生成 MailLocation -> 实例化 AO -> 写入 `__all_active_objects`。ProxyRef 绑定由调用者后续调用 `bind_ao_to_proxy_ref` 完成。

## 2. Mail 异步语义 (Enum_ReceiveMailMode 两种模式 verbatim)

`active_object_const.lua` 中定义了两种接收模式:

```lua
ActiveObjectConst.Enum_ReceiveMailMode = {
    fire_and_forget = 'fire_and_forget',
    request_and_response = 'request_and_response'
}
```

**fire_and_forget**:Mail 构造时 `__receive_promise` 为 nil,发送方不等待任何回执。`is_responsed()` 对此模式直接返回 true。

**request_and_response**:Mail 构造时创建 `Promise:new()`。Handler 返回值决定 settle 行为——可以是 `Promise`(lock_receive_promise)、`FulfilledResult`(立即 fulfill)、`Error`(reject)、或 nil(fulfill nil)。

**Effect Tags** 也定义在 const 文件中:

```lua
ActiveObjectConst.Enum_EffectTag = {
    SendMail = 'SendMail',
    RaiseReceiveMailException = 'RaiseReceiveMailException',
    Await = 'Await',
}
```

**预定义 Error 常量**(verbatim):`Error_Send_Mail_No_Handler`、`Error_Dispatch_Mail_No_Dispatcher`、`Error_Send_Remote_Mail_Failed`、`Error_Send_Remote_Mail_No_Stub`、`Error_Send_Mail_Invalid_Return_Type`、`Error_Send_Remote_Mail_Actor_Not_Exist`、`Error_Send_Mail_Receiver_Destroyed`。

## 3. MailSwitcher 路由模式 (case / copy_handler_item / 调用顺序)

`MailSwitcher` 实现 Actor 模型的 **become** 行为。核心是一张 `__mail_handlers` lookup table(key = mail_type string, value = handler function)。

**注册 handler**:

- `case(_mail_type, _mail_handler)` — 注册单个 mail_type,返回 self(链式调用)。
- `copy(_source_mail_switcher)` — 从另一个 MailSwitcher 批量拷贝全部 handler。
- `copy_handler_item(_handler_lut, _mail_type)` — 从外部 LUT 表中挑选单条 handler 拷贝。

**Default handler**:构造时若传入 `_default_handler`,会 `self:case(Const.Mail_Switcher_Default, _default_handler)`。Const 值为字符串 `'default'`。

**_receive 调度顺序** (`mail_switcher.lua:37-92`):
1. 检查 mail 是否已 response(防重复投递)。
2. 若 mail 处于 stashed 状态则 `_unstash()`。
3. 按 `mail_type` 查表;未命中则回退到 `Mail_Switcher_Default`。
4. 若仍无 handler -> `send_error(Error_Send_Mail_No_Handler)`。
5. `pcall(handler, self.__handler_owner, _mail, _cancel_token)` 安全调用。
6. 对 request_and_response 模式,根据 handler 返回值类型执行 `lock_receive_promise` / `_fulfill` / `send_error`。

**_filter_receive**:与 `_receive` 类似但 handler 签名多返回一个 `filter_out` boolean,用于 ActiveObjectRef 的 `__try_filter_mail` 路径。

## 4. Stash 机制 (StashBox — 何时入栈, 何时出栈)

**StashBox** (`stash_box.lua`):极简栈结构,核心只有 `__stash_box` 数组。

```lua
function StashBox:stash(_mail)
    _mail:_stash()                        -- 标记 mail.__is_stashed = true
    table.insert(self.__stash_box, 1, _mail)  -- 插入头部(逆序)
end

function StashBox:pop_all_mails()
    local mails = self.__stash_box
    self.__stash_box = {}
    return mails                          -- 一次性清空
end
```

**入栈时机**:NPC handler 在 MailSwitcher 的 handler 中调用 `ActiveObject:stash(_mail)` -> `MailQueue:stash` -> `StashBox:stash`。被 stash 的 mail 不会触发 response 流程。

**出栈时机**:调用 `ActiveObject:unstash_all()` -> `MailQueue:unstash_all()` -> `StashBox:pop_all_mails()`。所有 stashed mails 按到达顺序插回 MailQueue 头部。`_unstash()` 操作延迟到 MailSwitcher 的 `_receive` 下一次处理该 mail 时才执行。

**典型用法**:状态机 `become` 切换时,当前 handler 无法处理的 mail 先 stash,切换到新状态后 `unstash_all` 重新投递。

## 5. ProxyRef 跨服务器逻辑 (与 ActiveObjectRef 区别)

| 维度 | ActiveObjectRef | ActiveObjectProxyRef |
|---|---|---|
| 生命周期 | 绑定到具体 AO 实例 | 跨 AO destroy/re-create 周期存活 |
| AO 不存在时 | local: send_error; remote: 走 MailStub | 缓存 mail 到 `__pending_mail_queue`,等待 `_bind_ao_ref` 后 flush |
| 可靠投递 | 无 | `send_reliable_mail` — 若 AO 被 destroy(Error_Send_Mail_Receiver_Destroyed),mail 保留在 `__reliable_mails` 中,下次 bind 时重发 |
| 管理方 | ActiveObjectMgr 的 `__all_active_objects` / `__remote_active_object_refs` | ActiveObjectMgr 的 `__ao_proxy_refs` |

**ProxyRef 绑定/解绑**:
- `_bind_ao_ref(_ao_ref)` — 保存 ref,flush pending queue,flush reliable mails。
- `_unbind_ao_ref()` — 清空 ref,后续 send_mail 回到 pending 模式。
- `destroy_active_object` 时 Mgr 自动调用 `proxy_ref:_unbind_ao_ref()`。

**Reliable mail 重试逻辑**(`__dispatch_reliable_mail`):以 `request_and_response` 模式发送,监听 promise settled。fulfilled 则从 reliable 列表移除并 fulfill completion_promise;rejected 为 `Error_Send_Mail_Receiver_Destroyed` 则保留待下次 bind 重发;其他 error 直接传播给调用方。

**获取 ProxyRef 的三个 API**:`get_local_ao_proxy_ref`(优先绑定 local AO)、`get_remote_ao_proxy_ref`(优先绑定 remote ref)、`get_ao_proxy_ref`(先 local 后 remote)。

## 6. MailQueue + MailStub + MailLocation 关系

**MailQueue** (`mail_queue.lua`) — 每个 ActiveObject 持有一个。核心职责:
- `__mail_queue` 数组:Mail 的 FIFO 缓冲区。
- `__stash_box`:StashBox 实例。
- `__mail_switcher`:当前活跃的 MailSwitcher 引用(由 `set_mail_switcher` 设置)。
- `__async_mail_loop`:在 Kittens 协程中通过 `fire_and_forget` 启动的无限循环,取队头 mail 交给 `__mail_switcher:_receive()`。
- 通过 `__destroying_cts`(CancelTokenSource)实现优雅关闭:停止接收新 mail,对已排队的 request_and_response mail 返回 `Error_Send_Mail_Receiver_Destroyed`。

**MailStub** (`mail_stub.lua`) — 远程通信抽象层:
- 与 ActiveObjectMgr 双向绑定(构造时 `_active_object_mgr:_set_mail_stub(self)`)。
- `encode_mail` / `decode_mail`:Mail <-> JSON 序列化,通过 `thirdparty.json`。
- `send_remote_mail`:fire_and_forget 走 `impl_send`,request_and_response 走 `impl_send_for_response`(返回 Promise)。
- `receive_remote_mail`:反序列化后调用 `ActiveObjectMgr:_receive_remote_mail`。
- 三个 **必须子类重写** 的方法:`impl_send`、`impl_send_for_response`、`make_location`。

**MailLocation** (`mail_location.lua`) — 抽象寻址:
- `flat_class('MailLocation')`,仅定义 `marshal()` / `unmarshal(_location_str)` 两个空方法。
- 具体实现由 MailStub 子类通过 `make_location` 创建,承载服务器地址等路由信息。

**三者协作链路**(本地发送):
`ActiveObjectRef:send_mail` -> `Mail:new` -> `ActiveObjectRef:_send_mail` -> (local) `ActiveObject:_receive_mail` -> `MailQueue:append_mail` -> mail loop -> `MailSwitcher:_receive`

**三者协作链路**(远程发送):
`ActiveObjectRef:_send_mail` -> (remote) `ActiveObjectMgr:_send_remote_mail` -> `MailStub:send_remote_mail` -> `encode_mail` + `impl_send` / `impl_send_for_response`

## 7. 跨页关联线索

| 关联页 | 关联点 |
|---|---|
| npc-01 | NpcActiveObject 继承 ActiveObject,initialize 中调 super.initialize 创建 MailQueue |
| npc-03 | StateBase 通过 ActiveObject:become/unbecome 切 MailSwitcher 实现状态切换 |
| npc-05 | NPC 状态实现里大量 `MailSwitcher:case` 注册 handler、`active_object:stash` 入栈 |
| npc-06 | 跨服务器 NPC 用 ActiveObjectProxyRef 实现可靠投递(reliable mail) |
| npc-15 | `Const.Enum_Mail_Type` 60+ 类型即在 NPC 状态机里通过 case/copy_handler_item 路由 |
