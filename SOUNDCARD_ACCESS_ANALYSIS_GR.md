# Εξονυχιστική Ανάλυση: Πώς το SonoBus Παίρνει Ήχο από την Soundcard

## Εισαγωγή

Αυτή η ανάλυση εξηγεί βήμα-βήμα πώς το SonoBus συνδέεται με την soundcard και λαμβάνει audio samples. Το SonoBus χρησιμοποιεί το **JUCE framework**, το οποίο παρέχει αφηρημένη διεπαφή για audio I/O.

---

## 1. Η Αρχιτεκτονική - JUCE Audio System

### 1.1 Βασικές Έννοιες

Το JUCE παρέχει τρία βασικά components:

1. **AudioDeviceManager**: Διαχειρίζεται την soundcard
2. **AudioProcessorPlayer**: Συνδέει το AudioDeviceManager με το AudioProcessor
3. **AudioProcessor**: Ο επεξεργαστής ήχου (το SonoBus)

**Διάγραμμα:**
```
[Soundcard]
    ↓
[AudioDeviceManager]  ← Διαχειρίζεται soundcard
    ↓
[AudioIODeviceCallback]  ← Callback interface
    ↓
[AudioProcessorPlayer]  ← Μεταφέρει data
    ↓
[AudioProcessor.processBlock()]  ← Επεξεργασία
```

---

## 2. Standalone Mode - Όταν Τρέχει ως Standalone App

### 2.1 Αρχικοποίηση

Όταν το SonoBus τρέχει ως standalone app (όχι plugin), χρησιμοποιεί το `StandalonePluginHolder`:

**Κώδικας: `SonoStandaloneFilterApp::createWindow()`**

```cpp
StandaloneFilterWindow* SonobusStandaloneFilterApp::createWindow()
{
    // Δημιούργησε StandalonePluginHolder
    auto plugh = new StandalonePluginHolder(
        appProperties.getUserSettings(),
        false,  // takeOwnershipOfSettings
        prefDevname,  // preferredDefaultDeviceName
        &setupOptions  // AudioDeviceManager::AudioDeviceSetup
    );
    
    // Δημιούργησε το παράθυρο
    return new StandaloneFilterWindow(plugh);
}
```

**Εξήγηση:**
- `StandalonePluginHolder`: Κλάση που διαχειρίζεται audio device και processor
- `setupOptions`: Ρυθμίσεις για soundcard (sample rate, buffer size, κλπ.)

---

### 2.2 Δημιουργία AudioDeviceManager

**Κώδικας: `StandalonePluginHolder::init()`**

```cpp
void StandalonePluginHolder::init(
    bool enableAudioInput,  // Αν χρειάζεται input
    const String& preferredDefaultDeviceName
)
{
    // Ρύθμισε audio device type (ASIO για Windows, CoreAudio για Mac, κλπ.)
#if JUCE_WINDOWS
    deviceManager.setCurrentAudioDeviceType("ASIO", false);
#endif

    // Ρύθμισε audio devices
    setupAudioDevices(
        enableAudioInput,
        preferredDefaultDeviceName,
        options.get()
    );
    
    // Ξεκίνησε playback
    startPlaying();
}
```

**Εξήγηση:**
- `deviceManager`: Το `AudioDeviceManager` object
- `setCurrentAudioDeviceType()`: Επιλέγει audio driver (ASIO, DirectSound, CoreAudio, ALSA, κλπ.)
- `setupAudioDevices()`: Ρυθμίζει input/output devices

---

### 2.3 Ρύθμιση Audio Devices

**Κώδικας: `StandalonePluginHolder::setupAudioDevices()`**

