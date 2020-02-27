# TenDB Cluster参数说明
本篇文档描述在使用TenDB Cluster 过程中各种参数对系统的影响。

## 参数说明
TenDB Cluster兼容MySQL的参数，集群的存储层tendb也是一个MySQL分支。因此下面会主要列出接入层tspider侧的一些参数，及tendb上新增的一些参数。

TenDB Cluster 兼容 MySQL 的错误码，在大多数情况下会返回和 MySQL 一样的错误码。下表列出的则是另外一些TenDB Cluster特有的错误码。

### 常用参数配置
### 其它参数
static struct st_mysql_storage_engine spider_storage_engine =
{ MYSQL_HANDLERTON_INTERFACE_VERSION };

static struct st_mysql_sys_var* spider_system_variables[] = {
  MYSQL_SYSVAR(support_xa),
  MYSQL_SYSVAR(table_init_error_interval),
  MYSQL_SYSVAR(use_table_charset),
  MYSQL_SYSVAR(conn_recycle_mode),
  MYSQL_SYSVAR(conn_recycle_strict),
  MYSQL_SYSVAR(sync_trx_isolation),
  MYSQL_SYSVAR(use_consistent_snapshot),
  MYSQL_SYSVAR(internal_xa_snapshot),
  MYSQL_SYSVAR(force_commit),
  MYSQL_SYSVAR(xa_register_mode),
  MYSQL_SYSVAR(internal_offset),
  MYSQL_SYSVAR(internal_limit),
  MYSQL_SYSVAR(split_read),
  MYSQL_SYSVAR(semi_split_read),
  MYSQL_SYSVAR(semi_split_read_limit),
  MYSQL_SYSVAR(init_sql_alloc_size),
  MYSQL_SYSVAR(reset_sql_alloc),
#if defined(HS_HAS_SQLCOM) && defined(HAVE_HANDLERSOCKET)
  MYSQL_SYSVAR(hs_result_free_size),
#endif
  MYSQL_SYSVAR(multi_split_read),
  MYSQL_SYSVAR(max_order),
  MYSQL_SYSVAR(semi_trx_isolation),
  MYSQL_SYSVAR(semi_table_lock),
  MYSQL_SYSVAR(semi_table_lock_connection),
  MYSQL_SYSVAR(block_size),
  MYSQL_SYSVAR(selupd_lock_mode),
  MYSQL_SYSVAR(sync_autocommit),
  MYSQL_SYSVAR(sync_time_zone),
  MYSQL_SYSVAR(use_default_database),
  MYSQL_SYSVAR(internal_sql_log_off),
  MYSQL_SYSVAR(bulk_size),
  MYSQL_SYSVAR(bulk_update_mode),
  MYSQL_SYSVAR(bulk_update_size),
  MYSQL_SYSVAR(internal_optimize),
  MYSQL_SYSVAR(internal_optimize_local),
  MYSQL_SYSVAR(use_flash_logs),
  MYSQL_SYSVAR(use_snapshot_with_flush_tables),
  MYSQL_SYSVAR(use_all_conns_snapshot),
  MYSQL_SYSVAR(lock_exchange),
  MYSQL_SYSVAR(internal_unlock),
  MYSQL_SYSVAR(semi_trx),
  MYSQL_SYSVAR(connect_timeout),
  MYSQL_SYSVAR(net_read_timeout),
  MYSQL_SYSVAR(net_write_timeout),
  MYSQL_SYSVAR(quick_mode),
  MYSQL_SYSVAR(quick_page_size),
  MYSQL_SYSVAR(low_mem_read),
  MYSQL_SYSVAR(select_column_mode),
#ifndef WITHOUT_SPIDER_BG_SEARCH
  MYSQL_SYSVAR(bgs_mode),
  MYSQL_SYSVAR(bgs_dml),
  MYSQL_SYSVAR(bgs_first_read),
  MYSQL_SYSVAR(bgs_second_read),
#endif
  MYSQL_SYSVAR(first_read),
  MYSQL_SYSVAR(second_read),
  MYSQL_SYSVAR(crd_interval),
  MYSQL_SYSVAR(crd_mode),
#ifdef WITH_PARTITION_STORAGE_ENGINE
  MYSQL_SYSVAR(crd_sync),
#endif
  MYSQL_SYSVAR(store_last_crd),
  MYSQL_SYSVAR(load_crd_at_startup),
  MYSQL_SYSVAR(crd_type),
  MYSQL_SYSVAR(crd_weight),
#ifndef WITHOUT_SPIDER_BG_SEARCH
  MYSQL_SYSVAR(crd_bg_mode),
#endif
  MYSQL_SYSVAR(sts_interval),
  MYSQL_SYSVAR(sts_mode),
#ifdef WITH_PARTITION_STORAGE_ENGINE
  MYSQL_SYSVAR(sts_sync),
#endif
  MYSQL_SYSVAR(store_last_sts),
  MYSQL_SYSVAR(load_sts_at_startup),
#ifndef WITHOUT_SPIDER_BG_SEARCH
  MYSQL_SYSVAR(sts_bg_mode),
#endif
  MYSQL_SYSVAR(ping_interval_at_trx_start),
#if defined(HS_HAS_SQLCOM) && defined(HAVE_HANDLERSOCKET)
  MYSQL_SYSVAR(hs_ping_interval),
#endif
  MYSQL_SYSVAR(auto_increment_mode),
  MYSQL_SYSVAR(same_server_link),
  MYSQL_SYSVAR(trans_rollback),
  MYSQL_SYSVAR(with_begin_commit),
  MYSQL_SYSVAR(get_conn_from_idx),
  MYSQL_SYSVAR(get_sts_or_crd),
  MYSQL_SYSVAR(client_found_rows),
  MYSQL_SYSVAR(local_lock_table),
  MYSQL_SYSVAR(use_pushdown_udf),
  MYSQL_SYSVAR(direct_dup_insert),
  MYSQL_SYSVAR(udf_table_lock_mutex_count),
  MYSQL_SYSVAR(udf_table_mon_mutex_count),
  MYSQL_SYSVAR(udf_ds_bulk_insert_rows),
  MYSQL_SYSVAR(udf_ds_table_loop_mode),
  MYSQL_SYSVAR(remote_access_charset),
  MYSQL_SYSVAR(remote_autocommit),
  MYSQL_SYSVAR(remote_time_zone),
  MYSQL_SYSVAR(remote_sql_log_off),
  MYSQL_SYSVAR(remote_trx_isolation),
  MYSQL_SYSVAR(remote_default_database),
  MYSQL_SYSVAR(connect_retry_interval),
  MYSQL_SYSVAR(connect_retry_count),
  MYSQL_SYSVAR(connect_mutex),
  MYSQL_SYSVAR(bka_engine),
  MYSQL_SYSVAR(bka_mode),
  MYSQL_SYSVAR(udf_ct_bulk_insert_interval),
  MYSQL_SYSVAR(udf_ct_bulk_insert_rows),
#if defined(HS_HAS_SQLCOM) && defined(HAVE_HANDLERSOCKET)
  MYSQL_SYSVAR(hs_r_conn_recycle_mode),
  MYSQL_SYSVAR(hs_r_conn_recycle_strict),
  MYSQL_SYSVAR(hs_w_conn_recycle_mode),
  MYSQL_SYSVAR(hs_w_conn_recycle_strict),
  MYSQL_SYSVAR(use_hs_read),
  MYSQL_SYSVAR(use_hs_write),
#endif
  MYSQL_SYSVAR(use_handler),
  MYSQL_SYSVAR(error_read_mode),
  MYSQL_SYSVAR(error_write_mode),
  MYSQL_SYSVAR(skip_default_condition),
  MYSQL_SYSVAR(skip_parallel_search),
  MYSQL_SYSVAR(direct_order_limit),
  MYSQL_SYSVAR(read_only_mode),
#ifdef HA_CAN_BULK_ACCESS
  MYSQL_SYSVAR(bulk_access_free),
#endif
#if MYSQL_VERSION_ID < 50500
#else
  MYSQL_SYSVAR(udf_ds_use_real_table),
#endif
  MYSQL_SYSVAR(general_log),
  MYSQL_SYSVAR(index_hint_pushdown),
  MYSQL_SYSVAR(max_connections),
  MYSQL_SYSVAR(conn_wait_timeout),
  MYSQL_SYSVAR(ignore_autocommit),
  MYSQL_SYSVAR(fetch_minimum_columns),
  MYSQL_SYSVAR(log_result_errors),
  MYSQL_SYSVAR(log_result_error_with_sql),
  MYSQL_SYSVAR(version),
  MYSQL_SYSVAR(internal_xa_id_type),
  MYSQL_SYSVAR(casual_read),
  MYSQL_SYSVAR(dry_access),
  MYSQL_SYSVAR(delete_all_rows_type),
  MYSQL_SYSVAR(bka_table_name_type),
  MYSQL_SYSVAR(connect_error_interval),
#ifndef WITHOUT_SPIDER_BG_SEARCH
  MYSQL_SYSVAR(table_sts_thread_count),
  MYSQL_SYSVAR(table_crd_thread_count),
#endif
  MYSQL_SYSVAR(quick_mode_only_select),
  MYSQL_SYSVAR(enable_mem_calc),
  MYSQL_SYSVAR(enable_trx_ha),
  MYSQL_SYSVAR(idle_conn_recycle_interval),
  MYSQL_SYSVAR(conn_meta_max_invalid_duration),
  NULL
};
