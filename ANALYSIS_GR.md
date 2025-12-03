# Πλήρης Ανάλυση της Εφαρμογής SonoBus

## Εισαγωγή

Το SonoBus είναι μια εφαρμογή για streaming ήχου χαμηλής καθυστέρησης (low-latency audio streaming) που επιτρέπει σε μουσικούς να παίζουν μαζί από απόσταση. Η εφαρμογή είναι χτισμένη με το JUCE framework (C++) και χρησιμοποιεί το πρωτόκολλο AOO (Audio Over OSC) για τη μετάδοση ήχου μέσω δικτύου.

---

## 1. Αρχιτεκτονική Επισκόπηση

### 1.1 Βασική Δομή

Η εφαρμογή ακολουθεί το μοντέλο **Model-View-Controller (MVC)**:

- **Model (Μοντέλο)**: `SonobusAudioProcessor` - Διαχειρίζεται όλη τη λογική επεξεργασίας ήχου και δικτύου
- **View (Προβολή)**: `SonobusAudioProcessorEditor` - Διαχειρίζεται το γραφικό περιβάλλον χρήστη (GUI)
- **Controller (Ελεγκτής)**: Οι διάφορες view classes που συνδέουν το UI με το processor

### 1.2 Κύρια Αρχεία

```
Source/
├── SonobusPluginProcessor.h/cpp    # Κεντρικός επεξεργαστής ήχου
├── SonobusPluginEditor.h/cpp       # Κύριο γραφικό περιβάλλον
├── ConnectView.h/cpp               # Οθόνη σύνδεσης
├── PeersContainerView.h/cpp        # Προβολή των συνδεδεμένων peers
├── ChannelGroup.h/cpp              # Επεξεργασία καναλιών ομάδων
├── EffectParams.h                  # Παράμετροι effects
├── Soundboard.h/cpp                # Λειτουργία soundboard
├── Metronome.h/cpp                 # Μετρονόμος
└── ...
```

---

## 2. SonobusAudioProcessor - Ο Κεντρικός Επεξεργαστής

### 2.1 Τι Κάνει

Το `SonobusAudioProcessor` είναι η καρδιά της εφαρμογής. Είναι μια κλάση που κληρονομεί από το `AudioProcessor` του JUCE και είναι υπεύθυνη για:

1. **Επεξεργασία ήχου**: Λαμβάνει ήχο από το input, το επεξεργάζεται, και το στέλνει στο output
2. **Δικτυακή επικοινωνία**: Στέλνει και δέχεται ήχο από άλλους χρήστες μέσω UDP
3. **Διαχείριση peers**: Διαχειρίζεται όλους τους συνδεδεμένους χρήστες
4. **Effects και επεξεργασία**: Εφαρμόζει reverb, compressor, EQ, κλπ.
5. **Recording**: Ηχογραφεί το mix ή μεμονωμένους χρήστες

### 2.2 Βασικές Δομές Δεδομένων

#### AooServerConnectionInfo
```cpp
struct AooServerConnectionInfo {
    String userName;        // Όνομα χρήστη
    String userPassword;    // Κωδικός χρήστη
    String groupName;       // Όνομα ομάδας
    String groupPassword;   // Κωδικός ομάδας
    bool groupIsPublic;     // Αν η ομάδα είναι δημόσια
    String serverHost;      // Διεύθυνση server
    int serverPort;         // Port server
    int64 timestamp;        // Χρονική σήμανση
}
```
Αυτή η δομή αποθηκεύει όλες τις πληροφορίες που χρειάζονται για να συνδεθείς σε έναν server.

#### RemotePeer
Κάθε συνδεδεμένος χρήστης (peer) αντιπροσωπεύεται από ένα `RemotePeer` object που περιέχει:
- Πληροφορίες σύνδεσης (host, port)
- Audio sources/sinks για αποστολή/παραλαβή ήχου
- Παράμετροι επίπεδου, panning, effects
- Στατιστικά δικτύου (latency, packet loss, κλπ.)

