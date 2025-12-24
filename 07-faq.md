# FAQ - Frequently Asked Questions

This document answers common questions about the distributed file system.

---

## General Questions

### Q: What is this project?

**A:** This is an educational distributed file system written in Go. It demonstrates how to build a peer-to-peer network where multiple computers (nodes) work together to store and retrieve files. Think of it as a simplified version of systems like BitTorrent or IPFS.

### Q: Is this production-ready?

**A:** No! This is an educational project designed to teach distributed systems concepts. It lacks many features needed for production:
- No authentication or authorization
- No persistent peer connections
- Crude error handling
- No monitoring or metrics
- Each node has its own encryption key (can't decrypt each other's files)
- No tests for most components

### Q: What can I learn from this project?

**A:** You'll learn about:
- P2P networking basics
- Content-addressable storage
- Encryption and decryption
- TCP socket programming
- Concurrent programming in Go
- Message protocols
- Distributed system design patterns

### Q: What language and dependencies does it use?

**A:** 
- **Language:** Go 1.18+
- **Dependencies:** Minimal (only testing libraries in `go.mod`)
- All core functionality is built from scratch using Go's standard library

---

## Architecture Questions

### Q: How does the P2P network work?

**A:** Each node:
1. Listens for incoming TCP connections
2. Connects to "bootstrap" nodes when starting
3. Maintains a map of connected peers
4. Sends/receives messages through the peer connections

There's no central server - all nodes are equal (peer-to-peer).

### Q: Why are files stored on multiple nodes?

**A:** **Redundancy and availability**. If files were only stored on one node:
- That node goes down → files are lost
- That node is busy → poor performance

By storing files on multiple nodes:
- Files are available even if some nodes are offline
- Load is distributed across the network

### Q: What is content-addressable storage (CAS)?

**A:** Instead of storing files by name (like "photo.jpg"), we store them by their content's fingerprint (hash):

```
Traditional:
  Store "photo.jpg" at /documents/photo.jpg
  Store "vacation.jpg" at /pictures/vacation.jpg

CAS:
  Hash content → "abc123..."
  Store both at /storage/abc123.../
  (They have the same content, so same hash!)
```

**Benefits:**
- Automatic deduplication
- Integrity verification
- Location independence

### Q: How does encryption work?

**A:** We use **AES-256 in CTR mode**:
1. Generate a random encryption key (32 bytes)
2. When sending a file over the network:
   - Generate a random IV (16 bytes)
   - Encrypt the file with AES
   - Prepend the IV to the encrypted data
3. When receiving:
   - Read the IV
   - Decrypt the file with AES

**Important:** Files are encrypted over the network but stored unencrypted locally.

---

## Implementation Questions

### Q: Why use GOB encoding?

**A:** GOB (Go Binary) is Go's built-in serialization format. It:
- Automatically handles different types
- Is efficient and fast
- Is simple to use
- Works well for Go-to-Go communication

**Downsides:**
- Only works between Go programs
- Not human-readable
- No cross-language support

**Alternative:** Use Protocol Buffers, JSON, or MessagePack for cross-language support.

### Q: Why is there a sleep() after broadcasting messages?

```go
s.broadcast(&msg)
time.Sleep(time.Millisecond * 5)  // Why?
```

**A:** This is a **crude synchronization mechanism**. The sleep gives peers time to:
- Receive the metadata message
- Prepare to receive the file stream

**Better approach:** Use proper acknowledgments:
```go
s.broadcastAndWaitForAck(&msg)  // Wait for peers to respond
```

### Q: Why use io.TeeReader when storing files?

```go
fileBuffer := new(bytes.Buffer)
tee := io.TeeReader(r, fileBuffer)
s.store.Write(s.ID, key, tee)
```

**A:** We need the file data **twice**:
1. To write to local storage
2. To send to peers over the network

TeeReader splits the stream, so we can read once and write to two destinations simultaneously.

