# Εξονυχιστική Ανάλυση: Σύνδεση LAN και Μετάδοση Ήχου με PCM στο SonoBus

## Εισαγωγή

Αυτή η ανάλυση εξηγεί βήμα-βήμα πώς δύο υπολογιστές συνδέονται σε LAN και πώς μεταδίδουν ήχο μεταξύ τους χρησιμοποιώντας PCM encoding. Κάθε σημείο του κώδικα εξηγείται αναλυτικά για αρχάριους.

---

## 1. Αρχικοποίηση - Το Ξεκίνημα

### 1.1 Αρχικοποίηση του UDP Socket

Όταν ξεκινάει η εφαρμογή, πρέπει να ανοίξει ένα UDP socket για να μπορεί να στέλνει και να δέχεται packets.

**Κώδικας: `initializeAoo()`**

```cpp
void SonobusAudioProcessor::initializeAoo(int udpPort=0)
{
    // Δημιουργία UDP socket
    mUdpSocket = std::make_unique<DatagramSocket>();
    
    // Άνοιγμα socket - αν udpPort είναι 0, επιλέγει αυτόματα port
    bool bound = mUdpSocket->bindToPort(udpPort == 0 ? DEFAULT_UDP_PORT : udpPort);
    
    if (bound) {
        mUdpLocalPort = mUdpSocket->getBoundPort();
        mLocalIPAddress = IPAddress::getLocalAddress();
    }
}
```

**Εξήγηση:**
- `DatagramSocket`: Κλάση του JUCE για UDP sockets
- `bindToPort()`: Δέσμευση ενός port για να "ακούει" το socket
- Αν `udpPort=0`, το σύστημα επιλέγει ένα διαθέσιμο port
- `mUdpLocalPort`: Αποθηκεύει το port που χρησιμοποιείται
- `mLocalIPAddress`: Η IP διεύθυνση του υπολογιστή

**Παράδειγμα:**
```
Υπολογιστής Α: IP = 192.168.1.100, Port = 11000
Υπολογιστής Β: IP = 192.168.1.101, Port = 11000
```

---

### 1.2 Δημιουργία Network Threads

Το SonoBus χρησιμοποιεί ξεχωριστά threads για:
- **SendThread**: Στέλνει audio packets
- **RecvThread**: Δέχεται audio packets
- **EventThread**: Επεξεργάζεται events (συνδέσεις, αποσυνδέσεις)

**Κώδικας: `initializeAoo()` (συνέχεια)**

```cpp
// Δημιουργία threads
mSendThread = std::make_unique<SendThread>(*this);
mRecvThread = std::make_unique<RecvThread>(*this);
mEventThread = std::make_unique<EventThread>(*this);

// Ξεκίνημα threads
mSendThread->startThread();
mRecvThread->startThread();
mEventThread->startThread();
```

**Εξήγηση:**
- Κάθε thread τρέχει παράλληλα
- `startThread()`: Ξεκινάει το thread
- Threads έχουν `highest priority` για real-time audio

---

## 2. Η Διαδικασία Σύνδεσης - Βήμα-Βήμα

### 2.1 Σενάριο: Υπολογιστής Α θέλει να συνδεθεί με Υπολογιστή Β

**Υπολογιστής Α (Sender):**
- IP: 192.168.1.100
- Port: 11000

**Υπολογιστής Β (Receiver):**
- IP: 192.168.1.101
- Port: 11000

---

### 2.2 Βήμα 1: Καλέσμα `connectRemotePeer()`

Όταν ο χρήστης στο Υπολογιστή Α πατήσει "Connect" και εισάγει τη διεύθυνση του Υπολογιστή Β:

**Κώδικας: `connectRemotePeer()`**

```cpp
int SonobusAudioProcessor::connectRemotePeer(
    const String & host,        // "192.168.1.101"
    int port,                   // 11000
    const String & username,    // "UserA"
    const String & groupname,   // "MyGroup"
    bool reciprocate            // true
)
{
    // Βήμα 1: Βρες ή δημιούργησε EndpointState
    EndpointState * endpoint = findOrAddEndpoint(host, port);
    
    // Βήμα 2: Δημιούργησε RemotePeer object
    RemotePeer * remote = doAddRemotePeerIfNecessary(
        endpoint, 
        AOO_ID_NONE,  // Θα δοθεί ID αργότερα
        username, 
        groupname
    );
    
    // Βήμα 3: Ενεργοποίηση receive
    remote->recvAllow = !mMainRecvMute.get();
    
    // Βήμα 4: Στείλε "invitation" στον remote peer
    bool ret = remote->oursink->invite_source(
        endpoint, 
        0,  // Προσωρινό ID
        endpoint_send  // Callback function για αποστολή
    ) == 1;
    
    if (ret) {
        remote->connected = true;
        remote->invitedPeer = reciprocate;
        
        // Βήμα 5: Ξεκίνημα αποστολής ήχου
        if (!mMainSendMute.get()) {
            remote->sendActive = true;
            remote->oursource->start();
            updateRemotePeerUserFormat(-1, remote);
        }
    }
    
    return ret;
}
```