#### ChannelGroup
Ομάδες καναλιών που επιτρέπουν την ομαδοποίηση πολλαπλών καναλιών για κοινή επεξεργασία:
- Gain, panning
- Compressor, Expander, EQ, Limiter
- Reverb send
- Monitor delay

### 2.3 Audio Processing Pipeline

Η ροή επεξεργασίας ήχου ακολουθεί αυτή τη σειρά:

```
1. INPUT (από soundcard)
   ↓
2. Input Metering (μέτρηση επιπέδου)
   ↓
3. Input Channel Groups (ομαδοποίηση και επεξεργασία)
   ├─→ Gain/Pan
   ├─→ Compressor/Expander/Limiter
   ├─→ EQ
   └─→ Polarity Invert
   ↓
4. Input Reverb Send (αν ενεργό)
   ↓
5. Main Mix Buffer (συνδυασμός όλων των inputs)
   ├─→ Metronome (αν ενεργό)
   ├─→ File Playback (αν παίζει αρχείο)
   ├─→ Soundboard (αν παίζει)
   └─→ Remote Peers (ήχος από άλλους χρήστες)
   ↓
6. Main Reverb (αν ενεργό)
   ↓
7. Output Metering
   ↓
8. OUTPUT (προς soundcard)
```

### 2.4 Δικτυακή Επικοινωνία

Το SonoBus χρησιμοποιεί το **AOO (Audio Over OSC)** πρωτόκολλο για τη μετάδοση ήχου. Το πρωτόκολλο βασίζεται σε UDP packets.

#### Threads για Δικτυακή Επικοινωνία

1. **SendThread**: Στέλνει audio packets στους peers
2. **RecvThread**: Δέχεται audio packets από τους peers
3. **EventThread**: Επεξεργάζεται events (συνδέσεις, αποσυνδέσεις, κλπ.)
4. **ServerThread**: Αν τρέχει server mode
5. **ClientThread**: Αν συνδέεται σε server

#### Audio Codecs

Το SonoBus υποστηρίζει δύο τύπους codecs:

1. **PCM (Pulse Code Modulation)**: Ασυμπίεστος ήχος, υψηλή ποιότητα αλλά μεγάλο bandwidth
2. **Opus**: Συμπιεσμένος ήχος, καλή ποιότητα με χαμηλό bandwidth

Οι παράμετροι codec:
- Bitrate (για Opus)
- Bit depth (για PCM)
- Complexity (για Opus)
- Signal type (για Opus)

### 2.5 Jitter Buffer

Το **jitter buffer** είναι ένας buffer που αποθηκεύει incoming audio packets για να αντισταθμίσει:
- Μεταβλητή καθυστέρηση δικτύου (network jitter)
- Πακέτα που φτάνουν εκτός σειράς
- Πακέτα που χάνονται

**Auto-resize modes**:
- `AutoNetBufferModeOff`: Χειροκίνητος έλεγχος
- `AutoNetBufferModeAutoIncreaseOnly`: Αυξάνει μόνο όταν χρειάζεται
- `AutoNetBufferModeAutoFull`: Αυξομειώνει αυτόματα
- `AutoNetBufferModeInitAuto`: Αυτόματη αρχικοποίηση

---

## 3. SonobusAudioProcessorEditor - Το Γραφικό Περιβάλλον

### 3.1 Δομή

Το `SonobusAudioProcessorEditor` είναι το κύριο παράθυρο της εφαρμογής. Κληρονομεί από `AudioProcessorEditor` του JUCE.

### 3.2 Κύρια Components

#### ConnectView
Η οθόνη σύνδεσης που επιτρέπει:
- Σύνδεση σε server (δημόσιο ή ιδιωτικό)
- Direct connection (άμεση σύνδεση peer-to-peer)
- Προβολή δημόσιων ομάδων
- Ιστορικό συνδέσεων

#### PeersContainerView
Προβάλλει όλους τους συνδεδεμένους peers με:
- Meters (επίπεδο send/receive)
- Controls (gain, pan, mute, solo)
- Network stats (latency, packet loss)
- Jitter buffer controls
- Effects controls

