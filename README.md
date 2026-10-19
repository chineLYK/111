# admin 后台管理系统
admin 后台管理系统 · 功能说明书
编写口径：按“后台管理系统”的功能体系组织，以菜单和角色为主线描述系统能做什么、怎么做、对应哪些接口与数据。

标记说明：

【已实现】 仓库中存在可运行的代码路径；
【部分实现】 有数据模型 / 接口 / 注解，但关键逻辑缺失；
【规划】 属于系统设计目标，当前代码尚未落地。
文末附《落地情况对照表》，用于区分“设计功能”与“现有实现”。

1. 系统定位
admin 是一套面向中小企业的轻量级后台管理系统，把“系统管理”和“业务管理”放在同一个后台里：

系统管理：用户、角色、权限、菜单、数据字典、模块元数据、操作日志、接口追踪、文件、参数；
业务管理：客户、供应商、商品、项目、往来账户、销售订单、库存；
通用能力：一套由元数据驱动的 REST 接口，为每张登记过的表自动提供列表、新增、修改、删除能力，避免为每张表重复写 Controller/Service；
技术形态：Spring Boot 微服务拆分（system / module / ui），Spring Data JPA + MySQL，Feign 做服务门面，Vue + WebJars 做前端资源，Shiro 预留认证授权。
设计原则：

约定优于配置：模块名、表名、实体名、Repository 名按约定推导，减少配置。
元数据驱动：metadata 表登记一个模块，通用接口即可操作该模块。
CQRS 分离：查询走 QueryEvent，写操作走 CommandEvent，由统一调度器分派。
字典注解化：实体字段标 @Ddic，查询结果自动把编码翻译成中文。
统一出口：RestMessage 统一响应，@RestControllerAdvice 统一异常。
2. 部署形态
服务	上下文路径	端口	职责
admin-system	/admin/system	9091	系统管理域 + 通用 REST 入口
admin-module	/admin/module	9090	业务域实体与数据访问
admin-ui	/admin	9090	静态前端、Feign 门面、统一异常处理
MySQL	—	3306	admin-system、admin-module、admin 三个库
Redis（可选）	—	6379	会话/缓存，配置已预留
部署时 admin-ui 与 admin-module 需要分配不同端口；当前两份 application.yml 都写成 9090。

3. 角色与权限模型【规划】
系统按 RBAC + 数据范围 设计：

角色	典型用户	核心权限
超级管理员	IT 负责人	全部菜单、全部数据、系统参数
系统管理员	运维/实施	用户、角色、菜单、字典、元数据、日志、追踪
业务管理员	业务主管	业务模块全部数据、报表、审批
业务操作员	销售/采购/仓管	本人或本部门业务数据的录入与查询
只读审计	财务/审计	全局查询、日志与追踪查看，不可修改
权限粒度分四层：

菜单权限：控制左侧导航是否可见；
页面权限：控制路由是否可进入；
按钮权限：控制新增/编辑/删除/导出等按钮显隐；
接口权限：服务端校验，防止绕过前端直接调接口。
数据范围：全部 / 本部门 / 本人 / 自定义。

会话策略：登录后保持会话，默认 30 分钟无操作失效（Redis Session 配置在代码中已预留但被注释）。

4. 功能结构（菜单树）
代码块


首页 / 工作台
系统管理
 ├─ 用户管理
 ├─ 角色管理
 ├─ 权限资源
 ├─ 菜单管理
 ├─ 数据字典
 │   ├─ 字典分类
 │   └─ 字典明细
 ├─ 模块元数据
 ├─ 操作日志
 ├─ 接口追踪
 ├─ 文件管理
 └─ 参数配置
业务管理
 ├─ 基础资料
 │   ├─ 客户管理
 │   ├─ 供应商管理
 │   ├─ 商品管理
 │   └─ 项目管理
 ├─ 资金管理
 │   ├─ 客户账户
 │   └─ 供应商账户
 ├─ 订单管理
 │   └─ 销售订单
 └─ 库存管理
     └─ 商品库存
个人中心
 ├─ 个人信息
 └─ 修改密码
5. 功能详述
5.1 登录与认证
入口：系统登录页。

功能点	说明	状态
账号密码登录	校验账号、密码、状态，成功后建立会话	【规划】
登录失败提示	账号不存在 / 密码错误 / 账号停用分别提示	【规划】
未登录拦截	未认证请求返回 401，前端跳转登录页	【部分实现】仅异常处理器
登出	销毁会话，回到登录页	【规划】
记住我 / 超时	30 分钟无操作自动失效	【规划】
服务端约定：认证失败抛 AuthenticationException，由 RestExceptionHandler 统一转换为 RestMessage{success:false, code:401, body:"未授权"}。