**Εξήγηση:**
- `findOrAddEndpoint()`: Βρίσκει ή δημιουργεί ένα `EndpointState` για τη διεύθυνση
- `doAddRemotePeerIfNecessary()`: Δημιουργεί ένα `RemotePeer` object που αντιπροσωπεύει τον remote χρήστη
- `invite_source()`: Στέλνει ένα "invitation" packet στον remote peer
- `oursource->start()`: Ξεκινάει την αποστολή audio packets

---

### 2.3 Βήμα 2: Δημιουργία EndpointState

**Κώδικας: `findOrAddEndpoint()`**

```cpp
EndpointState * SonobusAudioProcessor::findOrAddEndpoint(
    const String & host,  // "192.168.1.101"
    int port              // 11000
)
{
    const ScopedLock sl(mEndpointsLock);
    
    // Ψάξε αν υπάρχει ήδη
    for (auto * ep : mEndpoints) {
        if (ep->ipaddr == host && ep->port == port) {
            return ep;
        }
    }
    
    // Αν δεν υπάρχει, δημιούργησε νέο
    auto * endpoint = new EndpointState(host, port);
    endpoint->owner = mUdpSocket.get();
    mEndpoints.add(endpoint);
    
    return endpoint;
}
```

**Εξήγηση:**
- `EndpointState`: Δομή που αποθηκεύει πληροφορίες για ένα remote endpoint
- `mEndpoints`: Λίστα με όλα τα endpoints
- `ScopedLock`: Προστατεύει από race conditions

**Δομή EndpointState:**
```cpp
struct EndpointState {
    DatagramSocket *owner;      // Pointer στο UDP socket
    String ipaddr;               // "192.168.1.101"
    int port;                   // 11000
    int64_t sentBytes;          // Bytes που έχουν σταλεί
    int64_t recvBytes;          // Bytes που έχουν ληφθεί
    struct sockaddr rawaddr;     // Raw socket address
};
```

---

### 2.4 Βήμα 3: Δημιουργία RemotePeer

**Κώδικας: `doAddRemotePeerIfNecessary()`**

```cpp
RemotePeer * SonobusAudioProcessor::doAddRemotePeerIfNecessary(
    EndpointState * endpoint,
    int32_t ourId,
    const String & username,
    const String & groupname
)
{
    // Δημιούργησε νέο RemotePeer
    auto * remote = new RemotePeer(endpoint, ourId);
    remote->userName = username;
    remote->groupName = groupname;
    
    // Δημιούργησε AOO source και sink
    remote->oursink.reset(aoo::isink::create(ourId));
    remote->oursource.reset(aoo::isource::create(ourId));
    
    // Προσθήκη στη λίστα
    {
        const ScopedWriteLock slw(mCoreLock);
        mRemotePeers.add(remote);
    }
    
    return remote;
}
```

**Εξήγηση:**
- `RemotePeer`: Κεντρική δομή για κάθε συνδεδεμένο peer
- `aoo::isink`: Αντικείμενο που δέχεται audio από remote peer
- `aoo::isource`: Αντικείμενο που στέλνει audio στον remote peer
- `mRemotePeers`: Λίστα με όλους τους συνδεδεμένους peers

**Δομή RemotePeer (συντομία):**
```cpp
struct RemotePeer {
    EndpointState * endpoint;           // Πού βρίσκεται
    int32_t ourId;                      // ID μας
    int32_t remoteSourceId;             // ID του remote source
    int32_t remoteSinkId;               // ID του remote sink
    aoo::isink::pointer oursink;        // Για να δέχεσαι
    aoo::isource::pointer oursource;     // Για να στέλνεις
    bool sendActive;                    // Αν στέλνεις
    bool recvActive;                    // Αν δέχεσαι
    int formatIndex;                    // Ποιο codec (PCM/Opus)
    int sendChannels;                   // Πόσα κανάλια στέλνεις
    int recvChannels;                   // Πόσα κανάλια δέχεσαι
    float buffertimeMs;                 // Jitter buffer time
    // ... και πολλά άλλα
};
```