#### ChannelGroupsView
Διαχείριση input channel groups:
- Δημιουργία/διαγραφή groups
- Gain, pan, mute, solo
- Effects (compressor, EQ, κλπ.)
- Monitor controls

#### OptionsView
Ρυθμίσεις εφαρμογής:
- Audio device settings
- Network settings
- Recording settings
- UI preferences

### 3.3 Layout System

Το SonoBus χρησιμοποιεί **FlexBox** για responsive layout:
- Προσαρμόζεται σε διαφορετικά μεγέθη παραθύρου
- Narrow mode για μικρές οθόνες
- Full mode για μεγάλες οθόνες

---

## 4. ChannelGroup - Επεξεργασία Ομάδων Καναλιών

### 4.1 Σκοπός

Το `ChannelGroup` είναι μια κλάση που επεξεργάζεται μια ομάδα καναλιών ως ενότητα. Χρησιμοποιείται για:
- Input channel groups (τοπικά inputs)
- Remote peer channel groups (ήχος από peers)

### 4.2 Effects Pipeline

Κάθε ChannelGroup μπορεί να έχει:

1. **Compressor**: Συμπίεση δυναμικού εύρους
   - Threshold, Ratio, Attack, Release
   - Makeup gain

2. **Expander/Gate**: Αύξηση δυναμικού εύρους ή gate
   - Ίδιες παράμετροι με compressor

3. **Parametric EQ**: Ισάριθμη equalization
   - Low shelf, 2 parametric bands, High shelf

4. **Limiter**: Περιορισμός peak levels
   - Fast limiter για προστασία από clipping

5. **Polarity Invert**: Αντιστροφή φάσης

### 4.3 Panning

Υποστηρίζει δύο τύπους panning:

1. **Multi-channel panning**: Για groups με >2 κανάλια
   - Κάθε κανάλι έχει δικό του pan value

2. **Stereo panning**: Για stereo groups
   - Χρησιμοποιεί pan law για smooth transitions

### 4.4 Monitor Delay

Κάθε channel group μπορεί να έχει monitor delay για:
- Αντιστάθμιση network latency
- Synchronization με remote peers

---

## 5. Soundboard - Η Λειτουργία Soundboard

### 5.1 Σκοπός

Το soundboard επιτρέπει την αναπαραγωγή ήχων/samples κατά τη διάρκεια μιας συνεδρίας.

### 5.2 SoundSample

Κάθε sample έχει:
- **File URL**: Το αρχείο ήχου
- **Name**: Όνομα για το UI
- **Button Colour**: Χρώμα κουμπιού
- **Hotkey**: Πλήκτρο για γρήγορη αναπαραγωγή
- **Gain**: Επίπεδο αναπαραγωγής

**Playback Behaviours**:
- `SIMULTANEOUS`: Παίζει μαζί με άλλα samples
- `BACK_TO_BACK`: Σταματάει άλλα samples πριν παίξει
- `BACKGROUND`: Δεν σταματάει από BACK_TO_BACK samples

**End Playback Behaviours**:
- `STOP_AT_END`: Σταματάει στο τέλος
- `LOOP_AT_END`: Loops συνεχώς
- `NEXT_AT_END`: Πηγαίνει στο επόμενο sample

**Button Behaviours**:
- `TOGGLE`: Κάνει toggle play/pause
- `HOLD`: Παίζει όσο κρατιέται
- `ONE_SHOT`: Παίζει μια φορά

### 5.3 SoundboardChannelProcessor

Επεξεργάζεται την αναπαραγωγή samples:
- Διαχείριση πολλαπλών samples ταυτόχρονα
- Routing προς main mix ή specific peers
- Effects processing

---

## 6. Metronome - Ο Μετρονόμος

### 6.1 Σκοπός

Παράγει click sounds για συγχρονισμό.

### 6.2 Λειτουργίες

- **Tempo**: BPM (beats per minute)
- **Beats per Bar**: Πόσα beats σε κάθε measure
- **Gain**: Επίπεδο click
- **Beat/Bar sounds**: Διαφορετικά sounds για beat και bar

