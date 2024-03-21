## microbench 跑分虚高 bug
## 现象
microbench 跑分比ref高三倍，文档里说这里买了坑，于是心里有点虚，时钟的实现大概率是出问题了：![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a6551c3b1f5c4787b0afae941978593d~tplv-k3u1fbpfcp-watermark.image?)

### 分析
（经提示后发现的）首先看时钟的调用链路：
```
am-tests/src/tests/rtc.c:7 io_read(AM_TIMER_UPTIME)
$AM_HOME/klib/include/klib-macros.h:20 ioe_read(reg, xxx)
$AM_HOME/am/src/platform/nemu/ioe/ioe.c:58 lut[reg])(buf)
$AM_HOME/am/src/platform/nemu/ioe/timer.c:7 __am_timer_uptime()
```
这是在abstract-machine中的调用链路，可以知道最底层是从 `RTC_ADDR` 地址处读的两个4字节数据；
为什么从这个地址中可以读出更新的时间？——很简单，因为nemu模拟计算机在这里写入了数据，再看一下nemu是怎么完成读地址的：
```
inst.c:22 #define Mr vaddr_read
vaddr.c:24 paddr_read
```
paddr_read函数如下：
```C
word_t paddr_read(paddr_t addr, int len) {
  if (likely(in_pmem(addr))) return pmem_read(addr, len);
  IFDEF(CONFIG_DEVICE, return mmio_read(addr, len));
  out_of_bound(addr);
  return 0;
}
```
这里一定会进入mmio_read函数：
```C
word_t mmio_read(paddr_t addr, int len) {
  return map_read(addr, len, fetch_mmio_map(addr));
}
word_t map_read(paddr_t addr, int len, IOMap *map) {
  assert(len >= 1 && len <= 8);
  check_bound(map, addr);
  paddr_t offset = addr - map->low;
  invoke_callback(map->callback, offset, len, false); // prepare data to read
  word_t ret = host_read(map->space + offset, len);
  return ret;
}
```
map_read里会调用map->callback函数指针，这是在init_timer里注册的：
```C
void init_timer() {
  rtc_port_base = (uint32_t *)new_space(8);
#ifdef CONFIG_HAS_PORT_IO
  add_pio_map ("rtc", CONFIG_RTC_PORT, rtc_port_base, 8, rtc_io_handler);
#else
  add_mmio_map("rtc", CONFIG_RTC_MMIO, rtc_port_base, 8, rtc_io_handler);
#endif
  IFNDEF(CONFIG_TARGET_AM, add_alarm_handle(timer_intr));
}
static void rtc_io_handler(uint32_t offset, int len, bool is_write) {
  assert(offset == 0 || offset == 4);
  if (!is_write && offset == 4) {
    uint64_t us = get_time();
    rtc_port_base[0] = (uint32_t)us;
    rtc_port_base[1] = us >> 32;
  }
}
```
而在这个回调函数中，发现了一个奇怪的地方：offset == 4 的时候才会有动作，否则就什么都不做。但是稍微一想，就明白了。这个动作是时间的更新，而在am-test中进行了两次读4字节的操作，如果两次都动作的话，就更新了两次时间，实际上只需要更新一次时间就好了。
再然后，我习惯性的在am中写先读前4个字节，再读后4个字节，但实际上前4个字节不触发时间更新，我读的是上一时刻的时间，而不是这一时刻的，所以时钟实现的有问题！

### 解决
解决方案有两种，第一在am中改变读的顺序，先读后4个字节触发时间更新，再读前4个字节；
第二是在nemu中改变触发时间更新的offset为0，这样就能在读前4个字节时就触发更新，不用调整am中的读取顺序了。