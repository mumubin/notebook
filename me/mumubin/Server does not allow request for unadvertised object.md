# Server does not allow request for unadvertised object

## 问题

在自己搭建的Gitlab中使用git fetch指定commit时，会出现权限问题

```shell
git fetch --depth=1 origin ada589e4a7f2003d0c31b8bec07df8d6745a9e3b
Server does not allow request for unadvertised object ada589e4a7f2003d0c31b8bec07df8d6745a9e3b
```

通过[官方文档](https://git-scm.com/docs/git-config#Documentation/git-config.txt-uploadpackallowAnySHA1InWant)可得以下三个参数可破
- `uploadpack.allowTipSHA1InWant`
When uploadpack.hideRefs is in effect, allow upload-pack to accept a fetch request that asks for an object at the tip of a hidden ref (by default, such a request is rejected). See also uploadpack.hideRefs. Even if this is false, a client may be able to steal objects via the techniques described in the "SECURITY" section of the gitnamespaces[7] man page; it’s best to keep private data in a separate repository.

- `uploadpack.allowReachableSHA1InWant`
Allow upload-pack to accept a fetch request that asks for an object that is reachable from any ref tip. However, note that calculating object reachability is computationally expensive. Defaults to false. Even if this is false, a client may be able to steal objects via the techniques described in the "SECURITY" section of the gitnamespaces[7] man page; it’s best to keep private data in a separate repository.

- `uploadpack.allowAnySHA1InWant`
Allow upload-pack to accept a fetch request that asks for any object at all. Defaults to false.


## Gitlab服务器端配置

GItlab Server 中的git config在`/opt/gitlab/embedded/etc/gitconfig`，可以通过修改girlab.rb来实现配置化统一管理

```rb
omnibus_gitconfig['system'] = { "uploadpack" => ["allowAnySHA1InWant = true"] }
```

`注意:` /var/opt/gitlab/.gitconfig是用户的config不是server的config,不要去错路径

gitlab重启后去查看`/opt/gitlab/embedded/etc/gitconfig`，可见uploadpack配置已经生效
```config
[pack]
  threads = 1
[receive]
  fsckObjects = true
advertisePushOptions = true
[repack]
  writeBitmaps = true
[transfer]
  hideRefs=^refs/tmp/
hideRefs=^refs/keep-around/
hideRefs=^refs/remotes/
[core]
  alternateRefsCommand="exit 0 #"
fsyncObjectFiles = true
[uploadpack]
  allowAnySHA1InWant = true
```

## 实测

1. Gitlab 打开 `allowReachableSHA1InWant`后
    ```shell
    ~/cmc-rules ᐅ git fetch --depth=1 origin ada589e4a7f2003d0c31b8bec07df8d6745a9e3b
    remote: Total 0 (delta 0), reused 0 (delta 0)
    From sdpgit.xxxx.com:sdp/rules
     * branch            ada589e4a7f2003d0c31b8bec07df8d6745a9e3b -> FETCH_HEAD
    ```
2. Gitlab `allowAnySHA1InWant`后
    ```shell
    ~/cmc-rules ᐅ git fetch --depth=1 origin 8726a92df167b4600ac99940aa7c3f8e9aabd52b
    remote: Enumerating objects: 7, done.
    remote: Counting objects: 100% (7/7), done.
    remote: Compressing objects: 100% (1/1), done.
    remote: Total 3 (delta 1), reused 3 (delta 1)
    Unpacking objects: 100% (3/3), done.
    From sdpgit.xxxx.com:sdp/rules
     * branch            8726a92df167b4600ac99940aa7c3f8e9aabd52b -> FETCH_HEAD
    ```

3. 根据[Stackoverflow](https://stackoverflow.com/questions/31278902/how-to-shallow-clone-a-specific-commit-with-depth-1), `allowTipSHA1InWant`对github有用
无测试环境，欢迎补充
