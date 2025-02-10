# mp_quic

```bash
./caddy -quic -mp
```

```bash
env QUIC_GO_LOG_LEVEL=DEBUG python src/AStream/dist/client/dash_client.py -m "https://localhost:4242/output_dash.mpd" -p 'basic' -q -mp
```

mpquic-sbd/src/quic-go/scheduler.go

`scheduler` module is responsible for **MP-QUIC (multipath QUIC) multipath scheduling** in the transport layer. Its main functions are:

1. **Path Selection**: selects appropriate transmission paths according to different scheduling policies.
2. **Retransmission processing**: check and retransmit lost packets.
3. **Flow Control Management**: send Window Update Frames (WUF) to adjust the flow control window.
4. **Packet Sending**: assemble and send packets, including control frames such as ACK, Stop Waiting, Path Closing, etc.

```go
type scheduler struct {
// Stores traffic quotas for each path and manages path loads
quotas map[protocol.PathID]uint
}

func (sch *scheduler) setup() {
sch.quotas = make(map[protocol.PathID]uint)
}
```

```go
func (sch *scheduler) getRetransmission(s *session) (hasRetransmission bool, retransmitPacket *ackhandler.Packet, pth *path) {
// check if retransmission is needed
// ...
return
}
```

This method iterates over all paths, checks for lost packets that need to be retransmitted, and returns the paths and packets that need to be retransmitted.

```go
func (sch *scheduler) selectPathRoundRobin(s *session, hasRetransmission bool, hasStreamRetransmission bool, fromPth *path) *path {
// rr pick path
// ...
return selectedPath
}
```

The method uses a **Round Robin** strategy to select an optimal path from all the paths.

```go
func (sch *scheduler) selectPathLowLatency(s *session, hasRetransmission bool, hasStreamRetransmission bool, fromPth *path) *path {
// base on RTT pick path
// ...
return selectedPath
}
```

This method selects the optimal path based on the **RTT** of the path, prioritizing the path with the lowest **RTT**.

According to  SmoothedRTT()

```go
func (sch *scheduler) selectPath(s *session, hasRetransmission bool, hasStreamRetransmission bool, fromPth *path) *path {
return sch.selectPathLowLatency(s, hasRetransmission, hasStreamRetransmission, fromPth)
// Or rr for path
// return sch.selectPathRoundRobin(s, hasRetransmission, hasStreamRetransmission, fromPth)
}
```

```go
func (sch *scheduler) performPacketSending(s *session, windowUpdateFrames []*wire.WindowUpdateFrame, pth *path) (*ackhandler.Packet, bool, error) {
// send the pkt
// ...
return pkt, true, nil
}
```

- **Parameters**:
  - `s *session`: the current session object.
  - `windowUpdateFrames []*wire.WindowUpdateFrame`: list of flow control window update frames.
  - `pth *path`: the selected transmission path.
- **Return Value**:
  - `ackhandler.Packet`: packet sent.
  - `bool`: whether the packet was successfully sent.
  - `error`: sending error.

This method is responsible for **sending packets**, including **control frames (e.g. `PingFrame`, `AckFrame`) and flow control window update frames**, and returning the send status.

```go
func (sch *scheduler) ackRemainingPaths(s *session, totalWindowUpdateFrames []*wire.WindowUpdateFrame) error {
// Send ACK frame to update flow control window
// ...
return nil
}
```

- **Action**: sends an ACK frame and updates the flow control window when there is no data to send.
- **Parameters**:
  - `s *session`: current session object.
  - `totalWindowUpdateFrames []*wire.WindowUpdateFrame`: list of flow control window update frames.
- **ReturnValue**: return error, if any.

Running Scripts:

Main download:
mpquic-sbd/src/AStream/dist/client/dash_client.py

```python
def start_playback_smart(dp_object, domain, playback_type=None, download=False, video_segment_duration=None):
```

