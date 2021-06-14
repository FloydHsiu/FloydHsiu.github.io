---
layout: post
title:  "GStreamer Basic Tutorial Note"
date:   2021-06-14 11:00:00 +0800
categories: GStreamer Tutorial
---
## Basic-Tutorial 2

![]({{'assets/images/2021-06-14-GStreamer-Tutorial/bt2-figure-1.png' | relative_url}})

1. `gst_pipeline_new(const gchar *name)`
    - 建立pipeline, bin

2. `gst_element_factory_make (const gchar * factoryname,
                          const gchar * name)`
    - 建立元件

3. `gst_bin_add_many (GstBin * bin,
                  GstElement * element_1,
                  ... ...)`：
    - 將元件放入pipeline中，使用NULL作為list結尾
    - Ex：`gst_bin_add_many(pipeline, source, sink, NULL);`
    - 也可使用`gst_bin_add()`一次將一個元件放入pipeline中
4. `gst_element_link(GstElement * src, GstElement * dest, GstCaps * filter)`
    - 將元件進行連接
    - Ex：`gst_element_link(source, sink);`
    - 只能將放置於同一個pipeline的元件進行連接，因此需要先執行`gst_bin_add_many()`將元件放入pipeline中。
    - 此function輸入參數有方向性

## Basic-Tutorial 3

![]({{'assets/images/2021-06-14-GStreamer-Tutorial/bt3-src-element.png' | relative_url}})
![]({{'assets/images/2021-06-14-GStreamer-Tutorial/bt3-filter-element.png' | relative_url}})
![]({{'assets/images/2021-06-14-GStreamer-Tutorial/bt3-sink-element.png' | relative_url}})

Fig. GStreamer elements with their pads.

![]({{'assets/images/2021-06-14-GStreamer-Tutorial/bt3-filter-element-multi.png' | relative_url}})

Fig. A demuxer with two source pads.

![]({{'assets/images/2021-06-14-GStreamer-Tutorial/bt3-simple-player.png' | relative_url}})

Fig. Example pipeline with two branches.

1. pad為
    - 一個元件與其他元件進行連接的接口
    - 分為source pad及sink pad
    - source element只有source pad
    - sink element只有sink pad
    - filter element有source pad及sink pad
2. demuxer
    - 在影音檔案中，通常包含多個類型的訊號源，如：影像串流、音訊串流
    - 因此gstreamer也提供相對應的element稱為**demuxer**，將不同類型的資訊分離開來，使我們得以針對個別類型進行處理
    - 對於demuxer來講，在未讀取到資訊前，它並不知道可能有哪些種類的資訊需要進行分離，因此需要動態的規劃pipeline路徑，將不同的source pad連接到對應的sink pad上。
    
> Embedding multiple streams inside a single file is called “multiplexing” or “muxing”, and such file is then known as a “container”. Common container formats are Matroska (.mkv), Quicktime (.qt, .mov, .mp4), Ogg (.ogg) or Webm (.webm).
    Retrieving the individual streams from within the container is called “demultiplexing” or “demuxing”.

3. `g_signal_connect(data.source, "pad-added", G_CALLBACK (pad_added_handler), &data)`
    - 建立一個針對source element進行監看的callback function，來監看source element是否有增加新的source pad，若有新增source pad則將該pad連接對應的sink pad。

## Basic-Tutorial 4

切換影片播放時間
- How to query the pipeline for information using `GstQuery`
- How to obtain common information like position and duration using `gst_element_query_position()` and `gst_element_query_duration()`
- How to seek to an arbitrary position in the stream using `gst_element_seek_simple()`

## Basic-Tutorial 5

簡易串流播放器
使用到gtk3.0來製作圖形化界面
```bash
pkg-config --cflags --libs gstreamer-video-1.0 gtk+-3.0 gstreamer-1.0
```
在complie時會出現pkg-config找不到gstreamer-video及gtk的錯誤，執行下列指令解決錯誤

```bash
sudo apt install libgstreamer1.0-dev libgstreamer-plugins-base1.0-dev
sudo apt-get install build-essential libgtk-3-dev
```

## Basic-Tutorial 6

### Pad Capabilities
- pad是資料進出元件的接口。而Pad Capabilities（Caps）則是定義哪些格式的資料可以通過這個接口。EX：影片的編碼格式、解析度、FPS。
- 一個pad可以接受多種格式的資料，但在真正傳遞資料時，pad和pad之間必須先透過**negotiation**來決定好**一個**最適合的傳遞格式。
- 因此，兩元件如果可以進行相連，則他們可接受的資料傳輸格式必須有所交集。

