# 参数列表

### `_add_stale_mv_to_dependency_list`
将过时的物化视图添加到依赖列表

### `_aggregation_optimization_settings`
聚合优化设置

### `_always_anti_join`
尽可能始终使用此方法进行反连接

### `_always_semi_join`
尽可能始终使用此方法进行半连接

### `_always_star_transformation`
始终优先使用星型转换

### `_and_pruning_enabled`
允许基于多种机制进行分区剪裁

### `_b_tree_bitmap_plans`
为仅包含 B 树索引的表启用位图计划

### `_bloom_filter_enabled`
启用或禁用布隆过滤器

### `_bloom_folding_enabled`
启用布隆过滤器折叠

### `_bloom_predicate_enabled`
启用或禁用布隆过滤器谓词下推

### `_bloom_predicate_pushdown_to_storage`
启用或禁用布隆过滤器谓词下推至存储层

### `_bloom_pruning_enabled`
启用使用布隆过滤进行分区剪裁

### `_bloom_pushing_max`
布隆过滤器下推大小上限

### `_bloom_vector_elements`
布隆过滤器向量中的元素数量

### `_bt_mmv_query_rewrite_enabled`
允许使用多个物化视图和基表进行重写

### `_complex_view_merging`
启用复杂视图合并

### `_connect_by_use_union_all`
为 CONNECT BY 使用 UNION ALL

### `_convert_set_to_join`
启用将集合运算符转换为连接

### `_cost_equality_semi_join`
启用等值半连接的成本计算

### `_cpu_to_io`
将 CPU 成本转换为 I/O 成本的除数

### `_db_file_optimizer_read_count`
常规客户端的多块读取计数

### `_default_non_equality_sel_check`
针对 LIKE/范围谓词的默认选择率进行合理性检查

### `_deferred_constant_folding_mode`
延迟常量折叠模式

### `_dimension_skip_null`
控制遇到 NULL 时跳过维度的特性

### `_direct_path_insert_features`
禁用直接路径插入特性

### `_disable_datalayer_sampling`
禁用数据层采样

### `_disable_function_based_index`
禁用基于函数的索引匹配

### `_disable_parallel_conventional_load`
禁用并行常规加载

### `_distinct_view_unnesting`
将 IN 子查询解嵌套为 DISTINCT 视图

### `_dm_max_shared_pool_pct`
挖掘模型可使用的共享池最大百分比

### `_dml_monitoring_enabled`
启用修改监控

### `_eliminate_common_subexpr`
启用消除公共子表达式

### `_enable_dml_lock_escalation`
如果为 TRUE，则启用针对分区表的 DML 锁升级

### `_enable_query_rewrite_on_remote_objs`
在远程表/视图上进行物化视图重写

### `_enable_row_shipping`
对宽表选择使用行传输优化

### `_enable_type_dep_selectivity`
启用依赖于类型的选择率估计

### `_extended_pruning_enabled`
如果设置为 TRUE，则在迭代器中进行运行时剪裁

### `_fast_full_scan_enabled`
启用/禁用索引快速全扫描

### `_fic_area_size`
频繁项集计数工作区的大小

### `_first_k_rows_dynamic_proration`
启用连接基数的动态按比例分配

### `_force_datefold_trunc`
强制对日期折叠重写使用 TRUNC

### `_force_rewrite_enable`
控制新的查询重写特性

### `_force_slave_mapping_intra_part_loads`
强制对分区内加载使用从属映射

### `_force_temptables_for_gsets`
使用临时表执行 ROLLUP 的连接操作

### `_force_tmp_segment_loads`
强制临时段加载

### `_full_pwise_join_enabled`
如果为 TRUE，则启用完全分区智能连接

### `_gby_hash_aggregation_enabled`
启用使用哈希方案的 GROUP BY 和聚合

### `_generalized_pruning_enabled`
控制针对一般谓词的分区剪裁扩展