### 6.3 Synchronization

Μπορεί να συγχρονιστεί με:
- Host tempo (DAW)
- File playback tempo
- Manual tempo

---

## 7. Recording - Ηχογράφηση

### 7.1 Recording Options

Το SonoBus μπορεί να ηχογραφήσει:

1. **RecordMix**: Ολόκληρο το mix (όλοι οι peers + input)
2. **RecordSelf**: Μόνο το δικό σου input
3. **RecordMixMinusSelf**: Mix χωρίς εσένα
4. **RecordIndividualUsers**: Ξεχωριστό αρχείο για κάθε peer

### 7.2 File Formats

Υποστηρίζει:
- **FLAC**: Lossless compression
- **WAV**: Uncompressed
- **OGG**: Compressed

### 7.3 Threaded Writing

Η ηχογράφηση γίνεται σε ξεχωριστό thread για να μην μπλοκάρει το audio thread.

---

## 8. Effects - Τα Effects

### 8.1 Main Reverb

Το main reverb εφαρμόζεται στο ολικό mix. Υποστηρίζει 3 μοντέλα:

1. **Freeverb**: Classic reverb algorithm
2. **MVerb**: Modern reverb
3. **Zita**: High-quality reverb

**Parameters**:
- Level (wet/dry mix)
- Size (room size)
- Damping (high-frequency damping)
- Pre-delay (initial delay)

### 8.2 Input Reverb

Ξεχωριστό reverb για input monitoring (δεν στέλνεται στους peers).

### 8.3 Compressor/Expander/Limiter

Υλοποιημένα με **Faust** (functional audio processing language):
- Real-time processing
- Low latency
- High quality

### 8.4 Parametric EQ

7-band parametric EQ:
- Low shelf
- 2 parametric bands
- High shelf

---

## 9. Network Protocol - Το AOO Protocol

### 9.1 Βασικές Έννοιες

**AOO (Audio Over OSC)** είναι ένα πρωτόκολλο που:
- Μεταδίδει ήχο μέσω UDP
- Χρησιμοποιεί OSC (Open Sound Control) για control messages
- Υποστηρίζει πολλαπλούς codecs

### 9.2 Source/Sink Model

- **Source**: Αποστέλλει ήχο (εσύ στέλνεις)
- **Sink**: Δέχεται ήχο (εσύ δέχεσαι)

Κάθε peer έχει:
- Ένα source (για να στέλνει)
- Ένα sink (για να δέχεται)

### 9.3 Server/Client Architecture

**Server Mode**:
- Κάποιος peer τρέχει server
- Άλλοι peers συνδέονται στον server
- Ο server διαχειρίζεται groups και authentication

**Client Mode**:
- Συνδέεσαι σε έναν server
- Μπορείς να join groups
- Ο server διαχειρίζεται peer discovery

### 9.4 Direct Connection

Peer-to-peer σύνδεση χωρίς server:
- Άμεση σύνδεση μεταξύ δύο peers
- Χρησιμοποιεί UDP hole punching αν χρειάζεται

---

## 10. Latency Management - Διαχείριση Καθυστέρησης

### 10.1 Latency Components

Η συνολική καθυστέρηση αποτελείται από:

1. **Audio I/O latency**: Χρόνος από soundcard
2. **Processing latency**: Χρόνος επεξεργασίας
3. **Network latency**: Χρόνος δικτύου (ping)
4. **Jitter buffer**: Buffer time για stability

### 10.2 Latency Measurement

Το SonoBus μετράει latency με:
- **Ping/Pong**: Round-trip time measurement
- **Echo test**: Audio echo για ακριβή μέτρηση
- **One-way latency**: Μετράει και outgoing και incoming latency

### 10.3 Latency Matching

Μπορεί να συγχρονίσει latency με άλλους peers:
- Όλοι οι peers ρυθμίζουν το jitter buffer στο ίδιο value
- Χρήσιμο για συγχρονισμό

---

## 11. State Management - Διαχείριση Κατάστασης

