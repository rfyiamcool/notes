### time cost

mac m1

```
github.com/cornelk/go-benchmark	327.366s
```

mac 2019 16inch 16G intel i7

```
github.com/cornelk/go-benchmark	375.283s
```

### test method

shell

```go
➜  go-benchmark git:(master) go test -bench=. -cpu=4
goos: darwin
goarch: amd64
pkg: github.com/cornelk/go-benchmark
```

> 在各平台分别跑10次，求一个均值.

### atomic

m1

```
BenchmarkAtomicInt32-4                  	  334891	      3539 ns/op
BenchmarkAtomicInt64-4                  	  335152	      3551 ns/op
BenchmarkAtomicUintptr-4                	  333566	      3551 ns/op
```

intel

```
BenchmarkAtomicInt32-4                  	  309592	      3785 ns/op
BenchmarkAtomicInt64-4                  	  310130	      3941 ns/op
BenchmarkAtomicUintptr-4                	  300513	      3910 ns/op
```

### defer

m1

```
BenchmarkDefer-4                        	 3812605	       315 ns/op
BenchmarkDeferNo-4                      	39484240	        30.4 ns/op
```

intel

```
BenchmarkDefer-4                        	 3523532	       328 ns/op
BenchmarkDeferNo-4                      	12503139	        91.2 ns/op
```

### goroutine

m1

```
BenchmarkGoroutineNew-4                 	   10000	   2556633 ns/op
BenchmarkGoroutineChan1RteCPU-4         	   10000	   1370799 ns/op
BenchmarkGoroutineChan10RteCPU-4        	   10000	    314480 ns/op
BenchmarkGoroutineChanCPURteCPU-4       	   10000	    349479 ns/op
BenchmarkGoroutineChan100RteCPU-4       	   10000	    150737 ns/op
BenchmarkGoroutineChan10000RteCPU-4     	   10000	    159831 ns/op
BenchmarkGoroutineChan10Rte10-4         	   10000	    320326 ns/op
BenchmarkGoroutineChan100Rte100-4       	   10000	    253850 ns/op
BenchmarkGoroutineChan10000Rte10000-4   	     100	  14479701 ns/op
```

intel

```
BenchmarkGoroutineNew-4                 	   10000	   1820902 ns/op
BenchmarkGoroutineChan1RteCPU-4         	   10000	    334300 ns/op
BenchmarkGoroutineChan10RteCPU-4        	   10000	    220070 ns/op
BenchmarkGoroutineChanCPURteCPU-4       	   10000	    204412 ns/op
BenchmarkGoroutineChan100RteCPU-4       	   10000	    170625 ns/op
BenchmarkGoroutineChan10000RteCPU-4     	   10000	    175718 ns/op
BenchmarkGoroutineChan10Rte10-4         	   10000	    213040 ns/op
BenchmarkGoroutineChan100Rte100-4       	   10000	    241891 ns/op
BenchmarkGoroutineChan10000Rte10000-4   	     100	  19974006 ns/op
```

### hash

m1

```
BenchmarkHashing64MD5-4                 	 5085172	       250 ns/op	  32.05 MB/s
BenchmarkHashing64SHA1-4                	 4629715	       280 ns/op	  28.53 MB/s
BenchmarkHashing64SHA256-4              	 3403106	       354 ns/op	  22.62 MB/s
BenchmarkHashing64SHA3B224-4            	 1511235	       828 ns/op	   9.66 MB/s
BenchmarkHashing64SHA3B256-4            	 1791288	       765 ns/op	  10.45 MB/s
BenchmarkHashing64RIPEMD160-4           	 2268490	       529 ns/op	  15.11 MB/s
BenchmarkHashing64Blake2B-4             	 1834280	       707 ns/op	  11.32 MB/s
BenchmarkHashing64Blake2BSimd-4         	 1937584	       622 ns/op	  12.86 MB/s
BenchmarkHashing64Murmur3-4             	13321485	       134 ns/op	  59.49 MB/s
BenchmarkHashing64Murmur3Twmb-4         	14345737	        89.2 ns/op	  89.73 MB/s
BenchmarkHashing64SipHash-4             	18197372	        85.3 ns/op	  93.77 MB/s
BenchmarkHashing64XXHash-4              	16784458	        68.6 ns/op	 116.63 MB/s
BenchmarkHashing64XXHashpier-4          	17751019	        84.3 ns/op	  94.90 MB/s
BenchmarkHashing64HighwayHash-4         	 8982220	       141 ns/op	  56.64 MB/s
BenchmarkHashing32XXHashvova-4          	18115452	        65.8 ns/op	 121.56 MB/s
BenchmarkHashing32XXHashpier-4          	20853364	        63.9 ns/op	 125.20 MB/s
BenchmarkHashing32XXHash-4              	13377956	        98.3 ns/op	  81.40 MB/s
BenchmarkHashing16XXHash-4              	18447655	        98.3 ns/op	  81.36 MB/s
BenchmarkHashing8XXHash-4               	18637072	        68.7 ns/op	 116.47 MB/s
```

intel

