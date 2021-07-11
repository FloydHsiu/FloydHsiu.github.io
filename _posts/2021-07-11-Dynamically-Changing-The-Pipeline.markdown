---
layout: post
title:  "Dynamically Changing The Pipeline"
date:   2021-07-11 21:00:00 +0800
categories: GStreamer Tutorial
---

Reference: 
1. [Pipeline Manipulation](https://gstreamer.freedesktop.org/documentation/application-development/advanced/pipeline-manipulation.html?gi-language=c)
2. [Preroll定義](https://gstreamer.freedesktop.org/documentation/additional/design/preroll.html?gi-language=c)
3. [Blocking Pad Probe](https://gstreamer.freedesktop.org/documentation/application-development/advanced/pipeline-manipulation.html?gi-language=c#using-probes)

當執行的pipeline正處於`PLAYING`狀態時，要如何在不中斷資料流(data flow)的狀態下變更pipeline呢？

以下有一些在進行動態pipeline操作時需要注意的事項：

- 在移除pipeline中的元件的時候，需要確保沒有資料流存在即將被取消連結的pad上，因為這會導致嚴重的pipeline錯誤。記得在解除pad的連結時，先將資料流阻擋於pad之外，例如：source pad設於push mode、sink pad設於pull mode。
- 在將新element加入pipeline當中時，確保將其設定至正確的狀態。通常在允許資料流進入之前，其狀態需要與其親屬元件(pipeline)相同。在一個element剛建立時，其狀態通常為`NULL`，如果此時有資料流傳入，會產生錯誤。
- 在取消元件的連結時，可以在sink pad傳送`EOS`事件，來清空element中正在處理的資料，並在source pad等待`EOS`事件到達後，才開始進行元件分離的處理。如果沒有將資料清空，將可能遺失部分正在元件中的資料。
- 添加元件可能導致pipeline的狀態改變
    - 添加一個**non-prerolled** sink，會將pipeline改變成prerolling state.
    - 移除一個**non-prerolled** sink，會將pipeline從`PAUSED`改變至`PLAYING`狀態
- 添加或移除元件，都會導致該元件的upstream及downstream都重新協議caps或allocators。不過這些元件會自動做到這些工作，所以你不需要做任何的改動。

## 改變pipeline中的元件

```graphviz
digraph graphname{
    rankdir=LR;
    node[shape=box style="rounded"]
    
    subgraph cluster_e1 {
        label="element1";
        style="rounded";
        color=black;
        subgraph cluster_e1_sink{
            label="";
            style="invis";
            e1_sink_node [label=sink];
        }
        subgraph cluster_e1_src{
            label="";
            style="invis";
            e1_src_node [label=src];
        }
        
        e1_sink_node->e1_src_node [style="invis"];
    }
    e1_src_node->e2_sink_node;
    subgraph cluster_e2 {
        label="element2";
        style="rounded";
        color=black;
        subgraph cluster_e2_sink{
            label="";
            style="invis";
            e2_sink_node [label=sink];
        }
        subgraph cluster_e2_src{
            label="";
            style="invis";
            e2_src_node [label=src];
        }
        
        e2_sink_node->e2_src_node [style="invis"];
    }
    e2_src_node->e3_sink_node [labeldistance="10", labelangle="0"];
    subgraph cluster_e3 {
        label="element3";
        style="rounded";
        color=black;
        subgraph cluster_e3_sink{
            label="";
            style="invis";
            e3_sink_node [label=sink];
        }
        subgraph cluster_e3_src{
            label="";
            style="invis";
            e3_src_node [label=src];
        }
        
        e3_sink_node->e3_src_node [style="invis"];
    }
}
```
上圖為一個由三個元件串起的鏈結
假設，我們要將處於PLAYING狀態當中pipeline的`element2`替換成一個新的元件`element4`。
就如上面注意事項提到的，我們不可以直接解除element1's source pad與element2's sink pad之間的連結，這會導致資料流出現錯誤。最合適的做法是，將來自element1 src pad的資料流暫時阻斷(block)，待我們將element2轉換成element2之後再恢復資料流的傳送。

整個執行的流程大致如下：

1. 確保沒有資料流從element1到element2
    - 使用blocking pad probe阻斷element1's source pad。當pad被阻斷時，會同時呼叫probe callback。
    - 此時，在blocking callback當中，將不會有任何的資料從element1流向element2，直到解除阻斷。
    - 此時可以將element1與element2間的連結解除。
2. 確保element2當中的資料可以被element2處理完畢，並將資料流傳送離開element2
    - 為element2's source pad建立一個event probe
    - 在element2's sink pad傳送一個`EOS`的訊號。
    - 待EOS訊號傳至source pad，將會觸發event probe callback，以提醒source pad，element2中的所有資料皆已經處理完畢並清空。
    - 此時，可以將element2及element3之間的連結切斷，並且將element2的狀態切換成NULL。
3. 將element4加入pipeline中，並且將element1,element4,element3連結再一起
    - 將element4的狀態切換成與pipeline之狀態相同
4. 解除element1's source pad的資料流阻斷，串流將繼續進行。

## 範例解說
目的：建立一個每一秒變化影片特效的簡單pipeline

Reference: [Changing elements in a pipeline](https://gstreamer.freedesktop.org/documentation/application-development/advanced/pipeline-manipulation.html?gi-language=c#changing-elements-in-a-pipeline)

0. 每一秒觸發一次影片特效元件切換的流程
    - `g_timeout_add_seconds (1, timeout_cb, loop)`
1. 確保沒有資料流從element1到element2
    ```c
    static gboolean
    timeout_cb (gpointer user_data)
    {
      /* 使用blocking pad probe暫時阻斷來自effect元件之前的資料流 */
      gst_pad_add_probe (blockpad, GST_PAD_PROBE_TYPE_BLOCK_DOWNSTREAM,
          pad_probe_cb, user_data, NULL);

      return TRUE;
    }
    ```
    
2. 確保element2當中的資料可以被element2處理完畢，並將資料流傳送離開element2
    - 在此有一點比較疑惑，在許多說明中都表示，呼叫`gst_pad_remove_probe`會使blocking pad unblock。但在此處，卻在剛剛進入pad_probe_cb就呼叫這個函式。
    - 我個人對此的解釋為，即便在一開始就呼叫此函式，程式執行步驟要將新的資料傳遞進來，也要等到該probe執行完畢。
```c
static GstPadProbeReturn
pad_probe_cb (GstPad * pad, GstPadProbeInfo * info, gpointer user_data)
{
  GstPad *srcpad, *sinkpad;

  GST_DEBUG_OBJECT (pad, "pad is blocked now");

  /* remove the probe first */
  gst_pad_remove_probe (pad, GST_PAD_PROBE_INFO_ID (info));

  /* install new probe for EOS */
  srcpad = gst_element_get_static_pad (cur_effect, "src");
  gst_pad_add_probe (srcpad, GST_PAD_PROBE_TYPE_BLOCK |
      GST_PAD_PROBE_TYPE_EVENT_DOWNSTREAM, event_probe_cb, user_data, NULL);
  gst_object_unref (srcpad);

  /* push EOS into the element, the probe will be fired when the
   * EOS leaves the effect and it has thus drained all of its data */
  sinkpad = gst_element_get_static_pad (cur_effect, "sink");
  gst_pad_send_event (sinkpad, gst_event_new_eos ());
  gst_object_unref (sinkpad);

  return GST_PAD_PROBE_OK;
}
```

3. element2 src pad持續接收不同的資料
    - 接收到EOS以外的event，忽略並往後傳遞
    - 接收到EOS，element2中的資料皆處理完畢，開始進行element的替換。
```c
static GstPadProbeReturn
event_probe_cb (GstPad * pad, GstPadProbeInfo * info, gpointer user_data)
{
  GMainLoop *loop = user_data;
  GstElement *next;

  if (GST_EVENT_TYPE (GST_PAD_PROBE_INFO_DATA (info)) != GST_EVENT_EOS)
    return GST_PAD_PROBE_PASS;

  gst_pad_remove_probe (pad, GST_PAD_PROBE_INFO_ID (info));

  /* push current effect back into the queue */
  g_queue_push_tail (&effects, gst_object_ref (cur_effect));
  /* take next effect from the queue */
  next = g_queue_pop_head (&effects);
  if (next == NULL) {
    GST_DEBUG_OBJECT (pad, "no more effects");
    g_main_loop_quit (loop);
    return GST_PAD_PROBE_DROP;
  }

  gst_element_set_state (cur_effect, GST_STATE_NULL);

  /* remove unlinks automatically */
  gst_bin_remove (GST_BIN (pipeline), cur_effect);
  gst_bin_add (GST_BIN (pipeline), next);
  gst_element_link_many (conv_before, next, conv_after, NULL);
  gst_element_set_state (next, GST_STATE_PLAYING);

  cur_effect = next;

  return GST_PAD_PROBE_DROP;
}
```