### `_globalindex_pnum_filter_enabled`
为使用分区扩展语法的全局索引启用过滤器

### `_gs_anti_semi_join_allowed`
为 GS 查询启用反/半连接

### `_hash_join_enabled`
启用/禁用哈希连接

### `_hash_multiblock_io_count`
哈希连接一次将读/写的块数

### `_improved_outerjoin_card`
改进的外连接基数计算

### `_improved_row_length_enabled`
启用计算平均行长度的改进方法

### `_index_join_enabled`
启用索引连接的使用

### `_kdt_buffering`
控制常规插入的 KDT 缓冲

### `_left_nested_loops_random`
为嵌套循环左侧启用随机分布方法

### `_like_with_bind_as_equality`
将带有绑定变量的 LIKE 谓词视为等值谓词

### `_local_communication_costing_enabled`
如果为 TRUE，则启用本地通信成本计算

### `_local_communication_ratio`
设置全局通信与本地通信的比率 (0..100)

### `_minimal_stats_aggregation`
禁止在编译/分区维护时进行统计信息聚合

### `_mmv_query_rewrite_enabled`
允许使用多个物化视图和/或基表进行重写

### `_mv_generalized_oj_refresh_opt`
启用/禁用针对具有广义外连接的物化视图的新算法

### `_nested_loop_fudge`
嵌套循环修正值

### `_new_initial_join_orders`
启用基于新排序启发式方法的初始连接顺序

### `_new_sort_cost_estimate`
启用使用新的成本估算方法进行排序

### `_nlj_batching_enabled`
在嵌套循环中启用右侧 I/O 的批处理

### `_no_or_expansion`
在优化期间禁用 OR 扩展

### `_oneside_colstat_for_equijoins`
针对 LIKE/范围谓词的默认选择率进行合理性检查

### `_optim_adjust_for_part_skews`
调整跨分区的统计信息偏差

### `_optim_enhance_nnull_detection`
TRUE 表示更频繁地启用索引[快速]全扫描

### `_optim_new_default_join_sel`
改进计算默认等值连接选择率的方法

### `_optim_peek_user_binds`
启用查看用户绑定变量

### `_optimizer_adaptive_cursor_sharing`
优化器自适应游标共享

### `_optimizer_adjust_for_nulls`
调整 NULL 值的选择率

### `_optimizer_aw_join_push_enabled`
启用 AW 连接下推优化

### `_optimizer_aw_stats_enabled`
在 AW olap_table 表函数上启用统计信息

### `_optimizer_better_inlist_costing`
启用改进的使用 IN 列表的索引访问成本计算

### `_optimizer_block_size`
优化器使用的标准块大小

### `_optimizer_cache_stats`
使用缓存统计信息进行成本计算

### `_optimizer_cartesian_enabled`
优化器笛卡尔连接已启用

### `_optimizer_cbqt_factor`
基于成本的查询转换的成本因子

### `_optimizer_cbqt_no_size_restriction`
禁用基于成本的转换查询大小限制

### `_optimizer_coalesce_subqueries`
考虑合并子查询优化

### `_optimizer_complex_pred_selectivity`
为内置函数启用选择率估计

### `_optimizer_compute_index_stats`
在索引创建/重建时强制收集索引统计信息

### `_optimizer_connect_by_cb_whr_only`
为 CONNECT BY 中的 WHERE 子句使用基于成本的转换

### `_optimizer_connect_by_combine_sw`
合并无过滤的 CONNECT BY 和 START WITH

### `_optimizer_connect_by_cost_based`
为 CONNECT BY 使用基于成本的转换

### `_optimizer_connect_by_elim_dups`
允许 CONNECT BY 从输入中消除重复项

### `_optimizer_correct_sq_selectivity`
强制正确计算子查询选择率

### `_optimizer_cost_based_transformation`
启用基于成本的查询转换

### `_optimizer_cost_filter_pred`
在 I/O 成本模型中启用过滤谓词的成本计算

