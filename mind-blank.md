# Mind blank

It's a crypto challenge where we have to guess 48 consecutive bits generated by LFSR, seeded with current time in seconds.

## 1. Guess 48 bits
Because the LFSR taps and seed are based on time.time(), each connection made in the same seconds will have the same bits

So, we can open 100 connections, ensuring that each one has the correct timestamp.

Next, we can guess the bits one by one, switching to a different connection if one guess was wrong (thus connection was closed).

This way, we get the flag, but it's encrypted with more LFSR bits.


guess_bits.go:
```go
package main

import (
	"errors"
	"fmt"
	"io"
	"io/ioutil"
	"log"
	"net"
	"strconv"
	"strings"
	"sync"
	"time"
)

var firstTs string
var connections = []net.Conn{}
var m = sync.Mutex{}

func readUntil(r io.Reader, char byte) ([]byte, error) {
	data := []byte{}
	buf := make([]byte, 1)
	for {
		_, err := r.Read(buf)
		if err != nil {
			return nil, err
		}
		data = append(data, buf[0])
		if buf[0] == char {
			break
		}
	}
	return data, nil
}

func connect() (net.Conn, error) {
	conn, err := net.DialTimeout("tcp", "mind-blank.ecsc23.hack.cert.pl:5001", time.Hour)
	if err != nil {
		return nil, err
	}

	ts, err := readUntil(conn, '?')
	tsStr := strings.Split(" can", string(ts))[0]
	if firstTs == "" {
		firstTs = tsStr
	} else if firstTs != tsStr {
		return nil, errors.New("invalid ts")
	}

	return conn, nil
}

func addConnection() error {
	conn, err := connect()
	if err != nil {
		return err
	}

	m.Lock()
	connections = append(connections, conn)
	m.Unlock()

	return nil
}

func getConnection() (net.Conn, error) {
	fmt.Println("getting connection", len(connections))

	if len(connections) == 0 {
		return nil, errors.New("no more connections")
	}
	m.Lock()
	defer m.Unlock()

	res := connections[0]
	connections = connections[1:]
	return res, nil
}

func getConnectionInitialBits(initialBits []int) (net.Conn, error) {
	c, err := getConnection()
	if err != nil {
		return nil, err
	}

	for _, bit := range initialBits {
		ok, err := guessNext(c, bit)
		if err != nil {
			return nil, err
		}
		if !ok {
			return nil, errors.New("initial bits send - not ok")
		}
	}

	return c, nil
}

func guessNext(conn net.Conn, bit int) (ok bool, err error) {
	_, err = readUntil(conn, '>')
	if err != nil {
		err = fmt.Errorf("read initial: %w", err)
		return
	}

	_, err = conn.Write([]byte(strconv.Itoa(int(bit)) + "\n"))
	if err != nil {
		err = fmt.Errorf("write: %w", err)
		return
	}

	res, err := readUntil(conn, '!')
	if err != nil {
		err = fmt.Errorf("read result: %w", err)
		return
	}

	ok = strings.Contains(string(res), "Correct")
	return
}

func bruteKnownBits(n int) ([]int, error) {
	knownBits := []int{}

	var (
		c   net.Conn
		err error
	)

	c, err = getConnection()
	if err != nil {
		return nil, err
	}

	i := 0
	for len(knownBits) < n {
		fmt.Printf("# %v - %v\n", i, knownBits)

		fmt.Println("sending guess - 0")
		ok, err := guessNext(c, 0)
		if err != nil {
			return nil, fmt.Errorf("send guess 0: %w", err)
		}
		if ok {
			knownBits = append(knownBits, 0)
		} else {
			fmt.Println("not ok")

			c, err = getConnectionInitialBits(knownBits)
			if err != nil {
				return nil, fmt.Errorf("get conn: %w", err)
			}

			fmt.Println("sending guess - 1")

			ok, err = guessNext(c, 1)

			if err != nil {
				return nil, fmt.Errorf("send guess 1: %w", err)
			}
			if ok {
				knownBits = append(knownBits, 1)
			} else {
				fmt.Println("something went very wrong", knownBits)
			}
		}

		i++
	}

	return knownBits, nil
}

func main() {
	finalConn, err := connect()
	if err != nil {
		log.Fatalln(err)
	}

	wg := sync.WaitGroup{}
	n := 100
	wg.Add(n)
	for i := 0; i < n; i++ {
		go func() {
			err := addConnection()
			if err != nil {
				fmt.Println(err)
			}
			wg.Done()
		}()
	}
	wg.Wait()
	fmt.Println(len(connections))

	knownBits, err := bruteKnownBits(48)
	if err != nil {
		log.Fatalln(err)
	}

	bitsStr := []string{}
	for _, x := range knownBits {
		bitsStr = append(bitsStr, fmt.Sprint(x))
	}
	fmt.Printf("got all bits: %v\n", strings.Join(bitsStr, ", "))

	for _, x := range knownBits {
		ok, err := guessNext(finalConn, x)
		if err != nil {
			log.Fatalln(err)
		}
		if !ok {
			log.Fatalln("final connection - not ok")
		}
	}

	res, err := ioutil.ReadAll(finalConn)
	if err != nil {
		log.Fatalln(err)
	}

	flagHex := strings.TrimSpace(string(res))
	fmt.Println("flag hex", flagHex)
}
```


