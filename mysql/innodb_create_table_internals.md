# MySQL (InnoDB) CREATE TABLE internals

## `mysqld.cc`

  - `int mysqld_main(int argc, char **argv)`

  - `void setup_conn_event_handler_threads()`

  - create connection handler thread => `socket_conn_event_handler`

  - `conn_acceptor->connection_event_loop()`

## `connection_acceptor.h`

  -  `void connection_event_loop()`

  - `Connection_handler_manager::get_instance();`

  - `process_new_connection`


## `connection_handler_manager.cc`

  - `process_new_connection(Channel_info* channel_info)`
  
  
## `connection_handler_per_thread.cc`

> could be connection_helper_one_thread.cc

  - `bool Per_thread_connection_handler::add_connection(Channel_info* channel_info)`

  - create thread => `handle_connection`
  
  - loop, wait for commands

  - `do_command` (sql_parse.cc)


## `sql_parse.cc`

  - `do_command`

  - `thd->get_protocol()->get_command(&com_data, &command);`

  - `dispatch_command`

  - big swtich statement, in most cases `COM_QUERY`

  - `mysql_parse`
  
  - `mysql_execute_command`

  - `switch(command)`

  - `case SQLCOM_CREATE_TABLE`

  - `mysql_create_table`

  - `mysql_create_table_impl`

## `unireg.cc`

  - `rea_create_table` (create frm file)

  - `ha_create_table` (innodb)
  
## `handler.cc`

```c
int ha_create_table(THD *thd, const char *path,
                    const char *db, const char *table_name,
                    HA_CREATE_INFO *create_info,
                    bool update_create_info,
                    bool is_temp_table)
```

```
  name= get_canonical_filename(table.file, share.path.str, name_buff);

  error= table.file->ha_create(name, &table, create_info);
```

  - `handler::ha_create(const char *name, TABLE *form, HA_CREATE_INFO *info)`

  - `create` => virtual that must be provided by engine implementations, in our case, its innodb.

## ha_innodb.cc

  - `int ha_innobase::create`

  - 