```
BenchmarkHashing64MD5-4                 	 7532625	       189 ns/op	  42.33 MB/s
BenchmarkHashing64SHA1-4                	 5739030	       186 ns/op	  43.08 MB/s
BenchmarkHashing64SHA256-4              	 5050978	       242 ns/op	  33.01 MB/s
BenchmarkHashing64SHA3B224-4            	 1565026	       901 ns/op	   8.88 MB/s
BenchmarkHashing64SHA3B256-4            	 1690936	       765 ns/op	  10.45 MB/s
BenchmarkHashing64RIPEMD160-4           	 2395820	       499 ns/op	  16.02 MB/s
BenchmarkHashing64Blake2B-4             	 3472456	       341 ns/op	  23.49 MB/s
BenchmarkHashing64Blake2BSimd-4         	 3831225	       340 ns/op	  23.54 MB/s
BenchmarkHashing64Murmur3-4             	18620680	        64.3 ns/op	 124.35 MB/s
BenchmarkHashing64Murmur3Twmb-4         	19444350	        63.5 ns/op	 125.97 MB/s
BenchmarkHashing64SipHash-4             	21155768	        56.0 ns/op	 142.77 MB/s
BenchmarkHashing64XXHash-4              	31849977	        51.3 ns/op	 155.94 MB/s
BenchmarkHashing64XXHashpier-4          	28207137	        44.9 ns/op	 178.29 MB/s
BenchmarkHashing64HighwayHash-4         	13295876	        92.0 ns/op	  86.95 MB/s
BenchmarkHashing32XXHashvova-4          	33265984	        35.6 ns/op	 224.68 MB/s
BenchmarkHashing32XXHashpier-4          	31837797	        37.0 ns/op	 216.20 MB/s
BenchmarkHashing32XXHash-4              	21471313	        56.1 ns/op	 142.53 MB/s
BenchmarkHashing16XXHash-4              	21619030	        55.0 ns/op	 145.49 MB/s
BenchmarkHashing8XXHash-4               	31882546	        36.8 ns/op	 217.49 MB/s
```

### value

m1

```
BenchmarkValueUnsafePointer-4           	53164192	        22.3 ns/op
BenchmarkValueInterface-4               	23458231	        51.0 ns/op
BenchmarkReflect-4                      	 2154338	       555 ns/op
BenchmarkCast-4                         	18145458	        66.1 ns/op
BenchmarkParameterPassedByPointer-4     	12744134	        92.5 ns/op
BenchmarkParameterPassedByValue-4       	12046428	        93.0 ns/op
```

intel

```
BenchmarkValueUnsafePointer-4           	68717324	        17.0 ns/op
BenchmarkValueInterface-4               	30155677	        38.9 ns/op
BenchmarkReflect-4                      	 7165081	       165 ns/op
BenchmarkCast-4                         	18640602	        62.4 ns/op
BenchmarkParameterPassedByPointer-4     	16345183	        76.5 ns/op
BenchmarkParameterPassedByValue-4       	10353224	       120 ns/op
```

### slice list

m1

```
BenchmarkSliceReadRange-4               	54937928	        21.7 ns/op
BenchmarkSliceReadForward-4             	38538789	        31.0 ns/op
BenchmarkSliceReadBackwards-4           	33178958	        36.0 ns/op
BenchmarkSliceReadLastItemFirst-4       	38209411	        31.4 ns/op
BenchmarkSliceFillByIndex-4             	55740060	        21.5 ns/op
BenchmarkSliceFillByIndexMake-4         	56151075	        26.9 ns/op
BenchmarkSliceFillMakeAppend-4          	50314112	        23.9 ns/op
BenchmarkSliceFillAppendNoMake-4        	 3114499	       601 ns/op
BenchmarkSliceFillSmallMakeAppend-4     	 2443990	       943 ns/op
BenchmarkFillLinkedListPushBack-4       	  619824	      3911 ns/op
BenchmarkFillLinkedListPushFront-4      	  692575	      2082 ns/op
```

intel

```
BenchmarkSliceReadRange-4               	59617837	        19.7 ns/op
BenchmarkSliceReadForward-4             	18860553	        61.5 ns/op
BenchmarkSliceReadBackwards-4           	18713661	        62.0 ns/op
BenchmarkSliceReadLastItemFirst-4       	47315362	        25.2 ns/op
BenchmarkSliceFillByIndex-4             	72714333	        16.3 ns/op
BenchmarkSliceFillByIndexMake-4         	35459602	        31.7 ns/op
BenchmarkSliceFillMakeAppend-4          	41651622	        28.7 ns/op
BenchmarkSliceFillAppendNoMake-4        	 3953661	       671 ns/op
BenchmarkSliceFillSmallMakeAppend-4     	 2419346	       430 ns/op
BenchmarkFillLinkedListPushBack-4       	  498951	      2398 ns/op
BenchmarkFillLinkedListPushFront-4      	  503970	      2394 ns/op
```

### sync

m1

```
BenchmarkSyncRWMutex-4                  	11111484	       109 ns/op
BenchmarkSyncRWAtomic-4                 	10568516	       113 ns/op
BenchmarkSyncRWAtomicGosched-4          	13135485	        91.8 ns/op
```

intel

```
BenchmarkSyncRWMutex-4                  	19353127	        61.8 ns/op
BenchmarkSyncRWAtomic-4                 	 4075513	       303 ns/op
BenchmarkSyncRWAtomicGosched-4          	18012901	        68.9 ns/op
```