---

### 2.5 Βήμα 4: Το "Invitation" Packet

Όταν καλείται `invite_source()`, στέλνεται ένα OSC packet μέσω UDP:

**Κώδικας: `endpoint_send()` (callback function)**

```cpp
static int32_t endpoint_send(void *e, const char *data, int32_t size)
{
    EndpointState * endpoint = static_cast<EndpointState*>(e);
    
    // Στείλε μέσω UDP socket
    int result = endpoint->owner->write(
        endpoint->ipaddr,  // "192.168.1.101"
        endpoint->port,     // 11000
        data,              // Τα bytes του packet
        size               // Μέγεθος σε bytes
    );
    
    if (result > 0) {
        endpoint->sentBytes += result + UDP_OVERHEAD_BYTES;
    }
    
    return result;
}
```

**Εξήγηση:**
- `endpoint->owner`: Το UDP socket
- `write()`: Στέλνει UDP packet
- Το packet περιέχει OSC messages που λένε "θέλω να συνδεθώ"

**Παράδειγμα Packet:**
```
OSC Address: /aoo/sink/invite
Arguments:
  - Source ID: 0 (προσωρινό)
  - Format info: (θα δοθεί αργότερα)
```

---

### 2.6 Βήμα 5: Λήψη του Invitation στον Υπολογιστή Β

**Κώδικας: `doReceiveData()` (RecvThread)**

```cpp
void SonobusAudioProcessor::doReceiveData()
{
    char buf[AOO_MAXPACKETSIZE];  // Buffer για το packet
    String senderIP;
    int senderPort;
    
    // Διάβασε packet από UDP socket
    int nbytes = mUdpSocket->read(
        buf,                    // Πού να αποθηκευτεί
        AOO_MAXPACKETSIZE,      // Μέγιστο μέγεθος
        false,                  // Non-blocking
        senderIP,               // Από πού ήρθε
        senderPort              // Port
    );
    
    if (nbytes <= 0) return;
    
    // Βρες το endpoint
    EndpointState * endpoint = findOrAddEndpoint(senderIP, senderPort);
    endpoint->recvBytes += nbytes + UDP_OVERHEAD_BYTES;
    
    // Parse το packet
    int32_t type, id;
    if (aoo_parse_pattern(buf, nbytes, &type, &id) > 0) {
        // Είναι AOO packet
        if (type == AOO_TYPE_SINK) {
            // Forward στο matching sink
            for (auto & remote : mRemotePeers) {
                if (remote->oursink->handle_message(
                    buf, nbytes, endpoint, endpoint_send
                )) {
                    remote->dataPacketsReceived += 1;
                    break;
                }
            }
        }
    }
}
```

**Εξήγηση:**
- `mUdpSocket->read()`: Διαβάζει UDP packet (non-blocking)
- `aoo_parse_pattern()`: Ελέγχει αν είναι AOO packet
- `handle_message()`: Επεξεργάζεται το message

---

### 2.7 Βήμα 6: Αποδοχή της Σύνδεσης

Όταν ο Υπολογιστής Β δέχεται το invitation:

**Κώδικας: `handleSinkEvents()` - AOO_SOURCE_ADD_EVENT**

```cpp
case AOO_SOURCE_ADD_EVENT:
{
    aoo_source_event *e = (aoo_source_event *)events[i];
    EndpointState * es = (EndpointState *)e->endpoint;
    
    RemotePeer * peer = findRemotePeer(es, sinkId);
    if (peer) {
        // Κάποιος μας πρόσθεσε, άρα αποδέχτηκε το invitation
        peer->remoteSourceId = e->id;
        
        // Αποδέξου τη σύνδεση
        if (peer->recvAllow) {
            peer->oursink->invite_source(
                es, 
                peer->remoteSourceId, 
                endpoint_send
            );
            peer->recvActive = true;
        }
    }
    break;
}
```

**Εξήγηση:**
- `AOO_SOURCE_ADD_EVENT`: Event που λέει "κάποιος πρόσθεσε source"
- `peer->remoteSourceId`: Το ID του remote source
- `invite_source()`: Αποδέχεται τη σύνδεση