```cpp
void StandalonePluginHolder::setupAudioDevices(
    bool enableAudioInput,
    const String& preferredDefaultDeviceName,
    const AudioDeviceManager::AudioDeviceSetup* preferredSetupOptions
)
{
    // Προσθήκη callback στο AudioDeviceManager
    deviceManager.addAudioCallback(&maxSizeEnforcer);
    deviceManager.addMidiInputCallback({}, &player);
    
    // Φόρτωσε saved settings
    reloadAudioDeviceState(
        enableAudioInput,
        preferredDefaultDeviceName,
        preferredSetupOptions
    );
}
```

**Εξήγηση:**
- `addAudioCallback()`: Προσθέτει callback που καλείται κάθε φορά που χρειάζεται audio
- `maxSizeEnforcer`: Wrapper που εξασφαλίζει ότι τα blocks δεν είναι πολύ μεγάλα
- `player`: `AudioProcessorPlayer` που συνδέει με το processor

---

### 2.4 Το AudioProcessorPlayer

**Κώδικας: `StandalonePluginHolder` (μέσα στην κλάση)**

```cpp
class StandalonePluginHolder : private AudioIODeviceCallback
{
    AudioDeviceManager deviceManager;
    AudioProcessorPlayer player;  // ← Αυτό συνδέει με το processor
    
    void setupAudioDevices(...)
    {
        // Ρύθμισε το player
        player.setProcessor(processor.get());  // processor = SonobusAudioProcessor
    }
};
```

**Εξήγηση:**
- `AudioProcessorPlayer`: Κλάση του JUCE που "παίζει" ένα AudioProcessor
- `setProcessor()`: Ορίζει ποιο processor να χρησιμοποιήσει

---

## 3. Το AudioIODeviceCallback - Η Σύνδεση με την Soundcard

### 3.1 Η Interface

Το `StandalonePluginHolder` υλοποιεί το `AudioIODeviceCallback`:

**Κώδικας: `StandalonePluginHolder::audioDeviceIOCallbackWithContext()`**

```cpp
void StandalonePluginHolder::audioDeviceIOCallbackWithContext(
    const float* const* inputChannelData,   // ← Audio από soundcard INPUT
    int numInputChannels,                   // ← Πόσα input channels
    float* const* outputChannelData,        // ← Audio προς soundcard OUTPUT
    int numOutputChannels,                  // ← Πόσα output channels
    int numSamples,                         // ← Πόσα samples
    const AudioIODeviceCallbackContext& context
) override
{
    // Αν input είναι muted (για feedback loop protection)
    const bool inputMuted = shouldMuteInput.getValue();
    
    if (inputMuted) {
        emptyBuffer.clear();
        inputChannelData = emptyBuffer.getArrayOfReadPointers();
    }
    
    // Καλέσμα του player, που καλεί το processor->processBlock()
    player.audioDeviceIOCallbackWithContext(
        inputChannelData,    // Audio από soundcard
        numInputChannels,
        outputChannelData,   // Audio προς soundcard
        numOutputChannels,
        numSamples,
        context
    );
}
```

**Εξήγηση:**
- `inputChannelData`: Array of pointers, ένα pointer ανά channel
  - `inputChannelData[0]` = pointer στο channel 0 (αριστερό)
  - `inputChannelData[1]` = pointer στο channel 1 (δεξί)
- `outputChannelData`: Ίδιο, αλλά για output
- `numSamples`: Πόσα samples να επεξεργαστεί (π.χ. 256)

**Παράδειγμα:**
```cpp
// Stereo input
inputChannelData[0] → [0.5, -0.3, 0.8, ...]  // Left channel
inputChannelData[1] → [0.2, 0.1, -0.5, ...]  // Right channel
numInputChannels = 2
numSamples = 256
```

---

### 3.2 Πώς Καλείται

Το `audioDeviceIOCallbackWithContext()` καλείται από το `AudioDeviceManager`:

**Εσωτερικά στο JUCE (ψευδο-κώδικας):**