### `_optimizer_cost_hjsmj_multimatch`
当每个键的行数 > 1 时，添加生成结果集的成本

### `_optimizer_cost_model`
优化器成本模型

### `_optimizer_degree`
强制优化器使用相同的并行度

### `_optimizer_dim_subq_join_sel`
在选择星型转换维度时使用连接选择率

### `_optimizer_disable_strans_sanity_checks`
禁用星型转换合理性检查

### `_optimizer_distinct_agg_transform`
将 DISTINCT 聚合转换为非 DISTINCT 聚合

### `_optimizer_distinct_elimination`
消除冗余的 SELECT DISTINCT

### `_optimizer_distinct_placement`
考虑 DISTINCT 放置优化

### `_optimizer_eliminate_filtering_join`
优化器过滤连接消除已启用

### `_optimizer_enable_density_improvements`
为选择率估计使用改进的密度计算

### `_optimizer_enable_extended_stats`
为选择率估计使用扩展统计信息

### `_optimizer_enhanced_filter_push`
在尝试基于成本的查询转换之前下推过滤器

### `_optimizer_extend_jppd_view_types`
在 GROUP BY、DISTINCT、半/反连接视图上进行连接谓词下推

### `_optimizer_extended_cursor_sharing`
优化器扩展游标共享

### `_optimizer_extended_cursor_sharing_rel`
针对关系运算符的优化器扩展游标共享

### `_optimizer_extended_stats_usage_control`
控制优化器对扩展统计信息的使用

### `_optimizer_fast_access_pred_analysis`
为物理优化器使用快速算法遍历谓词

### `_optimizer_fast_pred_transitivity`
使用快速算法生成传递性谓词

### `_optimizer_filter_pred_pullup`
使用基于成本的过滤谓词上拉转换

### `_optimizer_fkr_index_cost_bias`
在“前 K 行”模式下，优化器索引相对于全表扫描/快速全扫描的偏向

### `_optimizer_free_transformation_heap`
在每次转换后释放转换子堆

### `_optimizer_group_by_placement`
考虑 GROUP BY 放置优化

### `_optimizer_ignore_hints`
启用忽略嵌入式提示

### `_optimizer_improve_selectivity`
改进表和部分重叠连接的选择率计算

### `_optimizer_instance_count`
强制优化器使用指定数量的实例

### `_optimizer_join_elimination_enabled`
优化器连接消除已启用

### `_optimizer_join_factorization`
使用连接因式分解转换

### `_optimizer_join_order_control`
控制优化器连接顺序搜索算法

### `_optimizer_join_sel_sanity_check`
启用/禁用多列连接选择率合理性检查

### `_optimizer_max_permutations`
每个查询块的优化器最大连接排列数

### `_optimizer_min_cache_blocks`
设置最小缓存块数

### `_optimizer_mjc_enabled`
启用笛卡尔合并连接

### `_optimizer_mode_force`
强制为用户递归 SQL 也设置优化器模式

### `_optimizer_multi_level_push_pred`
考虑需要多级下推到基表的连接谓词下推

### `_optimizer_native_full_outer_join`
使用原生实现执行完全外连接

### `_optimizer_nested_rollup_for_gset`
超过此分组数时，我们对分组集合使用嵌套 ROLLUP 执行

### `_optimizer_new_join_card_computation`
使用未舍入的输入值计算连接基数

### `_optimizer_null_aware_antijoin`
空值感知反连接参数

### `_optimizer_or_expansion`
控制使用的 OR 扩展方法

### `_optimizer_or_expansion_subheap`
为优化器 OR 扩展使用子堆

### `_optimizer_order_by_elimination_enabled`
在查询转换之前从视图中消除 ORDER BY

### `_optimizer_outer_to_anti_enabled`
如果可能，启用将外连接转换为反连接

### `_optimizer_percent_parallel`
优化器并行百分比

