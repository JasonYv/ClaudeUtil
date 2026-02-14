# Java 后端开发规范（SmartAdmin V3.0）

编写 Java 后端代码时，必须严格遵守以下规范。所有规则默认为"必须"级别，标注"建议"的为推荐遵守。

---

## 一、项目结构

### 命名
- 项目名：全小写 + 中划线分隔（如 `order-service-client`），禁止下划线或驼峰

### 目录结构
```
src/
├── common/            通用类库（anno/constant/domain/exception/json/swagger/validator）
├── config/            项目配置
├── constant/          全局常量
├── handler/           全局处理器
├── interceptor/       全局拦截器
├── listener/          全局监听器
├── module/            各业务模块（按模块独立文件夹，内部 MVC 划分）
│   └── [模块名]/
│       ├── controller/
│       ├── service/
│       ├── manager/
│       ├── dao/
│       └── domain/    (entity/ dto/ vo/ bo/ form/)
├── third/             三方服务封装
├── util/              工具类
└── Application.java
```

### 方法参数
- 每个方法最多 5 个参数，超过必须封装为 JavaBean

---

## 二、MVC 四层架构

分层顺序：`Controller → Service → Manager → DAO`

### Controller 层

1. **`@RequestMapping` 只能加在方法上**，禁止加在类上
2. **URL 格式**：`/业务模块/子模块/动作`，仅用 GET 和 POST，不用 RESTful 风格
   ```
   GET  /department/get/{id}
   POST /department/query
   POST /department/add
   POST /department/update
   GET  /department/delete/{id}
   ```
3. **每个方法必须加** `@ApiOperation("描述 @author 作者名")`
4. **Controller 只做转发**，禁止：业务逻辑、业务校验（仅允许 `@Valid`）、数据组装拼接
5. **获取当前用户只能在 Controller 层**，通过 `SmartRequestUtil.getRequestUser()` 获取后传给 Service

```java
@RestController
public class DepartmentController {

    @ApiOperation("更新部门信息 @author 卓大")
    @PostMapping("/department/update")
    public ResponseDTO<String> updateDepartment(@Valid @RequestBody DepartmentUpdateForm form) {
        return departmentService.updateDepartment(form);
    }
}
```

### Service 层

1. **按功能拆分**（建议）：避免单个 Service 过大，按 `XxxQueryService`、`XxxCreateService` 等拆分
2. **`@Transactional` 谨慎使用**：
   - 验证逻辑放在 Service 层，不要放在 `@Transactional` 方法内
   - 纯数据库操作下沉到 Manager 层，由 Manager 承载 `@Transactional`
   - `rollbackFor` 必须指定为 `Exception.class`
3. **类内部方法调用 `@Transactional` 不生效**，必须通过注入 Manager 层调用

```java
// Service：验证 + 数据准备
public ResponseDTO<String> upOrDown(Long deptId, Long swapId) {
    DepartmentEntity dept = departmentDao.selectById(deptId);
    if (dept == null) {
        return ResponseDTO.wrap(ResponseCodeConst.ERROR_PARAM);
    }
    DepartmentEntity swapEntity = departmentDao.selectById(swapId);
    if (swapEntity == null) {
        return ResponseDTO.wrap(ResponseCodeConst.ERROR_PARAM);
    }
    departmentManager.upOrDown(dept, swapEntity);
    return ResponseDTO.succ();
}
```

### Manager 层

Manager 层是 Service 与 DAO 之间的中间层，核心规则：

1. **事务归 Manager** — `@Transactional` 注解放在 Manager 层方法上
2. **Manager 之间禁止互相调用** — 避免事务嵌套
3. **拆分复杂 SQL** — 不写复杂 JOIN，在 Manager 层拆分为多次简单查询
4. **可继承 `ServiceImpl`** — 使用 MyBatis-Plus 的 `saveBatch` 等批量能力
5. **封装三方服务** — 短信、支付、OSS 等第三方平台的封装层
6. **通用能力下沉** — 缓存方案、中间件处理等通用逻辑

```java
@Service
public class RoleMenuManager extends ServiceImpl<RoleMenuDao, RoleMenuEntity> {

    @Resource
    private RoleMenuDao roleMenuDao;

    @Transactional(rollbackFor = Exception.class)
    public void updateRoleMenu(Long roleId, List<RoleMenuEntity> roleMenuEntityList) {
        roleMenuDao.deleteByRoleId(roleId);
        saveBatch(roleMenuEntityList);
    }
}
```

### DAO 层

1. 优先使用 **MyBatis-Plus**，所有 Dao 继承 `BaseMapper`
2. **禁止使用 Wrapper 条件构建器**（SQL 无法复用、难以排查慢 SQL）
3. **禁止在 XML 中写死常量**，必须从 Dao 方法传参
4. **JOIN 使用表名全称**（建议），不用别名 `a`、`b`

```xml
<!-- 正确：使用表名全称 -->
<select id="queryPage" resultType="...">
    SELECT t_notice.*, t_employee.actual_name AS createUserName
    FROM t_notice
    LEFT JOIN t_employee ON t_notice.create_user_id = t_employee.employee_id
    WHERE t_notice.deleted_flag = #{queryForm.deletedFlag}
</select>
```

---

## 三、JavaBean 规范

### 通用要求
- 无业务逻辑，无默认值
- 基本类型使用包装类型（`Integer`、`Double`、`Boolean`）
- 使用 Lombok（`@Data`、`@Builder`、`@NoArgsConstructor`）
- 每个属性添加注释

### 命名分类

| 后缀 | 用途 | 禁止事项 |
|------|------|----------|
| `XxxEntity` | 数据库持久对象，字段与数据库一致 | 不允许组合其他对象 |
| `XxxForm` | 前端/RPC 请求参数 | 不继承 Entity |
| `XxxVO` | 返回前端/RPC 数据封装 | 不继承 Entity |
| `XxxDTO` | 跨层数据传输 | - |
| `XxxBO` | service/manager/dao 内部对象 | 不用于 Controller 层 |

### 特殊规则
- Entity 日期类型使用 `LocalDateTime` 或 `LocalDate`，不用 `Date`
- Form 和 VO 可继承/组合 DTO、VO、BO，但禁止继承 Entity
- **布尔属性不加 `is` 前缀**，统一用 `flag` 结尾：`deletedFlag`、`onlineFlag`、`disabledFlag`

---

## 四、数据库规范

### 命名
- 数据库名：全小写 + 下划线 + 区分环境（如 `smart_admin_v2_dev`）
- 表名：全小写 + 下划线 + `t_` 前缀（如 `t_employee`、`t_department`）

### 建表必备字段
每张表必须包含以下三个字段（简单关联表除外）：

| 字段 | 类型 | 约束 |
|------|------|------|
| `[模块]_id` | `BIGINT UNSIGNED` | 主键，自增 |
| `create_time` | `DATETIME` | `NOT NULL DEFAULT CURRENT_TIMESTAMP` |
| `update_time` | `DATETIME` | `DEFAULT NULL ON UPDATE CURRENT_TIMESTAMP` |

### 其他要求
- 枚举字段注释必须列明所有枚举值含义（如 `COMMENT '状态 0未开始 1进行中 2完成 3失败'`）
- 索引规范参照《阿里巴巴 Java 开发手册》

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
