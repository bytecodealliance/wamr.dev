---
title: "Transferring data from wasm apps to host : Part 1"
description: ""
excerpt: ""
date: 2023-04-10T18:24:33+08:00
lastmod: 2023-04-10T18:24:33+08:00
draft: false
weight: 50
images: []
categories: []
tags: []
contributors: ['Li Jiongqiang']
pinned: false
homepage: false
---

#### Overview: From Wasm to Native
This section will introduce how to transfer data from `the wasm world` to `the native world`.

This topic has a lot of content, so this article will be divided into 2 parts.
* The part 1 mainly discusses `the structure`.
* The part 2 mainly discussed `the function pointer`.

Unlike the case of "Transferring data from host to wasm apps", `the native context` can access memory from both worlds, so in many cases we can avoid copying memory.

#### 1 Integer, floating point or C-style string
(1) Integer and floating point
Integer and floating point can be directly passed and used.

(2) C-style string
According different parameters used when registering native apis, C-style strings can be either a native pointer and directly used in `the native layer`, or actually a wasm offset and be used after being converted to a native pointer by using WAMR's api `wasm_runtime_addr_app_to_native`.  

#### 2 Structure, and the memory layout is consistent
The native context can directly use the structure pointer passed by the wasm context to access members of the structure when the structure memory layout is consistent between two worlds.

But to ensure the structure memory layout is consistent, we need to do some assertion while compiling the host program and wasm apps.
We use `static_assert` to achieve this goal.
```cpp
//An example of using structure pointer with same layout in native

//32bits alignment, pack(4)
struct Node {
    double value;
    int day;
};
//assertions for ensuring the consistency of the memory layout 
static_assert(
    sizeof(struct Node) == 12
    && offsetof(struct Node, value) == 0
    && offsetof(struct Node, day) == 8
);

//the native wrapper for wasm call native
//register signature: "(*i)"
void setDay(wasm_exec_env_t exec_env, struct Node *data, int day){
    data->day = day;
}
```
#### 3 Structure, and the memory layout is inconsistent
The native context can't directly access some members (usually pointers) of the structure when the structure memory layout is inconsistent.

So when will the memory layout be inconsistent? 
There may be three reasons for this.
* The structure contains pointer members
* The alignment rules of two worlds are not inconsistent
* The pointer sizes of the two worlds are not inconsistent.
This section mainly discusses how to solve **the situation where a structure contains pointer members**.

The following discussion is based on **the premise of consistent pointer size and alignment rule**s.

```cpp
// The difference between native structure and wasm structure

// native structure in the native context
struct Node {
    int *data; // is a native pointer
    struct Node *next;// is a native pointer
    int size;
};

// wasm structure in the native context
struct Node {
    int *data;// is a offset
    struct Node *next;// is a offset
    int size;
};
```

**If you don't want to avoid memory copying, you can try the following two methods.**

##### (1) Copy the entire structure

