# ARCHITECTURE · 模块地图与数据流

## 顶层结构

```
项目根/
├── App/                  iOS 主 App(SwiftUI)
│   ├── Services/         无 UI 的核心逻辑(分析/AI/存储/连接)
│   ├── Views/            页面
│   └── Assets.xcassets   插画/图标(错误库 220 张、训练库配图等)
├── WatchApp/             watchOS App
├── Shared/               双端共享(消息协议/公共工具)
├── server/               FastAPI 后端(本地开发用,见 BACKEND.md)
├── web/                  网页版(展示/预约)
└── project.yml           xcodegen 工程描述(增删文件后重跑 xcodegen generate)
```

## 核心数据流(视频分析主线)

```
拍摄/相册导入
   → 录像落盘 Documents/Recordings/xxx.mov
   → (若戴表) 挥拍事件+心率经 WCSession 到手机,写同名 .swings.json 档案(sidecar)
   → 后台分析管线 AnalysisCenter:
        Vision 姿态逐帧 → poses 缓存(原子写)
        → 挥拍切分(无表时纯视频检测;有表时用手表时刻对齐)
        → 多人画面:按挥拍时刻找"谁在挥拍"自动认人,可手动重选
   → 指标计算(肘角/击球点/挥速) → 报告页
   → AI 教练点评 + 评级 自动并发生成(走服务器代理) → 缓存 .review/.rating
   → 本地通知"报告已生成" → 点击深链直达详情
```

关键原则:**视频和姿态数据永不上传**;发给 AI 的只有指标摘要+少量裁剪关键帧。

## 双端协议(Shared/SyncMessages)

- 消息 = type + JSON payload,MessageCodec 编解码;
- 通道:实时 sendMessage(可达时)+ transferUserInfo(断连排队补传),接收端两个回调都要处理;
- 会话指令(开始/停止)带时间戳,判陈旧规则见 PITFALLS #29;
- 数据事件(挥拍/心率)带**场次键**(sessionStartedAt),按键归属,见 PITFALLS #30;
- 对时:ping-pong 取最小 RTT,换算手表↔手机时钟偏移(PITFALLS #32)。

## 关键类职责(iOS)

| 类 | 职责 |
|----|------|
| CameraRecorder | 采集会话/录制,保活策略见 PITFALLS #7 |
| AnalysisCenter | 后台分析任务队列(全局单例,页面离开不中断) |
| PoseAnalyzer | Vision 姿态 + poses 缓存(原子写) |
| PhoneConnectivityManager | WCSession 手机侧:指令/事件接收、场次归属、档案落库 |
| AICoachService / AIRatingService | 点评/评级:prompt 组装、代理请求、三层解析、缓存 |
| CourseCenter | 课程引擎:画像→模板秒配→AI 定制/微调,打卡与连续天数 |
| RecordingStore | 录像扫描 + sidecar 读写(列表 IO 放后台线程) |
| CommunityService | 社区 API 封装(静态方法+统一错误类型) |
| ReportCenter | 报告自动生成协调 + 本地通知 + 深链通道 |

## 手表侧(WatchApp)

| 类 | 职责 |
|----|------|
| MotionStreamer | CoreMotion 挥拍检测(峰值角速度+滞回+不应期),读数发布 |
| WorkoutManager | HKWorkoutSession 心率采集(回调切主线程,PITFALLS #27) |
| WatchConnectivityManager | 双通道发送 + 指令接收(陈旧判定) |
| WatchSessionController | 训练状态机(空闲/进行/小结),小结快照兜底(PITFALLS #33) |

## AI 集成模式

- 唯一模型:doubao-seed-2.0-lite(多模态,一个模型吃视觉+文字全部场景);
- 客户端零配置:默认走自建服务器代理(Key 在服务器,按设备每日限额);
  用户可选填自己的 Key 直连(不限额);
- 强 JSON 场景规则见 PITFALLS #9;内容库随行策略见 #11;
- 单次成本参考(实测):点评(4 图+全目录)≈13K 输入+1.5K 输出;评级(6 图)≈9.4K+1.3K;纯文字点评≈0.4K+1.2K。
