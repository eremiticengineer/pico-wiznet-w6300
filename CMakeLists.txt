cmake_minimum_required(VERSION 3.13)

set(LIBRARY pico_wiznet_w6300)

if(CMAKE_SOURCE_DIR STREQUAL CMAKE_CURRENT_SOURCE_DIR)
    set(PICO_WIZNET_W6300_STANDALONE TRUE)
else()
    set(PICO_WIZNET_W6300_STANDALONE FALSE)
endif()

set(WIZNET_PORT_DIR
    "${CMAKE_CURRENT_LIST_DIR}/lib/WIZnet-PICO-FREERTOS-C/port"
)

set(WIZNET_IOLIB_DIR
    "${CMAKE_CURRENT_LIST_DIR}/lib/ioLibrary_Driver"
)

if(PICO_WIZNET_W6300_STANDALONE)
    set(PROJECT pico_wiznet_w6300_example)

    # W6300-EVB-Pico2 uses the RP2350A. The WIZnet-specific pin mapping is
    # selected by DEVICE_BOARD_NAME below; pico2 supplies the RP2350 platform.
    set(PICO_BOARD pico2 CACHE STRING "Board type" FORCE)

    include($ENV{PICO_SDK_PATH}/external/pico_sdk_import.cmake)

    set(FREERTOS_KERNEL_PATH
        "${CMAKE_CURRENT_LIST_DIR}/lib/FreeRTOS-Kernel"
    )

    set(FREERTOS_CONFIG_FILE_DIRECTORY
        "${CMAKE_CURRENT_LIST_DIR}/examples/include"
    )

    include(
        ${FREERTOS_KERNEL_PATH}/portable/ThirdParty/GCC/RP2040/FreeRTOS_Kernel_import.cmake
    )

    set(PICO_MBEDTLS_CONFIG_FILE
        "${CMAKE_CURRENT_LIST_DIR}/examples/include/mbedtls_config.h"
    )

    project(${PROJECT} C CXX ASM)

    set(CMAKE_C_STANDARD 11)
    set(CMAKE_CXX_STANDARD 23)

    pico_sdk_init()
endif()

if(NOT EXISTS "${WIZNET_IOLIB_DIR}/Ethernet/wizchip_conf.c")
    message(FATAL_ERROR "ioLibrary_Driver not found at ${WIZNET_IOLIB_DIR}")
endif()

if(NOT EXISTS "${WIZNET_IOLIB_DIR}/Ethernet/W6300/w6300.c")
    message(FATAL_ERROR
        "This ioLibrary_Driver does not contain W6300 support. "
        "Update lib/ioLibrary_Driver to a current WIZnet release."
    )
endif()

if(NOT EXISTS "${WIZNET_PORT_DIR}/ioLibrary_Driver/src/wizchip_qspi_pio.c")
    message(FATAL_ERROR
        "The W6300 QSPI/PIO port was not found. Update "
        "lib/WIZnet-PICO-FREERTOS-C to a version supporting W6300-EVB-Pico2."
    )
endif()

# WIZnet documentation has used both names for this PIO source in different
# revisions. Accept either so the project works with both layouts.
if(EXISTS "${WIZNET_PORT_DIR}/ioLibrary_Driver/src/wizchip_qspi_pio.pio")
    set(WIZNET_QSPI_PIO
        "${WIZNET_PORT_DIR}/ioLibrary_Driver/src/wizchip_qspi_pio.pio"
    )
elseif(EXISTS "${WIZNET_PORT_DIR}/ioLibrary_Driver/src/wizchip_spi_pio.pio")
    set(WIZNET_QSPI_PIO
        "${WIZNET_PORT_DIR}/ioLibrary_Driver/src/wizchip_spi_pio.pio"
    )
else()
    message(FATAL_ERROR "W6300 QSPI PIO program not found in ${WIZNET_PORT_DIR}")
endif()

add_library(pico_wiznet_w6300_driver STATIC
    ${WIZNET_IOLIB_DIR}/Ethernet/socket.c
    ${WIZNET_IOLIB_DIR}/Ethernet/wizchip_conf.c
    ${WIZNET_IOLIB_DIR}/Ethernet/W6300/w6300.c
    ${WIZNET_IOLIB_DIR}/Internet/DHCP/dhcp.c
    ${WIZNET_IOLIB_DIR}/Internet/DNS/dns.c
    ${WIZNET_PORT_DIR}/ioLibrary_Driver/src/wizchip_spi.c
    ${WIZNET_PORT_DIR}/ioLibrary_Driver/src/wizchip_qspi_pio.c
)

pico_generate_pio_header(
    pico_wiznet_w6300_driver
    ${WIZNET_QSPI_PIO}
)

target_compile_definitions(pico_wiznet_w6300_driver PUBLIC
    _WIZCHIP_=W6300
    DEVICE_BOARD_NAME=W6300_EVB_PICO2
    _WIZCHIP_SPI_SCLK_SPEED=43
    _WIZCHIP_QSPI_MODE_=QSPI_QUAD_MODE
    USE_PIO
    SIO_IRQ_FLAG=1
)

target_include_directories(pico_wiznet_w6300_driver PUBLIC
    ${WIZNET_IOLIB_DIR}/Ethernet
    ${WIZNET_IOLIB_DIR}/Ethernet/W6300
    ${WIZNET_IOLIB_DIR}/Internet/DHCP
    ${WIZNET_IOLIB_DIR}/Internet/DNS
    ${WIZNET_PORT_DIR}
    ${WIZNET_PORT_DIR}/ioLibrary_Driver/inc
)

target_link_libraries(pico_wiznet_w6300_driver PUBLIC
    pico_stdlib
    hardware_clocks
    hardware_dma
    hardware_spi
    hardware_pio
    FreeRTOS-Kernel
)

add_library(${LIBRARY}
    src/WiznetW6300.cpp
)

target_include_directories(${LIBRARY} PUBLIC
    ${CMAKE_CURRENT_LIST_DIR}/src
)

if(PICO_WIZNET_W6300_STANDALONE)
    target_include_directories(${LIBRARY} PRIVATE
        ${CMAKE_CURRENT_LIST_DIR}/examples/include
        ${CMAKE_CURRENT_LIST_DIR}/examples/config
    )
endif()

target_link_libraries(${LIBRARY} PUBLIC
    pico_stdlib
    pico_mbedtls
    pico_wiznet_w6300_driver
    FreeRTOS-Kernel
)

if(PICO_WIZNET_W6300_STANDALONE)
    if(NOT EXISTS "${CMAKE_CURRENT_LIST_DIR}/examples/config/secrets.hpp")
        message(FATAL_ERROR
            "Copy examples/config/secrets.example.hpp to examples/config/secrets.hpp and edit it."
        )
    endif()

    add_executable(${PROJECT}
        examples/main.cpp
    )

    target_include_directories(${PROJECT} PRIVATE
        ${CMAKE_CURRENT_LIST_DIR}/examples/include
        ${CMAKE_CURRENT_LIST_DIR}/examples/config
    )

    target_link_libraries(${PROJECT} PRIVATE
        ${LIBRARY}
        pico_stdlib
        FreeRTOS-Kernel
        FreeRTOS-Kernel-Heap4
    )

    pico_enable_stdio_usb(${PROJECT} 1)
    pico_enable_stdio_uart(${PROJECT} 1)
    pico_add_extra_outputs(${PROJECT})
endif()