### Pad Templates
- pad是參考pad templates來建立，其中指出pad可以接受的資料格式。
- 透過pad templates可以快速的進行初步的資料格式篩選，如果兩個即將相連的pad沒有可接受格式上的交集，那就不需要進行下一步的negotiation。
- 下方為一個pad templates的範例
    ```
    SINK template: 'sink'
    Availability: Always
    Capabilities:
    audio/x-raw
               format: S16LE
                 rate: [ 1, 2147483647 ]
             channels: [ 1, 2 ]
    audio/x-raw
               format: U8
                 rate: [ 1, 2147483647 ]
             channels: [ 1, 2 ]
    ```

### Element Factory
- `gst_element_factory_make()`
    - 直接產生element
- `gst_element_factory_find()`+`gst_element_factory_create()`
    - 先找到可以產生該element的factory，再利用該factory建立element
- 以上兩種方法等價

## Basic-Tutorial 7

### MultiThreading

![]({{'assets/images/2021-06-14-GStreamer-Tutorial/bt7.png' | relative_url}})

對於有多於一個sink pad的pipeline通常都需要multithread。因為，為了同步執行，通常一個sink在未準備完成前會被block，他們無法準備完成，因為只有一個thread進行運算，而它被第一個sink佔用了。

此時，則可以像圖中一樣，利用**Queue**元件，自動建立mutlithread。

### Type of Pad
1. Sometimes Pad
    - EX：uridecodebin在一開始並沒有pad，當資料開始讀入才知道會有哪些pad
2. Always Pad
    - 一般的狀態
3. Request Pad
    - 在要求時才產生
    - EX:**Tee**為一個有sink pad但一開始沒有src pad的元件，其用於將輸入資訊複製成多個輸出。
```c
GstElement *tee;
tee = gst_element_factory_make("tee", "tee");
GstPad *tee_audio_pad;
// 取得request pad
tee_audio_pad = gst_element_get_request_pad(tee, "src_%u");
// 連接pad
gst_pad_link(tee_audio_pad, queue_audio_pad) != GST_PAD_LINK_OK;
// 釋放已經不需要的request pad
gst_element_release_request_pad(tee, tee_audio_pad);
// 使用 gst_element_link_many()，可以自動連接Always pad及Request Pad
// 但使用此方式進行連接所產生的request pad不會在使用後自動被release，
// 因此不建議使用此方法來連接request pad。

// 使用後用不到的pad記憶體關聯，可以透過以下方式斬斷
gst_object_unref(tee_audio_pad);
```

## Basic-Tutorial 8
pipeline並不是完全封閉的，可以透過appsrc, appsink來將外部應用的資訊傳入及傳出。

### Buffer
- 在pipeline中傳遞的一整塊資料稱為buffer,也就是GstBuffer
- Buffer為一個資料的單位，但它不一定是固定大小或代表固定時間。一個buffer進入element並不代表會有一個buffer傳出來。
- 一個buffer可能包含多個實際的記憶體空間配置，也就是它可能包含多個GstMemory。
- 所有的buffer都可附加time-stamp及持續時間，用以描述buffer內的內容在何時需要進行解碼、繪製或顯示。

```c
/* Push the buffer into the appsrc */
g_signal_emit_by_name (data->app_source, "push-buffer", buffer, &ret);
/* Retrieve the buffer */
g_signal_emit_by_name (sink, "pull-sample", &sample);
```

### appsrc, appsink
- appsrc：將外部應用的資料提供給pipeline
    - pull mode：當需要資料時跟外部應用請求提供
    - push mode：外部應用根據自己的步調傳送資料
        - 可以監控`enough-data`或`need-data`的訊號來決定何時主動提供資料
- appsink：將pipeline內部的資料提供給外部應用
    - 可以監控`new-sample`訊號來進行傳入資料的接收
    - 其預設的訊號發送是被關閉的，因此需要額外設定
        - `g_object_set (data.app_sink, "emit-signals", TRUE, NULL);`

## Basic-Tutorial 10
link：https://gstreamer.freedesktop.org/documentation/tutorials/basic/gstreamer-tools.html?gi-language=c

- gst-launch-1.0
    - 在不用寫code的狀態下，快速設計gstreamer的pipeline
- gst-inspect-1.0
    - 用來查找element的caps等資訊
- gst-discoverer-1.0
    - 用來分析來源資料的資訊

## Basic-Tutorial 11
link:
https://gstreamer.freedesktop.org/documentation/tutorials/basic/debugging-tools.html?gi-language=c#

- 設定環境變數GST_DEBUG可以改變錯誤資訊顯示的層級
- 設定環境變數GST_DEBUG_DUMP_DOT_DIR可以設定pipeline的圖形化結構輸出圖位置

## Basic-Tutorial 14

link:
https://gstreamer.freedesktop.org/documentation/tutorials/basic/handy-elements.html?gi-language=c
- 一些常用的的元件介紹
