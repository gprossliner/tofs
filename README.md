# What is tofs?

  **t**-transactional  
  **o**-open  
  **f**-file  
  **s**-system  

tofs in the current version is implemented for environments with
limited memory and processing resources. It compiles in less than 15k of
flash program memory, and uses up from 1k (depending on the configuration) of ram.

# PROS
* tofs is a fully transactional.
* With tofs each file can have a priority level assigned to it. Low priority
files are automatically overwritten on demand.
* tofs don't allocate on the heap. All memory is allocated statically in source, and can be controlled by constants in tofs_config.h
* tofs don't implement a block-device, so it's independent of a specific kind of memory,
you can use it for NOR or NAND flash, eeprom, ram, in another file, ...
* To port it to a specific block-device, a struct has to be initialized, containing
function pointers to the specific implementation.
* It's organized in sequentially writting log records, so it's good for wearing
storage like flash.
* Deleted data is automatically recycled, deletion is done on "block" level,
while a block-device can specify the block-size in powers of two.
* Because data can also be stripped from the start of files, it's very good in
implementing persistent queues.

# LIMITS
* There is no intention to make a stdio.h / posix compatible filesystem.
  * This would require a massive filesystem with much more complexity.
  * The stdlib file api don't support features like transactions, priorities and queues.
  * Even if you need a full file system in your project, you may use tofs too,
  as a tofs volume can be mounted on a random access file, of an underlying file system.
  A ready to use implementation may be provided in future.
* tofs is good for sequential read access, random seek operations are possible,
but there is no cached data to support this operation.
* tofs allows no random write access. A file can only be replaced, or data can be appended to a file. For small files (like config files), this is normally no problem, because they can be rewritten completely.
* tofs currently don't support directories. All files are located in the root directory.
* a single tofs volume is limited to 2^32 (4GB)

# Example
I'll start with an example, because code says more than 1000 words.
Details will follow....

```c
#include "tofs.h"
#include "tofs_ramvolume.h"

char volume_buffer[0x2000];

int main()
{
  // create a volume in ram for this demo
  tofs_handle volume;

  tofs_ramvolume_init (
    (void*)volume_buffer,
    0x2000,  // 8k total space
    0x10,   // 1k block size
    &volume
  );

  // mount tofs to it, true specifies that it's automatically formatted.
  tofs_mount(volume, true);

  // shows how to write data to a file
  producer(volume);

  // shows how to consume data
  consumer(volume);

  // shows how to perform file enumeration
  list(volume);

  // deletion of a file
  cleanup(volume);

  return 0;
}

struct record {
  int i;
  int j;
}

void producer(tofs_handle volume)
{
  tofs_handle handle;

  tofs_open (
    volume,
    // the name of the file. max 15 chars
    "queue.dat",  
    // specifiy flags to set options for the file, like high priority
    tofs_file_flag_priority_high,
    // open in append mode
    tofs_open_append,
    &handle
  );

  // write 10 small records to the file
  for(int i=0; i<10; i++) {

    // sample data
    struct record buffer = {
      .i = i,
      .j = i * 10
    };

    // writing two times must be ACID, so we use an explicit transaction
    // here. If we wouldn't, each write would span an own implicit one.
    tofs_transaction(volume);

    // write the record #1
    tofs_write(h_writer, (void*)&buffer, sizeof(record));

    // write the record #2
    rec.j = 0;
    tofs_write(h_writer, (void*)&buffer, sizeof(record));

    // commiting the transaction
    tofs_commit(volume);
  }

  tofs_close(h_writer);
}

void consumer(tofs_handle volume)
{
  tofs_handle handle;

  tofs_open(
    volume,
    // the name of the file. max 15 chars
    "queue.dat",
    // by specifing default, we keep the existing flags
    tofs_file_flag_default,
    // if open_queue is specified, we can set the bookmark
    // we don't want a new file to be created on demand here
    tofs_open_queue | tofs_file_flag_dont_create,
    &handle
  );

  // read the records
  for(int i=0; i<10; i++) {

    // sample data
    struct record buffer;
    size_t record_size;

    // the size of the next record is tracked in metadata,
    // so we can query by using a null-buffer
    tofs_read(handle, NULL, NULL, &record_size);
    ASSERT(record_size == sizeof(buffer));

    // read the record #1
    tofs_read(handle, (void*)&buffer, sizeof(record), &record_size);
    printf("Record#1(%d,%d)\n", buffer.i, buffer.j);

    // read the record #2
    tofs_read(handle, (void*)&buffer, sizeof(record), &record_size);
    printf("Record#2(%d,%d)\n", buffer.i, buffer.j);

    // marks all data before the current position for truncation
    // after bookmarking, the record is no more accessible.
    // because it is a write-operation, this could be included
    // in an explicit transaction too, but this is not nessersary here,
    // so we do it implicitly.
    tofs_bookmark(handle);

  }

  tofs_close(handle);
}

void list(tofs_handle volume)
{
  tofs_file_info file_list[10];
  size_t nro_files;

  tofs_list(volume, &file_list, sizeof(file_list), nro_files, tofs_list_default);
  for(size_t i=0; i<nro_files; i++){
    printf("%s size:%d flags:%d",
      file_list[i].name,
      file_list[i].size,
      file_list[i].flags);
  }

  // we could also use tofs_list_callback() with a fn-pointer instead
  // of the buffer-version
}

void delete(tofs_handle volume)
{
  // delete starts an implicit transaction here
  tofs_delete(volume, "queue.dat");
}

```

## Operation overview
### Volume

tofs don't access any storage hardware. To use tofs a tofs volume must be implemented
for a specific kind of memory hardware.

Currently the following implementations are provided:
* tofs_ramvolume: volume on a memory buffer
* tofs_atmel_internal_flash: volume on the flash-controller, using the atmel software framework


The block-device must have the following capabilities:
* It is organized in a flat memory space
* The total memory space is organized in equal sized blocks, where the size is
a power of two.
* Each block can erased independently
* After a block is erased, all it's memory should be FF. If it's another value (like 00),
you may port it to tofs by running a xor of it's bits before handing it over to tofs.
* To make transaction rollbacks persistent, existing data is overwritten with a zero marker byte. It must be possible without erasing the block. This is normally possible
with flash memory, as individual bits can be set to 0 in more than one operation.
* To ensure ACID volumes must do all writes in original order. Phyiscal writing may be delayed only on specific calls.

#### Notes for implementors:

* Checking args is not needed, because the args are validated and properly masked in
 nfs implementation. Of cause there could be a bug, but in this case we are unhandled anyway
* The maximum size of a read or write request is never greater than the block-size,
and will never span more than one block.

#### Block-Size
The offset_bits field specifies the size of a block. Blocks sizes must always be
powers of two.

Values:

| offset_bits | size  |
|-------------|-------|
| 8           | 256b  |
| 9           | 512b  |
| 10          | 1k    |
| 11          | 2k    |
| 12          | 4k    |
| 14          | 16k   |
| 15          | 32k   |
| 16          | 64k   |

If a hardware allows is to choose between different block sizes, implementors
are free to let the user choose any if it.
The block-size can't be changed for a volume without formating.

If the block-size gets bigger...
* ... you can store a smaller number of files, because a single block can't be shared
between files.
* ... the storage efficiency is better. A single transactional write-operation
cannot span multiple blocks. So if a write-operation don't fit in the current block,
a new block needs to be allocated, and the rest of the block's memory is unused
* ... lesser metadata needs to be written and the allocator runs less frequent,
so the overhead is reduced.
