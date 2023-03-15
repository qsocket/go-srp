# Standalone SRP-6a implementation in go-lang

[![GoDoc](https://godoc.org/github.com/qsocket/go-srp?status.svg)](https://godoc.org/github.com/qsocket/go-srp)
[![Go Report Card](https://goreportcard.com/badge/github.com/qsocket/go-srp)](https://goreportcard.com/report/github.com/qsocket/go-srp)

This is a standalone implementation of SRP in golang. It uses the go
standard libraries and has no other external dependencies. This library
can be used by SRP clients or servers.

SRP is a protocol to authenticate a user and derive safe session keys.
It is the latest in the category of \"strong authentication protocols\".

SRP is documented here: <http://srp.stanford.edu/doc.html>. Briefly,

## Design Overview

### Conventions

    N    A large safe prime (N = 2q+1, where q is prime)
       All arithmetic is done modulo N.
    g    A generator modulo N
    k    Multiplier parameter (k = H(N, g) in SRP-6a, k = 3 for legacy SRP-6)
    s    User's salt
    I    Username
    p    Cleartext Password
    H()  One-way hash function
    ^    (Modular) Exponentiation
    u    Random scrambling parameter
    a,b  Secret ephemeral values
    A,B  Public ephemeral values
    x    Private key (derived from p and s)
    v    Password verifier

### Differences from SRP-6a and RFC 5054
We differ from the SRP-6a spec and RFC 5054 in a couple of key ways:

* We hash the identity `I`; this provides some (minimal) protection against
  dictionary attacks on the username.
* We hash the user passphrase `p`; this expands shorter passphrase
  into longer ones and extends the alphabet used in the passphrase.
* We differ from RFC 5054 in our choice of hash function; we use Blake-2b.
  SHA-1 is getting long in the tooth, Blake2b is the current
  state-of-the art. Equivalently, one may use SHA3 (see below for
  using a user supplied hash function).

### Generating and Storing the Password Verifier
The host calculates the password verifier using the following formula:

    s = randomsalt()          (same length as N)
    I = H(I)
    p = H(p)                  (hash/expand I & p)
    t = H(I, ":", p)
    x = H(s, t)
    v = g^x                   (computes password verifier)

The host then stores {I, s, v} in its password database - such that
the triple can be retrieved by using `I` as the index/key.

### Authentication Protocol
The authentication protocol itself goes as follows:

    Client                       Server
    --------------               ----------------
    un, pw = < user input >
    I = H(un)
    p = H(pw)
    a = random()
    A = g^a % N
                I, A -->
                              s, v = lookup(I)
                              b = random()
                              B = (kv + g^b) % N
                              u = H(A, B)
                              S = ((A * v^u) ^ b) % N
                              K = H(S)
                              M' = H(K, A, B, I, s, N, g)
                 <-- s, B
    u = H(A, B)
    x = H(s, p)
    S = ((B - k (g^x)) ^ (a + ux)) % N
    K = H(S)
    M = H(K, A, B, I, s, N, g)

                M -->
                              M must be equal to M'
                              Z = H(M, K)
                <-- Z

    Z' = H(M, K)
    Z' must equal Z

When the server receives `<I, A>`, it can compute everything: shared key
and proof-of-generation `M'`. The shared key is `K`.

To verify that the client has generated the same key `K`, the client sends
`M` -- a hash of all the data it has and it received from the server. To
validate that the server also has the same value, it requires the server to send
its own proof. In the SRP paper, the authors use:

    M = H(H(N) xor H(g), H(I), s, A, B, K)
    M' = H(A, M, K)

We use a simpler construction:

    M = H(K, A, B, I, s, N, g)
    M' = H(M, K)

The two parties also employ the following safeguards:

 1. The user will abort if he receives `B == 0 (mod N) or u == 0`.
 2. The host will abort if it detects that `A == 0 (mod N)`.
 3. The user must show his proof of K first. If the server detects that the
    user\'s proof is incorrect, it must abort without showing its own proof of K.


## Implementation Notes
In our implementation:

- The standard hash function is Blake2b-256; this can be changed by choosing an
  appropriate hash from `crypto`:
  ```go

       s, err := srp.NewWithHash(crypto.SHA256, 4096)
  ```


### Setting up the Verifiers on the Server
In order to authenticate and derive session keys, verifiers must be
stored in a non-volatile medium on the server. The client provides the
prime-field size, username and password when creating the verifier. The
server stores the triple in a non-volatile medium. The verifiers are
generated *once* when a user is created on the server.

The Client is the entity where the user enters their password and wishes
to be authenticated with a SRP server. The communication between client
and server can happen in clear text - SRP is immune to man in the middle
attacks.

Depending on the resources available on a given client, it can choose a
small or large prime-field; but once chosen it is recorded on the server
until a new verifier is generated.

For example, a client will do:

```go

    s, err := srp.New(n_bits)

    v, err := s.Verifier(username, password)
    id, verif := v.Encode()

    // Now, store 'id', 'verif' in non-volatile storage such that 'verif' can be
    // retrieved by providing 'id'.
```
Note that `id` is the hashed identity string for username. The server should store
the encoded verifier string `verif` in a DB such that it can be looked up using `id`
as the key.

### Changing the default hash function
A client may wish to change the default hash function to something else. e.g.,::

```go

    s, err := srp.NewWithHash(crypto.SHA256, n_bits)

    v, err := s.Verifier(username, password)
    id, verif := v.Encode()
```

### Authentication attempt from the Client
The client performs the following sequence of steps to authenticate and
derive session keys:

```go

    s, err := srp.New(n_bits)

    c, err := s.NewClient(user, pass)
    creds := c.Credentials()

    // 1. send the credentials to the server. It is already in ASCII string form; this
    //    is essentially the encoded form of identity and a random public key.

    // 2. Receive the server credentials into 'server_creds'; this is the server
    //    public key and random salt generated when the verifier was created.

    // It is assumed that there is some network communication that happens
    // to get this string from the server.

    // Now, generate a mutual authenticator to be sent to the server
    auth, err := c.Generate(server_creds)

    // 3. Send the mutual authenticator to the server
    // 4. receive "proof" that the server too computed the same result.

    // Verify that the server actually did what it claims
    if !c.ServerOk(proof) {
        panic("authentication failed")
    }

    // Generate session key
    rawkey := c.RawKey()
```

### Authenticating a Client on the Server
On the server, the authentication attempt begins after receiving the
initial user credentials. This is used to lookup the stored verifier and
other bits.:

```go

    // Assume that we received the user credentials via the network into 'creds'


    // Parse the user info and authenticator from the 'creds' string
    id, A, err := srp.ServerBegin(creds)

    // Use 'id' to lookup the user in some non-volatile DB and obtain
    // previously stored *encoded* verifier 'v'.
    verifier := db.Lookup(id)


    // Create an SRP instance and Verifier instance from the stored data.
    s, v, err := srp.MakeSRPVerifier(verifier)

    // Begin a new client-server SRP session using the verifier and received
    // public key.
    srv, err := s.NewServer(v, A)

    // Generate server credentials to send to the user
    s_creds := srv.Credentials()

    // 1. send 's_creds' to the client
    // 2. receive 'm_auth' from the client

    // Authenticate user and generate mutual proof of authentication
    proof, ok := srv.ClientOk(m_auth)
    if ok != nil {
         panic("Authentication failed")
    }

    // 3. Send proof to client

    // Auth succeeded, derive session key
    rawkey := s.RawKey()
```

### Generating new Safe Primes & Prime Field Generators
The SRP library uses a pre-calculated list of large safe prime for common widths
along wit their field generators. But, this is not advisable for large scale 
production use. It is best that a separate background process be used to generate
safe primes & the corresponding field generators - and store them in some cache.
The function `findPrimeField()` can be modified to fetch from this cache. Depending 
on the security stance, the cache can decide on a "use once" policy or
"use N times" policy.  

The function `srp.NewPrimeField()` generates and returns a new large safe prime
and its field generator.

### Building SRP

There is an example program that shows you the API usage (documented
above).:

```sh
    $ git clone https://github.com/qsocket/go-srp
    $ cd go-srp
    $ go test -v
```

Finally, build the example program:

```sh
    $ go build -o ex example/example.go
    $ ./ex
```

The example program outputs the raw-key from the client & server\'s
perspective (they should be identical).

There is also a companion program in the example directory that generates prime fields
of a given size:

```sh
    $ go build -o pf example/primefield.go
    $ ./pf 1024
```

The above program can be run to generate multiple fields on the command line:
```sh
    $ ./pf 1024 2048 4096 8192
```

The library uses `go modules`; so, it should be straight forward to import and use.


### Using the SRP Raw key to derive session keys
The client and server both derive the same value for RawKey(). This
is the crux of the SRP protocol. Treat this as a \"master key\".
It is not advisable to use the RawKey() for encryption purposes. It
is better to derive a separate key for each direction
(client-\>server and server-\>client). e.g.,:

```go
    c2s_k = KDF(rawkey, counter, "C2S")
    s2s_k = KDF(rawkey, counter, "S2C")
```

KDF above can be a reputable key derivation function such as PBKDF2
or Scrypt. The \"counter\" is incremented every time you derive a
new key.

*I am not a cryptographer*. Please consult your favorite crypto book
for deriving encryption keys from a master key.

#### Using `scrypt` as the KDF
Here is a example KDF using `scrypt`:

```go
    import "golang.org/x/crypto/scrypt"

    // Safe values for Scrypt() parameters
    const _N int = 65536
    const _r int = 1024
    const _p int = 64

    // Kdf derives a 'sz' byte key for use 'usage'
    func Kdf(key []byte, salt []byte, usage string, sz int) []byte {

        u0 := []byte(usage)
        pw := append(key, u0...)
        k, _ := scrypt.Key(pw, salt, _N, _r, _p, sz)
        return k
    }
```

#### Using `argon2` as the KDF
[Argon](https://tools.ietf.org/html/draft-irtf-cfrg-argon2-03) is
the new state of the art (2018) key derivation algorithm. The
`Argon2id` variant is resistant to timing, side-channel and
Time-memory tradeoff attacks. Here is an example using the `Argon2id`
variant:

```go
    import (
        "runtime"
        "golang.org/x/crypto/argon2"
    )

    // safe values for IDKey() borrowed from libsodium
    const _Time uint32 = 3
    const _Mem  uint32 = 256 * 1048576  // 256 MB

    // Kdf derives a 'sz' byte key for use 'usage'
    func Kdf(key, salt []byte, usage string, sz int) []byte {
        u0 := []byte(usage)
        pw := append(key, u0...)

        return argon2.IDKey(pw, salt, _Time, _Mem, runtime.NumCPU(), uint32(sz))
    }
```

