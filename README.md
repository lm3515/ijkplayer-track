# ijkplayer-track
通过ijkplayer实现切换音轨，适用于ios、android

4个文件为核心代码在ijkmedia/ijkplayer中实现

首先在ijkplayer.h中添加两个方法：

/* 获取音轨信息 */

int ijkmp_get_audio_track(IjkMediaPlayer *mp);

void ijkmp_switch_audio_track(IjkMediaPlayer *mp, int tracksNum, int index);

ijkplayer.c方法实现：
/*  -- 音轨信息----  */

int ijkmp_get_audio_track(IjkMediaPlayer *mp)
{
    assert(mp);
    pthread_mutex_lock(&mp->mutex);
    int ret = ffp_get_track_info_l(mp->ffplayer);
    pthread_mutex_unlock(&mp->mutex);
    
    return ret;
}

// 切换音轨
void ijkmp_switch_audio_track(IjkMediaPlayer *mp, int tracksNum, int index)
{
    assert(mp);
    pthread_mutex_lock(&mp->mutex);
    ffp_select_track_l(mp->ffplayer, tracksNum, index);
    pthread_mutex_unlock(&mp->mutex);
}


ff_ffplay.h中添加两个方法：

// 获取音轨信息
int ffp_get_track_info_l(FFPlayer *ffp);
void ffp_select_track_l(FFPlayer *ffp, int tracksNum, int index);


ff_ffplay.c实现：
/* 音轨信息 */
//获取音轨信息


```C
int ffp_get_track_info_l(FFPlayer *ffp)
{
    if (!ffp)
        return 0;
    
    assert(ffp);
    VideoState *is = ffp->is;
    int total = 0;
    if (!is)
        return EIJK_NULL_IS_PTR;
    
    AVFormatContext *ic = is->ic;
    int stream_index;
    AVStream *st;
    int codec_type = AVMEDIA_TYPE_AUDIO;
    
    for (stream_index = 0; stream_index < is->ic->nb_streams; stream_index++)
    {
        st = ic->streams[stream_index];
        if (st->codecpar->codec_type == codec_type) {
            /* check that parameters are OK */
            switch(codec_type) {
                case AVMEDIA_TYPE_AUDIO:
                    if (st->codecpar->sample_rate != 0 && st->codecpar->channels != 0)
                        total++;
                    break;
            }
        }
    }
    
    return total;
}


void ffp_select_track_l(FFPlayer *ffp, int tracksNum, int index) {
    if (!ffp)
        return;
    
    assert(ffp);
    VideoState *is = ffp->is;
    int total = 0;
    if (!is)
        return;
    
    AVFormatContext *ic = is->ic;
    int start_index = 0, stream_index = 0;
    AVStream *st;
    int codec_type = AVMEDIA_TYPE_AUDIO;
    
    if (codec_type == AVMEDIA_TYPE_VIDEO)
        start_index = is->video_stream;
    else if (codec_type == AVMEDIA_TYPE_AUDIO)
        start_index = is->audio_stream;
    /*else
     start_index = is->subtitle_stream;
     if (start_index < (codec_type == AVMEDIA_TYPE_SUBTITLE ? -1 : 0))
     return;*/
    
    for (stream_index = 0; stream_index < is->ic->nb_streams; stream_index++)
    {
        st = ic->streams[stream_index];
        if (st->codecpar->codec_type == codec_type) {
            /* check that parameters are OK */
            switch (codec_type) {
                case AVMEDIA_TYPE_AUDIO:
                    if (st->codecpar->sample_rate != 0 && st->codecpar->channels != 0)
                        total++;
                    if (total == index)
                        //跳出循环
                        goto the_end;
                    break;
            }
        }
    }
    
    return;
    
the_end:
    stream_component_close(ffp, start_index);
    //传递两个参数tracksNum：音轨的个数，stream_index：第几个音轨
    ffp_set_stream_selected(ffp, tracksNum, stream_index);
}

```