You can construct a new structure that belongs to `the native world` and copy the memory from the wasm structure, and then you also need to determine the owner of this new memory.
```cpp
// An example of copying from WASM to native
struct Array{
    // the data of array
    int *data;
    // the size of array
    int size;
};
struct Array *copy_array_from_wasm(wasm_module_inst_t wasm_module_inst, uint32_t offset_array){
    struct Array *wasm_array = NULL;
    int *wasm_array_data = NULL;

    struct Array *native_array = NULL;
    int *native_array_data = NULL;

    uint32_t total_size = 0;
    uint8_t *saved_buffer = NULL;
    uint8_t *now_buffer = NULL;

    if(!offset_array) {
        goto fail0;
    }
    //convert pointer
    wasm_array = (struct Array*)wasm_runtime_addr_app_to_native(
                        wasm_module_inst, offset_array);
    if(!wasm_array){
        goto fail0;
    }
    if(wasm_array->data && 0 != wasm_array->size) {
        wasm_array_data = (int *)wasm_runtime_addr_app_to_native(
                                wasm_module_inst, (uint32_t)wasm_array->data);
    }

    //caculate the size of level 1
    total_size += sizeof(struct Array);

    //caculate the size of level 2
    total_size += sizeof(int) * wasm_array->size;

    //allocate memory
    now_buffer = saved_buffer = (uint8_t*)hostMalloc(total_size);
    if(!now_buffer) {
        goto fail0;
    }
    //copy level 1
    native_array = (struct Array *)now_buffer;
    now_buffer += sizeof(struct Array);
    native_array->size = wasm_array->size;
    native_array->data = NULL;
    //copy level 2
    if (wasm_array_data) {
        native_array_data = (int *)now_buffer;
        now_buffer += wasm_array->size;
        memcpy(native_array_data, wasm_array_data, wasm_array->size);
    }
    //link level 2 to level 1
    native_array->data = native_array_data;

    //check the size of copied memory
    if (saved_buffer + total_size != now_buffer) {
        goto fail1;
    }

    //return result
    return native_array;
fail1:
    hostFree((void*)saved_buffer);
fail0:
    return NULL;
}
```

##### (2) Serialization

You can serialized structures, but you need to decide who owns the message being passed.

**If you don't want to copy or partially copy memory, you can try the following 4 methods.**

##### (3) change the code for accessing members of a structure in `the native layer`

If you don't directly access the pointer member, but use the WAMR's API `wasm_runtime_addr_app_to_native` to access the pointer, you can avoid copying memory.

##### (4) Temporarily convert pointer members of a wasm structure to native pointers

The disadvantage of doing so is that it may not be possible to achieve concurrent access, or you may need to set up a mutual exclusion mechanism when conducting concurrent access.

```cpp
// Temprarily converting pointer
// Note: The WASM app will call native api sumWrapper

//linked list node
struct Node {
    int data;
    struct Node *next;
};
// the native wrapper layer
int sumWrapper(wasm_exec_env_t exec_env, struct Node *first) {
    int ans = 0;
    struct Node *now = first;
    struct Node *tmp;
    wasm_module_inst_t wasm_module_inst = wasm_runtime_get_module_inst(exec_env);
    // Temprarily converting
    while(now) {
        now->next = (int*)wasm_runtime_addr_app_to_native(wasm_module_inst, (uint32_t)now->next);
        now = now->next;
    }
    ans = sum(first);
    now = first;
    // Recovering
    while(now) {
        tmp = now->next;
        now->next = (int*)wasm_runtime_addr_native_to_app(wasm_module_inst, (void *)now->next);
        now = tmp;
    }
    return ans;
}
// the native layer
int sum(struct Node *first) {
    int ans = 0;
    while(first){
        ans += first->data;
        first  = first->next;
    }
    return ans;
}
```
##### (5) partially copy memory
We can temprarily construct a new structure, but just copy a part of the original structure.
```cpp
// Note: The WASM app will call native api sumWrapper

struct Array {
    int *data;
    int size;
};

// the native wrapper for wasm call native
int sumWrapper(wasm_exec_env_t exec_env, struct Array *now) {
    int ans = 0;
    struct Array tmp;
    wasm_module_inst_t wasm_module_inst;
    if (!now || !exec_env
          || !(wasm_module_inst = wasm_runtime_get_module_inst(exec_env))) {
        assert(false);
    }
    tmp = *now;
    tmp.data = (int*)wasm_runtime_addr_app_to_native(wasm_module_inst, (uint32_t)tmp.data);
    return sum(&tmp);
}
// the native wrapper
int sum(struct Array *now) {
    int ans = 0;
    int size = now->size;
    while(size--) {
        ans += now->data[size];
    }
    return ans;
}
```
##### (6) move the structure to the native world and use handle
You can store structure in `the native world` and use `handle` + `wasm call native`in `the WASM context` to operate on this structure.