```cpp
// Μέσα στο AudioDeviceManager
void AudioDeviceManager::audioDeviceIOCallback(
    const float** inputChannelData,
    int numInputChannels,
    float** outputChannelData,
    int numOutputChannels,
    int numSamples
)
{
    // Για κάθε callback που έχει προστεθεί
    for (auto* callback : audioCallbacks) {
        callback->audioDeviceIOCallbackWithContext(
            inputChannelData,
            numInputChannels,
            outputChannelData,
            numOutputChannels,
            numSamples,
            context
        );
    }
}
```

**Εξήγηση:**
- Το `AudioDeviceManager` διαβάζει από soundcard
- Καλεί όλα τα registered callbacks
- Κάθε callback επεξεργάζεται τα samples

---

## 4. AudioProcessorPlayer - Μεταφορά στο Processor

### 4.1 Πώς Λειτουργεί

**Εσωτερικά στο AudioProcessorPlayer (ψευδο-κώδικας):**

```cpp
void AudioProcessorPlayer::audioDeviceIOCallbackWithContext(
    const float** inputChannelData,
    int numInputChannels,
    float** outputChannelData,
    int numOutputChannels,
    int numSamples,
    const AudioIODeviceCallbackContext& context
)
{
    // 1. Δημιούργησε AudioBuffer από input data
    AudioBuffer<float> inputBuffer(
        const_cast<float**>(inputChannelData),
        numInputChannels,
        numSamples
    );
    
    // 2. Δημιούργησε AudioBuffer για output
    AudioBuffer<float> outputBuffer(
        outputChannelData,
        numOutputChannels,
        numSamples
    );
    
    // 3. Καλέσμα του processor->processBlock()
    processor->processBlock(
        inputBuffer,   // ← Περιέχει audio από soundcard
        midiBuffer     // ← MIDI messages (αν υπάρχουν)
    );
    
    // 4. Το outputBuffer τώρα περιέχει επεξεργασμένο audio
    // (το processBlock() γράφει στο buffer)
}
```

**Εξήγηση:**
- `AudioBuffer`: Κλάση του JUCE για audio data
- `processBlock()`: Η κύρια μέθοδος επεξεργασίας
- Το `buffer` parameter είναι **input/output** - διαβάζει από input, γράφει στο output

---

## 5. Το processBlock() - Όπου Παίρνουμε τα Samples

### 5.1 Η Υπογραφή

**Κώδικας: `SonobusAudioProcessor::processBlock()`**

```cpp
void SonobusAudioProcessor::processBlock(
    AudioBuffer<float>& buffer,  // ← INPUT/OUTPUT buffer
    MidiBuffer& midiMessages
)
{
    // Το buffer ΠΕΡΙΕΧΕΙ ήδη audio από soundcard!
    // Δεν χρειάζεται να το διαβάσουμε - είναι ήδη εκεί
    
    int numSamples = buffer.getNumSamples();  // π.χ. 256
    int numChannels = buffer.getNumChannels(); // π.χ. 2 (stereo)
    
    // Διάβασε samples από buffer
    for (int channel = 0; channel < numChannels; ++channel) {
        const float* channelData = buffer.getReadPointer(channel);
        
        // channelData[0] = πρώτο sample
        // channelData[1] = δεύτερο sample
        // ...
        // channelData[255] = τελευταίο sample
    }
}
```

**Εξήγηση:**
- `buffer`: Περιέχει **ήδη** τα samples από soundcard
- `getReadPointer()`: Παίρνει pointer στα samples ενός channel
- Samples είναι float values από -1.0 έως 1.0

---

### 5.2 Παράδειγμα: Διάβασμα Samples

**Κώδικας: `processBlock()` - Input Metering**

