# JVM实现细节的讨论

> JVM/编程/Java

> 这个简单介绍，Cron源自Unix/Linux系统自带的crond守护进程，以一个简洁的表达式定义任务触发时间。在Spring中，也可以使用Cron表达式来执行Cron任务，在Spring中。

#### 3.1 商品评价-编写自定义mapper查询

1. 在mall-mapper项目中src/main/resources/mapper/ItemsMapperCustom.xml。直接看mapper文件的编写

    ```xml
    <select id="queryItemComments" parameterType="Map" resultType="cn.bravedawn.pojo.vo.ItemCommentVO">
            SELECT
            ic.comment_level as commentLevel,
            ic.content as content,
            ic.sepc_name as specName,
            ic.created_time as createdTime,
            u.face as userFace,
            u.nickname as nickname
            FROM
            items_comments ic
            LEFT JOIN
            users u
            ON
            ic.user_id = u.id
            WHERE
            ic.item_id = #{paramsMap.itemId}
            <if test=" paramsMap.level != null and paramsMap.level != '' ">
                AND ic.comment_level = #{paramsMap.level}
            </if>
    </select>
    ```

    在上面的编写方法中并没有编写特定的resultMap，而是在resultType中直接写了参数路径。在后面的开发工作中，值得借鉴。

#### 3.2 商品评价-Spring Boot整合Mybatis-pagehelper

1. 在mall项目的pom文件中引入MyBatis-pageHelper的依赖

    ```xml
    <!--pagehelper -->
    <dependency>
        <groupId>com.github.pagehelper</groupId>
        <artifactId>pagehelper-spring-boot-starter</artifactId>
        <version>1.2.12</version>
    </dependency>
    ```

2. 在mall-api的application.yml文件中配置

    ```yaml
    # 分页插件配置
    pagehelper:
        helperDialect: mysql                   # 配置方言
        supportMethodsArguments: true          # 分页是否支持传递方法参数
    ```

3. 使用分页插件，在查询前使用分页插件。原理：统一拦截SQL，为其通过分页功能

    ```java
    /**
    * page：第几页
    * pageSize：每页显示的条数
    */
    PageHelper.startPage(page, pageSize);
    ```

4. 分装数据到PagedGridResult传给前端

    ```java
    PageInfo<?> pageList = new PageInfo<>(list);
    PagedGridResult grid = new PagedGridResult();
    grid.setPage(page);
    grid.setRows(list);
    grid.setTotal(pageList.getPages());
    grid.setRecords(pageList.getTotal());
    ```

#### 3.3 用户信息脱敏处理

代码详情参见mall-common的cn.bravedawn.utils.DesensitizationUtil。下面我详细的分析一下这个脱敏算法的内部逻辑：

这个代码我debug了一下，感觉不是很难。分别对脱敏字符串长度为1、2、3、7、其他这些进行了处理。
