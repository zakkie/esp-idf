idf_build_get_property(target IDF_TARGET)

set(srcs)

if(CONFIG_IDF_ENV_FPGA)
    list(APPEND srcs "fpga_overrides.c")
endif()

if(BOOTLOADER_BUILD)
    # Bootloader relies on some Kconfig options defined in esp_system.
    idf_component_register(SRCS "${srcs}")
else()
    list(APPEND srcs "crosscore_int.c"
            "esp_err.c"
            "freertos_hooks.c"
            "int_wdt.c"
            "panic.c"
            "esp_system.c"
            "startup.c"
            "system_time.c"
            "stack_check.c"
            "task_wdt.c")

    if(NOT (${target} STREQUAL "esp32c3") )
        list(APPEND srcs  "dbg_stubs.c")
    endif()

    idf_component_register(SRCS "${srcs}"
                        INCLUDE_DIRS include
                        PRIV_REQUIRES spi_flash
                                    # [refactor-todo] requirements due to init code,
                                    # should be removable once using component init functions
                                    # link-time registration is used.
                                    esp_pm app_update nvs_flash pthread app_trace esp_gdbstub
                                    espcoredump
                        LDFRAGMENTS "linker.lf" "app.lf")
    add_subdirectory(port)

    # After system initialization, `start_app` (and its other cores variant) is called.
    # This is provided by the user or from another component. Since we can't establish
    # dependency on what we don't know, force linker to not drop the symbol regardless
    # of link line order.
    target_link_libraries(${COMPONENT_LIB} INTERFACE "-u start_app")

    if(NOT CONFIG_ESP_SYSTEM_SINGLE_CORE_MODE)
        target_link_libraries(${COMPONENT_LIB} INTERFACE "-u start_app_other_cores")
    endif()

    # Disable stack protection in files which are involved in initialization of that feature
    set_source_files_properties(
        "startup.c" "stack_check.c"
        PROPERTIES COMPILE_FLAGS
        -fno-stack-protector)

endif()

if(CONFIG_IDF_ENV_FPGA)
    # Forces the linker to include fpga stubs from this component
    target_link_libraries(${COMPONENT_LIB} INTERFACE "-u esp_common_include_fpga_overrides")
endif()
