cmake_minimum_required(VERSION 2.8.12)

find_package(PythonInterp 3.4)

if (NOT DEFINED NRF52_SDK_ROOT)
    message( FATAL_ERROR " define NRF52_SDK_ROOT to point to the root of the SDK installation location!" )
endif()

if (NOT DEFINED BLUETOE_ROOT)
    message( FATAL_ERROR " define BLUETOE_ROOT to point to the root of the bluetoe installation location!" )
endif()

set(CXX_DEFAULT_FLAGS "-Wall")
set(CXX_SPECIFIC_FLAGS "-std=c++11 -fno-threadsafe-statics")
set(C_SPECIFIC_FLAGS "-std=c99")

set(CXX_DISABLED_WARNINGS "-Wno-parentheses -Wno-array-bounds")

# Stop compilation after first error
set(CXX_DEFAULT_FLAGS "${CXX_DEFAULT_FLAGS} -Wfatal-errors -Werror")

set(CMAKE_SYSTEM_NAME Generic)
set(CMAKE_CXX_COMPILER "/usr/local/gcc-arm-none-eabi-6-2017-q2-update/bin/arm-none-eabi-g++")
set(CMAKE_C_COMPILER "/usr/local/gcc-arm-none-eabi-6-2017-q2-update/bin/arm-none-eabi-gcc")
set(CMAKE_ASM_COMPILER "/usr/local/gcc-arm-none-eabi-6-2017-q2-update/bin/arm-none-eabi-gcc")
set(CMAKE_C_COMPILER_FORCED 1)
set(CMAKE_CXX_COMPILER_FORCED 1)
set(ARM_FLAGS "--sysroot=/usr/local/arm/arm-none-eabi -mthumb -mabi=aapcs -mcpu=cortex-m4")
set(ARM_FLAGS "${ARM_FLAGS} -ffunction-sections -fdata-sections")
set(CXX_ARC_FLAGS "${ARM_FLAGS} -mfloat-abi=hard -mfpu=fpv4-sp-d16")
set(LINK_ARC_FLAGS "${ARM_FLAGS} --specs=nosys.specs --specs=nano.specs -static -lm -lstdc++ -lsupc++")
set(CXX_SPECIFIC_FLAGS "${CXX_SPECIFIC_FLAGS} -ftemplate-backtrace-limit=0 -fvisibility-inlines-hidden -fno-rtti -fno-exceptions")
set(CXX_DISABLED_WARNINGS "${CXX_DISABLED_WARNINGS} -Wno-unused-local-typedefs")
set(LINK_DEFAULT_FLAGS "${LINK_DEFAULT_FLAGS} -Wl,--gc-sections -L ${CMAKE_SOURCE_DIR}/examples")

# debugging
set(CXX_BUILD_TYPE_FLAGS "-g -O0")
set(LINK_BUILD_TYPE_FLAGS "-g")

project(discover_service_by_uuid CXX ASM C)

set(CMAKE_CXX_FLAGS "${CXX_DEFAULT_FLAGS} ${CXX_SPECIFIC_FLAGS} ${CXX_ARC_FLAGS} ${CXX_BUILD_TYPE_FLAGS} ${CXX_DISABLED_WARNINGS}")
set(CMAKE_C_FLAGS   "${CXX_DEFAULT_FLAGS} ${C_SPECIFIC_FLAGS} ${CXX_ARC_FLAGS} ${CXX_BUILD_TYPE_FLAGS} ${CXX_DISABLED_WARNINGS}")
set(CMAKE_EXE_LINKER_FLAGS "${LINK_DEFAULT_FLAGS} ${LINK_ARC_FLAGS} ${LINK_BUILD_TYPE_FLAGS} -L ${CMAKE_SOURCE_DIR}")

add_definitions(-D_LITTLE_ENDIAN=1)
add_definitions(-DNRF52 -DNRF52832_XXAA)
add_definitions(-DBOARD_PCA10040)
add_definitions(-D__HEAP_SIZE=0)
#add_definitions(-DNDEBUG=1)
add_definitions(-DBLUETOE_NRF51_RADIO_DEBUG=1)

set(DEFAULT_JLINK_DEVICE_NAME nRF52832_xxAA)

include_directories(${CMAKE_CURRENT_SOURCE_DIR})

include_directories(${BLUETOE_ROOT}/)
include_directories(${NRF52_SDK_ROOT}/components/device/)
include_directories(${NRF52_SDK_ROOT}/components/toolchain/gcc/)
include_directories(${NRF52_SDK_ROOT}/components/toolchain/cmsis/include/)

if (DEFINED JLINK_DEVICE)
    message( INFO " JLink ${JLINK_DEVICE} selected")
    set(SELECT_JLINK_DEVICE "usb ${JLINK_DEVICE}")
else()
    message( WARNING " no JLink selected (use \"JLINK_DEVICE\")" )
    set(SELECT_JLINK_DEVICE "")
endif()

function(new_executable TARGETNAME)
    add_executable(${TARGETNAME}.elf
        ${TARGETNAME}.cpp
        system_nrf52.c
        gcc_startup_nrf52.s
        ${BLUETOE_ROOT}/bluetoe/link_layer/delta_time.cpp
        ${BLUETOE_ROOT}/bluetoe/link_layer/address.cpp
        ${BLUETOE_ROOT}/bluetoe/link_layer/channel_map.cpp
        ${BLUETOE_ROOT}/bluetoe/link_layer/connection_details.cpp
        ${BLUETOE_ROOT}/bluetoe/bindings/nrf51.cpp
    )

    set_target_properties(${TARGETNAME}.elf PROPERTIES
        LINK_FLAGS "-T ${CMAKE_CURRENT_LIST_DIR}/nrf52.ld -Wl,-Map -Wl,${TARGETNAME}.map")

    add_custom_target(${TARGETNAME}.bin
        /usr/local/gcc-arm-none-eabi-6-2017-q2-update/bin/arm-none-eabi-objcopy -S -O binary ${TARGETNAME}.elf ${TARGETNAME}.bin
        )
    add_custom_target(${TARGETNAME}.hex
        /usr/local/gcc-arm-none-eabi-6-2017-q2-update/bin/arm-none-eabi-objcopy -S -O ihex ${TARGETNAME}.elf ${TARGETNAME}.hex
        )
    add_dependencies(${TARGETNAME}.bin ${TARGETNAME}.elf)
    add_dependencies(${TARGETNAME}.hex ${TARGETNAME}.elf)

    add_custom_target(${TARGETNAME}.flash
        COMMAND nrfjprog --family NRF52 --sectorerase --snr ${JLINK_DEVICE} --program ${TARGETNAME}.hex --verify --reset
        )

    add_dependencies(${TARGETNAME}.flash ${TARGETNAME}.hex)

    add_custom_target(${TARGETNAME}.lis
        /usr/local/gcc-arm-none-eabi-6-2017-q2-update/bin/arm-none-eabi-objdump -S ${TARGETNAME}.elf > ${TARGETNAME}.lis
        )

    add_dependencies(${TARGETNAME}.lis ${TARGETNAME}.elf)
endfunction()

add_custom_target(erase.flash
    COMMAND nrfjprog --family NRF52 --snr ${JLINK_DEVICE} --eraseall
    )


new_executable(blinky)