## Retrieve the LFSR state
- `LFSR.state` has 21 elements
- after getting 21th bit, `state` will consist only of previously yielded bits

Thus, we only need to brute force `taps`, using `state` as `guessed_bits[:21]` and checking if operations match.

Finally, we can decode the flag by utilizing LFSR state.


solve.py:
```py
import itertools
from typing import List

difficulty = 48

def chunk(input_data, size):
    assert len(input_data) % size == 0, \
        "can't split data into chunks of equal size, try using chunk_with_remainder or pad data"
    return [input_data[i:i + size] for i in range(0, len(input_data), size)]

def long_to_bits(n: int) -> List[int]:
    return [n >> i & 1 for i in range(0, n.bit_length())]

def long_to_bytes(data):
    if data == 0:
        return "\0"
    data = int(data)
    data = hex(data).rstrip('L').lstrip('0x')
    if len(data) % 2 == 1:
        data = '0' + data
    return bytes(bytearray(int(c, 16) for c in chunk(data, 2)))

class LFSR:
    def __init__(self, taps, seed):
        self.state = long_to_bits(seed)
        self.taps = taps

    def _new_bit(self):
        b = 0
        for t in self.taps:
            b = b ^ self.state[t]
        return b

    def next_bit(self):
        ret = self.state[0]
        self.state = self.state[1:] + [self._new_bit()]
        return ret

flag_enc = bytes.fromhex('c5cecd4d1a4a36d0e133ef0b7ed3ea9ca12912157411d47eed2519903d33fe2f864e0147dbb7269a235560d951d6ce92ae')
actual_bits = [1, 1, 1, 1, 1, 1, 0, 0, 1, 1, 1, 0, 1, 1, 0, 0, 0, 0, 1, 0, 1, 0, 1, 1, 0, 1, 0, 0, 1, 1, 0, 1, 0, 1, 0, 0, 0, 1, 0, 0, 0, 1, 1, 1, 1, 0, 1, 0]


def possible_taps_for_sequence(sequence):
    for taps in itertools.combinations(range(20), 10):
        state = sequence[:21]
        
        match = True
        for bit in sequence[21:]:
            new_bit = 0
            for t in taps:
                new_bit ^= state[t]
            if new_bit != bit:
                match = False
                break
            state = state[1:] + [new_bit]
            
        if match:
            return taps, state
            
    return None, None


taps, state = possible_taps_for_sequence(actual_bits)

for x in range(0, 48):
    lfsr = LFSR(taps, 0)
    lfsr.state = state

    [lfsr.next_bit() for _ in range(x)]

    bits = [lfsr.next_bit() for _ in range(len(flag_enc) * 8)]
    keystream = long_to_bytes(int(''.join([str(b) for b in bits]), 2))
    flag = bytes([c ^ k for c, k in zip(flag_enc, keystream)])

    if b'ecsc23{' in flag:
        print(flag)

```

`ecsc23{I_see_we_got_ourselves_a_mind_reader_here}`