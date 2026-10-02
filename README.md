# KOWALSKIANALYSIS
A fixed bug in the Linux kernel related to the ATH9271 driver.
All experimentation happened within a limited-closed lab and all policies of responsible disclosure were followed.

# ath9k_htc: unchecked TX aggregation exceeds `MAX_TX_BUF_SIZE`

**File:** `drivers/net/wireless/ath/ath9k/hif_usb.c`

**Tested:** Raspberry Pi 4, Raspberry Pi OS Trixie, kernel `6.18.39+rpt-rpi-v8`,
AR9271 (`0cf3:9271`), interface `wlan1`.

## Two limits, one buffer

The driver allocates eight TX buffers. On the tested 6.18 kernel each data
buffer is:

```c
tx_buf->buf = kzalloc(MAX_TX_BUF_SIZE, GFP_KERNEL);  /* 32768 */
```

`__hif_usb_tx()` copies queued frames into one of those buffers. It caps the
batch by record count, not by bytes:

```text
MAX_TX_BUF_SIZE = 32768     size of tx_buf->buf
MAX_TX_AGGR_NUM = 20        max records copied in one drain
```

Those are independent. Twenty records can exceed 32768 bytes.

Relevant parts of the function are shown below; queue/statistics bookkeeping
not relevant to the size calculation is omitted.

```c
tx_skb_cnt = min_t(u16, hif_dev->tx.tx_skb_cnt, MAX_TX_AGGR_NUM);

for (i = 0; i < tx_skb_cnt; i++) {
	nskb = __skb_dequeue(&hif_dev->tx.tx_skb_queue);
	hif_dev->tx.tx_skb_cnt--;

	buf = tx_buf->buf;
	buf += tx_buf->offset;
	hdr = (__le16 *)buf;
	*hdr++ = cpu_to_le16(nskb->len);
	*hdr++ = cpu_to_le16(ATH_USB_TX_STREAM_MODE_TAG);
	buf += 4;
	memcpy(buf, nskb->data, nskb->len);
	tx_buf->len = nskb->len + 4;

	if (i < (tx_skb_cnt - 1))
		tx_buf->offset += (((tx_buf->len - 1) / 4) + 1) * 4;

	if (i == (tx_skb_cnt - 1))
		tx_buf->len += tx_buf->offset;
}

usb_fill_bulk_urb(..., tx_buf->buf, tx_buf->len, ...);
usb_submit_urb(...);
```

There is no check that the next record fits in `MAX_TX_BUF_SIZE` before the
header stores or `memcpy()`.

Data frames use bulk OUT endpoint 1 (`USB_WLAN_TX_PIPE`). Endpoint 4 is used by
the register-output path and is not this aggregation path.

---

## Record layout and math

Each record is a 4-byte stream header (`le16` length + `le16` tag) plus
`nskb->len` bytes. Every record except the last advances `offset` to a
4-byte boundary:

```text
record = 4 + nskb->len
stride = ALIGN(record, 4)          /* all but the last record */
```

The flood used a 2200-byte UDP payload at MTU 2304. The driver logged
`nskb->len=2292`; that logged length is the value the loop copies.

```text
ALIGN(2292 + 4, 4) = 2296
20 * 2296 = 45920
45920 - 32768 = 13152 bytes past the allocation
```

The 13152-byte figure is the extent from the allocation boundary to the end
of the aggregate. It includes record headers as well as bytes copied from
the skbs.

Same formula, twenty equal records, first overflowing integer length:

```text
N=1632:  20 * 1636 = 32720     fits (48 bytes left)
N=1633:  19 * 1640 + 1637 = 32797     29 bytes past 32768
```

N=1633 was not a measured batch. It is the same arithmetic on the loop
above. Mixed sizes can cross the same byte limit.

---

## What the 20-frame batch looks like in memory

The following uses exclusive end offsets:

```text
valid allocation [0, 32768):
  [0, 32144)       records 1-14                 32144 bytes
  [32144, 32148)   record 15 header                  4 bytes
  [32148, 32768)   record 15 data[0..619]           620 bytes

out of bounds [32768, 45920):
  [32768, 34440)   record 15 data[620..2291]       1672 bytes
  [34440, 45920)   records 16-20                  11480 bytes

                         first OOB byte
                              |
                              v
  0 ---------------------- 32768 ------------------------------ 45920
  |       32768-byte allocation       |       13152-byte extent   |
```

The loop stops at 20 records. Extra queued frames are left for later drains,
not added to this aggregate.

A large batch forms for normal/AMPDU frames because `hif_usb_send_tx()` only
submits immediately when every TX buffer is free and fewer than two frames are
queued:

```c
if ((hif_dev->tx.tx_buf_cnt == MAX_TX_URB_NUM) &&
    (hif_dev->tx.tx_skb_cnt < 2))
	__hif_usb_tx(hif_dev);
```

Otherwise frames stay on `tx_skb_queue` until a TX URB completes, then one
drain copies up to 20. Four parallel UDP senders produced batches of 16, 19,
and 20 frames in separate runs. A 1-frame drain does not overflow for the
measured frame size.

---

## Evidence

`WOULD OVERFLOW` in the logs is diagnostic text. The instrumented loop still
did the copy.

### Probe 

```text
USB009: ath9k_hif_usb_alloc_tx_urbs START -- allocating 8 TX buffers, each 32768 bytes
USB009: alloc_tx_urbs[0] ... ksize(buf)=32768 (order=3, 8 pages)
...
USB009: alloc_tx_urbs[7] ... ksize(buf)=32768 (order=3, 8 pages)
USB009: ath9k_hif_usb_alloc_tx_urbs DONE -- 8 buffers allocated, tx_buf_cnt=8
```

