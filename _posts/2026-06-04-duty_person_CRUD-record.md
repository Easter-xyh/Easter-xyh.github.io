---
layout: post
title: "值班人信息模块，表格创建和CRUD"
date: 2026-06-04
categories: 开发记录 Java后端 CRUD
---

## 背景

在一个 Spring Boot 后端服务中，需要新增一套“值班人员信息”管理能力，支持新增、修改、删除、详情查询和分页查询。该功能本身是典型 CRUD，但开发过程中同时遇到了一个启动阶段的 Spring Bean 冲突问题，因此本次改动主要包含两部分：

1. 解决启动时工具类 Bean 重复注册导致的依赖注入冲突。
2. 基于 MyBatis-Plus 新增一套完整的值班人员信息 CRUD 链路。

本文按排查过程和分层实现展开。

## 1. 解决启动时的 Bean 冲突

### 问题现象

服务启动时，Spring 在按类型注入某个用户工具类时发现了多个候选 Bean，导致应用启动失败。

抽象后的冲突结构如下：

```java
@Bean
public UserUtils userUtils() {
    return new UserUtils();
}
```

同时，项目中还存在另一个继承自同类工具能力的组件：

```java
@Component
public class UserUtil extends UserUtils {
    // 更完整的用户上下文能力
}
```

当 Spring 按 `UserUtils` 类型进行依赖注入时，容器中同时存在两个可匹配的 Bean：

- 配置类中手动注册的 `UserUtils` Bean；
- 通过 `@Component` 自动扫描注册的 `UserUtil` Bean。

这会造成候选 Bean 不唯一，从而触发启动异常。

### 处理方式

删除配置类中重复注册的工具类 Bean，统一使用项目中已有、功能更完整的组件 Bean。

保留 MyBatis-Plus 相关配置，例如自动填充处理器：

```java
@Bean
public MybatisPlusMetaObjectHandler mybatisPlusMetaObjectHandler() {
    return new MybatisPlusMetaObjectHandler();
}
```

### 小结

这个问题的关键不在于工具类本身，而在于 Spring 容器中出现了多个同类型候选 Bean。处理类似问题时，可以优先检查：

- 是否既使用了 `@Bean` 又使用了 `@Component` 注册同类对象；
- 是否存在父类、子类同时进入容器，且注入点使用父类类型；
- 是否应该通过 `@Primary`、`@Qualifier` 或删除重复 Bean 来明确注入目标。

在本场景下，删除重复 Bean 是最干净的方案。

## 2. 数据库实体设计

新增值班人员信息实体，对应一张独立业务表。

```java
@Data
@TableName("duty_person")
public class DutyPerson extends BaseModel implements Serializable {

    @TableId(type = IdType.AUTO)
    private Long id;

    private String dutyPerson;
    private String department;
    private String phone;
}
```

字段说明：

| 字段 | 含义 |
|---|---|
| `id` | 主键 |
| `dutyPerson` | 值班人员姓名或展示名 |
| `department` | 所属部门文本 |
| `phone` | 联系电话 |

实体继承项目已有的基础模型，用于复用创建时间、修改时间、创建人、修改人、逻辑删除等公共字段。

## 3. 请求参数设计

为了避免 Controller 直接接收实体对象，新增独立的请求参数类，将新增、修改、分页查询三个场景拆开。

### 新增请求

```java
public class DutyPersonInsertForm {

    @NotBlank
    @Size(max = 64)
    private String dutyPerson;

    @NotBlank
    @Size(max = 128)
    private String department;

    @NotBlank
    @Size(max = 32)
    private String phone;
}
```

### 修改请求

```java
public class DutyPersonUpdateForm {

    @NotNull
    private Long id;

    @NotBlank
    @Size(max = 64)
    private String dutyPerson;

    @NotBlank
    @Size(max = 128)
    private String department;

    @NotBlank
    @Size(max = 32)
    private String phone;
}
```

### 分页查询请求

```java
public class DutyPersonPageForm extends BasePageQuery {
    private String dutyPerson;
    private String department;
    private String phone;
}
```

分页查询支持按值班人员、部门和联系电话进行模糊筛选。

## 4. 响应对象设计

新增接口响应对象，避免直接暴露数据库实体。

```java
public class DutyPersonResponse {
    private Long id;
    private String dutyPerson;
    private String department;
    private String phone;
    private Date createTime;
    private Date updateTime;
}
```

时间字段统一格式化为：