### `_optimizer_push_down_distinct`
将 DISTINCT 从查询块下推到表

### `_optimizer_push_down_distinct`
将 DISTINCT 从查询块下推到表

### `_optimizer_push_pred_cost_based`
为下推谓词优化使用基于成本的查询转换

### `_optimizer_random_plan`
用于随机计划的优化器种子值

### `_optimizer_reuse_cost_annotations`
在基于成本的查询转换期间重用成本注释

### `_optimizer_rownum_bind_default`
用于 ROWNUM 绑定的默认值

### `_optimizer_rownum_pred_based_fkr`
启用因 ROWNUM 谓词而使用“前 K 行”

### `_optimizer_search_limit`
优化器搜索限制

### `_optimizer_self_induced_cache_cost`
考虑自我引发的缓存

### `_optimizer_skip_scan_enabled`
启用/禁用索引跳跃扫描

### `_optimizer_skip_scan_guess`
为具有猜测选择率的谓词考虑索引跳跃扫描

### `_optimizer_sortmerge_join_enabled`
启用/禁用排序合并连接方法

### `_optimizer_sortmerge_join_inequality`
启用/禁用使用不等式谓词的排序合并连接

### `_optimizer_squ_bottomup`
以自底向上的方式启用子查询解嵌套

### `_optimizer_star_tran_in_with_clause`
在 WITH 子句查询中启用/禁用星型转换

### `_optimizer_star_trans_min_cost`
优化器星型转换最小成本

### `_optimizer_star_trans_min_ratio`
优化器星型转换最小比率

### `_optimizer_starplan_enabled`
优化器星型计划已启用

### `_optimizer_system_stats_usage`
系统统计信息使用

### `_optimizer_table_expansion`
考虑表扩展转换

### `_optimizer_transitivity_retain`
在生成传递性等值谓词时保留等值连接谓词

### `_optimizer_try_st_before_jppd`
在连接谓词下推之前尝试星型转换

### `_optimizer_undo_changes`
撤销对查询优化器的更改

### `_optimizer_undo_cost_change`
优化器撤销成本变更

### `_optimizer_unnest_all_subqueries`
启用解嵌套每种子查询类型

### `_optimizer_unnest_corr_set_subq`
解嵌套相关集合子查询 (TRUE/FALSE)

### `_optimizer_unnest_disjunctive_subq`
解嵌套析取子查询 (TRUE/FALSE)

### `_optimizer_use_cbqt_star_transformation`
使用基于 CBQT 框架重写的星型转换

### `_optimizer_use_feedback`
优化器使用反馈

### `_optimizer_use_subheap`
启用物理优化器子堆

### `_or_expand_nvl_predicate`
为 NVL/DECODE 谓词启用 OR 扩展计划

### `_ordered_nested_loop`
启用有序嵌套循环成本计算

### `_parallel_broadcast_enabled`
启用将小输入广播到哈希和排序合并连接

### `_parallel_cluster_cache_policy`
用于集群上并行执行的策略 (ADAPTIVE/CACHED)

### `_parallel_scalability`
用于推导并行度的并行可扩展性标准

### `_parallel_syspls_obey_force`
如果为 TRUE，则在系统 PL/SQL 下遵循强制并行查询/DML/DDL

### `_parallel_time_unit`
用于推导并行度的工作单位（以秒为单位）

### `_partial_pwise_join_enabled`
如果为 TRUE，则启用部分分区智能连接

### `_partition_view_enabled`
启用/禁用分区视图

### `_pga_max_size`
一个进程的 PGA 内存最大大小

### `_pivot_implementation_method`
PIVOT 实现方法

### `_pre_rewrite_push_pred`
在重写之前将谓词下推到视图中

### `_pred_move_around`
启用谓词移动

### `_predicate_elimination_enabled`
如果设置为 TRUE，则允许消除谓词

### `_project_view_columns`
启用投影出视图中未引用的列

