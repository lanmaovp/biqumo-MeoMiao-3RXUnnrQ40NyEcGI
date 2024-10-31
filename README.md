
# 别再忽视！PostgreSQL  Public 模式的风险以及安全迁移



> 作者：桦仔 
> 
> 
> 10余年DBA工作经验
> 
> 
> 微信：debolop
> 
> 
> QQ交流群：740052625
> 
> 
> 公众号：数据库实战派


## 问题起因


前几天有群友在群里面咨询



> PG12，13，14，public模式是否可以删除或改名?因为这位群友的公司的PG规范做了修改，不让使用public模式存放数据，但是遗留问题没办法。


另外一位群友说到



> 你还真不好动public。扩展的插件的函数大多默认都在public 下。


![](https://img2024.cnblogs.com/blog/257159/202410/257159-20241019111702684-1864297381.png)


![](https://img2024.cnblogs.com/blog/257159/202410/257159-20241019111713710-1015154007.png)


## PG中默认的public模式带来的问题


* **安全性问题**


public 模式默认对所有数据库用户都开放访问权限。换句话说，所有连接到数据库的用户默认都可以访问 public 模式中的对象（除非你手动修改权限）。


* **命名冲突**


public 模式是所有用户和所有扩展默认使用的模式，容易发生命名冲突。


* **可维护性和隔离性**


使用 public 模式进行业务操作会使数据库的架构设计显得杂乱无章，随着时间推移，尤其是在大型项目或多个项目共享数据库时，public模式中的对象数量会急剧增加


* **版本和扩展的兼容性问题**


许多 PostgreSQL 扩展默认使用 public 模式，如果修改 public 模式或删除它，可能会导致扩展无法正常工作


## 能否重命名 public 模式


我们能不能通过下面命令对public 模式名重命名 ？



```
ALTER SCHEMA public RENAME TO you_schema;
```

实际上重命名 public 模式是不推荐的做法，原因如下


1. **依赖性问题**：许多扩展、插件和默认的 PostgreSQL 设置都假定 public 模式存在。如果直接修改 public 的名称，会导致这些依赖出现问题。
2. **升级问题**：未来如果 PostgreSQL 版本升级，系统或新安装的扩展可能仍然依赖于 public 模式存在。


因此，最好的做法是保留 public 模式，但不在业务中使用它。


## 如何解决这个问题


实际上，我们可以使用迁移的方式，新建一个模式，然后把public模式下的所有业务对象迁移到新建模式下


具体步骤


***第一步***：创建新的模式



```
CREATE SCHEMA employee;
```

***第二步***：迁移所有对象：对表、视图、函数、存储过程等对象分别执行 SET SCHEMA 操作，将它们从 public 模式迁移到 employee 模式。


迁移对象时小心依赖关系，如外键、索引、函数依赖等，迁移时需要确保这些依赖关系不被破坏


使用以下命令逐个迁移：



```
-- 迁移所有表
ALTER TABLE public.table_name SET SCHEMA employee;

-- 迁移所有视图
ALTER VIEW public.view_name SET SCHEMA employee;

-- 迁移所有函数
ALTER FUNCTION public.function_name SET SCHEMA employee;

-- 迁移所有存储过程
ALTER PROCEDURE public.procedure_name SET SCHEMA employee;
```

使用 SQL 动态语句和 PL/pgSQL 编写一个循环来批量迁移 public 模式中的所有表、视图、函数和存储过程到 employee 模式。



```
DO $$ 
DECLARE
    obj record;
BEGIN
    -- 迁移所有表
    FOR obj IN
        SELECT tablename
        FROM pg_tables
        WHERE schemaname = 'public'
    LOOP
        EXECUTE format('ALTER TABLE public.%I SET SCHEMA employee;', obj.tablename);
    END LOOP;

    -- 迁移所有视图
    FOR obj IN
        SELECT viewname
        FROM pg_views
        WHERE schemaname = 'public'
    LOOP
        EXECUTE format('ALTER VIEW public.%I SET SCHEMA employee;', obj.viewname);
    END LOOP;

    -- 迁移所有函数
    FOR obj IN
        SELECT routine_name, routine_schema
        FROM information_schema.routines
        WHERE specific_schema = 'public'
    LOOP
        EXECUTE format('ALTER FUNCTION public.%I() SET SCHEMA employee;', obj.routine_name);
    END LOOP;

    -- 迁移所有存储过程
    FOR obj IN
        SELECT routine_name, routine_schema
        FROM information_schema.routines
        WHERE specific_schema = 'public' AND routine_type = 'PROCEDURE'
    LOOP
        EXECUTE format('ALTER PROCEDURE public.%I() SET SCHEMA employee;', obj.routine_name);
    END LOOP;

END $$;
```

 
***第三步***：设置 search\_path 通过调整 search\_path 让数据库默认使用 employee 模式。


search\_path 的设置顺序非常重要。


将 employee 模式放在前面，确保在业务操作时优先查找 employee 模式的对象，而 public 作为备选模式保留（方便扩展和插件的使用）。


可以修改 PostgreSQL 的 postgresql.conf 文件，或者在会话级别设置 search\_path：



```
SET search_path TO employee, public;
```

 
***第四步***：考虑扩展和插件


许多扩展和插件默认使用 public 模式，例如 PostGIS、pgcrypto 等。


为了避免问题，最好不要修改 public 模式，而是保持其作为扩展使用的默认模式。


## 为什么SQL Server 没有这个问题


SQL Server 没有像 PostgreSQL 那样对 public 模式的强烈依赖，并且其设计理念与 PostgreSQL 的 public 模式存在一些关键区别。


1. **权限管理的不同**


在 SQL Server 中，dbo 是默认的 schema，所有数据库用户默认情况下并不会拥有对 dbo 这个 schema 中对象的完全访问权限。只有拥有 db\_owner 角色的用户才可以完全控制 dbo 这个 schema。


也就是说，除非用户显式授予对 dbo 中对象的访问或修改权限，否则，普通用户是不能随意访问或修改 dbo 这个 schema 下的对象的。


相比之下，PostgreSQL 的 public 这个 schema 在默认情况下是对所有用户开放的。这意味着所有用户都可以在 public 这个 schema 中创建对象，除非手动限制权限。


PostgreSQL的设计会增加意外权限授予和数据泄露的风险，因此在 PostgreSQL 中有时需要避免使用 public schema。


2. **模式设计理念的不同**


在 PostgreSQL 中，public schema 设计为一个所有用户共享的默认命名空间，因此经常发生命名冲突、权限管理不严等问题。


在 SQL Server 中，dbo 是为拥有数据库完全控制权的用户预留的默认命名空间，通常普通用户和 DBA 可以自行创建自定义 schema 来组织和隔离各自的数据库对象。


 


 



 


**参考文章**


https://sdwh.dev/posts/2021/03/SQL\-Server\-What\-Is\-dbo/


https://www.ibm.com/support/pages/microsoft\-sql\-server\-tables\-get\-generated\-dbo\-schema 


https://www.postgresql.org/docs/current/ddl\-schemas.html


https://www.crunchydata.com/blog/be\-ready\-public\-schema\-changes\-in\-postgres\-15


 ![](https://img2024.cnblogs.com/blog/257159/202409/257159-20240908204310924-1005667056.png)


**本文版权归作者所有，未经作者同意不得转载。**


 本博客参考[楚门加速器p](https://tianchuang88.com)。转载请注明出处！
