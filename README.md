**目录**

* [设置st\-\>start\_time](#_label0)
* [计算ic\-\>start\_time](#_label1)
* [设置f\-\>ts\_offset](#_label2)
* [重新写入packet的时间戳](#_label3)
* [把AVPacket的时间戳传递给AVFrame](#_label4)
* [AVFrame中时间戳的调整](#_label5)
* [为重采样后的音频帧设置时间戳](#_label6)
* [从音频帧队列中取出固定长度的采样数据，调整时间戳](#_label7)
* [编码音频帧设置时间戳](#_label8)
* [为最终的编码帧设置时间戳](#_label9)
* [视频帧解码后的pts时间戳设置](#_label10)
* [在filter中对时间戳进行设置](#_label11)
* [在编码器中设置时间戳](#_label12)
* [在封装线程中把时间基修改成输出流协议的时间基](#_label13)
 

**正文**


# 音频时间戳设置


以下代码基于FFmpeg n5\.1\.2进行分析


以下文档中有关音频的具体时间戳数据来自以下转码命令：



```
./ffmpeg_g -rw_timeout 5000000 -i 'rtmp://rustxiu.com/live/test' -acodec libfdk_aac -b:a 64k -ac 2 -ar 48000 -profile:a aac_low  -vcodec libx264 -b:v 2000k -level 3.1 -vprofile high  -strict -2 -preset medium -bf 3  -f flv -loglevel level+info -vf "scale='720:-2'"  'rtmp://rustxiu.com/live/dest'

```

callstack的代码行数有可能和源码对不上，因为手动加了一些日志。


[回到顶部](#_labelTop)## 设置st\-\>start\_time


有两个AVStream的start\_time需要设置，也就是音频的和视频的


调用stack:



```
#0  update_initial_timestamps (s=0x2024dc0, stream_index=1, dts=, pts=579486901, pkt=) at libavformat/demux.c:899
#1  0x0000000000697e74 in compute_pkt_fields (s=s@entry=0x2024dc0, st=st@entry=0x2029340, pc=pc@entry=0x0, pkt=pkt@entry=0x20250c0,     next_dts=next_dts@entry=-9223372036854775808, next_pts=next_pts@entry=-9223372036854775808) at libavformat/demux.c:1124
#2  0x0000000000699274 in read_frame_internal (s=s@entry=0x2024dc0, pkt=pkt@entry=0x20250c0) at libavformat/demux.c:1371
#3  0x000000000069a723 in avformat_find_stream_info (ic=0x2024dc0, options=0x0) at libavformat/demux.c:2663
#4  0x00000000004a3abf in open_input_file (o=o@entry=0x7fffffffd098, filename=0x7fffffffe48c "rtmp://pl.live.weibo.com/alicdn/jingtian") at fftools/    ffmpeg_opt.c:1286
#5  0x000000000049fd24 in open_files (l=0x2024718, inout=inout@entry=0x15d599d "input", open_file=open_file@entry=0x4a346c ) at fftools/    ffmpeg_opt.c:3542
#6  0x00000000004a9f67 in ffmpeg_parse_options (argc=argc@entry=38, argv=argv@entry=0x7fffffffe028) at fftools/ffmpeg_opt.c:3582
#7  0x0000000000499311 in main (argc=38, argv=0x7fffffffe028) at fftools/ffmpeg.c:4683

```

代码：



```
static void update_initial_timestamps(AVFormatContext *s, int stream_index,
                                      int64_t dts, int64_t pts, AVPacket *pkt)
{
    ...

    if (has_decode_delay_been_guessed(st))
        update_dts_from_pts(s, stream_index, pktl);

    if (st->start_time == AV_NOPTS_VALUE) {
        if (st->codecpar->codec_type == AVMEDIA_TYPE_AUDIO || !(pkt->flags & AV_PKT_FLAG_DISCARD)) {
            st->start_time = pts; //取音频首帧时间
        }
        if (st->codecpar->codec_type == AVMEDIA_TYPE_AUDIO && st->codecpar->sample_rate)
            st->start_time = av_sat_add64(st->start_time, av_rescale_q(sti->skip_samples, (AVRational){1, st->codecpar->sample_rate}, st->time_base)); //这个逻辑也走进来了，但是sti->skip_samples为0，最后值没变化。
    }
}

```

[回到顶部](#_labelTop)## 计算ic\-\>start\_time


接下来会使用上一步的 st\-\>start\_time 计算ic\-\>start\_time，音视频分别保存了自己的st\-\>start\_time,计算ic\-\>start\_time的时候会取两个值的较小者。


注：update\_stream\_timings这个函数会被调用两次，但是最后的ic\-\>start\_time的结果是一样的。


调用stack:



```
#0  update_stream_timings (ic=ic@entry=0x2024dc0) at libavformat/demux.c:1619
#1  0x00000000006965cf in fill_all_stream_timings (ic=ic@entry=0x2024dc0) at libavformat/demux.c:1701
#2  0x000000000069bbd2 in estimate_timings (old_offset=13, ic=0x2024dc0) at libavformat/demux.c:1945
#3  avformat_find_stream_info (ic=0x2024dc0, options=) at libavformat/demux.c:2951
#4  0x00000000004a3abf in open_input_file (o=o@entry=0x7fffffffd098, filename=0x7fffffffe48c "rtmp://pl.live.weibo.com/alicdn/jingtian") at fftools/    ffmpeg_opt.c:1286
#5  0x000000000049fd24 in open_files (l=0x2024718, inout=inout@entry=0x15d58dd "input", open_file=open_file@entry=0x4a346c ) at fftools/    ffmpeg_opt.c:3542
#6  0x00000000004a9f67 in ffmpeg_parse_options (argc=argc@entry=38, argv=argv@entry=0x7fffffffe028) at fftools/ffmpeg_opt.c:3582
#7  0x0000000000499311 in main (argc=38, argv=0x7fffffffe028) at fftools/ffmpeg.c:4683

```

代码：



```
static void update_stream_timings(AVFormatContext *ic)
{
...

for (unsigned i = 0; i < ic->nb_streams; i++) { //这里会遍历音频和视频两个AVStream
    AVStream *const st = ic->streams[i];
    int is_text = st->codecpar->codec_type == AVMEDIA_TYPE_SUBTITLE ||
                  st->codecpar->codec_type == AVMEDIA_TYPE_DATA;

    if (st->start_time != AV_NOPTS_VALUE && st->time_base.den) {
        start_time1 = av_rescale_q(st->start_time, st->time_base,
                                   AV_TIME_BASE_Q); //把AVStream的start_time做一下时间基转换，1/1000转成1/100000，也就是把值增加1000倍
        if (is_text)
            start_time_text = FFMIN(start_time_text, start_time1);
        else
            start_time = FFMIN(start_time, start_time1);//这里取两者之间的较小值，也就是音视频首帧的较小者
        end_time1 = av_rescale_q_rnd(st->duration, st->time_base,
                                     AV_TIME_BASE_Q,
                                     AV_ROUND_NEAR_INF|AV_ROUND_PASS_MINMAX);
        if (end_time1 != AV_NOPTS_VALUE && (end_time1 > 0 ? start_time1 <= INT64_MAX - end_time1 : start_time1 >= INT64_MIN - end_time1)) {
            end_time1 += start_time1;
            if (is_text)
                end_time_text = FFMAX(end_time_text, end_time1);
            else
                end_time = FFMAX(end_time, end_time1);
        }
        for (AVProgram *p = NULL; (p = av_find_program_from_stream(ic, p, i)); ) {
            if (p->start_time == AV_NOPTS_VALUE || p->start_time > start_time1)
                p->start_time = start_time1;
            if (p->end_time < end_time1)
                p->end_time = end_time1;
        }
    }
    if (st->duration != AV_NOPTS_VALUE) {
        duration1 = av_rescale_q(st->duration, st->time_base,
                                 AV_TIME_BASE_Q);
        if (is_text)
            duration_text = FFMAX(duration_text, duration1);
        else
            duration = FFMAX(duration, duration1);
    }
}
...
if (start_time != INT64_MAX) {
    ic->start_time = start_time; //赋值给ic->start_time
    if (end_time != INT64_MIN) {
        if (ic->nb_programs > 1) {
            for (unsigned i = 0; i < ic->nb_programs; i++) {
                AVProgram *const p = ic->programs[i];

                if (p->start_time != AV_NOPTS_VALUE &&
                    p->end_time > p->start_time &&
                    p->end_time - (uint64_t)p->start_time <= INT64_MAX)
                    duration = FFMAX(duration, p->end_time - p->start_time);
            }
        } else if (end_time >= start_time && end_time - (uint64_t)start_time <= INT64_MAX) {
            duration = FFMAX(duration, end_time - start_time);
        }
    }
}

```

}


[回到顶部](#_labelTop)## 设置f\-\>ts\_offset


在上一步设置完ic\-\>start\_time之后，会把这个值赋值给f\-\>ts\_offset，接下来会用这个offset对每一个音视频帧的时间戳进行调整。


在 open\_input\_file中调用avformat\_find\_stream\_info设置完ic\-\>start\_time之后，在此函数下面的逻辑中设置f\-\>ts\_offset:



```
static int open_input_file(OptionsContext *o, const char *filename)
{
    ...

    timestamp = (o->start_time == AV_NOPTS_VALUE) ? 0 : o->start_time;
    /* add the stream start time */
    if (!o->seek_timestamp && ic->start_time != AV_NOPTS_VALUE)
        timestamp += ic->start_time; //timestamp设置为ic->start_time

    f->ts_offset = o->input_ts_offset - (copy_ts ? (start_at_zero && ic->start_time != AV_NOPTS_VALUE ? ic->start_time : 0) : timestamp); //这里为0-timestamp 把f->ts_offset设置为ic->start_time的相反数，变成负数了

 }

```

[回到顶部](#_labelTop):[FlowerCloud机场](https://hushicha.org)## 重新写入packet的时间戳


读取每一个音频帧，重新调整时间戳pts，调整后第一帧的时间戳从0开始，也就是每一个AVPacket都要减掉f\-\>ts\_offset，并且进行时间基的转换。



```
static int process_input(int file_index)
{
    ...
    if (pkt->dts != AV_NOPTS_VALUE)
        pkt->dts += av_rescale_q(ifile->ts_offset, AV_TIME_BASE_Q, ist->st->time_base);//把ts_offset的时间基从1/1000000转回1/1000, 也即是首帧音频时间戳的相对数，计算完后，每一帧的时间都减少首帧的时间戳。
    if (pkt->pts != AV_NOPTS_VALUE)
        pkt->pts += av_rescale_q(ifile->ts_offset, AV_TIME_BASE_Q, ist->st->time_base);
    ...
}

```

调用stack:



```
#0  process_input (file_index=0) at fftools/ffmpeg.c:4071
#1  transcode_step () at fftools/ffmpeg.c:4456
#2  transcode () at fftools/ffmpeg.c:4510
#3  main (argc=, argv=) at fftools/ffmpeg.c:4705

```

[回到顶部](#_labelTop)## 把AVPacket的时间戳传递给AVFrame


解封装之后，调整完时间戳，接下来需要解码音频帧了，解完之后，把AVPacket的时间戳直接赋给了AVFrame:



```
int ff_decode_frame_props(AVCodecContext *avctx, AVFrame *frame)
{
   AVPacket *pkt = avctx->internal->last_pkt_props;
   static const struct {
       enum AVPacketSideDataType packet;
       enum AVFrameSideDataType frame;
   } sd[] = {
       { AV_PKT_DATA_REPLAYGAIN ,                AV_FRAME_DATA_REPLAYGAIN },
       { AV_PKT_DATA_DISPLAYMATRIX,              AV_FRAME_DATA_DISPLAYMATRIX },
       { AV_PKT_DATA_SPHERICAL,                  AV_FRAME_DATA_SPHERICAL },
       { AV_PKT_DATA_STEREO3D,                   AV_FRAME_DATA_STEREO3D },
       { AV_PKT_DATA_AUDIO_SERVICE_TYPE,         AV_FRAME_DATA_AUDIO_SERVICE_TYPE },
       { AV_PKT_DATA_MASTERING_DISPLAY_METADATA, AV_FRAME_DATA_MASTERING_DISPLAY_METADATA },
       { AV_PKT_DATA_CONTENT_LIGHT_LEVEL,        AV_FRAME_DATA_CONTENT_LIGHT_LEVEL },
       { AV_PKT_DATA_A53_CC,                     AV_FRAME_DATA_A53_CC },
       { AV_PKT_DATA_ICC_PROFILE,                AV_FRAME_DATA_ICC_PROFILE },
       { AV_PKT_DATA_S12M_TIMECODE,              AV_FRAME_DATA_S12M_TIMECODE },
       { AV_PKT_DATA_DYNAMIC_HDR10_PLUS,         AV_FRAME_DATA_DYNAMIC_HDR_PLUS },
   };

   if (!(ffcodec(avctx->codec)->caps_internal & FF_CODEC_CAP_SETS_FRAME_PROPS)) {
       frame->pts = pkt->pts; // 在这里把AVPacket的时间戳赋值给了AVFrame
       frame->pkt_pos      = pkt->pos;
       frame->pkt_duration = pkt->duration;
       frame->pkt_size     = pkt->size;

    ...
}

```

调用stack:



```
#0  ff_decode_frame_props (avctx=avctx@entry=0x203a4c0, frame=frame@entry=0x2ea2800) at libavcodec/decode.c:1292
#1  0x00000000007fbc73 in ff_get_buffer (avctx=avctx@entry=0x203a4c0, frame=0x2ea2800, flags=flags@entry=0) at libavcodec/decode.c:1468
#2  0x0000000000b879fd in frame_configure_elements (avctx=avctx@entry=0x203a4c0) at libavcodec/aacdec_template.c:184
#3  0x0000000000b8bdc9 in aac_decode_frame_int (avctx=avctx@entry=0x203a4c0, frame=frame@entry=0x2ea2800, got_frame_ptr=got_frame_ptr@entry=0x7fffffffd3f4, gb=gb@entry=0x7fffffffd340, avpkt=0x2ea2b40) at libavcodec/aacdec_template.c:3271
#4  0x0000000000b8ce0f in aac_decode_frame (avctx=0x203a4c0, frame=0x2ea2800, got_frame_ptr=0x7fffffffd3f4, avpkt=0x2ea2b40) at libavcodec/aacdec_template.c:3513
#5  0x00000000007fa202 in decode_simple_internal (discarded_samples=, frame=0x2ea2800, avctx=0x203a4c0) at libavcodec/decode.c:317
#6  decode_simple_receive_frame (frame=, avctx=) at libavcodec/decode.c:526
#7  decode_receive_frame_internal (avctx=avctx@entry=0x203a4c0, frame=0x2ea2800) at libavcodec/decode.c:550
#8  0x00000000007faa55 in avcodec_send_packet (avctx=avctx@entry=0x203a4c0, avpkt=avpkt@entry=0x20465c0) at libavcodec/decode.c:620
#9  0x00000000004af5a9 in decode (avctx=0x203a4c0, frame=0x206a200, got_frame=0x7fffffffd52c, pkt=0x20465c0) at fftools/ffmpeg.c:2116
#10 0x00000000004b383b in decode_audio (decode_failed=, got_output=0x7fffffffd52c, pkt=0x20465c0, ist=0x203a2c0) at fftools/ffmpeg.c:2160
#11 process_input_packet (ist=ist@entry=0x203a2c0, pkt=0x2134c40, no_eof=no_eof@entry=0) at fftools/ffmpeg.c:2488
#12 0x000000000049b4db in process_input (file_index=) at fftools/ffmpeg.c:4310
#13 transcode_step () at fftools/ffmpeg.c:4457
#14 transcode () at fftools/ffmpeg.c:4511
#15 main (argc=, argv=) at fftools/ffmpeg.c:4706

```

[回到顶部](#_labelTop)## AVFrame中时间戳的调整


音频包解码成AVFrame，在上一步初步设置了一下时间戳，接下来还需要进行调整，也就是要把时间基从1/1000转换成1/采样率，转换采样率的时候并不是直接计算，因为音频采样率有优点特殊，算出来每帧的时间间隔不能保证严格相等，比如，44100Hz，通常设置几帧23ms后会设置一帧24ms的帧进行补齐，会对这两情况分别进行处理：


* 如果duration是23ms，时间戳计算会使用filter\_in\_rescale\_delta\_last这个值，它是计算上一帧时间戳时估算出来的当前帧的时间戳值。
* 如果duration不是23ms，直接对上个步骤中得到的时间戳进行时间基的转换（向上取整）。



```
static int decode_video(InputStream *ist, AVPacket *pkt, int *got_output, int64_t *duration_pts, int eof,
                    int *decode_failed)
{
    if (pkt && pkt->duration && ist->prev_pkt_pts != AV_NOPTS_VALUE &&
    pkt->pts != AV_NOPTS_VALUE && pkt->pts - ist->prev_pkt_pts > pkt->duration)
        ist->filter_in_rescale_delta_last = AV_NOPTS_VALUE; //如果两个相邻帧的差值大于duration，则忽略last值，也就是上面说的碰到了24ms
    if (pkt)
        ist->prev_pkt_pts = pkt->pts;
    if (decoded_frame->pts != AV_NOPTS_VALUE)
        decoded_frame->pts = av_rescale_delta(decoded_frame_tb, decoded_frame->pts,
                                          (AVRational){1, avctx->sample_rate}, decoded_frame->nb_samples, &ist->filter_in_rescale_delta_last,
                                          (AVRational){1, avctx->sample_rate});//计算最终的时间戳。
}

int64_t av_rescale_delta(AVRational in_tb, int64_t in_ts,  AVRational fs_tb, int duration, int64_t *last, AVRational out_tb){
    int64_t a, b, this;

    av_assert0(in_ts != AV_NOPTS_VALUE);
    av_assert0(duration >= 0);


    if (*last == AV_NOPTS_VALUE || !duration || in_tb.num*(int64_t)out_tb.den <= out_tb.num*(int64_t)in_tb.den) {
 simple_round:
     
        *last = av_rescale_q(in_ts, in_tb, fs_tb) + duration;
        //如果忽略last值，直接进行时间基的转换，上边也把last计算出来了，也就是当前时间戳+duration，估算得到下一帧的时间戳。
        return av_rescale_q(in_ts, in_tb, out_tb);
    }
    //如果不忽略，则会计算两个边界值
    a =  av_rescale_q_rnd(2*in_ts-1, in_tb, fs_tb, AV_ROUND_DOWN)   >>1;
    b = (av_rescale_q_rnd(2*in_ts+1, in_tb, fs_tb, AV_ROUND_UP  )+1)>>1;
    if (*last < 2*a - b || *last > 2*b - a)
        goto simple_round;
    //如果在边界内，则使用last值，否则使用边界值。
    this = av_clip64(*last, a, b);
    //计算last值
    *last = this + duration;
    //时间基转换
    return av_rescale_q(this, fs_tb, out_tb);
}

```

调用stack:



```
#0  decode_audio (decode_failed=, got_output=0x7fffffffd52c, pkt=0x209a680, ist=0x2028c40) at fftools/ffmpeg.c:2207
#1  process_input_packet (ist=ist@entry=0x2028c40, pkt=0x209a440, no_eof=no_eof@entry=0) at fftools/ffmpeg.c:2488
#2  0x000000000049b4db in process_input (file_index=) at fftools/ffmpeg.c:4310
#3  transcode_step () at fftools/ffmpeg.c:4457
#4  transcode () at fftools/ffmpeg.c:4511
#5  main (argc=, argv=) at fftools/ffmpeg.c:4706

```

[回到顶部](#_labelTop)## 为重采样后的音频帧设置时间戳


解码完成后会把AVFrame发给filter进行重采样，重采样后设置时间戳会考虑其造成的delay，计算时间戳的时候会有两种时间补偿方式：


* 不使用自动时间戳补偿（min\_compensation \>\= FLT\_MAX）：在这种情况下，时间戳将被传递并通过延迟进行补偿。
* 使用自动时间戳补偿（min\_compensation \< FLT\_MAX）：在这种情况下，输出的时间戳将与输出的样本编号匹配。
在测试中没有使用自动时间补偿，也就是通过计算delay并减掉delay的方式进行补偿。


整个计算过程是先把音频帧的时间基变成 1/输入采样率\*输出采样率，然后把时间传递给swr\_next\_pts进行时间戳的计算，最后把时间基变成 1/输出采样率。（delay算法的原理还需要进一步研究。）



```
static int filter_frame(AVFilterLink *inlink, AVFrame *insamplesref)
{
    AResampleContext *aresample = inlink->dst->priv;
    const int n_in  = insamplesref->nb_samples;
    int64_t delay;
    int n_out       = n_in * aresample->ratio + 32;
    AVFilterLink *const outlink = inlink->dst->outputs[0];
    AVFrame *outsamplesref;
    int ret;

    delay = swr_get_delay(aresample->swr, outlink->sample_rate);
    if (delay > 0)
        n_out += FFMIN(delay, FFMAX(4096, n_out));

    outsamplesref = ff_get_audio_buffer(outlink, n_out);

    if(!outsamplesref) {
        av_frame_free(&insamplesref);
        return AVERROR(ENOMEM);
    }

    av_frame_copy_props(outsamplesref, insamplesref);
    outsamplesref->format                = outlink->format;
#if FF_API_OLD_CHANNEL_LAYOUT
FF_DISABLE_DEPRECATION_WARNINGS
    outsamplesref->channels              = outlink->ch_layout.nb_channels;
    outsamplesref->channel_layout        = outlink->channel_layout;
FF_ENABLE_DEPRECATION_WARNINGS
#endif
    ret = av_channel_layout_copy(&outsamplesref->ch_layout, &outlink->ch_layout);
    if (ret < 0)
        return ret;
    outsamplesref->sample_rate           = outlink->sample_rate;

    if(insamplesref->pts != AV_NOPTS_VALUE) {
        int64_t inpts = av_rescale(insamplesref->pts, inlink->time_base.num * (int64_t)outlink->sample_rate * inlink->sample_rate,     inlink->time_base.den); //变换时间基
        int64_t outpts= swr_next_pts(aresample->swr, inpts);//计算输出时间戳
        aresample->next_pts =
        outsamplesref->pts  = ROUNDED_DIV(outpts, inlink->sample_rate); //最后再把时间基变成1/输出采样率
    } else {
        outsamplesref->pts  = AV_NOPTS_VALUE;
    }
    n_out = swr_convert(aresample->swr, outsamplesref->extended_data, n_out,
                                 (void *)insamplesref->extended_data, n_in);
    if (n_out <= 0) {
        av_frame_free(&outsamplesref);
        av_frame_free(&insamplesref);
        return 0;
    }

    aresample->more_data = outsamplesref->nb_samples == n_out; // Indicate that there is probably more data in our buffers

    outsamplesref->nb_samples  = n_out;

    ret = ff_filter_frame(outlink, outsamplesref);
    av_frame_free(&insamplesref);
    return ret;
}

 int64_t swr_next_pts(struct SwrContext *s, int64_t pts){
   if(pts == INT64_MIN)
       return s->outpts;

   if (s->firstpts == AV_NOPTS_VALUE)
       s->outpts = s->firstpts = pts;
   
   //不使用时间自动补偿，计算delay并减掉
   if(s->min_compensation >= FLT_MAX) { 
       return (s->outpts = pts - swr_get_delay(s, s->in_sample_rate * (int64_t)s->out_sample_rate));
   } else { //使用时间自动补偿
       int64_t delta = pts - swr_get_delay(s, s->in_sample_rate * (int64_t)s->out_sample_rate) - s->outpts + s->drop_output*(int64_t)s->in_sample_rate;
       double fdelta = delta /(double)(s->in_sample_rate * (int64_t)s->out_sample_rate);

       if(fabs(fdelta) > s->min_compensation) {
           if(s->outpts == s->firstpts || fabs(fdelta) > s->min_hard_compensation){
               int ret;
               if(delta > 0) ret = swr_inject_silence(s,  delta / s->out_sample_rate);
               else          ret = swr_drop_output   (s, -delta / s-> in_sample_rate);
               if(ret<0){
                   av_log(s, AV_LOG_ERROR, "Failed to compensate for timestamp delta of %f\n", fdelta);
               }
           } else if(s->soft_compensation_duration && s->max_soft_compensation) {
               int duration = s->out_sample_rate * s->soft_compensation_duration;
               double max_soft_compensation = s->max_soft_compensation / (s->max_soft_compensation < 0 ? -s->in_sample_rate : 1);
               int comp = av_clipf(fdelta, -max_soft_compensation, max_soft_compensation) * duration ;
               av_log(s, AV_LOG_VERBOSE, "compensating audio timestamp drift:%f compensation:%d in:%d\n", fdelta, comp, duration);
               swr_set_compensation(s, comp, duration);
           }
       }

       return s->outpts;
   }
 }

```

调用stack:



```
#0  filter_frame (inlink=inlink@entry=0x20a2f80, insamplesref=0x20a2cc0) at libavfilter/af_aresample.c:209
#1  0x00000000004ce109 in ff_filter_frame_framed (frame=0x20a2cc0, link=0x20a2f80) at libavfilter/avfilter.c:990
#2  ff_filter_frame_to_filter (link=0x20a2f80) at libavfilter/avfilter.c:1138
#3  ff_filter_activate_default (filter=) at libavfilter/avfilter.c:1187
#4  ff_filter_activate (filter=) at libavfilter/avfilter.c:1345
#5  0x00000000004d01a2 in ff_filter_graph_run_once (graph=graph@entry=0x209e900) at libavfilter/avfiltergraph.c:1351
#6  0x00000000004d1362 in push_frame (graph=0x209e900) at libavfilter/buffersrc.c:169
#7  av_buffersrc_add_frame_flags (ctx=0x20a1d40, frame=frame@entry=0x203d600, flags=flags@entry=4) at libavfilter/buffersrc.c:252
#8  0x00000000004b35a5 in ifilter_send_frame (keep_reference=, frame=0x203d600, ifilter=0x206a080) at fftools/ffmpeg.c:2068
#9  send_frame_to_filters (ist=ist@entry=0x2150a40, decoded_frame=decoded_frame@entry=0x203d600) at fftools/ffmpeg.c:2138
#10 0x00000000004b4c63 in decode_audio (decode_failed=, got_output=0x7fffffffd52c, pkt=0x203d880, ist=0x2150a40) at fftools/    ffmpeg.c:2209
#11 process_input_packet (ist=ist@entry=0x2150a40, pkt=0x215a3c0, no_eof=no_eof@entry=0) at fftools/ffmpeg.c:2488
#12 0x000000000049b4db in process_input (file_index=) at fftools/ffmpeg.c:4310
#13 transcode_step () at fftools/ffmpeg.c:4457
#14 transcode () at fftools/ffmpeg.c:4511
#15 main (argc=, argv=) at fftools/ffmpeg.c:4706

```

重采样完成后，会把处理好的音频帧放到队列里面：



```
int ff_filter_frame(AVFilterLink *link, AVFrame *frame)
 {
     int ret;
     FF_TPRINTF_START(NULL, filter_frame); ff_tlog_link(NULL, link, 1); ff_tlog(NULL, " "); tlog_ref(NULL, frame, 1);
 
     /* Consistency checks */
     if (link->type == AVMEDIA_TYPE_VIDEO) {
         if (strcmp(link->dst->filter->name, "buffersink") &&
             strcmp(link->dst->filter->name, "format") &&
             strcmp(link->dst->filter->name, "idet") &&
             strcmp(link->dst->filter->name, "null") &&
             strcmp(link->dst->filter->name, "scale")) {
             av_assert1(frame->format                 == link->format);
             av_assert1(frame->width               == link->w);
             av_assert1(frame->height               == link->h);
         }
     } else {
         if (frame->format != link->format) {
             av_log(link->dst, AV_LOG_ERROR, "Format change is not supported\n");
             goto error;
         }
         if (av_channel_layout_compare(&frame->ch_layout, &link->ch_layout)) {
             av_log(link->dst, AV_LOG_ERROR, "Channel layout change is not supported\n");
             goto error;
         }
         if (frame->sample_rate != link->sample_rate) {
             av_log(link->dst, AV_LOG_ERROR, "Sample rate change is not supported\n");
             goto error;
         }
     }
 
     link->frame_blocked_in = link->frame_wanted_out = 0;
     link->frame_count_in++;
     link->sample_count_in += frame->nb_samples;
     filter_unblock(link->dst);
     ret = ff_framequeue_add(&link->fifo, frame);  //插入队列
     if (ret < 0) {
         av_frame_free(&frame);
         return ret;
     }
     ff_filter_set_ready(link->dst, 300);
     return 0;
 
 error:
     av_frame_free(&frame);
     return AVERROR_PATCHWELCOME;
 }

```

[回到顶部](#_labelTop)## 从音频帧队列中取出固定长度的采样数据，调整时间戳


接下来从队列中取出采样数据，通常是1024个采样点，需要拆开一帧凑齐1024个采样点的时候，这一帧的时间戳需要被调整:



```
static int take_samples(AVFilterLink *link, unsigned min, unsigned max,
                        AVFrame **rframe)
{
    AVFrame *frame0, *frame, *buf;
    unsigned nb_samples, nb_frames, i, p;
    int ret;

    /* Note: this function relies on no format changes and must only be
       called with enough samples. */
    av_assert1(samples_ready(link, link->min_samples));
    frame0 = frame = ff_framequeue_peek(&link->fifo, 0);
    if (!link->fifo.samples_skipped && frame->nb_samples >= min && frame->nb_samples <= max) {
        *rframe = ff_framequeue_take(&link->fifo);
        return 0;
    }
    nb_frames = 0;
    nb_samples = 0;
    while (1) {
        if (nb_samples + frame->nb_samples > max) {
            if (nb_samples < min)
                nb_samples = max;
            break;
        }
        nb_samples += frame->nb_samples;
        nb_frames++;
        if (nb_frames == ff_framequeue_queued_frames(&link->fifo))
            break;
        frame = ff_framequeue_peek(&link->fifo, nb_frames);
    }

    buf = ff_get_audio_buffer(link, nb_samples);
    if (!buf)
        return AVERROR(ENOMEM);
    ret = av_frame_copy_props(buf, frame0);
    if (ret < 0) {
        av_frame_free(&buf);
        return ret;
    }

    p = 0;
    //先读取完整帧采样数据
    for (i = 0; i < nb_frames; i++) {
        frame = ff_framequeue_take(&link->fifo);
        av_samples_copy(buf->extended_data, frame->extended_data, p, 0,
                        frame->nb_samples, link->ch_layout.nb_channels, link->format);
        p += frame->nb_samples;
        av_frame_free(&frame);
    }
    //读完之后没有凑够，需要peek下一帧，取出n个采样点，然后调用ff_framequeue_skip_samples跳过这n个采样点
    if (p < nb_samples) {
        unsigned n = nb_samples - p;
        frame = ff_framequeue_peek(&link->fifo, 0);
        av_samples_copy(buf->extended_data, frame->extended_data, p, 0, n,
                        link->ch_layout.nb_channels, link->format);
        ff_framequeue_skip_samples(&link->fifo, n, link->time_base);
    }

    *rframe = buf;
    return 0;
}


void ff_framequeue_skip_samples(FFFrameQueue *fq, size_t samples, AVRational time_base)
{
    FFFrameBucket *b;
    size_t bytes;
    int planar, planes, i;

    check_consistency(fq);
    av_assert1(fq->queued);
    b = bucket(fq, 0);
    av_assert1(samples < b->frame->nb_samples);
    planar = av_sample_fmt_is_planar(b->frame->format);
    planes = planar ? b->frame->ch_layout.nb_channels : 1;
    bytes = samples * av_get_bytes_per_sample(b->frame->format);
    if (!planar)
        bytes *= b->frame->ch_layout.nb_channels;
    if (b->frame->pts != AV_NOPTS_VALUE)
        b->frame->pts += av_rescale_q(samples, av_make_q(1, b->frame->sample_rate), time_base);//跳过这些sample的时候时间戳也需要调整。
    b->frame->nb_samples -= samples;
    b->frame->linesize[0] -= bytes;
    for (i = 0; i < planes; i++)
        b->frame->extended_data[i] += bytes;
    for (i = 0; i < planes && i < AV_NUM_DATA_POINTERS; i++)
        b->frame->data[i] = b->frame->extended_data[i];
    fq->total_samples_tail += samples;
    fq->samples_skipped = 1;
    ff_framequeue_update_peeked(fq, 0);
}

```

调用stack:



```
#0  take_samples (rframe=, max=1024, min=1024, link=0x20b1300) at libavfilter/avfilter.c:1058
#1  ff_inlink_consume_samples (link=link@entry=0x20b1300, min=min@entry=1024, max=max@entry=1024, rframe=rframe@entry=0x7fffffffd548) at libavfilter/    avfilter.c:1437
#2  0x00000000004d08d5 in get_frame_internal (ctx=ctx@entry=0x20b0440, frame=frame@entry=0x2035100, flags=flags@entry=2, samples=1024) at libavfilter/    buffersink.c:135
#3  0x00000000004d0a2c in av_buffersink_get_frame_flags (ctx=ctx@entry=0x20b0440, frame=frame@entry=0x2035100, flags=flags@entry=2) at libavfilter/    buffersink.c:172
#4  0x00000000004b3134 in reap_filters (flush=flush@entry=0) at fftools/ffmpeg.c:1399
#5  0x000000000049b54b in transcode_step () at fftools/ffmpeg.c:4467
#6  transcode () at fftools/ffmpeg.c:4511
#7  main (argc=, argv=) at fftools/ffmpeg.c:4706

```

[回到顶部](#_labelTop)## 编码音频帧设置时间戳


冲采样完成后，调用do\_audio\_out进行编码，这里时间戳的设置用到一个队列，来取出正确的编码帧时间戳。


根据编码器的不同需要设置一个delay参数，也就是编码器需要delay时间才能出帧，libfdk\-aac这个编码器的时间是2048个采样点，要把第一帧的时间戳减掉这个delay，前边说过第一帧的时间戳为0，所以最后变为\-2048，加入队列。接下来加入队列的时间戳都是重采样后的时间戳：1024 2048 3072 4088\...， 所以这里有个问题：\-2048一下就跳到了1024，这中间需要插入2个时间戳来保证时间戳的平滑递增，这个逻辑在ff\_af\_queue\_remove中实现：



```
void ff_af_queue_remove(AudioFrameQueue *afq, int nb_samples, int64_t *pts,
                        int64_t *duration)
{
    int64_t out_pts = AV_NOPTS_VALUE;
    int removed_samples = 0;
    int i;

    if (afq->frame_count || afq->frame_alloc) {
        if (afq->frames->pts != AV_NOPTS_VALUE)
            out_pts = afq->frames->pts;
    }
    if(!afq->frame_count)
        av_log(afq->avctx, AV_LOG_WARNING, "Trying to remove %d samples, but the queue is empty\n", nb_samples);
    if (pts)
        *pts = ff_samples_to_time_base(afq->avctx, out_pts);

    for(i=0; nb_samples && i->frame_count; i++){
        int n= FFMIN(afq->frames[i].duration, nb_samples);
        afq->frames[i].duration -= n;
        nb_samples              -= n;
        removed_samples         += n;
        if(afq->frames[i].pts != AV_NOPTS_VALUE)
            afq->frames[i].pts      += n;
    }
    afq->remaining_samples -= removed_samples;
    i -= i && afq->frames[i-1].duration; //填入的两个时间戳不是从队列里面取出来的，这里的i为0
    //memmove后用的还是上一帧的数据，但是时间戳在下面的逻辑中更新
    memmove(afq->frames, afq->frames + i, sizeof(*afq->frames) * (afq->frame_count - i));
    afq->frame_count -= i;

    if(nb_samples){
        av_assert0(!afq->frame_count);
        av_assert0(afq->remaining_samples == afq->remaining_delay);
        if(afq->frames && afq->frames[0].pts != AV_NOPTS_VALUE)
            //在这里更新时间戳 （第一次-1024 第二次为0）
            afq->frames[0].pts += nb_samples;
        av_log(afq->avctx, AV_LOG_DEBUG, "Trying to remove %d more samples than there are in the queue\n", nb_samples);
    }
    if (duration)
        *duration = ff_samples_to_time_base(afq->avctx, removed_samples);
}

```

调用stack



```
#0  ff_af_queue_add (afq=afq@entry=0x2038e30, f=f@entry=0x20c85c0) at libavcodec/audio_frame_queue.c:49
#1  0x00000000008bf159 in aac_encode_frame (avctx=0x202d4c0, avpkt=0x20a7f00, frame=0x20c85c0, got_packet_ptr=0x7fffffffd2ac) at libavcodec/    libfdk-aacenc.c:387
#2  0x0000000000820eac in encode_simple_internal (avpkt=0x20a7f00, avctx=0x202d4c0) at libavcodec/encode.c:234
#3  encode_simple_receive_packet (avpkt=, avctx=) at libavcodec/encode.c:295
#4  encode_receive_packet_internal (avctx=avctx@entry=0x202d4c0, avpkt=0x20a7f00) at libavcodec/encode.c:348
#5  0x000000000082138a in avcodec_send_frame (avctx=avctx@entry=0x202d4c0, frame=frame@entry=0x204b380) at libavcodec/encode.c:444
#6  0x00000000004af758 in encode_frame (of=of@entry=0x215e580, ost=ost@entry=0x2038ac0, frame=frame@entry=0x204b380) at fftools/ffmpeg.c:922
#7  0x00000000004b32f6 in do_audio_out (frame=0x204b380, ost=0x2038ac0, of=0x215e580) at fftools/ffmpeg.c:1038
#8  reap_filters (flush=flush@entry=0) at fftools/ffmpeg.c:1436
#9  0x000000000049b54b in transcode_step () at fftools/ffmpeg.c:4467
#10 transcode () at fftools/ffmpeg.c:4511
#11 main (argc=, argv=) at fftools/ffmpeg.c:4706


#0  ff_af_queue_remove (afq=0x21700f0, nb_samples=1024, pts=0x2075188, duration=0x20751c0) at libavcodec/audio_frame_queue.c:99
#1  0x00000000008bf2cd in aac_encode_frame (avctx=0x206cc40, avpkt=0x2075180, frame=0x209b000, got_packet_ptr=0x7fffffffd2ac) at libavcodec/    libfdk-aacenc.c:436
#2  0x0000000000820eac in encode_simple_internal (avpkt=0x2075180, avctx=0x206cc40) at libavcodec/encode.c:234
#3  encode_simple_receive_packet (avpkt=, avctx=) at libavcodec/encode.c:295
#4  encode_receive_packet_internal (avctx=avctx@entry=0x206cc40, avpkt=0x2075180) at libavcodec/encode.c:348
#5  0x000000000082138a in avcodec_send_frame (avctx=avctx@entry=0x206cc40, frame=frame@entry=0x2065ec0) at libavcodec/encode.c:444
#6  0x00000000004af758 in encode_frame (of=of@entry=0x20378c0, ost=ost@entry=0x203c040, frame=frame@entry=0x2065ec0) at fftools/ffmpeg.c:922
#7  0x00000000004b32f6 in do_audio_out (frame=0x2065ec0, ost=0x203c040, of=0x20378c0) at fftools/ffmpeg.c:1038
#8  reap_filters (flush=flush@entry=0) at fftools/ffmpeg.c:1436
#9  0x000000000049b54b in transcode_step () at fftools/ffmpeg.c:4467
#10 transcode () at fftools/ffmpeg.c:4511
#11 main (argc=, argv=) at fftools/ffmpeg.c:4706

```

[回到顶部](#_labelTop)## 为最终的编码帧设置时间戳


上个步骤编码完成后的时间戳的时间基还是1/输出采样率，最终的时间戳的时间基为1/1000，需要做一下转换：



```
void of_write_packet(OutputFile *of, AVPacket *pkt, OutputStream *ost,
                 int unqueue)
{

...
    av_packet_rescale_ts(pkt, ost->mux_timebase, ost->st->time_base); //ost->mux_timebase为1/48000 ost->st->time_base为 1/1000
...
}

```

调用stack



```
#0  of_write_packet (of=of@entry=0x215f000, pkt=0x2046a40, ost=0x2046600, unqueue=unqueue@entry=0) at fftools/ffmpeg_mux.c:140
#1  0x00000000004af18b in output_packet (of=of@entry=0x215f000, pkt=pkt@entry=0x2046a40, ost=ost@entry=0x2046600, eof=eof@entry=0) at fftools/    ffmpeg.c:740
#2  0x00000000004afe53 in encode_frame (of=of@entry=0x215f000, ost=ost@entry=0x2046600, frame=frame@entry=0x2071a80) at fftools/ffmpeg.c:1017
#3  0x00000000004b32f6 in do_audio_out (frame=0x2071a80, ost=0x2046600, of=0x215f000) at fftools/ffmpeg.c:1038
#4  reap_filters (flush=flush@entry=0) at fftools/ffmpeg.c:1436
#5  0x000000000049b54b in transcode_step () at fftools/ffmpeg.c:4467
#6  transcode () at fftools/ffmpeg.c:4511
#7  main (argc=, argv=) at fftools/ffmpeg.c:4706

```

AVPacket的时间戳设置完成后，可以封装并发送出去了。


# 视频时间戳设置


[回到顶部](#_labelTop)## 视频帧解码后的pts时间戳设置


可以知道视频时间戳解封装后，也需要减掉ts\-\>offset，而ts\-\>offset取得是第一个音频和视频帧中较小的值，音频帧的时间戳值较小，故第一个视频帧的值会大于0。


解码成AVFrame后会发送给filter。


stack:



```
#0  sch_dec_send (sch=0x2f91700, dec_idx=1, frame=frame@entry=0x7fffc80008c0) at fftools/ffmpeg_sched.c:2169
#1  0x000000000049e0c9 in packet_decode (frame=0x7fffc80008c0, pkt=0x7fffc8000b40, dp=0x3faa740) at fftools/ffmpeg_dec.c:760
#2  decoder_thread (arg=0x3faa740) at fftools/ffmpeg_dec.c:889
#3  0x00000000004b85e9 in task_wrapper (arg=0x3faac68) at fftools/ffmpeg_sched.c:2447
#4  0x00007ffff60b4ea5 in start_thread () from /usr/lib64/libpthread.so.0
#5  0x00007ffff4dedb0d in clone () from /usr/lib64/libc.so.6

```

[回到顶部](#_labelTop)## 在filter中对时间戳进行设置


在filter中会对时间戳进行时间基的调整，从1/1000调整为1/out\_fps，也就是每输出一帧，时间戳加1。还有需要注意的点是在这一步会进行视频帧的特殊处理，相关的参数叫做vsync，总共有几种处理的方式，这里详细说一下VSYNC\_CFR这个参数的作用：在输出帧率和输入帧率比发生变化的时候，filter会拷贝帧或者丢弃帧来保证输出帧率的稳定。



```
static void video_sync_process(OutputFilterPriv *ofp, AVFrame *frame,
                               int64_t *nb_frames, int64_t *nb_frames_prev)
{

    //先计算当前帧在输出流中需要持续的时间，比如源流是20FPS，输出流是30FPS,那么源流中的一帧在输出流中要持续1.5帧的时间（一帧为50ms,每一帧在输出流中的展示时间变成75ms,这样就需要拷贝帧来凑足帧率）
    duration = frame->duration * av_q2d(frame->time_base) / av_q2d(ofp->tb_out);
    //把当前帧的时间戳转换成输出流的时间戳（输出流的时间基为1/out_fps）
    sync_ipts = adjust_frame_pts_to_encoder_tb(frame, ofp->tb_out, ofp->ts_offset);
    /* delta0 is the "drift" between the input frame and
     * where it would fall in the output. */
    //把当前帧的时间戳和输出帧下一个时间戳做对比。(可以这么理解，当前帧的时间戳经过转换后发现，超过了输出流下一帧的时间戳，
    //则应该复制一帧，补齐帧率（输出帧的时间戳需要赶上当前输出帧的时间戳），相反，如果当前帧的时间远远没到下一帧的时间戳，
    //则可以丢弃帧来等待输入帧的时间戳赶上来)
    delta0 = sync_ipts - ofp->next_pts;
    delta  = delta0 + duration;

    // tracks the number of times the PREVIOUS frame should be duplicated,
    // mostly for variable framerate (VFR)
    *nb_frames_prev = 0;
    /* by default, we output a single frame */
    *nb_frames = 1;

    if (delta0 < 0 &&
        delta > 0 &&
        ost->vsync_method != VSYNC_PASSTHROUGH
#if FFMPEG_OPT_VSYNC_DROP
        && ost->vsync_method != VSYNC_DROP
#endif
        ) {
        if (delta0 < -0.6) {
            av_log(ost, AV_LOG_VERBOSE, "Past duration %f too large\n", -delta0);
        } else
            av_log(ost, AV_LOG_DEBUG, "Clipping frame in rate conversion by %f\n", -delta0);
        sync_ipts = ofp->next_pts;
        duration += delta0;
        delta0 = 0;
    }

    switch (ost->vsync_method) {
    case VSYNC_VSCFR:
        if (fps->frame_number == 0 && delta0 >= 0.5) {
            av_log(ost, AV_LOG_DEBUG, "Not duplicating %d initial frames\n", (int)lrintf(delta0));
            delta = duration;
            delta0 = 0;
            ofp->next_pts = llrint(sync_ipts);
        }
    case VSYNC_CFR:
        // FIXME set to 0.5 after we fix some dts/pts bugs like in avidec.c
        if (frame_drop_threshold && delta < frame_drop_threshold && fps->frame_number) {
            *nb_frames = 0;
        } else if (delta < -1.1)
            *nb_frames = 0;
        else if (delta > 1.1) {
            *nb_frames = llrintf(delta);//四舍五入，delta>=1.5 则输出两帧。
            if (delta0 > 1.1)
                *nb_frames_prev = llrintf(delta0 - 0.6);
        }
        frame->duration = 1;
        break;
    case VSYNC_VFR:
        if (delta <= -0.6)
            *nb_frames = 0;
        else if (delta > 0.6)
            ofp->next_pts = llrint(sync_ipts);
        frame->duration = llrint(duration);
        break;
#if FFMPEG_OPT_VSYNC_DROP
    case VSYNC_DROP:
#endif
    case VSYNC_PASSTHROUGH:
        ofp->next_pts = llrint(sync_ipts);
        frame->duration = llrint(duration);
        break;
    default:
        av_assert0(0);
    }

finish:
    memmove(fps->frames_prev_hist + 1,
            fps->frames_prev_hist,
            sizeof(fps->frames_prev_hist[0]) * (FF_ARRAY_ELEMS(fps->frames_prev_hist) - 1));
    fps->frames_prev_hist[0] = *nb_frames_prev;

    if (*nb_frames_prev == 0 && fps->last_dropped) {
        atomic_fetch_add(&ofilter->nb_frames_drop, 1);
        av_log(ost, AV_LOG_VERBOSE,
               "*** dropping frame %"PRId64" at ts %"PRId64"\n",
               fps->frame_number, fps->last_frame->pts);
    }
    if (*nb_frames > (*nb_frames_prev && fps->last_dropped) + (*nb_frames > *nb_frames_prev)) {
        uint64_t nb_frames_dup;
        if (*nb_frames > dts_error_threshold * 30) {
            av_log(ost, AV_LOG_ERROR, "%"PRId64" frame duplication too large, skipping\n", *nb_frames - 1);
            atomic_fetch_add(&ofilter->nb_frames_drop, 1);
            *nb_frames = 0;
            return;
        }
        nb_frames_dup = atomic_fetch_add(&ofilter->nb_frames_dup,
                                         *nb_frames - (*nb_frames_prev && fps->last_dropped) - (*nb_frames > *nb_frames_prev));
        av_log(ost, AV_LOG_VERBOSE, "*** %"PRId64" dup!\n", *nb_frames - 1);
        if (nb_frames_dup > fps->dup_warning) {
            av_log(ost, AV_LOG_WARNING, "More than %"PRIu64" frames duplicated\n", fps->dup_warning);
            fps->dup_warning *= 10;
        }
    }

    fps->last_dropped = *nb_frames == *nb_frames_prev && frame;
    fps->dropped_keyframe |= fps->last_dropped && (frame->flags & AV_FRAME_FLAG_KEY);
}




static double adjust_frame_pts_to_encoder_tb(AVFrame *frame, AVRational tb_dst,
                                             int64_t start_time)
{
    double float_pts = AV_NOPTS_VALUE; // this is identical to frame.pts but with higher precision

    AVRational        tb = tb_dst;
    AVRational filter_tb = frame->time_base;
    const int extra_bits = av_clip(29 - av_log2(tb.den), 0, 16);

    if (frame->pts == AV_NOPTS_VALUE)
        goto early_exit;
    //为了提高时间戳的精度，把输出时间戳的时间基缩小2的extra_bits次方倍,输出的时间戳就会增大2的extra_bits次方倍（av_rescale_q的输出是int64_t）。
    tb.den <<= extra_bits;
    // 把当前帧时间戳（时间基为1/1000）按照输出流时间基进行转换。
    float_pts = av_rescale_q(frame->pts, filter_tb, tb) -
                av_rescale_q(start_time, AV_TIME_BASE_Q, tb);
    //计算完成后得到整数再缩小2的extra_bits次方倍得到浮点型的值。
    float_pts /= 1 << extra_bits;
    // when float_pts is not exactly an integer,
    // avoid exact midpoints to reduce the chance of rounding differences, this
    // can be removed in case the fps code is changed to work with integers
    if (float_pts != llrint(float_pts))
        float_pts += FFSIGN(float_pts) * 1.0 / (1<<17);

    frame->pts = av_rescale_q(frame->pts, filter_tb, tb_dst) -
                 av_rescale_q(start_time, AV_TIME_BASE_Q, tb_dst);
    frame->time_base = tb_dst;

early_exit:

    if (debug_ts) {
        av_log(NULL, AV_LOG_INFO, "filter -> pts:%s pts_time:%s exact:%f time_base:%d/%d\n",
               frame ? av_ts2str(frame->pts) : "NULL",
               av_ts2timestr(frame->pts, &tb_dst),
               float_pts, tb_dst.num, tb_dst.den);
    }

    return float_pts;
}

```

Call stack:



```
#0  video_sync_process (nb_frames_prev=, nb_frames=, frame=0x7fffdc0008c0, ofp=0x311cc00) at fftools/    ffmpeg_filter.c:2068
#1  fg_output_frame (ofp=ofp@entry=0x311cc00, fgt=fgt@entry=0x7fffe6198810, frame=frame@entry=0x7fffdc0008c0) at fftools/ffmpeg_filter.c:2230
#2  0x00000000004a84c3 in fg_output_step (frame=0x7fffdc0008c0, fgt=0x7fffe6198810, ofp=0x311cc00) at fftools/ffmpeg_filter.c:2371
#3  read_frames (fg=fg@entry=0x2fa6e80, fgt=fgt@entry=0x7fffe6198810, frame=0x7fffdc0008c0) at fftools/ffmpeg_filter.c:2432
#4  0x00000000004a873d in filter_thread (arg=0x2fa6e80) at fftools/ffmpeg_filter.c:2846
#5  0x00000000004b85e9 in task_wrapper (arg=0x3fb5370) at fftools/ffmpeg_sched.c:2447
#6  0x00007ffff60b4ea5 in start_thread () from /usr/lib64/libpthread.so.0
#7  0x00007ffff4dedb0d in clone () from /usr/lib64/libc.so.6

```

在filter里面把时间戳的时间基修改好后，可以把pts传递给编码器进行编码了。



```
#0  sch_filter_send (sch=0x2f92700, fg_idx=1, out_idx=0, frame=frame@entry=0x7fffd00008c0) at fftools/ffmpeg_sched.c:2390
#1  0x00000000004a7c8a in fg_output_frame (ofp=ofp@entry=0x3f7e840, fgt=fgt@entry=0x7fffe595e810, frame=frame@entry=0x7fffd00008c0) at fftools/    ffmpeg_filter.c:2269
#2  0x00000000004a84c3 in fg_output_step (frame=0x7fffd00008c0, fgt=0x7fffe595e810, ofp=0x3f7e840) at fftools/ffmpeg_filter.c:2371
#3  read_frames (fg=fg@entry=0x3f7dd40, fgt=fgt@entry=0x7fffe595e810, frame=0x7fffd00008c0) at fftools/ffmpeg_filter.c:2432
#4  0x00000000004a873d in filter_thread (arg=0x3f7dd40) at fftools/ffmpeg_filter.c:2846
#5  0x00000000004b85e9 in task_wrapper (arg=0x3f7eae0) at fftools/ffmpeg_sched.c:2453
#6  0x00007ffff60b4ea5 in start_thread () from /usr/lib64/libpthread.so.0
#7  0x00007ffff4dedb0d in clone () from /usr/lib64/libc.so.6

```

[回到顶部](#_labelTop)## 在编码器中设置时间戳


在编码器中设置的时间戳，时间基还是使用1/output\_fps，dts每出一帧增加1，如果有B帧，编码器会帮你设置正确的pts。



```
static int X264_frame(AVCodecContext *ctx, AVPacket *pkt, const AVFrame *frame,
                      int *got_packet)
{
    ...
    pkt->pts = pic_out.i_pts;
    pkt->dts = pic_out.i_dts;
    ...
   
    return 0;
}

```

call stack:



```
#0  X264_frame (ctx=0x2fbc140, pkt=0x7fffdc012780, frame=0x7fffdc014b00, got_packet=0x7fffe71619bc) at libavcodec/libx264.c:677
#1  0x00000000008737ae in ff_encode_encode_cb (avctx=0x2fbc140, avpkt=0x7fffdc012780, frame=0x7fffdc014b00, got_packet=0x7fffe71619bc) at libavcodec/    encode.c:253
#2  0x0000000000873b0c in encode_simple_internal (avpkt=0x7fffdc012780, avctx=0x2fbc140) at libavcodec/encode.c:339
#3  encode_simple_receive_packet (avpkt=, avctx=) at libavcodec/encode.c:353
#4  encode_receive_packet_internal (avctx=avctx@entry=0x2fbc140, avpkt=0x7fffdc012780) at libavcodec/encode.c:387
#5  0x0000000000873d98 in avcodec_send_frame (avctx=avctx@entry=0x2fbc140, frame=frame@entry=0x7fffe00008c0) at libavcodec/encode.c:530
#6  0x00000000004a49af in encode_frame (of=0x2fbf4c0, pkt=0x7fffe0000b40, frame=0x7fffe00008c0, ost=0x2fa0e40) at fftools/ffmpeg_enc.c:675
#7  frame_encode (ost=ost@entry=0x2fa0e40, frame=0x7fffe00008c0, pkt=0x7fffe0000b40) at fftools/ffmpeg_enc.c:843
#8  0x00000000004a5412 in encoder_thread (arg=0x2fa0e40) at fftools/ffmpeg_enc.c:929
#9  0x00000000004b85e7 in task_wrapper (arg=0x3f83308) at fftools/ffmpeg_sched.c:2447
#10 0x00007ffff60b4ea5 in start_thread () from /usr/lib64/libpthread.so.0
#11 0x00007ffff4dedb0d in clone () from /usr/lib64/libc.so.6

```

[回到顶部](#_labelTop)## 在封装线程中把时间基修改成输出流协议的时间基



```
static int write_packet(Muxer *mux, OutputStream *ost, AVPacket *pkt)
{
    MuxStream *ms = ms_from_ost(ost);
    AVFormatContext *s = mux->fc;
    int64_t fs;
    uint64_t frame_num;
    int ret;

    fs = filesize(s->pb);
    atomic_store(&mux->last_filesize, fs);
    if (fs >= mux->limit_filesize) {
        ret = AVERROR_EOF;
        goto fail;
    }
    //下面的函数中进行时间基的转换，这里是1/out_fps 转成 1/1000
    ret = mux_fixup_ts(mux, ms, pkt);
    if (ret < 0)
        goto fail;

    ms->data_size_mux += pkt->size;
    frame_num = atomic_fetch_add(&ost->packets_written, 1);

    pkt->stream_index = ost->index;

    if (ms->stats.io)
        enc_stats_write(ost, &ms->stats, NULL, pkt, frame_num);

    ret = av_interleaved_write_frame(s, pkt);
    if (ret < 0) {
        av_log(ost, AV_LOG_ERROR,
               "Error submitting a packet to the muxer: %s\n",
               av_err2str(ret));
        goto fail;
    }

    return 0;
fail:
    av_packet_unref(pkt);
    return ret;
}




#0  write_packet (mux=out>, ost=0x3f8f640, pkt=0x7fff480008c0) at fftools/ffmpeg_mux.c:228
#1  0x00000000004aadf3 in mux_packet_filter (mt=out>, stream_eof=out>, pkt=out>, ost=out>, mux=out>) at fftools/ffmpeg_mux.c:357
#2  muxer_thread (arg=0x30a4000) at fftools/ffmpeg_mux.c:438
#3  0x00000000004b858d in task_wrapper (arg=0x2fb7760) at fftools/ffmpeg_sched.c:2447
#4  0x00007ffff60b4ea5 in start_thread () from /usr/lib64/libpthread.so.0
#5  0x00007ffff4dedb0d in clone () from /usr/lib64/libc.so.6

```