### `_push_join_predicate`
启用将连接谓词推入视图内

### `_push_join_union_view`
启用将连接谓词推入 UNION ALL 视图内

### `_push_join_union_view2`
启用将连接谓词推入 UNION 视图内

### `_px_broadcast_fudge_factor`
设置 TQ 广播修正因子的百分比

### `_px_minus_intersect`
为 MINUS/INTERSECT 运算符启用并行查询

### `_px_pwg_enabled`
启用并行分区智能分组

### `_px_ual_serial_input`
为 UNION 运算符启用新的并行查询

### `_query_cost_rewrite`
使用物化视图执行基于成本的重写

### `_query_mmvrewrite_maxcmaps`
查询 MMV 重写每个查询析取中每个 DMAP 的最大 CMAP 数

### `_query_mmvrewrite_maxdmaps`
查询 MMV 重写每个查询析取的最大 DMAP 数

### `_query_mmvrewrite_maxinlists`
查询 MMV 重写每个析取的最大 IN 列表数

### `_query_mmvrewrite_maxintervals`
查询 MMV 重写每个析取的最大区间数

### `_query_mmvrewrite_maxmergedcmaps`
查询 MMV 重写最大合并 CMAP 数

### `_query_mmvrewrite_maxpreds`
查询 MMV 重写每个析取的最大谓词数

### `_query_mmvrewrite_maxqryinlistvals`
查询 MMV 重写最大查询 IN 列表值数

### `_query_mmvrewrite_maxregperm`
查询 MMV 重写最大区域排列数

### `_query_rewrite_1`
在视图合并之前或之后执行查询重写，或仅在之前执行

### `_query_rewrite_2`
在视图合并之前或之后执行查询重写，或仅在之后执行

### `_query_rewrite_drj`
物化视图重写并删除冗余连接

### `_query_rewrite_expression`
使用表达式的规范形式进行重写

### `_query_rewrite_fpc`
物化视图重写新鲜分区包含

### `_query_rewrite_fudge`
使用物化视图进行基于成本的查询重写修正因子

### `_query_rewrite_jgmigrate`
使用 JG 迁移的物化视图重写

### `_query_rewrite_maxdisjunct`
查询重写最大析取数

### `_query_rewrite_or_error`
如果引用的表不是空数据表，则允许查询重写

### `_query_rewrite_setopgrw_enable`
使用集合运算符摘要执行通用重写

### `_query_rewrite_vop_cleanup`
在视图合并后的重写之前修剪 frocol 链

### `_rdbms_internal_fplib_enabled`
在关系数据库管理系统内启用 CELL FPLIB 过滤

### `_remove_aggr_subquery`
启用移除被包含的聚合子查询

### `_replace_virtual_columns`
用虚拟列替换表达式

### `_result_cache_auto_size_threshold`
结果缓存自动允许的最大大小

### `_result_cache_auto_time_threshold`
结果缓存自动时间阈值

### `_right_outer_hash_enable`
右外/半/反哈希连接已启用

### `_row_shipping_explain`
启用行传输执行计划支持

### `_row_shipping_threshold`
行传输列选择阈值

### `_rowsrc_trace_level`
行源树跟踪级别

### `_selfjoin_mv_duplicates`
控制重写自连接算法

### `_simple_view_merging`
控制优化器执行的简单视图合并

### `_slave_mapping_enabled`
如果为 TRUE，则启用从属映射

### `_smm_auto_cost_enabled`
如果为 TRUE，则使用自动大小策略的成本函数

### `_smm_auto_max_io_size`
在自动模式下排序/哈希连接使用的最大 I/O 大小（以 KB 为单位）

### `_smm_auto_min_io_size`
在自动模式下排序/哈希连接使用的最小 I/O 大小（以 KB 为单位）

### `_smm_max_size`
在自动模式（串行）下的最大工作区大小

### `_smm_min_size`
在自动模式下的最小工作区大小