---

## 3. Ρύθμιση PCM Encoding

### 3.1 Επιλογή PCM Format

**Κώδικας: `setupSourceFormat()`**

```cpp
void SonobusAudioProcessor::setupSourceFormat(
    RemotePeer * peer, 
    aoo::isource * source, 
    bool latencymode
)
{
    // Επίλεξε format (PCM ή Opus)
    int formatIndex = (!peer || peer->formatIndex < 0) 
        ? mDefaultAudioFormatIndex 
        : peer->formatIndex;
    
    if (formatIndex < 0 || formatIndex >= mAudioFormats.size()) {
        formatIndex = 4; // Emergency default
    }
    
    const AudioCodecFormatInfo & info = mAudioFormats.getReference(formatIndex);
    
    // Μετατρέψε σε AOO format
    aoo_format_storage f;
    int channels = latencymode ? 1 : peer ? peer->sendChannels : 2;
    
    if (formatInfoToAooFormat(info, channels, f)) {
        source->set_format(f.header);
    }
}
```

**Εξήγηση:**
- `formatIndex`: Ποιο format να χρησιμοποιήσει (0=PCM 16-bit, 1=PCM 24-bit, κλπ.)
- `formatInfoToAooFormat()`: Μετατρέπει τις πληροφορίες σε AOO format structure

---

### 3.2 Μετατροπή σε AOO PCM Format

**Κώδικας: `formatInfoToAooFormat()`**

```cpp
bool SonobusAudioProcessor::formatInfoToAooFormat(
    const AudioCodecFormatInfo & info,  // PCM info
    int channels,                       // 1 (mono) ή 2 (stereo)
    aoo_format_storage & retformat      // Output
)
{
    if (info.codec == CodecPCM) {
        // Δημιούργησε PCM format structure
        aoo_format_pcm *fmt = (aoo_format_pcm *)&retformat;
        
        // Βασικές πληροφορίες
        fmt->header.codec = AOO_CODEC_PCM;
        fmt->header.blocksize = currSamplesPerBlock >= info.min_preferred_blocksize 
            ? currSamplesPerBlock 
            : info.min_preferred_blocksize;
        fmt->header.samplerate = getSampleRate();  // π.χ. 48000 Hz
        fmt->header.nchannels = channels;          // 1 ή 2
        
        // Bit depth (πόσα bits ανά sample)
        if (info.bitdepth == 2) {
            fmt->bitdepth = AOO_PCM_INT16;  // 16-bit signed integer
        } else if (info.bitdepth == 3) {
            fmt->bitdepth = AOO_PCM_INT24;  // 24-bit signed integer
        } else if (info.bitdepth == 4) {
            fmt->bitdepth = AOO_PCM_FLOAT32; // 32-bit float
        } else if (info.bitdepth == 8) {
            fmt->bitdepth = AOO_PCM_FLOAT64; // 64-bit float
        } else {
            fmt->bitdepth = AOO_PCM_INT16;   // Default
        }
        
        return true;
    }
    
    return false;
}
```

**Εξήγηση:**
- `AOO_CODEC_PCM`: Σταθερά που λέει "χρησιμοποιώ PCM"
- `blocksize`: Πόσα samples ανά block (π.χ. 256 samples)
- `samplerate`: Sample rate (π.χ. 48000 Hz)
- `nchannels`: Πόσα κανάλια (1=mono, 2=stereo)
- `bitdepth`: Bit depth (16-bit, 24-bit, float32, κλπ.)

**Παράδειγμα:**
```
Format: PCM 16-bit, Stereo, 48kHz, 256 samples/block
- Codec: PCM
- Bit depth: 16-bit integer
- Channels: 2 (stereo)
- Sample rate: 48000 Hz
- Block size: 256 samples
```

---

### 3.3 Ρύθμιση του Source

**Κώδικας: `setupSourceFormatsForAll()`**

```cpp
void SonobusAudioProcessor::setupSourceFormatsForAll()
{
    double sampleRate = getSampleRate();  // π.χ. 48000.0
    int sendChannels = peer->sendChannels; // π.χ. 2 (stereo)
    
    for (auto peer : mRemotePeers) {
        if (peer->oursource) {
            // Ρύθμισε το format
            setupSourceFormat(peer, peer->oursource.get());
            
            // Ρύθμισε το source
            peer->oursource->setup(
                sampleRate,        // 48000.0
                currSamplesPerBlock, // 256
                sendChannels        // 2
            );
            
            // Ρύθμισε buffer size
            float sendbufsize = jmax(10.0, 
                SENDBUFSIZE_SCALAR * 1000.0f * currSamplesPerBlock / getSampleRate()
            );
            peer->oursource->set_buffersize(sendbufsize);
        }
    }
}
```