### 20-frame batch 

No delay, no skipped drain, copy not stopped:

```text
USB009: [14/20] off=29848 nskb_len=2292 rec_end=32144 buf_ksize=32768 fits
USB009: [15/20] off=32144 nskb_len=2292 rec_end=34440 buf_ksize=32768 *** WOULD OVERFLOW ***
USB009: *** OVERFLOW at entry 15/20: off=32144 + 4 + skb=2292 = rec_end=34440 > 32768, overflow=1672 bytes; dst=buf+32148, first OOB byte at buf+32768 (nskb->data[620])
USB009: [16/20] off=34440 nskb_len=2292 rec_end=36736 buf_ksize=32768 *** WOULD OVERFLOW ***
USB009: [17/20] off=36736 nskb_len=2292 rec_end=39032 buf_ksize=32768 *** WOULD OVERFLOW ***
USB009: [18/20] off=39032 nskb_len=2292 rec_end=41328 buf_ksize=32768 *** WOULD OVERFLOW ***
USB009: [19/20] off=41328 nskb_len=2292 rec_end=43624 buf_ksize=32768 *** WOULD OVERFLOW ***
USB009: [20/20] off=43624 nskb_len=2292 rec_end=45920 buf_ksize=32768 *** WOULD OVERFLOW ***
USB009: *** OVERFLOW at entry 20/20: off=43624 + 4 + skb=2292 = rec_end=45920 > 32768, overflow=13152 bytes; dst=buf+43628, first OOB byte at buf+32768 (nskb->data[0])
USB009: *** OVERSIZED URB *** final_len=45920 > 32768 (overflow=13152 bytes) buf=ffffff802e8e8000
```

A 16-frame batch: `16 * 2296 = 36736` (3968 bytes past 32768).

### Canary tail 

This is an alternative to KASAN.

Separate lab build: `kzalloc(32768 + 4096)`, tail filled with `0xde`. The
allocator returned `ksize=65536`. That is not the stock 32768-byte object.
It only shows writes past the driver's logical 32768-byte region. 19-frame
batch, `final_len=43624`:

```text
USB009: [19/19] off=41328 nskb_len=2292 rec_end=43624 buf_ksize=65536 *** WOULD OVERFLOW ***
USB009: *** CANARY BROKEN (immediate) *** at buf+32768 (canary+0) = 0x41, 4096 contiguous OOB bytes; buf=ffffff8003ca0000, final_len=43624 offset=41328
USB009 canary: 00000000: 41 41 41 41 41 41 41 41 41 41 41 41 41 41 41 41  AAAAAAAAAAAAAAAA
```

The shown `0x41` bytes are the same byte pattern used by the flood
(`b"\x41" * 2200`). The check
ran after `memcpy()` and before `usb_submit_urb()`. Completion later reported
the same tail.

This deliberately enlarges the allocation (`ksize=65536`) so the write past
`buf+32768` stays inside the chunk. In that lab build it demonstrates the byte
pattern crossing the logical boundary; it does not reproduce the
stock-neighbor crash.

### Stock kernel, `usbmon`

Filter output when a bulk-OUT
submit exceeded 32768:

```text
>>> OOB PROOF: Submitted length = 45920 bytes (Buffer allocation was 32768) <<<
```

`usbmon` reported the requested submission length (45920). The 32768 in that
line is text from the awk filter, based on `MAX_TX_BUF_SIZE` and the separate
probe log; it is not a `usbmon` field. The `OOB PROOF` label and parenthetical
were also emitted by the awk filter. The `usbmon` result alone does not establish
the allocation size or prove the CPU write; those points come from the
source, probe/logging run, and canary run. It is a host-side request trace; it
does not establish that the host controller or device accepted the complete
requested length.

### Negative control

The same logging module and flood were also used at MTU 1500. The expected
result is no `OVERFLOW` / `OVERSIZED` lines: the 2200-byte datagram is
fragmented, so the frames in the batch are smaller and twenty of them fit in
32768. This is a useful discriminator, but the primary proof above does not
depend on it.

## Claim boundaries

The evidence establishes an unchecked host-side write beyond the tested
32768-byte TX data region and an oversized URB length on the tested kernel.
It does not establish what adjacent memory contains, whether a particular
crash was caused by this write, or what the AR9271 firmware does with an
oversized or corrupted transfer. It is a local TX-path reproduction on an
associated adapter; no claim is made here about an unassociated over-the-air
transmitter reaching this path.

The possible impact of the write is also not fully evaluated. If allocation
placement can be influenced (for example, through allocator/page-layout
manipulation sometimes called "feng shui"), the overrun could overlap live
nearby kernel memory or another object. That could corrupt kernel state or
cause a kernel crash, or potentially provide a basis for further exploitation.
This report does not demonstrate reliable placement, control of a particular
neighboring object, code execution, privilege escalation, or further
exploitation; those remain untested.

The report was prepared with AI assistance. In accordance with the kernel
security-bug guidance, send the reproducer privately to the maintainers and
security team rather than posting it to a public list.

---
This is a local TX-path reproduction on an associated adapter. We demonstrated no OTA attacks.

---
---

Fix: https://lore.kernel.org/linux-wireless/20260831173931.1672-1-gck.kara@gmail.com/#Z31drivers:net:wireless:ath:ath9k:hif_usb.c

Currently applied to kernel master: https://github.com/torvalds/linux/blob/master/drivers/net/wireless/ath/ath9k/hif_usb.c