5.2 工作台（首页）
区块	内容	状态
数据概览	今日订单数、待处理事项、库存预警数、接口错误数	【规划】
快捷入口	常用菜单快捷跳转	【规划】
最近操作	当前用户最近 10 条操作日志	【规划】
5.3 系统管理
5.3.1 用户管理【规划】
列表：账号、姓名、部门、角色、状态、最后登录时间、创建时间。
操作：新增、编辑、重置密码、启用/停用、分配角色、删除。
规则：
账号全局唯一，创建后不可修改；
密码加密存储，重置后首次登录强制修改；
停用账号立即禁止登录，但保留历史数据；
已产生业务数据的用户不允许物理删除，只能停用。
接口：/api/user/list、/save、/edit、/delete。
数据：sys_user、sys_user_role（待建）。
5.3.2 角色管理【规划】
列表：角色编码、角色名称、数据范围、关联用户数、状态。
操作：新增、编辑、分配权限、启用/停用、删除。
规则：角色编码唯一；删除前校验是否仍有用户引用；内置角色（超级管理员）不可删除。
接口：/api/role/list、/save、/edit、/delete、/grant。
数据：sys_role、sys_role_menu、sys_role_permission（待建）。
5.3.3 权限资源【规划】
形态：树形资源表，节点分“菜单 / 按钮 / 接口”三类。
字段：资源编码、名称、类型、父级、路由/组件、HTTP 方法、接口路径、图标、排序、是否可见。
规则：
编码唯一，采用 模块:动作 形式，例如 customer:add；
接口类资源必须绑定方法和路径，供服务端鉴权；
删除父节点前必须先处理子节点。
5.3.4 菜单管理【规划】
维护左侧导航树：目录、菜单、按钮三级。
支持调整父级、排序、显隐、图标、路由地址。
菜单变更后，前端刷新路由与权限缓存。
5.3.5 数据字典【部分实现】
后台维护“分类 + 明细”两级字典，前端下拉框、列表翻译、后端字段回填都依赖它。

功能点	说明	状态
字典分类维护	分类编码、中文名、备注	【部分实现】表与实体已有
字典明细维护	键、值、排序、启用、备注、扩展属性 1–3	【部分实现】表与实体已有
下拉数据接口	按分类编码返回启用项，按 sortNo 排序	【规划】
列表自动翻译	查询结果把 key 翻译为 value	【规划】
后端字段回填	实体字段标 @Ddic(name="分类", mapTo="目标字段")，查询后自动填充	【部分实现】注解与工具类已有，回填逻辑未启用
缓存与刷新	字典数据缓存，维护后主动失效	【规划】
数据结构：

ddic_category：id、category、cn_name、memo；
ddic_item：id、category_id、key、value、sort_no、enable、memo、attribute1~3。
约定：明细的 enable=1 才参与下拉与翻译；sort_no 决定展示顺序。

5.3.6 模块元数据【部分实现】
这是“元数据驱动 CRUD”的配置中心：一个模块登记一行，系统据此把该模块映射到实体、Repository 和字段。

字段	含义	示例
module	模块编码，也是接口路径变量	customer
cn_name	中文名称	客户
table_name	对应实体名/表名	CustomerEntity
sort_no	菜单排序	10
switch	开关位：启停、日志开关等	1
运行时行为：

应用启动时扫描 JPA 元模型，建立“实体名 → 实体类”映射；
读取 metadata 全表，按实体类名把 XxxEntity 推导为 XxxRepository，从 Spring 容器取出 Repository Bean；
反射得到实体的全部属性描述，供通用查询/表单渲染使用；
结果按 metadata 缓存，避免每次请求重新解析；
switch 预留给模块启停与日志开关（当前未参与判断）。
接口：/api/metadata/list、/save、/edit、/delete（复用通用入口）。

5.3.7 操作日志【部分实现】
记录“谁、在什么时间、对哪张表、做了什么、改前改后是什么”。

字段	说明
id	主键
table_name	操作的表
mode	操作模式：SAVE / UPDATE / DELETE / BATCH
operator	操作人
log_time	操作时间
pre_persist	持久化前数据（JSON）
post_persist	持久化后数据（JSON）
memo	备注
功能点：

列表查询：按时间范围、操作人、表名、操作模式过滤；
详情查看：对比变更前后 JSON；
导出：按查询条件导出 Excel/CSV（规划）；
保留策略：默认保留 180 天，超期归档（规划）。
现状：表、实体、Repository 已有；没有切面或服务在写日志，SysLogEntity.event 与表字段 mode 尚未对齐。

5.3.8 接口追踪【部分实现】
按周期统计每个接口的调用次数、总耗时、错误次数，用于发现慢接口和异常接口。

字段	说明
api_name	接口名
api_method	HTTP 方法
cycle	统计周期：DAY / WEEK / MONTH / SEASON / YEAR
cycle_zone	周期区间标识
total_time	总耗时（毫秒）
error_time	错误次数
is_open	是否开启追踪
功能点：

