---
layout: post
title: Use Map-in-Map in BPF programs via Libbpf
tags: [BPF,libbpf]
---

# Introduction
Among all BPF map types, two special ones, `BPF_MAP_TYPE_ARRAY_OF_MAPS` and
`BPF_MAP_TYPE_HASH_OF_MAPS`, are more complex than others. As the names imply,
they are “map-in-map”, meaning that the value of each entry is also a map.

![Map in Map](/img/map_in_map.jpg)


Due to this indirectness, CRUD (Creat, Read, Update, Delete) operations on a
map-in-map are different from those of a regular single-level map. In this
post, I will use a simple example to illustrate the basic usage of a map-in-map
and some caveats.

**Note:** Since the only difference between `BPF_MAP_TYPE_ARRAY_OF_MAPS` and
`BPF_MAP_TYPE_HASH_OF_MAPS` is whether the outer map is an array or a hash map,
I will only use `BPF_MAP_TYPE_ARRAY_OF_MAPS` as a representative. Also, error
handling is ignored for simplicity.

# Create
During load time, all regular maps defined in a BPF C program are created.
However, a map-in-map is different in that you only have to define the outer
map in the source code. It makes sense since inner maps are created and
inserted into the outer map at runtime by the user. The map meta data is
defined in `struct bpf_map_def`, same as all other regular map types. For
example,

```
struct bpf_map_def SEC("maps") outer_map = {
    .type = BPF_MAP_TYPE_HASH_OF_MAPS,
    .key_size = sizeof(__u32),
    .value_size = sizeof(__u32), // Must be u32 becuase it is inner map id
    .max_entries = 1,
};
```

**Note:**
* The value size of an outer map must be `__u32`, which is the size of inner
  map id (See Lookup section below for more details).
* `struct bpf_map_def` is a helper structure used by BPF C program to describe
  map attributes to the `elf_bpf` loader. You can define it yourself or you can
  use [the one defined in
  `bpf_helpers.h`](https://elixir.bootlin.com/linux/v4.16.18/source/tools/testing/selftests/bpf/bpf_helpers.h#L104).

Though you don’t have to define the inner map in BPF C program, the verifier
needs to know the inner map definition during load time. Therefore, before
calling `bpf_object__load`, you must create a dummy inner map and set its file
descriptor (fd) to the outer map by calling `bpf_map__set_inner_map_fd`. **The
dummy map fd must be closed after load since its only used to make the verifier
happy.**

```c
const char* outer_map_name = "outer_map";
struct bpf_map* outer_map = bpf_object__find_map_by_name(obj, outer_map_name);
int inner_map_fd = bpf_create_map(
    BPF_MAP_TYPE_HASH,  // type
    sizeof(__u32),      // key_size
    sizeof(__u32),      // value_size
    8,                  // max_entries
    0);                 // flag
bpf_map__set_inner_map_fd(outer_map, inner_map_fd);
bpf_object__load(obj);
close(inner_map_fd); // Important!
```

# Insert
## Insert into Outer Map
There are 3 steps to insert an entry into the outer map:

1. Create a new inner map.
2. Insert an entry to the outer map with the the **inner map fd** as the value.
3. **Close the inner map fd.**

```c
int inner_map_fd = bpf_create_map_name(
    BPF_MAP_TYPE_HASH,   // type
    "hechaol_inner_map", // name
    sizeof(__u32),       // key_size
    sizeof(__u32),       // value_size
    8,                   // max_entries
    0);                  // flag
__u32 outer_key = 42;
bpf_map_update_elem(outer_map_fd, &outer_key, &inner_map_fd, 0 /* flag */);
close(inner_map_fd); // Important!
```

**Note:**
* The value of each entry in an outer map is **the id of an inner map** (See
  Lookup section below). However, when you call `bpf_map_update_elem` API,
  **the value you give is the fd of the inner map.**
* You have to **close the inner map fd after insertion to avoid memory leak.**
  Because after map creation, you have a reference (fd) to the map in user
  space. After insertion, kernel has a reference to the map in kernel space. If
  you don’t close the map fd now, then after the entry is deleted from the
  outer map (kernel releases its reference), the inner map resource won’t be
  cleaned up since you still have the reference even though you may have lost
  the fd after your insertion function returns.

## Inert into Inner Map
As we mentioned above, the value of an entry in the outer map is the **id (NOT
fd)** of the inner map. Though we use **inner map fd** as the value when
calling `bpf_map_update_elem`, we get back the inner map id when calling
`bpf_map_lookup_elem`. To get the map fd, we need to call
`bpf_map_get_fd_by_id`.

```c
  const __u32 outer_key = 42;
  __u32 inner_map_id;
  bpf_map_lookup_elem(outer_map_fd, &outer_key, &inner_map_id);
  int inner_map_fd = bpf_map_get_fd_by_id(inner_map_id);
  const __u32 inner_key = 12;
  __u32 inner_value;
  bpf_map_lookup_elem(inner_map_fd, &inner_key, &inner_value);
  // ... Use inner_value;
  close(inner_map_fd); // Important!
```

**Note:**
* Each call to `bpf_map_get_fd_by_id` returns a new fd. **You have to close it
  after use to avoid memory leak.**

# Delete
Delete operation is as simple as other regular maps. 

```
const __u32 outer_key = 42;
bpf_map_delete_elem(outer_map_fd, &outer_key);
```
All inner map resources should be released after `bpf_map_delete_elem` as long
as you have avoided all the memory leak risks mentioned above. I would highly
recommend you do some stressful tests including bulk insertion/deletion to make
sure there is no memory leak, which w had in our project once.

# Code
Complete code can be found on [github](https://github.com/hechaoli/libbpf_map_in_map.git).
