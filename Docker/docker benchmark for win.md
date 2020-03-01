本篇文章就是来分享一下我测试的Docker在Windows上的benchmark。
## 结论
话不多说，直接上结论。Docker在Windows上的网络性能较差，这原因主要还是因为Windows docker是基于Hyper-V的，类似的还有在Windows上装VM，具体原因不在这里详细讨论了。

## 测试环境
**Tool**: NTttcp, 和一个自己写的Disk测试工具 </br>
**Host System**: Windows Server 2016 </br>
**Host Network**: 10Gb/s  (1280MB/s) </br>
**Docker Version**: 18.09.2 </br>
**Docker Network**: NAT </br>
**Benchmark**: Network, CPU, Disk  </br>

## 测试结果
**Network**:
| Network(MB/s) | 1 | 2 | 3 | Avg. |
| ------ | ------ | ------ | -----| -----|
| Docker | 334.235 | 239.756 | 255.544 | 275.512 |
| Host | 521.410 | 1008.100 | 1051.418 | 860.309 |

**Disk** </br>
Async I/O with buffer size of 4096, read QPS of 0 rand/0 seq, write QPS of 32000 rand/0 seq for 400 seconds and 20 threads on a file of size 107374

R95%
| μs(microsecond) | 1 | 2 | 3 | 4 | 5 | Avg. |
| ------ | ------ | ------ | -----| ----- | ----- | ----- |
| Docker | 127 | 123 | 148 | 373 | 128 | 179.8 |
| Host | 159 | 140 | 122 | 134 | 256 | 162.2 |

W95%
| μs(microsecond) | 1 | 2 | 3 | 4 | 5 | Avg. |
| ------ | ------ | ------ | -----| ----- | ----- | ----- |
| Docker | 140 | 139 | 138 | 152 | 142 | 142.2 |
| Host | 137 | 139 | 140 | 141 | 141 | 139.6 |


R75%
| μs(microsecond) | 1 | 2 | 3 | 4 | 5 | Avg. |
| ------ | ------ | ------ | -----| ----- | ----- | ----- |
| Docker | 92 | 97 | 88 | 105 | 105 | 97.4 |
| Host | 95 | 94 | 96 | 93 | 102 | 96 |

W75%
| μs(microsecond) | 1 | 2 | 3 | 4 | 5 | Avg. |
| ------ | ------ | ------ | -----| ----- | ----- | ----- |
| Docker | 93 | 92 | 92 | 93 | 92 | 92.4 |
| Host | 92 | 94 | 95 | 95 | 94 | 94 |

**CPU** </br>
| Avg. CPU % | 1 | 2 | 3 | Avg. |
| ------ | ------ | ------ | -----| -----|
| Docker | 35.019 | 17.141 | 17.902 | 23.354 |
| Host | 24.142 | 19.681 | 15.476 | 19.766 |