方法或字段标 @Trace(name="...", period="DAY") 即纳入统计；
按天聚合：select * from api_tracing where cycle='day' and cycle_zone>=? and cycle_zone<?（已配置在 custom-sql.yml）；
页面展示：接口排行、平均耗时、错误率趋势；
支持按接口单独开关。
现状：注解、实体、Repository、SQL 配置齐全；缺少采集切面、聚合定时任务和 api_tracing 建表语句。

5.3.9 文件管理【规划】
上传：限制扩展名与大小，服务端重命名，记录原始名、存储路径、上传人；
列表：按上传人、时间、类型检索；
下载/预览：图片可直接预览；
删除：逻辑删除，保留物理文件一段时间。
现状：无任何上传相关代码。

5.3.10 参数配置【规划】
键值对形式维护系统参数（分页大小、上传上限、日志保留天数等）；
支持按环境覆盖；
修改后即时生效或重启生效两种策略。
5.4 业务管理
业务库与系统库分离，实体统一继承 IdName（id、name）或 SortableIdName（追加 sortNo）。

5.4.1 基础资料
模块	关键字段	说明	状态
客户管理	id、name、sortNo	客户主数据	【部分实现】实体/Repository
供应商管理	id、name、sortNo	供应商主数据	【部分实现】实体/Repository
商品管理	id、name、sortNo	商品/物料主数据	【部分实现】实体/Repository
项目管理	id、name、sortNo、principal、setupDate、department、invoiceType	项目负责人、立项日期、部门、发票类型	【部分实现】实体/Repository
统一操作：列表、新增、编辑、删除、启用/停用、按名称模糊查询。 统一规则：id 由业务规则生成（如 CUS0001）；名称必填；被下游单据引用后不可删除。

5.4.2 资金管理【规划】
客户账户：客户、期初余额、应收、已收、期末余额、对账状态；
供应商账户：供应商、期初余额、应付、已付、期末余额；
账户流水：发生日期、单据号、方向（收/付）、金额、余额、备注。
现状：CustomerAccount、SupplierAccount 为空类，仅有包结构。

5.4.3 订单管理【规划】
销售订单：单号、客户、项目、下单日期、交货日期、状态（草稿/已确认/已发货/已完成/已取消）、金额合计；
订单明细：商品、数量、单价、金额、税率；
状态流转与库存、应收联动。
现状：SaleOrder 为空类。

5.4.4 库存管理【规划】
商品库存：商品、仓库、可用数量、锁定数量、预警下限；
出入库流水：单据类型、数量、方向、关联订单；
低于预警线在首页提示。
现状：MerchandiseStock 为空类。

5.5 通用 CRUD 能力【部分实现】
系统不为每张表单独写接口，而是用一套通用入口 + 元数据完成操作。

5.5.1 接口
方法	路径	说明
GET	/api/{module}/list	分页查询，请求体为 QueryEvent
POST	/api/{module}/save	新增
POST	/api/{module}/edit	修改
POST	/api/{module}/delete	删除
{module} 即 metadata.module，例如 customer。

5.5.2 请求模型
QueryEvent：

currentPage：当前页（从 0 开始）；
plength：每页条数；
orders：排序字段与方向；
extendMap：扩展查询条件；
pages：返回的分页结果。
CommandEvent：

operation：操作名，可填 Repository 方法名，实现“直接调数据层方法”。
DtoModel：event + module + operation + body，请求与响应共用。

5.5.3 调度规则
按 module 取元数据；
若 operation 命中 Repository 已声明方法，反射直接调用该方法；
否则按事件类型分派：查询 → AccessService，命令 → CommandService；
统一捕获数据层异常并转为业务异常。
5.5.4 查询与保存流程
代码块


查询：
前端 → GET /api/customer/list
     → 元数据定位 CustomerRepository
     → 组装分页/排序/条件
     → JPA 查询
     → 字典翻译（规划）
     → 返回 DtoModel.pages

保存：
前端 → POST /api/customer/save  { body: {...} }
     → 元数据定位 CustomerRepository
     → 校验（规划）
     → 写库
     → 记录操作日志（规划）
     → 返回保存后的对象
5.6 统一响应与异常【部分实现】
成功：

json


{ "success": true, "code": 0, "body": { } }
失败：

json


{ "success": false, "code": 400, "body": "参数不合法" }
场景	HTTP/业务码	处理
参数校验失败	400	RestException + 国际化消息
未认证	401	AuthenticationException
无权限	403	规划
资源不存在	404	规划
服务端异常	500	兜底 Exception
国际化：异常消息通过 MessageSource 解析，按请求 Locale 返回中/英文。

