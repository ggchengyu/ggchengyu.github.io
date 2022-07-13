---
title: deep-in-libvmi
date: 2022-07-11 17:08:59
tags:
- qemu-patch
categories:
- virtualization
Description:
- Deep in libVMI
---

```c
status_t vmi_init(
    vmi_instance_t *vmi,
    vmi_mode_t mode,
    const void *domain,
    uint64_t init_flags,
    vmi_init_data_t *init_data,
    vmi_init_error_t *error)
{
    // 检测libVMI是否允许该模式
    if ( VMI_FAILURE == driver_sanity_check(mode) ) {
        errprint("The selected LibVMI mode is not available!\n");
        return VMI_FAILURE;
    }
    ...
    // 初始化vmi_instance
    // glib是真的强，特别多开源项目用
    vmi_instance_t _vmi = (vmi_instance_t) g_try_malloc0(sizeof(struct vmi_instance));

    // 初始化flags和page_mode
    // 
    // page mode受对象的影响，不同架构的page mapping有区别；先初始化为UNKOWN
    _vmi->init_flags = init_flags;
    _vmi->page_mode = VMI_PM_UNKNOWN;

    // 初始化查table的函数指针数组
    arch_init_lookup_tables(_vmi);

    // init_data <-> vmi->memmap
    // init_data->count 统计条目数量
    // 将 VMI_INIT_DATA_MEMMAP 的条目拷贝到 vmi->memmap中
    if ( init_data && init_data->count ) {
        uint64_t i;
        for (i=0; i < init_data->count; i++) {
            switch (init_data->entry[i].type) {
                case VMI_INIT_DATA_MEMMAP:
                    _vmi->memmap = (memory_map_t*)g_memdup_compat(init_data->entry[i].data, sizeof(memory_map_t));
                    if ( !_vmi->memmap )
                        goto error_exit;
                    break;
                default:
                    break;
            };
        }
    }

    // 简单初始化page， shift为12，size为4KB
    if (VMI_FAILURE == init_page_offset(_vmi)) {
        if ( error )
            *error = VMI_INIT_ERROR_DRIVER;

        goto error_exit;
    }

    // 先根据 vmi 的类型初始化 driver 结构各函数指针， vmi->driver = driver
    // 成功后（vmi->driver && vmi->driver.init_ptr)，调用 vmi->driver.init_ptr(vmi, init_flags, init_data)
    // unused(init_flags)和unused(inti_data)...有点无语
    // init_ptr会初始化一个${arch}_instance_t，并初始化相关内容
    if (VMI_FAILURE == driver_init(_vmi, init_flags, init_data)) {
        if ( error )
            *error = VMI_INIT_ERROR_DRIVER;

        goto error_exit;
    }
    ...
    
    if (init_flags & VMI_INIT_DOMAINWATCH) {
        if ( VMI_FAILURE == driver_domainwatch_init(_vmi, init_flags) ) {
            if ( error )
                *error = VMI_INIT_ERROR_DRIVER;
            goto error_exit;
        }

        if ( init_flags == VMI_INIT_DOMAINWATCH ) {
            /* we have all we need to wait for domains. Return if there is nothing else */
            *vmi = _vmi;
            return VMI_SUCCESS;
        }
    }

    /* resolve the id and name */
    if (VMI_FAILURE == set_id_and_name(_vmi, domain)) {
        if ( error )
            *error = VMI_INIT_ERROR_VM_NOT_FOUND;

        goto error_exit;
    }

    /* init vmi for specific file/domain through the driver */
    if (VMI_FAILURE == driver_init_vmi(_vmi, init_flags, init_data)) {
        if ( error )
            *error = VMI_INIT_ERROR_DRIVER;

        goto error_exit;
    }

    /* get the memory size */
    if (driver_get_memsize(_vmi, &_vmi->allocated_ram_size, &_vmi->max_physical_address) == VMI_FAILURE) {
        if ( error )
            *error = VMI_INIT_ERROR_DRIVER;

        goto error_exit;
    }

    /* setup the caches */
    pid_cache_init(_vmi);
    sym_cache_init(_vmi);
    rva_cache_init(_vmi);
    v2p_cache_init(_vmi);

    status = VMI_SUCCESS;

    dbprint(VMI_DEBUG_CORE, "**set allocated_ram_size = %"PRIx64", "
            "max_physical_address = 0x%"PRIx64"\n",
            _vmi->allocated_ram_size,
            _vmi->max_physical_address);

    if ( init_flags & VMI_INIT_EVENTS ) {
        status = events_init(_vmi);
        if ( error && VMI_FAILURE == status )
            *error = VMI_INIT_ERROR_EVENTS;
    }

error_exit:
    if ( VMI_FAILURE == status ) {
        vmi_destroy(_vmi);
        *vmi = NULL;
    } else
        *vmi = _vmi;

    return status;
}
```