**Εξήγηση:**
- `setup()`: Ρυθμίζει sample rate, block size, channels
- `set_buffersize()`: Ρυθμίζει το μέγεθος του send buffer (σε ms)

**Παράδειγμα:**
```
Sample rate: 48000 Hz
Block size: 256 samples
Channels: 2 (stereo)
Buffer size: 20 ms

Buffer size σε samples = 48000 * 0.020 = 960 samples
```

---

## 4. Η Επεξεργασία Ήχου - processBlock()

### 4.1 Το Audio Thread

Το `processBlock()` καλείται από το audio thread κάθε φορά που χρειάζεται νέος ήχος.

**Κώδικας: `processBlock()` (συντομία)**

```cpp
void SonobusAudioProcessor::processBlock(
    AudioBuffer<float>& buffer,  // Input/Output buffer
    MidiBuffer& midiMessages
)
{
    int numSamples = buffer.getNumSamples();  // π.χ. 256 samples
    
    // 1. Επεξεργασία input
    // 2. Στέλνει στους remote peers
    // 3. Δέχεται από remote peers
    // 4. Mix όλα μαζί
    // 5. Output
}
```

**Εξήγηση:**
- `buffer`: Περιέχει τα audio samples (float, -1.0 έως 1.0)
- `numSamples`: Πόσα samples να επεξεργαστεί (π.χ. 256)
- Καλείται σε real-time (π.χ. κάθε 5.3ms για 256 samples @ 48kHz)

---

### 4.2 Επεξεργασία Input

**Κώδικας: `processBlock()` - Input Processing**

```cpp
// 1. Πάρε input από soundcard
auto totalInputChannels = getTotalNumInputChannels();  // π.χ. 2 (stereo)
float inGain = mInGain.get();  // π.χ. 1.0 (no gain)

// 2. Επεξεργασία input channel groups
for (int i = 0; i < mInputChannelGroupCount; ++i) {
    mInputChannelGroups[i].processBlock(
        buffer,              // Input από soundcard
        inputPostBuffer,     // Output buffer
        destch,              // Destination channel
        numChannels,         // Πόσα κανάλια
        numSamples,          // 256 samples
        inGain               // Gain
    );
}
```

**Εξήγηση:**
- `buffer`: Περιέχει input από soundcard
- `inputPostBuffer`: Περιέχει επεξεργασμένο input
- `processBlock()`: Εφαρμόζει gain, panning, effects

---

### 4.3 Αποστολή Ήχου στους Remote Peers

**Κώδικας: `processBlock()` - Sending**

```cpp
// Για κάθε remote peer
for (auto & remote : mRemotePeers) {
    if (remote->sendActive && remote->oursource) {
        // 1. Προετοιμασία buffer
        workBuffer.clear(0, numSamples);
        
        // 2. Αντιγραφή input στο work buffer
        for (int ch = 0; ch < remote->sendChannels; ++ch) {
            workBuffer.copyFrom(
                ch, 0,                    // Destination
                sendWorkBuffer, ch, 0,    // Source
                numSamples                // 256 samples
            );
        }
        
        // 3. Στείλε μέσω AOO source
        double t = Time::getMillisecondCounterHiRes() / 1000.0;
        remote->oursource->process(
            (const float **)workBuffer.getArrayOfReadPointers(),
            numSamples,  // 256
            t            // Timestamp
        );
    }
}
```

**Εξήγηση:**
- `workBuffer`: Προσωρινό buffer με audio samples
- `oursource->process()`: Στέλνει τα samples στο AOO source
- Το AOO source κωδικοποιεί (PCM) και στέλνει μέσω UDP

---

### 4.4 Κωδικοποίηση PCM

**Εσωτερικά στο AOO library:**

