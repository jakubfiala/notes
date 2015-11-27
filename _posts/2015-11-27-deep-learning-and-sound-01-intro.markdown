---
layout: post
title:  "Deep Learning and Sound ~ 01: Intro"
date:   2015-11-27 13:30:00 +0000
categories: deep-learning, sound, machine-learning, composition, lstm, neural-networks
---

This is the first from a series of posts I'm going to write over the next year or so, detailing my research in audio generation with deep learning techniques.
Due to the quick-and-dirty nature of this blog and my generally inadequate level of knowledge in this area, I'm not going to discuss existing theories and practices
of deep learning, neural networks etc. in much detail – this is rather an attempt to formalize my research and development, and provide readable explanations of the problem solving approaches I'm developing.

#### "But I want to find out EVERYTHING about deep learning!"

If you'd like to learn more about deep learning methods, you should definitely check out Andrej Karpathy's [cs231n](https://cs231n.github.io) course notes, which will gracefully guide you all the way from basic kNN algorithms (ancient stuff), through linear models and multi-layer perceptrons (80s stuff) to convolutional neural networks (21st century stuff). When you feel confident about those, be sure to check out Andrej's [beautiful blog post](http://karpathy.github.io/2015/05/21/rnn-effectiveness/) on Recurrent neural nets, which I'm going to be focusing on *a lot.* If that doesn't satiate your appetite for knowledge, the next step is to read some papers – Alex Graves' handwriting generation[^1] is mindblowing, as well as the smoking hot paper which has just been published[^2] by Alec Radford et al. There's a million others, too, and most of them are accessible on *arxiv* or via Google Scholar. Good luck!

---

### … back to business.

Having given you enough reading for the next couple of weeks, I can now <del>stop writing this post, order a takeaway pho and play some fallout 4</del> move on to explaining what my research is all about. Here's the brief:

## What we're aiming for

In the recent years, deep learning has become extremely hyped and widely used as a method for developing **robust, accurate models of complex data.** In other words, people realised that if your computer is fast enough, you can stack up many, many small mathematical functions (units, neurons, etc.), and using standard, decades-old backpropagation algorithms, tweak these functions so that they end up being capable of **classifying, predicting or transforming** not only simple data vectors such as wind speed records in Aberdeen, but also complex, semantically abstract and high-dimensional data such as images, sound, human-made text, etc.
<br>
_**Example:** Using deep learning, you can generate pictures of bedrooms that don't exist:_<br>
![DCGAN by Alex Radford et al.](https://raw.githubusercontent.com/Newmu/dcgan_code/master/images/lsun_bedrooms_five_epoch_samples.png)
_source: DCGAN by Alex Radford et al._
<br>
The magic of this technology is that in order to create an AV generation system, one can use example-based training – rather than tweak software parameters that only the computer really understands, one can describe their desired output by exposing the system to a range of things that "sound or look like" the desired output. **The aim of my research, as part of the [EAVI](http://eavi.goldsmithsdigital.com) research group at Goldsmiths, is to _build tools that harness the power of deep learning to generate musical and non-musical sound._** These tools may well prove to be much more intuitive and creative than any existing sound composition tools.


## State of the art

Now, people have been generating _music_ with neural networks for a while. Notable is the work of Douglas Eck et al.[^3], who managed to train a recurrent neural network (henceforth RNN) on a dataset of blues songs, and generate rather plausible blues compositions. The problem, as usual, lies in the data: rather than feeding their model with recordings of blues songs, they trained it on the _structural data_ of the individual songs – each song was represented as a sequence of note values, and timing was merely implicit, i.e. each step in the sequence is assumed to last the same amount of time.

As for training on/generating audio data, this has only really been possible with the recent rise of deep models, i.e. models with many layers of units, each performing a small calculation relevant to only a small part of the input data. Methods for doing this are largely inspired by efforts in __speech recognition,__ such as Graves et al.[^4], where audio is split into short (typically around 10ms) frames, and for each frame, we calculate feature values, which are then used to train a model. The training is usually performed on one or more high-end GPUs, using frameworks such as Caffe (Lua), Theano (python), or more recently, TensorFlow (python). However, since the final output of speech recognition models is usually a single label, or a set of probabilities for each possible label, it is generally possible to train on dramatically reduced representations of the data, such as MFCC, from which audio signals cannot be reconstructed.

![MFCC of 'four'](http://pmtk3.googlecode.com/svn-history/r663/trunk/docs/demos/dataDemos/plotMFCC_02.png)
_source: Tommi Jaakkola_

<br>
So the two of the main problems that arise when training models on audio data, the Scylla and Charybdis of neural nets and sound, are:

+ __high dimensionality__ – in one second of CD-quality audio, there are 44100 numbers. However, not much interesting stuff can happen in that time frame. This means that the model needs extremely large chunks of data to obtain a good idea of what's going on, unlike images from common datasets like CIFAR-10[^5], where only a few pixels may contain valuable semantic information.
+ __repetition__ – even when the model learns from huge frames (2048+ numbers in one example), in natural sounds, each distinctive waveform shape tends to repeat tens, or even hundreds of times, before a significant change occurs. This means that the model needs to be trained with data that barely ever changes, as far as the model's memory is concerned. This is also why LSTM models outperform standard RNNs in audio-related tasks.[^6] Even when the low-level repetition of waveforms is dealt with, repetition of beats, melodic phrases, and entire sections of songs come into play, particularly when working with recordings of popular music, which is extremely spectrally and semantically rich.

I'm way too happy about this mythological analogy. Let's move on.

### Fighting Scylla and Charybdis

There have only been a handful of efforts by researchers and coders to tackle the two big problems with audio generation. Perhaps the most notable one that I came across, is a project called __GRUV__ by Aran Nayebi and Matt Vitelli[^7], which is luckily all [on GitHub](https://github.com/MattVitelli/GRUV). In their [demo video](https://www.youtube.com/watch?v=0VTI1BBLydE) on YouTube, they demonstrate training a model with a dataset of Madeon songs (which is an abomination), and amazingly, after a good amount of training epochs, one can clearly hear the disgusting compressed EDM beat, traces of discount vocal samples from Beatport, and (thank the heavens!) a considerable amount of random noise.

After recovering from the musical experience, and looking at the python code, it seems that GRUV represents a good (albeit only partial) solution to the above problems. Its LSTM architecture is capable of recognizing the most important parts of EDM music style, and generating a somewhat meaningful sound pattern. The output does not seem to evolve beyond a steady beat, but in terms of frequency content, it's absolutely characteristic of the input dataset. More importantly, several smart solutions are introduced in the code, that seem to alleviate the aforementioned problems of audio data.

+ __Data is preprocessed by subtracting the mean and dividing by variance.__ These two are then stored as numpy arrays, and added into the output in the generation script. This means that a lot of the information that is the same in every frame never makes it into the model, but still appears in the output to reconstruct the original sound as closely as possible. The model is exposed to more diverse data, and can learn much easier (the following example is taken directly from the GRUV code).

~~~ python
# Mean across num examples and num timesteps
mean_x = np.mean(np.mean(x_data, axis=0), axis=0)
# STD across num examples and num timesteps
std_x = np.sqrt(np.mean(np.mean(np.abs(x_data-mean_x)**2, axis=0), axis=0))
# Clamp variance if too tiny
std_x = np.maximum(1.0e-8, std_x)
~~~

+ __Time-distributed FC layers envelop the LSTM layers in the model.__ This means that in both the input and output layer, the same transform is applied to each timestep of the training sequence.
+ __Data is transformed into the frequency domain via FFT, but the model is trained on both the real and imaginary parts of the transform.__ Frequency data, being more representative of what the sound is like to a human, are probably more useful to train on. However, because the model effectively trains on a complex spectrum, it does not lose the phase information, and it's easier to reconstruct the output in the time domain.

The last point is somewhat interesting, because the way GRUV implements it, is by concatenating the real vector to the imaginary vector. In the end, we end up with the same dimensionality as if we just fed in the original signal. This, beside the fact that the generated sounds tend to be noisy and repetitive, means that the two main problems – dimensionality and repetition – still need to be solved.

---

### Conclusion

In the next post, I'm going to discuss a few approaches I'm exploring that might help. My aim is to train with simpler audio material, such as drum & mallet samples, although initially I'm going to run tests with artificial waveforms. The short-term goal is to create a system that can generate a __relatively clean__, __smooth__, but __dynamic__ audio sequence, which can then be used as an instrument, rather than a finished audio track.




[^1]: [http://arxiv.org/abs/1308.0850](http://arxiv.org/abs/1308.0850)
[^2]: [http://arxiv.org/abs/1511.06434](http://arxiv.org/abs/1511.06434)
[^3]: [ftp://ftp.idsia.ch/pub/juergen/2002_icannMusic.pdf](ftp://ftp.idsia.ch/pub/juergen/2002_icannMusic.pdf)
[^4]: [http://arxiv.org/pdf/1303.5778](http://arxiv.org/pdf/1303.5778)
[^5]: [https://www.cs.toronto.edu/~kriz/cifar.html](https://www.cs.toronto.edu/~kriz/cifar.html)
[^6]: [http://www.cs.toronto.edu/~graves/nn_2005.pdf](http://www.cs.toronto.edu/~graves/nn_2005.pdf)
[^7]: [https://cs224d.stanford.edu/reports/NayebiAran.pdf](https://cs224d.stanford.edu/reports/NayebiAran.pdf)