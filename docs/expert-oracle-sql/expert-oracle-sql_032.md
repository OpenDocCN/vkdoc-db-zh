# 子查询因子化：SQL 中被低估的强大特性

子查询因子化是本章的第二个主题，也可能是 SQL 中最未被充分利用的特性。每当我撰写文章或进行演示时，我几乎总是找借口包含一两个此特性的例子，而因子化子查询在本书中也占有重要地位。我将首先简要解释什么是因子化子查询，然后给出四个你应该使用它们的充分理由。

## 子查询因子化的概念

我们都知道，数据字典中的视图是专门设计的，以便在语法上让我们的 SQL 语句可以像对待表一样对待它们。我们还知道，如果数据字典视图不存在或需要以某种方式修改，我们可以用一个*内联视图*来替换它。清单 1-11 展示了使用内联视图的传统方式。

清单 1-11. 不使用子查询因子化的传统内联视图

```sql
 SELECT channel_id, ROUND (AVG (total_cost),2) avg_cost
    FROM sh.profits
GROUP BY channel_id;

SELECT channel_id, ROUND (AVG (total_cost), 2) avg_cost
    FROM (SELECT s.channel_id
                ,GREATEST (c.unit_cost, 0)* s.quantity_sold total_cost
            FROM sh.costs c, sh.sales s
           WHERE     c.prod_id = s.prod_id
                 AND c.time_id = s.time_id
                 AND c.channel_id = s.channel_id
                 AND c.promo_id = s.promo_id)
GROUP BY channel_id;
```

你使用 `SH` 模式中的 `PROFITS` 视图，以一种直接的方式编写了 清单 1-11 中的第一个查询。随后你意识到 `UNIT_COST` 的一些值为负，并决定想要将此类成本视为零。一种实现方式是像第二个查询那样，用一个自定义的内联视图替换数据字典视图。

还有另一种，我认为更优的方式来实现同样的定制化。清单 1-12 展示了这种替代结构。

清单 1-12. 一个简单的因子化子查询

```sql
WITH myprofits
     AS (SELECT s.channel_id
               ,GREATEST (c.unit_cost, 0) * s.quantity_sold total_cost
           FROM sh.costs c, sh.sales s
          WHERE     c.prod_id = s.prod_id
                AND c.time_id = s.time_id
                AND c.channel_id = s.channel_id
                AND c.promo_id = s.promo_id)
  SELECT channel_id, ROUND (AVG (total_cost), 2) avg_cost
    FROM myprofits
GROUP BY channel_id;
```

这些语句所做的是将内联视图移出主查询。它现在被命名并指定在语句开头，在主查询之前。我将这个因子化子查询命名为 `MYPROFITS`，并可以在主查询中像引用数据字典视图一样引用它。为消除任何疑虑，因子化子查询与内联视图一样，是单个 SQL 语句私有的，并且因子化子查询本身没有权限问题。你只需要拥有访问因子化子查询所引用的基础对象的权限。

## 提高可读性

使用因子化子查询的第一个原因是为了使包含内联视图的查询更易于阅读。尽管内联视图有时对于 DML 语句是不可避免的，但当涉及到 `SELECT` 或 `INSERT` 语句时，我一般的建议是完全避免使用内联视图。假设你遇到 清单 1-13，它再次基于 `HR` 示例模式，而你想理解它在做什么。

清单 1-13. 一个包含内联视图的 `SELECT` 语句

```sql
 SELECT e.employee_id
        ,e.first_name
        ,e.last_name
        ,e.manager_id
        ,sub.mgr_cnt subordinates
        ,peers.mgr_cnt - 1 peers
        ,peers.job_id_cnt peer_job_id_cnt
        ,sub.job_id_cnt sub_job_id_cnt
    FROM hr.employees e
        ,(  SELECT e.manager_id, COUNT (*) mgr_cnt, job_id_cnt
              FROM hr.employees e
                  ,(  SELECT manager_id, COUNT (DISTINCT job_id) job_id_cnt
                        FROM hr.employees
                    GROUP BY manager_id) jid
             WHERE jid.manager_id = e.manager_id
          GROUP BY e.manager_id, jid.job_id_cnt) sub
        ,(  SELECT e.manager_id, COUNT (*) mgr_cnt, job_id_cnt
              FROM hr.employees e
                  ,(  SELECT manager_id, COUNT (DISTINCT job_id) job_id_cnt
                        FROM hr.employees
                    GROUP BY manager_id) jid
             WHERE jid.manager_id = e.manager_id
          GROUP BY e.manager_id, jid.job_id_cnt) peers
   WHERE sub.manager_id = e.employee_id AND peers.manager_id = e.employee_id
ORDER BY last_name, first_name;
```

这非常令人望而生畏，你深吸了一口气。我要做的第一件事是将这段代码粘贴到一个私有的编辑器窗口中，并将最外层的内联视图移动到因子化子查询中，以使整个事情更易于阅读。清单 1-14 展示了结果的样子。

清单 1-14. 清单 1-13 的修订版，这次用一层内联视图替换为因子化子查询

```sql
WITH sub
     AS (  SELECT e.manager_id, COUNT (*) mgr_cnt, job_id_cnt
             FROM hr.employees e
                 ,(  SELECT manager_id, COUNT (DISTINCT job_id) job_id_cnt
                       FROM hr.employees
                   GROUP BY manager_id) jid
            WHERE jid.manager_id = e.manager_id
         GROUP BY e.manager_id, jid.job_id_cnt)
    ,peers
     AS (  SELECT e.manager_id, COUNT (*) mgr_cnt, job_id_cnt
             FROM hr.employees e
                 ,(  SELECT manager_id, COUNT (DISTINCT job_id) job_id_cnt
                       FROM hr.employees
                   GROUP BY manager_id) jid
            WHERE jid.manager_id = e.manager_id
         GROUP BY e.manager_id, jid.job_id_cnt)
  SELECT e.employee_id
        ,e.first_name
        ,e.last_name
        ,e.manager_id
        ,sub.mgr_cnt subordinates
        ,peers.mgr_cnt - 1 peers
        ,peers.job_id_cnt peer_job_id_cnt
        ,sub.job_id_cnt sub_job_id_cnt
    FROM hr.employees e, sub, peers
   WHERE sub.manager_id = e.employee_id AND peers.manager_id = e.employee_id
ORDER BY last_name, first_name;
```

两个内联视图已被替换为查询开头的两个因子化子查询。因子化子查询由关键字 `WITH` 引导，并位于主查询的 `SELECT` 之前。这次，我能够使用原始内联视图的表别名为每个因子化子查询命名。因子化子查询然后在主查询中像表或数据字典视图一样被引用。

清单 1-14 仍然包含嵌套在我们的因子化子查询中的内联视图，所以我们需要重复这个过程。清单 1-15 展示了所有内联视图被移除后的情况。

清单 1-15. 所有内联视图都被消除



