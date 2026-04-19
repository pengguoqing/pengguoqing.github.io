+++
title = "理解OBS帧时间戳的计算与渲染同步方式"
date = 2026-04-19
draft = false
categories = ["OBS源码"]
tags = ["音视频时间戳"]
+++

# 背景
我在看OBS文件解码模块源码的时候，就感觉很奇怪为啥它每帧的时间戳要用好几个部分组成。之前我在看 ffplay 源码的时候，发现ffplay只需要用到每帧的pts就可以了。带着这种疑惑，我把它视频渲染里面 `tick_source` 流程的源码也研究一遍。经过两三天的思考，终于搞懂了OBS这么设计每帧时间戳的原因。

# 先理解帧时间戳每部分的含义
```cpp
	frame->timestamp = m->base_ts + d->frame_pts - m->start_ts +
			   m->play_sys_ts - base_sys_ts;
        
    //m->base_ts, d->frame_pts, m->start_ts 三个都是素材相关的时间
    //m->play_sys_ts, base_sys_ts 两者是系统时间
```
- `m->base_ts`:当前累计播放的时间，每次开始播放时都需要刷新。比如说seek到100帧的位置开始播放，那么`base_ts`的前100帧的总显示时间。
- `d->frame_pts`:当前帧的显示时间戳，由素材本身决定，用ffmpeg解码出来是多少就是多少。`ffplay`就用的这个。
- `m->start_ts`:素材开始播放的时间戳，同样由素材本身决定。
- `m->play_sys_ts`:每次开始播放时的系统时间，通过 `os_gettime_ns()` 来刷新。
- `base_sys_ts`:整个系统第一播放素材时的系统时间，后续所有素材都共享这个 `staic` 全局变量。

# 为什么要由这么多部分一起计算得到每帧的时间戳
我最开始想的是由 `frame_pts` 就够了，最多再要个 `start_ts`。于是我去读了`tick_source`里面同步视频帧的源码，我对相关部分加了注释
结论是**OBS要求播放时间是累计向前的**
```cpp
	struct obs_source_frame *next_frame = source->async_frames.array[0];
	struct obs_source_frame *frame = NULL;
	uint64_t sys_offset = sys_time - source->last_sys_timestamp;
	uint64_t frame_time = next_frame->timestamp;
	uint64_t frame_offset = 0;

	if (source->async_unbuffered) {
		while (source->async_frames.num > 1) {
			da_erase(source->async_frames, 0);
			remove_async_frame(source, next_frame);
			next_frame = source->async_frames.array[0];
		}

		source->last_frame_ts = next_frame->timestamp;
		return true;
	}

#if DEBUG_ASYNC_FRAMES
	blog(LOG_DEBUG,
	     "source->last_frame_ts: %llu, frame_time: %llu, "
	     "sys_offset: %llu, frame_offset: %llu, "
	     "number of frames: %lu",
	     source->last_frame_ts, frame_time, sys_offset,
	     frame_time - source->last_frame_ts,
	     (unsigned long)source->async_frames.num);
#endif
	//超过两秒就算是 out 了
	if (frame_out_of_bounds(source, frame_time)) {
#if DEBUG_ASYNC_FRAMES
		blog(LOG_DEBUG, "timing jump");
#endif
		source->last_frame_ts = next_frame->timestamp;
		return true;
	} else {
		frame_offset = frame_time - source->last_frame_ts;

        //这里很重要，因为它告诉我们，渲染线程里面每帧的时间戳由渲染线程驱动
        //和media_source里面帧的时间戳已经没有关系了，除非发生了跳跃
		source->last_frame_ts += sys_offset;
	}

    //判断是否渲染超前了，是的话，就要丢掉落后的帧
	while (source->last_frame_ts > next_frame->timestamp) {

        //为了保证平滑，2ms误差以内的都接受
		if ((source->last_frame_ts - next_frame->timestamp) < 2000000)
			break;

		if (frame)
			da_erase(source->async_frames, 0);

#if DEBUG_ASYNC_FRAMES
		blog(LOG_DEBUG,
		     "new frame, "
		     "source->last_frame_ts: %llu, "
		     "next_frame->timestamp: %llu",
		     source->last_frame_ts, next_frame->timestamp);
#endif

		remove_async_frame(source, frame);

		if (source->async_frames.num == 1)
			return true;

		frame = next_frame;
		next_frame = source->async_frames.array[1];

		//解码那边发生了2秒以上时间戳跳跃
        //那么source->last_frame_ts 就刷成离跳跃后最近的那个时间戳
		if ((next_frame->timestamp - frame_time) > MAX_TS_VAR) {
#if DEBUG_ASYNC_FRAMES
			blog(LOG_DEBUG, "timing jump");
#endif
			source->last_frame_ts =
				next_frame->timestamp - frame_offset;
		}

		frame_time = next_frame->timestamp;
		frame_offset = frame_time - source->last_frame_ts;
	}

#if DEBUG_ASYNC_FRAMES
	if (!frame)
		blog(LOG_DEBUG, "no frame!");
#endif

	return frame != NULL;
}
```
## 为什么要每次都要累加当前已经播放的时长
具体来讲就是当有 `Seek`， `循环播放`的时候，如果不加这个很有可能会导致丢帧。拿`Seek`来讲，如果`Seek`的位置与当前播放的位置时间间隔了 2秒以上， `frame_out_of_bounds`能够兼容到，这个时候不会丢帧；或者往右 `seek` 2秒以内，也不会丢帧，因为渲染线程落后了，当前就不会取帧去渲染，直到渲染线程追赶上 `seek` 那帧的时间戳。  
但是一旦往左`seek` 2秒以内就出问题了，`while (source->last_frame_ts > next_frame->timestamp)` 的条件就会成立， 所有在`seek`位置开始重新解码的帧在追赶上`video_time`之前几乎都会被丢掉，除非运气好，遇到了`if (source->async_frames.num == 1)`条件成立。  
而且因为 `sys_time` 不断向前，实际丢弃掉的帧数可能还会大于比向左`seek`区间内的帧数。

