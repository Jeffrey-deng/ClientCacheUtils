
# ClientCacheUtils
Client-side cache toolkit for requesting data

## [PeriodCache](/src/cache/period_cache.js "PeriodCache")

> 避免频繁请求的缓存工具类（在超时时间之内返回本地数据，反之请求新数据）
> 
>  使用静态方法：使用默认缓存池(localStorage)，所有连接共用一个缓存池
> 
>  使用实例方法：使用用户自定义的缓存池，每个实例共用一个缓存池

### 使用例子：
#### 1、从localStorage缓存中得到相册基本信息组连接，使用静态方法

    //从localStorage缓存中得到相册基本信息组连接
    var album_info_cache_conn = PeriodCache.getOrCreateGroup({
        "groupName": "album_info_cache",
       "timeOut": 900000,
       "reload": {
            "url": "photo.do?method=albumByAjax",
            "params": function (groupName, key) {
                return {"id": key};
            },
            "parse": function (cacheCtx, groupName, key, old_object_value, data) {
                if (data.flag == 200) {
                    var new_object_value = {"album_id": key, "size": data.album.size};
                    return new_object_value;
                } else {
                    return {"album_id": key, "size": 0};
                }
            }
        }
    });
    // 从缓存中获取相册照片数量
    album_info_cache_conn.get(album.album_id, function (cache_album) {
        if (cache_album) {
            album.size = cache_album.size;
        }
        album_handle.openUpdateAlbumModal(album);
    });
 
#### 2、从内存缓存实例中得到相册基本信息组连接，使用实例方法

    // 创建一个定期刷新的内存缓存实例
    var memoryPeriodCache = new PeriodCache({
        cacheCtx: { // 重写cacheCtx，修改存储的位置
            "ctx": {},
            "setItem": function (key, value) {
                this.ctx[key] = value;
            },
            "getItem": function (key) {
                return this.ctx[key];
            },
            "removeItem": function (key) {
                delete this.ctx[key];
            }
        }
    });
    // 从内存缓存实例中得到相册基本信息组连接
    var secureAlbumInfoConn = memoryPeriodCache.getOrCreateGroup({
        "groupName": "album_info_cache",
        "timeOut": 1800000,
        "reload": {
            "url": "photo.do?method=albumByAjax",
            "params": function (groupName, key) {
                return {"id": key, "photos": false};
            },
            "parse": function (cacheCtx, groupName, key, old_object_value, data) {
                if (data.flag == 200) {
                    return data.album;
                } else {
                    return null;
                }
            }
        }
    });
    //使用
    secureAlbumInfoConn.get(photo.album_id, function (album) {
        var album_url = "photo.do?method=album_detail&id=" + photo.album_id;
        if (album) {
            updateModal.find('span[name="album_id"]').text(album.name).parent().attr("href", album_url);
        } else {
            updateModal.find('span[name="album_id"]').text(photo.album_id).parent().attr("href", album_url);
        }
    });


## [ResultsCache](/src/cache/results_cache.js "ResultsCache")

> PeriodCache简化版，当数据只需要缓存在内存，且不需要刷新，或者不是ajax请求，而是耗时的计算任务时，建议使用

### 使用例子：

    // 缓存用户信息的ajax请求
    var user_base_info_cache = new ResultsCache(function (uid, handler) {
        $.get("user.do?method=profile", {"uid": uid}, function (user) {
            if (user) {
                handler(user);
            } else {
                handler(null);
            }
        }).fail(function (XHR, TS) {
            console.error("ResultsCache Error: found exception when load user from internet, text: " + TS);
            handler(null);
        });
    });
    //使用
    user_base_info_cache.compute(photo.uid).then(function(user) {
        var user_home_url = "user.do?method=home&uid=" + photo.uid;
        if (user) {
            updateModal.find('span[name="user_id"]').text(user.nickname).parent().attr("href", user_home_url);
        } else {
            updateModal.find('span[name="user_id"]').text(photo.uid).parent().attr("href", user_home_url);
        }
        // 回调
        formatPhotoToModal_callback(photo);
    });
    
    
## [TaskQueue](/src/queue/task_queue.js "TaskQueue")

> 将按队列执行任务 
>
> 可以队列ajax和普通任务

### 使用例子：

    if (maxIndex <= 100) {
        // 并发数在100内，直接下载
        for (var i = 0; i < maxIndex; i++) {
            downloadFile(files[i]);
        }
    } else {
        // 并发数在100之上，采用队列下载
        var queue = new TaskQueue(function (file) {
            if (file) {
                var dfd = $.Deferred();
                downloadFile(file, function () {
                    dfd.resolve();
                });
                return dfd;
            }
        });
        for (var i = 0; i < maxIndex; i++) {
            queue.append(files[i]);
        }
    }