```cpp
void SonobusAudioProcessor::processBlock(
    AudioBuffer<float>& buffer,
    MidiBuffer& midiMessages
)
{
    int numSamples = buffer.getNumSamples();
    
    // 1. Μέτρηση input level (πριν από επεξεργασία)
    inputMeterSource.measureBlock(buffer, 0, numSamples);
    
    // 2. Το buffer περιέχει audio από soundcard
    //    π.χ. για stereo:
    //    buffer[0] = [0.5, -0.3, 0.8, ...]  // Left
    //    buffer[1] = [0.2, 0.1, -0.5, ...]  // Right
    
    // 3. Επεξεργασία...
    // ...
    
    // 4. Γράψε output στο buffer
    //    (το buffer είναι input/output)
    for (int ch = 0; ch < numOutputChannels; ++ch) {
        float* outputData = buffer.getWritePointer(ch);
        // outputData τώρα περιέχει output audio
    }
}
```

**Εξήγηση:**
- `measureBlock()`: Μετράει RMS/Peak levels
- `getWritePointer()`: Παίρνει pointer για γραφή
- Το buffer είναι **input/output** - διαβάζεις από input, γράφεις στο output

---

### 5.3 Πλήρες Παράδειγμα

**Κώδικας: `processBlock()` - Input Processing**

```cpp
void SonobusAudioProcessor::processBlock(
    AudioBuffer<float>& buffer,
    MidiBuffer& midiMessages
)
{
    int numSamples = buffer.getNumSamples();  // 256
    auto totalInputChannels = getTotalNumInputChannels();  // 2 (stereo)
    
    // 1. Μέτρηση input
    inputMeterSource.measureBlock(buffer, 0, numSamples);
    
    // 2. Επεξεργασία input channel groups
    for (int i = 0; i < mInputChannelGroupCount; ++i) {
        // Το buffer περιέχει ήδη audio από soundcard
        mInputChannelGroups[i].processBlock(
            buffer,              // ← INPUT από soundcard
            inputPostBuffer,     // ← OUTPUT μετά από επεξεργασία
            destch,
            numChannels,
            numSamples,
            inGain
        );
    }
    
    // 3. Στέλνει στους remote peers
    for (auto & remote : mRemotePeers) {
        if (remote->sendActive) {
            // Αντιγραφή από inputPostBuffer στο workBuffer
            workBuffer.copyFrom(0, 0, inputPostBuffer, 0, 0, numSamples);
            
            // Στείλε μέσω network
            remote->oursource->process(
                (const float **)workBuffer.getArrayOfReadPointers(),
                numSamples,
                t
            );
        }
    }
    
    // 4. Δέχεται από remote peers
    // ... (προσθήκη στο buffer)
    
    // 5. Output στο buffer (για soundcard)
    // Το buffer τώρα περιέχει output audio
}
```

**Εξήγηση:**
- `buffer`: Περιέχει input από soundcard στην αρχή
- `inputPostBuffer`: Περιέχει επεξεργασμένο input
- `buffer`: Περιέχει output προς soundcard στο τέλος

---

## 6. Plugin Mode - Όταν Τρέχει ως Plugin

### 6.1 Η Διαφορά

Όταν το SonoBus τρέχει ως plugin (VST, AU, κλπ.), **δεν** χρησιμοποιεί `AudioDeviceManager`. Αντίθετα, το **host** (DAW) παρέχει τα audio data.

**Διάγραμμα:**
```
[DAW (Reaper, Logic, κλπ.)]
    ↓
[Host Audio Engine]
    ↓
[VST/AU Wrapper]
    ↓
[AudioProcessor.processBlock()]  ← Καλείται από host
```

---

### 6.2 Πώς Καλείται

**Εσωτερικά στο VST/AU wrapper (ψευδο-κώδικας):**

```cpp
// VST process function
void VSTPlugin::processReplacing(
    float** inputs,   // Audio από host
    float** outputs,  // Audio προς host
    int sampleFrames
)
{
    // Δημιούργησε AudioBuffer
    AudioBuffer<float> buffer(
        inputs,   // Input από host
        numInputChannels,
        sampleFrames
    );
    
    // Καλέσμα του processor
    processor->processBlock(buffer, midiBuffer);
    
    // Αντιγραφή output
    for (int ch = 0; ch < numOutputChannels; ++ch) {
        memcpy(outputs[ch], buffer.getReadPointer(ch), sampleFrames * sizeof(float));
    }
}
```