```cpp
// Ψευδο-κώδικας για το πώς το AOO κωδικοποιεί PCM

void aoo_source_process(float **samples, int numSamples) {
    // 1. Μετατροπή float σε integer (για 16-bit PCM)
    int16_t *pcmData = new int16_t[numSamples * numChannels];
    
    for (int i = 0; i < numSamples * numChannels; ++i) {
        // Float [-1.0, 1.0] -> Integer [-32768, 32767]
        float sample = samples[channel][i];
        pcmData[i] = (int16_t)(sample * 32767.0f);
    }
    
    // 2. Δημιουργία OSC packet
    char packet[AOO_MAXPACKETSIZE];
    // ... δημιουργία OSC message με PCM data
    
    // 3. Στείλε μέσω UDP
    send_udp_packet(packet, size);
}
```

**Εξήγηση:**
- Float samples ([-1.0, 1.0]) μετατρέπονται σε integer
- Για 16-bit: [-32768, 32767]
- Τα data τοποθετούνται σε OSC packet
- Το packet στέλνεται μέσω UDP

**Παράδειγμα:**
```
Input: float samples [0.5, -0.3, 0.8, ...]
PCM 16-bit: [16383, -9830, 26214, ...]
Packet: OSC message με PCM data
UDP: Στέλνεται στο 192.168.1.101:11000
```

---

### 4.5 Αποστολή μέσω SendThread

**Κώδικας: `SendThread::run()`**

```cpp
void SonobusAudioProcessor::SendThread::run()
{
    setPriority(Thread::Priority::highest);
    
    while (!threadShouldExit()) {
        // Περίμενε να υπάρχει κάτι να στείλεις
        _processor.mSendWaitable.wait(20);
        
        // Στείλε όλα τα pending packets
        _processor.doSendData();
    }
}
```

**Κώδικας: `doSendData()`**

```cpp
void SonobusAudioProcessor::doSendData()
{
    const ScopedReadLock sl(mCoreLock);
    
    // Για κάθε remote peer
    for (auto & remote : mRemotePeers) {
        if (remote->oursource) {
            // Στείλε pending packets
            int32_t didsend = remote->oursource->send();
            
            if (didsend) {
                remote->dataPacketsSent += 1;
            }
        }
    }
}
```

**Εξήγηση:**
- `SendThread`: Thread που στέλνει packets
- `doSendData()`: Στέλνει όλα τα pending packets
- `oursource->send()`: Στέλνει ένα packet μέσω UDP

---

## 5. Λήψη Ήχου από Remote Peer

### 5.1 Λήψη UDP Packets

**Κώδικας: `RecvThread::run()`**

```cpp
void SonobusAudioProcessor::RecvThread::run()
{
    setPriority(Thread::Priority::highest);
    
    while (!threadShouldExit()) {
        // Περίμενε να υπάρχει packet
        if (_processor.mUdpSocket->waitUntilReady(true, 20) == 1) {
            // Διάβασε το packet
            _processor.doReceiveData();
        }
    }
}
```

**Εξήγηση:**
- `waitUntilReady()`: Περιμένει να υπάρχει packet (timeout 20ms)
- `doReceiveData()`: Διαβάζει και επεξεργάζεται το packet

---

### 5.2 Επεξεργασία Received Packets

**Κώδικας: `doReceiveData()` (λεπτομερώς)**

```cpp
void SonobusAudioProcessor::doReceiveData()
{
    char buf[AOO_MAXPACKETSIZE];
    String senderIP;
    int senderPort;
    
    // 1. Διάβασε UDP packet
    int nbytes = mUdpSocket->read(
        buf, AOO_MAXPACKETSIZE, false, senderIP, senderPort
    );
    
    if (nbytes <= 0) return;
    
    // 2. Βρες το endpoint
    EndpointState * endpoint = findOrAddEndpoint(senderIP, senderPort);
    endpoint->recvBytes += nbytes + UDP_OVERHEAD_BYTES;
    
    // 3. Parse το packet
    int32_t type, id;
    if (aoo_parse_pattern(buf, nbytes, &type, &id) > 0) {
        if (type == AOO_TYPE_SINK) {
            // Είναι audio data packet
            const ScopedReadLock sl(mCoreLock);
            
            for (auto & remote : mRemotePeers) {
                if (!remote->oursink) continue;
                
                // Επεξεργάσου το message
                if (remote->oursink->handle_message(
                    buf, nbytes, endpoint, endpoint_send
                )) {
                    remote->dataPacketsReceived += 1;
                    
                    if (remote->recvAllow && !remote->recvActive) {
                        remote->recvActive = true;
                    }
                    break;
                }
            }
        }
    }
}
```

