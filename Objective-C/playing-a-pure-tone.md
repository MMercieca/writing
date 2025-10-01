## Playing a Pure Tone (Sine Wave) in IOS

### Introduction

If you just want the code:

[ToneGenerator.m](https://github.com/MMercieca/Handshake/blob/master/Handshake/ToneGenerator.m) – The finished class for this post

[HandShake](https://github.com/MMercieca/Handshake) – The complete project

Most programming languages have some variant of:
Beep(frequency, duration);

Objective C does not. Part of this is the nature of iOS devices: an interruption can happen at any time meaning there’s no way to guarantee that the call will have enough time to complete. To produce a tone on demand, the programmer must fill the audio buffer with the tone data and the device will play the data when it can.

### Audio Player Setup

I got most of the setup code from http://christianfloisand.wordpress.com/2013/07/30/building-a-tone-generator-for-ios-using-audio-units/. An explanation is also provided there so there’s no need to restate it here.

### Rendering the Audio Data

Forgetting my high school physics completely, I expected a constant frequency to need constant audio data. I expected a tone of 440 Hz to look something like, `[440, 440, 440, 440, 440]`.

A pure sound is a pure wave. Generating a wave is easy using the sine function. I started with code from http://www.cocoawithlove.com/2010/10/ios-tone-generator-introduction-to.html, but had to update it for portability (see “Where I Screwed Up #2″).

```
OSStatus RenderTone(
                    void *inRefCon,
                    AudioUnitRenderActionFlags 	*ioActionFlags,
                    const AudioTimeStamp *inTimeStamp,
                    UInt32 inBusNumber,
                    UInt32 inNumberFrames,
                    AudioBufferList *ioData)

{
	// Fixed amplitude is good enough for our purposes
	const double amplitude = 1;
   
	// Get the tone parameters out of the class
	
        ToneGenerator *toneGenerator = (__bridge ToneGenerator*)inRefCon;
        double theta = toneGenerator->_theta;
        double frequency = toneGenerator->_frequency;
	
	double theta_increment = 2.0 * M_PI * frequency / SAMPLE_RATE;
   
	// This is a mono tone generator so we only need the first buffer
	const int channel = 0;
	Float32 *buffer = (Float32 *)ioData->mBuffers[channel].mData;
	
	// Generate the samples
	for (UInt32 frame = 0; frame < inNumberFrames; frame++)
	{
		buffer[frame] = sin(theta) * amplitude;
		
		theta += theta_increment;
		if (theta > 2.0 * M_PI)
		{
			theta -= 2.0 * M_PI;
		}
	}
	
	// Store the theta back in the object
	toneGenerator->_theta = theta;
   
	return noErr;
}
```

Originally I didn’t save theta as a class variable. The result was that the wave never completed a cycle and instead of a beep it sounded like a click.

### Playing the Tone

The class now has enough in it to start playing a tone.

```
   [self.audioSession setActive:true error:nil];
   self.frequency = frequency;
   // Create the audio unit as shown above
   [self createToneUnit];
   
   // Start playback
   AudioOutputUnitStart(_toneUnit);
```

To keep track of time, I cheated and just slept the thread using `[NSThread sleepUntilDate:date]`.

To stop playing all that is necessary is just tearing down the toneUnit.

```
   self.frequency = 0;
   self.theta = 0;
   AudioOutputUnitStop(self.toneUnit);
   AudioUnitUninitialize(self.toneUnit);
   AudioComponentInstanceDispose(self.toneUnit);
   self.toneUnit = nil;
```

For completeness, the stop handler and interruption handlers should do the same thing.

### Where I Screwed Up #1

When I settled on my naive encoding, I figured I’d just assign a tone per bucket based on the ASCII value of the character to send. Since I wanted them all to be inaudible I’d make sure all sent tones were above 19 kHz.
`frequency = 19000 + (43 * (int)charToSend);`

I didn’t realize until later that the lowest ASCII value I was sending was the `.` character with a corresponding frequency of 20,978 kHz. As far as I can tell, that’s 978 Hz above the rated max of my iPhone speakers. It actually still works for values under `5` (ASCII 53) but I shouldn’t expect that.

It works for the proof of concept though. Production uses would require a better encoding scheme— for this and other reasons.

### Where I Screwed Up #2

A lot of the code samples I used were before automatic reference counting. In the beginning the code crashed after 1/43 of a second (one audio frame at 44100 Hz).

Through some trial and error I managed to not get through one entire tone, but all future tones wouldn’t play correctly— my cats really hated these problems.

Automatic reference counting turned out to be the culprit. I wasn’t correctly casting the ToneGenerator in the RenderTone function so theta wasn’t properly saved and new tone units would not use fresh variables. The solution turned out to be the __bridge cast in the RenderTone function.

### Resources

I found a number of helpful blog posts:

* https://github.com/hollance/AudioBufferPlayer
* http://christianfloisand.wordpress.com/2013/07/30/building-a-tone-generator-for-ios-using-audio-units/
* http://atastypixel.com/blog/using-remoteio-audio-unit/
* http://stackoverflow.com/questions/10180500/how-to-use-kaudiosessionproperty-overridecategorymixwithothers
* http://www.cocoawithlove.com/2010/10/ios-tone-generator-introduction-to.html