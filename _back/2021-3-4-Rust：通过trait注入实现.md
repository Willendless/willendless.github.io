---
layout: "post"
title: Rust：通过trait注入实现
author: LJR
category: 编程语言
tags:
    - rust
---

+ 区分数据和行为
+ 组合的优势：
  + 利用行为约束数据：Phantom
  + 利用行为增加数据的行为
  + 利用行为约束行为

The vfat/vfat.rs file also provides VFatHandle trait, which defines a way to share mutable access to VFat instance in a thread-safe way. When implementing your file system, you’ll likely need to share mutable access to the file system itself among your file and directory structures. You’ll rely on this trait to do so. Use clone() method for replicating the handle and lock() method for entering the critical section where the code can access &mut VFat.

VFat and a few other types in our file system such as File and Dir are generic over HANDLE type parameter that implements VFatHandle trait. This design allows the user of the library to inject lock implementation by implementing VFatHandle trait on their own type. Our kernel uses PiVFatHandle struct which internally uses its custom Mutex implementation, while the test code uses StdVFatHandle struct which is implemented with types in the standard library.

VFat is generic over VFatHandle, but VFat doesn’t physically own VFatHandle. The relationship is reverse; implementors of VFatHandle will manage VFat as their field. To represent such relationship, zero-sized marker type PhantomData has been added to VFat.

VFat is generic over VFatHandle, but VFat doesn’t physically own VFatHandle. The relationship is reverse; implementors of VFatHandle will manage VFat as their field. To represent such relationship, zero-sized marker type PhantomData has been added to VFat.

```rust
#[derive(Debug)]
pub struct VFat<HANDLE: VFatHandle> {
    phantom: PhantomData<HANDLE>,
    device: CachedPartition,
    bytes_per_sector: u32, // TODO: previously u16
    sectors_per_cluster: u8,
    sectors_per_fat: u32,
    fat_start_sector: u64,
    data_start_sector: u64,
    rootdir_cluster: Cluster,
}
```

### 通过blanker implementation简化代码

实现源码中调用该函数时需要提供具体的HANDLE类型，这样才能知道调用哪个实现了VFatHandle trait的类型方法。

```rust
impl<HANDLE: VFatHandle> VFat<HANDLE> {
    pub fn from<T>(mut device: T) -> Result<HANDLE, Error>
    where
        T: BlockDevice + 'static,
    {
        // data in partition_entry
        let mut flag = false;
        let mut partition_start_sector: u64 = 0;
        let mut partition_physical_sectors_num: u64 = 0;
        let mut bios_parameter_block: BiosParameterBlock = Default::default();

        let master_boot_record = MasterBootRecord::from(&mut device)?;
        for partition_entry in master_boot_record.partition_table.iter() {
            // currently only able to handle fat32
            if FAT32_PARTITION_TYPE.contains(&partition_entry.partition_type) {
                flag = true;
                partition_start_sector = partition_entry.relative_sector as u64;
                partition_physical_sectors_num = partition_entry.total_sectors_in_partition as u64;
                bios_parameter_block = BiosParameterBlock::from(&mut device, partition_entry.relative_sector as u64)?;
                break;
            }
        }

        if !flag {
            return Err(Error::Io(newioerr!(NotFound, "failed to find FAT32 format partition")));
        }

        let fat_start_sector = bios_parameter_block.reserved_sectors_num as u64;
        let fat_num = bios_parameter_block.fat_num as u64;
        let sectors_per_fat = bios_parameter_block.sectors_per_fat_2;
        let bytes_per_logical_sector = bios_parameter_block.bytes_per_sector as u32;
        let partition = Partition {
            start: partition_start_sector,
            num_sectors: partition_physical_sectors_num * 512 / (bytes_per_logical_sector as u64),
            sector_size: bytes_per_logical_sector as u64,
        };
        Ok(HANDLE::new(VFat {
            phantom: PhantomData,
            device: CachedPartition::new(device, partition),
            bytes_per_sector: bytes_per_logical_sector,
            sectors_per_cluster: bios_parameter_block.sectors_per_cluster,
            sectors_per_fat: sectors_per_fat,
            fat_start_sector: fat_start_sector,
            data_start_sector: fat_start_sector + fat_num * (sectors_per_fat as u64),
            rootdir_cluster: Cluster::from(bios_parameter_block.rootdir_cluster),
        }))
    }
```


### 从`From`trait的角度考虑