```python
Module that downloads the MPD-FIle and download
        all the representations of the Module to download
        the MPEG-DASH media.
        Example: start_playback_smart(dp_object, domain, "SMART", DOWNLOAD, video_segment_duration)

        :param dp_object:       The DASH-playback object
        :param domain:          The domain name of the server (The segment URLS are domain + relative_address)
        :param playback_type:   The type of playback
                                1. 'BASIC' - The basic adapataion scheme
                                2. 'SMART' - Segment Aware Rate Adaptation
                                3. 'NETFLIX' - Buffer based adaptation used by Netflix
        :param download: Set to True if the segments are to be stored locally (Boolean). Default False
        :param video_segment_duration: Playback duratoin of each segment
        :return:
```

```python
def download_segment(segment_url, dash_folder, download=False):
    """ Module to download the segment with one
        permanent HTTP connection.
        File is not written to disk.
    """
    cropped_fInd = segment_url.rfind('/')
    segment_name = segment_url[cropped_fInd+1:len(segment_url)]

    filename = ""
    if download:
        filename = os.path.join(dash_folder, segment_name)
        print("SAVING IN {0}".format(filename))

    segment_size = glueConnection.download_segment_PM(segment_url)
    if segment_size < 0:
        raise ValueError("invalid segment_size, connection dropped")
    return segment_size, segment_name
```

```python
    glueConnection.setupPM(QUIC, MP, not NO_KEEP_ALIVE, SCHEDULER)
```

/home/zhewen-yang/work/sbd/mpquic-sbd/src/AStream/dist/client/conn.py

- - Uses `ctypes` to bind a shared library (`proxy_module.so`) generated by the Go language  for  DASH fragment downloads via QUIC/MP-QUIC.
    - **Define `GoSlice` and `GoString` structures** for passing data between Python and Go.
    - **Sets the argument types (`argtypes`) of functions in the Go library** to ensure that Python passes arguments that are compatible with Go code.
    - **Provides Python API**:
      - `setupPM()`: initializes a QUIC connection, specifying scheduling algorithms and congestion control.
      - `closeConnection()`: closes the QUIC connection.
      - `download_segment_PM()`: Download DASH fragments via QUIC.
      - `startLogging()` and `stopLogging()`: enable/disable logging for QUIC connection.

```python
def setupPM(useQUIC, useMP, keepAlive, schedulerName, congestionControl='cubic'):
    scheduler = GoString(schedulerName.encode('ascii'), len(schedulerName))
    cc = GoString(congestionControl.encode('ascii'), len(congestionControl))
    lib.ClientSetup(useQUIC, useMP, keepAlive, scheduler, cc)

def closeConnection():
    lib.CloseConnection()

def download_segment_PM(segment_url):
    segment = GoString(segment_url.encode('ascii'), len(segment_url))
    return lib.DownloadSegment(segment)

def startLogging(period):
    lib.StartLogging(period)

def stopLogging():
    lib.StopLogging()
```

mpquic-sbd/src/dash/client/proxy_module/proxy_module.go

```go
var (
keepAlive = true // whether to keep the QUIC connection alive
useQUIC bool // whether to use QUIC (otherwise HTTP/2)
useMP bool // whether to enable MP-QUIC (multipath QUIC)
gohclient *http.Client // global HTTP/2 client
roundTripper *h2quic.RoundTripper // HTTP/2 QUIC transport component

loggerDeployed bool // whether logging is enabled or not
logFile *os.File // log file
logTicker *time.Ticker // Timer for periodic bandwidth logging
logStopChannel chan struct{} // Signal channel for stopping logging.

recvBytes uint64 // number of bytes received

```

)

Downloading from the server is not unsolicited by the server

```go
func ClientSetup(usequic, mp, keepalive bool, scheduler string, cc string) {
```

```go
func CloseConnection() {
```