```text
yyyy-MM-dd HH:mm:ss
```

## 5. Mapper 层实现

Mapper 继承 MyBatis-Plus 的 `BaseMapper`，复用基础增删改查能力，同时声明一个自定义分页查询方法。

```java
@Mapper
public interface DutyPersonMapper extends BaseMapper<DutyPerson> {

    IPage<DutyPersonResponse> selectDutyPersonPage(
            @Param("page") Page<DutyPersonResponse> page,
            @Param("form") DutyPersonPageForm form);
}
```

对应 SQL 的核心结构如下：

```sql
SELECT id,
       duty_person AS dutyPerson,
       department,
       phone,
       create_time AS createTime,
       update_time AS updateTime
FROM duty_person
WHERE del = 0
ORDER BY id DESC;
```

动态查询条件示例：

```sql
duty_person LIKE CONCAT('%', #{form.dutyPerson}, '%')
department LIKE CONCAT('%', #{form.department}, '%')
phone LIKE CONCAT('%', #{form.phone}, '%')
```

这里需要注意两点：

1. 查询时必须带上逻辑删除条件，例如 `del = 0`。
2. 模糊查询字段应做好空值判断，避免生成无意义 SQL 条件。

## 6. Service 层设计

Service 接口定义标准 CRUD 方法：

```java
public interface DutyPersonService {

    BaseResponse<Long> insert(DutyPersonInsertForm form);

    BaseResponse<Boolean> update(DutyPersonUpdateForm form);

    BaseResponse<Boolean> delete(DeleteForm form);

    BaseResponse<DutyPersonResponse> detail(DetailForm form);

    BaseResponse<IPage<DutyPersonResponse>> page(DutyPersonPageForm form);
}
```

实现类继承 MyBatis-Plus 的 `ServiceImpl`：

```java
@Service
public class DutyPersonServiceImpl
        extends ServiceImpl<DutyPersonMapper, DutyPerson>
        implements DutyPersonService {

    // insert / update / delete / detail / page
}
```

主要逻辑：

- `insert`：复制请求参数到实体对象，保存后返回新增记录 ID；
- `update`：根据 ID 更新值班人员、部门和联系电话；
- `delete`：按 ID 批量逻辑删除；
- `detail`：根据 ID 查询单条记录；
- `page`：分页查询，并支持多个字段模糊筛选。

## 7. Controller 层接口

Controller 提供统一的 REST 风格入口。

```java
@RestController
@RequestMapping("/api/duty/person")
public class DutyPersonController {

    @PostMapping("/insert")
    public BaseResponse<Long> insert(@Valid @RequestBody DutyPersonInsertForm form) {
        return dutyPersonService.insert(form);
    }

    @PostMapping("/update")
    public BaseResponse<Boolean> update(@Valid @RequestBody DutyPersonUpdateForm form) {
        return dutyPersonService.update(form);
    }

    @PostMapping("/delete")
    public BaseResponse<Boolean> delete(@Valid @RequestBody DeleteForm form) {
        return dutyPersonService.delete(form);
    }

    @PostMapping("/detail")
    public BaseResponse<DutyPersonResponse> detail(@Valid @RequestBody DetailForm form) {
        return dutyPersonService.detail(form);
    }

    @PostMapping("/page")
    public BaseResponse<IPage<DutyPersonResponse>> page(@RequestBody DutyPersonPageForm form) {
        return dutyPersonService.page(form);
    }
}
```

接口列表：

| 方法 | 路径 | 说明 |
|---|---|---|
| `POST` | `/api/duty/person/insert` | 新增值班人员 |
| `POST` | `/api/duty/person/update` | 修改值班人员 |
| `POST` | `/api/duty/person/delete` | 删除值班人员 |
| `POST` | `/api/duty/person/detail` | 查询详情 |
| `POST` | `/api/duty/person/page` | 分页查询 |

在内部项目中，如果存在权限系统，可以根据实际情况为这些接口补充权限注解。若权限数据需要额外初始化，应避免在未配置权限数据的情况下贸然加入强权限校验，以免影响现有功能。

## 8. 数据库脚本

建表示例：