6. 接口规范
6.1 通用约定
统一前缀：/admin/{service}，如 /admin/system、/admin/module；
请求/响应：application/json; charset=UTF-8；
时间格式：yyyy-MM-dd HH:mm:ss，时区 Asia/Shanghai；
分页参数：currentPage、plength；
排序参数：字段 + ASC/DESC；
响应包装：RestMessage（通用 CRUD 目前直接返回 DtoModel，规划统一）。
6.2 系统管理接口（规划）
模块	接口
用户	/api/user/list、/save、/edit、/delete、/reset-password
角色	/api/role/list、/save、/edit、/delete、/grant
菜单	/api/menu/tree、/save、/edit、/delete
字典	/api/ddic/category/list、/api/ddic/item/list、/save、/edit、/delete
元数据	/api/metadata/list、/save、/edit、/delete
日志	/api/log/list、/api/log/{id}、/api/log/export
追踪	/api/trace/list、/api/trace/stat
文件	/api/file/upload、/api/file/{id}、/api/file/delete
6.3 业务接口
业务模块复用通用入口：/api/{module}/list|save|edit|delete， module 取值为 customer、supplier、merchandise、project 等。

7. 数据模型
7.1 系统库（admin-system）
表	主键	说明
metadata	module	模块注册，驱动通用 CRUD
ddic_category	id	字典分类
ddic_item	id	字典明细，category_id 关联分类
sys_log	id	操作日志
api_tracing	id	接口追踪（建表语句待补）
sys_user / sys_role / sys_menu 等	id	权限体系（规划）
关系：

ddic_category 1 ── N ddic_item（category_id）；
metadata 1 ── 1 实体/Repository（按 table_name 约定推导）；
sys_log 通过 table_name 多态关联业务表，不与具体表建外键。
7.2 业务库（admin-module）
表	说明
customer	客户（id、name、sortNo）
supplier	供应商（id、name、sortNo）
merchandise	商品（id、name、sortNo）
project	项目（+ 负责人、立项日期、部门、发票类型）
customer_account / supplier_account	往来账户（规划）
sale_order / sale_order_item	销售订单（规划）
merchandise_stock	库存（规划）
8. 非功能设计
维度	要求
性能	列表分页；元数据与字典走缓存；常用查询字段建索引；单表百万级分页响应 < 1s
安全	密码加盐哈希；会话超时；接口鉴权；上传白名单；防 SQL 注入（JPA 参数绑定）；操作留痕
可用性	统一异常兜底；数据库连接池；关键接口追踪耗时
兼容性	Chrome / Edge 最近两个大版本；分辨率 1366×768 起
国际化	异常消息中英双语，按 Locale 返回
可维护性	模块纵向分层；通用能力集中在 common；配置外置
部署	三个可执行 jar + MySQL；Redis 可选
9. 附录 A：落地情况对照表
功能	状态	现有实现
登录 / 登出	【规划】	仅 AuthenticationException → 401
用户 / 角色 / 权限 / 菜单	【规划】	无实体、无表、无接口
数据字典维护	【部分实现】	ddic_category、ddic_item、实体、Repository
字典自动翻译	【部分实现】	@Ddic、ClazzUtil.findDdicProperty、Metadata.ddicInfo，回填未启用
元数据注册与装载	【部分实现】	MetadataServiceImpl 完成实体/Repository/属性解析与缓存
通用列表 / 增删改	【部分实现】	Action + ServiceImpl 路由；AccessService/CommandService 返回 null
操作日志	【部分实现】	表、实体、Repository；无写入切面
接口追踪	【部分实现】	@Trace、实体、Repository、聚合 SQL；无采集与建表
文件上传	【规划】	无
参数配置	【规划】	无
客户 / 供应商 / 商品 / 项目	【部分实现】	4 个实体 + 4 个 Repository；3 个 Repository 泛型写错
资金 / 订单 / 库存	【规划】	空实体类
前端界面	【部分实现】	Vue 2 教程示例页，无路由、无管理页面
统一异常	【部分实现】	RestExceptionHandler，缺 messages.properties
Feign 门面	【部分实现】	客户端接口为空，未启用 @EnableFeignClients
10. 附录 B：建议实施优先级
P0（可演示的最小闭环）

补 metadata 初始化数据与建表脚本；
实现 AccessService.query 与 CommandService.command；
CommandEvent 增加 body，Controller 补 operation/body；
修正三个业务 Repository 的实体泛型；
前端接入登录页 + 客户列表页。
P1（系统管理可用）

用户 / 角色 / 菜单 / 权限与 Shiro 集成；
字典维护接口与列表翻译；
日志切面与日志查询页；
统一响应包装与错误码。
P2（业务与运维）

资金、订单、库存模型与单据流转；
接口追踪采集与统计页；
文件上传、参数配置、导出；
缓存切换 Redis、会话共享。
  
