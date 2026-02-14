# Java 后端开发规范（完整版）

> 参考来源：[SmartAdmin 官方文档](https://smartadmin.vip/views/doc/standard/back.html)

---

## 一、Java 项目规范

### 1.1 项目命名规范

全部采用小写方式，以中划线分隔。

- 正例：`mall-management-system` / `order-service-client` / `user-api`
- 反例：`mall_management-system` / `mallManagementSystem`

### 1.2 方法参数规范

每个方法最多 **5 个参数**。超过 5 个需封装成 JavaBean 对象。

原因：
- 便于调用
- 降低出错率
- 保持代码整洁

### 1.3 代码目录结构

```
src/
├── common/          通用类库
├── config/          项目配置
├── constant/        全局常量
├── handler/         全局处理器
├── interceptor/     全局拦截器
├── listener/        全局监听器
├── module/          各业务模块
├── third/           三方服务
├── util/            工具类
└── Application.java 启动类
```

### 1.4 common 目录规范

```
common/
├── anno/            注解
├── constant/        常量
├── domain/          JavaBean
├── exception/       异常
├── json/            JSON 类库
├── swagger/         Swagger 配置
└── validator/       校验器
```

### 1.5 module 目录规范

每个业务独立文件夹，内部进行 MVC 划分，`domain` 包存放 `entity`、`dto`、`vo`、`bo` 等对象。

```
module/
├── order/
│   ├── controller/
│   ├── service/
│   ├── manager/
│   ├── dao/
│   └── domain/
│       ├── entity/
│       ├── dto/
│       ├── vo/
│       ├── bo/
│       └── form/
├── employee/
│   ├── controller/
│   ├── service/
│   ├── ...
```

---

## 二、MVC 规范

### 2.1 整体分层

```
Controller 层  →  Service 层  →  Manager 层  →  DAO 层
```

### 2.2 Controller 层规范

#### 2.2.1 RequestMapping 注解位置

**只允许在 method 上添加 `@RequestMapping` 注解，不允许加在 class 上。**

正例：

```java
@RestController
public class DepartmentController {

    @GetMapping("/department/list")
    public ResponseDTO<List<DepartmentVO>> listDepartment() {
        return departmentService.listDepartment();
    }
}
```

反例：

```java
@RestController
@RequestMapping("/department")  // 不允许
public class DepartmentController {

    @GetMapping("/list")
    public ResponseDTO<List<DepartmentVO>> listDepartment() {
        return departmentService.listDepartment();
    }
}
```

#### 2.2.2 URL 命名规范

不推荐 RESTful 风格，仅使用 GET / POST 方法。URL 格式为：`/业务模块/子模块/动作`

正例：

```
GET  /department/get/{id}       获取某个部门
POST /department/query          复杂查询
POST /department/add            添加部门
POST /department/update         更新部门
GET  /department/delete/{id}    删除部门
```

#### 2.2.3 Swagger 注解要求

每个方法必须添加 `@ApiOperation`，描述最后必须加接口作者信息。

格式：`@author 作者名`

```java
@ApiOperation("更新部门信息 @author 卓大")
@PostMapping("/department/update")
public ResponseDTO<String> updateDepartment(@Valid @RequestBody DepartmentUpdateForm form) {
    return departmentService.updateDepartment(form);
}
```

#### 2.2.4 Controller 方法职责

Controller 方法中 **不允许** 做以下操作：

- 不做业务逻辑操作
- 不做参数、业务校验（仅允许 `@Valid` 简单校验）
- 不做数据组合、拼装、赋值

#### 2.2.5 获取请求用户

**只能在 Controller 层获取当前请求用户**，并传递给 Service 层。

```java
@PostMapping("/employee/add")
public ResponseDTO<String> addEmployee(@Valid @RequestBody EmployeeAddForm form) {
    RequestUser requestUser = SmartRequestUtil.getRequestUser();
    return employeeService.addEmployee(form, requestUser);
}
```

### 2.3 Service 层规范

#### 2.3.1 业务拆分

不建议 Service 文件行数过大，应按功能拆分：

```
OrderQueryService     订单查询
OrderCreateService    订单创建
OrderUpdateService    订单修改
OrderDeleteService    订单删除
```

#### 2.3.2 @Transactional 注解使用

**谨慎使用 `@Transactional` 注解**，合并数据库操作，减少注解方法内业务逻辑。`rollbackFor` 值必须使用 `Exception.class`。

反例（验证逻辑混在 `@Transactional` 方法中）：

```java
@Transactional(rollbackFor = Exception.class)
public ResponseDTO<String> upOrDown(Long departmentId, Long swapId) {
    // 验证逻辑 —— 不应该放在这里
    DepartmentEntity dept = departmentDao.selectById(departmentId);
    if (dept == null) {
        return ResponseDTO.wrap(ResponseCodeConst.ERROR_PARAM);
    }
    // ... 更多验证步骤 ...

    // 数据库操作
    departmentDao.updateById(dept);
}
```

正例（将验证逻辑和事务操作分离）：

```java
// ========== Service 层 ==========
public ResponseDTO<String> upOrDown(Long deptId, Long swapId) {
    // 验证逻辑放在 Service 层
    DepartmentEntity dept = departmentDao.selectById(deptId);
    if (dept == null) {
        return ResponseDTO.wrap(ResponseCodeConst.ERROR_PARAM);
    }
    DepartmentEntity swapEntity = departmentDao.selectById(swapId);
    if (swapEntity == null) {
        return ResponseDTO.wrap(ResponseCodeConst.ERROR_PARAM);
    }

    // 将准备好的数据传给 Manager 层处理
    departmentManager.upOrDown(dept, swapEntity);
    return ResponseDTO.succ();
}

// ========== Manager 层 ==========
@Transactional(rollbackFor = Throwable.class)
public void upOrDown(DepartmentEntity dept, DepartmentEntity swap) {
    Long sort = dept.getSort();
    dept.setSort(swap.getSort());
    departmentDao.updateById(dept);
    swap.setSort(sort);
    departmentDao.updateById(swap);
}
```

#### 2.3.3 内部方法调用事务失效

**`@Transactional` 事务在类内部方法调用时不会生效。**

反例：

```java
@Service
public class OrderService {

    public void createOrder(OrderAddForm form) {
        this.saveData(form);  // 内部调用，事务不生效！
    }

    @Transactional(rollbackFor = Exception.class)
    public void saveData(OrderAddForm form) {
        orderDao.insert(form);
    }
}
```

解决方案：
- 方法放入 **Manager 层**，通过注入调用
- 启动类添加 `@EnableAspectJAutoProxy(exposeProxy = true)`

### 2.4 Manager 层规范

> 来源：https://smartadmin.vip/views/doc/back/ManagerLayer.html

#### 2.4.1 为什么需要 Manager 层

传统 SpringMVC 三层架构（Controller → Service → DAO）存在以下问题：

1. **Service 层代码臃肿** — 业务逻辑、事务、数据组装全部堆积
2. **Service 层事务嵌套** — 导致事务传播问题众多
3. **DAO 层混杂业务逻辑** — 职责不清
4. **DAO 层 SQL 复杂** — 关联查询过多，难以维护
5. **DAO 层频繁修改** — 牵一发而动全身

#### 2.4.2 Manager 层定义（引用《阿里规约》）

Manager 层具备三个核心特征：

- **第三方平台封装** — 预处理返回结果及转化异常信息（如短信、支付、OSS 等）
- **Service 通用能力下沉** — 如缓存方案、中间件通用处理
- **与 DAO 层交互** — 对多个 DAO 进行组合复用

#### 2.4.3 SmartAdmin 扩展规则

在《阿里规约》基础上追加以下规则：

| 规则 | 说明 |
|------|------|
| 事务下沉 | 复杂业务中，Service 提供数据给 Manager，**事务下沉到 Manager 层** |
| 禁止互相调用 | **Manager 层之间禁止相互调用**，避免事务嵌套 |
| 专注 SQL | Manager 专注于不带业务的 SQL 操作，通用 DAO 层封装 |
| 避免复杂 JOIN | 避免在 DAO 层写复杂 JOIN 查询，**在 Manager 层拆分为多次简单查询** |
| 可用 BaseService | Manager 层可使用 MyBatis-Plus 的 `ServiceImpl`（`BaseService`） |

#### 2.4.4 Manager 层代码示例

**示例 1：角色菜单管理 — RoleMenuManager**

```java
@Service
public class RoleMenuManager extends ServiceImpl<RoleMenuDao, RoleMenuEntity> {

    @Resource
    private RoleMenuDao roleMenuDao;

    /**
     * 更新角色菜单：先删除原有关联，再批量插入新关联
     * 事务保证删除和插入的原子性
     */
    @Transactional(rollbackFor = Exception.class)
    public void updateRoleMenu(Long roleId, List<RoleMenuEntity> roleMenuEntityList) {
        // 删除原有角色菜单关联
        roleMenuDao.deleteByRoleId(roleId);
        // 批量插入新的角色菜单关联（使用 MyBatis-Plus 的 saveBatch）
        saveBatch(roleMenuEntityList);
    }
}
```

**示例 2：Service 层与 Manager 层协作**

```java
// ========== Service 层：负责业务逻辑和数据验证 ==========
@Service
public class RoleMenuService {

    @Resource
    private RoleMenuManager roleMenuManager;
    @Resource
    private RoleDao roleDao;

    public ResponseDTO<String> updateRoleMenu(RoleMenuUpdateForm form) {
        // 1. 业务验证（不在事务内）
        RoleEntity role = roleDao.selectById(form.getRoleId());
        if (role == null) {
            return ResponseDTO.error("角色不存在");
        }

        // 2. 数据组装（不在事务内）
        List<RoleMenuEntity> menuList = form.getMenuIdList().stream()
            .map(menuId -> {
                RoleMenuEntity entity = new RoleMenuEntity();
                entity.setRoleId(form.getRoleId());
                entity.setMenuId(menuId);
                return entity;
            })
            .collect(Collectors.toList());

        // 3. 调用 Manager 层执行事务操作
        roleMenuManager.updateRoleMenu(form.getRoleId(), menuList);
        return ResponseDTO.succ();
    }
}
```

#### 2.4.5 Manager 层核心原则总结

```
┌─────────────────────────────────────────────────────────┐
│                    Manager 层核心原则                      │
├─────────────────────────────────────────────────────────┤
│  1. 事务归 Manager    — @Transactional 放在 Manager 层    │
│  2. 验证归 Service    — 业务校验在 Service 层完成          │
│  3. Manager 不互调    — 避免事务嵌套                      │
│  4. 拆分复杂 SQL      — 不写复杂 JOIN，拆为多次简单查询    │
│  5. 可继承 ServiceImpl — 使用 MyBatis-Plus 的批量操作能力  │
│  6. 通用能力封装      — 缓存、中间件、三方服务统一封装      │
└─────────────────────────────────────────────────────────┘
```

### 2.5 DAO 层规范

#### 2.5.1 框架选择

优先使用 **MyBatis-Plus**，多数据源可用 SmartDb 框架。

#### 2.5.2 MyBatis-Plus 要求

- 所有 Dao 继承 `BaseMapper`
- **禁止使用 MyBatis-Plus 的 Wrapper 条件构建器**

禁用 Wrapper 的原因：
- SQL 形式不一，无法复用
- 难以定位和排查慢 SQL
- 增加代码维护成本

#### 2.5.3 禁止在 XML 中写死常量

应从 Dao 方法中传入。

正例：

```java
// NoticeDao.java
public interface NoticeDao extends BaseMapper<NoticeEntity> {
    Integer noticeCount(@Param("sendStatus") Integer sendStatus);
}
```

```xml
<!-- NoticeMapper.xml -->
<select id="noticeCount" resultType="integer">
    SELECT COUNT(1) FROM t_notice
    WHERE send_status = #{sendStatus}
</select>
```

反例：

```xml
<!-- 写死常量值，不允许 -->
<select id="noticeCount" resultType="integer">
    SELECT COUNT(1) FROM t_notice
    WHERE send_status = 0
</select>
```

#### 2.5.4 JOIN 写法

**建议在 XML 中 JOIN 关联使用表名全称，而不是别名。**

正例：

```xml
<select id="queryPage" resultType="...">
    SELECT t_notice.*, t_employee.actual_name AS createUserName
    FROM t_notice
    LEFT JOIN t_employee ON t_notice.create_user_id = t_employee.employee_id
    WHERE t_notice.deleted_flag = #{queryForm.deletedFlag}
</select>
```

反例：

```xml
<!-- 使用别名，不推荐 -->
<select id="queryPage" resultType="...">
    SELECT a.*, b.actual_name AS createUserName
    FROM t_notice a
    LEFT JOIN t_employee b ON a.create_user_id = b.employee_id
    WHERE a.deleted_flag = #{queryForm.deletedFlag}
</select>
```

---

## 三、JavaBean 规范

### 3.1 整体要求

- 无业务逻辑或计算
- 基本数据类型使用**包装类型**（`Integer`、`Double`、`Boolean`）
- 无默认值
- 每个属性添加多行注释
- 使用 **Lombok** 简化 getter/setter
- 建议使用 `@Builder`、`@NoArgsConstructor`

### 3.2 命名划分

| 类型 | 命名格式 | 用途 |
|------|----------|------|
| `XxxEntity` | 数据库持久对象 | 字段与数据库一致，日期用 `LocalDateTime` |
| `XxxForm` | 前端请求对象 | 不继承 Entity，用于接收前端/RPC 参数 |
| `XxxVO` | 返回前端对象 | 不继承 Entity，用于返回数据封装 |
| `XxxDTO` | 数据传输对象 | 跨层数据传输 |
| `XxxBO` | 内部处理对象 | 仅用于 service/manager/dao 层，不用于 controller 层 |

### 3.3 Entity 要求

- 以 `Entity` 结尾
- 字段与数据库字段一致
- 每个字段添加注释，与数据库注释一致
- 不允许组合其他对象
- 日期类型使用 `LocalDateTime` 或 `LocalDate`

### 3.4 Form 要求

- 不继承 `Entity`
- 可继承、组合 `DTO`、`VO`、`BO`
- 仅用于前端、RPC 请求参数

### 3.5 VO 要求

- 不继承 `Entity`
- 可继承、组合 `DTO`、`VO`、`BO`
- 仅用于返回前端、RPC 数据封装

### 3.6 BO 要求

- 不继承 `Entity`
- 仅用于 service、manager、dao 层
- 不允许用于 controller 层

### 3.7 Boolean 属性命名规范

**类中布尔类型变量都不要加 `is`，否则框架解析会引起序列化错误。**

统一使用 `flag` 结尾：

| Java 属性 | 数据库字段 |
|-----------|-----------|
| `deletedFlag` | `deleted_flag` |
| `onlineFlag` | `online_flag` |
| `disabledFlag` | `disabled_flag` |

---

## 四、数据库规范

### 4.1 数据库命名

全部小写，下划线分隔，区分环境。

正例：`smart_admin_v2_dev` / `smart_admin_v2_prod` / `smart_admin_v2_test`

### 4.2 表命名

全部小写，下划线分隔，以 `t_` 开头。

正例：`t_employee`、`t_department`、`t_config`

### 4.3 建表规范

表必备三字段（简单表除外）：

| 字段 | 类型 | 说明 |
|------|------|------|
| `[module]_id` | `BIGINT` | Long 类型，单表自增，主键 |
| `create_time` | `DATETIME` | 默认 `CURRENT_TIMESTAMP` |
| `update_time` | `DATETIME` | 默认 `CURRENT_TIMESTAMP`，`ON UPDATE CURRENT_TIMESTAMP` |

枚举类字段注释**必须注明所有枚举含义**。

建表示例：

```sql
CREATE TABLE `t_change_data` (
    `change_data_id` BIGINT(20) UNSIGNED NOT NULL AUTO_INCREMENT COMMENT '主键ID',
    `sync_status` TINYINT(3) UNSIGNED NOT NULL DEFAULT '0' COMMENT '同步状态 0未开始 1同步中 2同步成功 3失败',
    `sync_time` DATETIME NULL DEFAULT NULL COMMENT '同步时间',
    `create_time` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
    `update_time` DATETIME NULL DEFAULT NULL ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
    PRIMARY KEY (`change_data_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='数据变更表';
```

### 4.4 索引规范

参照《阿里巴巴 Java 开发手册》的索引规约。

---

## 五、规范速查表

| 分类 | 规则 | 重要程度 |
|------|------|----------|
| 项目命名 | 小写 + 中划线 | 必须 |
| 方法参数 | 最多 5 个，超出封装 JavaBean | 必须 |
| Controller | `@RequestMapping` 只加方法上 | 必须 |
| Controller | 不做业务逻辑，只做转发 | 必须 |
| Controller | 只在此层获取当前用户 | 必须 |
| URL | `/模块/子模块/动作`，仅 GET/POST | 必须 |
| Swagger | `@ApiOperation` + `@author` | 必须 |
| Service | 按功能拆分，避免单文件过大 | 建议 |
| Transactional | 谨慎使用，验证逻辑分离，rollbackFor=Exception.class | 必须 |
| Manager | 承载事务、三方封装、通用能力下沉 | 必须 |
| DAO | 禁止 Wrapper，禁止 XML 写死常量 | 必须 |
| DAO | JOIN 用表名全称 | 建议 |
| JavaBean | Entity/Form/VO/DTO/BO 命名规范 | 必须 |
| Boolean | 不加 `is`，用 `flag` 结尾 | 必须 |
| 数据库 | 表名 `t_` 开头，必备三字段 | 必须 |
| 数据库 | 枚举字段注释标明所有含义 | 必须 |