**Εξήγηση:**
- `read()`: Διαβάζει UDP packet
- `aoo_parse_pattern()`: Ελέγχει αν είναι AOO packet
- `handle_message()`: Επεξεργάζεται το message (αποκωδικοποιεί PCM)

---

### 5.3 Αποκωδικοποίηση PCM

**Εσωτερικά στο AOO library:**

```cpp
// Ψευδο-κώδικας για το πώς το AOO αποκωδικοποιεί PCM

bool aoo_sink_handle_message(char *packet, int size) {
    // 1. Parse OSC message
    // 2. Εξαγωγή PCM data
    int16_t *pcmData = extract_pcm_data(packet);
    int numSamples = extract_num_samples(packet);
    int numChannels = extract_num_channels(packet);
    
    // 3. Μετατροπή integer σε float
    float *floatSamples = new float[numSamples * numChannels];
    
    for (int i = 0; i < numSamples * numChannels; ++i) {
        // Integer [-32768, 32767] -> Float [-1.0, 1.0]
        floatSamples[i] = (float)pcmData[i] / 32767.0f;
    }
    
    // 4. Προσθήκη στο jitter buffer
    jitter_buffer_write(floatSamples, numSamples, numChannels);
    
    return true;
}
```

**Εξήγηση:**
- Parse OSC message
- Εξαγωγή PCM data (integer samples)
- Μετατροπή σε float
- Προσθήκη στο jitter buffer

---

### 5.4 Ανάγνωση από Jitter Buffer

**Κώδικας: `processBlock()` - Receiving**

```cpp
// Για κάθε remote peer
for (auto & remote : mRemotePeers) {
    if (remote->recvActive && remote->oursink) {
        // 1. Διάβασε από jitter buffer
        workBuffer.clear(0, numSamples);
        
        double t = Time::getMillisecondCounterHiRes() / 1000.0;
        bool gotData = remote->oursink->process(
            (float **)workBuffer.getArrayOfWritePointers(),
            numSamples,  // 256
            t            // Timestamp
        );
        
        if (gotData) {
            // 2. Επεξεργασία (gain, panning, effects)
            float gain = remote->gain;
            
            // 3. Προσθήκη στο main mix
            for (int ch = 0; ch < remote->recvChannels; ++ch) {
                tempBuffer.addFrom(
                    ch, 0,                    // Destination
                    workBuffer, ch, 0,         // Source
                    numSamples,                // 256
                    gain                       // Gain
                );
            }
        }
    }
}
```

**Εξήγηση:**
- `oursink->process()`: Διαβάζει από jitter buffer
- `gotData`: Αν υπάρχουν διαθέσιμα samples
- `tempBuffer`: Προσωρινό buffer για remote audio
- Προστίθεται στο main mix

---

## 6. Το Jitter Buffer - Πώς Λειτουργεί

### 6.1 Σκοπός

Το jitter buffer αντισταθμίζει:
- Network jitter (μεταβλητή καθυστέρηση)
- Packets που φτάνουν εκτός σειράς
- Lost packets

---

### 6.2 Ρύθμιση Buffer Time

**Κώδικας: `setRemotePeerBufferTime()`**

```cpp
void SonobusAudioProcessor::setRemotePeerBufferTime(
    int index, 
    float bufferMs  // π.χ. 50.0 ms
)
{
    RemotePeer * remote = mRemotePeers.getUnchecked(index);
    remote->buffertimeMs = bufferMs;
    
    if (remote->oursink) {
        // Ρύθμισε το jitter buffer
        remote->oursink->set_buffertime(bufferMs / 1000.0);  // Convert to seconds
    }
}
```

**Εξήγηση:**
- `bufferMs`: Buffer time σε milliseconds (π.χ. 50ms)
- `set_buffertime()`: Ρυθμίζει το jitter buffer

**Παράδειγμα:**
```
Buffer time: 50 ms
Sample rate: 48000 Hz
Buffer size σε samples: 48000 * 0.050 = 2400 samples
```

---

### 6.3 Πώς Λειτουργεί

```
Time →
[Buffer: 2400 samples]

Write (από network):
[████████████████████] ← Νέο packet
[████████████████████] ← Παλιό packet
[████████████████████] ← Παλιότερο packet

Read (για audio thread):
[████████████████████] ← Διαβάζει από εδώ
```

**Εξήγηση:**
- **Write**: Όταν φτάνει packet, προστίθεται στο buffer
- **Read**: Το audio thread διαβάζει από το buffer
- Το buffer "συγκρατεί" samples για stability