### 11.1 ValueTree

Το SonoBus χρησιμοποιεί **ValueTree** (JUCE) για state management:
- Hierarchical data structure
- Thread-safe access
- Automatic change notifications

### 11.2 Saved State

Αποθηκεύει:
- Connection info (recent connections)
- Peer settings (gain, pan, effects)
- Input channel groups
- Soundboard samples
- UI preferences

### 11.3 Settings Files

Μπορεί να αποθηκεύσει/φορτώσει setup files:
- Όλες οι ρυθμίσεις σε ένα αρχείο
- Χρήσιμο για presets

---

## 12. Threading Model - Μοντέλο Threads

### 12.1 Audio Thread

Το **audio thread** είναι το πιο σημαντικό:
- Τρέχει σε real-time priority
- Δεν πρέπει να μπλοκάρει
- Επεξεργάζεται audio blocks

### 12.2 Network Threads

- **SendThread**: Στέλνει packets
- **RecvThread**: Δέχεται packets
- **EventThread**: Επεξεργάζεται events

### 12.3 UI Thread

- Όλα τα GUI updates
- User interactions
- Timer callbacks

### 12.4 Synchronization

Χρησιμοποιεί:
- **CriticalSection**: Για mutual exclusion
- **ReadWriteLock**: Για read/write access
- **WaitableEvent**: Για thread signaling
- **Atomic**: Για thread-safe variables

---

## 13. Metering - Μέτρηση Επιπέδων

### 13.1 Meter Types

Το SonoBus χρησιμοποιεί **foleys::LevelMeter** για:
- **Input meters**: Επίπεδο input
- **Output meters**: Επίπεδο output
- **Send meters**: Επίπεδο που στέλνεται
- **Receive meters**: Επίπεδο που δέχεσαι
- **Peer meters**: Επίπεδο κάθε peer

### 13.2 Meter Sources

Κάθε meter έχει ένα `LevelMeterSource` που:
- Συλλέγει audio samples
- Υπολογίζει RMS/Peak levels
- Ενημερώνει το UI

---

## 14. Chat System - Σύστημα Chat

### 14.1 SBChatEvent

Κάθε chat message είναι ένα `SBChatEvent`:
- **Type**: Self, User, ή System
- **From**: Ποιος το έστειλε
- **Targets**: Σε ποιους (για private messages)
- **Message**: Το μήνυμα
- **Tags**: Επιπλέον metadata

### 14.2 ChatView

Το UI component για chat:
- Προβολή messages
- Αποστολή messages
- Private messages
- System notifications

---

## 15. UI Components - Components του UI

### 15.1 Custom Components

Το SonoBus έχει custom components:

- **SonoTextButton**: Custom button style
- **SonoDrawableButton**: Button με drawable icons
- **SonoChoiceButton**: Dropdown button
- **SonoLookAndFeel**: Custom look and feel

### 15.2 Layout System

Χρησιμοποιεί **FlexBox** για responsive layout:
- Προσαρμόζεται αυτόματα
- Narrow/Full modes
- Dynamic resizing

---

## 16. File Playback - Αναπαραγωγή Αρχείων

### 16.1 AudioTransportSource

Χρησιμοποιεί `AudioTransportSource` (JUCE) για:
- Φόρτωση audio files
- Playback control (play, pause, stop)
- Seeking
- Loop

### 16.2 Waveform Display

Προβάλλει waveform του αρχείου:
- Visual feedback
- Seeking by clicking

### 16.3 Routing

Το file playback μπορεί να:
- Στείλει στον main mix
- Στείλει σε specific peers
- Monitor delay
- Gain control

---

## 17. Safety Features - Χαρακτηριστικά Ασφαλείας

### 17.1 Safety Muting

Αυτόματο mute όταν:
- Πολύ υψηλό επίπεδο (clipping protection)
- Network issues
- Buffer underruns

### 17.2 IP Blocking

Μπορείς να μπλοκάρεις IP addresses:
- Προστασία από unwanted connections
- Blacklist management

### 17.3 Authentication