**Εξήγηση:**
- Το host (DAW) διαβάζει από soundcard
- Καλεί το plugin's `processBlock()`
- Το plugin επεξεργάζεται και γράφει output
- Το host στέλνει output στην soundcard

---

## 7. prepareToPlay() - Προετοιμασία

### 7.1 Πότε Καλείται

Το `prepareToPlay()` καλείται **πριν** ξεκινήσει το audio streaming:

**Κώδικας: `SonobusAudioProcessor::prepareToPlay()`**

```cpp
void SonobusAudioProcessor::prepareToPlay(
    double sampleRate,      // π.χ. 48000.0 Hz
    int samplesPerBlock     // π.χ. 256 samples
)
{
    // Αποθήκευση παραμέτρων
    lastSamplesPerBlock = currSamplesPerBlock = samplesPerBlock;
    
    // Πάρε πληροφορίες για channels
    int inchannels = getTotalNumInputChannels();   // π.χ. 2 (stereo)
    int outchannels = getMainBusNumOutputChannels(); // π.χ. 2 (stereo)
    
    // Ρύθμιση effects
    mMetronome->setSampleRate(sampleRate);
    mMainReverb->setSampleRate(sampleRate);
    
    // Ρύθμιση buffers
    ensureBuffers(samplesPerBlock);
    
    // Ρύθμιση network
    setupSourceFormatsForAll();
}
```

**Εξήγηση:**
- Καλείται **μία φορά** πριν ξεκινήσει playback
- Ρυθμίζει sample rate, buffer size, channels
- Προετοιμάζει buffers και effects

---

### 7.2 Πώς Καλείται

**Standalone Mode:**
```cpp
// AudioDeviceManager καλεί audioDeviceAboutToStart()
void StandalonePluginHolder::audioDeviceAboutToStart(
    AudioIODevice* device
)
{
    // Καλέσμα prepareToPlay()
    player.audioDeviceAboutToStart(device);
    // ↑ Αυτό καλεί processor->prepareToPlay()
}
```

**Plugin Mode:**
```cpp
// Host καλεί prepareToPlay() απευθείας
host->prepareToPlay(sampleRate, bufferSize);
// ↑ Καλεί processor->prepareToPlay()
```

---

## 8. Timeline - Πλήρης Ροή

### 8.1 Αρχικοποίηση (T=0ms)

1. **StandalonePluginHolder** δημιουργείται
2. **AudioDeviceManager** αρχικοποιείται
3. **AudioProcessorPlayer** συνδέεται με processor
4. **AudioDeviceManager** ανοίγει soundcard
5. **prepareToPlay()** καλείται
6. Όλα έτοιμα!

---

### 8.2 Audio Streaming (T=5ms, 10ms, 15ms, ...)

**Κάθε 5.3ms (για 256 samples @ 48kHz):**

1. **Soundcard** διαβάζει samples από microphone/line input
2. **AudioDeviceManager** λαμβάνει samples
3. **audioDeviceIOCallbackWithContext()** καλείται
4. **AudioProcessorPlayer** μεταφέρει στο processor
5. **processBlock()** καλείται με audio στο buffer
6. **SonoBus** επεξεργάζεται audio
7. **SonoBus** γράφει output στο buffer
8. **AudioProcessorPlayer** μεταφέρει output
9. **AudioDeviceManager** στέλνει στην soundcard
10. **Soundcard** παίζει από speakers/headphones

---

## 9. Παράδειγμα: Πραγματική Ροή

### 9.1 Σενάριο

- **Soundcard**: Focusrite Scarlett 2i2
- **Input**: Microphone στο channel 1
- **Sample Rate**: 48000 Hz
- **Buffer Size**: 256 samples
- **Channels**: 2 (stereo)

---

### 9.2 Timeline

