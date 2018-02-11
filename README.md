# rochefort - PUSH DATA, GET FILE OFFSET; no shenanigans
---
* **disk write speed** storage service that returns offsets to stored values
* if you are ok with losing some data (does not fsync on write)
* supports: **append, get, multiget, close**
* clients: java, curl

```
$ go run main.go -buckets 10 -bind :8001 -root /tmp
2018/02/10 12:06:21 starting http server on :8001
....

```

### parameters
* buckets: number of filers per storagePrefix
* root: root directory, files will be created at `root/storagePrefix||default/append.%d.raw`
* bind: address to bind to


## STORE

method post /append?id=some_identifier returns `{"offset":3659174697238528,"file":"/tmp/append.3.raw"}`

the offset encodes the bucket and the actual offset, `bucket << 48 | offset`
since java doesnot have unsigned longs, we can have at most 2047 buckets (11 bits)
(otherwise we would've had 4095 buckets or 12 bits)

Since the offset contains the bucket as well you increase the number of buckets, but never decrease them

```
$ curl -XPOST -d 'some text' 'http://localhost:8001/append?id=some_identifier'
{"offset":3659174697238528,"file":"/tmp/append.3.raw"}

$ curl -XPOST -d 'some other data in same identifier' 'http://localhost:8001/append?id=some_identifier'
{"offset":3659174697238553,"file":"/tmp/append.3.raw"}
```

## GET

method get /get?offset=3659174697238553 returns the data stored

```
$ curl 'http://localhost:8001/get?offset=3659174697238553'
some other data in same identifier
```

## MULTIGET
method  post /getMulti the post data is binary 8 bytes per offset (LittleEndian), it reads until EOF

example: 3659174697238528 is \x00\x00\x00\x00\x00\x00\x0d\x00 little endian
and 3659174697238553 is \x19\x00\x00\x00\x00\x00\x0d\x00
so asking for offset 3659174697238528,3659174697238553,3659174697238528 looks like:

```
$ echo -n -e '\x00\x00\x00\x00\x00\x00\x0d\x00\x19\x00\x00\x00\x00\x00\x0d\x00\x00\x00\x00\x00\x00\x00\x0d\x00' | \
  curl -X POST --data-binary @- http://localhost:8001/getMulti
	some text"some other data in same identifier	some t...


$ echo -n -e '\x00\x00\x00\x00\x00\x00\x0d\x00\x19\x00\x00\x00\x00\x00\x0d\x00\x00\x00\x00\x00\x00\x00\x0d\x00' | \
  curl -s -X POST --data-binary @- http://localhost:8001/getMulti | \
  hexdump 
0000000 09 00 00 00 73 6f 6d 65 20 74 65 78 74 22 00 00
0000010 00 73 6f 6d 65 20 6f 74 68 65 72 20 64 61 74 61
0000020 20 69 6e 20 73 61 6d 65 20 69 64 65 6e 74 69 66
0000030 69 65 72 09 00 00 00 73 6f 6d 65 20 74 65 78 74
0000040

```


output:

```
4 bytes length (LittleEndian)
data
4 bytes length (LittleEndian)
data
```

in case of error length is 0 and what follows is the error text

```
4 bytes length (LittleEndian)
data
\0\0\0\0\0
error text
```

the protocol is very simple, it stores the length of the item in 4 bytes:
`[len]some text[len]some other data in same identifier`

## STORAGE PREFIX
you can also pass "storagePrefix" parameter and this will create different directories per storagePrefix, for example

```
?storagePrefix=events_from_20171111 
?storagePrefix=events_from_20171112
```

and then you simply delete the directories you dont need

## CLOSE
Closes storagePrefix so it can be deleted

```
$ curl http://localhost:8000/close?storagePrefix=events_from_20171112
{"success":true}
```

## STORAGE FORMAT

```
header is 16 bytes
D: data length: 4 bytes
T: current time in nanosecond: 8 bytes
C: crc32(length, time): 4 bytes
V: the stored value

DDDDTTTTTTTTCCCCVVVVVVVVVVVVVVVVVVVV...DDDDTTTTTTTTCCCCVVVVVV....

```

as you can see the value is not included in the checksum, I am
checking only the header as my usecase is quite ok with
missing/corrupting the data itself, but it is not ok if corrupted
header makes us allocate 10gb in `output := make([]byte, dataLen)`


## SCAN

scans all buckets from a storagePrefix

```
$ curl http://localhost:8000/scan?storagePrefix=someStoragePrefix > dump.txt
```

the format is
[len]data[len]data...[len]data..


## LICENSE

MIT