Server authentication:
- Username/password
- Group passwords
- Public/private groups

---

## 18. Performance Optimizations - Βελτιστοποιήσεις

### 18.1 Buffer Management

- Dynamic buffer allocation
- Reuse buffers όπου είναι δυνατό
- Minimal allocations στο audio thread

### 18.2 Processing Optimization

- SIMD instructions όπου είναι δυνατό
- Efficient algorithms
- Minimal processing στο audio thread

### 18.3 Network Optimization

- Packet batching
- Efficient codec usage
- Adaptive bitrate

---

## 19. Platform-Specific Code - Κώδικας Ανά Πλατφόρμα

### 19.1 CrossPlatformUtils

Αφηρημένη διεπαφή για platform-specific operations:
- File system access
- Network configuration
- Audio device management

### 19.2 Platform Implementations

- `CrossPlatformUtilsWindows.cpp`
- `CrossPlatformUtilsMac.mm`
- `CrossPlatformUtilsLinux.cpp`
- `CrossPlatformUtilsAndroid.cpp`
- `CrossPlatformUtilsIOS.mm`

---

## 20. Build System - Σύστημα Build

### 20.1 JUCE Project

Χρησιμοποιεί **JUCE Projucer**:
- `.jucer` file για configuration
- Multi-platform support
- Plugin/Standalone modes

### 20.2 CMake

Εναλλακτικά build με CMake:
- `buildcmake.sh` script
- Cross-platform builds

---

## 21. Συμπεράσματα

### 21.1 Βασικές Αρχές

1. **Real-time Audio**: Όλα τα audio operations πρέπει να είναι real-time safe
2. **Thread Safety**: Proper synchronization μεταξύ threads
3. **Low Latency**: Ελαχιστοποίηση καθυστέρησης
4. **User Experience**: Intuitive UI και smooth operation

### 21.2 Αρχιτεκτονική Patterns

- **MVC**: Separation of concerns
- **Observer Pattern**: Change notifications
- **Factory Pattern**: Component creation
- **Singleton Pattern**: Global state (processor)

### 21.3 Best Practices

- Minimal allocations στο audio thread
- Thread-safe data structures
- Efficient algorithms
- Clean code organization

---

## 22. Περαιτέρω Μελέτη

### 22.1 Βασικά Αρχεία για Μελέτη

1. **SonobusPluginProcessor.cpp**: Κεντρική λογική
2. **SonobusPluginEditor.cpp**: UI implementation
3. **ChannelGroup.cpp**: Audio processing
4. **ConnectView.cpp**: Network connection logic

### 22.2 JUCE Documentation

- AudioProcessor class
- AudioBuffer class
- Threading classes
- ValueTree class

### 22.3 AOO Protocol

- aoo library documentation
- Network audio streaming concepts

---

## 23. Συχνές Ερωτήσεις

**Q: Πώς λειτουργεί το jitter buffer;**
A: Αποθηκεύει incoming packets και τα αναπαράγει με σταθερό rate για να αντισταθμίσει network jitter.

**Q: Γιατί χρησιμοποιείται UDP αντί για TCP;**
A: Το UDP έχει χαμηλότερη καθυστέρηση, αλλά χρειάζεται jitter buffer για lost packets.

**Q: Πώς μετράται η latency;**
A: Με ping/pong packets και audio echo tests για ακριβή μέτρηση.

**Q: Μπορώ να έχω πολλά inputs;**
A: Ναι, με channel groups μπορείς να ομαδοποιήσεις πολλαπλά κανάλια.

---

## 24. Επίλογος

Αυτή η ανάλυση καλύπτει τα βασικά στοιχεία της αρχιτεκτονικής του SonoBus. Η εφαρμογή είναι μια πολύπλοκη, real-time audio streaming εφαρμογή που απαιτεί προσεκτικό σχεδιασμό για low latency και stability.

Για πιο προχωρημένη κατανόηση, συνιστάται η μελέτη του πηγαίου κώδικα και των σχολίων μέσα στα αρχεία.

---

*Τελευταία ενημέρωση: 2024*