**Alternative:** Read the entire file into memory first:
```go
data, _ := ioutil.ReadAll(r)
s.store.Write(s.ID, key, bytes.NewReader(data))
sendToPeers(data)
```

But this is less efficient for large files.

### Q: Why does each node have its own encryption key?

**A:** Simple design choice. Each node generates a random key at startup:

```go
fileServerOpts := FileServerOpts{
    EncKey: newEncryptionKey(),  // Random key
    ...
}
```

**Consequence:** Nodes can't decrypt files from each other!

**Solution:** Use a **shared key** or implement a **key exchange protocol** (like Diffie-Hellman).

### Q: What happens if a peer disconnects?

**A:** When the TCP connection closes:
1. `Read()` or `Decode()` returns an error
2. The `handleConn()` goroutine exits
3. The defer closes the connection

The peer is **not removed** from the peer map! This is a bug.

**Fix:** Track connection state and remove disconnected peers:
```go
defer func() {
    conn.Close()
    s.removePeer(conn.RemoteAddr().String())
}()
```

### Q: Why is the file hash different from the storage path hash?

**A:** We use two different hashes:

1. **Storage path:** SHA-1 of the key
   ```go
   CASPathTransformFunc("picture.png") → SHA-1 → "0eb98d43..."
   ```
   Used to organize files on disk in nested directories.

2. **Network identifier:** MD5 of the key
   ```go
   hashKey("picture.png") → MD5 → "a1b2c3..."
   ```
   Used in messages to identify files across the network.

**Why two?**
- SHA-1 produces 40 hex chars (good for deep directory structure)
- MD5 produces 32 hex chars (shorter for network messages)

In production, you'd likely use one hash for everything (SHA-256).

---

## Usage Questions

### Q: How do I run the project?

**A:**
```bash
# Build
make build

# Run (starts 3 nodes and demonstrates file operations)
make run

# Run tests
make test
```

### Q: How do I add more nodes?

**A:** In `main.go`, create more servers:
```go
s4 := makeServer(":9000", ":3000", ":7000", ":5000")
go s4.Start()
```

This creates a node on port 9000 that connects to nodes on 3000, 7000, and 5000.

### Q: How do I change the storage location?

**A:** Modify the `StorageRoot` in `makeServer()`:
```go
fileServerOpts := FileServerOpts{
    StorageRoot: "/my/custom/path",  // Instead of ":3000_network"
    ...
}
```

### Q: Can I use a different path organization?

**A:** Yes! Create a custom `PathTransformFunc`:

```go
// Store files flat (no nested directories)
func FlatPathTransformFunc(key string) PathKey {
    return PathKey{
        PathName: "",  // No subdirectories
        Filename: key, // Use key as-is
    }
}

fileServerOpts := FileServerOpts{
    PathTransformFunc: FlatPathTransformFunc,
    ...
}
```

### Q: How do I retrieve a specific file?

**A:** Use the `Get()` method:
```go
reader, err := server.Get("myfile.txt")
if err != nil {
    log.Fatal(err)
}

data, _ := ioutil.ReadAll(reader)
fmt.Println(string(data))
```

---

## Performance Questions

### Q: How fast is it?

**A:** It depends on:
- Network bandwidth
- File size
- Number of peers
- Encryption overhead

**Ballpark estimates:**
- Local storage: ~100MB/s (limited by disk speed)
- Network transfer: ~10MB/s (limited by TCP + encryption)
- Small files (<1KB): ~1000 ops/second

### Q: Does it scale to many nodes?

**A:** Not well! Current issues:

1. **Broadcast to all peers:** Every file is sent to every peer
   - 10 nodes → 10 copies
   - 1000 nodes → 1000 copies (ouch!)

2. **No routing:** Requests are broadcast to everyone

3. **No peer discovery:** Must manually configure bootstrap nodes