---

## 7. Παράδειγμα: Πλήρης Ροή

### 7.1 Σενάριο

**Υπολογιστής Α (192.168.1.100):**
- Παίζει κιθάρα
- Sample rate: 48000 Hz
- Format: PCM 16-bit Stereo
- Buffer: 50 ms

**Υπολογιστής Β (192.168.1.101):**
- Ακούει
- Sample rate: 48000 Hz
- Format: PCM 16-bit Stereo
- Buffer: 50 ms

---

### 7.2 Timeline

**T=0ms: Σύνδεση**
1. Α καλεί `connectRemotePeer("192.168.1.101", 11000)`
2. Δημιουργείται `RemotePeer` object
3. Στέλνεται invitation packet
4. Β δέχεται το packet
5. Β αποδέχεται τη σύνδεση
6. Στέλνεται confirmation packet
7. Α δέχεται confirmation
8. Σύνδεση ολοκληρώθηκε!

---

**T=100ms: Αρχικοποίηση**
1. Α ρυθμίζει PCM format (16-bit, stereo, 48kHz)
2. Στέλνεται format packet στον Β
3. Β ρυθμίζει το sink για PCM
4. Όλα έτοιμα!

---

**T=200ms: Αποστολή Ήχου**
1. **Audio Thread (Α):**
   - `processBlock()` καλείται
   - Διαβάζει 256 samples από soundcard
   - Επεξεργάζεται (gain, effects)
   - Καλεί `oursource->process()`

2. **AOO Source (Α):**
   - Κωδικοποιεί float → PCM 16-bit
   - Δημιουργεί OSC packet
   - Προσθέτει στο send buffer

3. **SendThread (Α):**
   - `doSendData()` καλείται
   - `oursource->send()` στέλνει packet
   - UDP packet στέλνεται στο 192.168.1.101:11000

---

**T=205ms: Λήψη Ήχου**
1. **RecvThread (Β):**
   - `doReceiveData()` καλείται
   - Διαβάζει UDP packet
   - Parse OSC message

2. **AOO Sink (Β):**
   - Αποκωδικοποιεί PCM → float
   - Προσθέτει στο jitter buffer

3. **Audio Thread (Β):**
   - `processBlock()` καλείται
   - `oursink->process()` διαβάζει από buffer
   - Προσθέτει στο main mix
   - Output στο soundcard

---

### 7.3 Παράδειγμα Packet

**UDP Packet Structure:**
```
[UDP Header]
  Source IP: 192.168.1.100
  Source Port: 11000
  Dest IP: 192.168.1.101
  Dest Port: 11000

[OSC Message]
  Address: /aoo/sink/data
  Arguments:
    - Source ID: 1
    - Sequence: 42
    - Timestamp: 123456.789
    - Format: PCM 16-bit Stereo
    - Data: [PCM samples...]
```

**PCM Data:**
```
256 samples * 2 channels * 2 bytes = 1024 bytes
[0x1234, 0x5678, 0x9ABC, ...]  (16-bit integers)
```

---

## 8. Συχνές Ερωτήσεις

**Q: Γιατί UDP και όχι TCP;**
A: Το UDP έχει χαμηλότερη καθυστέρηση, αλλά χρειάζεται jitter buffer για lost packets.

**Q: Τι γίνεται αν χαθεί packet;**
A: Το jitter buffer "συγκρατεί" samples, οπότε μικρές απώλειες δεν είναι αισθητές.

**Q: Πόσο bandwidth χρειάζεται;**
A: Για PCM 16-bit Stereo @ 48kHz: 2 channels * 2 bytes * 48000 = 192 KB/s = 1.5 Mbps

**Q: Πώς επιλέγεται το buffer time;**
A: Αυτόματα με βάση network latency, ή χειροκίνητα από τον χρήστη.

---

## 9. Συμπεράσματα

Η διαδικασία αποτελείται από:

1. **Σύνδεση**: UDP socket, invitation packets
2. **Format Negotiation**: PCM format setup
3. **Αποστολή**: Float → PCM → UDP packets
4. **Λήψη**: UDP packets → PCM → Float → Jitter buffer
5. **Playback**: Jitter buffer → Audio output

Όλα αυτά συμβαίνουν σε real-time με χαμηλή καθυστέρηση!

---

*Τελευταία ενημέρωση: 2024*
