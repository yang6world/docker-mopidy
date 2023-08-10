[![](https://images.microbadger.com/badges/image/wernight/mopidy.svg)](http://microbadger.com/images/wernight/mopidy "在microbadger.com上获取您自己的图像徽章")

什么是 Mopidy？
================

[**Mopidy**](https://www.mopidy.com/) 是一个具有对[MPD客户端](https://docs.mopidy.com/en/latest/clients/mpd/)和[HTTP客户端](https://docs.mopidy.com/en/latest/ext/web/#ext-web)支持的音乐服务器。

此镜像的功能
----------------------

  * 遵循 [官方安装](https://docs.mopidy.com/en/latest/installation/debian/)，构建在[Debian](https://registry.hub.docker.com/_/debian/)之上。
  * 带有以下后端扩展：
      * [Mopidy-Spotify](https://docs.mopidy.com/en/latest/ext/backends/#mopidy-spotify) 用于 **[Spotify](https://www.spotify.com/us/)**（高级版）
      * [Mopidy-GMusic](https://docs.mopidy.com/en/latest/ext/backends/#mopidy-gmusic) 用于 **[Google Play Music](https://play.google.com/music/listen)**
      * [Mopidy-SoundClound](https://docs.mopidy.com/en/latest/ext/backends/#mopidy-soundcloud) 用于 **[SoundCloud](https://soundcloud.com/stream)**
      * [Mopidy-Pandora](https://github.com/rectalogic/mopidy-pandora) 用于 **[Pandora](https://www.pandora.com/)**
      * [Mopidy-YouTube](https://docs.mopidy.com/en/latest/ext/backends/#mopidy-youtube) 用于 **[YouTube](https://www.youtube.com)**
  * 带有 [Mopidy-Moped](https://docs.mopidy.com/en/latest/ext/web/#mopidy-moped) 网页扩展。
  * 默认情况下以 UID/GID `84044` 用户身份在容器内运行（出于安全原因）。

您可以安装其他[后端扩展](https://docs.mopidy.com/en/latest/ext/backends/)。


用法
-----

### 在容器中播放声音

有多种方式可以让来自 Mopidy 的音频在容器中播放到系统的音频输出。以下是几种方式，请尝试并找到适合您的方法。

#### /dev/snd

最简单的方法是添加 Docker 参数：`--device /dev/snd`。尝试使用以下命令：

    $ docker run --rm \
        --user root --device /dev/snd \
        wernight/mopidy \
        gst-launch-1.0 audiotestsrc ! audioresample ! autoaudiosink

#### 本机 PulseAudio

将当前用户的 PulseAudio 目录挂载到 pulseuadio 用户（UID `105`）。基于 https://github.com/TheBiggerGuy/docker-pulseaudio-example。

    $ docker run --rm \
        --user $UID:$GID -v /run/user/$UID/pulse:/run/user/105/pulse \
        wernight/mopidy \
        gst-launch-1.0 audiotestsrc ! audioresample ! autoaudiosink

#### 通过网络的 PulseAudio

首先，为了使[来自 Docker 容器内的音频工作](http://stackoverflow.com/q/28985714/167897)，您应该启用[通过网络的 PulseAudio](https://wiki.freedesktop.org/www/Software/PulseAudio/Documentation/User/Network/)；
因此，如果您有 X11，您可以按照以下步骤操作：

 1. 安装 [PulseAudio Preferences](http://freedesktop.org/software/pulseaudio/paprefs/)。Debian/Ubuntu 用户可以执行以下命令：

        $ sudo apt-get install paprefs

 2. 启动 `paprefs`（PulseAudio Preferences） > "*Network Server*" 选项卡 > 勾选 "*Enable network access to local sound devices*"（您可以勾选 "*Don't require authentication*" 以避免挂载下面描述的 cookie 文件）。

 3. 重启 PulseAudio：

        $ sudo service pulseaudio restart

    或者

        $ pulseaudio -k
        $ pulseaudio --start

注意：在某些发行版上，可能需要完全重新启动计算机。您可以通过运行 `pax11publish | grep -Eo 'tcp:[^ ]*'` 来确认已成功应用设置。您应该会看到类似 `tcp:myhostname:4713` 的内容。

现在设置环境变量：

  * `PULSE_SERVER` - PulseAudio 服务器套接字。
  * `PULSE_COOKIE_DATA` - 十六进制编码的 PulseAudio cookie，通常位于 `~/.config/pulse/cookie`。

以下是一个验证它是否有效的示例：

    $ docker run --rm \
        -e "PULSE_SERVER=tcp:$(hostname -i):4713" \
        -e "PULSE_COOKIE_DATA=$(pax11publish -d | grep --color=never -Po '(?<=^Cookie: ).*')" \
        wernight/mopidy \
        gst-launch-1.0 audiotestsrc ! audioresample ! autoaudiosink

### 一般用法

    $ docker run -d \
        $将此处添加用于使音频正常工作的额外 Docker 参数 \
        -v "$PWD/media:/var/lib/mopidy/media:ro" \
        -v "$PWD/local:/var/lib/mopidy/local" \
        -p 6600:6600 -p 6680:6680 \
        --user $UID:$GID \
        wernight/mopidy \
        mopidy \
        -o spotify/username=用户名 -o spotify/password=密码 \
        -o gmusic/username=用户名 -o gmusic/password=密码 \
        -o soundcloud/auth_token=令牌

大多数参数是可选的（请参阅下面的一些示例）：

  * Docker 参数：
      * `$将此处添加用于使音频正常工作的额外 Docker 参数` 应该替换为在上面测试过的一些参数，以在 Docker 容器内播放音频。
      * `-v ...:/var/lib/mopidy/media:ro` - （可选）本地媒体文件所在的目录路径。
      * `-v ...:/var/lib/mopidy/local` - （可选）用于存储本地元数据（例如库和播放列表）的目录路径。
      * `-p 6600:6600` - （可选）公开 MPD 服务器（如果您使用 ncmpcpp 客户端）。
      * `-p 6680:6680` - （可选）公开 HTTP 服务器（如果您将浏览器用作客户端）。
      * `-p 5555:5555/udp` - （可选）公开 [UDP 用于 FIFE sink 的流媒体](https://github.com/mopidy/mopidy/issues/775)（例如用于可视化）。
      * `--user $UID:$GID` - （可选）您可以以任何 UID/GID 运行， 默认情况下它将以 UID/GID `84044`（容器内的 `mopidy:audio`）身份运行。
        主要限制是如果要读取本地媒体文件：您运行的用户（UID）应该具有对这些文件的读取权限。对于其他挂载也类似。如果遇到问题，请先尝试以 `--user root` 运行。
  * Mopidy 参数（请参阅 [mopidy 的命令](https://docs.mopidy.com/en/latest/command/)，了解可能的其他选项），
    根据需要相应地更换 `用户名`、`密码`、`令牌`，或禁用服务（例如，`-o spotify/enabled=false`）：
      * 对于 *Spotify* 您将需要一个 *高级* 帐户。
      * 对于 *Google Music* 使用您的 Google 帐户（如果您使用 *2步验证*，请生成一个[应用特定密码](https://security.google.com/settings/security/apppasswords)）。
      * 对于 *SoundCloud*，只需在注册后[获取令牌](https://www.mopidy.com/authenticate/)。

注意：您系统上的任何用户都可以运行 `ps aux` 并查看您正在运行的命令行，因此您的密码可能会被泄露。如果这是一个问题，更安全的选择是将这些密码放在基于 [mopidy.conf](mopidy.conf) 的 Mopidy 配置文件中：

    [core]
    data_dir = /var/lib/mopidy

    [local]
    media_dir = /var/lib/mopidy/media

    [audio]
    output = tee name=t ! queue ! autoaudiosink t. ! queue ! udpsink host=0.0.0.0 port=5555

    [m3u]
    playlists_dir = /var/lib/mopidy/playlists

    [http]
    hostname = 0.0.0.0

    [mpd]
    hostname = 0.0.0.0

    [spotify]
    username=用户名
    password=密码

    [gmusic]
    username=用户名
    password=密码

    [soundcloud]
    auth_token=令牌

然后运行：

    $ docker run -d \
        $将此处添加用于使音频正常工作的额外 Docker 参数 \
        -v "$PWD/media:/var/lib/mopidy/media:ro" \
        -v "$PWD/local:/var/lib/mopidy/local" \
        -v "$PWD/mopidy.conf:/config/mopidy.conf" \
        -p 6600:6600 -p 6680:6680 \
        --user $UID:$GID \
        wernight/mopidy


##### 使用 HTTP 客户端从容器中流式传输本地文件的示例

 1. 将您的音频文件的读取权限授予用户 **84044**、组 **84044** 或所有用户（例如，`$ chgrp -R 84044 $PWD/media && chmod -R g+rX $PWD/media`）。
 2. 对本地文件进行索引：

        $ docker run --rm \
            --device /dev/snd \
            -v "$PWD/media:/var/lib/mopidy/media:ro" \
            -v "$PWD/local:/var/lib/mopidy/local" \
            -p 6680:6680 \
            wernight/mopidy mopidy local scan

 3. 启动服务器：

        $ docker run -d \
            -e "PULSE_SERVER=tcp:$(hostname -i):4713" \
            -e "PULSE_COOKIE_DATA=$(pax11publish -d | grep --color=never -Po '(?<=^Cookie: ).*')" \
            -v "$PWD/media:/var/lib/mopidy/media:ro" \
            -v "$PWD/local:/var/lib/mopidy/local" \
            -p 6680:6680 \
            wernight/mopidy

 4. 浏览至 http://localhost:6680/

#### 使用 [ncmpcpp](https://docs.mopidy.com/en/latest/clients/mpd/#ncmpcpp) MPD 控制台客户端的示例

    $ docker run --name mopidy -d \
        -v /run/user/$UID/pulse:/run/user/105/pulse \
        wernight/mopidy
    $ docker run --rm -it --net container:mopidy wernight/ncmpcpp ncmpcpp

如果您不需要可视化效果，也可以：

    $ docker run --rm -it --link mopidy:mopidy wernight/ncmpcpp ncmpcpp --host mopidy


### 反馈

还有更多问题吗？[在 GitHub 上报告错误](https://github.com/wernight/docker-mopidy/issues)。如果您需要一些未安装的附加扩展/插件（请解释原因）。

### Alsa 音频

对于非 Debian 发行版，/etc/group 中音频组的 GID 必须为 29，以匹配 debians 的默认值，如 `audio:x:29:<Docker 外部的用户>`，这是为了匹配容器内的用户 ID。您还需要在 `mopidy.conf` 中的音频部分下添加 `output = alsasink` 配置行。

```
$ docker run -d -rm \
  --device /dev/snd \
  --name mopidy \
  --ipc=host \
  --privileged \
  -v $HOME/.config/mopidy:/var/lib/mopidy/.config/mopidy/ \
  -p 6600:6600/tcp -p 6680:6680/tcp -p 5555:5555/udp \
  mopidy
```

