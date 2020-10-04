---
layout: post
title:  "Visualizing an audio application's signal processing chain with an IDAPython debugging plugin"
date:   2018-03-12
categories: IDAPython
comments: true
---

*If you don't care about audio stuff and just want to see the results, or just want an example debugging plugin in IDAPython, feel free to skip the first two sections.*

The result was way nicer than I expected. I might even want to run this when debugging my own audio code!

### Introduction

In this blog post, I'll demonstrate a technique that allows us to automatically generate audio files for each intermediate step in the signal chain of a piece of audio software. We will take a look at the results generated for a free synthesizer.

Digital audio is made up of signals consisting of thousands of consecutive samples - each sample is a single number, usually floats when processing audio and ints when storing it in a lossless file. These samples will get eventually filtered by an audio interface to generate an analog signal that will get passed to a speaker. After going through thousands of instructions, an audio callback might finally put a sample in the audio buffer, but that sample doesn't mean anything by itself: you need to see a whole bunch of samples to understand what a signal is representing.

For this reason, singlestepping through an audio callback in a debugger isn't going to be very useful in understanding what is going on. Every value you're going to see in the registers window is meaningless. What you're interested in is what signals are produced by what instructions. This is exactly what we can do by tracing the execution of the callback and logging all results of instructions on xmm register destinations, as we'll see later. This particular synth uses SSE, and in particular we're looking for xmm registers containing 4 floats each, but the same principles would be applicable for every datatype.

### What a synthesizer looks like, internally, kind of

Here's a very simplified, unrealistic and just generally bad pseudocode example of what a software synthesizer might do to fill the audio callback. This is not meant to explain the way in which you would write a synthesizer, but why the tracing code we are going to write is supposed to work. This code would make for a very boring, very badly aliasing oscillator.

{% highlight C %}
float oscillator_tick() {
  oscillator->phase += oscillator->inc;
  wrap_oscillator_phase();
  int int_phase = int(oscillator->phase * oscillator->wavetable_size);

  float oy = oscillator->wavetable[int_phase];
  return oy;
}

float filter_tick(float x) {
  float fy = last_y * 0.9 + x * 0.1;
  last_y = fy;

  return fy;
}

void callback(float* sample_buffer, int number_of_samples) {
  for(int i = 0; i < number_of_samples; i++) {
    float y = filter_tick(oscillator_tick());

    sample_buffer[number_of_samples];
  }
}
{% endhighlight %}

`callback` is going to be called by the soundcard or by an audio DAW tens of times per second.

What you will notice is that before `filter_tick` can do anything, it needs data from the oscillator. `oscillator_tick` is always going to run before `filter_tick`. If we go in `oscillator_tick`, we see that the oscillator phase is always going to be incremented before it's wrapped, and it's going to be wrapped before it's used to get a sample from the wavetable.

A second thing to notice is that `float oy = oscillator->wavetable(int_phase);` is called once per sample sequentially, which means that if we plotted `oy` we would see a signal. The same is true for `fy` in `float fy = last_y * 0.9 + x * 0.1`. These signals would also be very easy to spot: in a virtual analog synth, `oy` would be a classic saw, triangle or square waveform, with sharp corners. `fy` would be the same wave, but with rounded corners.

### Single stepping in IDAPython to generate audio files

