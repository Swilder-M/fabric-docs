# 设备端数据上报

## 物模型数据上报

- 修改thing_model_demo.c 的三元组信息为设备对应的三元组信息

```c
char *product_key = "KS78JR0J";
char *device_name = "testdemo";
char *secret = "5b28b44337b53ef5261d6335af5c8075";
```

- 修改 host 和port

```C
char *host = "8.136.220.253";
int port = 1883;
```

- 设置重连间隔和重连次数

```C
int reconnect_interval = 10;
int reconnect_time = 3;
```

- 根据已定义好的物模型所对应TSL修改model_id

```c
char *model_id = "test_model_id";
```

- 属性参数修改

```c
// 原生类型参数设置
int    val_i1     = 0;
int    val_i2     = 0;
char   val_s3[16] = "text";
double val_d4     = 23.7;
double val_d5     = 23.7;
char   val_s6[16] = "1629344092";

// 结构体类型参数设置
emqc_tm_object val_struct[] = {
    { .type = EMQC_TM_INT, "aaaa", .val_i = 0 },
    { .type = EMQC_TM_INT, "bbbb", .val_i = 0 },
    { .type = EMQC_TM_DOUBLE, "cccc", .val_d = 0.0 },
    { .type = EMQC_TM_TERMINATOR, "", .val_o = NULL }
};

// 数组类型参数设置
int *val_arr_int[] = {&val_i1, &val_i2, NULL};
double *val_arr_dou[] = {&val_d4, &val_d5, NULL};
char *val_arr_str[] = {val_s3, val_s6, NULL};

// 结构体数组类型设置
emqc_tm_object val_struct1[] = {
    { EMQC_TM_INT, "asdf", .val_i = 0 },
    { EMQC_TM_INT, "hjkl", .val_i = 0 },
    // 这里是作为结束符必须的
    { EMQC_TM_TERMINATOR, "", .val_o = NULL } 
};


emqc_tm_object val_struct2[] = {
    { EMQC_TM_INT, "asdf", .val_i = 0 },
    { EMQC_TM_INT, "hjkl", .val_i = 0 },
    // 这里是作为结束符必须的
    { EMQC_TM_TERMINATOR, "", .val_o = NULL } 
};

emqc_tm_object *val_arr_struct[] = {
    val_struct1,
    val_struct2,
    NULL // 这里是作为结束符必须的
};

// 最终要上报的数据全部配置到这里
emqc_tm_object property_array[] = {
    { .type = EMQC_TM_INT,  .key = "property_a1", .val_i = 10 },
    { .type = EMQC_TM_DOUBLE,  .key = "property_a2", .val_d = 10.10 },
    { .type = EMQC_TM_STR,  .key = "property_a3", .val_s = val_s6 },
    { .type = EMQC_TM_STRUCT,  .key = "property_b", .val_o = &val_struct },
    { .type = EMQC_TM_ARR_INT,  .key = "property_c", .val_o = &val_arr_int },
    { .type = EMQC_TM_ARR_DOUBLE,  .key = "property_d", .val_o = &val_arr_dou },
    { .type = EMQC_TM_ARR_STR,  .key = "property_e", .val_o = &val_arr_str },
    { .type = EMQC_TM_ARR_STRUCT,  .key = "property_f", .val_o = &val_arr_struct },
    // 这里是作为结束符必须的
    { .type = EMQC_TM_TERMINATOR, .key = "", .val_o = NULL },
};
```

- 事件参数修改

```c
char *event_ident     = "event_type";
emqc_tm_object event_array[] = { 
    { .type = EMQC_TM_INT, .key = "event1111", .val_i = 0 }, 
    { .type = EMQC_TM_DOUBLE, .key = "event2222", .val_d = 0.0 },
    { .type = EMQC_TM_TERMINATOR, .key = "", .val_o = NULL },
};
```

- 服务参数修改

```C
char *service_ident = "service_type";
char *service_in = "input_type";
char *service_out = "output_type";
```

- 根据需求定义自己的服务，将自己的服务放到service_cb中，并处理服务服务的错误响应，以下仅为事例

```C
void *service_cb(void *payload)
{
    tm_request *tr = (tm_request *)payload;
    log_info("#RECV: id: %s, version: %s, method: %s, params: %s", tr->id, tr->version, tr->method, tr->params);
    json_object *jso_tmp = json_tokener_parse(tr->params);
    json_object *jso_val = NULL;
    int val = 0;

    if (json_object_object_get_ex(jso_tmp, service_in, &jso_val)) {
        // 你的服务函数从这里开始
        val = json_object_get_int(jso_val);
        val *= 2;
    }

    json_object *jso = json_object_new_object();
    json_object *jso_data = json_object_new_object();

    json_object_object_add(jso_data, service_out, json_object_new_int(val));
    json_object_object_add(jso, "id", json_object_new_string(tr->id));
    json_object_object_add(jso, "code", json_object_new_int(0));
    json_object_object_add(jso, "data", jso_data);
    char *ret = zstrdup(json_object_to_json_string(jso));
    json_object_put(jso);

    return (void*) ret;
}
```

- 到此，我们的物模型就修改完毕，重新编译，既可以上报数据了。

## 设备影子数据上报

- 同物模型相同首先要修改三元组信息以及host地址和端口号

- 通过sdk提供的一些设置函数，设置三元组，以及对应的回调函数

```C
emqc_ds *ds = emqc_ds_new();

emqc_set_secret(ds->handle, secret);
emqc_set_product_key(ds->handle, product_key);
emqc_set_device_name(ds->handle, device_name);
// 设置回调处理函数
emqc_ds_set_get_callback(ds, get_payload_call);
emqc_ds_set_generic_callback(ds, generic_payload_call);
emqc_ds_set_control_callback(ds, control_payload_call);
// 设置回调函数
emqc_ds_set_reconnect(ds, interval, times);

// 初始化已设置的参数
emqc_ds_init(ds, host, port);
rc = emqc_ds_connect(ds, 60, 1);
```

- 设置请求的body

```C
serv_req *re_update = new_request();
emqc_ds_state st = DESIRED;
char *key = "test"
char *t2[] = {"11","22","33","44","55"};
add_param_arr(re_update, key, t2, STR, 5, st); 
```

可以通过``add_param``系列函数添加不同类型的数据，如下：

```C
int add_param_null(serv_req *request, const char *key, emqc_ds_state s);
int add_param_int(serv_req *request, const char *key, int val, emqc_ds_state s);
int add_param_str(serv_req *request, const char *key, const char *val, emqc_ds_state s);
int add_param_arr(serv_req *request, const char *key, void *val, TAG type, int len, emqc_ds_state s);
```



- 请求数据构建好以后，我们可以调用对应的方法向服务器发送请求：

```C
emqc_ds_asyn_update(ds, re_update);
```

请求函数包括以下几种；

```C
EMQC_ERR_CODE emqc_ds_asyn_get(emqc_ds *handle);
EMQC_ERR_CODE emqc_ds_asyn_update(emqc_ds *handle, serv_req *resquest);
EMQC_ERR_CODE emqc_ds_asyn_delete(emqc_ds *handle, serv_req *request);
```

- 最后，我们释放资源

```C
emqc_disconnect(ds->handle);
emqc_ds_destroy(ds);
```