**T=0ms: Αρχικοποίηση**
```
1. StandalonePluginHolder::init()
   → deviceManager.initialise(2, 2, nullptr, true)
   → deviceManager.setCurrentAudioDeviceType("ASIO")
   → deviceManager.addAudioCallback(&maxSizeEnforcer)

2. audioDeviceAboutToStart()
   → player.audioDeviceAboutToStart(device)
   → processor->prepareToPlay(48000.0, 256)
   → mMetronome->setSampleRate(48000.0)
   → ensureBuffers(256)
```

---

**T=5.3ms: Πρώτο Audio Block**
```
1. Soundcard διαβάζει 256 samples από microphone
   Input: [0.1, 0.2, 0.15, ...] (256 samples)

2. AudioDeviceManager::audioDeviceIOCallback()
   inputChannelData[0] → [0.1, 0.2, 0.15, ...]
   inputChannelData[1] → [0.0, 0.0, 0.0, ...]  (no input on ch2)
   numSamples = 256

3. StandalonePluginHolder::audioDeviceIOCallbackWithContext()
   → player.audioDeviceIOCallbackWithContext(...)

4. AudioProcessorPlayer::audioDeviceIOCallbackWithContext()
   → Δημιουργεί AudioBuffer από inputChannelData
   → processor->processBlock(buffer, midiBuffer)

5. SonobusAudioProcessor::processBlock()
   buffer.getReadPointer(0) → [0.1, 0.2, 0.15, ...]
   buffer.getReadPointer(1) → [0.0, 0.0, 0.0, ...]
   
   // Επεξεργασία...
   inputMeterSource.measureBlock(buffer, 0, 256);
   mInputChannelGroups[0].processBlock(...);
   
   // Στέλνει στους peers...
   remote->oursource->process(...);
   
   // Γράφει output...
   buffer.getWritePointer(0) ← [0.1, 0.2, 0.15, ...]  // Monitor
   buffer.getWritePointer(1) ← [0.0, 0.0, 0.0, ...]

6. AudioProcessorPlayer μεταφέρει output
   outputChannelData[0] ← [0.1, 0.2, 0.15, ...]
   outputChannelData[1] ← [0.0, 0.0, 0.0, ...]

7. Soundcard παίζει από speakers
```

---

## 10. Συχνές Ερωτήσεις

**Q: Πώς ξέρει το SonoBus ποια soundcard να χρησιμοποιήσει;**
A: Το `AudioDeviceManager` διαχειρίζεται αυτό. Ο χρήστης επιλέγει από settings.

**Q: Τι γίνεται αν δεν υπάρχει input;**
A: Το `inputChannelData` περιέχει zeros ή silence.

**Q: Πόσο συχνά καλείται το processBlock();**
A: Κάθε `bufferSize / sampleRate` seconds. Για 256 samples @ 48kHz: 256/48000 = 5.3ms

**Q: Μπορώ να έχω διαφορετικό sample rate;**
A: Ναι, αλλά πρέπει να είναι συμβατό με την soundcard.

**Q: Τι γίνεται αν το processBlock() παίρνει πολύ χρόνο;**
A: Audio dropouts! Γι' αυτό πρέπει να είναι real-time safe.

---

## 11. Συμπεράσματα

Η ροή είναι:

1. **Soundcard** → Διαβάζει από input
2. **AudioDeviceManager** → Λαμβάνει samples
3. **AudioIODeviceCallback** → Καλείται με samples
4. **AudioProcessorPlayer** → Μεταφέρει στο processor
5. **processBlock()** → Επεξεργάζεται audio
6. **processBlock()** → Γράφει output
7. **AudioProcessorPlayer** → Μεταφέρει output
8. **AudioDeviceManager** → Στέλνει στην soundcard
9. **Soundcard** → Παίζει από output

Όλα αυτά συμβαίνουν σε **real-time** με χαμηλή καθυστέρηση!

---

*Τελευταία ενημέρωση: 2024*
