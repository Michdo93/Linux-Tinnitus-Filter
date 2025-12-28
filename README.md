# Linux-Tinnitus-Filter

A simple filter for tinnitus therapy for Linux systems (tested on Ubuntu). You can implement both notch therapy or peaking therapy.

> [!WARNING]  
> You should definitely see an ear doctor and discuss whether you need filter therapy for tinnitus! Organic damage, stress, psychological and other health problems can also be the cause! In addition, a hearing test is necessary to determine the frequencies or frequency ranges that need to be either filtered out (notch) or amplified (peaking). Your doctor should also decide which therapy would be best for you individually!

## Peaking therapy vs. Notch therapy

### Method 1: Peaking (Bell) filter

The idea behind this is that tinnitus sounds are caused by hearing impairments, such as after sudden hearing loss, and the whistling sound occurs because the ear can no longer perceive these noises well enough. In this case, the measured frequencies would need to be amplified.

In audio technology, a so-called peaking filter (also known as a bell filter, because the amplification curve looks like a bell) is used for this purpose.

- **Function**: Boosts a selected frequency range in a bell-shaped curve.
- **Purpose**: To compensate for hearing loss (hearing aid function) or to mask tinnitus with a more pleasant background noise in the same pitch.
- **Sound**: Music or ambient noise sounds “brighter” or “more present” in this range.

### Method 2: Notch (Removing) filter

The underlying idea here is that the nerve pathways are essentially going haywire. This can be caused by stress, but it can also have many other physical and psychological causes. You can think of it as a kind of sensory overload. Ergo, the solution here is to reduce these measured frequencies as much as possible or eliminate them entirely in order to relieve the burden on the auditory system.

- **Function**: Creates an extremely narrow and deep “notch” in the frequency spectrum.
- **Purpose**: The tinnitus frequency is removed from the audio signal to “wean” the brain off it.
- **Sound**: When listening to music, you will hardly notice this filter, as it only removes a tiny fraction of the sound.

### Why you often need both

In cases of “quite severe” tinnitus, there is often a combination of factors:

1. You have a hearing dip (mild hearing loss) at a certain frequency. Here, the peaking filter is used to compensate for the deficit.
2. At the same time, you want to calm the overactivity of the nerve cells. Here, you use the notch filter directly on the tinnitus frequency.












    