I put the code in a github repo: [https://github.com/0xAC44/xmm_ida_trace_dsp](https://github.com/0xAC44/xmm_ida_trace_dsp)

Disclaimer: this is a proof of concept I used to start analysing this particular software. The code is not polished and you'll have to edit the files themselves every time you want to use them, in order to specify the parameters. I've just started playing with IDA and IDAPython, so I'm not sure about the best practices yet.

The first file, trace_tracer.py, is going to do the actual tracing. You first break where you want to start the tracing in the IDA debugger, manually edit the script to specify where you want to break and end the trace (again, sorry!), and then load it from the IDA file menu. I only tested this on IDA starter 7.1, as that's the only version I own.

Here are some code snippets, so we can take a look at how this works:

{% highlight python %}
class TraceTracerHook(DBG_Hooks):
{% endhighlight %}

We define a subclass of DBG_Hooks. This will allow us to define callbacks that IDA will call when a debugging event is generated.

{% highlight python %}
    # ... __init__ and other internal stuff
    def dbg_exception(self, pid, tid, ea, exc_code, exc_can_cont, exc_ea, exc_info):
        print("passed exception")
        request_continue_process()
        run_requests()

        return 0
{% endhighlight %}

I was having problems with exceptions being thrown when I started writing this. I think that was because of a mistake I made in some other code, but I left this in anyway.

I'm going to add comments to this next snippet. This is the meat of the script:

{% highlight python %}
    # whether we're stepping into or over, we call the same function.
    # dbg_step_into should probably be the only one used since
    # we want to trace everything.
    def dbg_step_into(self):
        self.dbg_step_over()

    def dbg_step_over(self):
        # when this function gets called, EIP points to an instruction
        # that hasn't yet been executed.

        # we need to already have executed something to be able to
        # log what the result was
        if not self.last_eip:
            self.last_eip = get_reg_value("EIP")
            request_step_into()
            return

        # the EIP that we're seing the results of is the last EIP
        eip = self.last_eip

        # we only log if EIP is in one of the modules we care about

        # in that case, we log eip (of the already executed instruction)

        # if the destination operand was an xmm register, we also
        # log the value of that xmm register
        if any([eip > x[0] and eip < x[1] for x in self.mods]):
            this = {
                'eip': eip
            }

            dest = idc.GetOpnd(eip, 0)
            if "xmm" in dest and len(dest) == 4:
                this['xmm_destination'] = (dest, get_xmm(dest))

            self.eip_list.append(this)


        # if we executed more steps than max_steps or the EIP is where we
        # want to stop tracing (in this blog post, that's at the end of
        # the callback function)

        # we dump the trace, and unhook this plugin so that the user can
        # continue working in IDA

        self.steps += 1
        if eip in self.stop_on or self.steps > self.max_steps:
            self.dump_eip_list()
            self.unhook()
            request_continue_process()
        else:
            request_step_into()

        # we also log how fast we're going.
        LOG_EVERY = 1000
        if self.steps % LOG_EVERY == 0:
            if self.last_report:
                print(self.steps, int(LOG_EVERY / (time.time() - self.last_report)), "steps per second")

            self.last_report = time.time()

        self.last_eip = get_reg_value("EIP")
{% endhighlight %}

Something to note is that this isn't optimized at all. Since I have IDA for linux and I'm debugging windows VM with a remote debugger, I'm capped to less than a thousand steps per second anyway, but you might want to rewrite it to be more efficient if you need to use this for something that will run more than a couple hundred thousand instructions and you're using a native debugger.

You then run from a terminal xmm_graph.py with python2 to generate the wav files. You can optionally also run it from IDA to generate a callgraph (but not the wav files). You will need to install pysndfile and numpy to run this script.

The wav files' filenames are ordered by the time in the trace at which the instruction they refer to were first executed. For instance, if the instruction at address 0xf00b4r was first executed after 10000 thousands instructions recorded in the trace, its file name would be "10000.0xf00b4r.wav". Since we're dealing with xmm registers that are going to hold 4 float values, we actually use .0.wav, .1.wav, .2.wav and .3.wav, in case the application makes use of all of those values.

### Results

Here's some interesting stuff we can immediately see at a glance after running the scripts, no manual static or dynamic analysis needed:

![saw]({{ "/assets/xmm_tracing_1/1.png" | absolute_url }})

This is the first instruction where a sawtooth appears. The oscillators I selected in the software are indeed saw and triangle. This means that around there there must be an instruction that loads samples from some kind of a wavetable. There are some ripples, which means this shouldn't be be a naive oscillator.

![tri]({{ "/assets/xmm_tracing_1/2.png" | absolute_url }})

Here's the triangle oscillator popping up a bit later.

![correction]({{ "/assets/xmm_tracing_1/3.png" | absolute_url }})

This might be a correction signal to fix aliasing problems in an oscillator that is not bandlimited. Later in the trace there's a saw signal that is exactly the sum of the first saw plus this signal.

This is just my assumption, as I don't know enough about Digital Signal Processing to be sure, but for an example of a technique similar to this, see [polyblep](https://www.kvraudio.com/forum/viewtopic.php?t=375517).

![pulse]({{ "/assets/xmm_tracing_1/4.png" | absolute_url }})

Neat! That's a pulse oscillator. I think the synth is using this to modulate the oscillators we saw before, but I don't remember the exact patch I used to make these screenshots. This is another cool thing about this: we could vary one parameter at a time in the synth to see where the signals change, to quickly identify what everything in the code is.

![pulse2]({{ "/assets/xmm_tracing_1/5.png" | absolute_url }})

Here's the same signal, but scaled down. This is probably because the modulation from the pulse oscillator is not set at 100%.

![mix]({{ "/assets/xmm_tracing_1/6.png" | absolute_url }})

The sawtooth and triangle getting mixed together. The mix knob is set at about 50%.

![final]({{ "/assets/xmm_tracing_1/7.png" | absolute_url }})

There's a lot of stuff to explore in these files! A synth is a fairly complex things requiring a lot of steps, and each step is a file, but this last example, indeed the last wav file with the length of the audio buffer, has the final sound that I would hear from the speakers if I didn't run this in a debugger.

### That's it!

I hope you enjoyed this post! :) Let me know what you think.
