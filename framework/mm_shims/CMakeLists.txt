# Copyright 2024 Morse Micro
# SPDX-License-Identifier: Apache-2.0
set(src "mmosal_shim_freertos_esp32.c"
        "mmhal.c"
        "mmhal_wlan_binaries.c"
        "wlan_hal.c"
        "crypto_mbedtls_mm.c")
set(inc "include")

idf_component_register(INCLUDE_DIRS ${inc}
                       SRCS ${src}
                       PRIV_REQUIRES morselib spi_flash app_update log driver mbedtls
                       WHOLE_ARCHIVE)

set(FW_MBIN "${CMAKE_CURRENT_LIST_DIR}/../morsefirmware/mm6108.mbin")
set(FW_OBJ "${FW_MBIN}.o")

# Kconfig variables are used to determine which bcf file to add to the final application
if(CONFIG_MM_BCF_MF08651_US)
    message(STATUS "Using BCF for MM6108_MF08651_US")
    set(BCF_MBIN "${CMAKE_CURRENT_LIST_DIR}/../morsefirmware/bcf_mf08651_us.mbin")
elseif(CONFIG_MM_BCF_MF08551)
    message(STATUS "Using BCF for MM6108_MF08551")
    set(BCF_MBIN "${CMAKE_CURRENT_LIST_DIR}/../morsefirmware/bcf_mf08551.mbin")
elseif(CONFIG_MM_BCF_MF08251)
    message(STATUS "Using BCF for MM6108_MF08251")
    set(BCF_MBIN "${CMAKE_CURRENT_LIST_DIR}/../morsefirmware/bcf_mf08251.mbin")
elseif(CONFIG_MM_BCF_MF03120)
    message(STATUS "Using BCF for MM6108_MF03120")
    set(BCF_MBIN "${CMAKE_CURRENT_LIST_DIR}/../morsefirmware/bcf_mf03120.mbin")
elseif(CONFIG_MM_BCF_AW_HM593)
    message(STATUS "Using BCF for MM6108_AW_HM593")
    set(BCF_MBIN "${CMAKE_CURRENT_LIST_DIR}/../morsefirmware/bcf_aw_hm593.mbin")
elseif(CONFIG_MM_BCF_AW_HM593_4V3)
    message(STATUS "Using BCF for MM6108_AW_HM593_4V3")
    set(BCF_MBIN "${CMAKE_CURRENT_LIST_DIR}/../morsefirmware/bcf_aw_hm593_4v3.mbin")
elseif(CONFIG_MM_BCF_FGH100MAAMD)
    message(STATUS "Using BCF for MM6108_FGH100MAAMD")
    set(BCF_MBIN "${CMAKE_CURRENT_LIST_DIR}/../morsefirmware/bcf_fgh100maamd.mbin")
elseif(CONFIG_MM_BCF_FGH100MABMD)
    message(STATUS "Using BCF for MM6108_FGH100MABMD")
    set(BCF_MBIN "${CMAKE_CURRENT_LIST_DIR}/../morsefirmware/bcf_fgh100mabmd.mbin")
else()
    message(FATAL_ERROR "No BCF specified for mm_shims")
endif()

set(BCF_OBJ "${BCF_MBIN}.o")

# Define the architecture and other variables
set(BFDNAME "elf32-xtensa-le")
set(ARCH "xtensa")

function(make_binary_object __file __prefix)
    get_filename_component(__file_name ${__file} NAME)
    set(__object ${CMAKE_CURRENT_LIST_DIR}/../morsefirmware/${__file_name}.o)
    string(MAKE_C_IDENTIFIER ${__file} __file_c_identifier)
    # message(FATAL_ERROR ${__object}; ${CMAKE_OBJCOPY})
    add_custom_command(
        OUTPUT ${__object}
        COMMAND ${CMAKE_OBJCOPY} -I binary -O ${BFDNAME} -B ${ARCH} ${__file} ${__object}
            --redefine-sym _binary_${__file_c_identifier}_start=${__prefix}_start
            --redefine-sym _binary_${__file_c_identifier}_size=${__prefix}_size
            --redefine-sym _binary_${__file_c_identifier}_end=${__prefix}_end
            --rename-section .data=.rodata._fw_mbin,contents,alloc,load,readonly,data
        DEPENDS ${__file}
        COMMENT "Running objcopy on ${__file} to generate ${__object}"
    )
endfunction()

make_binary_object(${FW_MBIN} firmware_binary)
make_binary_object(${BCF_MBIN} bcf_binary)
add_custom_target(morsefirmware ALL DEPENDS ${FW_OBJ} ${BCF_OBJ})

add_dependencies(${COMPONENT_TARGET} morsefirmware)

target_link_libraries(${COMPONENT_TARGET} PRIVATE
    "${FW_OBJ}"
    "${BCF_OBJ}"
)