**Solutions:**
- Use a DHT (Distributed Hash Table) for routing
- Implement selective replication (don't store on every node)
- Add peer discovery protocols (e.g., gossip protocol)

### Q: How much storage does it use?

**A:** For each file:
- **Origin node:** File size (plaintext)
- **Each peer:** File size + 16 bytes (encrypted + IV)

**Example:** 1MB file with 5 peers:
- Origin: 1MB
- 5 peers: 5 × (1MB + 16B) ≈ 5MB
- **Total:** ~6MB across the network

**Optimization:** Implement erasure coding (like RAID 5) to get redundancy with less storage overhead.

---

## Debugging Questions

### Q: How do I see what's happening?

**A:** The code has extensive logging:
```
[:3000] starting fileserver...
[:3000] connected with remote 127.0.0.1:5000
[:3000] serving file (abc123...) over the network
[:3000] written 39 bytes to disk
```

The `[address]` prefix shows which node is logging.

**More debugging:** Add more log statements, use a debugger (delve), or add metrics.

### Q: How do I test individual components?

**A:** Unit test using Go's testing framework:

```go
func TestStore(t *testing.T) {
    store := NewStore(StoreOpts{
        Root: "test_storage",
        PathTransformFunc: CASPathTransformFunc,
    })
    defer store.Clear()
    
    // Test Write
    data := bytes.NewReader([]byte("test data"))
    size, err := store.Write("node1", "test.txt", data)
    assert.NoError(t, err)
    assert.Equal(t, 9, size)
    
    // Test Has
    assert.True(t, store.Has("node1", "test.txt"))
    
    // Test Read
    _, reader, err := store.Read("node1", "test.txt")
    assert.NoError(t, err)
    
    content, _ := ioutil.ReadAll(reader)
    assert.Equal(t, "test data", string(content))
}
```

### Q: What if files don't replicate?

**Check:**
1. Are nodes connected? (Look for "connected with remote" logs)
2. Is broadcast() succeeding? (Add log statements)
3. Are peers receiving messages? (Check peer's logs)
4. Is there a network issue? (Firewall, port already in use)

**Common issue:** Starting nodes too quickly. Add delays:
```go
go s1.Start()
time.Sleep(500 * time.Millisecond)  // Let s1 settle

go s2.Start()
time.Sleep(500 * time.Millisecond)
```

### Q: Why are files corrupted?

**Possible causes:**

1. **Wrong encryption key:** Each node has its own key. If you store with one key and decrypt with another, you get garbage.

2. **Network errors:** TCP connection died mid-transfer.

3. **Size mismatch:** `LimitReader` reading wrong number of bytes.

**Debug:** Log the sizes at each step:
```go
fmt.Printf("Original size: %d\n", originalSize)
fmt.Printf("Encrypted size: %d\n", encryptedSize)
fmt.Printf("Received size: %d\n", receivedSize)
```

---

## Design Questions

### Q: Why not use an existing framework?

**A:** This is an educational project! The goal is to learn by building from scratch. Using frameworks would hide the complexity we're trying to understand.

**Production:** Absolutely use frameworks like gRPC, libp2p, etc.

### Q: Why TCP instead of UDP?

**A:** **TCP** provides:
- Reliable delivery (packets arrive in order)
- Automatic retransmission (lost packets are resent)
- Flow control (prevents overwhelming the receiver)

**UDP** is:
- Faster (no handshaking)
- Less overhead
- But unreliable (packets can be lost or arrive out of order)

For file transfer, reliability is more important than raw speed, so TCP is a better choice.

### Q: Could this use HTTP/REST instead?

**A:** Yes! You could create a REST API:

```go
// Store a file
POST /files
Body: file data
Response: {"id": "abc123", "size": 1024}

// Get a file
GET /files/abc123
Response: file data

// List files
GET /files
Response: ["abc123", "def456", ...]
```

**Tradeoffs:**
- ✅ Easier to debug (can use curl, browser)
- ✅ Better tooling and ecosystem
- ✅ Standard protocol
- ❌ More overhead (HTTP headers)
- ❌ Not as efficient for streaming

### Q: Why not use WebSockets?

**A:** WebSockets are great for:
- Browser-based clients
- Bidirectional real-time communication
- When you need standardization

Our custom TCP protocol is simpler and has less overhead for Go-to-Go communication.

**When to use WebSockets:** If you want a web UI to interact with the file system!

---

## Security Questions

### Q: Is this secure?

**A:** No! Security issues include:

1. **No authentication:** Anyone can connect
2. **No authorization:** Anyone can read/write any file
3. **Local files unencrypted:** Anyone with disk access can read files
4. **No integrity checking:** Files could be tampered with
5. **No key management:** Keys are generated randomly and never shared

### Q: How would you secure it?

**A:** Several approaches:

1. **Authentication:**
   ```go
   func SecureHandshake(p Peer) error {
       // Exchange certificates or shared secrets
       // Verify peer identity
   }
   ```

2. **Authorization:**
   ```go
   func (s *FileServer) Store(key string, r io.Reader, perms Permissions) error {
       if !s.canWrite(currentUser, perms) {
           return ErrUnauthorized
       }
       // ... store file ...
   }
   ```

3. **Encrypt local storage:**
   ```go
   func (s *Store) Write(id, key string, r io.Reader) error {
       encrypted := encrypt(r, s.diskEncKey)
       return s.writeStream(id, key, encrypted)
   }
   ```

4. **Use TLS:**
   ```go
   conn, err := tls.Dial("tcp", addr, &tls.Config{...})
   ```

5. **Add HMAC for integrity:**
   ```go
   hmac := computeHMAC(fileData, secretKey)
   send(fileData, hmac)
   // Receiver verifies HMAC matches
   ```

### Q: Can someone steal my files?

**A:** In this implementation:

**Local files:** Anyone with access to the machine can read `:3000_network/` directory
**Network:** Files are encrypted in transit (good!), but:
- No authentication (anyone can connect)
- No authorization (anyone can request any file)
- No peer verification

**Bottom line:** Don't use this for sensitive data!

---

## Extension Questions

### Q: How would you add file deletion across the network?

**A:** Add a new message type:

```go
type MessageDeleteFile struct {
    ID  string
    Key string
}

func (s *FileServer) Delete(key string) error {
    // Delete locally
    s.store.Delete(s.ID, key)
    
    // Broadcast delete request
    msg := Message{
        Payload: MessageDeleteFile{
            ID:  s.ID,
            Key: hashKey(key),
        },
    }
    s.broadcast(&msg)
}

func (s *FileServer) handleMessageDeleteFile(from string, msg MessageDeleteFile) error {
    return s.store.Delete(msg.ID, msg.Key)
}
```

register the new type:
```go
func init() {
    gob.Register(MessageDeleteFile{})
}
```

### Q: How would you add file search?

**A:** Add a search message:

```go
type MessageSearchFile struct {
    Pattern string  // Search pattern
}

type MessageSearchResult struct {
    Files []string  // Matching file keys
}

func (s *FileServer) Search(pattern string) ([]string, error) {
    msg := Message{
        Payload: MessageSearchFile{Pattern: pattern},
    }
    s.broadcast(&msg)
    
    // Wait for responses...
    // Aggregate results...
}

func (s *FileServer) handleMessageSearchFile(from string, msg MessageSearchFile) error {
    matches := s.store.Search(msg.Pattern)
    
    response := Message{
        Payload: MessageSearchResult{Files: matches},
    }
    
    peer := s.peers[from]
    // Send response back to requester
}
```

### Q: How would you add versioning?

**A:** Store multiple versions of each file:

```go
type FileVersion struct {
    Version   int
    Timestamp time.Time
    Data      []byte
}

func (s *FileServer) Store(key string, r io.Reader) error {
    // Get current version
    currentVersion := s.getLatestVersion(key)
    
    // Store new version
    newVersion := currentVersion + 1
    versionedKey := fmt.Sprintf("%s@v%d", key, newVersion)
    
    s.store.Write(s.ID, versionedKey, r)
}

func (s *FileServer) Get(key string, version int) (io.Reader, error) {
    versionedKey := fmt.Sprintf("%s@v%d", key, version)
    return s.Get(versionedKey)
}
```

### Q: How would you add a CLI or REST API?

**A:** Create a simple HTTP server:

```go
func (s *FileServer) StartAPIServer(port int) {
    http.HandleFunc("/files", s.handleListFiles)
    http.HandleFunc("/files/upload", s.handleUpload)
    http.HandleFunc("/files/download", s.handleDownload)
    
    log.Fatal(http.ListenAndServe(fmt.Sprintf(":%d", port), nil))
}

func (s *FileServer) handleUpload(w http.ResponseWriter, r *http.Request) {
    file, header, _ := r.FormFile("file")
    defer file.Close()
    
    err := s.Store(header.Filename, file)
    if err != nil {
        http.Error(w, err.Error(), 500)
        return
    }
    
    fmt.Fprintf(w, "Uploaded: %s", header.Filename)
}
```

Then use curl to interact:
```bash
curl -F "file=@myfile.txt" http://localhost:8080/files/upload
curl http://localhost:8080/files/download?key=myfile.txt > myfile.txt
```

---

## Learning Questions

### Q: I'm new to Go. Where should I start?

**A:** Learn Go basics first:
1. [A Tour of Go](https://tour.golang.org/)
2. [Go by Example](https://gobyexample.com/)
3. [Effective Go](https://golang.org/doc/effective_go)

Then study this project's documentationin order:
1. [Overview](./00-overview.md)
2. [P2P Network](./01-p2p-network.md)
3. [Storage System](./02-storage-system.md)
4. [Encryption](./03-encryption.md)
5. [File Server](./04-file-server.md)
6. [Message Protocol](./05-message-protocol.md)
7. [Workflows](./06-workflows.md)

### Q: I'm new to distributed systems. What concepts should I learn?

**A:** Key concepts demonstrated in this project:

1. **Peer-to-Peer Architecture:** No central server
2. **Replication:** Storing data on multiple nodes
3. **Content-Addressable Storage:** Using hashes as identifiers
4. **Eventual Consistency:** Data will eventually be consistent
5. **Message Passing:** Communication via messages
6. **Concurrency:** Multiple operations happening simultaneously

**Further reading:**
- "Designing Data-Intensive Applications" by Martin Kleppmann
- "Distributed Systems" by Maarten van Steen

### Q: What should I implement next to learn more?

**A:** Try these extensions (in order of difficulty):

**Easy:**
1. Add a command-line interface
2. Implement file listing (list all files in the system)
3. Add file metadata (size, timestamp, owner)

**Medium:**
4. Implement proper peer disconnection handling
5. Add configuration files (instead of hardcoded values)
6. Create a web UI for uploading/downloading files
7. Add file search functionality

**Hard:**
8. Implement a DHT for better routing
9. Add selective replication (don't store everything everywhere)
10. Implement erasure coding for efficient redundancy
11. Add versioning and conflict resolution

Each of these will teach you new distributed systems concepts!

---

## Troubleshooting

### Problem: "port already in use"

**Error:**
```
bind: address already in use
```

**Solution:** A previous instance is still running. Kill it:
```bash
lsof -i :3000
kill <PID>
```

Or use different ports in `main.go`.

### Problem: Files aren't replicating

**Check:**
1. Are nodes connected? Look for "connected with remote" messages
2. Increase the sleep between broadcasts:
   ```go
   time.Sleep(time.Second * 1)  // Instead of 5ms
   ```

### Problem: "decoding error"

**Cause:** Garbled data or version mismatch.

**Solution:**
1. Make sure all nodes are running the same code version
2. Clear all storage directories and restart:
   ```bash
   rm -rf *_network/
   ```

### Problem: Out of memory

**Cause:** Too many files or very large files.

**Solution:**
1. Reduce the number of iterations in the loop
2. Use smaller test data
3. Add periodic garbage collection

---

This FAQ covers the most common questions. For more details, see the other documentation files!