```go
//export DownloadSegment
func DownloadSegment(segmentURL string) int {

    if hclient == nil || !keepAlive {
        createRemoteClient() // Create a QUIC or HTTP/2 client.
    }

    // Send the request
    rsp, err := hclient.Get(segmentURL)
    Get(segmentURL) if err ! = nil {
        log.Println(logTag, “error : ”, err)
        return -1
    }

    // Read the data
    body := &bytes.Buffer{}
    _, err = io.Copy(body, rsp.Body)
    if err ! = nil {
        log.Println(logTag, “error : ”, err)
        return -1
    }
    rsp.Body.Close()
    recvBytes += uint64(body.Len()) // update received bytes

    if !keepAlive {
        hclient.CloseIdleConnections()
        hclient = nil
    hclient = nil }

    return body.Len() // return downloaded bytes
}

```

```go
func createRemoteClient() {
tlsConfig := &tls.Config{InsecureSkipVerify: true}

if useQUIC {
    roundTripper = &h2quic.RoundTripper{
        TLSClientConfig: tlsConfig,
        QuicConfig:      &quic.Config{CreatePaths: useMP},  // MP-QUIC
    }
    hclient = &http.Client{ Transport: roundTripper }
    log.Printf("%s created http2 QUIC client (MP: %t, %s)", logTag, useMP, 0)
} else {
    hclient = &http.Client{}
    transport := &http.Transport{ TLSClientConfig: tlsConfig }
    err := http2.ConfigureTransport(transport)
    if err != nil {
        log.Printf("%s ConfigureTransport http2 %v", logTag, err)
    }
    hclient.Transport = transport
    log.Printf("%s created http2 TLS client", logTag)
}
}
```

```go
roundTripper = &h2quic.RoundTripper{
			TLSClientConfig: tlsConfig,
			QuicConfig:      &quic.Config{CreatePaths: useMP},
		}

		hclient = &http.Client{
			Transport: roundTripper,
		}
```

mpquic-sbd/src/quic-go/h2quic/roundtrip.go

```go
type RoundTripper struct {
mutex sync.

// Disable gzip compression
DisableCompression bool

// TLS configuration (for QUIC connections)
TLSClientConfig *tls.

// QUIC connection configuration
QuicConfig *quic.Config

// Store all established QUIC connections (by hostname)
clients map[string]roundTripCloser
}
```

```go
func (r *RoundTripper) RoundTripOpt(req *http.Request, opt RoundTripOpt) (*http.Response, error) {
```

```go
return cl.RoundTrip(req)
```

Processes **HTTP requests** and sends them via QUIC, returning an HTTP response.

```go
func (r *RoundTripper) getClient(hostname string, onlyCached bool) (http.RoundTripper, error) {
```

```go
	if !ok {
		if onlyCached {
			return nil, ErrNoCachedConn
		}
		client = newClient(hostname, r.TLSClientConfig, &roundTripperOpts{DisableCompression: r.DisableCompression}, r.QuicConfig)
		r.clients[hostname] = client
	}
```

mpquic-sbd/src/quic-go/h2quic/client.go

```go
Client type struct {
mutex sync.RWMutex
tlsConf *tls.Config
config *quic.Config
Options *roundTripperOpts
Hostname strings
session quic.Session // QUIC connection
headerStream quic.Stream // HTTP/2 header stream
headerErr *qerr.QuicError
headerErrored chan struct{} // Close this channel on header stream error

requestWriter *requestWriter
responses map[protocol.StreamID]chan *http.Response // Channels that store HTTP responses
}
```

```go
func newClient(hostname string, tlsConfig *tls.Config, opts *roundTripperOpts, quicConfig *quic.Config) *client {
config := defaultQuicConfig
if quicConfig != nil {
config = quicConfig
}
return &client{
hostname:      authorityAddr("https", hostname),
responses:     make(map[protocol.StreamID]chan *http.Response),
tlsConf:       tlsConfig,
config:        config,
opts:          opts,
headerErrored: make(chan struct{}),
}
}
```