### `_smm_px_max_size`
在自动模式（全局）下的最大工作区大小

### `_sort_elimination_cost_ratio`
在 FIRST_ROWS 模式下消除排序的成本比率

### `_sort_multiblock_read_count`
排序的多块读取计数

### `_spr_push_pred_refspr`
通过引用电子表格下推谓词

### `_sql_compatibility`
SQL 兼容性位向量

### `_sql_model_unfold_forloops`
指定 SQL MODEL FOR 循环的编译时展开

### `_subquery_pruning_enabled`
启用使用子查询谓词执行剪裁

### `_subquery_pruning_mv_enabled`
启用使用带有物化视图的子查询谓词执行剪裁

### `_system_index_caching`
优化器系统索引缓存百分比

### `_table_scan_cost_plus_one`
将估计的全表扫描和索引快速全扫描成本增加一

### `_trace_virtual_columns`
跟踪虚拟列表达式

### `_union_rewrite_for_gs`
将带有分组集合的查询展开为 UNION 进行重写

### `_unnest_subquery`
启用解嵌套复杂子查询

### `_use_column_stats_for_function`
为 DDP 函数启用使用列统计信息

### `_virtual_column_overload_allowed`
重载虚拟列表达式

### `_with_subquery`
WITH 子查询转换

### `active_instance_count`
集群数据库中的活动实例数

### `bitmap_merge_area_size`
允许用于位图合并的最大内存

### `cell_offload_compaction`
Cell 数据包压缩策略

### `cell_offload_plan_display`
Cell 下载执行计划显示

### `cell_offload_processing`
启用 SQL 处理卸载到单元格

### `cpu_count`
此实例的 CPU 数量

### `cursor_sharing`
游标共享模式

### `db_file_multiblock_read_count`
每次 I/O 要读取的数据库块数

### `dst_upgrade_insert_conv`
在 DST 升级期间启用/禁用内部转换

### `hash_area_size`
内存中哈希工作区的大小

### `optimizer_capture_sql_plan_baselines`
自动捕获可重复语句的 SQL 计划基线

### `optimizer_dynamic_sampling`
优化器动态采样

### `optimizer_features_enable`
优化器计划兼容性参数

### `optimizer_index_caching`
优化器索引缓存百分比

### `optimizer_index_cost_adj`
优化器索引成本调整

### `optimizer_mode`
优化器模式

### `optimizer_secure_view_merging`
优化器安全视图合并和谓词下推/移动

### `optimizer_use_invisible_indexes`
不可见索引的使用 (TRUE/FALSE)

### `optimizer_use_pending_statistics`
控制是否使用优化器待处理统计信息

### `optimizer_use_sql_plan_baselines`
对捕获的 SQL 语句使用 SQL 计划基线

### `parallel_degree_limit`
对并行度的限制

### `parallel_degree_policy`
用于计算并行度的策略 (MANUAL/LIMITED/AUTO)

### `parallel_force_local`
强制单实例执行

### `parallel_min_time_threshold`
计划成为并行化候选者的阈值（以秒为单位）

### `parallel_threads_per_cpu`
每个 CPU 的并行执行线程数

### `pga_aggregate_target`
实例消耗的聚合 PGA 内存的目标大小

### `query_rewrite_enabled`
如果启用，则允许使用物化视图重写查询

### `query_rewrite_integrity`
使用所需完整性级别使用物化视图执行重写

### `result_cache_mode`
结果缓存运算符使用模式

### `skip_unusable_indexes`
如果设置为 TRUE，则跳过不可用索引

### `sort_area_retained_size`
在提取调用之间保留的内存中排序工作区的大小

### `sort_area_size`
内存中排序工作区的大小

### `star_transformation_enabled`
启用星型转换的使用

### `statistics_level`
统计信息级别

### `workarea_size_policy`
用于确定 SQL 工作区大小的策略 (MANUAL/AUTO)




