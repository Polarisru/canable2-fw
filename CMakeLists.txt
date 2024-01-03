# usage:    cmake -G "MinGW Makefiles" . -Bbuild
#           cmake --build ./build
cmake_minimum_required(VERSION 3.5)
set(CMAKE_TOOLCHAIN_FILE arm-none-eabi-gcc.cmake)
project(canable2)

enable_language(C ASM)
set(CMAKE_C_STANDARD 99)
set(CMAKE_C_STANDARD_REQUIRED ON)
#set(CMAKE_C_EXTENSIONS OFF)

execute_process(COMMAND git log --pretty=format:%h -n 1
                OUTPUT_VARIABLE GIT_VERSION
                ERROR_QUIET)
                
execute_process(COMMAND git config --get remote.origin.url
                OUTPUT_VARIABLE GIT_REMOTE
                ERROR_QUIET)

set(EXEC ${PROJECT_NAME}-${GIT_VERSION})

# set output directory for generated binary file
set(CMAKE_RUNTIME_OUTPUT_DIRECTORY output)
# include directory with headers
include_directories(src 
  Drivers/CMSIS/Include 
  Drivers/CMSIS/Device/ST/STM32G4xx/Include
  Drivers/STM32G4xx_HAL_Driver/Inc
  inc
  Middlewares/ST/STM32_USB_Device_Library/Core/Inc
  Middlewares/ST/STM32_USB_Device_Library/Class/CDC/Inc 
  src
  )
# add all project files to sources
set(sources 
  src/can.c
  src/error.c
  src/interrupts.c
  src/led.c
  src/main.c
  src/printf.c
  src/slcan.c
  src/system.c
  src/system_stm32g4xx.c
  src/usb_device.c
  src/usbd_cdc_if.c
  src/usbd_conf.c
  src/usbd_desc.c
  )
file(GLOB Middlewares/ST/STM32_USB_Device_Library/Core/Src/*.c)
file(GLOB Middlewares/ST/STM32_USB_Device_Library/Class/CDC/Src/*.c)

# generate binary file from the list of sources
add_executable (${EXEC} ${sources})

target_compile_definitions(${EXEC} PRIVATE
    -D HSI48_VALUE=48000000 
    -D HSE_VALUE=16000000
    -D INTERNAL_OSCILLATOR
    -D STM32G431xx
    -D GIT_VERSION=${GIT_VERSION}
    -D GIT_REMOTE=${GIT_REMOTE}
    )

set(compile_options
    -mcpu=cortex-m4
    -mthumb
    -Wall 
    -g 
    -ffunction-sections 
    -fdata-sections 
    -Os
    )
    
set(link_options
    -TSTM32G431CBTx_FLASH.ld
    -mcpu=cortex-m4
    -mthumb
    -mfloat-abi=soft
    -lm
    -lc
    -lnosys
    -fno-exceptions
    -ffunction-sections
    -fdata-sections
    -Wl,--gc-sections
    )

function(set_dependencies target_name)
    target_compile_options(${target_name} PUBLIC ${compile_options})
    target_link_options(${target_name} PUBLIC ${link_options})
    target_link_libraries(${target_name} m)
    add_custom_command(TARGET ${target_name}
        POST_BUILD
        COMMAND ${CMAKE_SIZE_UTIL} ${CMAKE_RUNTIME_OUTPUT_DIRECTORY}/${target_name})
    add_custom_command(TARGET ${target_name}
        POST_BUILD
        COMMAND ${CMAKE_OBJCOPY} -O ihex ${CMAKE_RUNTIME_OUTPUT_DIRECTORY}/${target_name} ${CMAKE_RUNTIME_OUTPUT_DIRECTORY}/${target_name}.hex
    )
endfunction()

set_dependencies(${EXEC})