`dial()`（create QUIC conn）

```go
func (c *client) dial() error {
var err error
c.session, err = dialAddr(c.hostname, c.tlsConf, c.config) // Establish a QUIC connection via quic-go.
if err ! = nil {
return err
}
// Open the HTTP/2 header stream (StreamID = 3)
c.headerStream, err = c.session.OpenStream()
if err ! = nil {
    return err
}
if c.headerStream.StreamID() ! = 3 {
    return errors.New(“h2quic Client BUG: StreamID of Header Stream is not 3”)
}
c.requestWriter = newRequestWriter(c.headerStream)

go c.handleHeaderStream() // start the header processing thread
return nil
}
```

`RoundTrip()`（send HTTP request）

major part

```go
func (c *client) RoundTrip(req *http.Request) (*http.Response, error) {
if req.URL.Scheme ! Scheme != “https” {
return nil, errors.New(“quic http2: unsupported scheme”)
}
if authorityAddr(“https”, hostnameFromRequest(req)) ! = c.hostname {
return nil, fmt.Errorf(“h2quic Client BUG: RoundTrip called for the wrong client”)
}
c.dialOnce.Do(func() {
    c.handshakeErr = c.dial()
})
if c.handshakeErr ! = nil {
    return nil, c.handshakeErr
}

hasBody := (req.Body ! = nil)
responseChan := make(chan *http.Response)

// 🔹  1️⃣ Create a QUIC data stream
dataStream, err := c.session.OpenStreamSync()
if err ! = nil {
    _ = c.CloseWithError(err)
    return nil, err
}
c.mutex.Lock()
c.responses[dataStream.StreamID()] = responseChan
c.mutex.Unlock()

// 🔹  2️⃣ send HTTP request
err = c.requestWriter.WriteRequest(req, dataStream.StreamID(), !hasBody, false)
if err ! = nil {
    return nil, err
}

// 🔹  3️⃣ wait for HTTP response
res := <-responseChan
c.mutex.Lock()
delete(c.responses, dataStream.StreamID())
c.mutex.Unlock()

res.Request = req
return res, nil
}
```

`handleHeaderStream()`（Parsing HTTP Responses）(in dail)

```go
func (c *client) handleHeaderStream() {
decoder := hpack.NewDecoder(4096, func(hf hpack.HeaderField) {}) // 🔹 HPACK parser
h2framer := http2.NewFramer(nil, c.headerStream) // 🔹 Parsing Header Stream (StreamID = 3)
var lastStream protocol.StreamID

for {
    // 🔹  1️⃣ Read HTTP/2 header frame from Header Stream
    frame, err := h2framer.ReadFrame()
    if err ! = nil {
        c.headerErr = qerr.Error(qerr.HeadersStreamDataDecompressFailure, “cannot read frame”)
        break // ❌ exit on error
    }

    lastStream = protocol.StreamID(frame.Header().StreamID)

    // 🔹  2️⃣ make sure frame is `HeadersFrame`
    hframe, ok := frame.(*http2.HeadersFrame)
    if !ok {
        c.headerErr = qerr.Error(qerr.InvalidHeadersStreamData, “not a headers frame”)
        break
    }

    // 🔹  3️⃣ Parsing an HTTP/2 header field with HPACK
    mhframe := &http2.MetaHeadersFrame{HeadersFrame: hframe}
    mhframe.Fields, err = decoder.DecodeFull(hframe.HeaderBlockFragment())
    if err ! = nil {
        c.headerErr = qerr.Error(qerr.InvalidHeadersStreamData, “cannot read header fields”)
        break
    }

    // 🔹  4️⃣ find the corresponding HTTP request StreamID
    c.mutex.RLock()
    responseChan, ok := c.responses[protocol.StreamID(hframe.StreamID)]
    c.mutex.RUnlock()
    if !ok {
        c.headerErr = qerr.Error(qerr.InternalError, fmt.Sprintf(“h2client BUG: response channel for stream %d not found”, lastStream))
        break
    }

    // 🔹  5️⃣ Parses the HTTP response and sends it to `responseChan`
    rsp, err := responseFromHeaders(mhframe)
    if err ! = nil {
        c.headerErr = qerr.Error(qerr.InternalError, err.Error())
    }
    responseChan <- rsp // 🚀 sends the parsed HTTP response
}

// ❌  6️⃣ Close `headerErrored` on error, indicating Header Stream listening failed
utils.Debugf(“Error handling header stream %d: %s”, lastStream, c.headerErr.Error())
close(c.headerErrored)
}
```

