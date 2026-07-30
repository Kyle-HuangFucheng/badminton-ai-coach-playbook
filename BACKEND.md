# BACKEND · FastAPI 后端:本地跑法与接口清单

> 开发期一律连**本地**后端;线上部署另行处理,任何凭证都不在本仓库。

## 本地一键跑

```bash
cd server
python3 -m pip install -r requirements.txt
python3 -m uvicorn main:app --host 127.0.0.1 --port 8091
# 冒烟: curl http://127.0.0.1:8091/api/health
```

铁律:

- **单 worker**(SQLite WAL + 内存限额计数器,多 worker 会写冲突且各算各的额度);
- AI 代理需要环境变量 `ARK_API_KEY`(在火山方舟平台自行申请,**不要写进任何提交**);
  没有 Key 时后端照常起,只有 AI 接口报"未配置";
- iOS 端的服务器地址集中在常量里,开发期改成 `http://127.0.0.1:8091`
  (模拟器可直连本机;真机换成电脑局域网 IP)。

## 模块与接口清单

### 社区 community.py(/api/community)
| 接口 | 说明 |
|------|------|
| GET /posts | 信息流(置顶优先),带点赞数/我的点赞/评论数/配图/头像 |
| POST /posts | 发帖(类型/内容/城市/球馆/联系方式/图片文件名);每设备每日限条数;脏词过滤 |
| POST /posts/{id}/like | 点赞切换 |
| POST /posts/{id}/report | 举报(≥3 设备自动隐藏) |
| POST /posts/{id}/delete | 删除(本人或管理员令牌) |
| POST /posts/{id}/pin | 置顶(仅管理员令牌) |
| POST /upload_image | 传图(≤3MB,token_hex 落盘,登记归属) |
| GET/POST /posts/{id}/comments | 评论列表/发评论 |
| POST /avatar | 自定义头像(归属校验) |

### 私聊 dm.py(/api/dm)
会话开启(从帖子发起)/会话列表/消息列表/发消息/已读;客户端轮询拉取。

### 登录 auth.py(/api/auth)
| 接口 | 说明 |
|------|------|
| POST /apple | Sign in with Apple:服务端 JWKS 验签 identityToken,apple_sub↔deviceId 绑定;响应含 isAdmin/adminToken(管理员邮箱白名单) |
| POST /delete | 注销(5.1.1(v) 合规:服务端删档成功客户端才准登出) |

### AI 代理 server.py
| 接口 | 说明 |
|------|------|
| POST /api/ai/chat | 透传 chat/completions 到方舟;服务端强制锁定模型与 max_tokens 上限;按设备每日限额+全站总量;管理员令牌免额;自带 Key(X-User-Key 头)不占额 |
| GET /api/health | 存活探针 |
| GET /api/admin/stats | 运营看板数据(仅管理员令牌) |

### 其他
gyms.py 球馆目录/点评;静态托管:web/ 网页版 + /media 媒体(注意 PITFALLS #23 归属与可枚举性)。

## 安全清单(上线前过一遍)

1. 上传文件名不可枚举(token_hex),引用前校验归属;
2. 限流不信任 X-Forwarded-For(无可信反代时);
3. 删除/置顶/官方帖/看板全部走管理员令牌(登录下发,库里校验),不是裸口令;
4. 任何密钥只进环境变量;仓库、日志、错误信息里都不能出现;
5. 客户端敏感操作(注销)必须服务端确认成功才改本地状态。
