# Virtio Overview

Packed virtqueue, which complements the split virtqueue has been merged in the [virtio 1.1](https://docs.oasis-open.org/virtio/virtio/v1.1/cs01/virtio-v1.1-cs01.html) spec.

## Virtio devices: In and out the virtual world

A virtio device is a device that exposes a virtio interface for the software to manage and exchange information. It can be exposed to the emulated environment using PCI, Memory Mapping I/O (Just to exposed to the device in a region of memory).

The virtio interface consist of the following mandatory parts:

* Device status field
* Feature bits
* Notifications
* One or more virtqueues

### Device status field: Is everything ok?

The device status field is a sequence of bits the device and the driver use to perform their initialization.

There are serval status bits, let's review them one by one:

* **ACKNOWLEDGE (0x1)**: The guest driver set the bit in the device status field to indicate that it acknowledges the device.
* **DRIVER (0x2)**: Indicate an initialization in progress. After that, it starts a feature negotiation using the feature bits.
* **DRIVER_OK (0x4), FEATURES_OK (0x8)**: After negotiation of the feature, the driver can set the two bits to acknowledge the features, so communication can start.
* **DEVICE_NEEDS_RESET (0x40)**: If the device wants to indicate a fatal failure, it can set this bit.
* **FAILED (0x80)**: If the driver wants to indicate a fatal failure, it can set this bit.

### Feature bits: Setting the communication agreement points

Device's feature bits are used to communicate what features it supports, and to agree with the drivers about what of them will be used.

The driver will reads the feature bits the device offers, and sends back the subset that it can handle. If they agree on them, the driver will allocate and inform about the virtqueues to the device, and all other configuration needed.

### Notifications

Devices and drivers must notify that they have information to communicate using a notification.

Like the virtio_pci device will use the PCI interruption to notify.

#### One or more virtqueues: The communication vehicles

A virtqueue is just a queue of guest's buffers that the host consumes, either reading them or writing to them, and returns to the guest.

The current memory layout of a virtqueue implementation is a circular ring, so it is often called the virtring or vring.

## Virtio drivers

The virtio drivers will do some things:

* Look for the device
* To allocate shared memory in the guest for the communication

## Virtqueues

The mechanism for bulk data transport on virtio devices is called a virtqueue. Each device can have more virtqueues. Each queue has a 16-bit queue size parameter, which sets the number of entries and implies the total size of the queue.

Each virtqueue consists of three parts:

* Descriptor Table
* Available Ring
* Used Ring

Each part is physically-contiguous in guest memory.

`Queue Size` corresponds to the maximum number of buffers in the virtqueue. Queue Size value is always a power of 2. The maximum Queue Size value is 32768.

When the driver wants to send a buffer to the device, it fills in a slot in the descriptor table (or chains several together), and writes the descriptor index into the available ring. It then notifies the device. When the device has finished a buffer, it writes the descriptor index into the used ring, and sends an interrupt.

### The Virtqueue Descriptor Table

The descriptor table refers to the buffers the driver is using for the device. It's definition is showed as below:

```C
struct virtq_desc {
    // Guest physical Address
    le64 addr;
    
    // Length
    le32 len;

/* This marks a buffer as continuing via the next field. */
#define VIRTQ_DESC_F_NEXT	1
    le16 flags;
    
    le16 next;
};
```

### The virtqueue Available Ring





## References

https://www.redhat.com/en/blog/virtio-devices-and-drivers-overview-headjack-and-phone