- **Reads HTTP headers from a QUIC stream (`headerStream`) **
- **Parses `HeadersFrame` into `http.Response` **
- ** sends `http.Response` to `responseChan` in `RoundTrip()`**
- ** Closes the `headerErrored` channel if an error occurs** ** ** Closes the `headerErrored` channel if an error occurs.

## **What does it mean to open the HTTP/2 header stream (StreamID = 3)? **

In **HTTP/2 over QUIC**, **all HTTP/2 request and response headers (Headers)** are transmitted via a **specialized QUIC stream (StreamID = 3)**.

This **StreamID = 3** is called the **"Header Stream ”** and **it is responsible for the compression and parsing** of all HTTP/2 headers.

---

## **📌 1. Why is a “Header Stream” needed? **

In **HTTP/2 over TCP** (plain HTTP/2):

- **each request has its own stream (StreamID)**, with header information and data in **different HTTP/2 streams** for the same TCP connection.

In **HTTP/2 over QUIC**:

- **To reduce the amount of header parsing computation**, QUIC **uses a specialized HTTP/2 Header Stream (StreamID = 3)**.
- This **Header Stream is responsible for all HTTP request and response headers** and then transmits the HTTP Body via a **separate data stream**.

Benefits of doing this:
✅ **Avoids double counting of header parsing and improves performance**.

✅ **QUIC's data stream is independent, preventing packet loss leading to Header Blocking (HoL Blocking)**.

## Complete process of the QUIC connection

| **Steps**                 | **Methods**            | **Role**                                      |
| ------------------------- | ---------------------- | --------------------------------------------- |
| 1️⃣ Create QUIC Connection  | `dial()`               | `quic.DialAddr()` Create QUIC Connection      |
| 2️⃣ Open Header Stream      | `session.OpenStream()` | StreamID = 3                                  |
| 3️⃣ Listen to Header Stream | `handleHeaderStream()` | Parse HTTP/2 headers                          |
| 4️⃣ Send HTTP/2 Request     | `RoundTrip()`          | Transfer request body in new QUIC stream      |
| 5️⃣ Read HTTP response      | `handleHeaderStream()` | Parse HTTP response returned by Header Stream |

### **`handleHeaderStream()` Is it always running? **

✅ ** Yes, `handleHeaderStream()` runs until the QUIC connection closes or an error occurs**.

✅ **It is a long-running Goroutine** that constantly listens to the Header Stream (StreamID = 3)** for **HTTP/2 over QUIC.

✅ ** If `headerStream` parses with an error, `handleHeaderStream()` exits and closes the `headerErrored` channel**.

###  As long as `RoundTrip()` is still running, it will not exit

```go
res := <-responseChan

```

- `handleHeaderStream()` will send the data to `responseChan` as soon as it is able to successfully parse the HTTP response, and `RoundTrip()` will continue to fetch it.

### **Every HTTP header sent by the server is parsed by it**

```go
frame, err := h2framer.ReadFrame()

```

- `handleHeaderStream()`  Continuously parses the `headerStream`  which is parsed and passed to `responseChan` every time the server returns a new `HeadersFrame`.

mpquic-sbd/src/quic-go/h2quic/response.go