```sql
CREATE TABLE IF NOT EXISTS `duty_person`
(
    `id`           bigint       NOT NULL AUTO_INCREMENT COMMENT 'Primary key',
    `duty_person`  varchar(64)  NOT NULL COMMENT 'Duty person',
    `department`   varchar(128) NOT NULL COMMENT 'Department',
    `phone`        varchar(32)  NOT NULL COMMENT 'Contact phone',
    `create_time`  datetime     NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT 'Create time',
    `create_by`    bigint       NULL COMMENT 'Created by',
    `update_time`  datetime     NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT 'Update time',
    `update_by`    bigint       NULL COMMENT 'Updated by',
    `del`          tinyint(1)   NOT NULL DEFAULT 0 COMMENT 'Logical delete: 0 active, 1 deleted',
    PRIMARY KEY (`id`)
);
```

在真实项目中，可以根据初始化方式选择：

- 单独提供增量 SQL；
- 将表结构追加到完整初始化脚本；
- 使用 Flyway、Liquibase 等数据库迁移工具管理版本。

## 9. 接口调试示例

以下调试命令已脱敏，仅展示请求结构。

### 新增

```bash
curl -X POST "http://localhost:8080/api/duty/person/insert" \
  -H "Authorization: Bearer <ACCESS_TOKEN>" \
  -H "Content-Type: application/json" \
  --data-raw '{
    "dutyPerson": "测试人员A",
    "department": "测试部门",
    "phone": "13800000000"
  }'
```

响应示例：

```json
{
  "code": 200,
  "msg": "操作成功",
  "data": 1,
  "success": true
}
```

### 查询详情

```bash
curl -X POST "http://localhost:8080/api/duty/person/detail" \
  -H "Authorization: Bearer <ACCESS_TOKEN>" \
  -H "Content-Type: application/json" \
  --data-raw '{"id": 1}'
```

响应示例：

```json
{
  "code": 200,
  "msg": "操作成功",
  "data": {
    "id": 1,
    "dutyPerson": "测试人员A",
    "department": "测试部门",
    "phone": "13800000000",
    "createTime": "2026-01-01 10:00:00",
    "updateTime": "2026-01-01 10:00:00"
  },
  "success": true
}
```

### 分页查询

```bash
curl -X POST "http://localhost:8080/api/duty/person/page" \
  -H "Authorization: Bearer <ACCESS_TOKEN>" \
  -H "Content-Type: application/json" \
  --data-raw '{
    "page": 1,
    "size": 10,
    "dutyPerson": "测试",
    "department": "部门",
    "phone": "138"
  }'
```

响应示例：

```json
{
  "code": 200,
  "msg": "操作成功",
  "data": {
    "records": [
      {
        "id": 1,
        "dutyPerson": "测试人员A",
        "department": "测试部门",
        "phone": "13800000000",
        "createTime": "2026-01-01 10:00:00",
        "updateTime": "2026-01-01 10:00:00"
      }
    ],
    "total": 1,
    "size": 10,
    "current": 1,
    "pages": 1
  },
  "success": true
}
```

### 修改

```bash
curl -X POST "http://localhost:8080/api/duty/person/update" \
  -H "Authorization: Bearer <ACCESS_TOKEN>" \
  -H "Content-Type: application/json" \
  --data-raw '{
    "id": 1,
    "dutyPerson": "测试人员B",
    "department": "更新后的测试部门",
    "phone": "13900000000"
  }'
```

### 删除

```bash
curl -X POST "http://localhost:8080/api/duty/person/delete" \
  -H "Authorization: Bearer <ACCESS_TOKEN>" \
  -H "Content-Type: application/json" \
  --data-raw '{"ids": [1]}'
```

删除接口采用逻辑删除，不物理删除数据库记录。删除后再次分页查询时，结果集中不应再出现对应记录。

## 10. 本次改动边界

本次功能只关注值班人员信息 CRUD，没有修改以下模块：

- 登录与鉴权主流程；
- 用户、部门、角色、权限等系统管理能力；
- 文件导入、文件解析、知识库构建等业务能力；
- 对象存储、搜索引擎、向量数据库或模型服务调用；
- 通用 Bean 拷贝工具。

明确边界有助于代码评审和回归测试，也能降低一次功能迭代引入额外风险的概率。

## 总结

这次改动虽然是常见 CRUD，但覆盖了后端开发中比较完整的一条链路：

1. 先处理 Spring Bean 冲突，确保服务能够正常启动；
2. 再补齐实体、请求对象、响应对象、Mapper、Service、Controller；
3. 最后通过接口调试验证新增、查询、分页、模糊筛选、修改和逻辑删除。

对于业务后台系统来说，这类功能的重点不只是“能增删改查”，还包括参数校验、响应封装、逻辑删除、分页查询、权限边界和调试记录的可追溯性。
