# songthumbnail

## 需求

用户提供5000首歌的列表，从itunes查询歌曲对应的thumbnail  

## 需求分析

- 目前知道itunes api限制每分钟20次请求。5000首歌需要5小时。或许这是可以接受的时间，不需要考虑优化
- 确定itunes api是怎么限制请求。是不是IP封锁一定时间，比如1小时。为了不被封IP，需要限制请求发送的频率
- 歌曲可能有拼写错误
- 可能查询不到歌曲
- 一首歌可能查询到多条记录。或许可以默认选择第一条，最后人工确定是否正确。
- 为了修正错误，程序要跑多次。争取为尽量多的歌曲生成图片

## 总体设计

不考虑优化，程序跑在单机上。拿到歌曲列表后，转换为附带格外信息的列表

原始歌曲列表格式。特殊字符分割的文本文件，一般称为csv文件。
多数默认是以,号分割，但用什么都可以。程序都可以简单处理，而且导入很多软件比如word，都支持自定义分隔符:

```javascript
name||artist
```

歌曲处理列表

```code
id||name||artist||state||thumbnailUrl||fileName

state: 当前条目处理结果
    none: 或者空。新条目，没有被处理过
    ok: 已经处理，图片生成正确。
    error_network; 网络错误
    error_no_song: 找不到歌曲信息
    error_multiple: 多个条目无法确认

fileName: 去除特殊字符。一般允许字母和数字，其它的都用"-"替代

```

第一次拿到原始歌曲列表时，运行一个简单的转换程序，转换成以上格式的列表

## 程序逻辑

```javascript
// 伪代码

// 读入列表
songList = await readSongList(pathToSongListFile);

// 扫描没有被处理的首歌，抓图片
songList.forEach(function(song) {
    // 如果已经处理过，就跳过
    if (isSongProcessed(song)) {
        return;
    }

    // 拿到meta，主要是thumbnailUrl
    songMeta = await getSongMeta(song.songName, song.artist);

    // 更新歌曲的信息，以便最后保持列表文件
    song.state = songMeta.state;
    if (song.thumbnailUrl) {
        song.thumbnailUrl = songMeta.thumbnailUrl;

        // 如果拿到图片url，则下载回来
        await saveThumbnail(songMeta.fileName);
    }
});

// 保存列表文件。也可以考虑在前面遍历列表时，每处理完一条，就append一条，效率也比较高。
saveList(songList);


// 访问api，查询到歌曲的信息。返回一个object
// object = {
//     state: 可以使用歌曲列表的state
//     thumbnailUrl: 图片的url
// }
function getSongMeta(songName, artist) {
    // make api call to get song meta
}
```

```javascript
// 编程的最佳实践。一份高度可读的代码，应该是不需要注释的。
// 函数命代表了正在做的事情。通过合理的划分函数，可以清晰的描述程序流程。
// 比如以下代码就很清晰的表达了正在做什么事情：  
if (isSongProcessed(song)) {
    return;
}

songMeta = await getSongMeta(song.songName, song.artist);


以下代码就很难明白作者正在干啥
song.state = songMeta.state;
song.thumbnailUrl = songMeta.thumbnailUrl;

如果写成一个函数，就更清晰了。
updateSong(song, songMeta);

updateSong(song, songMeta) {
    song.state = songMeta.state;
    song.thumbnailUrl = songMeta.thumbnailUrl;
}

但很多时候细分到这个程度，需要更多的函数。
现实里往往需要取舍，而且很多程序员也并不在乎，领导更不在乎。
经常能看到上千行的一个函数。但如果你读一份成熟的代码，往往都是由小函数组成，逻辑清晰。
```

## 加分项。如何突破itunes api每分钟20次请求的限制，优化程序

- 分布式设计，程序可以在多台服务器上跑，突破单ip的限制。这里可以考虑lambda的微服务，有可能每个lambda实例都能分配到一个ip。在真正实现之前，先做简单的proof of concept尝试方案是否可行，实能能同时起多个lambda服务，每个服务得到不同ip，给itunes发请求而不被封。
- 因为分布式，每个实例不能再跑完整的列表，否则每个实例就在做重复的事情。可以考虑手动分配列表，比如一个实例上跑1000条，起5个实例。
- 因为跑在云上，所以图片文件和列表文件最好都放在s3上
- 更好的设计是列表文件放到数据库里。实例每次访问数据库，只拿一条未被处理的歌曲，访问itunes处理歌曲，然后更新列表条目。这样更灵活，也不需要手动维护每个实例上的列表。但这样要考虑并发问题。
- 数据库并发主要有两个概念. transaction和isolation level. 随便google能找到大量资料。
- https://blog.csdn.net/zhangzeyuaaa/article/details/46400419
- https://en.wikipedia.org/wiki/Isolation_(database_systems)
- 还可以考虑不让client直接访问客户端，而是再做一个控制层api，client只通过这个控制层来拿任务以及更新列表状态