## 映射到全局系统时间
`m->play_sys_ts - base_sys_ts` 的作用就是将前面3部分的媒体播放时间轴映射到OBS全局系统时间上，变量值的意思是在全局系统时间轴上已经播放了多久。两者都是通过 `os_gettime_ns()`函数获取的。最开始我好奇`base_sys_ts`为啥要等到第一个媒体播放的时候才初始化，而不是在渲染系统或者整个系统初始化的时候就同步初始化，感觉应该是惰性编程的原因，因为假如系统启动后等了20分钟才初始化，那么`base_sys_ts`的值就是20分钟，转换成纳秒就是一个很大的值，但是这个值没有啥意义。所以OBS设计成了第一个媒体播放的时候才初始化。

## OBS渲染同步的方式
OBS的播放帧率是一个设置的固定值，所有媒体源的帧都要固定到这个节拍上来。若果媒体源的帧率和OBS的渲染帧率一致，那就每帧都完美同频了。但是只要不一样，要么就丢帧要么就等时间戳重合了才渲染。
举两个例子来讲:
- OBS的渲染帧率小于素材帧率, 比如渲染帧率是25P，而素材帧率是50P，这个时候就会出现丢帧。因为 OBS 每次准备帧的时候 `while (source->last_frame_ts > next_frame->timestamp)`的条件都会成立，比如当OBS渲染完第一帧取下一帧时，`sys_offset`的值是 40ms, 下一帧需要的时间戳是 80ms位置的，所以素材在 60ms 位置的帧就会被丢掉。
- OBS的渲染帧率大于素材帧率，比如渲染帧率是50P，而素材帧率是25P。 OBS的 `ready_async_frame`就不会每次都刷新帧，而是间隔一次才刷新。实现细节是通过 `ready_async_frame` 函数里面的 `frame` 是否被赋值来判断。而 `frame` 的值只有在 `while (source->last_frame_ts > next_frame->timestamp)` 里面才会被刷新，所以当渲染帧率50P,而素材帧率25P时，frame是间隔一次才刷新。

## 总结
OBS这个视频渲染同步方式感觉整得好复杂。但是仔细想想也是，它是一个**实时多源合成系统**，而不是一个播放器。