## Identifying a Tone (Sine Wave) in iOS with the Accelerate Framework

### Introduction

If you just want the code:

[ToneReceiver.m](https://github.com/MMercieca/Handshake/blob/master/Handshake/ToneReceiver.m) – The finished code for this post

[Handshake](https://github.com/MMercieca/Handshake/tree/master/Handshake) – The complete project

I found a lot of people had the same question that I did, “How do you identify a frequency in <insert language here>?” The answer was usually the same: use a fast Fourier transform. It’s even built in to iOS.

### Fast Fourier Transforms

While I've learned more physics programming for my phone than I used in college, I did have physics classes. Fast Fourier transforms weren't mentioned in any of my math classes because they're primarily used in electrical engineering.

The Wikipedia entry lost me. I figured out how to use them before I reasoned how they (probably) work. I figured out that with the Accelerate framework I could use `vDSP_fft_zrip` with an array of samples to get an array of intensities at particular frequency ranges. The maximum value in that array corresponds to the strongest frequency range.

<b>A quick note on the Accelerate framework</b>

The Accelerate framework has a number of functions for digital signal processing. For speed, these are C functions which must be bridged so they can be used from Objective C. The fast Fourier transform functions are just some of what the framework provides. There are functions for taking integrals, derivatives, the Fourier transforms I’m using, and other processing that I’ve yet to learn.

### Packing the Data

The Fourier functions all operate on real, complex arrays and the microphone provides real data.

```
//Convert the microphone data to real data
float *samples = malloc(numSamples * sizeof(float));
vDSP_vflt16((short *)inSamples, 1, samples, 1, numSamples);

//Convert the real data to complex data
// 1. Populate *window with the values for a hamming window function
float *window = (float *)malloc(sizeof(float) * numSamples);
vDSP_hamm_window(window, numSamples, 0);

// 2. Window the samples
vDSP_vmul(samples, 1, window, 1, samples, 1, numSamples);
      
//3. Define complex buffer
COMPLEX_SPLIT A;
A.realp = (float *) malloc(halfSamples * sizeof(float));
A.imagp = (float *) malloc(halfSamples * sizeof(float));
      
// Pack samples:
vDSP_ctoz((COMPLEX*)samples, 2, &A, 1, numSamples/2);
```

`vDSP_fft_zrip` is an in place function, so the number of frequency ranges is exactly the same as the number of samples fed in, which works best if everything is a power of two. The iPhone takes 44,100 samples per second and 1024 is a nice power of two. So for 1/43 of a second I can identify the 43 Hz bucket that has the strongest frequency. Not good enough for a tuner, but good enough to communicate. If I wanted a greater resolution I could just take more samples. For this proof of concept, a 43 Hz resolution is enough.

```
// Setup the FFT
// 1. Setup the radix (exponent)
int fftRadix = log2(numSamples);
int halfSamples = (int)(numSamples / 2);
// 2. And setup the FFT
FFTSetup setup = vDSP_create_fftsetup(fftRadix, FFT_RADIX2);
```

And at the heart of the function, perform the fast Fourier transform.

```
// Perform a forward FFT using fftSetup and A
// Results are returned in A
vDSP_fft_zrip(setup, &A, 1, fftRadix, FFT_FORWARD);
      
// Convert COMPLEX_SPLIT A result to magnitudes
float amp[numSamples];
amp[0] = A.realp[0]/(numSamples*2);
      
// Find the max
int maxIndex = 0;
float maxMag = 0.0;
      
// We can't detect anything reliably above the Nyquist frequency
// which is bin n / 2 .
for(int i=1; i maxMag)
   {
      maxMag = amp[i];
      maxIndex = i;
   }
}
```

### Recording

Apple provided the delegate which is called when the audio buffer was full. This class just has to implement `AVCaptureAudioDataOutputSampleBufferDelegate`.

```
-(void)start
{
   AVAudioSession *session = [AVAudioSession sharedInstance];
   [session setActive:YES error:nil];
   
   self.captureSession = [[AVCaptureSession alloc] init];
   AVCaptureDevice *device = [AVCaptureDevice defaultDeviceWithMediaType:AVMediaTypeAudio];
   AVCaptureDeviceInput *input = [AVCaptureDeviceInput deviceInputWithDevice:device error:NULL];
 
   [self.captureSession addInput:input];
   
   AVCaptureAudioDataOutput *output = [[AVCaptureAudioDataOutput alloc] init];
   dispatch_queue_t queue = dispatch_queue_create("Sample callback", DISPATCH_QUEUE_SERIAL);
   [output setSampleBufferDelegate:self queue:queue];
   [self.captureSession addOutput:output];
   
   [self.captureSession startRunning];
}
```

And what gets called when the buffer is full:

```
- (void)captureOutput:(AVCaptureOutput *)captureOutput
didOutputSampleBuffer:(CMSampleBufferRef)sampleBuffer
       fromConnection:(AVCaptureConnection *)connection
```

**And a Note on a Hack**

I had a problem getting the sample buffer to be consistent. The first time I started recording I’d get 4096 samples, and all subsequent times I’d get 1024. For consistency I implemented a bit of a hack.

```
- (void)totalHackToGetAroundAppleNotSettingIOBufferDuration
{
   self.captureSession = [[AVCaptureSession alloc] init];
   AVCaptureDevice *device = [AVCaptureDevice defaultDeviceWithMediaType:AVMediaTypeAudio];
   AVCaptureDeviceInput *input = [AVCaptureDeviceInput deviceInputWithDevice:device error:NULL];
   
   [self.captureSession addInput:input];
   
   AVCaptureAudioDataOutput *output = [[AVCaptureAudioDataOutput alloc] init];
   dispatch_queue_t queue = dispatch_queue_create("Sample callback", DISPATCH_QUEUE_SERIAL);
   [output setSampleBufferDelegate:self queue:queue];
   [self.captureSession addOutput:output];
   
   [self.captureSession startRunning];
   [self.captureSession stopRunning];
}
```

### Sending the Result

It’s kind of anticlimactic after the trouble of figuring everything out sending the result back is a perfect case for a simple delegate.

```
NSNumber* toSend = [[NSNumber alloc] initWithInt:maxIndex];
      
if (self.delegate)
{
   [self.delegate didReceiveTone:toSend];
}
```

### Where I Screwed Up

I screwed up enough making this class that I have enough content for a blog post on its own. So it’s going to get one.

### Future Work

This code identifies the maximum frequency bucket across the entire range that the iPhone can receive. For a proof of concept that’s fine, but in production I’d likely look for local maximums across the frequency range that I was interested in and compare those to the maximums over the rest of the data.

I’d also like to remove that hack, but again this is a proof of concept.
