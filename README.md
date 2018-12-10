# DSP Filters project repo
This project is heavily based on the work of github user gavv. 
His github repo is [here](https://github.com/gavv/snippets/tree/master/decode_play)
see details in his [blog post](https://gavv.github.io/blog/decode-play/).

# Overview

This project implements a digital filter in the time domain in c++.
This is accomplished by implementing the difference equation directly.
For examples of this look at the AllpassFilter.{cpp,h} and FIRFilter.{cpp,h} files.
an output variable (y) is generated by using the input and delayed versions of an
intermediate variable. (This is a direct form 2 implementation, direct form 1 can
be used but it requires a second delay buffer.)

The project is split into three parts an audio decoding program (in ffmpeg_decode.cpp),
a playback program (ffmpeg_play.cpp), and the actual filter (FilterProject.{cpp,h}).

The decoding program outputs the raw samples from the audio file over stdout the filter program
takes the samples and filters them (well, duh) then again outputs them over stdout. Last, the 
playback program takes the samples and does magic on them so the linux audio system can play
it back.

One gotcha is that since the files are typically stereo left and right samples are interleaved.
this means that each buffer should be twice as long as designed, then every other index is used for each channel.
For example to implement a 2nd order filter a buffer with a length of 6 is needed then indexes
0, 2,and 4 will be used for the left channel, and 1, 3, and 5 will be used for the right channel.

**The filter should be implemented in the do_filtering() function.**

Decoded samples are always in the same format:
* linear PCM;
* two channels (front Left, front Right);
* interleaved format (L R L R ...);
* samples are 16-bits sighed integers in little endian (actually CPU should be little-endian too);
* sample rate is 44100 Hz.

Note that the raspberry pi works much better if the GUI has been turned off (just google it).
# Compiling the code 

A note on format any font that looks
```
like this (aka has three tick marks before and after it)
```
is meant to be typed into a terminal (command line) on the pi. Remember that pressing tab a few times auto-completes.

To get the project working start with a raspberry pi with a bootable sd card. 
The pi should be running, logged in, with a keyboard and screen plugged connected, **and connected to the internet**.
You probably should also disable the pi's GUI, as it isn't necessary, and will just bog down the pi.

Steps to filtering goodness
1. Change the keyboard layout (it defaults to a UK layout)
  Run the following command in the terminal to change the layout.
  ```
  sudo dpkg-reconfigure keyboard-configuration 
  ```

2. Reboot the pi (this makes the changes take effect)
  ``` 
  sudo reboot 
  ```

3. Get my code from git
  ``` 
  git clone https://github.com/sellicott/DSP-FFMpeg-Reverb.git 
  ```

4. Install the required packages (this is the audio decoding library)
  ``` 
  sudo apt install libavcodec-dev libavformat-dev libavdevice-dev 
  ```
  This installs the three listed packages from the raspbian software library.

5. Next move into the directory and compile the code 
  ```
  cd DSP-FFMpeg-Reverb
  make
  ```
  This will (hopefully) build all of the code needed to run your project.

6. Now you need some great music: 

  Put your files on a flash drive and connect the drive to the pi. Next, copy the files
  into the project directory.
  
  Your flash drive will be in the /media/pi/name_of_your_drive folder on the pi.
  Assuming that your file is on the root directory of your flash drive:
  ``` 
  cp /media/pi/name_of_drive your_snazzy_audio_file.mp3 ./ 
  ```

  If this doesn't work, you will need to mount your drive to the directory structure of the pi.

  First check what drive name your USB drive is. This command will list all of the drives (the different letters),
  and all the partitions on them (the numbers). You will want to remember the highest letter drive avalible
  ```
  ls /dev/sd??
  ```

  Now run the following command, replacing the question mark with the drive letter you found 
  (it will probably be 'a' or 'b').
  ```
  sudo mount /dev/sd?1 /media/
  cp /media/name_of_drive/your_snazzy_audio_file.mp3 ./ 
  ```
  

7. Now you can run your filter
  The decoder, filter, and playback programs must be connected via pipe.
  (A pipe takes the output from stdout from one program and "pipes" it into
  the input for another program.) To use your filter run the following command.
  ``` 
  ./ffmpeg_decode your_snazzy_audio_file.mp3 | ./ffmpeg_filter | ./ffmpeg_play 
  ```
  Note that the input audio file can be in almost any format.

# The Code
In the FilterProject.cpp file the function do_filtering implements the filters for the project.
(The main file calls another function get_samples() which calls do_filtering() in order to apply the filter to the buffer
provided by the audio decoder program.) 

**I have made simple template for you to edit in Template.cpp and Template.h, when you want to compile it you shoud rename Template.cpp to FilterProject.cpp.**
This will allow the makefile to build your code instead of my example code (which implements reverb). All of the comments below apply to the template code as
well.

**If you add any more files to the project you will have to add them to the ```SRC :=``` line in the Makfile** 

In my project (the reverb example) I used the do_filtering function to chain togeter all of the other filters I used 
(four allpass filters and a few FIR filters). These filters are implemented as c++ objects, meaining they need to be instantiated 
(they are defined near the end of FilterProject.h). I do this in the *base initilization section* 
(right after the beginning of the constructor, you remember CS1220 don't you?) of the FilterProject class. 

Here is an example (from Template.cpp)
```c++
FilterProject::FilterProject() : 
  //initilize filters
  allpassFilter(4410, 0.6), 

  //add your comma separated filters to initilize here 

  // pass in FIR coefficients to the FIR filter class
  firFilter({ 0.5, 0.7, 0.5 /* Put your comma separated filter coefficients here */})

  // note the last filter does not have a comma after it
{}
```

The actual filters for the project are implemented in the files 
AllpassFilter.cpp, FIRFilter.cpp and their corresponding .h files AllpassFilter.h and FIRFilter.h.
You can probably copy these files when implementing your own filter classes (or just use my FIR filter code as-is).

## Implementing Filters
This following code snippet initilizes my allpass filter (from AllpassFilter.cpp)
```c++
AllpassFilter::outType AllpassFilter::do_filtering(outType new_x) {
  // grab the delay line
  auto &g = *delay_buff.get();

  //get the latest value to come through the delay line
  auto g_out = g.back();
  g.pop_back();

  // do the thing! follows the standard allpass filter structure
  auto g_in = new_x + gain*g_out;
  g.push_front(g_in);
  auto y = -gain*g_in + g_out;

  //return the newest value for y
  return y;
}
```

In general an IIR implementation would look something like this 
(Just implementing equation 4.110 and 4.111 from p269 of the textbook)
note that the delay_buff deque needs to be 2x the lenth of your filter coefficients to make room
for both left and right audio samples
```c++
  //find g[n] each delay is an index in the array 0 is the latest value n is the oldest value
  //note that each index is multiplied by 2 because left and right audio samples are interleaved
  //also note that g[1] is actually g[n-1], and g[3] is g[n-2], etc
  auto n = g.size()/2; 
  auto g_n = new_x + a1*g[1] + a2 * g[3] + /*...*/ a_n * g[2*n-1];

  // pretty much the same thing here 
  auto y = b0 * g_n + b1 * g[1] + /*...*/ b_n * g[3];

  // add the value to the delay line
  g.pop_back();
  g.push_front(g_n);

  //return the newest value for y
  return y;
```

The delay line should be constructed in a way similar to how I did it in the FIRFilter code
Note that the delay_buff from the previous code examples serves the same purpose as input_samples, 
and they are constructed the same way.
```c++
// the taps input are the coefficients for the FIR filter
FIRFilter::FIRFilter(std::vector<float> taps_) :
  // initilize the class taps variable.
  taps(taps_)
{
  // multiply by 2 to make room for the L/R audio channels
  delay = taps_.size()*2;

  // initilize buffer to zeros
  input_samples = std::make_unique<deque>(delay, 0.0);
}
```
